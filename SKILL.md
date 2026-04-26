---
name: rust-nif
description: >
  Rust NIFs with Rustler for Elixir — safe native code, BEAM/OTP integration,
  error handling, scheduler management, and process communication. Organized by
  use-phase: planning (decide what to build), implementing (write the code),
  reviewing (audit / debug existing NIFs).
  ALWAYS use when writing Rust NIFs or Rustler code.
  ALWAYS use when integrating native Rust code with Elixir/BEAM.
  ALWAYS use when designing the API or scheduler/state strategy of a new NIF.
  ALWAYS use when reviewing a NIF PR or debugging a BEAM crash caused by a NIF.
---

# Rust NIFs with Rustler

This skill covers the Elixir-side Rust integration (Rustler, ResourceArc, scheduler, panic safety, type encoding). Load `rust-implementing` alongside for the Rust language itself.

## About this skill — three phases of NIF work

This skill is organized around the three moments at which different guidance fires:

| Phase | When | Primary content type | Section |
|---|---|---|---|
| **Planning** | Before writing — choosing whether/how to build the NIF | Decision tables, use-case rules, architecture choices | [Part 1](#part-1--planning-phase) |
| **Implementing** | At the keyboard — writing NIF functions | Numbered rules, "which construct?" tables, templates, BAD/GOOD pairs | [Part 2](#part-2--implementing-phase) |
| **Reviewing** | After writing — auditing PRs, debugging BEAM crashes caused by NIFs | Severity-classified checklist, anti-pattern catalog, debug playbook | [Part 3](#part-3--reviewing-phase) |

The most critical items (the ones that crash the BEAM or silently produce wrong output) appear in **all three** phases, framed differently:

| Critical item | Planning frame | Implementing frame | Reviewing frame |
|---|---|---|---|
| `{:ok, :ok}` double-wrap | Decide return shape during API design — pick `Atom` or meaningful `T` for `Result<T, E>` | Return type matrix + BAD/GOOD pair (§2.1 rule 2, §2.3) | Scan diff for `Result<Atom, _>` returning `Ok(atoms::ok())` (§3.1) |
| `Vec<u8>` returned as binary | Plan binary boundary: `NewBinary` out, `Binary<'a>` in | Rule 18 BAD/GOOD + Binary section (§2.1, §2.3) | Flag any NIF returning `Vec<u8>` from a binary-shaped contract (§3.1) |
| Missing `#[rustler::resource_impl]` | API-first design for resources includes the macro | Rule 20 BAD/GOOD (§2.1) | "Bare `impl Resource for T {}`" → block-severity (§3.1) |
| Blocking `.lock().unwrap()` | Choose state strategy: clone-immutable / try_lock / RwLock | Rule 4 + try_lock template (§2.10) | Flag `.lock().unwrap()` inside a NIF (§3.1) |
| Missing `runtime.enter()` for tokio resources | Plan tokio runtime ownership upfront (`OnceLock<Runtime>`) | Rule 19 + Tokio runtime-enter template (§2.2 production patterns) | Flag tokio constructor without enter (§3.1) |

This intentional repetition means the right form of guidance is reachable at the moment Claude needs it.

## How to use this skill

1. **Designing a new NIF crate or major feature?** — Read [Part 1](#part-1--planning-phase): use cases (§1.1), single-vs-split (§1.2), API-first (§1.3), planning decision table (§1.4), planning rules (§1.5).
2. **Adding a NIF to an existing crate?** — Skim [§1.4 planning decision table](#14-planning-decision-table), then go to [Part 2](#part-2--implementing-phase): rules (§2.1), "which construct?" table (§2.2), templates and patterns.
3. **Writing or extending NIF code right now?** — [Part 2](#part-2--implementing-phase) is the implementation reference. Always-loaded core: §2.1 rules + §2.2 decision table + §2.3 return type matrix + §2.4 production patterns. Setup, type encoding, scheduler, error handling, OTP, state management follow in §2.5–§2.13.
4. **Reviewing a NIF PR or diff?** — Start with [§3.1 review checklist](#31-review-checklist--flag-if-you-see-this), then [§3.2 anti-pattern catalog](#32-anti-pattern-catalog).
5. **Debugging a BEAM crash that may be NIF-caused?** — [§3.3 "When the NIF kills the VM" playbook](#33-when-the-nif-kills-the-vm--debug-playbook).

For deep references, see [reference.md](reference.md), [examples.md](examples.md), [patterns.md](patterns.md), and [advanced.md](advanced.md).

---

# PART 1 — PLANNING PHASE

The decisions made before writing the first NIF function. Wrong choices here surface as architectural pain weeks later — re-doing scheduler choice, state strategy, or single-vs-split is far more expensive than re-doing a function body.

## 1.1 Should this be a NIF at all?

### Good Use Cases

| Use Case | Why NIF? | Examples |
|----------|----------|----------|
| CPU-intensive computation | BEAM not optimized for number crunching | Image processing, crypto, compression |
| Large data structures | Rust's memory layout more efficient | Discord's SortedSet (11M users) |
| Specific Rust libraries | No Elixir equivalent | Polars, tokenizers, parsers |
| Performance-critical paths | 10x-100x speedup possible | JSON parsing, binary protocols |

### Performance Thresholds

```
Operation Time       | Recommendation
---------------------|----------------------------------
< 1 microsecond      | Probably not worth NIF overhead
1-100 microseconds   | Maybe, benchmark first
100 μs - 1 ms        | Good candidate for regular NIF
1-10 ms              | Use DirtyCpu scheduler
10-100 ms            | Use dirty scheduler + chunking
100+ ms              | Consider Port or yielding NIF
```

### When NOT to Use NIFs

1. **I/O-bound work** — Elixir's async model is better
2. **Simple transforms** — overhead of NIF call > savings
3. **Fault tolerance requirements** — NIF crash = entire VM crash (use Ports for isolation)
4. **Prototyping** — Rust has slower iteration than Elixir

| Alternative | Use When | Trade-off |
|-------------|----------|-----------|
| Port | Need isolation from crashes | Slower (IPC overhead) |
| :os.cmd | Simple scripts | Very slow, security risk |
| Task.async | Parallel Elixir | Limited by BEAM |
| Flow/Broadway | Data pipelines | Good for I/O, not CPU |

## 1.2 Single crate, conditional, or separate app?

When the NIF is needed on a subset of deployables (central API only; edge worker only; Nerves build only), you have three shapes:

| Shape | When to use | Cost |
|---|---|---|
| **NIF in its own umbrella app** (e.g. `apps/myapp_nif/`) | NIF has distinct lifecycle, tests, deps; runs on only some nodes; may be replaced by a pure-Elixir implementation via behaviour in other deployables | One extra app boundary; adds to mix deps |
| **Conditional module via behaviour + `put_env`** | Same code deployed everywhere, but runtime config picks NIF backend vs pure-Elixir backend per node | No structural split; one dead-code compile per node |
| **Feature-gated in a single app with `if Code.ensure_loaded?/1`** | Quick-and-dirty: some nodes don't even compile the NIF (e.g. Nerves ARM vs x86_64 dev) | Fragile — compile-time vs runtime confusion; hard to test the "NIF missing" path |

**Rule of thumb:**

- Always-present NIF, same API shape everywhere → no split. Single crate + single wrapper module.
- NIF is one of several implementations of the same behaviour (Signal.Backend: PureElixir / NifBackend) → behaviour + runtime-chosen backend (see implementing §2.1 rule 17 nested-Native pattern). No structural split.
- NIF compilation depends on target (e.g. Nerves firmware-only) or the app runs on nodes that can't compile Rust → dedicated umbrella app for the NIF, other apps depend on the **behaviour** not the NIF app. Easy to exclude the NIF app from non-target releases via `:only`.
- Experimental / potentially-crashy NIF → its own app AND run it on a separate BEAM via Port/erpc so crashes don't take down the main app.

Design this decision at planning time (see `elixir-planning` §5 and the "separate deployables + wire lib" row in §3.1). Don't default to "it's a NIF, so it gets its own crate" — the behaviour boundary is usually more important than the deployable-unit boundary.

## 1.3 API-First NIF Design

Design your NIF interface first by writing how you WANT to use it.

### Step 1: Design the Elixir Interface

```elixir
defmodule MyApp.ImageProcessor do
  def process(image_data, opts \\ []) do
    Native.process_image(image_data, opts)
  end
end
```

### Step 2: Write the NIF Signature to Match

```rust
#[rustler::nif(schedule = "DirtyCpu")]
fn process_image(data: Binary, opts: ProcessOptions) -> Result<Vec<u8>, NifError> {
    // Implementation follows the designed interface
}
```

### Step 3: Write Tests That Use the Interface

```rust
#[test]
fn process_image_returns_expected_result() {
    let input = load_test_image();
    let opts = ProcessOptions::default();
    let result = process_image_internal(&input, &opts).unwrap();
    assert!(result.len() > 0);
}
```

### Step 4: Decide module placement (sibling top-level vs nested vs no separate Native)

There are three valid shapes. Pick one at planning time so the module hierarchy doesn't churn later.

**Shape A — Sibling top-level `MyApp.Native` in its own file.** Used by Explorer (`Polars.Backend.Native`), Tokenizers, and mdex. The stub is its own top-level module that the public wrapper calls into:

```elixir
# lib/my_app/native.ex
defmodule MyApp.Native do
  @moduledoc false
  use Rustler, otp_app: :my_app, crate: "my_nif"

  def compute(_data), do: :erlang.nif_error(:nif_not_loaded)
  def encode(_map), do: :erlang.nif_error(:nif_not_loaded)
end

# lib/my_app.ex
defmodule MyApp do
  @spec compute(binary()) :: {:ok, map()} | {:error, atom()}
  def compute(data), do: MyApp.Native.compute(data)

  @spec encode(map()) :: {:ok, binary()} | {:error, atom()}
  def encode(map), do: MyApp.Native.encode(map)
end
```

**Shape B — Nested `defmodule Native` inside the wrapper.** Keeps the stub out of the user-facing module hierarchy entirely. Useful when you have multiple NIF-backed implementations of one behaviour and want each backend to own its own stub:

```elixir
defmodule MyApp.Signal.NifBackend do
  @moduledoc "NIF-backed signal processing. `backend/0` picks this at boot."
  @behaviour MyApp.Signal.Backend
  alias __MODULE__.Native

  @impl true
  @spec compute(binary()) :: {:ok, map()} | {:error, atom()}
  def compute(data), do: Native.compute(data)

  defmodule Native do
    @moduledoc false
    use Rustler, otp_app: :my_app, crate: "my_signal"
    def compute(_data), do: :erlang.nif_error(:nif_not_loaded)
  end
end
```

**Shape C — No separate Native module; the public module IS the stub host.** Suitable for tiny, single-purpose libraries (nimble_lz4 takes this shape). Specs go directly on the stub functions since there's no wrapper layer:

```elixir
defmodule NimbleLZ4 do
  use RustlerPrecompiled, otp_app: :nimble_lz4, crate: "nimble_lz4_nif"

  @spec compress(binary()) :: {:ok, binary()} | {:error, atom()}
  def compress(_data), do: :erlang.nif_error(:nif_not_loaded)
end
```

**Either Shape A (sibling top-level) or Shape B (nested) is the prevailing default in production Rustler libs** — Explorer/Tokenizers/mdex use sibling; html5ever_elixir and wasmex use nested. Pick by file-organization preference: sibling makes the stub easy to discover via the file tree; nested keeps the stub out of the user-facing module hierarchy. Shape C (stub-as-public) is appropriate for tiny, single-purpose libs (nimble_lz4) where the wrapper layer would be empty.

## 1.4 Planning decision table

The single most useful artifact for planning a NIF — every common decision in one place. Each row is a choice you'll face once per NIF; getting them right at planning time avoids rewrites.

| Decision | Choose this | When | Notes / cross-ref |
|---|---|---|---|
| Scheduler | `normal` (default) | Pure CPU < ~100µs, deterministic, no locks | The 1ms rule (§2.7) |
| Scheduler | `DirtyCpu` | CPU work > ~100µs typical | Hashing, parsing, encoding, training |
| Scheduler | `DirtyIo` | Blocking IO, channel waits, tokio-resource construction | Plus `runtime.enter()` if tokio-based |
| State strategy | Clone-based immutability | Default, when data clones cheaply | Explorer pattern — no mutex, no deadlocks |
| State strategy | `std::sync::Mutex` + `try_lock` | Mutable shared state | Discord SortedSet pattern |
| State strategy | `RwLock` | Read-heavy workloads (multiple concurrent readers) | Reads don't block reads |
| State strategy | `parking_lot::Mutex` | Existing dep; want unpoisoned locks; benchmarks show contention | mdex uses it in production; default of Explorer/Tokenizers/SortedSet remains `std::sync` |
| Error type | atom errors (`Result<T, Atom>`) | Small set of error reasons | Simple, no extra deps |
| Error type | `(error_atom, reason_string)` tuple + Elixir `defexception` | Small/single-purpose libs (mdex, nimble_lz4 pattern) | No Rust-side custom error type; Elixir layer wraps as exceptions |
| Error type | `thiserror` enum + `impl Encoder` | Library-scale: many distinct variants worth pattern-matching on | Explorer/Tokenizers pattern; heavier than needed for small libs |
| Error type | `rustler::Error::Term` | Standard Rustler errors | Default if no custom needs |
| Return type | Bare `T` | Infallible getter | Avoids `:ok` overhead |
| Return type | `Result<T, E>` (with T ≠ Atom) | Operation can fail | Avoid `{:ok, :ok}` |
| Return type | `Atom` | Void-like operation that can't fail | Returns `:ok` directly |
| Return type | `Option<T>` | Nullable getter | Returns `nil` or value |
| Binary type IN | `Binary<'a>` parameter | Reading bytes from Elixir | Zero-copy view |
| Binary type OUT | `NewBinary` | Returning bytes to Elixir | NEVER `Vec<u8>` (becomes list of ints) |
| Network IO | Elixir | Default for all network code | Better fault tolerance |
| Network IO | Rust NIF | Need Rust-only library, heavy CPU on net data, custom binary protocol | |
| Single app vs split crate | Single crate | NIF used everywhere with same API | See §1.2 |
| Single app vs split crate | Behaviour + runtime backend choice | NIF is one of several implementations | `:put_env` or `:persistent_term` |
| Single app vs split crate | Dedicated umbrella app | NIF on subset of deployables; compile-target-specific | Other apps depend on the behaviour |
| Single app vs split crate | Separate BEAM via Port | Experimental / crashy NIF | Crash isolation |
| Lock acquisition | `try_lock` + `:lock_fail` return | Normal-scheduler NIFs; shared cross-call state with realistic contention | NEVER blocking lock on the normal scheduler |
| Lock acquisition | Blocking `.lock().unwrap()` | DirtyCpu, per-call scratch state held inside a freshly-constructed struct (mdex pattern) | The same NIF call owns the only handle — no contender to deadlock against |
| Lock acquisition | `try_lock().unwrap()` (fail-loud) | Per-resource serialization is a convention; contention = bug (rhai_rustler pattern) | Panics surface the bug; blocking would silently serialize unrelated callers |
| Locking around expensive work | Drop guard before expensive work | Always | Hold for minimum duration (§2.10) |
| Test split | NIF wrapper thin; pure logic in plain Rust fns | Always | `cargo test` for internals, ExUnit at boundary |
| `@spec` | On public-facing wrapper module | Always | NIF stub may omit; Explorer/Tokenizers pattern |
| Native module placement | Sibling top-level `MyApp.Native` in `lib/my_app/native.ex` | One of two prevailing defaults — Explorer, Tokenizers, mdex | §1.3 Step 4 Shape A |
| Native module placement | Nested `defmodule Native` inside the wrapper | Equally prevailing default — html5ever_elixir, wasmex; useful for behaviour-backed libs | §1.3 Step 4 Shape B |
| Native module placement | No separate Native — public module IS the stub host | Small focused libs (nimble_lz4); single-purpose binding | §1.3 Step 4 Shape C |
| Tokio runtime | `OnceLock<Runtime>` at module scope | Any tokio-dependent crate (quinn, mDNS, libp2p) | `runtime.enter()` in every DirtyIo NIF (§2.4 Tokio template) |
| Panic-prone upstream library | Wrap every NIF in `nif_safe` (catch_unwind) | Library may panic on malformed input | §2.4 nif_safe template |
| `@spec` strategy | Specs on wrapper, omitted on `Native` stub | Following Explorer/Tokenizers convention | Shared `@type` aliases for repeated complex args |

## 1.5 Planning rules (LLM)

These rules fire at planning time. They're a subset of the implementation ruleset (§2.1) reframed as design-phase constraints. See §2.1 for the complete rule text and code examples.

1. **ALWAYS design the Elixir API first** — write how you want to call it from Elixir, then implement the Rust NIF to match. (Implements §2.1 rule 7.)
2. **PREFER Elixir for network I/O** — only use Rust NIFs when you need a Rust-only library, heavy CPU processing of network data, or custom binary protocol parsing. (§2.1 rule 11.)
3. **ALWAYS benchmark before assuming a NIF is faster** — the overhead of crossing the NIF boundary can negate small gains. Plan a benchmark step into the design. (§2.1 rule 12.)
4. **PREFER clone-based immutability over mutex** for ResourceArc state. Explorer's pattern: `clone_inner()` then return new resource. No mutex, no deadlocks. When mutex IS used, scope `try_lock()` to normal-scheduler NIFs and to genuinely shared cross-call state with realistic contention; per-call scratch state on DirtyCpu (mdex pattern) can use blocking `.lock()` since the call owns the only handle. (§2.1 rule 4.)
5. **ALWAYS split NIF wrappers from pure logic** at design time — NIF functions stay thin, internal logic gets `cargo test`, NIF integration gets ExUnit. Plan the module boundary upfront. (§2.1 rule 8.)
6. **PREFER `std::sync::Mutex`** as the default; use `parking_lot` when you already have it as a dep, want unpoisoned locks, or have benchmarks showing contention. Explorer, Tokenizers, and Discord SortedSet stick to `std::sync`; mdex uses `parking_lot` in production for `LumisAdapter` coordination — both shapes ship. (§2.1 rule 10.)
7. **PREFER normalizing input values (case, encoding, trimming) at the Rust layer** — design the API so Elixir callers don't pre-process. The Rust side knows what the underlying library expects. (§2.1 rule 16.)
8. **ALWAYS pick the right scheduler upfront** — `DirtyCpu` for CPU > 100µs, `DirtyIo` for any blocking, normal only for fast pure-CPU. Wrong choice = scheduler collapse later. (§2.1 rule 1.)
9. **PLAN panic-safety only where you need a custom error variant or panic-time cleanup.** Rustler catches panics at the NIF boundary by default — for many wrappers (mdex around comrak/lol_html/ammonia) the implicit catch is enough. Reach for explicit `nif_safe`/`catch_unwind` when you want a typed `MyError::Panic(msg)` variant, need to release a resource on panic, or are calling C/C++ from inside the NIF body. (§2.1 rule 3 + §2.4 catch_unwind / nif_safe templates.)
10. **ALWAYS plan tokio runtime ownership** for NIFs constructing tokio-dependent resources (quinn, mDNS, libp2p) — `OnceLock<Runtime>` + `runtime.enter()` is the pattern; designing this in avoids "no reactor running" panics later. (§2.1 rule 19 + §2.4 Tokio template.)
11. **PICK the native module shape at planning time.** Three valid shapes (§1.3 Step 4): (A) sibling top-level `MyApp.Native` in its own file (Explorer, mdex, Tokenizers); (B) nested `defmodule Native` inside the wrapper (html5ever_elixir, wasmex); (C) no separate Native — the public module is the stub host — fits small focused libs (nimble_lz4). A and B are roughly equally common in canonical libs; pick by file-organization preference. Use C only when the wrapper layer would be empty.
12. **ALWAYS add `@spec` where the type contract is exposed to callers.** With separate stub + wrapper (Shapes A and B): specs on the wrapper, stubs may omit. With stub-as-public (Shape C): specs go directly on the stub functions since there's no wrapper. The boundary between Rust and Elixir type systems is the one place specs are non-negotiable. (§2.1 rule 17.)

---

# PART 2 — IMPLEMENTING PHASE

What to type when writing NIF functions. This is the bulk of the skill — rules, decision tables, templates, and code examples for the moment of writing.

## 2.1 Rules for Writing Rust NIFs (LLM)

1. **ALWAYS use dirty schedulers** for operations >1ms — normal scheduler NIFs must complete in <1ms or they block the BEAM
   ```rust
   #[rustler::nif(schedule = "DirtyCpu")]  // CPU work
   #[rustler::nif(schedule = "DirtyIo")]   // I/O work
   ```

2. **NEVER return generic `Result<Atom, E>` where `Ok` wraps `:ok`** — for generic `Result` (with a user-defined error type that has its own `Encoder`), this produces `{:ok, :ok}` on the Elixir side. **Exception:** `NifResult<T>` (= `Result<T, rustler::Error>`) is special-cased by Rustler — `Ok(value)` returns `value` directly and `Err(e)` raises. Returning `NifResult<Atom>` with `Ok(atoms::ok())` is fine — y_ex and html5ever_elixir do this in production. See §2.2 Return Type Matrix for the full distinction.
   ```rust
   // BAD: generic Result<Atom, _> with Ok(:ok) → {:ok, :ok} in Elixir
   fn insert(set: ResourceArc<Set>, val: i64) -> Result<Atom, Atom> {
       // ...
       Ok(atoms::ok())  // Elixir sees {:ok, :ok}
   }

   // GOOD: NifResult<Atom> — Rustler unwraps Ok, raises on Err
   fn insert(set: ResourceArc<Set>, val: i64) -> NifResult<Atom> {
       // ...
       Ok(atoms::ok())  // Elixir sees :ok
   }

   // GOOD: generic Result with a meaningful value in Ok
   fn insert(set: ResourceArc<Set>, val: i64) -> Result<bool, Atom> {
       // ...
       Ok(true)  // Elixir sees {:ok, true}
   }

   // GOOD: bare Atom for void operations that can't fail
   fn insert(set: ResourceArc<Set>, val: i64) -> Atom {
       // ...
       atoms::ok()  // Elixir sees :ok
   }
   ```

3. **ALWAYS return Result types** instead of panicking — a panic in a NIF crashes the entire BEAM VM
   ```rust
   fn example() -> Result<T, Atom>  // Not T directly if it can fail
   ```

4. **PREFER clone-based immutability over mutex** for ResourceArc state. When mutex IS used, the lock-acquisition rule depends on scheduler, contention shape, and what contention *means* in your design:
   - **Normal-scheduler NIFs:** ALWAYS `try_lock()` and return `:lock_fail` — blocking locks can deadlock the scheduler.
   - **Cross-call shared state on dirty schedulers with realistic contention:** prefer `try_lock()` so callers can back off cleanly.
   - **Per-call scratch state held inside a freshly-constructed struct on DirtyCpu** (mdex's `PlaceholderRenderer` / `LumisAdapter` pattern): blocking `.lock().unwrap()` is normal — the only contender for the lock is the same NIF call.
   - **"Fail-loud" — per-resource serialization is a convention, contention indicates a bug** (rhai_rustler pattern): use `try_lock().unwrap()` to **panic on contention** rather than block or return an error. This surfaces the bug instead of silently serializing unrelated callers. Appropriate when the Elixir-side discipline is "one process holds this resource at a time" and any contention is a programming error.
   ```rust
   // BEST: Clone-based immutability (Explorer pattern — no mutex, no deadlocks)
   pub struct MyRef(pub MyData);
   impl MyRef { pub fn clone_inner(&self) -> MyData { self.0.clone() } }
   fn transform(res: ExMyData) -> Result<ExMyData, MyError> {
       let mut data = res.clone_inner();  // Clone, mutate, return new resource
       data.transform()?;
       Ok(ExMyData::new(data))
   }

   // GOOD: try_lock with error return (Discord SortedSet pattern — shared resource, contention possible)
   match resource.data.try_lock() {
       Some(guard) => Ok(process(&guard)),
       None => Err(atoms::lock_fail()),
   }

   // GOOD on DirtyCpu, scratch state with no real contender (mdex pattern):
   // a single NIF call owns the only handle; .lock() can never deadlock here
   let mut buf = scratch.lock().unwrap();
   buf.push(item);

   // BAD on the normal scheduler — blocks the BEAM scheduler
   let guard = resource.data.lock().unwrap();
   ```

5. **ALWAYS use derive macros** for Elixir ↔ Rust type conversion
   ```rust
   #[derive(NifStruct)]       // %Module{}
   #[derive(NifMap)]          // %{key: value}  (requires ALL keys present!)
   #[derive(NifTaggedEnum)]   // :atom | {:tag, value}
   #[derive(NifUnitEnum)]     // :atom
   ```

6. **NEVER use `unwrap()` or `expect()`** in NIF function bodies — convert to Result with `.ok_or()` or `?` operator. For infallible conversions where `None`/`Err` is logically impossible, use `.expect("known valid: reason")` to document the invariant. On locks, always use `try_lock()` with error return, never `.lock().unwrap()`

7. **ALWAYS design the Elixir API first** — write how you want to call it from Elixir, then implement the Rust NIF to match

8. **ALWAYS split NIF wrappers from pure logic** — NIF functions should be thin wrappers; test the internal logic with `cargo test`, test the NIF integration with ExUnit

9. **PREFER Rayon** for data parallelism over manual thread management
   ```rust
   use rayon::prelude::*;
   data.par_iter().map(|x| process(x)).collect()
   ```

10. **PREFER `std::sync::Mutex`** (or avoid mutex entirely via clone-based immutability). Explorer, Tokenizers, and Discord SortedSet all use `std::sync` rather than `parking_lot` — that's the default to reach for. `parking_lot::Mutex` / `RwLock` is acceptable in production (mdex pulls it in for `LumisAdapter`'s coordination state) when you already have it as a transitive dep, want unpoisoned locks, or have benchmarks showing contention is a bottleneck. Don't add `parking_lot` to a fresh NIF crate just for fun

11. **PREFER Elixir for network I/O** — only use Rust NIFs when you need a Rust-only library, heavy CPU processing of network data, or custom binary protocol parsing

12. **ALWAYS benchmark before assuming** a NIF is faster — the overhead of crossing the NIF boundary can negate small gains

13. **ALWAYS drop MutexGuards before expensive work** — hold locks for the minimum duration
    ```rust
    let value = {
        let guard = res.data.try_lock().ok_or(atoms::lock_fail())?;
        guard.clone()
    }; // Guard dropped here
    expensive_work(value); // Lock free
    ```

14. **NEVER call `send_and_clear()` from a BEAM-managed thread** — it will panic. Only use from `std::thread::spawn` or similar

15. **ALWAYS wrap NifMap NIFs** in Elixir functions that fill missing keys with `nil` — NifMap requires ALL keys present even for `Option<T>` fields

16. **PREFER normalizing input values (case, encoding, trimming) at the Rust layer** rather than requiring Elixir callers to pre-process — the Rust side knows what the underlying library expects

17. **ALWAYS add `@spec` where the type contract is exposed to callers.**
    - **If you have a separate stub module + wrapper (Shapes A and B in §1.3):** put `@spec` on every public-facing wrapper function (e.g., `MyApp.DataFrame`). The internal NIF stub module may omit specs — Explorer, Tokenizers, and mdex all skip specs on stubs and put them only on the public API.
    - **If the stub IS the public module (Shape C — small libs like nimble_lz4 with no separate wrapper):** put `@spec` directly on the stub functions, since there's no wrapper to spec.
    - Define shared `@type` aliases when multiple functions take the same complex argument pattern.

    Module placement (sibling top-level vs nested vs stub-as-public) is decided at planning time — see [§1.3 Step 4](#step-4-decide-module-placement-sibling-top-level-vs-nested-vs-no-separate-native) for the three shapes and which to pick. Sibling top-level `MyApp.Native` is the prevailing pattern (Explorer, mdex); nested `defmodule Native` is fine for behaviour-backed libraries; stub-as-public is fine for small focused libs.

18. **NEVER return `Vec<u8>` to represent binary data** — Rustler encodes `Vec<u8>` as a *list of integers*, not a binary. Use `NewBinary` (preferred on Rustler 0.30+) or `OwnedBinary::new(n).release(env)` (older API still in active use, e.g. nimble_lz4 on Rustler 0.36).
    ```rust
    // BAD: Vec<u8> becomes [104, 101, 108, 108, 111] in Elixir (a list!)
    fn compress(data: Binary) -> Vec<u8> {
        do_compress(data.as_slice())
    }

    // GOOD: NewBinary becomes a proper Elixir binary
    fn compress(env: Env, data: Binary) -> Binary {
        let compressed = do_compress(data.as_slice());
        let mut output = NewBinary::new(env, compressed.len());
        output.as_mut_slice().copy_from_slice(&compressed);
        output.into()
    }

    // ALSO GOOD: OwnedBinary::new(n).release(env) — older API, still valid
    // (nimble_lz4 uses this pattern on current Rustler)
    fn compress(env: Env, data: Binary) -> Binary {
        let compressed = do_compress(data.as_slice());
        let mut owned = OwnedBinary::new(compressed.len()).expect("alloc");
        owned.as_mut_slice().copy_from_slice(&compressed);
        Binary::from_owned(owned, env)
    }
    ```

19. **ALWAYS establish a tokio runtime context for NIFs that touch tokio-dependent crates** — quinn (QUIC), mDNS, wasmtime async, libp2p, etc. panic with "there is no reactor running" if not inside a runtime context. Two valid shapes:
    - **`runtime.enter()` guard** (the canonical sync-construction pattern): use when the NIF body calls a sync constructor (like `quinn::Endpoint::new`) that internally needs a tokio context.
      ```rust
      #[rustler::nif(schedule = "DirtyIo")]
      fn create_node(config: Config) -> Result<ResourceArc<NodeHandle>, String> {
          let runtime = RUNTIME.get().ok_or("runtime not initialized")?;
          let _guard = runtime.enter();  // Establishes tokio context for this thread
          let node = build_node(config).map_err(|e| e.to_string())?;
          Ok(ResourceArc::new(node))
      }
      ```
    - **`RUNTIME.spawn(async move { ... env.send(...) })`** (wasmex pattern): when the NIF wraps the entire body in a spawned task and replies via `OwnedEnv::send_and_clear`, no `.enter()` is needed — `spawn` establishes the context inside the task.
      ```rust
      #[rustler::nif(schedule = "DirtyIo")]
      fn run_async(env: Env, from: Term, work: ExWork) -> Atom {
          let pid = env.pid();
          let from_owned = OwnedEnv::new();  // capture from = {pid, ref}
          RUNTIME.spawn(async move {
              let result = work.do_async().await;
              let mut owned = OwnedEnv::new();
              let _ = owned.send_and_clear(&pid, |e| (atoms::reply(), result).encode(e));
          });
          atoms::ok()
      }
      ```

    For the runtime itself, **either `OnceLock<Runtime>` or `LazyLock<Runtime>`** is fine — `LazyLock` (stable since Rust 1.80) is more ergonomic since it doesn't need a wrapper `runtime()` function.

20. **ALWAYS use `#[rustler::resource_impl]`** on `impl Resource for T` blocks — in Rustler 0.36+, bare `impl Resource for T {}` does NOT register the resource type and causes a runtime panic (`called Option::unwrap() on a None value`). The attribute macro performs registration AND auto-generates `IMPLEMENTS_*` constants by detecting which callback functions (`down`, `destructor`, `dyncall`) are present — never set these booleans manually.
    ```rust
    // BAD: compiles but panics at runtime — resource not registered
    impl rustler::Resource for MyResource {}

    // BAD: macro on bare impl can't see callbacks, won't set IMPLEMENTS_DOWN
    impl rustler::Resource for MyResource {
        const IMPLEMENTS_DOWN: bool = true; // manual — fragile, unnecessary
        fn down<'a>(&'a self, env: Env<'a>, pid: LocalPid, mon: Monitor) { }
    }
    #[rustler::resource_impl]
    impl MyResource {}

    // GOOD: macro on Resource impl — detects `down`, auto-sets IMPLEMENTS_DOWN
    #[rustler::resource_impl]
    impl rustler::Resource for MyResource {
        fn down<'a>(&'a self, env: Env<'a>, pid: LocalPid, mon: Monitor) { }
    }
    ```

## 2.2 "Which construct?" decision table — at the moment of writing

This table fires when you've decided "I need to do X" and are about to type. Each row maps an intent to the correct Rustler construct and the common anti-pattern to avoid.

| When you need to... | Use this | NOT this |
|---|---|---|
| Return success-only void | `Atom` (return `atoms::ok()`) | `Result<Atom, _>` (causes `{:ok, :ok}`) |
| Return a value that may fail | `Result<T, E>` where `T` ≠ `Atom` | `Result<Atom, _>` |
| Return binary data to Elixir | `NewBinary` (or `Binary<'a>` from input) | `Vec<u8>` (becomes list of ints) |
| Accept binary data from Elixir | `Binary<'a>` parameter | `Vec<u8>` parameter (forces copy + decode) |
| Persist binary data on Rust side beyond NIF call | `data.as_slice().to_vec()` to own a `Vec<u8>` | Trying to keep `Binary<'a>` past NIF return |
| Mutate ResourceArc state | Clone-based: `clone_inner()` then return new `ResourceArc` | `Mutex` around the resource |
| Mutex on shared cross-call state, normal scheduler, or contention is plausible | `try_lock()` + return `:lock_fail` on failure | `.lock().unwrap()` (can deadlock the scheduler) |
| Mutex on per-call scratch state inside a freshly-constructed struct on DirtyCpu (mdex pattern) | Blocking `.lock().unwrap()` is fine — single owner, no real contender | Adding `try_lock` ceremony where there's no contention |
| Read-heavy shared state | `RwLock::try_read` / `try_write` | `Mutex` |
| Send a message back to Elixir from a NIF call | `env.send(&pid, term)` | `OwnedEnv` (only for non-BEAM threads) |
| Send a message from a non-BEAM thread | `OwnedEnv::new()` + `send_and_clear` | `env.send` (env is invalid after NIF returns) |
| Construct quinn / mDNS / tokio resource | `runtime.enter()` first, then construct | Direct construct (panics: "no reactor running") |
| Wrap upstream library that may panic | `nif_safe(\|\| ... )` (catch_unwind helper) | Bare call (BEAM crashes on panic) |
| Register a Resource type | `#[rustler::resource_impl] impl rustler::Resource for T { ... }` | Bare `impl Resource for T {}` (runtime panic) |
| Define resource callbacks (down/destructor/dyncall) | Define as `fn`s inside the `resource_impl` block | Manual `IMPLEMENTS_DOWN: bool = true` |
| Map atom → atom set | `rustler::atoms! { ok, error, ... }` | String allocation for known atoms |
| Convert error to Elixir term | impl `Encoder` on your error type | Manual term construction every NIF |
| CPU work > ~100µs | `#[rustler::nif(schedule = "DirtyCpu")]` | Default scheduler (blocks BEAM) |
| Blocking IO | `#[rustler::nif(schedule = "DirtyIo")]` | Default scheduler |
| Convert option to NIF result | `.ok_or(atoms::not_found())` | `.unwrap()` (BEAM crash) |
| Hold a lock around expensive work | Drop guard inside scoped block first | Hold guard across `expensive_work()` |
| Define a domain identifier | Newtype (`struct UserId(u64)`) | Bare `u64` (mixable in calls) |
| Pass NifMap from Elixir with optional fields | Wrap in Elixir helper that fills `nil` for absent keys | Direct call (`ArgumentError` on missing keys) |
| Read-only string param | `&str` | `String` (forces allocation) |
| Owned string for storage / async / struct field | `String` | `&str` (lifetime won't outlive NIF call) |

### Return Type Matrix — How Rustler Encodes Results

**Critical distinction**: `NifResult<T>` (= `Result<T, rustler::Error>`) is **special-cased** by Rustler — `Ok(value)` returns `value` directly and `Err(e)` raises an exception. Generic `Result<T, E>` with a user-defined error type that has its own `Encoder` produces `{:ok, T}` / `{:error, encoded}` — that's the case where double-wrapping bites.

| Rust return type | Elixir sees | Correct? |
|---|---|---|
| `NifResult<Atom>` returning `Ok(atoms::ok())` | `:ok` (raises on `Err`) | **Good** — Rustler unwraps `Ok` for `NifResult`; widely used in y_ex, html5ever_elixir |
| `NifResult<ResourceArc<T>>` | `ref` (raises on `Err`) | **Good** — same special-case |
| `NifResult<Vec<String>>` | `["..."]` (raises on `Err`) | **Good** |
| `Result<Atom, MyError>` (with user `Encoder` for `MyError`) returning `Ok(atoms::ok())` | `{:ok, :ok}` / `{:error, encoded}` | **BAD** — double-wrapped |
| `Result<Atom, Atom>` returning `Ok(atoms::ok())` | `{:ok, :ok}` / `{:error, e}` | **BAD** — double-wrapped |
| `Result<(Atom, ResourceArc<T>), MyError>` | `{:ok, {:ok, ref}}` / `{:error, e}` | **BAD** — double-wrapped |
| `Result<ResourceArc<T>, MyError>` | `{:ok, ref}` / `{:error, e}` | Good |
| `Result<Vec<String>, MyError>` | `{:ok, ["..."]}` / `{:error, e}` | Good |
| `Result<bool, MyError>` | `{:ok, true}` / `{:error, e}` | Good |
| `Atom` | `:ok` | Good — for fire-and-forget |
| `ResourceArc<T>` | `ref` | Good — when it can't fail |
| `(Atom, Vec<String>)` | `{:ok, ["..."]}` | Good — tuple becomes tagged tuple |

**The rules:**
1. **Use `NifResult<T>` (alias for `Result<T, rustler::Error>`) when you want exception-raising errors.** `Ok` is unwrapped; `Err` raises. Returning `NifResult<Atom>` with `Ok(atoms::ok())` is fine — Elixir sees plain `:ok`.
2. **Use generic `Result<T, MyError>` (with your own `Encoder` for `MyError`) when you want `{:ok, ...}`/`{:error, ...}` tuples.** In this case, the `Ok(...)` value is wrapped in `{:ok, ...}` — don't put a tagged atom inside or you double-wrap.
3. **The `{:ok, :ok}` trap** specifically affects generic `Result<Atom, MyError>` shapes (or `Result<Atom, Atom>`), NOT `NifResult<Atom>`. When in doubt, check the error type: if it's `rustler::Error`, you're in the special-cased path.

### Scheduler — which scheduler? (decision tree at writing time)

```
Does this NIF call:
├── Block on IO (file read, socket send/recv, disk sync, any `std::fs::*`)? → DirtyIo
├── Do CPU work > ~100µs typical (image proc, crypto, parsing, compression)?  → DirtyCpu
├── Call a blocking wait (channel recv without try, mutex blocking lock)?     → DirtyIo (safer)
├── Enter a tokio runtime or construct a tokio-dependent resource?            → DirtyIo + runtime.enter()
├── Call an upstream library that MAY panic?                                  → DirtyCpu/DirtyIo + catch_unwind
└── Pure CPU, < ~100µs, deterministic, no locks?                              → normal scheduler (default)
```

**Thresholds in practice:**

| NIF work | Scheduler | Notes |
|---|---|---|
| `a + b`, constant-time getters, struct field reads | normal | Pure arithmetic, inline |
| Small JSON decode (< 1KB), small binary parse (< few hundred bytes) | normal | Usually fast enough |
| Decode/encode larger data, hashing, small compression | DirtyCpu | Safer even if it might be fast |
| `reqwest::get`, `tokio::fs`, any network or disk I/O | DirtyIo | Plus `runtime.enter()` if tokio-based |
| Heavy math (image convolution, linear algebra, training step) | DirtyCpu | Plus `catch_unwind` if the upstream lib may panic |
| Wait on a channel or mutex that can genuinely block | DirtyIo | Normal scheduler must never block |

Err on DirtyCpu when in doubt — the overhead of scheduling on a dirty scheduler is much smaller than the cost of a 10ms stall on a normal scheduler.

## 2.3 Top BAD/GOOD pairs — the BEAM-crashers

The anti-patterns most likely to be written by an LLM and most likely to crash production. Each pair fires during code validation. (Code is intentionally close to rules 2/4/13/18/19/20 above — repetition is the design.)

### 1. Double-wrapping `{:ok, :ok}`

```rust
// BAD: generic Result<Atom, _> with Ok(atoms::ok()) → Elixir sees {:ok, :ok}
fn insert(set: ResourceArc<Set>) -> Result<Atom, Atom> { Ok(atoms::ok()) }
```

```rust
// GOOD: NifResult<Atom> — Rustler unwraps Ok and raises on Err; Elixir sees :ok
fn insert(set: ResourceArc<Set>) -> NifResult<Atom> { Ok(atoms::ok()) }

// GOOD: bare Atom for void operations that can't fail
fn insert(set: ResourceArc<Set>) -> Atom { atoms::ok() }

// GOOD: generic Result with a meaningful (non-tagged) Ok value
fn insert(set: ResourceArc<Set>) -> Result<usize, Atom> { Ok(set_len) }
```

The trap is specific to **generic** `Result<Atom, MyError>` (with a user `Encoder`). `NifResult<T>` (= `Result<T, rustler::Error>`) is special-cased and does NOT double-wrap.

### 2. `Vec<u8>` returned as binary

```rust
// BAD: Elixir sees [31, 139, 8, ...] — list of ints, not a binary
fn compress(data: Binary) -> Vec<u8> { gzip(data.as_slice()) }
```

```rust
// GOOD: NewBinary becomes a real Elixir binary
fn compress<'a>(env: Env<'a>, data: Binary) -> Binary<'a> {
    let out_bytes = gzip(data.as_slice());
    let mut output = NewBinary::new(env, out_bytes.len());
    output.as_mut_slice().copy_from_slice(&out_bytes);
    output.into()
}
```

### 3. Bare `impl Resource` without `#[resource_impl]`

```rust
// BAD: compiles, but runtime panic on first ResourceArc::new()
impl rustler::Resource for MyResource {}
```

```rust
// GOOD: macro registers the resource type AND auto-detects callbacks
#[rustler::resource_impl]
impl rustler::Resource for MyResource {}
```

### 4. Blocking `.lock().unwrap()` inside a NIF

```rust
// BAD: deadlocks the scheduler if locked; panics on poison → BEAM crash
let guard = res.data.lock().unwrap();
```

```rust
// GOOD: try_lock + explicit lock-fail return
let guard = res.data.try_lock().ok_or(atoms::lock_fail())?;
```

### 5. Holding MutexGuard across expensive work

```rust
// BAD: blocks every other NIF that touches this resource for the duration
let guard = res.data.try_lock().ok_or(atoms::lock_fail())?;
let result = expensive_computation(&guard);  // guard still held
result
```

```rust
// GOOD: drop the guard before the expensive work
let value = {
    let guard = res.data.try_lock().ok_or(atoms::lock_fail())?;
    guard.clone()
}; // guard dropped here
expensive_computation(value)
```

### 6. Constructing a tokio resource without `runtime.enter()`

```rust
// BAD: panics with "there is no reactor running"
#[rustler::nif(schedule = "DirtyIo")]
fn open_quic_endpoint(config: EndpointConfig) -> Result<ResourceArc<Endpoint>, String> {
    let endpoint = quinn::Endpoint::server(config.server, config.addr)
        .map_err(|e| e.to_string())?;
    Ok(ResourceArc::new(Endpoint(endpoint)))
}
```

```rust
// GOOD: enter the runtime so quinn can find the reactor
#[rustler::nif(schedule = "DirtyIo")]
fn open_quic_endpoint(config: EndpointConfig) -> Result<ResourceArc<Endpoint>, String> {
    let rt = runtime();
    let _guard = rt.enter();
    let endpoint = quinn::Endpoint::server(config.server, config.addr)
        .map_err(|e| e.to_string())?;
    Ok(ResourceArc::new(Endpoint(endpoint)))
}
```

### 7. Calling a panic-prone library without `catch_unwind`

```rust
// BAD: library panic on malformed input → entire BEAM crashes
#[rustler::nif(schedule = "DirtyCpu")]
fn train(tok: ExTokenizer, files: Vec<String>) -> Result<ExTokenizer, MyError> {
    let mut t = tok.clone_inner();
    t.train(&files)?;  // panics inside the library on bad data
    Ok(ExTokenizer::new(t))
}
```

```rust
// GOOD: nif_safe converts panics into MyError::Panic
#[rustler::nif(schedule = "DirtyCpu")]
fn train(tok: ExTokenizer, files: Vec<String>) -> Result<ExTokenizer, MyError> {
    nif_safe(|| {
        let mut t = tok.clone_inner();
        t.train(&files)?;
        Ok(ExTokenizer::new(t))
    })
}
```

### 8. `unwrap()` in a NIF body

```rust
// BAD: Option::None or Result::Err → panic → BEAM crash
fn get_user(id: i32) -> User { lookup_user(id).unwrap() }
```

```rust
// GOOD: convert to Result with ok_or
fn get_user(id: i32) -> Result<User, Atom> { lookup_user(id).ok_or(atoms::not_found()) }
```

### 9. Default scheduler with multi-ms work

```rust
// BAD: blocks the scheduler for 5 seconds — kills concurrency
#[rustler::nif]
fn slow_op() { std::thread::sleep(Duration::from_secs(5)); }
```

```rust
// GOOD: dirty scheduler tolerates blocking
#[rustler::nif(schedule = "DirtyIo")]
fn slow_op() { std::thread::sleep(Duration::from_secs(5)); }
```

### 10. `send_and_clear` from a BEAM thread

```rust
// BAD: panics — env is BEAM-managed, send_and_clear is for OwnedEnv from non-BEAM threads
#[rustler::nif]
fn notify(env: Env, pid: LocalPid) -> Atom {
    let mut owned = OwnedEnv::new();
    owned.send_and_clear(&pid, |e| atoms::ok().encode(e));
    atoms::ok()
}
```

```rust
// GOOD: from inside a NIF, use env.send directly
#[rustler::nif]
fn notify<'a>(env: Env<'a>, pid: LocalPid, msg: Term<'a>) -> Atom {
    let _ = env.send(&pid, msg);
    atoms::ok()
}

// GOOD: send_and_clear belongs in std::thread::spawn, NOT in a NIF body
#[rustler::nif]
fn async_work(env: Env, data: String) -> Atom {
    let pid = env.pid();
    std::thread::spawn(move || {
        let result = expensive(&data);
        let mut owned = OwnedEnv::new();
        let _ = owned.send_and_clear(&pid, |e| (atoms::result(), result).encode(e));
    });
    atoms::ok()
}
```

## 2.4 Production Patterns (from Explorer, Tokenizers, Discord SortedSet, mdex, nimble_lz4, html5ever_elixir, wasmex, franz, y_ex, rhai_rustler, midiex, ex_secp256k1, resvg_nif, candlex, extism_elixir)

The patterns below are drawn from production Rustler libraries. The canonical references are:

- **Explorer** (`elixir-explorer/explorer`) — DataFrame/Polars wrapper; clone-based immutability, `thiserror + Encoder`, sibling top-level `Polars.Backend.Native`.
- **Tokenizers** (`elixir-nx/tokenizers`) — HuggingFace tokenizers wrapper; multiple ResourceArc types, `thiserror + Encoder`, explicit `catch_unwind` for panicky training paths.
- **Discord SortedSet** (`discord/sorted_set_nif`) — high-throughput shared state; `try_lock` with `:lock_fail` return, atom errors, slab-based handle pool.
- **mdex** (`leandrocp/mdex`) — markdown framework lib (comrak/lol_html/ammonia); production use of `parking_lot`, blocking `.lock().unwrap()` in DirtyCpu scratch state, no explicit `catch_unwind` (relies on Rustler's boundary catch), sibling top-level `MDEx.Native`, errors as `(error_atom, reason_string)` tuples wrapped in Elixir `defexception`.
- **nimble_lz4** (`whatyouhide/nimble_lz4`) — small focused LZ4 binding; `OwnedBinary::new(n).release(env)` pattern, no separate Native module (stub-as-public).
- **html5ever_elixir** (`rusterlium/html5ever_elixir`) — HTML parser, by the Rustler maintainer; nested `Html5ever.Native`, encoder-as-string error shape, no resource types.
- **wasmex** (`tessi/wasmex`) — Wasmtime runtime wrapper; six `Mutex<!Sync foreign-handle>` resources, `LazyLock<Runtime>` + `RUNTIME.spawn(...)` (no `runtime.enter()`), GenServer-reply-from-spawned-task pattern, both `NewBinary` and `OwnedBinary` in the same crate.
- **franz** (`scrogson/franz`) — Kafka client (rdkafka + tokio); async actor-style worker per resource, `OnceCell + RUNTIME.spawn`, sibling stub-as-public `Franz.Native`, atom/tuple error shape.
- **y_ex** (`satoren/y_ex`) — Yjs CRDT bindings; observer/subscription patterns, `scoped_thread_local!` for sync foreign callbacks, `TermBox` for opaque term capture, `SliceIntoBinary` lazy encoder, `make_subbinary` zero-copy slicing, `Mutex<Option<T>>` RAII unsubscribe, `mem::transmute` to `'static` for resource-stored borrows (with companion ResourceArc).
- **rhai_rustler** (`rhaiscript/rhai_rustler`) — Rhai scripting engine bindings; `try_lock().unwrap()` fail-loud locking convention, sibling `cdylib` plugin crate workspace pattern, `load` callback used to pin global state (hash seed) before any NIF runs, typed Encoder error shape.
- **midiex** (`haubie/midiex`) — Cross-platform MIDI I/O; `thread_local!` for thread-affine OS device handles, dedicated `std::thread` + `CFRunLoop::run_current()` for macOS hot-plug callbacks, explicit-close NIF companion to RAII Drop for foreign handles requiring `.close()` to flush.
- **ex_secp256k1** (`ayrat555/ex_secp256k1`) — Ethereum secp256k1 ECDSA NIF; canonical fixed-width crypto-input pattern (`copy_from_slice` into `[u8; N]` after explicit length check), pre-baked size-error atoms in `rustler::atoms!`, no resources / no tokio / no dirty schedulers (sub-100µs operations on prehashed input).
- **resvg_nif** (`mrdotb/resvg_nif`) — SVG → PNG renderer (resvg/usvg/tiny-skia); custom `Decoder + Encoder` newtype wrappers for foreign-crate enums (orphan-rule workaround), `try_or_return_elixir_err!` macro for stringified errors. **Cautionary example**: ships *with* Rule 1 (no `DirtyCpu` on multi-ms render) and Rule 18 (`Vec<u8>` returned as binary, surfaced as `@type png_buffer :: [0..255]` charlist) violations despite hex.pm popularity — both correctly flagged by §3.1.
- **candlex** (`mimiquate/candlex`) — Hugging Face Candle bindings as Nx tensor backend; canonical tensor wrapper (`ExTensor { device: Atom, resource: ResourceArc<TensorRef> }` with `Deref`), per-dtype binary encoding via `to_ne_bytes` + exhaustive match including `BF16`/`F16`, `static Mutex<Option<Device>>` for lazy CUDA device cache, `mode: :release` pin in RustlerPrecompiled stub (debug-mode integer-overflow incompatibility with PRNG kernels), `default-features = false` Cargo declaration with explicit `nif_version_2_16` feature.
- **extism_elixir** (`extism/elixir-sdk`) — Extism WebAssembly plugin host; sibling `CancelHandle` resource for cooperative cancellation of long-running NIF calls (Elixir holds the handle, calls `cancel` from a sister process), `Mutex<Option<Plugin>>` for explicit-free RAII. **Cautionary example**: ships with Rule 1 violation (`plugin_call` runs arbitrary user WASM with no scheduler annotation) — correctly flagged.

Different libraries make different trade-offs — the patterns below aren't a single canonical recipe but a spectrum of approaches calibrated to library size and shape.

### Resource Wrapper Pattern (ResourceArc + NifStruct + Deref)

The de facto standard for exposing Rust types to Elixir, used by Explorer (4 types) and Tokenizers (8+ types):

```rust
// 1. Inner resource — holds the actual data
pub struct ExDataFrameRef(pub DataFrame);

#[rustler::resource_impl]
impl Resource for ExDataFrameRef {}

// 2. NifStruct wrapper — this is what Elixir sees as %Module{}
#[derive(NifStruct)]
#[module = "MyApp.DataFrame"]
pub struct ExDataFrame {
    pub resource: ResourceArc<ExDataFrameRef>,
}

// 3. Deref for ergonomic access in NIF functions
impl Deref for ExDataFrame {
    type Target = DataFrame;
    fn deref(&self) -> &Self::Target { &self.resource.0 }
}

// 4. Constructor
impl ExDataFrame {
    pub fn new(df: DataFrame) -> Self {
        Self { resource: ResourceArc::new(ExDataFrameRef(df)) }
    }
}

// 5. Clone for mutation (avoids mutex entirely)
impl ExDataFrameRef {
    pub fn clone_inner(&self) -> DataFrame { self.0.clone() }
}
```

### Custom Error Type with thiserror + Encoder

Used by Explorer and Tokenizers — the **library-scale** pattern when you're surfacing a complex domain error type with multiple meaningful variants (parse vs IO vs validation vs upstream-library-bug) and want each to be distinguishable on the Elixir side.

**Smaller libs return `(error_atom, reason_string)` tuples** and let Elixir wrap them in `defexception` modules. mdex (`MDEx.Native` returning `(:error, msg)` and `MDEx.Errors` defining the exception types) and nimble_lz4 both take this shape — no Rust-side custom error type needed.

**Encoder-as-string** is a third option (used by html5ever_elixir): a `thiserror` enum that encodes to a bare string term via `format!("{self}").encode(env)`. Cleaner than the tuple shape when the Rust side already has a meaningful Display impl per variant, and removes the need for an Elixir `defexception` layer:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Html5everError {
    #[error("invalid utf-8: {0}")]
    InvalidUtf8(String),
    #[error("parser failed: {0}")]
    Parser(String),
}

impl rustler::Encoder for Html5everError {
    fn encode<'a>(&self, env: Env<'a>) -> Term<'a> {
        format!("{self}").encode(env)  // bare string; Rustler wraps as {:error, "..."}
    }
}
```

Pick the heavier `thiserror + Encoder` (with structured terms) shape only when the variant set is large enough that Elixir-side callers want to pattern-match on it.

Library-scale shape:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum MyError {
    #[error("upstream: {0}")]
    Upstream(#[from] upstream::Error),  // auto-convert with ?
    #[error("internal: {0}")]
    Internal(String),
    #[error("{0}")]
    Other(String),
}

impl rustler::Encoder for MyError {
    fn encode<'a>(&self, env: Env<'a>) -> Term<'a> {
        format!("{self}").encode(env)  // Returns error string to Elixir
    }
}

// Required for catch_unwind compatibility
impl std::panic::RefUnwindSafe for MyError {}

// All NIFs return Result<T, MyError> — clean ? operator chains
#[rustler::nif(schedule = "DirtyCpu")]
fn process(data: ExMyData) -> Result<ExMyData, MyError> {
    let mut inner = data.clone_inner();
    inner.transform()?;  // upstream::Error auto-converts via #[from]
    Ok(ExMyData::new(inner))
}
```

### catch_unwind for Upstream Libraries That May Panic

**Note on default behavior:** Rustler catches panics at the NIF boundary by default — a panic inside a `#[rustler::nif]` function is converted to an error term raised on the Elixir side rather than crashing the BEAM. So for many wrappers the implicit boundary catch is sufficient (mdex wraps comrak / lol_html / ammonia without any explicit `catch_unwind`). Reach for explicit `catch_unwind` when you need:

- **Custom error variant** for the panic case (e.g., `MyError::Panic(msg)` with the panic payload extracted) so callers can pattern-match it differently from regular errors.
- **Cleanup work on panic** — releasing a resource, logging, recording a telemetry event before the error propagates.
- **Panic-safe FFI boundary** when calling into C/C++ from inside the NIF body — unwinding into non-Rust code is UB; convert to an error there.

When you do reach for it, the shape:

```rust
use std::panic;

#[rustler::nif(schedule = "DirtyCpu")]
fn train(tokenizer: ExTokenizer, files: Vec<String>) -> Result<ExTokenizer, MyError> {
    let result = panic::catch_unwind(|| {
        let mut tok = tokenizer.resource.0.clone();
        tok.train(&files).map_err(MyError::from)?;
        Ok(tok)
    });
    match result {
        Ok(Ok(tok)) => Ok(ExTokenizer::new(tok)),
        Ok(Err(e)) => Err(e),
        Err(panic_info) => Err(MyError::Internal(
            format!("panic during training: {:?}", panic_info)
        )),
    }
}
```

### `nif_safe` Helper — Reusable catch_unwind Wrapper

When you've decided you need the explicit-catch shape across multiple DirtyCpu NIFs (custom error variant or panic-time cleanup, per the previous section), the catch_unwind shape gets repetitive. Factor it out:

```rust
use std::panic::{self, UnwindSafe};

/// Wraps NIF work in catch_unwind. Converts panics into MyError::Panic.
/// Use for any DirtyCpu/DirtyIo NIF calling an upstream library that may panic.
fn nif_safe<T, F>(f: F) -> Result<T, MyError>
where
    F: FnOnce() -> Result<T, MyError> + UnwindSafe,
{
    match panic::catch_unwind(f) {
        Ok(Ok(v)) => Ok(v),
        Ok(Err(e)) => Err(e),
        Err(panic_payload) => {
            let msg = panic_payload
                .downcast_ref::<String>()
                .cloned()
                .or_else(|| panic_payload.downcast_ref::<&str>().map(|s| s.to_string()))
                .unwrap_or_else(|| "unknown panic".to_string());
            Err(MyError::Panic(msg))
        }
    }
}

// Usage — every panic-prone NIF becomes a thin wrapper
#[rustler::nif(schedule = "DirtyCpu")]
fn train(tokenizer: ExTokenizer, files: Vec<String>) -> Result<ExTokenizer, MyError> {
    nif_safe(|| {
        let mut tok = tokenizer.resource.0.clone();
        tok.train(&files)?;
        Ok(ExTokenizer::new(tok))
    })
}

#[rustler::nif(schedule = "DirtyCpu")]
fn encode_batch(tokenizer: ExTokenizer, texts: Vec<String>) -> Result<Vec<ExEncoding>, MyError> {
    nif_safe(|| {
        texts.iter()
            .map(|t| tokenizer.encode(t.clone(), true).map(ExEncoding::new))
            .collect::<Result<_, _>>()
            .map_err(MyError::from)
    })
}
```

Add this helper once per crate; use it on every NIF that might touch panicking upstream code. `UnwindSafe` bound comes free for most data types; if you hit a real `UnwindSafe` violation (usually with `RefCell` / `Mutex` in the captured state), wrap with `AssertUnwindSafe(...)` at the call site after thinking about why the guarantee is OK.

### Tokio Runtime-Enter Pattern

Any tokio-dependent crate (quinn/QUIC, mdns, reqwest with features, rust-libp2p, wasmtime async, some SDKs) panics with "there is no reactor running" if you construct its async resources outside a tokio runtime context. Use `OnceLock<Runtime>` at module scope plus `runtime.enter()` inside every NIF that touches async resources:

```rust
use std::sync::OnceLock;
use tokio::runtime::{Builder, Runtime};

static RUNTIME: OnceLock<Runtime> = OnceLock::new();

fn runtime() -> &'static Runtime {
    RUNTIME.get_or_init(|| {
        Builder::new_multi_thread()
            .worker_threads(2)                  // Small pool — the BEAM already handles scale
            .thread_name("my-nif-rt")
            .enable_all()
            .build()
            .expect("tokio runtime init")
    })
}

#[rustler::nif(schedule = "DirtyIo")]
fn start_node(config: NodeConfig) -> Result<ResourceArc<NodeHandle>, MyError> {
    let rt = runtime();
    let _guard = rt.enter();                 // Required — quinn::Endpoint::new() looks up runtime here

    let endpoint = rt.block_on(async move {
        quinn::Endpoint::server(config.server_config, config.addr)
    }).map_err(|e| MyError::Other(e.to_string()))?;

    let (tx, rx) = tokio::sync::mpsc::channel(256);
    rt.spawn(run_node_loop(endpoint.clone(), rx));   // Long-running work goes on tokio

    Ok(ResourceArc::new(NodeHandle { endpoint, sender: tx }))
}
```

Key points:
- `OnceLock<Runtime>` is lazy — built on first use, not at `init!` time.
- Small `worker_threads` (2–4) because the BEAM is the scale-out layer.
- `runtime.enter()` inside every NIF that *constructs* async resources.
- `block_on` is fine — we're on a DirtyIo scheduler, which can block.
- Long-running tasks go on `rt.spawn(...)` so they don't tie up the NIF call.

**Two production variants** (see [patterns.md](patterns.md) for full code):
- **`LazyLock<Runtime>` (Rust 1.80+)** drops the wrapper `runtime()` fn — direct `RUNTIME.enter()`.
- **`RUNTIME.spawn(...)` instead of `.enter()`** (wasmex pattern) — for fire-and-forget async NIFs that wrap the entire body in a spawned task and reply via `OwnedEnv::send_and_clear`. `spawn` establishes the tokio context inside the spawned task; no explicit `.enter()` needed.

Related: `libp2p` skill has the full rust-libp2p + NIF pattern built on this shape.

### Binary Data — `Binary` / `NewBinary` / `OwnedBinary`, NOT `Vec<u8>`

This is one of the most common NIF bugs: `Vec<u8>` does NOT become an Elixir binary. Rustler encodes it as a list of integers (`[104, 101, 108, 108, 111]` for `b"hello"`). To return binary data to Elixir, use `NewBinary` (preferred on current Rustler) or `OwnedBinary::new(n).release(env)` (older API still in active use, e.g. nimble_lz4 on Rustler 0.36 — both produce real binaries). To accept binary data from Elixir, use `Binary`.

```rust
// BAD — Vec<u8> becomes a list in Elixir, not a binary
#[rustler::nif(schedule = "DirtyCpu")]
fn compress(data: Binary) -> Vec<u8> {
    do_compress(data.as_slice())
    // Elixir sees: [31, 139, 8, 0, ...]  ← list of ints, NOT a binary
}
```

```rust
// GOOD — NewBinary produces a real Elixir binary
#[rustler::nif(schedule = "DirtyCpu")]
fn compress<'a>(env: Env<'a>, data: Binary) -> Binary<'a> {
    let compressed = do_compress(data.as_slice());
    let mut output = NewBinary::new(env, compressed.len());
    output.as_mut_slice().copy_from_slice(&compressed);
    output.into()
}
```

**Decoding binaries from Elixir** — use `Binary<'a>` as a parameter type. It's a zero-copy view into the Elixir-owned binary for the duration of the NIF call. For data that must outlive the call (ResourceArc state, sent to another thread), call `.as_slice().to_vec()` to own a `Vec<u8>` on the Rust side.

**Why Rustler encodes `Vec<u8>` as a list:** Rust's type system doesn't distinguish "a vector of bytes intended as binary" from "a vector of 8-bit integers intended as a list of small numbers." `NewBinary` was added specifically to disambiguate for the binary case. Same reasoning for accepting — `Binary` is an explicit contract that the Elixir caller passes a binary, not a list that happens to contain bytes.

### Bare Return Types for Infallible Getters

For simple property accessors that cannot fail, return the value directly without Result wrapping. This avoids unnecessary `{:ok, value}` on the Elixir side:

```rust
// GOOD: bare return for infallible getter — Elixir sees just the value
#[rustler::nif]
fn get_vocab_size(tokenizer: ExTokenizer) -> usize {
    tokenizer.get_vocab_size(true)
}

// GOOD: Option for nullable getters — Elixir sees value or nil
#[rustler::nif]
fn get_model(tokenizer: ExTokenizer) -> Option<ExModel> {
    tokenizer.get_model().map(ExModel::new)
}

// Use Result only when the operation can genuinely fail
#[rustler::nif(schedule = "DirtyCpu")]
fn encode(tokenizer: ExTokenizer, text: String) -> Result<ExEncoding, MyError> {
    let enc = tokenizer.encode(text, true)?;
    Ok(ExEncoding::new(enc))
}
```

### RAII Unsubscribe-on-Drop — `Mutex<Option<T>>`

Many foreign libraries' observer/subscription handles unsubscribe when dropped (yrs `Subscription`, libuv watchers, Wasmtime `Plugin`, `MidiOutputConnection`, etc.). Wrap the handle in `Mutex<Option<T>>` so an explicit `unsubscribe`/`close`/`free` NIF can `.take()` it before GC, while Drop handles the unhappy path:

```rust
pub struct SubResource(Mutex<Option<yrs::Subscription>>);

#[rustler::resource_impl]
impl rustler::Resource for SubResource {}

#[rustler::nif]
fn unsubscribe(sub: ResourceArc<SubResource>) -> Atom {
    // Take() drops the Subscription immediately, firing the unsubscribe.
    // If Elixir never calls this, the resource's Drop will GC eventually and unsubscribe then.
    let mut guard = sub.0.lock().expect("sub lock");
    drop(guard.take());
    atoms::ok()
}
```

The `Option` lets you distinguish "still subscribed" from "explicitly unsubscribed" without a separate flag. Used in y_ex (subscriptions), extism_elixir (plugin lifecycle), midiex (MIDI connection close) — see [patterns.md](patterns.md) for the full **explicit-close NIF companion to RAII Drop** variant when the foreign API needs `.close()` to flush.

### Specialized Patterns Index — see [patterns.md](patterns.md)

The following 17 production patterns live in [patterns.md](patterns.md) (loaded on demand for specialized work). Each is a complete template with rationale and use-when conditions:

| Domain | Pattern | When you need it |
|---|---|---|
| Foreign callbacks | `scoped_thread_local!` for sync foreign callbacks (y_ex) | Foreign lib invokes Rust callback synchronously during a NIF; callback needs `Env` |
| Foreign callbacks | `TermBox` for long-lived opaque term capture (y_ex) | Subscription/observer must retain a user-supplied `Term` past the NIF call |
| Binary encoding | `SliceIntoBinary` lazy `Encoder` wrapper (y_ex) | Defer `NewBinary` allocation until rustler actually encodes (avoids waste in `Result`/`Option`) |
| Binary encoding | `Binary::make_subbinary` zero-copy slicing (y_ex) | Return a contiguous sub-region of an input `Binary` without allocating |
| Decoder/Encoder | Atom-valued enum wrappers via custom `Decoder + Encoder` (resvg_nif) | Map Elixir atoms to a foreign-crate enum you can't derive on (orphan rule) |
| Resource lifecycle | `mem::transmute` to `'static` for resource-stored borrows (y_ex) | Stash a borrowed foreign handle (e.g., `TransactionMut<'doc>`) in `ResourceArc` — requires companion ResourceArc to owner |
| Resource lifecycle | Explicit-close NIF companion to RAII Drop (midiex) | Foreign lib needs `.close()` to flush, OR you want close at deterministic point (not GC) |
| OS-specific | `thread_local!` for thread-affine OS handles (midiex) | OS handle is `!Send`/`!Sync` or thread-affine (CoreMIDI client, OpenGL context) |
| OS-specific | macOS `CFRunLoop` for hot-plug callbacks (midiex) | macOS notification API requires CFRunLoop on listening thread (CoreMIDI, IOKit) |
| Crypto NIFs | Fixed-size binary parse + `copy_from_slice` (ex_secp256k1) | Decode 32/33/64/65-byte hashes/keys/signatures safely (vs panicking `try_into`) |
| Image/render NIFs | User-controlled allocation size validation (resvg_nif) | DoS surface: any NIF allocating from user-supplied dimensions or buffer length |
| ML / Tensor | Tensor backend wrapper (NifStruct + atom device + Deref) (candlex) | Nx-backend resource shape: introspectable device/dtype field + opaque ResourceArc |
| ML / Tensor | Per-dtype tensor binary encoding (`to_ne_bytes` + match) (candlex) | Dump typed tensor data as Elixir binary in Nx's native-endian convention; covers BF16/F16 |
| WASM / plugins | Cancel-handle resource for long-running NIFs (extism) | Cooperative cancellation of seconds-long NIF calls from a sister Elixir process |
| Static caching | `static Mutex<Option<T>>` for re-initializable global handles (candlex) | Expensive global (GPU device, runtime) needing first-touch init AND optional reset |
| Build & setup | `load` callback for one-time global setup (rhai_rustler) | Pin global state (hash seed, validate native dep version) before any NIF runs |
| Build & setup | Embedded engine with sibling `cdylib` plugin crate (rhai_rustler) | NIF embeds a scripting engine that loads `cdylib`s at runtime via `libloading` |
| Error shapes | `try_or_return_elixir_err!` macro for stringified errors (resvg) | NIFs returning `NifResult<Term<'a>>` with stringified errors — de-noises 5+ NIFs |
| Tokio variants | `LazyLock<Runtime>` and `RUNTIME.spawn` patterns (wasmex) | LazyLock variant of foundational pattern; spawn-without-enter for fire-and-forget async NIFs |

## 2.5 Rustler Project Setup

### Step 1: Add Dependencies
```elixir
# mix.exs
def application do
  # Convention: NIF wrappers pull :logger (and only :logger if no other
  # Elixir runtime deps) — both candlex and extism_elixir do this.
  [extra_applications: [:logger]]
end

defp deps do
  [
    {:rustler, "~> 0.37.2", runtime: false}
  ]
end
```

### Step 2: Generate NIF
```bash
mix deps.get
mix rustler.new
# Enter module name: MyApp.Native
# Enter NIF name: my_nif
```

### Step 3: Cargo.toml Configuration
```toml
[package]
name = "my_nif"
version = "0.1.0"
edition = "2021"

[lib]
name = "my_nif"
crate-type = ["cdylib"]

[dependencies]
rustler = "0.37.2"
# Or, for explicit NIF version pinning (better cross-OTP compatibility — candlex pattern):
# rustler = { version = "0.37.2", default-features = false, features = ["nif_version_2_16"] }

# Common additions:
# thiserror = "2.0"      # Error handling
# parking_lot = "0.12"   # Faster Mutex (only after measuring contention)
# rayon = "1.10"         # Data parallelism

# [profile.release] is OPTIONAL — most production crates omit it (franz, y_ex, html5ever_elixir,
# wasmex) or set only `lto = true` (mdex, nimble_lz4). Cargo's release defaults are usually fine.
# NEVER set `panic = "abort"` — breaks catch_unwind. Add `lto = "thin"` + `strip = true` for
# CPU-heavy NIFs after benchmarking. To debug a stripped release build:
# RUSTFLAGS="-C debuginfo=2" cargo build --release
```

### Step 4: Basic NIF Structure
```rust
// native/my_nif/src/lib.rs
#[rustler::nif]
fn add(a: i64, b: i64) -> i64 { a + b }

// Rename the Elixir-facing function (Rust name ≠ Elixir name)
#[rustler::nif(name = "compute_sum")]
fn internal_sum(a: i64, b: i64) -> i64 { a + b }

rustler::init!("Elixir.MyApp.Native");
// For one-time setup (validate config, seed PRNG): use the load-callback
// `rustler::init!("Elixir.MyApp.Native", load = on_load)` — see patterns.md.
```

### Step 5: Elixir Module
```elixir
defmodule MyApp.Native do
  use Rustler,
    otp_app: :my_app,
    crate: "my_nif"

  # Every NIF binding MUST have a @spec — it's the boundary between two type systems
  @spec add(integer(), integer()) :: integer()
  def add(_a, _b), do: :erlang.nif_error(:nif_not_loaded)
end
```

### Adding a New NIF Function to an Existing Crate

When adding a function to a crate that already has NIFs:

1. **Rust function** — Add `#[rustler::nif]` function in the appropriate `.rs` file
2. **Register in `lib.rs`** — Add the function name to `rustler::init!("Elixir.MyApp.Native")` (forgetting this is the most common mistake — the NIF compiles but raises `:nif_not_loaded` at runtime)
3. **Elixir stub** — Add `def my_func(...), do: :erlang.nif_error(:nif_not_loaded)` with `@spec` in the native module
4. **Elixir wrapper** — Add the public API function in the appropriate context module

## 2.6 Type Encoding/Decoding

### Type Mapping Table

| Elixir | Rust | Notes |
|--------|------|-------|
| `integer` | `i64`, `i32`, `u64`, etc. | Use appropriate size |
| `float` | `f64`, `f32` | f64 preferred |
| `String.t()` | `String` | UTF-8, heap allocated |
| `binary` | `Binary` | Zero-copy read from Elixir |
| `binary` | `NewBinary` | Efficient write — allocate BEAM binary |
| `binary` | `Vec<u8>` | Copies data (use NewBinary instead) |
| `atom` | `Atom` | Interned strings |
| `list` | `Vec<T>` | Converted to Vec |
| `map` | `HashMap<K,V>` | Or struct with NifMap |
| `tuple` | `(A, B, ...)` | Or struct with NifTuple |
| `nil` | `()` | Unit type |
| `pid` | `LocalPid` | Process identifier |
| `reference` | `Reference` | Unique reference |

Full type-mapping with edge cases lives in [reference.md §4](reference.md).

### Derive Macros

```rust
use rustler::{NifStruct, NifMap, NifTuple, NifTaggedEnum, NifUnitEnum};

// Elixir: %MyApp.User{name: "Alice", age: 30}
//
// IMPORTANT: do NOT include the "Elixir." prefix in `#[module = "..."]`.
// Rustler prepends it automatically. Writing `#[module = "Elixir.MyApp.User"]`
// produces `__struct__: Elixir.Elixir.MyApp.User` and the Elixir caller
// gets a `%{}` map with that broken atom — pattern matches against
// `%MyApp.User{}` silently fail to bind.
//
// Asymmetry to remember: `rustler::init!("Elixir.MyApp.Native")` DOES need
// the prefix; the NifStruct attribute does NOT. The hook
// `nif-module-elixir-prefix` (in bb-anti-slop-patterns.json) catches the
// prefix mistake at write time.
#[derive(NifStruct)]
#[module = "MyApp.User"]
struct User {
    name: String,
    age: i32,
    email: Option<String>,  // Optional field
}

// Elixir: %{host: "localhost", port: 8080}
#[derive(NifMap)]
struct Config {
    host: String,
    port: i32,
}

// Elixir: {1, 2}
#[derive(NifTuple)]
struct Point(i32, i32);

// Elixir: :ok | {:error, String} | {:pending, %{id: 1}}
#[derive(NifTaggedEnum)]
enum Status {
    Ok,                      // :ok
    Error(String),           // {:error, "message"}
    Pending { id: i32 },     // {:pending, %{id: 1}}
}

// Elixir: :red | :green | :blue
#[derive(NifUnitEnum)]
enum Color {
    Red,
    Green,
    Blue,
}
```

### Atoms Macro
```rust
mod atoms {
    rustler::atoms! {
        ok,
        error,
        nil,
        not_found,
        invalid_input,
        lock_fail,
    }
}

// Usage
fn example() -> Atom {
    atoms::ok()
}
```

## 2.7 Scheduler Safety Rules

**The 1ms rule:** NIFs on the normal scheduler MUST complete in under 1 ms — longer NIFs cause scheduler collapse, process starvation, and an unresponsive system. Decision tree and threshold table live in §2.2 (they fire at the moment of writing).

```rust
#[rustler::nif]                              // normal — must be <1 ms
fn fast(x: i64) -> i64 { x * 2 }

#[rustler::nif(schedule = "DirtyCpu")]       // CPU-bound work
fn heavy(data: Vec<i64>) -> i64 { data.iter().map(|x| calc(*x)).sum() }

#[rustler::nif(schedule = "DirtyIo")]        // file, network, blocking syscalls
fn read_file(path: String) -> Result<Vec<u8>, String> {
    std::fs::read(&path).map_err(|e| e.to_string())
}
```

## 2.8 Error Handling Patterns

### Auto-conversion: `Result<T, E>` → `{:ok, T}` / `{:error, E}`

```rust
// Elixir: {:ok, 5.0} or {:error, "division by zero"}
#[rustler::nif]
fn safe_divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 { Err("division by zero".into()) } else { Ok(a / b) }
}

// Atom errors → {:ok, %User{}} or {:error, :not_found}
#[rustler::nif]
fn find_user(id: i32) -> Result<User, Atom> {
    lookup_user(id).ok_or(atoms::not_found())
}
```

### Error Propagation with thiserror
```rust
// Add to Cargo.toml: thiserror = "2.0"
use thiserror::Error;

#[derive(Error, Debug)]
enum NifError {
    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Lock contention")]
    LockFail,

    #[error("Not found: {0}")]
    NotFound(String),
}

// Convert custom errors to Rustler errors. Two variants — pick by intent:
//   - rustler::Error::Term(_)      → returns {:error, encoded} on Elixir side
//   - rustler::Error::RaiseTerm(_) → raises ErlangError with the term
//   - rustler::Error::RaiseAtom(_) → raises ErlangError with the atom
// For NIFs declared as `Result<T, NifError>` (with NifError: Encoder),
// use Error::Term — the function is "tuple-shape" and Elixir gets a tuple.
impl From<NifError> for rustler::Error {
    fn from(e: NifError) -> Self {
        rustler::Error::Term(Box::new(e.to_string()))
    }
}

#[rustler::nif]
fn complex_operation(input: String) -> Result<i64, NifError> {
    let value: i64 = input.parse()?;  // ? works with #[from]!

    if value < 0 {
        return Err(NifError::NotFound(input));
    }

    Ok(value * 2)
}
```

### `Error::Term` vs `Error::RaiseTerm` — the trap that bites NifResult

The skill says repeatedly "NifResult raises on Err" — true, but **only if the
`Err` value uses the `Raise*` variants**. The bare `Error::Term(Box::new(_))`
shown in some examples does NOT raise; it encodes as `{:error, term}` on the
Elixir side. This produces silently-wrong behavior in `!`-style NIFs:

```rust
// BAD: NifResult signature suggests "raises on err", but Error::Term
//      actually encodes as {:error, _}. Elixir's `assert_raise` won't
//      fire; `put!/3` returns a tuple instead of raising.
impl From<MyError> for rustler::Error {
    fn from(e: MyError) -> Self {
        rustler::Error::Term(Box::new(e.to_atom()))   // ← encodes, doesn't raise
    }
}
#[rustler::nif]
fn put(arr: ResourceArc<R>, i: usize, v: f64) -> NifResult<Atom> {
    array::put(&arr, i, v)?;   // ?-converts MyError → Error::Term — returns tuple!
    Ok(atoms::ok())
}
```

```rust
// GOOD: RaiseTerm actually raises. NifResult<Atom> + Ok(atoms::ok())
//       gives Elixir a clean :ok on success, ErlangError on failure.
impl From<MyError> for rustler::Error {
    fn from(e: MyError) -> Self {
        rustler::Error::RaiseTerm(Box::new(e.to_atom()))   // ← raises
    }
}
```

The variants in full:

| `rustler::Error` variant | Behavior on Elixir side |
|---|---|
| `Term(Box<dyn Encoder>)` | Returns `{:error, encoded_term}` — **does NOT raise** |
| `RaiseTerm(Box<dyn Encoder>)` | Raises `ErlangError` with the term as `:reason` |
| `RaiseAtom(&'static str)` | Raises `ErlangError` with the atom as `:reason` |
| `BadArg` | Raises `:badarg` |
| `Atom(&'static str)` | Returns `{:error, atom}` — same shape as `Term` |

**Rule of thumb:** if the NIF's return type is `NifResult<T>` AND you want
`assert_raise ErlangError` to actually fire on the Elixir side, the `From`
impl MUST use `RaiseTerm` or `RaiseAtom`. `Error::Term` is for the
"fall back to `Result<T, MyError>` returns tuple" path, not the
NifResult-raises path.

### Option Handling

Use `.ok_or(atoms::not_found())?` to convert `Option` to `Result` at NIF boundaries; use `.unwrap_or(default)` for safe defaults. Never `.unwrap()` on user-controlled data (Rule 6 / §2.3 #8).

## 2.9 OTP Integration — Env and LocalPid

### Sending Messages
```rust
#[rustler::nif]
fn notify<'a>(env: Env<'a>, pid: LocalPid, message: Term<'a>) -> Atom {
    match env.send(&pid, message) {
        Ok(()) => atoms::ok(),
        Err(_) => atoms::error(),
    }
}
```

### OwnedEnv for Async Operations

`OwnedEnv` allows sending messages from threads NOT managed by the BEAM.

```rust
use rustler::env::OwnedEnv;
use std::thread;

#[rustler::nif]
fn async_work<'a>(env: Env<'a>, data: String) -> Atom {
    let pid = env.pid();  // Capture caller's PID

    thread::spawn(move || {
        let result = expensive_computation(&data);

        let mut msg_env = OwnedEnv::new();
        let _ = msg_env.send_and_clear(&pid, |env| {
            (atoms::result(), result).encode(env)
        });
    });

    atoms::ok()  // Return immediately
}
```

**Critical:** `send_and_clear()` **PANICS** if called from a BEAM-managed thread. Only use from `std::thread::spawn`.

## 2.10 ResourceArc for State Management

For Rust state that persists across NIF calls. The minimal shape — `#[resource_impl]` + `try_lock` returning `:lock_fail`:

```rust
use rustler::ResourceArc;
use std::sync::Mutex;

struct Connection { data: Mutex<Vec<String>> }

#[rustler::resource_impl]
impl rustler::Resource for Connection {}

#[rustler::nif]
fn create_connection() -> ResourceArc<Connection> {
    ResourceArc::new(Connection { data: Mutex::new(Vec::new()) })
}

#[rustler::nif]
fn add_item(conn: ResourceArc<Connection>, item: String) -> Result<usize, Atom> {
    let mut data = conn.data.try_lock().ok().ok_or(atoms::lock_fail())?;
    data.push(item);
    Ok(data.len())  // meaningful value, not :ok
}
```

For richer ResourceArc patterns (NifStruct wrapper + Deref + clone-based mutation) see §2.4 Resource Wrapper Pattern.

### MutexGuard Lifetime Management

Drop guards before expensive work — see §2.3 #5 for the BAD/GOOD pair. Rule of thumb: scope the guard, clone-out the value you need, then do the work outside the scope.

### RwLock for Read-Heavy Workloads

```rust
use parking_lot::RwLock;

struct DataFrame {
    data: RwLock<Vec<Vec<f64>>>,
}

#[rustler::resource_impl]
impl rustler::Resource for DataFrame {}

#[rustler::nif]
fn read_column(df: ResourceArc<DataFrame>, col: usize) -> Result<Vec<f64>, Atom> {
    // Multiple NIFs can read simultaneously
    let guard = df.data.try_read().ok_or(atoms::lock_fail())?;
    guard.get(col).cloned().ok_or(atoms::not_found())
}

#[rustler::nif]
fn add_column(df: ResourceArc<DataFrame>, data: Vec<f64>) -> Result<usize, Atom> {
    // Write blocks all readers (exclusive access)
    let mut guard = df.data.try_write().ok_or(atoms::lock_fail())?;
    guard.push(data);
    Ok(guard.len())  // Return meaningful value, not {:ok, :ok}
}
```

### `Mutex<T>` for `!Sync` foreign handles (wasmex pattern)

Many foreign-library handles (wasmtime `Engine`/`Store`/`Instance`, native parser contexts, ML model handles) are `!Sync`. `Resource` requires `Send + Sync`, so wrap them in `Mutex<T>` (or `RwLock<T>`). When the Elixir-side discipline is "one GenServer owns this resource end-to-end", lock contention is logically zero — blocking `.lock()` is fine; surface only the poison case.

```rust
pub struct EngineResource(Mutex<wasmtime::Engine>);

#[rustler::resource_impl]
impl rustler::Resource for EngineResource {}

// Helper — write once per crate, every NIF becomes a one-liner
pub fn lock_resource<'a, T>(m: &'a Mutex<T>, name: &str) -> NifResult<std::sync::MutexGuard<'a, T>> {
    m.lock().map_err(|e| Error::Term(Box::new(format!("{name} lock poisoned: {e}"))))
}

#[rustler::nif(schedule = "DirtyCpu")]
fn engine_precompile_module(arc: ResourceArc<EngineResource>, b: Binary) -> NifResult<Binary> {
    let engine = lock_resource(&arc.0, "engine")?;
    let bytes = engine.precompile_module(b.as_slice())
        .map_err(|e| Error::Term(Box::new(format!("{e}"))))?;
    todo!("return bytes as a NewBinary — see §2.4 Binary Data")
}
```

Document the safety contract at the resource definition: `// SAFETY: lock contention is impossible by API contract — one GenServer owns each handle.`

## 2.11 Resource Monitoring

Monitor Elixir processes and react when they die:

```rust
use rustler::resource::Monitor;
use parking_lot::Mutex;

struct Session {
    data: Mutex<Vec<String>>,
}

#[rustler::resource_impl]
impl rustler::Resource for Session {
    // No manual IMPLEMENTS_DOWN needed — the macro detects `down` and sets it automatically

    fn down<'a>(&'a self, _env: Env<'a>, pid: LocalPid, _mon: Monitor) {
        println!("Process {:?} died, cleaning up...", pid);
        if let Ok(mut data) = self.data.lock() {
            data.clear();
        }
    }
}

#[rustler::nif]
fn monitor_caller(env: Env, session: ResourceArc<Session>) -> bool {
    let pid = env.pid();
    env.monitor(&session, &pid).is_some()
}
```

## 2.12 Testing NIFs

Per Rule 8, split pure logic from NIF wrappers. Test the pure logic with `cargo test` (`#[cfg(test)] mod tests` blocks); test the NIF integration with ExUnit:

```elixir
defmodule MyNifTest do
  use ExUnit.Case

  test "add returns correct sum" do
    assert MyApp.Native.add(2, 3) == 5
  end

  test "async operation sends message" do
    MyApp.Native.async_work("test")
    assert_receive {:result, _}, 5000
  end
end
```

For benchmarks, use `Benchee` with `MIX_ENV=prod mix compile` (debug Rust skews results 5–50×).

## 2.13 Precompiled NIFs

Distribute without requiring Rust on user machines:

```elixir
# mix.exs
defp deps do
  [
    {:rustler, "~> 0.37.2", runtime: false},
    {:rustler_precompiled, "~> 0.8"}
  ]
end
```

```elixir
defmodule MyApp.Native do
  version = Mix.Project.config()[:version]

  use RustlerPrecompiled,
    otp_app: :my_app,
    crate: "my_nif",
    base_url: "https://github.com/user/repo/releases/download/v#{version}",
    version: version,
    targets: ~w(
      aarch64-apple-darwin
      x86_64-apple-darwin
      x86_64-unknown-linux-gnu
      x86_64-unknown-linux-musl
      aarch64-unknown-linux-gnu
      x86_64-pc-windows-msvc
    )

  def add(_a, _b), do: :erlang.nif_error(:nif_not_loaded)
end
```

CI workflow for building precompiled artifacts: see [advanced.md §8](advanced.md).

### `mode: :release` pin for numerically-sensitive NIFs (candlex pattern)

For NIFs whose Rust kernels would behave differently in `:debug` mode (typically: integer-overflow checks panicking on PRNGs, hash mixers, fixed-point math, or `unsafe` paths that rely on undefined-behavior assumptions), pin `mode: :release` in the precompiled stub so local `mix compile` matches the published artifact:

```elixir
defmodule Candlex.Native do
  use RustlerPrecompiled,
    otp_app: :candlex,
    crate: "candlex",
    # Pin :release because debug-mode integer overflow checks make
    # the PRNG kernel panic on intentional wraparound multiplications.
    mode: :release,
    base_url: "...",
    version: version,
    targets: ~w(...)
end
```

Without the pin, `MIX_ENV=dev mix compile` builds debug Rust and you get test failures or runtime panics that don't reproduce in CI/prod (which uses precompiled `:release` artifacts). Document the *reason* for the pin in a comment so future maintainers don't remove it.

## 2.14 Rust Fundamentals for NIFs (NIF-specific only)

For general Rust, load `rust-implementing`. The NIF-specific essentials:

**Lifetime `'a` is tied to the Env (NIF call duration).** `Env<'a>`, `Term<'a>`, `Binary<'a>` all share this lifetime. To send terms past the NIF return (e.g. from a spawned thread), capture into `OwnedEnv` + `SavedTerm` first (see [TermBox pattern in patterns.md](patterns.md)).

**Spawned thread closures need `move` + `'static` bounds.** `std::thread::spawn` requires `F: FnOnce() -> T + Send + 'static`. Capture by `move`, and don't capture references with shorter lifetimes than `'static`.

**Input string-type selection** (the four are interchangeable at the type level but differ on copy cost):

| Input from Elixir | Rust parameter | Cost |
|---|---|---|
| Read-only text, ≤NIF-call lifetime | `&str` | Zero-copy view |
| Owned text needed past the NIF call | `String` | Copies |
| Read-only bytes, zero-copy | `Binary<'a>` | Zero-copy view (refc-binary) |
| Owned bytes needed past the NIF call | `data.as_slice().to_vec()` from `Binary` | Copies into `Vec<u8>` |

For binary **outputs**, use `NewBinary` / `OwnedBinary::release(env)` — see Rule 18 and §2.4. `Vec<u8>` is only for intermediate Rust-side buffers.

**Resources require `Send + Sync`.** Wrap `!Sync` foreign handles in `Mutex<T>` / `RwLock<T>` (see §2.10). Prefer `std::sync::Mutex` over `parking_lot` unless benchmarks show contention (Rule 10).

**Smart pointers used in NIFs:** `Arc<T>` (`ResourceArc` uses this internally), `Mutex<T>` (exclusive access), `RwLock<T>` (multi-reader), `Box<T>` (single-owner heap).

## 2.15 Advanced Patterns Index

> **Full implementations:** See [examples.md](examples.md) for complete working examples

| Pattern | Purpose | examples.md |
|---------|---------|-------------|
| **Builder Pattern** | Complex config with fluent API and validation | §35 |
| **State Pattern** | Type-state or enum-based state machines | §36 |
| **Default Trait** | Sensible defaults with `unwrap_or_default()` | §41 |
| **VecDeque Ring Buffer** | Circular buffers for sliding windows | §22 |
| **Slab Handle Pool** | O(1) handle allocation/lookup (Discord pattern) | §23 |
| **Custom Iterators** | Chunked data streaming from NIF resources | §27 |
| **Custom Error Types** | thiserror-based errors with atom conversion | §32 |
| **Drop Trait** | Resource cleanup on GC | §13 |
| **RAII Guards** | Automatic transaction rollback on error/panic | §37 |
| **Rayon Parallelism** | `par_iter()`, `par_chunks()`, work-stealing | §16 |
| **OnceCell/Lazy** | Global lazy statics for expensive init | §21 |
| **Bitflags** | C-compatible flag operations | §26 |
| **Binary Protocol** | byteorder for endianness-aware parsing | §24 |
| **Compression** | gzip/zstd with flate2 crate | §25, §40 |
| **Atomic Operations** | Lock-free counters, compare-and-swap | §38 |
| **Newtype Pattern** | Domain type wrappers preventing argument swapping | §34 |
| **Provider Abstraction** | Testable HTTP clients with injectable base URLs | §29 |
| **Testable NIFs** | Split NIF wrapper from internal logic | §30 |
| **JSON Pointer** | Nested JSON extraction | §31 |
| **with_context** | Rich error messages with anyhow context chains | §32 |
| **Time Measurement** | NIF profiling with AtomicU64 metrics tracking | §33 |
| **Round-Trip Testing** | Verify encode/decode consistency | §40 |
| **Network/I/O Patterns** | Tokio runtime integration, blocking fetch | §39 |

**Quick Reference - Atomic Ordering:**

| Ordering | Use Case |
|----------|----------|
| `Relaxed` | Counters, statistics (no synchronization) |
| `Acquire` | Reading shared data (pairs with Release) |
| `Release` | Writing shared data (pairs with Acquire) |
| `SeqCst` | When uncertain, or for CAS operations |

---

# PART 3 — REVIEWING PHASE

Auditing existing NIFs for the bugs that survive into PRs. The same anti-patterns from Part 2 are presented here as **"flag if you see this"** — the framing is "what to look for in someone else's diff," not "what to avoid in your own writing."

## 3.1 Review checklist — flag if you see this

Severity legend: **block** = do not merge; **request-change** = needs revision before merge; **suggest** = optional improvement.

> **Don't downgrade severity based on hex.pm popularity.** Popular published Rustler libraries ship Rule violations: resvg_nif (a downloads-heavy SVG renderer) lacks `DirtyCpu` annotations on multi-millisecond render NIFs and surfaces `Vec<u8>` returns into its public spec as `@type png_buffer :: [0..255]`. If a published library's pattern conflicts with a block-severity rule here, treat the library as the bug, not the rule.

| Pattern in the diff | Severity | Why | Fix → see |
|---|---|---|---|
| Generic `Result<Atom, MyError>` (with user `Encoder`) returning `Ok(atoms::ok())` | block | Produces `{:ok, :ok}` in Elixir; awkward callsites and silent breakage when refactored. **Note:** `NifResult<Atom>` (= `Result<Atom, rustler::Error>`) is special-cased and does NOT double-wrap — don't flag it. | Rule 2 (§2.1), return matrix (§2.2) |
| NIF declared as returning binary data but signature is `Vec<u8>` | block | Encodes as a *list of ints*, not a binary — silent contract violation | Rule 18 (§2.1), Binary template (§2.4) |
| `impl rustler::Resource for T {}` without `#[rustler::resource_impl]` | block | Runtime panic on first use (`called Option::unwrap() on a None value`) | Rule 20 (§2.1) |
| `.lock().unwrap()` on a Mutex inside a normal-scheduler NIF, or on shared cross-call state where contention is plausible | block | Can deadlock the scheduler; `.unwrap()` panics on poison → BEAM crash | Rule 4 + 6 (§2.1), §2.10 |
| `.lock().unwrap()` on per-call scratch state inside a freshly-constructed struct on DirtyCpu (mdex `LumisAdapter`/`PlaceholderRenderer` shape) | suggest only | The call owns the only handle — no contender to deadlock against. Flag if the resource is actually shared cross-call. | Rule 4 (§2.1) |
| `unwrap()` / `expect()` on fallible operation in NIF body | block | Panic crashes the entire BEAM VM | Rule 6 (§2.1), §2.8 Option Handling |
| MutexGuard held across long computation (no scoped block) | request-change | Blocks all other NIFs touching this resource for the full duration | Rule 13 (§2.1), §2.10 MutexGuard Lifetime |
| `tokio::spawn` / `quinn::Endpoint::new()` / mDNS construct without `runtime.enter()` first | block | Panics with "there is no reactor running" | Rule 19 (§2.1), §2.4 Tokio template |
| Calling upstream library that may panic and you want a custom error variant or panic-time cleanup, but no `catch_unwind` / `nif_safe` | request-change | Rustler catches panics at the boundary by default and converts them to error terms — flag this only if a typed `Panic` variant or cleanup is genuinely needed (e.g., FFI into C/C++, or telemetry on panic) | §2.4 catch_unwind / nif_safe templates |
| `send_and_clear()` called inside a `#[rustler::nif]` body (not from `std::thread::spawn`) | block | Panics — env is BEAM-managed, not OwnedEnv-compatible | Rule 14 (§2.1), §2.9 |
| NIF body > 1ms work with no `schedule = "DirtyCpu"` / `"DirtyIo"` annotation | block | Blocks the BEAM scheduler; can collapse all schedulers under load | Rule 1 (§2.1), §2.7 |
| `NifMap` struct with `Option<T>` fields, no Elixir-side fill-nil wrapper | request-change | `ArgumentError` when caller omits keys | Rule 15 (§2.1), §3.2 |
| NIF stub module without `@spec` on bound functions | request-change | Loses Dialyzer at the most type-fragile boundary in the system | Rule 17 (§2.1), §3.2 |
| `panic = "abort"` set in `[profile.release]` for a NIF crate | block | `catch_unwind` requires unwinding panics; abort defeats it; also breaks Rustler's boundary catch | §2.5 Cargo.toml |
| Missing `lto = true` / `lto = "thin"` in `[profile.release]` | suggest only if NIF is hot-path/CPU-heavy | Most surveyed crates (franz, y_ex, html5ever_elixir, wasmex) omit `[profile.release]` entirely; mdex/nimble_lz4 set `lto = true`. Don't blanket-flag; ask what the workload is | §2.5 Cargo.toml |
| `mem::transmute<...<'a>, ...<'static>>` for storing borrowed foreign handles in a `ResourceArc` *without* a companion `ResourceArc` to the owner in the same Elixir-visible struct | block | Use-after-free risk: BEAM GC may drop the owner before the borrower. y_ex's pattern is safe ONLY because `NifTxn { doc: ResourceArc<DocResource>, ... }` carries the doc alongside the txn | §2.4 mem::transmute caution |
| Unbounded user-controlled allocation: `Pixmap::new(w, h).unwrap()`, `Vec::with_capacity(user_size)`, `read_to_end(&mut buf)` on a user-supplied stream, etc. | block | DoS surface — attacker controls allocation size with no upper bound. Even `.unwrap()` on a fallible allocator (which returns `Option`/`Result` precisely because alloc can fail) is a BEAM crash. Add a budget check + typed `:input_too_large` atom error | §2.4 Image NIF size validation |
| `rust-libp2p`/quinn/mDNS NIF with a global `Runtime::new()` not wrapped in `OnceLock` | request-change | Multiple runtimes can be created on retry; resource leak | §2.4 Tokio template |
| Fresh NIF crate adds `parking_lot` as a dep with no stated reason | suggest | `std::sync::Mutex` is the lighter default; mdex's use is fine but their reasons (existing dep, unpoisoned locks, measured contention) should be documented in PRs | Rule 10 (§2.1) |
| `String` parameters where `&str` or `Binary` would do | suggest | Forces extra allocation on every NIF call | §2.14 String Types |
| Native module placement that doesn't match the library's shape (e.g., big framework lib using stub-as-public; single-purpose lib using deep nesting) | suggest | Three valid shapes (§1.3 Step 4); flag mismatches between shape and library size/architecture rather than blanket-flagging "not nested" | §1.3 Step 4 |
| Dirty NIF returning `Vec<u8>` from a compression/encoding routine | block | Same `Vec<u8>` trap as above | Rule 18 (§2.1), §2.4 Binary Data |
| `bare impl Resource for T` with manual `IMPLEMENTS_DOWN: bool = true` | request-change | Manual booleans are fragile — let `#[rustler::resource_impl]` detect callbacks | Rule 20 (§2.1) |
| New NIF added to Rust source but not registered in `rustler::init!(...)` | block | Compiles cleanly; NIF stub raises `:nif_not_loaded` at runtime | §2.5 "Adding a New NIF Function" |
| Forgotten `runtime.enter()` for *additional* NIFs constructing tokio resources after the first one | block | Easy to miss — every NIF that constructs a tokio resource needs the guard | Rule 19 (§2.1), §2.4 Tokio template |

## 3.2 Anti-pattern catalog — review framing

Most patterns flagged in review are the same ones in §2.3 — see those code pairs for the fix. Use this catalog as a reviewer checklist; the right column is the suggested-change snippet for the PR comment.

| Spot in diff | Suggest |
|---|---|
| `Result<Atom, _>` returning `Ok(atoms::ok())` (generic Result, not `NifResult`) | Switch to `NifResult<Atom>`, or change the success type to `bool`/`usize`/etc. See §2.3 #1. |
| `Vec<u8>` return type for binary data | `NewBinary` (or `OwnedBinary::release(env)`). See §2.3 #2. |
| `impl rustler::Resource for T {}` without `#[rustler::resource_impl]` | Add `#[rustler::resource_impl]`. See §2.3 #3. |
| `.lock().unwrap()` in a normal-scheduler NIF | `try_lock().ok_or(atoms::lock_fail())?`. See §2.3 #4 / Rule 4 for dirty-scheduler exceptions. |
| `MutexGuard` held across an expensive call | Clone-out and drop the guard before the work. See §2.3 #5. |
| Constructing a tokio resource without runtime context | `let _g = runtime.enter();` first. See §2.3 #6 / Rule 19. |
| Unwrapped library call known to panic on bad input | Wrap in `nif_safe { ... }`. See §2.3 #7. |
| `unwrap()` / `expect()` on user-controlled input | `?` with `ok_or(...)` mapping to your error type. See §2.3 #8. |
| Default scheduler doing >1ms work | `#[rustler::nif(schedule = "DirtyCpu" \| "DirtyIo")]`. See §2.3 #9. |
| `OwnedEnv::send_and_clear` from a NIF body (BEAM-managed thread) | Use `env.send(&pid, msg)` from a NIF; reserve `send_and_clear` for `std::thread::spawn` / runtime tasks. See §2.3 #10. |
| `String` input where `Binary<'a>` would be zero-copy | `Binary<'a>` (read-only) or `data.to_vec()` (owned). See §2.14 input table. |

Two anti-patterns are review-only (they don't appear in §2.3 because they live on the Elixir side):

### NifMap requires ALL keys present

```rust
#[derive(NifMap)]
struct Opts { color_mode: Option<ColorMode>, filter_speckle: Option<i32> }
```
```elixir
# BAD: missing keys raise ArgumentError even though Rust fields are Option<T>
MyNif.call(%{color_mode: :binary})

# GOOD: wrap the NIF to fill missing keys with nil
@all_keys [:color_mode, :filter_speckle]
def call(opts \\ %{}) do
  full_opts = Map.merge(Map.new(@all_keys, &{&1, nil}), opts)
  nif_call(full_opts)
end
```

### Missing `@spec` on NIF bindings

NIF stubs are the boundary between two type systems — they are the most important functions to type. Per Rule 17, put `@spec` on the public-facing wrapper (or on the stub itself if stub-as-public).

```elixir
# BAD: no @spec — caller has no idea what types the NIF expects/returns
def compute(_matrix, _window, _points), do: :erlang.nif_error(:nif_not_loaded)

# GOOD: @spec on every NIF, shared @type for recurring patterns
@type point :: {float(), float()}
@spec compute([[float()]], non_neg_integer(), [point()]) :: {:ok, map()} | {:error, String.t()}
def compute(_matrix, _window, _points), do: :erlang.nif_error(:nif_not_loaded)
```

Dialyzer catches NIF-boundary type mismatches before runtime; editor hovers show parameter types — critical when the Rust side expects specific numeric types. Define `@type` aliases when multiple NIFs share complex argument shapes.

## 3.3 When the NIF Kills the VM — Debug Playbook

When a NIF takes down the whole BEAM (no `EXIT` report, just the erl crash dump), standard Elixir debugging doesn't apply — the VM is gone. Reach for these in order:

**1. Check if it's actually a NIF crash.** `cat erl_crash.dump | head -30` — look at `Slogan:`. A NIF-triggered crash typically says "Received SIGSEGV / SIGBUS" or mentions a Rust symbol. If the slogan is about mailbox overflow, scheduler collapse, or exceeded atom limit, it's an Elixir-level problem; don't blame the NIF.

**2. Rebuild with debuginfo + backtrace.**
```bash
# Cargo.toml: set debug = 2 in [profile.release] (see §2.5 step 3)
# Then rebuild and rerun with backtraces enabled:
RUST_BACKTRACE=full MIX_ENV=prod mix compile
RUST_BACKTRACE=full iex -S mix
```
On the next crash you'll see the Rust stack in the erl_crash.dump or stderr.

**3. Run under a debugger.** `gdb --args beam.smp -- -mode interactive -env ELIXIR_ERL_OPTIONS "+K true"` (adjust to your release's start command). When the NIF crashes, `bt full` gives you the Rust stack with symbols if you kept `debug = 2`.

**4. Narrow the trigger.** Instrument the Elixir caller to log every input before the NIF call (`Logger.warning("before NIF", args: inspect(args))`). Reproduce. The last-logged args were passed to the NIF that crashed — that's your input.

**5. Common NIF-kill causes, by likelihood:**

| Symptom | Likely cause | Fix |
|---|---|---|
| SIGSEGV on a specific input | Undefined behavior in `unsafe` block; out-of-bounds `from_raw_parts`; misuse of `Pin` | Convert `unsafe` to safe abstractions; audit any `unchecked_*` calls |
| SIGSEGV with upstream library | Library bug triggered by malformed input | Wrap in `catch_unwind` (§2.4 `nif_safe`), validate input in Rust before calling library |
| Abort on double-panic (panic during unwinding) | Your `Drop` impl panics; your `Encoder::encode` panics | Make Drop impls infallible; propagate errors from encoders via `Result` |
| "resource type not registered" → unwrap None at runtime | Rule 20 missing `#[rustler::resource_impl]` | Add the attribute, not `impl Resource for T {}` bare |
| "there is no reactor running" | Tokio crate constructed outside `runtime.enter()` | See §2.4 Tokio runtime-enter pattern |
| Memory grows unbounded until OOM | Rust-side leak: `Vec` retained in ResourceArc that's never released; circular `Arc` | Use `:recon.bin_leak/1` then audit Rust ownership |

**6. Gate experimental NIFs behind a feature flag or separate OTP app.** If you suspect a NIF is flaky, run it in a separate BEAM process via a port or a dedicated node. When it crashes, only that node dies. See §1.2 NIF crate split.

**7. Production telemetry for NIF health:** wrap every NIF call in an Elixir `:telemetry.span/3` so crashes leave a trail even if the VM dies right after. In a rolling deploy, the telemetry from the previous minute is often the only clue.

---

## Navigation

- **[reference.md](reference.md)** — Cargo.toml templates, complete type mapping tables, Env/OwnedEnv/Resource API reference, derive macro quick reference, lock patterns, thread patterns, data structure selection guide
- **[examples.md](examples.md)** — 43 complete working examples from basic arithmetic to production patterns (sorted sets, monitoring, streaming iterators, Tokio integration, compression, state machines)
- **[patterns.md](patterns.md)** — 17 specialized production templates extracted from §2.4: foreign callbacks (scoped_thread_local!, TermBox), binary encoding (SliceIntoBinary, make_subbinary), Decoder/Encoder for atom-tagged enums, resource lifecycle (mem::transmute, explicit-close), OS-specific (thread_local!, CFRunLoop), crypto fixed-size parsing, image allocation guards, ML/tensor backend wrapper, WASM cancel-handle, static caching, build (load callback, cdylib plugin workspace), error macros, Tokio runtime variants
- **[advanced.md](advanced.md)** — FFI fundamentals (CString, repr(C), raw pointers, linking), advanced threading (closure traits, scoped threads, channel pipelines, thread builders), module organization for large NIFs, string optimization, manual Term encoding (Encoder trait, keyword lists, dynamic maps), CI/CD for precompiled NIFs (GitHub Actions workflow)

## Related Skills

- **[rust-implementing](../rust-implementing/SKILL.md)** — General Rust idioms (ownership, traits, async/await, error handling, macros) at the moment of writing. Key: use `thiserror` for library errors, `anyhow` for application errors; prefer `impl Into<String>` over `String` parameters for ergonomic APIs.
- **[rust-planning](../rust-planning/SKILL.md)** — General Rust architectural planning (project layout, crate boundaries, error strategy, async strategy). Key: design dependencies inward, traits as ports, composition root in `main()`.
- **[rust-reviewing](../rust-reviewing/SKILL.md)** — General Rust code inspection (review, debug, profile). Key: classify severity, suggest the idiomatic refactor, prefer type-system fixes over runtime checks.
- **[elixir-implementing](../elixir-implementing/SKILL.md)** — Elixir at the moment of writing. Key: prefer `with` chains for multi-step operations, use ok/error tuples not exceptions for expected failures. **Integration:** When your NIF library uses Registry-based process management, callers MUST use your `via()` helper to address processes — raw atom names won't resolve. Document your library's process discovery mechanism.
- **[libp2p](../libp2p/SKILL.md)** — rust-libp2p wrapped via NIFs. Builds on §2.4 Tokio runtime-enter pattern with full Swarm/NetworkBehaviour integration.

## Consuming NIF Libraries in Phoenix/Elixir Apps

When a NIF library has its own OTP supervision tree (Registry, DynamicSupervisor, GenServers):

1. **Application ordering** — The NIF library's application must start before your processes. Ensure it's in `deps` (automatic) or `extra_applications` (manual).
2. **Process naming** — If the library registers processes via `{:via, Registry, ...}`, you MUST use the same via tuple to call them. Use the library's `via()` helper — never raw atom names.
3. **Initialization dependencies** — If your GenServer's `handle_continue` calls into the library, use `:rest_for_one` supervision or ensure the library's tree is fully started.

## Scope — what this skill does NOT cover

- General Rust programming (ownership patterns beyond NIF-relevant ones, async design, macro systems, trait coherence) → load `rust-implementing` (or `rust-planning` / `rust-reviewing` for the matching phase).
- Cross-language protocol design (wire format, schema evolution, versioning between BEAM and non-BEAM services) → design it in `elixir-planning` first, then apply here.
