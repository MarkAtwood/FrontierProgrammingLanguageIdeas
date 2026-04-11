# SemanticQuery Protocol Specification

SemanticQuery is a protocol for querying a compiler's internal semantic and verification state. It is designed to be implementable by any compiler infrastructure — Ferrum, LLVM/clang, GCC, rustc — and to be useful to any tool that needs to reason about programs at the semantic level: editors, AI agents, formal verifiers, fuzz harnesses, and debuggers.

SemanticQuery is specified as a set of JSON-RPC methods that extend the Language Server Protocol. Implementations may also speak SemanticQuery standalone over stdio or TCP without an LSP session.

---

## Motivation

The Language Server Protocol (LSP) gives tools access to the syntactic and basic semantic information a compiler holds: types on hover, diagnostics, go-to-definition, completions. This is sufficient for human-directed editing but insufficient for automated reasoning.

A tool that wants to close a verification gap needs to know not just "what is the type here" but:

- What is the borrow checker's aliasing picture at this point?
- What effects have accumulated through this call path?
- What contracts are active and which are unresolved?
- If a proposed invariant held here, would the remaining verification succeed?
- What is the SMT encoding of this function's verification conditions?

SemanticQuery is the protocol for these questions. It is a natural extension to LSP, not a replacement. An LSP session that also speaks SemanticQuery covers both the human-editor and the automated-reasoning cases.

---

## Relationship to LSP

SemanticQuery methods use the `semanticQuery/` request namespace. They are sent over the same JSON-RPC channel as LSP requests, using the same `initialize`/`shutdown` lifecycle. A server announces SemanticQuery support in the `capabilities` response to `initialize`:

```json
{
  "capabilities": {
    "semanticQueryProvider": {
      "pointQuery": true,
      "counterfactualQuery": true,
      "monitor": true,
      "scaffolding": true,
      "smtExport": true
    }
  }
}
```

A client that sees `semanticQueryProvider` in the server capabilities may send SemanticQuery requests. A client that does not see it falls back to LSP-only behavior.

---

## Common Types

```typescript
interface Location {
  uri: string;       // document URI
  position: Position;  // LSP Position: { line: number, character: number }
}

interface Range {
  start: Position;
  end: Position;
}

// A borrow at a program point
interface BorrowState {
  place: string;          // e.g. "data", "self.buf", "*ptr"
  kind: "shared" | "exclusive" | "moved";
  region: string;         // region name, e.g. "'outer", "'static", "'_"
  source_span: Range;     // where the borrow was introduced
}

// An active contract at a program point
interface ActiveContract {
  kind: "requires" | "ensures" | "invariant";
  expression: string;     // the contract expression in Ferrum source syntax
  status: "verified" | "runtime_checked" | "unresolved";
  source_span: Range;
}

// A region constraint visible at a point
interface RegionConstraint {
  region: string;
  outlives: string[];     // regions this region must outlive
  scope: Range;           // span of source where this region is live
}

// An unresolved item — something the compiler cannot determine without more info
interface Unresolved {
  kind: "region" | "effect" | "contract" | "borrow";
  description: string;
  hint: string;           // what kind of annotation or invariant would resolve it
}
```

---

## Operations

### 1. Point Query — `semanticQuery/pointQuery`

Ask the compiler what it knows at a specific location in the source.

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
    callgraph?: boolean;       // default false — outgoing calls and their effects
  };
}
```

**Response:**

```typescript
interface PointQueryResult {
  location: Location;
  type: string | null;                   // Ferrum type of expression at this point
  effects_accumulated: string[];         // effects active on the path to this point
  effects_declared: string[];            // effects declared on enclosing function
  borrow_state: BorrowState[];
  region_constraints: RegionConstraint[];
  active_contracts: ActiveContract[];
  unresolved: Unresolved[];
  callgraph?: CallSite[];
}

interface CallSite {
  callee: string;
  location: Location;
  effects: string[];
}
```

**Example:**

```json
// Request
{
  "method": "semanticQuery/pointQuery",
  "params": {
    "location": { "uri": "file:///src/search.fe", "position": { "line": 46, "character": 14 } }
  }
}

// Response
{
  "location": { "uri": "file:///src/search.fe", "position": { "line": 46, "character": 14 } },
  "type": "Option[usize]",
  "effects_accumulated": ["IO"],
  "effects_declared": ["IO"],
  "borrow_state": [
    { "place": "arr", "kind": "shared", "region": "'input", "source_span": { "start": {...}, "end": {...} } }
  ],
  "region_constraints": [
    { "region": "'input", "outlives": ["'static"], "scope": { "start": {...}, "end": {...} } }
  ],
  "active_contracts": [
    { "kind": "requires", "expression": "arr.is_sorted()", "status": "runtime_checked", "source_span": {...} }
  ],
  "unresolved": []
}
```

---

### 2. Counterfactual Query — `semanticQuery/counterfactualQuery`

Inject a hypothetical invariant or annotation and ask whether it resolves open verification questions — without modifying the source. This is the key operation for automated reasoning: an agent proposes candidates until one resolves the gap, then writes only the winning annotation.

**Request:**

```typescript
interface CounterfactualQueryParams {
  location: Location;
  hypothesis: Hypothesis;
  // The compiler evaluates: "if this hypothesis held at this location,
  // what would change in the verification state?"
}

type Hypothesis =
  | { kind: "invariant";    expression: string }
    // e.g. { kind: "invariant", expression: "ptr != null" }
  | { kind: "region_bound"; place: string; region: string }
    // e.g. { kind: "region_bound", place: "return", region: "'outer" }
  | { kind: "effect_bound"; expression: string; effects: string[] }
    // e.g. { kind: "effect_bound", expression: "helper()", effects: [] }
  | { kind: "annotation";   source: string }
    // raw Ferrum annotation to hypothetically apply
```

**Response:**

```typescript
interface CounterfactualQueryResult {
  hypothesis: Hypothesis;
  resolves: boolean;             // does the hypothesis close at least one Unresolved?
  resolved_items: Unresolved[];  // which Unresolved items this hypothesis closes
  remaining_unresolved: Unresolved[];
  new_unresolved: Unresolved[];  // unresolved items the hypothesis introduces
  suggested_source_edit: SourceEdit | null;  // concrete edit if resolves == true
  counterexample: Counterexample | null;     // witness to why resolves == false
}

interface SourceEdit {
  range: Range;
  new_text: string;
}

interface Counterexample {
  description: string;    // human-readable explanation
  witness: string | null; // SMT-LIB witness term, if available
}
```

**Example:**

```json
// Request: "if the return value borrows from 'outer, does the lifetime ambiguity resolve?"
{
  "method": "semanticQuery/counterfactualQuery",
  "params": {
    "location": { "uri": "file:///src/parse.fe", "position": { "line": 23, "character": 0 } },
    "hypothesis": { "kind": "region_bound", "place": "return", "region": "'outer" }
  }
}

// Response
{
  "hypothesis": { "kind": "region_bound", "place": "return", "region": "'outer" },
  "resolves": true,
  "resolved_items": [
    { "kind": "region", "description": "return value region is ambiguous", "hint": "annotate return with a region" }
  ],
  "remaining_unresolved": [],
  "new_unresolved": [],
  "suggested_source_edit": {
    "range": { "start": { "line": 22, "character": 15 }, "end": { "line": 22, "character": 18 } },
    "new_text": "&'outer str"
  },
  "counterexample": null
}
```

---

### 3. Monitor — `semanticQuery/installMonitor` / `semanticQuery/removeMonitor`

Place a semantically-aware trap in a compiled binary. The condition is expressed in Ferrum's type language; the compiler weaves it into the binary correctly, handling memory layout, borrow semantics, and type coercions. This is stronger than a raw debugger watchpoint: the compiler knows what the condition *means* in the program's semantic model.

**Install request:**

```typescript
interface InstallMonitorParams {
  location: Location;
  condition: string;         // Ferrum expression evaluated at this point
  action: MonitorAction;
  label: string;             // human-readable name for this monitor
}

type MonitorAction =
  | { kind: "log";    message: string }
  | { kind: "abort";  message: string }
  | { kind: "report"; channel: string }   // send to a named reporting channel
  | { kind: "capture"; variables: string[] }  // capture named places for inspection
```

**Install response:**

```typescript
interface InstallMonitorResult {
  monitor_id: string;
  compile_needed: boolean;  // true if binary must be recompiled to install monitor
  warnings: string[];       // e.g. "condition references moved value"
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
// Install: fire if a null pointer is observed at the return of `parse_header`
{
  "method": "semanticQuery/installMonitor",
  "params": {
    "location": { "uri": "file:///src/parse.fe", "position": { "line": 89, "character": 0 } },
    "condition": "result.is_none() && input.len() > 0",
    "action": { "kind": "report", "channel": "monitor_events" },
    "label": "parse_header_unexpected_none"
  }
}
```

---

### 4. Scaffolding — `semanticQuery/scaffolding`

Generate test scaffolding from the compiler's knowledge of a function's contracts. Every `requires` clause becomes a precondition the harness must satisfy; every `ensures` clause becomes an oracle the harness asserts. The compiler generates this mechanically — it already has the information.

**Request:**

```typescript
interface ScaffoldingParams {
  function_location: Location;  // any location inside the function
  style: ScaffoldingStyle;
}

type ScaffoldingStyle =
  | "unit_test"        // Ferrum #[test] function with handwritten inputs
  | "property_test"    // property-based test with generated inputs
  | "fuzz_target"      // libFuzzer/AFL-compatible harness
  | "smt_harness"      // SMT-LIB harness for solver-based testing
```

**Response:**

```typescript
interface ScaffoldingResult {
  function_name: string;
  preconditions: string[];    // requires expressions
  postconditions: string[];   // ensures expressions
  generated_source: string;   // Ferrum source code for the test
  notes: string[];            // e.g. "precondition arr.is_sorted() requires a generator"
}
```

**Example:**

```json
// Request scaffolding for binary_search
{
  "method": "semanticQuery/scaffolding",
  "params": {
    "function_location": { "uri": "file:///src/search.fe", "position": { "line": 51, "character": 4 } },
    "style": "property_test"
  }
}

// Response (generated_source is Ferrum code)
{
  "function_name": "binary_search",
  "preconditions": ["arr.is_sorted()"],
  "postconditions": [
    "match result { Some(i) => arr[i] == *target, None => !arr.contains(target) }"
  ],
  "generated_source": "
#[test(property, runs = 10000)]
fn prop_binary_search[T: Ord + Arbitrary]() {
    let mut arr: Vec[T] = Arbitrary::generate();
    arr.sort();
    let target: T = Arbitrary::generate();
    let result = binary_search(&arr, &target);
    match result {
        Some(i) => assert!(arr[i] == target),
        None    => assert!(!arr.contains(&target)),
    }
}
",
  "notes": []
}
```

---

### 5. SMT Export — `semanticQuery/smtExport`

Export the compiler's verification conditions for a function as SMT-LIB 2. The compiler's internal pipeline generates these conditions for Boogie/Z3; this operation exposes them for use with any SMT-LIB–compatible solver: Z3, CVC5, Yices2, Bitwuzla, OpenSMT.

This makes the verification pipeline composable. Tools are not locked to the compiler's internal solver choice.

**Request:**

```typescript
interface SmtExportParams {
  function_location: Location;
  scope: SmtExportScope;
  include_background?: boolean;   // include standard prelude (default true)
}

type SmtExportScope =
  | "full_function"        // all VCs for the function
  | "contracts_only"       // only requires/ensures VCs, no body
  | "loop_invariants"      // only loop invariant VCs
  | "single_vc";           // VC nearest to function_location position
```

**Response:**

```typescript
interface SmtExportResult {
  function_name: string;
  format: "smtlib2";
  solver_logic: string;        // SMT-LIB logic declaration, e.g. "AUFLIA", "QF_LIA"
  encoding: string;            // the SMT-LIB 2 text
  vc_count: number;            // number of verification conditions encoded
  notes: string[];             // e.g. "loop invariant not provided; VC is unprovable without one"
}
```

**Example:**

```json
// Request
{
  "method": "semanticQuery/smtExport",
  "params": {
    "function_location": { "uri": "file:///src/search.fe", "position": { "line": 51, "character": 4 } },
    "scope": "contracts_only"
  }
}

// Response
{
  "function_name": "binary_search",
  "format": "smtlib2",
  "solver_logic": "AUFLIA",
  "encoding": "(set-logic AUFLIA)\n(declare-sort Elem 0)\n(declare-fun is_sorted (Array Int Elem) Bool)\n(declare-fun contains (Array Int Elem) Bool)\n...\n(assert (is_sorted arr))\n(assert (not (=> (= result_tag 1) (= (select arr result_idx) target))))\n(check-sat)\n(get-model)\n",
  "vc_count": 2,
  "notes": []
}
```

---

## LSP Baseline

A Ferrum language server that implements SemanticQuery also implements the full LSP baseline. The LSP methods are standard; SemanticQuery adds the verification-specific operations on top.

| LSP Method | Ferrum behavior |
|---|---|
| `textDocument/hover` | Type, effects, and active contracts for the hovered expression |
| `textDocument/diagnostic` | Type errors, borrow errors, effect violations, contract violations |
| `textDocument/inlayHint` | Inferred effects and region annotations (where suppressed by inference) |
| `textDocument/codeAction` | "Add annotation" quick fixes suggested by counterfactual queries |
| `textDocument/completion` | Context-aware; respects effects at point (no Net completions in pure context) |

The `textDocument/hover` response for Ferrum extends the standard hover with SemanticQuery data:

```json
{
  "contents": {
    "kind": "markdown",
    "value": "**Type:** `Option[usize]`\n**Effects:** `! IO`\n**Contracts:** `requires arr.is_sorted()` ✓ runtime-checked"
  }
}
```

---

## SMT-LIB Standard Integration

The `semanticQuery/smtExport` operation produces standard SMT-LIB 2 output. Any SMT-LIB–compatible solver can consume it directly:

```bash
# Export VCs for a function
ferrum lsp --stdio < query.json > result.json

# Or via CLI shortcut
ferrum semquery smt-export src/search.fe:51 > search_vcs.smt2

# Run with alternative solvers
z3 search_vcs.smt2
cvc5 search_vcs.smt2
yices-smt2 search_vcs.smt2
bitwuzla search_vcs.smt2
```

The solver logic declaration (`AUFLIA`, `QF_LIA`, `QF_BV`, etc.) is chosen by the compiler based on the features used in the verification conditions. Tools that need a specific logic can request it via the `solver_logic` hint (not yet specified; future extension).

---

## Implementing SemanticQuery in Other Compilers

SemanticQuery is defined independently of Ferrum. Any compiler can implement it by exposing its internal semantic state through these five methods.

For LLVM/clang:
- Point Query maps to clang's `ASTContext` + dataflow analysis state
- Counterfactual Query maps to a hypothetical dataflow re-run with injected facts
- SMT Export maps to clang's existing Static Analyzer constraint sets

For rustc:
- Point Query maps to rustc's query system (`ty`, `effects`, MIR borrow check state)
- Counterfactual Query maps to a scoped re-run of region inference with injected constraints
- SMT Export maps to a Charon/Prusti-compatible encoding of MIR

The data model (types, effects, borrow state, contracts) is language-specific, but the *protocol shape* — request a location, get back structured semantic state, inject a hypothesis, get a verdict — is language-independent.

---

## Versioning

SemanticQuery follows the same versioning model as LSP: the `initialize` handshake negotiates a version. Breaking changes require a version bump. Additive changes (new optional fields, new `include` flags) do not.

Current version: `0.1`.

---

*See also: `ferrum-lang-verification.md` for Ferrum's contract and proof system. `ferrum-plan.md` for the implementation roadmap. `ferrum-learning-for-ai.md` for the language reference agents use with this protocol.*
