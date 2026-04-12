# Ferrum for Haskell and OCaml Programmers

**Audience:** CS postdocs and researchers who think in types — fluent in Haskell, OCaml, or both. This guide skips the basics and focuses on where your intuitions transfer, where they mislead, and what's genuinely new.

The short version: Ferrum is a strict, owned-memory, systems language with an algebraic effect system, region inference, and optional machine-checked proofs. It has ADTs, parametric polymorphism, and a trait system that resembles type classes — but no HKT, no lazy evaluation, and no monad hierarchy. Effects are tracked via annotations on function types, not via type constructors. Ownership is a disciplined form of linear typing at the memory level.

---

## The Two-Minute Orientation

```ferrum
// ADTs look familiar
enum Tree[T] {
    Leaf,
    Node { value: T, left: Box[Tree[T]], right: Box[Tree[T]] },
}

// Pattern matching is exhaustiveness-checked
fn depth[T](t: &Tree[T]): usize {
    match t {
        Tree.Leaf          => 0,
        Tree.Node { left, right, .. } =>
            1 + depth(left).max(depth(right)),
    }
}

// Traits are type classes, syntactically
trait Functor[F] {
    fn fmap[A, B](fa: F[A], f: fn(A): B): F[B]
}
// ^ This does not compile. There are no HKT. More on this shortly.

// Effects are annotations, not type constructors
fn read_config(path: &str): Result[Config] ! IO { ... }

// Proof functions are erased after verification
proof fn reverse_involutive[T](xs: &[T])
    ensures xs.reverse().reverse() == xs
{ ... }
```

Key syntax differences from Haskell/OCaml:

| Concept | Haskell | OCaml | Ferrum |
|---------|---------|-------|--------|
| Generic params | `f a b` | `'a t` | `T[A, B]` |
| Function return | `a -> b` | `a -> b` | `fn(a): b` |
| Type definition | `data` / `newtype` | `type` | `type` / `enum` |
| Module path | `.` | `.` | `.` |
| Effect tracking | `IO a`, mtl | effects (5.0) | `! IO` annotation |
| Pattern match | `case ... of` | `match ... with` | `match { ... }` |
| Type class / trait | `class`, `instance` | module type, impl | `trait`, `impl` |
| Struct update | `r { f = x }` | `{ r with f = x }` | `T { f: x, ..r }` |

---

## What Transfers Directly

### Algebraic Data Types

Ferrum's `enum` is the usual sum type. Product types are `type` (like a named record).

```ferrum
// Sum type
enum Shape {
    Circle { radius: f64 },
    Rect   { width: f64, height: f64 },
    Point,
}

// Product type (no anonymous tuples-as-constructors — fields are named)
type Span {
    start: usize,
    end:   usize,
}

// Parameterized — the usual
enum Option[T] {
    Some(T),
    None,
}

enum Result[T, E] {
    Ok(T),
    Err(E),
}
```

### Pattern Matching

Exhaustiveness-checked, guards supported, nested patterns work as expected.

```ferrum
fn area(s: &Shape): f64 {
    match s {
        Shape.Circle { radius: r }   => std.math.PI * r * r,
        Shape.Rect { width: w, height: h } => w * h,
        Shape.Point                  => 0.0,
    }
}

fn classify(n: i32): &str {
    match n {
        0          => "zero",
        n if n < 0 => "negative",
        _          => "positive",
    }
}
```

Irrefutable patterns, `..` for ignoring fields, `@`-bindings (written `n @ 1..=9`), slice patterns — all present.

### Result Handling

Ferrum uses `Result[T, E]` everywhere. The `?` operator desugars to early return on `Err`, roughly like `do`-notation with `ExceptT`:

```ferrum
fn load(path: &str): Result[Config] ! IO {
    let raw = fs.read(path)?      // Err propagates, with context added
    let cfg = parse(&raw)?
    Ok(cfg)
}
```

`?` adds location context automatically. It's not monadic bind — it only works for `Result` (and `Option`) in function bodies, not in nested closures.

### GADTs

Ferrum has them. Variant types can refine the type parameter:

```ferrum
enum Expr[T] {
    Lit(i32)                         : Expr[i32],
    Bool(bool)                       : Expr[bool],
    Add(Expr[i32], Expr[i32])        : Expr[i32],
    Eq[A](Expr[A], Expr[A])          : Expr[bool],
    If(Expr[bool], Expr[T], Expr[T]) : Expr[T],
}

fn eval[T](e: Expr[T]): T {
    match e {
        Expr.Lit(n)         => n,
        Expr.Bool(b)        => b,
        Expr.Add(a, b)      => eval(a) + eval(b),
        Expr.Eq(a, b)       => eval(a) == eval(b),
        Expr.If(c, t, f)    => if eval(c) { eval(t) } else { eval(f) },
    }
}
```

The typechecker uses GADT information during match arm checking — same semantics as GHC's GADTs.

### Parametricity

Ferrum's generics are parametric in the usual sense. `fn id[T](x: T): T` must return its argument; the type system enforces this. Ownership adds a wrinkle: a `T` passed by value is consumed (moved), so the only way to return a `T` from a function that takes a `T` is to return the same value — there's no way to conjure a new `T`. Parametricity plus ownership gives you stronger free theorems about resource usage than Haskell's parametricity alone.

---

## Traits Are Not Type Classes

The surface resemblance is strong. The critical difference: **Ferrum has no higher-kinded types.**

```ferrum
// This is the trait system working as expected:
trait Eq[T] {
    fn eq(self: &Self, other: &T): bool
    fn ne(self: &Self, other: &T): bool { !self.eq(other) }
}

trait Ord: Eq[Self] {
    fn cmp(self: &Self, other: &Self): Ordering
    // provided: lt, le, gt, ge, min, max, clamp
}
```

`Ord: Eq[Self]` is a superclass constraint — same concept as Haskell. Default method implementations work. Retroactive instances work (with coherence rules: you can `impl Trait for Type` only in the crate defining the trait or the type).

**The thing that doesn't exist:**

```ferrum
// DOES NOT COMPILE — no HKT
trait Functor[F] {
    fn fmap[A, B](fa: F[A], f: fn(A): B): F[B]
}

// DOES NOT COMPILE
trait Monad[M]: Functor[M] {
    fn pure[A](a: A): M[A]
    fn bind[A, B](ma: M[A], f: fn(A): M[B]): M[B]
}
```

There are no type constructors as first-class objects that can be applied in type signatures. `F[A]` where `F` is a type variable is not valid Ferrum.

**Consequences:**
- No general `Functor`, `Applicative`, `Monad` hierarchy
- No `traverse` / `sequence` in the general sense (only concrete implementations)
- No mtl-style transformer stacks
- No `Free` monad, no `Cont` monad
- No `fmap` that works uniformly over `Option`, `Result`, `Vec`, etc.

**What exists instead:**
- `Iterator` with `map`, `filter`, `flat_map`, `fold` — the standard sequence API
- `Option` and `Result` have their own `map`, `and_then`, `or_else` methods
- The effect system handles what monads handle in Haskell (see next section)

**Why no HKT?** The design document is explicit: HKT makes error messages significantly harder to read, monomorphization (the performance mechanism) becomes more complex, and the team judged that the concrete use cases (sequencing effects, transforming containers) are served well enough by the effect system and specialized implementations. Whether this tradeoff is right is a matter of taste; it is deliberate.

### Associated Types Fill Some of the Gap

Ferrum uses associated types where Haskell would use multi-parameter type classes or functional dependencies:

```ferrum
trait Iterator {
    type Item
    fn next(self: &mut Self): Option[Self.Item]
    // map, filter, fold, collect, ... all derived
}

trait Collection {
    type Item
    type Iter: Iterator[Item = Self.Item]
    fn iter(self: &Self): Self.Iter
    fn len(self: &Self): usize
}
```

The `Item = Self.Item` syntax is a type equality constraint — identical to GHC's `Iterator (Iter coll) ~ Item coll` via functional dependencies or associated type families.

---

## Effects Are Not Monads

This is the most important conceptual shift for Haskell programmers.

In Haskell, `IO` is a type constructor. `IO a` is a value of type "computation that, when run, produces an `a`." You compose IO computations by monad laws. The IO monad is a value-level object.

In Ferrum, `! IO` is a **property of a function's type signature**, not a type constructor wrapping the return value.

```ferrum
// Haskell: getLine :: IO String
// Ferrum:
pub fn get_line(): String ! IO

// Haskell: putStrLn :: String -> IO ()
// Ferrum:
pub fn println(s: &str) ! IO

// Haskell: foo :: IO String; foo = do { line <- getLine; return (reverse line) }
// Ferrum:
fn foo(): String ! IO {
    let line = get_line()
    line.reverse()
}
```

The `! IO` annotation is inferred for private functions and required at `pub` boundaries. A function with no `!` is pure — the compiler statically enforces this; calling any impure function is a type error.

### Effect Algebra

Effects compose with `+`. Effect polymorphism is supported:

```ferrum
// Concrete effects
pub fn fetch(url: &str): Result[Bytes] ! Net
pub fn log(msg: &str) ! IO
pub fn spawn[T](f: fn(): T): Task[T] ! Sync + Alloc[Heap]

// Effect-polymorphic — caller's effect leaks through
fn with_timing[T][eff](f: fn(): T ! eff): (T, Duration) ! eff {
    let t0 = Instant.now()
    let v  = f()
    (v, t0.elapsed())
}
```

The effect variable `eff` in `[eff]` is an effect parameter, not a type parameter. The function propagates whatever effects `f` has.

### Comparison to Algebraic Effects (Koka, Frank, Eff)

Ferrum's effect system is structurally closer to algebraic effect systems (row-polymorphic effects) than to Haskell's monadic effects. The key parallels:

| Concept | Koka | Ferrum |
|---------|------|--------|
| Effect declaration | `effect state<s> { get : s; set : s -> () }` | built-in effects (no user-defined effects currently) |
| Effect annotation | `: <state<int>, io> ()` | `! IO + Sync` |
| Polymorphic effect | `<e>` row variable | `[eff]` effect parameter |
| Pure function | `: <pure> a` | no `!` annotation |

**What Ferrum lacks from algebraic effects:** user-defined effects and effect handlers. There's no equivalent of `handle { ... } with { ... }`. The built-in effects (`IO`, `Net`, `Sync`, etc.) are baked in. This is a current limitation acknowledged in the design; the compiler roadmap includes user-defined effects.

### There Is No "Run" Function

In Haskell, `runST`, `runState`, `runWriter` etc. "peel off" a monad layer. There's no analog in Ferrum. An `! IO` function is `! IO` all the way to `main`. You cannot strip the effect. This is simpler but less flexible than mtl.

### What Replaces mtl Transformer Stacks?

| Haskell pattern | Ferrum idiom |
|----------------|-------------|
| `ReaderT Config IO a` | `given [C: Config]` capability (implicit parameter) |
| `StateT s IO a` | `&mut s` explicit parameter |
| `WriterT [Log] IO a` | `&mut Logger` parameter or return `(T, Log)` |
| `ExceptT E IO a` | `Result[T, E] ! IO` |
| `MaybeT IO a` | `Option[T] ! IO` or just `Result[T, E] ! IO` |

The capability system (`given`) is the closest thing to `ReaderT` — see the Capabilities section.

---

## Ownership Is Roughly Linear Typing

Ferrum's ownership model is a disciplined form of linear typing at the memory level, with extensions.

### The Correspondence

In Linear Haskell notation (`Multiplicity`):
- `a %1 -> b` (linear): the argument is used exactly once
- `a %Many -> b` (unrestricted): the argument may be used any number of times

In Ferrum:
- `T` (by value): the value is consumed (moved) — analogous to linear `%1`
- `&T` (shared ref): the value is borrowed immutably, not consumed — analogous to unrestricted read access
- `&mut T` (exclusive ref): the value is borrowed mutably — analogous to a linear borrow
- `T: Copy` (implements `Copy`): the value is duplicated on use — analogous to unrestricted `%Many`

The ownership system is **stronger** than Linear Haskell in one way: `&mut T` enforces the exclusivity invariant at compile time across the entire borrow lifetime. There is never more than one live `&mut T` to the same data. This is what enables safe mutable references without data races.

### What Ownership Prevents

- Double-free (freed when moved out of scope; moving invalidates the source)
- Use-after-free (type system tracks liveness)
- Dangling references (borrows cannot outlive referents)
- Data races (exclusive mutation)

These are theorems, not guidelines. The type system's proof is the safety guarantee.

### The `Copy` / `Clone` Distinction

- `Copy`: trivially duplicated (implicit bitcopy). Scalars, `&T`, arrays of `Copy` types.
- `Clone`: explicitly duplicated (may heap-allocate). Call `.clone()` at the call site — the cost is visible.
- Types with destructors cannot be `Copy`.

This is the "no hidden costs" principle. Allocation and duplication are never implicit.

### Pinned Types

For self-referential types (think of values that contain pointers into themselves), Ferrum has `pinned` as a first-class concept:

```ferrum
pinned type RingBuffer[T] {
    data:  [T; 256],
    read:  *const T,   // interior pointer
    write: *const T,
}
```

A `pinned` type cannot be moved after construction. This is analogous to an address-stable value. It's more ergonomic than Rust's `Pin<P>` because it's a type-level declaration, not a wrapper.

---

## Region Inference: Tofte-Talpin in Practice

If you know the Tofte-Talpin region calculus (POPL 1997), Ferrum's region system will feel familiar. If you know MLKit, even better.

The fundamental idea: instead of tracking individual allocation lifetimes, assign allocations to named regions. When a region goes out of scope, all its allocations are freed at once. This replaces GC for typical allocation patterns.

### What the Compiler Does

Region inference is constraint-based, analogous to HM type inference. The compiler:
1. Assigns a fresh region variable to each reference
2. Generates liveness constraints from the borrow rules
3. Solves the constraint system for minimal region assignments
4. Reports a type error if the system is unsatisfiable

This is why ~90% of code needs no lifetime annotations. Annotations are only required when the constraint system is genuinely underdetermined — when the programmer's intent carries information the types alone don't convey.

### When Annotations Are Required

```ferrum
// Underdetermined: two &str inputs, one &str output
// Which input does the output live as long as?
fn choose<'a, 'b>(x: &'a str, y: &'b str, flag: bool): &'a str
    where 'b: 'a
{
    if flag { x } else { y as &'a str }
}
```

If the output lifetime is determined by the function's logic (not its inputs' relationship), annotation is needed. This is mathematically correct: inference cannot produce information that isn't there.

**Mental model for the theorist:** region variables are to lifetimes as unification variables are to types in HM. Inference works until ambiguity. Annotations are type ascriptions that resolve ambiguity.

### Explicit Region Blocks

For arena allocation patterns:

```ferrum
fn process(src: &str): Result[Output] {
    region scratch {
        let buf: Vec[u8] | scratch = Vec.new()
        let tree = parse(src, &mut buf)?
        Ok(tree.into_owned())   // must move out; scratch-allocated data cannot escape
    }
    // scratch freed here regardless of panic
}
```

The `| scratch` annotation ties the allocation to the region. The compiler enforces that scratch-allocated values cannot be returned or stored past the region boundary.

---

## Capabilities: Implicit Reader, Sort Of

The `given [A: Allocator]` mechanism provides ambient implicit parameters. The analogy to `ReaderT Env IO a` is imperfect but useful as a first approximation.

```ferrum
// Function declares it uses an ambient allocator
fn parse[T: Deserialize](src: &str): T  given [A: Allocator]

// Caller provides one via a `with` scope
with Arena.new() as alloc {
    let doc: Document = parse(src)    // A = Arena, resolved implicitly
    let meta: Metadata = parse(src)   // same arena, same resolution
}   // arena freed; all allocations gone

// Default: Heap is implicitly in scope always
fn normal_code() {
    let v = Vec.new()   // implicitly: given Heap as alloc
}
```

**Where it differs from `ReaderT`:**
- No monad: the capability is resolved at compile time, not composed with other effects
- No `ask`: you don't "retrieve" the capability; it's used implicitly by functions that declare `given`
- No `local`: you change the scope with a new `with` block, which shadows the outer one
- Multiple independent capabilities can coexist: `given [A: Allocator, L: Logger]`

**What capabilities are not:** they're not effects. An `! IO` annotation means the function does IO. `given [A: Allocator]` means the function's behavior is parameterized by the allocator — it doesn't allocate in a way that makes the choice of allocator observable to the caller (beyond performance).

---

## Contracts and Proofs: LiquidHaskell + Coq, Practically

Ferrum's verification story has two parts that will each feel familiar.

### Part 1: Refinement Types (LiquidHaskell analog)

Constrained types attach predicates to base types:

```ferrum
type Percent   = u8  where value <= 100
type NonZero   = u32 where value != 0
type SortedVec[T: Ord] {
    data: Vec[T],
    invariant forall i, j where i < j => self.data[i] <= self.data[j]
}
```

Function contracts are `requires` / `ensures`:

```ferrum
fn binary_search[T: Ord](arr: &[T], target: &T): Option[usize]
    requires arr.is_sorted()
    ensures match result {
        Some(i) => arr[i] == target,
        None    => !arr.contains(target),
    }
{ ... }
```

The compiler discharges arithmetic and comparison constraints via an SMT backend (Z3). This covers what LiquidHaskell covers: array bounds, integer arithmetic, simple structural predicates. Unlike LiquidHaskell, the logic is not embedded in types themselves — contracts are annotations on functions, not type refinements.

### Part 2: Proof Functions (Coq/Agda analog)

For properties beyond SMT (induction, recursive structures):

```ferrum
proof fn reverse_involutive[T](xs: &[T])
    ensures xs.reverse().reverse() == xs
{
    match xs {
        []             => {},
        [head, ..tail] => {
            reverse_involutive(tail)
            // ... algebraic steps the checker verifies
        }
    }
}
```

`proof fn` is total, pure, and erased after verification. Writing a proof function is structurally like writing a Coq term: you're providing the derivation, and the compiler checks it.

Two annotations link a fast implementation to a specification:

**`tested_by`** — in debug builds, the compiler calls both implementations and asserts outputs match. Runtime equivalence checking; catches bugs on tested inputs. No guarantee for untested inputs. The Haskell analogy is QuickCheck, but wired automatically by the compiler.

**`proven_by`** — you provide a proof of equivalence; the compiler verifies it at compile time. All inputs, no runtime overhead. The Haskell analogy is a Coq/Agda proof term.

```ferrum
proof fn insertion_sort_spec[T: Ord](xs: Vec[T]): SortedVec[T]
    ensures result.is_permutation_of(&xs)
    ensures result.is_sorted()
{ /* reference implementation */ }

// Tested: debug builds compare against spec, release builds skip spec
fn insertion_sort[T: Ord](xs: Vec[T]): SortedVec[T]
    tested_by(insertion_sort_spec)
{ ... }

// Proven: formal proof covers all inputs, verified at compile time
proof fn insertion_sort_correct[T: Ord](xs: Vec[T]):
    Prop[insertion_sort(xs) == insertion_sort_spec(xs)]
{ ... }

fn insertion_sort[T: Ord](xs: Vec[T]): SortedVec[T]
    proven_by(insertion_sort_correct)
{ ... }
```

This is certified programming: the spec establishes correctness, the implementation claims to satisfy it, and the annotation names what kind of claim is being made. Unlike Fstar/Dafny, proof and implementation live in the same language and compiler.

### The Four-Level Hierarchy

| Level | Mechanism | Analogy |
|-------|-----------|---------|
| Types | Ownership, constrained types | Standard type theory |
| Contracts + SMT | `requires`/`ensures`, Z3 | LiquidHaskell |
| Spec comparison | `tested_by(spec_fn)` | QuickCheck wired automatically |
| Formal proof | `proof fn` + `proven_by` | Coq/Agda proof terms |

You work at the lowest level sufficient for your problem. Most application code uses only level 1. Library authors use levels 2–3. Core infrastructure (crypto, sort, arithmetic) uses level 4.

---

## What Ferrum Deliberately Does Not Have

### No Lazy Evaluation

All evaluation is strict. There are no thunks, no space leaks from unevaluated closures, no `seq` / `deepseq` gymnastics. `fib n = fib (n-1) + fib (n-2)` is not infinite — it terminates (or stack-overflows).

For genuinely deferred computation: `Lazy[T]` is a library type that wraps an `Option[T]` and a closure, evaluating once on first access. It's explicit and transparent.

### No Higher-Kinded Types

Already covered above. The design is deliberate and documented. If you are used to the expressive power of `mtl` or `polysemy`, this will feel restrictive. The compensation is that generic code is more predictable and errors are more local.

### No OCaml-Style Functors

There is no mechanism for parameterizing modules over other modules. The capability system handles part of this (parameterizing behavior over allocators/loggers/etc.), but it does not replicate the full power of OCaml's parameterized module system. You cannot abstract over the implementation of a data structure the way an OCaml functor does.

### No Implicit Coercions

There are no implicit numeric widening conversions. `i32` does not coerce to `i64`. Every conversion is explicit. This is verbose at first and then invisible once you're used to it.

### No Currying

Functions take their parameters all at once. Partial application is not built in. Use closures explicitly:

```ferrum
// Not: add 1  (returns fn(i32): i32)
// Instead:
let add_one = |x| add(1, x)
```

### No Rank-N Types

`forall` quantification is restricted to the outermost level (rank 1). There is no `runST`-style rank-2 polymorphism. In practice this means some ST-monad patterns are not expressible; the typical workaround is explicit region blocks.

### No Orphan Instances

The coherence rules are the same as Rust: you can implement `trait T for Type` only in the crate that defines `T` or the crate that defines `Type`. This prevents conflicting instances across libraries at the cost of occasional newtype boilerplate.

---

## Mental Model for the Type-Theoretically Trained

Here is the cleanest way to think about Ferrum's design:

**The type system is a four-axis space:**

```
              ┌───────────────────────────────────┐
              │  Effect axis     IO · Net · Sync  │   → what the function does
              │  Ownership axis  Owned · & · &mut │   → aliasing and lifetime
              │  Type axis       Scalar · Sum · ∏ │   → the value's structure
              │  Allocator axis  Heap · Arena · ⊥ │   → where memory lives
              └───────────────────────────────────┘
```

These axes are orthogonal. A type like `Result[Vec[u8] | Arena, Error] ! IO + Net` is a point in this space: the type axis is `Result[...]`, the allocator axis is `Arena`, the effect axis is `IO + Net`.

**Annotations are incomplete derivations:**

Every annotation in a Ferrum program is a place where the inference engine has insufficient information. The compiler's inference rules, if complete, would make all annotations redundant. As inference improves, annotations become warnings ("this is now inferred; annotation unnecessary"). The asymptote is zero annotations — not 100% proofs.

This is exactly how HM inference relates to explicit type annotations: an annotation is a hint to the type checker, useful when inference would otherwise produce an overly polymorphic or ambiguous type.

**Ownership is a substructural discipline:**

The borrow rules implement an affine type system (use at most once for owned values) plus a uniqueness type system (at most one live mutable reference). This is stronger than Linear Haskell's `Multiplicity` in that it tracks aliasing, not just consumption. The trade-off: you cannot share a `&mut T` — you can only pass it forward.

**Effects are row types without the row operations:**

The `! IO + Net` effect annotation is a subset of a row type over a fixed set of built-in effect labels. Effect polymorphism (`[eff]`) is row polymorphism in this restricted sense. What's missing: you cannot introduce new labels, you cannot handle/catch specific effects (no algebraic effect handlers). This makes the system predictable and simple to reason about, at the cost of expressiveness.

---

## Quick Translation Table

| Haskell / OCaml | Ferrum | Notes |
|----------------|--------|-------|
| `data Maybe a = Nothing \| Just a` | `enum Option[T] { None, Some(T) }` | |
| `data Either e a = Left e \| Right a` | `enum Result[T, E] { Ok(T), Err(E) }` | |
| `newtype Foo = Foo Bar` | `type Foo { inner: Bar }` or `type Foo = Bar` | no zero-cost newtype yet |
| `f <$> x` | `x.map(f)` | only on concrete types |
| `f <*> x` | no direct analog | |
| `x >>= f` | `x.and_then(f)` | only on `Option`/`Result` |
| `do { x <- m; f x }` | `let x = m?; f(x)` | `?` only for Result/Option |
| `IO a` | `(): T ! IO` (annotated fn) | effect is on fn, not value |
| `class Eq a` | `trait Eq[T]` | no HKT |
| `instance Eq Int` | `impl Eq[i32] for i32` | |
| `deriving (Show, Eq)` | `@derive(Debug, Eq)` | fixed set |
| `forall a. a -> a` | `fn id[T](x: T): T` | rank-1 only |
| `let rec f x = ...` | `fn f(x: T): U { ... }` | functions are recursive by default |
| `module M : S = struct ... end` | `mod m { ... }` | no functors |
| OCaml effect handlers | no equivalent currently | planned |
| LiquidHaskell `{-@ f :: x:Int -> {v:Int | v > x} @-}` | `fn f(x: i32): i32 ensures result > x` | |
| Coq `Theorem`, `Proof` | `proof fn` + `proven_by`, verified by type checker | |
| `unsafePerformIO` | `unsafe { ... }` | explicit unsafe block |
| `ST s a` with `runST` | `region` blocks | no rank-2 needed |

---

## Where to Start

If you're reading Ferrum code: the syntax will look strange for a day, then mechanical. Ignore the `!` annotations initially — they're consistent with your mental model of what functions do. Focus on the ownership constraints when something doesn't typecheck.

If you're writing Ferrum code: write it like strict Haskell with mutable references. Ownership errors are the compiler telling you where aliasing would be unsafe. The effect annotations write themselves once you know what your function does.

The type system enforces things you previously checked by convention or with proofs elsewhere. The budget of cognitive overhead shifts from "remember the invariants" to "understand the error message once."

---

*See also: `ferrum-learning-for-ai.md` for a dense syntax/feature reference. `ferrum-introduction-to-proofs.md` for verification details. `ferrum-introduction-to-ownership.md` for the borrow system in depth.*
