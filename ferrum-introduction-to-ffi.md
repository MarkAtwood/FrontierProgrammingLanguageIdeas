# Introduction to FFI and C Interoperability in Ferrum

**Audience:** Programmers who know C and Python, familiar with ctypes or cffi

---

## Why FFI Exists

You have a million lines of working C code. Your operating system exposes C APIs. OpenSSL is written in C. SQLite is written in C. The graphics driver you need talks C. The hardware vendor ships a C library.

Rewriting all of this in Ferrum would take years. You don't have years. You need to call it now.

That's what FFI is for: calling C code from Ferrum, and letting C code call Ferrum functions. It's the escape hatch that lets Ferrum coexist with the existing software ecosystem.

### Common Use Cases

**Calling existing C libraries.** You want to use zlib for compression, OpenSSL for crypto, SQLite for an embedded database. These libraries work. They're tested. They're fast. Just call them.

**Using OS APIs.** POSIX defines `open()`, `read()`, `write()`, `mmap()`. Windows has `CreateFile()`, `ReadFile()`. These are C functions. Every systems language needs to call them.

**Gradual migration.** You have a C codebase. You want to write new code in Ferrum, but you can't rewrite everything at once. FFI lets you migrate piece by piece: wrap the C functions, call them from Ferrum, replace them one at a time as you go.

**Performance-critical libraries.** Sometimes a heavily-optimized C library (BLAS, FFTW, video codecs) is faster than anything you could write. Use it.

---

## If You've Used Python's ctypes or cffi

You know the basics already. You declare the function signature, load the library, call the function. The mechanics in Ferrum are similar, but with one massive difference: **the compiler catches your mistakes**.

In Python:

```python
from ctypes import *

libc = CDLL("libc.so.6")
libc.printf(b"Hello, %s!\n", b"world")  # works

libc.printf(b"Hello, %d!\n", b"world")  # passes a pointer where int expected
                                         # runtime corruption, no error
```

Python's ctypes doesn't know what types `printf` expects. You tell it nothing, it believes you, and if you lie, you corrupt memory or crash. Maybe not immediately. Maybe three hours later, in an unrelated function, when the heap corruption finally manifests.

In Ferrum:

```ferrum
extern "C" fn printf(fmt: *const c_char, ...): c_int  ! Unsafe

// This function declaration tells the compiler:
// - printf uses C calling convention
// - First argument is a pointer to a null-terminated C string
// - It's variadic (takes additional arguments)
// - It returns a C int
// - Calling it requires the Unsafe effect
```

The types are checked at compile time. Pass the wrong type, get a compile error. The runtime segfault that ruined your Thursday afternoon in Python becomes a compiler error you fix in five seconds.

---

## Declaring External Functions

The basic syntax for declaring a C function you want to call:

```ferrum
extern "C" fn function_name(params): return_type  ! Unsafe
```

### Simple Functions

```ferrum
// <stdlib.h>
extern "C" fn abs(n: c_int): c_int  ! Unsafe
extern "C" fn exit(status: c_int): never  ! Unsafe
extern "C" fn malloc(size: c_size_t): *mut c_void  ! Unsafe
extern "C" fn free(ptr: *mut c_void)  ! Unsafe

// <string.h>
extern "C" fn strlen(s: *const c_char): c_size_t  ! Unsafe
extern "C" fn memcpy(dst: *mut c_void, src: *const c_void, n: c_size_t): *mut c_void  ! Unsafe

// <math.h>
extern "C" fn sqrt(x: f64): f64  ! Unsafe
extern "C" fn sin(x: f64): f64  ! Unsafe
```

### Variadic Functions

C has variadic functions like `printf`. Ferrum represents these with `...`:

```ferrum
extern "C" fn printf(fmt: *const c_char, ...): c_int  ! Unsafe
extern "C" fn sprintf(buf: *mut c_char, fmt: *const c_char, ...): c_int  ! Unsafe
extern "C" fn snprintf(buf: *mut c_char, size: c_size_t, fmt: *const c_char, ...): c_int  ! Unsafe
```

When you call a variadic function, each argument after `...` must be a type that C understands: integers, floats, pointers. Not Ferrum structs, not references, not strings.

### Grouping Declarations

If you have many functions from the same library, group them:

```ferrum
extern "C" {
    fn strlen(s: *const c_char): c_size_t  ! Unsafe
    fn strcmp(s1: *const c_char, s2: *const c_char): c_int  ! Unsafe
    fn strcpy(dst: *mut c_char, src: *const c_char): *mut c_char  ! Unsafe
}
```

---

## C Types: Platform-Correct Sizes

C's `int` isn't always 32 bits. `long` is 64 bits on Linux but 32 bits on Windows. `char` might be signed or unsigned. Ferrum provides types that match the C types on each platform:

```ferrum
import core.ffi.{c_int, c_uint, c_long, c_ulong, c_char, c_void, c_size_t}
```

### The Type Mapping

| C type | Ferrum type | Notes |
|--------|-------------|-------|
| `char` | `c_char` | i8 on most platforms, signedness varies |
| `signed char` | `i8` | Always signed |
| `unsigned char` | `u8` | Always unsigned |
| `short` | `i16` | |
| `unsigned short` | `u16` | |
| `int` | `c_int` | i32 on all modern platforms |
| `unsigned int` | `c_uint` | u32 |
| `long` | `c_long` | i64 on Unix LP64, i32 on Windows LLP64 |
| `unsigned long` | `c_ulong` | Same split as `long` |
| `long long` | `i64` | |
| `unsigned long long` | `u64` | |
| `size_t` | `c_size_t` | usize |
| `ssize_t` | `c_ssize_t` | isize |
| `ptrdiff_t` | `c_ptrdiff_t` | isize |
| `void*` | `*mut c_void` | |
| `const void*` | `*const c_void` | |
| `float` | `f32` | |
| `double` | `f64` | |

### Why Not Just Use i32 for int?

Because `c_int` carries semantic meaning. When you see `c_int`, you know "this crosses an FFI boundary." When you see `i32`, you don't know if it's FFI or internal Ferrum code.

Also, the compiler can warn if you mix them incorrectly:

```ferrum
extern "C" fn some_c_function(x: c_int): c_int  ! Unsafe

fn ferrum_code() {
    let x: i32 = 42
    let result = unsafe { some_c_function(x) }  // implicit conversion, compiler warns
    let result = unsafe { some_c_function(x as c_int) }  // explicit, no warning
}
```

---

## Pointers: Raw Pointers for C Interop

Ferrum has two kinds of pointers for FFI:

- `*const T` - raw const pointer (like `const T*` in C)
- `*mut T` - raw mutable pointer (like `T*` in C)

These are **not** the same as references (`&T`, `&mut T`). Raw pointers:

- Can be null
- Can dangle (point to freed memory)
- Can alias (multiple mutable pointers to the same data)
- Are not tracked by the borrow checker
- Require `unsafe` to dereference

### Creating Raw Pointers

```ferrum
// From references (safe — the reference is valid)
let x: i32 = 42
let ptr: *const i32 = &x as *const i32
let mut_ptr: *mut i32 = &mut x as *mut i32

// Null pointers
let null: *const i32 = core.ptr.null()
let null_mut: *mut i32 = core.ptr.null_mut()
```

### Dereferencing Raw Pointers

Dereferencing a raw pointer is unsafe. You're asserting the pointer is valid.

```ferrum
unsafe {
    let value = *ptr       // read
    *mut_ptr = 100         // write
}
```

### Pointer Arithmetic

```ferrum
let arr: [i32; 5] = [1, 2, 3, 4, 5]
let ptr = arr.as_ptr()

unsafe {
    let second = *ptr.add(1)   // arr[1]
    let third = *ptr.add(2)    // arr[2]

    // offset is signed (can go backwards)
    let before = ptr.offset(-1)  // UB if this goes before the array
}
```

### Checking for Null

```ferrum
let ptr: *const i32 = get_pointer()

if ptr.is_null() {
    // handle null case
} else {
    unsafe {
        let value = *ptr
    }
}
```

---

## Strings: The FFI Nightmare Made Manageable

Strings are where C and Ferrum disagree completely.

| Aspect | C strings | Ferrum strings |
|--------|-----------|----------------|
| Encoding | "Whatever" (usually ASCII or UTF-8) | UTF-8, always |
| Length | Computed by scanning for `\0` | Stored in the string |
| Null terminator | Required | Not present |
| Interior nulls | Impossible (terminates string) | Allowed |
| Mutability | Depends on declaration | Explicit (&str vs &mut str) |

Ferrum provides two types for C strings:

### CStr: Borrowed C String

`CStr` is like `&str` for C strings. It doesn't own the memory; it borrows it.

```ferrum
import core.ffi.CStr

// Create from a literal (compile-time checked)
let greeting: &CStr = c"Hello, World!"

// Create from a pointer (unsafe - you vouch for validity)
unsafe {
    let s: &CStr = CStr.from_ptr(some_c_char_ptr)
}

// Create from bytes (checked)
let bytes: [u8; 6] = [b'h', b'e', b'l', b'l', b'o', 0]
let s: &CStr = CStr.from_bytes_with_nul(&bytes)?

// Use it
extern "C" fn puts(s: *const c_char): c_int  ! Unsafe

unsafe {
    puts(greeting.as_ptr())
}
```

### CString: Owned C String

`CString` is like `String` for C strings. It owns the memory.

```ferrum
import alloc.ffi.CString

// Create from a Ferrum string
let s = CString.new("Hello, World!")?  // ? because the string might contain interior nulls

// Create from bytes
let s = CString.new(vec![b'h', b'e', b'l', b'l', b'o'])?

// Use it
extern "C" fn puts(s: *const c_char): c_int  ! Unsafe

unsafe {
    puts(s.as_ptr())
}

// Convert back to Ferrum string (if valid UTF-8)
let ferrum_str: &str = s.to_str()?
```

### Converting Between String Types

```ferrum
// Ferrum &str to CString (copies, adds null terminator)
let ferrum: &str = "hello"
let c: CString = CString.new(ferrum)?

// CStr to Ferrum &str (zero-copy if valid UTF-8)
let c: &CStr = c"hello"
let ferrum: &str = c.to_str()?

// CStr to String (always works, replaces invalid UTF-8)
let lossy: String = c.to_string_lossy().into_owned()
```

### The Interior Null Problem

C strings can't contain `\0` except at the end. If your Ferrum string has a null byte in the middle, you can't convert it to a CString:

```ferrum
let bad = CString.new("hello\0world")
// Returns Err(NulError { position: 5 })
```

---

## Matching C Struct Layouts: @repr(C)

Ferrum structs don't have the same memory layout as C structs. Ferrum can reorder fields, add padding differently, or use different alignment. If you need to pass a struct to C, you must tell Ferrum to use C's layout:

```ferrum
@repr(C)
type Point {
    x: f64,
    y: f64,
}

@repr(C)
type Person {
    name: *const c_char,
    age: c_int,
    height: f32,
}
```

### What @repr(C) Guarantees

1. Fields are in declaration order (Ferrum normally may reorder)
2. Padding is added where C would add padding
3. Alignment matches C's alignment rules
4. Total size matches what C's `sizeof` would return

### When You Need @repr(C)

- Passing structs to C functions
- Receiving structs from C functions
- Laying out data that C code will read/write
- Memory-mapped I/O with hardware (which expects C layout)
- File formats defined in terms of C structs

### When You Don't Need @repr(C)

- Structs that stay entirely within Ferrum code
- Ferrum's optimizer can pack them more efficiently if you don't force C layout

### The @layout Attribute for Bit-Level Control

For hardware registers, network protocols, or file formats where you need exact bit-level layout:

```ferrum
@layout
type TcpHeader {
    source_port:      u16,
    dest_port:        u16,
    sequence_num:     u32,
    ack_num:          u32,
    data_offset:      u4,     // 4 bits
    reserved:         u3,     // 3 bits
    flags:            u9,     // 9 bits
    window_size:      u16,
    checksum:         u16,
    urgent_pointer:   u16,
}
```

---

## Calling C from Ferrum

### Linking to C Libraries

Specify the library to link against:

```ferrum
@link(name = "z")          // links -lz (zlib)
extern "C" {
    fn compress(
        dest: *mut u8,
        dest_len: *mut c_ulong,
        source: *const u8,
        source_len: c_ulong,
    ): c_int  ! Unsafe

    fn uncompress(
        dest: *mut u8,
        dest_len: *mut c_ulong,
        source: *const u8,
        source_len: c_ulong,
    ): c_int  ! Unsafe
}
```

### System Libraries

```ferrum
@link(name = "c")          // libc
@link(name = "m")          // libm (math)
@link(name = "pthread")    // POSIX threads
@link(name = "ssl")        // OpenSSL
@link(name = "sqlite3")    // SQLite
```

### Static vs Dynamic Linking

```ferrum
@link(name = "foo", kind = "static")   // link libfoo.a
@link(name = "bar", kind = "dylib")    // link libbar.so / libbar.dylib
@link(name = "baz", kind = "framework") // macOS framework
```

### All FFI is Unsafe

Every call to an extern function requires unsafe:

```ferrum
extern "C" fn abs(n: c_int): c_int  ! Unsafe

fn main() {
    let x: c_int = -5

    // This doesn't compile:
    // let y = abs(x)

    // This does:
    let y = unsafe { abs(x) }
}
```

Why? Because the compiler can't verify what C code does. The C function might:
- Dereference a null pointer you passed
- Write past the end of a buffer
- Free memory you're still using
- Have undefined behavior for certain inputs
- Not be thread-safe

The `unsafe` block is you saying "I've read the docs for this C function. I understand its requirements. I'm calling it correctly."

---

## Exposing Ferrum to C

Sometimes you need C code to call Ferrum functions. This is common for:
- Callback functions passed to C libraries
- Writing a library in Ferrum that C code will use
- Embedding Ferrum in a C application

### Making a Function Callable from C

```ferrum
@no_mangle
extern "C" fn my_callback(data: *const c_void, len: c_size_t): c_int {
    // This function can be called from C
    // ...
    0
}
```

`@no_mangle` prevents Ferrum from mangling the function name. Without it, the function might be named something like `_ZN4mylib11my_callback17h5e6f7a8b9c0d1e2fE` in the binary.

`extern "C"` makes the function use C calling convention, so C code knows how to call it.

### Exposing Types to C

You can generate C header files for your Ferrum types:

```ferrum
@repr(C)
@export_c_header
type MyStruct {
    value: c_int,
    data: *mut c_void,
}

@no_mangle
@export_c_header
extern "C" fn my_struct_new(): *mut MyStruct { ... }

@no_mangle
@export_c_header
extern "C" fn my_struct_free(s: *mut MyStruct) { ... }
```

Running `ferrum build --emit=c-header` generates:

```c
/* Generated by Ferrum - do not edit */

typedef struct MyStruct {
    int value;
    void* data;
} MyStruct;

MyStruct* my_struct_new(void);
void my_struct_free(MyStruct* s);
```

---

## Wrapping a C Library: A Complete Example

Let's wrap a simple C library properly. We'll use a hypothetical "counter" library:

```c
// counter.h
typedef struct Counter Counter;

Counter* counter_new(int initial_value);
void counter_free(Counter* c);
int counter_get(const Counter* c);
void counter_increment(Counter* c);
void counter_add(Counter* c, int amount);
```

### Step 1: Raw FFI Bindings

First, create low-level bindings that match the C API exactly:

```ferrum
// counter_ffi.fe - raw bindings, all unsafe

import core.ffi.c_int

// Opaque type - we don't know what's inside, and we shouldn't
@repr(C)
type Counter {
    _opaque: [u8; 0],  // zero-sized, can't be instantiated directly
}

@link(name = "counter")
extern "C" {
    fn counter_new(initial_value: c_int): *mut Counter  ! Unsafe
    fn counter_free(c: *mut Counter)  ! Unsafe
    fn counter_get(c: *const Counter): c_int  ! Unsafe
    fn counter_increment(c: *mut Counter)  ! Unsafe
    fn counter_add(c: *mut Counter, amount: c_int)  ! Unsafe
}
```

### Step 2: Safe Wrapper

Now wrap it in a safe Ferrum API:

```ferrum
// counter.fe - safe wrapper

import counter_ffi

/// A counter that can be incremented.
///
/// Wraps the C `counter` library with a safe Ferrum interface.
pub type SafeCounter {
    ptr: *mut counter_ffi.Counter,
}

impl SafeCounter {
    /// Creates a new counter with the given initial value.
    pub fn new(initial: i32): Result[Self, CounterError] {
        let ptr = unsafe { counter_ffi.counter_new(initial as c_int) }
        if ptr.is_null() {
            Err(CounterError.AllocationFailed)
        } else {
            Ok(SafeCounter { ptr })
        }
    }

    /// Returns the current value.
    pub fn get(&self): i32 {
        unsafe { counter_ffi.counter_get(self.ptr) as i32 }
    }

    /// Increments the counter by 1.
    pub fn increment(&mut self) {
        unsafe { counter_ffi.counter_increment(self.ptr) }
    }

    /// Adds the given amount to the counter.
    pub fn add(&mut self, amount: i32) {
        unsafe { counter_ffi.counter_add(self.ptr, amount as c_int) }
    }
}

impl Drop for SafeCounter {
    fn drop(&mut self) {
        if !self.ptr.is_null() {
            unsafe { counter_ffi.counter_free(self.ptr) }
        }
    }
}

pub enum CounterError {
    AllocationFailed,
}
```

### Step 3: Use It Safely

Now users of your library never see the unsafe code:

```ferrum
import counter.SafeCounter

fn main(): Result[(), CounterError] {
    let mut c = SafeCounter.new(0)?

    c.increment()
    c.increment()
    c.add(10)

    print("Counter value: {}", c.get())  // prints 12

    Ok(())
}   // c is dropped here, counter_free is called automatically
```

No `unsafe` in sight. The memory management is automatic. The API is idiomatic Ferrum.

---

## Passing Callbacks to C

C libraries often take callback functions. Here's how to handle them:

```c
// In C
typedef void (*Callback)(void* user_data, int value);
void register_callback(Callback cb, void* user_data);
void trigger_callbacks(void);
```

```ferrum
// FFI bindings
type Callback = extern "C" fn(*mut c_void, c_int)

extern "C" {
    fn register_callback(cb: Callback, user_data: *mut c_void)  ! Unsafe
    fn trigger_callbacks()  ! Unsafe
}

// Using it
@no_mangle
extern "C" fn my_callback(user_data: *mut c_void, value: c_int) {
    // Recover the Ferrum data from the void pointer
    let data: &mut MyData = unsafe { &mut *(user_data as *mut MyData) }
    data.handle_value(value as i32)
}

fn setup() {
    let mut data = MyData.new()

    unsafe {
        register_callback(
            my_callback,
            &mut data as *mut MyData as *mut c_void,
        )
    }

    // IMPORTANT: data must stay alive as long as the callback might be called!
    // If data is dropped, the callback will receive a dangling pointer.

    unsafe { trigger_callbacks() }
}
```

### The Lifetime Problem with Callbacks

This is one of the most dangerous patterns in FFI. The C library holds a pointer to your data. If your data is dropped while the callback is still registered, the callback gets a dangling pointer.

Safe solutions:

1. **Pin the data** so it can't be moved or dropped while the callback is registered
2. **Box and leak** if the data needs to live forever
3. **Reference counting** with Arc if you need shared ownership
4. **Unregister before drop** - implement Drop to unregister the callback

---

## Common Pitfalls

### Null Pointers

C APIs often return null to indicate errors. Always check:

```ferrum
let ptr = unsafe { c_function_that_might_fail() }
if ptr.is_null() {
    return Err(MyError.NullPointer)
}
```

### Dangling Pointers

The C library might free memory that you still have a pointer to:

```ferrum
let ptr = unsafe { get_internal_buffer(handle) }
unsafe { destroy_handle(handle) }  // might free the buffer
unsafe { *ptr }  // DANGLING POINTER - undefined behavior
```

Solution: understand the C library's ownership model. Read the docs.

### Memory Ownership Across the Boundary

Who owns memory that crosses the FFI boundary?

**C allocates, Ferrum reads:**
```ferrum
let ptr = unsafe { c_allocate() }
// ... use ptr ...
unsafe { c_free(ptr) }  // C allocated it, C must free it
```

**Ferrum allocates, C reads:**
```ferrum
let s = CString.new("hello")?
unsafe { c_function(s.as_ptr()) }
// s is still owned by Ferrum, will be freed when s is dropped
// If C stored the pointer, you have a problem when s is dropped!
```

**Transfer ownership:**
```ferrum
let s = CString.new("hello")?
let ptr = s.into_raw()  // Ferrum gives up ownership
unsafe { c_function_that_takes_ownership(ptr) }
// Now C owns it. Do NOT drop s. Do NOT use ptr after this.
```

### Thread Safety

Most C libraries are not thread-safe. Ferrum can't know this. If you call non-thread-safe C code from multiple threads, you get data races that the compiler can't prevent.

Read the docs. If the C library isn't thread-safe, wrap it in a Mutex:

```ferrum
type ThreadSafeWrapper {
    inner: Mutex[UnsafeCLibrary],
}

impl ThreadSafeWrapper {
    fn do_thing(&self) {
        let guard = self.inner.lock()
        unsafe { c_do_thing(guard.ptr) }
    }
}
```

### String Encoding

C doesn't know about UTF-8. If you pass invalid UTF-8 to a C library that expects UTF-8, behavior is undefined. If the C library returns bytes that aren't valid UTF-8, `to_str()` will fail.

Always use `to_string_lossy()` if you're not sure:

```ferrum
let c_str: &CStr = unsafe { CStr.from_ptr(ptr) }
let s: String = c_str.to_string_lossy().into_owned()  // replaces invalid UTF-8 with U+FFFD
```

---

## Comparing to Python's FFI

| Aspect | Python ctypes/cffi | Ferrum FFI |
|--------|-------------------|------------|
| Type declarations | Runtime strings (`"int"`) | Compile-time types (`c_int`) |
| Type checking | Runtime (if at all) | Compile-time |
| Wrong types | Silent corruption or crash | Compile error |
| Null pointer deref | Segfault at runtime | Must check, or UB in unsafe |
| Memory management | Manual (easy to leak) | RAII through wrappers |
| Callbacks | Easy but dangerous | Same dangers, more explicit |
| Performance | Significant overhead | Zero overhead (same as C) |
| Safety | None | Explicit unsafe boundaries |

### The Python Experience

```python
from ctypes import *

# You declare the signature... or don't. ctypes doesn't care.
libc = CDLL("libc.so.6")
libc.strlen(b"hello")  # works, returns 5

# Forget the argtypes? Python guesses (badly).
libc.printf(b"Value: %d\n", "not an int")  # crashes or prints garbage
```

### The Ferrum Experience

```ferrum
extern "C" fn strlen(s: *const c_char): c_size_t  ! Unsafe
extern "C" fn printf(fmt: *const c_char, ...): c_int  ! Unsafe

fn main() {
    unsafe {
        strlen(c"hello".as_ptr())  // compiles, works

        // This doesn't compile:
        // printf(c"Value: %d\n".as_ptr(), "not an int")
        // error: expected c_int, found &str
    }
}
```

The Ferrum compiler knows the types. It catches mistakes. You still need unsafe for FFI, but the type system is still working.

---

## Quick Reference

### Type Mappings

```ferrum
import core.ffi.{c_int, c_uint, c_long, c_ulong, c_char, c_void, c_size_t, c_ssize_t}
```

### String Types

```ferrum
import core.ffi.CStr       // borrowed C string (no allocation)
import alloc.ffi.CString   // owned C string (allocates)

let literal: &CStr = c"hello"                // compile-time literal
let owned: CString = CString.new("hello")?   // runtime allocation
```

### Declaring External Functions

```ferrum
extern "C" fn name(args): return_type  ! Unsafe

@link(name = "library_name")
extern "C" {
    fn func1(...): ...  ! Unsafe
    fn func2(...): ...  ! Unsafe
}
```

### Making Ferrum Functions Callable from C

```ferrum
@no_mangle
extern "C" fn callable_from_c(args): return_type {
    // body
}
```

### Struct Layout for C Interop

```ferrum
@repr(C)                    // C-compatible layout
type MyCStruct { ... }

@layout                     // bit-level control
type MyBitFields { ... }
```

### Pointer Operations

```ferrum
// Creating
let ptr: *const T = &x as *const T
let ptr: *mut T = &mut x as *mut T
let null: *const T = core.ptr.null()

// Using
unsafe {
    *ptr              // dereference
    ptr.add(n)        // offset by n elements
    ptr.is_null()     // check for null (safe, doesn't deref)
}
```

---

## Summary

FFI is the bridge between Ferrum and the C ecosystem. It gives you:

- Access to existing C libraries without rewriting them
- OS APIs exactly as the OS exposes them
- A path for gradual migration from C

The cost:

- You must understand the C library's memory model
- You must handle null pointers, dangling pointers, ownership
- The compiler can't verify C code's behavior

The benefit over Python's ctypes:

- Type mismatches caught at compile time
- Explicit unsafe boundaries
- RAII wrappers for automatic cleanup
- Zero runtime overhead

Write the thin `extern "C"` layer, wrap it in safe Ferrum, and users of your wrapper never need to touch unsafe code. The unsafe boundary is narrow, explicit, and auditable.

---

*See also:*
- *[Ferrum Standard Library - core.ffi](ferrum-stdlib-core.md#310-coreffi--foreign-function-interface-types) for complete CStr/CString reference*
- *[Ferrum Standard Library - alloc.ffi](ferrum-stdlib-alloc.md#43-allocffi--owned-ffi-strings) for owned FFI string types*
- *[Ferrum Language Reference - Safety Levels](ferrum-lang-verification.md) for unsafe, trusted, and extern semantics*
