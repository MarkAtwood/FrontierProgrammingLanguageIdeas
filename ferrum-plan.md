# FERRUM-PLAN.md

**Project:** Ferrum programming language — compiler, standard library, toolchain  
**Implementation language:** Rust
**References:** `spec/ferrum-language-reference.md`, `spec/ferrum-stdlib.md`
**Bead prefix:** `fe-`  
**OpenSpec root:** `openspec/`

---

## Orientation

You are implementing Ferrum from scratch. Read both reference documents
before touching code. Every design decision is recorded there. Do not invent
new semantics; implement what is specified.

---

## Known Hard Problems

This plan is correct and complete as a roadmap. The failure modes are real
and specific. They are documented here so they can be scoped, deferred, or
mitigated — the dangerous problems are the unknown ones, and there are fewer
of those because the design is unusually precise.

### Region inference is a research problem disguised as engineering

Phase 7 (fe-007) targets 90% annotation-free region inference. This is
aspirational. Cyclone tried for years and got 80%. Rust's NLL took a
dedicated research team multiple years and still has edge cases.

**The constraint:** GADTs plus region inference is genuinely unsolved.
Self-referential structs (`pinned` types) plus region inference plus borrow
checking is three interacting systems that have never been jointly designed.

**The mitigation:** The fallback is not failure — it is honest scope
management. Require annotations when inference fails; improve over subsequent
compiler versions. The annotation layer is a deprecation queue, not technical
debt.

### Effects, regions, and the borrow checker are a joint fixpoint

The plan runs them sequentially: type inference → effect inference →
region inference → borrow checker. The reality is mutual dependencies.
Running these systems sequentially produces incorrect results for programs
where the interaction matters.

**The mitigation:** Design the type checker as a fixpoint from the start,
even if the first implementation is sequential with known incompleteness.
Document which interaction cases are not handled in v1. File them as beads.

### The stdlib has a circular dependency problem

The async runtime requires io traits. The io traits include AsyncRead and
AsyncWrite, which require the async runtime. When you sit down to write
`std.io` at Phase 19, the compiler will not yet have async (Phase 25),
sync primitives (Phase 26), or collections (Phase 23).

**The mitigation:** Define a "bootstrap stdlib" subset — the parts of core
and alloc that can be implemented without async or collections. Implement
the bootstrap stdlib first. Use it to implement everything else.

### The proof system is a dissertation, not a phase

Phase 31 (fe-031) has seven subtasks. What is actually required: implementing
a core dependent type theory for `Prop` types, totality checking for recursion,
Z3 discharge (which fails on nonlinear arithmetic and heap state), and
behavioral equivalence for `verified_by` (one of the hardest problems in
formal verification).

**The mitigation:** The v1 proof system is "contracts verified by SMT where
possible, tested where not, with an explicit audit trail of what was tested
vs proved." Do not let perfect be the enemy of shipped.

### Error message quality will kill bootstrapping

When type inference fails with "cannot unify ?T_42 with ?U_17", bootstrapping
grinds to a halt. Region inference errors are worse — you have an
unsatisfiable constraint graph and need to explain in English which borrow
conflicts with which.

**The mitigation:** Error message quality needs its own epic, not scattered
acceptance criteria. The bootstrap developer experience depends entirely on
this.

---

## The Annotation Layer as Deprecation Queue

Every `trusted` annotation in a Ferrum codebase is the compiler's TODO list.

When a previously-required annotation becomes redundant:

```
warning: annotation no longer needed
  --> src/parser.rs:47:5
   |
47 |     trusted("ptr valid for len bytes, aligned")
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |     region inference now verifies this automatically
   |     since ferrum 0.8
   |
   = note: `ferrum fix --remove-redundant` to clean up automatically
```

The annotation layer is not technical debt. It is a living record of where
the state of the art currently ends. As the compiler improves, annotations
become redundant and emit warnings. This is how Rust's lifetime elision
evolved from explicit to inferred — and Ferrum formalizes the cycle.

---

## SemanticQuery

The compiler is not a batch processor. It is a live reasoning engine that
tools can query, instrument, and inject hypotheses into — without
modifying the source.

SemanticQuery is a protocol (specified in `ferrum-semanticquery.md`) that
exposes four operations:

**Point Query:** Ask the compiler what it knows at any program point — type,
effects, regions, invariants, active contracts, borrow checker state.

**Counterfactual Query:** Inject a proposed invariant and ask whether it
closes a verification gap — without writing it into the source.

**Monitor:** Place semantically-aware traps in the binary — conditions
expressed in Ferrum's type language that the compiler weaves in correctly.

**Scaffolding:** Generate test scaffolding from what the compiler already
knows — every `requires` clause is a test precondition, every `ensures`
clause is a test oracle.

SemanticQuery is specified as LSP extension methods and can also be spoken
standalone. The SMT-LIB export operation exposes the compiler's verification
conditions for use with external solvers. See `ferrum-semanticquery.md`.

Your execution loop is the standard Beads agent cycle. At the start of every
session run `bd ready --json`. Claim one task. Implement it to its acceptance
criteria. File discovered issues with `discovered-from` links. Close the task.
Commit with the bead ID in the message. Push. Start the next session.

Tasks marked **[GATE]** are phase gates. No task in the next phase may be
claimed until all gates in the current phase are closed.

---

## Bootstrap (do this once, in order, before claiming any other task)

```bash
# 1. Initialize repository
git init ferrum && cd ferrum

# 2. Initialize Beads
bd init --prefix fe

# 3. Initialize OpenSpec
openspec init

# 4. Seed all epics from this file (see §Beads Epic Graph)
# Run the seed script produced in fe-000.3

# 5. Verify
bd ready --json
```

---

## Repository Structure

```
ferrum/
├── FERRUM-PLAN.md                      ← this file
│
├── spec/                               ← authoritative language specification
│   ├── ferrum-language-reference.md    ← Language Reference 0.2 (primary spec)
│   └── ferrum-stdlib.md                ← Standard Library 0.1
│
│   Every design decision lives in spec/. If behavior is not specified there,
│   do not implement it — file a spec-ambiguity bead instead.
│   These files are read-only during implementation. Changes require a
│   bead labeled spec-change with explicit approval.
│
├── openspec/               ← OpenSpec root (created by openspec init)
├── Cargo.toml              ← workspace root
├── compiler/               ← ferrumc — the compiler binary
│   ├── fe-lexer/
│   ├── fe-parser/
│   ├── fe-resolve/
│   ├── fe-types/
│   ├── fe-effects/
│   ├── fe-regions/
│   ├── fe-borrow/
│   ├── fe-constraints/
│   ├── fe-traits/
│   ├── fe-hir/
│   ├── fe-mir/
│   ├── fe-contracts/
│   ├── fe-mono/
│   ├── fe-codegen-c/
│   ├── fe-codegen-llvm/
│   └── fe-driver/
├── stdlib/                 ← standard library source (in Ferrum)
│   ├── core/
│   ├── alloc/
│   ├── std/
│   └── ...
├── tools/
│   ├── fe-repl/
│   ├── fe-lsp/
│   ├── fe-fmt/
│   ├── fe-pkg/
│   └── fe-doc/
├── tests/
│   ├── compile-pass/       ← programs that must compile
│   ├── compile-fail/       ← programs that must fail with specific errors
│   ├── run-pass/           ← programs that must compile and produce output
│   └── ui/                 ← error message snapshot tests
└── spec-tests/             ← generated from OpenSpec scenarios
```

---

## Implementation Phases

```
Phase 0: Bootstrap         fe-000
Phase 1: Lexer             fe-001
Phase 2: Parser            fe-002
Phase 3: Name Resolution   fe-003
Phase 4: Module System     fe-004
Phase 5: Type Inference    fe-005
Phase 6: Effect Inference  fe-006
Phase 7: Region Inference  fe-007
Phase 8: Borrow Check      fe-008
Phase 9: Constraints       fe-009
Phase 10: Traits           fe-010
Phase 11: HIR              fe-011
Phase 12: MIR + Dataflow   fe-012
Phase 13: Contracts        fe-013
Phase 14: Monomorphization fe-014
Phase 15: C Codegen        fe-015   ← first working compiler
Phase 16: Interpreter      fe-016   ← fast dev/test loop
Phase 17: core stdlib      fe-017
Phase 18: alloc stdlib     fe-018
Phase 19: io/fs stdlib     fe-019
Phase 20: net stdlib       fe-020
Phase 21: http stdlib      fe-021
Phase 22: sys stdlib       fe-022
Phase 23: collections      fe-023
Phase 24: math/linalg/simd fe-024
Phase 25: async runtime    fe-025
Phase 26: sync primitives  fe-026
Phase 27: Package manager  fe-027
Phase 28: LSP server       fe-028
Phase 29: Formatter        fe-029
Phase 30: Test runner      fe-030
Phase 31: Proof system     fe-031
Phase 32: LLVM codegen     fe-032
Phase 33: Numeric tower    fe-033
Phase 34: Binary layout    fe-034
```

---

## Beads Epic Graph

Seed these epics with `bd create` before beginning work. Dependencies use the
`blocks` relationship. Set status `open` for all.

### Phase 0 — Bootstrap

```yaml
id: fe-000
title: "Project bootstrap"
priority: 0
type: epic
description: >
  Repository, workspace, toolchain, CI skeleton, Beads seeding,
  OpenSpec initialization.

subtasks:
  fe-000.1:
    title: "Initialize Cargo workspace with all crate stubs"
    priority: 0
    acceptance: >
      cargo build succeeds with zero warnings on an empty workspace.
      All crate directories exist with stub lib.rs files.

  fe-000.2:
    title: "Create OpenSpec directory structure and seed component specs"
    priority: 0
    acceptance: >
      openspec/ directory matches §OpenSpec Directory Structure below.
      All spec.md files exist with at minimum a Purpose section.
    deps: [fe-000.1]

  fe-000.3:
    title: "Write and run Beads epic seeding script"
    priority: 0
    acceptance: >
      bd list shows all epics fe-001 through fe-034.
      bd stats shows correct dependency graph with no cycles.
    deps: [fe-000.2]

  fe-000.4:
    title: "Configure CI: cargo test, cargo clippy --deny warnings"
    priority: 0
    acceptance: >
      GitHub Actions (or equivalent) pipeline runs on every push.
      Fails on clippy warnings. Runs test suite.
    deps: [fe-000.1]

  fe-000.5:
    title: "Create test harness: compile-pass, compile-fail, run-pass dirs"
    priority: 0
    acceptance: >
      tests/ directory structure exists.
      A script runs all .fe files in compile-pass/ through the compiler
      (stub: just parse and exit 0).
      compile-fail/ files run compiler and match expected error messages.
    deps: [fe-000.1]
```

### Phase 1 — Lexer

```yaml
id: fe-001
title: "Lexer (fe-lexer)"
priority: 0
type: epic
blocks: [fe-002]
description: >
  Full tokenization of Ferrum source. All keywords, operators, literals,
  comments. Error recovery. Source span tracking for error messages.

subtasks:
  fe-001.1:
    title: "Define Token enum with all keyword, operator, literal variants"
    priority: 0
    acceptance: >
      Token enum covers every keyword in §2.5 of the language reference.
      All operators, all literal types. No string-keyed variants.

  fe-001.2:
    title: "Implement Span type: source file, byte offset, length"
    priority: 0
    acceptance: >
      Span::new(file_id, start, end). Span::merge. Span carries into AST.
      Every token has a span.

  fe-001.3:
    title: "Implement Lexer: character-by-character, UTF-8 aware"
    priority: 0
    acceptance: >
      All compile-pass/lexer/*.fe files lex without error.
      All compile-fail/lexer/*.fe files produce the expected LexError.
      Unicode identifiers lex correctly.
      Nested block comments /* /* */ */ lex correctly.
    deps: [fe-001.1, fe-001.2]

  fe-001.4:
    title: "Integer literal lexing: all bases, all suffixes, underscores"
    priority: 0
    acceptance: >
      0xFF_FFu32, 0b1010i64, 0o77usize all lex to correct token+value.
      All suffixes from §2.6 produce correctly typed literals.
      42u256, 42i1024 lex to BigIntLiteral with correct type suffix.
    deps: [fe-001.3]

  fe-001.5:
    title: "Float literal lexing: f16, bf16, f32, f64, f80, f128, f256, d32, d64, d128"
    priority: 0
    acceptance: >
      3.14f128 lexes to FloatLiteral { value: "3.14", suffix: F128 }.
      0.1d64 lexes to DecimalLiteral { value: "0.1", suffix: D64 }.
      Decimal literals store the original decimal string, not a float64.
    deps: [fe-001.3]

  fe-001.6:
    title: "String, byte string, raw string, character literal lexing"
    priority: 0
    acceptance: >
      r#"..."# lexes correctly for all nesting depths.
      b"..." produces ByteString token.
      All escape sequences in §2.6 lex correctly.
      Invalid escape sequences produce LexError::InvalidEscape with span.
    deps: [fe-001.3]

  fe-001.7:
    title: "Doc comment lexing: ///, //!"
    priority: 0
    acceptance: >
      /// and //! comments produce DocComment tokens with content.
      Non-doc comments are stripped (not passed to parser).
    deps: [fe-001.3]

  fe-001.8:
    title: "[GATE] Lexer spec-tests all pass"
    priority: 0
    acceptance: >
      spec-tests/lexer/ all pass.
      No clippy warnings in fe-lexer crate.
      fe-lexer has >90% line coverage.
    deps: [fe-001.4, fe-001.5, fe-001.6, fe-001.7]
```

### Phase 2 — Parser

```yaml
id: fe-002
title: "Parser and AST (fe-parser)"
priority: 0
type: epic
blocks: [fe-003, fe-004]
deps: [fe-001]
description: >
  Recursive descent parser. Produces typed AST. Error recovery via
  synchronization points. All language constructs from §9-§12.

subtasks:
  fe-002.1:
    title: "Define AST node types for all items (fn, type, enum, trait, impl, layout)"
    priority: 0

  fe-002.2:
    title: "Define AST node types for all expressions (§9.4-§9.8)"
    priority: 0
    deps: [fe-002.1]

  fe-002.3:
    title: "Define AST node types for all patterns (§10)"
    priority: 0
    deps: [fe-002.1]

  fe-002.4:
    title: "Define AST node types for all types (§3, §12)"
    priority: 0
    deps: [fe-002.1]

  fe-002.5:
    title: "Implement parser framework: token stream, peek, expect, error recovery"
    priority: 0
    acceptance: >
      Error recovery syncs on { } ; fn type enum impl.
      Parser never panics on any input.
      All errors carry a span.
    deps: [fe-002.1]

  fe-002.6:
    title: "Parse all item forms: fn, type, enum, trait, impl, layout, import, const, static"
    priority: 1
    deps: [fe-002.5]

  fe-002.7:
    title: "Parse all expression forms including select, scope, with, region"
    priority: 1
    deps: [fe-002.5, fe-002.2]

  fe-002.8:
    title: "Parse all pattern forms including slice patterns, or-patterns, @ bindings"
    priority: 1
    deps: [fe-002.5, fe-002.3]

  fe-002.9:
    title: "Parse all type forms including effect types (! IO + Net), given clauses"
    priority: 1
    deps: [fe-002.5, fe-002.4]

  fe-002.10:
    title: "Parse contract annotations: requires, ensures, invariant, old()"
    priority: 1
    deps: [fe-002.6]

  fe-002.11:
    title: "Parse layout declarations: all placement forms, field annotations"
    priority: 1
    deps: [fe-002.6]

  fe-002.12:
    title: "Parse safety-level prefixes: unchecked, trusted(\"...\"), extern, unsafe"
    priority: 1
    deps: [fe-002.6]

  fe-002.13:
    title: "Parse generic parameters: [T: Bound], const generics, where clauses"
    priority: 1
    deps: [fe-002.6]

  fe-002.14:
    title: "Parse proof functions: proof fn, proof obligation bodies"
    priority: 2
    deps: [fe-002.6]

  fe-002.15:
    title: "[GATE] Parser round-trip test: parse → pretty-print → re-parse produces identical AST"
    priority: 0
    deps: [fe-002.6, fe-002.7, fe-002.8, fe-002.9, fe-002.10,
           fe-002.11, fe-002.12, fe-002.13]
```

### Phase 3 — Name Resolution

```yaml
id: fe-003
title: "Name resolution (fe-resolve)"
priority: 0
type: epic
blocks: [fe-005]
deps: [fe-002]

subtasks:
  fe-003.1:
    title: "Build per-module symbol table: items, imports, re-exports"
    priority: 0

  fe-003.2:
    title: "Resolve use/import declarations including glob imports"
    priority: 0
    deps: [fe-003.1]

  fe-003.3:
    title: "Resolve all identifier references to their definition spans"
    priority: 0
    deps: [fe-003.2]

  fe-003.4:
    title: "Detect undefined names, ambiguous names, visibility violations"
    priority: 0
    deps: [fe-003.3]

  fe-003.5:
    title: "Resolve type aliases (non-recursive)"
    priority: 0
    deps: [fe-003.3]

  fe-003.6:
    title: "[GATE] Name resolution spec-tests pass"
    priority: 0
    deps: [fe-003.1, fe-003.2, fe-003.3, fe-003.4, fe-003.5]
```

### Phase 4 — Module System

```yaml
id: fe-004
title: "Module system (fe-resolve, module loading)"
priority: 0
type: epic
blocks: [fe-003]
deps: [fe-002]

subtasks:
  fe-004.1:
    title: "Implement file-to-module mapping: one file = one module"
    priority: 0

  fe-004.2:
    title: "Implement pub/pub(crate)/pub(super)/private visibility rules"
    priority: 0
    deps: [fe-004.1]

  fe-004.3:
    title: "Implement prelude injection for all modules"
    priority: 0
    deps: [fe-004.1]

  fe-004.4:
    title: "Implement package manifest parsing (Ferrum.toml)"
    priority: 1
    deps: [fe-004.1]

  fe-004.5:
    title: "[GATE] Multi-file compilation: two modules importing each other's types"
    priority: 0
    deps: [fe-004.1, fe-004.2, fe-004.3]
```

### Phase 5 — Type Inference

```yaml
id: fe-005
title: "Type inference (fe-types)"
priority: 0
type: epic
blocks: [fe-006, fe-007, fe-009, fe-010]
deps: [fe-003]
description: >
  Hindley-Milner extended with Ferrum's type system. Unification, inference
  variables, error reporting. Handles all scalar types, compound types,
  generics with bounds, associated types, GADTs.

subtasks:
  fe-005.1:
    title: "Implement inference variable pool and unification"
    priority: 0

  fe-005.2:
    title: "Infer types for all expression forms"
    priority: 0
    deps: [fe-005.1]

  fe-005.3:
    title: "Infer types for all literal forms including all numeric types"
    priority: 0
    deps: [fe-005.1]

  fe-005.4:
    title: "Infer types for function calls, method calls, field access"
    priority: 0
    deps: [fe-005.2]

  fe-005.5:
    title: "Infer types through if/match/loop with proper join points"
    priority: 0
    deps: [fe-005.2]

  fe-005.6:
    title: "Infer types for closures: capture types, FnOnce/FnMut/Fn selection"
    priority: 0
    deps: [fe-005.2]

  fe-005.7:
    title: "Generic function instantiation: collect constraints, solve"
    priority: 0
    deps: [fe-005.1]

  fe-005.8:
    title: "Associated type projection: resolve Self.Item, C.Alloc, etc."
    priority: 0
    deps: [fe-005.7]

  fe-005.9:
    title: "GADT type checking: variant type specialization"
    priority: 1
    deps: [fe-005.7]

  fe-005.10:
    title: "Error messages: point to the conflicting types with spans"
    priority: 0
    acceptance: >
      Type errors always show both conflicting types and their source locations.
      'expected T, found U' messages never show raw inference variable IDs.
    deps: [fe-005.2]

  fe-005.11:
    title: "[GATE] Type inference spec-tests pass"
    priority: 0
    deps: [fe-005.2, fe-005.3, fe-005.4, fe-005.5, fe-005.6,
           fe-005.7, fe-005.8, fe-005.10]
```

### Phase 6 — Effect Inference

```yaml
id: fe-006
title: "Effect inference (fe-effects)"
priority: 0
type: epic
blocks: [fe-011]
deps: [fe-005]
description: >
  Track IO, Net, Sync, Unsafe, Panic, Alloc[A] effects. Infer in function
  bodies. Verify pure functions have empty effect sets. Effect polymorphism.

subtasks:
  fe-006.1:
    title: "Define Effect lattice: sets of effects, union, subset check"
    priority: 0

  fe-006.2:
    title: "Bottom-up effect inference over typed AST"
    priority: 0
    deps: [fe-006.1]

  fe-006.3:
    title: "Verify pure functions: error if body has any effect"
    priority: 0
    deps: [fe-006.2]

  fe-006.4:
    title: "Effect polymorphism: [eff] type parameters"
    priority: 1
    deps: [fe-006.2]

  fe-006.5:
    title: "Alloc[A] effect: tracks which allocator capability is used"
    priority: 0
    deps: [fe-006.2]

  fe-006.6:
    title: "[GATE] Effect inference spec-tests pass"
    priority: 0
    deps: [fe-006.2, fe-006.3, fe-006.5]
```

### Phase 7 — Region Inference

```yaml
id: fe-007
title: "Region inference (fe-regions)"
priority: 0
type: epic
blocks: [fe-008]
deps: [fe-005]
description: >
  Region variable assignment, constraint generation from AST, constraint
  solving, explicit annotation synthesis for pub functions. Target: 90%
  of practical code requires zero explicit annotations.

subtasks:
  fe-007.1:
    title: "Region variable pool and constraint types"
    priority: 0
    acceptance: >
      Constraint types: Subset(r1, r2), Contains(r, scope), Union(r, r1, r2).

  fe-007.2:
    title: "Assign region variables to all reference types in function signatures"
    priority: 0
    deps: [fe-007.1]

  fe-007.3:
    title: "Generate constraints from function body: assignments, returns, field access"
    priority: 0
    deps: [fe-007.2]

  fe-007.4:
    title: "Constraint solver: worklist, minimal assignment, cycle detection"
    priority: 0
    deps: [fe-007.3]
    acceptance: >
      Solver finds minimal valid region assignment.
      Cycles (self-referential) are detected and reported.
      Unsatisfiable constraints produce an error pointing to the conflicting borrow.

  fe-007.5:
    title: "Annotation synthesis: generate explicit annotations for pub fn boundaries"
    priority: 1
    deps: [fe-007.4]

  fe-007.6:
    title: "Error reporting: explain which input region the return borrows from"
    priority: 0
    deps: [fe-007.4]

  fe-007.7:
    title: "[GATE] Region inference eliminates annotations in 90% of test cases"
    priority: 0
    acceptance: >
      Run region inference over the 100-function benchmark suite.
      At most 10 functions require explicit annotations.
      All explicit annotation requirements are genuinely ambiguous.
    deps: [fe-007.4, fe-007.6]
```

### Phase 8 — Borrow Checker

```yaml
id: fe-008
title: "Borrow checker (fe-borrow)"
priority: 0
type: epic
blocks: [fe-011]
deps: [fe-007]
description: >
  NLL-style borrow checker using MIR (once available) or HIR. Enforce:
  exclusive &mut, no shared+exclusive, no borrow outliving owner,
  no use of moved value, no use of uninitialized value.
  Respect unchecked blocks (aliasing rules suspended).

subtasks:
  fe-008.1:
    title: "Liveness analysis: variable live ranges"
    priority: 0

  fe-008.2:
    title: "Move tracking: detect use-after-move"
    priority: 0
    deps: [fe-008.1]

  fe-008.3:
    title: "Initialization tracking: detect use-before-init, partial struct init"
    priority: 0
    deps: [fe-008.1]

  fe-008.4:
    title: "Aliasing check: &mut exclusivity, no shared+exclusive overlap"
    priority: 0
    deps: [fe-008.1]

  fe-008.5:
    title: "Lifetime validity: borrow cannot outlive referent"
    priority: 0
    deps: [fe-008.1, fe-007.4]

  fe-008.6:
    title: "unchecked block handling: suspend aliasing rules within block"
    priority: 0
    deps: [fe-008.4]

  fe-008.7:
    title: "SyncGuard live-across-await detection"
    priority: 1
    deps: [fe-008.1]

  fe-008.8:
    title: "Error messages: show conflicting borrows with spans and explanation"
    priority: 0
    acceptance: >
      Every borrow error shows: (1) where the conflicting borrow is created,
      (2) where it is used after the second borrow, (3) why they conflict.
      No error mentions internal region variable IDs.
    deps: [fe-008.4]

  fe-008.9:
    title: "[GATE] Borrow checker spec-tests pass"
    priority: 0
    deps: [fe-008.2, fe-008.3, fe-008.4, fe-008.5, fe-008.6, fe-008.8]
```

### Phase 9 — Constraint System

```yaml
id: fe-009
title: "Constraint system and SMT bridge (fe-constraints)"
priority: 0
type: epic
blocks: [fe-013, fe-031]
deps: [fe-005]
description: >
  Integer/float range constraints from 'where' clauses. SMT discharge via Z3.
  Constrained integer types. Sub-byte integer constraints. Float constraints.

subtasks:
  fe-009.1:
    title: "Constraint language AST: ==, !=, <, <=, >, >=, and, or, not, arithmetic"
    priority: 0

  fe-009.2:
    title: "Compile-time constant constraint evaluation"
    priority: 0
    deps: [fe-009.1]

  fe-009.3:
    title: "Runtime constraint assertion generation (debug mode)"
    priority: 0
    deps: [fe-009.1]

  fe-009.4:
    title: "Z3 bindings via z3 crate or raw FFI"
    priority: 0
    acceptance: >
      Can discharge: x < 256 given x: u8. Can fail: x > 300 given x: u8.

  fe-009.5:
    title: "SMT discharge for requires/ensures in proof mode"
    priority: 1
    deps: [fe-009.4]

  fe-009.6:
    title: "[GATE] Constrained integer type spec-tests pass"
    priority: 0
    deps: [fe-009.2, fe-009.3, fe-009.4]
```

### Phase 10 — Trait Resolution

```yaml
id: fe-010
title: "Trait resolution (fe-traits)"
priority: 0
type: epic
blocks: [fe-011]
deps: [fe-005]
description: >
  Coherence, orphan rules, trait object vtable construction, impl selection,
  associated type resolution, blanket impls, provided methods,
  LSP contract inheritance checking.

subtasks:
  fe-010.1:
    title: "Impl collection: gather all impls visible in current scope"
    priority: 0

  fe-010.2:
    title: "Impl selection: choose most-specific impl for a trait obligation"
    priority: 0
    deps: [fe-010.1]

  fe-010.3:
    title: "Coherence checking: detect overlapping impls"
    priority: 0
    deps: [fe-010.1]

  fe-010.4:
    title: "Orphan rule enforcement"
    priority: 0
    deps: [fe-010.1]

  fe-010.5:
    title: "Provided method selection: prefer specific over default"
    priority: 0
    deps: [fe-010.2]

  fe-010.6:
    title: "Trait object vtable construction"
    priority: 1
    deps: [fe-010.2]

  fe-010.7:
    title: "LSP contract inheritance: verify precondition weakening, postcondition strengthening"
    priority: 1
    acceptance: >
      Strengthened precondition in impl emits error citing the trait's precondition.
      Weakened postcondition emits error.
    deps: [fe-010.2]

  fe-010.8:
    title: "[GATE] Trait resolution spec-tests pass"
    priority: 0
    deps: [fe-010.2, fe-010.3, fe-010.4, fe-010.5]
```

### Phase 11 — HIR

```yaml
id: fe-011
title: "High-level IR (fe-hir)"
priority: 0
type: epic
blocks: [fe-012]
deps: [fe-006, fe-008, fe-010]
description: >
  Desugared, type-annotated IR. Removes syntactic sugar, resolves all names
  to DefIds, records types on every node. This is the IR that MIR is
  lowered from and that codegen ultimately consumes for high-level operations.

subtasks:
  fe-011.1:
    title: "Define HIR node types with type annotations on every expression"
    priority: 0

  fe-011.2:
    title: "Lower typed AST to HIR: desugar for/while/if-let/while-let"
    priority: 0
    deps: [fe-011.1]

  fe-011.3:
    title: "Lower closures to explicit capture structs + call impls"
    priority: 0
    deps: [fe-011.2]

  fe-011.4:
    title: "Lower pattern matching to decision trees"
    priority: 0
    deps: [fe-011.2]

  fe-011.5:
    title: "Exhaustiveness checking on match arms"
    priority: 0
    deps: [fe-011.4]

  fe-011.6:
    title: "Lower ? operator to explicit match + return"
    priority: 0
    deps: [fe-011.2]

  fe-011.7:
    title: "[GATE] HIR is well-typed: type-check HIR in a second pass to verify"
    priority: 0
    deps: [fe-011.2, fe-011.3, fe-011.4, fe-011.5, fe-011.6]
```

### Phase 12 — MIR and Dataflow

```yaml
id: fe-012
title: "MIR and dataflow analysis (fe-mir)"
priority: 0
type: epic
blocks: [fe-013, fe-014, fe-015]
deps: [fe-011]
description: >
  3-address code IR in SSA-adjacent form. Basic blocks, explicit control
  flow. Dataflow analyses: initialization, liveness, alias sets.

subtasks:
  fe-012.1:
    title: "Define MIR: BasicBlock, Statement, Terminator, Place, Rvalue"
    priority: 0

  fe-012.2:
    title: "Lower HIR to MIR"
    priority: 0
    deps: [fe-012.1]

  fe-012.3:
    title: "Dominator tree and control flow graph construction"
    priority: 0
    deps: [fe-012.2]

  fe-012.4:
    title: "Liveness analysis (backwards dataflow)"
    priority: 0
    deps: [fe-012.3]

  fe-012.5:
    title: "Initialization analysis (forwards dataflow)"
    priority: 0
    deps: [fe-012.3]

  fe-012.6:
    title: "Move analysis (forwards dataflow)"
    priority: 0
    deps: [fe-012.3]

  fe-012.7:
    title: "Re-run borrow checker on MIR (more precise than HIR pass)"
    priority: 0
    deps: [fe-012.4, fe-012.5, fe-012.6]

  fe-012.8:
    title: "[GATE] MIR passes verify: all test programs produce valid MIR"
    priority: 0
    deps: [fe-012.2, fe-012.3, fe-012.4, fe-012.5, fe-012.6, fe-012.7]
```

### Phase 13 — Contract Elaboration

```yaml
id: fe-013
title: "Contract elaboration (fe-contracts)"
priority: 1
type: epic
blocks: [fe-015]
deps: [fe-009, fe-012]
description: >
  Insert requires/ensures runtime assertions in debug mode. Generate
  old() snapshots. Insert invariant checks after &mut method returns.
  Generate fuzz mode property test scaffolding.

subtasks:
  fe-013.1:
    title: "requires → assert!() insertion at function entry in debug MIR"
    priority: 1

  fe-013.2:
    title: "ensures → assert!() insertion at all return sites in debug MIR"
    priority: 1
    deps: [fe-013.1]

  fe-013.3:
    title: "old() snapshot variable generation"
    priority: 1
    deps: [fe-013.2]

  fe-013.4:
    title: "Type invariant checking after &mut method bodies"
    priority: 1
    deps: [fe-013.1]

  fe-013.5:
    title: "Contract elision in release mode"
    priority: 1
    deps: [fe-013.1]

  fe-013.6:
    title: "[GATE] Contract elaboration spec-tests pass"
    priority: 1
    deps: [fe-013.1, fe-013.2, fe-013.3, fe-013.4, fe-013.5]
```

### Phase 14 — Monomorphization

```yaml
id: fe-014
title: "Monomorphization (fe-mono)"
priority: 0
type: epic
blocks: [fe-015]
deps: [fe-012]

subtasks:
  fe-014.1:
    title: "Collect all instantiations from call graph"
    priority: 0

  fe-014.2:
    title: "Generate concrete MIR copy per instantiation"
    priority: 0
    deps: [fe-014.1]

  fe-014.3:
    title: "Detect recursive generic instantiation (error)"
    priority: 0
    deps: [fe-014.1]

  fe-014.4:
    title: "dyn Trait vtable generation"
    priority: 1
    deps: [fe-014.2]

  fe-014.5:
    title: "[GATE] Monomorphization spec-tests pass"
    priority: 0
    deps: [fe-014.2, fe-014.3]
```

### Phase 15 — C Codegen (Bootstrap Target)

```yaml
id: fe-015
title: "C codegen bootstrap target (fe-codegen-c)"
priority: 0
type: epic
blocks: [fe-017]
deps: [fe-014, fe-013]
description: >
  Generate C11 output from MIR. Not beautiful C — correct C.
  This is the first working end-to-end compiler.
  No GC. No runtime. Generates #include <stdint.h> and raw struct/function
  definitions. Target: compile ferrum/tests/run-pass/*.fe → C → binary.

subtasks:
  fe-015.1:
    title: "C type mapping: all scalar types to C types"
    priority: 0
    acceptance: >
      u8→uint8_t, i64→int64_t, f32→float, f64→double, bool→_Bool.
      u24/i24/u48/i48 as packed structs or uint32_t with mask.
      u256/u512/u1024 as struct { uint64_t limbs[N]; }.
      f128 as __float128 on gcc, long double on msvc/clang fallback.

  fe-015.2:
    title: "C struct/enum generation from Ferrum type definitions"
    priority: 0
    deps: [fe-015.1]

  fe-015.3:
    title: "C function generation from monomorphized MIR"
    priority: 0
    deps: [fe-015.2]

  fe-015.4:
    title: "C expression generation: arithmetic, comparison, logical, bitwise"
    priority: 0
    deps: [fe-015.3]

  fe-015.5:
    title: "C control flow: if, while, goto (for loop lowering)"
    priority: 0
    deps: [fe-015.3]

  fe-015.6:
    title: "Allocator lowering: ambient allocator calls to concrete alloc/free"
    priority: 0
    deps: [fe-015.3]

  fe-015.7:
    title: "Destructor insertion: call drop() at scope exit"
    priority: 0
    deps: [fe-015.3]

  fe-015.8:
    title: "volatile read/write lowering for MMIO"
    priority: 1
    deps: [fe-015.4]

  fe-015.9:
    title: "[GATE] run-pass test suite: all programs compile and produce correct output"
    priority: 0
    acceptance: >
      Every .fe file in tests/run-pass/ compiles to C, compiles to binary,
      and produces the expected output defined in the .expected file.
      This gate marks the first working end-to-end Ferrum compiler.
    deps: [fe-015.3, fe-015.4, fe-015.5, fe-015.6, fe-015.7]
```

### Phase 16 — Tree-walking Interpreter

```yaml
id: fe-016
title: "Tree-walking interpreter (fe-repl)"
priority: 1
type: epic
deps: [fe-011]
description: >
  Evaluate HIR directly. No codegen. Used for fast iteration, REPL,
  const fn evaluation, and proof-mode evaluation.
  Does not need to be fast — needs to be correct and complete.

subtasks:
  fe-016.1:
    title: "Interpreter value representation: all Ferrum types as Rust enums"
    priority: 1

  fe-016.2:
    title: "Evaluate all expression forms"
    priority: 1
    deps: [fe-016.1]

  fe-016.3:
    title: "Evaluate all control flow forms"
    priority: 1
    deps: [fe-016.2]

  fe-016.4:
    title: "Interpret standard library primitives via Rust implementations"
    priority: 1
    deps: [fe-016.2]

  fe-016.5:
    title: "REPL: read-eval-print loop with line editing"
    priority: 2
    deps: [fe-016.2]

  fe-016.6:
    title: "[GATE] run-pass suite passes via interpreter (same tests as fe-015.9)"
    priority: 1
    deps: [fe-016.2, fe-016.3, fe-016.4]
```

### Phase 17 — core stdlib

```yaml
id: fe-017
title: "core standard library"
priority: 0
type: epic
blocks: [fe-018]
deps: [fe-015]
description: >
  Implement core in Ferrum (compiled by the bootstrap compiler).
  All types, traits, and functions from ferrum-stdlib.md §3.

subtasks:
  fe-017.1:  { title: "core.ops: all operator traits",               priority: 0 }
  fe-017.2:  { title: "core.cmp: PartialEq, Eq, Ord, PartialOrd",   priority: 0, deps: [fe-017.1] }
  fe-017.3:  { title: "core.option: Option[T] and all methods",      priority: 0, deps: [fe-017.2] }
  fe-017.4:  { title: "core.result: Result[T,E] and all methods",    priority: 0, deps: [fe-017.2] }
  fe-017.5:  { title: "core.iter: Iterator and all combinators",     priority: 0, deps: [fe-017.3, fe-017.4] }
  fe-017.6:  { title: "core.mem: swap, replace, take, MaybeUninit",  priority: 0, deps: [fe-017.1] }
  fe-017.7:  { title: "core.ptr: all raw pointer operations",        priority: 0, deps: [fe-017.6] }
  fe-017.8:  { title: "core.slice: all slice methods",               priority: 0, deps: [fe-017.5] }
  fe-017.9:  { title: "core.str: all &str methods",                  priority: 0, deps: [fe-017.8] }
  fe-017.10: { title: "core.num: all integer and float methods",     priority: 0, deps: [fe-017.2] }
  fe-017.11:
    title: "[GATE] core compiles and passes core test suite"
    priority: 0
    deps: [fe-017.1, fe-017.2, fe-017.3, fe-017.4, fe-017.5,
           fe-017.6, fe-017.7, fe-017.8, fe-017.9, fe-017.10]
```

### Phase 18 — alloc stdlib

```yaml
id: fe-018
title: "alloc standard library"
priority: 0
type: epic
blocks: [fe-019]
deps: [fe-017]

subtasks:
  fe-018.1:  { title: "Allocator trait + Heap, Arena, Pool, Stack, Bump, Null impls", priority: 0 }
  fe-018.2:  { title: "Box[T]",                     priority: 0, deps: [fe-018.1] }
  fe-018.3:  { title: "Vec[T]",                     priority: 0, deps: [fe-018.1] }
  fe-018.4:  { title: "String",                     priority: 0, deps: [fe-018.3] }
  fe-018.5:  { title: "Rc[T] and Arc[T]",           priority: 0, deps: [fe-018.2] }
  fe-018.6:  { title: "fmt: Display, Debug, Formatter, format!()", priority: 0, deps: [fe-018.4] }
  fe-018.7:  { title: "[GATE] alloc test suite passes", priority: 0,
               deps: [fe-018.1, fe-018.2, fe-018.3, fe-018.4, fe-018.5, fe-018.6] }
```

### Phase 19 — io/fs

```yaml
id: fe-019
title: "io and fs standard library"
priority: 0
type: epic
blocks: [fe-020]
deps: [fe-018]

subtasks:
  fe-019.1:  { title: "io traits: Read, Write, Seek, BufRead",      priority: 0 }
  fe-019.2:  { title: "IoError, IoErrorKind",                        priority: 0 }
  fe-019.3:  { title: "BufReader, BufWriter, Cursor, Sink, Empty",  priority: 0, deps: [fe-019.1] }
  fe-019.4:  { title: "stdin(), stdout(), stderr() with lock()",     priority: 0, deps: [fe-019.1] }
  fe-019.5:  { title: "binary: BinaryRead, BinaryWrite, BitReader",  priority: 0, deps: [fe-019.1] }
  fe-019.6:  { title: "text: TextReader, TextWriter, Utf8/Latin1",  priority: 0, deps: [fe-019.1] }
  fe-019.7:  { title: "fs: File, OpenOptions, Path, PathBuf",        priority: 0, deps: [fe-019.1] }
  fe-019.8:  { title: "fs: read_dir, create_dir_all, remove, rename, copy", priority: 0, deps: [fe-019.7] }
  fe-019.9:  { title: "TempFile, TempDir",                           priority: 1, deps: [fe-019.7] }
  fe-019.10: { title: "[GATE] io/fs test suite passes",              priority: 0,
               deps: [fe-019.1, fe-019.2, fe-019.3, fe-019.4, fe-019.5,
                      fe-019.6, fe-019.7, fe-019.8] }
```

### Phase 20 — net

```yaml
id: fe-020
title: "net standard library"
priority: 1
type: epic
blocks: [fe-021]
deps: [fe-019]

subtasks:
  fe-020.1: { title: "IpAddr, Ipv4Addr, Ipv6Addr, SocketAddr, Port",  priority: 1 }
  fe-020.2: { title: "TcpListener, TcpStream",                         priority: 1, deps: [fe-020.1] }
  fe-020.3: { title: "UdpSocket",                                      priority: 1, deps: [fe-020.1] }
  fe-020.4: { title: "DNS: Resolver, LookupIp",                        priority: 1, deps: [fe-020.1] }
  fe-020.5: { title: "[GATE] TCP echo server test passes",             priority: 1,
              deps: [fe-020.1, fe-020.2] }
```

### Phase 21 — http

```yaml
id: fe-021
title: "http standard library"
priority: 1
type: epic
deps: [fe-020, fe-025]

subtasks:
  fe-021.1: { title: "Method, StatusCode, Version, HeaderName, HeaderValue, HeaderMap", priority: 1 }
  fe-021.2: { title: "Uri: parsing, components, query_pairs",           priority: 1, deps: [fe-021.1] }
  fe-021.3: { title: "Request, Response, Body, RequestBuilder",         priority: 1, deps: [fe-021.1] }
  fe-021.4: { title: "HTTP/1.1 client: connection pooling, redirects",  priority: 1, deps: [fe-021.3] }
  fe-021.5: { title: "HTTP/1.1 server: Server, Router, middleware chain", priority: 1, deps: [fe-021.3] }
  fe-021.6: { title: "WebSocket upgrade and frame codec",               priority: 2, deps: [fe-021.5] }
  fe-021.7: { title: "[GATE] HTTP hello-world server test passes",      priority: 1,
              deps: [fe-021.4, fe-021.5] }
```

### Phase 22 — sys

```yaml
id: fe-022
title: "sys standard library (posix, wasi, windows)"
priority: 1
type: epic
deps: [fe-019]

subtasks:
  fe-022.1: { title: "sys cross-platform: os_name, arch, pid, exit, current_dir", priority: 1 }
  fe-022.2: { title: "sys.posix: Fd type, open/read/write/close/lseek",           priority: 1 }
  fe-022.3: { title: "sys.posix: signal → channel conversion, no raw handlers",   priority: 1 }
  fe-022.4: { title: "sys.posix: mmap/munmap as MmapGuard",                       priority: 2 }
  fe-022.5: { title: "sys.posix: fork/exec/waitpid",                              priority: 2 }
  fe-022.6: { title: "sys.wasi: preopened dirs, clock, random, http",             priority: 2 }
  fe-022.7: { title: "sys.mmio: MmioReg[T, ADDR] typed register access",          priority: 2 }
  fe-022.8: { title: "[GATE] POSIX file ops test passes on Linux",                priority: 1,
              deps: [fe-022.1, fe-022.2] }
```

### Phase 23 — collections

```yaml
id: fe-023
title: "collections standard library"
priority: 1
type: epic
deps: [fe-018]

subtasks:
  fe-023.1: { title: "HashMap[K,V] with Entry API",            priority: 1 }
  fe-023.2: { title: "HashSet[T]",                              priority: 1, deps: [fe-023.1] }
  fe-023.3: { title: "BTreeMap[K,V] and BTreeSet[T]",          priority: 1 }
  fe-023.4: { title: "VecDeque[T]",                             priority: 1 }
  fe-023.5: { title: "BinaryHeap[T] + Reverse[T]",             priority: 1 }
  fe-023.6: { title: "LinkedList[T] with cursor API",           priority: 2 }
  fe-023.7: { title: "RingBuf[T, N] (stack allocated)",         priority: 2 }
  fe-023.8: { title: "SmallVec[T, N]",                          priority: 2 }
  fe-023.9: { title: "[GATE] collections test suite passes",   priority: 1,
              deps: [fe-023.1, fe-023.2, fe-023.3, fe-023.4, fe-023.5] }
```

### Phase 24 — math/linalg/simd

```yaml
id: fe-024
title: "math, linalg, simd standard library"
priority: 2
type: epic
deps: [fe-018]

subtasks:
  fe-024.1: { title: "math: all f32/f64 methods (floor, sqrt, sin, etc.)",        priority: 2 }
  fe-024.2: { title: "math: all integer methods (count_ones, ilog2, etc.)",       priority: 2 }
  fe-024.3: { title: "math.stats: mean, variance, RunningStats",                  priority: 2 }
  fe-024.4: { title: "linalg: Vector[T,N], Matrix[T,R,C] fixed-size",            priority: 2 }
  fe-024.5: { title: "linalg: Mat4 transforms, Quat",                            priority: 2, deps: [fe-024.4] }
  fe-024.6: { title: "linalg: DMatrix[T], DVector[T] dynamic",                   priority: 2, deps: [fe-024.4] }
  fe-024.7: { title: "simd: Simd[T,N] portable SIMD, Mask[T,N]",                 priority: 2 }
  fe-024.8: { title: "math.blas: axpy, dot, gemv, gemm",                         priority: 2, deps: [fe-024.6, fe-024.7] }
  fe-024.9: { title: "[GATE] math/linalg/simd test suite passes",                priority: 2,
              deps: [fe-024.1, fe-024.2, fe-024.4, fe-024.7] }
```

### Phase 25 — async runtime

```yaml
id: fe-025
title: "async runtime (async event loop)"
priority: 1
type: epic
blocks: [fe-021]
deps: [fe-026]

subtasks:
  fe-025.1: { title: "Future trait, Poll[T], Context, Waker",                    priority: 1 }
  fe-025.2: { title: "Runtime: thread pool, spawn, block_on",                    priority: 1, deps: [fe-025.1] }
  fe-025.3: { title: "AsyncRead, AsyncWrite, AsyncBufRead traits",               priority: 1, deps: [fe-025.1] }
  fe-025.4: { title: "sleep(), timeout(), interval()",                           priority: 1, deps: [fe-025.2] }
  fe-025.5: { title: "async channel: AsyncSender, AsyncReceiver",               priority: 1, deps: [fe-025.2] }
  fe-025.6: { title: "select2, join2, join_all, try_join2",                     priority: 1, deps: [fe-025.2] }
  fe-025.7: { title: "[GATE] async echo server handles 1000 concurrent connections", priority: 1,
              deps: [fe-025.2, fe-025.3, fe-025.4, fe-025.5] }
```

### Phase 26 — sync primitives

```yaml
id: fe-026
title: "sync primitives standard library"
priority: 0
type: epic
deps: [fe-018]

subtasks:
  fe-026.1: { title: "Atomic[T] for all AtomicSafe types, all orderings",        priority: 0 }
  fe-026.2: { title: "fence(), compiler_fence()",                                priority: 0 }
  fe-026.3: { title: "Mutex[T], MutexGuard, poison handling",                   priority: 0 }
  fe-026.4: { title: "RwLock[T], RwLockReadGuard, RwLockWriteGuard",            priority: 0 }
  fe-026.5: { title: "Condvar, Barrier, Semaphore",                              priority: 1 }
  fe-026.6: { title: "Once, LazyLock[T], OnceLock[T]",                          priority: 1 }
  fe-026.7: { title: "[GATE] sync test suite: no data races under TSan",        priority: 0,
              deps: [fe-026.1, fe-026.2, fe-026.3, fe-026.4] }
```

### Phase 27 — Package Manager

```yaml
id: fe-027
title: "Package manager and build system (fe-pkg)"
priority: 2
type: epic
deps: [fe-015]

subtasks:
  fe-027.1: { title: "Ferrum.toml manifest parsing",                             priority: 2 }
  fe-027.2: { title: "ferrum.lock lockfile: content-hash-addressed",             priority: 2 }
  fe-027.3: { title: "Dependency resolution: semver, registry, git URL",         priority: 2 }
  fe-027.4: { title: "Registry client: fetch by content hash, verify",           priority: 2 }
  fe-027.5: { title: "build command: resolve deps, compile, link",               priority: 2 }
  fe-027.6: { title: "test command: run tests/",                                 priority: 2 }
  fe-027.7: { title: "audit command: report by safety level",                    priority: 2 }
  fe-027.8: { title: "[GATE] ferrum build compiles itself (bootstrapping test)", priority: 2,
              deps: [fe-027.1, fe-027.2, fe-027.3, fe-027.5] }
```

### Phase 28 — LSP

```yaml
id: fe-028
title: "LSP server (fe-lsp)"
priority: 2
type: epic
deps: [fe-010]

subtasks:
  fe-028.1: { title: "LSP transport: JSON-RPC over stdin/stdout",                priority: 2 }
  fe-028.2: { title: "textDocument/hover: type at cursor",                       priority: 2 }
  fe-028.3: { title: "textDocument/completion: identifier + method completion",  priority: 2 }
  fe-028.4: { title: "textDocument/definition: go to definition",                priority: 2 }
  fe-028.5: { title: "textDocument/diagnostics: errors and warnings",            priority: 2 }
  fe-028.6: { title: "textDocument/inlayHints: inferred types and effects",      priority: 2 }
```

### Phase 29 — Formatter

```yaml
id: fe-029
title: "Formatter (fe-fmt)"
priority: 2
type: epic
deps: [fe-002]

subtasks:
  fe-029.1: { title: "Pretty-printer: produce canonical formatting from AST",    priority: 2 }
  fe-029.2: { title: "Idempotency: fmt(fmt(x)) == fmt(x) for all inputs",       priority: 2 }
  fe-029.3: { title: "[GATE] Formatter idempotency holds on all run-pass tests", priority: 2,
              deps: [fe-029.1, fe-029.2] }
```

### Phase 30 — Test Runner

```yaml
id: fe-030
title: "Test runner (fe-test)"
priority: 1
type: epic
deps: [fe-015]

subtasks:
  fe-030.1: { title: "#[test] attribute: collect and run test functions",         priority: 1 }
  fe-030.2: { title: "#[test, should_panic] support",                            priority: 1 }
  fe-030.3: { title: "Result-returning tests: fn test() -> Result",             priority: 1 }
  fe-030.4: { title: "#[bench] benchmarking support",                           priority: 2 }
  fe-030.5: { title: "#[property_test] property-based tests via contracts",      priority: 2 }
  fe-030.6: { title: "assert_eq!, assert_ne!, assert_matches!, assert_approx_eq!", priority: 1 }
  fe-030.7: { title: "[GATE] ferrum test runs the stdlib test suites",          priority: 1,
              deps: [fe-030.1, fe-030.2, fe-030.3, fe-030.6] }
```

### Phase 31 — Proof System

```yaml
id: fe-031
title: "Proof system and SMT bridge (fe-proof)"
priority: 2
type: epic
deps: [fe-009, fe-013]

subtasks:
  fe-031.1: { title: "Prop universe: compile-time-only types",                   priority: 2 }
  fe-031.2: { title: "Proof function checker: totality, purity",                 priority: 2 }
  fe-031.3: { title: "Structural recursion termination checker",                 priority: 2 }
  fe-031.4: { title: "SMT discharge of proof obligations via Z3",                priority: 2, deps: [fe-009.4] }
  fe-031.5: { title: "Proof term erasure before codegen",                        priority: 2 }
  fe-031.6: { title: "verified_by: link proof to fast implementation",           priority: 2 }
  fe-031.7: { title: "[GATE] bubble_sort proven correct via proof fn",          priority: 2,
              deps: [fe-031.1, fe-031.2, fe-031.3, fe-031.4, fe-031.5] }
```

### Phase 32 — LLVM Codegen

```yaml
id: fe-032
title: "LLVM codegen (fe-codegen-llvm)"
priority: 2
type: epic
deps: [fe-014]
description: >
  Production backend via inkwell (safe Rust LLVM bindings).
  Replaces C codegen for optimized output.

subtasks:
  fe-032.1: { title: "LLVM IR type mapping from Ferrum types",                   priority: 2 }
  fe-032.2: { title: "LLVM IR function generation from MIR",                    priority: 2 }
  fe-032.3: { title: "LLVM intrinsics: volatile, atomic, fence, SIMD",         priority: 2 }
  fe-032.4: { title: "Optimization passes: O0/O1/O2/O3/Os",                    priority: 2 }
  fe-032.5: { title: "Target triple support: x86-64, aarch64, riscv64, wasm32", priority: 2 }
  fe-032.6: { title: "[GATE] run-pass suite passes via LLVM codegen",          priority: 2,
              deps: [fe-032.1, fe-032.2, fe-032.3, fe-032.4] }
```

### Phase 33 — Extended Numeric Types

```yaml
id: fe-033
title: "Extended numeric type system"
priority: 1
type: epic
deps: [fe-005, fe-015]
description: >
  Full numeric tower as specified in ferrum-language-reference.md §3.2.
  Non-power-of-two integers, large fixed-width integers, f16/bf16/f80/f128/f256,
  d32/d64/d128 decimal floats, Fixed[I,F] fixed-point, BigInt/BigDecimal/Rational.

subtasks:
  fe-033.1: { title: "u24/i24/u48/i48/u96/i96 types: storage, arithmetic, layout", priority: 1 }
  fe-033.2: { title: "u256/u512/u1024/i256/i512/i1024: multi-limb arithmetic",    priority: 1 }
  fe-033.3: { title: "f16 type: IEEE 754 binary16, hardware or soft-float",        priority: 1 }
  fe-033.4: { title: "bf16 type: brain float, truncated f32",                      priority: 1 }
  fe-033.5: { title: "f80 type: x87 extended precision, x86-only hardware",        priority: 2 }
  fe-033.6: { title: "f128 type: IEEE quad precision, soft-float default",         priority: 1 }
  fe-033.7: { title: "f256 type: octuple precision, always soft-float",            priority: 2 }
  fe-033.8: { title: "d32/d64/d128: IEEE 754-2008 decimal float",                 priority: 1 }
  fe-033.9: { title: "Fixed[I,F] / UFixed[I,F]: fixed-point arithmetic",           priority: 1 }
  fe-033.10: { title: "BigInt, BigUint: arbitrary precision integers",             priority: 2 }
  fe-033.11: { title: "BigDecimal: arbitrary precision decimal",                   priority: 2 }
  fe-033.12: { title: "Rational: exact rational arithmetic",                       priority: 2 }
  fe-033.13: { title: "Complex[T]: complex numbers over any numeric type",         priority: 2 }
  fe-033.14: { title: "Numeric trait hierarchy: Numeric, Integer, Float, etc.",    priority: 1 }
  fe-033.15: { title: "#[no_soft_float] and #[require_hardware_float] attributes", priority: 2 }
  fe-033.16: { title: "[GATE] numeric tower spec-tests pass",                     priority: 1,
               deps: [fe-033.1, fe-033.2, fe-033.3, fe-033.4,
                      fe-033.6, fe-033.8, fe-033.9, fe-033.14] }
```

### Phase 34 — Binary Layout

```yaml
id: fe-034
title: "Binary layout declarations (fe-codegen-c, language reference §15)"
priority: 1
type: epic
deps: [fe-012, fe-015]
description: >
  Compiler verification and code generation for 'layout' declarations.
  Generates read/write/view codecs. Integrates with the C and LLVM backends.

subtasks:
  fe-034.1: { title: "Layout declaration parsing and storage in HIR",            priority: 1 }
  fe-034.2: { title: "Layout verification: no overlaps, no gaps, total_size matches", priority: 1 }
  fe-034.3: { title: "Layout codec generation: read() from &[u8] → Type",       priority: 1 }
  fe-034.4: { title: "Layout codec generation: write() from Type → [u8; N]",    priority: 1 }
  fe-034.5: { title: "Layout view generation: zero-copy field accessor struct", priority: 1 }
  fe-034.6: { title: "Byte order conversion insertion: big/little/native",       priority: 1 }
  fe-034.7: { title: "Multi-layout conversion: Type.convert[OtherLayout]()",    priority: 2 }
  fe-034.8: { title: "[GATE] IPv4 header layout test: read/write/view round-trip", priority: 1,
              deps: [fe-034.1, fe-034.2, fe-034.3, fe-034.4, fe-034.5, fe-034.6] }
```

---

## OpenSpec Directory Structure

Create this structure with `openspec init`, then populate each spec file.

```
openspec/
├── AGENTS.md
├── config.yaml
├── specs/
│   ├── lexer/spec.md
│   ├── parser/spec.md
│   ├── name-resolution/spec.md
│   ├── type-inference/spec.md
│   ├── effect-inference/spec.md
│   ├── region-inference/spec.md
│   ├── borrow-checker/spec.md
│   ├── constraint-system/spec.md
│   ├── trait-resolution/spec.md
│   ├── hir/spec.md
│   ├── mir/spec.md
│   ├── contract-elaboration/spec.md
│   ├── monomorphization/spec.md
│   ├── c-codegen/spec.md
│   ├── numeric-types/spec.md
│   ├── binary-layout/spec.md
│   ├── safety-levels/spec.md
│   ├── stdlib-core/spec.md
│   ├── stdlib-alloc/spec.md
│   ├── stdlib-io/spec.md
│   ├── stdlib-net/spec.md
│   ├── stdlib-http/spec.md
│   ├── stdlib-async/spec.md
│   ├── stdlib-sync/spec.md
│   └── proof-system/spec.md
└── changes/
    └── (populated as design evolves)
```

### openspec/config.yaml

```yaml
schema: spec-driven
context: |
  Ferrum is a new systems programming language. All design decisions are
  recorded in:
    - spec/ferrum-language-reference.md (language spec, including full numeric
      tower and concurrency primitive placement rationale)
    - spec/ferrum-stdlib.md (standard library)

  Implementation language: Rust
  Workspace: Cargo workspace at repo root
  First codegen target: C11 (fe-codegen-c)
  Second codegen target: LLVM IR via inkwell (fe-codegen-llvm)

  Do not invent semantics not in the reference documents.
  Consult the spec before implementing anything non-trivial.

rules:
  specs:
    - Every requirement uses SHALL or MUST for mandatory behavior
    - Every scenario uses GIVEN / WHEN / THEN format
    - Scenarios must be directly testable
  tasks:
    - Every task maps to at most one Beads subtask ID
    - Tasks must be completable in one agent session (~10 min)
```

### openspec/AGENTS.md

```markdown
# Ferrum Compiler — Agent Instructions

## Identity
You are implementing the Ferrum programming language compiler, standard
library, and toolchain. You are one of potentially many parallel agent
instances. Use Beads to avoid duplicate work.

## Session protocol
1. Run `bd ready --json` to get your next task
2. Read the OpenSpec spec for the component you are implementing
3. Read the relevant section(s) of the reference documents
4. Implement to the acceptance criteria
5. Run the relevant tests: `cargo test -p <crate>`
6. Run clippy: `cargo clippy -p <crate> -- -D warnings`
7. File discovered issues: `bd create "..." --deps discovered-from:<your-task-id>`
8. Close the task: `bd close <id> --reason "implemented and tested"`
9. Commit: `git commit -m "fe-NNN: <title>"`
10. Push

## Error handling philosophy
- Errors are values. Result everywhere. No panics in library code.
- Every error carries a source span. Users see file:line:col.
- Error messages explain the problem, not the compiler's internal state.

## Test-first on borrow checker and type system components
Write the compile-fail test cases before implementing the check.
The test suite is the specification made executable.

## Performance baseline
The bootstrap compiler (C codegen) does not need to be fast.
It needs to be correct and complete enough to compile the stdlib.
Performance optimization is Phase 32+ work.

## When you discover a spec ambiguity
File a bead with type `bug`, label `spec-ambiguity`, and block on it
until resolution. Do not invent semantics.
```

---

## OpenSpec Behavioral Specifications

The following specs seed the initial `openspec/specs/` files. Each agent
working on a component must read its spec before coding.

---

### openspec/specs/lexer/spec.md

```markdown
# Lexer Specification

## Purpose
Transform Ferrum source text (UTF-8) into a flat token stream with spans.

## Requirements

### Requirement: Token completeness
The lexer SHALL produce a token for every lexical element defined in
§2.5 and §2.6 of the language reference.

### Requirement: Span coverage
Every token SHALL carry a Span containing the source file ID, start byte
offset, and end byte offset. No span shall point outside the source buffer.

### Requirement: Error recovery
The lexer SHALL continue after a lexical error. It SHALL emit a LexError
token and continue from the next valid character. It SHALL never panic.

### Requirement: Decimal literal fidelity
Decimal float literals (d32, d64, d128 suffix) SHALL store the original
decimal string representation verbatim, not a binary float approximation.

#### Scenario: Basic integer literal
- GIVEN source text `42u32`
- WHEN lexed
- THEN produces token IntLiteral { value: 42, suffix: U32, span: 0..5 }

#### Scenario: Decimal literal preserves string
- GIVEN source text `0.1d64`
- WHEN lexed
- THEN produces token DecimalLiteral { text: "0.1", suffix: D64 }
- AND does NOT produce a token containing the binary value 0.1f64

#### Scenario: Nested block comment
- GIVEN source text `/* outer /* inner */ still outer */ code`
- WHEN lexed
- THEN the comment is consumed and `code` produces an Ident token

#### Scenario: Invalid escape sequence
- GIVEN source text `"\q"`
- WHEN lexed
- THEN produces LexError::InvalidEscape at span 1..3
- AND lexing continues after the string
```

---

### openspec/specs/type-inference/spec.md

```markdown
# Type Inference Specification

## Purpose
Assign a concrete type to every expression in a Ferrum program,
inferring type variables via unification.

## Requirements

### Requirement: No implicit coercions
The type inference engine SHALL NOT insert implicit numeric coercions.
A value of type f32 SHALL NOT unify with f64 without an explicit cast.
A value of type i32 SHALL NOT unify with i64 without an explicit cast.

### Requirement: Error quality
Type errors SHALL report the conflicting types with their source locations.
Errors SHALL NOT expose internal inference variable identifiers (e.g., `?T_42`).

### Requirement: Inference completeness
The engine SHALL infer the type of every expression that is
unambiguously determinable from context. Explicit annotations SHALL only
be required when the programmer's intent is genuinely ambiguous.

#### Scenario: Integer literal unification
- GIVEN `let x: u32 = 42`
- WHEN type-checked
- THEN the literal 42 has type u32
- AND no explicit annotation is needed on the literal

#### Scenario: Mismatched types
- GIVEN `let x: f64 = 1.0f32`
- WHEN type-checked
- THEN an error is produced
- AND the error message contains "expected f64" and "found f32"
- AND the error message does not contain any ?T_N variables

#### Scenario: No implicit widening
- GIVEN `fn add(a: f32, b: f64) -> f64 { a + b }`
- WHEN type-checked
- THEN an error is produced on the + expression
- AND the error suggests `a as f64 + b`
```

---

### openspec/specs/region-inference/spec.md

```markdown
# Region Inference Specification

## Purpose
Infer lifetime/region annotations automatically so that the programmer
rarely needs to write explicit annotations.

## Requirements

### Requirement: 90% annotation-free
Region inference SHALL successfully infer regions for at least 90% of
function signatures in a representative benchmark (tests/bench/region/).

### Requirement: Explicit annotations only for genuine ambiguity
An explicit annotation SHALL only be required when the relationship
between input and output regions is genuinely ambiguous — i.e., when a
human reader also cannot determine the relationship without additional
information.

### Requirement: Error explanation
When inference fails, the error SHALL explain which input region the
return value could borrow from and ask the programmer to disambiguate.
It SHALL NOT say "lifetime error" without explaining what borrows conflict.

#### Scenario: Single-input borrow inferred
- GIVEN `fn first(xs: &[T]) -> &T { &xs[0] }`
- WHEN region inference runs
- THEN no explicit annotation is required
- AND return region is inferred as region(xs)

#### Scenario: Multi-input ambiguity requires annotation
- GIVEN `fn pick(a: &str, b: &str, use_a: bool) -> &str { if use_a { a } else { b } }`
- WHEN region inference runs
- THEN an error is produced
- AND the error asks: "cannot determine which input this borrows from: a or b"
- AND the error suggests adding explicit region annotations
```

---

### openspec/specs/borrow-checker/spec.md

```markdown
# Borrow Checker Specification

## Purpose
Enforce memory safety: no use-after-move, no use-before-init, no shared
mutation (no &mut coexisting with & or &mut to the same place).

## Requirements

### Requirement: Move tracking
The borrow checker SHALL detect every use of a moved value and report
an error at the use site showing where the move occurred.

### Requirement: Initialization tracking
The borrow checker SHALL detect every use of a potentially-uninitialized
variable, including partial struct initialization.

### Requirement: Aliasing enforcement
The borrow checker SHALL reject any program where a &mut borrow and any
other borrow of the same place are both live at the same point.

### Requirement: unchecked relaxation
Within an `unchecked` block, the aliasing rule (§16.2) SHALL be suspended.
Move and initialization rules SHALL remain active.

### Requirement: Error quality
Every borrow error SHALL show:
(1) Where the conflicting borrow is created
(2) Where it is used after the second borrow  
(3) A plain English explanation of why they conflict

#### Scenario: Use after move
- GIVEN `let s = String.from("hi"); let t = s; print(s)`
- WHEN borrow-checked
- THEN an error is produced at `print(s)`
- AND the error shows "s moved here" pointing to `let t = s`

#### Scenario: Simultaneous &mut and &
- GIVEN `let mut v = Vec.new(); let r = &v[0]; v.push(1); print(r)`
- WHEN borrow-checked
- THEN an error is produced
- AND the error shows the shared borrow at r and the mutation at push

#### Scenario: unchecked permits aliasing assertion
- GIVEN a function prefixed with `unchecked` that creates two &mut slices of the same backing array
- WHEN borrow-checked
- THEN no aliasing error is produced
- AND the unchecked block is recorded in the audit trail
```

---

### openspec/specs/safety-levels/spec.md

```markdown
# Safety Level Specification

## Purpose
Define the five-level safety ladder and the audit mechanism.

## Requirements

### Requirement: Level ordering
The safety levels SHALL form a strict hierarchy where each level grants
strictly more permissions than the level below it.
Level 0 (default) < unchecked < trusted < extern < unsafe.

### Requirement: unchecked is memory safe
An `unchecked` block SHALL NOT permit raw pointer dereferences.
Raw pointer dereferences SHALL require `unsafe`. Attempting a raw pointer
dereference in `unchecked` SHALL be a compile error.

### Requirement: trusted assertions are auditable
Each `trusted("...")` annotation SHALL appear in the output of
`ferrum audit --level trusted` with the exact assertion string.

### Requirement: Level escalation required
A function at a lower safety level SHALL NOT perform operations requiring
a higher level. The compiler SHALL emit an error naming the required level.

#### Scenario: unchecked cannot dereference raw pointers
- GIVEN an `unchecked fn` that dereferences a `*const T`
- WHEN compiled
- THEN a compile error is produced
- AND the error says "raw pointer dereference requires `unsafe`, not `unchecked`"

#### Scenario: trusted assertions appear in audit
- GIVEN a function annotated `trusted("ptr is valid for len bytes")`
- WHEN `ferrum audit --level trusted` is run
- THEN the output includes the function name and the assertion string

#### Scenario: audit at unsafe level shows only unsafe
- GIVEN a codebase with 3 trusted, 2 extern, 1 unsafe annotations
- WHEN `ferrum audit --level unsafe` is run
- THEN exactly 1 site is reported
```

---

### openspec/specs/binary-layout/spec.md

```markdown
# Binary Layout Declaration Specification

## Purpose
Allow programmers to declare the exact binary representation of a type
and have the compiler verify correctness and generate codecs.

## Requirements

### Requirement: Gap detection
The compiler SHALL reject any layout declaration where the declared fields
do not account for every bit in total_size. Unaccounted bits are an error,
not silently padded.

### Requirement: Overlap detection
The compiler SHALL reject any layout declaration where two fields share any
bit position.

### Requirement: Alignment enforcement
Multi-byte fields not on natural alignment boundaries SHALL require an
explicit `unaligned` marker. Missing marker is a compile error.

### Requirement: Decimal literal fidelity in codec
The generated read() codec SHALL NOT convert decimal field values through
binary float representations.

### Requirement: Padding zeroing
Fields named with a leading _ SHALL be zeroed in generated write() codecs.
They are not accessible from the logical type.

#### Scenario: Gap detection
- GIVEN a layout with total_size 16 bits and fields covering only 12 bits
- WHEN compiled
- THEN a compile error is produced naming the 4 unaccounted bits

#### Scenario: Overlap detection  
- GIVEN a layout where field A covers bits 0..3 and field B covers bits 2..5
- WHEN compiled
- THEN a compile error names the overlapping range (bits 2..3)

#### Scenario: Codec round-trip
- GIVEN a StatusRegister type with layout declaration
- AND a value constructed with ready=true, mode=5, error=false
- WHEN write() and then read() are called
- THEN the resulting value is equal to the original
```

---

### openspec/specs/stdlib-http/spec.md

```markdown
# HTTP Stack Specification

## Purpose
Provide a first-class HTTP client and server that does not require
external crates for common use cases. Target experience: Go's net/http.

## Requirements

### Requirement: Typed status codes
StatusCode SHALL be a constrained integer type (u16 where 100 ≤ value ≤ 999).
Named constants SHALL exist for all common codes.

### Requirement: Typed methods
Method SHALL be an enum, not a string. Extension methods use Method.Extension(String).

### Requirement: No stringly-typed headers
Header names SHALL use HeaderName (interned, case-insensitive).
The `header` module SHALL export constants for all common headers.

### Requirement: Body is streaming by default
The Body type SHALL be streaming. Eagerly-loading the full body SHALL
require an explicit .bytes() or .text() await call.

### Requirement: error_for_status
Response SHALL provide error_for_status() → Result[Self, HttpError] that
returns Err for 4xx and 5xx responses.

#### Scenario: Simple GET request
- GIVEN a running HTTP server at localhost:8080 returning 200 OK with body "hello"
- WHEN `Client.new().get("http://localhost:8080/").send().await` is called
- THEN the result is Ok(response)
- AND response.status() == StatusCode.OK
- AND response.text().await == Ok("hello")

#### Scenario: error_for_status on 404
- GIVEN a server returning 404 Not Found
- WHEN .error_for_status() is called on the response
- THEN an Err(HttpError) is returned
- AND the error's status() returns StatusCode.NOT_FOUND

#### Scenario: Router dispatch
- GIVEN a Router with .get("/users/{id}", handler) registered
- WHEN a GET /users/42 request arrives
- THEN handler is called
- AND PathParams contains id = "42"
```

---

## Phase Gates Summary

Before beginning the next phase, all gates in the current phase must be closed.

| Gate | ID | Unlocks |
|------|----|---------|
| Lexer complete | fe-001.8 | Parser |
| Parser round-trip | fe-002.15 | Name resolution, module system |
| Name resolution | fe-003.6 | Type inference |
| Type inference | fe-005.11 | Effect, region inference, constraints, traits |
| Effect inference | fe-006.6 | HIR |
| Region inference | fe-007.7 | Borrow checker |
| Borrow checker | fe-008.9 | HIR |
| Constraints | fe-009.6 | Contracts, proof |
| Traits | fe-010.8 | HIR |
| HIR well-typed | fe-011.7 | MIR |
| MIR valid | fe-012.8 | Contracts, monomorphization |
| Monomorphization | fe-014.5 | C codegen |
| **First working compiler** | fe-015.9 | stdlib work begins |
| core stdlib | fe-017.11 | alloc stdlib |
| alloc stdlib | fe-018.7 | io/fs stdlib |
| io/fs stdlib | fe-019.10 | net stdlib |

---

## Discovered Issue Protocol

When implementing any task, if you discover:

- A spec ambiguity → `bd create "SPEC: <description>" --label spec-ambiguity --priority 0`
- A missing test case → `bd create "TEST: <description>" --label test --priority 1`  
- A performance concern → `bd create "PERF: <description>" --label perf --priority 3`
- A design conflict between reference docs → `bd create "CONFLICT: <description>" --label design --priority 0`

Always add `--deps discovered-from:<current-task-id>` to every filed issue.

Block your current task on any `priority 0` discovered issue before closing it.

---

## Test Corpus Bootstrapping

Before phase 1 begins, populate these test files. The test corpus is the
spec made executable. Agents implementing a component should add tests
before implementing the feature (test-first on all compiler phases).

```
tests/
  compile-pass/
    lexer/
      basic_literals.fe
      all_integer_suffixes.fe
      decimal_float_literals.fe
      nested_block_comments.fe
      unicode_identifiers.fe
    parser/
      all_items.fe
      all_expressions.fe
      all_patterns.fe
      generics.fe
      layout_declarations.fe
      contract_annotations.fe
    types/
      inference_simple.fe
      generics_basic.fe
      associated_types.fe
    effects/
      pure_function.fe
      io_propagation.fe
    regions/
      single_input_borrow.fe
      struct_with_references.fe
    borrow/
      simple_lifetime.fe
      split_at_mut_unchecked.fe

  compile-fail/
    lexer/
      invalid_escape.fe        # expected: LexError::InvalidEscape
    types/
      implicit_widening.fe     # expected: "expected f64, found f32"
      use_after_move.fe        # expected: "use of moved value"
      uninitialized.fe         # expected: "use of possibly uninitialized"
    safety/
      unchecked_raw_deref.fe   # expected: "requires unsafe, not unchecked"
      strengthened_precond.fe  # expected: "precondition strengthened in impl"
    layout/
      gap_in_layout.fe         # expected: "unaccounted bits"
      overlapping_fields.fe    # expected: "overlapping field ranges"

  run-pass/
    hello_world.fe             # print("hello, world")
    fibonacci.fe               # recursive fib, assert fib(10) == 55
    generic_vec.fe             # Vec[i32] push/pop
    pattern_match.fe           # Option and Result matching
    closures.fe                # closure captures and iteration
    channels.fe                # basic channel send/recv
    tcp_echo.fe                # TCP echo server + client
    http_hello.fe              # HTTP server returning 200 OK
```

---

*End of FERRUM-PLAN.md*
