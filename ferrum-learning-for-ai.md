# Ferrum: Fast Learning Guide for AI Agents

This document is a dense reference for AI assistants working with Ferrum source code or documentation. It assumes familiarity with Rust, Python, or C and focuses on what's surprising, what's different, and the mental models that make the design click.

---

## What Ferrum Is

A systems programming language with:
- **Rust's ownership model** but ~90% fewer lifetime annotations (region inference)
- **An effect system** that tracks IO, Net, Sync, etc. at function boundaries
- **An allocator capability system** that eliminates allocator-parameter explosion
- **Formal contracts** (`requires`/`ensures`) and optional machine-checked proofs
- **Constrained numeric types**, non-power-of-two integers, decimal floats
- **No user-defined macros** (security and simplicity)

Design philosophy: make the compiler do the work, make the programmer's judgments visible, reveal features as they're needed.

---

## Syntax At A Glance

```ferrum
// Generics: [] not <>
fn max[T: Ord](a: T, b: T): T { if a > b { a } else { b } }

// Return type: : not ->
fn parse(src: &str): Result[Data] ! IO { ... }

// Paths: . not ::
let m = std.collections.HashMap.new()

// Attributes: @ not #[]
@derive(Clone, Debug)
type Point { x: f64, y: f64 }

// Effects appended with !
pub fn read_file(path: &str): Result[String] ! IO

// Multiple effects
pub fn fetch_and_log(url: &str): Result[String] ! IO + Net

// No semicolons required; semicolons are statement separators, not terminators
```

### Key Syntax Differences from Rust

| Concept | Rust | Ferrum |
|---------|------|--------|
| Generic params | `fn f<T>()` | `fn f[T]()` |
| Return type | `fn f() -> T` | `fn f(): T` |
| Path separator | `std::io` | `std.io` |
| Attributes | `#[derive(...)]` | `@derive(...)` |
| Struct syntax | `struct Foo { ... }` | `type Foo { ... }` |
| Lifetime annotation | `'a` everywhere | rare; region inference handles most |
| Macros | `println!("...")` | `println("...")` (compiler intrinsic) |
| Effect tracking | none | `! IO`, `! Net`, `! Sync` |
| Allocator param | manual or ZST hack | `given [A: Allocator]` |

---

## Type System

### Four Orthogonal Layers

```
┌─────────────────────────────────┐
│  Effect layer     IO · Net · ... │  ← what the function does
├─────────────────────────────────┤
│  Ownership layer  Owned · &T    │  ← who owns the value
├─────────────────────────────────┤
│  Type layer       Scalar · Sum  │  ← what the value is
├─────────────────────────────────┤
│  Allocator layer  Heap · Arena  │  ← where memory comes from
└─────────────────────────────────┘
```

These layers are independent. A type like `Vec[u8] | Arena` says: the type is a vector of bytes (type layer), allocated from an arena (allocator layer).

### Numeric Types

Standard: `i8/u8` through `i128/u128`, `isize/usize`

Non-power-of-two (exact widths, no padding):
```
i24, i48, i96, u24, u48, u96
u256, u512, u1024, i256, i512, i1024  (crypto sizes)
```

Sub-byte (one byte standalone, pack in `layout` declarations):
```
u1..u7, i1..i7
```

Arbitrary width:
```
@Int(signed, 37)   // 37-bit signed integer
```

Floats:
```
f16, bf16, f32, f64, f80, f128, f256   (binary IEEE)
d32, d64, d128                          (decimal IEEE — 0.1d64 + 0.2d64 == 0.3d64)
Fixed[I, F], UFixed[I, F]              (fixed-point, also Q8_8, Q16_16 notation)
```

### Constrained Types

```ferrum
type Percent = u8  where value <= 100
type Port    = u16 where value >= 1024
type NonZero = u32 where value != 0

// Enforcement:
// - Debug: runtime assertion
// - Release: UB on violation (optimizer assumption)
// - --proof mode: statically verified at compile time
```

### Arithmetic Operators

No implicit overflow. Use explicit operators:

```ferrum
a + b          // panics on overflow in debug, UB in release
a +% b         // wrapping (modular)
a +| b         // saturating (clamps to min/max)

// Same pattern for -, *
// Shift amounts must be the right type (no shift by 64 on a u64)
```

### Pointer / Reference Types

```ferrum
&T           // shared reference — non-null, immutable, lifetime tracked
&mut T       // exclusive reference — non-null, mutable
*const T     // raw const pointer — nullable, unsafe to dereference
*mut T       // raw mutable pointer — nullable, unsafe to dereference
```

No difference from Rust here. The borrow rules are the same.

---

## Ownership and Regions

### Core Rules (Same as Rust)

- One owner; ownership moves on assignment for non-`Copy` types
- Any number of `&T` OR exactly one `&mut T`, never both
- A borrow cannot outlive its referent

### Region Inference (Major Difference from Rust)

Ferrum infers lifetimes in ~90% of cases. Explicit region annotations are only needed when the compiler cannot resolve ambiguity.

```ferrum
// No annotation needed — compiler infers
fn longest(a: &str, b: &str): &str {
    if a.len() > b.len() { a } else { b }
}

// Annotation required — two inputs, one output, genuinely ambiguous
fn choose[T]<'a, 'b>(x: &'a T, y: &'b T, use_x: bool): &'a T
    where 'b: 'a
{
    if use_x { x } else { y as &'a T }
}
```

**Mental model**: when you see lifetime annotations in Ferrum code, they carry real semantic information (as opposed to Rust where many are noise).

### Pinned Types

For self-referential types:

```ferrum
pinned type RingBuffer[T] {
    data: [T; 256],
    head: *const T,
    tail: *const T,
}
// Cannot move after construction; destructor guaranteed to run
// First-class language concept, not a library wrapper
```

### Arena Regions

```ferrum
fn process(data: &[u8]): Result[Output] {
    region arena {
        let buf: Vec[u8] | arena = Vec.new()
        let parsed = parse(buf)?
        Ok(parsed.into_owned())  // must move out before region exits
    }
    // arena freed here, even on panic
}
```

---

## Effect System

Effects track what a function does beyond computing. They're part of the function's type.

### Built-in Effects

| Effect | Meaning |
|--------|---------|
| `IO` | stdin, stdout, files, environment |
| `Net` | network operations |
| `Sync` | acquires/releases locks or synchronization primitives |
| `Unsafe` | contains unsafe operations |
| `Panic` | may call `panic` |
| `Alloc[A]` | allocates using allocator `A` |

### Effect Syntax

```ferrum
pub fn read_file(path: &str): Result[String] ! IO
pub fn fetch(url: &str): Result[Bytes] ! Net
pub fn spawn(f: fn()): Task ! Sync + Alloc[Heap]

// No ! = pure; compiler statically enforces no effects
fn add(x: i32, y: i32): i32 { x + y }
```

### Effect Inference Rule

**Private functions**: effects inferred from body — no annotation needed.
**Public functions**: annotation required — serves as documentation and API contract.

```ferrum
// Private — inferred
fn load_and_parse(path: &str): Result[Data] {
    let raw = read_file(path)?   // read_file is ! IO
    parse(raw)                    // parse is pure
}
// compiler infers: ! IO

// Public — annotation required
pub fn process(path: &str): Result[Output] ! IO {
    let data = load_and_parse(path)?
    transform(data)
}
```

### Effect Polymorphism

```ferrum
fn map_result[T, U, E][eff](f: fn(T): U ! eff, r: Result[T, E]): Result[U, E] ! eff {
    match r {
        Ok(v)  => Ok(f(v)),
        Err(e) => Err(e),
    }
}
```

---

## Capabilities (Ambient Implicit Parameters)

Capabilities solve "allocator parameter explosion" — context values that always travel together shouldn't appear in every signature.

### The `given` Keyword

```ferrum
// Function declares it needs an allocator capability
fn parse(src: &str): Json  given [A: Allocator]

// Caller provides it via a `with` block
with Arena.new() as alloc {
    let doc = parse(src)      // A = Arena, resolved from scope
}
// arena freed here
```

### Default Capability: Heap

Most code never mentions allocators. `Heap` is implicitly the default.

```ferrum
// No allocator annotation — uses Heap
fn build_index(docs: &[Doc]): Index {
    let mut idx = HashMap.new()
    for doc in docs {
        idx.insert(doc.id, doc.tokens.clone())
    }
    idx
}
```

### Overriding Capabilities (Scoped)

```ferrum
with Heap as alloc {
    let config = load_config(path)    // Heap

    with Arena.new() as alloc {
        let tmp = parse_request(data)   // Arena
    }   // arena freed

    let result = process(config)      // back to Heap
}
```

### Multiple Capabilities of the Same Type

```ferrum
fn copy_between(src: &Buf, dst: &mut Buf)
    given [SrcA: Allocator, DstA: Allocator]
```

### Capabilities at Task Boundaries

```ferrum
scope s {
    let arena = Arena.new()

    // ERROR: arena scoped to current function, cannot cross task boundary
    // s.spawn(with arena as alloc { worker(data) })

    // OK: Heap has static lifetime
    s.spawn(with Heap as alloc { worker(data.clone()) })
}
```

---

## Module System

### One File = One Module

```
src/
    main.fe          → module main
    parser.fe        → module parser
    net/
        http.fe      → module net.http
        tcp.fe       → module net.tcp
```

No header files. No forward declarations.

### Import Syntax

```ferrum
import std.collections.HashMap
import std.io.{read_line, write_line}
import std.net as network
import std.prelude.*          // glob import (lint warning outside preludes — see below)
```

### Glob Imports

`import module.*` imports all currently-exported names. The compiler emits a lint warning when used outside prelude contexts:

```
warning: glob import outside prelude
  --> src/lib.fe:3:1
   |
 3 | import std.collections.*
   | ^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: future versions of std.collections may add exports that silently shadow local names
   = help: use explicit imports: `import std.collections.{HashMap, BTreeMap}`
   = help: or suppress: `#[allow(glob_import, "reason")]`
```

`#[allow(glob_import, "reason")]` suppresses the warning. A reason string is required — the compiler emits a secondary warning if the string is empty. Legitimate uses:

- Prelude modules: `import std.prelude.*`
- Intrinsic namespaces: `import std.sys.arch.x86_64.*` (SIMD intrinsics — explicit listing is impractical)
- Test helpers: `import testing.*` inside `#[test]` modules

Libraries must not re-export a glob in their public API. A glob is resolved at import time in the importing module; it never propagates to downstream users.

### Visibility

```ferrum
pub fn public_function() { ... }       // exported
fn private_function() { ... }          // module-private (default)
pub(crate) fn crate_visible() { ... }  // package-internal
pub(super) fn parent_visible() { ... } // parent module only
```

### Package Manifest (`Ferrum.toml`)

```toml
[package]
name    = "mypackage"
version = "0.1.0"
edition = "2025"

[dependencies]
http    = "^2.1"
json    = { version = "1.0", features = ["streaming"] }
pinned  = { sha256 = "a1b2c3...", url = "https://..." }
local   = { path = "../my-local-lib" }
```

Lockfile (`Ferrum.lock`) pins exact content hashes. Source tells where; hash tells what.

---

## Traits

Same concept as Rust traits (interfaces + associated functions).

```ferrum
trait Display {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] ! IO
}

trait From[T] {
    fn from(value: T): Self
}

trait Iterator {
    type Item
    fn next(self: &mut Self): Option[Self.Item]
    // provided: map, filter, fold, collect, count, ...
}

impl Display for Point {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] ! IO {
        write(f, "({}, {})", self.x, self.y)
    }
}
```

### Associated Types

Bind related types together:

```ferrum
trait Collection {
    type Item
    type Iter: Iterator[Item = Self.Item]
    type Alloc: Allocator

    fn len(self: &Self): usize
    fn iter(self: &Self): Self.Iter
}
```

### GADTs

Variants can specialize type parameters:

```ferrum
enum Expr[T] {
    Lit(i32)                   : Expr[i32],
    Bool(bool)                 : Expr[bool],
    Add(Expr[i32], Expr[i32])  : Expr[i32],
    If(Expr[bool], Expr[T], Expr[T]) : Expr[T],
}

fn eval[T](expr: Expr[T]): T {
    match expr {
        Expr.Lit(n)      => n,
        Expr.Bool(b)     => b,
        Expr.Add(a, b)   => eval(a) + eval(b),
        Expr.If(c, t, f) => if eval(c) { eval(t) } else { eval(f) },
    }
}
// The type guarantees no ill-typed expressions exist.
```

---

## Contracts and Proofs

### Contract Syntax

```ferrum
fn pop(xs: &mut Vec[T]): T
    requires xs.len() > 0
    ensures  xs.len() == old(xs.len()) - 1
    ensures  result == old(xs.last().unwrap())
{
    xs.remove(xs.len() - 1)
}

type SortedVec[T: Ord] {
    data: Vec[T],
    invariant forall i, j where i < j => self.data[i] <= self.data[j]
}
```

### Contract Enforcement Modes

| Build mode | How contracts are enforced |
|-----------|---------------------------|
| Debug | Runtime assertions; violation aborts with diagnostic |
| Release | Optimizer assumptions; violation is UB |
| `--proof` | Static verification via SMT solver at compile time |

### Proof Functions

Pure, total, erased from runtime:

```ferrum
proof fn sorted_insert_preserves_order[T: Ord](
    xs: SortedVec[T], x: T,
): SortedVec[T]
    ensures result.len() == xs.len() + 1
    ensures forall i, j where i < j => result[i] <= result[j]
{
    ...
}
```

### `tested_by` and `proven_by` — Linking Fast Implementations to Specifications

Two annotations, two levels of guarantee:

- **`tested_by(spec_fn)`** — in debug builds, calls both implementations and asserts outputs match. Runtime check on tested inputs only. No formal guarantee.
- **`proven_by(proof_fn)`** — proof of equivalence verified at compile time. Covers all inputs. Requires writing a proof function.

```ferrum
proof fn sorted_insert_spec[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    ensures result.is_sorted()
{ ... }

// tested_by: debug builds compare outputs; release builds skip spec
fn sorted_insert[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    tested_by(sorted_insert_spec)
{ ... }

// proven_by: formal proof, compiler-verified, all inputs
proof fn sorted_insert_correct[T: Ord](xs: SortedVec[T], x: T):
    Prop[sorted_insert(xs, x) == sorted_insert_spec(xs, x)]
{ ... }

fn sorted_insert[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    proven_by(sorted_insert_correct)
{ ... }
```

---

## Safety Levels

```ferrum
// default: full borrow checking, all contracts enforced
fn normal_code() { ... }

// unchecked: borrow rules suspended; contracts still checked
fn unchecked_code() unchecked { ... }

// trusted: programmer asserts properties compiler cannot verify
fn trusted_code() trusted { ... }

// extern: foreign function interface
extern fn c_function(x: *mut u8, n: usize) ! Unsafe

// unsafe: raw hardware access, no guarantees
unsafe fn raw_code() { ... }
```

---

## Binary Layout Declarations

For protocol work requiring exact bit-level field placement:

```ferrum
type EthernetFrame {
    dst_mac:   [u8; 6],
    src_mac:   [u8; 6],
    ethertype: u16,
}

layout EthernetFrame {
    byte_order: big_endian,
    total_size: 112 bits,
    dst_mac   at bytes 0..5,
    src_mac   at bytes 6..11,
    ethertype at bytes 12..13,
}
```

Compiler verifies the logical type and physical layout are consistent.

---

## Standard Library Structure

Layered — each layer usable independently:

```
core     — no allocator, no OS (embedded, no_std)
alloc    — requires allocator, no OS (custom embedded allocators)
std      — requires OS or WASI (full programs)
platform — OS-specific shims (linux, darwin, windows, wasi, ...)
```

### Prelude (Auto-Imported)

```ferrum
// Types
Option[T]  Result[T, E]  String  Vec[T]  HashMap[K, V]
Box[T]     Arc[T]        Rc[T]

// Traits
Copy Clone Drop Display Debug
Eq PartialEq Ord PartialOrd
Iterator From Into Send Sync

// Functions
print  println  panic  unreachable  todo  unimplemented

// Macros (compiler intrinsics)
assert  assert_eq  assert_ne  dbg
```

---

## No User-Defined Macros

Ferrum has no `macro_rules!` and no procedural macros.

**Why**: proc macros execute arbitrary code at compile time — a supply chain attack vector. They also produce confusing error messages.

**Instead**:
- `println`, `format`, `vec`, `assert` etc. are compiler intrinsics
- `@derive(...)` supports a fixed set of derivable traits
- External code generation handles DSL use cases

**Implication for AI**: Don't suggest writing macros. If you see code that would be a macro in Rust, it's either a compiler intrinsic call or needs a different approach.

---

## Concurrency

Ferrum uses structured concurrency with scoped tasks:

```ferrum
scope s {
    s.spawn({
        let data = fetch(url).await?
        process(data)
    })
    s.spawn({
        let data = fetch(other_url).await?
        process(data)
    })
}
// both tasks complete (or one panics) before scope exits
```

Ferrum has no function coloring. All functions are `fn`; suspendable functions carry the `! Async` effect in their signature. There is no `async fn` keyword and no `async { }` block syntax.

Effect `Sync` is added to any function that acquires a synchronization primitive.

---

## Error Handling

All errors via `Result[T, E]`. No `errno`. No `null` returns. No exceptions.

```ferrum
fn read_and_parse(path: &str): Result[Data] ! IO {
    let raw = read_file(path)?    // ? propagates Err, adds context
    let parsed = parse(&raw)?
    Ok(transform(parsed))
}
```

`panic` exists but is for unrecoverable programmer errors, not expected conditions.

---

## Common Patterns

### Builder pattern via method chaining

```ferrum
let server = Server.new()
    .bind("0.0.0.0:8080")
    .timeout(Duration.from_secs(30))
    .build()?
```

### Iterator chaining

```ferrum
let sum: i64 = data
    .iter()
    .filter(|x| x.is_valid())
    .map(|x| x.score as i64)
    .sum()
```

### Pattern matching with guards

```ferrum
fn classify(x: Option[i32]): &str {
    match x {
        Some(n) if n < 0 => "negative",
        Some(n) if n > 0 => "positive",
        Some(_)          => "zero",
        None             => "none",
    }
}
```

### Struct update syntax

```ferrum
let p2 = Point { x: 1.0, ..p1 }
```

---

## Mental Models for AI Assistants

### "Effects are function signatures, not comments"

When you see `! IO`, that's not documentation — it's part of the type. A caller who is pure cannot call a function that is `! IO`. This cascades up the call stack. You cannot accidentally introduce IO into a pure computation.

### "Capabilities are dependency injection without the framework"

The `given [A: Allocator]` mechanism is compile-time dependency injection. `with Arena.new() as alloc` is the injection point. `Heap` is the default injection. Think of it as "this function works with any allocator; the caller decides which one."

### "Region inference is a constraint solver, not magic"

The compiler builds a constraint graph from borrow scopes, assigns minimal regions, and checks consistency. When it fails (ambiguous), it asks for an annotation. The annotation is always meaningful — it's resolving a genuine ambiguity, not noise.

### "Annotations are the compiler's TODO list"

Every explicit lifetime or effect annotation is a place where the compiler currently needs human help. As the compiler improves, *redundant* annotations (on private functions, or where inference now handles the relationship) generate warnings and can be auto-removed. Missing effect annotations on `pub` functions are always errors — the effect annotation is part of the public type signature and cannot be inferred away. The compiler tells you which effects to add.

### "The four layers are orthogonal"

You can change the allocator without changing the type. You can change the effect without changing the ownership. Think of each layer as an independent axis. Complex types like `&mut Vec[u8] | Arena ! IO` are just coordinates in a 4-dimensional space.

### "No macros means no magic"

If Rust code uses `println!`, Ferrum uses `println`. If Rust has a custom `#[derive(Serialize)]`, Ferrum either has a built-in derivation or uses external codegen. When reading or writing Ferrum, never reach for `macro_rules!` or proc-macros.

---

## What Ferrum Is NOT

- Not Rust with syntactic changes. The effect system, capability system, and region inference are first-class features that change how programs are written.
- Not a garbage-collected language. Ownership and regions handle memory. No GC pauses.
- Not a purely functional language. It has mutation, ownership, side effects — they're just tracked and controlled.
- Not a research language that sacrifices ergonomics for purity. Progressive disclosure is a design goal: inner layers are productive without knowing outer layers exist.

---

## Quick Reference Card

```
fn f[T: Bound](arg: T): RetType ! Effect { ... }
given [A: Allocator]              — allocator capability
with X.new() as alloc { ... }    — capability scope
requires <expr>                   — precondition
ensures <expr>                    — postcondition
invariant <expr>                  — type invariant
proof fn ...                      — compile-time proof, erased
tested_by(<spec_fn>)              — link fast impl to spec; debug builds compare outputs
proven_by(<proof_fn>)             — link fast impl to proof; compiler-verified for all inputs
region r { ... }                  — arena lifetime scope
unchecked / trusted / unsafe      — safety escape hatches
pinned type                       — self-referential, non-moveable
layout T { ... }                  — binary layout declaration
@derive(...)                      — fixed set of derivable traits
+% +|                             — wrapping / saturating arithmetic
import a.b.{c, d}                 — named imports
pub(crate) / pub(super)           — restricted visibility
```
