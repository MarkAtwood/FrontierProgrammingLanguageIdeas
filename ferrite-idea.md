# Ferrite — Idea Dump

Ferrite is "Ferrum with GC." Same family. Different tradeoffs.

This document is a raw thought dump from a design conversation. Not polished. Not final.
Come back to this when ready to turn it into a real spec.

---

## The Name

Ferrite is iron oxide — the same element as iron (ferrum), softer, more stable, chemically
combined with oxygen, used in completely different applications (transformer cores, inductors,
magnetic shielding — not structural steel). The metaphor holds exactly. Ferrite is Ferrum that
has been "oxidized" by a GC. Same element, different form, different domain.

---

## The Core Bet

Ferrum's load-bearing principle is "no hidden costs." Every allocation, copy, synchronization,
and effect is visible in the code or its type signature.

Ferrite relaxes that to one specific hidden cost: the GC. You don't know when it runs. You
don't see it in the code. It just handles memory for you.

In exchange, Ferrite proposes a different load-bearing principle: **"no surprising behavior."**

- No null: `Option[T]` tells you explicitly.
- No exceptions: `Result[T]` and `?` propagate explicitly.
- No hidden side effects: the effect system tells you explicitly.
- No resource leaks: `resource` types enforce close explicitly.
- Cycles: the GC handles them — not surprising, not your problem.
- GC pauses: the one admitted hidden cost. Bounded by `@no_gc_pause` where it matters.

---

## What Gets Dropped from Ferrum

### Pinned types

The entire `pinned` keyword and everything it implies goes away.

`pinned` existed in Ferrum because self-referential structs can't move — their interior
pointers would become dangling. Async state machines are the main case: they hold references
to their own locals across `.await` points.

In Ferrite, async state machines are GC heap objects. The GC has precise type information
(GC maps, see below). It knows exactly which fields are pointers. When it compacts, it updates
ALL pointers to a moved object — including interior pointers within the same object. A precise
GC can safely move self-referential objects.

What drops with `pinned`:
- The `pinned` keyword
- `UnsafePinned<T>` equivalent (not needed)
- `noalias` suppression on `&mut PinnedType` (not needed)
- `Pin<Box<dyn Future>>` equivalent (not needed)
- The whole design note about Rust spending years fixing this

Async in Ferrite is clean:

```ferrite
async fn fetch(url: &str): Result[String] ! Net {
    let body = http.get(url).await?
    Ok(body.text().await?)
}
```

The state machine is a GC object. The GC traces it. The GC can move it. No pinning required.

### Explicit allocators

The entire capability-based allocator system goes away:
- `given [A: Allocator]` — gone
- `with Arena.new() as alloc { }` — gone
- `! Alloc[A]` effect — gone
- `@no_alloc` attribute — gone (replaced by `@no_gc_pause @no_alloc` combination for hard guarantees)
- `NoAlloc` marker trait — gone
- Named capability system (`given`) — probably gone, or kept only for non-allocator use

This was the most complex part of Ferrum's ergonomics story for library authors. It's the right
design when you need to let callers choose between heap, arena, pool, and stack. Ferrite says:
the GC allocates, period. Callers don't choose.

What this loses: arena allocation for bulk parsing, pool allocation for hot paths, stack
allocation for tiny temporaries. You can still get most of this implicitly — a good generational
GC makes short-lived objects nearly free, because the young-gen bump pointer is a pointer
increment with no synchronization. You just can't control WHERE things live.

### Metal / bare-metal / OS targets

Ferrite assumes it runs on a real OS with a real runtime. No freestanding targets.
No interrupt handlers. No bootloaders.

What drops:
- `@no_alloc` (the embedded use case for it)
- Bare-metal concurrency primitives
- `Layout` declarations for hardware register maps (probably)
- No requirement for the language to bootstrap its own runtime

What this means practically: Ferrite requires a GC runtime to be present. This is a
dynamic library or a static link, like the JVM or the Go runtime. You can't compile a Ferrite
binary that runs before memory management is initialized.

### `Rc`, `Arc`, `Weak`

All three go away. These smart pointer types exist precisely because the language has no GC and
needs to implement reference semantics manually. `Weak` exists specifically to break cycles that
reference counting can't collect.

A tracing GC finds and collects cycles automatically. You just allocate. The GC figures out
what's reachable.

Gone:
- `Rc[T]` — single-threaded reference counting
- `Arc[T]` — atomic reference counting for thread-safe sharing
- `Weak[T]` — non-owning reference to break cycles
- `Rc.downgrade()`, `Weak.upgrade()`
- The whole "strong count / weak count" mental model
- The "forgot to break the cycle, now you have a memory leak" failure mode

What replaces them: just... variables. If multiple things reference the same heap object, the
GC keeps it alive. Cycles don't leak. You don't think about it.

### Lifetime / region annotations

The borrow checker's *liveness* job is done by the GC. If an object is referenced, it's alive.
It cannot be freed while anything points to it. There is nothing to enforce about "borrows not
outliving owners" because the GC ensures the owner is always alive as long as any borrow exists.

What drops:
- `'a` lifetime annotations on references
- `where 'b: 'a` lifetime bounds
- `region` blocks (from Ferrum)
- Region inference (the algorithm survives in a simpler form, see below)

What survives: the borrow checker exists but is reduced to **aliasing rules only**:
- Any number of `&T` (shared references) may coexist
- Exactly one `&mut T` (exclusive reference) may exist
- Shared and exclusive cannot coexist

These rules prevent data races on shared data without locks. They're checkable locally
(no interprocedural lifetime analysis needed). They're what programmers should think about
when designing APIs. The rules stay.

What's gone: any annotation on those references. `&T` has no lifetime. The compiler checks
the aliasing rules lexically within each function. It can't fail to compile code that's
logically correct about aliasing.

### `unsafe` blocks (mostly)

Most unsafe code in Ferrum exists because of raw pointers, manual allocation, and pinned
invariants. Remove those and the unsafe surface shrinks dramatically.

In Ferrite, `unsafe` might survive only for:
- FFI (calling C functions — always unsafe)
- Intrinsics that bypass the type system

Or it might go away entirely and FFI gets its own mechanism. TBD.

Raw pointers (`*const T`, `*mut T`) — probably gone from safe Ferrite code entirely.

### `RefCell[T]`

Probably goes away. `RefCell` exists to move borrow checking to runtime for cases where the
compile-time checker is too conservative. Without lifetimes, the borrow checker is much less
conservative (it only checks aliasing, which is local). Most cases that needed `RefCell` in
Ferrum/Rust don't need it in Ferrite because the borrow checker doesn't get confused about
whether a borrow "escapes" somewhere.

If runtime borrow checking is still needed in some edge case, it could be a library type.
But it's no longer a core primitive.

---

## What Gets Kept from Ferrum

### The effect system (simplified)

Effects survive intact, but the set shrinks. Without allocator capabilities, `! Alloc[A]`
goes away.

Remaining effects:

| Effect | Meaning |
|--------|---------|
| `! IO` | reads/writes files, pipes, stdin/stdout, environment |
| `! Net` | network operations |
| `! Sync` | acquires or releases locks/mutexes |
| `! Async` | contains `.await` points |
| `! Panic` | may abort via unwrap/assert/panic |
| `! Global` | reads/writes mutable global or thread-local state |

Effect inference survives: private functions don't need annotations, the compiler infers.
Public functions must annotate. The compiler tells you what to write if you omit it.

This is the biggest differentiator Ferrite has over every existing GC'd language. No Go,
Java, Kotlin, Python, JavaScript, or Ruby language tracks effects. This is novel for a GC'd
language.

`! GC` as an effect: probably don't add it. The whole point of a GC is that you don't think
about allocation. Requiring `! GC` on every allocating function would be noise. Exception:
if there's a `@no_gc_pause` interaction that needs to be effect-checked, maybe. But probably
not. The `@no_gc_pause` attribute handles that path.

### Sum types, traits, generics, pattern matching

All survive unchanged from Ferrum. `enum`, `struct`, `trait`, `impl`, `match`. Sum types
with data. Exhaustive pattern matching. No inheritance.

### `Option[T]`, `Result[T]`, `?`

All survive. No null. No exceptions. The `?` operator propagates errors.

### Contracts

`requires` and `ensures` survive. They don't care about the memory model. A function that
requires its input to be sorted is a function that requires its input to be sorted, whether
the input is on the stack, the heap, or the GC heap.

```ferrite
fn binary_search[T: Ord](haystack: &[T], needle: &T): Option[usize]
    requires haystack.is_sorted()
    ensures match result {
        Some(i) => haystack[i] == *needle,
        None => !haystack.contains(needle),
    }
```

This is one of Ferrite's value propositions over Go, Java, and Kotlin: design-by-contract
at the language level.

### Proofs

`proof fn` and formal verification survive. They care even less about the memory model.
A proof that a sort algorithm preserves length is valid regardless of GC.

### Module system

Survives unchanged.

### Structured concurrency

`scope s { s.spawn(...) }` survives. The GC threads through concurrency correctly
(it already has to handle multi-threaded programs). The scope-based model is still the
right design for structured task lifetime.

### `Mutex[T]`, `RwLock[T]`

Survive. The GC handles *memory* safety. You still need synchronization for *mutation*.
These are not the same problem. A GC'd language without `Mutex` would have data races.

### `Box[T]`

Survives, but its meaning changes. `Box[T]` in Ferrum was "single owner, RAII-managed,
heap allocated." In Ferrite, `Box[T]` could become "allocated on the GC heap" — basically
a synonym for heap allocation. Or it could go away and allocation just happens transparently.

Probably: `Box[T]` survives for recursive types and trait objects (`Box[dyn Trait]`) where
you need explicit heap indirection, but its RAII semantics go away. The GC collects it.

### `dyn Trait` (trait objects)

Survive. Dynamic dispatch is orthogonal to the memory model.

---

## New in Ferrite

### State-of-the-art GC

The GC design assumptions (2026-era state of the art):

**Precise tracing.** The compiler generates GC maps for every stack frame and every heap
type — a description of exactly which slots contain GC references. The GC never has to
guess. No conservative scanning. This is what enables compaction: if you know every pointer
to an object, you can move the object and update all of them.

**Generational.** Young generation uses per-thread bump-pointer allocation. The fast path
for allocating a short-lived object is a pointer increment with no synchronization and no
lock. Most objects are short-lived (temporaries, intermediate values). The young gen is
collected frequently at low cost. The old gen is collected less frequently.

**Concurrent marking.** GC marking runs on background threads while the application runs.
Write barriers track pointer mutations during marking (if you store a pointer to a new object
into an old-gen object, the barrier ensures the GC knows about it). The STW (stop-the-world)
phase is reduced to final remark and a brief evacuation phase.

**Compacting.** The GC moves objects to reduce fragmentation. Because tracing is precise,
all pointers are known and can be updated. This requires the STW pause for evacuation, but
concurrent evacuation (similar to ZGC/Shenandoah) can reduce this further.

**Sub-millisecond pause targets.** With concurrent marking and incremental/concurrent
compaction, most pauses are under 1ms. Not hard-realtime. Fine for applications, servers,
games that can absorb occasional jitter.

**Per-thread young gen.** Each thread has its own young gen region. No synchronization
needed for the allocation fast path. The young gen size is configurable. When full, the thread
does a minor GC (collecting its own young gen), or the GC thread does it.

Inspiration sources: MMTk, LXR (deferred reference counting + Immix), Shenandoah, ZGC,
JVM Epsilon (no-op GC for benchmarking), Go's GC (concurrent, but not compacting).

The fact that Ferrite has a type-safe language makes the GC simpler to implement than in
a language with unsafe memory access — the GC can rely on the type system for its maps.

### `resource` types — linear typing for OS resources

GC is great for data. It's wrong for resources.

A file descriptor, network socket, database connection, GPU buffer, or mutex needs to be
closed/released at a specific time. The GC might never collect these, or collect them in
the wrong order. Java's `finalize()` was a disaster for exactly this reason.

Ferrite's answer: `resource` types are **linearly typed** — they must be explicitly consumed
(closed), not left for the GC.

```ferrite
resource struct File {
    fd: i32,
}

impl File {
    fn close(self) ! IO    // takes ownership, consumes the File
    fn read(&mut self, buf: &mut [u8]): Result[usize] ! IO
    fn write(&mut self, data: &[u8]): Result[usize] ! IO
}
```

The `resource` keyword marks a type as linear. The compiler enforces:
- A `resource` value must be consumed (closed) before it goes out of scope
- You cannot ignore or silently drop a `resource` value
- The GC never manages `resource` values — they live on the stack or in `use` blocks

The `use` block provides automatic close at scope exit:

```ferrite
// Explicit use block — file is closed at }
use fs.open("data.csv") as f {
    let line = f.read_line()?
    process(line)
}   // f.close() called implicitly here — compile-time guaranteed

// Without `use`: compile error
let f = fs.open("data.csv")   // ERROR: `File` is a `resource` type
                               //        must be consumed — use `use` block or call .close()
```

`use` is syntactic sugar for "create this resource, bind it, and call .close() at every
exit point of the block, including early returns and panics." It's RAII without GC.

Resources are NOT finalizers. There's no "GC will eventually call close." Close happens at
lexical scope exit, deterministically, in source order. This is the right behavior.

Multiple resources:
```ferrite
use fs.open("input.txt") as input,
    fs.create("output.txt") as output
{
    pipe(input, output)?
}   // output closed first, then input — reverse declaration order, like RAII
```

Resources and GC data can coexist freely:
```ferrite
use fs.open("data.json") as f {
    let content = f.read_all()?     // content: String — GC managed
    let parsed = json.parse(&content)?   // parsed: Value — GC managed
    process(parsed)
    // f closed here, content and parsed GC'd when no longer reachable
}
```

### `@no_gc_pause` — suppress STW pauses in hotspots

The compiler inserts safepoint polls at function call sites and loop back-edges. When the GC
needs to stop the world, it sets a flag; threads pause at the next safepoint poll.

`@no_gc_pause` tells the compiler: omit safepoint polls in this function (and propagate
that suppression to its callees for the duration of those calls). Any pending STW pause
is deferred until the annotated function returns.

```ferrite
@no_gc_pause
fn render_frame(scene: &Scene, dt: f32) ! IO {
    physics_step(scene, dt)
    rasterize(scene)
    composite_layers(scene)
}
```

**What this guarantees:** no stop-the-world interruption during this code.

**What this does NOT guarantee:** concurrent GC marking still runs on its background thread.
You're suppressing the pause phase, not all GC activity.

**The constraint:** the function must complete in bounded time. A `@no_gc_pause` function
with an unbounded loop starves the GC — the runtime will eventually force a pause anyway
(starvation-prevention backstop). The compiler warns:

```
warning: `@no_gc_pause` function contains loop without `gc.yield()`
  --> game.fe:47:5
   |
47 | fn stream_decompress(data: &[u8]) {
   |    ^^^^^^^^^^^^^^^^^
   |
   = note: GC may be forced to pause here if collection is overdue
   = help: add `gc.yield()` in the loop body, or remove `@no_gc_pause`
```

**Block form** — function boundary isn't always the right granularity:

```ferrite
fn game_tick(dt: f32) ! IO {
    @no_gc_pause {
        physics_update(dt)
        render()
    }
    gc.yield()
    @no_gc_pause {
        audio_mix()
    }
    wait_for_vsync()
}
```

**Hard guarantee (no GC activity at all):** combine with `@no_alloc`:

```ferrite
@no_gc_pause @no_alloc
fn mix_audio_samples(output: &mut [f32], input: &[f32]) {
    for i in 0..output.len() {
        output[i] += input[i]
    }
}
```

`@no_gc_pause` defers the STW pause. `@no_alloc` prevents any allocation that could trigger
minor collection. Together: zero GC involvement, guaranteed. The compiler verifies `@no_alloc`
statically (no allocating calls reachable).

### `gc.yield()` — hint that now is a good time to GC

`gc.yield()` does two things:
1. **Marks an explicit safepoint** — if the GC was waiting to pause, it pauses here rather
   than at an arbitrary safepoint mid-computation.
2. **Hints that collection is opportunistically warranted** — the GC checks pressure and
   can collect even if it wasn't pending.

```ferrite
fn main_loop() ! IO {
    loop {
        let input = poll_input()
        update(input, frame_time())
        render()
        gc.yield()          // between frames: GC welcome here
        wait_for_vsync()
    }
}
```

If memory pressure is low, `gc.yield()` compiles to a single branch (check the flag, branch
not taken). Near-zero cost.

Useful patterns:

```ferrite
// After processing a large batch — lots of short-lived objects just became dead
fn process_batch(records: &[Record]) {
    for record in records {
        process_one(record)
    }
    gc.yield()   // batch done, temporaries dead — good time to collect
}

// In a server event loop
fn serve() ! Net + IO {
    loop {
        let req = accept().await?
        handle(req).await?
        gc.yield()   // request finished — collect before next one
    }
}

// In a long computation loop with no function calls
fn sum_squares(data: &[f64]): f64 {
    let mut acc = 0.0
    for (i, &x) in data.iter().enumerate() {
        acc += x * x
        if i % 10_000 == 0 {
            gc.yield()   // periodic yield in tight loop
        }
    }
    acc
}
```

`@no_gc_pause` and `gc.yield()` are inverses: yield says "pause here if needed," no_gc_pause
says "don't pause here." Together they give the programmer precise control over *where* pauses
happen, without controlling *whether* or *how much* GC runs.

---

## Design Questions Not Yet Resolved

### Does `Box[T]` survive?

Option A: `Box[T]` survives as "allocate this on the GC heap explicitly." Useful for:
- Recursive types: `struct List[T] { value: T, next: Option[Box[List[T]]] }`
- Trait objects: `Box[dyn Drawable]`
- Explicit heap allocation of large values

The `Box` doesn't own the value in the RAII sense — the GC collects it. But it gives
you explicit heap allocation syntax, which is useful for the cases above.

Option B: `Box[T]` goes away. Recursive types use GC references implicitly. Trait objects
use some other syntax (`dyn Trait`?). Allocation is fully implicit.

The recursive type case is the argument for keeping `Box[T]`. Without indirection,
`struct List[T] { value: T, next: Option[List[T]] }` has infinite size — same problem as
in Rust/Ferrum. You need heap indirection, and `Box[T]` names that indirection.

Tentative: keep `Box[T]` for the indirection use case. Its RAII semantics become "GC-managed"
instead of "freed at scope exit."

### Does the capabilities system survive in any form?

In Ferrum, `given`/`with` was used for allocators and (potentially) other ambient resources
like loggers, clocks, random number sources.

Without allocators, is the capability system still useful? Arguments for keeping it:
- Dependency injection for loggers, clocks, etc. without explicit parameters at every call
- Testability: inject a fake clock in tests, real clock in production

Arguments against:
- Complexity that may not be worth it
- Explicit parameters work fine for small numbers of ambient resources
- Go doesn't have it and manages

Tentative: drop the capability system. Use explicit parameters for ambient resources.
If this turns out to be painful, add it back as a library-level pattern or language feature.

### Does `unsafe` survive?

Ferrite is supposed to be simpler than Ferrum. `unsafe` in Ferrum exists primarily for:
- Raw pointer manipulation
- Manual allocation
- Pin invariants
- FFI

All of those go away or are reduced. If FFI gets its own mechanism (`extern` blocks), maybe
`unsafe` goes away entirely.

Arguments for keeping `unsafe`:
- Some genuinely unsafe things will always exist (intrinsics, hardware access)
- Having NO unsafe makes it impossible to write the GC runtime in Ferrite itself
- FFI always has safety implications; naming that "unsafe" is honest

Arguments against:
- Ferrite's target audience doesn't need `unsafe`
- The simplicity argument: if you need `unsafe`, use Ferrum

Tentative: keep a minimal `unsafe` for FFI only. No raw pointer arithmetic in safe code.

### What does `&T` mean in Ferrite without lifetimes?

`&T` is a reference to a GC-managed value. It's valid for a lexical scope. The GC keeps
the referenced value alive as long as any live reference (stack or heap) points to it.

The borrow checker checks: within a scope, are there conflicting borrows? It doesn't need
to track whether the borrow "escapes" because GC handles liveness.

Implication: `&T` in a function signature no longer carries implicit lifetime constraints.
`fn process(data: &[u8]): Result[String]` doesn't need a lifetime annotation because the
compiler doesn't need to know whether `data` lives longer than the return value. If the
return value is `Result[String]`, the String is GC-allocated. If it were `Result[&[u8]]`
(returning a slice of the input), the slice is valid as long as the caller holds it — and
since the caller holds a reference to the underlying array, the GC keeps it alive. It just
works.

Edge case: could there be a use-after-free with `&T` in Ferrite? Only if the GC has a bug.
The GC guarantees that any object with a live reference is not collected. A live reference
on the stack IS a GC root. So no: `&T` in Ferrite cannot dangle in correct GC operation.

This means the borrow checker's job is purely: are there two `&mut T` views, or a `&T` and
`&mut T` view, of the same object active simultaneously? That's it. Much simpler.

### How does `Send`/`Sync` work without `Arc`?

In Ferrum, `Send` (can be sent across threads) and `Sync` (can be shared across threads) are
traits that Rc doesn't implement (preventing cross-thread sharing). Since `Arc` is gone, the
whole Rc/Arc distinction disappears.

But we still need to prevent data races. Two threads can't both have `&mut T` to the same
object without synchronization.

With a concurrent GC, the GC itself is thread-safe (it already handles multi-threaded programs).
The memory is shared; the GC knows about it. But mutation still needs synchronization.

Options:
- Keep `Send`/`Sync` traits as type-level markers for which types are safe to cross thread
  boundaries. They'd be derived automatically in many cases (all GC-managed types are `Send`
  unless they contain a thread-local resource type).
- Drop them and use a simpler rule: GC-managed references can be shared across threads,
  but mutation always requires a `Mutex` or similar.

The `Mutex[T]` approach naturally handles this: if you want to mutate `T` from multiple threads,
wrap it in `Mutex`. The mutex provides both synchronization and a `&mut T` at a time.

Tentative: drop `Send`/`Sync` complexity. GC-managed values are shareable by default.
Mutation across threads requires `Mutex[T]` or `RwLock[T]`. Resource types are not shareable
across threads (they're linearly typed and stack-bound). This is simpler to explain.

### What about `Copy` types?

Scalars (integers, floats, booleans, chars) are still `Copy` — copy-on-assignment semantics.
Small aggregates of `Copy` types are also `Copy`.

This is orthogonal to GC and should survive unchanged.

### What about the proof system?

`proof fn` and formal contracts (`requires`/`ensures`) seem fully orthogonal to GC. Keep them.

But the proof system in Ferrum uses totality, SMT discharge, etc. — all of which are
independent of memory management. No changes needed.

### Does SemanticQuery survive?

SemanticQuery (the compiler-as-reasoning-engine API from Ferrum) seems fully applicable to
Ferrite. Maybe more applicable — Ferrite's type system without lifetimes might be easier for
agents to query and propose hypotheses about.

Tentative: yes, keep SemanticQuery concept. Defer details to implementation.

---

## What Ferrite Is NOT

- Not a replacement for Ferrum. Different domain. When you need zero-overhead control, use Ferrum.
- Not a GC'd version of Rust. Rust's approach is fundamentally different.
- Not Java with better syntax. Java doesn't have effects, sum types are weak, exceptions, null.
- Not Go with generics. Go has no effects, no sum types, no contracts, less principled type system.
- Not Haskell. Ferrite is strict (eager evaluation), imperative-friendly, has `! IO` not IO monad.
- Not Kotlin. Kotlin targets JVM or JS primarily; Ferrite is native. Kotlin has no effects.
- Not OCaml. OCaml has no effects (until OCaml 5's effect handlers, which are different), weaker
  contract system, different concurrency model.

---

## The Cascade of Simplicity

When you commit to "GC owns memory," the following complexity evaporates:

| Was complex in Ferrum | Gone in Ferrite | Why |
|----------------------|-----------------|-----|
| `pinned` types | Gone | GC can move self-referential objects precisely |
| `Pin<Box<dyn Future>>` | Gone | Async state machines are plain GC objects |
| `UnsafePinned<T>` | Gone | Pinning is not a concept |
| `noalias` suppression | Gone | No `pinned` means no special aliasing rules |
| Explicit lifetime annotations | Gone | GC handles liveness |
| `'a` syntax everywhere | Gone | No lifetimes to name |
| `where 'b: 'a` outlives bounds | Gone | GC makes outlives trivially true |
| Region inference algorithm | Simplified | Just aliasing, no liveness |
| `Rc[T]` / `Arc[T]` / `Weak[T]` | Gone | GC handles cycles and sharing |
| `RefCell[T]` (mostly) | Gone | Simpler borrow checker, less need |
| `given [A: Allocator]` | Gone | No allocator choices |
| `with Arena.new() as alloc` | Gone | No arenas |
| `! Alloc[A]` effect | Gone | GC allocation is ambient |
| `@no_alloc` | Reduced | Only needed for `@no_gc_pause` hard guarantee |
| `NoAlloc` marker trait | Gone | |
| `Layout` declarations | Gone | No manual memory layout |
| Allocator capability crossing task boundaries | Gone | |
| Most of `unsafe` | Gone or minimized | No raw pointer arithmetic in safe code |

---

## What the Language Footprint Looks Like

Ferrum has 45 keywords. Ferrite would have ~30. Rough removals:
- `region` — gone
- `pinned` — gone
- `layout` — gone
- `trusted` — probably gone (no unsafe pointer arithmetic to trust)
- `unchecked` — reduced scope

Possible additions:
- `resource` — for linear resource types
- `use` — for resource blocks (might repurpose a keyword)

The type system gets simpler:
- No smart pointer zoo (Box survives, Rc/Arc/Weak gone)
- No allocator traits
- No region variables
- `&T` and `&mut T` without lifetime annotations

The effect set gets smaller:
- Gone: `! Alloc[A]`
- Kept: `! IO`, `! Net`, `! Sync`, `! Async`, `! Panic`, `! Global`

---

## Market Positioning

### Who Ferrite is for

Programmers who want to write correct, performant application code without fighting a type
system designed for OS kernel authors.

The target: web backends, CLI tools, data pipelines, compilers, language servers, game scripts,
desktop applications, network services. Anything that lives above the OS abstraction layer.

### Why not Go

Go made the right call on GC, goroutines, and simplicity. It made the wrong call on:
- No sum types (error handling is stringly typed)
- Weak generics (until Go 1.18, then still limited)
- No effect tracking (any function can do IO, no annotation)
- No design-by-contract
- Interface types allow nil (the billion-dollar mistake, again)
- No `Option[T]` — nil is the option

Ferrite fixes all of these while keeping Go's "just write code and it works" feel.

### Why not Kotlin/Swift

Both are good languages with GC (or ARC, which is close). Both have sum types and null safety.
Neither has effect tracking. Neither compiles to native without JVM/ObjC overhead. Neither has
design-by-contract at the language level.

### Why not OCaml

OCaml has sum types, a great type system, and good GC. It lacks: effect tracking (OCaml 5 adds
effect handlers, but they're different — they're a control flow mechanism, not a constraint
system), imperative-friendly syntax, good story for systems-level performance.

### The one-liner

**Ferrite is what Go would have been if it had been designed by people who cared about types
and effects.**

---

## Interoperability with Ferrum

Both compile through LLVM. The question is whether they can call each other.

Calling Ferrum from Ferrite: Ferrum functions that are pure or have explicit effects could be
called from Ferrite. The GC runtime needs to treat Ferrum stack frames as GC roots (the GC
needs to know about any Ferrite references stored on Ferrum stack frames). This requires
cooperation: Ferrum code compiled for Ferrite interop would emit GC stack maps.

Calling Ferrite from Ferrum: Ferrite functions are GC-managed. Calling them from Ferrum code
means the Ferrum side must either be a GC root itself, or the called Ferrite function returns
only non-GC values. A foreign-function boundary annotation would handle this.

This is similar to calling JVM code from Rust via JNI, but hopefully less painful because
both languages share a type system lineage and could share a GC root registration protocol.

Realistic use case: write performance-critical inner loops in Ferrum (no GC, no overhead),
wrap them in Ferrite for the application layer. The inner loops take `&[T]` slices, do math,
return scalar values — no GC crossing needed. This is common and clean.

---

## Analogy Map

| | Ferrum | Ferrite |
|--|--------|---------|
| Memory | Borrow checker + RAII | Precise concurrent GC |
| Allocation | Explicit, allocator-parametric | Implicit, GC-managed |
| Cycles | Manual `Weak[T]` | GC handles automatically |
| Resources | Any `Drop` type | `resource` linear types |
| Lifetimes | Inferred or annotated | Absent (GC handles liveness) |
| Borrow checker | Liveness + aliasing | Aliasing only |
| Effects | `IO Net Sync Async Panic Global Alloc` | `IO Net Sync Async Panic Global` |
| Async internals | Pinned state machines | GC-managed state machines |
| `unsafe` | Substantial surface | Minimal (FFI only) |
| Target domain | OS, embedded, engines | Applications, backends, tools |
| Load-bearing principle | No hidden costs | No surprising behavior |

---

## Things to Think About Later

- How does the GC interact with `! Sync`? Acquiring a mutex is a sync effect. The GC's
  own internal locking — does that count? Probably not visible to the user.

- Write barriers for the concurrent GC — do these show up anywhere in the type system or
  are they fully hidden? Probably fully hidden, but noting the question.

- What happens if a `@no_gc_pause` function calls into a Ferrum function that triggers
  allocation? Ferrum allocation is outside the GC, so it's fine — the GC doesn't see it.
  But what if the Ferrum function calls back into Ferrite and causes GC pressure? Edge case
  worth thinking about.

- Thread-local GC vs. global GC. Go uses per-P (processor) heap arenas with a global GC.
  JVM has per-thread TLAB (thread-local allocation buffers) in the young gen. The right
  design for Ferrite is probably per-thread young gen + shared old gen, with concurrent
  collection threads.

- Finalizers: explicitly excluded. But what about close methods that need to run at GC time
  for objects that outlive their `use` block? Answer: that's a design error. If something
  needs deterministic close, it's a `resource` type and must be in a `use` block. If it's
  not a `resource` type, it has no close semantics and the GC just frees its memory.

- Large objects: a generational GC typically puts large objects directly in old gen.
  Objects above some size threshold (e.g., 256KB) skip the young gen. This is standard.

- Weak references (GC-level): distinct from Ferrum's `Weak[T]`. A GC-level weak reference
  lets you hold a reference that doesn't prevent collection — useful for caches. Could be
  a library type `GcWeak[T]` with `upgrade() -> Option[T]` semantics. Different from
  Ferrum's `Weak[T]` (which was about breaking Rc cycles).

- The SemanticQuery system becomes more interesting in Ferrite: without lifetime annotations,
  agents querying the compiler have fewer concepts to reason about. A query about "what
  effects does this function have" is a cleaner question when you don't also have to reason
  about which lifetime parameters the function is polymorphic over.

- Progressive disclosure: Ferrite should have the same layer-0/layer-1/layer-2 structure as
  Ferrum. Layer 0: just write code, GC handles memory, effects inferred. Layer 1: annotate
  effects at pub boundaries. Layer 2: contracts. Layer 3: proofs. Layer 4: `@no_gc_pause`
  and `gc.yield()` for latency-sensitive code.

- Can Ferrite programs be fully deterministic for testing? With a GC, collection timing is
  non-deterministic. This affects programs that use finalizers — but Ferrite bans finalizers.
  If your program has no finalizers, GC timing doesn't affect output. So Ferrite programs
  ARE deterministic in behavior, just not in memory pressure. This is important for testing.

- Compile-time evaluation: `const fn` or equivalent. Since Ferrite is simpler (no lifetimes,
  no allocators), compile-time evaluation might be easier to define. Compile-time evaluated
  code can't touch the GC (no runtime). This is a natural constraint.

- Format/display traits: same as Ferrum, orthogonal to GC.

- The `Copy` trait: survives. Scalars are `Copy`. GC references are NOT `Copy` in the Rust
  sense — but actually, in a GC'd language, copying a GC reference is just copying a pointer.
  Maybe all GC references are implicitly `Copy` (copying a reference = copying the pointer,
  the GC tracks both). This is how Java/Python work: references are value types, you copy
  them freely, the GC tracks the referents. This might simplify the type system significantly.
  If GC references are implicitly `Copy`, the move/copy distinction only applies to `resource`
  types (which are linear and cannot be copied).

---

*End of thought dump. Come back here when ready to design Ferrite seriously.*
