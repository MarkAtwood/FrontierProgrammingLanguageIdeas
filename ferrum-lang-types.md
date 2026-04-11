# Ferrum Language Reference — Type System

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Type System

### 1.1 Overview

Ferrum's type system is stratified into four layers that interact but are conceptually distinct:

```
┌─────────────────────────────────────────────────────┐
│  Effect layer     IO · Net · Sync · Unsafe · Panic  │
├─────────────────────────────────────────────────────┤
│  Ownership layer  Owned · &T · &mut T · Region      │
├─────────────────────────────────────────────────────┤
│  Type layer       Scalar · Sum · Product · Fn · ...  │
├─────────────────────────────────────────────────────┤
│  Allocator layer  Heap · Arena · Pool · Stack · Null │
└─────────────────────────────────────────────────────┘
```

Type parameters fall into exactly three categories, each with a different mechanism:

| Category | Examples | Mechanism | Visibility |
|---|---|---|---|
| Caller-meaningful variance | element type, key type | explicit `[T]` | always visible |
| Ambient capabilities | allocator, effects, locale | implicit `given` | at boundaries only |
| Covariant families | iterator + item + allocator | associated types | inside trait definitions |

### 1.2 Scalar Types

Ferrum provides the complete numeric spectrum required for systems programming:
ML/GPU work, scientific computing, finance, audio DSP, networking, cryptography,
and arbitrary precision.

#### 1.2.1 Integer Types

**Standard integers (hardware-native on all targets):**

| Type | Signed | Width | Value range |
|------|--------|-------|-------------|
| `u8` | no | 8 bits | 0 … 255 |
| `u16` | no | 16 bits | 0 … 65,535 |
| `u32` | no | 32 bits | 0 … 4,294,967,295 |
| `u64` | no | 64 bits | 0 … 1.8×10¹⁹ |
| `u128` | no | 128 bits | 0 … 3.4×10³⁸ |
| `i8` | yes | 8 bits | −128 … 127 |
| `i16` | yes | 16 bits | −32,768 … 32,767 |
| `i32` | yes | 32 bits | −2,147,483,648 … 2,147,483,647 |
| `i64` | yes | 64 bits | −9.2×10¹⁸ … 9.2×10¹⁸ |
| `i128` | yes | 128 bits | −1.7×10³⁸ … 1.7×10³⁸ |
| `usize` | no | ptr width | platform |
| `isize` | yes | ptr width | platform |

`byte` is an alias for `u8`. `char` is a Unicode scalar value (4 bytes).
`bool` is 1 byte. `never` is the bottom type (0 bytes, no values).

**Non-power-of-two byte-aligned integers:**

These are first-class types stored in exactly their byte width.
They are not aliases for wider types — arithmetic uses the narrower width
with defined wrapping/overflow behavior.

| Type | Signed | Width | Common use |
|------|--------|-------|------------|
| `u24` | no | 24 bits (3 bytes) | 24-bit audio samples, pixel formats |
| `u48` | no | 48 bits (6 bytes) | MAC addresses, PTP timestamps |
| `u96` | no | 96 bits (12 bytes) | IPv6 interface identifiers, UUIDs |
| `i24` | yes | 24 bits | 24-bit signed audio, DSP |
| `i48` | yes | 48 bits | Signed PTP offset |
| `i96` | yes | 96 bits | Large signed quantities |

```ferrum
// u24 in a struct — stored as 3 consecutive bytes, no padding
type AudioSample {
    left:  i24,
    right: i24,
}

// Arithmetic
let a: u24 = 0xFF_FF_FFu24    // max value
let b: u24 = a +% 1u24        // wraps to 0 — wrap is explicit
```

**Large fixed-width integers:**

Stored as arrays of `u64` internally. On targets with wide SIMD, the compiler
may use vector instructions. Otherwise, software multi-limb arithmetic.

| Type | Signed | Width | Common use |
|------|--------|-------|------------|
| `u256` | no | 256 bits | Cryptographic keys, Ethereum addresses |
| `u512` | no | 512 bits | Cryptographic hashes, RSA moduli |
| `u1024` | no | 1024 bits | RSA private keys, large multiplications |
| `i256` | yes | 256 bits | Signed cryptographic arithmetic |
| `i512` | yes | 512 bits | Large signed arithmetic |
| `i1024` | yes | 1024 bits | Very large signed arithmetic |

```ferrum
let key: u256 = 0xDEAD_BEEF_CAFE_BABE_u256
let sum:  u256 = a + b              // checked in debug, wrapping in release
let prod: u256 = a *% b             // explicit wrapping
let rot: u256 = a.rotate_left(64)   // hardware-assisted on wide SIMD
```

**Sub-byte integer types:**

`u1` through `u7` and `i1` through `i7` are constrained types derived from
their smallest containing power-of-two type. As standalone values they occupy
one byte. In `layout` declarations they pack to their exact bit width.

```ferrum
type u1 = u8  where value <= 1
type u2 = u8  where value <= 3
type u3 = u8  where value <= 7
type u4 = u8  where value <= 15
// ... and so on

type i1 = i8  where value >= -1 && value <= 0
type i2 = i8  where value >= -2 && value <= 1
// ... and so on
```

**Arbitrary bit-width integers:**

Beyond the predefined widths, Ferrum supports integers of any bit width from 0 to
65535 bits. These are written `uN` or `iN` where N is a compile-time constant:

```ferrum
type u7  = @Int(false, 7)    // 7-bit unsigned: 0..127
type i13 = @Int(true, 13)    // 13-bit signed: -4096..4095
type u24 = @Int(false, 24)   // 24-bit unsigned (also a keyword)

// Any width up to 65535 bits
let hash: u384 = compute_hash()
let tiny: u3   = 5            // stored in smallest containing type
```

The `@Int(signed, bits)` builtin constructs an integer type with the specified
signedness and width. For widths ≤ 8, values are stored in `u8`/`i8`. For larger
widths, storage rounds up to the next power of two for standalone values. In
`layout` declarations, arbitrary-width integers pack to their exact bit width.

Arithmetic on arbitrary-width integers follows the same overflow rules as standard
integers. The compiler generates optimal code for the target: native instructions
where the width matches hardware, SIMD for widths matching vector lanes, and
multi-limb software arithmetic for larger widths.

```ferrum
// Custom protocol field widths
type SequenceNumber = @Int(false, 12)   // 12-bit sequence
type FragmentOffset = @Int(false, 13)   // 13-bit offset

// Layout packs to exact bits
@layout
type IPv4Flags {
    reserved:    u1,
    dont_frag:   u1,
    more_frags:  u1,
    frag_offset: FragmentOffset,
}  // 16 bits total, no padding
```

**Integer arithmetic — overflow behavior:**

```ferrum
let a = x + y          // panics on overflow in debug, wraps in release
let b = x +% y         // always wraps (modular arithmetic)
let c = x +| y         // always saturates
let d = x.checked_add(y)  // returns Option[T]
let (e, overflow) = x.overflowing_add(y)  // returns (result, bool)
```

#### 1.2.2 Binary Float Types

| Type | Standard | Bits | Mantissa | Decimal digits | Hardware |
|------|----------|------|----------|----------------|----------|
| `f16` | IEEE 754-2008 | 16 | 10 | ~3.3 | GPUs, ARM v8.2+ |
| `bf16` | Non-standard | 16 | 7 | ~2.3 | TPU, A100+, AMX |
| `f32` | IEEE 754-1985 | 32 | 23 | ~7.2 | Universal |
| `f64` | IEEE 754-1985 | 64 | 52 | ~15.9 | Universal |
| `f80` | Non-standard (x87) | 80+pad | 63+int | ~18.5 | x86 only native |
| `f128` | IEEE 754-2008 | 128 | 112 | ~33.6 | POWER9+, z/Arch |
| `f256` | Non-standard | 256 | 236 | ~71.3 | Always soft |

`f16` is half precision for ML inference and texture formats. `bf16` (brain
float) is truncated `f32` — wider range but less precision than `f16`.
`f80` is x87 extended precision (soft-float on non-x86). `f128` is quad
precision for high-precision numerical analysis. `f256` is octuple precision,
always software-implemented.

```ferrum
let x: f16 = 3.14f16
let y: f32 = x as f32     // explicit widening — never implicit

// f16 does not automatically widen to f32 for arithmetic
// On hardware without f16 arithmetic, ops use software fallback
```

**`bf16` and `f16` are distinct types.** Converting between them requires
passing through `f32`.

#### 1.2.3 Decimal Float Types

Decimal floats implement IEEE 754-2008 decimal floating-point arithmetic.
They solve the binary representation problem: `0.1` cannot be represented
exactly in binary float.

```ferrum
// Binary float: wrong
0.1f64 + 0.2f64 == 0.3f64    // false — binary rounding

// Decimal float: correct
0.1d64 + 0.2d64 == 0.3d64    // true — exact decimal arithmetic
```

| Type | Width | Significant digits | Use case |
|------|-------|-------------------|----------|
| `d32` | 32 bits | 7 | Compact storage |
| `d64` | 64 bits | 16 | Financial arithmetic |
| `d128` | 128 bits | 34 | High-precision finance |

Decimal floats support arithmetic operators but **not** bitwise operations.
Transcendental functions are **not** defined on decimal floats — convert to
`f64` first if needed. Hardware acceleration exists on IBM POWER6+ and z10+.

```ferrum
let price:  d64 = 1234567.89d64
let tax:    d64 = price * 0.0825d64
let rounded: d64 = total.round_to(2)  // round to cents
```

#### 1.2.4 Fixed-Point Types

Fixed-point arithmetic represents fractional values as scaled integers.
No rounding during arithmetic (except division). Zero overhead on hardware.

```ferrum
// Signed fixed-point: I integer bits + F fractional bits
type Fixed[const I: usize, const F: usize]

// Unsigned fixed-point
type UFixed[const I: usize, const F: usize]

// Standard aliases (DSP Q notation):
type Q8_8   = Fixed[8, 8]     // 16-bit, range −128 to 127.996...
type Q16_16 = Fixed[16, 16]   // 32-bit, common in game engines
type Q1_15  = Fixed[1, 15]    // 16-bit, range −1 to 0.99997, audio DSP
```

```ferrum
let x: Q16_16 = 3.75           // from float literal — exact if representable
let sum:  Q16_16 = x + y       // stays Q16_16, wraps on overflow
let prod: Q16_16 = (x * y).rescale()  // multiplication doubles F bits
let clamped: Q1_15 = (x + y).saturate()  // clamps instead of wrapping
```

#### 1.2.5 Arbitrary Precision Types

Arbitrary precision types allocate as needed and live in `alloc`. They are
**not** `Copy` — they are heap-allocated and moved.

```ferrum
import alloc.bigint.{BigInt, BigUint}
import alloc.bigdecimal.BigDecimal
import alloc.rational.Rational

let x: BigInt = BigInt.parse("99999999999999999999999999999999")?
let y: Rational = Rational.new(1, 3) + Rational.new(1, 3)  // exactly 2/3
let z: Complex[f64] = Complex.new(3.0, 4.0)  // 3 + 4i
```

#### 1.2.6 Compile-Time Numeric Types

At compile time, all numeric literals have unlimited precision. The type
`comptime_int` represents an integer of arbitrary size during compilation,
and `comptime_float` represents a floating-point number with arbitrary precision.

```ferrum
const FACTORIAL_50: comptime_int = 30414093201713378043612608166064768844377641568960512000000000000
const PI_100: comptime_float = 3.14159265358979323846264338327950288419716939937510...

// comptime values participate in constant folding with no overflow
const BIG: comptime_int = 1 << 1000   // no problem at comptime

// When assigned to a runtime type, the value is checked to fit
const X: u64 = FACTORIAL_50           // error: value too large for u64
const Y: u64 = 1 << 63                // ok: fits in u64
```

**Key properties:**

- `comptime_int` and `comptime_float` exist only at compile time. They cannot
  be stored in runtime variables or passed to runtime functions.
- All integer literals in constant expressions are `comptime_int` until coerced.
- Division and remainder at comptime are exact for `comptime_int` (truncating)
  and arbitrary-precision for `comptime_float`.
- When a comptime value is assigned to a concrete type, the compiler checks
  that it fits. Overflow is a compile error, not runtime behavior.

```ferrum
// Compile-time computation with unlimited precision
const fn compute_constant(): u128 {
    let huge: comptime_int = 2 ** 200
    let scaled = huge / (2 ** 72)
    scaled as u128  // checked to fit at comptime
}

// Generic code works with comptime numerics
const fn power[const N: comptime_int](base: comptime_int): comptime_int {
    if N == 0 { 1 }
    else { base * power[N - 1](base) }
}
```

`comptime_float` retains enough precision to round correctly to any target
float type. The exact internal representation is implementation-defined but
must be at least 256 bits of mantissa.

#### 1.2.7 Numeric Trait Hierarchy

```ferrum
trait Numeric: Copy + PartialEq + Display + Debug {
    const ZERO: Self
    const ONE:  Self
}

trait Integer: NumericOrd + Eq + Ord + BitAnd + BitOr + ... {
    const BITS: u32
    const MIN:  Self
    const MAX:  Self
    fn count_ones(&self): u32
    fn checked_add(&self, rhs: Self): Option[Self]
    // ...
}

trait Float: NumericOrd {
    // Special value constants
    const NAN: Self
    const INFINITY: Self
    const NEG_INFINITY: Self
    const ZERO: Self
    const NEG_ZERO: Self
    const MIN: Self              // smallest finite value
    const MAX: Self              // largest finite value
    const MIN_POSITIVE: Self     // smallest positive normal
    const EPSILON: Self          // difference between 1.0 and next representable

    // Classification
    fn is_nan(&self): bool
    fn is_infinite(&self): bool
    fn is_finite(&self): bool
    fn is_normal(&self): bool
    fn is_subnormal(&self): bool
    fn is_sign_positive(&self): bool
    fn is_sign_negative(&self): bool
    fn classify(&self): FpCategory

    // Arithmetic
    fn sqrt(&self): Self
    fn abs(&self): Self
    fn copysign(&self, sign: Self): Self
    fn signum(&self): Self     // -1.0, 0.0, 1.0, or NaN

    // Comparison (NaN-aware)
    fn min(&self, other: Self): Self      // NaN propagating
    fn max(&self, other: Self): Self      // NaN propagating
    fn min_num(&self, other: Self): Self  // NaN non-propagating (IEEE 754-2008)
    fn max_num(&self, other: Self): Self  // NaN non-propagating
    fn total_cmp(&self, other: &Self): Ordering  // total order including NaN
}

enum FpCategory { Nan, Infinite, Zero, Subnormal, Normal }

trait BinaryFloat: Float { /* to_bits, from_bits, ulp, next_up, next_down */ }
trait DecimalFloat: Float { /* round_to, quantize, is_integer */ }
trait FixedPoint: NumericOrd { /* from_raw, to_raw, saturate */ }
```

#### 1.2.8 Special Floating-Point Values

IEEE 754 floating-point types have special values beyond the normal numeric range.
Ferrum provides explicit, predictable semantics for all of them.

**The special values:**

| Value | Bit pattern (f64) | Description |
|-------|-------------------|-------------|
| `+0.0` | `0x0000000000000000` | Positive zero |
| `-0.0` | `0x8000000000000000` | Negative zero (distinct bit pattern) |
| `+∞` | `0x7FF0000000000000` | Positive infinity |
| `-∞` | `0xFFF0000000000000` | Negative infinity |
| `NaN` | `0x7FF8000000000000` | Quiet NaN (canonical) |
| `sNaN` | `0x7FF0000000000001` | Signaling NaN (raises exception) |
| subnormal | exponent = 0 | Gradual underflow values |

**Signed zero semantics:**

Positive and negative zero are equal under `==` but distinguishable via bit
pattern and `is_sign_negative()`:

```ferrum
let pos: f64 = 0.0
let neg: f64 = -0.0

pos == neg                    // true — IEEE 754 requires this
pos.to_bits() == neg.to_bits() // false — different bit patterns
pos.is_sign_negative()        // false
neg.is_sign_negative()        // true

// Signed zero matters for:
1.0 / pos                     // +∞
1.0 / neg                     // -∞
pos.atan2(neg)                // π (not 0)
```

**NaN semantics:**

NaN (Not a Number) represents undefined or unrepresentable results. Ferrum
follows IEEE 754 with explicit handling:

```ferrum
let nan: f64 = f64.NAN

// NaN is not equal to anything, including itself
nan == nan                    // false
nan != nan                    // true
nan < 1.0                     // false
nan > 1.0                     // false
nan <= nan                    // false

// NaN detection
nan.is_nan()                  // true
f64.NAN.is_nan()            // true

// NaN propagation — any operation involving NaN produces NaN
nan + 1.0                     // NaN
nan * 0.0                     // NaN (not 0.0!)
nan.sqrt()                    // NaN

// Operations that produce NaN:
0.0 / 0.0                     // NaN
f64.INFINITY - f64.INFINITY // NaN
f64.INFINITY * 0.0           // NaN
(-1.0f64).sqrt()              // NaN
```

**Quiet vs signaling NaN:**

Ferrum distinguishes quiet NaN (qNaN) and signaling NaN (sNaN):

- **Quiet NaN** (`f64.NAN`): Propagates silently through arithmetic
- **Signaling NaN**: Raises a floating-point exception when used in arithmetic

Signaling NaN is primarily for debugging — it marks uninitialized float storage:

```ferrum
// Create signaling NaN
let snan: f64 = f64.signaling_nan()

// Using sNaN in arithmetic raises Panic effect (in debug) or converts to qNaN (in release)
let x = snan + 1.0    // debug: panic; release: qNaN

// Check for signaling NaN
snan.is_signaling_nan()  // true
f64.NAN.is_signaling_nan()  // false
```

**NaN payloads:**

NaN values carry a payload in their mantissa bits. Ferrum preserves payloads
through unary operations but not through binary operations:

```ferrum
let nan_with_payload = f64.nan_with_payload(0x123456)
nan_with_payload.nan_payload()  // Some(0x123456)

// Unary operations preserve payload
(-nan_with_payload).nan_payload()  // Some(0x123456)

// Binary operations produce canonical NaN
(nan_with_payload + 1.0).nan_payload()  // Some(0) — canonical
```

**Infinity semantics:**

```ferrum
let inf: f64 = f64.INFINITY
let neg_inf: f64 = f64.NEG_INFINITY

inf.is_infinite()             // true
inf.is_finite()               // false
inf > f64.MAX                // true
inf + 1.0                     // +∞
inf + inf                     // +∞
inf - inf                     // NaN
1.0 / inf                     // 0.0
inf / inf                     // NaN

// Operations that produce infinity:
1.0 / 0.0                     // +∞
-1.0 / 0.0                    // -∞
f64.MAX * 2.0                // +∞ (overflow)
```

**Subnormal (denormal) values:**

Subnormals are tiny values between zero and the smallest normal number. They
provide gradual underflow at the cost of precision:

```ferrum
let tiny: f64 = f64.MIN_POSITIVE / 2.0
tiny.is_subnormal()           // true
tiny.is_normal()              // false
tiny.is_finite()              // true

// Subnormals have reduced precision (leading zeros in mantissa)
// Some hardware flushes subnormals to zero for performance
```

**Total ordering:**

Standard comparison (`<`, `==`, etc.) follows IEEE 754 and does not provide
a total order because NaN is unordered. For sorting and hashing, use `total_cmp`:

```ferrum
// Standard comparison — NaN breaks ordering
[1.0, f64.NAN, 2.0].sort()   // error: Float does not implement Ord

// total_cmp provides a total order:
// -NaN < -∞ < -finite < -0.0 < +0.0 < +finite < +∞ < +NaN
[1.0, f64.NAN, 2.0].sort_by(f64.total_cmp)  // [1.0, 2.0, NaN]

// For HashMap/HashSet, floats must use total_cmp-based wrapper:
import collections.TotalFloat
let set: HashSet[TotalFloat[f64]] = ...
```

**Comparison summary:**

| Operation | `-0.0` vs `+0.0` | `NaN` vs anything |
|-----------|------------------|-------------------|
| `==` | equal | false (including NaN == NaN) |
| `<`, `>`, `<=`, `>=` | equal | false |
| `total_cmp` | `-0.0 < +0.0` | NaN has defined position |
| `to_bits` | different | different per payload |
| `min` / `max` | either | propagates NaN |
| `min_num` / `max_num` | either | returns non-NaN operand |

#### 1.2.9 Type Conversion Rules

**The fundamental rule: no implicit numeric conversions. Ever.**

Every numeric type conversion is explicit. Ferrum provides four conversion
mechanisms with different overflow behaviors:

| Mechanism | Overflow behavior | When to use |
|-----------|-------------------|-------------|
| `as` | Defined (wraps, saturates, or truncates) | When you accept the defined behavior |
| `try_into()` | Returns `Result` | When overflow is an error |
| `.saturating_into()` | Clamps to target range | When clamping is correct |
| `.wrapping_into()` | Wraps (modular) | When wraparound is intentional |

```ferrum
let big: i32 = 300

// as — wraps for integers
let a: u8 = big as u8           // 44 (wraps)

// try_into — returns Result
let b: Result[u8, _] = big.try_into()  // Err(OutOfRange)

// saturating_into — clamps
let c: u8 = big.saturating_into()      // 255

// wrapping_into — explicit wrap
let d: u8 = big.wrapping_into()        // 44
```

**Conversion matrix — what `as` does:**

| From | To | `as` behavior |
|------|----|----|
| Integer | Wider integer | Zero-extend (unsigned) or sign-extend (signed) |
| Integer | Narrower integer | Truncate (keep low bits) |
| Signed | Unsigned (same width) | Reinterpret bits |
| Unsigned | Signed (same width) | Reinterpret bits |
| Float | Wider float | Exact (no precision loss) |
| Float | Narrower float | Round to nearest, ties to even |
| Float | Integer | Truncate toward zero, saturate on overflow |
| Integer | Float | Round to nearest representable |
| Decimal | Binary float | Approximate (may lose precision) |
| Binary float | Decimal | Exact if representable, else round |

**Integer conversions:**

```ferrum
// Widening — always exact
let x: i8 = -5
let y: i32 = x as i32     // -5 (sign-extended)
let z: u32 = 200u8 as u32 // 200 (zero-extended)

// Narrowing — truncates (keeps low bits)
300i32 as u8              // 44  (300 & 0xFF = 44)
-1i32 as u8               // 255 (0xFFFFFFFF & 0xFF = 0xFF)
256i32 as u8              // 0   (256 & 0xFF = 0)

// Signed ↔ unsigned — reinterprets bits
-1i8 as u8                // 255 (same bits: 0xFF)
255u8 as i8               // -1  (same bits: 0xFF)
128u8 as i8               // -128 (0x80 as signed)

// Checked alternatives
300i32.try_into(): Result[u8, _]     // Err
300i32.saturating_into(): u8         // 255
```

**Float ↔ integer conversions:**

```ferrum
// Float to integer — truncates toward zero, saturates on overflow
3.7f64 as i32             // 3 (truncated)
-3.7f64 as i32            // -3 (truncated toward zero)
1e30f64 as i32            // i32.MAX (saturated, not UB)
-1e30f64 as i32           // i32.MIN (saturated)
f64.NAN as i32           // 0 (defined, not UB)
f64.INFINITY as i32      // i32.MAX (saturated)
f64.NEG_INFINITY as i32  // i32.MIN (saturated)

// Integer to float — rounds to nearest representable
16777217i32 as f32        // 16777216.0 (rounded, can't represent exactly)
i64.MAX as f64           // 9223372036854775808.0 (rounded)

// Checked float → integer
3.7f64.try_into(): Result[i32, _]    // Ok(3)
1e30f64.try_into(): Result[i32, _]   // Err(Overflow)
f64.NAN.try_into(): Result[i32, _]  // Err(NotANumber)
```

**Float ↔ float conversions:**

```ferrum
// Widening — always exact
3.14f32 as f64            // 3.140000104904175 (exact f32 value in f64)

// Narrowing — rounds to nearest
3.141592653589793f64 as f32  // 3.1415927 (rounded)
1e40f64 as f32            // f32.INFINITY (overflow → infinity)
1e-50f64 as f32           // 0.0 (underflow → zero, or subnormal)

// NaN and infinity
f64.NAN as f32           // f32.NAN (canonical, payload may change)
f64.INFINITY as f32      // f32.INFINITY
f64.NEG_INFINITY as f32  // f32.NEG_INFINITY
```

**Decimal ↔ binary float conversions:**

```ferrum
// Decimal to binary — approximates
0.1d64 as f64             // 0.1000000000000000055... (binary approximation)
// WARNING: This loses the decimal exactness!

// Binary to decimal — exact if representable
0.5f64 as d64             // 0.5d64 (exact: 1/2 is representable)
0.1f64 as d64             // 0.1000000000000000055...d64 (preserves binary error)

// For exact decimal conversion, parse from string:
let price = d64.parse("0.10")?  // exactly 0.10
```

**Fixed-point conversions:**

```ferrum
// Float to fixed-point — may lose precision or overflow
3.75f64 as Q16_16         // exact (3 + 3/4)
3.7f64 as Q16_16          // rounded (3.7 not exactly representable)
100000.0f64 as Q16_16     // saturates to Q16_16.MAX

// Fixed-point to float — may round
let q: Q16_16 = 3.75
q as f64                  // 3.75 (exact for this value)
q as f32                  // may round for some Q16_16 values

// Between fixed-point types
let q1: Q8_8 = 3.5
let q2: Q16_16 = q1 as Q16_16    // exact (more precision)
let q3: Q4_4 = q1 as Q4_4        // may overflow or lose precision
```

**Constrained type conversions:**

```ferrum
type Percent = u8 where value <= 100
type Port = u16 where value >= 1024

// Into constrained type — checked at conversion point
let p: Percent = 50u8.try_into()?      // Ok(50)
let p: Percent = 150u8.try_into()?     // Err(ConstraintViolation)

// From constrained type — always succeeds (constraint satisfied)
let x: u8 = p.into()                   // 50

// as bypasses constraint checking (unsafe for constrained types)
let p: Percent = 150u8 as Percent      // undefined behavior in safe code!
// Constrained types require try_into() or explicit unsafe
```

**Conversion method summary:**

```ferrum
// For any numeric conversion:
value as T                    // defined behavior per matrix above
value.try_into(): Result[T]   // Err if not exactly representable
value.saturating_into(): T    // clamp to T.MIN..=T.MAX
value.wrapping_into(): T      // modular arithmetic (integers only)

// Additional float → integer methods:
value.floor() as i32          // round toward -∞, then convert
value.ceil() as i32           // round toward +∞, then convert
value.round() as i32          // round to nearest, ties away from zero
value.round_ties_even() as i32 // round to nearest, ties to even (banker's)
```

> **Implementation Warning: Numeric Conversion Edge Cases**
>
> - `NaN as i32` produces `0`. Check `is_nan()` before converting if you need to detect this.
> - `0.1d64 as f64` produces the binary approximation — decimal exactness is lost.
> - Float to integer saturates on overflow — not UB like C, but silent.
> - `f128 as f16` NaN payload — only high bits survive; payload may change.
> - Subnormal floats may flush to zero on some hardware after conversion.

#### 1.2.10 Hardware Availability

| Type | x86-64 | ARM64 | RISC-V64 | IBM POWER |
|------|--------|-------|----------|-----------|
| `u8`–`u64`, `i8`–`i64` | native | native | native | native |
| `u128`, `i128` | partial | partial | partial | native |
| `u256`+ | SIMD | SIMD | soft | soft |
| `f16` | AVX-512 FP16 | v8.2+ | Zfh | soft |
| `bf16` | AMX | v8.6+ | Zfbfmin | soft |
| `f80` | x87 | soft | soft | soft |
| `f128` | soft | soft | soft | native |
| `d32`–`d128` | soft | soft | soft | native |

To prohibit soft-float (for real-time code): `@no_soft_float`
To require native arithmetic: `@require_hardware_float`

### 1.3 Constrained Types

```ferrum
// Constrained integers
type Percent  = u8  where value <= 100
type Port     = u16 where value >= 1024
type NonZero  = u32 where value != 0
type BcdDigit = u8  where value <= 9

// Compound constraints
type SafeIndex[N: usize] = usize where value < N

// Constrained large integers
type Secp256k1Scalar = u256
    where value < 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141u256

// Float constraints
type UnitInterval  = f64  where value >= 0.0 && value <= 1.0
type PositiveFloat = f32  where value > 0.0 && !value.is_nan()
type FiniteF128    = f128 where value.is_finite()

// Decimal constraints
type NonNegativeDecimal = d64 where value >= 0.0d64

// Fixed-point constraints
type SafeQ16_16 = Q16_16 where value >= -1000.0 and value <= 1000.0

// Constraints are checked:
// - statically when the value is a constant
// - by the SMT solver in proof mode
// - as a runtime assertion in debug mode
// - elided in release mode (unless annotated `safe`)
```

Constraint language supports: `==`, `!=`, `<`, `<=`, `>`, `>=`, `&&`, `||`,
`!`, arithmetic expressions over `value` and constants, method calls like
`is_nan()` and `is_finite()`, references to other constrained type parameters.

### 1.4 Compound Types

**Tuple types:**
```ferrum
let pair: (i32, str) = (1, "hello")
let unit: ()         = ()
pair.0    // field access by index
pair.1
```

**Array types (fixed size, size in type):**
```ferrum
let arr: [u8; 16] = [0; 16]    // 16 zeros
let arr: [u8; 4]  = [1, 2, 3, 4]
arr[0]    // bounds checked in debug, unchecked in release unless annotated
```

**Slice types (unsized, always behind a reference):**
```ferrum
let s: &[u8] = &arr[1..3]
let s: &mut [u8] = &mut arr[..]
```

**String types:**
```ferrum
&str      // immutable UTF-8 string slice (unsized, always a reference)
String    // owned UTF-8 string (heap-allocated by default, allocator-parameterized)
```

### 1.5 Pointer Types

```ferrum
&T           // shared reference — immutable, non-null
&mut T       // exclusive reference — mutable, non-null
*const T     // raw const pointer — nullable, unsafe to dereference
*mut T       // raw mutable pointer — nullable, unsafe to dereference
```

References are never null. Raw pointers are only accessible in `unsafe` or `trusted` blocks.

### 1.6 Function Types

```ferrum
fn(i32, i32): i32          // function pointer
fn(i32): i32 ! IO          // function pointer with effect
fn[T: Clone](T): T         // generic function pointer (rare)
```

Closures have anonymous types that implement one of:
```ferrum
trait Fn[Args, Out]      // can be called multiple times, immutable capture
trait FnMut[Args, Out]   // can be called multiple times, mutable capture
trait FnOnce[Args, Out]  // can be called once, consumes capture
```

### 1.7 The `never` Type

`never` is the bottom type. A value of type `never` can never exist. Expressions that never return (infinite loops, `panic`, `return`, `break`, `continue`) have type `never`. `never` coerces to any type.

```ferrum
fn diverge(): never {
    loop {}
}

let x: i32 = if condition { 42 } else { diverge() }  // valid: never coerces to i32
```

---
