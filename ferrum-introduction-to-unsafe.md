# Introduction to Unsafe Code in Ferrum

If you know C, all of your code is unsafe. Every array access might be out of bounds. Every pointer might be null, dangling, or pointing to the wrong type. Every cast might be wrong. The compiler does not track any of this. You are always one typo away from a segfault, or worse, silent memory corruption that shows up three weeks later in production.

If you know Python with ctypes, the danger is hidden. You call `lib.some_function(pointer, length)` and it looks like a normal Python function call. But that C function might crash, corrupt memory, or expect the buffer to outlive the call. The syntax gives you no warning.

Ferrum takes a different approach. Most code is safe by default. The compiler tracks ownership, checks bounds, verifies aliasing rules, and prevents data races at compile time. But systems programming sometimes needs to break the rules, so Ferrum provides a graduated system of safety levels that make the dangerous parts explicit, auditable, and contained.

This document explains when and how to write unsafe code in Ferrum. The target reader is someone who has written C and knows that "undefined behavior" is not an abstract concern, and who has used Python FFI and discovered the hard way that ctypes does not protect you from anything.

---

## Why Have Unsafe Code At All?

You cannot build an operating system, a database, or a language runtime entirely within the safe subset of any language. At some point, you need to:

- **Talk to hardware.** Memory-mapped I/O requires writing to fixed addresses that do not come from the allocator.
- **Call C libraries.** Most of the world's useful code is in C. You need FFI.
- **Implement fundamental data structures.** A `Vec` internally manages raw memory. A linked list has pointers the borrow checker cannot track. A lock-free queue requires atomic operations on raw pointers.
- **Optimize hot paths.** Sometimes the compiler inserts bounds checks that profiling proves are redundant. You have verified the bounds externally and want to skip the check.

Ferrum does not pretend these cases do not exist. Instead, it provides four levels of relaxed safety, each with explicit syntax that marks where human judgment replaces compiler verification.

---

## The Four Safety Levels

Ferrum has four safety levels beyond the default. Each level relaxes specific guarantees while keeping others intact.

| Level | Keyword | What It Does | When You Need It |
|-------|---------|--------------|------------------|
| 1 | `unchecked` | Skips runtime checks (bounds, overflow) | Hot loops where you verified bounds externally |
| 2 | `trusted` | Declares an invariant the compiler cannot verify | Wrapping unsafe code in a safe API |
| 3 | `extern` | Calls foreign code (C, syscalls) | FFI with C libraries, system calls |
| 4 | `unsafe` | Raw pointer operations, transmute, volatile | Implementing data structures, hardware access |

This is a ladder, not a menu. Each level exists because the one below it is insufficient for certain real use cases. You cannot use `unchecked` to dereference a raw pointer. You cannot use `trusted` without a concrete reason string.

### The Key Insight

In C, the answer to "is this code safe?" is always "I don't know, let me read it carefully." In Ferrum, you can `grep` for `unsafe`, `trusted`, `unchecked`, and `extern` to find every location where safety depends on human judgment. Everything else is verified by the compiler.

---

## Level 1: `unchecked` --- Skipping Runtime Checks

### What It Does

The `unchecked` block tells the compiler: "Skip the runtime check here. I have already verified this condition through other means."

The most common use cases are:

- **Bounds checks.** Ferrum inserts bounds checks on array/slice access. In a hot loop where the index is provably in range, this adds overhead.
- **Integer overflow checks.** In debug builds, Ferrum checks for overflow. Sometimes you want wrapping arithmetic and know overflow will not happen.

### What It Does NOT Do

`unchecked` does not let you dereference raw pointers, call FFI, or violate aliasing rules. It only skips checks. The memory safety model is still in effect.

If you write `data[i]` inside an `unchecked` block and `i` is out of bounds, you get undefined behavior. The compiler will not catch it. But you cannot accidentally create a dangling reference or null dereference through `unchecked` alone.

### The C Comparison

In C, there are no bounds checks to skip. Every array access is unchecked:

```c
// C: No bounds check. Silent corruption if i >= len.
int value = arr[i];
```

In Ferrum, bounds checks are the default:

```ferrum
// Ferrum: Bounds check at runtime.
let value = arr[i]  // panics if i >= arr.len()

// Ferrum: Bounds check skipped. You are taking responsibility.
let value = unchecked { arr[i] }
```

The difference is that the dangerous version is visually distinct. Code review can find every `unchecked` block.

### Example: Binary Search Inner Loop

```ferrum
fn binary_search(haystack: &[i32], needle: i32): Option[usize] {
    let mut low = 0usize
    let mut high = haystack.len()

    while low < high {
        let mid = low + (high - low) / 2

        // Loop invariant: low <= mid < high <= haystack.len()
        // Therefore mid is always a valid index.
        // The compiler cannot see this relationship.
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

The loop invariant guarantees `mid < len`. A human can verify this by reading the code. The compiler cannot, because the relationship between `mid`, `high`, and `len` is not visible in the type system.

### Example: Matrix Multiplication Hot Loop

When profiling shows bounds checks dominating a hot loop:

```ferrum
fn matrix_multiply(a: &Matrix, b: &Matrix, out: &mut Matrix) {
    // Preconditions verified once, upfront
    assert(a.cols == b.rows, "dimension mismatch")
    assert(out.rows == a.rows && out.cols == b.cols, "output size mismatch")

    for i in 0..a.rows {
        for j in 0..b.cols {
            let mut sum = 0.0
            for k in 0..a.cols {
                // Inner loop runs millions of times.
                // Bounds already verified by the asserts above.
                unchecked {
                    sum += a[(i, k)] * b[(k, j)]
                }
            }
            unchecked { out[(i, j)] = sum }
        }
    }
}
```

The asserts establish the preconditions. The inner loop runs without per-access bounds checks. This is the same optimization a C programmer makes implicitly; in Ferrum you make it explicit.

### When NOT to Use `unchecked`

Do not use `unchecked` because you "think" the code is correct. Profile first. The compiler eliminates more bounds checks than you expect, especially when:

- The index comes from `enumerate()` or a range bounded by `len()`
- The slice was just created from a known-length source
- LLVM can prove the bounds relationship

If profiling does not show bounds checks in your hot path, do not add `unchecked`. You are giving up a safety net for no benefit.

---

## Level 2: `trusted` --- Asserting Unprovable Invariants

### What It Does

The `trusted` block marks code where the programmer makes a semantic assertion the compiler cannot verify. Unlike `unchecked` which skips a specific runtime check, `trusted` says: "I am asserting something about memory, aliasing, or invariants that the type system cannot express."

The distinguishing feature: **`trusted` requires a reason string.** This string documents what you are asserting. It appears in audit reports.

```ferrum
trusted("ptr is valid for len bytes, properly aligned, no aliasing mutable references") {
    // ... code that depends on this assertion ...
}
```

### Why This Level Exists

Consider `split_at_mut`, which splits a mutable slice into two non-overlapping halves:

```ferrum
impl [T] {
    fn split_at_mut(&mut self, mid: usize): (&mut [T], &mut [T]) {
        assert(mid <= self.len(), "split point out of bounds")

        // We want to return (&mut self[0..mid], &mut self[mid..])
        // But the borrow checker sees two &mut borrows of self.
        // It cannot see that the ranges do not overlap.
    }
}
```

The borrow checker rejects this. It sees two mutable borrows of the same slice and conservatively says "no." But we know the ranges `[0..mid]` and `[mid..len]` are disjoint. The `trusted` block lets us express this:

```ferrum
impl [T] {
    fn split_at_mut(&mut self, mid: usize): (&mut [T], &mut [T]) {
        assert(mid <= self.len(), "split point out of bounds")

        let ptr = self.as_mut_ptr()
        let len = self.len()

        trusted("left [0..mid] and right [mid..len] do not overlap; mid <= len verified by assert") {
            unsafe {
                (
                    slice_from_raw_parts_mut(ptr, mid),
                    slice_from_raw_parts_mut(ptr.add(mid), len - mid),
                )
            }
        }
    }
}
```

The `trusted` annotation documents the invariant. An auditor can read the reason string and verify that the assertion is correct.

### The C Comparison

In C, you would just write the code:

```c
void split_at_mut(int* arr, size_t len, size_t mid, int** left, int** right) {
    *left = arr;
    *right = arr + mid;
    // Hope the caller knows left and right are the same buffer
}
```

There is no indication that safety depends on `mid <= len` or that the caller must not use both halves simultaneously in a way that violates aliasing. The invariants are in comments (maybe) or in the programmer's head.

In Ferrum, the `trusted` annotation forces you to write down what you are assuming. The assumption is searchable, auditable, and appears in tooling output.

### Example: Safe Wrapper for Raw Buffer Creation

```ferrum
// This function creates a slice from a raw pointer.
// The caller asserts the pointer is valid.
trusted("ptr is non-null, properly aligned for T, valid for len elements, no aliasing &mut")
pub fn slice_from_raw_parts[T](ptr: *const T, len: usize): &[T] {
    unsafe {
        &*core.ptr.slice_from_raw_parts(ptr, len)
    }
}
```

Any caller of this function sees the `trusted` annotation and knows they are responsible for the stated invariants. If they call it with an invalid pointer, the bug is theirs.

### The Audit Trail

Every `trusted` annotation is searchable:

```
$ ferrum audit src/ --level trusted

src/collections/vec.fe:142  trusted  "capacity >= len, ptr valid for capacity elements"
src/collections/vec.fe:178  trusted  "index < len verified by bounds check above"
src/io/mmap.fe:87           trusted  "mmap returned valid pointer, len matches requested size"
src/net/socket.fe:203       trusted  "fd is valid open file descriptor"
src/crypto/aes.fe:44        trusted  "key and iv are exactly 16 bytes, verified by type"
```

This makes security review tractable. Instead of auditing 100,000 lines of code, auditors focus on 50 `trusted` boundaries. Each annotation documents what the programmer is claiming and where. If the claim is wrong, the bug is in that block.

---

## Level 3: `extern` --- Foreign Function Interface

### What It Does

The `extern` block declares functions implemented in other languages, typically C. Ferrum type-checks the Ferrum side of the call, but the foreign code's behavior is unknown.

```ferrum
extern "C" {
    fn strlen(s: *const u8): usize
    fn memcpy(dst: *mut u8, src: *const u8, n: usize): *mut u8
    fn open(path: *const u8, flags: i32): i32
    fn read(fd: i32, buf: *mut u8, count: usize): isize
    fn close(fd: i32): i32
}
```

### Why This Is Its Own Level

Calling C code is different from "just" using raw pointers. The foreign code might:

- Not follow Ferrum's aliasing rules
- Write to global state
- Allocate memory you need to free with a specific allocator
- Expect callbacks with specific calling conventions
- Not be thread-safe
- Crash on invalid input instead of returning an error

All `extern` functions implicitly have the `! Unsafe` effect. You cannot call them from safe code without a wrapper.

### The Python Comparison

Python with ctypes hides the danger:

```python
from ctypes import *

lib = CDLL("libz.so")
lib.compress(dest_ptr, byref(dest_len), src_ptr, src_len)
```

This looks like a normal function call. Nothing in the syntax indicates you just called arbitrary C code that might crash, corrupt memory, or expect buffers to have specific lifetimes.

In Ferrum, FFI is visually distinct:

```ferrum
extern "C" {
    fn compress(dest: *mut u8, dest_len: *mut usize, src: *const u8, src_len: usize): i32
}

// This does not compile outside unsafe:
compress(dest_ptr, &mut dest_len, src_ptr, src_len)  // ERROR: requires unsafe

// You must acknowledge the boundary:
unsafe {
    compress(dest_ptr, &mut dest_len, src_ptr, src_len)
}
```

### Example: Wrapping a System Call

Let's wrap the POSIX `read` system call in a safe Ferrum API.

```ferrum
// Low level: extern declaration
extern "C" {
    fn read(fd: i32, buf: *mut u8, count: usize): isize
}

// High level: safe wrapper
pub fn read_file(fd: FileDescriptor, buffer: &mut [u8]): Result[usize] ! IO {
    // FileDescriptor is an opaque type that guarantees fd is valid.
    // buffer is a Ferrum slice, so we know it's valid for buffer.len() bytes.

    trusted("fd is valid (guaranteed by FileDescriptor type), buffer valid for len bytes (guaranteed by slice)") {
        let result = unsafe {
            read(fd.raw(), buffer.as_mut_ptr(), buffer.len())
        }

        if result < 0 {
            Err(IOError.from_errno())
        } else {
            Ok(result as usize)
        }
    }
}
```

Users of `read_file` get a clean API:

```ferrum
let mut buffer = [0u8; 1024]
let bytes_read = read_file(my_file, &mut buffer)?
// No unsafe, no raw pointers, no FFI visible here
```

The safe wrapper:
1. Takes safe Ferrum types (`FileDescriptor`, `&mut [u8]`)
2. Converts to C types at the boundary
3. Handles error codes and converts to `Result`
4. Documents the invariants in `trusted`

### Example: Wrapping a C Library (zlib)

```ferrum
// Raw FFI declarations - direct mapping to C API
mod zlib_raw {
    extern "C" {
        fn compressBound(source_len: usize): usize
        fn compress2(
            dest: *mut u8,
            dest_len: *mut usize,
            source: *const u8,
            source_len: usize,
            level: i32,
        ): i32
        fn uncompress(
            dest: *mut u8,
            dest_len: *mut usize,
            source: *const u8,
            source_len: usize,
        ): i32
    }

    pub const Z_OK: i32 = 0
    pub const Z_BUF_ERROR: i32 = -5
    pub const Z_DEFAULT_COMPRESSION: i32 = -1
}

// Safe public API
pub fn compress(data: &[u8]): Result[Vec[u8]] ! Alloc {
    if data.is_empty() {
        return Ok(Vec.new())
    }

    // zlib tells us the maximum compressed size
    let max_len = zlib_raw.compressBound(data.len())
    let mut output = Vec.with_capacity(max_len)
    let mut actual_len = max_len

    trusted("output valid for max_len bytes (just allocated), data valid (Ferrum slice)") {
        let result = unsafe {
            zlib_raw.compress2(
                output.as_mut_ptr(),
                &mut actual_len,
                data.as_ptr(),
                data.len(),
                zlib_raw.Z_DEFAULT_COMPRESSION,
            )
        }

        if result != zlib_raw.Z_OK {
            return Err(ZlibError.new(result))
        }

        // SAFETY: compress2 wrote actual_len bytes to output
        unsafe { output.set_len(actual_len) }
    }

    Ok(output)
}

pub fn decompress(data: &[u8], max_output_size: usize): Result[Vec[u8]] ! Alloc {
    let mut output = Vec.with_capacity(max_output_size)
    let mut actual_len = max_output_size

    trusted("output valid for max_output_size bytes, data valid (Ferrum slice)") {
        let result = unsafe {
            zlib_raw.uncompress(
                output.as_mut_ptr(),
                &mut actual_len,
                data.as_ptr(),
                data.len(),
            )
        }

        match result {
            zlib_raw.Z_OK => {
                unsafe { output.set_len(actual_len) }
                Ok(output)
            }
            zlib_raw.Z_BUF_ERROR => Err(ZlibError.BufferTooSmall)
            _ => Err(ZlibError.new(result))
        }
    }
}
```

Users call `compress(data)` and get a `Result[Vec[u8]]`. They never see raw pointers, `extern`, or `unsafe`. All the FFI complexity is contained in one module with documented invariants.

---

## Level 4: `unsafe` --- Raw Memory Operations

### What It Does

The `unsafe` block allows operations that can violate memory safety if misused:

- **Dereferencing raw pointers:** `*ptr`, `(*ptr) = value`
- **Calling unsafe functions:** functions marked `unsafe fn`
- **Reading/writing mutable statics:** global mutable state
- **Implementing unsafe traits:** traits with invariants the compiler cannot verify
- **Transmuting between types:** reinterpreting the bytes of one type as another
- **Volatile operations:** hardware register access

### The C Comparison

In C, all of these operations are normal code:

```c
// All of this is just... C
int* p = (int*)0x1000;        // cast address to pointer
*p = 42;                       // dereference and write
memcpy(&x, &y, sizeof(x));    // reinterpret bytes
global_counter++;              // mutable global access

// None of this is syntactically marked as "dangerous"
```

In Ferrum, each of these requires explicit `unsafe`:

```ferrum
// Creating a raw pointer is safe (it's just a number)
let p: *mut i32 = 0x1000 as *mut i32

// But dereferencing it requires unsafe
unsafe { *p = 42 }

// Transmute requires unsafe
let bytes: [u8; 4] = unsafe { core.mem.transmute(value) }

// Mutable static access requires unsafe
unsafe { GLOBAL_COUNTER += 1 }
```

The syntax makes the dangerous operations visible. An auditor can find every location where raw memory operations occur.

### Example: Implementing a Ring Buffer

A ring buffer is a fixed-size circular queue. It needs internal pointer arithmetic that the borrow checker cannot track.

```ferrum
type RingBuffer[T] {
    buffer: *mut T,      // Raw pointer to storage
    capacity: usize,
    head: usize,         // Next read position
    tail: usize,         // Next write position
}

impl RingBuffer[T] {
    pub fn new(capacity: usize): RingBuffer[T] ! Alloc {
        assert(capacity > 0, "capacity must be positive")

        let layout = core.alloc.Layout.array[T](capacity)
        let ptr = core.alloc.alloc(layout) as *mut T

        if ptr.is_null() {
            panic("allocation failed")
        }

        RingBuffer {
            buffer: ptr,
            capacity: capacity,
            head: 0,
            tail: 0,
        }
    }

    pub fn push(&mut self, value: T): bool {
        let next_tail = (self.tail + 1) % self.capacity

        if next_tail == self.head {
            return false  // Buffer full
        }

        trusted("tail < capacity by modulo, buffer valid for capacity elements") {
            unsafe {
                self.buffer.add(self.tail).write(value)
            }
        }

        self.tail = next_tail
        true
    }

    pub fn pop(&mut self): Option[T] {
        if self.head == self.tail {
            return None  // Buffer empty
        }

        trusted("head < capacity by modulo, buffer valid, element was written by push") {
            let value = unsafe {
                self.buffer.add(self.head).read()
            }
            self.head = (self.head + 1) % self.capacity
            Some(value)
        }
    }

    pub fn len(&self): usize {
        if self.tail >= self.head {
            self.tail - self.head
        } else {
            self.capacity - self.head + self.tail
        }
    }
}

impl Drop for RingBuffer[T] {
    fn drop(&mut self) {
        // Drop all remaining elements
        while let Some(_) = self.pop() {}

        // Free the buffer
        let layout = core.alloc.Layout.array[T](self.capacity)
        unsafe {
            core.alloc.dealloc(self.buffer as *mut u8, layout)
        }
    }
}
```

Users of `RingBuffer` see only the safe API:

```ferrum
let mut rb = RingBuffer[i32].new(16)
rb.push(1)
rb.push(2)
let x = rb.pop()  // Some(1)
```

The internal pointer arithmetic is contained. The `trusted` annotations document the invariants. The `Drop` implementation ensures memory is freed.

### Example: Hardware Register Access

When writing a device driver, you need to read and write to specific memory addresses:

```ferrum
// Memory-mapped I/O register addresses for a hypothetical UART
const UART_BASE: usize = 0x1000_0000
const UART_DATA: usize = UART_BASE + 0x00
const UART_STATUS: usize = UART_BASE + 0x04
const UART_CONTROL: usize = UART_BASE + 0x08

const STATUS_TX_READY: u32 = 1 << 0
const STATUS_RX_READY: u32 = 1 << 1

// Volatile read: prevents compiler from optimizing away the read
unsafe fn read_volatile[T](addr: usize): T {
    let ptr = addr as *const T
    core.intrinsics.volatile_load(ptr)
}

// Volatile write: prevents compiler from optimizing away the write
unsafe fn write_volatile[T](addr: usize, value: T) {
    let ptr = addr as *mut T
    core.intrinsics.volatile_store(ptr, value)
}

// Safe wrapper for UART operations
pub fn uart_write_byte(byte: u8) {
    trusted("UART registers mapped at UART_BASE, hardware initialized") {
        // Wait for transmit buffer to be ready
        unsafe {
            while (read_volatile[u32](UART_STATUS) & STATUS_TX_READY) == 0 {
                // Spin
            }
            write_volatile[u32](UART_DATA, byte as u32)
        }
    }
}

pub fn uart_read_byte(): Option[u8] {
    trusted("UART registers mapped at UART_BASE, hardware initialized") {
        unsafe {
            if (read_volatile[u32](UART_STATUS) & STATUS_RX_READY) != 0 {
                Some(read_volatile[u32](UART_DATA) as u8)
            } else {
                None
            }
        }
    }
}
```

The `trusted` annotation documents the hardware setup assumption. The safe wrappers (`uart_write_byte`, `uart_read_byte`) hide the raw address arithmetic from callers.

### Example: Type Punning with Transmute

Sometimes you need to reinterpret the bytes of one type as another:

```ferrum
// Convert u32 to little-endian bytes
pub fn u32_to_le_bytes(value: u32): [u8; 4] {
    // On little-endian platforms, this is just a reinterpretation
    #[cfg(target_endian = "little")]
    {
        trusted("u32 and [u8; 4] have the same size and alignment on this platform") {
            unsafe { core.mem.transmute(value) }
        }
    }

    // On big-endian platforms, we need to swap
    #[cfg(target_endian = "big")]
    {
        [
            (value & 0xFF) as u8,
            ((value >> 8) & 0xFF) as u8,
            ((value >> 16) & 0xFF) as u8,
            ((value >> 24) & 0xFF) as u8,
        ]
    }
}

// Convert little-endian bytes to u32
pub fn le_bytes_to_u32(bytes: [u8; 4]): u32 {
    #[cfg(target_endian = "little")]
    {
        trusted("[u8; 4] and u32 have the same size and alignment on this platform") {
            unsafe { core.mem.transmute(bytes) }
        }
    }

    #[cfg(target_endian = "big")]
    {
        (bytes[0] as u32)
            | ((bytes[1] as u32) << 8)
            | ((bytes[2] as u32) << 16)
            | ((bytes[3] as u32) << 24)
    }
}
```

The `trusted` annotation documents the platform-specific assumption. The `#[cfg]` attribute ensures the transmute only happens when it's valid.

---

## The Safe Wrapper Pattern

Most unsafe code in practice follows this pattern:

1. **Internal implementation uses unsafe** for raw pointers, FFI, or hardware access
2. **Public API is completely safe** with Ferrum types and bounds checking
3. **Invariants are documented** in `trusted` annotations
4. **The module boundary is the safety boundary**

### Detailed Example: Implementing Vec

Here's a more complete implementation of `Vec` showing the pattern:

```ferrum
mod vec {
    type Vec[T] {
        ptr: *mut T,     // Raw pointer to heap allocation
        len: usize,      // Number of initialized elements
        cap: usize,      // Total capacity in elements
    }

    impl Vec[T] {
        // Safe: creates an empty Vec with no allocation
        pub fn new(): Vec[T] {
            Vec { ptr: core.ptr.null_mut(), len: 0, cap: 0 }
        }

        // Safe: creates a Vec with pre-allocated capacity
        pub fn with_capacity(capacity: usize): Vec[T] ! Alloc {
            if capacity == 0 {
                return Vec.new()
            }

            let ptr = Self.alloc_buffer(capacity)
            Vec { ptr: ptr, len: 0, cap: capacity }
        }

        // Safe: adds an element to the end
        pub fn push(&mut self, value: T) ! Alloc {
            if self.len == self.cap {
                self.grow()
            }

            trusted("len < cap after grow(), ptr valid for cap elements") {
                unsafe {
                    self.ptr.add(self.len).write(value)
                }
            }
            self.len += 1
        }

        // Safe: removes and returns the last element
        pub fn pop(&mut self): Option[T] {
            if self.len == 0 {
                return None
            }

            self.len -= 1
            trusted("len was > 0, so len is now a valid index; element was initialized by push") {
                Some(unsafe { self.ptr.add(self.len).read() })
            }
        }

        // Safe: returns a reference to an element, or None if out of bounds
        pub fn get(&self, index: usize): Option[&T] {
            if index >= self.len {
                return None
            }

            trusted("index < len verified above, ptr valid for len elements") {
                Some(unsafe { &*self.ptr.add(index) })
            }
        }

        // Safe: returns a mutable reference to an element
        pub fn get_mut(&mut self, index: usize): Option[&mut T] {
            if index >= self.len {
                return None
            }

            trusted("index < len verified above, ptr valid for len elements, &mut self ensures no aliasing") {
                Some(unsafe { &mut *self.ptr.add(index) })
            }
        }

        // Safe: returns a slice view of the contents
        pub fn as_slice(&self): &[T] {
            if self.len == 0 {
                return &[]
            }

            trusted("ptr valid for len elements, len > 0") {
                unsafe {
                    core.slice.from_raw_parts(self.ptr, self.len)
                }
            }
        }

        // Internal: allocates a buffer (unsafe helper)
        fn alloc_buffer(capacity: usize): *mut T ! Alloc {
            let layout = core.alloc.Layout.array[T](capacity)
            let ptr = core.alloc.alloc(layout) as *mut T

            if ptr.is_null() {
                panic("allocation failed")
            }

            ptr
        }

        // Internal: grows the buffer (unsafe helper)
        fn grow(&mut self) ! Alloc {
            let new_cap = if self.cap == 0 { 4 } else { self.cap * 2 }
            let new_ptr = Self.alloc_buffer(new_cap)

            if self.len > 0 {
                trusted("old ptr valid for len elements, new ptr valid for new_cap >= len elements") {
                    unsafe {
                        core.ptr.copy_nonoverlapping(self.ptr, new_ptr, self.len)
                    }
                }

                // Free old buffer
                let old_layout = core.alloc.Layout.array[T](self.cap)
                unsafe {
                    core.alloc.dealloc(self.ptr as *mut u8, old_layout)
                }
            }

            self.ptr = new_ptr
            self.cap = new_cap
        }
    }

    // Indexing operator: panics on out of bounds
    impl Index[usize] for Vec[T] {
        type Output = T

        fn index(&self, index: usize): &T {
            assert(index < self.len, "index out of bounds")

            trusted("index < len verified by assert") {
                unsafe { &*self.ptr.add(index) }
            }
        }
    }

    // Cleanup: drop all elements and free memory
    impl Drop for Vec[T] {
        fn drop(&mut self) {
            // Drop all elements in reverse order
            while self.len > 0 {
                self.len -= 1
                trusted("len was > 0, element exists") {
                    unsafe {
                        core.ptr.drop_in_place(self.ptr.add(self.len))
                    }
                }
            }

            // Free the buffer
            if self.cap > 0 {
                let layout = core.alloc.Layout.array[T](self.cap)
                unsafe {
                    core.alloc.dealloc(self.ptr as *mut u8, layout)
                }
            }
        }
    }
}
```

Notice the structure:
- **Public methods** (`push`, `pop`, `get`, `as_slice`) have safe signatures
- **Bounds checks** happen in safe code before entering `trusted`/`unsafe`
- **Each `trusted` block** documents exactly what invariant is being relied upon
- **Unsafe operations** are contained in small blocks with clear purpose
- **Drop** correctly cleans up all resources

Users of `Vec` write:

```ferrum
let mut v = Vec.new()
v.push(1)
v.push(2)
let x = v[0]      // bounds checked
let y = v.get(5)  // returns None, no panic
```

They never see `unsafe`. The module author maintains the invariants.

---

## Finding All Unsafe Code: The Audit Trail

Ferrum provides tooling to find every location where safety depends on human judgment.

### Basic Audit

```
$ ferrum audit src/

Summary:
  Total files scanned: 142
  unchecked blocks: 23
  trusted blocks: 47
  extern declarations: 12
  unsafe blocks: 31

Run with --details for full listing.
```

### Detailed Audit by Level

```
$ ferrum audit src/ --level unsafe

src/collections/vec.fe:48   unsafe  ptr.add(self.len).write(value)
src/collections/vec.fe:62   unsafe  self.ptr.add(self.len).read()
src/collections/vec.fe:103  unsafe  core.ptr.copy_nonoverlapping(...)
src/io/file.fe:87           unsafe  read(fd, buf.as_mut_ptr(), buf.len())
src/io/file.fe:112          unsafe  write(fd, buf.as_ptr(), buf.len())
...
```

### Audit Trusted Annotations

```
$ ferrum audit src/ --level trusted --show-reasons

src/collections/vec.fe:47
  trusted: "len < cap after grow(), ptr valid for cap elements"

src/collections/vec.fe:61
  trusted: "len was > 0, so len is now a valid index; element was initialized by push"

src/io/mmap.fe:43
  trusted: "mmap returned valid pointer for requested length, or MAP_FAILED"

src/crypto/aes.fe:89
  trusted: "key is exactly 16/24/32 bytes, iv is exactly 16 bytes (verified by type)"
```

### Filtering by Module

```
$ ferrum audit src/crypto/ --level trusted --level unsafe

src/crypto/aes.fe:89    trusted  "key is exactly 16/24/32 bytes..."
src/crypto/aes.fe:94    unsafe   transmute(key_bytes)
src/crypto/sha256.fe:67 trusted  "block is exactly 64 bytes..."
src/crypto/sha256.fe:71 unsafe   ptr::read_unaligned(...)
```

### Integration with CI

```yaml
# In your CI configuration
- name: Audit unsafe code
  run: |
    ferrum audit src/ --json > audit.json
    # Fail if unsafe count increased
    python scripts/check_audit.py audit.json
```

### What Auditors Look For

When reviewing `trusted` blocks, auditors verify:

1. **Is the claimed invariant actually true?** Does the code above the block actually establish the condition?
2. **Are all preconditions checked?** If the trusted block says "index < len verified above," is there actually an assert or if-check?
3. **Could the invariant be violated by concurrent access?** Is there proper synchronization?
4. **Does the reason string match what the code does?** Stale comments are bugs.

When reviewing `unsafe` blocks, auditors verify:

1. **Are raw pointers valid?** Where did they come from? Are they null-checked?
2. **Is the lifetime correct?** Does the pointed-to memory outlive the pointer?
3. **Are alignment requirements met?** Transmute and pointer casts can violate alignment.
4. **Is there proper cleanup?** Do Drop implementations free all resources?

---

## Decision Tree: Which Level Do I Need?

```
START: What are you trying to do?
       |
       +-- Skip a bounds/overflow check?
       |   |
       |   +-- Is the check actually in a hot path? (Did you profile?)
       |   |   |
       |   |   +-- NO --> Don't use unchecked. The compiler probably
       |   |   |          optimizes it away anyway.
       |   |   |
       |   |   +-- YES --> Can you prove the check always passes?
       |   |               |
       |   |               +-- NO --> Don't use unchecked. The check
       |   |               |          is protecting you from a bug.
       |   |               |
       |   |               +-- YES --> Use UNCHECKED
       |
       +-- Call a C function or syscall?
       |   |
       |   +-- Use EXTERN to declare it
       |   +-- Use UNSAFE to call it
       |   +-- Use TRUSTED to wrap it in a safe API (document your assumptions)
       |
       +-- Dereference a raw pointer?
       |   |
       |   +-- Use UNSAFE
       |   +-- Wrap in TRUSTED to document why the pointer is valid
       |
       +-- Transmute between types?
       |   |
       |   +-- Are the types the same size? Are alignment requirements met?
       |   |   |
       |   |   +-- NO --> Don't transmute. Use a safe conversion.
       |   |   |
       |   |   +-- YES --> Use UNSAFE inside TRUSTED
       |   |               (document why the transmute is valid)
       |
       +-- Access a mutable static?
       |   |
       |   +-- Use UNSAFE
       |   +-- Consider if you actually need global mutable state
       |       (usually there's a better design)
       |
       +-- Make an assertion the borrow checker rejects?
           |
           +-- Is the borrow checker wrong, or is your design wrong?
           |   |
           |   +-- DESIGN WRONG --> Refactor. Don't fight the borrow checker.
           |   |
           |   +-- BORROW CHECKER WRONG --> Use TRUSTED to document your
           |                                invariant, then UNSAFE if needed
```

### Quick Reference

| I want to... | Use this |
|--------------|----------|
| Skip a bounds check in a hot loop | `unchecked` |
| Skip overflow check when I know it won't overflow | `unchecked` |
| Call a C function | `extern` + `unsafe` |
| Wrap FFI in a safe API | `extern` + `unsafe` + `trusted` |
| Dereference a raw pointer | `unsafe` |
| Create two &mut refs to non-overlapping parts | `trusted` + `unsafe` |
| Reinterpret bytes as a different type | `trusted` + `unsafe` |
| Write to a hardware register | `unsafe` |
| Access global mutable state | `unsafe` (but reconsider your design) |

---

## Writing Unsafe Code Responsibly

### Rule 1: Profile Before You Optimize

Do not use `unchecked` because you "think" bounds checks are slow. Profile first.

```ferrum
// Don't do this
fn sum(data: &[i32]): i32 {
    let mut total = 0
    for i in 0..data.len() {
        unchecked { total += data[i] }  // "it's faster!"
    }
    total
}

// Do this instead
fn sum(data: &[i32]): i32 {
    data.iter().sum()  // Often just as fast, always safe
}
```

LLVM is very good at eliminating bounds checks when it can prove they are redundant. The safe version is often the same speed.

### Rule 2: Keep Unsafe Blocks Small

Every line of unsafe code is a line you must audit manually. Minimize the surface area.

```ferrum
// Bad: large unsafe block
unsafe {
    let ptr = data.as_mut_ptr()
    for i in 0..data.len() {
        let value = *ptr.add(i)
        let transformed = expensive_safe_computation(value)
        *ptr.add(i) = transformed
    }
}

// Good: unsafe only for the pointer operations
for i in 0..data.len() {
    let value = unsafe { *data.as_ptr().add(i) }
    let transformed = expensive_safe_computation(value)  // Safe
    unsafe { *data.as_mut_ptr().add(i) = transformed }
}
```

In the second version, the safe computation is outside the unsafe block. If there's a bug in `expensive_safe_computation`, it cannot cause memory unsafety.

### Rule 3: Document Every Invariant

Every `trusted` block should answer: "Why is this safe?" Be specific.

```ferrum
// Bad: vague
trusted("this is safe") { ... }

// Bad: describes what, not why
trusted("dereferencing pointer") { ... }

// Good: specific invariant
trusted("ptr came from Vec.as_mut_ptr(), index < len verified by assert on line 42") { ... }

// Good: documents the relationship
trusted("left [0..mid] and right [mid..len] are disjoint; mid <= len verified above") { ... }
```

### Rule 4: Test Edge Cases Aggressively

Unsafe code should have more tests, not fewer. Test:

- Empty inputs
- Maximum-size inputs
- Boundary conditions (index = 0, index = len - 1, index = len)
- Alignment edge cases
- Allocation failures (if you can simulate them)

```ferrum
#[test]
fn test_ring_buffer_full() {
    let mut rb = RingBuffer[i32].new(4)
    assert(rb.push(1))
    assert(rb.push(2))
    assert(rb.push(3))
    assert(!rb.push(4))  // Buffer full, should return false
    assert_eq(rb.len(), 3)
}

#[test]
fn test_ring_buffer_wrap() {
    let mut rb = RingBuffer[i32].new(4)
    rb.push(1); rb.push(2); rb.push(3)
    rb.pop(); rb.pop()  // head = 2
    rb.push(4); rb.push(5)  // tail wraps around
    assert_eq(rb.pop(), Some(3))
    assert_eq(rb.pop(), Some(4))
    assert_eq(rb.pop(), Some(5))
}
```

### Rule 5: Contain Unsafe in Modules

Unsafe code should be in dedicated modules with safe public APIs. Do not scatter `unsafe` throughout the codebase.

```
src/
  collections/
    vec.fe          # Vec implementation, unsafe internal
    ring_buffer.fe  # Ring buffer, unsafe internal
    mod.fe          # Re-exports only safe types and functions

  ffi/
    libc.fe         # extern declarations
    wrappers.fe     # safe wrappers around libc
    mod.fe          # Re-exports only safe wrappers

  app/
    main.fe         # Application code, no unsafe at all
    logic.fe        # Business logic, no unsafe at all
```

The `app/` directory has no `unsafe`, `trusted`, or `extern`. It uses the safe APIs from `collections/` and `ffi/`.

### Rule 6: Review Unsafe Code in PRs

Pull requests that add or modify unsafe code should get extra scrutiny:

- Does the `trusted` annotation accurately describe the invariant?
- Is the invariant actually established by the code?
- Are all edge cases handled?
- Is the unsafe block as small as possible?
- Is there a way to do this safely?

Many projects require multiple reviewers for unsafe code changes.

---

## Common Mistakes

### Mistake: Trusting Unverified Input

```ferrum
// WRONG: assumes index is valid
pub fn get_unchecked(&self, index: usize): &T {
    unchecked { &self.data[index] }
}
```

This function trusts the caller to provide a valid index. But callers might pass user input directly. Use `get()` which returns `Option`, or at minimum add `assert(index < self.len())`.

### Mistake: Stale Trusted Annotations

```ferrum
// Code was refactored, but the annotation wasn't updated
trusted("buffer valid for n bytes")  // Actually, we changed to using len!
fn process(buffer: &[u8], len: usize) {
    unsafe { ... use buffer.len(), not len ... }
}
```

The annotation says one thing, the code does another. This is a bug. Keep annotations synchronized with code.

### Mistake: Forgetting Cleanup

```ferrum
// WRONG: leaks memory if function returns early
fn process() {
    let ptr = alloc(1024)

    if condition {
        return  // Leaked!
    }

    // ... use ptr ...

    dealloc(ptr)
}
```

Use RAII (wrap the allocation in a type with Drop) or ensure all paths through the function clean up properly.

### Mistake: Violating Aliasing Rules

```ferrum
// WRONG: creates two mutable references to the same memory
let ptr = data.as_mut_ptr()
let ref1 = unsafe { &mut *ptr }
let ref2 = unsafe { &mut *ptr.add(0) }  // Same address!
// Using both ref1 and ref2 is undefined behavior
```

Even in unsafe code, you cannot have two `&mut` references to the same memory active at the same time.

### Mistake: Using Unsafe to Fight the Borrow Checker

```ferrum
// WRONG: using unsafe because the borrow checker is "annoying"
fn process_both(&mut self) {
    let a = unsafe { &mut *(&mut self.field_a as *mut _) }
    let b = unsafe { &mut *(&mut self.field_b as *mut _) }
    // "Now I can use both!"
}
```

The borrow checker is usually right. If you find yourself using unsafe to create multiple mutable references, consider:

1. Are the fields actually independent? Use separate methods.
2. Do you need both references simultaneously? Restructure the code.
3. Is there a safe pattern (like `split_at_mut`) that does what you need?

Only use unsafe for aliasing when you have proven the references are disjoint (like in `split_at_mut`) and documented the proof.

---

## Summary

Ferrum provides four levels of relaxed safety:

| Level | Keyword | When to Use |
|-------|---------|-------------|
| 1 | `unchecked` | Hot loops where you verified bounds externally (profile first!) |
| 2 | `trusted` | Wrapping unsafe code in safe APIs; documenting invariants |
| 3 | `extern` | Declaring C functions and syscalls |
| 4 | `unsafe` | Raw pointers, transmute, volatile access, FFI calls |

The key principles:

1. **Safe by default.** Most code should have no unsafe blocks.
2. **Explicit unsafety.** Every dangerous operation is syntactically marked.
3. **Documented invariants.** `trusted` annotations explain what you're claiming.
4. **Auditable boundaries.** `ferrum audit` finds every location requiring review.
5. **Contained unsafety.** Unsafe code lives in modules with safe public APIs.

In C, every line of code must be audited for memory safety. In Ferrum, you audit only the `unsafe`, `trusted`, `unchecked`, and `extern` blocks. Everything else is verified by the compiler.

The goal is not zero unsafe blocks. The goal is that every unsafe block is:
- **Justified:** There's no safe way to do this.
- **Documented:** The `trusted` annotation explains the invariant.
- **Contained:** The module exposes a safe API.
- **Reviewed:** Someone verified the invariant is actually maintained.
