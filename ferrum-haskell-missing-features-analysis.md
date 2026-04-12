# Can the "Missing" Haskell Features Be Added to Ferrum?

**Context:** The Haskell introduction lists seven things Ferrum deliberately lacks. This document analyzes each: can it be added without breaking the compiler, the cognitive model, or the semantic coherence of the language?

The short answer for each, before the detail:

| Feature | Verdict | Limiting factor |
|---------|---------|----------------|
| Lazy evaluation | **No** | Fundamental triple conflict: ownership + effects + no-hidden-costs |
| Higher-kinded types | **Partial yes** | Already in compiler (stdlib uses it). Pure containers: go. IO monad pattern: architecturally blocked. |
| OCaml-style functors | **Probably not worth it** | Marginal gain over capabilities + HKT; large complexity cost |
| Implicit coercions | **Deref: yes / Numeric: no** | Deref coercions are zero-cost and safe. Numeric widening violates a core design principle. |
| Currying | **Yes (sugar), with caveats** | `&mut` partial application gets subtle; owned/Copy cases are clean |
| Rank-2 types | **Yes** | Feasible; primary use case already covered by `region` blocks |
| Orphan instances | **Partial relaxation only** | Can relax at intra-crate level; full relaxation breaks ecosystem coherence |

---

## 1. Lazy Evaluation — No

Not a design choice. Three simultaneous structural conflicts.

### Conflict 1: Ownership

A thunk captures values. An owned value captured in a thunk transfers ownership into the thunk. Now the thunk's evaluation site is the "use" of that owned value — but the thunk may be stored arbitrarily far from the capture site, evaluated at an arbitrary time, or evaluated multiple times (if the thunk is a function rather than a `Once` cell).

For borrows: a thunk that captures `&T` holds that borrow live for an arbitrarily long time. The borrow checker currently bounds borrow lifetimes by lexical scope and region inference. Lazy evaluation makes borrow duration depend on when the thunk is forced — runtime data. The borrow checker cannot reason about runtime data. You would need to either:
- Conservative-extend borrows to the thunk's entire scope (making lifetimes very wide, killing aliasing analysis)
- Or do some form of demand analysis (an undecidable research problem in the presence of arbitrary aliasing)

Either way, the borrow checker either becomes unsound or refuses large amounts of valid code.

### Conflict 2: Effects

Ferrum's effect annotations are on *call sites* — when you call a function, you know immediately which effects it has. This is what makes effect inference tractable and what gives the `!` annotation its meaning.

Lazy evaluation decouples the point of capturing a computation from the point of running it. A thunk that captures `get_line()` doesn't do IO when the thunk is created — it does IO when the thunk is forced. The caller creating the thunk has no `! IO` in its signature; the caller forcing the thunk does.

This is exactly Haskell's situation, and it explains why Haskell's `IO` is a type constructor rather than a function annotation. The effect is in the value's type precisely because you can't know at the call site when evaluation happens.

Ferrum's effect system design is the *dual* of Haskell's: effects on function types (transparent, inferrable) vs. effects on value types (composable, but requiring explicit binding). Adding lazy evaluation to Ferrum would require migrating the effect system to the Haskell architecture — an architectural replacement, not an addition.

### Conflict 3: No Hidden Costs

A thunk allocation is an allocation the type system doesn't see. The principle "every allocation visible in type or code" would require every lazy expression to have a visible annotation. At that point you have explicit `Lazy[T]` (which Ferrum already has as a library type). That is the correct boundary: explicit `Lazy[T]` when you want laziness, strict everywhere else.

**The limit of laziness in Ferrum:** `Lazy[T]` (call-once thunk), `Iterator` (pull-based lazy sequences). These are the two lazy patterns that don't conflict with ownership or effects because they're explicit types, not a global evaluation strategy.

---

## 2. Higher-Kinded Types — Partial Yes

This requires the most nuanced answer.

### Key fact: HKT is already in the compiler

The `ferrum-lang-core.md` non-goals list says: "Higher-kinded types in user-facing code. The stdlib uses them internally." The compiler implements HKT. The decision was to not expose it. This reframes the question as: *should it be exposed, and for what?*

### Pure container operations: yes, no conflict

```ferrum
// This is feasible
trait Functor[F[_]] {
    fn fmap[A, B](fa: F[A], f: fn(A): B): F[B]
}

impl Functor[Option] {
    fn fmap[A, B](fa: Option[A], f: fn(A): B): Option[B] {
        match fa {
            Some(a) => Some(f(a)),
            None    => None,
        }
    }
}

trait Foldable[F[_]] {
    fn fold_right[A, B](fa: F[A], init: B, f: fn(A, B): B): B
}
```

`fmap` over `Option`, `Result`, `Vec`, or user-defined containers works. There's no conflict with the ownership system (you take `F[A]` by value, return `F[B]` by value — same as any other transformation) and no conflict with the effect system (the function `f` is pure here; you could make it effect-polymorphic: `fn fmap[A, B][eff](fa: F[A], f: fn(A): B ! eff): F[B] ! eff`).

The `Applicative` hierarchy works for pure containers too. `Traversable` works if you define it over specific effect types rather than abstractly.

### Effectful computations (IO monad): architecturally blocked

This is the crux.

In Haskell: `IO a` is a value. `IO` is a type constructor. `fmap :: (a -> b) -> IO a -> IO b` is well-typed because `IO` is first-class.

In Ferrum: there is no type `IO[A]`. There is only the annotation `! IO` on function types. The closest encoding is `type Thunk[A, eff] = fn(): A ! eff` — a zero-argument closure that has some effects.

Could you define `Monad[Thunk[_, IO]]`?

```ferrum
impl Monad[fn(): _ ! IO] {
    fn pure[A](a: A): fn(): A ! IO { || a }
    fn bind[A, B](ma: fn(): A ! IO, f: fn(A): fn(): B ! IO): fn(): B ! IO {
        || f(ma())()
    }
}
```

This is *syntactically* expressible if HKT is exposed. But it has a semantic problem: calling `bind(ma, f)` is pure — it just creates a new closure. No IO happens. The IO happens when you call the result. The `! IO` effect of `bind` itself would be nothing; only the *result* carries the effect.

This decouples effect-at-construction from effect-at-execution — which is exactly the structure of the IO monad in Haskell, and exactly what Ferrum's effect system was designed to avoid.

**The architectural conflict in one sentence:** Ferrum guarantees that if you call a function without `! IO` in its type, *no IO happens when you call it*. The IO monad pattern requires that calling `bind` (or `>>=`) doesn't do IO — it constructs a value that *represents* IO. These two properties cannot coexist for the same concept of "IO."

### What HKT exposure would give you

- `Functor`, `Foldable`, `Traversable` for pure containers — yes
- `Applicative` for pure containers — yes
- `Monad` for `Option` and `Result` (no effects) — yes, this is just `and_then` with a name
- `Monad[IO]` in the Haskell sense — no, and making this work would require redesigning the effect system

### The research path forward

If someone wanted both HKT and an IO monad in a system with Ferrum-style effects, the relevant literature is *graded monads* (Katsumata 2014) and *parameterized monads* — type constructors that carry effect information in their type index: `M[eff, A]` where `eff` is a type-level effect set. This is active research. It doesn't exist as a deployed language feature anywhere at scale.

---

## 3. OCaml-Style Functors — Probably Not Worth It

OCaml functors solve "parameterize a module over another module's implementation." The canonical example:

```ocaml
module Make(Key : OrderedType) : S with type key = Key.t
```

Ferrum has two mechanisms that together cover most functor use cases:

1. **Capabilities** (`given [K: Ord]`): Parameterize behavior over a trait implementation. This handles the `OrderedType` parameterization directly.

2. **HKT** (if exposed): Generic data structures over container type constructors.

What genuine OCaml functors add beyond these:
- *Type abstraction at the module level* — the result `S with type key = Key.t` exposes a selected part of the signature. This is more than just generic functions; it's parameterized opaque types.
- *Sharing constraints* — the ability to say "the Key type in this module is the same as the Key type in that module" across module boundaries.

These are real capabilities. They'd require a new kind of module-level parameterization in Ferrum. The compiler cost is significant (module-level type checking is a separate pass). The cognitive cost is significant (OCaml programmers spend years getting comfortable with functors).

Given that capabilities handle the allocation/strategy parameterization case and HKT handles the container-shape case, the remaining functor use cases are architectural pattern cases (pluggable data structure implementations). These come up in library design, not application code. The case for adding full functors is weak against the complexity cost.

**Conclusion:** Don't add. If the HKT for pure containers is exposed, that gets you `Map.Make`-style parameterization via trait bounds. That's enough.

---

## 4. Implicit Coercions — Deref Yes, Numeric No

These are two different questions that happen to share a name.

### Deref coercions (e.g., `&String → &str`, `&Vec[T] → &[T]`): yes

These are:
- Zero-cost (same pointer, narrower type)
- Always safe (reference narrowing cannot alias in unsafe ways)
- Already in Rust and well-understood
- Direction-consistent (always from owned/wider to borrowed/narrower)

They don't violate "no hidden costs" because there's no cost to hide. They don't interact badly with ownership because they're always on borrows. Adding a `Deref` trait with implicit coercion at borrow sites is feasible and ergonomic.

### Numeric widening coercions (e.g., `i32 → i64` implicitly): no

The specific objection isn't cost (widening a primitive is zero-cost). It's the slippery slope:

- `i32 → i64`: fine, zero cost
- `i32 → f64`: technically defined behavior, small but nonzero semantic change (not all i32 values are exactly representable as f64)
- `i32 → BigInt`: heap allocation hidden in an arithmetic expression
- `String → &str`: a deref, fine
- `&str → String`: heap allocation

Once implicit coercions exist, every programmer must mentally track which conversions are implicit and which allocate. The rule "all conversions are explicit" is simple and complete. Any relaxation requires knowing the coercion lattice. The tradeoff in Ferrum's context (systems programming, no hidden costs) favors the strict rule.

The ergonomic hit is real but bounded: numeric conversions appear at system boundaries (parsing, FFI, serialization), not in every expression. The strictness is a cost you pay once at integration points.

---

## 5. Currying — Yes as Sugar, With a Specific Caveat

### Owned and Copy types: clean

```ferrum
fn add(x: i32, y: i32): i32 { x + y }
// With currying sugar, partial application:
let add5 = add(5, _)   // or: add(5) as fn(i32): i32
```

For `Copy` types, partial application captures a copy. For owned types, partial application moves the value into the closure. Both are semantically well-defined and the ownership rules are satisfied.

### Borrowed types: the specific caveat

```ferrum
fn longest<'a>(a: &'a str, b: &str): &'a str { ... }
```

If you partially apply with the first argument: `longest(s)` returns `fn(&str): &'a str` where `'a` is tied to `s`'s lifetime. The closure type captures the lifetime of `s`. This is expressible in Ferrum's type system but the resulting closure type is complex: `fn(&str): &str` where the output lifetime depends on the captured `s`.

Callers of the partially applied function then need to understand that the returned closure has a lifetime dependency on `s`. This is correct but non-obvious. In Haskell, lifetimes don't exist, so this complexity disappears.

### The `&mut` case: genuinely problematic

```ferrum
fn modify(buf: &mut Vec[u8], val: u8) { buf.push(val) }
```

Partially applying with `buf`: the returned closure holds a `&mut Vec[u8]`. The original binding is now exclusively borrowed by the closure until the closure is dropped. No other code can touch `buf` until you either call the closure or drop it. This is semantically correct — the borrow checker enforces it. But it means that partial application of a mutable-reference-taking function is a source of "why can't I use this variable?" confusion that's hard to diagnose.

### Verdict

Currying as syntactic sugar (with explicit `_` or trailing partial application syntax) can be added without changing the language semantics. The compiler desugars to closures. The ownership rules work correctly. The cognitive overhead comes only at borrow sites, and specifically `&mut` partial application should probably be syntactically distinguishable so the exclusive borrow is visible.

---

## 6. Rank-2 Types — Yes

### The ST pattern: already covered

The canonical motivation for rank-2 (`runST :: (forall s. ST s a) -> a`) is: prevent state from escaping the computation by making the state type universally quantified over the argument. The caller provides a function that works for *any* state thread, guaranteeing the state can't be smuggled out.

Ferrum's `region` blocks provide the same guarantee at the language level:

```ferrum
region r {
    let buf: Vec[u8] | r = Vec.new()
    let result = compute(&mut buf)
    result.into_owned()  // must move out; r-allocated values cannot escape
}
```

The `region r` scope ensures `r`-allocated values don't escape. The compiler enforces this without rank-2. So the primary use case is covered.

### What rank-2 would add beyond regions

Callback APIs where the callback must be polymorphic:

```ferrum
fn with_temp_buffer[A](f: forall[r]. fn(&mut Vec[u8] | r): A): A {
    region r {
        let mut buf: Vec[u8] | r = Vec.new()
        f(&mut buf)
    }
}
```

Here `f` must work regardless of which region the buffer is in. With explicit regions and rank-2, you can express "this callback works with buffers in any region." This is a real pattern in resource-safe APIs.

### Feasibility

Rank-2 type inference requires bidirectional checking — the type of the rank-2 argument must be provided by annotation (it can't be fully inferred). Ferrum already has annotations; this is no different. The compiler extension is moderate and well-understood (GHC has done it, MLton has done it, Standard ML extensions have explored it).

The cognitive overhead is low for rank-2 specifically: the only new concept is "this argument is a polymorphic function." Users who need this will already be thinking about polymorphic callbacks.

**Conclusion:** Add rank-2 when the use cases arise. It's additive, doesn't conflict with anything, and region blocks mean you won't need it often.

---

## 7. Orphan Instances — Partial Relaxation Only

The current rule (coherence: impl only in the crate defining the trait or the type) is correct as a default. The question is whether it can be relaxed for legitimate use cases.

### Why full relaxation fails

If crate A defines `impl Serialize for uuid::Uuid` and crate B also defines `impl Serialize for uuid::Uuid`, any program depending on both A and B gets a conflict. Since the `uuid` crate and the `Serialize` trait are in neither A nor B, neither crate has violated anything locally. But together they break. This is the orphan problem: local correctness doesn't imply global correctness.

In a large ecosystem, this scales catastrophically. The Haskell ecosystem has experienced this pain repeatedly.

### Partial relaxations that don't blow up

- **Option 1: Explicit orphan with warning.** `@allow(orphan) impl Trait for Type { ... }`. Compiles with a warning. If two crates define conflicting orphan impls and are linked together, the linker error is local and fixable by the linker, not by the trait or type authors.
- **Option 2: Intra-crate orphans.** Within a single crate, orphan impls are allowed. At crate boundaries, coherence is enforced. This prevents ecosystem collisions while allowing a crate to define impl glue between two external types it depends on.
- **Option 3: Sealed traits.** A trait author can mark their trait `#[allow_orphan_impls]`, opting in to allowing external impls. The trait author owns the coherence risk.
- **Option 4: Newtype requirement lifted for local types.** If the *type* is new (even if defined in another crate) you can implement any trait for it. Currently the rule requires you to own the trait or type; relaxing to "or the type is used only in your crate's public interface" covers some glue-code cases.

Of these, options 2 (intra-crate orphans) and 3 (sealed traits) are the cleanest. Neither creates global coherence problems.

### What doesn't change

The newtype workaround remains necessary for true external-external combos. This is a low but real ergonomic cost. The payoff — coherent, deterministic generic dispatch — is worth it for a language that wants to be predictable.

---

## The Asymmetry Worth Noting

Six of seven missing features have reasonable paths forward. Lazy evaluation does not. That's not a coincidence — Ferrum's three interacting core systems (ownership, effects, no-hidden-costs) are each individually compatible with laziness in some language, but together they form a constraint that excludes it.

The right way to read the "what's missing" list: all of these are **design choices except lazy evaluation**. The lazy evaluation omission is a **consequence** of the design.

For a Haskell researcher: the architectural choice between "effects on function types" (Ferrum/Koka) and "effects as value types" (Haskell's IO) is the deepest fork. Everything else can be grafted onto either architecture. The effect architecture choice closes the door on pervasive laziness and the IO monad specifically, while opening the door to transparent effect inference that Haskell cannot have.

---

## Addendum: The HKT + Effects Research Frontier

If you want both HKT *and* effect-carrying type constructors that compose like monads, the current research is:

- **Graded monads** (Katsumata 2014, Orchard & Yoshida 2016): `M[eff, A]` where `eff` is a type-level effect annotation. `bind :: M[e1, A] -> (A -> M[e2, B]) -> M[e1 ∪ e2, B]` — the effects compose in the type.
- **Parameterized monads / indexed monads** (McBride, Atkey): pre/postcondition types for sequencing.
- **Algebraic effects with row polymorphism** (Koka): what Ferrum is closer to, but without user-defined effect handlers.

None of these is in a mainstream deployed language at scale. Ferrum's effect system is the row-polymorphism design without handlers. Adding graded monads on top of that, combined with ownership, would be genuinely novel — and genuinely hard. It is not "can we add HKT" but "can we design a new kind of type system that hasn't been built before." The answer is probably yes, eventually, but it's a research program, not a feature request.
