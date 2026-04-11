# Ferrum Language Reference

**Audience:** Compiler implementers, standard library authors, language users

---

## Document Structure

The language reference is split into focused documents:

| Document | Contents |
|----------|----------|
| [ferrum-lang-core.md](ferrum-lang-core.md) | Design philosophy, lexical structure, keywords, literals |
| [ferrum-lang-types.md](ferrum-lang-types.md) | Type system: scalars, numerics, floats, constrained types, pointers |
| [ferrum-lang-ownership.md](ferrum-lang-ownership.md) | Ownership, regions, effect system, capabilities |
| [ferrum-lang-generics.md](ferrum-lang-generics.md) | Allocators, generics, traits, GADTs |
| [ferrum-lang-syntax.md](ferrum-lang-syntax.md) | Expressions, pattern matching, functions, type declarations |
| [ferrum-lang-concurrency.md](ferrum-lang-concurrency.md) | Tasks, channels, synchronization, atomics |
| [ferrum-lang-verification.md](ferrum-lang-verification.md) | Contracts, proofs, binary layout, safety levels |
| [ferrum-lang-modules.md](ferrum-lang-modules.md) | Module system, packages, standard library overview |
| [ferrum-lang-internals.md](ferrum-lang-internals.md) | Compiler algorithms, grammar summary |

---

## Quick Reference

### Keywords (45)

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

### Builtin Types

```
i8  i16  i24  i32  i48  i64  i96  i128  i256  i512  i1024
u8  u16  u24  u32  u48  u64  u96  u128  u256  u512  u1024
f16  f32  f64  f80  f128  f256  bf16
d32  d64  d128
bool  byte  char  never  isize  usize
```

### Syntax Overview

```ferrum
// Function with return type (uses : not ->)
fn add(a: i32, b: i32): i32 {
    a + b
}

// Generics use [] not <>
fn max[T: Ord](a: T, b: T): T {
    if a > b { a } else { b }
}

// Paths use . not ::
let x = std.collections.HashMap.new()
let nan = f64.NAN

// Attributes use @ not #[]
@derive(Clone, Debug)
type Point { x: f64, y: f64 }

// Lifetimes use 'a syntax
fn longest<'a>(a: &'a str, b: &'a str): &'a str
```

### Operators

| Category | Operators |
|----------|-----------|
| Arithmetic | `+` `-` `*` `/` `%` |
| Wrapping | `+%` `-%` `*%` |
| Saturating | `+\|` `-\|` `*\|` |
| Comparison | `==` `!=` `<` `<=` `>` `>=` |
| Logical | `&&` `\|\|` `!` `^^` |
| Bitwise | `&` `\|` `^` `~` `<<` `>>` |
| Assignment | `=` `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` |
| Other | `?` `..` `..=` `as` |

### Effects

```
! IO        // File, network, environment I/O
! Net       // Network operations specifically
! Sync      // Synchronization primitives
! Unsafe    // Raw pointer operations
! Panic     // May panic
! Alloc[A]  // Allocates with allocator A
```

### Safety Levels

```
default     // Full safety guarantees
unchecked   // Borrow rules suspended, contracts still checked
trusted     // Asserts properties compiler cannot verify
extern      // Foreign function interface
unsafe      // Raw hardware access, no guarantees
```

---

## See Also

- [ferrum-stdlib.md](ferrum-stdlib.md) — Standard library design
- [ferrum-plan.md](ferrum-plan.md) — Implementation roadmap
- [ferrum-fears-and-hopes.md](ferrum-fears-and-hopes.md) — Design philosophy deep dive
