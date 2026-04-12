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

**What region inference covers:** Inference handles three broad categories of borrow patterns without annotations:

1. **Simple borrows** — a function takes a reference, uses it, and the result either doesn't alias the input or obviously does (single reference in, single reference out, same region).

2. **Linear flows** — references that flow in at call sites and out at returns without ambiguity about which input the output aliases.

3. **Local borrows** — references that don't escape the function or are consumed before the function returns.

Annotations are required when the constraint system is genuinely underdetermined:
- Multiple reference inputs where the output could alias any of them (the compiler cannot guess which)
- Split borrows that cross function-call boundaries in non-obvious ways
- Generic types that need to be region-polymorphic across call sites

The hope is this will cover 90% of common cases without herculean efforts. It should improve with time, as compiler design and CS theory advance.

> **Design decision:** The "90%" figure appears in lifetime-annotation literature but is not a measured fact for Ferrum — it is a design aspiration, not a guarantee. The real claim is qualitative: inference is designed to handle the common patterns so that the annotations that remain are load-bearing knowledge a reader needs to understand anyway.

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
    head: *const T,   // interior pointer into data
    tail: *const T,
}
```

`pinned` is a property of the **type**, declared at the type definition. Once constructed, the value's memory address is fixed for its lifetime.

#### What "move" means

A **move** is any operation that copies a value's bytes to a new memory address. The compiler rejects all of the following for `pinned` types:

| Operation | Example | Status |
|-----------|---------|--------|
| Assignment to a new binding | `let b = ring_buf` | Compile error |
| Pass by value to a function | `f(ring_buf)` | Compile error |
| Return by value | `return ring_buf` | Compile error |
| Placement into a container | `vec.push(ring_buf)` | Compile error |
| Destructuring by value | `let RingBuffer { data, .. } = ring_buf` | Compile error |

`pinned` types can be **borrowed** (`&T`, `&mut T`) freely — borrowing takes a reference to the existing address, not a copy.

#### Drop guarantee

Once a `pinned` value is constructed, its destructor **must** run before its memory is reused. Forgetting a pinned value (the equivalent of `mem::forget`) is safe-code UB: it violates the invariant that interior pointers remain valid for the value's lifetime. The compiler does not permit safe code to forget a `pinned` value.

#### Mutable references do not carry `noalias`

Normally, `&mut T` carries an aliasing guarantee: no other pointer to the same memory exists. For `pinned` types, this guarantee is automatically suppressed — `&mut PinnedType` does **not** carry `noalias`. The compiler knows the type is `pinned` and applies this suppression unconditionally.

This is why Ferrum does not need an equivalent of Rust's `UnsafePinned<T>` (RFC 3467): because `pinned` is a first-class type property rather than a library wrapper, the compiler can suppress `noalias` automatically wherever it matters.

#### Generics and pinned types

Generic code that moves `T` will fail to compile when instantiated with a `pinned T`:

```ferrum
fn swap[T](a: &mut T, b: &mut T) { ... }  // moves T internally

swap(&mut buf1, &mut buf2)  // ERROR: cannot move pinned type RingBuffer[u8]
```

There is no `Unpin`-equivalent trait. Non-`pinned` types are unaffected by this restriction — only types declared `pinned` constrain their generic contexts. Code that needs to accept `pinned` types must operate through references:

```ferrum
fn inspect[T](r: &T): usize { ... }  // borrows only — works with any T including pinned
```

Containers of `pinned` types must use indirection: `Vec<Box<RingBuffer<u8>>>` is valid; `Vec<RingBuffer<u8>>` is not (the vec would need to move elements on reallocation).

#### Unsafe code contract

Unsafe code that copies the bytes of a `pinned` value to a new memory address has **undefined behavior**. The interior pointer invariant is violated: the self-pointers now point at the old location, which may be freed or reused. This is the same category as violating aliasing rules: the compiler cannot detect it, and the programmer takes full responsibility.

Specifically, unsafe code must not:
- `ptr::copy` a `pinned` value to a different address
- Reinterpret a `&mut PinnedType` as a byte slice and copy it elsewhere
- Transmute or otherwise bypass the no-move restriction

What unsafe code **may** do: obtain a `&mut PinnedType` for in-place mutation (e.g., setting interior pointers after construction) — provided it does not move the value. This is the intended use case.

> **Design decision:** Rust chose a library approach (`Pin<P>` wrapper + `Unpin` auto-trait) to avoid "infecting" all generic code with `?Move` bounds. The cost was years of formal-UB async code: `Pin::get_unchecked_mut()` returns `&mut T` carrying `noalias`, while the self-referential struct contains aliasing raw pointers. This required `UnsafePinned<T>` (2026) to fix. Ferrum avoids this by making `pinned` a first-class type property: the compiler sees `pinned` in the type and suppresses `noalias` on mutable references automatically, with no library wrapper needed. The tradeoff — generic instantiation failure instead of `Unpin` bounds — is cleaner at definition sites and noisier at use sites, but the use sites are rare (containers of pinned types are inherently indirect) and the failure message is actionable.

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

> **Design decision:** this is an error, not a warning. A warning can be suppressed or ignored; a missing effect annotation on a public function leaves the type signature incomplete — callers cannot know what the function actually does. The effect annotation is part of the function's type. Missing it is a type error, the same as a missing return type.

### 2.4 Pure Functions

A function with no `!` annotation is **pure** — the compiler statically verifies it produces no effects:

```ferrum
fn checksum(data: &[u8]): u32 {
    // Any call to an impure function here is a compile error
    data.iter().fold(0u32, |acc, &b| acc.wrapping_add(b as u32))
}
```

Pure functions are referentially transparent: calling them twice with the same arguments produces the same result. This property is exploited by the proof system (§14).

#### Global and Thread-Local Mutable State

Any access to mutable state that exists outside the function's own stack frame and is not passed in as a parameter generates `! IO`. This includes:

- Process-wide global mutable state (`static mut`, atomics used as globals)
- Thread-local mutable state (`thread_local!` variables)

Thread-local state is still global state — just scoped to a thread. A function that reads a thread-local counter can return different values on successive calls with identical arguments, depending on what other code on the same thread has done. That violates referential transparency. Since `! IO` is not suppressible by `@pure`, no function accessing thread-local mutable state can be `@pure`.

```ferrum
thread_local! { static COUNTER: Cell<i32> = Cell::new(0); }

fn get_count(): i32 { COUNTER.get() }
// Compiler infers: ! IO — not pure, cannot be @pure
```

Immutable statics (constants, `static` without interior mutability) generate no effect — they are part of the program's read-only data segment and are referentially transparent.

#### `@pure` and No-Alloc Contexts

`@pure` suppresses `! Alloc` from the function's visible effect signature. This means the capability system cannot see the internal allocation:

```ferrum
@pure
fn cached_compute(x: i32): i32 {
    static CACHE: Mutex[HashMap[i32, i32]] = ...
    // allocates on heap on first call per value
}

// In a no-alloc context (interrupt handler, embedded bare-metal):
fn on_interrupt(n: i32): i32 {
    cached_compute(n)   // compiles — ! Alloc hidden by @pure
                        // may allocate at runtime → abort or corruption
}
```

This is an acknowledged limitation of `@pure`, not a design bug. `@pure` makes a **correctness** claim (internal allocation does not affect the return value) — it is not a **resource-usage** claim (no allocation will occur). These are different questions.

No-alloc callers must audit `@pure` callees manually. If a function is marked `@pure` and you are in a no-alloc context, inspect its implementation or documentation to verify it has no internal allocation path. This cannot be enforced by the type system without defeating the purpose of `@pure`.

### 2.5 Effect Polymorphism

Functions can be polymorphic over effects using **effect variables** written with a `?` prefix:

```ferrum
fn map_result[T, U, E](f: fn(T): U ! ?E, r: Result[T, E]): Result[U, E] ! ?E {
    match r {
        Ok(v)  => Ok(f(v)),
        Err(e) => Err(e),
    }
}
```

#### Syntax and Inference

Effect variables (`?E`, `?Eff`, `?E1`, `?E2`, …) are declared by use — any identifier prefixed with `?` in an effect position is an effect variable. They are **always inferred** from the argument types at each call site; you never write them explicitly as type arguments.

```ferrum
// ?E is inferred from f's concrete type at the call site
map_result(pure_fn, r)        // ?E = {} (pure)
map_result(io_fn, r)          // ?E = ! IO
map_result(net_io_fn, r)      // ?E = ! Net + IO
```

#### Multiple Independent Effect Variables

Each `?`-prefixed name is an independent variable. Two callbacks with different effects use different names:

```ferrum
fn run_both[F, G](f: F, g: G) ! ?Ef + ?Eg
    where F: Fn() ! ?Ef,
          G: Fn() ! ?Eg
{
    f()
    g()
}
// Called with f: !IO, g: !Net  →  run_both has effects ! IO + Net
// Called with f: pure, g: pure →  run_both is pure
```

#### Constraints on Effect Variables

Effect variables can be upper-bounded to restrict what the caller may pass:

```ferrum
// Sandbox: callback may only use IO or Net, never Unsafe
fn sandbox[F, T](f: F): T ! ?E
    where F: Fn(): T ! ?E,
          ?E <: {IO, Net}      // compile error if f has Unsafe, Panic, etc.
{
    f()
}

// Adapter: callback must not do IO (pure or allocation only)
fn memoize[F](f: F): impl Fn(): i32
    where F: Fn(): i32 ! ?E,
          ?E <: {Alloc, Sync}
{
    // ...
}
```

`?E <: {effects}` means "?E must be a subset of this set." The bound is checked at the call site — a caller passing a function with disallowed effects gets a compile error naming the offending effect.

#### Interaction with `@pure`

`@pure` suppresses `{Alloc, Sync}` from the visible effect set. A `@pure` function cannot accept an unconstrained `?E` callback — if `?E` resolves to `! IO`, the `@pure` guarantee would be violated:

```ferrum
// COMPILE ERROR: unconstrained ?E is incompatible with @pure
@pure
fn transform[F](f: F): i32 where F: Fn(): i32 ! ?E { f() }

// OK: ?E constrained to suppressible effects
@pure
fn transform[F](f: F): i32
    where F: Fn(): i32 ! ?E,
          ?E <: {Alloc, Sync}
{ f() }
```

#### Unification Error Messages

When `?E` resolves to an incompatible effect set, the compiler names the variable, the resolved set, and why it's rejected:

```
error: effect variable ?E resolves to `! IO + Net` but is not permitted here
  --> src/lib.fe:8:1
   |
 8 | fn transform[F](f: F): i32
   |    ^^^^^^^^^
   |
   = note: ?E inferred as `! IO + Net` from argument `f` at this call site
   = note: @pure permits only { Alloc, Sync } effects
   = help: remove @pure, or constrain `?E <: { Alloc, Sync }`
```

#### Closure Captures and the Effect System

A closure that mutates captured state has no named effect for the mutation itself — and this is correct, not a gap.

```ferrum
let mut count = 0;
let bump = || { count += 1; };  // FnMut — mutates captured state
                                 // effect: {} (pure — wrapping arithmetic, no IO/Net/Panic)
```

`bump` has no named effects. It also cannot be passed where `Fn` is required — it is `FnMut`, meaning the caller must hold `&mut` access to the closure to call it. The borrow checker enforces this. The mutation is to the caller's own local data, captured through exclusive `&mut` access that the borrow checker already tracks.

The effect system and the ownership system answer different questions and are orthogonal:

| System | Question | Mechanism |
|--------|----------|-----------|
| Effect system | Does this function affect the world *outside* the call? | `! IO`, `! Net`, `! Panic`, etc. |
| Ownership / `Fn` hierarchy | Does this function mutate *the caller's* data? | `Fn` / `FnMut` / `FnOnce` trait bounds |

Mutation through closure captures is "inside the call" from the perspective of the caller who created the captures — it is state the caller explicitly handed to the closure via `&mut`. It is not a surprise observable side effect. The caller knows the data may be mutated because the borrow checker required them to give up exclusive access to provide it.

`?E` in a higher-order function captures the named effects of the closure — IO, Net, Panic, etc. The `Fn`/`FnMut` distinction captures whether the closure mutates captured state. Both are part of the closure's full type; neither subsumes the other.

A practical consequence: a `Fn` bound in a `?E` context is not a claim that the callback does nothing surprising — it is specifically a claim that the callback does not mutate its own capture environment. The closure may still have `! IO` or `! Panic` effects, which `?E` propagates correctly.

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
extern(c) fn malloc(size: usize): *mut u8  ! Unsafe + Alloc[Heap]
```

**Proof mode** — proof functions may have no runtime effects (see §14):
```ferrum
proof fn sorted(xs: &[i32]): bool  // no effects — pure and total
```

### 2.7 `! Panic` — Sources and Inference

`! Panic` tracks functions that may abort the process via an unwind. The effect is only useful if it is both **complete** (nothing that can panic lacks the annotation) and **narrow** (only functions that can genuinely panic carry it). Both properties depend on a precise definition of what generates `! Panic`.

#### What generates `! Panic`

| Source | Example | Effect |
|--------|---------|--------|
| `unwrap()` on `Result` or `Option` | `r.unwrap()` | `! Panic` |
| `expect()` | `r.expect("msg")` | `! Panic` |
| `panic!` macro | `panic!("unreachable")` | `! Panic` |
| Explicit assertions | `assert!(x > 0)`, `assert_eq!(a, b)` | `! Panic` |
| Array/slice indexing | `arr[i]` | `! Panic` (bounds check) |

These are **explicit panic sources** — operations where the programmer has written something that may abort. The standard library declares all of them `! Panic`.

#### What does NOT generate `! Panic`

| Operation | Reason |
|-----------|--------|
| Integer arithmetic (`+`, `-`, `*`, `/`) | Wrapping semantics — overflow is defined, not a panic |
| The `?` operator | Early return on `Err` — a return, not an abort |
| `checked_add()`, `saturating_mul()`, etc. | Return `Option[T]` — no panic possible |
| `get()` on a slice | Returns `Option[&T]` — no bounds panic |

Integer arithmetic uses wrapping semantics in Ferrum. `a + b` on `i32` wraps on overflow — it does not panic. This is the critical decision that keeps `! Panic` from infecting every function that does arithmetic. Programs needing overflow detection use `checked_add()` and friends, which return `Option` and compose cleanly with `?`.

#### Closures infer `! Panic` from their bodies

Closures follow the same inference rule as private functions: the compiler infers effects from the body. A closure that calls something with `! Panic` has `! Panic` inferred automatically — no annotation required:

```ferrum
// closure body calls unwrap() — compiler infers ! Panic for the closure
let parsed: Vec<u32> = strings.iter()
    .map(|s| s.parse::<u32>().unwrap())  // closure: Fn(&str): u32 ! Panic
    .collect();
```

This propagates correctly through iterator adapters via effect polymorphism (§2.5). `Iterator::map` has the signature:

```ferrum
fn map[B, F](self, f: F): Map[Self, F]
    where F: Fn(Self::Item): B ! ?E
    ! ?E
```

`?E` is inferred as `! Panic` at this call site, so `map` itself has `! Panic`, and the enclosing function infers `! Panic`. If that function is `pub`, it must declare `! Panic` explicitly.

#### The full chain

```ferrum
// All effects inferred — no annotations needed internally
fn parse_all(strings: &[&str]): Vec[u32] {
    strings.iter()
        .map(|s| s.parse::<u32>().unwrap())   // ! Panic inferred for closure
        .collect()                              // ! Panic propagated through map
}
// Compiler infers: parse_all is ! Panic
// If pub, must declare: pub fn parse_all(strings: &[&str]): Vec[u32] ! Panic

// Contrast — no panic source, so no ! Panic
fn sum_all(numbers: &[u32]): u32 {
    numbers.iter().fold(0u32, |acc, &n| acc.wrapping_add(n))  // pure
}
// Compiler infers: sum_all is pure — no ! Panic
```

#### Contrast with other languages

Swift's `rethrows` is the closest analogue — `map` propagates throwing closures to the call site, and non-throwing closures leave the call site non-throwing. The difference: Swift does not infer that a closure throws from its body; the closure type must be explicitly declared `(T) throws -> U`. Ferrum infers it, following Koka's model.

Java's checked exceptions through streams were a failure: `Function<T,R>` in the streams API cannot declare checked exceptions, so thrown exceptions inside lambdas are rejected by the compiler. Ferrum avoids this because effect polymorphism (`?E`) is part of the type, not an afterthought.

> **Design decision:** integer arithmetic uses wrapping semantics specifically to prevent `! Panic` from becoming noise. If `+` could panic on overflow, `! Panic` would appear on nearly every non-trivial function, making it useless as a distinguishing annotation. The alternatives — Ada-style `Constraint_Error` on overflow (very noisy), Koka-style bignum default (incompatible with systems-level layout control), Rust-style no tracking at all (loses the benefit entirely) — were all worse tradeoffs. Wrapping arithmetic is the only design that preserves `! Panic` as a meaningful, sparse annotation. Programmers who need overflow detection use the explicit checked/saturating/wrapping API.

---

## 3. Capabilities and Implicit Parameters

### 3.0 Effects vs. Capabilities — Why Two Systems?

Ferrum has two constraint systems that both appear in function signatures. A function that allocates on an arena and does IO looks like:

```ferrum
fn process(data: &str): Vec[u8]  given [A: Allocator]  ! IO
```

This is deliberate. The two systems answer different questions and have different propagation rules. Understanding the distinction prevents confusion when both appear in the same signature.

#### Effects answer: "what does this function *do*?"

Effects describe **observable side effects** — actions visible to callers and the outside world. `! IO` means "this function reads or writes files, the console, or environment variables." `! Net` means "this function touches the network." `! Panic` means "this function might abort."

Effects **propagate upward**: if a callee has `! IO`, the caller must also declare `! IO` (or suppress it with `@pure` if appropriate). They appear in the function's public contract. Callers use effect annotations to audit what a function can do without reading its implementation.

#### Capabilities answer: "what does this function *need*?"

Capabilities describe **ambient resources** the function requires but does not produce. `given [A: Allocator]` means "I need a memory allocator — you decide which one." The function is parameterized over the allocator; the caller (or an enclosing `with` block) injects it.

Capabilities **propagate inward** via `with` injection, or fall back to a default. They are invisible at most call sites: `parse(src)` looks pure even though it allocates, because the allocator is ambient.

#### The key distinction: `! Alloc` vs. `given [A: Allocator]`

These two overlap conceptually but serve different purposes:

| Annotation | Meaning | Caller control |
|-----------|---------|----------------|
| `! Alloc[Heap]` | this function allocates — always on the heap | none |
| `given [A: Allocator]` | this function allocates — caller chooses where | full |
| `given [A: Allocator]` + `! Alloc[A]` | allocates; allocator is caller-controlled and named in effect | full |
| neither | no allocation | — |

A function can hard-code `! Alloc[Heap]` without `given` when the allocator is not a caller concern. A library function uses `given [A: Allocator]` to let callers substitute an arena for performance or scoping.

#### How they interact

When a function has `given [A: Allocator]` and actually allocates, its effect set includes `! Alloc[A]`. The capability *determines which allocator* appears in the effect:

```ferrum
fn build_index(data: &[u8]): Index  given [A: Allocator]  ! Alloc[A] + IO
// The allocator used for Index's memory is A — whatever the caller injected.
// The IO effect is independent — it exists regardless of which allocator is used.
```

#### Error messages come from different subsystems — intentionally

Effect violations look like:
```
error: function `process` calls `! IO` but its signature declares no IO effect
```

Capability violations look like:
```
error: no `Allocator` capability in scope
  = help: add `with Arena.new() as alloc { ... }` or `with Heap as alloc { ... }`
```

They are different errors from different parts of the compiler, which is correct: missing an effect is a type error in the function's contract; missing a capability is a resource injection error.

#### When a function has both

```ferrum
fn process(data: &str): Vec[u8]  given [A: Allocator]  ! IO
```

Read this as:
- `given [A: Allocator]` — caller decides where the returned `Vec`'s memory lives
- `! IO` — function reads from `data` (or logs) — this happens regardless of allocator choice

The two annotations are independent and can be read separately. There is no interaction between them unless the function also allocates IO-related data, in which case both appear:

```ferrum
fn read_file(path: &str): Vec[u8]  given [A: Allocator]  ! IO + Alloc[A]
// The Vec's memory uses A. The file read is IO. Both are explicit.
```

> **Design decision:** absorbing capabilities into the effect system (e.g. making `given [A: Allocator]` a form of `! Alloc[A]`) would unify the syntax, but `given` is a general mechanism for any ambient resource — not just allocators. Future capabilities (loggers, random sources, clocks) would all need effect-system representation, which would bloat the effect set with infrastructure concerns. Keeping them separate preserves the effect system's meaning: effects are about *observable side effects*, not about *which implementation of a resource is used*.

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
