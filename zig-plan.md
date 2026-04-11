# ZIG-PLAN.md

**Project:** Zinc feature proposals to upstream Zig
**Bead prefix:** `zp-`
**Goal:** Land the Zinc numeric tower, binary layout, and concurrency stdlib
          additions into upstream Zig. Establish track record first.
          Bring language-level proposals only after credibility is built.
**Reference:** zinc-spec.md for full feature specifications

---

## What We Are NOT Proposing

State this clearly before anything else, because the temptation to bundle
everything will be strong and must be resisted.

**Not proposing: ambient allocators (`using`/`with`).**
Kelley has explicitly and repeatedly defended the explicit allocator
threading as correct design. His position is principled and consistent.
Arguing against it publicly will damage credibility and waste time.
If Zinc ships this as a language fork eventually, fine. It will not land
in upstream Zig.

**Not proposing: design by contract (`requires`/`ensures`/`invariant`).**
Kelley's position is that `assert` already does this and language-level
contracts add complexity for a benefit achievable in userspace. He is not
wrong. The philosophical case for contracts requires accepting that
documentation-that-runs is worth new syntax, and that is a values
disagreement, not a technical one.

**Not proposing: the `SyncGuard` linter warning.**
This depends on async semantics that are still in flux in Zig. Not the
right time. Revisit after Zig 1.0 async story stabilizes.

**Not proposing: `std.stm`.**
Community interest exists but the lack of compile-time enforcement makes
this a half-measure. Ship it as a Zinc-specific package first, demonstrate
real usage, then consider upstreaming with evidence.

Everything else in the Zinc spec is fair game in the right order.

---

## The Core Principle

Every PR must stand on its own merits without reference to Zinc. Each
proposal is framed as: "here is a gap in Zig's current capabilities,
here is the hardware/use-case evidence, here is a complete implementation."

Never say "this is part of a larger language project." Never bundle multiple
features. Never argue philosophy with Kelley in a PR thread.

One feature. One PR. Complete implementation. Good tests. Responsive to
review. That is the entire strategy.

---

## Kelley's Review Style

Understand this before opening the first PR:

He responds quickly to well-prepared proposals. He rejects incomplete ones
immediately and cleanly. He will not soften a rejection to protect feelings.
"This doesn't fit Zig's design" is a complete answer and arguing further is
counterproductive.

The right response to rejection is: "Thank you, understood" and then
either accept it, revise substantially and resubmit, or take the feature
to Zinc. Do not relitigate closed decisions.

He has strong opinions about what belongs in the language versus userspace.
The test he applies consistently: "Can this be done in a library with
existing language features?" If yes, it probably does not belong in the
language. If no, he will hear the argument.

The features most likely to land are the ones where the answer to that
test is genuinely no. Binary layout is the strongest case. Decimal floats
are a reasonable case. `bf16` is straightforward because it is a hardware
type, not a design philosophy question.

---

## The Sequence

```
Phase 1: Pure stdlib (no compiler changes)
  zp-001  std.meta numeric interfaces        ← one week, no risk
  zp-002  std.math.fixed (Fixed, UFixed)     ← two weeks, no risk
  zp-003  std.atomic.Value improvements      ← one week, no risk
  zp-004  std.mem.TaggedPtr                  ← one week, no risk

Phase 2: Compiler additions (new types)
  zp-005  bf16 type                          ← one month, precedented
  zp-006  std.math.big.Decimal and Rational  ← six weeks, stdlib only

Phase 3: Larger compiler addition
  zp-007  d32/d64/d128 decimal floats        ← two months, DFP library dep

Phase 4: Language-level proposal
  zp-008  Binary layout declarations         ← full proposal + prototype
```

Phases 1 and 2 run roughly in parallel where possible. Phase 3 waits until
at least two Phase 1 PRs have merged. Phase 4 waits until Phase 2 is done
and Phase 3 is at minimum in review.

The rationale for this ordering: by the time the binary layout proposal
lands, you have a documented track record of shipping complete, tested,
well-reviewed features. You are not a stranger with an ambitious proposal.
You are a contributor who has already improved Zig in several concrete ways.

---

## Beads Task Graph

```yaml
zp-001:
  title: "std.meta numeric comptime interfaces"
  priority: 0
  type: task
  blocks: [zp-002, zp-005]
  description: >
    Add Integer, UnsignedInt, SignedInt, Float, DecimalFloat, Numeric
    comptime validation functions to std.meta.
    Pure stdlib. No compiler changes. ~200-400 lines.

  subtasks:
    zp-001.1:
      title: "Implement Integer, UnsignedInt, SignedInt in std.meta"
      acceptance: >
        fn foo(x: std.meta.Integer) void compiles for u8, u32, i64, usize.
        Compile error with clear message for f32, bool, struct types.

    zp-001.2:
      title: "Implement Float, DecimalFloat, Numeric in std.meta"
      acceptance: >
        Float accepts f16, f32, f64, f80, f128, comptime_float.
        Numeric accepts all integers and floats.
        All produce clear @compileError on non-conforming types.
      deps: [zp-001.1]

    zp-001.3:
      title: "Write tests for all interfaces"
      acceptance: >
        Tests cover: correct types pass, incorrect types fail with expected
        error messages. Tests run in zig test.
      deps: [zp-001.1, zp-001.2]

    zp-001.4:
      title: "Open PR to ziglang/zig"
      acceptance: >
        PR is open. Description explains the problem (ad-hoc @typeInfo checks
        in generic numeric functions), shows before/after examples.
        All CI passes.
      deps: [zp-001.3]

---

zp-002:
  title: "std.math.fixed — Fixed and UFixed point types"
  priority: 0
  type: task
  blocks: [zp-008]
  description: >
    Add Fixed(I, F) and UFixed(I, F) comptime-parameterized fixed-point
    arithmetic types to std.math.
    Pure stdlib. No compiler changes. ~1000-2000 lines.

  subtasks:
    zp-002.1:
      title: "Implement Fixed(I, F) core type with storage and conversion"
      acceptance: >
        Fixed(16, 16) stores i32. fromFloat(3.75) produces correct raw value.
        toFloat() round-trips. fromRaw/toRaw work.

    zp-002.2:
      title: "Implement arithmetic: add, sub, mul, mulWide, div, rescale"
      acceptance: >
        add/sub produce same-format result.
        mul returns Fixed(I*2, F*2) to preserve precision.
        mulWide widening multiply correct for Q1_15 DSP case.
        rescale truncates correctly.
        saturating variants clamp at type bounds.
      deps: [zp-002.1]

    zp-002.3:
      title: "Implement UFixed(I, F) unsigned variant"
      deps: [zp-002.1]

    zp-002.4:
      title: "Standard aliases: Q8_8, Q16_16, Q1_15, Q1_31, Q0_15, Q0_31"
      deps: [zp-002.1]

    zp-002.5:
      title: "Comprehensive tests including DSP FIR filter example"
      acceptance: >
        All arithmetic operations tested.
        FIR filter example from spec compiles and produces correct output.
        Edge cases: overflow, saturation, zero, max value.
      deps: [zp-002.2, zp-002.3, zp-002.4]

    zp-002.6:
      title: "Open PR to ziglang/zig"
      acceptance: >
        PR description shows the embedded/DSP use case concretely.
        Points to existing community requests for fixed-point support.
        Explains why this fits Zig's comptime model naturally.
      deps: [zp-002.5]

---

zp-003:
  title: "std.atomic.Value(T) additions: fetchMax, fetchMin, compareExchangeLoop"
  priority: 1
  type: task
  description: >
    Three missing operations on std.atomic.Value that exist in LLVM and
    are used in practice. Pure stdlib. ~200-400 lines.

  subtasks:
    zp-003.1:
      title: "Implement fetchMax and fetchMin using @atomicRmw .Max/.Min"
      acceptance: >
        fetchMax(v, .monotonic) returns previous value, stores max(current, v).
        fetchMin symmetric. Works for all integer types.

    zp-003.2:
      title: "Implement compareExchangeLoop with optional onFail callback"
      acceptance: >
        Loops until CAS succeeds. onFail called with actual value on each
        failure. Works for all types that support @cmpxchg.

    zp-003.3:
      title: "Tests for all three operations, including concurrent stress test"
      deps: [zp-003.1, zp-003.2]

    zp-003.4:
      title: "Open PR to ziglang/zig"
      deps: [zp-003.3]

---

zp-004:
  title: "std.mem.TaggedPtr(T, n_bits) — typed tagged pointer wrapper"
  priority: 1
  type: task
  description: >
    Comptime struct that enforces alignment requirement at comptime,
    provides typed accessors, and works with std.atomic.Value.
    Pure stdlib. ~200-300 lines.

  subtasks:
    zp-004.1:
      title: "Implement TaggedPtr with comptime alignment check"
      acceptance: >
        TaggedPtr(*Node, 2) where @alignOf(Node) == 4 compiles.
        TaggedPtr(*u8, 2) is a compile error with clear message.
        init/ptr/tag/setTag/raw all work correctly.

    zp-004.2:
      title: "Verify compatibility with std.atomic.Value(TaggedPtr(T, N))"
      acceptance: >
        compareExchangeWeak works on atomic TaggedPtr.

    zp-004.3:
      title: "Tests: basic usage, atomic usage, misaligned type rejection"
      deps: [zp-004.1, zp-004.2]

    zp-004.4:
      title: "Open PR to ziglang/zig"
      acceptance: >
        PR shows the before/after: current usize+manual-masking convention
        vs TaggedPtr with compile-time alignment guarantee.
      deps: [zp-004.3]

---

zp-005:
  title: "bf16 type — brain float 16"
  priority: 0
  type: task
  blocks: [zp-007]
  deps: [zp-001]
  description: >
    New compiler-supported float type. Same exponent range as f32 (8 bits),
    7 mantissa bits. Hardware-native on TPU, A100+, AMX, Apple Silicon.
    Follow the same implementation path as f16. ~500-1000 lines.

  subtasks:
    zp-005.1:
      title: "Add bf16 to the Zig type system alongside f16, f32, etc."
      acceptance: >
        var x: bf16 = 3.14; compiles.
        @TypeOf(x) == bf16.
        @sizeOf(bf16) == 2.
        @typeInfo(bf16) == .Float with bits == 16.

    zp-005.2:
      title: "Implement bf16 literal suffix in lexer"
      acceptance: >
        3.14bf16 lexes to a bf16 literal.
        Negative literals work: -1.0bf16.
        All standard float literal forms work with bf16 suffix.

    zp-005.3:
      title: "Implement f32 <-> bf16 conversion via bitshift (no library)"
      acceptance: >
        @as(f32, some_bf16) zero-extends mantissa — lossless.
        @floatCast(bf16, some_f32) right-shifts mantissa — lossy but fast.
        Both compile to 2-3 instructions on x86-64. No library call.

    zp-005.4:
      title: "LLVM codegen: hardware on supported targets, soft-float elsewhere"
      acceptance: >
        On x86-64 with AVX-512-BF16: arithmetic uses vbcstnesh2ps etc.
        On ARM with BF16 extension: uses BFCVT etc.
        On other targets: links soft-float library (same as f128 today).

    zp-005.5:
      title: "Arithmetic operations: all standard float ops work on bf16"
      acceptance: >
        +, -, *, / all work.
        @sqrt, @sin, @cos etc. work (may go through f32 on soft-float targets).
        @floatCast conversions between all float types work.

    zp-005.6:
      title: "Tests: arithmetic, conversion, comptime evaluation"
      acceptance: >
        All operations tested at runtime.
        Comptime bf16 arithmetic tested.
        Hardware vs soft-float paths produce identical results.
      deps: [zp-005.1, zp-005.2, zp-005.3, zp-005.4, zp-005.5]

    zp-005.7:
      title: "Open PR to ziglang/zig"
      acceptance: >
        PR description makes the hardware case: list of hardware with native
        bf16 support (A100, AMX, TPU, Apple ANE), explain the current
        workaround (cast through f32, losing performance), show the
        implementation follows f16's established pattern exactly.
      deps: [zp-005.6]

---

zp-006:
  title: "std.math.big.Decimal and std.math.big.Rational"
  priority: 1
  type: task
  deps: [zp-002]
  description: >
    Build on existing std.math.big.Int. Pure stdlib. ~3000-5000 lines.
    Uses ambient allocator idiom (passes allocator explicitly — Zig style).

  subtasks:
    zp-006.1:
      title: "Implement BigDecimal: BigInt * 10^scale, parse, arithmetic"
      acceptance: >
        BigDecimal.parse("3.141592653589793238462643383279", allocator) works.
        add/sub/mul produce exact results.
        sqrt(n, precision) returns n significant digits.
        toString() round-trips through parse.

    zp-006.2:
      title: "Implement Rational: BigInt/BigUint, always reduced to lowest terms"
      acceptance: >
        Rational.init(1, 3, allocator) + Rational.init(1, 3) = 2/3.
        2/3 + 1/3 = 1/1 (auto-reduces).
        Never stores non-reduced fractions.
        toFloat(f64) conversion works.

    zp-006.3:
      title: "Tests including the 1/n^2 series sum from the spec"
      deps: [zp-006.1, zp-006.2]

    zp-006.4:
      title: "Open PR to ziglang/zig"
      deps: [zp-006.3]

---

zp-007:
  title: "d32/d64/d128 — IEEE 754-2008 decimal floating point"
  priority: 1
  type: task
  deps: [zp-005]
  description: >
    New compiler types. Requires Intel DFP Math Library or equivalent as
    runtime dependency. Critical: decimal literals must never touch binary
    float conversion. ~2000-4000 compiler lines + library dependency.

  subtasks:
    zp-007.1:
      title: "Evaluate and select the DFP library dependency"
      acceptance: >
        Decision documented: Intel DFP Math Library (BSD licensed) OR
        a pure-Zig implementation for the subset needed.
        License is compatible with Zig's MIT license.
        Performance benchmarked on x86-64 and AArch64.

    zp-007.2:
      title: "Add d32, d64, d128 to the Zig type system"
      acceptance: >
        var x: d64 = 0.1; (from decimal literal) compiles.
        @sizeOf(d64) == 8.
        @typeInfo(d64) shows decimal float type info.

    zp-007.3:
      title: "Implement decimal literal lexing: store as string, never binary"
      acceptance: >
        CRITICAL: 0.1d64 stored as exact decimal 0.1, not 0.100000000000000005...
        Verify: 0.1d64 + 0.2d64 == 0.3d64 is true at runtime.
        Verify: @as(d64, 0.1) is a compile error (use decimal literal instead).
        The literal "0.1d64" must produce a different bit pattern than
        @floatCast(d64, @as(f64, 0.1)).

    zp-007.4:
      title: "Arithmetic via DFP library: +, -, *, /"
      acceptance: >
        0.1d64 + 0.2d64 == 0.3d64 passes.
        All four operations work for d32, d64, d128.
        Division by zero returns Infinity, not undefined behavior.

    zp-007.5:
      title: "Comparisons and conversions between decimal types"
      acceptance: >
        d32 -> d64 conversion is lossless (d64 is strictly wider).
        d64 -> d32 is lossy, requires explicit @floatCast.
        d64 -> f64 is lossy, requires explicit @floatCast.

    zp-007.6:
      title: "Rounding mode support: std.math.decimal.round with explicit mode"
      deps: [zp-007.4]

    zp-007.7:
      title: "Comptime decimal arithmetic"
      acceptance: >
        comptime { const x: d64 = 0.1 + 0.2; std.debug.assert(x == 0.3); }
        passes. Comptime decimal is exact.

    zp-007.8:
      title: "Tests: correctness, conversion, comptime, the classic 0.1+0.2==0.3"
      deps: [zp-007.2, zp-007.3, zp-007.4, zp-007.5, zp-007.7]

    zp-007.9:
      title: "[GATE] Open PR to ziglang/zig"
      acceptance: >
        This PR requires the most careful framing.
        Lead with the hardware case: IBM POWER has native decimal float
        instructions. Zig cannot target them.
        Show the financial correctness case: 0.1 + 0.2 == 0.3.
        Show the standard: IEEE 754-2008 decimal float is an existing
        standard, not a Zinc invention.
        Acknowledge the DFP library dependency clearly upfront.
        Do not hide the complexity.
      deps: [zp-007.8]

---

zp-008:
  title: "Binary layout declarations — language proposal"
  priority: 0
  type: epic
  deps: [zp-005, zp-002, zp-001]
  description: >
    Language-level feature. New syntax: layout block attached to a struct.
    Compiler verifies no gaps, no overlaps, total size correct.
    Generates read()/write()/view() codecs automatically.
    The strongest case for a language-level feature in this plan.
    Requires: real bug evidence, full proposal, prototype implementation.

  subtasks:
    zp-008.1:
      title: "Collect real bugs from the Zig community caused by layout errors"
      priority: 0
      acceptance: >
        At minimum 3 real bugs documented:
        - A hardware register map with a missing reserved field
        - A network protocol parser with wrong field bit positions
        - A binary file format with incorrect total size assumption
        Sources: Zig issue tracker, Discord, mailing list, real codebases.
        These bugs go in the PR description. They are the motivation.

    zp-008.2:
      title: "Write full proposal document: syntax, semantics, verification rules"
      priority: 0
      acceptance: >
        Document covers:
        - layout block syntax (full grammar)
        - All verification rules (gaps, overlaps, size check, type width check)
        - Generated codec semantics (read, write, view)
        - Byte order handling (layout-level and per-field)
        - Bit order handling
        - Mixed endian fields
        - Padding field convention (_pad, _reserved, anonymous _)
        - Multi-byte fields spanning byte boundaries
        - MMIO view() type and volatile access
        - Examples: StatusRegister, EthernetHeader, IpHeader
        Complete enough that someone could implement it from the document alone.
      deps: [zp-008.1]

    zp-008.3:
      title: "Implement prototype: layout verification pass only"
      priority: 0
      acceptance: >
        The prototype does not need to generate codecs.
        It needs to parse layout blocks, verify the rules, and emit
        correct compile errors for violations.
        The compile errors should be the best-in-class: name the exact
        unaccounted bits, name the exact overlapping ranges.
        Build against the current Zig compiler source.
      deps: [zp-008.2]

    zp-008.4:
      title: "Extend prototype: generate read() and write() codecs"
      priority: 1
      acceptance: >
        For the StatusRegister and EthernetHeader examples from the proposal:
        generated read() and write() functions compile and produce correct output.
        Byte order conversion works for big/little/native.
        Padding fields are zeroed in write().
      deps: [zp-008.3]

    zp-008.5:
      title: "Extend prototype: generate view() for MMIO"
      priority: 1
      acceptance: >
        view() over a *volatile [N]u8 produces field accessors.
        Reads are @atomicLoad-style volatile loads.
        Writes are @atomicStore-style volatile stores.
        Compiles for embedded targets (no std dependency).
      deps: [zp-008.4]

    zp-008.6:
      title: "Write test suite for the prototype"
      priority: 0
      acceptance: >
        Tests cover all verification rules (one test per rule, showing
        correct compile error for each violation).
        Tests cover codec correctness: round-trip read(write(v)) == v.
        Tests cover MMIO view on a mock register.
        Tests cover mixed endian.
      deps: [zp-008.4]

    zp-008.7:
      title: "Post RFC to ziglang/zig-showtime or equivalent before opening PR"
      priority: 0
      acceptance: >
        Community has seen the proposal and major objections are documented.
        Kelley has not immediately rejected it on first mention.
        If Kelley rejects it at RFC stage, do not open the PR —
        take the feature to Zinc instead.
      deps: [zp-008.2]

    zp-008.8:
      title: "[GATE] Open PR to ziglang/zig"
      priority: 0
      acceptance: >
        PR includes:
        1. The real bugs from zp-008.1 as motivation
        2. The full proposal document as the PR description
        3. The prototype implementation as the diff
        4. The full test suite
        5. Benchmark showing generated codec vs hand-written bit manipulation:
           identical output, identical performance
        
        Framing: "closing the gap with Ada's representation clauses."
        Not "here is a new feature" but "here is the missing piece for
        hardware interface code that Zig programmers are writing incorrectly
        today because the language gives them no better option."
      deps: [zp-008.3, zp-008.4, zp-008.5, zp-008.6, zp-008.7]
```

---

## Per-PR Preparation Checklist

Before opening any PR, complete all of these:

```
[ ] Implementation is complete and passes all existing Zig tests
[ ] New tests cover the feature fully, including edge cases
[ ] Tests include at least one negative test (compile error case)
[ ] zig fmt applied to all new code
[ ] No performance regressions (run relevant benchmarks)
[ ] Documentation written in the Zig doc comment style
[ ] PR description explains: the problem, the solution, the tradeoffs
[ ] PR description includes before/after code examples
[ ] PR description does NOT mention Zinc, this plan, or other features
[ ] PR is against the correct target branch (usually master or latest tag)
[ ] CI is green before requesting review
```

---

## Handling Review

**If Kelley asks for changes:** Do them. Do them well. Do not argue about
the direction of the change unless it breaks the feature fundamentally.
A PR that goes through three rounds of review and merges is better than a
PR that gets closed because the author was difficult.

**If Kelley closes the PR with "doesn't fit Zig's design":** Thank him,
close the thread, file the feature for Zinc. Do not reopen or argue.
Reopening closed PRs after a final decision is the fastest way to get
blocked.

**If the PR stalls in review for more than three weeks:** Ping once,
politely. If no response, wait another two weeks, ping again. If still
no response, post in the Zig Discord #dev channel asking if the PR needs
anything to move forward. Do not escalate beyond that.

**If a community member raises a strong objection:** Address it directly
and technically. If the objection is valid, revise the PR. If the objection
is philosophical (e.g., "this should be in userspace"), explain concisely
why it cannot be and let Kelley make the final call. Do not argue with
community members about whether a feature should exist.

---

## If PRs Are Rejected

Some of these will be rejected. That is fine. The table below shows the
fallback for each:

| PR | If rejected | Fallback |
|----|-------------|----------|
| zp-001 numeric interfaces | Accept, ship as Zinc stdlib convention | Low loss |
| zp-002 Fixed(I,F) | Publish as standalone `zinc-fixed` package | Low loss |
| zp-003 atomic improvements | Accept, Zinc patches on top | Low loss |
| zp-004 TaggedPtr | Publish as standalone `zinc-ptr` package | Low loss |
| zp-005 bf16 | Zinc compiler adds it independently | Medium work |
| zp-006 BigDecimal/Rational | Publish as standalone packages | Low loss |
| zp-007 decimal floats | Zinc compiler adds d64 independently | High work |
| zp-008 binary layout | This becomes the core of the Zinc fork justification | Medium loss, high gain |

A rejected binary layout PR is not a failure. It is a clean justification
for why Zinc exists as a separate language rather than a set of upstream
patches. The PR will have demonstrated that the feature is implementable,
correct, and fills a real gap. The community will have seen it. The people
who wanted it will find Zinc.

---

## Timeline

These are not commitments. They are sanity checks.

| Month | Target |
|-------|--------|
| 1 | zp-001 and zp-002 open |
| 2 | zp-003 and zp-004 open. zp-001 or zp-002 merged. |
| 3 | zp-005 (bf16) open. At least two stdlib PRs merged. |
| 4 | zp-006 open. zp-005 in active review. |
| 5 | zp-007 (decimal floats) open. Track record established. |
| 6 | zp-008 RFC posted. Community response documented. |
| 7 | zp-008 PR open if RFC received positively. |
| 8+ | Review, revision, merge or redirect to Zinc. |

If any PR takes substantially longer than expected, do not rush the next
one. The track record only works if the PRs are good. A rushed PR that
gets closed for quality reasons damages the track record rather than
building it.

---

## Relationship to Zinc

These PRs are the path of least resistance — getting Zinc's best ideas
into the language that Zinc would build on. If they succeed, Zinc's
implementation becomes cleaner (inherit from upstream rather than fork).
If they fail selectively, the failures document exactly why Zinc diverges
from Zig where it does.

The PRs and Zinc development are not in competition. They run in parallel.
Zinc's compiler can use the features before they land upstream by
implementing them independently, then remove the duplicate implementation
when/if the upstream version lands.

The binary layout proposal deserves special treatment: do not implement it
in Zinc before opening the upstream PR. Open the upstream PR first. If it
lands, use the upstream implementation. If it is rejected, implement it in
Zinc with the knowledge that it will not land upstream. Trying to upstream
a feature that already exists in a fork is significantly harder than
upstreaming a new proposal.

---

*End of ZIG-PLAN.md*
