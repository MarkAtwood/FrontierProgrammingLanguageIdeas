# Ferrum Standard Library — math, linalg, simd

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 16. math — Mathematics

### 16.1 Floating Point

```ferrum
// All float operations are explicit about what they compute.
// No surprising implicit conversions.

impl f32 {
    // Special value constants
    const NAN:          Self          // canonical quiet NaN
    const INFINITY:     Self          // +∞
    const NEG_INFINITY: Self          // -∞
    const ZERO:         Self = 0.0    // +0.0
    const NEG_ZERO:     Self = -0.0   // -0.0 (distinct bit pattern)
    const MAX:          Self          // largest finite positive
    const MIN:          Self          // smallest finite (most negative)
    const MIN_POSITIVE: Self          // smallest positive normal
    const EPSILON:      Self          // 1.0.next_up() - 1.0

    // Mathematical constants
    const PI:           Self
    const E:            Self
    const SQRT_2:       Self
    const LN_2:         Self
    const LN_10:        Self
    const LOG2_E:       Self
    const LOG10_E:      Self

    // Classification
    fn is_nan(&self): bool
    fn is_infinite(&self): bool
    fn is_finite(&self): bool
    fn is_normal(&self): bool
    fn is_sign_positive(&self): bool
    fn is_sign_negative(&self): bool
    fn is_subnormal(&self): bool
    fn classify(&self): FpCategory

    // Signaling NaN support
    fn signaling_nan(): Self              // create signaling NaN
    fn is_signaling_nan(&self): bool      // true for sNaN, false for qNaN
    fn nan_with_payload(payload: u32): Self   // NaN with specific payload
    fn nan_payload(&self): Option[u32]    // extract payload (None if not NaN)

    fn abs(&self): Self
    fn signum(&self): Self
    fn copysign(&self, sign: Self): Self
    fn floor(&self): Self
    fn ceil(&self): Self
    fn round(&self): Self
    fn round_ties_even(&self): Self   // banker's rounding
    fn trunc(&self): Self
    fn fract(&self): Self

    fn sqrt(&self): Self
    fn cbrt(&self): Self
    fn exp(&self): Self
    fn exp2(&self): Self
    fn ln(&self): Self
    fn log(&self, base: Self): Self
    fn log2(&self): Self
    fn log10(&self): Self
    fn pow(&self, exp: Self): Self
    fn powi(&self, exp: i32): Self
    fn hypot(&self, other: Self): Self

    fn sin(&self): Self
    fn cos(&self): Self
    fn tan(&self): Self
    fn asin(&self): Self
    fn acos(&self): Self
    fn atan(&self): Self
    fn atan2(&self, other: Self): Self
    fn sin_cos(&self): (Self, Self)  // returns both, more efficient
    fn sinh(&self): Self
    fn cosh(&self): Self
    fn tanh(&self): Self
    fn asinh(&self): Self
    fn acosh(&self): Self
    fn atanh(&self): Self

    fn min(&self, other: Self): Self   // NaN propagating
    fn max(&self, other: Self): Self
    fn clamp(&self, min: Self, max: Self): Self
    fn min_num(&self, other: Self): Self   // NaN non-propagating (IEEE 754-2008 minNum)
    fn max_num(&self, other: Self): Self

    fn mul_add(&self, a: Self, b: Self): Self  // fused multiply-add, one rounding
    fn recip(&self): Self    // 1.0 / self

    fn to_radians(&self): Self
    fn to_degrees(&self): Self

    fn to_bits(&self): u32
    fn from_bits(bits: u32): Self
    fn to_f64(&self): f64
    fn from_f64(v: f64): Self

    // Precision and ULP (Unit in Last Place)
    fn next_up(&self): Self       // next representable value toward +∞
    fn next_down(&self): Self     // next representable value toward -∞
    fn ulp(&self): Self           // magnitude of one ULP at this value

    fn total_cmp(&self, other: &Self): Ordering  // total order including NaN
}

// f64 has identical methods (with u64 for to_bits/from_bits/nan_payload)

enum FpCategory { Nan, Infinite, Zero, Subnormal, Normal }
```

### 16.1.1 ElementaryFloat Trait

A trait for elementary mathematical functions that works for any floating-point type. This enables generic numeric algorithms.

```ferrum
/// Elementary mathematical functions for floating-point types.
trait ElementaryFloat: Float {
    // Exponential and logarithmic
    fn exp(self): Self
    fn exp2(self): Self
    fn exp_m1(self): Self      // e^x - 1, accurate for small x
    fn ln(self): Self
    fn ln_1p(self): Self       // ln(1 + x), accurate for small x
    fn log(self, base: Self): Self
    fn log2(self): Self
    fn log10(self): Self

    // Power
    fn pow(self, exp: Self): Self
    fn powi(self, exp: i32): Self
    fn sqrt(self): Self
    fn cbrt(self): Self
    fn hypot(self, other: Self): Self

    // Trigonometric
    fn sin(self): Self
    fn cos(self): Self
    fn tan(self): Self
    fn sin_cos(self): (Self, Self)   // more efficient than separate calls
    fn asin(self): Self
    fn acos(self): Self
    fn atan(self): Self
    fn atan2(self, x: Self): Self

    // Hyperbolic
    fn sinh(self): Self
    fn cosh(self): Self
    fn tanh(self): Self
    fn asinh(self): Self
    fn acosh(self): Self
    fn atanh(self): Self

    // Rounding
    fn floor(self): Self
    fn ceil(self): Self
    fn round(self): Self
    fn trunc(self): Self
    fn fract(self): Self

    // Other
    fn abs(self): Self
    fn signum(self): Self
    fn copysign(self, sign: Self): Self
    fn mul_add(self, a: Self, b: Self): Self   // fused multiply-add

    // Constants (as associated consts)
    const PI: Self
    const TAU: Self           // 2π
    const E: Self
    const SQRT_2: Self
    const LN_2: Self
    const LN_10: Self
    const LOG2_E: Self
    const LOG10_E: Self
}

// Implemented for all float types
impl ElementaryFloat for f16 { /* hardware or soft-float */ }
impl ElementaryFloat for bf16 { /* via f32 */ }
impl ElementaryFloat for f32 { /* hardware */ }
impl ElementaryFloat for f64 { /* hardware */ }
impl ElementaryFloat for f80 { /* x87 or soft-float */ }
impl ElementaryFloat for f128 { /* POWER or soft-float */ }

// Also for decimal floats (with explicit precision control)
impl ElementaryFloat for d64  given [ctx: ArithmeticContext] { /* soft-float */ }
impl ElementaryFloat for d128 given [ctx: ArithmeticContext] { /* soft-float */ }

// And arbitrary precision
impl ElementaryFloat for BigDecimal given [ctx: ArithmeticContext, A: Allocator] { /* soft-float */ }

// Generic functions that work for any ElementaryFloat
fn distance[F: ElementaryFloat](x1: F, y1: F, x2: F, y2: F): F {
    let dx = x2 - x1
    let dy = y2 - y1
    dx.hypot(dy)
}

fn normalize_angle[F: ElementaryFloat](angle: F): F {
    let tau = F.TAU
    let mut result = angle % tau
    if result < F.zero() {
        result = result + tau
    }
    result
}

fn lerp[F: ElementaryFloat](a: F, b: F, t: F): F
    requires t >= F.zero() and t <= F.one()
{
    a + (b - a) * t
}

fn smoothstep[F: ElementaryFloat](edge0: F, edge1: F, x: F): F {
    let t = ((x - edge0) / (edge1 - edge0)).clamp(F.zero(), F.one())
    t * t * (F.from_i32(3) - F.from_i32(2) * t)
}
```

### 16.2 Integer Math

```ferrum
impl i32 {  // representative — same for all integer types
    const MAX: Self
    const MIN: Self
    const BITS: u32
    const BYTES: usize

    fn count_ones(&self): u32
    fn count_zeros(&self): u32
    fn leading_zeros(&self): u32
    fn trailing_zeros(&self): u32
    fn leading_ones(&self): u32
    fn trailing_ones(&self): u32
    fn rotate_left(&self, n: u32): Self
    fn rotate_right(&self, n: u32): Self
    fn swap_bytes(&self): Self
    fn reverse_bits(&self): Self
    fn to_be(&self): Self   // to big-endian byte order
    fn to_le(&self): Self   // to little-endian
    fn to_ne(&self): Self   // to native byte order
    fn from_be(x: Self): Self
    fn from_le(x: Self): Self
    fn from_ne(x: Self): Self
    fn to_be_bytes(&self): [u8; N]
    fn to_le_bytes(&self): [u8; N]
    fn to_ne_bytes(&self): [u8; N]
    fn from_be_bytes(bytes: [u8; N]): Self
    fn from_le_bytes(bytes: [u8; N]): Self
    fn from_ne_bytes(bytes: [u8; N]): Self
    fn pow(&self, exp: u32): Self    // panics on overflow in debug
    fn pow_wrapping(&self, exp: u32): Self
    fn pow_checked(&self, exp: u32): Option[Self]
    fn pow_saturating(&self, exp: u32): Self
    fn checked_add(&self, rhs: Self): Option[Self]
    fn checked_sub(&self, rhs: Self): Option[Self]
    fn checked_mul(&self, rhs: Self): Option[Self]
    fn checked_div(&self, rhs: Self): Option[Self]
    fn checked_rem(&self, rhs: Self): Option[Self]
    fn checked_neg(&self): Option[Self]
    fn checked_shl(&self, rhs: u32): Option[Self]
    fn checked_shr(&self, rhs: u32): Option[Self]
    fn saturating_add(&self, rhs: Self): Self
    fn saturating_sub(&self, rhs: Self): Self
    fn saturating_mul(&self, rhs: Self): Self
    fn wrapping_add(&self, rhs: Self): Self
    fn wrapping_sub(&self, rhs: Self): Self
    fn wrapping_mul(&self, rhs: Self): Self
    fn wrapping_div(&self, rhs: Self): Self
    fn overflowing_add(&self, rhs: Self): (Self, bool)
    fn overflowing_sub(&self, rhs: Self): (Self, bool)
    fn overflowing_mul(&self, rhs: Self): (Self, bool)
    fn isqrt(&self): Self   // integer square root, rounds down
    fn ilog2(&self): u32    // integer log2, panics on 0
    fn ilog10(&self): u32
    fn ilog(&self, base: Self): u32
    fn abs_diff(&self, other: Self): Self  // absolute difference, no overflow
    fn div_ceil(&self, rhs: Self): Self    // ceiling division
    fn div_floor(&self, rhs: Self): Self   // floor division (same as / for positive)
    fn next_multiple_of(&self, rhs: Self): Self
    fn is_power_of_two(&self): bool
    fn next_power_of_two(&self): Self
    fn prev_power_of_two(&self): Self
}

// GCD and LCM
fn gcd[T: Integer](a: T, b: T): T
fn lcm[T: Integer](a: T, b: T): T
fn extended_gcd[T: Integer](a: T, b: T): (T, T, T)  // (gcd, x, y) where ax + by = gcd
```

### 16.3 Statistics

```ferrum
// In math.stats — no allocation for slices, allocation for iterators

fn mean(data: &[f64]): Option[f64]
fn variance(data: &[f64]): Option[f64]
fn std_dev(data: &[f64]): Option[f64]
fn median(data: &mut [f64]): Option[f64]   // sorts data in-place
fn mode[T: Eq + Hash](data: &[T]): Option[T]  given [A: Allocator]
fn percentile(data: &mut [f64], p: f64 where p >= 0.0 and p <= 100.0): Option[f64]
fn sum(data: &[f64]): f64
fn product(data: &[f64]): f64
fn min_max(data: &[f64]): Option[(f64, f64)]
fn covariance(x: &[f64], y: &[f64]): Option[f64]
fn correlation(x: &[f64], y: &[f64]): Option[f64]

// Streaming statistics — O(1) memory
type RunningStats {
    fn new(): Self
    fn update(&mut self, x: f64)
    fn count(&self): u64
    fn mean(&self): f64
    fn variance(&self): f64
    fn std_dev(&self): f64
    fn min(&self): f64
    fn max(&self): f64
}
```

---

## 17. linalg — Linear Algebra and Matrices

### 17.1 Design

Fixed-size vectors and matrices are stack-allocated and SIMD-friendly. Dynamic matrices use the ambient allocator.

```ferrum
// Fixed-size vector — size in type, stack allocated
type Vec2  = Vector[f32, 2]
type Vec3  = Vector[f32, 3]
type Vec4  = Vector[f32, 4]
type IVec2 = Vector[i32, 2]
type DVec3 = Vector[f64, 3]

type Vector[T: Scalar, const N: usize]
    where N > 0

impl[T: Scalar, const N: usize] Vector[T, N] {
    fn new(data: [T; N]): Self
    fn zero(): Self
    fn one(): Self
    fn splat(v: T): Self

    fn x(&self): T   where N >= 1
    fn y(&self): T   where N >= 2
    fn z(&self): T   where N >= 3
    fn w(&self): T   where N >= 4

    fn dot(&self, other: &Self): T
    fn length_sq(&self): T
    fn length(&self): T     where T: Float
    fn normalize(&self): Self  where T: Float
        requires self.length_sq() != T.zero()
    fn distance(&self, other: &Self): T   where T: Float
    fn lerp(&self, other: &Self, t: T): Self   where T: Float
    fn abs(&self): Self
    fn min(&self, other: &Self): Self
    fn max(&self, other: &Self): Self
    fn clamp(&self, lo: &Self, hi: &Self): Self
    fn sum(&self): T
    fn product(&self): T
    fn map[U: Scalar](self, f: fn(T): U): Vector[U, N]
    fn zip_map(self, other: Self, f: fn(T, T): T): Self
    fn as_slice(&self): &[T]
    fn as_array(&self): &[T; N]
}

// Cross product (3D only)
fn cross(a: &Vec3, b: &Vec3): Vec3
fn cross_d(a: &DVec3, b: &DVec3): DVec3

// Swizzle — compile-time checked
// a.xy()  → Vec2    (from a Vec3 or Vec4)
// a.zyx() → Vec3    (reversed)
// a.xxyy() → Vec4
// All swizzle combinations generated by macro
```

### 17.2 Matrices (Fixed-size)

```ferrum
type Mat2  = Matrix[f32, 2, 2]
type Mat3  = Matrix[f32, 3, 3]
type Mat4  = Matrix[f32, 4, 4]
type DMat4 = Matrix[f64, 4, 4]

type Matrix[T: Scalar, const R: usize, const C: usize]
    where R > 0 and C > 0

impl[T: Scalar, const R: usize, const C: usize] Matrix[T, R, C] {
    fn new(data: [[T; C]; R]): Self    // row-major
    fn zero(): Self
    fn identity(): Self    where R == C   // square only

    fn row(&self, i: usize): Vector[T, C]
        requires i < R
    fn col(&self, j: usize): Vector[T, R]
        requires j < C
    fn get(&self, i: usize, j: usize): T
        requires i < R and j < C
    fn set(&mut self, i: usize, j: usize, val: T)
        requires i < R and j < C

    fn transpose(&self): Matrix[T, C, R]
    fn trace(&self): T   where R == C
    fn det(&self): T     where R == C   // compile-time specialization for 2x2, 3x3, 4x4
    fn inverse(&self): Option[Self]    where R == C and T: Float
    fn mul_vec(&self, v: &Vector[T, C]): Vector[T, R]
    fn map[U: Scalar](self, f: fn(T): U): Matrix[U, R, C]
}

// Ops: Mat * Mat, Mat * Vec, Mat + Mat, scalar * Mat

// Transforms (Mat4 specialization)
impl Mat4 {
    fn translation(v: Vec3): Self
    fn rotation_x(angle: f32): Self    // radians
    fn rotation_y(angle: f32): Self
    fn rotation_z(angle: f32): Self
    fn rotation_axis(axis: Vec3, angle: f32): Self
    fn scale(v: Vec3): Self
    fn perspective(fovy: f32, aspect: f32, near: f32, far: f32): Self
    fn orthographic(left: f32, right: f32, bottom: f32, top: f32, near: f32, far: f32): Self
    fn look_at(eye: Vec3, center: Vec3, up: Vec3): Self
}

// Quaternion
type Quat(f32, f32, f32, f32)   // (x, y, z, w)

impl Quat {
    fn identity(): Self
    fn from_axis_angle(axis: Vec3, angle: f32): Self
    fn from_euler(roll: f32, pitch: f32, yaw: f32): Self
    fn from_mat3(m: &Mat3): Self
    fn to_mat3(&self): Mat3
    fn to_mat4(&self): Mat4
    fn normalize(&self): Self
    fn conjugate(&self): Self
    fn inverse(&self): Self
    fn dot(&self, other: &Self): f32
    fn slerp(&self, other: &Self, t: f32): Self
    fn rotate_vec(&self, v: Vec3): Vec3
}
```

### 17.3 Dynamic Matrices

```ferrum
type DMatrix[T: Scalar]  given [A: Allocator]

impl[T: Scalar] DMatrix[T] {
    fn new(rows: usize, cols: usize): Self
    fn zeros(rows: usize, cols: usize): Self
    fn ones(rows: usize, cols: usize): Self
    fn identity(n: usize): Self
    fn from_vec(rows: usize, cols: usize, data: Vec[T]): Result[Self, ShapeError]
    fn from_fn(rows: usize, cols: usize, f: fn(usize, usize): T): Self

    fn rows(&self): usize
    fn cols(&self): usize
    fn shape(&self): (usize, usize)
    fn len(&self): usize    // rows * cols
    fn is_empty(&self): bool
    fn is_square(&self): bool

    fn get(&self, i: usize, j: usize): Option[&T]
    fn get_unchecked(&self, i: usize, j: usize): &T   // unchecked
    fn row(&self, i: usize): DVector[T]    // view (no copy)
    fn col(&self, j: usize): DVector[T]
    fn row_slice(&self, i: usize): &[T]
    fn slice(&self, rows: Range[usize], cols: Range[usize]): DMatrixView[T]

    fn transpose(&self): Self
    fn trace(&self): T    requires self.is_square()
    fn map[U: Scalar](&self, f: fn(&T): U): DMatrix[U]
    fn zip_map[U: Scalar](&self, other: &Self, f: fn(&T, &T): U): Result[DMatrix[U], ShapeError]

    // BLAS-level operations — dispatches to SIMD/hardware when available
    fn mul(&self, other: &Self): Result[Self, ShapeError]
    fn add(&self, other: &Self): Result[Self, ShapeError]
    fn scale(&self, s: T): Self
    fn dot(&self, other: &Self): Result[T, ShapeError]  // Frobenius inner product

    // Decompositions
    fn lu(&self): Result[LuDecomp[T], LinAlgError]
    fn qr(&self): Result[QrDecomp[T], LinAlgError]
    fn svd(&self): Result[SvdDecomp[T], LinAlgError]
    fn cholesky(&self): Result[CholeskyDecomp[T], LinAlgError]
    fn eigenvalues(&self): Result[Vec[Complex[T]], LinAlgError]
    fn eigenvectors(&self): Result[EigenDecomp[T], LinAlgError]

    // Solve Ax = b
    fn solve(&self, b: &DVector[T]): Result[DVector[T], LinAlgError]

    fn norm(&self, ord: MatrixNorm): T
    fn condition_number(&self): Result[T, LinAlgError]
    fn rank(&self, tol: T): usize

    fn reshape(&self, rows: usize, cols: usize): Result[Self, ShapeError]
    fn flatten(&self): DVector[T]
    fn stack_rows(&self, other: &Self): Result[Self, ShapeError]  // vertical stack
    fn stack_cols(&self, other: &Self): Result[Self, ShapeError]  // horizontal stack
}

type DVector[T: Scalar]  given [A: Allocator]
// 1D case of DMatrix, with vector-specific methods
impl[T: Scalar] DVector[T] {
    fn dot(&self, other: &Self): Result[T, ShapeError]
    fn cross(&self, other: &Self): Result[Self, ShapeError]   requires self.len() == 3
    fn length(&self): T    where T: Float
    fn normalize(&self): Self   where T: Float
    fn outer(&self, other: &Self): DMatrix[T]
}

enum MatrixNorm { Frobenius, L1, Linf, Nuclear, Spectral }
```

---

## 18. simd — Fast Math and Vectorization

### 18.1 Design Philosophy

SIMD in Ferrum is **explicit but not painful**. The programmer chooses when to use SIMD; the compiler uses it opportunistically in autovectorizable loops.

Three levels:
1. **Auto**: annotate a loop, compiler vectorizes.
2. **Portable SIMD**: width-typed lane vectors that compile to the best available instruction set.
3. **Platform intrinsics**: raw access to SSE/AVX/NEON/etc. (`unsafe`, in `std.sys.arch`).

### 18.2 Portable SIMD

```ferrum
// SIMD lane types — portable, width in type
type f32x4   // 4 f32 lanes, 128-bit
type f32x8   // 8 f32 lanes, 256-bit
type f32x16  // 16 f32 lanes, 512-bit
type f64x2
type f64x4
type f64x8
type i32x4
type i32x8
type i64x4
type u8x16
type u8x32
type u16x8
// ... all combinations of (i8 i16 i32 i64 u8 u16 u32 u64 f32 f64) x (2 4 8 16 32 64)
// availability of wide types depends on target; compiler emulates with multiple instructions

type Simd[T, const N: usize]   // generic form

impl[T: SimdElement, const N: usize] Simd[T, N] {
    fn splat(val: T): Self          // broadcast scalar to all lanes
    fn from_array(arr: [T; N]): Self
    fn to_array(self): [T; N]
    fn as_array(&self): &[T; N]

    // Arithmetic — lane-wise
    // Operator overloading: +, -, *, /, % work lane-wise
    fn abs(self): Self              where T: SignedSimdElement
    fn neg(self): Self              where T: SignedSimdElement
    fn min(self, other: Self): Self
    fn max(self, other: Self): Self
    fn clamp(self, lo: Self, hi: Self): Self

    // Float-specific lane ops
    fn sqrt(self): Self          where T: Float
    fn recip(self): Self         where T: Float  // 1.0 / x (approximate on some platforms)
    fn rsqrt(self): Self         where T: Float  // 1.0 / sqrt(x), approximate
    fn floor(self): Self         where T: Float
    fn ceil(self): Self          where T: Float
    fn round(self): Self         where T: Float
    fn trunc(self): Self         where T: Float
    fn mul_add(self, a: Self, b: Self): Self  where T: Float  // fused

    // Horizontal reductions
    fn reduce_sum(self): T
    fn reduce_product(self): T
    fn reduce_min(self): T
    fn reduce_max(self): T
    fn reduce_and(self): T       where T: Integer
    fn reduce_or(self): T        where T: Integer
    fn reduce_xor(self): T       where T: Integer

    // Masks and comparisons — return Mask type
    fn lanes_eq(self, other: Self): Mask[T, N]
    fn lanes_ne(self, other: Self): Mask[T, N]
    fn lanes_lt(self, other: Self): Mask[T, N]
    fn lanes_le(self, other: Self): Mask[T, N]
    fn lanes_gt(self, other: Self): Mask[T, N]
    fn lanes_ge(self, other: Self): Mask[T, N]

    // Gather/scatter — non-contiguous memory access
    unsafe fn gather(base: *const T, indices: Simd[usize, N]): Self
    unsafe fn scatter(self, base: *mut T, indices: Simd[usize, N])

    // Load/store
    fn load(slice: &[T]): Self
        requires slice.len() >= N
    fn store(self, slice: &mut [T])
        requires slice.len() >= N
    unsafe fn load_ptr(ptr: *const T): Self
    unsafe fn store_ptr(self, ptr: *mut T)

    // Shuffle/permute
    fn shuffle[const IDX: [usize; N]](self, other: Self): Self
    fn reverse(self): Self
    fn rotate_lanes_left[const OFFSET: usize](self): Self
    fn rotate_lanes_right[const OFFSET: usize](self): Self
    fn interleave(self, other: Self): (Self, Self)
    fn deinterleave(self, other: Self): (Self, Self)

    // Type conversion
    fn cast[U: SimdElement](self): Simd[U, N]   // lane-wise as/cast
}

type Mask[T, const N: usize]  // result of lane comparisons
impl[T, const N: usize] Mask[T, N] {
    fn select(self, true_val: Simd[T, N], false_val: Simd[T, N]): Simd[T, N]
    fn all(self): bool
    fn any(self): bool
    fn none(self): bool
    fn count(self): usize
    fn to_bitmask(self): u64
    fn from_bitmask(mask: u64): Self
}
```

### 18.3 Auto-vectorization

```ferrum
// Hint to the compiler that this loop should be vectorized
// Compiler emits a warning if it cannot vectorize
@vectorize
for i in 0..n {
    c[i] = a[i] + b[i]
}

// Assert that the loop WILL be vectorized — compile error if not
@must_vectorize
for i in 0..n {
    c[i] = a[i] * b[i]
}

// Specify the vector width
@vectorize(width = 8)
for i in 0..n {
    c[i] = a[i].sqrt()
}
```

### 18.4 BLAS-style Bulk Operations

```ferrum
// In math.blas — uses SIMD internally

// Level 1: vector-vector
fn axpy(alpha: f32, x: &[f32], y: &mut [f32])
    requires x.len() == y.len()
    // y = alpha * x + y, in-place
fn dot(x: &[f32], y: &[f32]): f32
    requires x.len() == y.len()
fn nrm2(x: &[f32]): f32   // Euclidean norm
fn asum(x: &[f32]): f32   // sum of absolute values
fn scal(alpha: f32, x: &mut [f32])   // x *= alpha
fn copy(x: &[f32], y: &mut [f32])
    requires x.len() == y.len()

// Level 2: matrix-vector
fn gemv(alpha: f32, a: &DMatrix[f32], x: &DVector[f32], beta: f32, y: &mut DVector[f32])
    // y = alpha * A * x + beta * y

// Level 3: matrix-matrix
fn gemm(
    alpha: f32,
    a: &DMatrix[f32],
    b: &DMatrix[f32],
    beta: f32,
    c: &mut DMatrix[f32],
)
    // C = alpha * A * B + beta * C
    requires a.cols() == b.rows()
    requires a.rows() == c.rows()
    requires b.cols() == c.cols()
```

### 18.5 Platform Intrinsics

Platform-specific SIMD lives in `std.sys.arch`. Unstable between targets. Use portable SIMD unless you need something specific.

```ferrum
import std.sys.arch.x86_64.*

unsafe fn fast_memcpy_avx2(dst: *mut u8, src: *const u8, len: usize) {
    // Direct AVX2 intrinsics
    let v = _mm256_loadu_si256(src as *const __m256i)
    _mm256_storeu_si256(dst as *mut __m256i, v)
}
```

---

*End of math, linalg, simd modules — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
