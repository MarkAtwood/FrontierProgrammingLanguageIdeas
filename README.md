# Frontier Programming Language Ideas

Explorations at the edge of programming language design. These are detailed specifications and proposals for what systems programming could look like with different tradeoffs—not production-ready implementations, but serious design work.

## Projects

### Zinc

**Zig extensions for safety and ergonomics.**

Zinc proposes six additions to Zig (v0.13):

| Feature | Description |
|---------|-------------|
| Ambient Allocators | Scoped allocator capabilities that eliminate explicit allocator threading |
| Design by Contract | `requires`/`ensures`/`invariant` clauses with runtime and comptime checking |
| Verified Binary Layout | `@layout` declarations with compile-time verification against C headers |
| Extended Numeric Types | `f16`, `bf16`, `f80`, `f128`, decimal floats, fixed-point, rationals |
| Numeric Comptime Interfaces | `Arithmetic`, `Ordered`, `Bounded` for generic numeric code |
| Concurrency Stdlib | Channels, thread pools, and structured concurrency primitives |

**Files:**
- `zinc-spec.md` — Full language specification (v0.3)
- `zig-plan.md` — Strategy for upstreaming features to Zig

### Ferrum

**A new systems language with ownership, effects, and proofs.**

Ferrum explores what Rust might look like with region inference, an effect system, and SMT-backed verification:

| Feature | Description |
|---------|-------------|
| Ownership + Regions | Move semantics with inferred region annotations (no lifetime syntax) |
| Effect System | Tracked effects like `! IO + Net` with effect polymorphism |
| Capabilities | First-class capability tokens for ambient authority control |
| Design by Contract | `requires`/`ensures`/`invariant` with proof obligations |
| Proof Functions | `proof fn` blocks discharged by Z3 SMT solver |
| Binary Layout | `@layout` declarations verified against external headers |
| Graduated Safety | Five levels: `default` → `unchecked` → `trusted` → `extern` → `unsafe` |
| Extended Numerics | IEEE 754 extended, decimal floats, fixed-point, arbitrary precision |

**Language Reference** (split for manageability):
- `ferrum-language-reference.md` — Index and quick reference
- `ferrum-lang-core.md` — Design philosophy, lexical structure, keywords
- `ferrum-lang-types.md` — Type system, numerics, constrained types
- `ferrum-lang-ownership.md` — Ownership, regions, effects, capabilities
- `ferrum-lang-generics.md` — Allocators, generics, traits, GADTs
- `ferrum-lang-syntax.md` — Expressions, patterns, functions, type declarations
- `ferrum-lang-concurrency.md` — Tasks, channels, synchronization
- `ferrum-lang-verification.md` — Contracts, proofs, binary layout, safety levels
- `ferrum-lang-modules.md` — Module system, packages
- `ferrum-lang-internals.md` — Compiler algorithms, grammar

**Standard Library:**
- `ferrum-stdlib.md` — Standard library index and design principles
- `ferrum-stdlib-core.md` — core module (ops, cmp, option, result, iter, mem, ptr, slice, str, ffi, cell, pin, error, NonZero)
- `ferrum-stdlib-alloc.md` — alloc + collections modules (Vec, String, HashMap, CString, OsString, Cow, Rc/Weak)
- `ferrum-stdlib-io.md` — io, binary, text, fs modules
- `ferrum-stdlib-async-net.md` — async, net, http, poll modules
- `ferrum-stdlib-sys.md` — sys, posix, windows (cross-platform, threading APIs)
- `ferrum-stdlib-platform.md` — Platform abstraction layer (POSIX, Linux, BSD, Windows, WASI, Zephyr)
- `ferrum-stdlib-math.md` — math, linalg, simd modules
- `ferrum-stdlib-numeric.md` — BigInt, BigDecimal, Rational, Complex (arbitrary precision)
- `ferrum-stdlib-util.md` — fmt, time, sync, process, env, backtrace, log modules
- `ferrum-stdlib-crypto-testing.md` — crypto, testing modules, C mistakes reference
- `ferrum-plan.md` — 34-phase implementation plan
- `ferrum-fears-and-hopes.md` — Design philosophy and failure mode analysis

### Compiler Agent API

**Exposing compiler internals for AI-assisted programming.**

A proposal for turning compilers from batch processors into interactive reasoning engines that AI agents can query, probe, and instrument.

| Capability | Description |
|------------|-------------|
| Query | Ask the compiler what it knows at any program point (types, aliasing, lifetimes, optimization blockers) |
| Hypothesize | Inject proposed invariants and ask if they close verification gaps—without modifying source |
| Trap | Place semantic monitors that the compiler weaves into binaries because it understands the type system |
| Instrument | Extract contracts and type invariants as executable test scaffolding |

Includes implementation paths for Clang/LLVM, Rust, Python, and GCC.

**Files:**
- `compiler-agent-api.md` — Full proposal with per-toolchain analysis

## Design Philosophy

These projects share some principles:

- **Explicit over implicit** — but with inference where the compiler can prove correctness
- **Zero-cost abstractions** — safety features that compile away when verified
- **Graduated escape hatches** — multiple safety levels rather than a single `unsafe` keyword
- **Contracts as code** — preconditions, postconditions, and invariants are first-class
- **Allocator awareness** — memory allocation is never hidden from the programmer

## Status

These are design documents and thought experiments. No compilers exist yet. Ferrum has an implementation plan targeting a Rust bootstrap compiler; Zinc is designed for gradual upstream contribution to Zig.

