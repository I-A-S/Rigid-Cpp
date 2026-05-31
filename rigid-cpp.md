# Rigid C++: A Pragmatic Architecture for High-Performance Systems

> [!NOTE]
> This manifesto is a practical TLDR for engineers; it aims to be concise and on-point.
>
> For the theory and deep rationale, read the Rigid C++ academic paper (to be published on June 5th 2026).

**Rigid C++** is not a new language. It is a disciplined **architectural subset of C++23** that aggressively adopts C++23's compile-time toolkit while enforcing **runtime rigidity and predictability**.

It targets the hot path: engines, runtimes, packet processors, and the parts of an application where a single cache miss costs revenue. Business logic, configuration tooling, and developer scripts have no reason to pay the Rigid C++ rigor tax.

**Rigid C++** rests on Four pillars:

1. **Strict Runtime, Fluid Compile-Time**
2. **The Two-Tier Error Protocol**
3. **Total Memory Sovereignty**
4. **The Pragmatic STL Pact**

## 1. Strict Runtime, Fluid Compile-Time

Reject Modern C++'s **runtime** costs; aggressively adopt its **compile-time** features.

- **No exceptions.** Ban exceptions<sup>1</sup>, use `Result<T, E>` (See Pillar 2).

- **No RTTI.** `dynamic_cast` and runtime type discrimination are out. Virtual functions remain fine for boundary polymorphism (plugins, test doubles); vtables aren't RTTI.

- **Utilize Modules and Concepts.** Use C++20 Named Modules, Concepts, and C++23's `import std`.

- **Avoid Deep TMP.** Prefer `if constexpr` over SFINAE and `std::enable_if`. Avoid deep template metaprogramming hacks.

<sup>1</sup> `-fno-exceptions` on Clang/GCC, `/EHs-c-` and `/D_HAS_EXCEPTIONS=0` on MSVC

## 2. The Two-Tier Error Protocol

Exceptions add invisible branching and ABI overhead (unwind tables). C-style error codes are easily ignored. Reject both.

All runtime anomalies fall into exactly two tiers: **Recoverable Errors** (Tier 1) and **Unrecoverable Panics** (Tier 2).

### 2.1 Tier 1: Recoverable Errors

Environmental practicalities: file missing, socket dropped, malformed input. Not bugs; expected states.

Handled via a strict `Result<T, E>`. C++23 `std::expected` interoperates, but Rigid C++ uses a custom `Result` for three reasons:

1. `.value()` on empty `expected` throws `std::bad_expected_access`; under `-fno-exceptions` that compiles to an unconditional terminate (exact choice of `std::terminate`, `abort`, or a stub varies by vendor), losing the failure site. `Result`, by contrast, routes unauthorized unwraps through `panic()` with `std::source_location`.

2. Implementations apply `[[nodiscard]]` inconsistently; some at the class level, others only on selected member functions. (Microsoft STL added class-level `[[nodiscard]]` in late 2024; libstdc++ followed in March 2026; libc++ applied it function-by-function in late 2025 but has explicitly declined to mark the class itself.) `Result` carries class-level `[[nodiscard]]` as a guaranteed contract.

3. Long monadic chains (`expected.and_then(...).or_else(...)`) tend to produce undebuggable template stacks and slow compile times.

Using `Result<T, E>` brings:

- **Enforced handling.** `Result<T, E>` is `[[nodiscard]]`; the compiler refuses silent drops.
- **Ergonomic propagation.** TRY macros (`AU_TRY`, `AU_TRY_VAR`) early-return on failure and bind the success value inline. A deliberate interim solution until C++ standardizes a postfix control-flow / error-propagation operator (cf. P2561).
- **Flat happy path.** Early-exit propagation keeps the main flow flush against the left margin, free of nested error-checking.

### 2.2 Tier 2: Unrecoverable Panics

Logic bugs: out-of-bounds access, null deref, broken invariant, process-wide OOM. State is corrupted by definition; do not unwind.

- **Immediate termination.** Call a central `panic()` handler that logs and traps (`__builtin_trap()`/`__debugbreak()` or `std::abort()`). The process is killed before corrupted state can propagate.

- **Zero-overhead diagnostics.** `std::source_location` captures file and line at compile time. No runtime stack trace, no RTTI.

## 3. Total Memory Sovereignty

For many hot-path workloads, memory bus is the bottleneck; not the CPU. Node-based containers and opaque global allocations are hostile to modern superscalar CPUs. Take total control of where memory lives and how it is accessed.

### 3.1 The Allocator Contract

Global `new`/`delete` are forbidden. Every allocating type takes an allocator that satisfies a compile-time `AllocatorType` concept.

- **Explicit routing.** Whether it is an arena, pool, or a general-purpose backend like rpmalloc, the allocator is always an explicit parameter of the system.

- **STL adapter bridging.** When composing with `std::vector` etc., bind them to a custom `StdAllocatorAdapter` so even standard implementations are pinned to the application's memory domains. The adapter must declare its propagation traits explicitly (`propagate_on_container_copy_assignment`, `…_move_assignment`, `…_swap`, `select_on_container_copy_construction`, `is_always_equal`); wrong defaults silently produce use-after-free across arena boundaries.

- **OOM has two flavors:**
  1. **Allocator-domain exhaustion** (a bounded arena, pool, or budget being exhausted): Tier 1 `Result<T, E>` → caller may shed load, evict, or retry against another domain. (Note: When using STL adapters, domain exhaustion degrades to a Tier 2 panic. See §4.4)
  2. **Process-wide OOM** (the host allocator itself fails): Tier 2 panic → trap immediately.

- **Why not `std::pmr`?** `std::pmr` adds a virtual `do_allocate`/`do_deallocate` dispatch per call. `AllocatorType` bakes the allocator type into the container at compile time; same routing flexibility, no vtable. `std::pmr` may be wrapped behind an `AllocatorType`-satisfying adapter at module boundaries, but it is not the default contract.

- **Hostile dependencies.** Third-party libs (zlib, OpenSSL, ICU, the glibc resolver) allocate via `malloc` by default and won't honor `AllocatorType`. The three remediations (in order of preference) are:
  1. **Library-provided hooks (where they exist).** `z_stream::zalloc`/`zfree` (per-stream), ICU's `u_setMemoryFunctions` (must precede `u_init`), OpenSSL's `CRYPTO_set_mem_functions` (must precede any allocation). Wrap each behind a shim that routes to an `AllocatorType` domain.
  2. **Process isolation.** Confine the dep to its own address space with a bounded heap. (Per-thread isolation only works if the process malloc is replaced with one that partitions per-thread.)
  3. **Documented unaudited domain.** If neither (1) nor (2) is feasible (system resolver, vendor blobs), record the dep's measured working-set ceiling and treat it as a budgeted line item.

The mandate is auditability of every allocation path, not the impossible claim that no `malloc` ever runs.

### 3.2 Data-Oriented Containers

The STL was designed before the cache latency dominated performance to the degree it does today. `std::unordered_map` and `std::list` force pointer-chasing across the heap. Rigid C++ demands DOD for all core data structures.

- **SIMD-probed, dense-iterating HashMap.** Combine SwissTable's group-scanned control bytes (8- or 16-slot SIMD probe over 7-bit fingerprints, quadratic probing with triangular numbers) with the Python 3.6 compact-dict layout (a dense `Vec<Entry>` plus a separate per-slot index). Most lookups resolve on the first probe group; iteration walks the dense array contiguously. Lowers to SSE2 on x86_64, NEON or SWAR on ARM64, WASM SIMD128 on the web, and a SWAR scalar fallback elsewhere.

- **SoA over AoS.** Parallel contiguous arrays let the hardware prefetcher fill cache lines with data the hot path actually touches.

- **Stack-first strings.** SSO is mandatory. The reference implementation (`String`) packs up to 23 bytes inline on 64-bit (libc++ defaults to 22, libstdc++ and MSVC to 15) and encodes the heap/inline flag in the low bit of the capacity field. Exact layout is your choice.

### 3.3 Cache Lines and False Sharing

Contended atomics (e.g., head/tail in an SPSC ring) must never share a cache line. Use `std::hardware_destructive_interference_size` where available; fall back to 64 bytes. Note that the hardware line is 128 on POWER9/10 and 256 on Fujitsu A64FX, but `std::hardware_destructive_interference_size` is implementation-defined and may report a smaller value for ABI stability; hardcode the platform-specific value if you need to match real cache geometry. The outcome is invariant: contended atomics physically separated.

## 4. The Pragmatic STL Pact

The STL is a component repository, not a religion. Compose with it where memory layout and error handling are sound; mercilessly replace where they aren't.

### 4.1 The Line of Demarcation

The filter: *does this component dictate an unacceptable memory layout or error-handling style?*

- **Compose or alias** when it doesn't:
  - Vocabulary types that don't own heap memory: `std::span`, `std::string_view`.
  - OS abstractions: `std::mutex`, `std::thread`, `std::filesystem` (only the `std::error_code` overloads; throwing overloads are banned).
  - Interoperate cleanly with their standard counterparts (`Result` ↔ `std::expected`, `Option` ↔ `std::optional`). At the boundary, an empty `expected` becoming a `Result<T, E>` must route a later unauthorized unwrap through `panic()`, not `std::terminate`.
- **Adapter-bridge** structurally sound containers that default to the global heap. `Vec<T>` aliases `std::vector` bound to `StdAllocatorAdapter` (see §3.1 for the propagation-trait contract).

### 4.2 Use `<algorithm>`

Don't rewrite `sort`, `find`, or `binary_search`; use `std::ranges`. All custom containers (`String`, `CompactVec`..) must model C++20 `std::ranges::contiguous_range` and `std::contiguous_iterator` so they pick up the full algorithm set, including C++23's expanded `ranges`, with zero abstraction penalty.

### 4.3 Surgical Replacement

When a STL component violates Rigid C++ mandates, ban and replace:

- **Cache poison.** `std::unordered_map` → SwissTable-derived `HashMap` (see §3.2). `std::list` → versioned `{index, generation}` handles in contiguous storage (slot-map pattern); trades C++ reference stability for handle stability.

- **Allocator bypass.** `std::shared_ptr` → `Arc<T>` or intrusive pointers that route through `AllocatorType`. `shared_ptr` carries weak-ref overhead even with `make_shared`; also, `make_shared` co-allocates object and control block, so the memory backing the (already-destroyed) object stays allocated until the last `weak_ptr` dies. `allocate_shared` is too unergonomic. (Note: Favour `Box<T>` over `Arc<T>` whenever possible)

- **ABI and I/O bloat.** `<iostream>` is permanently banned (static-init overhead, binary bloat..). Use C++20 `std::format` and C++23 `std::print`; combined they provide printf-comparable speed with full compile-time type safety. No regression to `printf`.

### 4.4 Throw Sites Become Panic Sites

Several adopted STL APIs still throw (which under `-fno-exceptions` becomes an unconditional `abort`/`terminate` call): 

- `std::vector::push_back` → `bad_alloc`
- `std::mutex::lock` → `system_error` 
- `std::vformat` with runtime format strings → `format_error`
- `std::thread` ctor → `system_error`

Rigid C++ accepts this trade. By composing with these APIs, we surrender the Tier 1 return path they never offered: `push_back`'s allocation failure is reclassified as Tier 2 at the boundary, even though the same condition through a Rigid-native API would be Tier 1. Rigid C++ classifies OS-primitive resource exhaustion alongside process-wide OOM in Tier 2: by the time the kernel refuses, the typical caller has no recovery path. Malformed runtime format strings are programmer errors and intrinsically Tier 2.

## Non-Goals

- **Not a kernel discipline.** Assumes a hosted runtime (libc, OS threading, filesystem). Freestanding/firmware needs further restrictions this manifesto does not address.

- **Not a borrow checker, not a memory-safety guarantee.** Killing exceptions and global `new` reduces failure modes; it does not eliminate use-after-free, data races, or buffer overruns. Run `AddressSanitizer`, `ThreadSanitizer`, and `UBSan` in CI.

- **Not about coroutines or structured concurrency.** C++20 coroutines and the C++26 sender/receiver `std::execution` model are out of scope for this revision. The principles still apply (coroutine frames route through `AllocatorType`, awaitable errors are `Result`-shaped, scheduler invariant violations are Tier 2 panics), but integration warrants a dedicated treatment.

## What Rigid C++ Does Not Claim

- **Rigid C++ does not claim to invent this territory:** EA's EASTL, Google's Abseil, and Chromium's `//base` are all production-scale "rigid" dialects that converged on overlapping instincts that Rigid C++ shares: custom allocators, contiguous and fixed-storage alternatives to node-based defaults, error-returning paths where possible, and hand-rolled vocabulary types.

- **Rigid C++ does not claim to be universally faster:** Banning exceptions in favor of `Result<T, E>` eliminates the catastrophic tail latencies of unwinding and frees the compiler from the optimization barriers that potentially-throwing calls impose. In exchange, happy path takes a hit (measurably so on MSVC, judging by our [benchmarks](https://github.com/I-A-S/Auxid/blob/main/benchmarks/results/summary.csv)); every call now carries a discriminant check cost that an unthrown exception never had to deal with. On modern table-based EH targets (the Itanium C++ ABI on GCC/Clang, and MSVC's table-based SEH on Windows x64, especially with `/d2FH4`) happy path is nearly free. Rigid C++ accepts that trade-off in exchange for deterministic worst-case latency.

## Auxid: The Reference Implementation

The principles above are implemented in **Auxid**, an open-source C++23 library shipped as a Named Module (`import auxid;`).

Auxid materializes each pillar:

- **Strict runtime, fluid compile-time.** Module-first surface, concept-driven overloads, built under `-fno-exceptions` (`/EHs-c-` with `/D_HAS_EXCEPTIONS=0` on MSVC) and `-fno-rtti` (`/GR-` on MSVC).
- **Two-tier errors.** A class-level `[[nodiscard]]` `Result<T, E>` whose unauthorized unwraps route through `panic()`, plus `AU_TRY` / `AU_TRY_VAR` propagation macros.
- **Total memory sovereignty.** A single `AllocatorType` concept threaded through every container; `StdAllocatorAdapter` bridges to the STL.
- **Pragmatic STL pact.** DOD-native primitives where the STL falls short: a SIMD-probed SwissTable `HashMap` over a dense entry array, a 23-byte SSO `String`, `CompactVec<T>`, and cache-aligned SPSC queues.

Source: [github.com/I-A-S/Auxid](https://github.com/I-A-S/Auxid).
