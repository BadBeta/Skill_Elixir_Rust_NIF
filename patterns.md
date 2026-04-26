# Rust NIF Patterns — Specialized Templates

Production patterns extracted from real Rustler libraries, organized by domain. Loaded on-demand when working in a specialized area. For foundational templates (Resource Wrapper, Custom Error Type, catch_unwind/nif_safe, Tokio Runtime, NewBinary, Bare Returns, RAII `Mutex<Option<T>>`), see SKILL.md §2.4.

Each pattern is sourced from one or more of the 15 production crates listed in SKILL.md §2.4 evidence base.

## Index

| Domain | Pattern | Source |
|---|---|---|
| Foreign callbacks & term lifecycle | [`scoped_thread_local!` for sync callbacks](#scoped_thread_local-for-synchronous-foreign-callbacks-y_ex) | y_ex |
| Foreign callbacks & term lifecycle | [`TermBox` for long-lived term capture](#termbox-for-long-lived-opaque-term-capture-y_ex) | y_ex |
| Binary encoding optimizations | [`SliceIntoBinary` lazy encoder](#sliceintobinary-lazy-encoder-wrapper-y_ex) | y_ex |
| Binary encoding optimizations | [`Binary::make_subbinary` zero-copy slicing](#binarymake_subbinary-for-zero-copy-sub-region-returns-y_ex) | y_ex |
| Decoder/Encoder for foreign types | [Atom-valued enum wrappers](#atom-valued-enum-wrappers-via-custom-decoder--encoder-resvg_nif) | resvg_nif |
| Resource lifecycle — borrowed handles | [`mem::transmute` to `'static` caution](#caution-memtransmute-to-static-for-resource-stored-borrows-y_ex) | y_ex |
| Resource lifecycle — explicit cleanup | [Explicit-close NIF companion to RAII Drop](#explicit-close-nif-companion-to-raii-drop-midiex) | midiex |
| OS-specific patterns | [`thread_local!` for thread-affine OS handles](#thread_local-for-thread-affine-os-handles-midiex) | midiex |
| OS-specific patterns | [macOS `CFRunLoop` for hot-plug callbacks](#macos-hot-plug--notification-callbacks--spawn-thread--cfrunloop-midiex) | midiex |
| Crypto NIF patterns | [Fixed-size binary parse + `copy_from_slice`](#fixed-size-binary-parse--copy_from_slice-crypto-nif-pattern-ex_secp256k1) | ex_secp256k1 |
| Image/render NIF patterns | [User-controlled allocation size validation (DoS surface)](#user-controlled-allocation-size-validation-imagerender-nif-dos-surface) | resvg_nif |
| ML / Tensor backend patterns | [Tensor backend wrapper (NifStruct + atom device)](#tensor-backend-wrapper--nifstruct--resourcearc--atom-tagged-device--deref-candlex) | candlex |
| ML / Tensor backend patterns | [Per-dtype tensor binary encoding](#per-dtype-tensor-binary-encoding--to_ne_bytes--exhaustive-match-candlex) | candlex |
| WASM / plugin patterns | [Cancel-handle resource for long-running NIFs](#cancel-handle-resource-for-long-running-nifs-extism) | extism_elixir |
| Static handle caching | [Static lazy device-handle cache](#static-lazy-device-handle-cache--static-mutexoptiont-candlex) | candlex |
| Build & setup patterns | [`load` callback for one-time global setup](#load-callback-for-one-time-global-setup-rhai_rustler) | rhai_rustler |
| Build & setup patterns | [Embedded engine with sibling `cdylib` plugin crate](#embedded-engine-with-sibling-plugin-crate-rhai_rustler) | rhai_rustler |
| Error-shape macros | [`try_or_return_elixir_err!` macro](#try_or_return_elixir_err-macro-for-stringified-errors-resvg) | resvg_nif |
| Tokio runtime variants | [`LazyLock<Runtime>` + `RUNTIME.spawn` patterns](#tokio-runtime-variants--lazylock-and-runtimespawn-wasmex) | wasmex |

---

## Foreign callbacks & term lifecycle

### `scoped_thread_local!` for synchronous foreign callbacks (y_ex)

When a foreign library invokes a Rust callback synchronously during a NIF call (yrs observers fired during transaction commit, libxml2 SAX callbacks, etc.) and the callback needs to encode/send Elixir terms, threading the live `Env<'a>` through every API is impractical. Stash it in a scoped thread-local instead:

```rust
use rustler::{scoped_thread_local, Env, Term};

scoped_thread_local!(static ENV: Env<'static>);

// Wrapper called from your NIF: sets the slot for the duration of `f`,
// SAFETY-transmuting the lifetime because the slot only lives across a sync call.
pub fn with_env<F, R>(env: Env<'_>, f: F) -> R
where
    F: FnOnce() -> R,
{
    // SAFETY: ENV.set holds the value only for the synchronous duration of f.
    // We extend Env<'a> to Env<'static> to satisfy scoped_thread_local!'s
    // signature; the value is removed before f returns.
    let env_static: Env<'static> = unsafe { std::mem::transmute(env) };
    ENV.set(&env_static, f)
}

// In a NIF that triggers foreign callbacks
#[rustler::nif]
fn doc_apply_update(doc: ResourceArc<DocResource>, update: Binary) -> NifResult<Atom> {
    let env = doc.env();  // capture from the NIF's Env<'a>
    with_env(env, || {
        // Inside this scope, observer callbacks fired by yrs can recover Env via:
        //   ENV.with(|e| (atoms::change(), some_term).encode(*e))
        doc.apply_update(update.as_slice())?;
        Ok(atoms::ok())
    })
}
```

Reach for this pattern when:
- A foreign library calls back into your Rust during a NIF invocation
- The callback needs to construct an Elixir term to send/encode
- You'd otherwise have to thread `Env` through 5+ layers of the foreign API

If the callback fires asynchronously from a non-BEAM thread, use `OwnedEnv` + `send_and_clear` instead — the scoped TLS pattern is for synchronous callbacks only.

### `TermBox` for long-lived opaque term capture (y_ex)

When a subscription/observer needs to retain a user-supplied opaque `metadata: Term` across many future events (potentially after the originating NIF call returns), you can't store the raw `Term<'a>` — the `'a` lifetime ends with the NIF call. Wrap it in `OwnedEnv + SavedTerm`, behind a `Mutex` for `Sync`:

```rust
use rustler::env::{OwnedEnv, SavedTerm};
use rustler::{Env, Term};
use std::sync::Mutex;

pub struct TermBox(Mutex<(OwnedEnv, SavedTerm)>);

impl TermBox {
    pub fn new(env: Env<'_>, term: Term<'_>) -> Self {
        let owned = OwnedEnv::new();
        let saved = owned.save(term);
        Self(Mutex::new((owned, saved)))
    }

    /// Project the saved term back into the caller's Env.
    pub fn get<'a>(&self, env: Env<'a>) -> Term<'a> {
        let guard = self.0.lock().expect("TermBox lock");
        guard.0.run(|owned_env| guard.1.load(owned_env)).in_env(env)
    }
}

// Usage: subscription stores TermBox; on each event, projects into the live Env.
#[rustler::nif]
fn doc_observe(doc: ResourceArc<DocResource>, env: Env, metadata: Term)
    -> NifResult<ResourceArc<SubResource>>
{
    let boxed = TermBox::new(env, metadata);
    let sub = doc.observe(move |event_env, event_data| {
        let user_meta = boxed.get(event_env);
        // Send (event_data, user_meta) to subscriber...
    });
    Ok(ResourceArc::new(SubResource(sub)))
}
```

Never store a raw `Term<'a>` past the NIF call that produced it.

## Binary encoding optimizations

### `SliceIntoBinary` lazy encoder wrapper (y_ex)

When the value you'll return is built from a `&[u8]` you already own, allocating a `NewBinary` eagerly wastes work if the caller wraps the result in `Result`/`Option` and later discards it. A small `Encoder` wrapper defers allocation until rustler actually encodes:

```rust
pub struct SliceIntoBinary<'a> {
    bytes: &'a [u8],
}

impl<'a> SliceIntoBinary<'a> {
    pub fn new(bytes: &'a [u8]) -> Self { Self { bytes } }
}

impl<'a> rustler::Encoder for SliceIntoBinary<'a> {
    fn encode<'b>(&self, env: Env<'b>) -> Term<'b> {
        let mut new = NewBinary::new(env, self.bytes.len());
        new.as_mut_slice().copy_from_slice(self.bytes);
        Term::from(new)
    }
}

// Usage: NewBinary is only allocated if rustler reaches the encode call.
#[rustler::nif]
fn maybe_serialize(doc: ResourceArc<DocResource>) -> Option<SliceIntoBinary<'_>> {
    if doc.has_changes() {
        Some(SliceIntoBinary::new(doc.serialized_bytes()))
    } else {
        None
    }
}
```

### `Binary::make_subbinary` for zero-copy sub-region returns (y_ex)

When returning a contiguous sub-region of an input `Binary`, share the underlying refc-binary instead of allocating:

```rust
#[rustler::nif]
fn binary_slice(input: Binary, offset: usize, len: usize) -> NifResult<Binary> {
    input
        .make_subbinary(offset, len)
        .ok_or_else(|| Error::Term(Box::new("offset/len out of range")))
}
```

Useful for parsers that return offsets into the input — avoids allocating a fresh binary per result.

## Decoder / Encoder for foreign types

### Atom-valued enum wrappers via custom `Decoder + Encoder` (resvg_nif)

When you need an Elixir-side atom (`:fast` / `:precise` / `:bilinear` / ...) to map to an enum from a foreign crate you don't own, you can't `#[derive(NifUnitEnum)]` on it (orphan rule). Wrap it in a local newtype and implement `Decoder + Encoder` against `term.atom_to_string()?`:

```rust
use rustler::{Decoder, Encoder, Env, NifResult, Term};
use resvg::usvg::ShapeRendering;

// Local newtype around the foreign enum
pub struct ShapeRenderingWrapper(pub ShapeRendering);

impl<'a> Decoder<'a> for ShapeRenderingWrapper {
    fn decode(term: Term<'a>) -> NifResult<Self> {
        let atom = term.atom_to_string()?;
        let inner = match atom.as_str() {
            "optimize_speed" => ShapeRendering::OptimizeSpeed,
            "crisp_edges" => ShapeRendering::CrispEdges,
            "geometric_precision" => ShapeRendering::GeometricPrecision,
            _ => return Err(rustler::Error::BadArg),
        };
        Ok(ShapeRenderingWrapper(inner))
    }
}

impl Encoder for ShapeRenderingWrapper {
    fn encode<'a>(&self, env: Env<'a>) -> Term<'a> {
        match self.0 {
            ShapeRendering::OptimizeSpeed => atoms::optimize_speed().encode(env),
            ShapeRendering::CrispEdges => atoms::crisp_edges().encode(env),
            ShapeRendering::GeometricPrecision => atoms::geometric_precision().encode(env),
        }
    }
}

// Now the wrapper participates in NifStruct derives
#[derive(NifStruct)]
#[module = "Resvg.Options"]
pub struct Options {
    pub shape_rendering: ShapeRenderingWrapper,
    pub text_rendering: TextRenderingWrapper,
    // ...
}
```

Use this when:
- The enum lives in an upstream crate (`#[derive(NifUnitEnum)]` would violate the orphan rule)
- Variants are atoms on the Elixir side, not strings
- The enum participates in larger `NifStruct`/`NifMap` derives

If the enum IS yours, prefer `#[derive(NifUnitEnum)]` — much less code.

## Resource lifecycle — borrowed handles & explicit cleanup

### Caution: `mem::transmute` to `'static` for resource-stored borrows (y_ex)

Sometimes a foreign API gives you a borrowed handle (e.g., yrs `TransactionMut<'doc>` borrows the `Doc`) and you want to stash it in a `ResourceArc`. ResourceArc requires `'static`, so the only way is `mem::transmute<...<'a>, ...<'static>>`. This is `unsafe` and can be UAF if the owner is dropped first.

```rust
// y_ex transaction.rs shape — transaction borrows the doc
pub struct TxnResource(RwLock<Option<yrs::TransactionMut<'static>>>);

#[rustler::nif]
fn doc_begin_txn<'a>(doc_arc: ResourceArc<DocResource>) -> NifResult<NifTxn> {
    let txn = doc_arc.0.transact_mut();
    // SAFETY: the returned NifTxn carries a ResourceArc<DocResource> alongside,
    // so the Doc cannot be GC'd before the TxnResource. Without that companion
    // ResourceArc, this transmute would be UAF.
    let txn_static: yrs::TransactionMut<'static> = unsafe { std::mem::transmute(txn) };
    let txn_resource = ResourceArc::new(TxnResource(RwLock::new(Some(txn_static))));
    Ok(NifTxn { doc: doc_arc, txn: txn_resource })  // ← `doc` keeps owner alive
}
```

**Acceptable IFF**: the owning resource (here `DocResource`) is held alongside in the same Elixir-visible struct so the BEAM GC can't drop the owner before the borrower. **Otherwise**: this is use-after-free. Document the SAFETY contract at the type definition; an `unsafe` block without a companion ResourceArc is a block-severity finding in review.

### Explicit-close NIF companion to RAII Drop (midiex)

The skill's `Mutex<Option<T>>` template (SKILL.md §2.4) covers Drop-fires-cleanup. Some foreign libraries require an explicit `.close()` call to flush buffers or release exclusive OS resources — Drop alone isn't enough, OR you want the close to happen at a deterministic point in the Elixir caller's lifecycle (not at GC time).

The complete shape pairs the resource type with both an explicit `close` NIF AND the implicit Drop:

```rust
pub struct OutConnRef(Mutex<Option<MidiOutputConnection>>);

#[rustler::resource_impl]
impl rustler::Resource for OutConnRef {}

impl Drop for OutConnRef {
    fn drop(&mut self) {
        // Best-effort close on GC. close() flushes; ignore errors here since
        // Drop must be infallible.
        if let Ok(mut guard) = self.0.lock() {
            if let Some(conn) = guard.take() {
                let _ = conn.close();  // returns the underlying MidiOutput; we drop it
            }
        }
    }
}

// Explicit close NIF — Elixir caller decides when to flush
#[rustler::nif]
fn close_out_connection(conn: ResourceArc<OutConnRef>) -> NifResult<Atom> {
    let mut guard = conn.0.lock().expect("conn lock");
    match guard.take() {
        Some(c) => {
            c.close();  // flush + release OS resource
            Ok(atoms::ok())
        }
        None => Ok(atoms::ok()),  // already closed; idempotent
    }
}
```

The `.take()` makes both halves safe — the explicit close races with Drop, but only one will see `Some`.

## OS-specific patterns

### `thread_local!` for thread-affine OS handles (midiex)

Some OS resources (audio/MIDI/graphics device handles, certain GPU contexts, OS-level event loops) are **thread-affine** — they're cheap-but-not-free to construct AND cannot be moved between threads. For these, `thread_local!` gives each scheduler dirty-thread its own handle and avoids the cross-thread `Send + Sync` constraint that `LazyLock` / `OnceLock` would require.

```rust
use std::cell::RefCell;
use midir::MidiInput;

thread_local! {
    // Per-thread handle. Constructed lazily on first access in each scheduler thread.
    static MIDI_INPUT: RefCell<Result<MidiInput, midir::InitError>> =
        RefCell::new(MidiInput::new("midiex"));
}

#[rustler::nif(schedule = "DirtyIo")]
fn list_input_ports() -> Result<Vec<String>, String> {
    MIDI_INPUT.with(|cell| {
        let guard = cell.borrow();
        let input = guard.as_ref().map_err(|e| e.to_string())?;
        Ok(input.ports().iter().map(|p| input.port_name(p).unwrap_or_default()).collect())
    })
}
```

**When to reach for `thread_local!` over `LazyLock`/`OnceLock`:**
- The handle type is `!Send` or `!Sync` (can't go in a global)
- The OS API has thread affinity (e.g. CoreMIDI client objects, OpenGL contexts)
- Construction is non-trivial (some FFI work) but tolerable per-thread
- You want construction errors to be per-thread rather than poisoning a global

The dirty scheduler pool has a small bounded thread count, so per-thread allocation is bounded.

### macOS hot-plug / notification callbacks — spawn thread + `CFRunLoop` (midiex)

macOS notification, hot-plug, and many system event APIs (CoreMIDI, IOKit, Core Audio device-change) require an active CFRunLoop on the listening thread. The BEAM scheduler threads do NOT run a CFRunLoop, so you must spawn a dedicated `std::thread` and run one inside it. Communicate with the BEAM via OwnedEnv messages (the standard non-BEAM-thread path).

```rust
#[cfg(target_os = "macos")]
fn spawn_midi_hotplug_listener(notify_pid: LocalPid) {
    std::thread::spawn(move || {
        // Register for CoreMIDI notifications first...
        let _client = unsafe { register_midi_client_notify_callback(notify_pid) };

        // Then drive the CFRunLoop. This blocks the spawned thread forever.
        // CALLBACK fires on this thread inside the run loop iteration.
        unsafe {
            core_foundation::runloop::CFRunLoop::run_current();
        }
    });
}

// The C-callback (called by macOS on the spawned thread) sends to BEAM via OwnedEnv:
extern "C" fn midi_notify_callback(/* CoreMIDI args */) {
    let mut env = rustler::env::OwnedEnv::new();
    let _ = env.send_and_clear(&NOTIFY_PID.load(), |e| {
        (atoms::midi_hotplug(), /* payload */).encode(e)
    });
}
```

**Critical**: never call `CFRunLoop::run_current()` on a BEAM scheduler thread — it blocks forever and will collapse the scheduler. Always a dedicated `std::thread`.

This pattern generalizes to any OS API requiring a long-lived event loop (Linux `inotify`, Windows `RegNotifyChangeKeyValue`, etc.) — spawn a thread, run the loop, communicate via OwnedEnv.

## Crypto NIF patterns

### Fixed-size binary parse + `copy_from_slice` (crypto NIF pattern, ex_secp256k1)

Crypto NIFs constantly decode fixed-width inputs — 32-byte hashes, 33/65-byte public keys, 64-byte signatures. The canonical safe shape:

```rust
#[rustler::nif]
fn verify<'a>(env: Env<'a>, message_hash: Binary, signature: Binary, public_key: Binary)
    -> Term<'a>
{
    // 1. Length-validate up front; bail with a typed atom error.
    if message_hash.len() != 32 {
        return (atoms::error(), atoms::wrong_message_size()).encode(env);
    }
    if signature.len() != 64 {
        return (atoms::error(), atoms::wrong_signature_size()).encode(env);
    }
    if public_key.len() != 33 && public_key.len() != 65 {
        return (atoms::error(), atoms::wrong_public_key_size()).encode(env);
    }

    // 2. Allocate fixed-size arrays on the stack and copy_from_slice.
    //    Safer than try_into().unwrap() (which panics) and clearer than slicing.
    let mut hash: [u8; 32] = [0; 32];
    hash.copy_from_slice(message_hash.as_slice());

    let mut sig: [u8; 64] = [0; 64];
    sig.copy_from_slice(signature.as_slice());

    // 3. Hand the fixed-size arrays to the crypto API.
    match secp256k1_verify(&hash, &sig, public_key.as_slice()) {
        Ok(true)  => atoms::ok().encode(env),
        Ok(false) => (atoms::error(), atoms::invalid_signature()).encode(env),
        Err(e)    => (atoms::error(), e.to_string()).encode(env),
    }
}
```

**Why this shape:**
- `try_into::<[u8; N]>().unwrap()` panics on length mismatch — translates user input error into a BEAM crash.
- Slicing `&binary[..N]` carries the input lifetime, restricting where the array can flow (some FFI APIs want owned arrays).
- `copy_from_slice` is bounds-checked (panics if lengths differ), but the explicit `if len != N` check ahead of it makes that path unreachable.

Pre-bake the size-error atoms in a `rustler::atoms!` block so each variant is a typed atom, not a stringified message.

## Image / render NIF patterns

### User-controlled allocation size validation (image/render NIF DoS surface)

Any NIF that allocates memory whose size is derived from user input (image dimensions, decompressed buffer length, parser depth, etc.) is a DoS surface. The pattern:

```rust
const MAX_PIXELS: u64 = 100_000_000;  // 100 megapixels — calibrate per use case

#[rustler::nif(schedule = "DirtyCpu")]
fn render_svg<'a>(env: Env<'a>, svg: String, width: u32, height: u32, zoom: f32)
    -> NifResult<Term<'a>>
{
    let final_w = (width as f32 * zoom) as u64;
    let final_h = (height as f32 * zoom) as u64;
    let pixels = final_w.checked_mul(final_h)
        .ok_or_else(|| rustler::Error::Term(Box::new("dimensions overflow")))?;

    if pixels > MAX_PIXELS {
        return Ok((atoms::error(), atoms::image_too_large()).encode(env));
    }

    // Now safe to allocate — Pixmap::new returns Option<Pixmap> precisely
    // because allocation can fail; surface that as an error, never `.unwrap()`.
    let mut pixmap = tiny_skia::Pixmap::new(final_w as u32, final_h as u32)
        .ok_or_else(|| rustler::Error::Term(Box::new("pixmap alloc failed")))?;

    /* render ... */
    todo!("encode pixmap as PNG via NewBinary")
}
```

The trap to flag in review: `Pixmap::new(w, h).unwrap()`, `Vec::with_capacity(user_size)` without a budget check, `read_to_end(&mut buf)` on a user-supplied stream — any pattern where the size is attacker-controlled and unbounded.

Calibrate `MAX_PIXELS` (or analogous bound) by what your callers actually need. Surface the rejection as a typed atom (`:image_too_large`, `:input_too_large`) so callers can pattern-match.

## ML / Tensor backend patterns

### Tensor backend wrapper — NifStruct + ResourceArc + Atom-tagged device + Deref (candlex)

The canonical Nx-backend resource shape: pair a small Elixir-introspectable field (device atom, dtype string) with the opaque `ResourceArc` to the foreign tensor type, then `impl Deref` to the inner type so NIF bodies stay short.

```rust
use rustler::{Atom, NifStruct, ResourceArc};
use candle_core::{Device, Tensor};
use std::ops::Deref;

mod atoms { rustler::atoms! { cpu, cuda, metal } }

// 1. Inner resource — wraps the foreign Tensor handle
pub struct TensorRef(pub Tensor);

#[rustler::resource_impl]
impl rustler::Resource for TensorRef {}

// 2. Public NifStruct — what Elixir sees as %Candlex.Tensor{}
#[derive(NifStruct)]
#[module = "Candlex.Tensor"]
pub struct ExTensor {
    pub device: Atom,                       // small introspectable: :cpu / :cuda / :metal
    pub resource: ResourceArc<TensorRef>,   // opaque handle
}

// 3. Constructor derives the atom from the Rust enum at construction time
impl ExTensor {
    pub fn new(t: Tensor) -> Self {
        let device = match t.device() {
            Device::Cpu => atoms::cpu(),
            Device::Cuda(_) => atoms::cuda(),
            Device::Metal(_) => atoms::metal(),
        };
        Self { device, resource: ResourceArc::new(TensorRef(t)) }
    }
}

// 4. Deref keeps NIF function bodies short
impl Deref for ExTensor {
    type Target = Tensor;
    fn deref(&self) -> &Tensor { &self.resource.0 }
}

// Usage in a NIF — body is a one-liner thanks to Deref
#[rustler::nif(schedule = "DirtyCpu")]
fn tensor_add(a: ExTensor, b: ExTensor) -> Result<ExTensor, MyError> {
    Ok(ExTensor::new((&*a + &*b)?))
}
```

The atom field gives Elixir code something to pattern-match on (`%Candlex.Tensor{device: :cuda} = t`) without crossing the NIF boundary just to ask. Use this shape for ML/tensor backends, image handles with format metadata, database connection wrappers with adapter type, etc.

### Per-dtype tensor binary encoding — `to_ne_bytes` + exhaustive match (candlex)

When returning typed numeric tensor data as an Elixir binary, use **native-endian** `to_ne_bytes` (matches Nx's binary backend convention) and exhaustively match on dtype — including `BF16`/`F16` via the [`half`](https://crates.io/crates/half) crate:

```rust
use candle_core::{DType, Tensor};
use half::{bf16, f16};
use rustler::{Binary, Env, NewBinary, NifResult};

#[rustler::nif(schedule = "DirtyCpu")]
fn tensor_to_binary<'a>(env: Env<'a>, t: ExTensor) -> NifResult<Binary<'a>> {
    let bytes: Vec<u8> = match t.dtype() {
        DType::U8  => t.flatten_all()?.to_vec1::<u8>()?
            .into_iter().flat_map(u8::to_ne_bytes).collect(),
        DType::U32 => t.flatten_all()?.to_vec1::<u32>()?
            .into_iter().flat_map(u32::to_ne_bytes).collect(),
        DType::I64 => t.flatten_all()?.to_vec1::<i64>()?
            .into_iter().flat_map(i64::to_ne_bytes).collect(),
        DType::F32 => t.flatten_all()?.to_vec1::<f32>()?
            .into_iter().flat_map(f32::to_ne_bytes).collect(),
        DType::F64 => t.flatten_all()?.to_vec1::<f64>()?
            .into_iter().flat_map(f64::to_ne_bytes).collect(),
        DType::F16 => t.flatten_all()?.to_vec1::<f16>()?
            .into_iter().flat_map(f16::to_ne_bytes).collect(),
        DType::BF16 => t.flatten_all()?.to_vec1::<bf16>()?
            .into_iter().flat_map(bf16::to_ne_bytes).collect(),
    };
    let mut out = NewBinary::new(env, bytes.len());
    out.as_mut_slice().copy_from_slice(&bytes);
    Ok(out.into())
}
```

Two subtle requirements:
- **Native-endian** (`to_ne_bytes`), not big-endian or little-endian. Nx's binary backend is native-endian by convention. If you mix, your tensors will have garbled values on the Elixir side.
- **Exhaustive match** — let the compiler tell you when a new dtype is added upstream. Match-arm fallthrough hides correctness bugs.

For zero-copy where possible, prefer `t.storage_offset()` + raw slice access if the upstream library exposes it; the `flatten_all().to_vec1()` path above always copies.

## WASM / plugin patterns

### Cancel-handle resource for long-running NIFs (extism)

For NIFs that may run for seconds (WASM plugin invocations, model inference, regex backtracking on adversarial input), expose a **sibling `CancelHandle` resource** derived from the main resource. Elixir holds the handle and calls `cancel` from a *sister process* to trigger cooperative cancellation in the running NIF:

```rust
pub struct PluginRef {
    plugin: Mutex<Option<extism::Plugin>>,
    cancel_handle: extism::CancelHandle,  // cheap handle the plugin exposes
}

#[rustler::resource_impl]
impl rustler::Resource for PluginRef {}

// Sibling resource — Elixir holds this separately from the plugin
pub struct ExtismCancelHandle(extism::CancelHandle);

#[rustler::resource_impl]
impl rustler::Resource for ExtismCancelHandle {}

// Derive the cancel handle at construction
#[rustler::nif(schedule = "DirtyIo")]
fn make_plugin(wasm: Binary) -> NifResult<(ResourceArc<PluginRef>, ResourceArc<ExtismCancelHandle>)> {
    let plugin = extism::Plugin::new(wasm.as_slice(), [], false)
        .map_err(|e| Error::Term(Box::new(e.to_string())))?;
    let cancel = plugin.cancel_handle();
    let plugin_ref = ResourceArc::new(PluginRef {
        plugin: Mutex::new(Some(plugin)),
        cancel_handle: cancel.clone(),
    });
    let handle_ref = ResourceArc::new(ExtismCancelHandle(cancel));
    Ok((plugin_ref, handle_ref))
}

// The cancel NIF is called from a DIFFERENT Elixir process than the one
// running plugin_call. Cooperative cancellation: the upstream library
// checks a flag periodically inside its work loop.
#[rustler::nif]
fn cancel(handle: ResourceArc<ExtismCancelHandle>) -> Atom {
    handle.0.cancel();
    atoms::ok()
}
```

```elixir
# Elixir-side usage: spawn the NIF call in a Task, hold the handle, cancel from caller
defmodule MyPlugin do
  def call_with_timeout(plugin, handle, input, timeout) do
    task = Task.async(fn -> ExtismNif.plugin_call(plugin, input) end)
    case Task.yield(task, timeout) do
      {:ok, result} -> result
      nil -> ExtismNif.cancel(handle); Task.shutdown(task); {:error, :timeout}
    end
  end
end
```

This is the only safe way to interrupt a running DirtyCpu/DirtyIo NIF — the BEAM cannot kill a NIF call from outside, only the NIF itself can check a flag and return early. The upstream library must support cooperative cancellation; if it doesn't, you can't add this from the outside.

## Static handle caching

### Static lazy device-handle cache — `static Mutex<Option<T>>` (candlex)

The `Mutex<Option<T>>` pattern (SKILL.md §2.4, for explicit-close) also fits expensive **global** handles where you want first-touch initialization AND the option to re-initialize later (device reset, runtime swap). Different from `OnceLock`/`LazyLock`, which initialize once and never replace:

```rust
use std::sync::Mutex;
use candle_core::Device;

// Global handle, lazy-init on first use, re-initializable on device reset
static CUDA_DEVICE: Mutex<Option<Device>> = Mutex::new(None);

fn cuda_device() -> Result<Device, candle_core::Error> {
    let mut guard = CUDA_DEVICE.lock().expect("CUDA_DEVICE lock");
    if guard.is_none() {
        *guard = Some(Device::new_cuda(0)?);
    }
    Ok(guard.as_ref().unwrap().clone())  // Device is Arc-internal, cheap clone
}

// Re-init NIF for device reset (rare but useful)
#[rustler::nif(schedule = "DirtyIo")]
fn reset_cuda() -> Atom {
    let mut guard = CUDA_DEVICE.lock().expect("CUDA_DEVICE lock");
    *guard = None;
    atoms::ok()
}
```

When to pick `Mutex<Option<T>>` over `LazyLock`:
- The handle can be **invalidated** (device reset, connection re-established, runtime swap)
- Initialization must **return Result** (`LazyLock`'s init closure is infallible; you'd have to panic)
- You want the explicit `reset` NIF for ops/debugging

When `LazyLock`/`OnceLock` is fine: handle is set once at startup, never invalidated, init can panic on failure.

## Build & setup patterns

### `load` callback for one-time global setup (rhai_rustler)

The §2.5 setup snippet mentions `rustler::init!("...", load = on_load)` briefly. Real use cases:

- **Fix a global before any NIF runs.** rhai_rustler calls `rhai::config::hashing::set_ahash_seed(...)` in `on_load` to pin the hash seed across BEAM restarts (rhai's compiled `Module` files are content-addressed by hash, so per-process random seeds would invalidate caches).
- **Pre-load resource types** (older Rustler, pre-0.36): `rustler::resource!(MyResource, env)` had to be called inside `load`.
- **Validate native dependency versions** — abort load if a runtime-loaded library is the wrong version.
- **Initialize a logger** that bridges Rust `log`/`tracing` events to Elixir Logger.

```rust
fn on_load(env: Env, _info: Term) -> bool {
    // Pin the hash seed BEFORE any NIF can be called.
    // Returning false aborts module loading — Elixir sees a clean load error
    // rather than a panic mid-call later.
    if let Err(e) = init_global_state() {
        eprintln!("rust-nif load failed: {e}");
        return false;
    }
    true
}

rustler::init!("Elixir.MyApp.Native", load = on_load);
```

Use `load` for setup that **must happen exactly once and before any NIF**. Don't put per-call work here (it runs on the BEAM thread loading the .so). Returning `false` is the cleanest failure path — much better than panicking on the first NIF call.

### Embedded engine with sibling plugin crate (rhai_rustler)

When your NIF embeds a scripting/plugin engine that loads `cdylib`s at runtime via `libloading` (rhai's `EvalAltResult::ErrorModuleNotFound` path, Lua hosts loading `.so` modules, etc.), keep the test plugins as **sibling Cargo crates under `native/`**, not as inline Rust test code:

```
native/
├── my_nif/
│   ├── Cargo.toml          # crate-type = ["cdylib"], the main NIF
│   └── src/lib.rs
└── test_dylib_module/
    ├── Cargo.toml          # crate-type = ["cdylib"], a test plugin
    └── src/lib.rs          # exports symbols the main NIF will load at runtime
```

```toml
# native/test_dylib_module/Cargo.toml
[package]
name = "test_dylib_module"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
# Same engine API as the main NIF, so the plugin compiles against it
rhai = { version = "1.x", features = ["sync"] }
```

Build both in CI; reference the test plugin's compiled artifact path (`target/debug/libtest_dylib_module.so`) from ExUnit fixtures. This keeps the test plugin honest (it must be a real loadable cdylib, not a mock) and gives you a working example of the plugin API for users.

Both `cdylib`s build with one `cargo build` from a workspace root. Add a top-level `Cargo.toml` with `[workspace] members = ["native/my_nif", "native/test_dylib_module"]`.

## Error-shape macros

### `try_or_return_elixir_err!` macro for stringified errors (resvg)

When NIFs return `NifResult<Term<'a>>` and you want stringified errors without typing out the same `match` block five times, factor the boilerplate into a small macro:

```rust
macro_rules! try_or_return_elixir_err {
    ($expression:expr, $env:ident) => {
        match $expression.map_err(|e| e.to_string()) {
            Ok(v) => v,
            Err(err) => return Ok((atoms::error(), err).encode($env)),
        }
    };
}

// Usage — every fallible step becomes a one-liner.
#[rustler::nif(schedule = "DirtyCpu")]
fn svg_to_png<'a>(env: Env<'a>, path: String, output: String) -> NifResult<Term<'a>> {
    let svg_data = try_or_return_elixir_err!(std::fs::read(&path), env);
    let tree = try_or_return_elixir_err!(usvg::Tree::from_data(&svg_data, &Default::default()), env);
    let pixmap = render_tree(&tree);
    try_or_return_elixir_err!(pixmap.encode_png(), env);
    try_or_return_elixir_err!(std::fs::write(&output, png_bytes), env);
    Ok(atoms::ok().encode(env))
}
```

Pairs with the html5ever_elixir bare-string Encoder shape (Error shape #3 in SKILL.md §2.4). Use when:
- The variant set isn't large enough to justify `thiserror + typed Encoder`
- Callers don't need to pattern-match on error variants — a string is sufficient
- You'd otherwise repeat the same 4-line `match` block in every NIF

Avoid it (use `thiserror + Encoder` instead) when callers need typed errors to dispatch on.

## Tokio runtime variants

### Tokio runtime variants — `LazyLock` and `RUNTIME.spawn` (wasmex)

The foundational Tokio Runtime-Enter pattern is in SKILL.md §2.4. Two production variants:

**Variant 1: `LazyLock<Runtime>` (Rust 1.80+)** — drops the wrapper `runtime()` fn:

```rust
use std::sync::LazyLock;
static RUNTIME: LazyLock<Runtime> = LazyLock::new(|| {
    Builder::new_multi_thread().worker_threads(2).enable_all().build().expect("tokio init")
});

#[rustler::nif(schedule = "DirtyIo")]
fn start_node(...) -> Result<...> {
    let _guard = RUNTIME.enter();   // direct access, no get_or_init
    ...
}
```

**Variant 2: `RUNTIME.spawn(...)` instead of `enter()` (wasmex pattern)** — when the entire NIF body is async work that replies via OwnedEnv, `spawn` itself establishes the tokio context inside the spawned task. No `.enter()` needed. Use this for fire-and-forget async NIFs that bypass GenServer's reply path; use the `enter()` shape when the NIF body has sync calls that internally need a tokio context (like `quinn::Endpoint::new`).

```rust
#[rustler::nif(schedule = "DirtyIo")]
fn run_async(env: Env, from: Term, work: ExWork) -> NifResult<Atom> {
    // GenServer reply pattern: decode `from = {pid, ref}` and reply via OwnedEnv
    let (pid, ref_term): (LocalPid, Term) = from.decode()?;
    let mut owned_ref = OwnedEnv::new();
    let saved_ref = owned_ref.save(ref_term);

    RUNTIME.spawn(async move {
        let result = work.do_async().await;
        let mut owned = OwnedEnv::new();
        let _ = owned.send_and_clear(&pid, |env| {
            let r = saved_ref.load(env);
            (r, result).encode(env)  // {ref, result} — matches GenServer.reply expectation
        });
    });
    Ok(atoms::ok())
}
```

The Elixir caller does a `GenServer.call`, the GenServer's `handle_call` invokes the NIF and then `{:noreply, state}` (returning `:no_reply`); the spawned task completes asynchronously and sends `{ref, result}` back via OwnedEnv, which `GenServer.call` is blocked waiting for. This bypasses GenServer's normal reply path entirely.
