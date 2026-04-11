# SemanticQuery MCP Server Specification

**Version:** 0.1 (draft)
**Depends on:** SemanticQuery Protocol Specification 0.1

---

## Overview

The SemanticQuery MCP server bridges two protocols:

- **Inward-facing:** MCP (Model Context Protocol) — the protocol AI assistants use to invoke tools and read resources
- **Outward-facing:** SemanticQuery over LSP JSON-RPC — the protocol compilers use to expose semantic state

An AI agent that speaks MCP can ask a compiler what it knows at line 47, inject a hypothesis, and receive a structured verdict — without speaking raw JSON-RPC, without understanding LSP lifecycle, and without parsing human-readable compiler output.

```
AI agent (Claude, etc.)
    ↕  MCP (stdio / SSE)
SemanticQuery MCP Server
    ↕  SemanticQuery / LSP JSON-RPC (stdio / TCP)
Compiler (ferrum-lsp, clangd, rust-analyzer, gnatprove-sq, ...)
```

The MCP server is a thin, stateless adapter. It holds a persistent connection to one SemanticQuery server (or a pool of them), translates MCP tool calls into SemanticQuery requests, and returns results as MCP tool responses.

---

## MCP Concepts Used

**Tools** — functions the AI can call. SemanticQuery operations map to MCP tools. The AI invokes a tool with arguments; the MCP server executes the corresponding SemanticQuery request and returns the result.

**Resources** — data the AI can read by URI. Compiler state that is useful to read without a specific question (current diagnostics, all unresolved items in a file) is exposed as resources.

**Prompts** — named workflow templates. Multi-step SemanticQuery workflows (close a verification gap, generate tests for a module) are exposed as prompts so the AI can be handed a task description rather than having to construct the workflow from scratch.

---

## Configuration

The MCP server is configured with a connection to one or more SemanticQuery servers:

```json
{
  "servers": [
    {
      "name": "ferrum",
      "command": ["ferrum", "lsp"],
      "args": ["--workspace", "/path/to/project"],
      "transport": "stdio"
    },
    {
      "name": "clang",
      "command": ["clangd", "--background-index"],
      "transport": "stdio"
    }
  ],
  "default_server": "ferrum"
}
```

When a workspace has multiple languages, the MCP server routes requests to the appropriate SemanticQuery server based on file extension or explicit `server` parameter.

---

## Tools

### `point_query`

Ask the compiler what it knows at a specific source location.

**Input schema:**
```json
{
  "uri": "string — file URI (file:///absolute/path/to/file.ext)",
  "line": "integer — 0-based line number",
  "character": "integer — 0-based character offset",
  "include": {
    "type": "boolean (default true)",
    "effects": "boolean (default true)",
    "borrows": "boolean (default true)",
    "contracts": "boolean (default true)",
    "unresolved": "boolean (default true)",
    "callgraph": "boolean (default false)"
  },
  "server": "string (optional — which SemanticQuery server to use)"
}
```

**Output:** The full `PointQueryResult` from the SemanticQuery protocol. Key fields:
- `type` — type of the expression at this location
- `effects_accumulated` — effects active on all paths to this point
- `borrow_state` — aliasing and borrow relationships
- `active_contracts` — requires/ensures clauses and their verification status
- `unresolved` — items the compiler cannot determine without more information

**Example agent invocation:**

> "What does the compiler know about the state at line 47, character 12 in src/parser.rs?"

```json
{
  "tool": "point_query",
  "arguments": {
    "uri": "file:///home/user/project/src/parser.rs",
    "line": 46,
    "character": 12
  }
}
```

---

### `counterfactual_query`

Inject a hypothetical invariant and ask the compiler whether it resolves an open verification question — without modifying the source.

**Input schema:**
```json
{
  "uri": "string",
  "line": "integer",
  "character": "integer",
  "hypothesis": {
    "kind": "invariant | region_bound | effect_bound | alias_fact | type_narrowing | annotation",
    "...": "kind-specific fields (see SemanticQuery spec §5.2)"
  },
  "server": "string (optional)"
}
```

**Output:** `CounterfactualQueryResult`. Key fields:
- `resolves` — whether the hypothesis closes at least one unresolved item
- `resolved_items` — which unresolved items are closed
- `suggested_source_edit` — the concrete source change to make this hypothesis permanent, if resolves is true
- `counterexample` — a witness to why the hypothesis does not help, if resolves is false

**Example agent invocation:**

> "If the return value borrows from region 'outer, does the lifetime ambiguity at line 38 resolve?"

```json
{
  "tool": "counterfactual_query",
  "arguments": {
    "uri": "file:///home/user/project/src/parser.rs",
    "line": 37,
    "character": 0,
    "hypothesis": {
      "kind": "region_bound",
      "place": "return",
      "region": "'outer"
    }
  }
}
```

---

### `install_monitor`

Place a semantically-aware trap in a compiled binary. The condition is expressed in the source language; the compiler compiles it with full type awareness.

**Input schema:**
```json
{
  "uri": "string",
  "line": "integer",
  "character": "integer",
  "condition": "string — source-language boolean expression",
  "action": {
    "kind": "log | abort | report | capture",
    "...": "kind-specific fields"
  },
  "label": "string — identifier for this monitor",
  "server": "string (optional)"
}
```

**Output:** `InstallMonitorResult`. Returns a `monitor_id` for later removal. Sets `compile_needed: true` if the binary must be recompiled to install the monitor.

---

### `remove_monitor`

Remove a previously installed monitor.

**Input schema:**
```json
{
  "monitor_id": "string",
  "server": "string (optional)"
}
```

---

### `generate_scaffolding`

Generate a test harness from the compiler's knowledge of a function's contracts and types.

**Input schema:**
```json
{
  "uri": "string",
  "line": "integer — any line inside the target function",
  "style": "unit_test | property_test | fuzz_target | smt_harness",
  "language": "string (optional — target language for the harness if different from source)",
  "server": "string (optional)"
}
```

**Output:** `ScaffoldingResult`. The `generated_source` field contains ready-to-run test code. The `preconditions` and `postconditions` fields list the contracts the harness is derived from.

---

### `export_smt`

Export verification conditions as SMT-LIB 2 for an external solver.

**Input schema:**
```json
{
  "uri": "string",
  "line": "integer — any line inside the target function",
  "scope": "full_function | contracts_only | loop_invariants | single_vc",
  "include_background": "boolean (default true)",
  "server": "string (optional)"
}
```

**Output:** `SmtExportResult`. The `encoding` field contains the SMT-LIB 2 text, ready to pipe to `z3`, `cvc5`, `yices-smt2`, or similar.

---

### `list_unresolved`

Convenience tool: return all unresolved items across an entire file or workspace. Useful for getting a full picture of what the compiler cannot determine before deciding where to focus.

**Input schema:**
```json
{
  "uri": "string — file URI, or workspace root URI",
  "kinds": ["region", "effect", "contract", "alias", "type"],
  "server": "string (optional)"
}
```

**Output:**
```json
{
  "unresolved": [
    {
      "location": { "uri": "...", "position": { "line": 47, "character": 12 } },
      "kind": "region",
      "description": "lifetime of return value is ambiguous",
      "hint": "annotate return type with an explicit lifetime"
    }
  ],
  "total_count": 3
}
```

This tool is implemented at the MCP server level by iterating `point_query` over compiler diagnostics; it is not a direct SemanticQuery protocol operation.

---

### `resolve_unresolved`

High-level tool: given one unresolved item, automatically iterate over candidate hypotheses and return the winning annotation. This implements the counterfactual query loop that would otherwise require multiple agent tool calls.

**Input schema:**
```json
{
  "uri": "string",
  "line": "integer",
  "character": "integer",
  "unresolved_description": "string (optional — from list_unresolved, to focus the search)",
  "max_attempts": "integer (default 20)",
  "server": "string (optional)"
}
```

**Output:**
```json
{
  "resolved": true,
  "winning_hypothesis": { "kind": "region_bound", "place": "return", "region": "'outer" },
  "suggested_source_edit": {
    "uri": "...",
    "range": { "start": {...}, "end": {...} },
    "new_text": "&'outer str"
  },
  "attempts": 3,
  "rejected_hypotheses": [
    {
      "hypothesis": { "kind": "region_bound", "place": "return", "region": "'static" },
      "reason": "counterexample: value does not live long enough"
    }
  ]
}
```

Or, if no hypothesis resolves within `max_attempts`:

```json
{
  "resolved": false,
  "attempts": 20,
  "closest_hypothesis": { "kind": "region_bound", "place": "return", "region": "'input" },
  "closest_remaining_unresolved": 1,
  "recommendation": "human judgment required — the region constraint involves a non-obvious aliasing relationship"
}
```

This tool is implemented at the MCP server level. It runs the counterfactual query loop internally, trying hypothesis templates in a defined order, without requiring the AI to manage the iteration.

---

## Resources

MCP resources are read by URI without invoking a tool. They provide ambient compiler state the AI can access at any time.

### `semanticquery://capabilities`

The capabilities of the connected SemanticQuery server(s).

```json
{
  "servers": [
    {
      "name": "ferrum",
      "version": "0.1",
      "capabilities": {
        "pointQuery": true,
        "counterfactualQuery": true,
        "monitor": true,
        "scaffolding": true,
        "smtExport": true
      },
      "languages": ["fe"]
    }
  ]
}
```

### `semanticquery://file/{encoded_uri}/diagnostics`

All current diagnostics for a file, including SemanticQuery unresolved items alongside standard LSP diagnostics.

```json
{
  "uri": "file:///src/parser.rs",
  "diagnostics": [
    {
      "range": { "start": {"line": 38, "character": 0}, "end": {"line": 38, "character": 30} },
      "severity": "error",
      "source": "semanticquery",
      "code": "unresolved_region",
      "message": "lifetime of return value is ambiguous",
      "hint": "annotate return type with an explicit lifetime"
    }
  ]
}
```

### `semanticquery://file/{encoded_uri}/contracts`

All contracts declared in a file and their current verification status.

```json
{
  "uri": "file:///src/search.fe",
  "contracts": [
    {
      "function": "binary_search",
      "location": { "line": 51, "character": 0 },
      "requires": [
        { "expression": "arr.is_sorted()", "status": "runtime_checked" }
      ],
      "ensures": [
        { "expression": "match result { Some(i) => arr[i] == *target, None => !arr.contains(target) }",
          "status": "runtime_checked" }
      ]
    }
  ]
}
```

### `semanticquery://workspace/unresolved`

All unresolved items across the entire workspace, grouped by file.

```json
{
  "total": 7,
  "by_file": [
    {
      "uri": "file:///src/parser.rs",
      "unresolved": [...]
    }
  ]
}
```

---

## Prompts

MCP prompts are named workflow templates the AI can invoke by name. They describe a multi-step task in terms of which tools to use and in what order.

### `close_verification_gap`

Close all unresolved items in a function or file.

**Arguments:**
- `uri` — file to work on
- `scope` — `"function"` (focus on one function, requires line/character) or `"file"`

**Workflow the prompt describes:**
1. Use `list_unresolved` to get all unresolved items in scope
2. For each unresolved item, use `resolve_unresolved` to find the winning annotation
3. Apply `suggested_source_edit` for each resolved item
4. Re-run `list_unresolved` to verify the edits closed the gaps
5. Report items that could not be resolved automatically (require human judgment)

---

### `generate_tests`

Generate a full test suite for a module from its contracts.

**Arguments:**
- `uri` — file containing the module
- `style` — `"unit_test"`, `"property_test"`, or `"fuzz_target"`

**Workflow the prompt describes:**
1. Scan the file for all functions with contracts (via `semanticquery://file/{uri}/contracts`)
2. For each function with contracts, use `generate_scaffolding`
3. Collect all generated test sources into a single test file
4. Report functions that have no contracts and could not generate meaningful scaffolding

---

### `audit_verification_status`

Produce a structured report of what is verified, what is runtime-checked, and what is unresolved — for security audit or compliance purposes.

**Arguments:**
- `uri` — workspace root

**Workflow the prompt describes:**
1. Read `semanticquery://workspace/unresolved`
2. Read `semanticquery://file/{uri}/contracts` for each file
3. Group by status: statically verified / runtime checked / unresolved / no contracts
4. Produce a markdown summary table with counts and source locations

---

### `explain_borrow_error`

Given a borrow checker error at a location, explain it and propose fixes.

**Arguments:**
- `uri` — file
- `line`, `character` — location of the error

**Workflow the prompt describes:**
1. Use `point_query` to get the full semantic state at the error location
2. Use `counterfactual_query` to probe which hypotheses would resolve the error
3. Explain in plain language: what the borrow checker sees, why it cannot resolve it, what the winning annotation means
4. Apply the `suggested_source_edit` if approved

---

## Example: The Full Agent Workflow

This sequence shows how an AI agent uses the MCP tools to close a verification gap without human involvement after the initial task assignment.

**Task:** "The function `parse_header` at line 38 of `src/parser.rs` has a lifetime error. Fix it."

```
Agent → MCP: point_query(uri="file:///src/parser.rs", line=38, character=0)

MCP → SemanticQuery server: semanticQuery/pointQuery
SemanticQuery server → MCP: {
  type: "&'_#3r [u8]",
  borrow_state: [{ place: "buf", kind: "exclusive", region: "'_#3r" }],
  unresolved: [{ kind: "region",
                 description: "lifetime of return value is ambiguous",
                 hint: "annotate return type with an explicit lifetime" }]
}
MCP → Agent: [full PointQueryResult]

Agent understands: the return lifetime is ambiguous between two candidates.
Agent → MCP: counterfactual_query(line=38, hypothesis={kind: "region_bound", place: "return", region: "'_#3r"})

MCP → SemanticQuery server: semanticQuery/counterfactualQuery
SemanticQuery server → MCP: { resolves: false, counterexample: "buf does not live long enough" }
MCP → Agent: [CounterfactualQueryResult, resolves=false]

Agent tries the other candidate.
Agent → MCP: counterfactual_query(line=38, hypothesis={kind: "region_bound", place: "return", region: "'input"})

MCP → SemanticQuery server: semanticQuery/counterfactualQuery
SemanticQuery server → MCP: {
  resolves: true,
  suggested_source_edit: { range: {...}, new_text: "&'input [u8]" }
}
MCP → Agent: [CounterfactualQueryResult, resolves=true, edit confirmed]

Agent writes the annotation. Done. No human involvement after initial task assignment.
```

The same workflow using `resolve_unresolved` directly:

```
Agent → MCP: resolve_unresolved(uri=..., line=38)
[MCP server runs the loop internally]
MCP → Agent: { resolved: true, suggested_source_edit: { new_text: "&'input [u8]" }, attempts: 2 }

Agent writes the annotation.
```

---

## Implementation Notes

### Connecting to the SemanticQuery server

The MCP server launches the SemanticQuery server as a subprocess (for stdio transport) or connects to it over TCP. It sends the LSP `initialize` request, checks `semanticQueryProvider` capabilities, and holds the session open for the duration of the MCP server's lifetime.

If the SemanticQuery server crashes or disconnects, the MCP server relaunches it and re-initializes. It does not propagate the crash to the AI client; instead it retries and reports a transient error if retries fail.

### Document synchronization

The MCP server must keep the SemanticQuery server's document state synchronized with the actual files on disk. It watches the workspace for file changes and sends `textDocument/didChange` notifications to the SemanticQuery server. Without this, point queries return stale results.

### Caching

Point query results are cached by (uri, version, position) and invalidated on `textDocument/didChange`. Counterfactual query results are not cached (they depend on hypothesis content and may change as the document changes).

### The `resolve_unresolved` hypothesis search order

The built-in hypothesis search in `resolve_unresolved` tries templates in this order:

1. Region bounds suggested by the unresolved item's `hint` field
2. Common region candidates: `'static`, input parameter regions (by parameter order), return region
3. Effect bounds: empty effect set, then effects of called functions
4. Alias facts: `no_alias` between all pairs of pointer-typed places at the location
5. Type narrowings: non-null, non-empty, sorted (for collection types)
6. Raw annotations: common annotation patterns for the language (language-server-specific)

The search stops at the first hypothesis where `resolves == true`. If no hypothesis resolves within `max_attempts`, the tool reports the closest result (fewest remaining unresolved items) and sets `recommendation: "human judgment required"`.

### Error handling

If the SemanticQuery server returns `LocationNotAnalyzed`, the MCP server triggers a re-analysis (by sending a `workspace/diagnostic` request) and retries the query once. If the retry also fails, it returns a tool error explaining that the location has not been analyzed yet.

---

## Relationship to SemanticQuery Protocol

The MCP server does not extend the SemanticQuery protocol. It is a client of the SemanticQuery protocol. The five base MCP tools (`point_query`, `counterfactual_query`, `install_monitor`, `generate_scaffolding`, `export_smt`) are direct wrappers of the five SemanticQuery operations. The higher-level tools (`list_unresolved`, `resolve_unresolved`) and the prompts are implemented at the MCP server layer, on top of the base operations.

This layering is intentional. The SemanticQuery protocol stays minimal and implementable. The MCP server adds the workflow intelligence — the hypothesis search loop, the document watching, the routing between multiple servers — without burdening the protocol with it.

---

*SemanticQuery MCP Server Specification, version 0.1 draft.*
*See SemanticQuery-protocol-spec.md for the underlying protocol.*
