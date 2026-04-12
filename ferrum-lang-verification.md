# Ferrum Language Reference — Verification and Safety

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Contracts and Proofs

### 1.1 Overview

Ferrum's contract system spans a spectrum from runtime assertions to formal proofs. The same annotation serves all modes; the enforcement mechanism is selected by build configuration.

### 1.2 Function Contracts

```ferrum
fn pop(xs: &mut Vec[T]): T
    requires xs.len() > 0
    ensures  xs.len() == old(xs.len()) - 1
    ensures  result == old(xs.last().unwrap())
{
    ...
}
```

**`requires`** — precondition. The caller's obligation. Violation is the caller's bug.

**`ensures`** — postcondition. The implementer's obligation. Violation is the implementer's bug.

**`old(expr)`** — captures the value of `expr` at function entry for use in `ensures`. The compiler generates a snapshot only for expressions actually referenced in `ensures` clauses. Snapshotting is a copy; for large types, use `old` judiciously.

**`result`** — the return value, usable in `ensures` clauses.

### 1.3 Type Invariants

Declared on the type, enforced at visibility boundaries:

```ferrum
type BoundedQueue[T] {
    data:  Vec[T],
    limit: usize,

    invariant self.data.len() <= self.limit
}
```

### 1.4 Runtime Behavior

| Build | Contracts |
|-------|-----------|
| Debug | Runtime assertions — violations abort with diagnostic |
| Release | Runtime assertions — violations abort with diagnostic (same as debug) |
| Proven | No runtime check — the proof guarantees the contract holds for all inputs |

In **debug builds**, contracts are runtime checks. A violated `requires` aborts at the call site with a message naming the caller's bug. A violated `ensures` aborts at the return site with a message naming the implementer's bug. Type invariants are checked at two points:

1. **Public API boundaries** — when an owned value of a type with invariants crosses into or out of a public function. The invariant is checked on the value itself.

2. **Exit of `trusted`/`unchecked` blocks holding `&mut T`** — when a `trusted` or `unchecked` block exits (by any means: normal completion, `return`, `?`, or panic unwind) while holding a `&mut T` where `T` has declared invariants, the invariant is checked on `*T` before the exit proceeds. This covers the case where the caller still holds the reference and will observe any violation.

```ferrum
fn push_batch(q: &mut BoundedQueue[T], items: &[T]) ! IO {
    trusted("...") {
        for item in items { q.data.push(item.clone()) }
        log_progress(q)?   // exit via ?: invariant checked here on *q before Err propagates
    }
    // boundary: also checked here at function exit
}
```

**Lint: `mut_invariant_unguarded_exit`**

The compiler emits this lint when a `trusted` or `unchecked` block:
- holds a `&mut T` where `T` has declared invariants, AND
- contains an early exit (`?`, `return`, labeled `break`) that may occur while the invariant is temporarily violated, AND
- has no `defer` expression that restores the invariant before any such exit.

The lint is a warning, not an error, because the programmer may know the invariant holds on all early-exit paths even without a `defer`. The warning text names the specific `&mut` binding and the early-exit expression.

**Ownership principle for unrestorable invariants**

If a function cannot guarantee invariant restoration on all exit paths — because the restoration itself can fail, or because the transformation is fundamentally non-atomic — the function should take **ownership** rather than `&mut`. A function that owns the value and fails can return the broken value wrapped in an error, or simply drop it. The caller never observes an intermediate state because they gave up ownership for the duration of the call.

```ferrum
// If restoration can fail, take ownership — caller never sees a broken BoundedQueue
fn push_batch(q: BoundedQueue[T], items: &[T]): Result[BoundedQueue[T], (BoundedQueue[T], IoError)] ! IO {
    // on failure, return (q_in_some_valid_state, error) so caller can recover
    // or drop q if recovery is impossible
}
```

In **release builds**, contracts remain runtime assertions — exactly as in debug builds. A violated `requires` aborts at the call site; a violated `ensures` aborts at the return site. There is no mode in which a violated contract becomes undefined behavior.

> **Design decision:** Contracts are correctness guarantees, not performance hints. Making unproven contracts into optimizer UB assumptions in release mode would mean callers cannot trust `ensures` clauses in production — a violated postcondition would silently corrupt state rather than abort. This contradicts the purpose of the contract system. The Eiffel and Ada precedent is correct: assertions check in all builds by default.
>
> The performance escape hatch is `@contracts(runtime = "never")`, which is only appropriate when the contract has been formally proven via `proven_by` — at which point the runtime check is genuinely redundant, not merely hoped to be.

For contracts you can **prove** (see §1.11), neither runtime checks nor trust are needed — the proof guarantees correctness at compile time, and `@contracts(runtime = "never")` is appropriate.

### 1.5 Proof Functions

Proof functions are pure, total, and may not have runtime effects:

```ferrum
proof fn sorted_insert_preserves_order[T: Ord](
    xs: SortedVec[T],
    x: T,
): SortedVec[T]
    ensures result.len() == xs.len() + 1
    ensures forall i, j where i < j => result[i] <= result[j]
{
    ...
}
```

Proof functions:
- Must be **total** — terminate on all inputs.
- Must be **pure** — no effects.
- Are checked for termination via structural recursion (see Section 1.7).
- May be evaluated at compile time (`@evaluate`).
- Are erased from all runtime builds — they exist only for verification.

### 1.6 `Prop`: Propositions as Types

`Prop` is a universe of types that represent logical propositions. Values of type `Prop` are proof terms. They are erased before code generation.

```ferrum
// A proposition
type Sorted[T: Ord, xs: &[T]] : Prop =
    forall i, j where i < j => xs[i] <= xs[j]

// A function that requires a proof (erased at runtime)
fn binary_search[T: Ord](
    xs: &[T],
    proof: Sorted[T, xs],     // compile-time only, zero cost
    key: &T,
): Option[usize]
{
    // The proof guarantees precondition — no runtime check needed
    ...
}
```

#### Proof Witness Opacity

`Prop[P]` is a **compiler-intrinsic sealed type**. It has no constructor accessible to user code. The only source of a `Prop[P]` value is the compiler's proof checker, which produces one as the return value of a successfully verified `proof fn`.

Direct construction is a compile error in all contexts:

```ferrum
// COMPILE ERROR: Prop has no user-accessible constructor
let fake: Prop[xs.is_sorted()] = Prop { }

// COMPILE ERROR: transmute cannot produce a Prop value
let fake: Prop[xs.is_sorted()] = unsafe { transmute(()) }
```

> **Design decision:** `unsafe` is an escape hatch for *memory safety* — raw pointer arithmetic, FFI, layout assumptions. It is not an escape hatch for *logical soundness*. These are orthogonal axes. A `Prop[P]` makes a compile-time logical claim; constructing one without a proof would not be "unsafe code" — it would be an unsound type system. The `unsafe` keyword explicitly does not extend to proof witness construction. If it did, `proven_by` guarantees would be trivially bypassable and the entire verification system would be documentation, not enforcement.

### 1.7 Totality and Termination Checking

Proof functions must terminate. Termination is verified by structural recursion:

- A recursive call is valid if at least one argument is **structurally smaller** than the corresponding parameter.
- For inductively defined types (lists, trees, natural numbers), "structurally smaller" means a sub-term produced by pattern matching.

```ferrum
// OK: xs is structurally decreasing
proof fn length[T](xs: List[T]): Nat {
    match xs {
        List.Nil        => 0,
        List.Cons(_, t) => 1 + length(t)  // t is structurally smaller than xs
    }
}

// ERROR: not obviously terminating
proof fn collatz(n: Nat): Nat { ... }
```

Mutual recursion is supported; the checker verifies that the combined recursion is well-founded.

### 1.8 The SMT Bridge

For refinement-type obligations (constraints on `where` clauses, `requires`/`ensures` involving arithmetic and comparisons), the compiler generates queries to Z3 (or a compatible SMT solver). Obligations that Z3 discharges automatically require no user-written proof. Obligations Z3 cannot discharge fall through to require manual proof terms.

```ferrum
fn safe_divide(a: u32, b: u32 where b != 0): u32 {
    a / b
}

// Caller must prove b != 0
safe_divide(10, x)       // requires proof that x != 0
safe_divide(10, 2)       // discharged by SMT: 2 != 0 is trivially true
safe_divide(10, y + 1)   // SMT may or may not discharge, depending on what's known about y
```

### 1.9 Linking Fast Implementations to Specifications

Two annotations exist, at different levels of guarantee. Choose based on what the consequences of a wrong implementation are.

#### `tested_by` — runtime equivalence checking

```ferrum
proof fn sorted_insert_spec[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    ensures result.len() == xs.len() + 1
    ensures result.is_sorted()
    ensures result.contains(&x)
{ /* reference implementation — correct but not optimized */ }

fn sorted_insert[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    tested_by(sorted_insert_spec)
{
    // fast path — binary search, in-place insert
}
```

In **debug builds**: the compiler calls both `sorted_insert` and `sorted_insert_spec` with the same inputs and asserts their outputs are equal. A mismatch panics:

```
tested_by mismatch: sorted_insert vs sorted_insert_spec
  input: xs=[1,3,5], x=2
  fast:  [1,3,5]     ← missing x
  spec:  [1,2,3,5]
```

In **release builds**: only the fast implementation runs. The spec function is never called.

`tested_by` is honest about what it provides: behavioral equivalence on inputs you actually run. It does not prove equivalence for all possible inputs. A fast implementation that is wrong only for inputs not covered by your test suite will not be caught. Use `tested_by` for performance-optimized implementations where testing gives sufficient confidence.

#### `proven_by` — formal proof of equivalence

```ferrum
proof fn sorted_insert_correct[T: Ord](xs: SortedVec[T], x: T):
    Prop[sorted_insert(xs, x) == sorted_insert_spec(xs, x)]
{ /* formal proof — verified by the compiler */ }

fn sorted_insert[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    proven_by(sorted_insert_correct)
{
    // fast path
}
```

The compiler verifies the proof function at compile time. No runtime check, no coverage concern — the proof holds for all inputs or the build fails. Use `proven_by` for cryptographic implementations, safety-critical paths, and anything where a missed edge case causes serious harm.

`proven_by` requires writing a proof function that establishes equivalence using Ferrum's proof system (§1.5–1.8). This is hard. Expect it in core libraries and security-critical code, not everyday application code.

#### What is checked in both cases

Both annotations verify at minimum that the signatures are compatible — same type parameters, same argument types, same return type. A mismatch is a compile error regardless of which annotation you use.

> **Design decision:** The original `verified_by` checked only signature compatibility, not behavioral
> equivalence — providing no actual verification guarantee while implying one. Renamed and split into
> `tested_by` (runtime comparison in debug, honest about coverage limits) and `proven_by` (formal
> proof, covers all inputs). The names say what the annotations actually do.

### 1.10 Design-by-Contract Integration

Ferrum adopts Eiffel's contract model with the following mappings:

| Eiffel | Ferrum | Notes |
|---|---|---|
| `require` | `requires` | Precondition |
| `ensure` | `ensures` | Postcondition |
| `invariant` | `invariant` on type | Class invariant |
| `old x` | `old(x)` | Pre-call value |
| `Result` | `result` | Return value in `ensures` |

### 1.11 Static Verification

Contracts that are **proven** need no runtime checks — the proof IS the verification. Unproven contracts check at runtime in all builds (debug and release).

```
ferrum build --proof
```

The `--proof` flag requires all contracts to be statically verified. The compiler generates proof obligations and discharges them via:
- **SMT solver** — arithmetic, comparisons, simple logic (automatic)
- **Proof functions** — complex properties requiring explicit proof terms
- **`Prop` types** — compile-time proof witnesses passed as (erased) parameters

Code that passes `--proof` has all contracts proven to hold. No runtime checks are generated because none are needed — the proofs guarantee correctness. The optimizer uses proven contracts as assumptions.

This is the "proofs are programs" property: proofs exist at compile time, guide type checking and optimization, but have zero runtime cost. A `Prop` parameter is erased before code generation. A `proof fn` is erased from all builds. The verification happens at compile time; the runtime sees only the fast path.

### 1.12 Fuzz Testing

```
ferrum fuzz [target]
```

In fuzz mode, contracts become test infrastructure:
- `requires` clauses become **input constraints** — the fuzzer generates inputs satisfying preconditions
- `ensures` clauses become **test oracles** — the fuzzer checks postconditions on every generated input
- `invariant` clauses are checked after every operation

This provides property-based testing for free from contract annotations. Every function with contracts is a fuzz target.

---

## 2. Binary Layout Declarations

### 2.1 Overview

Ferrum separates logical type declarations from physical binary layout declarations. The logical type defines the fields and their value constraints. The layout declaration defines exactly how those fields map to bits and bytes. The compiler verifies consistency between the two.

Binary layout declarations provide:
- Explicit, compiler-verified bit-level field placement
- Automatic byte order conversion
- Zero-copy view types for memory-mapped I/O
- Generated read/write codecs with constraint validation

### 2.2 Syntax

```ferrum
// Logical type
type EthernetFrame {
    dst_mac:  [u8; 6],
    src_mac:  [u8; 6],
    ethertype: u16,
}

// Physical layout
layout EthernetFrame {
    byte_order:  big_endian,
    total_size:  112 bits,

    dst_mac   at bytes 0..5,
    src_mac   at bytes 6..11,
    ethertype at bytes 12..13,
}
```

### 2.3 Layout Properties

**`byte_order`**: `big_endian` | `little_endian` | `native`

Applies to all multi-byte fields unless overridden per-field. Byte swapping between orders is generated automatically.

**`bit_order`**: `lsb_first` | `msb_first`

Controls bit numbering within a byte. `lsb_first` (default for x86): bit 0 is the least significant bit. `msb_first`: bit 0 is the most significant bit.

**`total_size`**: in bits or bytes. Compiler error if fields do not exactly fill the declared size.

### 2.4 Field Placement

```ferrum
layout StatusRegister {
    byte_order: little_endian,
    bit_order:  lsb_first,
    total_size: 16 bits,

    ready    at byte 0, bits 0..0,     // single bit
    mode     at byte 0, bits 1..3,     // 3-bit field
    reserved at byte 0, bits 4..7,     // padding (must be accounted for)
    error    at byte 1, bits 0..0,
    _pad     at byte 1, bits 1..7,     // unnamed padding, zeroed on write
}
```

**Placement specifications:**
```
at byte N               — single byte, field fills the byte
at bytes N..M           — inclusive byte range
at byte N, bits B..B    — bit range within a byte (inclusive)
at bytes N..M, bits B..B — bit range within a byte range
```

**Compiler verifications:**
- No overlapping field ranges.
- No gaps (all bits accounted for — use `_pad` for explicit padding).
- Field type is wide enough for the bit range.
- Total size matches `total_size` declaration.
- Multi-byte fields on non-natural alignment require explicit `unaligned` marker.

### 2.5 Padding

Padding must be explicit. Implicit padding is a compile error.

```ferrum
layout Packet {
    total_size: 32 bits,

    version  at byte 0, bits 0..3,
    _pad0    at byte 0, bits 4..7,    // explicit 4-bit pad, zeroed on write
    length   at bytes 1..2,
    _pad1    at byte 3,               // explicit byte pad, zeroed on write
}
```

Named padding (`_pad0`, `_pad1`) is inaccessible from the logical type and is always zero-initialized on write. The `_` prefix is required.

### 2.6 Per-field Overrides

```ferrum
layout MixedEndian {
    total_size: 32 bits,

    upper at bytes 0..1 byte_order: big_endian,
    lower at bytes 2..3 byte_order: little_endian,
}
```

Per-field `byte_order` overrides the layout-level default.

### 2.7 Multiple Layouts

A single logical type may have multiple layout declarations, each named:

```ferrum
type Timestamp {
    seconds:     u32,
    nanoseconds: u32 where value < 1_000_000_000,
}

layout Timestamp as PosixTimespec {
    byte_order: native,
    total_size: 64 bits,
    seconds     at bytes 0..3,
    nanoseconds at bytes 4..7,
}

layout Timestamp as NtpTimestamp {
    byte_order: big_endian,
    total_size: 64 bits,
    seconds     at bytes 0..3,
    fraction    at bytes 4..7,
        convert:      fn(ns: u32): u32 { ... },
        convert_back: fn(f:  u32): u32 { ... },
}
```

`convert` and `convert_back` are value-level transformations applied during read and write respectively. They are pure functions.

### 2.8 Generated Operations

For each layout declaration, the compiler generates:

**`Type.read(bytes: &[u8]): Result[Type]`**
- Extracts fields from raw bytes per the layout.
- Applies byte/bit order conversion.
- Applies `convert_back` functions.
- Validates all field constraints from the logical type.
- Returns `Err` on constraint violation.

**`value.write(): [u8; N]`**
- Packs fields into raw bytes per the layout.
- Applies `convert` functions.
- Applies byte/bit order conversion.
- Zeros all `_pad` fields.

**`Type.view(bytes: &[u8]): TypeView`**
- Zero-copy reference into raw bytes.
- Field accessor methods extract bits on read.
- Field mutator methods pack bits on write.
- Suitable for memory-mapped I/O.

**`value.convert[OtherLayout](): Type layout OtherLayout`**
- Converts between named layouts.
- Applies any necessary byte swapping and value conversions.

### 2.9 Endianness Safety

When a value with an explicit layout is passed to a function expecting a different byte order, the compiler inserts the conversion automatically if the layout is statically known:

```ferrum
fn send_packet(frame: EthernetFrame layout NetworkOrder) ! Net { ... }

let frame = EthernetFrame { ... }   // host byte order (native)
send_packet(frame)                  // auto-converts to NetworkOrder
```

If the byte order is not statically known, explicit conversion is required.

---

## 3. Safety Levels

### 3.1 The Permission Ladder

Ferrum provides a graduated ladder of five safety levels. Each level is **strictly additive** — a higher level cannot be used to do what a lower level prohibits through misrepresentation.

| Level | Keyword | What it relaxes |
|---|---|---|
| 0 | (default) | Nothing. Full checker, all guarantees. |
| 1 | `unchecked` | Borrow checker aliasing rules only. Memory safety preserved. |
| 2 | `trusted` | Named assertions replace compiler verification. |
| 3 | `extern` | FFI boundary. Ferrum types checked; foreign behavior unknown. |
| 4 | `unsafe` | Raw pointers, arbitrary memory. Full responsibility. |

### 3.2 `unchecked`

Relaxes the aliasing rules of the borrow checker. Memory is still safe — no raw pointers, no undefined behavior. The only thing relaxed is the compiler's conservative approximation of aliasing.

```ferrum
unchecked fn split_at_mut[T](xs: &mut [T], mid: usize): (&mut [T], &mut [T]) {
    // Checker rejects this: two &mut to the same backing array
    // unchecked: "I assert these halves do not overlap"
    // Memory safety: preserved. No raw pointers used.
    let len = xs.len()
    assert(mid <= len)
    let ptr = xs.as_mut_ptr()
    (
        core.slice.from_raw_parts_mut(ptr, mid),
        core.slice.from_raw_parts_mut(ptr.add(mid), len - mid),
    )
}
```

`unchecked` blocks within otherwise-safe functions are permitted:

```ferrum
fn process(xs: &mut [T]) {
    let (left, right) = unchecked { split_at_mut(xs, xs.len() / 2) }
    // left and right: full borrow checker from this point on
    use(left, right)
}
```

### 3.3 `trusted`

Makes a named assertion about a property the compiler cannot verify. The assertion string is machine-readable and auditable.

```ferrum
trusted("ptr is valid for len bytes, properly aligned, no aliasing references")
trusted("len does not exceed isize.MAX")
fn from_raw_parts[T](ptr: *const T, len: usize): &[T] {
    ...
}
```

Multiple `trusted` annotations are permitted on one function, each naming a distinct claim.

`trusted` functions appear in the audit report (`ferrum audit --level trusted`) with their assertion strings. This makes the trust boundary explicit, nameable, and searchable.

#### Prohibited assertions

`trusted` may not assert `Send` or `Sync` on a type whose fields include a `!Send` or `!Sync` type. The compiler enforces this: if a `trusted` block attempts to use or produce a type in a `Send`/`Sync` position and the compiler has determined that type is `!Send`/`!Sync` due to its fields, the assertion is a compile error.

```ferrum
// COMPILE ERROR: RefCell[u64] is !Sync — trusted cannot override a known structural fact.
// The compiler determined !Sync from the field type, not from conservative analysis.
trusted("we manage synchronization ourselves") {
    share_across_threads(Arc::new(RefCell::new(0u64)))
}
```

The distinction that matters: `trusted` exists for properties the compiler's analysis cannot see — aliasing patterns, pointer validity, domain-specific invariants. Whether a type is `Send`/`Sync` is *not* something the compiler cannot see — it is derived directly from field types, and that derivation is correct. `trusted` cannot contradict a correct derivation; it can only fill in information the compiler lacks.

**Legitimate alternative for custom synchronization primitives:**

If you are wrapping a `!Sync` type with your own synchronization (e.g., building a new lock type around a `!Sync` primitive), use `unsafe impl`:

```ferrum
// MyMutex wraps UnsafeCell (which is !Sync) but provides its own locking protocol.
// This requires unsafe impl, not trusted, because it declares a new synchronization contract.
unsafe impl Sync for MyMutex[T] {}
unsafe impl Send for MyMutex[T] {}
```

`unsafe impl Sync` is appropriate when you are *implementing* a synchronization primitive. `trusted` is never appropriate for asserting thread safety on a type that contains `!Sync` fields you did not wrap with synchronization.

> **Design decision:** `trusted` was prohibited from asserting `Send`/`Sync` on `!Sync`-field types
> because the `!Sync` determination is structural and correct — the compiler is not being overly
> conservative, the type genuinely is not thread-safe. Allowing `trusted` to override a correct
> structural determination would hollow out data-race freedom: any `!Sync` type could be made
> "trusted Sync" with a string argument. The legitimate use case (implementing a new
> synchronization primitive) already has a proper mechanism: `unsafe impl Sync`.

### 3.4 `extern`

FFI boundary. Ferrum's type checker still checks the types on the Ferrum side of the boundary. The foreign code's behavior is unknown.

```ferrum
extern(c) {
    fn strlen(s: *const u8): usize  ! Unsafe
    fn memcpy(dst: *mut u8, src: *const u8, n: usize): *mut u8  ! Unsafe
}

extern(c) fn callback_from_c(data: *const u8, len: usize) {
    // This function is called by C code — types are as declared
}
```

`extern(c)` declares the C foreign model with default cdecl calling convention.
Calling convention variants: `extern(c.system)`, `extern(c.fastcall)`, `extern(c.stdcall)`, `extern(c.aapcs)`, `extern(c.win64)`.

### 3.5 `unsafe`

Raw pointers, manual memory operations, inline assembly. All compiler guarantees are the programmer's responsibility.

```ferrum
unsafe fn write_volatile[T](ptr: *mut T, val: T) {
    core.intrinsics.volatile_store(ptr, val)
}

unsafe {
    // unsafe block within a safe function
    *raw_ptr = value
}
```

### 3.6 Block-level Scoping

All levels can be applied to a block rather than a whole function, minimizing the blast radius:

```ferrum
fn mostly_safe(xs: &mut [T]) {
    // Full checker here
    let mid = xs.len() / 2

    let (left, right) = unchecked { split_at_mut(xs, mid) }

    // Full checker resumes here
    process(left)
    process(right)
}
```

### 3.7 The Audit Command

```
ferrum audit [path] [--level <level>] [--format <text|json|sarif>]
```

Reports all sites at the specified safety level and above. Useful for security reviews and compliance.

```
ferrum audit src/ --level trusted

src/net/packet.rs:47  trusted  "ptr valid for len bytes, aligned"
src/mem/arena.rs:112  trusted  "offset within bounds, no overflow"
src/io/file.rs:203    trusted  "fd is valid and open"
```

### 3.8 The Annotation Philosophy

The compiler handles the common cases automatically. The safety ladder exists for the cases where the compiler's reasoning is insufficient — where human judgment must fill the gap.

Every `trusted("...")` annotation is:
- A statement a security auditor can search for and read
- Documentation of a human judgment that would otherwise be invisible
- A machine-readable signal the compiler can use for targeted verification
- Load-bearing specification of something the compiler cannot verify automatically

This is strictly better than a language that either rejects valid code silently or accepts invalid code silently. The boundary between compiler knowledge and human knowledge is explicit.

The borrow checker handles aliasing in the common cases. The `unchecked` level exists for valid patterns the checker cannot yet recognize. The `trusted` level exists for properties that require domain knowledge to assert. Each annotation that remains in a codebase is load-bearing — it carries information the compiler needs but cannot derive.

The audit report is a map of where human judgment was required. In a mature codebase, these annotations cluster around the interesting parts: real-time paths, security-critical code, performance kernels, hardware interfaces.

---
