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
    type CanAlloc: bool = true
}

marker trait NoAlloc  // implemented by allocators that always return Err
```

### 1.2 Standard Allocators

| Allocator | Behavior | Use case |
|---|---|---|
| `Heap` | System allocator (malloc/free) | General purpose |
| `Arena[N]` | Bump allocator, frees all at once | Short-lived batches |
| `Pool[T, N]` | Fixed-size object pool | Repeated allocation of same type |
| `Stack[N]` | Stack allocator (LIFO) | Nested allocations |
| `Bump` | Simple bump pointer, no free | Parse buffers |
| `Null` | Always returns `AllocError` | Enforcing zero-allocation paths |
| `Counted[A]` | Wraps A, tracks allocation count | Testing, debugging |
| `Failing[A, N]` | Fails after N allocations | Fault injection testing |

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
    let tmp: ParseTree | ParseAlloc = parse_tree(src)
    tmp.into_ast()   // converts to Ast, allocated with OutputAlloc
}
```

The `given [A: Allocator]` clause makes the function generic over allocators. Callers can then provide a specific allocator using `with`:

```ferrum
let arena = Arena.new(64.kb())
with arena as alloc {
    let idx = build_temp_index(&docs)  // uses arena
}  // arena and all its allocations freed here
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

Collections carry their allocator as an associated type:

```ferrum
type Vec[T] {
    ptr: *mut T,
    len: usize,
    cap: usize,
    alloc: A,   // A resolved at construction
    ...
}

// Construction uses ambient allocator
let v: Vec[i32] = Vec.new()           // A = ambient
let v: Vec[i32] = Vec.new_in(Arena)   // A = Arena, explicit override
```

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

For return types and function parameters where the concrete type is unimportant:

```ferrum
// Parameter: accepts any Iterator over i32
fn sum(iter: impl Iterator[Item = i32]): i32 {
    iter.fold(0, |a, b| a + b)
}

// Return type: returns some Iterator, caller doesn't know which
fn evens(n: usize): impl Iterator[Item = usize] {
    (0..n).filter(|x| x % 2 == 0)
}
```

`impl Trait` in return position is monomorphized — the concrete type is determined at compile time. For dynamic dispatch, use `dyn Trait` (see 2.6).

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
