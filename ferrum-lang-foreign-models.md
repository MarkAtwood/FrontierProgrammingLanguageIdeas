# Ferrum Language Reference — Foreign Models

Foreign models describe the ecosystem at an FFI boundary. They are the principled replacement for ABI strings like `extern "C"`.

---

## The Problem with ABI Strings

The C++ / Rust convention uses a string to select calling conventions:

```
extern "C" fn foo()          // C calling convention
extern "stdcall" fn bar()    // Windows stdcall
extern "system" fn baz()     // platform-appropriate
```

This conflates two distinct concepts:
1. **The foreign ecosystem** — what kind of thing is on the other side (C library, JVM, CLR, Objective-C runtime)
2. **The calling convention** — how arguments are passed in registers and stack within that ecosystem

And it can't scale. `extern "jvm"` and `extern "clr"` would be magic strings with no structure.

---

## Foreign Model Syntax

```ferrum
extern(model) fn foo(...)
extern(model) { fn foo(...); fn bar(...) }
extern(model.variant) fn foo(...)   // calling convention variant
```

The model name is an identifier, not a string. It refers to a registered foreign model. The compiler knows the semantics of each model — object lifetime, dispatch mechanism, exception handling, type mapping.

---

## Built-in Models

### `c` — The C Foreign Model

The C ecosystem. Default calling convention is cdecl.

```ferrum
extern(c) fn malloc(size: usize): *mut c_void  ! Unsafe
extern(c) fn free(ptr: *mut c_void)  ! Unsafe

extern(c) {
    fn strlen(s: *const c_char): c_size_t  ! Unsafe
    fn strcmp(a: *const c_char, b: *const c_char): c_int  ! Unsafe
}

// Exporting a Ferrum function to be called by C
extern(c) fn ferrum_callback(data: *const u8, len: usize) {
    // called by C — signature as declared
}
```

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(c)` | cdecl — default for C on all platforms |
| `extern(c.system)` | Platform system convention: stdcall on Win32, cdecl everywhere else |
| `extern(c.stdcall)` | x86 stdcall — callee cleans stack. Win32 API (`WINAPI`) |
| `extern(c.fastcall)` | x86 fastcall — first two args in registers |
| `extern(c.win64)` | x64 Windows calling convention |
| `extern(c.aapcs)` | ARM Procedure Call Standard |
| `extern(c.aapcs_vfp)` | ARM hard-float ABI |
| `extern(c.sysv64)` | x86-64 System V ABI (Linux, macOS) |

```ferrum
// Windows API uses system convention (stdcall on 32-bit, cdecl on 64-bit)
extern(c.system) fn CreateFileW(
    path: *const u16,
    access: u32,
    share: u32,
    security: *mut c_void,
    disposition: u32,
    flags: u32,
    template: *mut c_void,
): *mut c_void  ! Unsafe

// ARM bare metal
extern(c.aapcs) fn hardware_interrupt_handler() ! Unsafe
```

**C model properties:**
- Object model: unmanaged (raw pointers, programmer manages lifetime)
- Dispatch: direct (call the address)
- Exceptions: none (C has no exceptions; errors are return values or errno)
- Generics: none (C has no generics)
- Name mangling: none (symbol name is exactly the function name)

---

### `jvm` — The JVM Foreign Model

The Java Virtual Machine ecosystem. Used when Ferrum targets the JVM or embeds the JVM.

```ferrum
extern(jvm) "java/util/ArrayList" {
    fn new(): Self
    fn add(self: &mut Self, obj: &dyn JvmObject): bool
    fn size(self: &Self): i32
    fn get(self: &Self, index: i32): &dyn JvmObject
    fn clear(self: &mut Self)
}

// Implementing a Java interface in Ferrum
@jvm_implements("java/lang/Runnable")
impl Runnable for MyTask {
    fn run(self: &mut Self) ! IO { ... }
}
```

**JVM model properties:**
- Object model: GC-managed (references are opaque; do not take raw pointers into JVM objects)
- Dispatch: virtual bytecode (invokeinterface / invokevirtual)
- Exceptions: Java exceptions translate to `Result[T, JvmException]` at the boundary
- Generics: erased (type parameters are `Object` at runtime)
- Name mangling: JVM method descriptors (`(Ljava/lang/String;I)V`)

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(jvm)` | Standard JVM virtual dispatch |
| `extern(jvm.static)` | Static method (invokestatic) |
| `extern(jvm.interface)` | Interface method (invokeinterface) |
| `extern(jvm.special)` | Constructor or super call (invokespecial) |

```ferrum
extern(jvm.static) "java/lang/System" {
    fn currentTimeMillis(): i64
    fn arraycopy(src: &dyn JvmObject, src_pos: i32,
                 dst: &mut dyn JvmObject, dst_pos: i32, len: i32)
}
```

---

### `clr` — The CLR Foreign Model

The .NET Common Language Runtime ecosystem.

```ferrum
extern(clr) "System.Collections.Generic.List`1" {
    fn new(): Self
    fn Add(self: &mut Self, item: &dyn ClrObject)
    fn get_Count(self: &Self): i32
    fn get_Item(self: &Self, index: i32): &dyn ClrObject
}

// Implementing a .NET interface
@clr_implements("System.IDisposable")
impl IDisposable for MyResource {
    fn Dispose(self: &mut Self) { ... }
}
```

**CLR model properties:**
- Object model: GC-managed
- Dispatch: virtual CLR (callvirt)
- Exceptions: .NET exceptions translate to `Result[T, ClrException]`
- Generics: reified (CLR specializes generics at runtime — closer to Ferrum's monomorphization)
- Name mangling: CLR metadata (assembly-qualified type names)

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(clr)` | Standard CLR virtual dispatch |
| `extern(clr.static)` | Static method |
| `extern(clr.pinvoke)` | P/Invoke into native code from CLR |
| `extern(clr.cominterop)` | COM object interop |

---

### `wasm` — The WebAssembly Foreign Model

The WebAssembly component model. Used for WASM-to-WASM interop across component boundaries.

```ferrum
extern(wasm) "wasi:filesystem/types" {
    fn open_at(fd: u32, path: &str, flags: u32): Result[u32, u32]
    fn read(fd: u32, buf: &mut [u8]): Result[usize, u32]
    fn close(fd: u32)
}
```

**WASM model properties:**
- Object model: linear memory with explicit lifetimes
- Dispatch: direct (WASM call instruction)
- Exceptions: none at the WASM level (errors are return values)
- Generics: none
- Name mangling: WIT (WebAssembly Interface Types) identifiers

---

### `objc` — The Objective-C Foreign Model

The Objective-C runtime. Used on macOS and iOS.

```ferrum
extern(objc) "NSString" {
    fn stringWithUTF8String(s: *const c_char): &Self  // class method
    fn UTF8String(self: &Self): *const c_char
    fn length(self: &Self): usize
}

extern(objc) "NSArray" {
    fn arrayWithObjects(first: &dyn ObjcObject, ...): &Self
    fn objectAtIndex(self: &Self, index: usize): &dyn ObjcObject
    fn count(self: &Self): usize
}
```

**Objective-C model properties:**
- Object model: reference-counted (ARC)
- Dispatch: message send (objc_msgSend)
- Exceptions: `@try`/`@catch` → `Result[T, ObjcException]`
- Generics: none (Obj-C uses `id` for generics)
- Name mangling: Objective-C selector syntax

---

## Registering a Custom Model

For ecosystems not covered by the built-in models, a custom model can be declared in a library:

```ferrum
foreign model python {
    object_model: ref_counted,           // Python uses reference counting + cycle collector
    dispatch: dynamic,                   // Python dispatch is fully dynamic
    exceptions: translate_to_result,     // PyErr → Result
    generics: none,
    abi: c,                              // CPython C API is the boundary
    null_convention: error,              // NULL return = exception was set
}
```

Custom model declarations live in the crate that owns the interop layer. Application code sees only `extern(python)`.

---

## Type Mapping Across Models

Each model specifies how Ferrum types map to foreign types at the boundary.

### C model type mapping

| Ferrum type | C type | Notes |
|-------------|--------|-------|
| `i8..i64` | `int8_t..int64_t` | Direct |
| `u8..u64` | `uint8_t..uint64_t` | Direct |
| `f32, f64` | `float, double` | Direct |
| `bool` | `_Bool` | |
| `*const T` | `const T*` | Raw pointer |
| `*mut T` | `T*` | Raw pointer |
| `&str` | — | Not C-compatible; use `*const c_char` + length |
| `c_char, c_int, c_long` | `char, int, long` | Platform-correct sizes |

### JVM model type mapping

| Ferrum type | JVM type | Notes |
|-------------|----------|-------|
| `i32` | `int` | |
| `i64` | `long` | |
| `f32` | `float` | |
| `f64` | `double` | |
| `bool` | `boolean` | |
| `u16` | `char` | JVM char is 16-bit unsigned |
| `&str` | `java/lang/String` | Automatic conversion |
| `&[u8]` | `byte[]` | Array reference |
| `Option[T]` | nullable reference | `None` → null |
| `Result[T, JvmException]` | try/catch boundary | |

---

## Why This Is Principled

`extern "C"` was always secretly a model: C calling convention, C name mangling, C type mapping, no exceptions. Making the model explicit means:

- The compiler knows what crosses each boundary and how
- New ecosystems (Swift, Python, Lua) register their models without compiler changes
- Calling convention is a variant of the model, not a parallel concept
- Type mapping is part of the model spec, not implicit tribal knowledge
- The dot notation (`c.stdcall`) makes hierarchy explicit: ecosystem first, variant second

The string `"C"` in `extern "C"` was load-bearing information encoded as an opaque token. `extern(c)` encodes the same information as a name in the type system.

---

*See also: `ferrum-introduction-to-ffi.md` for practical usage. `ferrum-stdlib-sys.md` for platform-specific model registrations.*
