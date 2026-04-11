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
| `extern(c.nvptx)` | NVIDIA PTX kernel calling convention (CUDA) |
| `extern(c.amdgpu)` | AMD GPU calling convention (ROCm/HIP) |
| `extern(c.spirv)` | SPIR-V kernel/shader calling convention (Vulkan, OpenCL) |

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

// GPU kernels — these are entry points dispatched by the GPU driver
extern(c.nvptx) fn vector_add(a: *const f32, b: *const f32, c: *mut f32, n: u32) ! Unsafe
extern(c.amdgpu) fn matrix_mul(a: *const f32, b: *const f32, c: *mut f32, n: u32) ! Unsafe
extern(c.spirv) fn blur_kernel(src: *const u32, dst: *mut u32, w: u32, h: u32) ! Unsafe
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
| `extern(jvm.android)` | Android ART — ART-managed heap, DEX bytecode dispatch |

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

### `swift` — The Swift Foreign Model

The Swift runtime. Used on Apple platforms (macOS, iOS, tvOS, watchOS) when calling Swift libraries or exposing Ferrum to Swift consumers.

```ferrum
extern(swift) "Foundation.Data" {
    fn init(bytes: *const u8, count: usize): Self
    fn withUnsafeBytes[R](self: &Self, body: fn(*const u8, usize): R): R
    fn count(self: &Self): usize
}

extern(swift) "MyApp.AudioEngine" {
    fn shared(): &Self   // Swift class property — synthesized getter
    fn start(self: &mut Self) ! IO
    fn stop(self: &mut Self) ! IO
}
```

**Swift model properties:**
- Object model: ARC (automatic reference counting, same as Objective-C ARC but distinct runtime)
- Dispatch: direct for `final` / protocol witness table dispatch for protocol methods
- Exceptions: Swift `throws` → `Result[T, SwiftError]` at the boundary
- Generics: reified (Swift specializes generics; Ferrum sees the concrete instantiation at the boundary)
- Name mangling: Swift ABI mangling (demangled via `swift-demangle`; Ferrum uses the stable `@_silgen_name` interface)

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(swift)` | Standard Swift calling convention |
| `extern(swift.async)` | Swift concurrency (async/await) — boundary marshals to/from Ferrum async |
| `extern(swift.objc)` | `@objc`-bridged Swift — dispatches through Obj-C runtime |

```ferrum
// Swift async function bridged to Ferrum
extern(swift.async) "NetworkClient" {
    fn fetchData(url: &SwiftString): Result[SwiftData, SwiftError]
}
```

---

### `lua` — The Lua Foreign Model

The Lua VM. Used when embedding Lua in a Ferrum application or writing Ferrum functions callable from Lua.

```ferrum
// Opening a Lua state and calling into it
extern(lua) {
    fn luaL_newstate(): *mut LuaState  ! Unsafe
    fn luaL_openlibs(L: *mut LuaState)  ! Unsafe
    fn luaL_dostring(L: *mut LuaState, code: *const c_char): lua_int  ! Unsafe
    fn lua_getglobal(L: *mut LuaState, name: *const c_char)  ! Unsafe
    fn lua_pcall(L: *mut LuaState, nargs: lua_int, nresults: lua_int, msgh: lua_int): lua_int  ! Unsafe
    fn lua_close(L: *mut LuaState)  ! Unsafe
}

// Exporting a Ferrum function as a Lua C function
extern(lua) fn lua_myfunction(L: *mut LuaState): lua_int ! Unsafe {
    // read args from Lua stack, push results
    1  // number of return values
}
```

**Lua model properties:**
- Object model: GC-managed (Lua GC; Ferrum values passed by copy or as lightuserdata)
- Dispatch: C function pointer via `lua_CFunction` typedef
- Exceptions: Lua `error()` / `pcall` → `Result[T, LuaError]`; `lua_pcall` is the boundary
- Generics: none (Lua is dynamically typed; values are `lua_Value` at the boundary)
- Name mangling: none (C ABI; Lua sees C function pointers)

**Notes:** The Lua model is thin over the C model. `extern(lua)` asserts that the C functions follow Lua's stack-based calling protocol, enabling higher-level tooling to generate Lua bindings automatically.

---

### `python` — The Python Foreign Model

The CPython C API. Used when embedding Python in a Ferrum application or writing Ferrum extension modules loadable by Python.

```ferrum
// Embedding Python
extern(python) {
    fn Py_Initialize()  ! Unsafe
    fn Py_Finalize()  ! Unsafe
    fn PyRun_SimpleString(code: *const c_char): c_int  ! Unsafe
    fn PyErr_Occurred(): *mut PyObject  ! Unsafe
    fn PyErr_Fetch(ptype: *mut *mut PyObject, pvalue: *mut *mut PyObject, ptraceback: *mut *mut PyObject)  ! Unsafe
}

// Building a Python extension module
extern(python) fn PyInit_mymodule(): *mut PyObject ! Unsafe {
    // return a PyModuleDef initialized module object
}

// Calling a Python function from Ferrum
extern(python) {
    fn PyObject_CallFunctionObjArgs(callable: *mut PyObject, ...): *mut PyObject  ! Unsafe
    fn PyLong_AsLongLong(obj: *mut PyObject): i64  ! Unsafe
    fn Py_DecRef(obj: *mut PyObject)  ! Unsafe
}
```

**Python model properties:**
- Object model: reference-counted + cycle collector (GC); all objects are `*mut PyObject`
- Dispatch: C function pointers in `PyMethodDef` tables; `PyObject_Call` for dynamic dispatch
- Exceptions: Python exception state (`PyErr_*` family) → `Result[T, PyException]`; NULL return signals exception set
- Generics: none (Python is dynamically typed at the boundary)
- Name mangling: C ABI; Python discovers functions via `PyMethodDef` tables

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(python)` | CPython C API — direct `PyObject*` protocol |
| `extern(python.limited)` | CPython stable ABI (`Py_LIMITED_API`) — compatible across minor versions |

---

### `napi` — The Node.js Native API Foreign Model

Node.js N-API (Node-API). Used when writing native Node.js addons in Ferrum that are ABI-stable across Node.js versions.

```ferrum
// Defining a native addon entry point
extern(napi) fn napi_register_module_v1(env: napi_env, exports: napi_value): napi_value ! Unsafe {
    // attach methods to exports object
    exports
}

// Wrapping a Ferrum function for export to JavaScript
extern(napi) {
    fn napi_create_function(
        env: napi_env,
        name: *const c_char,
        name_len: usize,
        cb: napi_callback,
        data: *mut c_void,
        result: *mut napi_value,
    ): napi_status  ! Unsafe

    fn napi_set_named_property(
        env: napi_env,
        object: napi_value,
        name: *const c_char,
        value: napi_value,
    ): napi_status  ! Unsafe
}
```

**N-API model properties:**
- Object model: V8 GC-managed (values are opaque `napi_value` handles; Ferrum holds handles, not raw pointers)
- Dispatch: C callback function pointers (`napi_callback` typedef)
- Exceptions: N-API status codes → `Result[T, NapiError]`; exceptions are set via `napi_throw_*` family
- Generics: none (JavaScript is dynamically typed)
- Name mangling: none (C ABI; Node discovers the module via a known symbol)

**Note:** N-API is deliberately ABI-stable: a Ferrum addon compiled against Node 18 runs on Node 22 without recompilation.

---

### `beam` — The BEAM Foreign Model

The Erlang VM (BEAM). Used when writing Ferrum code callable from Erlang or Elixir via NIFs (Native Implemented Functions). The model is inverted relative to most others: BEAM calls Ferrum, not the other way around.

```ferrum
// Declaring NIF implementations — BEAM calls these
extern(beam) fn add(env: *mut ErlNifEnv, argc: c_int, argv: *const ErlNifTerm): ErlNifTerm ! Unsafe {
    // unpack args, compute, return term
}

// Module init — called once at load
extern(beam) fn load(env: *mut ErlNifEnv, priv_data: *mut *mut c_void, load_info: ErlNifTerm): c_int ! Unsafe {
    0  // success
}

// NIF table — exported via ERL_NIF_INIT macro equivalent
extern(beam) {
    fn nif_add(env: *mut ErlNifEnv, argc: c_int, argv: *const ErlNifTerm): ErlNifTerm  ! Unsafe
    fn nif_stats(env: *mut ErlNifEnv, argc: c_int, argv: *const ErlNifTerm): ErlNifTerm  ! Unsafe
}
```

**BEAM model properties:**
- Object model: BEAM GC (terms are `ErlNifTerm`, an opaque integer; no raw pointer into BEAM heap)
- Dispatch: BEAM calls C function pointers from the NIF table
- Exceptions: NIF errors raised via `enif_make_badarg` / `enif_raise_exception` → Erlang exception
- Generics: none (BEAM types are dynamic; Ferrum sees opaque terms)
- Name mangling: C ABI; BEAM discovers NIFs via `ERL_NIF_INIT` entry point

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(beam)` | Standard synchronous NIF |
| `extern(beam.dirty_cpu)` | Dirty NIF — long CPU-bound work (runs on dirty CPU scheduler) |
| `extern(beam.dirty_io)` | Dirty NIF — long I/O-bound work (runs on dirty I/O scheduler) |

**Warning:** NIFs that block the BEAM scheduler stall the VM. Use `extern(beam.dirty_cpu)` or `extern(beam.dirty_io)` for any operation that may take more than ~1 ms.

---

### `dart` — The Dart Foreign Model

The Dart VM FFI. Used when writing native extensions for Flutter or standalone Dart applications.

```ferrum
// Calling a Ferrum function from Dart via dart:ffi
// On the Ferrum side, export with C ABI — Dart FFI crosses the C boundary
extern(dart) fn compute_fft(input: *const f64, output: *mut f64, n: usize) ! Unsafe {
    // Dart calls this as a NativeFunction<Void Function(Pointer<Double>, Pointer<Double>, Uint64)>
}

// Dart native extensions (older API): Ferrum registers a resolver
extern(dart) fn ResolveName(
    name: *const c_char,
    args: c_int,
    auto_setup_scope: *mut u8,
): *mut c_void  ! Unsafe

// Dart FFI Callbacks — Dart passing a function pointer back to Ferrum
extern(dart) fn dart_callback(result: f64) ! Unsafe {
    // called by Dart runtime via NativeCallable
}
```

**Dart model properties:**
- Object model: Dart VM GC (opaque `Dart_Handle`; do not retain beyond FFI boundary without `Dart_NewPersistentHandle`)
- Dispatch: C ABI function pointers (Dart FFI resolves by symbol name from the shared library)
- Exceptions: Dart exceptions do not propagate into native code; errors are return values or out-params
- Generics: none at the FFI boundary (Dart FFI uses concrete `NativeFunction` types)
- Name mangling: none (Dart FFI loads by C symbol name)

**Note:** `extern(dart)` is a thin assertion over `extern(c)`. Its value is tooling: the compiler can generate Dart FFI binding stubs (`.dart` files) automatically from `extern(dart)` declarations.

---

### `graalvm` — The GraalVM Polyglot Foreign Model

GraalVM's Truffle/Polyglot API. Used when Ferrum runs on GraalVM (via Espresso JVM or native-image) and interacts with other GraalVM languages (JavaScript, Python, Ruby, R) at runtime without crossing a C boundary.

```ferrum
// Evaluating another language inline
extern(graalvm) {
    fn polyglot_eval(language_id: *const c_char, source: *const c_char): PolyglotValue  ! IO
    fn polyglot_get_member(obj: PolyglotValue, key: *const c_char): PolyglotValue
    fn polyglot_execute(callable: PolyglotValue, args: &[PolyglotValue]): PolyglotValue  ! IO
    fn polyglot_as_i64(val: PolyglotValue): i64
    fn polyglot_as_string(val: PolyglotValue): GraalString
}

// Example: call a JavaScript function from Ferrum on GraalVM
fn call_js_sort(data: &[f64]): Vec[f64] ! IO {
    let sort_fn = polyglot_eval("js", "Array.prototype.sort")
    // ... marshal and call
}
```

**GraalVM model properties:**
- Object model: GraalVM host heap — `PolyglotValue` is a reference into the polyglot context; lifetime is context-scoped
- Dispatch: Truffle interop messages (READ, EXECUTE, INVOKE, etc.) — resolved at Truffle partial-evaluation time
- Exceptions: guest language exceptions → `Result[T, PolyglotException]`
- Generics: none (polyglot values are dynamically typed; Ferrum coerces via `polyglot_as_*`)
- Name mangling: none (interop by Truffle message protocol, not symbol names)

**Calling convention variants:**

| Variant | Meaning |
|---------|---------|
| `extern(graalvm)` | Standard Truffle polyglot interop |
| `extern(graalvm.native_image)` | GraalVM native-image — AOT context, no dynamic language eval |

**Note:** `extern(graalvm)` is only valid when Ferrum targets the GraalVM JVM backend. On LLVM or standalone JVM targets, the compiler rejects it.

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
- New ecosystems (Swift, Python, Lua, BEAM, Dart, GraalVM) register their models without compiler changes
- Calling convention is a variant of the model, not a parallel concept
- Type mapping is part of the model spec, not implicit tribal knowledge
- The dot notation (`c.stdcall`) makes hierarchy explicit: ecosystem first, variant second

The string `"C"` in `extern "C"` was load-bearing information encoded as an opaque token. `extern(c)` encodes the same information as a name in the type system.

---

*See also: `ferrum-introduction-to-ffi.md` for practical usage. `ferrum-stdlib-sys.md` for platform-specific model registrations.*
