# Ferrum Language Reference — Ownership and Effects

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Ownership and Regions

### 1.1 Ownership Rules

Every value has exactly one owner at any point in time. Ownership is transferred (moved) or borrowed. When the owner goes out of scope the value is dropped (destructor called, memory freed via the ambient allocator).

**Move semantics are the default.** A type is moved unless it implements the `Copy` trait.

```ferrum
let a = String.from("hello")
let b = a           // a is moved into b; a is no longer valid
// print(a)         // compile error: use of moved value
print(b)            // fine
```

**Borrow rules:**
- Any number of shared (`&T`) borrows may coexist.
- Exactly one exclusive (`&mut T`) borrow may exist.
- Shared and exclusive borrows cannot coexist.
- A borrow cannot outlive the owner.

These rules are enforced by the compiler's borrow checker (see §19).

### 1.2 Region Inference

Ferrum uses region inference to eliminate explicit lifetime annotations in the common case. A *region* is a compile-time abstraction representing the scope during which a reference is valid.

> **Design Note:** Region inference eliminates annotations in the common case.
> The fallback — requiring annotations when inference fails — is not failure.
> It is honest scope management. The annotations that remain carry genuine
> semantic content that a reader needs.

**The inference algorithm:**
1. Assign a fresh region variable to every reference in a function signature.
2. Generate constraints from the function body (assignments, returns, field accesses).
3. Solve constraints to the smallest valid region for each variable.
4. Report an error only if no solution exists.

**When inference succeeds (no annotations needed):**
```ferrum
fn longest(a: &str, b: &str): &str {
    if a.len() > b.len() { a } else { b }
}
// Inferred: return region = min(region(a), region(b))
```

**When inference fails (annotation required):**
Annotations are required when the function signature is genuinely ambiguous — i.e., when a human reader also cannot determine the relationship between input and output lifetimes without additional information.

```ferrum
// Ambiguous: does the return borrow from x, y, or either?
// Must annotate
fn choose[T]<'a, 'b>(x: &'a T, y: &'b T, use_x: bool): &'a T
    where 'b: 'a    // 'b outlives 'a
{
    if use_x { x } else { y as &'a T }
}
```

Region annotations use the `'name` syntax. The `where 'b: 'a` clause asserts that region `'b` outlives region `'a`.

**The 90% rule:** Ferrum's region inference is designed to eliminate annotations in at least 90% of practical code. The remaining 10% — complex borrowing patterns at API boundaries — requires annotations, and these annotations carry genuine semantic content that a reader needs.

### The Annotation Layer as Audit Trail

The annotations in a Ferrum codebase are not noise — they are signal. Every annotation that remains is load-bearing knowledge that a human needs to own.

When the compiler can verify something automatically, it does. When it cannot, the human judgment that fills the gap is explicit, named, and in the audit trail.

```
warning: redundant annotation
  --> src/parser.rs:47:5
   |
47 |     trusted("ptr valid for len bytes, aligned")
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |     region inference verifies this automatically
   |
   = note: `ferrum fix --remove-redundant` to clean up automatically
```

The warning says why the annotation is redundant and how to remove it automatically.

The goal is not 100% verified. The goal is: every annotation that remains is load-bearing knowledge that a human needs to own.

### 1.3 Explicit Region Declarations

For performance-critical or embedded code, regions can be declared explicitly:

```ferrum
fn process(data: &[u8]): Result[Output] {
    region arena {
        let buf: Vec[u8] | arena = Vec.new()
        // buf is allocated in the arena, freed when region exits
        let parsed = parse(buf)?
        Ok(parsed.into_owned())  // must move out before region exits
    }
    // arena freed here, regardless of panic
}
```

The `| region_name` syntax annotates that a value is allocated in a specific region. This is only required when multiple regions are in scope simultaneously or when overriding the ambient allocator capability.

### 1.4 The `Copy` Trait

Types that implement `Copy` are duplicated on assignment rather than moved.

```ferrum
// These types are Copy by default:
// all scalar types, references (&T but not &mut T), raw pointers,
// arrays of Copy types, tuples of Copy types

// Derive Copy explicitly:
@derive(Copy, Clone)
type Point { x: f32, y: f32 }
```

A type may only implement `Copy` if all its fields are `Copy` and it has no destructor.

### 1.5 Pinned Types

Self-referential types use `pinned` to guarantee they are never moved after construction:

```ferrum
pinned type RingBuffer[T] {
    data: [T; 256],
    head: *const T,   // interior pointer
    tail: *const T,
}
```

`pinned` types:
- Cannot be moved after construction completes.
- Can be borrowed (`&T`, `&mut T`).
- Cannot be passed by value to functions (only by reference).
- Their destructor is guaranteed to run before their memory is freed.

The compiler inserts a compile-time check at every site where a `pinned` type would be moved. This is a first-class language concept, not a library workaround.

---

## 2. Effect System

### 2.1 Overview

Effects describe what a function does beyond computing a return value. Effects are part of the function's type, visible to callers, and inferred in function bodies.

**Core effects:**

| Effect | Meaning |
|---|---|
| `IO` | Reads or writes to I/O (files, stdin/stdout, environment) |
| `Net` | Network operations |
| `Sync` | Acquires or releases synchronization primitives |
| `Unsafe` | Contains unsafe operations |
| `Panic` | May call `panic` |
| `Alloc[A]` | Allocates using allocator `A` (see §7) |

### 2.2 Effect Syntax

Effects appear after the return type, separated by `!`:

```ferrum
pub fn read_file(path: &str): Result[String] ! IO
pub fn connect(addr: Addr): Result[Socket]  ! Net
pub fn spawn_task(f: fn()): Task            ! Sync + Alloc[Heap]
pub fn pure_fn(x: i32): i32               // no ! means pure — compiler enforced
```

Multiple effects use `+`:
```ferrum
pub fn fetch_and_log(url: &str): Result[String] ! IO + Net
```

### 2.3 Effect Inference

Effects are **inferred within modules.** You write effect annotations only at `pub` boundaries — the public API of your module. Private functions don't need annotations; the compiler infers their effects from what they call.

```ferrum
// Private helper — no annotation needed, effects inferred
fn load_and_parse(path: &str): Result[Data] {
    let raw = read_file(path)?      // read_file is ! IO
    parse(raw)                      // parse is pure
}
// Compiler infers: load_and_parse is ! IO

// Public API — annotation required
pub fn process(path: &str): Result[Output] ! IO {
    let data = load_and_parse(path)?
    transform(data)
}
```

If you omit the annotation on a `pub` function, the compiler emits an error telling you which effects were inferred. This keeps internal code light while ensuring public APIs are explicit about their effects.

### 2.4 Pure Functions

A function with no `!` annotation is **pure** — the compiler statically verifies it produces no effects:

```ferrum
fn checksum(data: &[u8]): u32 {
    // Any call to an impure function here is a compile error
    data.iter().fold(0u32, |acc, &b| acc.wrapping_add(b as u32))
}
```

Pure functions are referentially transparent: calling them twice with the same arguments produces the same result. This property is exploited by the proof system (§14).

### 2.5 Effect Polymorphism

Functions can be polymorphic over effects:

```ferrum
fn map_result[T, U, E][eff](f: fn(T): U ! eff, r: Result[T, E]): Result[U, E] ! eff {
    match r {
        Ok(v)  => Ok(f(v)),
        Err(e) => Err(e),
    }
}
```

The `[eff]` parameter ranges over sets of effects. This is primarily used in the standard library for higher-order functions.

### 2.6 Effect Boundaries

Crossing certain architectural boundaries requires explicit effect declaration:

**Task spawn** — the spawned task has its own effect context:
```ferrum
scope s {
    s.spawn(worker())   // worker's effects must be compatible with Sync + Send
}
```

**FFI** — foreign functions are always `! Unsafe + IO` unless declared otherwise:
```ferrum
extern "C" fn malloc(size: usize): *mut u8  ! Unsafe + Alloc[Heap]
```

**Proof mode** — proof functions may have no runtime effects (see §14):
```ferrum
proof fn sorted(xs: &[i32]): bool  // no effects — pure and total
```

---

## 3. Capabilities and Implicit Parameters

### 3.1 Overview

Capabilities are ambient values that are threaded through the program automatically by the compiler. They are part of the type system but invisible at most call sites.

Capabilities solve generic explosion: parameters that always travel together and are never independently varied by the caller should not appear in every function signature.

### 3.2 The `given` Keyword

```ferrum
// Declare that this function requires a capability
fn parse(src: &str): Json  given [A: Allocator]

// Declare a capability scope
with Arena.new() as alloc {
    let a = parse(src)      // A = Arena, resolved from scope
    let b = process(a)      // A = Arena, same resolution
}  // arena freed
```

Capabilities are resolved **lexically from the innermost enclosing `with` block** that provides a matching capability. If no `with` block is in scope, the compiler uses the default for that capability type (for `Allocator`, the default is `Heap`).

### 3.3 Capability Declaration

```ferrum
// Single capability
fn parse(src: &str): Json  given [A: Allocator]

// Multiple capabilities
fn serve(port: Port): never  given [A: Allocator]  ! Net + IO

// Constrained capability
fn realtime_path(data: &[u8]): u32  given [A: Allocator where A: NoAlloc]
// NoAlloc is a marker trait — the allocator promises zero allocation

// Named capability (when you need two of the same type simultaneously)
fn copy_between(src: &Buf, dst: &mut Buf)
    given [SrcA: Allocator, DstA: Allocator]
```

### 3.4 Overriding Capabilities

```ferrum
with Heap as alloc {
    // Default path: heap allocated
    let config = load_config(path)    // uses Heap

    with Arena.new() as alloc {
        // Override for this inner scope
        let tmp = parse_request(data)   // uses Arena
    }   // arena freed

    let result = process(config)    // back to Heap
}
```

### 3.5 Capabilities at Concurrency Boundaries

Spawning a task is a capability boundary. The compiler verifies that capabilities crossing the boundary are valid for the task's lifetime:

```ferrum
scope s {
    // Arena is scoped to the current function — cannot cross to a task
    // that might outlive it
    let arena = Arena.new()

    // ERROR: arena capability cannot cross task boundary
    // s.spawn(with arena as alloc { worker(data) })

    // OK: heap capability has static lifetime
    s.spawn(with Heap as alloc { worker(data.clone()) })
}
```

This is enforced at compile time. It is the primary mechanism preventing use-after-free through task spawning.

### 3.6 Implicit vs Explicit: The Decision Rule

Use explicit `[T]` type parameters when the caller must think about `T` — when different callers will pass different concrete types and the choice matters for correctness or performance from the caller's perspective.

Use `given` capabilities when:
- The parameter is an ambient context (allocator, locale, clock, random source)
- It is the same throughout a scope and varies only at scope boundaries
- The caller should not need to think about it

Use associated types (§8.4) when the parameter is always determined by another type — it is part of a family.

---
