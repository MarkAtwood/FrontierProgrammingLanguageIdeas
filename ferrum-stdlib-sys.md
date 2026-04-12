# Ferrum Standard Library — sys, posix, windows

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)
**See also:** [Platform Abstraction Layer](ferrum-stdlib-platform.md) for platform traits, capability mapping, and detailed specs for Linux, BSD, WASI, Zephyr · [Binding Authoring Guide](ferrum-binding-authoring.md) for the process of adding a new OS

---

## 12. sys — System Interfaces

### 12.1 Organization

`sys` is the portability layer. Platform-specific behavior lives in sub-namespaces.

```
std.sys               // cross-platform abstractions; bare-metal (none) is always available
std.sys.posix         // POSIX-specific (opt-in, platform crate)
std.sys.linux         // Linux-specific (opt-in, platform crate) — see platform spec
std.sys.bsd           // BSD-specific (opt-in, platform crate) — see platform spec
std.sys.darwin        // macOS-specific (opt-in, platform crate) — see platform spec
std.sys.windows       // Windows-specific (opt-in, platform crate)
std.sys.fuchsia       // Fuchsia-specific (opt-in, platform crate) — see platform spec
std.sys.ohos          // OpenHarmony-specific (opt-in, platform crate) — see platform spec
std.sys.wasi          // WASI Preview 2 (opt-in, platform crate) — see platform spec
std.sys.zephyr        // Zephyr RTOS (opt-in, platform crate) — see platform spec
std.sys.jvm           // JVM platform (opt-in, platform crate) — available when targeting JVM backend
std.sys.jvm.android   // Android ART specifics (opt-in, requires sys.jvm)
std.sys.clr           // .NET CLR platform (opt-in, platform crate) — available when targeting CLR backend
```

> **Architecture note:** The compiler's OS knowledge bottoms out at `none` — bare metal with no operating system. Every named platform above is defined in a platform crate, not compiled into the language core. The standard library ships platform crates for common targets, but a new OS or RTOS can be supported by adding a crate without modifying the compiler.

This document covers the **cross-platform `sys` module** and the **POSIX/Windows threading APIs**.
For platform-specific features (io_uring, kqueue, Landlock, Capsicum, WASI component model, Zephyr drivers),
see [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md).

### 12.2 Cross-platform sys

```ferrum
// OS name and properties
fn os_name(): &'static str      // "linux", "macos", "windows", "wasi"
fn arch(): &'static str         // "x86_64", "aarch64", "riscv64", "wasm32"
fn pointer_width(): usize       // 32 or 64

// Environment variables — explicit, not global mutable
type EnvVars  // snapshot of environment at a point in time

fn env_vars(): EnvVars ! IO                       // current environment
fn env_var(key: &str): Result[String, VarError] ! IO
fn set_env_var(key: &str, val: &str) ! IO           // affects current process
fn remove_env_var(key: &str) ! IO

// NOTE: set_env_var is inherently thread-unsafe on POSIX.
// Calling it after spawning threads is a detectable misuse.
// The compiler warns on this pattern.

// Exit
fn exit(code: i32): never ! IO
fn abort(): never            // immediate, no destructors

// Process info
fn pid(): u32 ! IO
fn page_size(): usize
fn num_cpus(): usize

// Executable path
fn current_exe(): Result[PathBuf, IoError] ! IO

// Working directory
fn current_dir(): Result[PathBuf, IoError] ! IO
// NOTE: set_current_dir exists but is in sys.posix — it is process-global
// and thread-unsafe. The cross-platform API does not expose it.
```

---

## 13. Platform-Specific Modules (Summary)

The following platform modules are fully specified in [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md):

| Module | Platform | Key Features |
|--------|----------|--------------|
| `sys.linux` | Linux 5.4+ | io_uring, epoll, landlock, seccomp, pidfd, namespaces |
| `sys.bsd` | FreeBSD/OpenBSD/NetBSD | kqueue, Capsicum, pledge/unveil |
| `sys.darwin` | macOS/iOS | kqueue, Grand Central Dispatch, XPC, sandbox |
| `sys.fuchsia` | Fuchsia | Zircon kernel, FIDL, capabilities, namespaces, components |
| `sys.ohos` | OpenHarmony 4.0+ | HDF drivers, distributed soft bus, ability framework |
| `sys.wasi` | WASI Preview 2 | Component model, capability-based filesystem, HTTP |
| `sys.zephyr` | Zephyr RTOS | Kernel primitives, GPIO, I2C, SPI, UART, Bluetooth |
| `sys.jvm` | JVM backend | Runtime info, GC control, class loading, NIO, JVM threads |
| `sys.jvm.android` | Android ART | Looper/Handler, Binder IPC, ART GC, DEX class loading |
| `sys.clr` | CLR backend | Runtime info, GC control, assembly loading, ThreadPool, Marshal |

These modules follow the layered architecture:
- **Layer 2:** Safe wrappers with Ferrum types and effects
- **Layer 1:** Raw FFI bindings in `sys.{platform}.ffi` (unsafe)

Import any platform module to signal explicit platform dependency:

```ferrum
import std.sys.linux.io_uring.{IoUring, SqEntry}  // Linux-specific
import std.sys.wasi.filesystem.{Descriptor, preopens}  // WASI-specific
import std.sys.bsd.kqueue.{Kqueue, KEvent}  // BSD-specific
```

---

## 14. posix — POSIX Namespace

Explicit opt-in. Using `std.sys.posix` signals platform dependency.

```ferrum
import std.sys.posix

// File descriptors — typed, not raw integers
type Fd(i32)
    where value >= 0

impl Fd {
    const STDIN:  Self = Fd(0)
    const STDOUT: Self = Fd(1)
    const STDERR: Self = Fd(2)

    unsafe fn from_raw(fd: i32): Self
    fn into_raw(self): i32     // does NOT close
    fn close(self): Result[(), PosixError]  // consumes — double-close is impossible
}

// Low-level file ops
fn open(path: &CStr, flags: OpenFlags, mode: Mode): Result[Fd, PosixError] ! Unsafe + IO
fn read(fd: &Fd, buf: &mut [u8]): Result[usize, PosixError] ! Unsafe + IO
fn write(fd: &Fd, buf: &[u8]): Result[usize, PosixError] ! Unsafe + IO
fn lseek(fd: &Fd, offset: i64, whence: Whence): Result[i64, PosixError] ! Unsafe + IO
fn fstat(fd: &Fd): Result[Stat, PosixError] ! IO
fn ftruncate(fd: &Fd, len: i64): Result[(), PosixError] ! IO
fn fsync(fd: &Fd): Result[(), PosixError] ! IO
fn fcntl(fd: &Fd, cmd: FcntlCmd): Result[i32, PosixError] ! Unsafe + IO
fn ioctl(fd: &Fd, request: u64, ...): Result[i32, PosixError] ! Unsafe + IO
fn dup(fd: &Fd): Result[Fd, PosixError] ! IO
fn dup2(fd: &Fd, newfd: Fd): Result[Fd, PosixError] ! IO

// Signals — NOT raw signal handlers.
// Signals are converted to messages delivered to a channel.
fn signal_channel(signals: &[Signal]): Result[SignalReceiver, PosixError] ! IO
    // Returns a channel that receives signals as messages.
    // This is the ONLY way to handle signals in Ferrum.
    // There are no raw signal handlers. This is intentional.
    // Raw signal handlers are impossible to use correctly in multi-threaded programs.

struct Signal {
    const SIGHUP:  Self = Signal(1)
    const SIGINT:  Self = Signal(2)
    const SIGTERM: Self = Signal(15)
    const SIGUSR1: Self = Signal(10)
    const SIGUSR2: Self = Signal(12)
    // ... etc.
}

// mmap
struct MmapFlags { ... }
struct Protection { ... }

unsafe fn mmap(
    addr: Option[*mut u8],
    len: usize,
    prot: Protection,
    flags: MmapFlags,
    fd: Option[&Fd],
    offset: i64,
): Result[MmapGuard, PosixError] ! Unsafe + IO

struct MmapGuard {
    // Drops the mapping on Drop
    fn as_slice(&self): &[u8]
    fn as_mut_slice(&mut self): &mut [u8]
    fn len(&self): usize
    fn flush(&self): Result[(), PosixError] ! IO
    fn flush_range(&self, offset: usize, len: usize): Result[(), PosixError] ! IO
}

// PosixError — errno mapped to a proper type
struct PosixError {
    code: i32,
    fn kind(&self): IoErrorKind
    fn message(&self): &str
}

// errno is NOT exposed directly.
// All syscall wrappers translate errno to PosixError on return.
// There is no way to read or write errno directly from safe code.

// Process
fn fork(): Result[ForkResult, PosixError] ! Unsafe + IO
enum ForkResult { Parent(u32), Child }   // u32 = child PID

fn execve(path: &CStr, args: &[&CStr], env: &[&CStr]): Result[never, PosixError] ! Unsafe + IO
fn waitpid(pid: i32, options: WaitFlags): Result[WaitStatus, PosixError] ! IO

// sockets (low-level)
fn socket(domain: AfFamily, kind: SockType, protocol: i32): Result[Fd, PosixError] ! IO
fn bind_raw(fd: &Fd, addr: &SockAddr): Result[(), PosixError] ! IO
fn listen(fd: &Fd, backlog: i32): Result[(), PosixError] ! IO
fn accept_raw(fd: &Fd): Result[(Fd, SockAddr), PosixError] ! IO
fn connect_raw(fd: &Fd, addr: &SockAddr): Result[(), PosixError] ! IO
fn setsockopt(fd: &Fd, level: i32, optname: i32, optval: &[u8]): Result[(), PosixError] ! IO
fn getsockopt(fd: &Fd, level: i32, optname: i32, optval: &mut [u8]): Result[(), PosixError] ! IO

// epoll/kqueue — abstracted as Poller
struct Poller {
    fn new(): Result[Self, PosixError] ! IO
    fn add(&mut self, fd: &Fd, events: Events, token: u64): Result[(), PosixError] ! IO
    fn modify(&mut self, fd: &Fd, events: Events, token: u64): Result[(), PosixError] ! IO
    fn delete(&mut self, fd: &Fd): Result[(), PosixError] ! IO
    fn wait(&mut self, events: &mut Vec[Event], timeout: Option[Duration])
        : Result[usize, PosixError] ! IO
}
```

### 14.1 pthread — Raw POSIX threads

NOTE: Prefer `scope.spawn()` for portable code. Use pthread only when you
need platform-specific thread attributes (affinity, priority, stack size,
detach state) that the portable runtime does not expose.

```ferrum
mod pthread {
    struct Thread {
        fn join(self): Result[(), PosixError] ! IO  // consumes handle
        fn detach(self): Result[(), PosixError] ! IO
        fn id(&self): ThreadId
    }

    type ThreadId(u64)

    struct ThreadAttrs {
        fn default(): Self
        fn stack_size(self, size: usize): Self
        fn detached(self): Self
        fn guard_size(self, size: usize): Self
    }

    fn spawn[F: FnOnce() + Send](
        attrs: ThreadAttrs,
        f: F,
    ): Result[Thread, PosixError] ! IO + Unsafe
        // Unsafe because spawned thread has no scope bound —
        // it can outlive the caller if not joined.

    fn current(): ThreadId
    fn yield_now() ! IO
    fn sleep(duration: Duration) ! IO

    // Thread-local storage (beyond @thread_local statics)
    struct Key[T] {
        fn new(destructor: Option[fn(*mut T)]): Result[Self, PosixError]
        unsafe fn get(&self): *mut T
        unsafe fn set(&self, value: *mut T): Result[(), PosixError]
    }

    // Affinity (Linux-specific, cfg-gated)
    @cfg(target_os = "linux")
    struct CpuSet { ... }

    @cfg(target_os = "linux")
    fn set_affinity(thread: &Thread, cpuset: &CpuSet): Result[(), PosixError] ! IO

    @cfg(target_os = "linux")
    fn get_affinity(thread: &Thread): Result[CpuSet, PosixError] ! IO

    // Scheduling (where available)
    enum SchedPolicy { Other, Fifo, RoundRobin, Batch, Idle }

    fn set_schedparam(
        thread: &Thread,
        policy: SchedPolicy,
        priority: i32,
    ): Result[(), PosixError] ! IO + Unsafe
        // Unsafe: SCHED_FIFO/RR require CAP_SYS_NICE or root

    // pthread mutex (when you need priority inheritance, robustness, etc.)
    struct Mutex {
        fn new(attrs: MutexAttrs): Result[Self, PosixError]
        fn lock(&self): Result[MutexGuard, PosixError] ! IO
        fn try_lock(&self): Result[MutexGuard, PosixError]
        fn unlock(guard: MutexGuard)  // consumed by guard drop
    }

    struct MutexAttrs {
        fn default(): Self
        fn kind(self, kind: MutexKind): Self
        fn robust(self): Self           // PTHREAD_MUTEX_ROBUST
        fn pshared(self): Self          // PTHREAD_PROCESS_SHARED
        fn protocol(self, p: MutexProtocol): Self
    }

    enum MutexKind { Normal, Recursive, ErrorCheck }
    enum MutexProtocol { None, Inherit, Protect }

    // pthread condition variables
    struct Condvar {
        fn new(): Result[Self, PosixError]
        fn wait(&self, guard: &mut MutexGuard): Result[(), PosixError] ! IO
        fn wait_timeout(&self, guard: &mut MutexGuard, timeout: Duration)
            : Result[bool, PosixError] ! IO  // returns false on timeout
        fn signal(&self): Result[(), PosixError]
        fn broadcast(&self): Result[(), PosixError]
    }

    // pthread barriers
    struct Barrier {
        fn new(count: u32): Result[Self, PosixError]
        fn wait(&self): Result[BarrierWaitResult, PosixError] ! IO
    }

    enum BarrierWaitResult { Leader, Follower }

    // pthread read-write locks
    struct RwLock {
        fn new(): Result[Self, PosixError]
        fn read(&self): Result[RwLockReadGuard, PosixError] ! IO
        fn write(&self): Result[RwLockWriteGuard, PosixError] ! IO
        fn try_read(&self): Result[RwLockReadGuard, PosixError]
        fn try_write(&self): Result[RwLockWriteGuard, PosixError]
    }
}
```

### 14.2 Why Raw Threads Are Here, Not in `std`

Raw OS threads have platform-specific semantics that cannot be fully
abstracted without losing functionality:

- **Affinity** — Linux `sched_setaffinity` vs macOS thread policy vs Windows `SetThreadAffinityMask`
- **Priority** — POSIX `sched_setscheduler` vs Windows priority classes
- **Stack size** — platform-specific limits and guard page behavior
- **Detach semantics** — pthread detach vs Windows thread handle lifecycle

The portable `scope.spawn()` abstraction handles 99% of use cases. When you
need the platform-specific features, opt into `sys.posix.pthread` or
`sys.windows.thread` explicitly. The import is the declaration of
non-portability.

---

## 15. windows — Windows Thread and Sync API

```ferrum
import std.sys.windows

mod thread {
    struct Thread {
        fn join(self): Result[(), WinError] ! IO
        fn id(&self): ThreadId
        fn handle(&self): Handle  // raw HANDLE for WinAPI interop
    }

    type ThreadId(u32)

    struct ThreadAttrs {
        fn default(): Self
        fn stack_size(self, size: usize): Self
        fn creation_flags(self, flags: CreationFlags): Self
    }

    bitflags CreationFlags {
        const CREATE_SUSPENDED = 0x00000004
        const STACK_SIZE_PARAM_IS_A_RESERVATION = 0x00010000
    }

    fn spawn[F: FnOnce() + Send](
        attrs: ThreadAttrs,
        f: F,
    ): Result[Thread, WinError] ! IO + Unsafe

    fn current(): ThreadId
    fn yield_now() ! IO
    fn sleep(duration: Duration) ! IO

    // Priority
    enum Priority {
        Idle, Lowest, BelowNormal, Normal,
        AboveNormal, Highest, TimeCritical,
    }

    fn set_priority(thread: &Thread, priority: Priority): Result[(), WinError] ! IO
    fn get_priority(thread: &Thread): Result[Priority, WinError] ! IO

    // Affinity
    fn set_affinity_mask(thread: &Thread, mask: u64): Result[u64, WinError] ! IO
    fn get_affinity_mask(thread: &Thread): Result[u64, WinError] ! IO

    // Processor group (for >64 CPUs)
    fn set_ideal_processor(thread: &Thread, processor: u32): Result[u32, WinError] ! IO
}

// Windows synchronization primitives
mod sync {
    // CRITICAL_SECTION (lighter than Mutex for same-process use)
    struct CriticalSection { ... }

    // SRWLOCK (slim reader-writer lock)
    struct SrwLock { ... }

    // CONDITION_VARIABLE
    struct ConditionVariable { ... }

    // Event objects
    struct Event {
        fn new(manual_reset: bool, initial_state: bool): Result[Self, WinError] ! IO
        fn set(&self): Result[(), WinError] ! IO
        fn reset(&self): Result[(), WinError] ! IO
        fn wait(&self, timeout: Option[Duration]): Result[WaitResult, WinError] ! IO
    }

    // Semaphore
    struct Semaphore {
        fn new(initial: u32, maximum: u32): Result[Self, WinError] ! IO
        fn release(&self, count: u32): Result[u32, WinError] ! IO
        fn wait(&self, timeout: Option[Duration]): Result[WaitResult, WinError] ! IO
    }

    enum WaitResult { Signaled, Timeout, Abandoned }
}
```

---

---

## 16. jvm — JVM Platform Module

Available only when Ferrum targets the JVM backend. Importing `std.sys.jvm` signals that the code is JVM-specific and will not compile to native or WASM targets.

```ferrum
import std.sys.jvm
```

### 16.1 Runtime Information

```ferrum
mod runtime {
    // JVM version (e.g. "21.0.2+13")
    fn version(): &'static str

    // Runtime name ("OpenJDK 64-Bit Server VM", "GraalVM CE", ...)
    fn vm_name(): &'static str

    // Number of logical processors available to the JVM
    fn available_processors(): usize

    // Peak / current thread count
    fn thread_count(): usize
    fn peak_thread_count(): usize

    // Uptime since JVM start
    fn uptime(): Duration

    // JVM input arguments (flags passed to the JVM, not the application)
    fn jvm_input_arguments(): Vec[String] ! IO

    // Classpath entries
    fn classpath(): Vec[PathBuf]

    // System properties (java.home, java.version, file.separator, etc.)
    fn system_property(key: &str): Option[String] ! IO
    fn system_properties(): HashMap[String, String] ! IO
}
```

### 16.2 Garbage Collector

```ferrum
mod gc {
    // Suggest a GC cycle (equivalent to System.gc() — advisory, not guaranteed)
    fn suggest_collection() ! IO

    // GC pool names and stats
    struct MemoryPool {
        fn name(&self): &str
        fn used_bytes(&self): i64
        fn committed_bytes(&self): i64
        fn max_bytes(&self): i64
    }

    fn memory_pools(): Vec[MemoryPool] ! IO

    // Heap summary
    struct HeapSummary {
        used_bytes: i64,
        committed_bytes: i64,
        max_bytes: i64,
    }

    fn heap_summary(): HeapSummary ! IO

    // GC notifications — a channel receives an event after each GC cycle
    struct GcEvent {
        gc_name: String,       // e.g. "G1 Young Generation"
        action: String,        // e.g. "end of minor GC"
        duration: Duration,
        before: HeapSummary,
        after: HeapSummary,
    }

    fn gc_event_channel(): Receiver[GcEvent] ! IO
        // Subscribe to GC events via MXBean NotificationListener.
        // Drop the Receiver to unsubscribe.
}
```

### 16.3 Threads

JVM threading beyond the portable `scope.spawn()` API. Use when you need
JVM-specific thread attributes (daemon status, thread group, priority).

```ferrum
mod thread {
    // JVM thread priority (1 = MIN, 5 = NORM, 10 = MAX)
    enum Priority { Min = 1, Normal = 5, Max = 10 }

    struct ThreadAttrs {
        fn default(): Self
        fn name(self, name: &str): Self
        fn daemon(self, daemon: bool): Self  // daemon threads don't block JVM exit
        fn priority(self, p: Priority): Self
        fn stack_size(self, bytes: usize): Self
    }

    struct Thread {
        fn join(self): Result[(), JvmError] ! IO
        fn id(&self): i64        // java.lang.Thread.getId()
        fn name(&self): String
        fn is_daemon(&self): bool
        fn is_alive(&self): bool ! IO
        fn interrupt(&self) ! IO
    }

    fn spawn[F: FnOnce() + Send](
        attrs: ThreadAttrs,
        f: F,
    ): Result[Thread, JvmError] ! IO

    fn current_id(): i64
    fn current_name(): String
    fn yield_now() ! IO
    fn sleep(duration: Duration) ! IO

    // Enumerate all live threads (via ThreadMXBean)
    fn all_thread_ids(): Vec[i64] ! IO
    fn thread_info(id: i64): Option[ThreadInfo] ! IO

    struct ThreadInfo {
        id: i64,
        name: String,
        state: ThreadState,
        blocked_count: i64,
        waited_count: i64,
        lock_name: Option[String],
        lock_owner_id: Option[i64],
    }

    enum ThreadState {
        New, Runnable, Blocked, Waiting, TimedWaiting, Terminated
    }
}
```

### 16.4 Class Loading

```ferrum
mod classloader {
    // Load a class by its JVM internal name (e.g. "java/util/ArrayList")
    fn load_class(name: &str): Result[JvmClass, JvmError] ! IO

    struct JvmClass {
        fn name(&self): String
        fn superclass(&self): Option[JvmClass]
        fn interfaces(&self): Vec[JvmClass]
        fn is_interface(&self): bool
        fn is_array(&self): bool

        // Reflective invocation — use carefully; bypasses Ferrum type checking
        unsafe fn new_instance(&self): Result[JvmObject, JvmError] ! IO
        unsafe fn invoke_method(
            &self,
            obj: Option[&JvmObject],
            method_name: &str,
            args: &[JvmValue],
        ): Result[JvmValue, JvmError] ! IO
    }

    // Load a class from a byte array (e.g. generated bytecode, downloaded JAR)
    unsafe fn define_class(
        name: &str,
        bytecode: &[u8],
        parent: Option[ClassLoaderRef],
    ): Result[JvmClass, JvmError] ! IO
}

// JVM value union — the dynamic type at the JVM level
enum JvmValue {
    Boolean(bool),
    Byte(i8), Short(i16), Int(i32), Long(i64),
    Float(f32), Double(f64),
    Char(u16),
    Object(JvmObject),
    Null,
}
```

### 16.5 NIO and Filesystem

The JVM NIO path API, exposed with Ferrum types.

```ferrum
mod nio {
    // java.nio.file.Path — JVM path type (distinct from Ferrum's PathBuf)
    struct NioPath {
        fn to_string(&self): String
        fn resolve(&self, other: &str): Self
        fn parent(&self): Option[Self]
        fn file_name(&self): Option[String]
        fn is_absolute(&self): bool
        fn exists(&self): bool ! IO
    }

    fn path(s: &str): NioPath

    // java.nio.file.Files wrappers
    fn read_bytes(path: &NioPath): Result[Vec[u8], IoError] ! IO
    fn write_bytes(path: &NioPath, data: &[u8]): Result[(), IoError] ! IO
    fn copy(src: &NioPath, dst: &NioPath): Result[u64, IoError] ! IO
    fn move_file(src: &NioPath, dst: &NioPath): Result[(), IoError] ! IO
    fn delete(path: &NioPath): Result[(), IoError] ! IO
    fn create_dir(path: &NioPath): Result[(), IoError] ! IO
    fn create_dirs(path: &NioPath): Result[(), IoError] ! IO
    fn list_dir(path: &NioPath): Result[Vec[NioPath], IoError] ! IO

    // WatchService — directory change events
    struct WatchService {
        fn new(): Result[Self, IoError] ! IO
        fn watch(
            &self,
            path: &NioPath,
            events: &[WatchEvent],
        ): Result[WatchKey, IoError] ! IO
        fn poll(&self): Option[WatchKey]
        fn take(&self): Result[WatchKey, IoError] ! IO  // blocks
    }

    enum WatchEvent { Create, Modify, Delete, Overflow }

    struct WatchKey {
        fn events(&self): Vec[(WatchEvent, NioPath)]
        fn reset(&self): bool
        fn cancel(&self)
    }
}

// java.nio.channels — non-blocking I/O
mod channels {
    struct Selector {
        fn open(): Result[Self, IoError] ! IO
        fn select(&mut self, timeout: Option[Duration]): Result[usize, IoError] ! IO
        fn selected_keys(&self): &[SelectionKey]
        fn close(self): Result[(), IoError] ! IO
    }

    struct SelectionKey {
        fn is_readable(&self): bool
        fn is_writable(&self): bool
        fn is_connectable(&self): bool
        fn is_acceptable(&self): bool
    }

    struct SocketChannel {
        fn open(): Result[Self, IoError] ! IO
        fn connect(&self, addr: &SocketAddr): Result[bool, IoError] ! IO
        fn finish_connect(&self): Result[bool, IoError] ! IO
        fn configure_blocking(&self, blocking: bool): Result[(), IoError] ! IO
        fn read(&self, buf: &mut [u8]): Result[i32, IoError] ! IO
        fn write(&self, buf: &[u8]): Result[i32, IoError] ! IO
        fn register(&self, selector: &Selector, ops: i32): Result[SelectionKey, IoError] ! IO
        fn close(self): Result[(), IoError] ! IO
    }

    struct ServerSocketChannel {
        fn open(): Result[Self, IoError] ! IO
        fn bind(&self, addr: &SocketAddr): Result[(), IoError] ! IO
        fn accept(&self): Result[Option[SocketChannel], IoError] ! IO
        fn configure_blocking(&self, blocking: bool): Result[(), IoError] ! IO
        fn register(&self, selector: &Selector, ops: i32): Result[SelectionKey, IoError] ! IO
        fn close(self): Result[(), IoError] ! IO
    }
}
```

### 16.6 Process

```ferrum
mod process {
    // java.lang.ProcessBuilder wrapper
    struct ProcessBuilder {
        fn new(program: &str, args: &[&str]): Self
        fn env(self, key: &str, val: &str): Self
        fn env_remove(self, key: &str): Self
        fn env_clear(self): Self
        fn working_dir(self, path: &NioPath): Self
        fn stdin(self, cfg: Stdio): Self
        fn stdout(self, cfg: Stdio): Self
        fn stderr(self, cfg: Stdio): Self
        fn spawn(self): Result[Child, IoError] ! IO
    }

    enum Stdio { Inherit, Pipe, Null }

    struct Child {
        fn wait(&mut self): Result[ExitStatus, IoError] ! IO
        fn wait_timeout(&mut self, d: Duration): Result[Option[ExitStatus], IoError] ! IO
        fn kill(&mut self): Result[(), IoError] ! IO
        fn pid(&self): i64
        fn stdin(&mut self): Option[&mut OutputStream]
        fn stdout(&mut self): Option[&mut InputStream]
        fn stderr(&mut self): Option[&mut InputStream>
    }

    struct ExitStatus {
        fn code(&self): Option[i32]
        fn success(&self): bool
    }
}
```

### 16.7 Android ART (`sys.jvm.android`)

ART-specific APIs beyond the base JVM model. Requires `std.sys.jvm`.

```ferrum
import std.sys.jvm.android

// Looper / Handler — Android main-thread message loop
mod looper {
    struct Looper {
        fn prepare() ! IO   // makes a Looper for the current thread
        fn main_looper(): &'static Looper
        fn loop_forever() ! IO   // never returns; processes messages
        fn quit(&self) ! IO
    }

    struct Handler {
        fn new(looper: &Looper): Self
        fn post(self: &Self, f: fn() ! IO) ! IO
        fn post_delayed(self: &Self, f: fn() ! IO, delay: Duration) ! IO
        fn send_empty_message(self: &Self, what: i32) ! IO
    }
}

// Binder IPC — Android inter-process communication
mod binder {
    struct Binder {
        // implement android.os.IBinder in Ferrum
        fn new(): Self
        unsafe fn transact(
            &self,
            code: u32,
            data: &Parcel,
            reply: &mut Parcel,
            flags: u32,
        ): Result[(), BinderError] ! IO
    }

    struct Parcel {
        fn obtain(): Self
        fn write_i32(&mut self, v: i32)
        fn write_i64(&mut self, v: i64)
        fn write_f32(&mut self, v: f32)
        fn write_f64(&mut self, v: f64)
        fn write_string(&mut self, s: &str)
        fn write_bytes(&mut self, b: &[u8])
        fn read_i32(&mut self): i32
        fn read_i64(&mut self): i64
        fn read_f32(&mut self): f32
        fn read_f64(&mut self): f64
        fn read_string(&mut self): String
        fn read_bytes(&mut self): Vec[u8]
        fn recycle(self)
    }
}

// ART GC hints
mod gc {
    // Hint to ART that now is a good time to trim the heap
    fn trim_memory(level: TrimLevel) ! IO

    enum TrimLevel {
        Running,          // app is running, light trim
        UiHidden,         // UI no longer visible
        Background,       // process was moved to background
        Moderate,         // memory pressure is moderate
        Complete,         // memory pressure is critical
    }
}

// DEX class loading
mod dex {
    // Load Ferrum code from a DEX file at runtime (plugin systems, hot patches)
    unsafe fn load_dex(
        dex_path: &str,
        optimize_dir: &str,
        library_path: &str,
    ): Result[classloader::ClassLoaderRef, JvmError] ! IO
}
```

---

## 17. clr — CLR Platform Module

Available only when Ferrum targets the CLR backend. Importing `std.sys.clr` signals CLR-only code.

```ferrum
import std.sys.clr
```

### 17.1 Runtime Information

```ferrum
mod runtime {
    // CLR version ("8.0.1" for .NET 8)
    fn version(): &'static str

    // Framework description ("Microsoft .NET 8.0.1", "Mono 6.12", ...)
    fn framework_description(): String ! IO

    // Runtime identifier (matches dotnet RID: "linux-x64", "win-arm64", ...)
    fn runtime_identifier(): String

    // Number of logical processors
    fn processor_count(): usize

    // Time since process start
    fn uptime(): Duration

    // OS platform detection
    fn is_windows(): bool
    fn is_linux(): bool
    fn is_macos(): bool
    fn is_platform(platform: OsPlatform): bool

    enum OsPlatform { Windows, Linux, MacOS, FreeBSD }

    // OS architecture
    fn os_architecture(): Architecture
    fn process_architecture(): Architecture

    enum Architecture { X86, X64, Arm, Arm64, Wasm, S390x, LoongArch64, Armv6, Ppc64le }
}
```

### 17.2 Garbage Collector

```ferrum
mod gc {
    // Suggest a full GC (equivalent to GC.Collect())
    fn collect(generation: Option[i32]) ! IO
    fn collect_with(
        generation: i32,
        mode: GcCollectionMode,
        blocking: bool,
    ) ! IO

    enum GcCollectionMode { Default, Forced, Optimized, Aggressive }

    // Current generation of a managed object (0, 1, or 2)
    fn get_generation[T](obj: &T): i32

    // Memory info
    struct GcMemoryInfo {
        high_memory_load_threshold_bytes: i64,
        total_available_memory_bytes: i64,
        memory_load_bytes: i64,
        heap_size_bytes: i64,
        fragmented_bytes: i64,
        promoted_bytes: i64,
        pinned_objects_count: i64,
        generation: i32,
        compacted: bool,
        concurrent: bool,
    }

    fn memory_info(kind: GcGenerationKind): GcMemoryInfo ! IO

    enum GcGenerationKind { Any, Ephemeral, FullBlocking, Background }

    // GC notification registration (for server GC threshold alerts)
    fn register_for_full_gc_notification(
        gen2_threshold: i32,
        large_object_heap_threshold: i32,
    ) ! IO

    fn wait_for_full_gc_approach(timeout: Option[Duration]): GcNotificationStatus ! IO
    fn wait_for_full_gc_complete(timeout: Option[Duration]): GcNotificationStatus ! IO

    enum GcNotificationStatus { Succeeded, Failed, Canceled, Timeout, NotApplicable }

    // Suppress/re-register finalizer for an object
    fn suppress_finalize[T](obj: &T) ! IO
    fn re_register_for_finalize[T](obj: &T) ! IO

    // Weak references
    struct WeakRef[T] {
        fn new(target: &T, track_resurrection: bool): Self
        fn is_alive(&self): bool ! IO
        fn try_get_target(&self): Option[ClrRef[T]] ! IO
    }
}
```

### 17.3 Assembly Loading

```ferrum
mod assembly {
    // The default load context
    fn default_context(): AssemblyLoadContext

    struct AssemblyLoadContext {
        fn name(&self): Option[String]

        // Load from disk
        fn load_from_path(&self, path: &str): Result[Assembly, ClrError] ! IO

        // Load from byte array (e.g. downloaded, generated)
        unsafe fn load_from_bytes(
            &self,
            assembly_bytes: &[u8],
            symbols_bytes: Option[&[u8]],
        ): Result[Assembly, ClrError] ! IO

        // Unload (only for collectible contexts)
        fn unload(self): Result[(), ClrError] ! IO
    }

    // Create an isolated, collectible load context (for plugin unloading)
    fn new_collectible_context(name: &str): AssemblyLoadContext

    struct Assembly {
        fn full_name(&self): Option[String]
        fn location(&self): String
        fn get_type(&self, name: &str): Option[ClrType] ! IO
        fn get_types(&self): Vec[ClrType] ! IO
        fn get_exported_types(&self): Vec[ClrType] ! IO
    }

    struct ClrType {
        fn full_name(&self): Option[String]
        fn is_interface(&self): bool
        fn is_abstract(&self): bool
        fn is_value_type(&self): bool
        fn is_generic_type(&self): bool
        fn base_type(&self): Option[ClrType]
        fn interfaces(&self): Vec[ClrType]

        // Reflective construction — bypasses Ferrum type checking
        unsafe fn create_instance(): Result[ClrObject, ClrError] ! IO
        unsafe fn invoke_method(
            obj: Option[&ClrObject],
            method_name: &str,
            args: &[ClrValue],
        ): Result[ClrValue, ClrError] ! IO
    }
}

// CLR value union — the dynamic type at the CLR level
enum ClrValue {
    Bool(bool),
    I8(i8), I16(i16), I32(i32), I64(i64),
    U8(u8), U16(u16), U32(u32), U64(u64),
    F32(f32), F64(f64),
    Char(char),
    Object(ClrObject),
    Null,
}
```

### 17.4 ThreadPool

```ferrum
mod threadpool {
    // Queue a work item to the .NET ThreadPool
    fn queue_work_item(f: fn() ! IO): Result[(), ClrError] ! IO

    // Adjust pool limits
    fn set_min_threads(worker: i32, completion_port: i32): Result[bool, ClrError>
    fn set_max_threads(worker: i32, completion_port: i32): Result[bool, ClrError>
    fn get_min_threads(): (i32, i32)   // (worker, completionPort)
    fn get_max_threads(): (i32, i32)
    fn get_available_threads(): (i32, i32)

    // Wait for a kernel object (file handle, event, etc.)
    fn register_wait(
        handle: Handle,
        callback: fn(state: *mut c_void, timed_out: bool),
        state: *mut c_void,
        timeout: Option[Duration],
        execute_once: bool,
    ): Result[RegisteredWait, ClrError> ! IO + Unsafe

    struct RegisteredWait {
        fn unregister(self, signal_handle: Option[Handle]): bool
    }
}
```

### 17.5 Managed Threads

```ferrum
mod thread {
    enum ThreadPriority { Lowest, BelowNormal, Normal, AboveNormal, Highest }
    enum ApartmentState { Sta, Mta, Unknown }

    struct ThreadAttrs {
        fn default(): Self
        fn name(self, name: &str): Self
        fn is_background(self, bg: bool): Self
        fn priority(self, p: ThreadPriority): Self
        fn apartment_state(self, state: ApartmentState): Self  // COM interop
    }

    struct Thread {
        fn join(self): Result[(), ClrError> ! IO
        fn join_timeout(self, timeout: Duration): Result[bool, ClrError> ! IO
        fn interrupt(&self) ! IO
        fn id(&self): i32
        fn name(&self): Option[String]
        fn is_alive(&self): bool ! IO
        fn is_background(&self): bool
        fn priority(&self): ThreadPriority
    }

    fn spawn[F: FnOnce() + Send](
        attrs: ThreadAttrs,
        f: F,
    ): Result[Thread, ClrError> ! IO

    fn current_id(): i32
    fn yield_now() ! IO
    fn sleep(duration: Duration) ! IO
    fn sleep_zero() ! IO   // Thread.Sleep(0) — yield to equal-or-higher-priority threads
    fn memory_barrier() ! IO   // Thread.MemoryBarrier()
}
```

### 17.6 Interop and Marshaling

```ferrum
mod marshal {
    // Native memory — outside the managed heap
    unsafe fn alloc_hglobal(bytes: usize): Result[*mut u8, ClrError>
    unsafe fn realloc_hglobal(ptr: *mut u8, bytes: usize): Result[*mut u8, ClrError>
    unsafe fn free_hglobal(ptr: *mut u8)

    // CoTaskMem — for COM interop
    unsafe fn alloc_co_task_mem(bytes: usize): Result[*mut u8, ClrError>
    unsafe fn free_co_task_mem(ptr: *mut u8)

    // Pin a managed object so the GC won't move it
    // (needed when passing a managed object pointer to native code)
    unsafe fn alloc_h_global_ansi(s: &str): Result[*mut u8, ClrError>
    unsafe fn ptr_to_string_ansi(ptr: *const u8): String

    // Safe handles — typed wrappers around OS handles that auto-close
    struct SafeHandle {
        fn is_invalid(&self): bool
        fn close(self)
    }

    // GCHandle — pin or keep alive a managed object from native code
    struct GcHandle {
        unsafe fn alloc(target: &ClrObject, kind: GcHandleKind): Self
        unsafe fn target(&self): &ClrObject
        fn free(self)
        unsafe fn addr_of_pinned_object(&self): *mut u8
    }

    enum GcHandleKind { Weak, WeakTrackResurrection, Normal, Pinned }

    // COM interop
    unsafe fn get_iunknown_for_object(obj: &ClrObject): *mut IUnknown
    unsafe fn get_object_for_iunknown(pUnk: *mut IUnknown): Result[ClrObject, ClrError>
    unsafe fn release_com_object(obj: &ClrObject): i32
}
```

### 17.7 Why These Modules Are VM-Specific

`sys.jvm`, `sys.clr`, and `sys.ohos` are opt-in for the same reason `sys.posix` is opt-in: importing them is a declaration of non-portability. Code that uses these modules cannot compile to the native/LLVM backend or cross-compile to another VM target. The import is the platform-dependency contract.

The cross-platform `std` library is built on top of the backend abstractions; it does not expose `sys.jvm`, `sys.clr`, or `sys.ohos` types. Application code that wants portability never sees these modules.

---

## 18. ohos — OpenHarmony Platform Module

Available only when Ferrum targets OpenHarmony (ART/ARK backend or native with NAPI). Importing `std.sys.ohos` signals OpenHarmony-specific code.

```ferrum
import std.sys.ohos
```

**Architecture note:** OpenHarmony's native interop boundary is **NAPI** — the same N-API spec as Node.js. Ferrum code that is called from ArkTS, or that calls ArkTS, uses `extern(napi)`. `sys.ohos` provides the OpenHarmony-specific platform APIs that live *beneath* that boundary: runtime introspection, the message loop, IPC, drivers, and system services.

### 18.1 Runtime Information

```ferrum
mod runtime {
    // OpenHarmony API version level (4 = API 10, analogous to Android SDK level)
    fn api_version(): u32

    // ARK VM version string (e.g. "ArkCompiler 5.0.1")
    fn ark_version(): String

    // Device type — what kind of hardware this is running on
    fn device_type(): DeviceType

    enum DeviceType {
        Phone,
        Tablet,
        Tv,
        Wearable,
        Car,
        TwoinOne,   // 2-in-1 (foldable laptop/tablet)
        Default,    // unspecified embedded device
    }

    // Product model string (e.g. "NOH-AN00")
    fn product_model(): String

    // Brand and manufacturer
    fn brand(): String
    fn manufacturer(): String

    // System parameter lookup — read-only properties from param service
    // (e.g. "const.product.devicetype", "const.ohos.version.release")
    fn system_param(key: &str): Result[String, OhosError] ! IO

    // Number of logical processors
    fn processor_count(): usize

    // OS full version string
    fn os_version(): String
}
```

### 18.2 ARK GC Pressure Hints

Native code should respond to memory pressure signals from the ARK runtime's memory manager. These correspond to the `Memory.Level` enum in the ArkTS `@ohos.app.ability.AbilityConstant` API.

```ferrum
mod gc {
    // Notify ARK runtime of memory pressure from native side
    // (triggers GC and asks other components to trim)
    fn notify_memory_level(level: MemoryLevel) ! IO

    enum MemoryLevel {
        Moderate,   // MEMORY_LEVEL_MODERATE — soft pressure; release caches
        Low,        // MEMORY_LEVEL_LOW — release non-critical resources
        Critical,   // MEMORY_LEVEL_CRITICAL — release everything possible
    }

    // Register a callback invoked when the system requests memory trimming.
    // Called on the thread that registered; must not block.
    fn on_memory_level(f: fn(MemoryLevel)) ! IO

    // Deregister the callback
    fn remove_memory_level_callback() ! IO

    // Current reported memory usage from native side (RSS, PSS)
    struct NativeMemoryInfo {
        rss_kb: u64,
        pss_kb: u64,
        private_dirty_kb: u64,
    }

    fn native_memory_info(): Result[NativeMemoryInfo, OhosError] ! IO
}
```

### 18.3 EventRunner / EventHandler

OpenHarmony's message loop system. `EventRunner` is a thread with a message queue; `EventHandler` posts tasks and events to it. The main UI thread has a pre-existing runner obtainable via `EventRunner::main()`.

```ferrum
mod event {
    struct EventRunner {
        // Create a new named background runner thread
        fn create(name: &str): Result[Self, OhosError] ! IO

        // Get the runner for the calling thread (None if not a runner thread)
        fn current(): Option[Self]

        // The main UI thread's runner
        fn main(): Self

        // Stop the runner loop after all pending events are processed
        fn stop(&self) ! IO
    }

    struct EventHandler {
        fn new(runner: &EventRunner): Result[Self, OhosError] ! IO

        // Post a task (closure) to run on the runner thread
        fn post_task(
            &self,
            name: &str,
            f: fn() ! IO,
            delay: Option[Duration],
        ): Result[(), OhosError] ! IO

        // Post a synchronous task — blocks the calling thread until done
        fn post_sync_task(
            &self,
            name: &str,
            f: fn() ! IO,
        ): Result[(), OhosError] ! IO

        // Remove all pending tasks with the given name
        fn remove_task(&self, name: &str) ! IO

        // Check if calling thread is this handler's runner thread
        fn is_current_runner(&self): bool ! IO
    }

    // Idle handler — called when the runner has no pending events
    fn set_idle_handler(runner: &EventRunner, f: fn() ! IO) ! IO
}
```

### 18.4 IPC — Same-Device Inter-Process Communication

OpenHarmony IPC uses `MessageParcel` and `IRemoteObject`, backed by the SoftBus local transport (not a kernel Binder driver — it is a userspace bus with kernel-assisted zero-copy for large payloads).

```ferrum
mod ipc {
    // MessageParcel — serialization container for IPC arguments
    struct MessageParcel {
        fn new(): Self

        // Write primitives
        fn write_bool(&mut self, v: bool): Result[(), OhosError]
        fn write_i32(&mut self, v: i32): Result[(), OhosError]
        fn write_i64(&mut self, v: i64): Result[(), OhosError]
        fn write_f32(&mut self, v: f32): Result[(), OhosError]
        fn write_f64(&mut self, v: f64): Result[(), OhosError]
        fn write_string(&mut self, s: &str): Result[(), OhosError]
        fn write_bytes(&mut self, b: &[u8]): Result[(), OhosError]
        fn write_remote_object(&mut self, obj: &RemoteObject): Result[(), OhosError]

        // Read primitives
        fn read_bool(&mut self): Result[bool, OhosError]
        fn read_i32(&mut self): Result[i32, OhosError]
        fn read_i64(&mut self): Result[i64, OhosError]
        fn read_f32(&mut self): Result[f32, OhosError]
        fn read_f64(&mut self): Result[f64, OhosError]
        fn read_string(&mut self): Result[String, OhosError]
        fn read_bytes(&mut self, len: usize): Result[Vec[u8], OhosError]
        fn read_remote_object(&mut self): Result[RemoteObject, OhosError]

        fn reclaim(self)
    }

    struct MessageOption {
        fn sync(): Self    // caller blocks until stub processes request
        fn async(): Self   // caller returns immediately
    }

    // RemoteObject — a reference to a service stub, possibly in another process
    struct RemoteObject {
        fn send_request(
            &self,
            code: u32,
            data: &MessageParcel,
            reply: &mut MessageParcel,
            option: MessageOption,
        ): Result[(), OhosError] ! IO

        fn is_dead(&self): bool
        fn add_death_recipient(&self, f: fn()) ! IO
        fn remove_death_recipient(&self, f: fn()) ! IO
    }

    // Implement a service stub — Ferrum struct callable from other processes
    trait RemoteStub {
        fn on_remote_request(
            &mut self,
            code: u32,
            data: &MessageParcel,
            reply: &mut MessageParcel,
            option: MessageOption,
        ): Result[(), OhosError] ! IO
    }

    // Expose a RemoteStub implementation to other processes
    fn register_stub[S: RemoteStub](stub: S): Result[RemoteObject, OhosError] ! IO

    // Look up a system service by name (returns a proxy RemoteObject)
    fn get_system_ability(ability_id: i32): Result[RemoteObject, OhosError] ! IO

    // Well-known system ability IDs (partial list)
    mod ability_id {
        const AUDIO_POLICY:     i32 = 3001
        const CAMERA:           i32 = 3008
        const DISPLAY:          i32 = 1003
        const INPUT:            i32 = 3101
        const NET_MANAGER:      i32 = 1151
        const WIFI:             i32 = 1120
        const BLUETOOTH:        i32 = 1130
        const LOCATION:         i32 = 2802
        const SENSOR:           i32 = 3601
        const DISTRIBUTED_DATA: i32 = 1301
    }
}
```

### 18.5 Distributed SoftBus

OpenHarmony's cross-device communication layer. Devices on the same LAN (or Bluetooth range) discover each other and establish typed sessions. This is architecturally distinct from Android SoftBus — it is a first-class distributed OS primitive.

```ferrum
mod softbus {
    // Discovery — find other OpenHarmony devices on the network
    struct DiscoveryInfo {
        device_id: String,
        device_name: String,
        device_type: runtime::DeviceType,
        capability: CapabilityMask,
    }

    bitflags CapabilityMask {
        const OSD        = 0x01   // one-shot display
        const DVKIT      = 0x02   // distributed virtual kit
        const CASTPLUS   = 0x04
        const AA         = 0x08   // ability awareness
        const SHARE      = 0x10
        const APPROACH   = 0x20
    }

    struct DiscoveryCallbacks {
        on_found: fn(info: &DiscoveryInfo),
        on_lost: fn(device_id: &str),
    }

    fn start_discovery(
        pkg_name: &str,
        capability: CapabilityMask,
        callbacks: DiscoveryCallbacks,
    ): Result[u32, OhosError] ! IO   // returns discovery ID

    fn stop_discovery(id: u32): Result[(), OhosError] ! IO

    // Publish this device's presence to others
    fn publish_service(
        pkg_name: &str,
        capability: CapabilityMask,
    ): Result[u32, OhosError] ! IO

    fn unpublish_service(id: u32): Result[(), OhosError] ! IO

    // Sessions — bidirectional byte-stream channels between devices
    struct SessionConfig {
        fn bytes(): Self        // raw byte channel
        fn message(): Self      // message-oriented channel
        fn stream(): Self       // low-latency streaming (audio/video)
    }

    struct Session {
        fn send_bytes(&self, data: &[u8]): Result[(), OhosError] ! IO
        fn send_message(&self, data: &[u8]): Result[(), OhosError] ! IO
        fn send_stream(&self, data: &StreamData): Result[(), OhosError> ! IO
        fn close(self) ! IO
        fn peer_device_id(&self): String
        fn is_server_side(&self): bool
    }

    struct SessionCallbacks {
        on_opened: fn(session: &Session) -> i32,   // return 0 to accept
        on_closed: fn(session: &Session),
        on_bytes_received: fn(session: &Session, data: &[u8]),
        on_message_received: fn(session: &Session, data: &[u8]),
        on_stream_received: fn(session: &Session, data: &StreamData),
    }

    fn create_session_server(
        pkg_name: &str,
        session_name: &str,
        callbacks: SessionCallbacks,
    ): Result[(), OhosError] ! IO

    fn remove_session_server(pkg_name: &str, session_name: &str): Result[(), OhosError] ! IO

    fn open_session(
        local_session_name: &str,
        peer_session_name: &str,
        peer_device_id: &str,
        group_id: &str,
        config: SessionConfig,
    ): Result[Session, OhosError] ! IO

    struct StreamData {
        buf: Vec[u8],
        ext: Vec[u8],   // extension data (timestamps, sequence numbers)
        stream_type: StreamType,
    }

    enum StreamType { Raw, File, CommonVideo, CommonAudio }
}
```

### 18.6 HDF — Hardware Driver Foundation

OpenHarmony's driver framework. Drivers live in a separate driver process and expose typed interfaces over HDF IPC. `sys.ohos.hdf` is the Ferrum wrapper for the HDI (Hardware Driver Interface) client side.

```ferrum
mod hdf {
    // Open a driver service by its HDI service name
    fn open_service(service_name: &str): Result[HdfService, OhosError] ! IO

    struct HdfService {
        fn dispatch(
            &self,
            cmd_id: u32,
            data: &HdfSbuf,
            reply: &mut HdfSbuf,
        ): Result[(), OhosError] ! IO

        fn is_alive(&self): bool

        fn on_death(self: &Self, f: fn()) ! IO
    }

    // HdfSbuf — serialization buffer for HDF IPC
    struct HdfSbuf {
        fn new(): Result[Self, OhosError>
        fn new_bound(capacity: usize): Result[Self, OhosError>

        fn write_u8(&mut self, v: u8): Result[(), OhosError]
        fn write_u16(&mut self, v: u16): Result[(), OhosError]
        fn write_u32(&mut self, v: u32): Result[(), OhosError]
        fn write_u64(&mut self, v: u64): Result[(), OhosError]
        fn write_i8(&mut self, v: i8): Result[(), OhosError]
        fn write_i32(&mut self, v: i32): Result[(), OhosError]
        fn write_i64(&mut self, v: i64): Result[(), OhosError]
        fn write_float(&mut self, v: f32): Result[(), OhosError]
        fn write_double(&mut self, v: f64): Result[(), OhosError]
        fn write_string(&mut self, s: &str): Result[(), OhosError]
        fn write_bytes(&mut self, b: &[u8]): Result[(), OhosError]
        fn write_file_descriptor(&mut self, fd: i32): Result[(), OhosError]

        fn read_u8(&mut self): Result[u8, OhosError]
        fn read_u32(&mut self): Result[u32, OhosError]
        fn read_u64(&mut self): Result[u64, OhosError]
        fn read_i32(&mut self): Result[i32, OhosError]
        fn read_i64(&mut self): Result[i64, OhosError]
        fn read_float(&mut self): Result[f32, OhosError]
        fn read_double(&mut self): Result[f64, OhosError]
        fn read_string(&mut self): Result[&str, OhosError]
        fn read_bytes(&mut self, len: usize): Result[&[u8], OhosError]
        fn read_file_descriptor(&mut self): Result[i32, OhosError]
    }

    // Register as a driver service (for driver process code, not app code)
    unsafe fn register_service(
        service_name: &str,
        dispatch: fn(cmd_id: u32, data: &HdfSbuf, reply: &mut HdfSbuf): i32,
    ): Result[(), OhosError> ! IO
}
```

### 18.7 Ability Context

OpenHarmony application lifecycle. Ferrum native code typically runs inside a `UIAbility` or `ExtensionAbility`; the context gives access to resource directories, preferences, and WantAgent.

```ferrum
mod ability {
    // Obtain the current ability's context (set up by the NAPI glue layer)
    fn current_context(): Result[AbilityContext, OhosError] ! IO

    struct AbilityContext {
        // Directories
        fn files_dir(&self): String
        fn cache_dir(&self): String
        fn databases_dir(&self): String
        fn temp_dir(&self): String
        fn bundle_code_dir(&self): String

        // App identity
        fn bundle_name(&self): String
        fn ability_name(&self): String

        // Terminate self
        fn terminate_ability(&self) ! IO
    }

    // Want — intent-like structure for ability launching
    struct Want {
        fn new(): Self
        fn bundle_name(self, name: &str): Self
        fn ability_name(self, name: &str): Self
        fn device_id(self, id: &str): Self     // empty = local device
        fn action(self, action: &str): Self
        fn entity(self, entity: &str): Self
        fn param_bool(self, key: &str, val: bool): Self
        fn param_i32(self, key: &str, val: i32): Self
        fn param_i64(self, key: &str, val: i64): Self
        fn param_string(self, key: &str, val: &str): Self
    }

    // WantAgent — deferred Want that can be triggered by system events
    // (used for notifications, alarms, background work)
    struct WantAgent {
        fn trigger(&self) ! IO
        fn cancel(&self) ! IO
    }

    fn create_want_agent(
        wants: &[Want],
        operation_type: WantAgentOperation,
        request_code: i32,
    ): Result[WantAgent, OhosError] ! IO

    enum WantAgentOperation {
        StartAbility,
        StartAbilities,
        StartService,
        SendCommonEvent,
    }
}
```

### 18.8 CommonEvent

OpenHarmony's system-wide publish/subscribe bus. Replaces Android's LocalBroadcastManager and system broadcast receivers.

```ferrum
mod common_event {
    // Publish a custom event
    fn publish(event: &str, data: CommonEventData): Result[(), OhosError] ! IO

    struct CommonEventData {
        fn new(want: ability::Want): Self
        fn code(self, code: i32): Self
        fn data(self, data: &str): Self
    }

    // Subscribe to events
    struct SubscribeInfo {
        fn new(events: &[&str]): Self
        fn priority(self, priority: i32): Self
        fn device_id(self, id: &str): Self
        fn permission(self, perm: &str): Self
    }

    struct Subscriber {
        fn new(info: SubscribeInfo): Result[Self, OhosError] ! IO
        fn subscribe(self, f: fn(event: &str, data: &CommonEventData)) ! IO
        fn unsubscribe(self) ! IO
    }

    // Well-known system event strings (partial list)
    mod events {
        const BOOT_COMPLETED:        &str = "usual.event.BOOT_COMPLETED"
        const BATTERY_LOW:           &str = "usual.event.BATTERY_LOW"
        const BATTERY_OKAY:          &str = "usual.event.BATTERY_OKAY"
        const SCREEN_ON:             &str = "usual.event.SCREEN_ON"
        const SCREEN_OFF:            &str = "usual.event.SCREEN_OFF"
        const CONNECTIVITY_CHANGE:   &str = "usual.event.CONNECTIVITY_CHANGE"
        const WIFI_STATE_CHANGED:    &str = "usual.event.WIFI_STATE_CHANGED"
        const PACKAGE_ADDED:         &str = "usual.event.PACKAGE_ADDED"
        const PACKAGE_REMOVED:       &str = "usual.event.PACKAGE_REMOVED"
        const USER_UNLOCKED:         &str = "usual.event.USER_UNLOCKED"
        const TIME_CHANGED:          &str = "usual.event.TIME_CHANGED"
        const TIMEZONE_CHANGED:      &str = "usual.event.TIMEZONE_CHANGED"
    }
}
```

### 18.9 HiLog — Structured Logging

OpenHarmony's logging system. Distinct from `std::log` — it writes to the hilog ring buffer readable via `hilog` CLI and the DevEco Studio log viewer.

```ferrum
mod hilog {
    // Log domain: a u16 identifier for your subsystem (0x0000–0xFFFF)
    // Register your domain in your HAP's module.json5 under "metadata"
    type Domain(u16)

    // Log tag: short ASCII string, max 32 bytes
    type Tag(&'static str)

    enum Level { Debug, Info, Warn, Error, Fatal }

    fn write(
        level: Level,
        domain: Domain,
        tag: Tag,
        msg: &str,
    ) ! IO

    // Convenience macros (expand to write() with file/line context)
    //   hilog_debug!(domain, tag, "message {}", arg)
    //   hilog_info!(domain, tag, "message {}", arg)
    //   hilog_warn!(domain, tag, "message {}", arg)
    //   hilog_error!(domain, tag, "message {}", arg)

    // Check if a level is loggable (avoids formatting cost when disabled)
    fn is_loggable(level: Level, domain: Domain, tag: Tag): bool
}
```

### 18.10 NAPI Is the Boundary

All Ferrum ↔ ArkTS interop crosses the NAPI boundary:

```ferrum
// Ferrum native module loaded by ArkTS via import
extern(napi) fn napi_register_module_v1(env: napi_env, exports: napi_value): napi_value ! Unsafe {
    // attach sys.ohos functions via NAPI method registration
    exports
}
```

`sys.ohos` is for OpenHarmony platform APIs that Ferrum native code calls directly — the runtime, the message loop, IPC, drivers, system events. It is not an alternative to `extern(napi)`; it is what you use on the *native side* after you have crossed into native via NAPI.

---

*End of sys, posix, windows, jvm, clr, ohos modules.*
*For platform-specific features (io_uring, kqueue, WASI, Zephyr, etc.), see [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md).*
*See [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
