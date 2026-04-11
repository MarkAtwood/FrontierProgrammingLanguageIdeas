# Ferrum Standard Library — sys, posix, windows

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)
**See also:** [Platform Abstraction Layer](ferrum-stdlib-platform.md) for platform traits, capability mapping, and detailed specs for Linux, BSD, WASI, Zephyr

---

## 12. sys — System Interfaces

### 12.1 Organization

`sys` is the portability layer. Platform-specific behavior lives in sub-namespaces.

```
std.sys               // cross-platform abstractions
std.sys.posix         // POSIX-specific (opt-in)
std.sys.linux         // Linux-specific (opt-in) — see platform spec
std.sys.bsd           // BSD-specific (opt-in) — see platform spec
std.sys.darwin        // macOS-specific (opt-in) — see platform spec
std.sys.windows       // Windows-specific (opt-in)
std.sys.fuchsia       // Fuchsia-specific (opt-in) — see platform spec
std.sys.ohos          // OpenHarmony-specific (opt-in) — see platform spec
std.sys.wasi          // WASI Preview 2 (opt-in) — see platform spec
std.sys.zephyr        // Zephyr RTOS (opt-in) — see platform spec
```

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

type Signal {
    const SIGHUP:  Self = Signal(1)
    const SIGINT:  Self = Signal(2)
    const SIGTERM: Self = Signal(15)
    const SIGUSR1: Self = Signal(10)
    const SIGUSR2: Self = Signal(12)
    // ... etc.
}

// mmap
type MmapFlags { ... }
type Protection { ... }

unsafe fn mmap(
    addr: Option[*mut u8],
    len: usize,
    prot: Protection,
    flags: MmapFlags,
    fd: Option[&Fd],
    offset: i64,
): Result[MmapGuard, PosixError] ! Unsafe + IO

type MmapGuard {
    // Drops the mapping on Drop
    fn as_slice(&self): &[u8]
    fn as_mut_slice(&mut self): &mut [u8]
    fn len(&self): usize
    fn flush(&self): Result[(), PosixError] ! IO
    fn flush_range(&self, offset: usize, len: usize): Result[(), PosixError] ! IO
}

// PosixError — errno mapped to a proper type
type PosixError {
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
type Poller {
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
    type Thread {
        fn join(self): Result[(), PosixError] ! IO  // consumes handle
        fn detach(self): Result[(), PosixError] ! IO
        fn id(&self): ThreadId
    }

    type ThreadId(u64)

    type ThreadAttrs {
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
    type Key[T] {
        fn new(destructor: Option[fn(*mut T)]): Result[Self, PosixError]
        unsafe fn get(&self): *mut T
        unsafe fn set(&self, value: *mut T): Result[(), PosixError]
    }

    // Affinity (Linux-specific, cfg-gated)
    @cfg(target_os = "linux")
    type CpuSet { ... }

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
    type Mutex {
        fn new(attrs: MutexAttrs): Result[Self, PosixError]
        fn lock(&self): Result[MutexGuard, PosixError] ! IO
        fn try_lock(&self): Result[MutexGuard, PosixError]
        fn unlock(guard: MutexGuard)  // consumed by guard drop
    }

    type MutexAttrs {
        fn default(): Self
        fn kind(self, kind: MutexKind): Self
        fn robust(self): Self           // PTHREAD_MUTEX_ROBUST
        fn pshared(self): Self          // PTHREAD_PROCESS_SHARED
        fn protocol(self, p: MutexProtocol): Self
    }

    enum MutexKind { Normal, Recursive, ErrorCheck }
    enum MutexProtocol { None, Inherit, Protect }

    // pthread condition variables
    type Condvar {
        fn new(): Result[Self, PosixError]
        fn wait(&self, guard: &mut MutexGuard): Result[(), PosixError] ! IO
        fn wait_timeout(&self, guard: &mut MutexGuard, timeout: Duration)
            : Result[bool, PosixError] ! IO  // returns false on timeout
        fn signal(&self): Result[(), PosixError]
        fn broadcast(&self): Result[(), PosixError]
    }

    // pthread barriers
    type Barrier {
        fn new(count: u32): Result[Self, PosixError]
        fn wait(&self): Result[BarrierWaitResult, PosixError] ! IO
    }

    enum BarrierWaitResult { Leader, Follower }

    // pthread read-write locks
    type RwLock {
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
    type Thread {
        fn join(self): Result[(), WinError] ! IO
        fn id(&self): ThreadId
        fn handle(&self): Handle  // raw HANDLE for WinAPI interop
    }

    type ThreadId(u32)

    type ThreadAttrs {
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
    type CriticalSection { ... }

    // SRWLOCK (slim reader-writer lock)
    type SrwLock { ... }

    // CONDITION_VARIABLE
    type ConditionVariable { ... }

    // Event objects
    type Event {
        fn new(manual_reset: bool, initial_state: bool): Result[Self, WinError] ! IO
        fn set(&self): Result[(), WinError] ! IO
        fn reset(&self): Result[(), WinError] ! IO
        fn wait(&self, timeout: Option[Duration]): Result[WaitResult, WinError] ! IO
    }

    // Semaphore
    type Semaphore {
        fn new(initial: u32, maximum: u32): Result[Self, WinError] ! IO
        fn release(&self, count: u32): Result[u32, WinError] ! IO
        fn wait(&self, timeout: Option[Duration]): Result[WaitResult, WinError] ! IO
    }

    enum WaitResult { Signaled, Timeout, Abandoned }
}
```

---

---

*End of sys, posix, windows modules.*
*For platform-specific features (io_uring, kqueue, WASI, Zephyr, etc.), see [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md).*
*See [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
