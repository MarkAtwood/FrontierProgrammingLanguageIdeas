# Ferrum Language Reference — Concurrency

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Concurrency

### 1.1 Tasks and Structured Concurrency

Tasks are the unit of concurrency. They run on a managed thread pool.
Structured concurrency means tasks cannot outlive the scope that spawned
them — no leaks, no orphaned threads.

```ferrum
scope s {
    let h1 = s.spawn(fetch(url1))
    let h2 = s.spawn(fetch(url2))
    let h3 = s.spawn(fetch(url3))

    let r1 = h1.await
    let r2 = h2.await
    let r3 = h3.await
}   // scope waits for all tasks before exiting
    // even if the body panics or returns early
```

`scope` is always lexically scoped. There is no way to spawn a task that
outlives the enclosing `scope`.

**Thread model:** Ferrum does not expose OS threads directly in portable code.
The `scope.spawn()` abstraction runs tasks on a runtime-managed thread pool.
This is what 99% of concurrent code should use.

Raw OS thread primitives are available in platform-specific modules:
- `sys.posix.pthread` — POSIX threads (Linux, macOS, BSDs)
- `sys.windows.thread` — Windows threads
- `sys.wasi.thread` — WASI threads (when available)

This layering follows the same pattern as filesystem and networking:
portable abstractions in `std`, raw platform primitives in `sys.*`.

```ferrum
// Portable — use this
scope s {
    s.spawn(work())
}

// Platform-specific — when you need pthread_setaffinity, thread priority, etc.
import sys.posix.pthread

let handle = pthread.spawn(ThreadAttrs.default(), || {
    work()
})? ! IO + Unsafe
handle.join()?
```

The thread pool that powers `scope` is configurable at program startup:

```ferrum
fn main() {
    // Configure before first scope (optional — sensible defaults exist)
    Runtime.configure(RuntimeConfig {
        worker_threads: num_cpus(),      // default
        stack_size: 2 * 1024 * 1024,     // 2 MiB default
        thread_name_prefix: "ferrum-worker",
    })

    // Now use scope normally
    scope s { ... }
}
```

### 1.2 `spawn` Semantics

```ferrum
s.spawn(expr)             // spawn a task, returns Task[T] where T = type of expr
s.spawn_detached(expr)    // no handle returned, task still bounded by scope

// With capability override
s.spawn(with Heap as alloc { worker(data) })
```

The expression passed to `spawn` runs in a new task. Its captures must implement `Send`. Its effects must include `Sync` if it communicates with other tasks.

### 1.3 Channels

```ferrum
// Create a bounded channel
let (tx, rx) = Chan.bounded[Message](capacity: 32)

// Create an unbounded channel
let (tx, rx) = Chan.unbounded[Message]()

// Types
tx: Chan[Message, Send]   // send end
rx: Chan[Message, Recv]   // receive end
```

**Sending and receiving:**
```ferrum
tx.send(msg)?           // blocks if full (bounded), returns Result
tx.try_send(msg)?       // non-blocking, Err(Full) if full
tx.send_timeout(msg, 1s)?  // blocks up to duration

rx.recv()?              // blocks until message available
rx.try_recv()?          // non-blocking, Err(Empty) if empty
rx.recv_timeout(1s)?    // blocks up to duration
```

**Channel as iterator:**
```ferrum
for msg in rx {
    process(msg)
}
// Iterates until all senders are dropped and channel is empty
```

### 1.4 `select!`

```ferrum
let result = select! {
    rx1 -> msg       => handle_a(msg),
    rx2 -> msg       => handle_b(msg),
    tx  <- outgoing  => (),                    // send when ready
    timeout!(500ms)  => Err(Timeout),
    default          => continue_without_block, // non-blocking select
}
```

`select!` is an expression macro. All branches must yield the same type. Branches are evaluated in random order to avoid starvation. `default` makes the select non-blocking.

### 1.5 Synchronization Primitives

```ferrum
// Mutex
let m: Arc[Mutex[T]] = Arc.new(Mutex.new(value))
{
    let guard = m.lock()          // blocks, returns MutexGuard[T]
    *guard = new_value            // access through guard
}                                  // guard dropped here, lock released

// Read-Write Lock
let rw: Arc[RwLock[T]] = Arc.new(RwLock.new(value))
let r = rw.read()     // multiple readers allowed
let w = rw.write()    // exclusive writer

// Semaphore
let sem = Semaphore.new(permits: 4)
let permit = sem.acquire()    // blocks until a permit is available
// permit released when dropped

// Barrier
let barrier = Arc.new(Barrier.new(n_threads))
barrier.wait()    // blocks until n_threads have reached this point

// Once
static INIT: Once = Once.new()
INIT.call_once(|| expensive_initialization())
```

**Lock-holding detection:** The compiler tracks whether a synchronization guard is live across an `await` or `spawn` boundary and warns. Holding a lock across a blocking point is almost always a bug.

**Deadlock detection:** The compiler performs static analysis to detect obvious deadlock patterns (lock ordering violations where statically determinable). Non-obvious cases are reported by the runtime in debug mode.

### 1.6 Atomic Types

```ferrum
let a: Atomic[u32] = Atomic.new(0)

a.load(Ordering.Acquire)
a.store(1, Ordering.Release)
a.fetch_add(1, Ordering.SeqCst)
a.compare_exchange(expected, new, success_ord, fail_ord)
```

Memory orderings: `Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`.

### 1.7 `Arc` and `Rc`

```ferrum
Arc[T]   // atomically reference-counted, Send + Sync
Rc[T]    // single-threaded reference counted, !Send

let shared = Arc.new(value)
let clone  = shared.clone()   // increments count
// value dropped when count reaches zero
```

`Arc` and `Rc` provide shared ownership. The inner value is immutable without `Mutex` or `RwLock`.

### 1.8 Concurrency Primitive Placement

Ferrum organizes concurrency primitives across three tiers:

- **Language** — constructs the compiler must understand to maintain safety guarantees
- **Module** — constructs the type system can handle as opaque types
- **Out of band** — platform-specific, non-portable, requires explicit opt-in

The assignment of a primitive to a tier follows a single principle:

> A construct belongs in the language if and only if the compiler must
> understand its semantics to maintain correctness of region inference,
> the borrow checker, effect tracking, or proof obligations.
>
> Everything else is a module or a platform shim.

**The compiler has four systems that require semantic understanding:**

1. **Region inference** — tracking which memory a reference points to and how long it is valid
2. **The borrow checker** — enforcing aliasing rules (`&mut` exclusivity, no `&` concurrent with `&mut`)
3. **Effect inference** — tracking which effects a function performs
4. **Proof obligations** — generating verification conditions the SMT solver reasons about

If a concurrency primitive touches none of these four systems, it is a module.
If it touches one or more, only the specific interaction belongs in the language.

#### Atomics — Module with Compiler Intrinsics

`Atomic[T]` is a module-level type. The borrow checker sees it as an opaque
type with interior mutability. No special compiler knowledge needed.

What the compiler does know:
- **Memory ordering** — the `Ordering` enum (`Relaxed`, `Acquire`, `Release`,
  `AcqRel`, `SeqCst`) is normative in the language specification. The optimizer
  must respect these orderings.
- **The `fence` intrinsic** — a point the optimizer must not move code across.
  Lives in `core.sync`, not user-definable.

#### The `SyncGuard` Marker Trait — Language Hook

The compiler warns when a synchronization guard is live across a suspension point:

```ferrum
async fn bad() {
    let guard = mutex.lock()
    something_async().await    // warning: MutexGuard live across await
    drop(guard)
}
```

This requires the compiler to know which types are "guards":

```ferrum
marker trait SyncGuard
// Implemented by: MutexGuard, RwLockReadGuard, RwLockWriteGuard,
//                 SemaphorePermit, any user type that opts in
```

The compiler emits a warning (not an error) when a `SyncGuard` implementor
is live at `.await` or `scope.spawn(...)`. This is the full extent of the
compiler's special knowledge about synchronization types.

#### Hardware Transactional Memory — Out of Band

HTM (Intel TSX, IBM POWER HTM, ARM TME) is non-portable and best-effort.
A transaction can abort for reasons outside the programmer's control. Any
correct use requires a fallback path.

Putting HTM in the standard library would imply reliability. It is neither
reliable nor portable. Placement in `sys.arch.x86_64.htm`, `sys.arch.power.htm`,
etc. is honest about what it is.

```ferrum
import sys.arch.x86_64.htm   // opt-in, non-portable

fn transfer_htm(from: &mut u64, to: &mut u64, amount: u64): bool {
    match htm.begin() {
        TransactionState.Started => {
            if *from < amount { htm.abort(1) }
            *from -= amount
            *to   += amount
            htm.commit()
            true
        }
        TransactionState.Aborted(_) => false,
        TransactionState.Unsupported => false,
    }
    // Caller provides the fallback — HTM never guarantees progress
}
```

#### Software Transactional Memory — Module

STM (`sync.stm`) is fully implementable in library code using `Atomic`
compare-and-swap. The type `Stm[T]` statically prevents `TVar` reads and
writes outside of `atomically` — a type system constraint, not a language
constraint.

```ferrum
import sync.stm.{TVar, Stm, atomically, retry}

fn transfer(from: &TVar[u64], to: &TVar[u64], amount: u64) {
    atomically(do {
        let bal = read(from).await
        if bal < amount { retry() }
        write(from, bal - amount).await
        write(to, read(to).await + amount).await
    })
}
```

#### CHERI Capability Pointers — Language (Target Mode)

CHERI is an ISA extension where pointers are 128-bit capabilities carrying
bounds and permissions. This requires language-level treatment because:

- **Region inference** can use capability bounds as additional evidence
- **`unsafe` semantics change** — memory safety is hardware-enforced
- **Pointer layout changes** — `*const T` is 128 bits, not 64

CHERI compilation is activated by target triple (`aarch64-unknown-freebsd-purecap`).
No syntax changes for users. The target changes what pointers compile to.

#### MMIO Typed Registers — Module with Intrinsic

Register type definitions are a module concern (`sys.mmio`). The `volatile`
semantics — the compiler must not optimize away reads/writes or reorder them —
are a compiler intrinsic (`core.ptr.read_volatile`, `core.ptr.write_volatile`).

```ferrum
type MmioReg[T, const ADDR: usize] { ... }

impl[T: Copy, const ADDR: usize] MmioReg[T, ADDR] {
    fn read(&self): T ! Unsafe {
        unsafe { core.ptr.read_volatile(ADDR as *const T) }
    }
    fn write(&self, val: T) ! Unsafe {
        unsafe { core.ptr.write_volatile(ADDR as *mut T, val) }
    }
}
```

#### Summary Table

| Primitive | Tier | Key reason |
|---|---|---|
| `Atomic[T]` | Module (`sync`) | Type system sufficient; no compiler semantic knowledge needed |
| `Ordering` enum | Language spec | Defines the memory model; optimizer must respect it |
| `fence` / `sfence` / `lfence` | Compiler intrinsic | Optimizer must not move code across |
| `SyncGuard` marker trait | Language hook | Enables deadlock warning at suspension points |
| Hardware TM (TSX/ARM TME/POWER HTM) | Out of band (`sys.arch.*`) | Non-portable, best-effort, requires fallback |
| Software STM (`TVar`, `atomically`) | Module (`sync.stm`) | Fully implementable via `Atomic` and type system |
| CHERI capability pointers | Language (target mode) | Compiler must know for regions, layout, `unsafe` semantics |
| Tagged pointers | Module (`core.ptr`), `unchecked` | Bit manipulation + aliasing assertion |
| MMIO typed registers | Module + `volatile` intrinsic | Types are library; volatile must be compiler-enforced |
| `@thread_local` statics | Language attribute | Compiler must generate TLS addressing |
| Seqlock, RCU, lock-free structures | Module (`sync.*`) | Built on `Atomic`; no new primitives needed |

### 1.9 What This Design Avoids

**No compiler magic for common patterns.** Interior mutability is achieved
with `unchecked` blocks and `trusted` annotations — explicit, auditable,
no secret compiler knowledge.

**Platform-specific features in the portable stdlib.** HTM in the standard
library would imply it is reliable. It is not. Keeping it in `sys.arch.*`
is accurate documentation.

**Overloading `unsafe` for distinct concepts.** Dereferencing a raw pointer,
relaxing an aliasing rule, and asserting a hardware property are different
things with different audit implications. The graduated safety ladder
(`unchecked`, `trusted`, `extern`, `unsafe`) makes these distinct.

**STM as a language construct.** Languages that bake STM into syntax make
non-STM concurrency feel second-class. Ferrum's module-level STM composes
naturally with all other concurrency primitives.

**Implicit volatile.** C's `volatile` qualifier on variable declarations
makes it easy to forget (a bug) or over-apply (a performance problem).
Ferrum's `read_volatile` and `write_volatile` are explicit at every access.

### 1.10 Concurrency Safety Guarantees

Data-race freedom is enforced by the type system's `Send` and `Sync` bounds. The type system handles the common cases — the vast majority of concurrent code compiles with full safety guarantees and no annotations.

The `unchecked` and `trusted` levels exist for the cases the type system cannot handle: lock-free algorithms, custom synchronization primitives, FFI boundaries, and hardware-specific code. Every `unchecked` block in concurrent code is:
- A statement a security auditor can search for and read
- Documentation of a human judgment that would otherwise be invisible
- A machine-readable signal the compiler can use for targeted verification
- Load-bearing specification of something the compiler cannot verify automatically

The formal proof system provides additional verification for FFI boundaries, `trusted` annotations, and CHERI-targeted code. The proof system covers the cases where the type system cannot help.

---
