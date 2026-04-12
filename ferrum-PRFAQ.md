# Ferrum — PR/FAQ

> **Aspirational document.** Ferrum does not yet exist as a working implementation. This PR/FAQ is a design artifact — written to make goals, tradeoffs, and hard questions concrete before writing code.

---

## Press Release

### Ferrum: Systems Programming With Correctness Checks Included

*A compiled systems language that tracks memory, effects, and contracts at compile time — without asking programmers to annotate everything twice.*

---

Today we are announcing Ferrum, a compiled systems programming language for software where correctness is not optional. Ferrum combines Rust-style ownership with automated lifetime inference, algebraic effect tracking, and first-class contracts — and it checks all three before your program runs.

**The problem.** Systems programmers have two bad choices. They can write C or C++, where safety is conventional and correctness is aspirational. Or they can write Rust, where the borrow checker provides real safety guarantees but extracts payment in annotation burden, compiler fights, and a learning curve steep enough to drive many teams back to C++. Neither choice makes effects or contracts part of the language. Both make correctness the programmer's problem.

**The solution.** Ferrum makes correctness the compiler's problem.

Region inference eliminates roughly 90% of lifetime annotations without weakening the safety model — the compiler infers where values live, so programmers only annotate what the compiler genuinely cannot determine. Effect types (`! IO`, `! Net`, `! Sync`) make every function's side effects visible in its signature, enforced by the type checker, not aspirational documentation. Contracts (`requires`, `ensures`) are checked at runtime in debug mode and verified statically with an SMT solver where possible. For code where correctness is worth proving — cryptography, safety-critical firmware, financial logic — a proof system lets you establish properties with mathematical certainty.

None of this requires programmers to learn all at once. Ferrum uses progressive disclosure: Layer 0 is a working systems language smaller than Go. Effects, custom allocators, contracts, and proofs are layers on top, each opt-in, each self-contained.

```ferrum
fn binary_search[T: Ord](arr: &[T], target: &T): Option[usize]
    requires arr.is_sorted()
    ensures match result {
        Some(i) => arr[i] == *target,
        None    => !arr.contains(target),
    }
{
    // implementation
}
```

The `requires` and `ensures` here are not comments. If you call `binary_search` on an unsorted slice, you get an error before the function runs — with a source location, a message naming the violated contract, and a suggestion for how to fix it.

> *"I've been writing embedded systems code for fifteen years. Ferrum is the first language where I can look at a function signature and know — without reading the body — whether it touches hardware, allocates memory, or can fail. That's not documentation. That's a guarantee."*
> — Embedded systems engineer

```bash
cargo install ferrum-compiler
echo 'fn main() ! IO { println("hello, world") }' > hello.fe
ferrum run hello.fe
```

The specification, bootstrap compiler, and test suite are available at [repository link]. We are particularly interested in early adopters writing cryptographic libraries, embedded firmware, and safety-critical systems who can stress-test the contract verification system against real codebases.

> *"Rust proved that systems programmers will accept ownership semantics when the compiler helps enough. Ferrum's bet is that they'll also accept effect systems and formal contracts — when the compiler does the inference work. Safety without cognitive overhead isn't a contradiction. It's an engineering problem."*
> — Ferrum team

---

## External FAQ

*Questions from potential users, contributors, and skeptics.*

---

**How is this different from Rust?**

Three differences that matter day to day.

1. **~90% fewer lifetime annotations.** Rust's borrow checker requires programmers to name lifetimes when the compiler cannot infer them; in practice this is frequent enough to be a significant burden. Ferrum uses region inference (Tofte-Talpin style) to infer lifetimes as a constraint-solving problem. Explicit region annotations appear only when the system is genuinely ambiguous, and each annotation is load-bearing — not boilerplate.

2. **Effects in the type system.** Rust does not track IO, network access, or synchronization in types. A Rust function signature tells you its inputs and outputs; it does not tell you whether it writes to disk or touches shared state. Ferrum's effect system makes this part of the type, checked by the compiler.

3. **Contracts.** Rust has no built-in precondition or postcondition system. Ferrum's `requires`/`ensures` are verified — statically where an SMT solver can discharge them, at runtime otherwise — so behavioral specifications have compiler backing.

---

**What is region inference and how much does it actually reduce annotations?**

Lifetimes in Rust describe where in the program a value is valid. When a function returns a reference, the compiler needs to know which input the reference came from. When the borrow checker cannot determine this itself, the programmer must annotate.

Region inference treats lifetime constraints as a system of inequalities and solves them. The compiler determines the minimal region that satisfies all constraints. In practice: straightforward borrows, single-return-path functions, and most collection iterator patterns all infer cleanly. Complex patterns — multiple borrowed inputs with independent lifetimes, self-referential structures, cross-function constraint propagation — sometimes require annotation.

In the Ferrum test suite, fewer than 10% of functions require explicit lifetime annotations. That number will grow toward the actual hard cases as the standard library and ecosystem mature, but the structural claim holds: inference handles the common case, annotation handles the genuinely ambiguous case.

---

**What are effects and why should I care?**

An effect is something a function does to the world beyond producing a return value. Reading from disk is an effect. Writing to the network is an effect. Taking a lock is an effect.

In most languages, effects are undeclared. You learn about them by reading the implementation, consulting documentation (if it exists and is accurate), or discovering them in production. In Ferrum, effects are declared in the type signature and enforced by the compiler:

```ferrum
fn read_config(path: &str): Result[Config, IoError] ! IO
fn send_metrics(data: &Metrics) ! Net + IO
fn pure_transform(input: &Data): Data   // no annotation = compiler-verified pure
```

The compiler enforces that a function with no effect annotation cannot call any function that has one. This has practical consequences:
- You can write a pure compression function and know it will never silently write to a log.
- You can boundary-test impure code by separating the pure core from the IO shell.
- Libraries can declare their contracts — "this function never touches the network" — and the compiler enforces them.

---

**What are contracts? How are they different from assertions?**

Assertions say "this should be true here." They fire at one point in the program, with no connection to callers or callees.

Contracts say "this function requires X from callers and promises Y to them." They are checked at the call boundary, attributed to the right party, and compose across call stacks. When a precondition fires, the error points to the call site — the code that violated the requirement — not the implementation.

```ferrum
fn divide(a: f64, b: f64): f64
    requires b != 0.0
    ensures result * b == a
```

A call to `divide(x, 0.0)` produces a structured error at the call site: which contract was violated, what value caused the violation, and which function declared the requirement.

In `--proof` mode, the compiler attempts to verify contracts statically using an SMT solver. Linear arithmetic contracts (bounds checks, divisor nonzero, sorted array preconditions) are discharged without runtime cost. Contracts the solver cannot discharge remain as runtime checks. The programmer can promote a runtime check to a static proof by writing a proof function that provides the solver with the reasoning it needs.

---

**What is the proof system? Do I have to use it?**

No. Proofs are Layer 4 of a five-layer progressive disclosure model. Most programs never need them.

The proof system lets you write `proof fn` — pure, total functions, erased before code generation — that express invariants and reasoning steps in a form the compiler can verify. A `verified_by` annotation links a fast implementation to its proof:

```ferrum
proof fn sorted_after_insert[T: Ord](vec: &Vec[T], val: T): Prop
    requires vec.is_sorted()
    ensures vec.with_pushed(val).sorted_equivalent_to(vec.insert_sorted(val))
{ ... }

fn insert_sorted[T: Ord](vec: &mut Vec[T], val: T)
    requires vec.is_sorted()
    ensures vec.is_sorted()
    verified_by sorted_after_insert
```

The proof system is primarily useful for cryptographic implementations (where a subtle off-by-one matters), safety-critical firmware (where you must demonstrate to a regulator that bounds are never exceeded), and security-sensitive parsing (where an injected test might not find the attack).

For everything else, contracts with SMT discharge are sufficient.

---

**What platforms does Ferrum target?**

The bootstrap compiler outputs JVM bytecode (for rapid iteration and correctness verification). The 0.2 roadmap adds LLVM codegen, which covers x86_64, ARM64, RISC-V, and WebAssembly.

The standard library has platform-specific modules (`sys.jvm`, `sys.clr`, `sys.posix`, `sys.linux`, `sys.windows`, `sys.ohos`, `sys.wasi`, `sys.zephyr`) for runtimes and operating systems. Platform-neutral code never imports these directly; the portability cost is always explicit.

The FFI system uses a principled foreign model syntax (`extern(c)`, `extern(jvm)`, `extern(clr)`, `extern(swift)`, `extern(napi)`, etc.) rather than opaque ABI strings. Each model specifies the calling convention, object lifetime, exception protocol, and name mangling for that ecosystem.

---

**Can I use Ferrum with existing C/C++ code?**

Yes. The `extern(c)` foreign model covers C interop fully, with calling convention variants for stdcall, fastcall, ARM AAPCS, Win64, and System V:

```ferrum
extern(c) fn malloc(size: usize): *mut c_void  ! Unsafe
extern(c.system) fn CreateFileW(path: *const u16, ...): *mut c_void  ! Unsafe
```

C interop requires `! Unsafe` — it is marked, auditable, and scoped to the declaration site. Safe Ferrum wrappers around C libraries are the norm.

C++ interop is handled through the C ABI boundary (same as Rust). There is no direct C++ class or template interop; C++ code intended for Ferrum consumption should expose a C API.

---

**How does Ferrum handle async?**

Structured concurrency via `scope.spawn()` and `scope.race()`, not async/await coloring. Async in Ferrum is an effect (`! Async`) rather than a syntactic transformation:

```ferrum
fn fetch_all(urls: &[Url]): Vec[Response] ! Net + Async {
    scope.race(urls.iter().map(|u| fetch(u)))
}
```

There is no function color problem: async and sync code are distinguished by the `! Async` effect in the type, not by syntax. A function that awaits a value simply has `! Async` in its effect set. The runtime is structured; tasks have explicit scopes and cannot outlive them.

---

**What is the performance like?**

The bootstrap JVM compiler runs programs at 3.1–8.4× faster than CPython 3.12 with all safety checks enabled. That is an aspirational data point from our design analysis, not a measured benchmark on shipped code.

The LLVM target (0.2) is expected to be competitive with hand-written C for straightforward programs. Ferrum's ownership model means no GC pauses, no reference counting overhead for most values, and predictable allocation patterns compatible with LLVM's optimizer. Contracts add overhead in debug mode and are discharged or stripped in release mode.

---

**What is the learning curve?**

Shorter than Rust's, longer than Go's.

Layer 0 (the working set) is a systems language about the size of Go — structs, enums, pattern matching, traits, closures, iterators, basic ownership, no lifetime annotations for simple programs. A Rust or Go programmer can write Layer 0 Ferrum in a day.

Effects (Layer 1) take a few days. The mental model is: every function says what it does to the world, and the compiler enforces those declarations. This is analogous to learning checked exceptions, but without the pain, because inference handles most of the work.

Contracts (Layer 3) require familiarity with pre/postcondition reasoning, which most systems programmers have implicitly — it is just making the reasoning explicit. The proof system (Layer 4) is a research skill; it is opt-in and most codebases never need it.

---

**Why no user-defined macros?**

Two reasons.

- **Security:** macros that execute at compile time are a supply-chain attack surface. A compromised dependency that runs arbitrary code during compilation can silently modify the compiled output. Ferrum eliminates this attack vector entirely. (Ferrum's `proof fn` is not in this category — proof functions are verified by the compiler against a constrained logical sublanguage, cannot generate code, cannot access system resources, and are erased before the binary is produced.)
- **Simplicity:** macros in Rust are a parallel language with its own syntax and semantics. They make code harder to read, harder to tool, and harder to reason about. Ferrum's design philosophy is that the language itself should be expressive enough that macros are not needed for common patterns — generics, traits, effects, and layout declarations cover the structural cases.

The compiler has built-in derive-equivalent attributes (`@derive(Debug, Clone, Hash, ...)`) for common implementations.

---

**Why no exceptions?**

`Result[T, E]` makes error handling explicit and composable. Exceptions make error propagation implicit and non-local, which means errors can silently skip cleanup code and cross module boundaries without appearing in signatures.

The `!` effect on a function that can propagate errors — `! IO` or `! Panic` — is in the type signature. If a function can fail, its signature says so.

---

**Why no garbage collection?**

Ferrum is a systems language. GC is incompatible with hard real-time constraints, predictable memory layout for hardware interaction, and zero-allocation contexts (interrupt handlers, kernel code, embedded firmware). Region inference and ownership give the same ergonomic wins — no manual free, no use-after-free — without GC.

Applications that want GC ease can use languages with GC. Ferrum is for applications where you must control your memory.

---

## Internal FAQ

*Hard questions the team must answer honestly.*

---

**Is region inference actually tractable for a production compiler?**

Partially. Tofte-Talpin region inference is well-studied in research but has not been shipped in a production compiler at the scale Rust operates. The known hard cases are:
- Higher-order functions with complex lifetime polymorphism
- Self-referential data structures (handled separately via `pinned type`)
- Cross-crate inference boundaries

The mitigation is explicit fallback: when inference fails, the compiler asks for an annotation. The annotation is a data point for which cases the inference does not cover. As the inference improves, annotations become redundant (programs still compile, just without the explicit annotation). This is the same path NLL took in Rust.

The risk is that the "1% requiring annotation" turns out to be 20% in practice, undermining the primary usability claim. We will know this only by building real programs. The honest answer is: we believe the architecture is sound, but the annotation rate is an empirical question.

---

**Can effects, region inference, and contracts really coexist without mutual interference?**

This is the core research question. The interactions are:
- Effects on closures (capturing state + declaring effects)
- Region bounds on effect handlers
- Contracts that quantify over lifetimes
- Proof functions that reason about effects

Each pair has known solutions in the literature. The triple interaction (effect polymorphism + region constraints + dependent contracts) is less charted. The plan is to design the formalism as a fixpoint from the start, document incompleteness explicitly, and build a comprehensive compile-fail test suite before implementation. That test suite is the specification. Any interaction that fails a compile-fail test is a bug, not a design question.

---

**Why build the bootstrap compiler on JVM instead of LLVM from day one?**

JVM gives us a real target with a working GC, a rich reflection and debugging infrastructure, Boogie/Z3 integration for contract verification, and load-time verification that looks like compile-time verification from the user's perspective. It lets us validate the language design — effects, region inference, contracts — before committing to LLVM codegen, which is a separate large engineering investment.

The cost is that the bootstrap JVM compiler cannot be the long-term target: no raw pointer arithmetic, no bare-metal, no sub-millisecond GC pause guarantees. The 0.2 LLVM target is not optional. It is the real product; JVM is the development environment.

---

**Rust already exists. Why does Ferrum need to?**

Rust proved that systems programmers will accept a strong safety model if the ergonomics are right. It also revealed exactly where the ergonomic cost is: lifetime annotations, the borrow checker fight, and the absence of effects and contracts.

Ferrum's thesis is that region inference, algebraic effects, and first-class contracts can be added to an ownership-based language without blowing up the compiler or the learning curve — and that this combination unlocks a class of programs (formally verified cryptography, certified firmware, audited financial logic) that neither Rust nor C++ handles well today.

If this thesis is wrong — if the combination is unworkable or the annotation burden does not decrease — Rust is the right answer and Ferrum is a useful research experiment. If the thesis is right, Ferrum opens a tier of correctness that the current landscape does not offer.

---

**What is the risk of the proof system becoming dissertation-scope creep?**

High, if not managed. The mitigation is explicit scoping:

- **Version 0.1:** no proof system. Contracts with runtime checking only.
- **Version 0.2:** SMT discharge for linear arithmetic. Proof functions parsed but not verified.
- **Version 0.3:** Proof functions verified for termination and typing only.
- **Version 0.4:** Full `verified_by` integration for the subset of contracts Z3 can check.

The proof system is the last layer, not the first. It must not block shipping a useful language. Every proof-system feature is gated on "is there a real program that needs this today."

---

**Who is going to adopt a language with no working implementation?**

Nobody rational, for production use. The early adopters are:
- Researchers and students exploring the design space
- Embedded and safety-critical engineers who are currently writing the specification before any code, and want a language whose specification is already structured for formal verification
- Compiler engineers who find the region inference + effects problem interesting

The 0.1 release targets the bootstrap implementation, not general availability. General availability requires the LLVM target (0.2), a usable standard library (0.3), and a package manager (0.3). Until those exist, adoption is necessarily limited and that is acceptable.

---

**How do you handle the ecosystem problem? No packages means no language.**

Ecosystem is built, not designed. The path is:
1. Make C interop first-class (`extern(c)`) so any C library works from day one.
2. Provide a compatibility surface for Rust crates through the C ABI (most Rust library crates expose a C API or can be wrapped in one).
3. Ship a stdlib that covers the common cases — collections, IO, crypto primitives, JSON, HTTP — so most applications do not need third-party packages for basic work.
4. Build the package manager before 1.0, not after.

The language that requires "learn the ecosystem" before writing anything is not learnable. Ferrum must be useful with just the stdlib.

---

**Why not just extend Rust instead of building a new language?**

Region inference and algebraic effects require deep compiler changes — modifications to how types are checked, how the borrow checker works, and what the IR looks like. These are not additive features; they interact with every existing feature. Adding them to Rust would require either breaking existing Rust programs (not acceptable to the Rust project) or adding them as optional modes that do not compose cleanly with existing Rust code (not useful). A clean language is a better design surface than an extended one.

The answer may also be: in the long run, some of Ferrum's ideas migrate into Rust. That is fine. Research languages and production languages have a long history of cross-pollination.

---

**What is the realistic timeline?**

Assuming one focused developer with compiler experience working full time:

- 6 months: bootstrap transpiler producing valid JVM bytecode; parser, basic type checker, effect inference for built-in effects; region inference for single-function scope
- 12 months: load-time borrow checker; SMT contract discharge; runtime contract checking; basic stdlib (core, alloc, IO)
- 18 months: LLVM codegen; generics; cross-function region inference
- 24 months: complete stdlib; package manager; self-hosting in progress

Assuming one motivated engineer with a day job: double these numbers.

Assuming a small team (3–4) with compiler experience: the 12-month mark is 0.1 quality.

These are not commitments. They are calibration data for planning purposes.

---

**What is the plan for error message quality?**

Error messages are a first-class deliverable, not a polish pass. The specification for every diagnostic includes:
- The error code
- The source span
- The specific violated rule in human language
- The inferred context that triggered it
- At least one concrete suggestion for fixing it

Borrow checker errors in particular are the reputation-defining errors; Rust's NLL errors substantially drove adoption once they became readable. Ferrum's borrow errors must be at least as clear as Rust's 2018 NLL errors before any public release. This is a blocking requirement, not a nice-to-have.

A dedicated compile-fail test suite drives error message quality — each test asserts exact error codes and checks that the source span is correct.
