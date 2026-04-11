# Ferrum Standard Library — Arbitrary Precision Numerics

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## Design Rationale

Ferrum provides arbitrary precision numerics as first-class library types:

1. **`BigDecimal`** — Exact decimal arithmetic with configurable precision
2. **`ArithmeticContext`** — Explicit control over rounding, precision, and error handling
3. **`Rational`** — Exact rational arithmetic (always in lowest terms)
4. **`Complex[T]`** — Complex numbers parameterized on the scalar type
5. **Numeric tower traits** — `Zero`, `One`, `Num`, `Signed`, `Integral`, `Real`

Key design decisions:
- Context is explicit via `given`, not thread-local state
- Exceptional conditions become `Result` types, not panics
- No implicit coercions between numeric types
- All heap-allocated types use explicit allocators

---

## 1. alloc.bigint — Arbitrary Precision Integers

```ferrum
type BigInt  given [A: Allocator] {
    // Stored as sign + magnitude (Vec[u64] limbs, little-endian)
}

impl BigInt {
    // Construction
    fn zero(): Self
    fn one(): Self
    fn from_i64(n: i64): Self
    fn from_u64(n: u64): Self
    fn from_i128(n: i128): Self
    fn from_u128(n: u128): Self
    fn from_bytes_be(bytes: &[u8], signed: bool): Self
    fn from_bytes_le(bytes: &[u8], signed: bool): Self

    /// Parses a string in the given radix (2-36).
    fn parse(s: &str, radix: u32): Result[Self, ParseBigIntError]
        requires radix >= 2 and radix <= 36

    /// Parses decimal string.
    fn from_str(s: &str): Result[Self, ParseBigIntError]

    // Properties
    fn is_zero(&self): bool
    fn is_positive(&self): bool
    fn is_negative(&self): bool
    fn sign(&self): i8    // -1, 0, or 1
    fn bits(&self): u64   // number of bits (excluding leading zeros)
    fn bit(&self, n: u64): bool   // nth bit

    // Conversion
    fn to_i64(&self): Option[i64]
    fn to_u64(&self): Option[u64]
    fn to_i128(&self): Option[i128]
    fn to_u128(&self): Option[u128]
    fn to_f64(&self): f64    // may lose precision
    fn to_bytes_be(&self): Vec[u8]
    fn to_bytes_le(&self): Vec[u8]
    fn to_string_radix(&self, radix: u32): String
        requires radix >= 2 and radix <= 36

    // Arithmetic (operators also implemented)
    fn abs(&self): Self
    fn neg(&self): Self
    fn add(&self, other: &Self): Self
    fn sub(&self, other: &Self): Self
    fn mul(&self, other: &Self): Self
    fn div(&self, other: &Self): Self
        requires !other.is_zero()
    fn rem(&self, other: &Self): Self
        requires !other.is_zero()
    fn div_rem(&self, other: &Self): (Self, Self)
        requires !other.is_zero()

    /// Division rounding toward negative infinity (Python-style).
    fn div_floor(&self, other: &Self): Self
        requires !other.is_zero()

    /// Modulo with result sign matching divisor (Python-style).
    fn mod_floor(&self, other: &Self): Self
        requires !other.is_zero()

    /// Division rounding toward zero (C-style).
    fn div_trunc(&self, other: &Self): Self
        requires !other.is_zero()

    /// Division rounding toward positive infinity.
    fn div_ceil(&self, other: &Self): Self
        requires !other.is_zero()

    fn pow(&self, exp: u32): Self
    fn pow_mod(&self, exp: &Self, modulus: &Self): Self
        requires !modulus.is_zero()

    // Bitwise
    fn and(&self, other: &Self): Self
    fn or(&self, other: &Self): Self
    fn xor(&self, other: &Self): Self
    fn not(&self): Self    // two's complement
    fn shl(&self, n: u64): Self
    fn shr(&self, n: u64): Self

    // Number theory
    fn gcd(&self, other: &Self): Self
    fn lcm(&self, other: &Self): Self
    fn extended_gcd(&self, other: &Self): (Self, Self, Self)
        // Returns (gcd, x, y) where self*x + other*y = gcd
    fn mod_inverse(&self, modulus: &Self): Option[Self]
        // Returns x where self*x ≡ 1 (mod modulus), if it exists
    fn is_prime(&self, certainty: u32): bool
        // Miller-Rabin with `certainty` rounds
    fn next_prime(&self): Self
    fn sqrt(&self): Self   // integer square root (floor)
    fn nth_root(&self, n: u32): Self

    // Comparison
    fn cmp(&self, other: &Self): Ordering
    fn cmp_abs(&self, other: &Self): Ordering
}

// Operators
impl Add for BigInt { type Output = BigInt; fn add(self, rhs: Self): Self }
impl Sub for BigInt { type Output = BigInt; fn sub(self, rhs: Self): Self }
impl Mul for BigInt { type Output = BigInt; fn mul(self, rhs: Self): Self }
impl Div for BigInt { type Output = BigInt; fn div(self, rhs: Self): Self }
impl Rem for BigInt { type Output = BigInt; fn rem(self, rhs: Self): Self }
impl Neg for BigInt { type Output = BigInt; fn neg(self): Self }
impl BitAnd for BigInt { /* ... */ }
impl BitOr for BigInt { /* ... */ }
impl BitXor for BigInt { /* ... */ }
impl Not for BigInt { /* ... */ }
impl Shl[u64] for BigInt { /* ... */ }
impl Shr[u64] for BigInt { /* ... */ }
impl Eq for BigInt
impl Ord for BigInt
impl Hash for BigInt
impl Display for BigInt
impl Debug for BigInt
impl Clone for BigInt
impl Default for BigInt  // zero

// BigUint — unsigned arbitrary precision integer
type BigUint  given [A: Allocator] {
    // Stored as Vec[u64] limbs, little-endian
}

// Same API as BigInt except:
// - No sign-related methods
// - Division/modulo always positive
// - No two's complement bitwise not
impl BigUint {
    // All unsigned-appropriate methods from BigInt
    // ...
}
```

---

## 2. alloc.bigdecimal — Arbitrary Precision Decimal

Implements the IEEE 754-2008 and General Decimal Arithmetic specifications.

### 2.1 ArithmeticContext

```ferrum
/// Controls precision, rounding, and error handling for decimal arithmetic.
/// Used via `given` to provide ambient context.
type ArithmeticContext {
    /// Maximum number of significant digits.
    /// Operations round to this precision.
    prec: u32,

    /// Rounding mode for inexact results.
    rounding: RoundingMode,

    /// Minimum exponent (for underflow detection).
    e_min: i32,

    /// Maximum exponent (for overflow detection).
    e_max: i32,

    /// How to handle exceptional conditions.
    traps: SignalFlags,
}

impl ArithmeticContext {
    /// Default context: 28 digits precision, half-even rounding.
    /// Matches Python's decimal.DefaultContext.
    const DEFAULT: Self = ArithmeticContext {
        prec: 28,
        rounding: RoundingMode.HalfEven,
        e_min: -999_999,
        e_max: 999_999,
        traps: SignalFlags.DEFAULT_TRAPS,
    }

    /// High precision context for scientific computing.
    const EXTENDED: Self = ArithmeticContext {
        prec: 50,
        rounding: RoundingMode.HalfEven,
        e_min: -999_999,
        e_max: 999_999,
        traps: SignalFlags.empty(),
    }

    /// Financial context: 2 decimal places, half-up rounding.
    fn financial(decimal_places: u32): Self {
        ArithmeticContext {
            prec: decimal_places + 15,  // room for intermediate results
            rounding: RoundingMode.HalfUp,
            e_min: -(decimal_places as i32),
            e_max: 999_999,
            traps: SignalFlags.OVERFLOW | SignalFlags.INVALID_OPERATION,
        }
    }

    fn with_prec(self, prec: u32): Self
    fn with_rounding(self, mode: RoundingMode): Self
    fn with_traps(self, traps: SignalFlags): Self
}

/// Rounding modes (IEEE 754-2008 + Python decimal).
enum RoundingMode {
    /// Round toward positive infinity.
    Ceiling,

    /// Round toward zero (truncate).
    Down,

    /// Round toward negative infinity.
    Floor,

    /// Round to nearest, ties go toward zero.
    HalfDown,

    /// Round to nearest, ties go to even (banker's rounding).
    /// DEFAULT — minimizes cumulative rounding error.
    HalfEven,

    /// Round to nearest, ties go away from zero.
    HalfUp,

    /// Round away from zero.
    Up,

    /// Round away from zero if last digit is 0 or 5, otherwise toward zero.
    ZeroFiveUp,
}

/// Exceptional conditions that can occur during decimal arithmetic.
bitflags type SignalFlags: u16 {
    const CLAMPED           = 0x0001
    const DIVISION_BY_ZERO  = 0x0002
    const INEXACT           = 0x0004
    const INVALID_OPERATION = 0x0008
    const OVERFLOW          = 0x0010
    const ROUNDED           = 0x0020
    const SUBNORMAL         = 0x0040
    const UNDERFLOW         = 0x0080

    /// Default traps: division by zero and invalid operation.
    const DEFAULT_TRAPS = Self.DIVISION_BY_ZERO | Self.INVALID_OPERATION
}
```

### 2.2 BigDecimal

```ferrum
/// Arbitrary precision decimal floating-point number.
/// Stored as coefficient × 10^exponent where coefficient is a BigInt.
type BigDecimal  given [A: Allocator] {
    // Internal: sign, coefficient (BigUint), exponent (i32)
    // Special values: Infinity, -Infinity, NaN, sNaN (signaling NaN)
}

impl BigDecimal {
    // Construction
    fn zero(): Self
    fn one(): Self
    fn from_i64(n: i64): Self
    fn from_u64(n: u64): Self
    fn from_bigint(n: BigInt): Self

    /// Parse a decimal string.
    /// Accepts: "123.456", "-0.001", "1.23E+10", "Infinity", "NaN"
    fn parse(s: &str): Result[Self, ParseDecimalError]

    fn from_str(s: &str): Result[Self, ParseDecimalError]

    /// Convert from f64. May be inexact.
    fn from_f64(f: f64): Self  given [ctx: ArithmeticContext]

    /// Create from coefficient and exponent: coef × 10^exp.
    fn from_parts(coefficient: BigInt, exponent: i32): Self

    // Special values
    fn infinity(): Self
    fn neg_infinity(): Self
    fn nan(): Self
    fn signaling_nan(): Self               // signaling NaN
    fn is_finite(&self): bool
    fn is_infinite(&self): bool
    fn is_nan(&self): bool
    fn is_signaling_nan(&self): bool       // true for sNaN
    fn is_zero(&self): bool
    fn is_positive(&self): bool
    fn is_negative(&self): bool
    fn is_signed(&self): bool              // true if negative (including -0)
    fn is_subnormal(&self): bool  given [ctx: ArithmeticContext]

    // Components
    fn sign(&self): i8            // -1, 0, or 1
    fn coefficient(&self): &BigInt
    fn exponent(&self): i32
    fn adjusted(&self): i32       // exponent + (digits - 1)

    // Precision and scale
    fn precision(&self): u32      // number of significant digits
    fn scale(&self): i32          // -exponent (decimal places after point)

    /// Round to n significant digits using context rounding mode.
    fn round_to_prec(&self, n: u32): Self  given [ctx: ArithmeticContext]

    /// Round to n decimal places after the point.
    fn round_to_scale(&self, n: i32): Self  given [ctx: ArithmeticContext]

    /// Quantize: round to match the exponent of `other`.
    fn quantize(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    // Arithmetic — all use ambient context
    fn add(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn sub(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn mul(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn div(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn rem(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn div_rem(&self, other: &Self): Result[(Self, Self), DecimalError]
        given [ctx: ArithmeticContext]

    /// Integer division (floor).
    fn div_int(&self, other: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn abs(&self): Self
    fn neg(&self): Self
    fn copy_sign(&self, other: &Self): Self

    // Powers and roots
    fn pow(&self, exp: &Self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn sqrt(&self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn ln(&self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn log10(&self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    fn exp(&self): Result[Self, DecimalError]
        given [ctx: ArithmeticContext]

    // Comparison
    fn cmp(&self, other: &Self): Option[Ordering]   // None if either is NaN
    fn total_cmp(&self, other: &Self): Ordering     // total order including NaN
    fn same_quantum(&self, other: &Self): bool      // same exponent

    // Conversion
    fn to_f64(&self): f64
    fn to_bigint(&self): Option[BigInt]   // None if fractional or too large
    fn to_string(&self): String
    fn to_eng_string(&self): String       // engineering notation (exp multiple of 3)

    // Normalization
    fn normalize(&self): Self             // remove trailing zeros
    fn reduce(&self): Self                // alias for normalize
}

/// Error type for decimal operations.
enum DecimalError {
    DivisionByZero,
    InvalidOperation(String),
    Overflow,
    Underflow,
    Inexact,
    Clamped,
}

impl Error for DecimalError

// Operators use ambient context
impl Add for BigDecimal  given [ctx: ArithmeticContext] {
    type Output = Result[BigDecimal, DecimalError]
    fn add(self, rhs: Self): Self.Output
}
// Similar for Sub, Mul, Div, Rem, Neg

impl Eq for BigDecimal     // NaN != NaN
impl PartialOrd for BigDecimal
impl Hash for BigDecimal
impl Display for BigDecimal
impl Debug for BigDecimal
impl Clone for BigDecimal
impl Default for BigDecimal  // zero
```

### 2.3 Usage Examples

```ferrum
// Basic usage with default context
fn calculate_tax(price: BigDecimal): Result[BigDecimal, DecimalError]
    given [ctx: ArithmeticContext]
{
    let tax_rate = BigDecimal.parse("0.0825")?
    price.mul(&tax_rate)
}

// Using explicit context
fn financial_calculation() ! IO {
    let ctx = ArithmeticContext.financial(2)  // 2 decimal places

    using ctx {
        let price = BigDecimal.parse("19.99")?
        let qty = BigDecimal.from_i64(3)
        let subtotal = price.mul(&qty)?       // uses ctx
        let tax = calculate_tax(subtotal)?    // uses ctx
        let total = subtotal.add(&tax)?

        // Round to cents for display
        let rounded = total.round_to_scale(2)
        println!("Total: ${}", rounded)
    }
}

// High precision scientific computation
fn compute_pi(digits: u32): BigDecimal  given [A: Allocator] {
    let ctx = ArithmeticContext {
        prec: digits + 10,  // extra precision for intermediate
        rounding: RoundingMode.HalfEven,
        ..ArithmeticContext.DEFAULT
    }

    using ctx {
        // Chudnovsky algorithm or similar...
        // ...
    }
}

// Exact decimal arithmetic — no binary float errors
fn test_exact() {
    using ArithmeticContext.DEFAULT {
        let a = BigDecimal.parse("0.1").unwrap()
        let b = BigDecimal.parse("0.2").unwrap()
        let c = BigDecimal.parse("0.3").unwrap()

        assert_eq!(a.add(&b).unwrap(), c)  // true! (unlike 0.1f64 + 0.2f64)
    }
}
```

---

## 3. alloc.rational — Exact Rational Arithmetic

```ferrum
/// Exact rational number as numerator/denominator.
/// Always stored in lowest terms (reduced by GCD).
/// Denominator is always positive.
type Rational  given [A: Allocator] {
    // Internal: numerator (BigInt), denominator (BigUint, always > 0)
}

impl Rational {
    // Construction
    fn zero(): Self
    fn one(): Self

    /// Create from numerator and denominator.
    /// Automatically reduces to lowest terms.
    fn new(numer: impl Into[BigInt], denom: impl Into[BigInt]): Result[Self, RationalError]
        requires denom != 0

    /// Create from integer.
    fn from_integer(n: impl Into[BigInt]): Self

    /// Parse a fraction string: "1/3", "-22/7", "42"
    fn parse(s: &str): Result[Self, ParseRationalError]

    /// Best rational approximation of a float within tolerance.
    fn from_f64_approx(f: f64, max_denominator: u64): Self

    /// Exact conversion from f64 (may have large denominator).
    fn from_f64_exact(f: f64): Option[Self]

    // Components
    fn numerator(&self): &BigInt
    fn denominator(&self): &BigUint

    fn is_zero(&self): bool
    fn is_positive(&self): bool
    fn is_negative(&self): bool
    fn is_integer(&self): bool    // denominator == 1
    fn sign(&self): i8

    // Arithmetic — all exact, no rounding ever
    fn add(&self, other: &Self): Self
    fn sub(&self, other: &Self): Self
    fn mul(&self, other: &Self): Self
    fn div(&self, other: &Self): Result[Self, RationalError]
        requires !other.is_zero()

    fn abs(&self): Self
    fn neg(&self): Self
    fn recip(&self): Result[Self, RationalError]
        requires !self.is_zero()

    fn pow(&self, exp: i32): Result[Self, RationalError]

    // Rounding (returns BigInt)
    fn floor(&self): BigInt
    fn ceil(&self): BigInt
    fn trunc(&self): BigInt
    fn round(&self): BigInt   // round half away from zero

    // Conversion
    fn to_f64(&self): f64     // may lose precision
    fn to_bigint(&self): Option[BigInt]   // Some if is_integer

    /// Find closest rational with denominator <= max_denom.
    /// Uses continued fraction algorithm.
    fn limit_denominator(&self, max_denom: u64): Self

    // Comparison
    fn cmp(&self, other: &Self): Ordering

    /// Compare with a float (exact comparison).
    fn cmp_f64(&self, f: f64): Ordering
}

// Operators
impl Add for Rational { type Output = Rational; fn add(self, rhs: Self): Self }
impl Sub for Rational { type Output = Rational; fn sub(self, rhs: Self): Self }
impl Mul for Rational { type Output = Rational; fn mul(self, rhs: Self): Self }
impl Div for Rational { type Output = Result[Rational, RationalError]; fn div(self, rhs: Self): Self.Output }
impl Neg for Rational { type Output = Rational; fn neg(self): Self }

impl Eq for Rational
impl Ord for Rational
impl Hash for Rational
impl Display for Rational   // "3/4" or "5" if integer
impl Debug for Rational
impl Clone for Rational
impl Default for Rational   // zero

enum RationalError {
    DivisionByZero,
}

impl Error for RationalError
```

### 3.1 Usage Examples

```ferrum
// Exact arithmetic where floats fail
fn harmonic_series(n: u32): Rational  given [A: Allocator] {
    let mut sum = Rational.zero()
    for i in 1..=n {
        let term = Rational.new(1, i).unwrap()
        sum = sum.add(&term)
    }
    sum
}

// H_100 exactly
let h100 = harmonic_series(100)
println!("H_100 = {}", h100)  // exact fraction

// Convert to decimal with arbitrary precision
let decimal = h100.to_f64()  // approximate

// Best rational approximation of pi
let pi_approx = Rational.from_f64_approx(f64.PI, 1000)
println!("{}", pi_approx)  // "355/113" (Milü)

// Limit denominator for display
let third = Rational.new(1, 3).unwrap()
let repeated = third.mul(&Rational.from_integer(7))
let simplified = repeated.limit_denominator(10)
```

---

## 4. core.complex — Complex Numbers

Complex numbers are generic over the scalar type. When `T` is a primitive
like `f64`, `Complex[T]` is stack-allocated and `Copy`.

```ferrum
/// Complex number: real + imag*i
type Complex[T: Num] {
    real: T,
    imag: T,
}

impl[T: Num] Complex[T] {
    fn new(real: T, imag: T): Self
    fn from_real(real: T): Self   // imag = 0
    fn from_imag(imag: T): Self   // real = 0
    fn zero(): Self  where T: Zero
    fn one(): Self   where T: One
    fn i(): Self     where T: Zero + One   // 0 + 1i

    fn real(&self): T
    fn imag(&self): T

    fn conj(&self): Self   // complex conjugate: a - bi

    /// Squared magnitude: |z|² = a² + b²
    fn norm_sqr(&self): T

    /// Magnitude: |z| = sqrt(a² + b²)
    fn norm(&self): T  where T: Float

    /// Phase angle in radians: atan2(imag, real)
    fn arg(&self): T  where T: Float

    /// Convert to polar form: (magnitude, phase)
    fn to_polar(&self): (T, T)  where T: Float

    /// Create from polar form: r * e^(iθ)
    fn from_polar(r: T, theta: T): Self  where T: Float

    // Arithmetic
    fn add(&self, other: &Self): Self
    fn sub(&self, other: &Self): Self
    fn mul(&self, other: &Self): Self
    fn div(&self, other: &Self): Self  where T: Real  // needs T division, not sqrt
    fn neg(&self): Self

    /// Multiplicative inverse: 1/z = conj(z)/|z|²
    fn inv(&self): Self  where T: Real  // needs T division, not sqrt

    /// Scale by real number
    fn scale(&self, s: T): Self

    // Powers and functions (require Float)
    fn sqrt(&self): Self  where T: Float
    fn exp(&self): Self   where T: Float
    fn ln(&self): Self    where T: Float
    fn pow(&self, exp: &Self): Self  where T: Float
    fn powf(&self, exp: T): Self     where T: Float
    fn powi(&self, exp: i32): Self   where T: Float

    // Trigonometric
    fn sin(&self): Self   where T: Float
    fn cos(&self): Self   where T: Float
    fn tan(&self): Self   where T: Float
    fn sinh(&self): Self  where T: Float
    fn cosh(&self): Self  where T: Float
    fn tanh(&self): Self  where T: Float

    // Inverse trig
    fn asin(&self): Self  where T: Float
    fn acos(&self): Self  where T: Float
    fn atan(&self): Self  where T: Float

    /// Check if approximately equal within epsilon.
    fn approx_eq(&self, other: &Self, epsilon: T): bool
        where T: Float
}

// Type aliases for common cases
type Complex32 = Complex[f32]
type Complex64 = Complex[f64]

// Works with BigDecimal too (allocates)
type ComplexDecimal = Complex[BigDecimal]  given [A: Allocator]

// Operators
impl[T: Num] Add for Complex[T] { type Output = Self; fn add(self, rhs: Self): Self }
impl[T: Num] Sub for Complex[T] { type Output = Self; fn sub(self, rhs: Self): Self }
impl[T: Num] Mul for Complex[T] { type Output = Self; fn mul(self, rhs: Self): Self }
impl[T: Real] Div for Complex[T] { type Output = Self; fn div(self, rhs: Self): Self }
impl[T: Num] Neg for Complex[T] { type Output = Self; fn neg(self): Self }

impl[T: Num + Eq] Eq for Complex[T]
impl[T: Num + Hash] Hash for Complex[T]
impl[T: Num + Display] Display for Complex[T]   // "3+4i" or "3-4i"
impl[T: Num + Debug] Debug for Complex[T]
impl[T: Copy] Copy for Complex[T]
impl[T: Clone] Clone for Complex[T]
impl[T: Zero] Default for Complex[T]
```

### 4.1 Usage Examples

```ferrum
// Basic complex arithmetic
let z1 = Complex64.new(3.0, 4.0)   // 3 + 4i
let z2 = Complex64.new(1.0, 2.0)   // 1 + 2i

let sum = z1 + z2                   // 4 + 6i
let product = z1 * z2               // (3*1 - 4*2) + (3*2 + 4*1)i = -5 + 10i
let quotient = z1 / z2              // (3+4i)/(1+2i)

println!("z1 = {}", z1)             // "3+4i"
println!("|z1| = {}", z1.norm())    // 5.0
println!("arg(z1) = {}", z1.arg())  // ~0.927 radians

// Euler's identity: e^(iπ) + 1 = 0
let i_pi = Complex64.from_imag(f64.PI)
let result = i_pi.exp() + Complex64.one()
assert!(result.norm() < 1e-15)  // approximately zero

// Roots of unity
fn nth_roots_of_unity(n: u32): Vec[Complex64] {
    (0..n).map(|k| {
        let angle = 2.0 * f64.PI * (k as f64) / (n as f64)
        Complex64.from_polar(1.0, angle)
    }).collect()
}

// Mandelbrot iteration
fn mandelbrot_escape(c: Complex64, max_iter: u32): Option[u32] {
    let mut z = Complex64.zero()
    for i in 0..max_iter {
        if z.norm_sqr() > 4.0 {
            return Some(i)
        }
        z = z * z + c
    }
    None
}
```

---

## 5. Numeric Tower Traits (in alloc.num)

The numeric tower provides trait bounds for generic numeric code that works
across both primitive types and arbitrary-precision types.

**Relationship to language primitives (§3.2.6):**
- `core.Integer` — fixed-width integers (`BITS`, `MIN`, `MAX`, `count_ones`)
- `core.Float` — IEEE 754 floats (`NAN`, `INFINITY`, `is_nan`, `sqrt`, `sin`, `exp`)

The numeric tower adds traits for concepts the language doesn't cover:

```ferrum
// In alloc.num — only traits that DON'T duplicate language primitives

/// Type has an additive identity (0).
trait Zero {
    fn zero(): Self
    fn is_zero(&self): bool
}

/// Type has a multiplicative identity (1).
trait One {
    fn one(): Self
    fn is_one(&self): bool
}

/// Marker trait for numeric types that support basic arithmetic.
/// Like core.Numeric but doesn't require Copy — works with BigInt.
trait Num: Eq + Zero + One + Add[Output=Self] + Sub[Output=Self] + Mul[Output=Self] {}

/// Signed numeric type (has negation and absolute value).
trait Signed: Num + Neg[Output=Self] {
    fn abs(&self): Self
    fn signum(&self): Self   // -1, 0, or 1
    fn is_positive(&self): bool
    fn is_negative(&self): bool
}

/// Mathematical integers (integral domain).
/// Distinct from core.Integer which has BITS/MIN/MAX for fixed-width types.
/// Both i32 and BigInt implement this; BigInt can't implement core.Integer.
trait Integral: Num + Ord + Eq {
    fn div_floor(&self, other: &Self): Self
    fn mod_floor(&self, other: &Self): Self
    fn gcd(&self, other: &Self): Self
    fn lcm(&self, other: &Self): Self
    fn is_even(&self): bool
    fn is_odd(&self): bool
}

/// Real number (totally ordered, supports rounding).
/// Distinct from core.Float which has NaN (not totally ordered).
/// Rational implements Real but not Float.
trait Real: Num + Signed + Ord + Div[Output=Self] {
    fn floor(&self): Self
    fn ceil(&self): Self
    fn round(&self): Self
    fn trunc(&self): Self
}

// That's it. Six traits total: Zero, One, Num, Signed, Integral, Real.
//
// Removed as unnecessary:
// - Unsigned: empty marker, no use case for "must be unsigned" constraint
// - SignedIntegral/UnsignedIntegral: just aliases, use Integral + Signed directly
// - RationalField: only Rational implements it, use inherent methods
// - ComplexField: only Complex implements it, use inherent methods
//
// What implements what:
//
// | Type       | core.Integer | core.Float | Integral | Real |
// |------------|--------------|------------|----------|------|
// | i32, u64   | ✓            | -          | ✓        | -    |
// | f32, f64   | -            | ✓          | -        | ✓    |
// | d64, d128  | -            | ✓          | -        | ✓    |
// | BigInt     | -            | -          | ✓        | -    |
// | BigDecimal | -            | ✓          | -        | ✓    |
// | Rational   | -            | -          | -        | ✓    |
// | Fixed[N,M] | -            | -          | -        | ✓    |
```

### 5.1 Generic Numeric Functions

```ferrum
/// Sum a collection of any numeric type.
fn sum[T: Num + Clone, I: Iterator[Item=T]](iter: I): T {
    let mut acc = T.zero()
    for x in iter {
        acc = acc + x
    }
    acc
}

/// Product of a collection.
fn product[T: Num + Clone, I: Iterator[Item=T]](iter: I): T {
    let mut acc = T.one()
    for x in iter {
        acc = acc * x
    }
    acc
}

/// Polynomial evaluation using Horner's method.
fn eval_poly[T: Num + Clone](coeffs: &[T], x: &T): T {
    let mut result = T.zero()
    for c in coeffs.iter().rev() {
        result = result * x.clone() + c.clone()
    }
    result
}

/// Dot product of two slices.
fn dot[T: Num + Clone](a: &[T], b: &[T]): T
    requires a.len() == b.len()
{
    a.iter().zip(b.iter())
        .map(|(x, y)| x.clone() * y.clone())
        .fold(T.zero(), |acc, x| acc + x)
}

/// Compute mean (requires division).
fn mean[T: Real + Clone](data: &[T]): Option[T] {
    if data.is_empty() {
        return None
    }
    let sum: T = data.iter().cloned().fold(T.zero(), |a, b| a + b)
    let n = T.from_usize(data.len())?
    Some(sum / n)
}
```

---

## 6. Implementation Notes

### 6.1 Performance Considerations

| Type | Memory | Arithmetic Speed | Use Case |
|------|--------|------------------|----------|
| Fixed-width (`d64`, `Q16_16`) | Fixed, stack | Hardware or 1 instruction sequence | Real-time, embedded, finance |
| `BigInt` | Heap, grows | O(n) for n-limb | Cryptography, number theory |
| `BigDecimal` | Heap, grows | O(n) for n-digit | Financial, scientific |
| `Rational` | Heap × 2 | O(n) per op, grows fast | Exact symbolic math |
| `Complex[f64]` | 16 bytes, stack | Hardware | Signal processing, physics |

### 6.2 When to Use What

| Scenario | Recommended Type |
|----------|------------------|
| Financial calculations | `BigDecimal` with `ArithmeticContext.financial(2)` |
| Scientific computing | `f64`, `Complex64`, or `BigDecimal` for extra precision |
| Cryptographic keys | `BigInt` / `BigUint` |
| Exact fractions | `Rational` |
| Signal processing | `Complex32` / `Complex64` |
| Game engine | `Fixed[16,16]` for determinism |
| Currency storage | `d64` or `BigDecimal` |

### 6.3 Interoperability

```ferrum
// All types can convert to/from strings
let bd = BigDecimal.parse("123.456")?
let rat = Rational.parse("22/7")?

// Conversion between types
let bd_from_rat = BigDecimal.from_rational(&rat, ctx)
let rat_from_bd = Rational.from_bigdecimal(&bd)?  // fails if repeating decimal

// Floating point conversions are explicit and may lose precision
let f = bd.to_f64()
let bd2 = BigDecimal.from_f64(f)  // round-trip may differ

// BigInt and Rational interoperate cleanly
let n: BigInt = BigInt.from_i64(42)
let r = Rational.from_integer(n)
```

---

*End of Arbitrary Precision Numerics — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
