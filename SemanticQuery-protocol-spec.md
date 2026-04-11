# SemanticQuery Protocol Specification

**Version:** 0.1 (draft)
**Status:** Proposal
**Namespace:** `semanticQuery/`

---

## Abstract

Compilers maintain rich internal models of the programs they analyze: types, aliasing relationships, data flow, ownership and borrow state, accumulated effects, active contracts, and verification conditions. Tools that need this information — editors, AI agents, formal verifiers, fuzz harnesses, security auditors — currently have no standardized way to access it. Each compiler exposes different APIs at different levels of abstraction; most expose nothing at all beyond diagnostics.

SemanticQuery is a protocol that addresses this gap. It defines five operations for querying a compiler's semantic and verification state: point queries over program locations, counterfactual queries that inject hypothetical invariants, semantic monitors placed in compiled binaries, test scaffolding generated from compiler-known contracts, and SMT-LIB export of verification conditions for external solvers. The protocol is specified as a set of JSON-RPC methods that extend the Language Server Protocol and can also be spoken standalone.

SemanticQuery is designed to be implementable by any compiler infrastructure. The data model is language-specific — what a Rust borrow checker knows differs from what a C++ alias analyzer knows — but the protocol shape is language-independent. One client implementation works against any compliant server.

---

## 1. Introduction

### 1.1 The Problem

A compiler that type-checks a function knows the type of every expression. A compiler that runs borrow checking knows the aliasing picture at every program point. A compiler that verifies contracts knows which ones are discharged and which remain open. A compiler that runs alias analysis knows which pointers may overlap.

Tools need this information. An AI agent closing a verification gap needs to know what the compiler knows at the ambiguous point. A fuzz harness needs the compiler's preconditions to generate valid inputs. A formal verifier needs the compiler's verification conditions to run an alternative solver. A security auditor needs the alias analysis results to find use-after-free paths.

Today, none of this is accessible through a standard interface. rust-analyzer's `viewMir` returns a human-readable string. Clang's Static Analyzer produces text reports. GNATprove writes proof directories in its own format. Roslyn's `SemanticModel` is a .NET library, not a wire protocol. Tools that want this information must either parse text output (fragile, format-specific) or link against the compiler as a library (language-specific, version-coupled).

The result: every tool reimplements its own partial version of the compiler's semantic model, from scratch, in isolation.

### 1.2 The Solution

SemanticQuery defines a standard protocol for accessing compiler semantic state. It is structured around a simple observation: the compiler has already done the hard work. The goal is to expose that work through a stable, language-agnostic interface.

The protocol has five operations:

1. **Point Query** — ask the compiler what it knows at a specific program location
2. **Counterfactual Query** — inject a hypothetical invariant and ask what changes
3. **Monitor** — place a semantically-aware trap in the compiled binary
4. **Scaffolding** — generate test harnesses from compiler-known contracts
5. **SMT Export** — retrieve verification conditions as SMT-LIB 2 for external solvers

### 1.3 Design Principles

**Language-agnostic protocol, language-specific data.** The JSON-RPC method names and request/response shapes are fixed across all implementations. The content of the `type` field, the `effects` field, the `borrow_state` field — these are language-specific. A Python implementation returns mypy-inferred types; a Rust implementation returns NLL region constraints. The protocol does not impose a single type theory.

**Additive over LSP.** SemanticQuery is specified as LSP extension methods. It requires an LSP session but adds to it, not replaces it. An editor that supports LSP gets SemanticQuery on the same connection without additional infrastructure.

**Counterfactual queries are the key operation.** Every other operation in this spec exists in some form in prior art. Point queries exist in Roslyn and ASIS. SMT export exists in GNATprove. Monitor-like behavior exists in sanitizers. What has not existed before is the ability to inject a hypothetical invariant and receive a structured verdict — without modifying source code. This is the operation that turns verification from a yes/no gate into a conversation.

**The compiler's internal state is an asset.** A compiler that implements SemanticQuery makes its internal reasoning accessible to the ecosystem. Tools do not have to rebuild what the compiler already knows.

---

## 2. Prior Art

SemanticQuery synthesizes ideas that have appeared separately in compiler infrastructure over the past 25 years. None of the prior systems combined all five operations; each stopped short in different ways.

### 2.1 ASIS — Ada Semantic Interface Specification (ISO/IEC 15291:1999)

The original. ASIS is a standardized API, mandated by the Ada Reference Manual, for tools to query the semantic information an Ada compiler produces. Defined in the late 1990s, implemented by GNAT and other Ada compilers, ASIS gave tool authors access to declarations, expression types, call graphs, package hierarchies, and overloading resolution. It was used for static analyzers, metrics tools, documentation generators, and refactoring tools.

The key insight ASIS formalized: **the compiler has already done the hard semantic work; tools should not redo it from scratch.** SemanticQuery is the direct descendant of this insight.

What ASIS lacked: read-only (no counterfactual queries); post-compilation (not incremental); no verification state; Ada-specific, not designed for cross-language adoption.

### 2.2 Libadalang (AdaCore, ~2017)

The modern replacement for ASIS. Libadalang is a library (not a protocol) that provides incremental parsing, name resolution, and semantic queries over Ada source. It tolerates incomplete and syntactically invalid code, making it suitable for IDE use.

Libadalang shows that the library model has a key limitation: it requires language-specific bindings for each client language. A protocol — SemanticQuery's approach — works for any client that speaks JSON-RPC, regardless of implementation language.

### 2.3 Roslyn — .NET Compiler Platform (Microsoft, 2014)

Roslyn reframed the C# and VB compilers as libraries — "compiler as a service." The `SemanticModel` API answers questions about compiled symbols, types, and data flow:
- `GetTypeInfo(node)` — type of an expression
- `GetSymbolInfo(node)` — resolved symbol for a name
- `AnalyzeDataFlow(statements)` — definitely-assigned, read/written variables across a block
- `AnalyzeControlFlow(statements)` — reachability, exit points

`AnalyzeDataFlow` is a direct ancestor of SemanticQuery's Point Query borrow state field. Roslyn also proved that the market exists: tool authors use a well-designed compiler semantic API heavily if it is available. LSP itself was partly inspired by the insight that Roslyn-style semantic information should be accessible over a wire protocol, not just from within Visual Studio.

What Roslyn lacked: read-only (no counterfactual queries); no formal verification state (C# has no ownership or effect system); library model, not a wire protocol.

### 2.4 rust-analyzer LSP Extensions

rust-analyzer implements LSP and adds `rust-analyzer/`-namespaced custom requests that expose Rust compiler internals: `viewMir` (MIR with borrow annotations), `viewHir` (desugared representation), `syntaxTree`, `expandMacro`, `interpretFunction`.

`viewMir` in particular exposes borrow checker state — basic blocks, moves, borrows, drops — and is the closest prior system to SemanticQuery's borrow state field. The pattern rust-analyzer establishes — LSP as transport, custom namespace for extensions, incremental and live — is precisely the architecture SemanticQuery adopts.

The critical limitation: all rust-analyzer extensions return human-readable strings, not structured data. `viewMir` returns a pretty-printed representation for a human who already knows what MIR looks like. A tool that wants borrow state at line 47 must parse that string — fragile, format-specific, not a stable API. SemanticQuery returns structured JSON with typed fields that programs can consume directly.

### 2.5 Dafny Language Server

Dafny is a verification-aware language with an LSP server that exposes verification state to editors: per-method verification status (verified, failed, unknown, timed out), counterexamples with concrete values when verification fails, ghost variable states. Dafny proves the use case: developers and tools want verification state incrementally, over LSP.

What Dafny's language server lacked: no counterfactual query, no SMT-LIB export, Dafny-specific data model not designed for cross-language adoption.

### 2.6 SPARK / GNATprove

SPARK is a formally verifiable subset of Ada. GNATprove runs flow analysis and proof (via Why3 and Z3) and reports proof status per subprogram, unproved checks with their verification condition, and flow analysis results. Its `--proof-dir` output exports verification conditions in Why3's logic for external consumption.

GNATprove's `--proof-dir` is the direct predecessor of SemanticQuery's `smtExport` operation. The insight is the same: make the compiler's internal verification conditions available to external tools. SemanticQuery standardizes the access as a protocol operation.

### 2.7 What SemanticQuery Adds

| Feature | ASIS | Roslyn | rust-analyzer | Dafny LSP | GNATprove | SemanticQuery |
|---|---|---|---|---|---|---|
| Standardized cross-language protocol | ✓ | — | — | — | — | ✓ |
| LSP-native, incremental | — | — | ✓ | ✓ | — | ✓ |
| Structured (machine-readable) responses | ✓ | ✓ | — | partial | — | ✓ |
| Verification state (contracts, borrows, effects) | — | — | partial | ✓ | ✓ | ✓ |
| Counterfactual queries | — | — | — | — | — | ✓ |
| SMT-LIB export | — | — | — | — | partial | ✓ |

The counterfactual query is the genuinely new operation. Everything else is synthesis and standardization.

---

## 3. Protocol Overview

SemanticQuery is a set of JSON-RPC 2.0 methods in the `semanticQuery/` namespace. They are carried over the same transport as an LSP session (stdio or TCP, line-delimited JSON). A SemanticQuery server is an LSP server that announces SemanticQuery capabilities in the `initialize` response.

### 3.1 Capability Negotiation

The server announces support in the `initialize` response:

```json
{
  "capabilities": {
    "semanticQueryProvider": {
      "version": "0.1",
      "pointQuery": true,
      "counterfactualQuery": true,
      "monitor": true,
      "scaffolding": true,
      "smtExport": true
    }
  }
}
```

Each capability may be `true` (supported) or absent/`false` (not supported). A client negotiates against what the server announces. Partial implementations are valid and useful.

### 3.2 Lifecycle

SemanticQuery requests follow the LSP lifecycle: they may be sent after `initialized` and before `shutdown`. They reference documents by URI; the server tracks document state via LSP `textDocument/didOpen`, `didChange`, `didClose` notifications.

### 3.3 Error Codes

SemanticQuery defines the following error codes in the JSON-RPC error range:

| Code | Name | Meaning |
|---|---|---|
| -32100 | `LocationNotAnalyzed` | The requested location is in code not yet analyzed |
| -32101 | `UnsupportedLanguage` | Server does not support SemanticQuery for this file type |
| -32102 | `HypothesisParseError` | Counterfactual hypothesis could not be parsed |
| -32103 | `MonitorCompileRequired` | Monitor installation requires recompilation |
| -32104 | `SmtUnavailable` | SMT encoding not available for this function |

---

## 4. Common Types

```typescript
// LSP Position: { line: number, character: number }
// LSP Range: { start: Position, end: Position }

interface Location {
  uri: string;
  position: Position;
}

// A borrow or alias relationship at a program point.
// For languages without borrow checking, this field describes alias analysis results.
interface BorrowEntry {
  place: string;              // e.g. "buf", "self.data", "*ptr", "v[i]"
  kind: "shared"              // read-only access, may alias
       | "exclusive"          // read-write access, no other access
       | "moved"              // value has been consumed
       | "may_alias"          // alias analysis: may alias with another place
       | "no_alias"           // alias analysis: provably does not alias
       | "must_alias";        // alias analysis: must alias
  region: string | null;      // lifetime/region name, null if not applicable
  source_span: Range;         // where the borrow/alias was introduced
  must_call: string[] | null; // for ownership systems: obligations before drop
}

// A contract visible at a program point.
interface ContractEntry {
  kind: "requires" | "ensures" | "invariant" | "assert" | "assume";
  expression: string;         // the contract in the source language's expression syntax
  status: "verified"          // statically discharged (SMT or proof)
         | "runtime_checked"  // enforced at runtime
         | "assumed"          // taken as axiomatic (trusted annotation)
         | "unresolved";      // not yet verified or checked
  source_span: Range;
  discharge_method: string | null; // "z3", "cvc5", "prusti", "manual_proof", null
}

// A region or lifetime constraint.
interface RegionConstraint {
  region: string;
  outlives: string[];         // regions this region is required to outlive
  scope: Range;               // source span where this region is live
}

// Something the compiler cannot determine without additional information.
interface UnresolvedItem {
  kind: "region" | "effect" | "contract" | "alias" | "type";
  description: string;        // human-readable description of what is unknown
  hint: string;               // what annotation or invariant would resolve it
  source_span: Range | null;
}

// A source edit (for suggested_source_edit in counterfactual responses).
interface SourceEdit {
  uri: string;
  range: Range;
  new_text: string;
}

// A counterexample witness (for counterfactual responses where resolves == false).
interface Counterexample {
  description: string;
  witness_values: { [variable: string]: string } | null;
  witness_smt: string | null;   // SMT-LIB model, if available
}
```

---

## 5. Operations

### 5.1 Point Query — `semanticQuery/pointQuery`

Ask the server what it knows about the program state at a specific source location. The response captures the compiler's complete semantic picture at that point: types, accumulated effects, borrow/alias relationships, active contracts, and what remains unresolved.

**Request:**

```typescript
interface PointQueryParams {
  location: Location;
  include?: {
    type?: boolean;            // default true
    effects?: boolean;         // default true
    borrows?: boolean;         // default true
    regions?: boolean;         // default true
    contracts?: boolean;       // default true
    unresolved?: boolean;      // default true
    callgraph?: boolean;       // default false
  };
}
```

**Response:**

```typescript
interface PointQueryResult {
  location: Location;

  // The type of the primary expression at this location.
  // Syntax is server-defined (language-specific).
  type: string | null;

  // Effects that have accumulated on all paths to this point.
  // Empty for languages without effect systems.
  // For languages with partial effect information, includes what is known
  // (e.g. "noexcept_violated" for C++, "unsafe_block" for Rust).
  effects_accumulated: string[];

  // Effects declared on the enclosing function.
  effects_declared: string[];

  // Borrow / alias state at this point.
  borrow_state: BorrowEntry[];

  // Region/lifetime constraints visible at this point.
  region_constraints: RegionConstraint[];

  // Contracts that are active at this point.
  active_contracts: ContractEntry[];

  // Things the compiler cannot determine without more information.
  unresolved: UnresolvedItem[];

  // Outgoing calls visible from this point and their effects.
  callgraph?: CallSite[];
}

interface CallSite {
  callee: string;
  location: Location;
  effects: string[];
  is_virtual: boolean;
}
```

**Example — Rust:**

```json
{
  "location": { "uri": "file:///src/parser.rs", "position": { "line": 47, "character": 12 } },
  "type": "&'_#3r [u8]",
  "effects_accumulated": [],
  "effects_declared": [],
  "borrow_state": [
    {
      "place": "buf",
      "kind": "exclusive",
      "region": "'_#3r",
      "source_span": { "start": { "line": 41, "character": 4 }, "end": { "line": 41, "character": 7 } },
      "must_call": null
    },
    {
      "place": "self.data",
      "kind": "shared",
      "region": "'_#1r",
      "source_span": { "start": { "line": 38, "character": 8 }, "end": { "line": 38, "character": 17 } },
      "must_call": null
    }
  ],
  "region_constraints": [
    { "region": "'_#3r", "outlives": ["'_#1r"], "scope": { "start": {"line": 41, "character": 4}, "end": {"line": 52, "character": 1} } }
  ],
  "active_contracts": [],
  "unresolved": [
    {
      "kind": "region",
      "description": "lifetime of return value is ambiguous between 'self and 'buf",
      "hint": "annotate return type with an explicit lifetime",
      "source_span": { "start": { "line": 38, "character": 0 }, "end": { "line": 38, "character": 30 } }
    }
  ]
}
```

**Example — SPARK/Ada:**

```json
{
  "location": { "uri": "file:///src/search.adb", "position": { "line": 23, "character": 8 } },
  "type": "Positive (constrained: 1 .. S'Last)",
  "effects_accumulated": [],
  "effects_declared": [],
  "borrow_state": [
    { "place": "S", "kind": "shared", "region": "call_scope", "source_span": {...}, "must_call": null }
  ],
  "region_constraints": [],
  "active_contracts": [
    { "kind": "requires", "expression": "S'Length > 0", "status": "verified",
      "source_span": {...}, "discharge_method": "z3" },
    { "kind": "ensures", "expression": "Result in S'Range", "status": "runtime_checked",
      "source_span": {...}, "discharge_method": null }
  ],
  "unresolved": [
    { "kind": "contract", "description": "loop invariant not provided",
      "hint": "add pragma Loop_Invariant inside the while loop", "source_span": {...} }
  ]
}
```

**Example — C++ (Clang Static Analyzer):**

```json
{
  "location": { "uri": "file:///src/buffer.cpp", "position": { "line": 89, "character": 14 } },
  "type": "const std::vector<int> & (nonnull)",
  "effects_accumulated": ["may_throw"],
  "effects_declared": [],
  "borrow_state": [
    { "place": "v", "kind": "shared", "region": "caller_scope", "source_span": {...}, "must_call": null },
    { "place": "internal_ptr", "kind": "may_alias", "region": null, "source_span": {...}, "must_call": null }
  ],
  "region_constraints": [],
  "active_contracts": [],
  "unresolved": [
    { "kind": "alias", "description": "potential use-after-free on path through line 82",
      "hint": "ensure pointer is not invalidated before dereference", "source_span": {...} }
  ]
}
```

---

### 5.2 Counterfactual Query — `semanticQuery/counterfactualQuery`

Inject a hypothetical invariant at a program point and ask the server what would change in the verification state — without modifying the source. The server evaluates the hypothesis against its current analysis state and returns a structured verdict.

This is the operation that turns verification from a yes/no gate into a conversation. An automated tool can iterate over candidate hypotheses until one resolves an open question, then propose only the winning annotation to a human. The human never sees the failed candidates; the tool already filtered them.

**Request:**

```typescript
interface CounterfactualQueryParams {
  location: Location;
  hypothesis: Hypothesis;
}

type Hypothesis =
  | {
      kind: "invariant";
      expression: string;
      // An assertion in the source language's expression syntax,
      // hypothetically true at this location.
      // Examples: "ptr != null", "v.len() > 0", "x >= 0 && x < bound"
    }
  | {
      kind: "region_bound";
      place: string;
      region: string;
      // The named place borrows from the named region.
      // Examples: place="return", region="'outer"
    }
  | {
      kind: "effect_bound";
      expression: string;
      effects: string[];
      // The named expression has only the listed effects.
      // Examples: expression="helper(x)", effects=[]
    }
  | {
      kind: "alias_fact";
      place_a: string;
      place_b: string;
      relationship: "no_alias" | "must_alias" | "may_alias";
      // Hypothetically assert an aliasing relationship.
    }
  | {
      kind: "type_narrowing";
      place: string;
      narrowed_type: string;
      // Hypothetically narrow the type of a place.
      // Useful for gradual type systems (Python/TypeScript).
      // Examples: place="x", narrowed_type="str"
    }
  | {
      kind: "annotation";
      source: string;
      // A raw source-level annotation applied hypothetically.
      // Server interprets this in its own annotation language.
    }
```

**Response:**

```typescript
interface CounterfactualQueryResult {
  hypothesis: Hypothesis;

  // Does the hypothesis close at least one UnresolvedItem?
  resolves: boolean;

  // Which UnresolvedItems this hypothesis closes.
  resolved_items: UnresolvedItem[];

  // UnresolvedItems that remain after the hypothesis is applied.
  remaining_unresolved: UnresolvedItem[];

  // New UnresolvedItems the hypothesis introduces (e.g. the hypothesis
  // itself now needs justification).
  new_unresolved: UnresolvedItem[];

  // If resolves == true: the concrete source edit that would make this
  // hypothesis true permanently.
  suggested_source_edit: SourceEdit | null;

  // If resolves == false: a witness to why the hypothesis does not help.
  counterexample: Counterexample | null;
}
```

**Example — resolving a lifetime ambiguity:**

```
Tool → Server: "If the return value borrows from region 'outer, does the ambiguity resolve?"

Server → Tool: {
  hypothesis: { kind: "region_bound", place: "return", region: "'outer" },
  resolves: true,
  resolved_items: [{ kind: "region", description: "lifetime of return value is ambiguous..." }],
  remaining_unresolved: [],
  new_unresolved: [],
  suggested_source_edit: {
    uri: "file:///src/parser.rs",
    range: { start: { line: 38, character: 23 }, end: { line: 38, character: 26 } },
    new_text: "&'outer str"
  },
  counterexample: null
}

Tool: writes "&'outer str" to the source
```

**Example — a hypothesis that does not resolve:**

```
Tool → Server: "If ptr != null here, does the use-after-free warning go away?"

Server → Tool: {
  hypothesis: { kind: "invariant", expression: "ptr != null" },
  resolves: false,
  resolved_items: [],
  remaining_unresolved: [{ kind: "alias", description: "potential use-after-free..." }],
  new_unresolved: [],
  suggested_source_edit: null,
  counterexample: {
    description: "ptr being non-null does not prevent invalidation by realloc on the path through line 82",
    witness_values: { "ptr": "0x7fff1234", "realloc_called": "true" },
    witness_smt: "(model (define-fun ptr () (_ BitVec 64) #x00007fff1234) ...)"
  }
}

Tool: tries a different hypothesis
```

**The agent workflow:**

```
1. Point Query: "What does the compiler know at line 47?"
   → gets UnresolvedItem: "return lifetime ambiguous"

2. Counterfactual Query: "If return borrows from 'a, does it resolve?"
   → resolves: false, counterexample: "no path from 'a to return region"

3. Counterfactual Query: "If return borrows from 'outer, does it resolve?"
   → resolves: true, suggested_source_edit: "&'outer str"

4. Agent writes the annotation. Done.
```

The agent queried state, iterated over hypotheses, received verdicts, and wrote only the annotation that was confirmed to work. It never had to understand region inference from first principles.

---

### 5.3 Monitor — `semanticQuery/installMonitor` / `semanticQuery/removeMonitor`

Place a semantically-aware trap in a compiled binary. The condition is expressed in the source language's expression syntax; the server compiles it with full awareness of types, memory layout, and semantic meaning. This is stronger than a raw debugger watchpoint: the server knows what the condition *means* in the program's semantic model.

**Install request:**

```typescript
interface InstallMonitorParams {
  location: Location;
  condition: string;           // source-language expression, evaluated at this point
  action: MonitorAction;
  label: string;               // identifier for this monitor
}

type MonitorAction =
  | { kind: "log";     message: string }
  | { kind: "abort";   message: string }
  | { kind: "report";  channel: string }
  | { kind: "capture"; variables: string[] };
```

**Install response:**

```typescript
interface InstallMonitorResult {
  monitor_id: string;
  compile_needed: boolean;     // true if binary must be recompiled
  warnings: string[];
}
```

**Remove request:**

```typescript
interface RemoveMonitorParams {
  monitor_id: string;
}
```

**Example:**

```json
{
  "method": "semanticQuery/installMonitor",
  "params": {
    "location": { "uri": "file:///src/parse.c", "position": { "line": 89, "character": 0 } },
    "condition": "result == NULL && input_len > 0",
    "action": { "kind": "report", "channel": "monitor_events" },
    "label": "parse_header_unexpected_null"
  }
}
```

---

### 5.4 Scaffolding — `semanticQuery/scaffolding`

Generate a test harness from the compiler's knowledge of a function's contracts and types. The compiler already knows the preconditions (what valid inputs look like), the postconditions (what correct outputs look like), and the types of all parameters. This operation makes that knowledge executable.

**Request:**

```typescript
interface ScaffoldingParams {
  function_location: Location;
  style: "unit_test"       // handwritten-input test skeleton
        | "property_test"  // property-based test with generated inputs
        | "fuzz_target"    // libFuzzer / AFL-compatible harness
        | "smt_harness";   // SMT-LIB harness for solver-based testing
  language?: string;       // target language for generated harness, if different from source
}
```

**Response:**

```typescript
interface ScaffoldingResult {
  function_name: string;
  preconditions: string[];    // requires / pre expressions
  postconditions: string[];   // ensures / post expressions
  generated_source: string;   // the test harness as source code
  notes: string[];
}
```

**Example — Rust property test:**

```json
{
  "function_name": "binary_search",
  "preconditions": ["arr.is_sorted()"],
  "postconditions": [
    "match result { Some(i) => arr[i] == *target, None => !arr.contains(target) }"
  ],
  "generated_source": "#[test]\nfn prop_binary_search() {\n  ...\n}",
  "notes": ["precondition arr.is_sorted() requires a sorted-array generator"]
}
```

**Example — C fuzz target:**

```json
{
  "function_name": "parse_packet",
  "preconditions": ["buf != NULL", "len > 0", "len <= MAX_PACKET_SIZE"],
  "postconditions": ["return >= 0 || return == PARSE_ERROR"],
  "generated_source": "int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {\n  if (size == 0 || size > MAX_PACKET_SIZE) return 0;\n  parse_packet(data, size);\n  return 0;\n}\n",
  "notes": []
}
```

---

### 5.5 SMT Export — `semanticQuery/smtExport`

Export the server's verification conditions for a function as SMT-LIB 2. The server's internal verification pipeline generates these conditions (via Z3, CVC5, Boogie, Why3, or similar); this operation surfaces them for use with any SMT-LIB–compatible solver.

This makes the verification pipeline composable. Consumers are not locked to the server's internal solver choice. Alternative solvers, research tools, and custom decision procedures can all consume the output.

**Request:**

```typescript
interface SmtExportParams {
  function_location: Location;
  scope: "full_function"    // all VCs for the function
        | "contracts_only"  // only requires/ensures VCs
        | "loop_invariants" // only loop invariant VCs
        | "single_vc";      // VC nearest to the given location
  include_background?: boolean;  // include standard prelude (default true)
}
```

**Response:**

```typescript
interface SmtExportResult {
  function_name: string;
  format: "smtlib2";
  solver_logic: string;        // SMT-LIB logic, e.g. "AUFLIA", "QF_LIA", "QF_BV"
  encoding: string;            // the SMT-LIB 2 text
  vc_count: number;
  notes: string[];
}
```

**Usage with alternative solvers:**

```bash
# Export VCs
ferrum-lsp --stdio < smtexport_request.json | jq -r '.result.encoding' > vcs.smt2

# Run with any SMT-LIB 2 solver
z3 vcs.smt2
cvc5 vcs.smt2
yices-smt2 vcs.smt2
bitwuzla vcs.smt2
opensmt vcs.smt2
```

---

## 6. Relationship to LSP

SemanticQuery is specified as LSP extension methods. A SemanticQuery server is an LSP server. The five SemanticQuery operations run on the same JSON-RPC channel as LSP requests, using the same transport, the same document tracking, and the same lifecycle.

### 6.1 LSP Baseline

A compliant SemanticQuery server also implements the LSP baseline. The following LSP methods are expected; SemanticQuery extends their behavior:

| LSP Method | SemanticQuery extension |
|---|---|
| `textDocument/hover` | Includes type, effects, and contract status for the hovered expression (richer than standard type-on-hover) |
| `textDocument/diagnostic` | Includes unresolved SemanticQuery items as structured diagnostics |
| `textDocument/inlayHint` | Inferred effects and region annotations suppressed by inference, shown inline |
| `textDocument/codeAction` | "Add annotation" quick fixes derived from counterfactual query `suggested_source_edit` |
| `textDocument/completion` | Context-aware completions that respect semantic state at point |

### 6.2 Hover Extension

The `textDocument/hover` response for a SemanticQuery-aware server includes semantic state as structured data in the response's `data` field (which LSP defines as extension-specific):

```json
{
  "contents": {
    "kind": "markdown",
    "value": "**Type:** `Option<usize>`\n**Contracts:** `requires arr.is_sorted()` ✓ verified"
  },
  "data": {
    "semanticQuery": {
      "type": "Option<usize>",
      "effects_accumulated": [],
      "active_contracts": [
        { "kind": "requires", "expression": "arr.is_sorted()", "status": "verified" }
      ],
      "unresolved": []
    }
  }
}
```

Clients that do not understand SemanticQuery display the `contents` markdown normally. Clients that do understand SemanticQuery can use the structured `data.semanticQuery` field for richer display or automated processing.

### 6.3 Standalone Mode

SemanticQuery can be spoken standalone, without an LSP session, over stdio or TCP. The same JSON-RPC methods are used; the `initialize` / `shutdown` lifecycle is simplified. A standalone SemanticQuery server is useful for batch analysis tools, CI pipelines, and automated agents that do not require a full editor integration.

```bash
# Standalone CLI (implementation-defined)
ferrum-sq pointQuery --file src/search.fe --line 47 --col 12
clang-sq pointQuery --file src/buffer.cpp --line 89 --col 14
gnatprove-sq smtExport --file src/search.adb --line 23 --scope contracts_only
```

---

## 7. SMT-LIB Integration

SemanticQuery's `smtExport` operation is defined to produce standard SMT-LIB 2 output. This section describes the expected encoding conventions.

### 7.1 Logic Selection

The server selects the most appropriate SMT-LIB logic based on the features used in the verification conditions:

| Features | Logic |
|---|---|
| Linear integer arithmetic, arrays | `AUFLIA` |
| Linear integer arithmetic only | `QF_LIA` or `LIA` |
| Bitvector arithmetic | `QF_BV` or `ABV` |
| Nonlinear arithmetic | `NIA` or `AUFNIA` |
| Real arithmetic | `QF_LRA` |

### 7.2 Encoding Structure

A well-formed `smtExport` encoding follows this structure:

```smt2
; SemanticQuery SMT Export
; Function: <function_name>
; Source: <file>:<line>
; Server: <server_name> <version>
; Logic: <logic>

(set-logic <logic>)

; --- Background theory (if include_background == true) ---
; Standard definitions for types and operations used in VCs

; --- Declarations ---
; Function inputs and outputs as SMT constants

; --- Preconditions ---
; (assert <requires_expression>)

; --- Verification conditions ---
; One (check-sat) per VC, preceded by the negation of what is to be proved
; (assert (not <ensures_expression>))
; (check-sat)
; (get-model)  ; for counterexample generation
```

### 7.3 Naming

SMT names are derived from source-language identifiers using a deterministic escaping scheme defined by the server. The server must document its naming scheme so external tools can correlate SMT names with source locations.

---

## 8. Implementation Guide

This section describes how to implement SemanticQuery for specific compiler infrastructures. The goal is to help compiler teams understand what data they already have and what additional work is needed.

### 8.1 LLVM / Clang

**Recommended implementation layer:** LLVM IR + Clang Static Analyzer.

Implementing SemanticQuery at the LLVM IR level gives the highest leverage: every language that compiles to LLVM IR gets SemanticQuery for free. Clang-specific additions (C/C++ types, C++26 contracts, Clang attributes) can be surfaced as extensions.

**Point Query data sources:**
- `type`: LLVM `Type*` for the IR value at the location, plus Clang's `QualType` for C/C++ source types
- `effects_accumulated`: LLVM function memory effect attributes (`readnone`, `readonly`, `writeonly`, `argmemonly`, `inaccessiblememonly`); `noexcept` for C++
- `borrow_state`: LLVM Alias Analysis results (`AliasResult` from `AAResults::alias()`); Clang LifetimeChecker results for C++ reference lifetime extension
- `active_contracts`: Clang `[[clang::nonnull]]`, `[[clang::returns_nonnull]]`; C++26 `pre:` / `post:` when available
- `unresolved`: Clang Static Analyzer bug reports (path-sensitive); Clang `-Wlifetime` warnings

**Counterfactual Query:** Inject a fact into CSA's `ProgramState` constraint map and re-run the checker from that point forward. CSA's `assumeTrue` / `assumeFalse` operations do approximately this already for internal use.

**SMT Export:** CSA's Z3 backend (`-analyzer-constraints=z3`) builds SMT constraint sets for path conditions. `alive2` encodes LLVM IR semantics in SMT-LIB for peephole verification. Either is a viable export source.

**Monitor:** LLVM instrumentation passes (ASan, MSan, UBSan, MemorySanitizer) are the natural Monitor implementation. SemanticQuery's Monitor condition is compiled to IR and inserted as an instrumentation pass.

---

### 8.2 Rust / rustc

**Recommended implementation layer:** rustc MIR pipeline, with Prusti or Creusot for contract verification.

**Point Query data sources:**
- `type`: `rustc_middle::ty::Ty` from the type checker
- `effects_accumulated`: not natively available; approximate with `unsafe` block presence, `Send`/`Sync` bounds. (Rust effect system is an open research question as of 2025.)
- `borrow_state`: MIR NLL (Non-Lexical Lifetimes) region inference results — `RegionInferenceContext`, `OutlivesConstraint` sets. This is the richest borrow state of any mainstream language.
- `active_contracts`: MIRAI annotations, Prusti `#[requires]`/`#[ensures]`, Kani `kani::assume` / `kani::assert` if those tools are active
- `unresolved`: NLL region errors (`RegionResolutionError`)

**Counterfactual Query:** Re-run NLL region inference with an injected `RegionConstraint`. The NLL solver is designed to be re-runnable on a constraint set; this is architecturally feasible and is the most valuable thing a Rust SemanticQuery implementation could offer.

**SMT Export:** Prusti translates Rust MIR to Viper, which translates to SMT via Z3. Creusot translates to Why3, which supports multiple backends. Either can serve as the SMT export path.

**Note on rust-analyzer:** rust-analyzer is not the right implementation layer. It does not fully reimplement the borrow checker and does not have access to NLL region inference results. The implementation must be at the rustc level.

---

### 8.3 Ada / SPARK (GNATprove)

**Recommended implementation layer:** GNATprove, with ASIS or Libadalang for syntactic queries.

Ada/SPARK is the most complete natural fit for SemanticQuery. SPARK 2022 added Rust-style ownership and borrowing. GNATprove already performs SMT verification via Why3/Z3. Approximately 80% of SemanticQuery's functionality already exists in GNATprove internals.

**Point Query data sources:**
- `type`: Ada type system via ASIS/Libadalang (`Ada.Semantic_Interface` or `Libadalang.Analysis`)
- `effects_accumulated`: SPARK data dependency contracts (`Global`, `Depends`)
- `borrow_state`: SPARK 2022 ownership annotations; GNATprove flow analysis results
- `active_contracts`: Ada `pragma Pre` / `pragma Post` / `pragma Contract_Cases`; SPARK `Global` / `Depends` / `Loop_Invariant`
- `unresolved`: GNATprove unproved checks

**Counterfactual Query:** Inject an `Assume` annotation into GNATprove's proof context. GNATprove's `pragma Assume` does approximately this for source-level use; the counterfactual query exposes it as a protocol operation without source modification.

**SMT Export:** GNATprove's `--proof-dir` exports Why3 verification conditions per subprogram. SemanticQuery's `smtExport` is a protocol wrapper around this existing capability.

---

### 8.4 Java / JVM (OpenJML, Checker Framework)

**Point Query data sources:**
- `type`: JDT `ITypeBinding` (Eclipse) or `javax.lang.model.type.TypeMirror` (javac API)
- `effects_accumulated`: Checker Framework `@SideEffectFree` / `@Pure` / `@Deterministic`
- `borrow_state`: Checker Framework `@MustCall` / `@Owning` / `@NotOwning` (Resource Leak Checker)
- `active_contracts`: JML `requires` / `ensures` / `invariant` clauses; Checker Framework annotations
- `unresolved`: OpenJML unproved VCs; NullAway / Checker Framework warnings

**SMT Export:** OpenJML already translates JML contracts to SMT-LIB for Z3 and CVC5. SemanticQuery's `smtExport` is a protocol wrapper.

---

### 8.5 Python (mypy / pyright)

Python has the thinnest static semantic model of any mainstream language. Most `borrow_state` and `effects_accumulated` fields will be empty. However, even the type narrowing information is useful, and the Counterfactual Query is valuable for type narrowing analysis.

**Point Query data sources:**
- `type`: mypy `Type` or pyright inferred type (may be `Unknown`, `str | None`, etc.)
- `effects_accumulated`: empty unless type stubs declare effects
- `borrow_state`: empty (Python has GC, no ownership)
- `active_contracts`: `icontract` or `beartype` decorator preconditions/postconditions
- `unresolved`: mypy/pyright type errors at this point; potential `None` dereferences

**Counterfactual Query for Python:** The most useful form is type narrowing: "if `x` is narrowed to `str` here (not `str | None`), does pyright accept the rest of the function?" Pyright's type narrowing analysis already does this internally when it sees `if x is not None:`. The counterfactual query exposes it without source modification.

---

### 8.6 Summary

| Implementation target | Point Query richness | Counterfactual feasibility | SMT Export | Existing infrastructure |
|---|---|---|---|---|
| LLVM IR | Rich (alias, UB, memory effects) | Feasible (CSA constraint injection) | alive2, CSA Z3 backend | Highest adoption leverage |
| Ada/SPARK | Full (ownership, contracts) | Already exists (pragma Assume) | GNATprove `--proof-dir` | 80% exists already |
| Rust/rustc | Rich (NLL borrow state) | Feasible (NLL re-run) | Prusti / Creusot | Large community |
| C++ / Clang | Moderate (CSA, alias analysis) | Feasible (CSA state injection) | CSA Z3 backend | C++26 contracts coming |
| Java / JVM | Moderate (JML, Checker Framework) | Feasible (CF inference re-run) | OpenJML already does it | Good existing tools |
| Python | Thin (type narrowing) | Useful (narrowing query) | Minimal | mypy / pyright |

---

## 9. Adoption Strategy

SemanticQuery is a protocol, not a product. It succeeds when independent compiler teams implement it independently because it is useful to them. The right adoption path is:

**1. Publish the spec and implement for one compiler.** A reference implementation demonstrates feasibility and exercises the protocol. The first implementation will reveal gaps in the spec; that is expected and useful.

**2. Target LLVM first for maximum leverage.** One implementation at the LLVM IR level benefits every LLVM-targeting language simultaneously. A `clang-sq` tool or `semanticQuery/` methods in clangd would reach C, C++, Rust (via clippy/rustc), Swift, Zig, Julia, and Objective-C users with a single implementation.

**3. Let Ada/SPARK follow naturally.** GNATprove already has 80% of the required data. An Ada SemanticQuery implementation is primarily integration work, not research. AdaCore is the natural owner.

**4. Do not port rust-analyzer.** rust-analyzer is not the right Rust implementation layer. The Rust implementation belongs in rustc's MIR pipeline or in Prusti/Creusot. Let the Rust community adopt SemanticQuery on their own terms.

**5. The counterfactual query is the differentiator.** Prior systems that implemented point queries (ASIS, Roslyn) were adopted because point queries are useful. Implementations that add counterfactual queries will be adopted for a qualitatively different reason: they enable automated reasoning that was not previously possible. Lead with this operation in implementation documentation.

---

## 10. Versioning

SemanticQuery follows the same versioning model as LSP.

- The `initialize` handshake negotiates a version via the `semanticQueryProvider.version` field.
- Breaking changes (removed fields, changed types, renamed operations) require a version bump.
- Additive changes (new optional fields, new `include` flags, new `Hypothesis` variants, new `MonitorAction` kinds) do not require a version bump.
- Servers must ignore unknown fields in requests. Clients must handle absent optional fields in responses.

Current version: **0.1** (draft, unstable).

Version 1.0 will be declared when at least two independent implementations exist and have interoperated successfully.

---

## 11. Acknowledgements

SemanticQuery stands on the shoulders of:

- **ASIS** (ISO/IEC 15291:1999) — the original vision of a standardized semantic interface into a compiler
- **Roslyn** — proved that "compiler as a service" creates an ecosystem, not just a tool
- **LSP** (Microsoft) — established JSON-RPC over stdio as the right transport for compiler-tool communication
- **GNATprove / SPARK** — demonstrated that verification condition export enables an ecosystem of external provers
- **Dafny** (Microsoft Research) — demonstrated verification state over LSP in a production language server
- **rust-analyzer** — established the LSP extension method pattern for compiler-specific capabilities
- **alive2** — demonstrated SMT-LIB encoding of IR semantics as a practical, production-quality approach

---

*SemanticQuery Protocol Specification, version 0.1 draft.*
