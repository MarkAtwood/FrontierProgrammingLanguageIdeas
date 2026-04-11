# Ferrum Language Reference — Core

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Overview

Ferrum is a systems programming language with the following properties, listed in priority order:

1. **Correct by construction.** Illegal states are unrepresentable at the type level wherever possible. Contracts enforce the rest.
2. **No hidden costs.** Every allocation, copy, synchronization, and effect is visible in the code or its type signature.
3. **Minimal annotation burden.** The compiler infers what it can. Annotations appear only when they carry information a human reader needs. Effects are inferred within modules, required only at `pub` boundaries. Allocators default to `Heap`, explicit only for custom allocation.
4. **Graduated trust.** Safety is a spectrum, not a binary. The compiler handles the common cases automatically; explicit annotations mark where human judgment fills the gap. The boundary between compiler knowledge and human knowledge is explicit and auditable.
5. **Proofs are programs.** The same language that compiles to machine code can express and verify formal properties.

### Design Influences

| Feature | Inspiration | Ferrum's Approach |
|---|---|---|
| Ownership and borrowing | Rust | Region inference eliminates most lifetime annotations |
| Explicit allocators | Zig | Allocator is a trait in the type system, not a convention |
| Channel-based concurrency | Go | Channels are typed and directional; tasks are scope-structured |
| Typeclasses, sum types, GADTs | Haskell | No HKTs in user-facing code; effects tracked but inferred |
| Dataflow analysis, constrained integers | Ada | Integer constraints are type-level; layout is a first-class declaration |
| Design by contract | Eiffel | Contracts are runtime checks in debug, optimizer assumptions in release |
| C-family syntax | C, Python | No semicolons, no headers, `[T]` generics, modules as files |

### Non-goals

- Garbage collection. The language has no GC. If you want GC semantics, implement a GC allocator.
- Runtime reflection. Compile-time introspection only.
- Implicit coercions of any kind.
- Exceptions. `Result[T]` and `?` propagation only. `panic` is unrecoverable abort.
- Inheritance. Traits only.
- Higher-kinded types in user-facing code. The stdlib uses them internally.

---

## 2. Lexical Structure

### 2.1 Source Files

Source files are UTF-8 encoded. File extension is `.fe`. One module per file. No header files.

### 2.2 Whitespace and Newlines

Newlines are significant as statement terminators in limited contexts (see §9). Elsewhere whitespace is insignificant. There are no semicolons.

### 2.3 Comments

```ferrum
// Line comment

/* Block comment
   may span lines */

/// Doc comment — attached to the next item
/// Supports Markdown.

//! Module-level doc comment — at top of file
```

### 2.4 Identifiers

```
identifier := [a-zA-Z_][a-zA-Z0-9_]*
```

Identifiers are case-sensitive. The following naming conventions are enforced as warnings, not errors:

- Types, traits, enum variants: `PascalCase`
- Functions, variables, fields: `snake_case`
- Constants, statics: `SCREAMING_SNAKE_CASE`
- Type parameters: single uppercase letter or `PascalCase`
- Lifetime/region parameters: lowercase with leading tick: `'a`

### 2.5 Keywords

```
as          async       await       break       const
continue    else        ensures     enum        extern
false       fn          for         given       if
impl        import      in          invariant   isize
layout      let         loop        match       mod
mut         pinned      proof       pub         region
requires    return      self        Self        static
struct      trait       true        trusted     type
unchecked   unsafe      usize       where       with
```

**45 keywords total.** Ferrum minimizes keywords by using:

- **Symbols for logic:** `&&`, `||`, `!`, `^^` instead of `and`, `or`, `not`, `xor`
- **Builtin types:** Numeric types (`i32`, `f64`, etc.) are builtin type identifiers, not keywords
- **Macros:** `select!`, `timeout!` instead of keywords
- **Attributes:** `@borrow_safe` instead of `assert_borrow_safe`

### 2.5.1 Builtin Type Identifiers

These are predefined type names, not keywords. They can be shadowed (though
doing so is a warning), and literal suffixes like `42i32` work via special
parsing, not keyword status.

```
// Integers
i8    i16   i24   i32   i48   i64   i96   i128  i256  i512  i1024
u8    u16   u24   u32   u48   u64   u96   u128  u256  u512  u1024

// Floats
f16   f32   f64   f80   f128  f256  bf16

// Decimal floats
d32   d64   d128

// Other
bool  byte  char  never
comptime_int  comptime_float
```

### 2.5.2 Operators

**Logical operators (short-circuit):**
```ferrum
a && b       // logical and (short-circuit)
a || b       // logical or (short-circuit)
!a           // logical not
a ^^ b       // logical xor (evaluates both sides)
```

**Bitwise operators:**
```ferrum
a & b        // bitwise and
a | b        // bitwise or
~a           // bitwise not (complement)
a ^ b        // bitwise xor
a << n       // left shift
a >> n       // right shift
```

Note: `!` is logical not (returns `bool`), `~` is bitwise not (returns same integer type).

### 2.6 Literals

**Integer literals:**
```ferrum
42           // untyped integer literal — type inferred from context
42i32        // type suffix forces type
0xFF         // hex
0o77         // octal
0b1010_1010  // binary — underscores anywhere
1_000_000    // decimal with separators

// All integer suffixes:
42u8    42u16   42u24   42u32   42u48
42u64   42u96   42u128  42u256  42u512  42u1024
42i8    42i16   42i24   42i32   42i48
42i64   42i96   42i128  42i256  42i512  42i1024
42usize 42isize
```

Untyped integer literals default to `i32` when no context constrains the type.
An untyped literal that doesn't fit in `i32` defaults to the smallest signed
type that holds it (then `i64`, `i128`, `i256`, etc.).

Sub-byte integer literals use the smallest containing type by default
and are range-checked at compile time:

```ferrum
// Sub-byte via constrained types (see §3.3)
let nibble: u4  = 15       // stored as u8, value must be <= 15
let three:  u3  = 7        // stored as u8, value must be <= 7
let bit:    u1  = 1        // stored as u8, value must be 0 or 1
// In layout declarations these pack to their exact bit width (see §15)
```

**Float literals:**
```ferrum
3.14           // untyped float — defaults to f64
3.14f16        // half precision
3.14bf16       // brain float 16
3.14f32        // single precision
3.14f64        // double precision
3.14f80        // x87 extended (x86/x86-64 only; soft-float elsewhere)
3.14f128       // quad precision (soft-float on most targets)
3.14f256       // octuple precision (always soft-float)

2.0e10         // scientific notation (any float suffix)
0x1.8p+1       // hex float (f32/f64 only — C99 compatible)
```

**Decimal float literals:**
```ferrum
3.14d32        // decimal32 — 7 significant digits
3.14d64        // decimal64 — 16 significant digits
3.14d128       // decimal128 — 34 significant digits

// These are exact:
0.1d64 + 0.2d64 == 0.3d64  // true — no binary rounding error
```

**Fixed-point literals:**
```ferrum
// Fixed-point via the type annotation — the literal is an integer or float
// that gets converted at compile time
let x: Q16_16 = 3.75      // exact: 3 + 3/4
let y: Q1_15  = 0.5       // exact: 1/2
let z: Fixed[8,8] = -1.5  // exact: -1 - 1/2

// No runtime conversion cost — the conversion is compile-time
```

**String literals:**
```ferrum
"hello"                  // UTF-8 string, escape sequences
"line1\nline2"
r"raw \n no escapes"
r#"raw with "quotes" inside"#
b"byte string"           // &[u8], not &str
```

**Byte literals:**
```ferrum
b'A'    // u8 value 65
```

**Character literals:**
```ferrum
'A'     // Unicode scalar value, type char (u32)
'\n'
'\u{1F600}'
```

**Boolean literals:** `true`, `false`

**Escape sequences in strings:**
```
\n  \r  \t  \\  \"  \'  \0
\xNN        (byte value, string only)
\u{NNNNNN}  (Unicode scalar)
```

**Underscore separators:**

Underscores may appear anywhere in a numeric literal except the start
and immediately after a `0x`/`0o`/`0b` prefix:

```ferrum
1_000_000_000u64
0xDEAD_BEEF
0b1111_0000_1111_0000
3.141_592_653_589_793f64
```

---
