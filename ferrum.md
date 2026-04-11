# Ferrum

Ferrum is a systems programming language. It compiles to native code, has no garbage collector, and gives you control over memory layout, allocation, and performance. It is designed for the same domains as C, C++, and Rust: operating systems, embedded systems, game engines, databases, compilers, and performance-critical applications.

## Core Properties

1. **Correct by construction.** Illegal states are unrepresentable at the type level wherever possible. Contracts enforce the rest.

2. **No hidden costs.** Every allocation, copy, synchronization, and effect is visible in the code or its type signature.

3. **Minimal annotation burden.** The compiler infers what it can. Annotations appear only when they carry information a human reader needs. Effects are inferred within modules, required only at `pub` boundaries. Allocators default to `Heap`, explicit only for custom allocation.

4. **Graduated trust.** Safety is a spectrum, not a binary. The compiler handles the common cases automatically; explicit annotations mark where human judgment fills the gap.

5. **Proofs are programs.** The same language that compiles to machine code can express and verify formal properties.

## Quick Example

```ferrum
fn main() ! IO {
    let args = env.args()
    if args.len() < 2 {
        println("Usage: greet <name>")
        return
    }
    println("Hello, {}!", args[1])
}
```

No semicolons. No header files. Generics use `[T]` not `<T>`. Paths use `.` not `::`. Effects like `! IO` appear at `pub` boundaries.

---

## Documentation

### Introductions

New to Ferrum? Start here. Each guide explains one concept for programmers who know C and Python:

| Document | Topic |
|----------|-------|
| [ferrum-introduction-for-programmers.md](ferrum-introduction-for-programmers.md) | **Start here** — Language basics, syntax, types, control flow |
| [ferrum-introduction-to-ownership.md](ferrum-introduction-to-ownership.md) | Ownership, moving, borrowing — no garbage collector, no manual free |
| [ferrum-introduction-to-option-result.md](ferrum-introduction-to-option-result.md) | Option and Result — no null pointers, no exceptions |
| [ferrum-introduction-to-traits.md](ferrum-introduction-to-traits.md) | Traits — polymorphism without inheritance |
| [ferrum-introduction-to-enums.md](ferrum-introduction-to-enums.md) | Enums with data, pattern matching — no more instanceof chains |
| [ferrum-introduction-to-generics.md](ferrum-introduction-to-generics.md) | Generics — type-safe containers without void* |
| [ferrum-introduction-to-iterators.md](ferrum-introduction-to-iterators.md) | Iterators and closures — lazy, chainable, zero-cost |
| [ferrum-introduction-to-strings.md](ferrum-introduction-to-strings.md) | Strings vs bytes — UTF-8 always, encoding explicit |
| [ferrum-introduction-to-smart-pointers.md](ferrum-introduction-to-smart-pointers.md) | Box, Rc, Arc — heap allocation without GC |
| [ferrum-introduction-to-modules.md](ferrum-introduction-to-modules.md) | Modules and packages — no header files, real namespaces |
| [ferrum-introduction-to-effects.md](ferrum-introduction-to-effects.md) | Effects — tracking IO, Net, Async in the type system |
| [ferrum-introduction-to-proofs.md](ferrum-introduction-to-proofs.md) | Contracts and proofs — catching bugs testing can't find |
| [ferrum-introduction-to-async.md](ferrum-introduction-to-async.md) | Async and concurrency — structured, not colored |
| [ferrum-introduction-to-ffi.md](ferrum-introduction-to-ffi.md) | FFI — calling C, exposing Ferrum to C |
| [ferrum-introduction-to-unsafe.md](ferrum-introduction-to-unsafe.md) | Safety levels — unchecked, trusted, extern, unsafe |

### Language Reference

The complete language specification:

| Document | Contents |
|----------|----------|
| [ferrum-language-reference.md](ferrum-language-reference.md) | Quick reference and index |
| [ferrum-lang-core.md](ferrum-lang-core.md) | Overview, lexical structure, keywords, literals |
| [ferrum-lang-types.md](ferrum-lang-types.md) | Type system: scalars, floats, constrained types, pointers |
| [ferrum-lang-ownership.md](ferrum-lang-ownership.md) | Ownership, regions, borrowing, effects, capabilities |
| [ferrum-lang-generics.md](ferrum-lang-generics.md) | Generics, traits, allocators, GADTs |
| [ferrum-lang-syntax.md](ferrum-lang-syntax.md) | Expressions, pattern matching, functions, declarations |
| [ferrum-lang-concurrency.md](ferrum-lang-concurrency.md) | Tasks, channels, synchronization, atomics |
| [ferrum-lang-verification.md](ferrum-lang-verification.md) | Contracts, proofs, binary layout, safety levels |
| [ferrum-lang-modules.md](ferrum-lang-modules.md) | Module system, packages, imports |
| [ferrum-lang-internals.md](ferrum-lang-internals.md) | Compiler algorithms, grammar |

### Standard Library

The standard library documentation:

| Document | Contents |
|----------|----------|
| [ferrum-stdlib.md](ferrum-stdlib.md) | Design principles and index |
| [ferrum-stdlib-core.md](ferrum-stdlib-core.md) | `core` — No-alloc primitives |
| [ferrum-stdlib-alloc.md](ferrum-stdlib-alloc.md) | `alloc` — Collections, strings, smart pointers |
| [ferrum-stdlib-io.md](ferrum-stdlib-io.md) | `io`, `fs` — IO traits, filesystem |
| [ferrum-stdlib-async-net.md](ferrum-stdlib-async-net.md) | `async`, `net`, `http` — Networking and async runtime |
| [ferrum-stdlib-math.md](ferrum-stdlib-math.md) | `math`, `linalg`, `simd` — Mathematics |
| [ferrum-stdlib-numeric.md](ferrum-stdlib-numeric.md) | `bigint`, `decimal`, `rational`, `complex` |
| [ferrum-stdlib-crypto-testing.md](ferrum-stdlib-crypto-testing.md) | `crypto`, `testing` — Cryptography and test framework |
| [ferrum-stdlib-util.md](ferrum-stdlib-util.md) | `fmt`, `time`, `sync`, `log` — Utilities |
| [ferrum-stdlib-sys.md](ferrum-stdlib-sys.md) | `sys` — Platform-specific APIs |
| [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md) | Platform abstraction layer |

### Meta-Documentation

Design rationale and implementation planning (not part of the language specification):

| Document | Contents |
|----------|----------|
| [ferrum-fears-and-hopes.md](ferrum-fears-and-hopes.md) | Design rationale and strategic thinking |
| [ferrum-plan.md](ferrum-plan.md) | Implementation plan and phasing |
