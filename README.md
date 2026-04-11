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
| Effect System | Tracked effects like `! IO + Net`, inferred within modules, required at `pub` boundaries |
| Capabilities | First-class capability tokens for ambient authority control |
| Design by Contract | `requires`/`ensures`/`invariant` with proof obligations |
| Proof Functions | `proof fn` blocks discharged by Z3 SMT solver |
| Binary Layout | `@layout` declarations verified against external headers |
| Graduated Safety | Four levels: `unchecked` → `trusted` → `extern` → `unsafe` |
| Extended Numerics | IEEE 754 extended, decimal floats, fixed-point, arbitrary precision |
| Allocator Defaults | `Heap` is default; custom allocators are opt-in via `given [A: Allocator]` |

**Start here:**
- `ferrum.md` — Overview and documentation index

**Introductions:**
- `ferrum-introduction-for-programmers.md` — **For C/Python programmers** — Language basics, syntax, types
- `ferrum-introduction-for-rust-programmers.md` — **For Rust programmers** — What's different, what's new
- `ferrum-introduction-to-ownership.md` — Ownership, moving, borrowing
- `ferrum-introduction-to-option-result.md` — Option, Result, the `?` operator
- `ferrum-introduction-to-traits.md` — Traits, impl blocks, bounds
- `ferrum-introduction-to-enums.md` — Enums with data, pattern matching
- `ferrum-introduction-to-generics.md` — Generic types and functions
- `ferrum-introduction-to-iterators.md` — Iterators, closures, lazy evaluation
- `ferrum-introduction-to-strings.md` — Strings vs bytes, UTF-8, encoding
- `ferrum-introduction-to-smart-pointers.md` — Box, Rc, Arc, Weak
- `ferrum-introduction-to-modules.md` — Modules, packages, dependencies
- `ferrum-introduction-to-effects.md` — Effect system basics
- `ferrum-introduction-to-proofs.md` — Contracts and verification
- `ferrum-introduction-to-async.md` — Async, futures, structured concurrency
- `ferrum-introduction-to-ffi.md` — FFI, calling C, exposing to C
- `ferrum-introduction-to-unsafe.md` — Safety levels, when to use unsafe

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
- `ferrum-lang-foreign-models.md` — Foreign model system (`extern(swift)`, `extern(python)`, `extern(beam)`, GPU targets, etc.)

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

**Extended Libraries** (32 external library design documents, `ferrum_extlib_*.md`):
Graphics, UI, networking, cryptography, serialization, text, accessibility, fuzzing, and more. See `ferrum_extlib_*.md`.

**SemanticQuery (Ferrum-specific):**
- `ferrum-semanticquery.md` — SemanticQuery integration with Ferrum's type system and effects

**Implementation:**
- `ferrum-plan.md` — 34-phase implementation plan (long-arc)
- `ferrum-PLAN-FAST.md` — Fast-path MVP plan: correct JVM-target compiler in 8 weeks

**Design rationale and outreach:**
- `ferrum-fears-and-hopes.md` — Design philosophy and failure mode analysis
- `ferrum-PRFAQ.md` — Press release and FAQ (aspirational)
- `ferrum-release-announcement.md` — Release announcement (aspirational)
- `ferrum-arxiv-paper.md` — arXiv-style research paper (aspirational)

### SemanticQuery

**A protocol for exposing compiler internals to AI agents and tools.**

SemanticQuery turns compilers from batch processors into interactive reasoning engines. It is designed as a language-agnostic protocol (parallel to LSP) that any compiler can implement.

| Operation | Description |
|-----------|-------------|
| Point Query | Ask the compiler what it knows at any program point: types, effects, regions, active contracts, borrow state |
| Counterfactual Query | Inject a proposed invariant and ask if it closes a verification gap — without modifying source |
| Monitor | Place semantically-aware traps that the compiler weaves into the binary correctly because it understands the type system |
| Scaffolding | Generate test scaffolding from what the compiler already knows: `requires` → preconditions, `ensures` → oracles |
| SMT Export | Export verification conditions as SMT-LIB for external solvers |

Prior art: ASIS (Ada, ISO/IEC 15291:1999), Libadalang, Roslyn, rust-analyzer extensions, Dafny LSP, SPARK/GNATprove. The Counterfactual Query operation is novel — no prior system exposes this.

**Files:**
- `SemanticQuery-protocol-spec.md` — Language-agnostic protocol specification (start here for non-Ferrum use)
- `SemanticQuery-mcp-spec.md` — MCP server specification (AI agent integration)
- `ferrum-semanticquery.md` — Ferrum-specific SemanticQuery implementation and integration
- `compiler-agent-api.md` — Earlier, informal version of the same idea

## Design Philosophy

These projects share some principles:

- **Explicit over implicit** — but with inference where the compiler can prove correctness
- **Zero-cost abstractions** — safety features that compile away when verified
- **Graduated escape hatches** — multiple safety levels rather than a single `unsafe` keyword
- **Contracts as code** — preconditions, postconditions, and invariants are first-class
- **Allocator awareness** — memory allocation is never hidden from the programmer

## Status

These are design documents and thought experiments. No compilers exist yet. Ferrum has a fast-path MVP plan (`ferrum-PLAN-FAST.md`) targeting a JVM bootstrap compiler (Ferrum → Kotlin source → JVM, with a load-time verification agent); Zinc is designed for gradual upstream contribution to Zig. SemanticQuery is a standalone protocol proposal intended for adoption by LLVM, rustc, and other compiler infrastructure.

