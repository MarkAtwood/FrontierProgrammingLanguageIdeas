# Compiler APIs for Agentic Programming

**Status:** Proposal for exposing compiler internals as interactive APIs for AI agents

---

## The Problem

Compilers know far more about your program than they tell you. Type inference, control flow analysis, alias analysis, escape analysis, lifetime inference, optimization decisions—all of this reasoning happens internally and is exposed only through:

1. **Error messages** — read-only, reactive, unpredictable format
2. **Warnings** — same limitations
3. **Debug info** — post-hoc, limited to what DWARF/PDB can express

An AI agent working on code has no way to ask:

- "What do you know about the type of this expression?"
- "If I added this assertion here, would the rest of the function become verifiable?"
- "Which call sites could trigger this code path?"
- "What invariants are you already tracking that I could leverage?"

The agent must reverse-engineer compiler knowledge from error messages, or re-derive it independently. This is wasteful. The compiler already did the work.

---

## The Proposal

Expose the compiler's internal reasoning as a structured API with four capabilities:

### 1. Query

Ask the compiler what it knows at any program point.

```
agent → compiler: "What do you know about state at line 47?"

compiler → agent: {
  type: "Option<&str>",
  null_state: "may be None (from branch at line 42)",
  live_borrows: [("data", "shared", "valid until line 63")],
  aliasing: ["ptr may alias buffer[0..len]"],
  optimizations_blocked_by: ["potential null deref at line 49"]
}
```

The compiler already computes all of this for optimization and diagnostics. The API externalizes it.

### 2. Hypothesize

Inject a proposed invariant and ask whether it changes anything—without modifying the source.

```
agent → compiler: "Hypothetically, if x is non-null here, what changes?"

compiler → agent: {
  enables: ["null check at line 49 becomes dead code"],
  unlocks: ["function becomes inlineable"],
  required_to_establish: ["caller must prove x != null"]
}
```

This is proof search without committing to anything. The agent proposes, the compiler evaluates, the agent learns what invariants are load-bearing.

### 3. Trap

Place semantically-aware monitors that the compiler weaves into the binary correctly because it understands the program's structure.

```
agent → compiler: "Alert if any write to buffer[i] where i >= len"

compiler → agent: {
  instrumented_sites: 14,
  cannot_instrument: ["inline asm at line 203"],
  estimated_overhead: "~3% on hot path"
}
```

Not debugger breakpoints—conditions expressed in the language's type system that the compiler knows how to check efficiently. The agent gets systematic property testing without writing test code.

### 4. Instrument

Extract the compiler's knowledge as test scaffolding.

```
agent → compiler: "Generate test harness for parse_config()"

compiler → agent: {
  preconditions: ["input.len() < 64KB", "valid UTF-8"],
  postconditions: ["result.is_ok() implies config.validate()"],
  generated_harness: "...",
  edge_cases_from_types: ["empty input", "max-length input", "invalid UTF-8"]
}
```

The compiler already knows the function's contracts from its types, assertions, and documentation. The API extracts that knowledge into executable tests.

---

## Implementation by Toolchain

### Clang/LLVM

**What already exists:**
- `libclang` — C API for AST traversal, type queries, code completion
- `clang-query` — AST matcher queries
- Clang Static Analyzer — path-sensitive dataflow analysis
- LLVM's analysis passes — alias analysis, scalar evolution, loop analysis

**What's missing:**
- **Hypothetical reasoning.** No way to ask "if I added `__assume(x != NULL)` here, what optimizations unlock?" without actually adding it and recompiling.
- **Semantic traps.** Address Sanitizer instruments memory access, but there's no API to say "instrument all writes to this struct field" based on semantic meaning.
- **Unified state query.** Information is scattered across the AST (types), Clang's CFG (control flow), and LLVM's analysis passes (aliasing, optimization). No single query point.

**Implementation path:**
1. Extend `libclang` with analysis pass results—expose what LLVM already computes
2. Add a hypothetical mode to the optimizer: "evaluate this assume, report effects, don't commit"
3. Build a semantic instrumentation API on top of LLVM's existing sanitizer infrastructure
4. Create a unified query protocol that fans out to AST, Clang analyzer, and LLVM passes

The infrastructure is 80% there. The missing piece is the unified API and hypothetical mode.

### Rust (rustc)

**What already exists:**
- `rust-analyzer` — IDE-oriented queries (types, completions, references)
- MIR (Mid-level IR) — borrow checker state, lifetime analysis
- Chalk — trait solver with queryable proof search
- `-Z` flags exposing internals (dump MIR, show lifetime errors)

**What's missing:**
- **Hypothetical borrowing.** No way to ask "if this borrow were `'static`, would the function compile?" without modifying source.
- **Region query.** The borrow checker computes region constraints but doesn't expose them. You see errors, not the constraint graph.
- **Trap API.** Miri instruments memory operations but isn't queryable from outside.

**Implementation path:**
1. Expose the region constraint solver as an API—not just pass/fail, but the full constraint graph
2. Add hypothetical mode: "assume this lifetime relationship, check downstream"
3. Extend Chalk's query interface to answer "what traits does this type need to implement for this to work?"
4. Build on Miri for semantic traps—let external tools specify conditions to monitor

Rust's query-based compiler architecture (Salsa) makes this more tractable than it would be in a traditional batch compiler.

### Python

**What already exists:**
- `ast` module — syntax tree queries
- `mypy` daemon — incremental type checking with some query support
- `pyright` — Language Server Protocol with type queries
- `sys.settrace` / `sys.setprofile` — runtime instrumentation

**What's missing:**
- **Type state query.** Mypy and Pyright compute type narrowing, but there's no API to ask "what's the inferred type of `x` after this branch?"
- **Hypothetical typing.** No way to ask "if I annotated this as `list[int]`, would the rest of the function type-check?"
- **Semantic traps.** `settrace` is line-level. There's no way to say "trap when this variable changes type" or "trap when this function returns a value inconsistent with its annotation."

**Implementation path:**
1. Extend LSP implementations to expose type narrowing state at arbitrary points
2. Add hypothetical annotation mode: "evaluate this type annotation, report conflicts, don't persist"
3. Build semantic traps on top of `settrace` + type checker integration—the trap condition is type-aware
4. Expose control flow graph and reachability analysis from type checkers

Python's dynamic nature makes static analysis weaker, but the same API shape applies. The answers are probabilistic or conditional rather than definitive.

### GCC

**What already exists:**
- Plugin API — access to GIMPLE (middle IR), SSA form, analysis results
- `-fdump-*` flags — extensive internal state dumps
- Static analyzer (`-fanalyzer`) — path-sensitive analysis

**What's missing:**
- **Query interface.** The plugin API is procedural, not query-based. You can walk the IR, but you can't ask "what do you know about this variable?"
- **Hypothetical mode.** No way to evaluate assumptions without committing them
- **Instrumentation API.** Sanitizers are built-in, not extensible

**Implementation path:**
1. Build a query layer on top of the plugin API—translate queries into IR walks
2. Add assume intrinsics that the optimizer evaluates but optionally discards
3. Expose the static analyzer's state as a queryable API
4. Factor sanitizer instrumentation into an extensible framework

GCC's architecture is older and more monolithic, making this harder than Clang/LLVM. But the analysis infrastructure exists.

---

## The Agent Workflow

With this API, an agent debugging a failing verification works like an expert:

```
1. Query: "What's failing?"
   → "Cannot prove buffer access is in bounds at line 47"

2. Query: "What would prove it?"
   → "Needs: index < buffer.len()"

3. Hypothesize: "What if the loop invariant includes index < len?"
   → "That would close the proof. But you need to establish it at loop entry."

4. Hypothesize: "What if I assert len <= buffer.len() at function entry?"
   → "That closes the loop entry. But callers would need to prove it."

5. Query: "What do callers pass for len?"
   → "3 call sites: (1) literal 10, (2) data.len(), (3) user input"

6. Trap: "Instrument call site 3, log when len > buffer.len()"
   → [runs tests, trap never fires]

7. Agent: adds assertion at function entry with comment:
   "// Validated: 3 call sites checked, fuzz testing passed, trap never fired"
```

The agent never had to understand the full verification problem. It explored hypotheses, validated them dynamically, and committed only what was necessary. The compiler did the reasoning; the agent did the search.

---

## What This Changes

### For AI agents

Verification stops being a black box. The agent can have a structured conversation with the compiler instead of parsing error messages and guessing. The search space becomes tractable.

### For humans

The same API powers better IDE experiences. "Why can't I do this?" gets a real answer with the full constraint graph, not a one-line error. Hypothetical mode lets you explore refactoring consequences without committing.

### For formal verification

The proof search workflow that experts do manually—propose lemmas, see what they enable, iterate—becomes automatable. The expertise is in knowing which hypotheses to try. The API handles the evaluation.

### For testing

Contracts, type invariants, and assertions become test oracles automatically. The compiler already knows the specification. It just needs an API to export it.

---

## Incremental Implementation

This doesn't require rewriting compilers. It requires:

1. **Query consolidation.** Most analysis results already exist; they need a unified query interface.
2. **Hypothetical mode.** A flag that says "evaluate this assumption, report effects, don't commit."
3. **Trap weaving.** Generalize existing sanitizer infrastructure to accept external conditions.
4. **Test extraction.** Traverse existing type/contract annotations and emit harnesses.

Each piece is useful independently. A compiler that only implements Query is still valuable. The full API is an incremental build on existing infrastructure.

---

## Open Questions

**Stability.** Internal representations change between compiler versions. The API needs a stable abstraction layer that survives internal refactoring.

**Performance.** Hypothetical mode may require re-running expensive analyses. Caching and incrementality matter.

**Security.** Exposing compiler internals could leak information about proprietary code. The API needs access controls when used in multi-tenant environments.

**Standardization.** Should this be language-specific or a cross-language protocol? LSP succeeded by being language-agnostic. A Compiler Agent Protocol could do the same.

---

## Prior Art

- **Dafny / F* / Lean** — Interactive proof assistants with query-based interfaces. The model here is to bring that interaction style to mainstream compilers.
- **LSP (Language Server Protocol)** — Standardized IDE queries. This proposal extends LSP's scope to include verification state, hypotheticals, and instrumentation.
- **Clang's `libclang`** — Partial implementation of Query. Shows the API shape works.
- **Miri** — Rust's interpreter with memory model checking. Shows Trap is feasible.
- **Property-based testing (QuickCheck, Hypothesis)** — Instrument is the compiler-native version of this.

---

*End of compiler-agent-api.md*
