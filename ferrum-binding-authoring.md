# Ferrum Binding Authoring Guide

**Audience:** Authors writing new `foreign model` declarations (`extern(model)`) or new OS platform crates (`sys.{platform}`)
**See also:** [ferrum-lang-foreign-models.md](ferrum-lang-foreign-models.md) · [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md) · [ferrum-stdlib-sys.md](ferrum-stdlib-sys.md)

This guide defines the process for mapping an external ecosystem into Ferrum. Part 1 covers foreign language model authoring. Part 2 covers OS platform crate authoring. Parts 3 and 4 apply the process to every existing model and OS, serving as worked examples and cross-references.

When a new language runtime or operating system appears, this document is the starting point.

---

## Part 1: Foreign Model Authoring

A foreign model describes what is on the other side of an FFI boundary: how objects are owned, how calls are dispatched, how errors are reported, and how symbols are named in the binary. With this information the compiler can enforce correct usage and tooling can generate binding stubs automatically.

The only model the compiler understands natively is `c`. Every other model is a `foreign model` declaration that lives in a crate. The stdlib ships the common ones; a third-party crate has identical standing.

### 1.1 Step 1 — Characterize the Ecosystem

Answer the following questions before writing any declarations. These answers drive every subsequent decision.

| Question | What it determines |
|---|---|
| **Object model** | Memory ownership and lifetime — who keeps objects alive? |
| **Dispatch** | How are function calls resolved at the binary level? |
| **Exceptions** | How do errors and exceptions cross the boundary? |
| **Generics** | How are type parameters handled at the boundary? |
| **Name mangling** | How are symbols named in the binary or VM? |
| **Threading** | What thread-safety guarantees does the runtime provide? |
| **Initialization** | Does the runtime need explicit setup and teardown? |
| **Null convention** | What does a null/nil/zero pointer mean at this boundary? |

**Object model values:**

| Value | Meaning | Examples |
|---|---|---|
| `unmanaged` | Caller manages allocation and lifetime | C, bare C++ |
| `ref_counted` | Runtime maintains a reference count | Objective-C ARC, Swift ARC, Python (CPython), COM/Windows |
| `gc_tracing` | Tracing GC owns all heap objects; do not take raw interior pointers | JVM, CLR, Lua, V8 (NAPI), Dart |
| `vm_opaque` | Objects are opaque tokens in a VM; Ferrum sees only handles | BEAM ErlNifTerm, GraalVM PolyglotValue |

**Dispatch values:**

| Value | Meaning | Examples |
|---|---|---|
| `direct` | Symbol address is called directly | C, WASM, non-virtual C++ |
| `vtable` | Function pointer table resolved at construction | C++ virtual, CLR callvirt, Swift protocol witness table |
| `message_send` | Selector-based runtime dispatch | Objective-C `objc_msgSend`, Smalltalk |
| `bytecode` | VM interprets or JIT-compiles; dispatch is through the VM | JVM `invokevirtual` / `invokeinterface` |
| `function_table` | Runtime provides a table of C function pointers to call Ferrum | CPython C API (`PyMethodDef`), Lua C API (`lua_CFunction`), BEAM NIF table |

**Exceptions values:**

| Value | Meaning |
|---|---|
| `none` | No exception mechanism; errors are return values or out-params (C, WASM) |
| `translate_to_result` | Foreign exceptions are caught at the boundary and returned as `Result[T, E]` |
| `propagated` | Foreign exceptions propagate without translation — UB if they escape into Ferrum |

**Null convention values:**

| Value | Meaning |
|---|---|
| `not_present` | The boundary type has no null concept; use `Option[T]` |
| `error` | A null return signals an error was set (CPython) |
| `valid` | Null is a legitimate value for reference types (JVM, CLR, ObjC nil) |
| `sentinel(N)` | A specific non-zero sentinel value signals error or end |

### 1.2 Step 2 — Write the `foreign model` Declaration

```ferrum
foreign model name {
    object_model: ...,          // required
    dispatch: ...,              // required
    exceptions: ...,            // required
    generics: ...,              // required
    abi: c,                     // required; almost always c (the underlying binary ABI)
    null_convention: ...,       // required
    // Documentation fields (do not affect codegen; inform tooling and readers):
    // name_mangling: ...,      // how symbols are named in the binary
    // threading: ...,          // thread-safety guarantee from the runtime
    // initialization: ...,     // explicit | implicit | none
}
```

The `abi` field is the underlying binary interface through which the boundary is crossed. Almost every foreign model uses `abi: c` — JNI is C function pointers, the CPython C API is C, BEAM NIFs are C. Pure Ferrum-to-Ferrum communication does not use a `foreign model`.

Place the declaration in the crate that owns the interop layer, not in application code. Application code sees only `extern(name)`.

### 1.3 Step 3 — Declare Boundary Types

Define the opaque handle types that represent foreign objects. Ferrum never constructs these — they are received from or passed to the foreign runtime.

```ferrum
// Opaque foreign object — never constructed by Ferrum
struct FooObject { _private: () }

// The pointer types Ferrum works with
type FooHandle = *mut FooObject       // owned or mutably borrowed handle
type FooRef    = *const FooObject     // immutable borrow

// Error type crossing the boundary
struct FooException {
    message: String,
    code:    i32,
}
impl Error for FooException { ... }
```

Rule of thumb: **one opaque type per GC/ownership domain.** If the runtime uses a single opaque value (Lua's `lua_State`, BEAM's `ErlNifTerm`), declare that. If the runtime has a class hierarchy, declare the root type and use `*mut Root` as the universal handle.

### 1.4 Step 4 — Write the Type Mapping Table

Document how Ferrum types map to foreign types. Include notes where the mapping is lossy, has preconditions, or requires explicit conversion.

| Ferrum type | Foreign type | Notes |
|---|---|---|
| `i8..i64` | native integer types | Direct where widths match |
| `u8..u64` | native unsigned types | Direct where widths match |
| `f32, f64` | native float types | Direct |
| `bool` | boolean type | Representation varies by platform |
| `*const T` / `*mut T` | pointer type | Raw pointer; no ownership signal |
| `&str` | string type | Requires conversion; document encoding |
| `Option[T]` | nullable type | `None` → null/nil, if applicable |
| `Result[T, E]` | exception mechanism | Boundary translation; document error type |
| `&[T]` / `&mut [T]` | array/slice | Pointer + length, separately or combined |

Mark types with no clean mapping as **not directly representable** and specify the workaround (opaque handles, explicit conversion functions, marshalling helpers).

### 1.5 Step 5 — Write the Raw `extern(model)` Layer

The raw layer is a direct transcription of the foreign API into Ferrum types. Every function is `! Unsafe`.

```ferrum
// Lifecycle
extern(name) {
    fn foo_runtime_init() ! Unsafe
    fn foo_runtime_fini() ! Unsafe
}

// Object operations
extern(name) {
    fn foo_create():                    *mut FooObject ! Unsafe
    fn foo_destroy(obj: *mut FooObject)                ! Unsafe
    fn foo_call(obj: *mut FooObject, arg: i32): i32    ! Unsafe
}
```

Rules:
- Every function is `! Unsafe`
- Types match the foreign ABI exactly — no wrapping yet
- No error translation — return codes and out-params as declared in the foreign API
- Symbol names match exactly; use `#[link_name = "original_name"]` if Ferrum naming conventions would change them
- The raw layer module is `pub(crate)` — do not expose it outside the binding crate

### 1.6 Step 6 — Write the Idiomatic Wrapper Layer

The wrapper layer is the public-facing API. It is safe Ferrum — no `! Unsafe` in public signatures.

```ferrum
struct FooRuntime { _handle: *mut FooState }

impl FooRuntime {
    fn new() -> Result[Self, FooError] ! IO {
        let state = unsafe { foo_runtime_init() };
        if state.is_null() { return Err(FooError::InitFailed) }
        Ok(FooRuntime { _handle: state })
    }
}

impl Drop for FooRuntime {
    fn drop(self: &mut Self) { unsafe { foo_runtime_fini(self._handle) } }
}

struct FooObj { _handle: *mut FooObject }

impl FooObj {
    fn call(self: &Self, arg: i32) -> Result[i32, FooException] {
        let result = unsafe { foo_call(self._handle, arg) };
        if result < 0 { Err(FooException { code: result, message: "...".into() }) }
        else { Ok(result) }
    }
}
```

Rules:
- No `! Unsafe` in public signatures
- RAII handles init/fini and object lifetimes via `Drop`
- All fallible operations return `Result[T, E]`
- Error codes are converted to typed `Result` immediately after each foreign call
- Lifetimes are explicit where foreign objects can outlive the runtime or context
- Document any preconditions that cannot be statically enforced

### 1.7 Step 7 — Define Calling Convention Variants

If the model has multiple calling conventions, define them as `name.variant` and document with a table:

| Variant | Meaning |
|---|---|
| `extern(name)` | Default calling convention |
| `extern(name.variant1)` | What this selects and when to use it |
| `extern(name.variant2)` | What this selects and when to use it |

Examples: `c.stdcall`, `c.system`, `jvm.static`, `jvm.android`, `beam.dirty_cpu`.

### 1.8 Foreign Model Authoring Checklist

- [ ] All eight characterization questions answered in a doc comment on the `foreign model` declaration
- [ ] Boundary types declared as opaque structs
- [ ] Type mapping table covers all types used in the public API
- [ ] Raw layer: every function `! Unsafe`
- [ ] Raw layer: symbol names match the foreign library exactly
- [ ] Raw layer: module is `pub(crate)` only
- [ ] Wrapper layer: no `! Unsafe` in public signatures
- [ ] Wrapper layer: all fallible operations return `Result`
- [ ] Wrapper layer: RAII for any runtime requiring init/fini
- [ ] Calling convention variants documented with a table
- [ ] At least one usage example in the module docs

---

## Part 2: OS Platform Crate Authoring

An OS platform crate provides Ferrum access to a specific operating system. It implements the platform traits (`FileSystem`, `Network`, `Process`, `Memory`, `Time`, `Entropy`, `Threading`) and exposes OS-specific features under `sys.{platform}`.

The compiler knows only `none` (bare metal, no OS). Every named OS is a platform crate. The standard library ships crates for common targets; a new OS crate has identical standing.

### 2.1 Step 1 — Characterize the OS

| Question | Purpose |
|---|---|
| **API style** | How does userspace talk to the kernel? (direct syscalls, C library, WASM interface, custom ABI) |
| **Filesystem model** | POSIX paths? Win32? Capability-based? Plan 9? None? |
| **Networking model** | BSD sockets? Winsock? WASI sockets? Platform SDK? None? |
| **Process model** | fork/exec? CreateProcess? tasks? capabilities? embedded (no processes)? |
| **Threading model** | pthreads? Win32 threads? Zircon threads? None (single-threaded embedded)? |
| **Capability/security model** | DAC/MAC? Capabilities? Pledge/unveil? Seccomp? WASI caps? None? |
| **Memory model** | Virtual memory? Flat? Segmented? No-MMU? VMO? |
| **Minimum version** | What is the oldest supported OS/kernel/SDK version? |

### 2.2 Step 2 — Define the Module Structure

Every platform crate follows this layout:

```
sys.{platform}
├── ffi/          raw unsafe bindings — syscall numbers, C types, struct layouts
├── types/        Ferrum-safe type wrappers over ffi types
├── fs/           FileSystem trait implementation
├── net/          Network trait implementation (if applicable)
├── process/      Process trait implementation (if applicable)
├── thread/       Threading primitives
├── time/         Time trait implementation
├── mem/          Memory trait implementation
└── platform/     Platform-specific extras not covered by any trait
```

The `ffi` sub-module is always `! Unsafe` and `pub(crate)`. Everything in `types` and above should be safe by default.

If the OS does not support a trait (e.g., an embedded OS with no networking), document that in the crate root and provide a compile-time error that names the missing capability.

### 2.3 Step 3 — Write the Raw FFI Layer (`sys.{platform}.ffi`)

```ferrum
// For syscall-based OSes:
const SYS_READ:  usize = 0
const SYS_WRITE: usize = 1
const SYS_OPEN:  usize = 2
// ... (cite the kernel header file and version for each constant)

@repr(C)
struct stat {          // layout must match the ABI spec exactly
    st_dev:   u64,
    st_ino:   u64,
    st_mode:  u32,
    // ...
}

fn syscall(nr: usize, a1: usize, a2: usize, a3: usize): isize ! Unsafe

// For C-library-based OSes:
extern(c) {
    fn open(path: *const c_char, flags: c_int, mode: c_uint): c_int   ! Unsafe
    fn read(fd: c_int, buf: *mut c_void, count: usize): isize          ! Unsafe
    fn write(fd: c_int, buf: *const c_void, count: usize): isize       ! Unsafe
    fn close(fd: c_int): c_int                                         ! Unsafe
}
```

Rules for the raw FFI layer:
- Struct layouts must exactly match the OS ABI — cite the header file and version in a comment
- Constant values verified against OS headers, not inferred from other platforms
- Function signatures match the OS API exactly — types are C types, not Ferrum types
- Everything `! Unsafe`; module is `pub(crate)` only

### 2.4 Step 4 — Write the Safe Wrapper Layer (`sys.{platform}`)

```ferrum
struct FileDescriptor { fd: c_int }

impl FileDescriptor {
    fn open(path: &Path, flags: OpenFlags) -> Result[Self, IoError] ! IO {
        let fd = unsafe { ffi::open(path.as_c_str().as_ptr(), flags.bits(), 0o666) };
        if fd < 0 { Err(IoError::from_errno()) }
        else { Ok(FileDescriptor { fd }) }
    }

    fn read(self: &Self, buf: &mut [u8]) -> Result[usize, IoError] ! IO { ... }
    fn write(self: &Self, buf: &[u8])    -> Result[usize, IoError] ! IO { ... }
}

impl Drop for FileDescriptor {
    fn drop(self: &mut Self) { unsafe { ffi::close(self.fd) }; }
}
```

Rules:
- No `! Unsafe` in public signatures
- RAII for all OS resources that need cleanup (file descriptors, handles, mappings)
- `errno`/error codes converted to `Result` immediately after each call
- Public API uses `Path`, `&str`, and Ferrum types — never expose `*const c_char`
- Platform-specific types are clearly namespaced (`linux::Epoll`, not just `Epoll`)

### 2.5 Step 5 — Implement the Platform Traits

Every platform crate implements the traits it supports. These traits are the abstraction layer that `std.*` builds on.

```ferrum
trait FileSystem {
    fn open(path: &Path, flags: OpenFlags) -> Result[FileHandle, IoError] ! IO
    fn stat(path: &Path)                   -> Result[FileMeta, IoError]   ! IO
    fn read_dir(path: &Path)               -> Result[impl Iterator[Item = DirEntry], IoError] ! IO
    fn rename(from: &Path, to: &Path)      -> Result[(), IoError]         ! IO
    fn remove_file(path: &Path)            -> Result[(), IoError]         ! IO
    fn create_dir(path: &Path)             -> Result[(), IoError]         ! IO
}

trait Network {
    fn connect(addr: SocketAddr) -> Result[impl Read + Write, IoError] ! IO
    fn listen(addr: SocketAddr)  -> Result[impl Listener, IoError]     ! IO
}

trait Process {
    fn spawn(cmd: &Command)    -> Result[Child, IoError] ! IO
    fn current_pid()           -> u32                   ! IO
    fn exit(code: i32)         -> never                 ! IO
}

trait Time {
    fn now()          -> Instant
    fn system_time()  -> Timestamp ! IO
    fn sleep(dur: Duration)    ! IO
}

trait Entropy {
    fn fill_random(buf: &mut [u8]) -> Result[(), IoError] ! IO
}

trait Threading {
    fn spawn(f: fn() ! IO) -> Result[JoinHandle, ThreadError] ! IO
    fn current_id()        -> ThreadId
}
```

Provide an `impl` block in the platform crate:

```ferrum
struct LinuxPlatform

impl FileSystem for LinuxPlatform { ... }
impl Network    for LinuxPlatform { ... }
// etc.
```

For unsupported traits, document why and provide a compile error:

```ferrum
// ZephyrPlatform does not implement Process:
// Zephyr has no fork/exec model. Use threads (Threading trait) instead.
```

### 2.6 Step 6 — Register `@cfg` Values

Add the OS to the known `target_os` values by declaring it in the platform crate manifest:

```ferrum
// Ferrum.toml
[platform]
target_os     = "newos"
target_family = "unix"       // or "windows", "embedded", "wasm"
target_env    = ""           // libc variant if applicable (gnu, musl, msvc)
min_version   = "newos:1.0"
```

This makes `@cfg(target_os = "newos")` available in consuming code:

```ferrum
@cfg(target_os = "newos")
fn platform_specific_init() ! IO { ... }
```

### 2.7 Step 7 — Map Capabilities to Effects

Document how the OS's permission model maps to Ferrum effect annotations. This table is also the basis for compile-time or runtime capability enforcement.

| Ferrum effect | OS mechanism | Enforcement point |
|---|---|---|
| `! IO` | all syscalls | — |
| `! FS` | open/stat/readdir | landlock (Linux), pledge fs (OpenBSD), WASI fs capability |
| `! Net` | connect/bind/accept | seccomp socket filter, pledge inet, WASI net capability |
| `! Spawn` | fork/clone/exec | pledge proc, Capsicum (limits exec), WASI (no spawn) |
| `! Time` | clock_gettime | generally unrestricted |
| `! Entropy` | getrandom/dev/random | generally unrestricted |

### 2.8 Step 8 — Set Tier and Minimum Version

| Tier | Meaning | CI commitment |
|---|---|---|
| 1 | Core target | Tested on every commit; regression is a release blocker |
| 2 | Supported | Tested regularly; regressions are bugs |
| 3 | Community | Builds when last tested; regressions tracked but not blocking |

Document the minimum version in the platform manifest and wherever `@cfg(target_os_version >= ...)` is used.

### 2.9 OS Platform Authoring Checklist

- [ ] All eight characterization questions answered in the crate README
- [ ] Module structure follows the template (ffi / types / fs / net / process / thread / time / mem / platform)
- [ ] Raw FFI layer: struct layouts verified against OS ABI spec — header file and version cited
- [ ] Raw FFI layer: all functions `! Unsafe`; module `pub(crate)` only
- [ ] Safe wrapper layer: no `! Unsafe` in public signatures
- [ ] Safe wrapper layer: RAII for all OS resources
- [ ] All applicable platform traits implemented; unsupported ones documented
- [ ] `@cfg(target_os = "newos")` registered in the platform manifest
- [ ] Capability → effect mapping table written
- [ ] Tier and minimum version documented
- [ ] At least one end-to-end integration test per implemented trait

---

## Part 3: Foreign Model Worked Examples

Each entry summarizes how the authoring process was applied. Full `extern(model)` API declarations are in [ferrum-lang-foreign-models.md](ferrum-lang-foreign-models.md).

### 3.1 `c` — Reference Model (Compiler Built-In)

| Question | Answer | Rationale |
|---|---|---|
| Object model | `unmanaged` | Caller owns every allocation; no runtime tracks it |
| Dispatch | `direct` | Call the symbol address |
| Exceptions | `none` | C has no exception mechanism; errors are return values |
| Generics | `none` | C has no generics |
| Name mangling | none | Symbol is exactly the declared name |
| Threading | none | No runtime guarantees |
| Initialization | none | No runtime to initialize |
| Null convention | `error` (function-dependent) | NULL usually signals error, but caller must consult documentation |

`c` is the only model baked into the compiler. Its `foreign model` declaration is implicit.

### 3.2 `jvm` — JVM Foreign Model

| Question | Answer | Rationale |
|---|---|---|
| Object model | `gc_tracing` | JVM GC owns all heap objects; do not take raw interior pointers |
| Dispatch | `bytecode` | invokevirtual / invokeinterface / invokespecial through the JVM |
| Exceptions | `translate_to_result` | Java exceptions caught at JNI boundary → `Result[T, JvmException]` |
| Generics | `erased` | Type parameters erased to `Object` at runtime |
| Name mangling | JVM descriptor | `(Ljava/lang/String;I)V` method descriptor syntax |
| Threading | thread-safe | JVM provides its own synchronization |
| Initialization | explicit | JVM must be started (CreateJavaVM) and stopped (DestroyJavaVM) |
| Null convention | `valid` | null is a legitimate value for any reference type |

```ferrum
foreign model jvm {
    object_model: gc_tracing,
    dispatch:     bytecode,
    exceptions:   translate_to_result,
    generics:     erased,
    abi:          c,          // JNI crosses a C boundary
    null_convention: valid,
}
```

### 3.3 `clr` — .NET CLR Foreign Model

| Question | Answer | Rationale |
|---|---|---|
| Object model | `gc_tracing` | CLR GC owns all managed objects |
| Dispatch | `vtable` | callvirt through CLR method table |
| Exceptions | `translate_to_result` | .NET exceptions → `Result[T, ClrException]` at boundary |
| Generics | `reified` | CLR specializes generic types at runtime |
| Name mangling | CLR metadata | Assembly-qualified names and CLR metadata tokens |
| Threading | thread-safe | CLR managed thread pool and synchronization |
| Initialization | explicit | CLR runtime must be hosted and initialized (ICLRRuntimeHost) |
| Null convention | `valid` | null is a valid reference type value |

```ferrum
foreign model clr {
    object_model: gc_tracing,
    dispatch:     vtable,
    exceptions:   translate_to_result,
    generics:     reified,
    abi:          c,
    null_convention: valid,
}
```

### 3.4 `wasm` — WebAssembly Component Model

| Question | Answer | Rationale |
|---|---|---|
| Object model | `unmanaged` | Linear memory; explicit lifetimes |
| Dispatch | `direct` | WASM call instruction to imported/exported function |
| Exceptions | `none` | No exception mechanism at the component boundary; errors are return values |
| Generics | `none` | WIT has no generics; concrete types only |
| Name mangling | WIT identifier | WIT interface names and function names |
| Threading | none | WASM components are single-threaded by default |
| Initialization | none | No separate init step |
| Null convention | `not_present` | WASM has no null; use Option[T] |

```ferrum
foreign model wasm {
    object_model: unmanaged,
    dispatch:     direct,
    exceptions:   none,
    generics:     none,
    abi:          c,
    null_convention: not_present,
}
```

### 3.5 `objc` — Objective-C Runtime

| Question | Answer | Rationale |
|---|---|---|
| Object model | `ref_counted` | ARC — objc_retain / objc_release |
| Dispatch | `message_send` | objc_msgSend — selector-based at runtime |
| Exceptions | `translate_to_result` | `@try` / `@catch` → `Result[T, ObjcException]` |
| Generics | `none` | Obj-C uses `id` (untyped pointer) for generics |
| Name mangling | ObjC selector | SEL syntax: `doThing:withArg:` |
| Threading | thread-safe | ObjC runtime is thread-safe for dispatch |
| Initialization | implicit | ObjC runtime auto-initialized on Apple platforms |
| Null convention | `valid` | nil is a valid object reference; messaging nil is a no-op |

```ferrum
foreign model objc {
    object_model: ref_counted,
    dispatch:     message_send,
    exceptions:   translate_to_result,
    generics:     none,
    abi:          c,
    null_convention: valid,
}
```

### 3.6 `swift` — Swift ABI

| Question | Answer | Rationale |
|---|---|---|
| Object model | `ref_counted` | ARC — distinct from Obj-C ARC at the ABI level |
| Dispatch | `vtable` | Direct for `final`; protocol witness table for protocol methods |
| Exceptions | `translate_to_result` | Swift `throws` → `Result[T, SwiftError]` |
| Generics | `reified` | Swift specializes generic code at compile time |
| Name mangling | Swift ABI | Swift name mangling; `@_silgen_name` for stable exported symbols |
| Threading | thread-safe | Swift actors provide structured concurrency |
| Initialization | implicit | Swift runtime auto-initialized |
| Null convention | `not_present` | Swift uses `Optional[T]` explicitly; non-optional values are never nil |

```ferrum
foreign model swift {
    object_model: ref_counted,
    dispatch:     vtable,
    exceptions:   translate_to_result,
    generics:     reified,
    abi:          c,
    null_convention: not_present,
}
```

### 3.7 `lua` — Lua C API

| Question | Answer | Rationale |
|---|---|---|
| Object model | `gc_tracing` | Lua GC; values are stack slots or light userdata |
| Dispatch | `function_table` | C function pointer via `lua_CFunction` typedef |
| Exceptions | `translate_to_result` | `lua_pcall` catches errors → `Result[T, LuaError]` |
| Generics | `none` | Lua is dynamically typed |
| Name mangling | none | C ABI; Lua discovers functions by lua_CFunction pointer |
| Threading | single-threaded | Each `lua_State` is single-threaded |
| Initialization | explicit | luaL_newstate / lua_close |
| Null convention | `not_present` | Lua nil is a stack value, not a pointer |

```ferrum
foreign model lua {
    object_model: gc_tracing,
    dispatch:     function_table,
    exceptions:   translate_to_result,
    generics:     none,
    abi:          c,
    null_convention: not_present,
}
```

### 3.8 `python` — CPython C API

| Question | Answer | Rationale |
|---|---|---|
| Object model | `ref_counted` | Py_INCREF / Py_DECREF; cycle collector on top |
| Dispatch | `function_table` | `PyMethodDef` tables; `PyObject_Call` for dynamic dispatch |
| Exceptions | `translate_to_result` | `PyErr_Occurred()` + NULL return → `Result[T, PyException]` |
| Generics | `none` | Python is dynamically typed at the C API level |
| Name mangling | none | C ABI; Python discovers via PyMethodDef tables |
| Threading | lock-required | GIL must be held for all PyObject operations |
| Initialization | explicit | Py_Initialize / Py_Finalize |
| Null convention | `error` | NULL return means an exception was set |

```ferrum
foreign model python {
    object_model: ref_counted,
    dispatch:     function_table,
    exceptions:   translate_to_result,
    generics:     none,
    abi:          c,
    null_convention: error,
}
```

### 3.9 `napi` — Node.js N-API

| Question | Answer | Rationale |
|---|---|---|
| Object model | `gc_tracing` | V8 GC owns all JS values; `napi_value` is an opaque handle |
| Dispatch | `function_table` | `napi_callback` function pointers |
| Exceptions | `translate_to_result` | `napi_status` codes + `napi_throw_*` → `Result[T, NapiError]` |
| Generics | `none` | JavaScript is dynamically typed |
| Name mangling | none | C ABI; Node discovers the module via a known entry-point symbol |
| Threading | single-threaded | N-API operates on the JS main thread |
| Initialization | implicit | `napi_register_module_v1` called by Node at load time |
| Null convention | `not_present` | `napi_value` is opaque; null is fetched via `napi_get_null` |

```ferrum
foreign model napi {
    object_model: gc_tracing,
    dispatch:     function_table,
    exceptions:   translate_to_result,
    generics:     none,
    abi:          c,
    null_convention: not_present,
}
```

### 3.10 `beam` — Erlang NIF

| Question | Answer | Rationale |
|---|---|---|
| Object model | `vm_opaque` | BEAM GC owns heap; `ErlNifTerm` is an opaque integer token |
| Dispatch | `function_table` | NIF table registered via `ERL_NIF_INIT` |
| Exceptions | `translate_to_result` | `enif_make_badarg` / `enif_raise_exception` → BEAM exception on return |
| Generics | `none` | BEAM is dynamically typed |
| Name mangling | none | C ABI; BEAM discovers NIFs via the NIF table |
| Threading | lock-required | Must not block BEAM scheduler; use dirty schedulers for long work |
| Initialization | explicit | `nif_load` called once at module load |
| Null convention | `not_present` | No null; errors are BEAM terms |

```ferrum
foreign model beam {
    object_model: vm_opaque,
    dispatch:     function_table,
    exceptions:   translate_to_result,
    generics:     none,
    abi:          c,
    null_convention: not_present,
}
```

### 3.11 `dart` — Dart FFI

| Question | Answer | Rationale |
|---|---|---|
| Object model | `gc_tracing` | Dart VM GC; `Dart_Handle` is opaque |
| Dispatch | `direct` | Dart FFI resolves by C symbol name from the shared library |
| Exceptions | `none` | Dart exceptions do not propagate into native code |
| Generics | `none` | Dart FFI uses concrete `NativeFunction` types |
| Name mangling | none | C ABI; Dart loads by symbol name |
| Threading | thread-safe | Dart isolates are independent; native code may run on any thread |
| Initialization | implicit | Dart runtime is already running when native code is called |
| Null convention | `not_present` | Dart is null-safe; null only appears if declared as a nullable type |

```ferrum
foreign model dart {
    object_model: gc_tracing,
    dispatch:     direct,
    exceptions:   none,
    generics:     none,
    abi:          c,
    null_convention: not_present,
}
```

### 3.12 `graalvm` — GraalVM Polyglot

| Question | Answer | Rationale |
|---|---|---|
| Object model | `vm_opaque` | GraalVM Truffle owns all polyglot values; `PolyglotValue` is a context-scoped reference |
| Dispatch | `message_send` | Truffle interop messages (EXECUTE, READ, INVOKE, etc.) |
| Exceptions | `translate_to_result` | Guest language exceptions → `Result[T, PolyglotException]` |
| Generics | `none` | All values are dynamically typed `PolyglotValue` at the boundary |
| Name mangling | none | No symbol-level interface; uses Truffle message protocol |
| Threading | single-threaded | Polyglot context is bound to its creating thread |
| Initialization | explicit | Polyglot context must be created and closed |
| Null convention | `not_present` | No null; use `Option[T]` |

```ferrum
foreign model graalvm {
    object_model: vm_opaque,
    dispatch:     message_send,
    exceptions:   translate_to_result,
    generics:     none,
    abi:          c,
    null_convention: not_present,
}
```

---

## Part 4: OS Platform Worked Examples

Each entry summarizes how the authoring process was applied. Full trait implementations and capability maps are in [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md) and [ferrum-stdlib-sys.md](ferrum-stdlib-sys.md).

### 4.1 `none` — Bare Metal (Compiler Built-In Reference)

| Question | Answer |
|---|---|
| API style | Direct hardware access; no OS API |
| Filesystem | None |
| Networking | Hardware peripheral drivers via direct MMIO |
| Process model | None — single-image, no process separation |
| Threading | Hardware threads and ISRs only; no OS scheduler |
| Capability model | None — full hardware access or nothing |
| Memory model | Flat or hardware-segmented; no virtual memory unless hardware provides it |
| Min version | Hardware-dependent |

**Platform traits implemented:** `Time` and `Entropy` (if hardware RNG present); all others not applicable.

`none` is the only OS the compiler knows natively. There is no `sys.none` module — bare-metal code uses `core` and `alloc` directly, with hardware access through `extern(c)` or inline assembly. There is no kernel intermediary to wrap.

### 4.2 `posix` — POSIX Base Layer

| Question | Answer |
|---|---|
| API style | C library (libc); POSIX.1-2008 syscall interface |
| Filesystem | POSIX paths; open/read/write/stat/readdir |
| Networking | BSD sockets (socket/bind/connect/accept/send/recv) |
| Process model | fork/exec/waitpid; POSIX signals converted to channel messages |
| Threading | pthreads (create/join/mutex/condvar/rwlock/barrier) |
| Capability model | DAC (uid/gid/permission bits); optional MAC (SELinux, AppArmor) |
| Memory model | Virtual memory; mmap/munmap/mprotect |
| Min version | POSIX.1-2008 (SUSv4) |

**Platform traits implemented:** all six.

`posix` is a base layer, not a standalone target. Linux, BSD, and macOS all build on it. Code targeting `sys.posix` gets the lowest common denominator. For platform-specific features, use `sys.linux`, `sys.bsd`, or `sys.darwin`.

### 4.3 `linux` — Linux Platform

| Question | Answer |
|---|---|
| API style | Linux syscall ABI + glibc or musl (target_env = "gnu" or "musl") |
| Filesystem | POSIX + inotify, fanotify, io_uring file operations |
| Networking | POSIX sockets + epoll, io_uring net, netlink, AF_XDP |
| Process model | clone/execve, pidfd, namespaces, cgroups |
| Threading | pthreads + futex primitives; io_uring for async I/O |
| Capability model | DAC + Linux capabilities + seccomp-BPF + Landlock + AppArmor/SELinux |
| Memory model | Virtual memory + mmap + huge pages + io_uring registered buffers |
| Min version | Linux 5.4 (io_uring baseline); later kernels required for newer features |

**Platform traits implemented:** all six; Linux-specific extensions under `sys.linux`.

### 4.4 `bsd` — BSD Family

| Question | Answer |
|---|---|
| API style | BSD/POSIX C library |
| Filesystem | POSIX + kqueue file watch events |
| Networking | POSIX sockets + kqueue I/O readiness |
| Process model | fork/exec; FreeBSD Capsicum for capability mode |
| Threading | pthreads |
| Capability model | DAC + Capsicum (FreeBSD) + pledge/unveil (OpenBSD) |
| Memory model | Virtual memory + mmap |
| Min version | FreeBSD 13.0, OpenBSD 7.0, NetBSD 9.0 |

**Platform traits implemented:** all six; BSD-specific extras under `sys.bsd`.

### 4.5 `windows` — Windows Platform

| Question | Answer |
|---|---|
| API style | Win32 API (kernel32, ntdll, ws2_32) via `extern(c.system)` |
| Filesystem | Win32 CreateFile/ReadFile/WriteFile; NTFS; UNC and extended-length paths |
| Networking | Winsock2 (WSAStartup); I/O Completion Ports for async |
| Process model | CreateProcess/TerminateProcess; Job Objects for resource containment |
| Threading | CreateThread; SRW locks; I/O Completion Ports |
| Capability model | ACL/token-based security; AppContainer for sandboxing |
| Memory model | Virtual memory + VirtualAlloc/VirtualProtect; heap manager |
| Min version | Windows 10 build 17763 (1809) |

**Platform traits implemented:** all six; Windows-specific extras under `sys.windows`.

### 4.6 `wasi` — WebAssembly System Interface

| Question | Answer |
|---|---|
| API style | WASI Preview 2 component model; WIT interface types |
| Filesystem | Capability-based via `wasi:filesystem/types` — no absolute paths |
| Networking | `wasi:sockets` — TCP/UDP, capability-gated by host |
| Process model | None — WASM is a single component; no fork or exec |
| Threading | `wasi:threads` proposal (opt-in; not in Preview 2 core) |
| Capability model | Capability-based — all resources are explicit capabilities granted by host |
| Memory model | Linear memory (wasm32 = 4 GB address space); no virtual memory pages |
| Min version | WASI Preview 2 (2024) |

**Platform traits implemented:** FileSystem, Network, Time, Entropy; Process and Threading not applicable.

### 4.7 `zephyr` — Zephyr RTOS

| Question | Answer |
|---|---|
| API style | Zephyr kernel API (C); no libc |
| Filesystem | LittleFS or FAT via Zephyr FS API (`fs_open`, `fs_read`) — optional |
| Networking | Zephyr network stack via `zsock_*` BSD-compatible socket interface — optional |
| Process model | None — threads only; no fork or exec |
| Threading | Zephyr threads (`k_thread_create`); semaphores, mutexes, message queues |
| Capability model | None at default configuration; optional MPU isolation |
| Memory model | Static allocation strongly preferred; heap is optional and bounded |
| Min version | Zephyr 3.4 |

**Platform traits implemented:** FileSystem (if storage configured), Network (if net stack enabled), Time, Threading. Process and Entropy are hardware-dependent.

### 4.8 `fuchsia` — Fuchsia / Zircon

| Question | Answer |
|---|---|
| API style | Zircon system calls + FIDL component framework |
| Filesystem | VFS via `fuchsia.io` FIDL protocol; capability-based |
| Networking | `fuchsia.net` FIDL protocol stack |
| Process model | Component framework — isolated components, not POSIX processes |
| Threading | Zircon threads (zx_thread_create); futex-based synchronization |
| Capability model | Capability-based — all resources are Zircon handles; handles carry explicit rights |
| Memory model | VMO (Virtual Memory Objects); demand paging; no POSIX mmap |
| Min version | Fuchsia API level 15 |

**Platform traits implemented:** all six via FIDL proxies; Fuchsia-specific extras under `sys.fuchsia`.

### 4.9 `ohos` — OpenHarmony

| Question | Answer |
|---|---|
| API style | OpenHarmony NDK (C/C++) + HDF driver framework |
| Filesystem | POSIX-compatible with OpenHarmony sandbox model |
| Networking | SoftBus distributed networking (discovery + sessions) |
| Process model | Ability lifecycle model (not POSIX fork/exec) |
| Threading | pthreads + EventRunner/EventHandler message loop |
| Capability model | OpenHarmony permission model (coarse-grained app permissions) |
| Memory model | Virtual memory + JeMalloc |
| Min version | OpenHarmony API level 10 |

**Platform traits implemented:** FileSystem, Network, Time, Threading. Process maps to the Ability lifecycle; Entropy via `/dev/random`.

---

## Part 5: Adding a New OS — Worked Example

This section walks through the full authoring process for a real OS that is not yet in the standard library: **QNX Neutrino 8**.

### Step 1 — Characterize

| Question | Answer |
|---|---|
| API style | POSIX-compatible C library + QNX-specific APIs (`MsgSend`/`MsgReceive`, pulses) |
| Filesystem | POSIX paths + QNX resource manager protocol |
| Networking | POSIX sockets; io-pkt network stack |
| Process model | fork/exec/spawn; QNX message-passing IPC via channels |
| Threading | pthreads + QNX-specific scheduling classes (FIFO, RR, SPORADIC, ADAPTIVE) |
| Capability model | POSIX DAC + QNX abilities (coarse-grained capability system) |
| Memory model | Virtual memory + typed memory objects (`posix_typed_mem_open`) |
| Min version | QNX SDP 8.0 |

### Step 2 — Module Structure

```
sys.qnx
├── ffi/          QNX-specific types: pulse_t, _pulse, _msg_info, sigevent, etc.
├── types/        Ferrum wrappers: ChannelId, ConnectionId, PulseCode
├── fs/           FileSystem impl (POSIX base + resource manager client)
├── net/          Network impl (POSIX sockets via io-pkt)
├── process/      Process impl (fork/exec/spawn + channels)
├── thread/       pthreads + QNX scheduling: sched_setscheduler, SPORADIC params
├── time/         Time impl (CLOCK_MONOTONIC + QNX timer pulse delivery)
├── mem/          mmap + posix_typed_mem_open for typed physical memory regions
└── platform/     QNX-specific: channel IPC, pulse messages, QNX abilities
```

### Step 3 — Raw FFI Layer (excerpt)

```ferrum
// sys/qnx/ffi/channel.fe
extern(c) {
    fn ChannelCreate(flags: c_uint): c_int ! Unsafe
    fn ChannelDestroy(chid: c_int): c_int ! Unsafe
    fn ConnectAttach(nd: u32, pid: pid_t, chid: c_int, index: c_uint, flags: c_int): c_int ! Unsafe
    fn MsgSend(coid: c_int, smsg: *const c_void, sbytes: usize,
               rmsg: *mut c_void, rbytes: usize): c_long ! Unsafe
    fn MsgReceive(chid: c_int, msg: *mut c_void, bytes: usize,
                  info: *mut MsgInfo): c_int ! Unsafe
}

@repr(C)
struct MsgInfo {
    nd:        u32,
    srcnd:     u32,
    pid:       pid_t,
    tid:       i32,
    chid:      i32,
    scoid:     i32,
    coid:      i32,
    msglen:    i16,
    srcmsglen: i16,
    dstmsglen: i16,
    priority:  i16,
    flags:     u32,
    reserved:  u32,
}
// Source: QNX SDP 8.0, <sys/neutrino.h>, verified 2025-01
```

### Step 4 — Safe Wrapper Layer (excerpt)

```ferrum
// sys/qnx/platform/channel.fe
struct Channel { chid: c_int }

impl Channel {
    fn create(flags: ChannelFlags) -> Result[Self, QnxError] ! IO {
        let chid = unsafe { ffi::ChannelCreate(flags.bits()) };
        if chid < 0 { Err(QnxError::from_errno()) }
        else { Ok(Channel { chid }) }
    }

    fn receive(self: &Self, buf: &mut [u8]) -> Result[(usize, MsgInfo), QnxError] ! IO { ... }
}

impl Drop for Channel {
    fn drop(self: &mut Self) { unsafe { ffi::ChannelDestroy(self.chid) }; }
}
```

### Step 5 — Platform Traits

```ferrum
struct QnxPlatform

impl FileSystem  for QnxPlatform { ... }   // delegate to sys.posix internals
impl Network     for QnxPlatform { ... }   // delegate to sys.posix sockets
impl Process     for QnxPlatform { ... }   // fork/exec + QNX spawn
impl Time        for QnxPlatform { ... }   // CLOCK_MONOTONIC + QNX timer pulses
impl Entropy     for QnxPlatform { ... }   // /dev/random
impl Threading   for QnxPlatform { ... }   // pthreads + QNX scheduling classes

// QNX channel IPC is not a standard trait — lives in sys.qnx.platform
```

### Step 6 — cfg Registration

```
# Ferrum.toml
[platform]
target_os     = "qnx"
target_family = "unix"
target_env    = "nto"      # QNX Neutrino OS ABI environment
min_version   = "qnx:8.0"
```

### Step 7 — Capability Map

| Ferrum effect | QNX mechanism | Notes |
|---|---|---|
| `! FS` | open/stat/readdir | QNX abilities: `PROCMGR_AID_PATHSPACE` |
| `! Net` | socket/connect | QNX abilities: `PROCMGR_AID_NETWORK` |
| `! Spawn` | fork/posix_spawn | QNX abilities: `PROCMGR_AID_SPAWN` |
| `! Time` | clock_gettime | Generally unrestricted |
| `! Entropy` | /dev/random | Generally unrestricted |
| `! IO` (IPC) | MsgSend/MsgReceive | QNX-specific; always unrestricted for channels the process owns |

### Step 8 — Tier and Version

- **Tier:** Start at Tier 3 (community). Promote to Tier 2 once CI is configured. Tier 1 requires dedicated hardware in the test infrastructure.
- **Min version:** QNX SDP 8.0 (first release with modern process ability API)

---

*End of Ferrum Binding Authoring Guide.*

*Full `extern(model)` API declarations: [ferrum-lang-foreign-models.md](ferrum-lang-foreign-models.md)*
*Full platform trait and capability mapping specs: [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md)*
*sys module organization: [ferrum-stdlib-sys.md](ferrum-stdlib-sys.md)*
