# Introduction to Unsafe Code in Ferrum

Ferrum is safe by default. The compiler tracks ownership, checks bounds, verifies aliasing rules, and prevents data races at compile time. Most Ferrum code compiles with full safety guarantees and zero runtime overhead for those guarantees.

But systems programming sometimes needs to break the rules. You are writing an operating system kernel and need to write to a hardware register at a fixed memory address. You are implementing a data structure that requires internal pointers the borrow checker cannot reason about. You are calling a C library with no Ferrum equivalent. You are optimizing a hot loop where you have already verified bounds externally and the redundant checks show up in profiling.

Ferrum does not pretend these cases do not exist. Instead, it provides a graduated system of safety levels that make the dangerous parts explicit, auditable, and contained.

---

## The Four Safety Levels

Ferrum has four safety levels beyond the default. Each level relaxes specific guarantees while keeping others intact. The levels are strictly additive: you cannot use a higher level to circumvent what a lower level prohibits.

| Level | Keyword | What You Gain | What You Give Up |
|-------|---------|---------------|------------------|
| 1 | `unchecked` | Skip runtime checks (bounds, overflow) | Compiler-verified safety of those checks |
| 2 | `trusted` | Make assertions the compiler cannot verify | Compiler proof of those properties |
| 3 | `extern` | Call foreign code (C, syscalls) | Knowledge of what foreign code does |
| 4 | `unsafe` | Raw pointer operations, transmute | All automatic memory safety |

This is a ladder, not a menu. Each level exists because the one below it is insufficient for certain real use cases.

---

## Level 1: `unchecked` — I Have Already Verified This

The `unchecked` block skips runtime checks that the compiler cannot statically eliminate. The most common cases are bounds checks and integer overflow checks.

```ferrum
fn sum_unchecked(data: &[i32]): i32 {
    let mut total = 0i32
    for i in 0..data.len() {
        // The compiler inserts a bounds check here in debug mode.
        // In a hot loop over millions of elements, this matters.
        unchecked {
            total += data[i]   // no bounds check
        }
    }
    total
}
```

**When to use `unchecked`:**
- You have already verified the bounds or overflow condition through other means
- The code is performance-critical and profiling shows the checks matter
- The logic guarantees safety but the compiler cannot see it

**What `unchecked` does NOT allow:**
- Raw pointer dereference
- Transmute between types
- Calling foreign functions
- Violating aliasing rules

The `unchecked` block is memory-safe. It skips *checks*, not *safety*. If you write `data[i]` where `i` is out of bounds, you get undefined behavior. But you cannot accidentally dereference a null pointer or create a dangling reference through `unchecked` alone.

### Example: Binary Search Inner Loop

```ferrum
fn binary_search(haystack: &[i32], needle: i32): Option[usize] {
    let mut low = 0usize
    let mut high = haystack.len()

    while low < high {
        let mid = low + (high - low) / 2

        // We know mid < high <= haystack.len() by construction.
        // The compiler sees the bounds check but cannot eliminate it
        // because the relationship between mid and len is not visible.
        let value = unchecked { haystack[mid] }

        if value < needle {
            low = mid + 1
        } else if value > needle {
            high = mid
        } else {
            return Some(mid)
        }
    }
    None
}
```

The loop invariant guarantees `mid < len`. A human can verify this. The compiler cannot. `unchecked` says: "I have done the verification; skip the redundant check."

---

## Level 2: `trusted` — I Assert This Is True

The `trusted` block marks code where the programmer makes an assertion the compiler cannot verify. Unlike `unchecked`, which skips runtime checks, `trusted` makes semantic claims about memory, aliasing, or invariants.

The key feature: `trusted` requires a reason string. This string is machine-readable and appears in audit reports.

```ferrum
trusted("ptr is valid for len bytes, properly aligned, no aliasing references")
fn from_raw_parts[T](ptr: *const T, len: usize): &[T] {
    // Implementation
}
```

**When to use `trusted`:**
- You are wrapping an unsafe operation in a safe API
- The safety depends on invariants you maintain externally
- The compiler cannot verify the relationship between parameters

**The audit trail:**

Every `trusted` annotation is searchable:

```
$ ferrum audit src/ --level trusted

src/collections/vec.rs:142  trusted  "capacity >= len, ptr valid for capacity elements"
src/io/mmap.rs:87           trusted  "mmap returned valid pointer, len matches requested"
src/net/socket.rs:203       trusted  "fd is valid and open"
```

This makes security review tractable. Instead of auditing every line of code, auditors focus on the `trusted` boundaries. Each annotation documents *what* the programmer is claiming and *where* they are claiming it.

### Example: Implementing `split_at_mut`

```ferrum
impl [T] {
    fn split_at_mut(&mut self, mid: usize): (&mut [T], &mut [T]) {
        assert(mid <= self.len(), "split point out of bounds")

        let ptr = self.as_mut_ptr()
        let len = self.len()

        trusted("left and right halves do not overlap; mid <= len verified above") {
            // The borrow checker rejects two &mut to the same array.
            // We assert the halves are disjoint.
            (
                slice_from_raw_parts_mut(ptr, mid),
                slice_from_raw_parts_mut(ptr.add(mid), len - mid),
            )
        }
    }
}
```

The borrow checker cannot see that `[0..mid]` and `[mid..len]` are disjoint. The `trusted` annotation makes the human assertion explicit and auditable.

---

## Level 3: `extern` — Calling Foreign Code

The `extern` block declares functions implemented in other languages (typically C). Ferrum type-checks the Ferrum side of the call, but the foreign code's behavior is unknown.

```ferrum
extern "C" {
    fn strlen(s: *const u8): usize
    fn memcpy(dst: *mut u8, src: *const u8, n: usize): *mut u8
    fn open(path: *const u8, flags: i32): i32
}
```

**Why `extern` is its own level:**
- The foreign code might not follow Ferrum's conventions
- Memory safety depends on the foreign code's behavior
- Effects are unknown (the C function might write to global state, allocate memory, or crash)

All `extern` functions implicitly have the `! Unsafe` effect. You cannot call them from pure code without an `unsafe` or `trusted` wrapper.

### Example: Wrapping a C Library

```ferrum
// Low-level: extern declarations
extern "C" {
    fn zlib_compress(
        dest: *mut u8,
        dest_len: *mut usize,
        src: *const u8,
        src_len: usize,
    ): i32
}

// High-level: safe wrapper
pub fn compress(data: &[u8]): Result[Vec[u8]] ! IO {
    let max_len = data.len() + 1024  // zlib overhead
    let mut output = Vec.with_capacity(max_len)
    let mut actual_len = max_len

    trusted("dest valid for dest_len bytes, src valid for src_len bytes") {
        let result = unsafe {
            zlib_compress(
                output.as_mut_ptr(),
                &mut actual_len,
                data.as_ptr(),
                data.len(),
            )
        }

        if result != 0 {
            return Err(CompressionError.new(result))
        }

        // SAFETY: zlib_compress sets actual_len to bytes written
        output.set_len(actual_len)
    }

    Ok(output)
}
```

The safe `compress` function hides the FFI complexity. Callers get a clean API that takes a slice and returns a `Result`. The `trusted` and `unsafe` blocks are contained, documented, and auditable.

---

## Level 4: `unsafe` — Raw Operations

The `unsafe` block allows operations that can violate memory safety if misused:

- Dereferencing raw pointers (`*ptr`, `*mut_ptr = value`)
- Calling `unsafe` functions
- Accessing mutable statics
- Implementing `unsafe` traits
- Transmuting between types

```ferrum
unsafe fn write_volatile[T](ptr: *mut T, value: T) {
    core.intrinsics.volatile_store(ptr, value)
}

fn poke_hardware_register(addr: usize, value: u32) {
    unsafe {
        let ptr = addr as *mut u32
        write_volatile(ptr, value)
    }
}
```

**When to use `unsafe`:**
- Hardware register access at fixed addresses
- Implementing data structures with internal pointers (intrusive lists, arenas)
- Performance-critical code where you need full control over memory layout
- Interop with non-Ferrum code at the lowest level

**What makes `unsafe` different:**
- No compiler guarantees inside the block
- Every raw pointer operation is your responsibility
- Undefined behavior is possible if you violate invariants
- The blast radius extends to any code that depends on the invariants you maintain

### Example: Implementing a Simple Arena

```ferrum
type Arena {
    buffer: Vec[u8],
    offset: usize,
}

impl Arena {
    pub fn new(capacity: usize): Arena {
        Arena {
            buffer: Vec.with_capacity(capacity),
            offset: 0,
        }
    }

    pub fn alloc[T](&mut self, value: T): &mut T {
        let size = core.mem.size_of[T]()
        let align = core.mem.align_of[T]()

        // Align the offset
        let aligned_offset = (self.offset + align - 1) & !(align - 1)
        let new_offset = aligned_offset + size

        assert(new_offset <= self.buffer.capacity(), "arena full")

        trusted("aligned_offset is aligned for T, space verified above") {
            unsafe {
                let ptr = self.buffer.as_mut_ptr().add(aligned_offset) as *mut T
                ptr.write(value)
                self.offset = new_offset
                &mut *ptr
            }
        }
    }
}
```

The arena uses raw pointer arithmetic internally but exposes a safe API. The `assert` verifies capacity. The `trusted` annotation documents the alignment invariant. The `unsafe` block contains the actual raw pointer operations.

---

## Containing Unsafety

The goal is not to eliminate unsafe code. The goal is to contain it.

**The pattern:**
1. Write the unsafe implementation as a small, focused module
2. Wrap it in a safe API that maintains invariants
3. Document the safety requirements in `trusted` annotations
4. Test the wrapper extensively

**Bad:**
```ferrum
// Unsafe scattered throughout
fn process_data(data: &[u8]): Vec[u8] {
    let mut result = Vec.new()
    for i in 0..data.len() {
        unsafe {
            let ptr = data.as_ptr().add(i)
            let byte = *ptr
            // ... more unsafe operations mixed with logic ...
        }
    }
    result
}
```

**Good:**
```ferrum
// Unsafe contained in one place
mod raw_buffer {
    // All unsafe operations here, with documented invariants
    trusted("buffer valid for len bytes")
    pub fn copy_byte(buffer: &[u8], index: usize): u8 {
        assert(index < buffer.len())
        unsafe { *buffer.as_ptr().add(index) }
    }
}

// Safe code uses the safe wrapper
fn process_data(data: &[u8]): Vec[u8] {
    let mut result = Vec.new()
    for i in 0..data.len() {
        let byte = raw_buffer.copy_byte(data, i)
        // ... safe logic ...
    }
    result
}
```

The second version isolates the unsafe operation, documents its requirements, and lets the rest of the code remain fully verified.

---

## Comparing to C

In C, everything is unsafe. There is no boundary between safe and dangerous operations.

```c
// All of these are equally "normal" in C
int x = arr[i];           // bounds check? what bounds check?
int* p = (int*)0x1000;    // sure, why not
memcpy(dst, src, n);      // hope those don't overlap
*p = 42;                  // is p valid? who knows
```

The compiler does not track which operations are dangerous. Code review must examine every line. Static analyzers can help but produce false positives. The culture normalizes undefined behavior.

In Ferrum, the dangerous parts are explicit:

```ferrum
let x = arr[i]              // bounds checked
let x = unchecked { arr[i] } // bounds check skipped (I verified elsewhere)

let p: *mut i32 = addr as *mut i32  // raw pointer, but can't dereference yet
unsafe { *p = 42 }                   // NOW you're taking responsibility

trusted("regions do not overlap, n <= min(dst.len, src.len)") {
    extern_memcpy(dst, src, n)       // FFI with documented assumptions
}
```

An auditor can search for `unsafe`, `trusted`, and `unchecked` to find every location where human judgment replaces compiler verification. In C, that search returns every line of code.

---

## Comparing to Python

Python with `ctypes` looks deceptively safe:

```python
from ctypes import *

lib = CDLL("libfoo.so")
lib.do_something(c_char_p(b"hello"), c_int(42))
```

Nothing in the syntax indicates you just called arbitrary C code. The Python syntax looks like a normal function call. But:
- The C function might crash
- It might corrupt memory
- It might not be thread-safe
- It might expect the buffer to outlive the call

The danger is invisible.

In Ferrum, FFI is visually distinct:

```ferrum
extern "C" {
    fn do_something(s: *const u8, n: i32)
}

// This will not compile outside unsafe or trusted
do_something(b"hello".as_ptr(), 42)  // ERROR: unsafe function

// You must acknowledge the boundary
trusted("s is null-terminated, valid for duration of call") {
    unsafe { do_something(b"hello\0".as_ptr(), 42) }
}
```

The `extern`, `trusted`, and `unsafe` markers force acknowledgment of the boundary. Code review can focus on these boundaries.

---

## Common Patterns

### Safe Wrapper Around Unsafe Implementation

Most unsafe code in practice follows this pattern: the module internally uses unsafe operations but exposes a safe API.

```ferrum
mod vec {
    type Vec[T] {
        ptr: *mut T,
        len: usize,
        cap: usize,
    }

    impl Vec[T] {
        // Safe API
        pub fn push(&mut self, value: T) {
            if self.len == self.cap {
                self.grow()
            }
            trusted("len < cap after grow") {
                unsafe {
                    self.ptr.add(self.len).write(value)
                    self.len += 1
                }
            }
        }

        // Safe API
        pub fn get(&self, index: usize): Option[&T] {
            if index < self.len {
                trusted("index < len verified") {
                    Some(unsafe { &*self.ptr.add(index) })
                }
            } else {
                None
            }
        }

        // Internal unsafe helper
        fn grow(&mut self) {
            // ... reallocation logic ...
        }
    }
}
```

Users of `Vec` never see `unsafe`. The module author maintains the invariants (len <= cap, ptr valid for cap elements). The `trusted` annotations document what those invariants are.

### Unchecked for Performance-Critical Inner Loops

When profiling shows bounds checks in a hot loop, and you can verify bounds externally:

```ferrum
fn matrix_multiply(a: &Matrix, b: &Matrix, out: &mut Matrix) {
    assert(a.cols == b.rows, "dimension mismatch")
    assert(out.rows == a.rows && out.cols == b.cols, "output dimension mismatch")

    // After the asserts, we know all accesses will be in bounds.
    // The compiler cannot see through the Matrix abstraction.
    for i in 0..a.rows {
        for j in 0..b.cols {
            let mut sum = 0.0
            for k in 0..a.cols {
                unchecked {
                    sum += a[(i, k)] * b[(k, j)]
                }
            }
            unchecked {
                out[(i, j)] = sum
            }
        }
    }
}
```

The asserts establish the preconditions. The inner loop runs without per-access bounds checks.

### Transmute for Low-Level Conversions

When you need to reinterpret bytes as a different type:

```ferrum
fn bytes_to_u32_le(bytes: &[u8; 4]): u32 {
    trusted("4-byte array is valid u32 representation") {
        unsafe {
            core.mem.transmute(*bytes)
        }
    }
}

fn u32_to_bytes_le(value: u32): [u8; 4] {
    trusted("u32 is valid 4-byte array representation") {
        unsafe {
            core.mem.transmute(value)
        }
    }
}
```

The `trusted` annotation documents the implicit assumption: these representations are compatible on this platform.

---

## The Rule

**If you can do it safely, do it safely.**

Unsafe code exists for cases where:
1. The operation is fundamentally impossible to verify statically (hardware access, FFI)
2. The compiler's analysis is too conservative for your specific case
3. You have verified correctness by other means (proof, testing, external invariant)

Do not reach for `unchecked` because you think the code might be faster. Profile first. The compiler eliminates more checks than you expect.

Do not use `unsafe` to avoid fighting the borrow checker. The borrow checker is usually right. When it rejects your code, consider whether your design has a hidden aliasing bug. Often it does.

When you genuinely need unsafe code:
- Keep the blocks small
- Document the invariants in `trusted` annotations
- Wrap the unsafe code in a safe API
- Test exhaustively, including edge cases
- Run `ferrum audit` regularly and review the results

The goal is not zero unsafe blocks. The goal is that every unsafe block is justified, documented, contained, and reviewed.
