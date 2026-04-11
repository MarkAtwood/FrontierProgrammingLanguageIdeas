# Ferrum: Fast Path to a Correct MVP Compiler

**Goal:** A working Ferrum compiler that catches every safety violation before incorrect output is produced. Not fast output. Not a complete standard library. Not LLVM. Correct.

**Definition of correct:** 100% detection of the six fault categories from the arXiv paper — precondition violations, postcondition violations, borrow aliasing, use-after-region-exit, effect violations, and type invariant violations — before the program produces any observable incorrect output.

**Target:** 8 focused weeks for one experienced developer. The arXiv paper claims this timeline; we take it seriously.

---

## The Architecture in One Paragraph

The Rust transpiler reads Ferrum source and emits Kotlin source files with Ferrum semantic metadata attached as Kotlin runtime annotations. It does **no verification** — it just translates. `kotlinc` compiles the emitted Kotlin to JVM class files. A Kotlin runtime library provides the borrow tracker, effect stack, and contract evaluator that the instrumented code calls into. A Kotlin load-time JVM agent runs before any user code executes, walks all loaded Ferrum-annotated methods, and either accepts them or produces structured errors with source locations. The runtime library is the backstop for anything dynamic. The load-time agent catches everything static. Together they achieve 100% detection.

Boogie/Z3 is explicitly out of scope for this plan. It makes some errors appear earlier (compile-time instead of load-time) but does not change the detection rate. Add it in 0.2 when the core is solid.

---

## What Is In Scope

- Hello world, binary search, basic data structures run correctly
- `fn`, `let`, `if/else`, `while`, `for`, `match`, `return`
- Structs (`type`), enums, `impl`, basic traits (`Ord`, `Debug`, `Clone`)
- `&T`, `&mut T` — ownership tracked at runtime and load-time
- Effect annotations: `! IO`, `! Net`, `! Sync` — enforced at load-time
- `requires` and `ensures` — evaluated at runtime before/after function body
- Basic `Option[T]`, `Result[T,E]`, `Vec[T]`
- `println`, `env.args()`, basic string operations
- Structured error messages with source location for all violations
- Fault injection test suite: all 6 fault categories, 0% missed

## What Is Explicitly Out of Scope

- Boogie/Z3 static discharge (contracts are always runtime-checked)
- Full region inference (all regions approximate to enclosing function scope)
- Full generics inference (monomorphize concrete type params, erase reference params to Object)
- LLVM codegen — JVM is the only target
- Networking, async, crypto — stdlib minimum only
- Package manager
- SemanticQuery server
- Self-hosting

---

## Technology Decisions

Every decision below was chosen to minimize the distance between "nothing" and "running program that catches violations."

**Transpiler language: Rust.** The compiler ecosystem (logos, chumsky) is mature and fast to iterate on. Rust's type system catches structural bugs in the AST transformation early.

**Lexer: `logos`.** Declarative, fast, excellent for a language with straightforward tokenization. Ferrum has no significant lexer complexity (no heredocs, no raw string edge cases, no significant whitespace).

**Parser: `chumsky`.** Combinator-based recursive descent with error recovery. Good error messages out of the box. Ferrum's grammar is context-free; chumsky handles it cleanly.

**Intermediate target: Kotlin source.** The transpiler emits `.kt` files. Codegen is string interpolation over the typed AST — no bytecode APIs, no stack frame calculation. `kotlinc` handles verification. The emitted Kotlin is readable and debuggable. Async Ferrum functions emit blocking stubs annotated `@FerrumAsync`; coroutine mapping is a 0.2 problem.

**Semantic metadata: Kotlin runtime annotations.** Annotation types defined in `ferrum-runtime.jar` carry the same information that would have been custom JVM attributes: effects, borrows, contracts, source location. Kotlin `@Retention(RUNTIME)` annotations survive `kotlinc` and are readable by the load-time agent via ASM `visitAnnotation` — same mechanism, standard API.

**Runtime library and load-time agent: Kotlin.** JVM-native. The runtime library is a pure Kotlin library (`.jar`) linked at JVM startup. The load-time agent is a JVM premain agent that runs before main, reading annotations via ASM.

**JVM target version: 17 LTS.** Stable, widely available, good tooling.

**Build: Cargo (transpiler) + Gradle (runtime + agent).** Wrapped by a top-level shell script `ferrum` that orchestrates the pipeline.

---

## Phase Plan

Each phase ends with a concrete gate test. The phase is not done until the gate passes.

---

### Phase 0 — Infrastructure (Days 1–3)

**Goal:** Plumbing exists, nothing interesting yet.

- Cargo workspace: `ferrum-transpiler` crate
- Gradle project: `ferrum-runtime` and `ferrum-agent` modules
- Top-level `ferrum` shell script: parses subcommands (`run`, `build`, `check`)
- Error reporting scaffolding: `FerrumDiagnostic` struct, ANSI-formatted output
- Kotlin annotation types defined in `ferrum-runtime`: `@FerrumSource`, `@FerrumEffects`, `@FerrumBorrows`, `@FerrumBorrow`, `@FerrumRequires`, `@FerrumEnsures`, `@FerrumInvariant`, `@FerrumAsync`

**Gate:** `ferrum --version` prints a version string without crashing.

---

### Phase 1 — Lexer (Days 4–7)

**Goal:** Every Ferrum token recognized correctly.

- `logos`-based lexer over the full Ferrum token set
- Token types: keywords, identifiers, literals (integer, float, string, bool), operators, delimiters, comment stripping
- Error token for unrecognized characters with source span
- Span information preserved on every token (file, line, column)

**Gate:** `ferrum lex examples/binary_search.fe` produces a correct token stream with no panics. Lex all example programs from the language spec.

---

### Phase 2 — Parser and AST (Days 8–17)

**Goal:** Ferrum source → typed AST. No type checking yet.

The AST is the central data structure everything else reads. Get it right. Define it completely before implementing the parser.

**AST nodes (complete list):**
- `Item`: `FnDef`, `TypeDef`, `EnumDef`, `ImplBlock`, `TraitDef`, `ExternBlock`
- `Stmt`: `Let`, `Assign`, `Expr`, `Return`, `While`, `For`, `If`
- `Expr`: `Literal`, `Name`, `Call`, `MethodCall`, `Field`, `Index`, `Unary`, `Binary`, `Match`, `Block`, `Closure`, `Ref`, `Deref`
- `Type`: `Named`, `Ref` (shared/exclusive), `Slice`, `Fn`, `Infer`
- `Effect`: set of effect names
- `Contract`: `Requires(Expr)`, `Ensures(Expr)`
- `Pattern`: `Wildcard`, `Name`, `Struct`, `Enum`, `Tuple`, `Literal`

Every node carries a `SourceSpan`. There are no source-span-free AST nodes.

**Parser:** chumsky combinators, one function per AST node. Error recovery via `recover_with(skip_then_retry_until(...))` at statement boundaries.

**Gate:** `ferrum parse examples/binary_search.fe` prints the AST without panics. Parse all programs in the test suite. No parse panics on any syntactically invalid input (error recovery, not crash).

---

### Phase 3 — Type Inference and Name Resolution (Days 18–28)

**Goal:** Every expression has a type. Every name resolves to a definition.

This is the hardest phase intellectually. Do it incrementally:

**Step 3a — Name resolution:** Single-pass, build symbol table per scope. Resolve all `Name` references to their definitions. Report unresolved names as errors.

**Step 3b — Basic type inference:** Hindley-Milner over the resolved AST. Unification with occurs check. Type variables for let-polymorphism. Trait constraints as predicates on type variables (don't solve them fully yet — defer to monomorphization).

**Step 3c — Method resolution:** Given an expression of type `T` and a method name, find the `impl` block that provides it. Fail clearly if ambiguous or missing.

**Step 3d — Effect inference for private functions:** Walk the body, collect effects of all callees, union them. Record on the function definition. Public functions must declare their effects; private functions get them inferred.

**Step 3e — Region approximation:** All regions are `'fn` (the enclosing function). Borrows live as long as the function. This is wrong for some programs but correct for most programs in the MVP test suite. Programs that need cross-function regions will fail at load-time with an unresolved-region error; they require an explicit annotation. This is an honest limitation, not a silent bug.

**Gate:** `ferrum typecheck examples/binary_search.fe` reports all types and effects with no errors on a correctly typed program. Reports a type error (with source location) on a deliberately mistyped program.

---

### Phase 4 — Transpiler: Kotlin Source Emission (Days 29–42)

**Goal:** Ferrum source → Kotlin source with Ferrum annotations → compiled `.class` files. Programs run.

This phase does **no verification**. The transpiler's job is translation.

**Type mapping (Ferrum → Kotlin):**
- Scalars: `i8` → `Byte`, `i32` → `Int`, `i64` → `Long`, `f32` → `Float`, `f64` → `Double`, `bool` → `Boolean`, `u8`/`u16`/`u32`/`u64` → widened signed Kotlin types
- Strings: `&str` → `String`
- Structs → Kotlin `data class`
- Enums → Kotlin `sealed class` hierarchy (one subclass per variant)
- `Vec[T]` → `MutableList<T>`
- `Option[T]` → sealed class (`Some<T>` / `None`) — not nullable; keeps `when` pattern matching clean
- `Result[T,E]` → `FerrumResult<T,E>` in the runtime library
- Closures → Kotlin lambda `(T) -> R`
- Match → Kotlin `when`
- Async functions → blocking stub, `@FerrumAsync` annotation, coroutine mapping deferred to 0.2

**Annotation types (defined in `ferrum-runtime.jar`, emitted by transpiler):**

| Annotation | Emitted when |
|---|---|
| `@FerrumSource(file, line, col)` | Every function |
| `@FerrumEffects(vararg effects: String)` | Every function (empty = pure) |
| `@FerrumBorrows(vararg borrows: FerrumBorrow)` | Functions with `&T` / `&mut T` params |
| `@FerrumRequires(expr: String)` | Every `requires` clause (compact JSON AST) |
| `@FerrumEnsures(expr: String, result: String)` | Every `ensures` clause |
| `@FerrumInvariant(expr: String)` | Every type `invariant` |
| `@FerrumAsync` | Async functions emitted as blocking stubs |

Example emitted Kotlin for `fn binary_search`:

```kotlin
@FerrumSource(file = "search.fe", line = 1, col = 1)
@FerrumEffects()  // pure
@FerrumBorrows(FerrumBorrow(param = "haystack", kind = "Shared", region = "fn"))
@FerrumRequires(expr = """{"op":"call","fn":"is_sorted","args":["haystack"]}""")
@FerrumEnsures(expr = """{"op":"match","scrutinee":"result",...}""", result = "result")
fun <T : Comparable<T>> binarySearch(haystack: List<T>, needle: T): Option<T> {
    BorrowTracker.enter("search.fe:1")
    ContractEvaluator.checkRequires("""...""", mapOf("haystack" to haystack))
    BorrowTracker.acquireShared(haystack, "search.fe:1")
    // ... body ...
    val result: Option<T> = None
    ContractEvaluator.checkEnsures("""...""", mapOf("haystack" to haystack), result)
    BorrowTracker.exit("search.fe:1")
    return result
}
```

**Instrumentation** is emitted inline in the Kotlin source (not a post-processing step):
- On entry: `BorrowTracker.enter(method_id)`, `EffectStack.push(effects)`, `ContractEvaluator.checkRequires(expr, locals)`
- On exit: `ContractEvaluator.checkEnsures(expr, locals, result)`, `EffectStack.pop()`, `BorrowTracker.exit(method_id)`
- At `&mut` acquisition: `BorrowTracker.acquireExclusive(obj, source_span)`
- At `&` acquisition: `BorrowTracker.acquireShared(obj, source_span)`
- At borrow release: `BorrowTracker.release(obj, source_span)`
- At region exit: `RegionManager.exit(region_id)`

**Gate:** `ferrum run examples/hello.fe` prints "hello, world". `ferrum run examples/binary_search.fe` runs correctly on valid input (contract violations silently pass — that is Phase 5). `javap -verbose` on compiled class files shows `@FerrumSource`, `@FerrumEffects` annotations.

---

### Phase 5 — Runtime Library (Days 43–49)

**Goal:** The instrumentation calls from Phase 4 actually catch violations.

Four components, all Kotlin, all in `ferrum-runtime.jar`:

**`BorrowTracker`**
```
Per-object shared borrow count (AtomicInteger).
acquireExclusive(obj, span): CAS from 0 to -1. Fails if count != 0.
  → error[B001]: exclusive borrow of already-borrowed value
acquireShared(obj, span): CAS increment if count >= 0. Fails if count == -1.
  → error[B002]: shared borrow of exclusively-borrowed value
release(obj, span): decrement or reset to 0
regionExit(region_id): release all borrows tagged to this region
  → error[B003]: use after region exit (if accessed after this)
```

**`EffectStack`**
```
Thread-local stack of active effect sets.
push(effects): push onto stack
pop(): pop
checkPure(callee_effects, call_span): if current frame is pure and callee_effects != empty:
  → error[E001]: pure function calls impure function
```

**`RegionManager`**
```
Thread-local map: region_id → Set[obj_id].
enter(region_id): create entry
register(region_id, obj): add to set
exit(region_id): mark all registered objects as invalid
checkLive(obj, span): if obj is in an exited region:
  → error[R001]: use of value after region exit
```

**`ContractEvaluator`**
```
evaluate(serialized_ast, locals: Map<String, Any>): Boolean
  Interprets the contract AST against captured local variable bindings.
  Supports: arithmetic, comparison, boolean ops, method calls on known types,
            match expressions, len(), is_sorted(), is_empty(), contains().
  Unknown method calls: reflect and call. Log a warning if reflection used.

checkRequires(serialized, locals, span):
  if !evaluate(serialized, locals):
    → error[C001]: precondition violated (at call site)

checkEnsures(serialized, locals, result, span):
  if !evaluate(serialized, locals ∪ {result: result}):
    → error[C002]: postcondition violated (at function exit)
```

**Error format** (all four components use the same format):
```
error[B001]: exclusive borrow of already-borrowed value
  --> src/search.fe:47:5
   |
47 |     let idx = binary_search(&mut data, &query)
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: 'data' is already borrowed as shared at line 39
   = stage: runtime
```

**Gate:** Run a program that violates a borrow rule → runtime error with correct source location. Run a program that calls impure from pure → runtime error. Run a program with a violated requires → runtime error at call site. All errors have correct error codes, source spans, and messages.

---

### Phase 6 — Load-Time Verification Pass (Days 50–60)

**Goal:** Violations caught before any user code executes. The runtime library becomes the backstop, not the primary detector.

The load-time pass is a JVM premain agent. It attaches before the main class loads, walks all loaded Ferrum-annotated class files using the ASM library, and performs three analyses:

**Borrow flow analysis.** Forward dataflow over the bytecode of each method. Tracks borrow state at each bytecode instruction. If any instruction would create a conflicting borrow (exclusive while shared exists, or vice versa), emit a load-time error and abort before user code runs.

Implementation: the `@FerrumBorrows` annotation on the method tells the pass which parameters and locals are borrows, their mode, and their region. The pass tracks borrow state through the bytecode's control flow graph via ASM `visitAnnotation`. `BorrowTracker.acquireExclusive` calls in the bytecode identify borrow acquisition points. The pass simulates these calls statically, propagating state through all paths.

**Effect annotation verification.** For each method: read its `@FerrumEffects` annotation (its declared effect set). Walk all `invokevirtual`/`invokestatic` instructions. Look up the `@FerrumEffects` annotation of each callee. If the calling method is pure (empty effects) and any callee has non-empty effects, emit an effect violation error.

This catches 100% of effect violations statically for programs where all methods are visible at load time (the common case). Effect violations that depend on dynamic dispatch fall through to the runtime `EffectStack`.

**Contract gate insertion.** For contracts not yet statically verified (all contracts in MVP, since Boogie is out of scope): verify that the `checkRequires` and `checkEnsures` instrumentation calls are present in the bytecode for every method carrying `@FerrumRequires` / `@FerrumEnsures` annotations (the transpiler put them there in Phase 4). If any annotated method is missing its instrumentation (e.g., due to a transpiler bug), insert it now via ASM. This makes the load-time pass a correctness backstop for transpiler mistakes.

**Abort on violation.** If the load-time pass finds any violation, it constructs a `FerrumDiagnostic` and writes it to stderr in the standard error format, then calls `System.exit(1)`. No user code runs.

**Gate:**
- All 6 fault categories from the arXiv paper: inject one instance of each, verify caught before `main()` executes (borrow, effect violations) or before the violating function's body runs (contract violations).
- Effect violations caught at load time (not runtime) for static dispatch.
- Borrow aliasing violations caught at load time for statically detectable patterns.
- Programs without violations run correctly with zero load-time overhead after the pass completes.

---

### Phase 7 — Fault Injection Suite and Error Quality (Days 61–70)

**Goal:** The system is demonstrably correct against a systematic fault suite. Error messages are good enough that the error tells you what to fix.

**Fault injection suite (minimum):**

Build a test harness that runs `ferrum run` on each fault-injected program and checks:
- Exit code is non-zero
- Error code matches expected (e.g., `B001` for exclusive borrow conflict)
- Source span in error message references the correct line

One test per fault category, five variants each = 30 baseline tests. Each test is a small Ferrum program (~20 lines) with one injected fault.

| Category | Error code | Test count |
|---|---|---|
| Precondition violation | C001 | 5 |
| Postcondition violation | C002 | 5 |
| Exclusive borrow conflict | B001 | 5 |
| Shared borrow of exclusive | B002 | 5 |
| Use after region exit | R001 | 5 |
| Pure calls impure | E001 | 5 |

**Error message quality standard:**

Every error must have:
- Correct error code
- Source span pointing to the correct line (not a nearby line, not line 1)
- A human-readable message describing what was violated
- A `help:` line suggesting a fix for the most common cases

Borrow errors are the hardest. For MVP, the minimum acceptable borrow error points to the correct call site and names the place that is already borrowed. It does not need to explain the full borrow graph. That is a 0.2 problem.

**Gate:** All 30 fault injection tests pass (correct error code, correct line, non-zero exit). The 7 benchmark programs from the arXiv paper run correctly. `ferrum run examples/hello.fe` still works.

---

## Complete Dependency List

```toml
# ferrum-transpiler/Cargo.toml
logos = "0.14"
chumsky = "0.9"
serde = { version = "1", features = ["derive"] }
serde_json = "1"   # for serialized contract AST in annotations
ariadne = "0.4"    # diagnostic rendering (colors, spans, arrows)
```

```kotlin
// ferrum-runtime/build.gradle.kts
dependencies {
    implementation("org.ow2.asm:asm:9.6")        // annotation reading in load-time agent
    implementation("org.ow2.asm:asm-util:9.6")
}
```

**External tools required (must be on PATH):**
- JDK 17+
- `kotlinc` (Kotlin compiler — used to compile emitted Kotlin source)
- `java` 17+

---

## The `ferrum` CLI

One shell script wraps the pipeline:

```bash
ferrum run file.fe          # transpile + run
ferrum build file.fe        # transpile to .class files only
ferrum check file.fe        # run load-time pass, report errors, exit
ferrum lex file.fe          # debug: dump token stream
ferrum parse file.fe        # debug: dump AST
ferrum typecheck file.fe    # debug: dump typed AST
```

`ferrum run` does:
1. `ferrum-transpiler file.fe` → emits `.ferrum-out/*.kt`
2. `kotlinc .ferrum-out/*.kt -cp ferrum-runtime.jar -d .ferrum-out/classes/`
3. `java -javaagent:ferrum-agent.jar -cp ferrum-runtime.jar:.ferrum-out/classes MainClass`

The agent runs before `MainClass.main()`, performs the load-time pass, and either aborts with a diagnostic or allows execution to proceed.

---

## Risks and Mitigations

**Risk: `kotlinc` compile step is slow for large emitted outputs.**
Mitigation: For MVP, all programs are small (~hundreds of lines of emitted Kotlin). `kotlinc` daemon mode (`kotlinc -daemon`) eliminates JVM startup overhead on subsequent compilations. Not a concern until programs exceed ~5000 lines of emitted Kotlin.

**Risk: ContractEvaluator cannot evaluate complex contract expressions.**
Mitigation: For MVP, restrict contract expressions to a well-defined subset: arithmetic over integer locals, comparison operators, boolean operators, len(), is_sorted(), is_empty(), contains() on Vec/slice types, and None-checks. Document the restriction. Contracts using unsupported expressions emit a warning and fall back to always-runtime-checked without static analysis. The subset covers the arXiv paper's benchmark contracts.

**Risk: chumsky's error recovery produces confusing partial ASTs.**
Mitigation: Add a "strict mode" (default) that aborts on first parse error. Error recovery is a 0.2 feature. For MVP, parse errors abort with the source span and a clear message. This is acceptable for a developer tool.

**Risk: Load-time pass borrow flow analysis is incomplete for complex control flow.**
Mitigation: The analysis is conservative: if it cannot determine borrow state on a path (e.g., complex loop with break), it marks the borrows as "unknown" and lets the runtime library handle those paths. The runtime is the backstop. This means some violations are caught at runtime instead of load-time, but the detection rate is still 100%.

**Risk: 8 weeks is too aggressive.**
Mitigation: Each phase is independently deployable. At any phase boundary the system is in a valid state: programs run, errors are structured, no silent failures. If week 8 ends with Phase 6 incomplete, the system is still correct (runtime catches everything), just not as ergonomic (some violations caught later than ideal). The fault injection tests from Phase 7 are the correctness gate, not the load-time pass specifically.

---

## Test Strategy

Three levels, all automated:

**Level 1: Unit tests (in Rust and Kotlin)**
- Lexer: round-trip property tests (lex-then-reprint)
- Parser: parse each language spec example, compare AST structure
- Type inference: unification unit tests against known cases
- ContractEvaluator: evaluate contracts against known-good/bad inputs
- BorrowTracker: concurrent borrow acquisition tests

**Level 2: Compile-fail tests**
- One `.fe` file per expected error
- `ferrum check` must exit non-zero with the expected error code
- Source span in error must reference the correct line
- Run via: `ferrum check --expected-error=C001 examples/bad_contract.fe`

**Level 3: Integration tests (the 7 benchmark programs)**
- All 7 programs from the arXiv paper
- Each must run correctly on valid input
- Each must catch injected faults (one fault variant per program per fault category)
- Performance: not measured. Correctness only.

The fault injection suite from Phase 7 covers Level 2 and Level 3 together.

---

## Definition of Done

The MVP is done when:

1. `ferrum run examples/hello.fe` prints `hello, world`
2. `ferrum run examples/binary_search.fe` runs correctly on sorted input
3. All 30 fault injection tests pass (6 categories × 5 variants, 0% missed)
4. The 7 benchmark programs from the arXiv paper run correctly
5. Every violation error carries a correct source location
6. No violation produces incorrect output before being caught

Speed of the compiled programs: irrelevant to MVP. Speed of the compiler itself: irrelevant to MVP. These are 0.2 problems.

---

## What 0.2 Adds (Not Your Problem Right Now)

For orientation only — knowing what comes next prevents over-engineering the MVP:

**0.2:** Boogie/Z3 integration. Move contract checks from runtime-only to static-where-possible. Persistent verification cache. Better borrow error messages.

**0.3:** Full region inference. LLVM codegen (native binaries). Complete generics. Package manager.

**0.4:** Move borrow checking from load-time to compile-time. Load-time pass becomes redundant for borrow checking.

**1.0:** Self-hosting. SemanticQuery server implementation.

None of these require redesigning the MVP architecture. Each is additive. The decorated bytecode format and the four-stage pipeline are stable across all versions.

---

*Ferrum MVP. Correct before fast.*
