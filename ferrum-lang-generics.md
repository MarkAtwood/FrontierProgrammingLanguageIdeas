# Ferrum Language Reference — Allocators and Generics

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Allocators

### 1.1 The Allocator Trait

```ferrum
trait Allocator {
    fn alloc(self: &Self, size: usize, align: usize): Result[*mut u8, AllocError]
        ! Unsafe

    fn free(self: &Self, ptr: *mut u8, size: usize, align: usize)
        ! Unsafe

    fn realloc(
        self: &Self,
        ptr: *mut u8,
        old_size: usize,
        new_size: usize,
        align: usize,
    ): Result[*mut u8, AllocError]
        ! Unsafe

    // Optional: hint that the allocator cannot allocate (compile-time check)
    const CanAlloc: bool = true
}

marker trait NoAlloc  // implemented by allocators that always return Err
```

### 1.2 Standard Allocators

| Allocator | Behavior | Use case |
|---|---|---|
| `Heap` | System allocator (malloc/free) | General purpose |
| `Arena[N]` | Bump allocator with drop tracking; drops values in reverse order on scope exit, then frees backing memory | Short-lived batches of any type |
| `Bump[N]` | Simple bump pointer; no individual frees, no drop calls — scope exit frees the whole block | Parse buffers, plain-data types only |
| `Pool[T, N]` | Fixed-size object pool | Repeated allocation of same type |
| `Stack[N]` | Stack allocator (LIFO) | Nested allocations |
| `Null` | Always returns `AllocError` | Enforcing zero-allocation paths |
| `Counted[A]` | Wraps A, tracks allocation count | Testing, debugging |
| `Failing[A, N]` | Fails after N allocations | Fault injection testing |

**`Arena` vs `Bump`:** `Arena` can store any type — it maintains a drop list and calls destructors in reverse order when the scope exits. `Bump` is simpler and faster (no drop list, one pointer reset), but the compiler rejects storing types with drop glue in a `Bump` allocator. Use `Bump` when you know your data is plain and want to pay nothing for cleanup; use `Arena` for everything else.

**Linter:** The lint `arena_trivial_drop` fires when every type stored in an `Arena` is drop-glue-free — the compiler can see that `Bump` would have been sufficient and the drop tracking overhead is wasted. This is informational, not an error: use `Bump` explicitly if you want to enforce the constraint and pay nothing for drop tracking.

### 1.3 Allocator as Capability

**The default allocator is `Heap`.** Most code never mentions allocators — you write `Vec.new()` and it allocates from the heap. Allocator annotations are only needed when you want something other than the default.

```ferrum
// Normal code — no allocator annotation, uses Heap
fn build_index(docs: &[Doc]): Index {
    let mut idx = HashMap.new()   // allocates from Heap
    for doc in docs {
        idx.insert(doc.id, doc.tokens.clone())
    }
    idx
}

// Custom allocator — explicit annotation
fn build_temp_index(docs: &[Doc]): Index  given [A: Allocator] {
    let mut idx = HashMap.new()   // allocates using ambient A
    for doc in docs {
        idx.insert(doc.id, doc.tokens.clone())
    }
    idx
}

// Two allocators: parse into arena, result goes to heap
fn parse_and_keep(src: &str)
    given [ParseAlloc: Allocator, OutputAlloc: Allocator]
    : Ast
{
    let tmp: ParseTree[ParseAlloc] = parse_tree(src)
    tmp.into_ast()   // converts to Ast, allocated with OutputAlloc
}
```

The `given [A: Allocator]` clause makes the function generic over allocators. Callers can then provide a specific allocator using `with`:

```ferrum
let arena = Arena.new(64.kb())
with arena as alloc {
    let idx = build_temp_index(&docs)  // uses arena
}  // arena drops all stored values in reverse order, then frees backing memory
```

### 1.4 Zero-Allocation Enforcement

```ferrum
// Compile error if any reachable code allocates
fn interrupt_handler(regs: &Registers)
    given [A: Allocator where A: NoAlloc]
    ! Unsafe
{
    // Any call to Vec.new(), String.from(), etc. is a compile error
    handle_irq(regs)
}

// At the call site:
with Null as alloc {
    interrupt_handler(&regs)
}
```

### 1.5 Allocation in Types

Collections are generic over their allocator. The allocator parameter is always present and always tracked by the compiler — but it is invisible at the surface by default, the same way region annotations are invisible until you need them.

```ferrum
// Surface type — allocator hidden, inferred from context
let v: Vec[i32] = Vec.new()           // allocator = ambient (Heap by default)
let v: Vec[i32] = Vec.new_in(arena)   // allocator = arena (inferred from argument)

// Explicit allocator parameter — surfaced when you need to name it
let v: Vec[i32, Arena] = Vec.new_in(arena)   // allocator named explicitly
let v: Vec[i32, _]     = Vec.new_in(arena)   // compiler infers Arena, but parameter is visible
```

The compiler always knows the allocator — `Vec[i32]` in the type of `v` is shorthand for `Vec[i32, Heap]` or `Vec[i32, Arena]` depending on how it was constructed. The short form is used when the allocator doesn't need to be communicated; the long form is used when it does.

**Allocator provenance and spawn safety:**

A value's tracked allocator determines where it can go. A task may only capture values whose allocator outlives the task's scope. This check is **recursive**: the compiler walks the full type tree of every captured value and checks the allocator of every heap-backed field, not just the outermost type.

```ferrum
scope s {
    let arena = Arena.new(64.kb())
    let data: Vec[u8] = Vec.new_in(&arena)   // compiler tracks: Vec[u8, &arena]

    // COMPILE ERROR: data's allocator (&arena) does not outlive the Heap task.
    // The task may run past the scope exit; arena drops and frees at scope exit.
    s.spawn(with Heap as alloc { process(data) })

    // ok: task is bounded by scope s, arena is also bounded by scope s
    s.spawn(with arena as alloc { process(data) })
}

// The check reaches into nested types:
scope s {
    let arena = Arena.new(64.kb())
    let strings: Vec[String[Arena], Heap] = ...  // Heap Vec, Arena String contents

    // COMPILE ERROR: String[Arena] fields inside strings use arena,
    // which does not outlive this Heap task.
    s.spawn(with Heap as alloc { process(strings) })
}
```

The error message names the mismatch and the path to it: "cannot move `Vec[String[Arena], Heap]` into task — field `String.buf` uses allocator `arena` which does not outlive this task's scope."

`Heap`-allocated values at every level of the type tree are always movable across task boundaries because `Heap` has no scope.

> **Design decision:** Allocator provenance is tracked invisibly by the compiler, not erased from
> the type system. The surface type `Vec[i32]` hides the allocator the same way `&T` hides the
> region. The spawn check is recursive through the full type tree because a scoped sub-allocation
> at any depth causes a use-after-free if it escapes the scope — checking only the outermost
> allocator would leave nested arena-backed fields unguarded. The cost of drop tracking in `Arena`
> is surfaced by the `arena_trivial_drop` lint, not enforced as a restriction.

---

## 2. Generics and Traits

### 2.1 Generic Functions

```ferrum
fn max[T: Ord](a: T, b: T): T {
    if a > b { a } else { b }
}

// Multiple bounds
fn print_sorted[T: Ord + Debug](xs: &[T]) ! IO { ... }

// Where clauses for complex bounds
fn zip[A, B, C](
    a: impl Iterator[Item = A],
    b: impl Iterator[Item = B],
    f: fn(A, B): C,
): impl Iterator[Item = C]
where
    A: Clone,
    B: Clone,
{ ... }
```

### 2.2 Trait Definitions

```ferrum
trait Display {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] ! IO
}

trait From[T] {
    fn from(value: T): Self
}

trait Iterator {
    type Item                           // associated type

    fn next(self: &mut Self): Option[Self.Item]

    // Provided methods (have default implementations)
    fn map[U](self: Self, f: fn(Self.Item): U): Map[Self, U]
    fn filter(self: Self, pred: fn(&Self.Item): bool): Filter[Self]
    fn collect[C: FromIterator[Self.Item]](self: Self): C
    fn count(self: Self): usize
    fn fold[Acc](self: Self, init: Acc, f: fn(Acc, Self.Item): Acc): Acc
}
```

### 2.3 Trait Implementations

```ferrum
impl Display for Point {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] ! IO {
        write(f, "({}, {})", self.x, self.y)
    }
}

// Blanket implementations
impl[T: Display] Display for Vec[T] {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] ! IO {
        write(f, "[")?
        for (i, item) in self.iter().enumerate() {
            if i > 0 { write(f, ", ")? }
            item.fmt(f)?
        }
        write(f, "]")
    }
}
```

### 2.4 Associated Types

Associated types bind related types together, eliminating parameter explosion:

```ferrum
trait Collection {
    type Item
    type Iter: Iterator[Item = Self.Item]
    type Alloc: Allocator

    fn len(self: &Self): usize
    fn iter(self: &Self): Self.Iter
    fn alloc(self: &Self): &Self.Alloc
}

// One parameter. The allocator and iterator type come for free.
fn count_if[C: Collection](c: &C, pred: fn(&C.Item): bool): usize {
    c.iter().filter(pred).count()
}
```

### 2.5 `impl Trait` Syntax

`impl Trait` in **parameter position** is syntactic sugar for an anonymous type parameter. These two are identical:

```ferrum
fn sum(iter: impl Iterator[Item = i32]): i32 { ... }
fn sum[I: Iterator[Item = i32]](iter: I): i32 { ... }
```

Both are statically dispatched — the compiler generates a separate version per concrete iterator type. Prefer `[T: Trait]` in parameter position when you need to name the type (e.g., for multiple parameters that must agree), and `impl Trait` when the type name is truly irrelevant.

`impl Trait` in **return position** is different — it hides the concrete type from the caller, but the type is still determined at compile time (no vtable):

```ferrum
// Return type: returns some Iterator, caller doesn't know which concrete type
fn evens(n: usize): impl Iterator[Item = usize] {
    (0..n).filter(|x| x % 2 == 0)
}
```

**Dispatch guidance:** there are two meaningful choices.

| Form | Dispatch | Use when |
|---|---|---|
| `fn f[T: Trait](x: T)` or `fn f(x: impl Trait)` | Static — monomorphized per type | Performance matters; concrete type known at compile time |
| `fn f(x: &dyn Trait)` | Dynamic — one function + vtable | Heterogeneous collections; concrete type varies at runtime |

The `impl Trait` parameter form is just abbreviated `[T: Trait]` — it does not introduce a third dispatch mechanism. When in doubt, prefer `[T: Trait]` for clarity.

For dynamic dispatch, use `dyn Trait` (see 2.6).

### 2.6 Dynamic Dispatch

```ferrum
// Trait object: dynamically dispatched
let d: &dyn Display = &point
let d: Box[dyn Display] = Box.new(point)

// Multiple traits via + (object-safe traits only)
let d: &(dyn Display + Debug)
```

Trait objects carry a vtable pointer. Only object-safe traits can be used as `dyn Trait` — a trait is object-safe if all its methods can be dispatched through a vtable (no generic methods, no `Self: Sized` requirements in method positions).

### 2.7 GADTs

GADTs (Generalized Algebraic Data Types) allow variants to specialize the type parameter:

```ferrum
enum Expr[T] {
    Lit(i32)              : Expr[i32],
    Bool(bool)            : Expr[bool],
    Add(Expr[i32], Expr[i32]) : Expr[i32],
    If(Expr[bool], Expr[T], Expr[T]) : Expr[T],
}

// Evaluator is total — the type guarantees no ill-typed expressions
fn eval[T](expr: Expr[T]): T {
    match expr {
        Expr.Lit(n)       => n,
        Expr.Bool(b)      => b,
        Expr.Add(a, b)    => eval(a) + eval(b),
        Expr.If(c, t, f)  => if eval(c) { eval(t) } else { eval(f) },
    }
}
```

GADTs enable encoding protocol state machines, typed expression languages, and other invariants that flat enums cannot express.

### 2.8 Contract Inheritance in Traits

When a trait declares contracts, implementations must respect the Liskov Substitution Principle:

- **Preconditions may only be weakened** in implementations (accept more than the trait requires).
- **Postconditions may only be strengthened** (promise more than the trait guarantees).
- **Invariants may only be strengthened.**

The compiler enforces this:

```ferrum
trait Stack[T] {
    fn push(&mut self, x: T)
        requires self.len() < self.capacity()
        ensures  self.len() == old(self.len()) + 1
}

impl Stack[T] for BoundedStack[T] {
    fn push(&mut self, x: T)
        // ERROR: strengthened precondition violates LSP
        requires self.len() < self.capacity() / 2
        ...
}
```
