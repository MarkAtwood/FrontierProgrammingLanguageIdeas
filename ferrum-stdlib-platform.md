# Ferrum Standard Library — Platform Abstraction Layer

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)
**Status:** Meta-specification for OS/runtime bindings
**Scope:** POSIX, Linux, BSD, Windows, WASI, Zephyr, and future platforms

---

## 1. Design Philosophy

### 1.1 The Platform Problem

Every systems language faces the same tension:

1. **Portability** — Write once, run anywhere
2. **Power** — Access platform-specific features
3. **Safety** — Don't expose undefined behavior
4. **Futureproofing** — New platforms and APIs appear constantly

Previous approaches and their failures:

| Approach | Example | Failure Mode |
|----------|---------|--------------|
| Lowest common denominator | Java pre-NIO | Can't use platform strengths |
| Massive `#ifdef` trees | C/C++ | Unmaintainable, bugs hide in untested branches |
| Runtime abstraction layer | libuv, SDL | Performance overhead, impedance mismatch |
| Ignore the problem | Early Go | Forces platform-specific forks |

### 1.2 Ferrum's Approach

**Layered abstraction with explicit escape hatches:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 4: std.* — Portable, safe, opinionated                       │
│  (io, fs, net, process, time, env)                                  │
│  Uses platform traits internally. User code targets this layer.     │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 3: Platform Traits — Abstract interfaces                     │
│  (FileSystem, Network, Process, Memory, Time, Entropy)             │
│  Implemented by each platform. Testable via mock implementations.   │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: sys.{platform} — Platform-specific, safe wrappers         │
│  (sys.posix, sys.linux, sys.bsd, sys.windows, sys.fuchsia, sys.wasi)│
│  Safe Ferrum types over raw syscalls. Platform-specific features.   │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 1: sys.{platform}.ffi — Raw bindings, unsafe                 │
│  Direct syscall/API access. Minimal wrapper. For experts only.      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key principles:**

1. **Portable by default** — `std.*` works everywhere
2. **Platform power when needed** — `sys.linux.io_uring` for those who need it
3. **Unsafe is explicit** — Raw FFI is always `unsafe`
4. **Traits enable testing** — Mock any platform interface
5. **Capabilities flow through** — Platform permissions map to Ferrum capabilities
6. **Versions are explicit** — Platform APIs declare their version requirements

---

## 2. Platform Detection

### 2.1 Compile-Time Detection

```ferrum
// Built-in cfg attributes
@cfg(target_os = "linux")
@cfg(target_os = "macos")
@cfg(target_os = "windows")
@cfg(target_os = "freebsd")
@cfg(target_os = "openbsd")
@cfg(target_os = "netbsd")
@cfg(target_os = "fuchsia")
@cfg(target_os = "ohos")        // OpenHarmony
@cfg(target_os = "wasi")
@cfg(target_os = "zephyr")
@cfg(target_os = "none")        // bare metal

@cfg(target_family = "unix")    // POSIX-like
@cfg(target_family = "windows")
@cfg(target_family = "wasm")
@cfg(target_family = "embedded")

@cfg(target_arch = "x86_64")
@cfg(target_arch = "aarch64")
@cfg(target_arch = "riscv64")
@cfg(target_arch = "wasm32")
@cfg(target_arch = "wasm64")

@cfg(target_env = "gnu")        // glibc
@cfg(target_env = "musl")
@cfg(target_env = "msvc")

@cfg(target_pointer_width = "32")
@cfg(target_pointer_width = "64")

@cfg(target_endian = "little")
@cfg(target_endian = "big")

// Platform version requirements
@cfg(target_os_version >= "linux:5.1")      // io_uring
@cfg(target_os_version >= "macos:10.15")    // Catalyst
@cfg(target_os_version >= "windows:10.0.17763")  // Win10 1809
@cfg(target_os_version >= "wasi:0.2")       // Preview 2
```

### 2.2 Compile-Time Platform Constants

```ferrum
// In core.env
const TARGET_OS: &str = ...           // "linux", "macos", "windows", etc.
const TARGET_FAMILY: &str = ...       // "unix", "windows", "wasm", "embedded"
const TARGET_ARCH: &str = ...         // "x86_64", "aarch64", etc.
const TARGET_POINTER_WIDTH: u32 = ... // 32 or 64
const TARGET_ENDIAN: Endian = ...     // Endian.Little or Endian.Big

// Platform-specific version (if applicable)
const TARGET_OS_VERSION: Option[OsVersion] = ...

type OsVersion {
    major: u32,
    minor: u32,
    patch: u32,
}
```

### 2.3 Runtime Feature Detection

For features that vary at runtime (CPU features, kernel config, etc.):

```ferrum
// In sys.features
fn has_feature(feature: Feature) : bool

enum Feature {
    // CPU features
    Sse4_2,
    Avx2,
    Avx512,
    Neon,
    Sve,
    Sve2,

    // Kernel features (Linux)
    IoUring,
    Landlock,
    Seccomp,
    Bpf,
    Userfaultfd,

    // Windows features
    Wsl,
    HyperV,

    // WASI features
    WasiThreads,
    WasiSockets,
    WasiFilesystem,
}

// Cache results — feature detection is expensive
static FEATURES: OnceLock[FeatureSet] = OnceLock.new()

fn features() : &FeatureSet {
    FEATURES.get_or_init(|| detect_features())
}
```

---

## 3. Platform Traits

Abstract interfaces that all platforms implement. These enable:
- Portable `std.*` code
- Mock implementations for testing
- Future platforms without API changes

### 3.1 Core Platform Trait

```ferrum
/// The root platform abstraction.
/// Every supported platform provides an implementation.
trait Platform: Send + Sync + 'static {
    type FileSystem: FileSystemPlatform
    type Network: NetworkPlatform
    type Process: ProcessPlatform
    type Memory: MemoryPlatform
    type Time: TimePlatform
    type Entropy: EntropyPlatform
    type Threading: ThreadingPlatform

    /// Platform identifier for logging/debugging.
    fn name(&self) : &str

    /// Platform version.
    fn version(&self) : OsVersion

    /// Available capabilities on this platform.
    fn capabilities(&self) : CapabilitySet
}

/// Get the current platform implementation.
fn current_platform() : &'static dyn Platform
```

### 3.2 FileSystem Platform Trait

```ferrum
trait FileSystemPlatform: Send + Sync {
    type File: FileHandle
    type Dir: DirHandle
    type Metadata: FileMetadata

    // Basic operations
    fn open(&self, path: &Path, opts: OpenOptions) : Result[Self.File, IoError] ! IO
    fn create(&self, path: &Path) : Result[Self.File, IoError] ! IO
    fn remove(&self, path: &Path) : Result[(), IoError] ! IO
    fn rename(&self, from: &Path, to: &Path) : Result[(), IoError] ! IO
    fn copy(&self, from: &Path, to: &Path) : Result[u64, IoError] ! IO
    fn metadata(&self, path: &Path) : Result[Self.Metadata, IoError] ! IO
    fn symlink_metadata(&self, path: &Path) : Result[Self.Metadata, IoError] ! IO

    // Directory operations
    fn create_dir(&self, path: &Path) : Result[(), IoError] ! IO
    fn create_dir_all(&self, path: &Path) : Result[(), IoError] ! IO
    fn remove_dir(&self, path: &Path) : Result[(), IoError] ! IO
    fn remove_dir_all(&self, path: &Path) : Result[(), IoError] ! IO
    fn read_dir(&self, path: &Path) : Result[Self.Dir, IoError] ! IO

    // Links
    fn hard_link(&self, src: &Path, dst: &Path) : Result[(), IoError] ! IO
    fn symlink(&self, src: &Path, dst: &Path) : Result[(), IoError] ! IO
    fn read_link(&self, path: &Path) : Result[PathBuf, IoError] ! IO

    // Permissions (platform-specific details hidden)
    fn set_permissions(&self, path: &Path, perm: Permissions) : Result[(), IoError] ! IO

    // Platform capabilities
    fn supports_symlinks(&self) : bool
    fn supports_hard_links(&self) : bool
    fn supports_permissions(&self) : bool
    fn supports_sparse_files(&self) : bool
    fn max_path_length(&self) : usize
    fn path_separator(&self) : char
}

trait FileHandle: Read + Write + Seek + Send + Sync {
    fn sync_all(&self) : Result[(), IoError] ! IO
    fn sync_data(&self) : Result[(), IoError] ! IO
    fn set_len(&self, size: u64) : Result[(), IoError] ! IO
    fn metadata(&self) : Result[FileMetadata, IoError] ! IO

    // Advisory locking (best-effort on some platforms)
    fn try_lock_shared(&self) : Result[bool, IoError] ! IO
    fn try_lock_exclusive(&self) : Result[bool, IoError] ! IO
    fn unlock(&self) : Result[(), IoError] ! IO
}

trait FileMetadata {
    fn file_type(&self) : FileType
    fn len(&self) : u64
    fn permissions(&self) : Permissions
    fn modified(&self) : Result[Timestamp, IoError]
    fn accessed(&self) : Result[Timestamp, IoError]
    fn created(&self) : Result[Timestamp, IoError]   // Not available on all platforms
    fn is_file(&self) : bool
    fn is_dir(&self) : bool
    fn is_symlink(&self) : bool
}
```

### 3.3 Network Platform Trait

```ferrum
trait NetworkPlatform: Send + Sync {
    type TcpListener: TcpListenerHandle
    type TcpStream: TcpStreamHandle
    type UdpSocket: UdpSocketHandle

    // TCP
    fn tcp_bind(&self, addr: SocketAddr) : Result[Self.TcpListener, IoError] ! Net
    fn tcp_connect(&self, addr: SocketAddr) : Result[Self.TcpStream, IoError] ! Net

    // UDP
    fn udp_bind(&self, addr: SocketAddr) : Result[Self.UdpSocket, IoError] ! Net

    // DNS
    fn resolve(&self, host: &str, service: &str) : Result[Vec[SocketAddr], IoError] ! Net
    fn resolve_async(&self, host: &str, service: &str) : Future[Result[Vec[SocketAddr], IoError]] ! Net + Async

    // Platform capabilities
    fn supports_ipv6(&self) : bool
    fn supports_unix_sockets(&self) : bool
    fn supports_multicast(&self) : bool
    fn max_connections(&self) : Option[usize]
}

trait TcpListenerHandle: Send + Sync {
    type Stream: TcpStreamHandle
    fn accept(&self) : Result[(Self.Stream, SocketAddr), IoError] ! Net
    fn local_addr(&self) : Result[SocketAddr, IoError]
    fn set_nonblocking(&self, nonblocking: bool) : Result[(), IoError]
}

trait TcpStreamHandle: Read + Write + Send + Sync {
    fn peer_addr(&self) : Result[SocketAddr, IoError]
    fn local_addr(&self) : Result[SocketAddr, IoError]
    fn shutdown(&self, how: Shutdown) : Result[(), IoError] ! Net
    fn set_nodelay(&self, nodelay: bool) : Result[(), IoError]
    fn set_nonblocking(&self, nonblocking: bool) : Result[(), IoError]
    fn set_read_timeout(&self, dur: Option[Duration]) : Result[(), IoError]
    fn set_write_timeout(&self, dur: Option[Duration]) : Result[(), IoError]
}
```

### 3.4 Process Platform Trait

```ferrum
trait ProcessPlatform: Send + Sync {
    type Process: ProcessHandle
    type Child: ChildHandle

    // Current process
    fn current(&self) : Self.Process
    fn exit(&self, code: i32) : never ! IO
    fn abort(&self) : never

    // Spawn
    fn spawn(&self, cmd: &Command) : Result[Self.Child, IoError] ! IO + Process

    // Environment
    fn env_var(&self, key: &str) : Result[String, EnvError]
    fn env_vars(&self) : impl Iterator[Item = (String, String)]
    fn set_env_var(&self, key: &str, value: &str) ! IO
    fn remove_env_var(&self, key: &str) ! IO
    fn current_dir(&self) : Result[PathBuf, IoError] ! IO
    fn set_current_dir(&self, path: &Path) : Result[(), IoError] ! IO

    // Platform capabilities
    fn supports_signals(&self) : bool
    fn supports_job_control(&self) : bool
    fn max_args_length(&self) : usize
}

trait ProcessHandle {
    fn id(&self) : u32
    fn kill(&self) : Result[(), IoError] ! IO + Process
}

trait ChildHandle {
    fn id(&self) : u32
    fn wait(&self) : Result[ExitStatus, IoError] ! IO
    fn try_wait(&self) : Result[Option[ExitStatus], IoError] ! IO
    fn kill(&self) : Result[(), IoError] ! IO + Process
    fn stdin(&mut self) : Option[&mut dyn Write]
    fn stdout(&mut self) : Option[&mut dyn Read]
    fn stderr(&mut self) : Option[&mut dyn Read]
}
```

### 3.5 Memory Platform Trait

```ferrum
trait MemoryPlatform: Send + Sync {
    /// Allocate memory pages from the OS.
    fn alloc_pages(&self, size: usize) : Result[*mut u8, MemoryError] ! Unsafe

    /// Free memory pages.
    unsafe fn free_pages(&self, ptr: *mut u8, size: usize) ! Unsafe

    /// Change page protection.
    unsafe fn protect(&self, ptr: *mut u8, size: usize, prot: Protection) : Result[(), MemoryError] ! Unsafe

    /// Map a file into memory.
    fn map_file(&self, file: &impl FileHandle, offset: u64, size: usize, prot: Protection)
        : Result[MappedRegion, MemoryError] ! IO + Unsafe

    /// Page size.
    fn page_size(&self) : usize

    /// Huge page sizes (if supported).
    fn huge_page_sizes(&self) : &[usize]

    /// Memory info.
    fn physical_memory(&self) : u64
    fn available_memory(&self) : u64 ! IO
}

bitflags type Protection: u8 {
    const READ    = 0b001
    const WRITE   = 0b010
    const EXECUTE = 0b100
}

type MappedRegion {
    ptr: *mut u8,
    len: usize,
}

impl Drop for MappedRegion {
    fn drop(&mut self) {
        // Unmap via platform
    }
}
```

### 3.6 Time Platform Trait

```ferrum
trait TimePlatform: Send + Sync {
    /// Monotonic clock — never goes backwards.
    fn monotonic_now(&self) : Instant

    /// Wall clock — can jump (NTP, manual adjustment).
    fn timestamp_now(&self) : Timestamp

    /// The Julian Day Number at this platform's epoch (Timestamp.nanos = 0).
    /// This is the only thing needed to convert timestamps to/from any calendar.
    /// Example: Unix epoch = JDN 2440587.5, Windows epoch = JDN 2305813.5
    fn epoch_jdn(&self) : f64

    /// Sleep the current thread.
    fn sleep(&self, dur: Duration) ! IO

    /// High-resolution timer tick (for benchmarking).
    fn high_res_tick(&self) : u64

    /// Ticks per second for high_res_tick.
    fn high_res_frequency(&self) : u64

    /// Platform capabilities.
    fn monotonic_resolution(&self) : Duration
    fn system_resolution(&self) : Duration
}
```

### 3.7 Entropy Platform Trait

```ferrum
trait EntropyPlatform: Send + Sync {
    /// Fill buffer with cryptographically secure random bytes.
    fn fill_random(&self, buf: &mut [u8]) : Result[(), EntropyError] ! IO

    /// Get a random u64.
    fn random_u64(&self) : Result[u64, EntropyError] ! IO

    /// Entropy source quality.
    fn entropy_quality(&self) : EntropyQuality
}

enum EntropyQuality {
    /// Hardware RNG (RDRAND, etc.)
    Hardware,
    /// OS entropy pool (/dev/urandom, getrandom, BCryptGenRandom)
    OsPool,
    /// Deterministic (for testing only)
    Deterministic,
}
```

### 3.8 Threading Platform Trait

```ferrum
trait ThreadingPlatform: Send + Sync {
    type Thread: ThreadHandle
    type Mutex: MutexHandle
    type Condvar: CondvarHandle
    type RwLock: RwLockHandle

    // Threading
    fn spawn(&self, f: impl FnOnce() + Send + 'static) : Result[Self.Thread, ThreadError] ! IO
    fn current(&self) : Self.Thread
    fn yield_now(&self)
    fn park(&self)
    fn unpark(&self, thread: &Self.Thread)

    // Synchronization primitives
    fn new_mutex(&self) : Self.Mutex
    fn new_condvar(&self) : Self.Condvar
    fn new_rwlock(&self) : Self.RwLock

    // Platform capabilities
    fn available_parallelism(&self) : usize
    fn supports_thread_local(&self) : bool
    fn stack_size_range(&self) : (usize, usize)
}
```

---

## 4. Platform Implementations

### 4.1 Platform Module Structure

```
sys/
├── platform.fe          # Platform trait definitions
├── posix/
│   ├── mod.fe           # POSIX-compliant (all Unixes)
│   ├── fs.fe
│   ├── net.fe
│   ├── process.fe
│   ├── signal.fe
│   ├── mman.fe          # mmap
│   ├── pthread.fe
│   └── ffi/             # Raw bindings
│       ├── libc.fe
│       └── syscall.fe
├── linux/
│   ├── mod.fe           # Linux-specific extensions
│   ├── io_uring.fe
│   ├── epoll.fe
│   ├── eventfd.fe
│   ├── signalfd.fe
│   ├── timerfd.fe
│   ├── memfd.fe
│   ├── pidfd.fe
│   ├── seccomp.fe
│   ├── landlock.fe
│   ├── bpf.fe
│   ├── netlink.fe
│   ├── cgroups.fe
│   ├── namespaces.fe
│   └── ffi/
│       └── syscall.fe
├── bsd/
│   ├── mod.fe           # BSD-family common
│   ├── kqueue.fe
│   ├── pledge.fe        # OpenBSD
│   ├── capsicum.fe      # FreeBSD
│   ├── ktrace.fe
│   └── ffi/
├── darwin/
│   ├── mod.fe           # macOS/iOS specific
│   ├── dispatch.fe      # Grand Central Dispatch
│   ├── xpc.fe
│   ├── sandbox.fe
│   ├── keychain.fe
│   └── ffi/
├── windows/
│   ├── mod.fe           # Windows API
│   ├── async_io.fe      # IOCP
│   ├── registry.fe
│   ├── service.fe
│   ├── named_pipe.fe
│   ├── console.fe
│   ├── security.fe
│   ├── winsock.fe
│   └── ffi/
│       ├── kernel32.fe
│       ├── ntdll.fe
│       ├── ws2_32.fe
│       └── advapi32.fe
├── fuchsia/
│   ├── mod.fe           # Fuchsia (Zircon kernel)
│   ├── zx.fe            # Zircon syscalls
│   ├── handle.fe        # Kernel object handles
│   ├── channel.fe       # IPC channels
│   ├── vmo.fe           # Virtual Memory Objects
│   ├── port.fe          # Async I/O ports
│   ├── fidl.fe          # FIDL protocol bindings
│   ├── component.fe     # Component framework
│   ├── namespace.fe     # Capability namespaces
│   └── ffi/
│       └── zircon.fe
├── ohos/
│   ├── mod.fe           # OpenHarmony
│   ├── ability.fe       # Ability framework
│   ├── softbus.fe       # Distributed soft bus
│   ├── hdf.fe           # Hardware Driver Foundation
│   ├── hilog.fe         # HiLog logging
│   ├── hitrace.fe       # HiTrace distributed tracing
│   ├── ipc.fe           # IPC/RPC framework
│   └── ffi/
│       └── napi.fe      # N-API bindings
├── wasi/
│   ├── mod.fe           # WASI Preview 2+
│   ├── filesystem.fe
│   ├── sockets.fe
│   ├── clocks.fe
│   ├── random.fe
│   ├── poll.fe
│   ├── cli.fe
│   ├── http.fe
│   └── ffi/
│       └── bindings.fe
├── zephyr/
│   ├── mod.fe           # Zephyr RTOS
│   ├── kernel.fe
│   ├── device.fe
│   ├── gpio.fe
│   ├── i2c.fe
│   ├── spi.fe
│   ├── uart.fe
│   ├── bluetooth.fe
│   ├── net.fe
│   └── ffi/
└── bare/
    ├── mod.fe           # No OS (bare metal)
    ├── boot.fe
    ├── interrupts.fe
    └── mmio.fe
```

### 4.2 POSIX Implementation

The base for all Unix-like systems. Linux, BSD, and Darwin extend this.

```ferrum
// sys/posix/mod.fe

/// POSIX platform — base implementation for Unix-like systems.
/// Conforms to POSIX.1-2017 (IEEE Std 1003.1-2017).
pub type PosixPlatform {
    version: PosixVersion,
}

pub enum PosixVersion {
    Posix2001,  // POSIX.1-2001
    Posix2008,  // POSIX.1-2008
    Posix2017,  // POSIX.1-2017 (current)
}

impl Platform for PosixPlatform {
    type FileSystem = PosixFileSystem
    type Network = PosixNetwork
    type Process = PosixProcess
    type Memory = PosixMemory
    type Time = PosixTime
    type Entropy = PosixEntropy
    type Threading = PosixThreading

    fn name(&self) : &str { "posix" }
    fn version(&self) : OsVersion { /* ... */ }
    fn capabilities(&self) : CapabilitySet { /* ... */ }
}

// POSIX-specific types and functions
pub mod signal {
    pub enum Signal {
        Hup = 1,
        Int = 2,
        Quit = 3,
        Ill = 4,
        Trap = 5,
        Abrt = 6,
        // ... all POSIX signals
        User1 = 10,
        User2 = 12,
        // ...
    }

    pub fn kill(pid: Pid, sig: Signal) : Result[(), PosixError] ! IO + Unsafe
    pub fn raise(sig: Signal) : Result[(), PosixError] ! IO + Unsafe

    // Safe signal handling via channels (see ferrum-stdlib-sys.md)
    pub fn signal_stream(signals: &[Signal]) : Receiver[Signal] ! IO
}

pub mod mman {
    pub fn mmap(
        addr: Option[*mut u8],
        len: usize,
        prot: Protection,
        flags: MapFlags,
        fd: Option[RawFd>,
        offset: i64,
    ) : Result[*mut u8, PosixError] ! IO + Unsafe

    pub unsafe fn munmap(addr: *mut u8, len: usize) : Result[(), PosixError] ! Unsafe

    pub unsafe fn mprotect(addr: *mut u8, len: usize, prot: Protection)
        : Result[(), PosixError] ! Unsafe

    pub fn mlock(addr: *const u8, len: usize) : Result[(), PosixError] ! IO + Unsafe
    pub fn munlock(addr: *const u8, len: usize) : Result[(), PosixError] ! Unsafe

    bitflags pub type MapFlags: i32 {
        const SHARED    = 0x01
        const PRIVATE   = 0x02
        const ANONYMOUS = 0x20
        const FIXED     = 0x10
        const NORESERVE = 0x4000
        const POPULATE  = 0x8000
        const HUGETLB   = 0x40000
    }
}

pub mod pthread {
    pub type PthreadAttr { /* ... */ }
    pub type PthreadMutex { /* ... */ }
    pub type PthreadCond { /* ... */ }
    pub type PthreadRwLock { /* ... */ }

    // Low-level pthread wrappers
    // Higher-level abstractions in std.sync
}
```

### 4.3 Linux Implementation

Extends POSIX with Linux-specific features.

```ferrum
// sys/linux/mod.fe

/// Linux platform — extends POSIX with Linux-specific APIs.
/// Minimum supported kernel: 5.4 LTS
/// Recommended kernel: 6.1+ (for io_uring, landlock)
pub type LinuxPlatform {
    kernel_version: KernelVersion,
    posix: PosixPlatform,
}

pub type KernelVersion {
    major: u32,
    minor: u32,
    patch: u32,
}

impl Platform for LinuxPlatform {
    // Delegates to PosixPlatform for standard operations
    // Overrides where Linux has better implementations
}

/// io_uring — high-performance async I/O (kernel 5.1+)
pub mod io_uring {
    /// Minimum kernel version for io_uring.
    pub const MIN_KERNEL: KernelVersion = KernelVersion { major: 5, minor: 1, patch: 0 }

    /// io_uring instance.
    pub type IoUring {
        ring_fd: RawFd,
        sq: SubmissionQueue,
        cq: CompletionQueue,
    }

    impl IoUring {
        pub fn new(entries: u32) : Result[Self, IoUringError] ! IO
            requires entries.is_power_of_two()
            requires entries >= 1 and entries <= 32768

        pub fn submit(&self) : Result[u32, IoUringError] ! IO
        pub fn submit_and_wait(&self, wait_nr: u32) : Result[u32, IoUringError] ! IO

        // Operation builders
        pub fn prep_read(&mut self, fd: RawFd, buf: &mut [u8], offset: u64) : &mut SqEntry
        pub fn prep_write(&mut self, fd: RawFd, buf: &[u8], offset: u64) : &mut SqEntry
        pub fn prep_accept(&mut self, fd: RawFd) : &mut SqEntry
        pub fn prep_connect(&mut self, fd: RawFd, addr: &SocketAddr) : &mut SqEntry
        pub fn prep_send(&mut self, fd: RawFd, buf: &[u8]) : &mut SqEntry
        pub fn prep_recv(&mut self, fd: RawFd, buf: &mut [u8]) : &mut SqEntry
        pub fn prep_fsync(&mut self, fd: RawFd) : &mut SqEntry
        pub fn prep_timeout(&mut self, ts: &Timespec) : &mut SqEntry
        pub fn prep_cancel(&mut self, user_data: u64) : &mut SqEntry
        // ... many more operations
    }

    // Completion queue iteration
    impl IoUring {
        pub fn completions(&self) : impl Iterator[Item = CqEntry]
    }
}

/// epoll — event notification (fallback for older kernels)
pub mod epoll {
    pub type Epoll { fd: RawFd }

    impl Epoll {
        pub fn new() : Result[Self, IoError] ! IO
        pub fn add(&self, fd: RawFd, events: EpollEvents, data: u64) : Result[(), IoError] ! IO
        pub fn modify(&self, fd: RawFd, events: EpollEvents, data: u64) : Result[(), IoError] ! IO
        pub fn remove(&self, fd: RawFd) : Result[(), IoError] ! IO
        pub fn wait(&self, events: &mut [EpollEvent], timeout_ms: i32) : Result[usize, IoError] ! IO
    }

    bitflags pub type EpollEvents: u32 {
        const IN        = 0x001
        const OUT       = 0x004
        const ERR       = 0x008
        const HUP       = 0x010
        const RDHUP     = 0x2000
        const PRI       = 0x002
        const ET        = 1 << 31   // edge-triggered
        const ONESHOT   = 1 << 30
    }
}

/// Landlock — application sandboxing (kernel 5.13+)
pub mod landlock {
    pub const MIN_KERNEL: KernelVersion = KernelVersion { major: 5, minor: 13, patch: 0 }

    pub type Ruleset { /* ... */ }

    impl Ruleset {
        pub fn new() : Result[Self, LandlockError] ! IO
        pub fn add_rule(&mut self, rule: Rule) : Result[(), LandlockError]
        pub fn restrict_self(self) : Result[(), LandlockError] ! IO + Unsafe
    }

    pub enum Rule {
        PathBeneath { fd: RawFd, access: FsAccess },
        // Future: NetPort, NetConnect (kernel 6.x)
    }

    bitflags pub type FsAccess: u64 {
        const EXECUTE    = 1 << 0
        const WRITE_FILE = 1 << 1
        const READ_FILE  = 1 << 2
        const READ_DIR   = 1 << 3
        const REMOVE_DIR = 1 << 4
        const REMOVE_FILE = 1 << 5
        const MAKE_CHAR  = 1 << 6
        const MAKE_DIR   = 1 << 7
        const MAKE_REG   = 1 << 8
        const MAKE_SOCK  = 1 << 9
        const MAKE_FIFO  = 1 << 10
        const MAKE_BLOCK = 1 << 11
        const MAKE_SYM   = 1 << 12
        const REFER      = 1 << 13
        const TRUNCATE   = 1 << 14
    }
}

/// seccomp — syscall filtering
pub mod seccomp {
    pub type Filter { /* BPF program */ }

    impl Filter {
        pub fn new() : Self
        pub fn allow(&mut self, syscall: Syscall) : &mut Self
        pub fn deny(&mut self, syscall: Syscall) : &mut Self
        pub fn trap(&mut self, syscall: Syscall) : &mut Self
        pub fn apply(self) : Result[(), SeccompError] ! IO + Unsafe
    }
}

/// pidfd — process file descriptors (kernel 5.3+)
pub mod pidfd {
    pub type PidFd { fd: RawFd }

    impl PidFd {
        pub fn open(pid: Pid) : Result[Self, IoError] ! IO
        pub fn wait(&self) : Result[ExitStatus, IoError] ! IO
        pub fn send_signal(&self, sig: Signal) : Result[(), IoError] ! IO + Unsafe
    }

    impl Pollable for PidFd {
        // Can be polled for process exit
    }
}

/// eventfd, signalfd, timerfd — specialized file descriptors
pub mod eventfd {
    pub type EventFd { fd: RawFd }
    impl EventFd {
        pub fn new(init: u64, flags: EventFdFlags) : Result[Self, IoError] ! IO
        pub fn read(&self) : Result[u64, IoError] ! IO
        pub fn write(&self, val: u64) : Result[(), IoError] ! IO
    }
}

pub mod signalfd {
    pub type SignalFd { fd: RawFd }
    impl SignalFd {
        pub fn new(mask: &SignalSet) : Result[Self, IoError] ! IO
        pub fn read(&self) : Result[SignalInfo, IoError] ! IO
    }
}

pub mod timerfd {
    pub type TimerFd { fd: RawFd }
    impl TimerFd {
        pub fn new(clockid: ClockId) : Result[Self, IoError] ! IO
        pub fn set(&self, new_value: &TimerSpec) : Result[TimerSpec, IoError] ! IO
        pub fn read(&self) : Result[u64, IoError] ! IO  // expirations since last read
    }
}
```

### 4.4 BSD Implementation

Common BSD features plus platform-specific extensions.

```ferrum
// sys/bsd/mod.fe

/// BSD platform family — FreeBSD, OpenBSD, NetBSD, DragonFlyBSD.
/// Extends POSIX with BSD-specific features.
pub type BsdPlatform {
    variant: BsdVariant,
    posix: PosixPlatform,
}

pub enum BsdVariant {
    FreeBSD { version: BsdVersion },
    OpenBSD { version: BsdVersion },
    NetBSD { version: BsdVersion },
    DragonFlyBSD { version: BsdVersion },
}

/// kqueue — event notification (all BSDs)
pub mod kqueue {
    pub type Kqueue { fd: RawFd }

    impl Kqueue {
        pub fn new() : Result[Self, IoError] ! IO

        pub fn register(&self, events: &[KEvent]) : Result[(), IoError] ! IO
        pub fn poll(&self, events: &mut [KEvent], timeout: Option[Duration])
            : Result[usize, IoError] ! IO
    }

    pub type KEvent {
        ident: usize,
        filter: Filter,
        flags: Flags,
        fflags: u32,
        data: i64,
        udata: *mut (),
    }

    pub enum Filter {
        Read,
        Write,
        Aio,
        Vnode,
        Proc,
        Signal,
        Timer,
        User,
        // FreeBSD additions
        @cfg(target_os = "freebsd")
        Empty,
        @cfg(target_os = "freebsd")
        Procdesc,
    }

    bitflags pub type Flags: u16 {
        const ADD      = 0x0001
        const DELETE   = 0x0002
        const ENABLE   = 0x0004
        const DISABLE  = 0x0008
        const ONESHOT  = 0x0010
        const CLEAR    = 0x0020
        const RECEIPT  = 0x0040
        const DISPATCH = 0x0080
        const EOF      = 0x8000
        const ERROR    = 0x4000
    }
}

/// FreeBSD Capsicum — capability-based security
@cfg(target_os = "freebsd")
pub mod capsicum {
    pub fn cap_enter() : Result[(), CapError] ! IO + Unsafe

    pub fn cap_rights_limit(fd: RawFd, rights: CapRights) : Result[(), CapError] ! IO

    bitflags pub type CapRights: u64 {
        const READ   = 1 << 0
        const WRITE  = 1 << 1
        const SEEK   = 1 << 2
        const MMAP   = 1 << 3
        const CREATE = 1 << 4
        const FEXEC  = 1 << 5
        const FSYNC  = 1 << 6
        const FTRUNC = 1 << 7
        const LOOKUP = 1 << 8
        // ... many more
    }
}

/// OpenBSD pledge — promise-based sandboxing
@cfg(target_os = "openbsd")
pub mod pledge {
    pub fn pledge(promises: &str, execpromises: Option[&str])
        : Result[(), PledgeError] ! IO + Unsafe

    // Common promise strings
    pub const STDIO: &str = "stdio"
    pub const RPATH: &str = "rpath"
    pub const WPATH: &str = "wpath"
    pub const CPATH: &str = "cpath"
    pub const DPATH: &str = "dpath"
    pub const TMPPATH: &str = "tmppath"
    pub const INET: &str = "inet"
    pub const UNIX: &str = "unix"
    pub const DNS: &str = "dns"
    pub const PROC: &str = "proc"
    pub const EXEC: &str = "exec"
    pub const PROT_EXEC: &str = "prot_exec"
    pub const SETTIME: &str = "settime"
    pub const PS: &str = "ps"
    pub const VMINFO: &str = "vminfo"
    pub const ID: &str = "id"
    // ...
}

/// OpenBSD unveil — filesystem access control
@cfg(target_os = "openbsd")
pub mod unveil {
    pub fn unveil(path: &Path, permissions: &str) : Result[(), UnveilError] ! IO + Unsafe
    pub fn unveil_lock() : Result[(), UnveilError] ! IO + Unsafe

    // Permission strings: "r" read, "w" write, "x" execute, "c" create
}
```

### 4.5 Windows Implementation

```ferrum
// sys/windows/mod.fe

/// Windows platform — Win32/NT API bindings.
/// Minimum supported: Windows 10 1809 (build 17763)
/// Recommended: Windows 11 / Server 2022
pub type WindowsPlatform {
    version: WindowsVersion,
}

pub type WindowsVersion {
    major: u32,
    minor: u32,
    build: u32,
}

impl Platform for WindowsPlatform {
    type FileSystem = WindowsFileSystem
    type Network = WindowsNetwork
    type Process = WindowsProcess
    type Memory = WindowsMemory
    type Time = WindowsTime
    type Entropy = WindowsEntropy
    type Threading = WindowsThreading

    fn name(&self) : &str { "windows" }
    fn version(&self) : OsVersion { /* ... */ }
    fn capabilities(&self) : CapabilitySet { /* ... */ }
}

/// Windows IOCP — I/O Completion Ports (async I/O)
pub mod iocp {
    pub type CompletionPort { handle: Handle }

    impl CompletionPort {
        pub fn new(concurrency: u32) : Result[Self, WinError] ! IO
        pub fn associate(&self, handle: Handle, key: usize) : Result[(), WinError] ! IO
        pub fn post(&self, bytes: u32, key: usize, overlapped: *mut Overlapped)
            : Result[(), WinError] ! IO
        pub fn get(&self, timeout_ms: u32) : Result[CompletionStatus, WinError] ! IO
        pub fn get_many(&self, entries: &mut [CompletionStatus], timeout_ms: u32)
            : Result[usize, WinError] ! IO
    }

    pub type CompletionStatus {
        bytes_transferred: u32,
        completion_key: usize,
        overlapped: *mut Overlapped,
    }

    pub type Overlapped {
        // Matches Windows OVERLAPPED structure
        internal: usize,
        internal_high: usize,
        offset: u64,
        event: Handle,
    }
}

/// Windows handles
pub mod handle {
    pub type Handle(isize)   // HANDLE

    impl Handle {
        pub const INVALID: Self = Handle(-1)
        pub fn is_valid(&self) : bool { self.0 != -1 && self.0 != 0 }
        pub fn close(self) : Result[(), WinError] ! IO
        pub fn duplicate(&self) : Result[Self, WinError] ! IO
    }

    // Standard handles
    pub fn stdin() : Handle ! IO
    pub fn stdout() : Handle ! IO
    pub fn stderr() : Handle ! IO
}

/// Windows registry
pub mod registry {
    pub type RegKey { handle: Handle }

    pub enum Hive {
        ClassesRoot,
        CurrentUser,
        LocalMachine,
        Users,
        CurrentConfig,
    }

    impl RegKey {
        pub fn open(hive: Hive, path: &str, access: RegAccess) : Result[Self, WinError] ! IO
        pub fn create(hive: Hive, path: &str) : Result[(Self, bool), WinError] ! IO  // (key, created)
        pub fn delete(hive: Hive, path: &str) : Result[(), WinError] ! IO

        pub fn get_value(&self, name: &str) : Result[RegValue, WinError] ! IO
        pub fn set_value(&self, name: &str, value: &RegValue) : Result[(), WinError] ! IO
        pub fn delete_value(&self, name: &str) : Result[(), WinError] ! IO

        pub fn subkeys(&self) : impl Iterator[Item = Result[String, WinError]] ! IO
        pub fn values(&self) : impl Iterator[Item = Result[(String, RegValue), WinError]] ! IO
    }

    pub enum RegValue {
        String(String),
        ExpandString(String),
        Binary(Vec[u8]),
        Dword(u32),
        Qword(u64),
        MultiString(Vec[String]),
    }

    bitflags pub type RegAccess: u32 {
        const QUERY_VALUE   = 0x0001
        const SET_VALUE     = 0x0002
        const CREATE_SUBKEY = 0x0004
        const ENUM_SUBKEYS  = 0x0008
        const NOTIFY        = 0x0010
        const READ          = 0x20019
        const WRITE         = 0x20006
        const ALL_ACCESS    = 0xF003F
    }
}

/// Windows services
pub mod service {
    pub type ServiceManager { handle: Handle }
    pub type Service { handle: Handle }

    impl ServiceManager {
        pub fn connect() : Result[Self, WinError] ! IO
        pub fn open_service(&self, name: &str, access: ServiceAccess)
            : Result[Service, WinError] ! IO
        pub fn create_service(&self, config: &ServiceConfig)
            : Result[Service, WinError] ! IO
    }

    impl Service {
        pub fn start(&self, args: &[&str]) : Result[(), WinError] ! IO
        pub fn stop(&self) : Result[ServiceStatus, WinError] ! IO
        pub fn query_status(&self) : Result[ServiceStatus, WinError] ! IO
        pub fn delete(self) : Result[(), WinError] ! IO
    }

    pub enum ServiceState {
        Stopped,
        StartPending,
        StopPending,
        Running,
        ContinuePending,
        PausePending,
        Paused,
    }
}

/// Windows console
pub mod console {
    pub fn set_console_mode(handle: Handle, mode: ConsoleMode) : Result[(), WinError] ! IO
    pub fn get_console_mode(handle: Handle) : Result[ConsoleMode, WinError] ! IO

    bitflags pub type ConsoleMode: u32 {
        // Input modes
        const ENABLE_PROCESSED_INPUT = 0x0001
        const ENABLE_LINE_INPUT      = 0x0002
        const ENABLE_ECHO_INPUT      = 0x0004
        const ENABLE_WINDOW_INPUT    = 0x0008
        const ENABLE_MOUSE_INPUT     = 0x0010
        const ENABLE_VIRTUAL_TERMINAL_INPUT = 0x0200

        // Output modes
        const ENABLE_PROCESSED_OUTPUT = 0x0001
        const ENABLE_WRAP_AT_EOL_OUTPUT = 0x0002
        const ENABLE_VIRTUAL_TERMINAL_PROCESSING = 0x0004
        const DISABLE_NEWLINE_AUTO_RETURN = 0x0008
    }

    // Virtual terminal support detection
    pub fn supports_virtual_terminal() : bool ! IO
}

/// Windows security
pub mod security {
    pub type SecurityDescriptor { /* ... */ }
    pub type Sid { /* ... */ }  // Security Identifier
    pub type Acl { /* ... */ }  // Access Control List

    // Token operations
    pub fn open_process_token(access: TokenAccess) : Result[Handle, WinError] ! IO
    pub fn get_token_user(token: Handle) : Result[Sid, WinError] ! IO
    pub fn check_membership(token: Handle, sid: &Sid) : Result[bool, WinError] ! IO

    // Privilege operations
    pub fn enable_privilege(token: Handle, privilege: &str) : Result[bool, WinError] ! IO

    // Well-known SIDs
    pub fn well_known_sid(kind: WellKnownSid) : Result[Sid, WinError]

    pub enum WellKnownSid {
        LocalSystem,
        LocalService,
        NetworkService,
        Administrators,
        Users,
        Guests,
        Everyone,
        // ...
    }
}
```

### 4.6 WASI Implementation

WebAssembly System Interface — component model.

```ferrum
// sys/wasi/mod.fe

/// WASI platform — WebAssembly System Interface.
/// Targets WASI Preview 2 (Component Model).
/// Preview 1 compatibility layer available.
pub type WasiPlatform {
    preview: WasiPreview,
}

pub enum WasiPreview {
    Preview1,   // Deprecated but supported for compatibility
    Preview2,   // Current stable (2024+)
    Preview3,   // Future
}

impl Platform for WasiPlatform {
    type FileSystem = WasiFileSystem
    type Network = WasiNetwork
    type Process = WasiProcess     // Limited — no fork/exec
    type Memory = WasiMemory       // Limited — no mmap
    type Time = WasiTime
    type Entropy = WasiEntropy
    type Threading = WasiThreading // Optional — requires wasi-threads

    fn name(&self) : &str { "wasi" }
    fn version(&self) : OsVersion { /* WASI version */ }
    fn capabilities(&self) : CapabilitySet {
        // WASI capabilities depend on what the host grants
        wasi_capabilities()
    }
}

/// WASI filesystem — capability-based
pub mod filesystem {
    /// A directory handle that grants access to files within.
    /// This is WASI's capability model — you can only access files
    /// if you have a handle to a parent directory.
    pub type Descriptor { handle: u32 }

    impl Descriptor {
        /// Open a file relative to this directory.
        pub fn open_at(&self, path: &str, flags: OpenFlags, mode: FileMode)
            : Result[Descriptor, WasiError] ! IO

        pub fn read(&self, buf: &mut [u8]) : Result[usize, WasiError] ! IO
        pub fn write(&self, buf: &[u8]) : Result[usize, WasiError] ! IO
        pub fn seek(&self, offset: i64, whence: Whence) : Result[u64, WasiError] ! IO

        pub fn stat(&self) : Result[Filestat, WasiError] ! IO
        pub fn stat_at(&self, path: &str) : Result[Filestat, WasiError] ! IO

        pub fn readdir(&self) : impl Iterator[Item = Result[DirEntry, WasiError]] ! IO

        pub fn sync(&self) : Result[(), WasiError] ! IO
        pub fn close(self) : Result[(), WasiError] ! IO
    }

    /// Get preopened directories from the WASI host.
    pub fn preopens() : impl Iterator[Item = (Descriptor, String)] ! IO
}

/// WASI sockets — Preview 2
pub mod sockets {
    pub type TcpSocket { /* ... */ }
    pub type UdpSocket { /* ... */ }

    impl TcpSocket {
        pub fn new(family: AddressFamily) : Result[Self, WasiError] ! Net
        pub fn bind(&self, addr: &IpSocketAddress) : Result[(), WasiError] ! Net
        pub fn listen(&self, backlog: u32) : Result[(), WasiError] ! Net
        pub fn accept(&self) : Result[(Self, IpSocketAddress), WasiError] ! Net
        pub fn connect(&self, addr: &IpSocketAddress) : Result[(), WasiError] ! Net
        pub fn send(&self, data: &[u8]) : Result[usize, WasiError] ! Net
        pub fn recv(&self, buf: &mut [u8]) : Result[usize, WasiError] ! Net
        pub fn shutdown(&self, how: ShutdownType) : Result[(), WasiError] ! Net
    }

    pub enum AddressFamily {
        Ipv4,
        Ipv6,
    }

    pub type IpSocketAddress {
        family: AddressFamily,
        port: u16,
        address: IpAddress,
    }
}

/// WASI clocks
pub mod clocks {
    pub fn monotonic_now() : u64   // nanoseconds
    pub fn wall_clock_now() : (u64, u32)   // (seconds, nanoseconds)
    pub fn resolution() : u64      // nanoseconds

    /// Subscribe to a clock for async polling.
    pub fn subscribe_monotonic(when: u64, absolute: bool) : Pollable
    pub fn subscribe_duration(duration: u64) : Pollable
}

/// WASI random
pub mod random {
    pub fn get_random_bytes(len: usize) : Vec[u8] ! IO
    pub fn get_random_u64() : u64 ! IO
    pub fn insecure_random() : (u64, u64)   // Not cryptographic!
}

/// WASI poll — unified readiness polling
pub mod poll {
    pub type Pollable { handle: u32 }

    /// Poll multiple pollables, returning indices of ready ones.
    pub fn poll_list(pollables: &[Pollable]) : Vec[u32] ! Async

    /// Poll with timeout.
    pub fn poll_list_timeout(pollables: &[Pollable], timeout_ns: u64) : Vec[u32] ! Async
}

/// WASI HTTP — outgoing requests (Preview 2)
pub mod http {
    pub type OutgoingRequest { /* ... */ }
    pub type IncomingResponse { /* ... */ }
    pub type OutgoingBody { /* ... */ }
    pub type IncomingBody { /* ... */ }

    impl OutgoingRequest {
        pub fn new(method: Method, path: &str, headers: &[(String, String)])
            : Result[Self, WasiError]
        pub fn set_body(&mut self, body: OutgoingBody)
        pub fn send(self, target: &str) : Result[FutureIncomingResponse, WasiError] ! Net
    }

    pub type FutureIncomingResponse { /* pollable */ }

    impl FutureIncomingResponse {
        pub fn subscribe(&self) : Pollable
        pub fn get(&self) : Option[Result[IncomingResponse, WasiError]]
    }
}

/// WASI CLI — command-line interface
pub mod cli {
    pub fn args() : impl Iterator[Item = String]
    pub fn env_vars() : impl Iterator[Item = (String, String)]
    pub fn stdin() : InputStream ! IO
    pub fn stdout() : OutputStream ! IO
    pub fn stderr() : OutputStream ! IO
    pub fn exit(code: u32) : never
}
```

### 4.7 Fuchsia Implementation

Google's capability-based operating system built on the Zircon microkernel.

```ferrum
// sys/fuchsia/mod.fe

/// Fuchsia platform — capability-based OS on Zircon microkernel.
/// Minimum supported: Fuchsia F15+
pub type FuchsiaPlatform {
    version: FuchsiaVersion,
}

impl Platform for FuchsiaPlatform {
    type FileSystem = FuchsiaFileSystem    // fdio-based
    type Network = FuchsiaNetwork          // fdio sockets
    type Process = FuchsiaProcess          // Zircon processes
    type Memory = FuchsiaMemory            // VMOs
    type Time = FuchsiaTime                // Zircon clocks
    type Entropy = FuchsiaEntropy          // cprng
    type Threading = FuchsiaThreading      // Zircon threads

    fn name(&self) : &str { "fuchsia" }
    fn version(&self) : OsVersion { /* ... */ }
    fn capabilities(&self) : CapabilitySet { /* namespace-based */ }
}

/// Zircon kernel objects and handles
pub mod zx {
    /// A handle to a Zircon kernel object.
    /// Handles are capabilities — they grant specific rights to objects.
    pub type Handle(u32)

    impl Handle {
        pub const INVALID: Self = Handle(0)
        pub fn is_valid(&self) : bool { self.0 != 0 }
        pub fn close(self) : Result[(), ZxStatus] ! IO
        pub fn duplicate(&self, rights: Rights) : Result[Self, ZxStatus] ! IO
        pub fn replace(self, rights: Rights) : Result[Self, ZxStatus] ! IO
        pub fn get_info[T: ObjectInfo](&self) : Result[T, ZxStatus] ! IO
    }

    /// Zircon status codes
    pub type ZxStatus(i32)

    impl ZxStatus {
        pub const OK: Self = ZxStatus(0)
        pub const ERR_INTERNAL: Self = ZxStatus(-1)
        pub const ERR_NOT_SUPPORTED: Self = ZxStatus(-2)
        pub const ERR_NO_RESOURCES: Self = ZxStatus(-3)
        pub const ERR_NO_MEMORY: Self = ZxStatus(-4)
        pub const ERR_BAD_HANDLE: Self = ZxStatus(-11)
        pub const ERR_ACCESS_DENIED: Self = ZxStatus(-30)
        pub const ERR_TIMED_OUT: Self = ZxStatus(-21)
        pub const ERR_PEER_CLOSED: Self = ZxStatus(-24)
        // ... many more
    }

    /// Rights that can be held on a handle
    bitflags pub type Rights: u32 {
        const DUPLICATE  = 1 << 0
        const TRANSFER   = 1 << 1
        const READ       = 1 << 2
        const WRITE      = 1 << 3
        const EXECUTE    = 1 << 4
        const MAP        = 1 << 5
        const GET_PROPERTY = 1 << 6
        const SET_PROPERTY = 1 << 7
        const SIGNAL     = 1 << 12
        const WAIT       = 1 << 13
        // ...
    }

    /// Wait on object signals
    pub fn object_wait_one(
        handle: &Handle,
        signals: Signals,
        deadline: Time,
    ) : Result[Signals, ZxStatus] ! IO

    pub fn object_wait_many(
        items: &mut [WaitItem],
        deadline: Time,
    ) : Result[(), ZxStatus] ! IO

    bitflags pub type Signals: u32 {
        const READABLE      = 1 << 0
        const WRITABLE      = 1 << 1
        const PEER_CLOSED   = 1 << 2
        const USER_0        = 1 << 24
        const USER_1        = 1 << 25
        // ...
    }

    pub type Time(i64)  // nanoseconds since boot

    impl Time {
        pub const INFINITE: Self = Time(i64.MAX)
        pub fn after(duration: Duration) : Self
    }
}

/// Channels — bidirectional IPC
pub mod channel {
    pub type Channel { handle: zx.Handle }

    impl Channel {
        pub fn create() : Result[(Self, Self), ZxStatus] ! IO

        pub fn write(&self, bytes: &[u8], handles: &mut [zx.Handle])
            : Result[(), ZxStatus] ! IO

        pub fn read(&self, bytes: &mut [u8], handles: &mut [zx.Handle])
            : Result[(usize, usize), ZxStatus] ! IO

        pub fn call(
            &self,
            deadline: zx.Time,
            bytes: &[u8],
            handles: &mut [zx.Handle],
            reply_bytes: &mut [u8],
            reply_handles: &mut [zx.Handle],
        ) : Result[(usize, usize), ZxStatus] ! IO
    }
}

/// Virtual Memory Objects
pub mod vmo {
    pub type Vmo { handle: zx.Handle }

    impl Vmo {
        pub fn create(size: u64) : Result[Self, ZxStatus] ! IO
        pub fn create_contiguous(bti: &Bti, size: usize) : Result[Self, ZxStatus] ! IO

        pub fn read(&self, offset: u64, buf: &mut [u8]) : Result[(), ZxStatus] ! IO
        pub fn write(&self, offset: u64, data: &[u8]) : Result[(), ZxStatus] ! IO

        pub fn get_size(&self) : Result[u64, ZxStatus] ! IO
        pub fn set_size(&self, size: u64) : Result[(), ZxStatus] ! IO

        pub fn create_child(&self, options: VmoChildOptions, offset: u64, size: u64)
            : Result[Self, ZxStatus] ! IO
    }

    bitflags pub type VmoChildOptions: u32 {
        const SNAPSHOT         = 1 << 0
        const SNAPSHOT_AT_LEAST_ON_WRITE = 1 << 4
        const RESIZABLE        = 1 << 2
        const SLICE            = 1 << 3
        const NO_WRITE         = 1 << 5
    }
}

/// Ports — async event aggregation
pub mod port {
    pub type Port { handle: zx.Handle }

    impl Port {
        pub fn create() : Result[Self, ZxStatus] ! IO

        pub fn queue(&self, packet: &Packet) : Result[(), ZxStatus] ! IO

        pub fn wait(&self, deadline: zx.Time) : Result[Packet, ZxStatus] ! IO

        pub fn cancel(&self, source: &zx.Handle, key: u64) : Result[(), ZxStatus] ! IO
    }

    pub type Packet {
        key: u64,
        type_: PacketType,
        status: ZxStatus,
        // union of signal/guest/interrupt/page_request data
    }
}

/// FIDL — Fuchsia Interface Definition Language bindings
pub mod fidl {
    /// Marker trait for FIDL protocols
    pub trait Protocol {
        const NAME: &str
    }

    /// Client end of a FIDL channel
    pub type ClientEnd[P: Protocol] { channel: channel.Channel }

    /// Server end of a FIDL channel
    pub type ServerEnd[P: Protocol] { channel: channel.Channel }

    impl[P: Protocol] ClientEnd[P] {
        pub fn into_channel(self) : channel.Channel
    }

    impl[P: Protocol] ServerEnd[P] {
        pub fn into_channel(self) : channel.Channel
    }

    /// Connect to a protocol in the component's namespace
    pub fn connect_to_protocol[P: Protocol]() : Result[ClientEnd[P], FidlError] ! IO

    /// Example FIDL protocol usage:
    /// ```ferrum
    /// // Generated from FIDL definition
    /// mod fuchsia_io {
    ///     pub trait Directory: Protocol {
    ///         fn open(&self, flags: OpenFlags, path: &str) -> Result[NodeProxy, Status]
    ///         fn read_dirents(&self, max_bytes: u64) -> Result[Vec[u8], Status]
    ///     }
    /// }
    /// ```
}

/// Component framework
pub mod component {
    pub type Namespace { /* capability namespace */ }

    impl Namespace {
        /// Get the component's incoming namespace
        pub fn take_incoming() : Self

        /// Open a capability from the namespace
        pub fn open[P: fidl.Protocol](&self, path: &str)
            : Result[fidl.ClientEnd[P], ComponentError] ! IO
    }

    /// Expose capabilities to other components
    pub type OutgoingDir { /* ... */ }

    impl OutgoingDir {
        pub fn new() : Self
        pub fn add_protocol[P: fidl.Protocol](&mut self, server: impl Fn() -> fidl.ServerEnd[P])
        pub fn serve(self, channel: channel.Channel) : Result[(), ComponentError] ! IO
    }
}

/// Process management
pub mod process {
    pub type Process { handle: zx.Handle }
    pub type Job { handle: zx.Handle }
    pub type Thread { handle: zx.Handle }

    impl Process {
        pub fn self_() : Self ! IO
        pub fn create(job: &Job, name: &str) : Result[Self, ZxStatus] ! IO
        pub fn start(&self, entry: usize, stack: usize, arg1: zx.Handle, arg2: usize)
            : Result[(), ZxStatus] ! IO
        pub fn kill(&self) : Result[(), ZxStatus] ! IO
        pub fn wait(&self) : Result[i64, ZxStatus] ! IO  // return code
    }

    impl Job {
        pub fn default() : Self ! IO  // current job
        pub fn create(parent: &Job) : Result[Self, ZxStatus] ! IO
        pub fn kill(&self) : Result[(), ZxStatus] ! IO
        pub fn set_policy(&self, policy: &[JobPolicy]) : Result[(), ZxStatus] ! IO
    }
}

/// File I/O via fdio
pub mod fdio {
    /// fdio provides POSIX-like file operations over Fuchsia protocols
    pub fn open(path: &str, flags: OpenFlags) : Result[Fd, ZxStatus] ! IO
    pub fn openat(dirfd: Fd, path: &str, flags: OpenFlags) : Result[Fd, ZxStatus] ! IO

    /// Get the underlying Zircon handle for a file descriptor
    pub fn fd_to_handle(fd: Fd) : Result[zx.Handle, ZxStatus] ! IO

    /// Create a file descriptor from a handle
    pub fn handle_to_fd(handle: zx.Handle, flags: i32) : Result[Fd, ZxStatus] ! IO
}
```

### 4.8 OpenHarmony Implementation

Huawei's open-source distributed operating system.

```ferrum
// sys/ohos/mod.fe

/// OpenHarmony platform — distributed OS for IoT and mobile.
/// Minimum supported: OpenHarmony 4.0+
pub type OhosPlatform {
    version: OhosVersion,
    device_type: DeviceType,
}

pub enum DeviceType {
    Phone,
    Tablet,
    Wearable,
    SmartTV,
    Car,
    IoT,
}

impl Platform for OhosPlatform {
    type FileSystem = OhosFileSystem
    type Network = OhosNetwork
    type Process = OhosProcess
    type Memory = OhosMemory
    type Time = OhosTime
    type Entropy = OhosEntropy
    type Threading = OhosThreading

    fn name(&self) : &str { "ohos" }
    fn version(&self) : OsVersion { /* ... */ }
    fn capabilities(&self) : CapabilitySet { /* permission-based */ }
}

/// Ability framework — application components
pub mod ability {
    /// Base trait for all ability types
    pub trait Ability {
        fn on_start(&mut self)
        fn on_stop(&mut self)
        fn on_foreground(&mut self)
        fn on_background(&mut self)
    }

    /// UI Ability — has a window
    pub trait UIAbility: Ability {
        fn on_window_stage_create(&mut self, stage: WindowStage)
        fn on_window_stage_destroy(&mut self)
    }

    /// Service Ability — background service
    pub trait ServiceAbility: Ability {
        fn on_connect(&mut self, want: &Want) : Option[RemoteObject]
        fn on_disconnect(&mut self, want: &Want)
    }

    /// Want — intent to start abilities
    pub type Want {
        bundle_name: Option[String],
        ability_name: Option[String],
        device_id: Option[String],
        parameters: HashMap[String, WantParam],
    }

    /// Start an ability
    pub fn start_ability(want: &Want) : Result[(), AbilityError] ! IO

    /// Start ability for result
    pub fn start_ability_for_result(want: &Want)
        : Result[AbilityResult, AbilityError] ! IO + Async
}

/// Distributed soft bus — cross-device communication
pub mod softbus {
    /// Discover nearby devices
    pub fn start_discovery(info: &PublishInfo)
        : Result[Receiver[DeviceInfo], SoftBusError] ! Net

    pub fn stop_discovery() : Result[(), SoftBusError] ! Net

    /// Device information
    pub type DeviceInfo {
        device_id: String,
        device_name: String,
        device_type: DeviceType,
        network_id: String,
    }

    /// Create a session for data transfer
    pub type Session { /* ... */ }

    impl Session {
        pub fn open(peer_device_id: &str, group_id: &str)
            : Result[Self, SoftBusError] ! Net

        pub fn send(&self, data: &[u8]) : Result[(), SoftBusError] ! Net
        pub fn receive(&self) : Result[Vec[u8], SoftBusError] ! Net + Async

        pub fn close(self) : Result[(), SoftBusError] ! Net
    }

    /// Distributed data management
    pub mod data {
        pub type DistributedKvStore { /* ... */ }

        impl DistributedKvStore {
            pub fn open(store_id: &str, options: &KvStoreOptions)
                : Result[Self, DataError] ! IO

            pub fn put(&self, key: &str, value: &[u8]) : Result[(), DataError] ! IO
            pub fn get(&self, key: &str) : Result[Option[Vec[u8]], DataError] ! IO
            pub fn delete(&self, key: &str) : Result[(), DataError] ! IO

            /// Sync data to other devices
            pub fn sync(&self, device_ids: &[&str]) : Result[(), DataError] ! Net
        }
    }
}

/// Hardware Driver Foundation
pub mod hdf {
    pub type DeviceService { /* ... */ }

    /// Get a device service by name
    pub fn get_service(service_name: &str) : Result[DeviceService, HdfError] ! IO

    /// Device manager
    pub type DeviceManager { /* ... */ }

    impl DeviceManager {
        pub fn list_devices() : Result[Vec[DeviceInfo], HdfError] ! IO
        pub fn bind_device(device_id: &str) : Result[DeviceService, HdfError] ! IO
    }
}

/// HiLog — logging framework
pub mod hilog {
    pub enum LogLevel {
        Debug,
        Info,
        Warn,
        Error,
        Fatal,
    }

    pub fn log(level: LogLevel, domain: u32, tag: &str, msg: &str) ! IO

    // Convenience macros map to these
    pub fn debug(domain: u32, tag: &str, msg: &str) ! IO
    pub fn info(domain: u32, tag: &str, msg: &str) ! IO
    pub fn warn(domain: u32, tag: &str, msg: &str) ! IO
    pub fn error(domain: u32, tag: &str, msg: &str) ! IO
}

/// HiTrace — distributed tracing
pub mod hitrace {
    pub type TraceId {
        chain_id: u64,
        span_id: u64,
        parent_span_id: u64,
    }

    pub fn begin(name: &str) : TraceId
    pub fn end(id: TraceId)
    pub fn async_begin(name: &str, task_id: i32) : TraceId
    pub fn async_end(id: TraceId, task_id: i32)

    /// Cross-device trace propagation
    pub fn get_current_trace() : Option[TraceId]
    pub fn set_current_trace(id: TraceId)
}

/// IPC/RPC framework
pub mod ipc {
    pub type RemoteObject { /* ... */ }
    pub type MessageParcel { /* ... */ }

    impl MessageParcel {
        pub fn new() : Self
        pub fn write_int(&mut self, val: i32) : Result[(), IpcError]
        pub fn write_string(&mut self, val: &str) : Result[(), IpcError]
        pub fn write_buffer(&mut self, buf: &[u8]) : Result[(), IpcError]
        pub fn read_int(&mut self) : Result[i32, IpcError]
        pub fn read_string(&mut self) : Result[String, IpcError]
        pub fn read_buffer(&mut self, len: usize) : Result[Vec[u8], IpcError]
    }

    impl RemoteObject {
        pub fn send_request(
            &self,
            code: u32,
            data: &MessageParcel,
            reply: &mut MessageParcel,
        ) : Result[(), IpcError] ! IO
    }
}
```

### 4.9 Zephyr RTOS Implementation

For embedded systems.

```ferrum
// sys/zephyr/mod.fe

/// Zephyr RTOS platform — real-time embedded systems.
/// Zephyr version: 3.5+
pub type ZephyrPlatform {
    version: ZephyrVersion,
}

impl Platform for ZephyrPlatform {
    type FileSystem = ZephyrFileSystem   // Optional — depends on config
    type Network = ZephyrNetwork         // Optional — depends on config
    type Process = ZephyrProcess         // Limited — no fork, single address space
    type Memory = ZephyrMemory
    type Time = ZephyrTime
    type Entropy = ZephyrEntropy
    type Threading = ZephyrThreading     // Zephyr threads, not pthreads

    fn name(&self) : &str { "zephyr" }
    fn version(&self) : OsVersion { /* ... */ }
    fn capabilities(&self) : CapabilitySet { /* hardware-dependent */ }
}

/// Zephyr kernel primitives
pub mod kernel {
    pub type Thread { /* k_thread */ }
    pub type Mutex { /* k_mutex */ }
    pub type Semaphore { /* k_sem */ }
    pub type MessageQueue { /* k_msgq */ }
    pub type Timer { /* k_timer */ }
    pub type WorkQueue { /* k_work_q */ }

    impl Thread {
        pub fn spawn(
            stack: &'static mut [u8],
            priority: i32,
            entry: fn(*mut ()),
            arg: *mut (),
        ) : Result[Self, ZephyrError]

        pub fn current() : Self
        pub fn abort(&self)
        pub fn suspend(&self)
        pub fn resume(&self)
        pub fn join(&self)
        pub fn sleep(duration: Duration)
        pub fn yield_()
    }

    impl Mutex {
        pub const fn new() : Self
        pub fn lock(&self, timeout: Duration) : Result[(), ZephyrError]
        pub fn unlock(&self)
    }

    impl Semaphore {
        pub const fn new(initial: u32, limit: u32) : Self
        pub fn take(&self, timeout: Duration) : Result[(), ZephyrError]
        pub fn give(&self)
        pub fn count(&self) : u32
    }

    // Interrupt context detection
    pub fn in_isr() : bool
    pub fn irq_lock() : IrqKey
    pub fn irq_unlock(key: IrqKey)

    pub type IrqKey(u32)
}

/// Device drivers
pub mod device {
    pub type Device { /* struct device * */ }

    impl Device {
        pub fn get_binding(name: &str) : Option[Self]
        pub fn is_ready(&self) : bool
    }
}

/// GPIO
pub mod gpio {
    pub type GpioPort { device: Device }
    pub type GpioPin(u8)

    impl GpioPort {
        pub fn get(name: &str) : Option[Self]
        pub fn configure(&self, pin: GpioPin, flags: GpioFlags) : Result[(), ZephyrError]
        pub fn set(&self, pin: GpioPin, value: bool) : Result[(), ZephyrError]
        pub fn get(&self, pin: GpioPin) : Result[bool, ZephyrError]
        pub fn toggle(&self, pin: GpioPin) : Result[(), ZephyrError]
    }

    bitflags pub type GpioFlags: u32 {
        const INPUT     = 1 << 0
        const OUTPUT    = 1 << 1
        const PULL_UP   = 1 << 4
        const PULL_DOWN = 1 << 5
        const ACTIVE_LOW = 1 << 8
        const INT_EDGE_RISING  = 1 << 16
        const INT_EDGE_FALLING = 1 << 17
        const INT_LEVEL_HIGH   = 1 << 18
        const INT_LEVEL_LOW    = 1 << 19
    }

    pub type GpioCallback {
        // Callback registration for interrupts
    }
}

/// I2C
pub mod i2c {
    pub type I2cBus { device: Device }

    impl I2cBus {
        pub fn get(name: &str) : Option[Self]

        pub fn write(&self, addr: u16, data: &[u8]) : Result[(), ZephyrError]
        pub fn read(&self, addr: u16, buf: &mut [u8]) : Result[(), ZephyrError]
        pub fn write_read(&self, addr: u16, write: &[u8], read: &mut [u8])
            : Result[(), ZephyrError]

        pub fn transfer(&self, addr: u16, msgs: &mut [I2cMsg]) : Result[(), ZephyrError]
    }

    pub type I2cMsg {
        buf: *mut u8,
        len: u32,
        flags: I2cMsgFlags,
    }

    bitflags pub type I2cMsgFlags: u8 {
        const WRITE   = 0
        const READ    = 1 << 0
        const STOP    = 1 << 1
        const RESTART = 1 << 2
    }
}

/// SPI
pub mod spi {
    pub type SpiBus { device: Device }
    pub type SpiConfig {
        frequency: u32,
        operation: SpiOperation,
        cs: Option[GpioCs],
    }

    impl SpiBus {
        pub fn get(name: &str) : Option[Self]
        pub fn transceive(&self, config: &SpiConfig, tx: &[u8], rx: &mut [u8])
            : Result[(), ZephyrError]
    }

    bitflags pub type SpiOperation: u16 {
        const MODE_CPOL = 1 << 0
        const MODE_CPHA = 1 << 1
        const WORD_8BIT = 8 << 5
        const WORD_16BIT = 16 << 5
        const MSB_FIRST = 0
        const LSB_FIRST = 1 << 4
    }
}

/// UART
pub mod uart {
    pub type Uart { device: Device }

    impl Uart {
        pub fn get(name: &str) : Option[Self]

        pub fn poll_in(&self) : Option[u8]
        pub fn poll_out(&self, byte: u8)

        pub fn fifo_read(&self, buf: &mut [u8]) : usize
        pub fn fifo_fill(&self, data: &[u8]) : usize

        pub fn irq_rx_enable(&self)
        pub fn irq_rx_disable(&self)
        pub fn irq_tx_enable(&self)
        pub fn irq_tx_disable(&self)

        pub fn set_callback(&self, cb: fn(&Uart))
    }

    pub type UartConfig {
        baudrate: u32,
        parity: Parity,
        stop_bits: StopBits,
        data_bits: DataBits,
        flow_control: FlowControl,
    }
}

/// Bluetooth (if enabled)
@cfg(feature = "bluetooth")
pub mod bluetooth {
    pub mod gap {
        pub fn enable() : Result[(), BtError]
        pub fn start_advertising(params: &AdvParams) : Result[(), BtError]
        pub fn stop_advertising() : Result[(), BtError]
        pub fn start_scanning(params: &ScanParams, cb: fn(&AdvReport)) : Result[(), BtError]
    }

    pub mod gatt {
        // GATT server/client
    }
}

/// Networking stack (if enabled)
@cfg(feature = "networking")
pub mod net {
    // Zephyr networking APIs
    pub type Socket { /* ... */ }
    pub type Interface { /* ... */ }
}
```

---

## 5. Capability Mapping

How Ferrum's capability/effect system maps to platform permissions.

### 5.1 Effect to Platform Permission Mapping

```ferrum
/// Maps Ferrum effects to platform-specific permissions.
trait CapabilityMapper {
    /// Check if the current process has a capability.
    fn has_capability(&self, cap: Capability) : bool

    /// Request a capability (may prompt user or fail).
    fn request_capability(&self, cap: Capability) : Result[(), CapabilityError] ! IO

    /// Drop a capability (for sandboxing).
    fn drop_capability(&self, cap: Capability) : Result[(), CapabilityError] ! IO
}

enum Capability {
    // File system
    ReadFile(PathPattern),
    WriteFile(PathPattern),
    CreateFile(PathPattern),
    DeleteFile(PathPattern),
    ExecuteFile(PathPattern),

    // Network
    NetConnect(HostPattern, PortRange),
    NetListen(PortRange),
    NetRawSocket,

    // Process
    SpawnProcess,
    SignalProcess,
    ReadProcessInfo,

    // System
    ReadEnv,
    WriteEnv,
    ReadTime,
    WriteTime,
    Entropy,

    // Hardware (embedded)
    Gpio,
    I2c,
    Spi,
    Uart,

    // Dangerous
    RawMemory,
    RawSyscall,
    LoadCode,
}

type PathPattern = String   // glob pattern: "/tmp/*", "/home/user/**"
type HostPattern = String   // "*.example.com", "192.168.1.*"
type PortRange = (u16, u16) // (start, end) inclusive
```

### 5.2 Platform-Specific Capability Enforcement

```ferrum
// Linux: seccomp + landlock
@cfg(target_os = "linux")
impl CapabilityMapper for LinuxPlatform {
    fn drop_capability(&self, cap: Capability) : Result[(), CapabilityError] ! IO {
        match cap {
            Capability.ReadFile(pattern) => {
                // Use Landlock to restrict file reads
                let ruleset = landlock.Ruleset.new()?
                // ... configure rules based on pattern ...
                ruleset.restrict_self()
            }
            Capability.NetConnect(_, _) => {
                // Use seccomp to block connect() syscall
                let filter = seccomp.Filter.new()
                filter.deny(seccomp.Syscall.Connect)
                filter.apply()
            }
            // ...
        }
    }
}

// OpenBSD: pledge + unveil
@cfg(target_os = "openbsd")
impl CapabilityMapper for BsdPlatform {
    fn drop_capability(&self, cap: Capability) : Result[(), CapabilityError] ! IO {
        // Build pledge string from remaining capabilities
        let promises = capabilities_to_pledge(remaining_caps())
        pledge.pledge(&promises, None)
    }
}

// FreeBSD: Capsicum
@cfg(target_os = "freebsd")
impl CapabilityMapper for BsdPlatform {
    fn drop_capability(&self, cap: Capability) : Result[(), CapabilityError] ! IO {
        // Enter capability mode and limit fd rights
        capsicum.cap_enter()?
        for fd in open_fds() {
            let rights = capability_to_cap_rights(cap)
            capsicum.cap_rights_limit(fd, rights)?
        }
        Ok(())
    }
}

// WASI: Inherently capability-based
@cfg(target_os = "wasi")
impl CapabilityMapper for WasiPlatform {
    fn has_capability(&self, cap: Capability) : bool {
        // Check preopened directories and granted interfaces
        match cap {
            Capability.ReadFile(pattern) => {
                wasi.filesystem.preopens().any(|(_, path)| {
                    pattern_matches(pattern, path)
                })
            }
            Capability.NetConnect(_, _) => {
                wasi.sockets.is_available()
            }
            // ...
        }
    }
}

// Windows: Process tokens and ACLs
@cfg(target_os = "windows")
impl CapabilityMapper for WindowsPlatform {
    fn has_capability(&self, cap: Capability) : bool {
        let token = security.open_process_token(TokenAccess.QUERY).ok()?
        match cap {
            Capability.WriteTime => {
                security.check_privilege(token, "SeSystemtimePrivilege")
            }
            // ...
        }
    }
}
```

---

## 6. Versioning and Evolution

### 6.1 Platform API Versioning

```ferrum
/// Every platform API declares its version requirements.
/// The compiler checks these at build time.

@platform_version(linux >= "5.1")
pub mod io_uring { /* ... */ }

@platform_version(linux >= "5.13")
pub mod landlock { /* ... */ }

@platform_version(wasi >= "0.2")
pub mod component_model { /* ... */ }

@platform_version(windows >= "10.0.17763")
pub mod virtual_terminal { /* ... */ }

/// Runtime version checking for optional features.
fn check_io_uring_support() : bool {
    @cfg(target_os = "linux")
    {
        sys.linux.kernel_version() >= KernelVersion { major: 5, minor: 1, patch: 0 }
            && sys.features.has_feature(Feature.IoUring)
    }
    @cfg(not(target_os = "linux"))
    {
        false
    }
}
```

### 6.2 Deprecation Strategy

```ferrum
/// APIs are deprecated with a clear timeline.
/// Deprecated APIs continue to work but emit warnings.

#[deprecated(since = "0.5", note = "Use io_uring instead", removal = "1.0")]
pub fn epoll_wait(/* ... */) { /* ... */ }

/// Platform preview APIs are marked unstable.
#[unstable(feature = "linux_6_5_features", issue = "1234")]
pub mod io_uring_multishot { /* ... */ }

/// Experimental APIs require explicit opt-in.
#[cfg(feature = "experimental_pidfd")]
pub mod pidfd_getfd { /* ... */ }
```

### 6.3 Forward Compatibility

```ferrum
/// Unknown syscall handling for future kernel features.
pub fn syscall_available(nr: i64) -> bool {
    // Try syscall with invalid args; check for ENOSYS
    unsafe {
        let ret = sys.linux.ffi.syscall(nr, 0, 0, 0, 0, 0, 0)
        errno() != libc.ENOSYS
    }
}

/// Feature flags for conditional compilation.
/// New features can be added without breaking existing code.
#[cfg(feature = "linux_6_8")]
pub mod io_uring_zcrx { /* zero-copy receive */ }

/// Extension traits for platform-specific additions.
/// Core traits remain stable; extensions add new methods.
pub trait LinuxFileExt: FileHandle {
    fn copy_file_range(&self, src_off: u64, dst: &impl FileHandle, dst_off: u64, len: usize)
        -> Result[usize, IoError] ! IO
}

impl LinuxFileExt for LinuxFile {
    // Linux-specific implementation
}
```

### 6.4 Platform Abstraction Layers for Testing

```ferrum
/// Mock platform for testing without an OS.
pub type MockPlatform {
    fs: MockFileSystem,
    net: MockNetwork,
    time: MockTime,
    // ...
}

impl MockPlatform {
    pub fn new() -> Self
    pub fn with_file(mut self, path: &str, content: &[u8]) -> Self
    pub fn with_time(mut self, now: Timestamp) -> Self
    pub fn advance_time(&self, duration: Duration)
    pub fn inject_error(&self, on: MockOperation, error: IoError)
}

impl Platform for MockPlatform {
    // Implement all platform traits with controllable behavior
}

// Usage in tests
#[test]
fn test_file_read() {
    let platform = MockPlatform.new()
        .with_file("/test.txt", b"hello")

    using platform {
        let content = std.fs.read_text("/test.txt")?
        assert_eq!(content, "hello")
    }
}
```

---

## 7. Summary

### Platform Support Matrix

| Platform | Status | Minimum Version | Key Features |
|----------|--------|-----------------|--------------|
| Linux | Tier 1 | 5.4 LTS | io_uring, epoll, landlock, seccomp, namespaces |
| macOS | Tier 1 | 10.15 | kqueue, GCD, sandbox, XPC |
| Windows | Tier 1 | 10 1809 | IOCP, registry, services, virtual terminal |
| FreeBSD | Tier 2 | 13.0 | kqueue, Capsicum, jails |
| OpenBSD | Tier 2 | 7.0 | kqueue, pledge, unveil |
| Fuchsia | Tier 2 | F15 | Zircon kernel, FIDL, capabilities, components |
| OpenHarmony | Tier 2 | 4.0 | Distributed soft bus, ability framework, HDF |
| WASI | Tier 2 | Preview 2 | Component model, capabilities |
| Zephyr | Tier 3 | 3.5 | Real-time, GPIO, I2C, SPI, BLE |
| Bare metal | Tier 3 | N/A | No OS, direct hardware |

### Design Principles Recap

1. **Portable by default** — `std.*` works everywhere with platform-appropriate behavior
2. **Power when needed** — `sys.linux.io_uring` for Linux-specific performance
3. **Capability-based security** — Ferrum effects map to platform permissions
4. **Explicit versioning** — API version requirements are compile-time checked
5. **Futureproof** — Extension traits, feature flags, and deprecation paths
6. **Testable** — Mock platforms for unit testing without OS dependencies

---

*End of Platform Abstraction Layer — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
