# Ferrum — Goals, Hopes, and Fears

**Status:** Design rationale and strategic planning

---

## What Ferrum Is For

Ferrum exists to answer a question that has haunted systems programming for
decades: **can we have both safety and control without making the programmer
pay for it in annotations, fighting the compiler, or learning a language too
large to hold in one head?**

The answer is yes, if the language is designed around three principles:

1. **The compiler does the work.** Region inference, effect inference, borrow
   checking, contract verification — these are compiler problems, not
   programmer problems. The programmer writes code. The compiler figures out
   why it is safe.

2. **The 1% is explicit.** When the compiler cannot figure something out, the
   programmer's judgment fills the gap with an annotation that is visible,
   auditable, and becomes redundant as the compiler improves.

3. **Features are encountered, not learned.** The language has layers. You
   can be productive in the inner layers without knowing the outer layers
   exist. You move outward when your problem requires it, not when the
   language demands it.

---

## The Goals

### A Language You Can Hold in Your Head

C++ veterans have a specific trauma: the language grew features for thirty
years, and now nobody can hold the whole thing in their head. Every codebase
uses a different subset. Every team has a style guide that bans half the
language.

Rust veterans have a different trauma: the language is smaller than C++, but
the learning curve is front-loaded. You fight the borrow checker for weeks
before you can write useful code.

Both groups look at a new systems language and ask: **will this happen again?**

Ferrum's answer is progressive disclosure by design.

**Layer 0: The working set**

```ferrum
fn process(items: &[Item]): Vec[Result] {
    let mut results = Vec.new()
    for item in items {
        results.push(transform(item))
    }
    results
}

pub fn main() ! IO {
    let data = fs.read("input.txt")?
    let output = process(&parse(&data))
    fs.write("output.txt", &serialize(&output))?
}
```

No lifetime annotations. No effect annotations on private functions. No
allocator specifications. No region variables. No capability tokens. No
contracts. No proofs.

This is not a simplified teaching subset. This is production code. The
compiler infers everything that is not written:

- `process` is pure (no `!` annotation, compiler verifies)
- `main` has `! IO` because it is `pub` and performs IO
- All allocations use `Heap` (the default)
- Lifetimes are inferred from the borrowing structure

A programmer can write useful systems code in this subset indefinitely.

**Layer 1: Effects at boundaries**

When you write a public API, you annotate its effects:

```ferrum
pub fn fetch_config(path: &str): Result[Config] ! IO
pub fn send_request(req: &Request): Result[Response] ! Net
```

This is not a new concept to learn — it is making explicit what was already
true. The compiler tells you which effects to annotate.

**Layer 2: Custom allocators**

When you need arena allocation or a memory pool:

```ferrum
fn parse_document(src: &str): Document  given [A: Allocator] {
    // All allocations use the ambient allocator
    ...
}

let arena = Arena.new(1.mb())
with arena {
    let doc = parse_document(&source)  // uses arena
}  // arena freed, all allocations gone
```

You encounter this when *you* need it — when you are optimizing memory
allocation patterns, not when you are learning the language.

**Layer 3: Contracts and invariants**

When you need to express properties the type system cannot capture:

```ferrum
fn binary_search(haystack: &[T], needle: &T): Option[usize]
    requires haystack.is_sorted()
    ensures match result {
        Some(i) => haystack[i] == *needle,
        None => !haystack.contains(needle),
    }
```

You encounter this when you are writing code that needs formal specification.
Libraries use contracts. Application code often does not.

**Layer 4: Proofs**

When you need machine-checked guarantees:

```ferrum
proof fn sort_preserves_length[T](xs: &[T], ys: &[T])
    requires ys == xs.sorted()
    ensures xs.len() == ys.len()
{ ... }
```

Most programmers never write proofs. They benefit from proofs in the standard
library and in security-critical dependencies.

**The key insight:** In C++, you cannot predict which features you will
encounter in code you read. In Rust, you must engage with lifetimes
immediately. In Ferrum, you encounter features when you need them, not when
the language forces you.

The question "can I hold this language in my head" has a specific answer:

- **The working set:** Yes. It is smaller than Go.
- **The full language:** No, and you do not need to.

---

### The 99% Principle

Ferrum does not need to be perfect. It needs to be 99% good enough, and let
the developer make explicit annotations at the 1%.

The borrow checker does not need to solve region inference completely. It
needs to be right on the common cases and have a clean, auditable escape
hatch. That escape hatch is the `unchecked`/`trusted` ladder. The 1% where
the checker cannot reason far enough gets an annotation.

Every `trusted("this pointer is valid for len bytes")` annotation is:
- A statement a security auditor can search for and read
- Documentation of a human judgment that would otherwise be invisible
- A machine-readable signal to future compiler versions
- Load-bearing specification of something the compiler cannot yet verify

This is strictly better than a language that either rejects valid code
silently or accepts invalid code silently. The 1% is explicitly owned.

---

### The Annotation Layer is a Deprecation Queue

Every annotation in a Ferrum codebase is the compiler's TODO list.

Rust's borrow checker in 2015 required explicit lifetime annotations
everywhere. NLL in 2018 eliminated most of them. The Polonius rewrite
eliminated more. Programs written in 2015 still compile. The annotations
were never wrong — they became redundant.

Ferrum formalizes this cycle deliberately.

**Will definitely be solved:** Simple region inference cases, obvious bounds
checks, linear arithmetic contracts. SMT handles more of this every year.

**Will probably be solved:** Nonlinear arithmetic in contracts, simple
termination proofs, common aliasing patterns.

**Will be solved for common cases:** Effect inference for most practical
patterns, capability crossing at task boundaries.

**May never be fully solved:** Arbitrary termination, full behavioral
equivalence, aliasing in arbitrary graph structures. But even these shrink.

When a previously-required annotation becomes redundant:

```
warning: annotation no longer needed
  --> src/parser.rs:47:5
   |
47 |     trusted("ptr valid for len bytes, aligned")
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |     region inference now verifies this automatically
   |     since ferrum 0.8
   |
   = note: `ferrum fix --remove-redundant` to clean up automatically
```

The warning says why it became redundant, which compiler version solved it,
and how to remove it automatically.

**The asymptote** is not 100% verified. The asymptote is: every annotation
that remains is load-bearing knowledge that a human needs to own. Not zero
annotations. Zero unnecessary annotations.

---

### The Compiler API for Agentic Coders

The compiler is not a batch processor. It is a live reasoning engine that
agentic coders can query, instrument, and inject hypotheses into.

**Query.** Ask the compiler what it knows at any program point. Type, effects,
regions, invariants, active contracts, borrow checker state.

**Hypothesize.** Inject a proposed invariant and ask whether it closes a
verification gap — without writing it into the source. "If I assert that this
pointer is non-null here, does the rest of the function become verifiable?"

**Trap.** Place semantically-aware monitors in the binary. Conditions expressed
in Ferrum's type language that the compiler weaves into the binary correctly
because it understands the type system.

**Instrument.** Generate test scaffolding from what the compiler already knows.
Every `requires` clause is a test precondition. Every `ensures` clause is a
test oracle.

The agent workflow:

```
agent → compiler: "What do you know about state at line 47?"

compiler → agent: {
  live_borrows: [("data", Shared, region: 'outer)],
  effects_so_far: [IO, Alloc(Heap)],
  active_contracts: ["requires data.len() > 0"],
  unresolved_regions: ["return value region is ambiguous"]
}

agent → compiler: "Hypothesize: return borrows from 'outer. Resolves?"

compiler → agent: {
  resolves: true,
  suggested_annotation: "&'outer str"
}

agent: writes the annotation
```

The agent never had to understand region inference from first principles. It
queried state, proposed a hypothesis, got a verdict, acted.

The 1% that the compiler cannot automatically verify becomes a structured
exploration space:

1. Query what the compiler knows
2. Propose hypothetical invariants to close the gap
3. Use traps to validate invariants dynamically
4. Run generated tests against the invariants
5. Promote successful invariants to `trusted` annotations with evidence

The `trusted` annotation at the end of this process is not "I gave up." It is
"I ran 10,000 generated tests, placed semantic traps that never fired, and
proposed this invariant to the compiler which confirmed it would close the
verification gap."

Humans get involved at the 0.1% — the cases where the invariant genuinely
requires domain knowledge to state correctly.

---

## Where This Gets Hard

Every goal above has implementation risks. Here they are, unvarnished.

### 1. Region inference is a research problem disguised as engineering

The 90% annotation-free target is a research goal that nobody has fully
achieved. Cyclone tried for years and got 80% of the way there. Rust's NLL
took a dedicated research team multiple years.

The constraint solver needs to produce minimal regions, but "minimal" is
underdetermined when there are multiple valid solutions. The heuristics for
choosing among valid solutions determine whether the result feels ergonomic
or infuriating.

GADTs plus region inference is genuinely unsolved. Self-referential structs
plus region inference plus borrow checking is three interacting systems that
have never been jointly designed.

**Likely outcome:** Phase 7 takes 6–12x longer than estimated and produces
a region inference system that works on simple cases and requires explicit
annotations on anything interesting.

**Mitigation:** Separate region inference as an explicit research task with
a documented fallback — require annotations when inference fails, improve
over subsequent compiler versions. The fallback is not failure. It is honest
scope management.

---

### 2. Effects, regions, and the borrow checker are a joint fixpoint

The plan runs them sequentially: type inference → effect inference → region
inference → borrow checker. The reality is that they have mutual dependencies.

A function's effect set can depend on which allocator capability is in scope.
Which allocator is in scope depends on region inference. Region inference
output feeds the borrow checker. The borrow checker's decisions affect which
effects are observable.

Running these systems sequentially produces incorrect results for any program
where the interaction matters.

**Mitigation:** Design the type checker as a fixpoint from the start, even if
the first implementation is sequential with known incompleteness. Document
which interaction cases are not handled in v1.

---

### 3. The stdlib has a circular dependency problem

The async runtime requires io traits. The io traits include AsyncRead and
AsyncWrite, which require the async runtime. The HTTP stack requires async.
The test runner requires the stdlib. The stdlib requires the test runner.

When you sit down to write `std.io`, the compiler will not yet have async,
sync primitives, or collections.

**Mitigation:** Define a "bootstrap stdlib" subset explicitly — the parts of
core and alloc that can be implemented without async or collections. Implement
the bootstrap stdlib first. Use it to implement everything else.

---

### 4. The proof system is a dissertation, not a phase

`Prop` types and erasure require implementing a core dependent type theory.
Totality checking via structural recursion handles the easy cases. Z3
discharge works for linear arithmetic but fails on nonlinear arithmetic,
quantifiers over user-defined types, anything involving recursive functions.

`verified_by` requires a formal notion of behavioral equivalence between a
proof function and a fast implementation. This is one of the hardest problems
in formal verification.

**Mitigation:** Scope the proof system explicitly for v1:

- SMT discharge for `requires`/`ensures` with linear arithmetic: ship it
- `proof fn` as pure reference implementations that run in test mode: ship it
- `verified_by` as a developer-written assertion with test suite evidence: ship it
- Dependent types, termination proofs, full behavioral equivalence: defer

The v1 proof system is "contracts verified by SMT where possible, tested where
not, with an explicit audit trail." That is genuinely valuable and novel.

---

### 5. Error message quality will kill bootstrapping

When type inference fails, you have a unification failure between two types
that may contain inference variables. Early implementations universally
produce errors like "cannot unify ?T_42 with ?U_17" which are useless.

Region inference errors are worse. You have an unsatisfiable constraint graph
and need to explain in English which borrow conflicts with which other borrow.
Rust spent years on this.

**Mitigation:** Error message quality needs its own epic, not scattered
acceptance criteria. The bootstrap developer experience depends entirely on
this.

---

### 6. The numeric conversion rules are underspecified

The numeric amendment specifies that `as` never panics and is defined for all
conversions. What it does not specify:

- `NaN as i32` produces 0. This breaks code that checks `is_nan()` then casts.
- `0.1d64 as f64` produces the binary approximation, not 0.1.
- `BigInt as f32` when the value exceeds f32::MAX — saturate? truncate?
- `Fixed[16,16] as d64` — is this exact?
- `f128 as f16` NaN payload propagation — which bits survive?

**Mitigation:** Write a complete numeric conversion table in the spec before
implementing any conversion. Every source type × destination type pair gets
a defined behavior, an example, and a test case.

---

### 7. The capability/given/effect overlap needs a unified model

The capability system, the implicit parameter system, and allocator-as-effect
are three representations of the same concept:

- `given [A: Allocator]` — implicit parameter system
- `! Alloc[A]` — effect system
- The allocator is a region-scoped value — region system

These representations must be kept consistent through type inference, effect
inference, monomorphization, and codegen. The unified model is not described
anywhere.

**Mitigation:** Write the unified model before implementing any of the three
systems. A single document that describes how capabilities, effects, and
regions interact at every boundary.

---

### 8. Novel interaction surface has no prior art

Each individual feature in Ferrum exists in some other language. No language
has all of them at once. The interactions between all of them simultaneously
have never been explored.

The specific incompatibility most likely to surface: effect inference and the
proof system. Proof functions are pure and total. Effect inference is
bottom-up from function bodies. The boundary between "has no effects" and
"effects are erased at compile time" is not clearly defined.

There will be other interactions like this. They will be discovered during
implementation.

**Mitigation:** Build the test suite before the compiler. Specifically, build
the compile-fail test cases for every interaction boundary before writing the
code that enforces them.

---

## What Happens When This Ships

### The academic reaction

The first 48 hours are mostly silence. The people who understand what they
are looking at need time to verify it before saying anything publicly.

**The legitimizers** are senior researchers who immediately grasp the scope
and write public explanations.

**The auditors** are PhD students and junior faculty whose specific research
area is directly implicated. They will spend two weeks reading every design
decision and writing detailed technical responses. The ones who find real
problems are collaborators whether they intend to be or not.

**The dismissers** come in two flavors. "This is just Rust/Haskell/Ada with
different syntax." And "a language without a formal semantics paper is not
real."

### What will actually disturb people

Not the language features. The specification documents.

The ferrum-language-reference.md, the stdlib spec, the concurrency decisions
doc are unusually precise and complete for a language that does not have a
working compiler yet. Most languages ship with a working compiler and informal
documentation that accretes over years. Ferrum ships with formal behavioral
specifications and a working compiler simultaneously.

The immediate question: "did a language designer actually write all of this,
or was this AI-assisted?" The honest answer is both — the design decisions
and the deep experience that motivated them are human, the articulation and
formalization is AI-assisted. That is a new answer that does not fit existing
models of how languages get designed.

### The thing that does not change

The model did not design Ferrum. The designer did, drawing on decades of
accumulated experience knowing what Go got right about networking, what Rust
got wrong about lifetime annotations, what Ada got right about layout
declarations, what Eiffel got right about contracts, what Haskell got right
about the type system and wrong about monadic everything.

The model executed on a specification that was already good. The specification
being good required the years of experience. That is the part that does not
get cheaper.

The bottleneck was never implementation. It was always knowing what to build
and why. The people who knew that remain as relevant as ever.

---

## The One-Sentence Versions

**On the goals:** A systems language where the compiler does the work, the 1%
is explicit, and features are encountered when you need them.

**On progressive disclosure:** The working set is smaller than Go; the full
language is large but you do not need to know it all.

**On the 99% principle:** The 1% where the developer's judgment exceeds the
compiler's knowledge is not a bug — it is the feature that makes the other
99% trustworthy.

**On the annotation queue:** Every annotation that becomes redundant is a
compiler improvement delivered silently to existing codebases.

**On the compiler API:** The compiler API turns verification from a yes/no
gate into a conversation that an agent can have systematically.

**On the fears:** Every hard problem in this plan is known and documented,
which means it can be scoped, deferred, or mitigated.

**On the academic reaction:** Extremely disorienting to the people who thought
manual translation was the skill that mattered, and completely invisible to
the people who were already thinking about what programs should do.

**On what does not change:** The bottleneck was never implementation. It was
always knowing what to build and why.

---

*End of Ferrum — Goals, Hopes, and Fears*
