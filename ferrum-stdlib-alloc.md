# Ferrum Standard Library — alloc + collections

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

`alloc` requires an allocator capability but no OS.

**Allocator defaults:** All types in this module are allocator-generic (`given [A: Allocator]`), but **`Heap` is the default.** Normal code never mentions allocators:

```ferrum
let v = Vec.new()           // uses Heap
let s = String.from("hi")   // uses Heap
let m = HashMap.new()       // uses Heap
```

You only write allocator annotations when you need custom allocation — arenas, pools, or zero-allocation enforcement. See [Allocators](ferrum-lang-generics.md#1-allocators) for details.

---

## 4. alloc — Allocation-dependent Types

### 4.1 alloc.vec

```ferrum
struct Vec[T]  given [A: Allocator] {
    // ...
    invariant self.len <= self.cap
}

impl[T] Vec[T] {
    fn new(): Self                    given [A: Allocator]
    fn with_capacity(cap: usize): Self  given [A: Allocator]
    fn from_slice(s: &[T]): Self      given [A: Allocator]  where T: Clone

    fn len(&self): usize
    fn capacity(&self): usize
    fn is_empty(&self): bool

    fn push(&mut self, val: T)
        ensures self.len() == old(self.len()) + 1
    fn pop(&mut self): Option[T]
        ensures match result {
            Some(_) => self.len() == old(self.len()) - 1,
            None    => self.len() == 0,
        }
    fn insert(&mut self, idx: usize, val: T)
        requires idx <= self.len()
    fn remove(&mut self, idx: usize): T
        requires idx < self.len()
    fn swap_remove(&mut self, idx: usize): T  // O(1), changes order
        requires idx < self.len()

    fn extend_from_slice(&mut self, other: &[T])  where T: Clone
    fn truncate(&mut self, len: usize)
    fn clear(&mut self)
    fn retain(&mut self, pred: fn(&T): bool)
    fn dedup(&mut self)                 where T: PartialEq
    fn dedup_by_key[K: PartialEq](&mut self, key: fn(&mut T): K)
    fn sort(&mut self)                  where T: Ord
    fn sort_by(&mut self, cmp: fn(&T, &T): Ordering)
    fn sort_unstable(&mut self)         where T: Ord

    fn reserve(&mut self, additional: usize)
    fn shrink_to_fit(&mut self)
    fn shrink_to(&mut self, min_cap: usize)

    fn as_slice(&self): &[T]
    fn as_mut_slice(&mut self): &mut [T]
    fn into_boxed_slice(self): Box[[T]]

    fn drain(&mut self, range: impl RangeBounds[usize]): Drain[T]
    fn split_off(&mut self, at: usize): Vec[T]
    fn splice[I: IntoIterator[Item=T]](
        &mut self,
        range: impl RangeBounds[usize],
        replace: I,
    ): Splice[T]
}
```

### 4.2 alloc.string

```ferrum
type String  given [A: Allocator]
// String is Vec[u8] with UTF-8 invariant guaranteed

impl String {
    fn new(): Self                    given [A: Allocator]
    fn with_capacity(cap: usize): Self
    fn from(s: &str): Self
    fn from_utf8(bytes: Vec[u8]): Result[Self, FromUtf8Error]
    unsafe fn from_utf8_unchecked(bytes: Vec[u8]): Self

    fn as_str(&self): &str
    fn as_bytes(&self): &[u8]
    fn len(&self): usize
    fn is_empty(&self): bool
    fn capacity(&self): usize

    fn push(&mut self, c: char)
    fn push_str(&mut self, s: &str)
    fn pop(&mut self): Option[char]
    fn insert(&mut self, byte_idx: usize, c: char)
        requires self.is_char_boundary(byte_idx)
    fn insert_str(&mut self, byte_idx: usize, s: &str)
        requires self.is_char_boundary(byte_idx)
    fn remove(&mut self, byte_idx: usize): char
        requires self.is_char_boundary(byte_idx)
    fn truncate(&mut self, byte_idx: usize)
        requires self.is_char_boundary(byte_idx)
    fn clear(&mut self)

    fn contains(&self, pat: impl Pattern): bool
    fn starts_with(&self, pat: impl Pattern): bool
    fn ends_with(&self, pat: impl Pattern): bool
    fn find(&self, pat: impl Pattern): Option[usize]
    fn replace(&self, from: impl Pattern, to: &str): String
    fn trim(&self): &str
    fn split(&self, pat: impl Pattern): Split
    fn lines(&self): Lines
    fn chars(&self): Chars

    fn is_char_boundary(&self, idx: usize): bool

    // Checked: byte index at non-char-boundary returns Err, not panic
    fn char_at_byte(&self, idx: usize): Result[char, ByteIndexError]

    fn into_bytes(self): Vec[u8]
}
```

---

## 15. collections — Data Structures

### 15.1 HashMap and HashSet

```ferrum
type HashMap[K: Eq + Hash, V]  given [A: Allocator]

impl[K: Eq + Hash, V] HashMap[K, V] {
    fn new(): Self
    fn with_capacity(cap: usize): Self
    fn with_hasher[S: BuildHasher](hasher: S): Self

    fn insert(&mut self, k: K, v: V): Option[V]
    fn remove(&mut self, k: &K): Option[V]
    fn get(&self, k: &K): Option[&V]
    fn get_mut(&mut self, k: &K): Option[&mut V]
    fn get_or_insert(&mut self, k: K, default: V): &mut V
    fn get_or_insert_with(&mut self, k: K, f: fn(): V): &mut V
    fn contains_key(&self, k: &K): bool
    fn len(&self): usize
    fn is_empty(&self): bool
    fn clear(&mut self)
    fn reserve(&mut self, additional: usize)
    fn shrink_to_fit(&mut self)
    fn entry(&mut self, k: K): Entry[K, V]  // Entry API
    fn iter(&self): Iter[K, V]
    fn iter_mut(&mut self): IterMut[K, V]
    fn keys(&self): Keys[K, V]
    fn values(&self): Values[K, V]
    fn values_mut(&mut self): ValuesMut[K, V]
    fn into_keys(self): IntoKeys[K, V]
    fn into_values(self): IntoValues[K, V]
    fn drain(&mut self): Drain[K, V]
    fn retain(&mut self, pred: fn(&K, &mut V): bool)
    fn extend[I: IntoIterator[Item=(K,V)]](&mut self, iter: I)
}

// Entry API — avoids double lookup
enum Entry[K, V] {
    Occupied(OccupiedEntry[K, V]),
    Vacant(VacantEntry[K, V]),
}

impl[K, V] Entry[K, V] {
    fn or_insert(self, default: V): &mut V
    fn or_insert_with(self, f: fn(): V): &mut V
    fn or_default(self): &mut V    where V: Default
    fn and_modify(self, f: fn(&mut V)): Self
    fn key(&self): &K
}

type HashSet[T: Eq + Hash]  given [A: Allocator]
// Set operations:
impl[T: Eq + Hash] HashSet[T] {
    fn union<'a>(&'a self, other: &'a Self): Union[T]
    fn intersection<'a>(&'a self, other: &'a Self): Intersection[T]
    fn difference<'a>(&'a self, other: &'a Self): Difference[T]
    fn symmetric_difference<'a>(&'a self, other: &'a Self): SymmetricDifference[T]
    fn is_subset(&self, other: &Self): bool
    fn is_superset(&self, other: &Self): bool
    fn is_disjoint(&self, other: &Self): bool
    // ... plus standard insert/remove/contains/len/iter
}
```

### 15.2 BTreeMap and BTreeSet

Ordered maps and sets. O(log n) operations.

```ferrum
type BTreeMap[K: Ord, V]  given [A: Allocator]

impl[K: Ord, V] BTreeMap[K, V] {
    // ... all HashMap ops plus:
    fn range[R: RangeBounds[K]](&self, range: R): Range[K, V]
    fn range_mut[R: RangeBounds[K]](&mut self, range: R): RangeMut[K, V]
    fn first_key_value(&self): Option[(&K, &V)]
    fn last_key_value(&self): Option[(&K, &V)]
    fn first_entry(&mut self): Option[OccupiedEntry[K, V]]
    fn last_entry(&mut self): Option[OccupiedEntry[K, V]]
    fn pop_first(&mut self): Option[(K, V)]
    fn pop_last(&mut self): Option[(K, V)]
    fn split_off(&mut self, key: &K): Self
}

type BTreeSet[T: Ord]  given [A: Allocator]
```

### 15.3 Deque

```ferrum
type VecDeque[T]  given [A: Allocator]
// O(1) push/pop at both ends

impl[T] VecDeque[T] {
    fn push_front(&mut self, val: T)
    fn push_back(&mut self, val: T)
    fn pop_front(&mut self): Option[T]
    fn pop_back(&mut self): Option[T]
    fn front(&self): Option[&T]
    fn back(&self): Option[&T]
    fn front_mut(&mut self): Option[&mut T]
    fn back_mut(&mut self): Option[&mut T]
    fn rotate_left(&mut self, n: usize)
    fn rotate_right(&mut self, n: usize)
    fn make_contiguous(&mut self): &mut [T]
    fn as_slices(&self): (&[T], &[T])  // deque may be split internally
    // ... plus standard len/is_empty/iter/clear/retain
}
```

### 15.4 Heap (Priority Queue)

```ferrum
type BinaryHeap[T: Ord]  given [A: Allocator]
// Max-heap by default. For min-heap, use Reverse[T] as the element type.

impl[T: Ord] BinaryHeap[T] {
    fn push(&mut self, val: T)
    fn pop(&mut self): Option[T]
    fn peek(&self): Option[&T]
    fn peek_mut(&mut self): Option[PeekMut[T]]
    fn len(&self): usize
    fn is_empty(&self): bool
    fn into_sorted_vec(self): Vec[T]
    fn drain(&mut self): Drain[T]
    fn drain_sorted(&mut self): DrainSorted[T]  // sorted descending
    fn retain(&mut self, pred: fn(&T): bool)
    fn from_vec(v: Vec[T]): Self   // O(n) heapify
}

// For min-heap:
type Reverse[T: Ord](T)
impl[T: Ord] Ord for Reverse[T] {
    fn cmp(&self, other: &Self): Ordering { other.0.cmp(&self.0) }
}
```

### 15.5 LinkedList

```ferrum
type LinkedList[T]  given [A: Allocator]
// Doubly-linked. O(1) insert/remove with cursor. Rarely the right choice.
// Use VecDeque unless you specifically need O(1) mid-list insert.

impl[T] LinkedList[T] {
    fn cursor_front(&self): Cursor[T]
    fn cursor_back(&self): Cursor[T]
    fn cursor_front_mut(&mut self): CursorMut[T]
    fn cursor_back_mut(&mut self): CursorMut[T]
    fn split_off(&mut self, at: usize): Self
    fn append(&mut self, other: &mut Self)
    // ... standard push/pop front/back/iter/len
}
```

### 15.6 Specialized Structures

```ferrum
// Ring buffer — fixed capacity, O(1) all operations
type RingBuf[T, const N: usize]  // stack-allocated, no heap

impl[T, const N: usize] RingBuf[T, N] {
    fn push(&mut self, val: T): Option[T]  // returns displaced item if full
    fn pop(&mut self): Option[T]
    fn peek(&self): Option[&T]
    fn is_full(&self): bool
    fn len(&self): usize
    fn capacity(&self): usize  // always N
    fn iter(&self): impl Iterator[Item=&T]

    // Zero-copy IO operations — for TLS record writing, network buffers, etc.
    //
    // Pattern: reserve() -> write directly to slice -> commit()
    // Avoids copying data through intermediate buffers.

    fn reserve(&mut self, len: usize): Option[&mut [T]]
        // Returns a contiguous slice of at least `len` free slots.
        // Does NOT advance the write pointer — call commit() after writing.
        // Returns None if not enough contiguous space available.

    fn commit(&mut self, len: usize)
        // Advances write pointer by `len`. Call after writing to reserved slice.
        requires len <= self.reserved_len()

    fn peek_all(&self): (&[T], &[T])
        // Returns all readable data as up to two slices (handles wraparound).
        // First slice starts at read position. Second slice (if non-empty) wraps.

    fn consume(&mut self, len: usize)
        // Advances read pointer by `len`. Call after reading from peek_all.
        requires len <= self.len()

    fn clear(&mut self)
        // Resets to empty without dropping elements (T must be Copy or manually dropped)
}

// Heap-allocated ring buffer — for larger buffers
type RingBuffer[T]  given [A: Allocator]

impl[T] RingBuffer[T] {
    fn with_capacity(cap: usize): Self
    // Same API as RingBuf plus resize operations
}

// SmallVec — stack-allocated up to N, spills to heap
type SmallVec[T, const N: usize]  given [A: Allocator]
// Identical API to Vec — stack/heap is an implementation detail

// IndexMap — insertion-ordered HashMap (useful for JSON/config)
type IndexMap[K: Eq + Hash, V]  given [A: Allocator]
// Same as HashMap but iteration order is insertion order
// O(1) lookup by key, O(1) lookup by index

// MultiMap — one key, multiple values
type MultiMap[K: Eq + Hash, V]  given [A: Allocator]
impl[K: Eq + Hash, V] MultiMap[K, V] {
    fn insert(&mut self, k: K, v: V)
    fn get_all(&self, k: &K): &[V]
    fn remove_all(&mut self, k: &K): Vec[V]
}
```

---

## 4.3 alloc.ffi — Owned FFI Strings

```ferrum
// CString — owned null-terminated C string (like String but for C)
struct CString  given [A: Allocator] {
    inner: Vec[u8],   // includes null terminator
    invariant inner.last() == Some(0) and !inner[..inner.len()-1].contains(&0)
}

impl CString {
    /// Creates a new CString from anything that can become bytes.
    /// Fails if the input contains interior null bytes.
    fn new(t: impl Into[Vec[u8]]): Result[Self, NulError]

    /// Creates from a byte vector, without checking for interior nulls.
    unsafe fn from_vec_unchecked(v: Vec[u8]): Self

    /// Creates from raw parts. Takes ownership of the pointer.
    unsafe fn from_raw(ptr: *mut c_char): Self

    /// Transfers ownership to a raw pointer. Caller must free with from_raw.
    fn into_raw(self): *mut c_char

    /// Borrows as a CStr.
    fn as_c_str(&self): &CStr

    /// Converts to a byte vector WITHOUT the null terminator.
    fn into_bytes(self): Vec[u8]

    /// Converts to a byte vector WITH the null terminator.
    fn into_bytes_with_nul(self): Vec[u8]

    /// Borrows as bytes WITHOUT the null terminator.
    fn as_bytes(&self): &[u8]

    /// Borrows as bytes WITH the null terminator.
    fn as_bytes_with_nul(&self): &[u8]

    /// Converts to String if valid UTF-8.
    fn into_string(self): Result[String, IntoStringError]
}

impl Deref for CString {
    type Target = CStr
    fn deref(&self): &CStr
}

// Construction from string literals
impl From[&str] for CString {
    fn from(s: &str): Self
        // Panics if s contains interior nulls
}

struct NulError {
    fn nul_position(&self): usize
    fn into_vec(self): Vec[u8]   // recover the original data
}

struct IntoStringError {
    fn utf8_error(&self): Utf8Error
    fn into_cstring(self): CString   // recover the CString
}

// OsString — owned platform-native string
struct OsString  given [A: Allocator] {
    inner: Vec[u8],   // WTF-8 on Windows, bytes on Unix
}

impl OsString {
    fn new(): Self
    fn with_capacity(cap: usize): Self

    fn push(&mut self, s: impl AsRef[OsStr])
    fn reserve(&mut self, additional: usize)
    fn shrink_to_fit(&mut self)
    fn capacity(&self): usize
    fn clear(&mut self)

    fn into_string(self): Result[String, OsString]
    fn as_os_str(&self): &OsStr
}

impl Deref for OsString {
    type Target = OsStr
}

impl From[String] for OsString
impl From[&str] for OsString
impl From[PathBuf] for OsString
```

---

## 4.4 alloc.borrow — Clone-on-Write

Copy-on-write smart pointer for efficient borrowed-or-owned semantics.

```ferrum
// Cow — Clone-on-Write smart pointer
// Wraps either a borrowed reference or an owned value.
// Clones to owned only when mutation is required.

enum Cow[B: ToOwned + ?Sized] {
    Borrowed(&B),
    Owned(B.Owned),
}

impl[B: ToOwned + ?Sized] Cow[B] {
    /// Returns true if the data is borrowed.
    fn is_borrowed(&self): bool

    /// Returns true if the data is owned.
    fn is_owned(&self): bool

    /// Acquires a mutable reference to the data.
    /// Clones the data if not already owned.
    fn to_mut(&mut self): &mut B.Owned
        where B.Owned: Clone

    /// Extracts the owned data.
    /// Clones the data if not already owned.
    fn into_owned(self): B.Owned
        where B.Owned: Clone
}

impl[B: ToOwned + ?Sized] Deref for Cow[B] {
    type Target = B

    fn deref(&self): &B {
        match self {
            Cow.Borrowed(b) => b,
            Cow.Owned(ref o) => o.borrow(),
        }
    }
}

impl[B: ToOwned + ?Sized] Clone for Cow[B]
    where B.Owned: Clone
{
    fn clone(&self): Self {
        match self {
            Cow.Borrowed(b) => Cow.Borrowed(b),
            Cow.Owned(o) => Cow.Owned(o.clone()),
        }
    }

    fn clone_from(&mut self, source: &Self) {
        // Optimize: if both are owned, clone_from in place
        match (self, source) {
            (Cow.Owned(dest), Cow.Owned(src)) => dest.clone_from(src),
            _ => *self = source.clone(),
        }
    }
}

// ToOwned — generalized Clone for borrowed types
trait ToOwned {
    type Owned: Borrow[Self]

    fn to_owned(&self): Self.Owned

    fn clone_into(&self, target: &mut Self.Owned) {
        *target = self.to_owned()
    }
}

// Standard implementations
impl ToOwned for str {
    type Owned = String
    fn to_owned(&self): String { String.from(self) }
}

impl ToOwned for [T] where T: Clone {
    type Owned = Vec[T]
    fn to_owned(&self): Vec[T] { self.to_vec() }
}

impl ToOwned for CStr {
    type Owned = CString
    fn to_owned(&self): CString
}

impl ToOwned for OsStr {
    type Owned = OsString
    fn to_owned(&self): OsString
}

impl ToOwned for Path {
    type Owned = PathBuf
    fn to_owned(&self): PathBuf
}

// Borrow — the other direction of ToOwned
trait Borrow[Borrowed: ?Sized] {
    fn borrow(&self): &Borrowed
}

impl Borrow[str] for String {
    fn borrow(&self): &str { self.as_str() }
}

impl[T] Borrow[[T]] for Vec[T] {
    fn borrow(&self): &[T] { self.as_slice() }
}

// Convenience conversions
impl From[&str] for Cow[str] {
    fn from(s: &str): Self { Cow.Borrowed(s) }
}

impl From[String] for Cow[str] {
    fn from(s: String): Self { Cow.Owned(s) }
}

impl[T: Clone] From[&[T]] for Cow[[T]] {
    fn from(s: &[T]): Self { Cow.Borrowed(s) }
}

impl[T: Clone] From[Vec[T]] for Cow[[T]] {
    fn from(v: Vec[T]): Self { Cow.Owned(v) }
}
```

### 4.4.1 Cow Use Cases

```ferrum
// 1. Functions that sometimes need to modify input
fn normalize_path(path: &str): Cow[str] {
    if path.contains("//") {
        Cow.Owned(path.replace("//", "/"))
    } else {
        Cow.Borrowed(path)
    }
}

// Caller doesn't pay for allocation if no normalization needed
let p = normalize_path("/clean/path")    // Borrowed, no alloc
let q = normalize_path("/messy//path")   // Owned, allocated

// 2. Text processing — avoid allocation when no changes
fn escape_html(input: &str): Cow[str] {
    if input.chars().any(|c| matches!(c, '<' | '>' | '&' | '"')) {
        let mut output = String.with_capacity(input.len())
        for c in input.chars() {
            match c {
                '<' => output.push_str("&lt;"),
                '>' => output.push_str("&gt;"),
                '&' => output.push_str("&amp;"),
                '"' => output.push_str("&quot;"),
                c => output.push(c),
            }
        }
        Cow.Owned(output)
    } else {
        Cow.Borrowed(input)
    }
}

// 3. Message formatting
fn format_error(code: u32, details: Option[&str]): Cow[str] {
    match details {
        Some(d) => Cow.Owned(format("Error {}: {}", code, d)),
        None => Cow.Borrowed(ERROR_MESSAGES[code]),
    }
}
```

---

## 4.5 alloc.rc — Reference Counting

Non-thread-safe reference-counted pointers for shared ownership.

```ferrum
// Rc — single-threaded reference-counted pointer
// For thread-safe version, see sync.Arc

struct Rc[T]  given [A: Allocator] {
    ptr: NonNull[RcBox[T]],
    phantom: PhantomData[RcBox[T]],
    alloc: A,
}

struct RcBox[T] {
    strong: Cell[usize],
    weak: Cell[usize],      // +1 for all strong refs
    value: T,
}

impl[T] Rc[T] {
    /// Creates a new Rc.
    fn new(value: T): Self

    /// Creates an Rc with uninitialized contents.
    fn new_uninit(): Rc[MaybeUninit[T]]

    /// Creates an Rc with zeroed contents.
    fn new_zeroed(): Rc[MaybeUninit[T]]

    /// Constructs from a raw pointer. Takes ownership.
    unsafe fn from_raw(ptr: *const T): Self

    /// Returns a mutable reference if this is the only Rc.
    fn get_mut(this: &mut Rc[T]): Option[&mut T]

    /// Makes a mutable reference. Clones if not unique.
    fn make_mut(this: &mut Rc[T]): &mut T
        where T: Clone

    /// Returns the inner value if this is the only Rc.
    fn try_unwrap(this: Rc[T]): Result[T, Rc[T]]

    /// Returns the inner value. Panics if not unique.
    fn unwrap(this: Rc[T]): T
        where T: Clone

    /// Returns the inner value, cloning if necessary.
    fn unwrap_or_clone(this: Rc[T]): T
        where T: Clone

    /// Consumes the Rc, returning a raw pointer.
    fn into_raw(this: Rc[T]): *const T

    /// Returns a raw pointer without consuming.
    fn as_ptr(this: &Rc[T]): *const T

    /// Creates a Weak pointer.
    fn downgrade(this: &Rc[T]): Weak[T]

    /// Returns the number of strong references.
    fn strong_count(this: &Rc[T]): usize

    /// Returns the number of weak references.
    fn weak_count(this: &Rc[T]): usize

    /// Returns true if there's only one strong reference.
    fn is_unique(this: &Rc[T]): bool

    /// Increments the strong count (for manual management).
    unsafe fn increment_strong_count(ptr: *const T)

    /// Decrements the strong count.
    unsafe fn decrement_strong_count(ptr: *const T)

    /// Returns true if two Rcs point to the same allocation.
    fn ptr_eq(this: &Rc[T], other: &Rc[T]): bool
}

impl[T] Clone for Rc[T] {
    fn clone(&self): Self {
        self.inner().strong.set(self.inner().strong.get() + 1)
        Rc { ptr: self.ptr, phantom: PhantomData, alloc: self.alloc }
    }
}

impl[T] Drop for Rc[T] {
    fn drop(&mut self) {
        let inner = self.inner()
        let strong = inner.strong.get()
        if strong == 1 {
            // Drop the value
            drop_in_place(&mut inner.value)
            // Decrement weak count (the implicit weak for strong refs)
            inner.weak.set(inner.weak.get() - 1)
            if inner.weak.get() == 0 {
                // Deallocate
                dealloc(self.ptr)
            }
        } else {
            inner.strong.set(strong - 1)
        }
    }
}

impl[T] Deref for Rc[T] {
    type Target = T
    fn deref(&self): &T { &self.inner().value }
}

// Rc is !Send and !Sync — cannot cross threads
impl[T] !Send for Rc[T]
impl[T] !Sync for Rc[T]

// Weak — non-owning reference that can be upgraded
struct Weak[T]  given [A: Allocator] {
    ptr: NonNull[RcBox[T]],
    alloc: A,
}

impl[T] Weak[T] {
    /// Creates a new Weak that points to nothing.
    fn new(): Self

    /// Attempts to upgrade to an Rc.
    /// Returns None if the value has been dropped.
    fn upgrade(&self): Option[Rc[T]]

    /// Returns the number of strong references.
    /// Returns 0 if the value has been dropped.
    fn strong_count(&self): usize

    /// Returns the number of weak references.
    fn weak_count(&self): usize

    /// Returns true if two Weaks point to the same allocation.
    fn ptr_eq(&self, other: &Weak[T]): bool

    /// Consumes the Weak, returning a raw pointer.
    fn into_raw(self): *const T

    /// Creates from a raw pointer.
    unsafe fn from_raw(ptr: *const T): Self
}

impl[T] Clone for Weak[T] {
    fn clone(&self): Self {
        if let Some(inner) = self.inner() {
            inner.weak.set(inner.weak.get() + 1)
        }
        Weak { ptr: self.ptr, alloc: self.alloc }
    }
}

impl[T] Drop for Weak[T] {
    fn drop(&mut self) {
        if let Some(inner) = self.inner() {
            let weak = inner.weak.get()
            if weak == 1 {
                dealloc(self.ptr)
            } else {
                inner.weak.set(weak - 1)
            }
        }
    }
}

// Weak is !Send and !Sync
impl[T] !Send for Weak[T]
impl[T] !Sync for Weak[T]
```

### 4.5.1 Rc vs Arc

```ferrum
// Use Rc when:
// - Single-threaded only
// - Need shared ownership
// - Want to avoid atomic overhead

// Use Arc when:
// - Multi-threaded
// - Shared across thread boundaries
// - Need Send + Sync

// Example: Tree with parent references
struct TreeNode {
    value: i32,
    parent: Weak[TreeNode],
    children: Vec[Rc[TreeNode]],
}

impl TreeNode {
    fn new(value: i32): Rc[Self] {
        Rc.new(TreeNode {
            value,
            parent: Weak.new(),
            children: Vec.new(),
        })
    }

    fn add_child(parent: &Rc[TreeNode], child: Rc[TreeNode]) {
        Rc.make_mut(child).parent = Rc.downgrade(parent)
        Rc.make_mut(parent).children.push(child)
    }
}

// Cycle: parent owns children, children have weak refs to parent
// When parent is dropped, children's weak refs become invalid
```

---

*End of alloc + collections modules — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
