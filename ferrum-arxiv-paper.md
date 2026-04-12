> **Aspirational document.** Ferrum does not yet exist as a working implementation. This paper is a design artifact — written in the style of a completed research paper to make the intended architecture, claims, and evaluation methodology precise, not to report actual experimental results.

---

# Ferrum: Progressive Load-Time Verification for a Systems Language with Ownership, Algebraic Effects, and First-Class Contracts

**Abstract**

We present Ferrum, a systems programming language combining ownership-based memory management, algebraic effect tracking, region inference, and first-class pre/postcondition contracts. We describe its bootstrap implementation architecture, which achieves the user experience of compile-time verification without requiring a complete static analysis pipeline. The key insight is that *load-time verification* — checking safety properties before any user code executes — is operationally equivalent to compile-time verification from the programmer's perspective, while requiring dramatically less compiler infrastructure to implement. Ferrum's architecture is explicitly progressive: the same verification code runs at load time initially and migrates to compile time as the static analysis matures. Each migration step is an engineering task, not a redesign. We evaluate Ferrum on a suite of systems programming benchmarks and find that it outperforms CPython 3.12 by 3.1–8.4× with all safety checks enabled, while catching every injected safety violation in our fault injection suite. The full Ferrum language specification, compiler source, and benchmark suite are available at [URL].

---

## 1. Introduction

Systems programming languages face a fundamental tension between safety and control. Memory-safe languages (Java, Python, Go) provide strong safety guarantees at the cost of garbage collection, unpredictable latency, and loss of direct hardware access. Unsafe languages (C, C++) provide full control at the cost of undefined behavior, buffer overflows, and data races. Rust demonstrated that this tradeoff is not fundamental — ownership-based memory management can eliminate entire classes of safety violations with no runtime overhead. But Rust's adoption has been slowed by its steep learning curve, particularly the burden of explicit lifetime annotations and the adversarial early experience with the borrow checker.

Ferrum addresses both the language design problem and the implementation problem. On the language side, Ferrum extends Rust's ownership model with region inference (eliminating ~90% of lifetime annotations in our test suite), an algebraic effect system that tracks IO, networking, and synchronization in function types, and first-class pre/postcondition contracts verified by an SMT solver. On the implementation side, Ferrum introduces a novel bootstrap architecture that provides compile-time-quality verification from the first implementation, without requiring a complete static analysis pipeline.

The bootstrap architecture rests on three observations:

1. **Load-time is before run-time.** A safety check that runs before any user code executes provides the same guarantee to the programmer as a check at compile time: the program either passes or is rejected before doing anything observable.

2. **Decorated bytecode is a verification interchange format.** A compiler that preserves semantic annotations (contracts, ownership framing, effect tags) as metadata in its output creates a target that both a runtime and a static verifier can consume. The static verifier need not understand Ferrum syntax — only the annotations.

3. **Verification and optimization are duals.** A static proof that a runtime assertion is unreachable is equivalent to an optimization that removes it. The same infrastructure that proves contracts moves checks from runtime to compile time. No redesign is required.

These observations yield the Ferrum verification pipeline: an unsafe transpiler emits decorated JVM bytecode; a Boogie-based verifier discharges what it can statically; a load-time pass enforces the remainder before execution; a runtime library catches anything that reaches execution. As the static verifier matures, checks migrate earlier. The user experience is stable throughout.

We make the following contributions:

- The Ferrum language design: ownership with region inference, algebraic effects, and first-class contracts in a unified system (§3)
- The progressive load-time verification architecture (§4)
- The decorated bytecode intermediate representation and its dual use by runtime and static verifiers (§4.2)
- An implementation of the complete pipeline in approximately 8,000 lines of Rust (§5)
- An evaluation demonstrating correctness (100% fault detection in our suite) and performance (3.1–8.4× over CPython with all checks enabled) (§6)

---

## 2. Background and Related Work

### 2.1 Ownership-Based Memory Safety

Rust [Matsakis & Klock 2014] demonstrated practical ownership-based memory safety. Its borrow checker enforces that at any point, a value has either any number of shared references or exactly one mutable reference, but never both. This eliminates use-after-free, double-free, and data race vulnerabilities without garbage collection. Ferrum retains Rust's core ownership model and extends it with region inference [Tofte & Talpin 1997] to eliminate the majority of explicit lifetime annotations.

Cyclone [Jim et al. 2002] explored region-based memory management in a C-compatible language. It achieved partial region inference but required explicit annotations for complex cases. Ferrum's region inference is more aggressive, targeting 90% annotation-free code in typical programs, with explicit annotations serving as load-bearing specifications rather than noise.

### 2.2 Effect Systems

Effect systems track computational effects (IO, exceptions, state) in types. Lucassen & Gifford [1988] introduced region effects for memory. More recent work on algebraic effects [Plotkin & Pretnar 2009] and row-polymorphic effect systems [Leijen 2017, Koka language] provides a foundation for Ferrum's `!` annotation model. Unlike monadic effect systems (Haskell's IO monad, mtl), Ferrum's effects are annotations on function types rather than type constructors, enabling effect inference and avoiding the need for monadic threading.

Frank [Lindley et al. 2017] and Koka [Leijen 2014] are the closest precedents for effect-annotated systems languages. Ferrum differs in targeting systems programming specifically, integrating effects with ownership (an effect annotation constrains what borrows are observable across an effect boundary), and adopting an inference-first model where annotations are required only at public API boundaries.

### 2.3 Contract Verification

Eiffel [Meyer 1992] introduced design-by-contract with `require`/`ensure` clauses checked at runtime. Dafny [Leino 2010] verifies contracts statically using Boogie and Z3. LiquidHaskell [Vazou et al. 2014] adds refinement types to Haskell, verified by SMT. Frama-C [Cuoq et al. 2012] verifies C programs against ACSL annotations.

Ferrum's contract system is closest to Dafny's in structure. The key difference is that Ferrum contracts coexist with ownership: a `requires` clause can constrain borrow state (`requires arr.is_sorted()`), and `ensures` clauses can describe ownership transfer (`ensures result` is the sole owner of the returned allocation). The ownership framing translates to separation logic predicates in the Boogie encoding.

### 2.4 Bytecode Verification

The JVM bytecode verifier [Gosling et al. 1996] performs type-safety checking at load time, before execution. This established that load-time verification is a viable alternative to compile-time verification for some properties. Ferrum extends this insight to a richer property set (ownership, effects, contracts) and to a VM that was not designed with these properties in mind, using custom attributes as the annotation carrier.

WALA [Fink & Tip 2005] and Soot [Vallée-Rai et al. 1999] demonstrated that JVM bytecode is amenable to program analysis beyond type checking. Ferrum's load-time pass follows this tradition.

### 2.5 Bootstrap Compiler Architectures

Multiple languages have used unsafe or simplified bootstrap compilers to achieve an initial working implementation. Rust's original implementation was in OCaml [Hoare 2010]; the current compiler bootstraps via an earlier Rust compiler version. Go 1.5 replaced its C bootstrap compiler with a self-hosted Go compiler [Cox 2015]. Ferrum's bootstrap architecture is distinguished by its explicit design for progressive migration: unlike Go's one-time replacement, Ferrum's architecture allows individual verification passes to migrate from load-time to compile-time independently.

---

## 3. The Ferrum Language

We describe Ferrum's key design decisions. The full language specification is in the companion document; this section focuses on aspects relevant to the verification architecture.

### 3.1 Type System Overview

Ferrum's type system is stratified into four orthogonal layers:

```
Effect layer:     IO · Net · Sync · Alloc[A]
Ownership layer:  Owned T · &T · &mut T · region
Type layer:       Scalar · Sum · Product · fn
Allocator layer:  Heap · Arena · Pool · Null
```

These layers compose independently: `Result[Vec[u8] | Arena, Error] ! IO` denotes a Result-typed value whose Vec component is arena-allocated, returned from a function that may perform IO.

### 3.2 Ownership and Region Inference

Ferrum enforces Rust's borrow rules: any number of shared references (`&T`) or exactly one exclusive reference (`&mut T`), but never both; references cannot outlive their referents. The borrow checker is a constraint solver over region variables.

Region inference assigns the minimal valid region to each reference through a constraint-based algorithm analogous to Hindley-Milner type inference. Most programs require no explicit annotations; the algorithm infers regions from the program's borrow structure. When the constraint system is underdetermined, the compiler requests an annotation that resolves the ambiguity. In our experience, fewer than 10% of functions in typical code require explicit annotations.

### 3.3 Algebraic Effects

Effects are tracked as sets in function type signatures:

```
pub fn read_file(path: &str): Result[String] ! IO
pub fn fetch(url: &str): Result[Bytes] ! Net
pub fn spawn(f: fn(): T): Task[T] ! Sync + Alloc[Heap]
```

A function with no `!` annotation is pure; the compiler verifies this statically. Effects are inferred for private functions and required at public boundaries. Effect polymorphism is supported via effect variables: `fn map[T,U][eff](f: fn(T):U!eff, opt: Option[T]): Option[U]!eff`.

### 3.4 Contracts

Pre and postconditions are written as `requires` and `ensures` clauses:

```ferrum
fn binary_search[T: Ord](arr: &[T], target: &T): Option[usize]
    requires arr.is_sorted()
    ensures match result {
        Some(i) => arr[i] == *target,
        None    => !arr.contains(target),
    }
```

Type invariants are declared inline:

```ferrum
struct SortedVec[T: Ord] {
    data: Vec[T],
    invariant forall i, j where i < j => self.data[i] <= self.data[j]
}
```

Proof functions are pure, total, and erased after verification:

```ferrum
proof fn reverse_involutive[T](xs: &[T])
    ensures xs.reverse().reverse() == xs
{ ... }
```

### 3.5 Design Principles

**Correct before fast.** No optimization removes a semantic check unless the check is proven redundant. Every annotation is a claim that can be verified.

**Annotations as deprecation queue.** Every explicit annotation marks a gap in compiler inference. As inference improves, annotations become redundant warnings. The asymptote is zero unnecessary annotations, not zero annotations.

**Progressive disclosure.** The core working set (no explicit lifetimes, no effect annotations on private functions, no contracts) is productive indefinitely. Advanced features are encountered when needed.

---

## 4. The Progressive Verification Architecture

### 4.1 Overview

The Ferrum compilation pipeline consists of four stages with explicit progression semantics:

```
Stage 1: Unsafe transpiler
         Ferrum source → decorated JVM bytecode
         No verification. Preserves all annotations.

Stage 2: Boogie verification pass
         Decorated bytecode → Z3 queries
         Discharges contracts statically where possible.
         Marks verified assertions as dead.

Stage 3: Load-time verification pass
         Decorated bytecode → accept or reject + errors
         Enforces ownership, effects, region discipline.
         Runs before any user code.

Stage 4: Runtime library
         Instrumented bytecode → runtime borrow/effect/contract checks
         Backstop for anything not caught earlier.
```

Stages 1 and 2 run at compile time. Stage 3 runs at program load. Stage 4 runs during program execution. Over time, checks migrate from Stage 4 to Stage 3 (load-time), then to Stage 2 (static), then to Stage 1 (rejected without emitting bytecode). The user experience is stable throughout this migration — errors are structured, carry source locations, and use the same message format regardless of which stage emits them.

### 4.2 Decorated Bytecode

The transpiler emits standard JVM bytecode augmented with custom attributes in the `Ferrum.*` namespace. These attributes are ignored by the JVM and consumed by the Boogie and load-time passes.

Core attribute types:

```
Ferrum.Requires(expr: SerializedAST)
Ferrum.Ensures(expr: SerializedAST, result_binding: String)
Ferrum.Invariant(expr: SerializedAST)
Ferrum.Borrows(var: String, mode: Shared|Exclusive, region: RegionId)
Ferrum.Effects(set: Set[Effect])
Ferrum.RegionEntry(id: RegionId)
Ferrum.RegionExit(id: RegionId)
Ferrum.VerifiedBy(proof_fn: MethodRef)
Ferrum.SourceLocation(file: String, line: u32, col: u32)
```

The `SerializedAST` format is a compact S-expression encoding of the contract expression, referencing local variable names from the method's debug table. The Boogie pass translates these to Boogie expressions. The load-time pass evaluates them against the runtime state.

### 4.3 Boogie Encoding

For each Ferrum function with contracts, the Boogie pass emits a Boogie procedure with the corresponding `requires` and `ensures` clauses. The ownership model is encoded as separation logic using a standard heap model.

Ferrum's exclusive borrow (`&mut T`) maps to Boogie's `acc(x, write)` permission assertion. Shared borrows (`&T`) map to `acc(x, read)`. Ownership transfer maps to permission transfer. This encoding is standard in the Viper verification infrastructure [Müller et al. 2016]; we adapt it for Ferrum's specific borrow rules.

Z3 discharge succeeds for: linear arithmetic contracts, simple structural predicates (`is_sorted`, `is_empty`, `len == n`), array bounds, and contracts expressible as quantifier-free first-order formulas. Contracts that discharge statically have their runtime assertions marked with a `Ferrum.Verified` attribute; the load-time pass skips them and the runtime library skips them.

For contracts Z3 cannot discharge, the Boogie pass emits a warning indicating that runtime checking will remain active. This provides visibility into verification coverage without failing the build.

### 4.4 Load-Time Verification Pass

The load-time pass runs after class loading and before the JVM executes the program's entry point. It traverses all loaded Ferrum-annotated methods and performs:

**Borrow flow analysis.** For each method, the pass performs a forward dataflow analysis over the bytecode, tracking borrow state at each program point. It verifies that: no exclusive borrow exists when a shared borrow is created (and vice versa); no reference is used after its borrow is released; region-local values do not escape their region.

**Effect annotation verification.** The pass checks that every call from a pure function (no `Ferrum.Effects` attribute or empty effect set) targets only other pure functions. It reports violations as errors.

**Contract consistency.** For contracts not discharged by Boogie, the pass inserts JVM assertion bytecode immediately preceding and following the method body. These assertions reference the serialized contract AST and are evaluated by the runtime library.

If the load-time pass finds any violation, it emits a structured error and terminates before any user code runs. The error format is identical to compile-time errors: source file, line, column, error code, message, and suggested fix where available.

### 4.5 The Runtime Library

The runtime library is a JVM library providing:

- `BorrowTracker`: per-object shared/exclusive borrow counters with source-location-aware error messages
- `EffectStack`: thread-local stack of active effects, checked on function entry
- `RegionManager`: region entry/exit with reachability tracking for escape detection
- `ContractEvaluator`: evaluates serialized contract AST expressions against runtime values

The runtime library is the final backstop. In practice, with the load-time pass enabled, runtime violations represent either dynamic conditions the load-time pass cannot anticipate (e.g., a contract that depends on runtime data) or bugs in the load-time pass itself. The latter are logged and reported as compiler bugs.

### 4.6 Error Message Unification

All four stages use a common error message infrastructure. Each stage produces a `FerrumDiagnostic` value:

```
FerrumDiagnostic {
    code: ErrorCode,
    severity: Error | Warning | Note,
    primary_span: SourceSpan,
    secondary_spans: Vec[(SourceSpan, String)],
    message: String,
    help: Option[String],
    stage: Transpiler | Boogie | LoadTime | Runtime,
}
```

The `stage` field is included in verbose output but not in the default display. A runtime borrow error and a compile-time borrow error look identical to the programmer; the only difference is when they appeared. This is intentional: the migration from runtime to compile-time detection should be invisible to users.

---

## 5. Implementation

### 5.1 The Unsafe Transpiler

The transpiler is implemented in Rust (~8,000 lines). It uses the `logos` crate for lexing and `chumsky` for parsing, producing a typed AST. A single-pass lowering stage emits JVM bytecode using the `cafebabe` crate for class file generation.

The transpiler performs no borrow checking, effect inference, or contract verification. It performs basic name resolution and enough type inference (Hindley-Milner with trait constraints) to resolve method dispatch and emit typed bytecode. Type parameters are monomorphized for primitive types and erased to `Object` for reference types, consistent with JVM generic semantics.

Known limitations in 0.1: generic type inference is incomplete for higher-order cases; the effect system is tracked but not fully enforced at the transpiler level; region inference is not implemented (all regions are approximated as the enclosing function scope).

### 5.2 The Boogie Pass

The Boogie pass is implemented as a JVM agent (~2,000 lines of Kotlin) that attaches at load time before user code. It reads `Ferrum.*` custom attributes using the ASM bytecode library, generates Boogie source, invokes the Boogie/Z3 toolchain via subprocess, and processes results.

The heap model is adapted from the Viper project's `HeapModule`, extended with Ferrum-specific axioms for borrow exclusivity and region containment. The encoding of Ferrum's borrow rules as separation logic permissions required approximately 400 lines of Boogie axioms.

The Boogie pass runs in parallel across all methods using a thread pool sized to available cores. For the benchmark suite, Boogie verification adds approximately 0.8 seconds to program startup for a program with 200 annotated functions. This is acceptable for a 0.1 release and will be addressed by persistent verification caching in 0.2.

### 5.3 The Runtime Library

The runtime library (~1,500 lines of Kotlin) is structured as a pure library with no JVM agent requirements. `BorrowTracker` uses a per-object `AtomicInteger` for shared borrow counts and a `compareAndSwap` for exclusive borrow acquisition, providing correct behavior under concurrent access.

`ContractEvaluator` interprets serialized AST expressions against a `Map<String, Any>` of local variable bindings captured at the contract check point. The evaluator handles the subset of Ferrum expressions that can appear in contracts: arithmetic, comparisons, method calls on known types (`Vec.len`, `Slice.is_sorted`, etc.), and match expressions. Unknown method calls in contracts fall back to a dynamic reflection-based evaluation with a warning.

---

## 6. Evaluation

### 6.1 Benchmarks

We evaluate on seven programs from the Ferrum benchmark suite:

| Program | Description | Lines |
|---------|-------------|-------|
| `json-parse` | Recursive descent JSON parser | 412 |
| `text-index` | Inverted index over text files | 287 |
| `http-client` | HTTP/1.1 request and response parsing | 531 |
| `matrix-mul` | Dense matrix multiplication, f64 | 89 |
| `bsearch` | Binary search with full contracts | 47 |
| `sort-bench` | Timsort implementation | 318 |
| `channel-perf` | Multi-producer single-consumer channel | 203 |

All programs are correct implementations with full contract annotations on public functions. We compare against CPython 3.12 and against the JVM baseline (equivalent Java programs, no Ferrum overhead).

### 6.2 Performance Results

| Benchmark | CPython 3.12 | JVM baseline | Ferrum (full checks) | Ferrum (checks off) |
|-----------|-------------|-------------|---------------------|-------------------|
| json-parse | 1.00× | 6.3× | 4.1× | 5.9× |
| text-index | 1.00× | 7.8× | 5.2× | 7.1× |
| http-client | 1.00× | 5.1× | 3.8× | 4.9× |
| matrix-mul | 1.00× | 11.2× | 8.4× | 10.8× |
| bsearch | 1.00× | 9.4× | 6.9× | 9.1× |
| sort-bench | 1.00× | 8.3× | 5.7× | 7.9× |
| channel-perf | 1.00× | 4.2× | 3.1× | 4.0× |

Ferrum with full checks enabled is 3.1–8.4× faster than CPython. The overhead of Ferrum's safety checks relative to the JVM baseline is 22–34%, concentrated in borrow tracking on hot paths. Contract overhead accounts for 8–12% on programs with dense contract annotations (`bsearch`, `sort-bench`).

We note that this comparison is against CPython without any safety guarantees. A fair comparison would include CPython with equivalent runtime checks (e.g., via beartype for type checking, explicit contract assertions). We do not have a CPython baseline with equivalent safety, but expect it to be substantially slower than the unchecked CPython numbers.

### 6.3 Fault Injection

To evaluate whether Ferrum catches safety violations, we inject 847 faults across the benchmark suite:

| Fault type | Injected | Caught by Boogie | Caught at load time | Caught at runtime | Missed |
|-----------|---------|-----------------|--------------------|--------------------|--------|
| Precondition violation | 203 | 89 (44%) | 114 (56%) | 0 | 0 |
| Postcondition violation | 187 | 71 (38%) | 116 (62%) | 0 | 0 |
| Borrow alias (mut + shared) | 156 | 0 | 141 (90%) | 15 (10%) | 0 |
| Use after region exit | 98 | 0 | 82 (84%) | 16 (16%) | 0 |
| Effect violation (pure calls impure) | 127 | 0 | 127 (100%) | 0 | 0 |
| Type invariant violation | 76 | 31 (41%) | 45 (59%) | 0 | 0 |
| **Total** | **847** | **191 (23%)** | **625 (74%)** | **31 (4%)** | **0** |

Detection rate: **100%**. All 847 injected faults were caught before producing incorrect output. The 31 runtime catches represent dynamic borrow patterns (borrow state depends on runtime data) that the load-time dataflow analysis conservatively approximated.

### 6.4 Verification Coverage

Of 1,247 contract clauses across the benchmark suite, Boogie discharged 412 statically (33%). The remaining 835 are active runtime checks. Boogie coverage is highest for arithmetic contracts (linear arithmetic, 71% discharged) and lowest for structural contracts (is_sorted, is_permutation — 0% discharged in 0.1, pending better heap model axioms).

### 6.5 Compilation Time

The transpiler processes 1,000 lines of Ferrum per second on a modern workstation. The Boogie pass adds 0.3–1.2 seconds per 100 annotated functions. Total pipeline latency for the benchmark programs (89–531 lines) ranges from 0.4 to 2.1 seconds. This is acceptable for interactive development; caching will address it for large programs.

---

## 7. Discussion

### 7.1 Load-Time vs. Compile-Time Verification

The key claim of this paper — that load-time verification is operationally equivalent to compile-time verification from the programmer's perspective — deserves scrutiny.

The practical differences: load-time verification runs after distribution (a binary can be distributed without verification artifacts) and adds to program startup time. Compile-time verification runs before distribution and has no startup cost.

For the use cases we target (developer tools, server applications, systems utilities), neither difference is significant in 0.1. Startup latency of under 2 seconds is acceptable. Distribution of unverified binaries is addressed by signing the verification result into the binary (future work).

For safety-critical embedded systems where startup latency is constrained, load-time verification would be unacceptable. This points to a clear upgrade path: move verification to compile time for release builds, retain load-time verification for debug builds as a second check.

### 7.2 The Progressive Migration Path

The architecture's progressive migration property has implications beyond performance. Each migration of a check from runtime to load-time to static represents a proof that the check is redundant for a class of programs. This proof is executable documentation: the annotation that was previously required becomes a warning, then is auto-removed. The codebase gets simpler as the compiler improves.

This is a different relationship between language evolution and existing code than is typical. Usually, new language versions add features that require code changes to adopt. Ferrum's annotation-as-deprecation-queue model means new compiler versions make existing code simpler, not more complex.

### 7.3 Boogie as Verification Backend

Using an existing SMT-backed verifier (Boogie/Z3) as the static verification layer provides significant leverage but introduces dependencies. Boogie's verification time is unpredictable for complex contracts; Z3's performance varies with query structure; neither is designed for interactive use.

For 0.1, these limitations are acceptable — Boogie runs offline and its results are cached. For interactive IDE integration (inline error display during editing), we will need either a faster incremental verifier or a tighter integration with the compiler's constraint solver. This is future work.

---

## 8. Related Work (Extended)

**Viper** [Müller et al. 2016] is a verification infrastructure for permission-based reasoning over heap-manipulating programs. Ferrum's Boogie encoding of ownership is directly adapted from Viper's heap model. Unlike Viper, Ferrum is not a verification-first language — it is a systems language that uses verification as one layer of a progressive safety stack.

**Prusti** [Astrauskas et al. 2019] verifies Rust programs by encoding them in Viper. This is the closest prior work to Ferrum's approach. Ferrum differs in: (1) targeting a new language with a cleaner contract surface than Rust, (2) the progressive architecture (Prusti is compile-time only), and (3) the scope — Ferrum's verification is designed as part of the language rather than an external tool.

**Creusot** [Denis et al. 2022] deductively verifies Rust programs using Why3. Like Prusti, it is a compile-time-only tool for an existing language. Ferrum's `verified_by` mechanism is inspired by Creusot's approach to linking fast implementations to proof functions.

**Stainless** [Blanc et al. 2013] verifies Scala programs. It demonstrates that JVM-targeting languages can support deductive verification. Ferrum's choice of JVM as bootstrap target is partly informed by this work.

---

## 9. Conclusion

We have presented Ferrum, a systems programming language and a novel bootstrap verification architecture. The core contribution is the observation that load-time verification is operationally equivalent to compile-time verification for the developer experience, enabling a correct-first, fast-later implementation strategy.

The Ferrum 0.1 implementation — built in approximately 11,500 lines of Rust and Kotlin over eight weeks — detects 100% of injected safety violations, outperforms CPython by 3.1–8.4× with all checks enabled, and provides structured error messages indistinguishable in form from compile-time errors.

The progressive architecture provides a clear roadmap to full static verification: each pass that currently runs at load time is a candidate for migration to compile time. The migration is incremental, independently deployable for each property, and invisible to users. We believe this architecture is applicable beyond Ferrum — any language that wishes to provide strong static guarantees but faces the bootstrapping challenge of building those guarantees before the static analysis pipeline is mature.

---

## References

Astrauskas, V., Müller, P., Poli, F., Summers, A.J. (2019). Leveraging Rust types for modular specification and verification. *OOPSLA 2019*.

Blanc, R., Kuncak, V., Kneuss, E., Suter, P. (2013). An overview of the Leon verification system. *Scala Workshop 2013*.

Cox, R. (2015). Go 1.5 bootstrap plan. *golang.org/s/go15bootstrap*.

Cuoq, P., Kirchner, F., Kosmatov, N., Prevosto, V., Signoles, J., Yakobowski, B. (2012). Frama-C: A software analysis perspective. *SEFM 2012*.

Denis, X., Jourdan, J.H., Marché, C. (2022). Creusot: a foundry for the deductive verification of Rust programs. *ICFEM 2022*.

Fink, S., Tip, F. (2005). WALA — the T.J. Watson Libraries for Analysis. *IBM Research*.

Gosling, J., Joy, B., Steele, G. (1996). *The Java Language Specification*. Addison-Wesley.

Hoare, G. (2010). Rust: safe systems programming. *Mozilla Research*.

Jim, T., Morrisett, G., Grossman, D., Hicks, M., Cheney, J., Wang, Y. (2002). Cyclone: a safe dialect of C. *USENIX ATC 2002*.

Leijen, D. (2017). Type directed compilation of row-typed algebraic effects. *POPL 2017*.

Leino, K.R.M. (2010). Dafny: an automatic program verifier for functional correctness. *LPAR 2010*.

Lindley, S., McBride, C., McLaughlin, C. (2017). Do be do be do. *POPL 2017*.

Lucassen, J.M., Gifford, D.K. (1988). Polymorphic effect systems. *POPL 1988*.

Matsakis, N., Klock, F. (2014). The Rust language. *HILT 2014*.

Meyer, B. (1992). Applying design by contract. *IEEE Computer 25(10)*.

Müller, P., Schwerhoff, M., Summers, A.J. (2016). Viper: a verification infrastructure for permission-based reasoning. *VMCAI 2016*.

Plotkin, G., Pretnar, M. (2009). Handlers of algebraic effects. *ESOP 2009*.

Tofte, M., Talpin, J.P. (1997). Region-based memory management. *Information and Computation 132(2)*.

Vallée-Rai, R., Co, P., Gagnon, E., Hendren, L., Lam, P., Sundaresan, V. (1999). Soot: a Java bytecode optimization framework. *CASCON 1999*.

Vazou, N., Seidel, E.L., Jhala, R., Vytiniotis, D., Peyton-Jones, S. (2014). Refinement types for Haskell. *ICFP 2014*.
