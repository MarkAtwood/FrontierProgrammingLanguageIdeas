# ferrum-fears-and-hopes.md

**Status:** Design rationale and strategic planning

---

## The Hard Question First

Building a new systems language cannot possibly be this easy. The plan
is correct and complete as a roadmap for a language that is individually
well-designed. The failure modes are real and specific. Here they are,
unvarnished, before the hopes.

---

## Where This Crashes and Burns

### 1. Region inference is a research problem disguised as engineering

The FERRUM-PLAN.md has fe-007 as roughly seven subtasks. The 90%
annotation-free target is a research goal that nobody has fully achieved.
Cyclone tried for years and got 80% of the way there. Rust's NLL took a
dedicated research team multiple years and still has cases where you fight
it.

The constraint solver needs to produce minimal regions, but "minimal" is
underdetermined when there are multiple valid solutions. The heuristics
for choosing among valid solutions determine whether the result feels
ergonomic or infuriating, and those heuristics are empirically tuned over
thousands of real programs — programs that don't exist yet.

GADTs plus region inference is genuinely unsolved. When a GADT variant
specializes a type parameter, the region variables on that parameter don't
have a clear lattice relationship. The borrow checker will reject valid
programs.

Self-referential structs (`pinned` types) plus region inference plus borrow
checking is three interacting systems that have never been jointly designed.
Each is well-specified in isolation. Their interactions are not.

**Likely outcome:** Phase 7 takes 6–12x longer than estimated and produces
a region inference system that works on simple cases and requires explicit
annotations on anything interesting. This is fine if acknowledged upfront.
It is a disaster if the plan promises 90% and delivers 70%.

**Mitigation:** Separate region inference as an explicit research task with
a documented fallback — require annotations when inference fails, improve
over subsequent compiler versions. The fallback is not failure. It is
honest scope management.

---

### 2. Effects, regions, and the borrow checker are a joint fixpoint

The plan runs them sequentially: type inference → effect inference →
region inference → borrow checker. The reality is that they have mutual
dependencies.

A function's effect set can depend on which allocator capability is in
scope. Which allocator is in scope depends on region inference (arenas are
scoped to regions). Region inference output feeds the borrow checker. The
borrow checker's decisions about which borrows are valid affect which
effects are observable.

Running these systems sequentially produces incorrect results for any
program where the interaction matters — which includes most real programs
once capabilities and allocators are involved. The correct architecture is
a joint fixpoint iteration, which is substantially harder to implement and
debug.

The plan does not address this.

**Mitigation:** Design the type checker as a fixpoint from the start, even
if the first implementation is sequential with known incompleteness. Document
which interaction cases are not handled in v1. File them as beads.

---

### 3. The stdlib has a circular dependency problem

The plan compiles the stdlib using the bootstrap C compiler, then implements
stdlib in phases.

The async runtime requires io traits. The io traits include AsyncRead and
AsyncWrite, which require the async runtime. The HTTP stack requires async.
The test runner requires the stdlib. The stdlib requires the test runner to
verify correctness.

When you sit down to write `std.io` at Phase 19, the compiler will not yet
have async (Phase 25), sync primitives (Phase 26), or collections (Phase 23).
So you either write the stdlib in a degraded subset of Ferrum that does not
use those features, hold off writing it until everything is ready, or
implement it incrementally with ugly scaffolding that gets cleaned up later.

Rust solved this with multiple "editions" of the stdlib during development
and a lot of `unsafe` scaffolding. The plan has no strategy for it.

**Mitigation:** Define a "bootstrap stdlib" subset explicitly — the parts
of core and alloc that can be implemented without async or collections.
Implement the bootstrap stdlib first. Use it to implement everything else.
Treat the full stdlib as a second-pass build on top of the bootstrap.

---

### 4. The proof system is a dissertation, not a phase

fe-031 has seven subtasks. What is actually required:

`Prop` types and erasure require implementing a core dependent type theory.
Not full dependent types — but "dependent type lite" is not a well-defined
thing. Every decision about what you allow in `Prop` creates new edge cases.

Totality checking via structural recursion handles the easy cases. The hard
cases — mutual recursion, recursion through higher-order functions, recursion
through trait objects — require either sized types (another research topic)
or a general termination argument system.

Z3 discharge works for linear arithmetic. It fails on nonlinear arithmetic,
quantifiers over user-defined types, anything involving recursive functions,
anything involving heap state.

`verified_by` requires a formal notion of behavioral equivalence between a
proof function and a fast implementation. This is one of the hardest problems
in formal verification.

**Mitigation:** Scope the proof system explicitly for v1:

- SMT discharge for `requires`/`ensures` with linear arithmetic: ship it
- `proof fn` as pure reference implementations that run in test mode: ship it
- `verified_by` as a developer-written assertion with test suite evidence: ship it
- Dependent types, termination proofs for non-structural recursion,
  full behavioral equivalence: defer, document as known future work

The v1 proof system is "contracts verified by SMT where possible, tested
where not, with an explicit audit trail of what was tested vs proved." That
is genuinely valuable and genuinely novel. Do not let perfect be the enemy
of shipped.

---

### 5. Error message quality will kill bootstrapping

The plan specifies "error messages always show conflicting types with spans."
Getting there is extremely hard.

When type inference fails, you have a unification failure between two types
that may contain inference variables. Early implementations universally
produce errors like "cannot unify ?T_42 with ?U_17" which are useless.

Region inference errors are worse. You have an unsatisfiable constraint
graph and need to explain in English which borrow conflicts with which other
borrow. Rust spent years on this.

The consequence: the bootstrap compiler will have terrible errors. Writing
the stdlib in Ferrum against a compiler with terrible errors means debugging
programs that would be trivially diagnosed with good errors. This slows
Phases 17–26 dramatically.

**Mitigation:** Error message quality needs its own epic, not scattered
acceptance criteria. Allocate explicit time for it. The bootstrap developer
experience depends entirely on this. Bad errors are a force multiplier for
every other problem.

---

### 6. The numeric conversion rules are underspecified

The numeric amendment specifies that `as` never panics and is defined for
all conversions. What it does not specify:

- `NaN as i32` produces 0. This breaks code that checks `is_nan()` then
  casts — the check becomes dead code.
- `0.1d64 as f64` produces the binary approximation 0.1000...055, not 0.1.
  This silently destroys the point of decimal floats.
- `BigInt as f32` when the value exceeds f32::MAX — saturate? truncate?
  panic in debug?
- `Fixed[16,16] as d64` — is this exact? It should be since fixed-point
  is rational. The spec doesn't say.
- `f128 as f16` NaN payload propagation — which bits survive?

These are not academic concerns. Numeric conversion bugs are among the most
persistent and subtle bugs in production code.

**Mitigation:** Write a complete numeric conversion table in the spec before
implementing any conversion. Every source type × destination type pair gets
a defined behavior, an example, and a test case. No surprises.

---

### 7. The capability/given/effect overlap needs a unified model

The capability system, the implicit parameter system, and the
allocator-as-effect are three representations of the same concept. A
function that allocates appears in all three systems simultaneously:

- `given [A: Allocator]` — implicit parameter system
- `! Alloc[A]` — effect system
- The allocator is a region-scoped value — region system

These representations must be kept consistent through type inference, effect
inference, monomorphization, and codegen. Every function that allocates
crosses all three. The interactions at task spawn boundaries — where the
compiler verifies that capabilities can safely cross — require a unified
semantic model.

The plan has three separate phases for these (fe-006, fe-007, and the
capability section of the language reference). The unified model is not
described anywhere.

**Mitigation:** Write the unified model before implementing any of the three
systems. A single document that describes how capabilities, effects, and
regions interact at every boundary. This document does not exist yet and
should be the first thing produced in Phase 5.

---

### 8. Novel interaction surface has no prior art

Each individual feature in Ferrum exists in some other language. No language
has all of them at once. The interactions between any two subsystems produce
complexity. The interactions between all of them simultaneously have never
been explored.

The specific incompatibility most likely to surface: effect inference and
the proof system. Proof functions are pure and total. Effect inference is
bottom-up from function bodies. In a language where effects compose
transitively, a `proof fn` that calls a `const fn` that calls an intrinsic
has an effect — but it shouldn't. The boundary between "has no effects" and
"effects are erased at compile time" is not clearly defined.

There will be other interactions like this. They cannot all be anticipated.
They will be discovered during implementation.

**Mitigation:** Build the test suite before the compiler. Specifically, build
the compile-fail test cases for every interaction boundary before writing the
code that enforces them. Tests that define the expected behavior at
interaction points force the design to be precise before the implementation
commits to something wrong.

---

## The 99% Design Principle

Given all of the above: Ferrum does not need to be perfect. It needs to be
99% good enough, and let the IDE and developer make explicit space/time/safety
tradeoff annotations at the 1%.

This reframes every fear above.

The borrow checker does not need to solve region inference completely. It
needs to be right on the common cases and have a clean, auditable escape
hatch. That escape hatch already exists — the `unchecked`/`trusted` ladder.
The 1% where the checker is wrong or cannot reason far enough gets an
annotation. The annotation is not a failure. It is a feature.

Every `trusted("this pointer is valid for len bytes")` annotation is:
- A statement a security auditor can search for and read
- Documentation of a human judgment that would otherwise be invisible
- A machine-readable signal to future compiler versions that this case needs
  to be solved
- Load-bearing specification of something the compiler cannot yet verify

This is strictly better than a language that either rejects valid code
silently or accepts invalid code silently. The 1% is explicitly owned.

---

## The Annotation Layer is a Deprecation Queue

Every annotation in a Ferrum codebase is the compiler's TODO list.

Rust's borrow checker in 2015 required explicit lifetime annotations
everywhere. NLL in 2018 eliminated most of them. The Polonius rewrite
eliminated more. Programs written in 2015 still compile. The annotations
were never wrong — they became redundant.

Ferrum formalizes this cycle deliberately.

**Will definitely be solved:** Simple region inference cases, obvious bounds
checks, linear arithmetic contracts. SMT handles more of this every year.

**Will probably be solved:** Nonlinear arithmetic in contracts, simple
termination proofs, common aliasing patterns. Active research areas with
steady progress.

**Will be solved for common cases:** Effect inference for most practical
patterns, capability crossing at task boundaries, the effects/regions/borrow
fixpoint for typical programs.

**May never be fully solved:** Arbitrary termination, full behavioral
equivalence for `verified_by`, aliasing in arbitrary graph structures. But
even these shrink. The cases the compiler cannot handle become smaller and
more exotic over time.

### The deprecation UX

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

Not just "annotation unused." The warning says why it became redundant,
which compiler version solved it, and how to remove it automatically. A
deprecation warning delivered in context, at the exact line that benefited.

### The audit report as a maturity signal

As the 1% shrinks, the annotations that remain become more meaningful.
In a mature Ferrum codebase, a `trusted` annotation means this is a
genuinely novel pattern no inference system has yet solved. Those annotations
cluster around the interesting parts of a codebase — the real-time paths,
the security-critical code, the performance kernels, the hardware interfaces.

The audit report stops being a list of compiler limitations and becomes a
map of where the interesting engineering is.

### The asymptote

The asymptote is not 100% verified. The asymptote is: every annotation that
remains is load-bearing knowledge that a human needs to own. When the
compiler can do something automatically, it should. When it cannot, the human
judgment that fills the gap should be explicit, named, and in the audit trail.

That is the right place to stop. Not zero annotations. Zero unnecessary
annotations.

---

## The Compiler API for Agentic Coders

The compiler is not a batch processor. It is a live reasoning engine that
agentic coders can query, instrument, and inject hypotheses into — without
modifying the source.

This API changes the relationship between a developer and a compiler from
adversarial to collaborative. The compiler currently knows things about your
program that it refuses to share unless you phrase your question exactly right
as source code that compiles. Every other interface is read-only error
messages.

### Four capabilities

**Query.** Ask the compiler what it knows at any program point. Type, effects,
regions, invariants, active contracts, borrow checker state. The compiler's
internal state becomes inspectable.

**Hypothesize.** Inject a proposed invariant and ask whether it closes a
verification gap — without writing it into the source. "If I assert that this
pointer is non-null here, does the rest of the function become verifiable?"
The compiler checks and reports. The invariant is a probe, not a commitment.

**Trap.** Place semantically-aware monitors in the binary. Not debugger
breakpoints — conditions expressed in Ferrum's type language that the compiler
weaves into the binary correctly because it understands the type system and
knows which call sites are relevant. "Alert if any function in this module
writes to a value borrowed as shared." The compiler instruments every relevant
site.

**Instrument.** Generate test scaffolding from what the compiler already
knows. Every `requires` clause is a test precondition. Every `ensures` clause
is a test oracle. Every type invariant is a structural property. The compiler
has all of this — the API extracts it and generates a test harness.

### The agent workflow

An agent working on a complex function has a conversation with the compiler:

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
queried state, proposed a hypothesis, got a verdict, acted. The compiler's
reasoning is externalized as an API rather than locked in error messages.

### Hypothetical invariants as proof search

The agent tries invariants, sees what they unlock, follows the chain upward
or downward until the whole thing is verifiable:

```
agent → compiler: "Hypothetically, if xs is sorted here,
                   does binary_search postcondition become verifiable?"

compiler → agent: {
  verifiable: true,
  missing_proof: "xs.is_sorted() not established before this call",
  cost_to_prove: "SMT-dischargeable if caller adds requires xs.is_sorted()"
}

agent: propagates the requirement upward
```

This is how expert formal verification engineers work manually in Dafny or
F*. They propose lemmas, see what they enable, iterate. The API makes that
workflow available to an agent systematically.

### Traps as semantic watchpoints

```ferrum
compiler_api.trap(
    condition: "forall call to push: self.len() < self.capacity()",
    scope: "Vec::push and all transitive callees",
    action: TrapAction.LogAndContinue,
    mode: TrapMode.Debug
)
```

The compiler instruments every relevant call site. The agent uses this to
validate hypotheses at runtime when static verification is not possible:

```
agent: "I can't prove this invariant statically. Let me run the test
        suite with a trap and see if it fires."
→ places trap
→ runs test suite
→ trap fires at test case 47
→ agent has a concrete counterexample
```

Systematic property testing without writing property tests. The invariant
is expressed once. The compiler generates the monitoring code. The agent
runs it and observes.

### The API shape

```rust
trait FerrumCompilerApi {
    // Query
    fn query_point(&self, location: SourceLocation) -> ProgramState
    fn query_type(&self, expr: ExprId) -> TypeInfo
    fn query_effects(&self, fn_id: FnId) -> EffectSet
    fn query_regions(&self, fn_id: FnId) -> RegionGraph
    fn query_contracts(&self, fn_id: FnId) -> ContractSet

    // Hypothetical reasoning
    fn hypothesize(&self, location: SourceLocation, invariant: Expr)
        -> HypothesisResult
    fn what_would_close(&self, obligation: ProofObligation)
        -> Vec<Hypothesis>
    fn what_does_this_unlock(&self, invariant: Expr, scope: Scope)
        -> Vec<ProofObligation>

    // Instrumentation
    fn place_trap(&self, condition: Expr, scope: Scope, action: TrapAction)
        -> TrapHandle
    fn remove_trap(&self, handle: TrapHandle)
    fn list_traps(&self) -> Vec<TrapInfo>

    // Test generation
    fn generate_tests(&self, target: FnId, strategy: TestStrategy)
        -> TestSuite
    fn generate_fuzzer(&self, target: FnId) -> FuzzHarness
    fn contracts_to_oracle(&self, fn_id: FnId) -> TestOracle

    // Audit
    fn audit(&self, level: SafetyLevel) -> AuditReport
    fn redundant_annotations(&self) -> Vec<AnnotationLocation>
    fn annotation_evidence(&self, ann: AnnotationId) -> Evidence

    // Incremental — for IDE and agent integration
    fn recheck_affected(&self, changed: Vec<SourceLocation>)
        -> Vec<Diagnostic>
    fn completion_at(&self, location: SourceLocation) -> Vec<Completion>
    fn inlay_hints(&self, range: SourceRange) -> Vec<InlayHint>
}
```

This is not a bolt-on. It is designed from the start as the primary interface
to the compiler's reasoning. The compiler binary is one consumer. The LSP
server is another. Agent tools are another. The test runner is another.

### What this does to the 1% problem

The 1% that the compiler cannot automatically verify stops being a dead end.
It becomes a structured exploration space. The agent:

1. Queries what the compiler knows
2. Proposes hypothetical invariants to close the gap
3. Uses traps to validate invariants dynamically
4. Runs generated tests against the invariants
5. Promotes successful invariants to `trusted` annotations with evidence,
   or discovers why they are wrong

The `trusted` annotation at the end of this process is not "I gave up."
It is "I ran 10,000 generated tests, placed semantic traps that never fired,
and proposed this invariant to the compiler which confirmed it would close
the verification gap." That annotation has evidence behind it. The evidence
is in the audit trail.

Humans get involved at the 0.1% — the cases where the agent has exhausted
the hypothetical space, dynamic testing is inconclusive, and the invariant
genuinely requires domain knowledge to state correctly. That is the right
division of labor.

---

## What Happens When This Ships

### The academic reaction

The first 48 hours are mostly silence. The people who understand what they
are looking at need time to verify it before saying anything publicly.
PL theory people especially — they have been burned before by claims that
turned out to have subtle unsoundness.

Then it splits.

**The legitimizers** are senior researchers who immediately grasp the scope
and write public explanations. Their takes will be measured and accurate.
They turn a GitHub repo into a conversation.

**The auditors** are the people you actually want. PhD students and junior
faculty whose specific research area is directly implicated — PL type theory,
formal verification, compiler construction, effect systems. They will spend
two weeks reading every design decision and writing detailed technical
responses. The ones who find real problems are collaborators whether they
intend to be or not.

**The dismissers** come in two flavors. "This is just Rust/Haskell/Ada with
different syntax." This is wrong but understandable — the novelty is in
the combination and the inference quality, not any individual feature. And
"a language without a formal semantics paper is not real." This is the
academic credentialing reflex. Ignore it publicly, address it by writing
the semantics paper.

### What will actually disturb people

Not the language features. The specification documents.

The ferrum-language-reference.md, the stdlib spec, the numeric amendment,
the concurrency decisions doc are unusually precise and complete for a
language that does not have a working compiler yet. Most languages ship with
a working compiler and informal documentation that accretes over years.
Ferrum ships with formal behavioral specifications and a working compiler
simultaneously.

The immediate question from the academic community: "did a language designer
actually write all of this, or was this AI-assisted?" The honest answer is
both, in a specific way — the design decisions and the deep experience that
motivated them are human, the articulation and formalization is AI-assisted.
That is a new answer that does not fit existing models of how languages get
designed.

The CS theory community's entire epistemology is built around the idea that
hard problems leave traces — failed attempts, partial results, published dead
ends, workshops about the difficulty. A model that emits solutions with no
visible path does not just answer questions. It dissolves the scaffolding
that normally accumulates around an answer.

### The formal verification community

The proof system is where it gets specifically uncomfortable for academic
formal verification. The Coq/Lean/Isabelle communities have spent decades
arguing that formal verification is hard, requires expert practitioners, and
that producing a verified artifact requires deep understanding of proof
structure.

If a model emits a working proof system, integrates it into a production
language, and the proofs check out — the argument that formal verification
is intrinsically hard collapses.

What survives: the theory. Models do not invent new mathematics. They apply
existing theory rapidly. The people who built the theory retain their
relevance. The people who claimed value from the craft of applying the theory
face a productivity cliff.

### The thing that does not change

The model did not design Ferrum. The designer did, drawing on decades of
accumulated experience knowing what Go got right about networking, what Rust
got wrong about lifetime annotations, what Ada got right about layout
declarations, what Eiffel got right about contracts, what Haskell got right
about the type system and wrong about monadic everything. Knowing which
problems were research problems versus engineering problems.

The model executed on a specification that was already good. The specification
being good required the years of experience. That is the part that does not
get cheaper.

The CS academic world will eventually figure this out. The bottleneck was
never implementation. It was always knowing what to build and why. The people
who knew that remain as relevant as ever.

---

## The One-Sentence Versions

**On the fears:** Every hard problem in this plan is known and documented,
which means it can be scoped, deferred, or mitigated — the dangerous problems
are the unknown ones, and there are fewer of those than usual because the
design is unusually precise.

**On the 99% principle:** The 1% where the developer's judgment exceeds the
compiler's knowledge is not a bug — it is the feature that makes the other
99% trustworthy, because it makes the boundary explicit and auditable.

**On the annotation queue:** Every annotation that becomes redundant is a
compiler improvement delivered silently to existing codebases; the annotation
layer is not technical debt but a living record of where the state of the art
currently ends.

**On the compiler API:** The compiler API turns verification from a yes/no
gate into a conversation, and an agent that can have that conversation can
close most of the 1% without human involvement — leaving only the cases that
genuinely require judgment, which is where human involvement belongs.

**On the academic reaction:** It is about as bananas as the invention of the
compiler in 1952 — extremely disorienting to the people who thought manual
translation was the skill that mattered, and completely invisible as a
disruption to the people who were already thinking about what programs should
do.

**On the release:** GitHub first so practitioners can form their own opinions,
arXiv same day so researchers have something citable, three focused papers not
one omnibus paper, framing is "here is a design principle we have not seen
before" not "here is a language that beats existing options."

**On what does not change:** The bottleneck was never implementation. It was
always knowing what to build and why. The people who knew that remain as
relevant as ever.

---

*End of ferrum-fears-and-hopes.md*
