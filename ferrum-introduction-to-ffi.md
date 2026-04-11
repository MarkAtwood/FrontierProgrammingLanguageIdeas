# Introduction to FFI and C Interoperability in Ferrum

**Audience:** Programmers who know C well and have used Python's ctypes or cffi. Not a CS academic. You want to call C libraries from Ferrum, and you want to know what can go wrong.

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

### Python: Runtime Errors (Maybe)

```python
from ctypes import *

libc = CDLL("libc.so.6")
libc.printf(b"Hello, %s!\n", b"world")  # works

libc.printf(b"Hello, %d!\n", b"world")  # passes a pointer where int expected
                                         # runtime corruption, no error
```

Python's ctypes doesn't know what types `printf` expects. You tell it nothing, it believes you, and if you lie, you corrupt memory or crash. Maybe not immediately. Maybe three hours later, in an unrelated function, when the heap corruption finally manifests.

With cffi, you can declare types, but mistakes are still runtime errors:

```python
from cffi import FFI
ffi = FFI()
ffi.cdef("int strlen(const char* s);")
lib = ffi.dlopen(None)

lib.strlen("hello")      # works
lib.strlen(42)           # TypeError at runtime
lib.strlen(None)         # segfault
```

### Ferrum: Compile-Time Errors

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

### Side-by-Side Comparison

| What You Do | Python ctypes | Ferrum |
|-------------|---------------|--------|
| Wrong argument type | Compiles, crashes at runtime (maybe) | Compile error |
| Missing argument | Compiles, undefined behavior | Compile error |
| Wrong return type | Compiles, garbage value or crash | Compile error |
| Pass null where non-null expected | Segfault | Must be handled in unsafe block |
| Forget to free memory | Memory leak, no warning | RAII wrapper handles it |
| Call non-thread-safe code from threads | Data race, silent corruption | You still have this problem |

The last row is important: **FFI can't make C code safer than it is.** If the C library has bugs, undefined behavior, or thread-safety issues, Ferrum can't fix that. But Ferrum can prevent you from making *new* mistakes at the boundary.

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

```ferrum
// What you CAN pass to variadic args:
unsafe {
    printf(c"%d %f %p\n".as_ptr(), 42 as c_int, 3.14, some_ptr)
}

// What you CANNOT pass:
unsafe {
    printf(c"%s\n".as_ptr(), "hello")  // error: &str is not C-compatible
    printf(c"%d\n".as_ptr(), my_struct) // error: Ferrum struct
}
```

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

### What the Compiler Tells You

If you get the types wrong:

```ferrum
extern "C" fn process(n: c_int): c_int  ! Unsafe

fn main() {
    let big: i64 = 1_000_000_000_000
    unsafe { process(big) }
}
```

Compiler error:

```
error[E0308]: mismatched types
 --> src/main.fe:5:22
  |
5 |     unsafe { process(big) }
  |              ------- ^^^ expected `c_int`, found `i64`
  |              |
  |              arguments to this function are incorrect
  |
  = note: c_int is i32 on this platform
help: consider using `as` for a truncating conversion
  |
5 |     unsafe { process(big as c_int) }
  |                          +++++++++
```

Compare to Python ctypes, where this would silently truncate or corrupt memory.

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

### What Happens When You Dereference a Bad Pointer

In C, dereferencing a null or dangling pointer is undefined behavior. The same is true in Ferrum's unsafe blocks. Here's what you might see:

```ferrum
fn dangerous() {
    let null: *const i32 = core.ptr.null()
    unsafe {
        let value = *null  // undefined behavior
    }
}
```

If you're lucky: immediate segfault with a stack trace pointing to the line.

If you're unlucky: the program continues with garbage data, corrupts something else, and crashes hours later in unrelated code.

This is why you always check for null:

```ferrum
let ptr: *const i32 = get_pointer()

if ptr.is_null() {
    return Err(MyError.NullPointer)
}

// Now safe to dereference (but still requires unsafe block)
unsafe {
    let value = *ptr
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

---

## Strings: The FFI Nightmare Made Manageable

Strings are where C and Ferrum disagree completely. This is the #1 source of FFI bugs. Read this section carefully.

### The Fundamental Difference

| Aspect | C strings | Ferrum strings |
|--------|-----------|----------------|
| Encoding | "Whatever" (usually ASCII or UTF-8) | UTF-8, always |
| Length | Computed by scanning for `\0` | Stored in the string |
| Null terminator | Required | Not present |
| Interior nulls | Impossible (terminates string) | Allowed |
| Mutability | Depends on declaration | Explicit (&str vs &mut str) |

C strings end with a null byte (`\0`). Ferrum strings don't have a null terminator; they store their length separately. This means you can't just pass a Ferrum `&str` to a C function expecting `char*`.

### The Two String Types: CStr and CString

Ferrum provides two types for C strings. Understanding when to use each is critical.

**`CStr`** - Borrowed C string. Like `&str` for Ferrum strings. You don't own the memory; you're borrowing it from somewhere else.

**`CString`** - Owned C string. Like `String` for Ferrum strings. You own the memory and are responsible for it (but Ferrum handles cleanup automatically via Drop).

### Memory Ownership: Who Frees What?

This is the most important thing to understand about FFI strings.

```
┌────────────────────────────────────────────────────────────────────┐
│                     STRING MEMORY OWNERSHIP                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  CStr (&CStr)           CString                                    │
│  ┌─────────────┐        ┌─────────────┐                            │
│  │ ptr ────────┼───┐    │ ptr ────────┼───┐                        │
│  │ len         │   │    │ len         │   │                        │
│  └─────────────┘   │    │ cap         │   │                        │
│                    │    └─────────────┘   │                        │
│                    │                      │                        │
│                    ▼                      ▼                        │
│            ┌───────────────┐      ┌───────────────┐                │
│            │ h │ i │ \0 │  │      │ h │ i │ \0 │  │                │
│            └───────────────┘      └───────────────┘                │
│            Memory owned by        Memory owned by                  │
│            SOMEONE ELSE           THIS CString                     │
│            (C code, static        (will be freed                   │
│            data, etc.)            when CString drops)              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### CStr: Borrowed C String

Use `CStr` when:
- You have a compile-time string literal you want to pass to C
- A C function returned a `char*` that C still owns
- You're looking at memory that someone else will free

```ferrum
import core.ffi.CStr

// Create from a literal (compile-time checked, zero runtime cost)
let greeting: &CStr = c"Hello, World!"

// The c"..." syntax:
// - Adds a null terminator automatically
// - Verifies no interior nulls at compile time
// - Creates a static string (lives forever, no allocation)
```

Creating `CStr` from a C pointer:

```ferrum
extern "C" fn getenv(name: *const c_char): *const c_char  ! Unsafe

fn get_home_dir(): Option[&CStr] {
    let ptr = unsafe { getenv(c"HOME".as_ptr()) }

    if ptr.is_null() {
        return None
    }

    // SAFETY: getenv returns a pointer to static environment data
    // that remains valid for the lifetime of the process.
    Some(unsafe { CStr.from_ptr(ptr) })
}
```

**Warning:** `CStr.from_ptr()` is unsafe because you're vouching that:
1. The pointer is not null
2. The pointer points to a null-terminated string
3. The memory won't be freed while the `CStr` exists

If any of these are false, you get undefined behavior.

### CString: Owned C String

Use `CString` when:
- You have a Ferrum `String` or `&str` you need to pass to C
- You're creating a new string at runtime to pass to C
- You need to own the string and control its lifetime

```ferrum
import alloc.ffi.CString

// Create from a Ferrum string
let name = "Alice"
let c_name = CString.new(name)?  // ? because might contain interior null

// Use it
extern "C" fn greet(name: *const c_char)  ! Unsafe

unsafe {
    greet(c_name.as_ptr())  // passes the pointer to C
}

// c_name still owns the memory
// When c_name goes out of scope, the memory is freed
```

### The Interior Null Problem

C strings can't contain `\0` except at the end. If your Ferrum string has a null byte in the middle, you can't convert it to a CString:

```ferrum
let bad = CString.new("hello\0world")
// Returns Err(NulError { position: 5 })

// Why? Because C would see: "hello" (5 characters)
// The "world" part would be invisible, unreachable.
```

If you have data with embedded nulls, you need to pass it as a byte buffer with an explicit length, not as a C string.

### Converting Back: CStr to Ferrum Strings

```ferrum
let c: &CStr = c"hello"

// If you KNOW it's valid UTF-8:
let s: &str = c.to_str()?  // Returns Err if invalid UTF-8

// If you're not sure (replaces invalid sequences with U+FFFD):
let s: String = c.to_string_lossy().into_owned()
```

### Common String Mistakes

**Mistake 1: Using a CString after it's dropped**

```ferrum
fn get_name_ptr(): *const c_char {
    let name = CString.new("Alice").unwrap()
    name.as_ptr()  // Returns pointer to name's buffer
}   // name is dropped here, buffer is freed

fn main() {
    let ptr = get_name_ptr()  // ptr is now dangling!
    unsafe { puts(ptr) }       // undefined behavior
}
```

The compiler can't catch this because raw pointers aren't tracked. The fix:

```ferrum
fn greet_user() {
    let name = CString.new("Alice").unwrap()
    unsafe {
        greet(name.as_ptr())
        // name is still alive here, ptr is valid
    }
}   // name dropped after greet() returns
```

**Mistake 2: Passing &str directly to C**

```ferrum
extern "C" fn puts(s: *const c_char): c_int  ! Unsafe

fn bad() {
    let s = "hello"
    unsafe {
        puts(s.as_ptr() as *const c_char)  // WRONG!
    }
}
```

This compiles but is wrong: Ferrum strings don't have a null terminator. The C code will read past the end of the string until it finds a `\0` byte somewhere in memory.

The fix:

```ferrum
fn good() {
    unsafe {
        puts(c"hello".as_ptr())  // literal with null terminator
    }
}

// Or at runtime:
fn also_good(s: &str) {
    let c_str = CString.new(s).unwrap()
    unsafe {
        puts(c_str.as_ptr())
    }
}
```

**Mistake 3: C frees memory you're still using**

```ferrum
extern "C" fn get_message(): *const c_char  ! Unsafe
extern "C" fn free_message(ptr: *const c_char)  ! Unsafe

fn use_message() {
    let ptr = unsafe { get_message() }
    let msg: &CStr = unsafe { CStr.from_ptr(ptr) }

    unsafe { free_message(ptr) }  // C frees the memory

    println("{}", msg.to_str().unwrap())  // DANGLING POINTER
}
```

The `CStr` doesn't own the memory; it just points to it. When C frees the underlying buffer, the `CStr` becomes invalid.

---

## Matching C Struct Layouts: @repr(C)

Ferrum structs don't have the same memory layout as C structs. Ferrum can reorder fields, add padding differently, or use different alignment. If you need to pass a struct to C, you must tell Ferrum to use C's layout.

### Basic Usage

```ferrum
@repr(C)
type Point {
    x: f64,
    y: f64,
}
```

This guarantees `Point` has the same layout as:

```c
struct Point {
    double x;
    double y;
};
```

### What @repr(C) Guarantees

1. **Fields are in declaration order.** Ferrum normally may reorder fields for efficiency.
2. **Padding matches C.** Fields are aligned as C would align them.
3. **Alignment matches C.** The struct's overall alignment matches C's rules.
4. **Total size matches.** `size_of::<Point>()` equals C's `sizeof(struct Point)`.

### A Detailed Example: The stat Structure

Here's a real-world example wrapping part of POSIX `stat`:

```c
// C definition (simplified, Linux x86_64)
struct stat {
    dev_t     st_dev;      // 8 bytes
    ino_t     st_ino;      // 8 bytes
    mode_t    st_mode;     // 4 bytes
    nlink_t   st_nlink;    // 8 bytes
    uid_t     st_uid;      // 4 bytes
    gid_t     st_gid;      // 4 bytes
    // ... more fields ...
    off_t     st_size;     // 8 bytes
    time_t    st_atime;    // 8 bytes
    time_t    st_mtime;    // 8 bytes
    time_t    st_ctime;    // 8 bytes
};
```

```ferrum
@repr(C)
type Stat {
    st_dev:   c_ulong,
    st_ino:   c_ulong,
    st_mode:  c_uint,
    st_nlink: c_ulong,
    st_uid:   c_uint,
    st_gid:   c_uint,
    // padding would go here in the full struct
    st_size:  c_long,
    st_atime: c_long,
    st_mtime: c_long,
    st_ctime: c_long,
}

extern "C" fn stat(path: *const c_char, buf: *mut Stat): c_int  ! Unsafe

fn get_file_size(path: &CStr): Result[i64, IoError] {
    let mut buf: Stat = zeroed()
    let result = unsafe { stat(path.as_ptr(), &mut buf) }

    if result == -1 {
        return Err(IoError.from_errno())
    }

    Ok(buf.st_size as i64)
}
```

### When Layout Matters: Getting It Wrong

If you forget `@repr(C)`, you get undefined behavior:

```ferrum
// BUG: Missing @repr(C)
type BadPoint {
    x: f64,
    y: f64,
}

extern "C" fn use_point(p: *const BadPoint)  ! Unsafe

fn main() {
    let p = BadPoint { x: 1.0, y: 2.0 }
    unsafe { use_point(&p) }  // C sees garbage!
}
```

The Ferrum compiler might lay out `BadPoint` as:
- `y` first, then `x` (for cache efficiency)
- Different padding
- Different alignment

The C code expects x at offset 0, y at offset 8. Instead it gets... whatever Ferrum decided.

**The compiler doesn't warn you.** This compiles fine. It just doesn't work.

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

The `@layout` attribute:
- Packs fields at bit granularity
- No implicit padding
- Fields laid out in declaration order
- Useful for network protocols, file formats, hardware registers

### Controlling Alignment with @align

Sometimes C code requires specific alignment:

```ferrum
@repr(C)
@align(16)
type SIMDVector {
    data: [f32; 4],
}
```

This ensures the struct is 16-byte aligned, which is required for SSE/AVX instructions.

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
    // error: cannot call function with `Unsafe` effect outside `unsafe` block

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

## Complete Example: Wrapping xxHash

Let's wrap a real C library: xxHash, a fast non-cryptographic hash function. This example shows the complete pattern you'll use for most C libraries.

### The C API We're Wrapping

```c
// xxhash.h (simplified)

#include <stdint.h>
#include <stddef.h>

// One-shot hashing
uint64_t XXH64(const void* input, size_t length, uint64_t seed);

// Streaming API
typedef struct XXH64_state_s XXH64_state_t;

XXH64_state_t* XXH64_createState(void);
int            XXH64_freeState(XXH64_state_t* state);
int            XXH64_reset(XXH64_state_t* state, uint64_t seed);
int            XXH64_update(XXH64_state_t* state, const void* input, size_t length);
uint64_t       XXH64_digest(const XXH64_state_t* state);

// Error codes
#define XXH_OK    0
#define XXH_ERROR 1
```

### Step 1: Raw FFI Bindings

First, create exact bindings to the C API. This layer is all unsafe and mirrors C exactly.

```ferrum
// xxhash_ffi.fe - raw bindings

import core.ffi.{c_int, c_void, c_size_t}

/// Opaque type representing xxHash state.
/// We don't know (or care) what's inside; the C library manages it.
@repr(C)
pub type XXH64State {
    _opaque: [u8; 0],  // zero-sized, prevents direct instantiation
}

// Error codes from the C header
pub const XXH_OK: c_int = 0
pub const XXH_ERROR: c_int = 1

@link(name = "xxhash")
extern "C" {
    /// Compute XXH64 hash of a buffer in one call.
    pub fn XXH64(input: *const c_void, length: c_size_t, seed: u64): u64  ! Unsafe

    /// Create a new streaming hash state.
    pub fn XXH64_createState(): *mut XXH64State  ! Unsafe

    /// Free a streaming hash state. Returns XXH_OK or XXH_ERROR.
    pub fn XXH64_freeState(state: *mut XXH64State): c_int  ! Unsafe

    /// Reset a state to start a new hash with the given seed.
    pub fn XXH64_reset(state: *mut XXH64State, seed: u64): c_int  ! Unsafe

    /// Add more data to the hash.
    pub fn XXH64_update(
        state: *mut XXH64State,
        input: *const c_void,
        length: c_size_t,
    ): c_int  ! Unsafe

    /// Compute the final hash value. State can continue to be used.
    pub fn XXH64_digest(state: *const XXH64State): u64  ! Unsafe
}
```

### Step 2: Safe Wrapper API

Now wrap the unsafe FFI in a safe, idiomatic Ferrum API.

```ferrum
// xxhash.fe - safe wrapper

import xxhash_ffi.{self, XXH64State, XXH_OK}

/// Error from xxHash operations.
pub enum XxHashError {
    /// Failed to allocate state (out of memory).
    AllocationFailed,
    /// The C library returned an error code.
    OperationFailed,
}

/// Compute the XXH64 hash of a byte slice.
///
/// This is the simplest way to hash data. For large data or streaming,
/// use `XxHash64` instead.
///
/// # Example
/// ```
/// let hash = xxhash.hash64(b"hello world", seed: 0)
/// assert_eq(hash, 0x74e3b98a0d3c5877)
/// ```
pub fn hash64(data: &[u8], seed: u64 = 0): u64 {
    unsafe {
        xxhash_ffi.XXH64(
            data.as_ptr() as *const c_void,
            data.len() as c_size_t,
            seed,
        )
    }
}

/// Streaming XXH64 hasher.
///
/// Use this when you can't load all data into memory at once,
/// or when data arrives in chunks.
///
/// # Example
/// ```
/// let mut hasher = XxHash64.new()?
/// hasher.update(b"hello ")?
/// hasher.update(b"world")?
/// let hash = hasher.finish()
/// ```
pub type XxHash64 {
    state: *mut XXH64State,
}

impl XxHash64 {
    /// Create a new hasher with the given seed.
    pub fn new(seed: u64 = 0): Result[Self, XxHashError] {
        let state = unsafe { xxhash_ffi.XXH64_createState() }

        if state.is_null() {
            return Err(XxHashError.AllocationFailed)
        }

        let result = unsafe { xxhash_ffi.XXH64_reset(state, seed) }
        if result != XXH_OK {
            // Clean up the state we just allocated
            unsafe { xxhash_ffi.XXH64_freeState(state) }
            return Err(XxHashError.OperationFailed)
        }

        Ok(XxHash64 { state })
    }

    /// Reset the hasher to compute a new hash with the given seed.
    ///
    /// This is more efficient than creating a new hasher.
    pub fn reset(&mut self, seed: u64 = 0): Result[(), XxHashError] {
        let result = unsafe { xxhash_ffi.XXH64_reset(self.state, seed) }
        if result != XXH_OK {
            Err(XxHashError.OperationFailed)
        } else {
            Ok(())
        }
    }

    /// Add data to the hash.
    pub fn update(&mut self, data: &[u8]): Result[(), XxHashError] {
        let result = unsafe {
            xxhash_ffi.XXH64_update(
                self.state,
                data.as_ptr() as *const c_void,
                data.len() as c_size_t,
            )
        };

        if result != XXH_OK {
            Err(XxHashError.OperationFailed)
        } else {
            Ok(())
        }
    }

    /// Compute the hash of all data added so far.
    ///
    /// The hasher can continue to be used after this call.
    pub fn finish(&self): u64 {
        unsafe { xxhash_ffi.XXH64_digest(self.state) }
    }

    /// Compute the hash and reset the hasher in one operation.
    pub fn finish_reset(&mut self, seed: u64 = 0): Result[u64, XxHashError] {
        let hash = self.finish()
        self.reset(seed)?
        Ok(hash)
    }
}

impl Drop for XxHash64 {
    fn drop(&mut self) {
        // Always clean up the C state when the Ferrum wrapper is dropped.
        // We ignore the return value because we can't report errors from drop.
        unsafe { xxhash_ffi.XXH64_freeState(self.state) }
    }
}

// Prevent sending XxHash64 to other threads.
// The xxHash state might not be thread-safe.
impl !Send for XxHash64 {}
impl !Sync for XxHash64 {}
```

### Step 3: Usage

Now users get a completely safe, idiomatic API:

```ferrum
import xxhash.{hash64, XxHash64}

fn main(): Result[(), XxHashError] {
    // One-shot hashing
    let hash = hash64(b"hello world")
    println("Hash: {hash:016x}")

    // Streaming hashing
    let mut hasher = XxHash64.new()?

    // Hash a file in chunks
    let file = File.open("large_file.bin")?
    let mut buffer = [0u8; 8192]

    loop {
        let n = file.read(&mut buffer)?
        if n == 0 { break }
        hasher.update(&buffer[..n])?
    }

    let file_hash = hasher.finish()
    println("File hash: {file_hash:016x}")

    Ok(())
}   // hasher is dropped here, XXH64_freeState called automatically
```

No `unsafe` anywhere in the user's code. Memory management is automatic. Errors are handled with `Result`.

### What We Achieved

1. **Type safety**: Can't pass wrong types to the hasher
2. **Memory safety**: State is freed automatically via Drop
3. **Error handling**: C error codes become Ferrum Results
4. **Idiomatic API**: Looks like native Ferrum code
5. **Zero overhead**: The safe wrapper compiles to the same code as calling C directly

---

## Callbacks: C Calling Back Into Ferrum

C libraries often take callback functions. This is one of the most dangerous FFI patterns because lifetime issues are invisible to the compiler.

### The Pattern

C callbacks typically look like this:

```c
// C header
typedef void (*callback_fn)(void* user_data, int event);

void register_callback(callback_fn cb, void* user_data);
void trigger_events(void);
void unregister_callback(callback_fn cb);
```

The `void* user_data` is a "context pointer" - opaque data that gets passed to the callback. This is how C passes state to callbacks.

### Basic Callback Example

```ferrum
// FFI bindings
type CallbackFn = extern "C" fn(*mut c_void, c_int)

extern "C" {
    fn register_callback(cb: CallbackFn, user_data: *mut c_void)  ! Unsafe
    fn trigger_events()  ! Unsafe
    fn unregister_callback(cb: CallbackFn)  ! Unsafe
}

// Our callback function - must be extern "C" and @no_mangle
@no_mangle
extern "C" fn my_callback(user_data: *mut c_void, event: c_int) {
    // Recover the Ferrum data from the void pointer
    // SAFETY: We registered this pointer, and it must still be valid
    let handler: &mut EventHandler = unsafe {
        &mut *(user_data as *mut EventHandler)
    }

    handler.on_event(event as i32)
}

type EventHandler {
    name: String,
    count: i32,
}

impl EventHandler {
    fn on_event(&mut self, event: i32) {
        self.count += 1
        println("[{self.name}] Event {event}, total: {self.count}")
    }
}
```

### The Lifetime Problem

Here's what goes wrong:

```ferrum
fn broken() {
    let handler = EventHandler { name: "test".into(), count: 0 }

    unsafe {
        register_callback(
            my_callback,
            &mut handler as *mut EventHandler as *mut c_void,
        )
    }
}   // handler is dropped here!

fn main() {
    broken()
    unsafe { trigger_events() }  // callback receives dangling pointer!
}
```

The C library holds a pointer to `handler`. When `handler` goes out of scope, the pointer dangles. When the callback is invoked, it accesses freed memory.

**The compiler cannot catch this.** Raw pointers aren't tracked by the borrow checker.

### Safe Callback Pattern 1: Box and Pin

Keep the data alive by boxing it and ensuring it won't move:

```ferrum
import core.pin.Pin
import alloc.boxed.Box

type CallbackState {
    handler: Pin[Box[EventHandler]],
}

impl CallbackState {
    fn new(name: String): Self {
        let handler = Box.pin(EventHandler { name, count: 0 })
        CallbackState { handler }
    }

    fn register(&mut self) {
        let ptr = &mut *self.handler as *mut EventHandler as *mut c_void
        unsafe {
            register_callback(my_callback, ptr)
        }
    }

    fn unregister(&self) {
        unsafe {
            unregister_callback(my_callback)
        }
    }
}

impl Drop for CallbackState {
    fn drop(&mut self) {
        // CRITICAL: Unregister before the handler is dropped!
        self.unregister()
    }
}

fn main() {
    let mut state = CallbackState.new("test".into())
    state.register()

    unsafe { trigger_events() }  // safe: state is still alive

    // state dropped here, unregister called, then handler freed
}
```

### Safe Callback Pattern 2: Static or Leaked Data

For callbacks that live for the entire program:

```ferrum
fn register_permanent_callback() {
    // Box::leak intentionally leaks the memory - it lives forever
    let handler: &'static mut EventHandler = Box.leak(Box.new(
        EventHandler { name: "permanent".into(), count: 0 }
    ))

    unsafe {
        register_callback(
            my_callback,
            handler as *mut EventHandler as *mut c_void,
        )
    }

    // No cleanup needed - the handler lives until the process exits
}
```

### Callback Error Handling

What if the callback needs to report an error? C callbacks often return error codes:

```c
typedef int (*validator_fn)(void* user_data, const char* input);
// Returns 0 for valid, non-zero for invalid
```

```ferrum
type ValidatorFn = extern "C" fn(*mut c_void, *const c_char): c_int

@no_mangle
extern "C" fn validate_callback(user_data: *mut c_void, input: *const c_char): c_int {
    let validator: &Validator = unsafe { &*(user_data as *const Validator) }

    // Handle null input
    if input.is_null() {
        return -1  // error code
    }

    // Convert C string to Ferrum, handling invalid UTF-8
    let input_str = unsafe { CStr.from_ptr(input) }
    let Ok(s) = input_str.to_str() else {
        return -2  // invalid UTF-8 error
    }

    // Run validation
    if validator.is_valid(s) {
        0
    } else {
        1
    }
}
```

### Panics in Callbacks

**Never let a panic escape a callback.** Unwinding across FFI boundaries is undefined behavior.

```ferrum
@no_mangle
extern "C" fn safe_callback(user_data: *mut c_void, value: c_int): c_int {
    // Catch any panic and convert to error code
    let result = catch_unwind(|| {
        let handler: &mut Handler = unsafe { &mut *(user_data as *mut Handler) }
        handler.process(value)
    })

    match result {
        Ok(()) => 0,
        Err(_) => {
            // Log the panic somewhere if possible
            eprintln("Panic in callback, returning error")
            -1
        }
    }
}
```

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

### Generating C Headers

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

### Creating a Ferrum Library for C Consumers

Here's a complete example of a Ferrum library that C code can use:

```ferrum
// mylib.fe - A key-value store library

import alloc.ffi.CString
import core.ffi.{CStr, c_int, c_char}
import collections.HashMap

/// Opaque handle to a key-value store.
/// C code only sees a pointer; they can't inspect the contents.
pub type KvStore {
    data: HashMap[String, String],
}

/// Create a new key-value store. Returns null on allocation failure.
@no_mangle
@export_c_header
pub extern "C" fn kvstore_new(): *mut KvStore {
    let store = Box.new(KvStore { data: HashMap.new() })
    Box.into_raw(store)
}

/// Free a key-value store. Safe to call with null.
@no_mangle
@export_c_header
pub extern "C" fn kvstore_free(store: *mut KvStore) {
    if !store.is_null() {
        // Take ownership back and drop it
        unsafe { drop(Box.from_raw(store)) }
    }
}

/// Set a key-value pair. Returns 0 on success, -1 on error.
/// Both key and value must be valid, non-null, UTF-8 C strings.
@no_mangle
@export_c_header
pub extern "C" fn kvstore_set(
    store: *mut KvStore,
    key: *const c_char,
    value: *const c_char,
): c_int {
    if store.is_null() || key.is_null() || value.is_null() {
        return -1
    }

    let store = unsafe { &mut *store }
    let key = unsafe { CStr.from_ptr(key) }
    let value = unsafe { CStr.from_ptr(value) }

    let Ok(key) = key.to_str() else { return -1 }
    let Ok(value) = value.to_str() else { return -1 }

    store.data.insert(key.to_string(), value.to_string())
    0
}

/// Get a value by key. Returns null if not found or on error.
/// The returned string is owned by the store; do not free it.
/// The pointer is valid until the next kvstore_set or kvstore_free.
@no_mangle
@export_c_header
pub extern "C" fn kvstore_get(
    store: *const KvStore,
    key: *const c_char,
): *const c_char {
    if store.is_null() || key.is_null() {
        return core.ptr.null()
    }

    let store = unsafe { &*store }
    let key = unsafe { CStr.from_ptr(key) }

    let Ok(key) = key.to_str() else { return core.ptr.null() }

    match store.data.get(key) {
        Some(value) => value.as_ptr() as *const c_char,
        None => core.ptr.null(),
    }
}
```

Generated header:

```c
/* mylib.h - Generated by Ferrum */
#ifndef MYLIB_H
#define MYLIB_H

#ifdef __cplusplus
extern "C" {
#endif

typedef struct KvStore KvStore;

KvStore* kvstore_new(void);
void kvstore_free(KvStore* store);
int kvstore_set(KvStore* store, const char* key, const char* value);
const char* kvstore_get(const KvStore* store, const char* key);

#ifdef __cplusplus
}
#endif

#endif /* MYLIB_H */
```

---

## Troubleshooting Common FFI Bugs

### Problem: Segfault on Startup

**Symptom:** Program crashes immediately with SIGSEGV before main() runs.

**Likely causes:**
1. **Library not found.** Check `LD_LIBRARY_PATH` (Linux), `DYLD_LIBRARY_PATH` (macOS), or `PATH` (Windows).
2. **ABI mismatch.** Library compiled with different compiler or flags.
3. **Static constructor crashed.** Some C libraries run code at load time.

**Debug steps:**
```bash
# Check if library is found
ldd ./myprogram           # Linux
otool -L ./myprogram      # macOS

# Run with library debugging
LD_DEBUG=libs ./myprogram  # Linux - shows library loading
```

### Problem: Segfault When Calling C Function

**Symptom:** Crash inside C code, often with meaningless backtrace.

**Likely causes:**
1. **Null pointer passed.** Did you check all pointers for null?
2. **Dangling pointer.** Was the data freed before the call?
3. **Wrong struct layout.** Missing `@repr(C)`?
4. **Wrong calling convention.** Missing `extern "C"`?
5. **Type size mismatch.** Using `i32` instead of `c_long` on Linux?

**Debug steps:**
```ferrum
// Add null checks before every FFI call
assert(!ptr.is_null(), "ptr is null at line X")

// Print pointer values to verify they're reasonable
eprintln("ptr = {ptr:?}")
```

### Problem: Garbage Data Returned from C

**Symptom:** C function returns, but the data makes no sense.

**Likely causes:**
1. **Endianness mismatch.** Rare on modern systems, but check.
2. **Padding/alignment.** Missing `@repr(C)` on structs.
3. **Type size mismatch.** Especially `long` (different on Linux vs Windows).
4. **String encoding.** Expected UTF-8 but got Latin-1 or garbage.

**Debug steps:**
```ferrum
// Print struct size to verify it matches C's sizeof
println("Ferrum size: {}", size_of::<MyStruct>())
```

```c
// In C:
printf("C size: %zu\n", sizeof(struct MyStruct));
```

### Problem: Memory Leak

**Symptom:** Memory usage grows over time, program eventually crashes with OOM.

**Likely causes:**
1. **Forgot to free C-allocated memory.** Every `create/alloc/new` needs a matching `free/destroy/delete`.
2. **CString dropped too early.** C code stored the pointer, but Ferrum freed the string.
3. **Circular references through C.** C holds Ferrum pointer, Ferrum holds C pointer.

**Debug steps:**
```bash
# Run with memory sanitizer or valgrind
valgrind --leak-check=full ./myprogram

# Or use AddressSanitizer
ASAN_OPTIONS=detect_leaks=1 ./myprogram
```

### Problem: Use-After-Free

**Symptom:** Sometimes works, sometimes crashes with garbage, sometimes corrupts unrelated data.

**Likely cause:** C code or Ferrum code using a pointer after the memory was freed.

**Pattern to avoid:**
```ferrum
fn bad() -> *const c_char {
    let s = CString.new("hello").unwrap()
    s.as_ptr()  // Returns pointer to s's buffer
}   // s is dropped, buffer freed
    // Returned pointer is now dangling!
```

**Fix:**
```ferrum
// Option 1: Keep the CString alive
fn good(f: impl Fn(*const c_char)) {
    let s = CString.new("hello").unwrap()
    f(s.as_ptr())  // Use pointer while s is alive
}   // s dropped after f returns

// Option 2: Transfer ownership
fn transfer_ownership() -> *const c_char {
    let s = CString.new("hello").unwrap()
    s.into_raw()  // Ferrum gives up ownership, caller must free
}
```

### Problem: Data Corruption in Multithreaded Code

**Symptom:** Occasional wrong results, crashes, or hangs under load.

**Likely cause:** C library isn't thread-safe, called from multiple threads.

**Fix:**
```ferrum
// Wrap in Mutex
type ThreadSafeWrapper {
    inner: Mutex[UnsafeHandle],
}

impl ThreadSafeWrapper {
    fn do_thing(&self) {
        let guard = self.inner.lock().unwrap()
        unsafe { c_do_thing(guard.ptr) }
    }
}

// Or use thread-local storage
thread_local {
    static HANDLE: RefCell[Option[UnsafeHandle]] = RefCell.new(None)
}
```

### Problem: Different Behavior on Different Platforms

**Symptom:** Works on Linux, crashes on Windows (or vice versa).

**Likely causes:**
1. **`long` size.** 64 bits on Linux LP64, 32 bits on Windows LLP64.
2. **Path separators.** `/` vs `\`.
3. **Calling conventions.** `cdecl` vs `stdcall` on Windows.
4. **Library names.** `libfoo.so` vs `foo.dll`.

**Fix:**
```ferrum
// Use c_long, not i64 or i32
extern "C" fn takes_long(x: c_long)  ! Unsafe

// Platform-specific linking
#[cfg(target_os = "windows")]
@link(name = "foo", kind = "dylib")
extern "C" { ... }

#[cfg(not(target_os = "windows"))]
@link(name = "foo")
extern "C" { ... }
```

### Problem: Callback Causes Crash or Deadlock

**Symptom:** Crash or hang when C code calls back into Ferrum.

**Likely causes:**
1. **Callback data was dropped.** The `void* user_data` points to freed memory.
2. **Reentrancy.** Callback takes a lock that the calling code already holds.
3. **Panic in callback.** Unwinding across FFI is undefined behavior.
4. **Thread mismatch.** Callback called from different thread than expected.

**Fix checklist:**
- [ ] Is the callback data pinned or leaked? (Won't be moved or freed)
- [ ] Does the callback avoid taking locks the caller might hold?
- [ ] Does the callback catch panics?
- [ ] Is it safe to call from any thread? (If not, document it)

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

### Callback Function Types

```ferrum
// C: typedef void (*callback)(void* data, int value);
type Callback = extern "C" fn(*mut c_void, c_int)

// C: typedef int (*comparator)(const void*, const void*);
type Comparator = extern "C" fn(*const c_void, *const c_void): c_int
```

### Common Error Handling Pattern

```ferrum
fn call_c_api(): Result[Value, MyError] {
    let ptr = unsafe { c_create_thing() }
    if ptr.is_null() {
        return Err(MyError.AllocationFailed)
    }

    let result = unsafe { c_do_operation(ptr) }
    if result < 0 {
        unsafe { c_free_thing(ptr) }
        return Err(MyError.OperationFailed(result))
    }

    // Wrap in RAII type that handles cleanup
    Ok(SafeWrapper { ptr })
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

**The pattern:**
1. Raw FFI bindings (unsafe, mirrors C exactly)
2. Safe wrapper API (idiomatic Ferrum, hides unsafe)
3. Users never see unsafe

---

*See also:*
- *[Ferrum Standard Library - core.ffi](ferrum-stdlib-core.md#310-coreffi--foreign-function-interface-types) for complete CStr/CString reference*
- *[Ferrum Standard Library - alloc.ffi](ferrum-stdlib-alloc.md#43-allocffi--owned-ffi-strings) for owned FFI string types*
- *[Ferrum Language Reference - Safety Levels](ferrum-lang-verification.md) for unsafe, trusted, and extern semantics*
