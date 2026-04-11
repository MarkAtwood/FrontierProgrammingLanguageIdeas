# Ferrum Standard Library — core

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

`core` is available everywhere: embedded, kernel, WASM, and hosted. No allocation. No OS.

---

## 3.0 core.num — Numeric Traits and Introspection

Type introspection traits for compile-time bounds and enumeration handling.

```ferrum
/// Trait for types with compile-time bounds.
trait Bounded {
    /// The minimum value representable by this type.
    const MIN: Self

    /// The maximum value representable by this type.
    const MAX: Self

    /// The number of distinct values (for discrete types).
    /// None for floats.
    const CARDINALITY: Option[usize]
}

// Implemented for all numeric types
impl Bounded for i32 {
    const MIN: Self = -2_147_483_648
    const MAX: Self = 2_147_483_647
    const CARDINALITY: Option[usize] = Some(4_294_967_296)
}

impl Bounded for f64 {
    const MIN: Self = f64.MIN
    const MAX: Self = f64.MAX
    const CARDINALITY: Option[usize] = None  // uncountable
}

// For constrained types, bounds reflect the constraint
type Percent = u8 where value <= 100

impl Bounded for Percent {
    const MIN: Self = 0
    const MAX: Self = 100
    const CARDINALITY: Option[usize] = Some(101)
}

/// Trait for discrete types with predecessor/successor operations.
trait Discrete: Bounded + Ord {
    /// Returns the predecessor, or None at MIN.
    fn pred(self): Option[Self]

    /// Returns the successor, or None at MAX.
    fn succ(self): Option[Self]

    /// Returns the position (0-indexed ordinal).
    fn pos(self): usize

    /// Returns the value at position n, or None if out of range.
    fn val(n: usize): Option[Self]
}

impl Discrete for i32 {
    fn pred(self): Option[Self] {
        if self == Self.MIN { None } else { Some(self - 1) }
    }
    fn succ(self): Option[Self] {
        if self == Self.MAX { None } else { Some(self + 1) }
    }
    fn pos(self): usize {
        (self as i64 - Self.MIN as i64) as usize
    }
    fn val(n: usize): Option[Self] {
        let v = Self.MIN as i64 + n as i64
        if v > Self.MAX as i64 { None } else { Some(v as Self) }
    }
}

// Enumerations implement Discrete automatically
enum Color { Red, Green, Blue }

// Compiler generates:
impl Bounded for Color {
    const MIN: Self = Color.Red
    const MAX: Self = Color.Blue
    const CARDINALITY: Option[usize] = Some(3)
}

impl Discrete for Color {
    fn pred(self): Option[Self] {
        match self {
            Color.Red => None,
            Color.Green => Some(Color.Red),
            Color.Blue => Some(Color.Green),
        }
    }
    fn succ(self): Option[Self] {
        match self {
            Color.Red => Some(Color.Green),
            Color.Green => Some(Color.Blue),
            Color.Blue => None,
        }
    }
    fn pos(self): usize {
        match self {
            Color.Red => 0,
            Color.Green => 1,
            Color.Blue => 2,
        }
    }
    fn val(n: usize): Option[Self] {
        match n {
            0 => Some(Color.Red),
            1 => Some(Color.Green),
            2 => Some(Color.Blue),
            _ => None,
        }
    }
}

/// Trait for enum name ↔ string conversion.
trait Named: Sized {
    /// Returns the name of the value as a string.
    fn name(&self): &'static str

    /// Parses a name to get the value.
    fn from_name(s: &str): Option[Self]
}

// Compiler generates for enums:
impl Named for Color {
    fn name(&self): &'static str {
        match self {
            Color.Red => "Red",
            Color.Green => "Green",
            Color.Blue => "Blue",
        }
    }
    fn from_name(s: &str): Option[Self] {
        match s {
            "Red" => Some(Color.Red),
            "Green" => Some(Color.Green),
            "Blue" => Some(Color.Blue),
            _ => None,
        }
    }
}

/// Trait for types that can have invalid bit patterns.
/// Critical for FFI: untrusted data may contain invalid values.
trait Validatable {
    /// Checks if the value is valid for its type.
    /// Always true for types without invalid patterns.
    fn is_valid(&self): bool
}

// For constrained types:
type Day = u8 where value >= 1 and value <= 31

impl Validatable for Day {
    fn is_valid(&self): bool {
        self.0 >= 1 && self.0 <= 31
    }
}

// For enumerations:
impl Validatable for Color {
    fn is_valid(&self): bool {
        let disc = unsafe { transmute::<Self, u8>(*self) }
        disc <= 2
    }
}

// Usage: validating untrusted data
fn parse_color(byte: u8): Option[Color] {
    let color = unsafe { transmute::<u8, Color>(byte) }
    if color.is_valid() {
        Some(color)
    } else {
        None
    }
}

// For floats: checks for NaN/Inf depending on context
impl Validatable for f64 {
    fn is_valid(&self): bool {
        self.is_finite()
    }
}
```

---

## 3.1 core.ops

All operator traits. The type system section of the language reference covers these; they live here.

```ferrum
trait Add[Rhs=Self]     { type Output; fn add(self, rhs: Rhs): Output }
trait Sub[Rhs=Self]     { type Output; fn sub(self, rhs: Rhs): Output }
trait Mul[Rhs=Self]     { type Output; fn mul(self, rhs: Rhs): Output }
trait Div[Rhs=Self]     { type Output; fn div(self, rhs: Rhs): Output }
trait Rem[Rhs=Self]     { type Output; fn rem(self, rhs: Rhs): Output }
trait Neg               { type Output; fn neg(self): Output }
trait Not               { type Output; fn not(self): Output }
trait BitAnd[Rhs=Self]  { type Output; fn bitand(self, rhs: Rhs): Output }
trait BitOr[Rhs=Self]   { type Output; fn bitor(self, rhs: Rhs): Output }
trait BitXor[Rhs=Self]  { type Output; fn bitxor(self, rhs: Rhs): Output }
trait Shl[Rhs=usize]    { type Output; fn shl(self, rhs: Rhs): Output }
trait Shr[Rhs=usize]    { type Output; fn shr(self, rhs: Rhs): Output }

// Compound assignment variants: AddAssign SubAssign ... (same pattern)

trait Index[Idx]        { type Output; fn index(&self, idx: Idx): &Output }
trait IndexMut[Idx]: Index[Idx] { fn index_mut(&mut self, idx: Idx): &mut Output }

trait Deref             { type Target; fn deref(&self): &Target }
trait DerefMut: Deref   { fn deref_mut(&mut self): &mut Target }

trait Fn[Args]      : FnMut[Args] { fn call(&self, args: Args): Output }
trait FnMut[Args]   : FnOnce[Args] { fn call_mut(&mut self, args: Args): Output }
trait FnOnce[Args]  { type Output; fn call_once(self, args: Args): Output }

trait Drop          { fn drop(&mut self) }
```

---

## 3.2 core.cmp

```ferrum
trait PartialEq[Rhs=Self] {
    fn eq(&self, other: &Rhs): bool
    fn ne(&self, other: &Rhs): bool { !self.eq(other) }
}

trait Eq: PartialEq {}   // reflexivity guaranteed — marker

enum Ordering { Less, Equal, Greater }

trait PartialOrd[Rhs=Self]: PartialEq[Rhs] {
    fn partial_cmp(&self, other: &Rhs): Option[Ordering]
    fn lt(&self, other: &Rhs): bool
    fn le(&self, other: &Rhs): bool
    fn gt(&self, other: &Rhs): bool
    fn ge(&self, other: &Rhs): bool
}

trait Ord: Eq + PartialOrd {
    fn cmp(&self, other: &Self): Ordering
    fn min(self, other: Self): Self
    fn max(self, other: Self): Self
    fn clamp(self, min: Self, max: Self): Self
}
```

---

## 3.3 core.option

```ferrum
enum Option[T] {
    Some(T),
    None,
}

impl[T] Option[T] {
    fn is_some(&self): bool
    fn is_none(&self): bool
    fn unwrap(self): T                       // panics on None
    fn unwrap_or(self, default: T): T
    fn unwrap_or_else(self, f: fn(): T): T
    fn unwrap_or_default(self): T  where T: Default
    fn expect(self, msg: &str): T            // panics with msg on None
    fn map[U](self, f: fn(T): U): Option[U]
    fn map_or[U](self, default: U, f: fn(T): U): U
    fn and_then[U](self, f: fn(T): Option[U]): Option[U]
    fn or(self, other: Option[T]): Option[T]
    fn or_else(self, f: fn(): Option[T]): Option[T]
    fn filter(self, pred: fn(&T): bool): Option[T]
    fn as_ref(&self): Option[&T]
    fn as_mut(&mut self): Option[&mut T]
    fn take(&mut self): Option[T]
    fn replace(&mut self, value: T): Option[T]
    fn zip[U](self, other: Option[U]): Option[(T, U)]
    fn unzip[A, B](self: Option[(A, B)]): (Option[A], Option[B])
    fn flatten(self: Option[Option[T]]): Option[T]
    fn ok_or[E](self, err: E): Result[T, E]
    fn ok_or_else[E](self, f: fn(): E): Result[T, E]
    fn transpose[E](self: Option[Result[T, E]]): Result[Option[T], E]
}
```

---

## 3.4 core.result

```ferrum
enum Result[T, E] {
    Ok(T),
    Err(E),
}

impl[T, E] Result[T, E] {
    fn is_ok(&self): bool
    fn is_err(&self): bool
    fn ok(self): Option[T]
    fn err(self): Option[E]
    fn unwrap(self): T                where E: Debug   // panics on Err
    fn unwrap_err(self): E            where T: Debug
    fn expect(self, msg: &str): T     where E: Debug
    fn map[U](self, f: fn(T): U): Result[U, E]
    fn map_err[F](self, f: fn(E): F): Result[T, F]
    fn map_or[U](self, default: U, f: fn(T): U): U
    fn and_then[U](self, f: fn(T): Result[U, E]): Result[U, E]
    fn or_else[F](self, f: fn(E): Result[T, F]): Result[T, F]
    fn unwrap_or(self, default: T): T
    fn unwrap_or_else(self, f: fn(E): T): T
    fn unwrap_or_default(self): T     where T: Default
    fn flatten(self: Result[Result[T, E], E]): Result[T, E]
    fn transpose[T2](self: Result[Option[T2], E]): Option[Result[T2, E]]
    fn as_ref(&self): Result[&T, &E]
    fn as_mut(&mut self): Result[&mut T, &mut E]
}
```

---

## 3.5 core.iter

The iterator protocol. All collection types implement `IntoIterator`. All iterators implement `Iterator`.

```ferrum
trait Iterator {
    type Item

    fn next(&mut self): Option[Self.Item]

    // Provided — all implemented via next():
    fn map[B](self, f: fn(Self.Item): B): Map[Self, B]
    fn filter(self, pred: fn(&Self.Item): bool): Filter[Self]
    fn filter_map[B](self, f: fn(Self.Item): Option[B]): FilterMap[Self, B]
    fn flat_map[U: IntoIterator](self, f: fn(Self.Item): U): FlatMap[Self, U]
    fn flatten(self): Flatten[Self]   where Self.Item: IntoIterator
    fn enumerate(self): Enumerate[Self]
    fn zip[U: Iterator](self, other: U): Zip[Self, U]
    fn chain[U: Iterator[Item=Self.Item]](self, other: U): Chain[Self, U]
    fn take(self, n: usize): Take[Self]
    fn skip(self, n: usize): Skip[Self]
    fn take_while(self, pred: fn(&Self.Item): bool): TakeWhile[Self]
    fn skip_while(self, pred: fn(&Self.Item): bool): SkipWhile[Self]
    fn peekable(self): Peekable[Self]
    fn step_by(self, step: usize): StepBy[Self]
    fn inspect(self, f: fn(&Self.Item)): Inspect[Self]
    fn cloned(self): Cloned[Self]     where Self.Item: Clone  // actually copies
    fn copied(self): Copied[Self]     where Self.Item: Copy
    fn rev(self): Rev[Self]           where Self: DoubleEndedIterator

    fn fold[Acc](self, init: Acc, f: fn(Acc, Self.Item): Acc): Acc
    fn reduce(self, f: fn(Self.Item, Self.Item): Self.Item): Option[Self.Item]
    fn scan[St, B](self, init: St, f: fn(&mut St, Self.Item): Option[B]): Scan[Self, St, B]

    fn for_each(self, f: fn(Self.Item)) ! eff    // inherits effect of f
    fn count(self): usize
    fn last(self): Option[Self.Item]
    fn nth(&mut self, n: usize): Option[Self.Item]
    fn find(&mut self, pred: fn(&Self.Item): bool): Option[Self.Item]
    fn find_map[B](&mut self, f: fn(Self.Item): Option[B]): Option[B]
    fn position(&mut self, pred: fn(Self.Item): bool): Option[usize]
    fn any(&mut self, pred: fn(Self.Item): bool): bool
    fn all(&mut self, pred: fn(Self.Item): bool): bool
    fn sum[S: Sum[Self.Item]](self): S
    fn product[P: Product[Self.Item]](self): P
    fn min(self): Option[Self.Item]   where Self.Item: Ord
    fn max(self): Option[Self.Item]   where Self.Item: Ord
    fn min_by_key[K: Ord](self, f: fn(&Self.Item): K): Option[Self.Item]
    fn max_by_key[K: Ord](self, f: fn(&Self.Item): K): Option[Self.Item]
    fn collect[C: FromIterator[Self.Item]](self): C
    fn unzip[A, B](self): (Vec[A], Vec[B]) where Self.Item == (A, B)
    fn partition[C: Default + Extend[Self.Item]](self, pred: fn(&Self.Item): bool): (C, C)

    // Parallel execution — returns a parallel iterator wrapper
    fn par(self): Par[Self]           where Self.Item: Send
}

trait DoubleEndedIterator: Iterator {
    fn next_back(&mut self): Option[Self.Item]
    fn rfold[Acc](self, init: Acc, f: fn(Acc, Self.Item): Acc): Acc
    fn rfind(&mut self, pred: fn(&Self.Item): bool): Option[Self.Item]
}

trait ExactSizeIterator: Iterator {
    fn len(&self): usize
    fn is_empty(&self): bool { self.len() == 0 }
}

trait IntoIterator {
    type Item
    type IntoIter: Iterator[Item = Self.Item]
    fn into_iter(self): Self.IntoIter
}

trait FromIterator[A] {
    fn from_iter[I: IntoIterator[Item=A]](iter: I): Self
}

trait Extend[A] {
    fn extend[I: IntoIterator[Item=A]](&mut self, iter: I)
}
```

---

## 3.6 core.mem

```ferrum
// Sizes and alignment
const fn size_of[T](): usize
const fn align_of[T](): usize
const fn size_of_val[T: ?Sized](val: &T): usize
const fn align_of_val[T: ?Sized](val: &T): usize

// Manipulation
fn swap[T](a: &mut T, b: &mut T)
fn replace[T](dest: &mut T, src: T): T       // returns old value
fn take[T: Default](dest: &mut T): T          // replaces with Default
fn drop[T](val: T)                              // explicit drop

// Transmutation — unsafe
unsafe fn transmute[T, U](val: T): U
    requires size_of[T]() == size_of[U]()
unsafe fn transmute_copy[T, U](src: &T): U
    requires size_of[U]() <= size_of[T]()

// Zeroing — for security-sensitive data
fn zeroed[T](): T  ! Unsafe    // returns zero-bit-pattern value
fn forget[T](val: T)             // does not run destructor

// MaybeUninit — safe uninitialized memory
type MaybeUninit[T] { ... }
impl[T] MaybeUninit[T] {
    fn uninit(): Self
    fn new(val: T): Self
    fn zeroed(): Self
    unsafe fn assume_init(self): T
    fn write(&mut self, val: T): &mut T
    unsafe fn assume_init_ref(&self): &T
    unsafe fn assume_init_mut(&mut self): &mut T
}
```

---

## 3.7 core.ptr

Raw pointer operations. All unsafe. Used by allocator implementers.

```ferrum
unsafe fn read[T](src: *const T): T
unsafe fn write[T](dst: *mut T, src: T)
unsafe fn copy[T](src: *const T, dst: *mut T, count: usize)
    // src and dst must not overlap — see copy_overlapping for that
unsafe fn copy_overlapping[T](src: *const T, dst: *mut T, count: usize)
    // handles overlap correctly (memmove semantics)
unsafe fn write_bytes(dst: *mut u8, val: u8, count: usize)
unsafe fn read_volatile[T](src: *const T): T
unsafe fn write_volatile[T](dst: *mut T, src: T)

fn null[T](): *const T
fn null_mut[T](): *mut T
fn is_null[T](ptr: *const T): bool
unsafe fn offset[T](ptr: *const T, count: isize): *const T
unsafe fn add[T](ptr: *const T, count: usize): *const T
unsafe fn sub[T](ptr: *const T, count: usize): *const T

// NOTE: copy vs copy_overlapping is a deliberate split.
// C's memcpy UB on overlap was a real-world bug source.
// Ferrum makes the non-overlapping assumption explicit in the name.
```

---

## 3.8 core.slice

```ferrum
impl[T] [T] {   // methods on slice type
    fn len(&self): usize
    fn is_empty(&self): bool
    fn first(&self): Option[&T]
    fn last(&self): Option[&T]
    fn first_mut(&mut self): Option[&mut T]
    fn last_mut(&mut self): Option[&mut T]
    fn get(&self, idx: usize): Option[&T]
    fn get_mut(&mut self, idx: usize): Option[&mut T]
    unsafe fn get_unchecked(&self, idx: usize): &T
    fn split_at(&self, mid: usize): (&[T], &[T])
    fn split_at_mut(&mut self, mid: usize): (&mut [T], &mut [T])
    fn split_first(&self): Option[(&T, &[T])]
    fn split_last(&self): Option[(&T, &[T])]
    fn windows(&self, size: usize): Windows[T]
    fn chunks(&self, size: usize): Chunks[T]
    fn chunks_exact(&self, size: usize): ChunksExact[T]
    fn iter(&self): Iter[T]
    fn iter_mut(&mut self): IterMut[T]
    fn contains(&self, x: &T): bool   where T: PartialEq
    fn starts_with(&self, prefix: &[T]): bool   where T: PartialEq
    fn ends_with(&self, suffix: &[T]): bool     where T: PartialEq
    fn sort(&mut self)                  where T: Ord
    fn sort_by(self, cmp: fn(&T, &T): Ordering)
    fn sort_by_key[K: Ord](&mut self, key: fn(&T): K)
    fn sort_unstable(&mut self)         where T: Ord
    fn binary_search(&self, x: &T): Result[usize, usize]  where T: Ord
    fn binary_search_by(&self, f: fn(&T): Ordering): Result[usize, usize]
    fn reverse(&mut self)
    fn rotate_left(&mut self, mid: usize)
    fn rotate_right(&mut self, mid: usize)
    fn fill(&mut self, value: T)        where T: Clone
    fn copy_from_slice(&mut self, src: &[T])  where T: Copy
    fn clone_from_slice(&mut self, src: &[T]) where T: Clone
    fn as_ptr(&self): *const T
    fn as_mut_ptr(&mut self): *mut T
    fn as_chunks[const N: usize](&self): (&[[T; N]], &[T])
}
```

---

## 3.9 core.str

Methods on `&str`. All assume valid UTF-8 (guaranteed by the type).

```ferrum
impl str {
    fn len(&self): usize              // byte length
    fn char_len(&self): usize         // Unicode scalar count (O(n))
    fn is_empty(&self): bool
    fn as_bytes(&self): &[u8]
    fn chars(&self): Chars            // iterator over char (Unicode scalars)
    fn char_indices(&self): CharIndices
    fn bytes(&self): Bytes
    fn lines(&self): Lines            // iterator over lines, strips \n/\r\n
    fn split(&self, pat: impl Pattern): Split
    fn splitn(&self, n: usize, pat: impl Pattern): SplitN
    fn split_once(&self, pat: impl Pattern): Option[(&str, &str)]
    fn trim(&self): &str              // trims Unicode whitespace
    fn trim_start(&self): &str
    fn trim_end(&self): &str
    fn trim_matches(&self, pat: impl Pattern): &str
    fn starts_with(&self, pat: impl Pattern): bool
    fn ends_with(&self, pat: impl Pattern): bool
    fn contains(&self, pat: impl Pattern): bool
    fn find(&self, pat: impl Pattern): Option[usize]   // byte index
    fn rfind(&self, pat: impl Pattern): Option[usize]
    fn replace(&self, from: impl Pattern, to: &str): String
    fn replacen(&self, from: impl Pattern, to: &str, n: usize): String
    fn to_uppercase(&self): String
    fn to_lowercase(&self): String
    fn repeat(&self, n: usize): String
    fn parse[T: FromStr](&self): Result[T, T.Err]
    fn is_ascii(&self): bool
    fn to_ascii_uppercase(&self): String
    fn to_ascii_lowercase(&self): String

    // Slicing by byte index — panics if not on char boundary
    // Prefer char-level operations when possible
    fn get(&self, range: impl RangeBounds[usize]): Option[&str]
    unsafe fn get_unchecked(&self, range: impl RangeBounds[usize]): &str
}

// Pattern — anything that can be used as a search pattern
trait Pattern {
    // implemented by: char, &str, &[char], fn(char)->bool
}
```

### 3.9.1 BoundedString — Fixed-Capacity Strings

A string with compile-time maximum capacity. No heap allocation, stack-friendly, perfect for embedded systems and protocol messages.

```ferrum
/// A string with a compile-time maximum capacity.
/// Stored inline, no heap allocation.
type BoundedString[const N: usize] {
    data: [u8; N],
    len: usize,

    invariant len <= N
}

impl[const N: usize] BoundedString[N] {
    /// Creates an empty bounded string.
    const fn new(): Self {
        BoundedString {
            data: [0u8; N],
            len: 0,
        }
    }

    /// Creates from a string literal. Compile error if too long.
    const fn from_static(s: &'static str): Self
        requires s.len() <= N

    /// Tries to create from a string slice.
    fn try_from_str(s: &str): Option[Self] {
        if s.len() > N {
            None
        } else {
            let mut bs = Self.new()
            bs.data[..s.len()].copy_from_slice(s.as_bytes())
            bs.len = s.len()
            Some(bs)
        }
    }

    /// Maximum capacity.
    const fn capacity(&self): usize { N }

    /// Current length in bytes.
    fn len(&self): usize { self.len }

    /// Remaining space.
    fn remaining(&self): usize { N - self.len }

    /// Is empty?
    fn is_empty(&self): bool { self.len == 0 }

    /// Is full?
    fn is_full(&self): bool { self.len == N }

    /// As a string slice.
    fn as_str(&self): &str {
        // Safety: we maintain UTF-8 invariant
        unsafe { str.from_utf8_unchecked(&self.data[..self.len]) }
    }

    /// Clear the string.
    fn clear(&mut self) {
        self.len = 0
    }

    /// Append a character. Returns false if no space.
    fn push(&mut self, c: char): bool {
        let char_len = c.len_utf8()
        if self.len + char_len > N {
            false
        } else {
            c.encode_utf8(&mut self.data[self.len..])
            self.len += char_len
            true
        }
    }

    /// Append a string. Returns false if no space.
    fn push_str(&mut self, s: &str): bool {
        if self.len + s.len() > N {
            false
        } else {
            self.data[self.len..self.len + s.len()].copy_from_slice(s.as_bytes())
            self.len += s.len()
            true
        }
    }

    /// Pop the last character.
    fn pop(&mut self): Option[char] {
        if self.len == 0 {
            return None
        }
        let s = self.as_str()
        let c = s.chars().last()?
        self.len -= c.len_utf8()
        Some(c)
    }

    /// Truncate to a byte length.
    fn truncate(&mut self, new_len: usize)
        requires new_len <= self.len
        requires self.as_str().is_char_boundary(new_len)
    {
        self.len = new_len
    }
}

// Display and Debug
impl[const N: usize] Display for BoundedString[N] {
    fn fmt(&self, f: &mut Formatter): Result[(), FmtError] {
        f.write_str(self.as_str())
    }
}

impl[const N: usize] Debug for BoundedString[N] {
    fn fmt(&self, f: &mut Formatter): Result[(), FmtError] {
        write!(f, "BoundedString<{}>({:?})", N, self.as_str())
    }
}

// Comparison — works across different capacities
impl[const N: usize] Eq for BoundedString[N]
impl[const N: usize] PartialEq for BoundedString[N] {
    fn eq(&self, other: &Self): bool {
        self.as_str() == other.as_str()
    }
}

impl[const N: usize, const M: usize] PartialEq[BoundedString[M]] for BoundedString[N] {
    fn eq(&self, other: &BoundedString[M]): bool {
        self.as_str() == other.as_str()
    }
}

impl[const N: usize] PartialEq[str] for BoundedString[N] {
    fn eq(&self, other: &str): bool {
        self.as_str() == other
    }
}

impl[const N: usize] Hash for BoundedString[N] {
    fn hash[H: Hasher](&self, state: &mut H) {
        self.as_str().hash(state)
    }
}

// Deref to str
impl[const N: usize] Deref for BoundedString[N] {
    type Target = str
    fn deref(&self): &str { self.as_str() }
}

// Common aliases
type TinyString = BoundedString[16]
type ShortString = BoundedString[64]
type SmallString = BoundedString[256]

// Usage examples:
// fn format_sensor_reading(sensor_id: u8, value: f32): BoundedString[32] {
//     let mut s = BoundedString.new()
//     write!(&mut s, "S{:02}: {:.2}", sensor_id, value).ok()
//     s
// }
```

---

## 3.10 core.ffi — Foreign Function Interface Types

C interop requires types that bridge Ferrum's UTF-8 strings and C's null-terminated byte strings.

```ferrum
// CStr — borrowed null-terminated C string (like &str but for C)
// Lives in core because it doesn't allocate
type CStr {
    // Unsized, always behind a reference
    // Invariant: ends with null byte, contains no interior nulls
}

impl CStr {
    /// Wraps a raw C string pointer. The pointer must be valid and null-terminated.
    /// The returned lifetime is unbounded — caller must ensure validity.
    unsafe fn from_ptr<'a>(ptr: *const c_char): &'a CStr

    /// Creates from a byte slice WITH the null terminator included.
    /// Returns Err if no null terminator or interior nulls exist.
    fn from_bytes_with_nul(bytes: &[u8]): Result[&CStr, FromBytesWithNulError]

    /// Creates from a byte slice WITHOUT null terminator.
    /// Returns Err if interior nulls exist.
    fn from_bytes_until_nul(bytes: &[u8]): Result[&CStr, FromBytesUntilNulError]

    /// Unchecked version — UB if invariants violated
    unsafe fn from_bytes_with_nul_unchecked(bytes: &[u8]): &CStr

    /// Returns the inner pointer to the C string.
    fn as_ptr(&self): *const c_char

    /// Returns the length in bytes, NOT including the null terminator.
    fn len(&self): usize
        // NOTE: This is O(n) — must scan for null. Cached after first call.

    fn is_empty(&self): bool

    /// Converts to a byte slice WITHOUT the null terminator.
    fn to_bytes(&self): &[u8]

    /// Converts to a byte slice WITH the null terminator.
    fn to_bytes_with_nul(&self): &[u8]

    /// Attempts to convert to a UTF-8 string slice.
    fn to_str(&self): Result[&str, Utf8Error]

    /// Converts to UTF-8, replacing invalid sequences with U+FFFD.
    fn to_string_lossy(&self): Cow[str]
}

// Literal syntax for C strings
const GREETING: &CStr = c"Hello, World!"
// Desugars to a static byte array with null terminator

// Error types
type FromBytesWithNulError {
    fn kind(&self): FromBytesWithNulErrorKind
}

enum FromBytesWithNulErrorKind {
    InteriorNul(usize),   // position of interior null
    NotNulTerminated,
}

type FromBytesUntilNulError {
    // No null found in slice
}

// C char type — platform-dependent signedness
@cfg(target_os = "linux")
type c_char = i8
@cfg(target_os = "macos")
type c_char = i8
@cfg(target_os = "windows")
type c_char = i8

// Other C types for FFI
type c_void = ()       // opaque type for void*
type c_int = i32       // C int (always 32-bit in modern ABIs)
type c_uint = u32
type c_long = i64      // 64-bit on LP64 (Unix), 32-bit on LLP64 (Windows)
type c_ulong = u64
type c_longlong = i64
type c_ulonglong = u64
type c_size_t = usize
type c_ssize_t = isize
type c_ptrdiff_t = isize

// Platform width types (for when C long differs)
@cfg(target_os = "windows")
type c_long = i32
@cfg(target_os = "windows")
type c_ulong = u32

// OsStr — borrowed platform-native string
type OsStr {
    // Unsized, always behind a reference
    // On Unix: arbitrary bytes (usually UTF-8 but not guaranteed)
    // On Windows: potentially ill-formed UTF-16
    // On WASM: UTF-8
}

impl OsStr {
    fn new(s: &str): &OsStr

    /// Checks if the OsStr is empty.
    fn is_empty(&self): bool

    /// Returns the length in bytes (Unix) or WTF-8 bytes (Windows).
    fn len(&self): usize

    /// Converts to a UTF-8 string slice if valid.
    fn to_str(&self): Option[&str]

    /// Converts to String, replacing invalid sequences.
    fn to_string_lossy(&self): Cow[str]

    /// Converts to an owned OsString.
    fn to_os_string(&self): OsString

    /// Returns the raw bytes (Unix) or WTF-8 encoding (Windows).
    fn as_encoded_bytes(&self): &[u8]

    /// Creates from raw encoded bytes. Unsafe on Windows.
    unsafe fn from_encoded_bytes_unchecked(bytes: &[u8]): &OsStr

    // Path-like operations
    fn is_ascii(&self): bool
    fn eq_ignore_ascii_case(&self, other: &OsStr): bool
    fn make_ascii_lowercase(&mut self)
    fn make_ascii_uppercase(&mut self)
}
```

---

## 3.11 core.cell — Interior Mutability

Safe interior mutability patterns that allow mutation through shared references.

```ferrum
// UnsafeCell — the primitive for interior mutability
// This is the ONLY way to get &T -> &mut T legally.
// All other cell types are built on this.
type UnsafeCell[T] {
    value: T,
}

impl[T] UnsafeCell[T] {
    const fn new(value: T): Self

    /// Gets a mutable pointer to the wrapped value.
    /// Caller must ensure no aliasing violations.
    fn get(&self): *mut T

    /// Gets the wrapped value, consuming the cell.
    fn into_inner(self): T

    /// Gets a mutable reference. Only valid when you have &mut UnsafeCell.
    fn get_mut(&mut self): &mut T
}

// UnsafeCell is !Sync — cannot be shared across threads without synchronization
impl[T] !Sync for UnsafeCell[T]

// Cell — single-threaded interior mutability for Copy types
type Cell[T] {
    value: UnsafeCell[T],
}

impl[T] Cell[T] {
    const fn new(value: T): Self

    /// Replaces the value, returning the old one.
    fn replace(&self, val: T): T

    /// Takes the value, leaving Default::default() in its place.
    fn take(&self): T  where T: Default

    /// Swaps values with another Cell.
    fn swap(&self, other: &Cell[T])

    /// Sets the value.
    fn set(&self, val: T)

    /// Gets the value. Requires T: Copy.
    fn get(&self): T  where T: Copy

    /// Updates the value in-place using a function.
    fn update(&self, f: fn(T): T): T  where T: Copy

    /// Gets a mutable pointer to the value.
    fn as_ptr(&self): *mut T

    /// Gets a mutable reference when you have exclusive access.
    fn get_mut(&mut self): &mut T

    /// Consumes the Cell, returning the value.
    fn into_inner(self): T
}

// Cell is !Sync
impl[T] !Sync for Cell[T]

// RefCell — single-threaded interior mutability with runtime borrow checking
type RefCell[T] {
    borrow: Cell[BorrowFlag],
    value: UnsafeCell[T],
}

type BorrowFlag = isize   // negative = mutable, positive = shared count, 0 = unused

impl[T] RefCell[T] {
    const fn new(value: T): Self

    /// Immutably borrows the value.
    /// Panics if currently mutably borrowed.
    fn borrow(&self): Ref[T]

    /// Attempts to immutably borrow.
    fn try_borrow(&self): Result[Ref[T], BorrowError]

    /// Mutably borrows the value.
    /// Panics if currently borrowed (shared or mutable).
    fn borrow_mut(&self): RefMut[T]

    /// Attempts to mutably borrow.
    fn try_borrow_mut(&self): Result[RefMut[T], BorrowMutError]

    /// Consumes the RefCell, returning the value.
    fn into_inner(self): T

    /// Returns a mutable reference when you have exclusive access.
    fn get_mut(&mut self): &mut T

    /// Replaces the value, returning the old one.
    /// Panics if currently borrowed.
    fn replace(&self, t: T): T

    /// Replaces with Default::default(), returning the old value.
    fn take(&self): T  where T: Default

    /// Swaps values with another RefCell.
    /// Panics if either is currently borrowed.
    fn swap(&self, other: &RefCell[T])

    /// Undoes the effect of leaked borrows.
    /// Returns Err if there are outstanding borrows.
    fn undo_leak(&mut self): Result[&mut T, BorrowError]
}

// RefCell is !Sync
impl[T] !Sync for RefCell[T]

// Ref — shared borrow guard for RefCell
type Ref[T] {
    // ... internal fields
}

impl[T] Ref[T] {
    /// Clones a Ref. Both point to the same data.
    fn clone(orig: &Ref[T]): Ref[T]

    /// Maps a Ref to a Ref of a component.
    fn map[U](orig: Ref[T], f: fn(&T): &U): Ref[U]

    /// Like map, but returns None if the function returns None.
    fn filter_map[U](orig: Ref[T], f: fn(&T): Option[&U]): Option[Ref[U]]

    /// Splits into two Refs.
    fn map_split[U, V](orig: Ref[T], f: fn(&T): (&U, &V)): (Ref[U], Ref[V])

    /// Leaks the borrow. The RefCell remains borrowed forever.
    fn leak(orig: Ref[T]): &T
}

impl[T] Deref for Ref[T] {
    type Target = T
}

// RefMut — mutable borrow guard for RefCell
type RefMut[T] {
    // ... internal fields
}

impl[T] RefMut[T] {
    fn map[U](orig: RefMut[T], f: fn(&mut T): &mut U): RefMut[U]
    fn filter_map[U](orig: RefMut[T], f: fn(&mut T): Option[&mut U]): Option[RefMut[U]]
    fn map_split[U, V](orig: RefMut[T], f: fn(&mut T): (&mut U, &mut V)): (RefMut[U], RefMut[V])
    fn leak(orig: RefMut[T]): &mut T
}

impl[T] Deref for RefMut[T] {
    type Target = T
}

impl[T] DerefMut for RefMut[T]

// Errors
type BorrowError {
    // Attempted to borrow while mutably borrowed
}

type BorrowMutError {
    // Attempted to mutably borrow while borrowed
}

// OnceCell — single-assignment cell (can be set once, then immutable)
type OnceCell[T] {
    inner: UnsafeCell[Option[T]],
}

impl[T] OnceCell[T] {
    const fn new(): Self
    const fn new_with(value: T): Self   // pre-initialized

    fn get(&self): Option[&T]
    fn get_mut(&mut self): Option[&mut T]

    /// Sets the value if empty. Returns Err(value) if already set.
    fn set(&self, value: T): Result[(), T]

    /// Gets or initializes with the provided closure.
    fn get_or_init(&self, f: fn(): T): &T

    /// Fallible version of get_or_init.
    fn get_or_try_init[E](&self, f: fn(): Result[T, E]): Result[&T, E]

    fn into_inner(self): Option[T]
    fn take(&mut self): Option[T]
}

// LazyCell — lazily initialized value
type LazyCell[T, F = fn(): T] {
    cell: OnceCell[T],
    init: Cell[Option[F]],
}

impl[T, F: FnOnce(): T] LazyCell[T, F] {
    const fn new(f: F): Self

    /// Forces initialization and returns a reference.
    fn force(this: &Self): &T

    /// Gets the value if already initialized.
    fn get(this: &Self): Option[&T]
}

impl[T, F: FnOnce(): T] Deref for LazyCell[T, F] {
    type Target = T
    fn deref(&self): &T   // forces initialization
}
```

---

## 3.12 core.pin — Pinning

Pinning prevents a value from being moved, enabling self-referential structures and async.

```ferrum
// Pin — a pointer wrapper that prevents moving the pointee
// Critical for:
// - Self-referential types (async state machines)
// - Intrusive data structures
// - FFI with pinned data

type Pin[P] {
    pointer: P,
}

impl[P: Deref] Pin[P] {
    /// Constructs a Pin from a pointer to an Unpin type.
    /// Safe because Unpin types can be freely moved.
    fn new(pointer: P): Self  where P.Target: Unpin

    /// Constructs a Pin without checking Unpin.
    /// SAFETY: Caller must ensure the pointee is never moved.
    unsafe fn new_unchecked(pointer: P): Self

    /// Gets a shared reference to the pinned value.
    fn as_ref(&self): Pin[&P.Target]

    /// Gets a mutable reference if we have mutable access.
    fn as_mut(&mut self): Pin[&mut P.Target]  where P: DerefMut

    /// Gets the underlying value if it's Unpin.
    fn into_inner(self): P  where P.Target: Unpin

    /// Gets the underlying value unsafely.
    unsafe fn into_inner_unchecked(self): P

    /// Maps the pinned reference.
    fn map[U](self, f: fn(Pin[&P.Target]): Pin[&U]): Pin[&U]
}

// Pin projections for struct fields
impl[T> Pin[&mut T] {
    /// Projects to a field. Safe if the field is structurally pinned.
    /// Use the `@pin` attribute on fields to indicate pinning.
    unsafe fn map_unchecked_mut[U](self, f: fn(&mut T): &mut U): Pin[&mut U]
}

impl[T] Pin[&T] {
    unsafe fn map_unchecked[U](self, f: fn(&T): &U): Pin[&U]

    /// Gets a shared reference to the value.
    /// Safe because shared references can't move the value.
    fn get_ref(self): &T
}

impl[T] Pin[&mut T] {
    /// Gets a mutable reference if the type is Unpin.
    fn get_mut(self): &mut T  where T: Unpin

    /// Gets a mutable reference unsafely.
    unsafe fn get_unchecked_mut(self): &mut T

    /// Sets a new value. The old value is dropped in place.
    fn set(&mut self, value: T)  where T: Sized
}

// Unpin — marker trait for types that can be moved even when pinned
// Auto-implemented for most types. NOT implemented for:
// - Types with @pin fields
// - Types containing PhantomPinned
// - Self-referential types
auto trait Unpin {}

// PhantomPinned — marker to make a type !Unpin
type PhantomPinned {
    _marker: (),
}

impl !Unpin for PhantomPinned

// Convenience functions
fn pin[T](value: T): Pin[Box[T]]  given [A: Allocator]
    // Boxes the value and pins it

// Pin projection macro (conceptual — would be a proc macro)
// @derive(PinProject)
// type MyFuture {
//     @pin           // this field is structurally pinned
//     inner: InnerFuture,
//     not_pinned: u32, // this field is not pinned
// }

// Stack pinning (in a block)
// pin_mut!(value);
// Rebinds `value` to Pin<&mut T> for the rest of the scope
macro pin_mut($value:ident) {
    // Implementation: shadows $value with a pinned reference
    // Uses a guard type to prevent the original from being used
}
```

### 3.12.1 Why Pinning Matters

```ferrum
// Without Pin, this async block could break:

async fn example() {
    let data = vec![1, 2, 3]
    let reference = &data[0]   // self-reference!

    some_async_op().await      // suspend point

    // If the future were moved here, `reference` would dangle
    println!("{}", reference)
}

// The compiler generates a state machine struct like:
type ExampleFuture {
    state: u8,
    data: Vec[i32],
    reference: *const i32,  // points into data!
}

// Moving this struct would invalidate `reference`.
// Pin ensures the future can't be moved after polling starts.
```

---

## 3.13 core.error — Error Trait

A proper error trait hierarchy for error handling and chaining.

```ferrum
// The Error trait — foundation of error handling
trait Error: Debug + Display {
    /// The lower-level source of this error, if any.
    fn source(&self): Option[&(dyn Error + 'static)] {
        None   // default: no source
    }

    /// Returns a stack backtrace, if available.
    fn backtrace(&self): Option[&Backtrace] {
        None   // default: no backtrace
    }

    /// Provides type-based access to context.
    /// Enables downcasting through the error chain.
    fn provide(&self, request: &mut Request) {
        // default: provide nothing
    }
}

// All Error types are object-safe and can be made into trait objects
// dyn Error + Send + Sync + 'static is the universal error type

// Request — for type-based error introspection (like Any but for errors)
type Request {
    // Internal implementation
}

impl Request {
    /// Provides a value of type T to the request.
    fn provide_value[T: 'static](&mut self, value: T)

    /// Provides a reference of type T.
    fn provide_ref[T: 'static + ?Sized](&mut self, value: &T)

    /// Checks if the request is for type T.
    fn would_be_satisfied_by_value_of[T: 'static](&self): bool

    fn would_be_satisfied_by_ref_of[T: 'static + ?Sized](&self): bool
}

// Requesting a value from an error
fn request_value[T: 'static>(err: &dyn Error): Option[T]
fn request_ref[T: 'static + ?Sized](err: &dyn Error): Option[&T]

// Error implements From for conversion
impl From[&str] for Box[dyn Error + Send + Sync]
impl From[String] for Box[dyn Error + Send + Sync]

// Helper for creating simple error types
@derive(Debug)
type SimpleError {
    message: String,
}

impl SimpleError {
    fn new(msg: impl Into[String]): Self
}

impl Display for SimpleError {
    fn fmt(&self, f: &mut Formatter): Result[(), FmtError] {
        write!(f, "{}", self.message)
    }
}

impl Error for SimpleError {}

// Error Derive Macro
@derive(Debug, Error)
type MyError {
    @source                    // marks the source field
    source: IoError,

    @backtrace                 // captures backtrace on construction
    backtrace: Backtrace,

    context: String,
}

// Display can be derived from a format string
@derive(Debug, Error)
@error("failed to process {filename}: {source}")
type ProcessError {
    filename: String,

    @source
    source: Box[dyn Error + Send + Sync],
}

// Enum errors
@derive(Debug, Error)
enum DataError {
    @error("I/O error: {0}")
    Io(@source IoError),

    @error("parse error at line {line}: {message}")
    Parse { line: usize, message: String },

    @error("validation failed")
    Validation(@source ValidationError),
}
```

---

## 3.14 core.num — NonZero Types

Integer types that cannot be zero, enabling niche optimization.

```ferrum
// NonZero types — guaranteed non-zero integers
// Key property: Option[NonZeroU32] has the same size as u32
// because the None case uses the forbidden zero bit pattern

type NonZeroU8(u8)
    where value != 0

type NonZeroU16(u16)
    where value != 0

type NonZeroU32(u32)
    where value != 0

type NonZeroU64(u64)
    where value != 0

type NonZeroU128(u128)
    where value != 0

type NonZeroUsize(usize)
    where value != 0

type NonZeroI8(i8)
    where value != 0

type NonZeroI16(i16)
    where value != 0

type NonZeroI32(i32)
    where value != 0

type NonZeroI64(i64)
    where value != 0

type NonZeroI128(i128)
    where value != 0

type NonZeroIsize(isize)
    where value != 0

// Common implementation (shown for NonZeroU32, same for all)
impl NonZeroU32 {
    /// Creates a NonZero if the value is not zero.
    const fn new(n: u32): Option[Self]

    /// Creates a NonZero without checking. UB if n == 0.
    const unsafe fn new_unchecked(n: u32): Self

    /// Returns the value as the primitive type.
    const fn get(self): u32

    /// Returns the number of leading zeros.
    const fn leading_zeros(self): u32

    /// Returns the number of trailing zeros.
    const fn trailing_zeros(self): u32

    /// Checked addition. Returns None on overflow or if result would be zero.
    const fn checked_add(self, other: u32): Option[Self]

    /// Saturating addition.
    const fn saturating_add(self, other: u32): Self

    /// Unchecked addition. UB on overflow.
    const unsafe fn unchecked_add(self, other: u32): Self

    /// Checked multiplication.
    const fn checked_mul(self, other: Self): Option[Self]

    /// Saturating multiplication.
    const fn saturating_mul(self, other: Self): Self

    /// Checked exponentiation.
    const fn checked_pow(self, exp: u32): Option[Self]

    /// Saturating exponentiation.
    const fn saturating_pow(self, exp: u32): Self

    /// Computes self | other, which is never zero since self is non-zero.
    const fn bitor(self, other: u32): Self

    // Integer log operations never return zero for NonZero inputs
    const fn ilog2(self): u32
    const fn ilog10(self): u32
    const fn ilog(self, base: u32): u32
}

// Signed NonZero also has:
impl NonZeroI32 {
    /// Checked absolute value. Returns None for MIN (would overflow).
    const fn checked_abs(self): Option[Self]

    /// Wrapping absolute value.
    const fn wrapping_abs(self): Self

    /// Saturating absolute value.
    const fn saturating_abs(self): Self

    /// Checked negation. Returns None for MIN.
    const fn checked_neg(self): Option[Self]

    /// Wrapping negation.
    const fn wrapping_neg(self): Self

    /// Saturating negation.
    const fn saturating_neg(self): Self

    /// Returns true if positive.
    const fn is_positive(self): bool

    /// Returns true if negative.
    const fn is_negative(self): bool

    /// Returns the sign of self: -1 for negative, 1 for positive.
    const fn signum(self): Self
}

// Conversions
impl From[NonZeroU32] for u32
impl From[NonZeroU32] for NonZeroU64   // widening
impl From[NonZeroU32] for NonZeroU128
impl From[NonZeroU32] for NonZeroI64
impl From[NonZeroU32] for NonZeroI128

// TryFrom for narrowing
impl TryFrom[NonZeroU32] for NonZeroU16
impl TryFrom[NonZeroU32] for NonZeroU8
impl TryFrom[NonZeroI32] for NonZeroI16

// NonZero types are Copy, Clone, Eq, Ord, Hash
impl Copy for NonZeroU32
impl Clone for NonZeroU32
impl Eq for NonZeroU32
impl Ord for NonZeroU32
impl Hash for NonZeroU32
impl Debug for NonZeroU32
impl Display for NonZeroU32

// Size optimization proof
const_assert!(size_of[Option[NonZeroU32]]() == size_of[u32]())
const_assert!(size_of[Option[NonZeroU64]]() == size_of[u64]())
const_assert!(size_of[Option[Option[NonZeroU32]]]() == size_of[u32]())  // nested!
```

### 3.14.1 Niche Optimization Examples

```ferrum
// These are all the same size due to niche optimization:
size_of[u32]()                      // 4 bytes
size_of[Option[NonZeroU32]]()       // 4 bytes (None = 0)
size_of[Result[NonZeroU32, ()]]()   // 4 bytes (Err = 0)

// Useful for space-efficient data structures:
type Handle(NonZeroU32)   // 0 is never a valid handle

type Node {
    value: i32,
    next: Option[NodeIndex],   // same size as NodeIndex!
}

type NodeIndex(NonZeroU32)

// File descriptors (fd 0 is stdin, but handles are non-negative)
type FileHandle(NonZeroU32)

// Array indices where 0 is a valid index — use offset
type ArrayIndex(NonZeroUsize)

impl ArrayIndex {
    fn from_index(idx: usize): Self {
        // Store as idx + 1 internally
        NonZeroUsize.new(idx.checked_add(1).expect("index overflow")).unwrap()
    }

    fn to_index(self): usize {
        self.get() - 1
    }
}
```

---

*End of core module — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
