> **Aspirational document.** Ferrum does not yet exist as a working implementation. This release announcement is a design artifact — written to make the intended architecture and goals concrete, not to describe shipped software.

---

# Ferrum 0.1: A Systems Language That Checks Its Work

We are releasing the first working implementation of Ferrum — a systems programming language with ownership semantics, algebraic effect tracking, and first-class contracts. It runs faster than CPython on representative benchmarks, and every safety violation is caught before your program executes.

---

## What Ferrum Is

Ferrum is a compiled systems language designed around one question: can safety and control coexist without making programmers pay for it in annotations, compiler fights, or cognitive overhead?

The answer, we believe, is yes — if the language is built on four foundations:

1. **Ownership without lifetime annotations.** Ferrum tracks memory lifetimes using region inference. In our test suite, fewer than 10% of functions require explicit lifetime annotations. The compiler infers the rest.

2. **Effects as part of the type.** A function that does IO says so: `! IO`. A function that touches the network says so: `! Net`. Pure functions are verified pure by the compiler — calling anything impure is a type error. This is not documentation. It is a checked contract.

3. **Contracts that mean something.** `requires` and `ensures` are not comments. They are checked. In this release, they are checked at load time via runtime assertion and Boogie-backed SMT verification where the solver can discharge them statically.

4. **Correct before fast.** The compiler emits maximally instrumented code. Every contract fires on every call. Every borrow is tracked. Every effect is checked. Performance optimizations come later; correctness comes first.

---

## The Numbers

On a representative suite of programs (JSON parsing, text processing, HTTP request handling, numerical computation, data structure operations):

| Implementation | Relative speed | Safety model |
|---------------|---------------|-------------|
| CPython 3.12 | 1.0× (baseline) | None |
| Ferrum 0.1 | 3.1–8.4× | Full (ownership, effects, contracts) |
| Ferrum 0.1 (contracts off) | 4.2–11.7× | Ownership + effects |

Ferrum is faster than CPython with all safety checks enabled. This is not a surprising result — CPython is not fast, and the JVM JIT is genuinely good — but it validates the architecture. Safety does not cost performance relative to the baseline most Python programmers are actually running.

---

## What It Looks Like

```ferrum
fn main() ! IO {
    let args = env.args()
    if args.len() < 2 {
        println("usage: greet <name>")
        return
    }
    println("Hello, {}!", args[1])
}
```

```ferrum
fn binary_search[T: Ord](arr: &[T], target: &T): Option[usize]
    requires arr.is_sorted()
    ensures match result {
        Some(i) => arr[i] == *target,
        None    => !arr.contains(target),
    }
{
    let mut lo: usize = 0
    let mut hi: usize = arr.len()
    while lo < hi {
        let mid = lo + (hi - lo) / 2
        match arr[mid].cmp(target) {
            Less    => lo = mid + 1,
            Greater => hi = mid,
            Equal   => return Some(mid),
        }
    }
    None
}
```

```ferrum
fn process(docs: &[Document]): Summary ! IO
    requires docs.len() > 0
    ensures result.count == docs.len()
{
    let mut acc = Summary.new()
    for doc in docs {
        acc.add(doc)
    }
    acc
}
```

The `requires` and `ensures` on these functions are checked. If you call `binary_search` on an unsorted slice, you get an error before the function runs, with a source location and a message that tells you which contract was violated. The error looks like this:

```
error[C001]: precondition violated
  --> src/search.fe:47:5
   |
47 |     let idx = binary_search(&data, &query)
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = requires: arr.is_sorted()
   = note: violated at call site; arr is not sorted
   = help: sort the slice before calling, or use linear_search for unsorted data
```

---

## The Architecture

Ferrum 0.1 is not a traditional ahead-of-time compiler. It uses a three-layer pipeline:

```
Ferrum source
     ↓  (unsafe transpiler — Rust, ~8k lines)
Decorated JVM bytecode
(standard bytecode + Ferrum annotations as custom attributes)
     ↓  (Boogie verification pass)
Static verification of contracts (where solver can discharge)
     ↓  (load-time verification pass)
Runtime checks for anything not statically verified
     ↓
Your program runs
```

**The unsafe transpiler** parses Ferrum and emits JVM bytecode with all semantic annotations preserved as custom attributes — contracts, ownership framing, effect tags, region membership. It does not perform borrow checking, effect inference, or contract verification. It translates.

**The Boogie pass** reads the decorated bytecode and emits verification conditions for each annotated function. Contracts that Z3 can discharge are marked verified and their runtime assertions are stripped. Contracts it cannot discharge remain as runtime checks.

**The load-time verification pass** runs before any user code executes. It walks the decorated bytecode, checks borrow flow, verifies effect annotations, and enforces region discipline. Violations produce structured errors with source locations. From the user's perspective, this is indistinguishable from compile-time checking.

**The runtime library** provides the borrow tracker, effect stack, and region manager used by the instrumented code. It is the backstop: anything not caught by Boogie or the load-time pass will be caught here, at the first call that violates a contract.

---

## What Is Not Done

This is 0.1. The following are known limitations:

- **Generics are partially implemented.** Monomorphization works for concrete type parameters. Type parameter inference is incomplete for higher-order cases.
- **The standard library is minimal.** `core` is substantially complete. `alloc` (collections, String, Vec) is functional. `std.io` is partial. Networking is not implemented.
- **LLVM codegen is not implemented.** The JVM backend is the only target. Native binaries require waiting for 0.2.
- **The Boogie coverage is modest.** Linear arithmetic and simple structural contracts are discharged statically. Loop invariants and recursive contracts require manual proof functions, which are parsed but not yet verified.
- **Error messages need work.** Borrow errors in particular can point to the wrong location in complex cases.

---

## The Path Forward

0.1 validates the architecture. The next milestones:

**0.2:** LLVM codegen. Native binaries. Performance competitive with hand-written C for simple programs.

**0.3:** Complete generics. Full standard library. Package manager.

**0.4:** Full borrow checking moved to compile time. Load-time borrow pass becomes redundant and is removed.

**1.0:** Self-hosting. The Ferrum compiler compiles itself.

---

## Getting Started

```bash
cargo install ferrum-compiler
echo 'fn main() ! IO { println("hello, world") }' > hello.fe
ferrum run hello.fe
```

The source, specification documents, and test suite are at [repository link].

Bug reports, contract violations that weren't caught, and incorrect error messages are all high-priority issues. If Ferrum missed a safety violation or emitted a false positive, we want to know.

---

*Ferrum 0.1. Correct before fast.*
