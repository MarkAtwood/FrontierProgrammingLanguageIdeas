# Ferrum Language Reference — Compiler Internals

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Compiler Internals (Implementer Notes)

This section is for compiler and standard library authors. Users do not need to read it.

### 1.1 Compilation Pipeline

```
Source (.fe)
    ↓
Lexer → Token stream
    ↓
Parser → AST
    ↓
Name resolution → resolved AST
    ↓
Type inference → typed AST
    ↓
Effect inference → effect-annotated typed AST
    ↓
Region inference → region-annotated typed AST
    ↓
Borrow check → borrow-checked typed AST
    ↓
Contract elaboration → contracts expanded, old() snapshots inserted
    ↓
Dataflow analysis → initialized/moved/aliased state verified
    ↓
Monomorphization → concrete typed IR
    ↓
Layout resolution → layout declarations applied
    ↓
Codegen (C or LLVM IR)
    ↓
Native binary
```

Proof mode inserts an additional pass between contract elaboration and codegen:

```
SMT discharge → residual obligations
    ↓
Proof term checking → verified
```

### 1.2 Region Inference Algorithm

Region inference is a constraint-based algorithm over the borrow graph.

**Phase 1: Variable assignment.** Assign a fresh region variable `ρᵢ` to every reference type in the current function's signature and body.

**Phase 2: Constraint generation.** Walk the function body generating:
- `ρᵢ ⊇ ρⱼ` (region i outlives region j) from assignments `let x: &'ρᵢ T = y` where `y: &'ρⱼ T`
- `ρᵢ ⊇ {s}` (region i contains scope s) from borrows `&expr` where `expr` is live in scope `s`
- `ρᵢ ⊇ ρⱼ ∪ ρₖ` from control flow joins (if-else, loop back-edges)

**Phase 3: Constraint solving.** Solve the constraint system to the **minimal** valid assignment (smallest regions). Use a worklist algorithm. Cycles indicate self-referential borrows (allowed for `pinned` types, error otherwise).

**Phase 4: Error reporting.** If no valid assignment exists, report the specific constraint that is unsatisfiable. The error message should explain in terms of the user's code, not the constraint system internals.

**Phase 5: Annotation inference.** For `pub` functions, synthesize lifetime annotations from the solved regions to produce the canonical signature. These are the annotations a user would write if they were writing the function with explicit annotations.

**Annotation trigger:** The compiler requires explicit annotations when the solved region assignment for a return type is ambiguous — i.e., when the return region is not uniquely determined by a single input region. In this case the compiler reports: "cannot determine which input lifetime this return value borrows from; add explicit annotations."

### 1.3 Effect Inference Algorithm

Effects are inferred bottom-up. Each expression has an effect set. A function's effect set is the union of its body's effect sets. Pure functions have an empty effect set.

**Built-in effect sources:**
- `std.io.*` calls → `IO`
- `std.net.*` calls → `Net`
- `std.sync.Mutex.lock()` etc. → `Sync`
- Any `unsafe` block → `Unsafe`
- `panic!()` → `Panic`
- Any allocation via ambient allocator → `Alloc[A]`

Effects compose by union. Effect polymorphism (`[eff]`) is resolved at instantiation.

A function declared without a `!` clause is verified to have an empty effect set. Any call to an effectful function inside a pure function is a compile error.

### 1.4 Borrow Checker

The borrow checker runs after region inference. Its job is to verify:

1. No two `&mut` borrows to the same place are live simultaneously.
2. No `&` and `&mut` borrow to the same place are live simultaneously.
3. No borrow outlives its referent.
4. No use of a moved value.
5. No use of a potentially-uninitialized value.

The checker operates on the MIR (Mid-level Intermediate Representation) using NLL (Non-Lexical Lifetimes) with region annotations from the inference phase.

**`unchecked` blocks:** Rules 1 and 2 are suspended within `unchecked` blocks. Rules 3, 4, and 5 remain active.

**Dataflow:** Rules 4 and 5 are verified by forward dataflow analysis tracking initialization and move state per variable, per control flow path.

### 1.5 Capability Resolution

Capabilities are resolved by walking the scope chain outward from the call site, looking for the innermost `with X as cap` block providing a capability matching the required type and bound. If no explicit `with` block is found, the compiler uses the registered default for that capability type.

**Default capability registry** (mutable at crate level):

```ferrum
// In package root
default_capability Allocator = Heap
```

**Capability inference for tasks:** Before accepting a `spawn` call, the compiler checks that every capability required by the spawned function either (a) has a static lifetime, (b) is re-declared inside the spawn expression, or (c) is proven by the region system to outlive the scope's task join point.

### 1.6 Layout Verification Algorithm

For each `layout L for Type`:

1. Collect all named fields from `Type`'s definition.
2. Collect all placements from the `layout` block.
3. Verify bijection: every field appears exactly once; no field is absent; no unknown names appear.
4. Build a bit-level occupancy map of size `total_size`.
5. For each placement, mark the corresponding bits occupied. Error on overlap.
6. Verify all bits in `[0, total_size)` are occupied. Error on gap.
7. For each field, verify the bit width of the placement is compatible with the field's type (i.e., the type can represent all values encodable in that many bits).
8. For multi-byte fields, verify alignment or record as `unaligned`.
9. Generate read/write/view functions.

### 1.7 Contract Elaboration

In debug mode:

- `requires P` → `assert(P, "precondition violated: P")` at function entry.
- `ensures Q` → at every return site, `assert(Q, "postcondition violated: Q")`.
- `old(expr)` → `let __old_expr = snapshot(expr)` at function entry; references to `old(expr)` in `ensures` replaced with `__old_expr`.
- `snapshot(expr)` deep-copies the value (clones). For types implementing `Copy`, this is a copy. For others, `Clone` is called.
- Type `invariant I` → after every `&mut self` method body, `assert(self.I, "invariant violated: I")`.

In proof mode:

- `requires P` → a proof obligation discharged by the SMT solver or a proof term.
- `ensures Q` → a proof obligation discharged from the function body.
- `old(expr)` → symbolic pre-state.
- Type `invariant I` → proof obligation on every mutating method.

### 1.8 Monomorphization

Generic functions are monomorphized: a concrete copy is generated for each distinct instantiation. The compiler performs:

1. Collect all instantiations from the call graph.
2. For each distinct set of concrete type arguments, generate a concrete copy of the function with type parameters substituted.
3. Run the borrow checker and dataflow analysis on each concrete copy.
4. Generate code for each concrete copy.

Recursive generics (where a function instantiates itself with different types) are an error.

The monomorphization boundary for `dyn Trait` is a vtable. `dyn Trait` functions are not monomorphized; they dispatch at runtime through the vtable.

### 1.9 The Proof Interpreter

For proof functions evaluated at compile time (`@evaluate`) or in proof-mode runtime (`ferrum run --proof-mode`), the compiler uses a termination-safe interpreter:

- No mutation of external state.
- Structural recursion depth bound (configurable, default 10,000).
- All arithmetic in arbitrary precision (no overflow).
- SMT queries made synchronously; timeouts reported as "cannot verify."

The proof interpreter is a separate process from the compilation pipeline. It communicates via a well-defined IPC protocol. Compiler implementers may replace the default Z3 backend with any SMT-LIB 2-compatible solver.

---

## 2. Grammar Summary

The following is a condensed formal grammar. Terminal symbols are in `"quotes"`. Alternatives separated by `|`. Optional elements in `[square brackets]`. Repetition as `item*` (zero or more) or `item+` (one or more).

```
program         ::= item*

item            ::= fn_decl
                  | type_decl
                  | enum_decl
                  | trait_decl
                  | impl_block
                  | layout_decl
                  | import_decl
                  | const_decl
                  | static_decl
                  | proof_decl

fn_decl         ::= safety_kw* ["pub"] "fn" ident type_params? "(" params ")"
                    [":" type] [effects] [given_clause]
                    [contract_clause]*
                    block

safety_kw       ::= "unchecked" | "trusted" "(" string_lit ")" | "extern" | "unsafe"

type_params     ::= "[" type_param ("," type_param)* "]"
type_param      ::= ident [":" bounds] | "'" ident
bounds          ::= bound ("+" bound)*
bound           ::= type_path | lifetime

effects         ::= "!" effect ("+" effect)*
effect          ::= "IO" | "Net" | "Sync" | "Unsafe" | "Panic" | "Alloc" "[" type "]"
                  | "[" ident "]"

given_clause    ::= "given" "[" capability_param ("," capability_param)* "]"
capability_param ::= ident ":" bounds ["where" constraint]

contract_clause ::= "requires" expr
                  | "ensures"  expr
                  | "invariant" expr

type_decl       ::= ["pub"] ["pinned"] "type" ident type_params? "{" fields [invariant_decl*] "}"
                  | ["pub"] "type" ident type_params? "(" types ")"
                  | ["pub"] "type" ident type_params? "=" type

enum_decl       ::= ["pub"] "enum" ident type_params? "{" variant ("," variant)* [","] "}"
variant         ::= ident ["(" types ")"] [":" type] ["," ident "{" fields "}"]

trait_decl      ::= ["pub"] "trait" ident type_params? [":" bounds] "{" trait_item* "}"
trait_item      ::= fn_decl | "type" ident [":" bounds] ["=" type]

impl_block      ::= "impl" type_params? type ["for" type] [where_clause] "{" impl_item* "}"
impl_item       ::= fn_decl | "type" ident "=" type

layout_decl     ::= "layout" type_path ["as" ident] "{" layout_prop* layout_field* "}"
layout_prop     ::= "byte_order" ":" byte_order_val
                  | "bit_order"  ":" bit_order_val
                  | "total_size" ":" int_lit ("bits" | "bytes")
layout_field    ::= ident "at" placement [per_field_props]
                  | "_" ident "at" placement
placement       ::= "byte" int_lit ["," "bits" int_lit ".." int_lit]
                  | "bytes" int_lit ".." int_lit ["," "bits" int_lit ".." int_lit]

import_decl     ::= "import" path ["as" ident]
                  | "import" path "." "{" ident ("," ident)* "}"
                  | "import" path "." "*"

proof_decl      ::= "proof" "fn" ident type_params? "(" params ")" [":" type]
                    contract_clause*
                    block

block           ::= "{" stmt* [expr] "}"
stmt            ::= let_stmt | expr_stmt | item
let_stmt        ::= "let" ["mut"] pattern [":" type] "=" expr
expr_stmt       ::= expr

expr            ::= lit
                  | path
                  | block
                  | "if" expr block ["else" (block | if_expr)]
                  | "loop" block
                  | "while" expr block
                  | "for" pattern "in" expr block
                  | "match" expr "{" arm+ "}"
                  | "with" expr "as" ident "{" block "}"
                  | "select" "{" select_arm+ "}"   // intrinsic
                  | "return" [expr]
                  | "break" [lifetime] [expr]
                  | "continue" [lifetime]
                  | expr "?"
                  | expr bin_op expr
                  | unary_op expr
                  | expr "(" args ")"          // function call
                  | expr "." ident ["(" args ")"] // field/method
                  | expr "[" expr "]"           // index
                  | ident "{" field_inits "}"   // struct construction
                  | "|" params "|" expr         // closure
                  | "move" "|" params "|" expr  // move closure
                  | "unsafe" "{" block "}"
                  | "unchecked" "{" block "}"
                  | "old" "(" expr ")"          // in ensures only

arm             ::= pattern ["if" expr] "=>" expr [","]

select_arm      ::= select_pattern "=>" expr [","]
select_pattern  ::= ident "->" ident              // recv(channel) -> msg
                  | ident "<-" expr               // send(channel, value)
                  | "timeout" "(" expr ")"        // timeout(duration)
                  | "default"                     // non-blocking fallback

pattern         ::= "_"
                  | lit
                  | ident
                  | path "(" patterns ")"
                  | path "{" field_patterns "}"
                  | "(" patterns ")"
                  | "[" slice_patterns "]"
                  | pattern "|" pattern
                  | pattern "@" pattern
                  | pattern "if" expr           // guard (in match only)
                  | ".." pattern                // rest pattern
                  | pattern ".."

type            ::= path
                  | "&" ["mut"] type
                  | "*" ("const" | "mut") type
                  | "[" type ";" expr "]"       // array
                  | "[" type "]"                // slice
                  | "(" types ")"               // tuple
                  | "fn" "(" types ")" [":" type] [effects]
                  | "impl" bounds
                  | "dyn" bounds
                  | "never"
```
