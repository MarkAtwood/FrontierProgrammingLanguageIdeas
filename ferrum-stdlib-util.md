# Ferrum Standard Library — fmt, time, sync, process, env

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 19. fmt — Formatting

### 19.1 Display and Debug

```ferrum
// Display — for human-readable output
trait Display {
    fn fmt(&self, f: &mut Formatter): Result[(), IoError] ! IO
}

// Debug — for programmer-readable output
trait Debug {
    fn fmt(&self, f: &mut Formatter): Result[(), IoError] ! IO
}

// Derive both automatically for most types
@derive(Debug, Display)
type Point { x: f32, y: f32 }
// Display: "Point { x: 1.0, y: 2.0 }"  (same as Debug by default unless overridden)
```

### 19.2 Formatter

```ferrum
type Formatter {
    fn write_str(&mut self, s: &str): Result[(), IoError] ! IO
    fn write_char(&mut self, c: char): Result[(), IoError] ! IO
    fn write_fmt(&mut self, args: Arguments): Result[(), IoError] ! IO
    fn width(&self): Option[usize]
    fn precision(&self): Option[usize]
    fn sign_plus(&self): bool     // {:+} flag
    fn sign_minus(&self): bool    // {:-} flag
    fn alternate(&self): bool     // {:#} flag
    fn sign_aware_zero_pad(&self): bool  // {:0} flag
    fn fill(&self): char
    fn align(&self): Option[Alignment]
    fn debug_struct(&mut self, name: &str): DebugStruct   // helpers for Debug impl
    fn debug_tuple(&mut self, name: &str): DebugTuple
    fn debug_list(&mut self): DebugList
    fn debug_map(&mut self): DebugMap
    fn debug_set(&mut self): DebugSet
}

// DebugStruct — for implementing Debug on structs
type DebugStruct { ... }
impl DebugStruct {
    fn field(&mut self, name: &str, value: &dyn Debug): &mut Self
    fn finish(&mut self): Result[(), IoError] ! IO
}
```

### 19.3 Format Intrinsics

```ferrum
format("...")       : String          // format to owned String
print("...")        : ()  ! IO        // to stdout, no newline
println("...")      : ()  ! IO        // to stdout, with newline
eprint("...")       : ()  ! IO        // to stderr, no newline
eprintln("...")     : ()  ! IO        // to stderr, with newline
write(w, "...")     : Result[(), IoError] ! IO   // to any Write
writeln(w, "...")   : Result[(), IoError] ! IO

// These are compiler intrinsics, not user-definable.
// Format string must be a string literal.
// Argument count and types are verified at compile time.
// No runtime format string parsing. No format string injection.
```

---

## 20. time — Time and Duration

### 20.1 Duration

```ferrum
// Duration — unsigned, nanosecond precision, no overflow for reasonable values
type Duration {
    secs:  u64,
    nanos: u32 where value < 1_000_000_000,

    invariant // enforced

    const ZERO:    Self = Duration { secs: 0, nanos: 0 }
    const MAX:     Self = Duration { secs: u64.MAX, nanos: 999_999_999 }
    const SECOND:  Self = Duration.from_secs(1)
    const MINUTE:  Self = Duration.from_secs(60)
    const HOUR:    Self = Duration.from_secs(3600)
    const DAY:     Self = Duration.from_secs(86400)
}

impl Duration {
    fn from_secs(secs: u64): Self
    fn from_millis(millis: u64): Self
    fn from_micros(micros: u64): Self
    fn from_nanos(nanos: u128): Self
    fn from_secs_f64(secs: f64): Self
    fn from_secs_f32(secs: f32): Self

    fn as_secs(&self): u64
    fn as_millis(&self): u128
    fn as_micros(&self): u128
    fn as_nanos(&self): u128
    fn as_secs_f64(&self): f64
    fn as_secs_f32(&self): f32
    fn subsec_millis(&self): u32
    fn subsec_micros(&self): u32
    fn subsec_nanos(&self): u32

    fn checked_add(&self, rhs: Self): Option[Self]
    fn checked_sub(&self, rhs: Self): Option[Self]
    fn checked_mul(&self, rhs: u32): Option[Self]
    fn checked_div(&self, rhs: u32): Option[Self]
    fn saturating_add(&self, rhs: Self): Self
    fn saturating_sub(&self, rhs: Self): Self
    fn mul_f64(&self, rhs: f64): Self
    fn div_f64(&self, rhs: f64): Self
    fn div_duration_f64(&self, rhs: Self): f64

    fn is_zero(&self): bool
}

// Ops: Duration + Duration, Duration - Duration (checked), Duration * u32, Duration / u32
```

### 20.1.1 Absolute Time Scheduling

Relative delays (`sleep(duration)`) cause drift when work takes time. Absolute delays (`sleep_until(instant)`) don't.

```ferrum
// WRONG: drift accumulates
fn periodic_task_bad(period: Duration) ! IO {
    loop {
        do_work()
        sleep(period)  // relative delay — doesn't account for work time
    }
}
// If do_work() takes 10ms and period is 100ms:
// - Iteration 1: work at t=0, sleep 100ms, wake at t=110
// - Iteration 2: work at t=110, sleep 100ms, wake at t=220
// - Drift: 10ms per iteration!

// CORRECT: absolute time scheduling
fn periodic_task_good(period: Duration) ! IO {
    let mut next = Instant.now()
    loop {
        do_work()
        next = next + period
        sleep_until(next)  // absolute delay — no drift
    }
}

/// Sleep until an absolute instant.
/// If the instant is in the past, returns immediately.
fn sleep_until(deadline: Instant): () ! IO

/// Sleep for a duration (relative).
fn sleep(duration: Duration): () ! IO

// Async versions
fn sleep_until_async(deadline: Instant): () ! Async
fn sleep_async(duration: Duration): () ! Async
```

### 20.1.2 PeriodicTimer — Drift-Free Periodic Tasks

```ferrum
/// A periodic timer that doesn't drift.
type PeriodicTimer {
    period: Duration,
    next: Instant,
}

impl PeriodicTimer {
    /// Creates a new periodic timer starting now.
    fn new(period: Duration): Self {
        PeriodicTimer {
            period,
            next: Instant.now() + period,
        }
    }

    /// Creates a periodic timer with a specific start time.
    fn starting_at(start: Instant, period: Duration): Self {
        PeriodicTimer {
            period,
            next: start + period,
        }
    }

    /// Waits until the next period boundary.
    /// Returns the number of periods that have elapsed (usually 1, but
    /// may be more if the system is overloaded).
    fn wait(&mut self): u64 ! IO {
        let now = Instant.now()
        if now >= self.next {
            // We're late — calculate how many periods we missed
            let elapsed = now.duration_since(self.next)
            let missed = (elapsed.as_nanos() / self.period.as_nanos()) as u64
            self.next = self.next + self.period * (missed + 1)
            missed + 1
        } else {
            sleep_until(self.next)
            self.next = self.next + self.period
            1
        }
    }

    /// Returns the next deadline.
    fn next_deadline(&self): Instant {
        self.next
    }

    /// Returns the period.
    fn period(&self): Duration {
        self.period
    }
}

// Usage
fn control_loop() ! IO {
    let mut timer = PeriodicTimer.new(Duration.from_millis(10))  // 100 Hz

    loop {
        let missed = timer.wait()
        if missed > 1 {
            warn!("Missed {} periods", missed - 1)
        }
        read_sensors()
        compute_control()
        write_actuators()
    }
}
```

### 20.2 Instant (Monotonic Clock)

```ferrum
// Instant — monotonic, opaque, for measuring elapsed time
// NOT a wall-clock. NOT convertible to a date/time.
type Instant { ... }

impl Instant {
    fn now(): Self ! IO                       // current monotonic time
    fn elapsed(&self): Duration ! IO          // time since this instant
    fn duration_since(&self, earlier: Instant): Duration
        requires self >= earlier
    fn checked_duration_since(&self, earlier: Instant): Option[Duration]
    fn saturating_duration_since(&self, earlier: Instant): Duration
    fn checked_add(&self, d: Duration): Option[Instant]
    fn checked_sub(&self, d: Duration): Option[Instant]
}

// Ops: Instant - Instant (Duration), Instant + Duration, Instant - Duration
```

### 20.3 Timestamp (Fundamental Time)

```ferrum
// Timestamp — a point in time, calendar-agnostic
// Stored as nanoseconds since a platform-defined epoch.
// The epoch is whatever the local OS provides; what matters is
// that we can query its offset from JDN 0.0 for calendar conversion.
// 64-bit signed nanoseconds = ±292 years from epoch.

type Timestamp {
    nanos: i64,
}

impl Timestamp {
    fn now(): Self ! IO

    // Arithmetic
    fn duration_since(&self, earlier: Self): Duration
    fn checked_add(&self, d: Duration): Option[Self]
    fn checked_sub(&self, d: Duration): Option[Self]
    fn elapsed(&self): Duration ! IO

    // Julian Day Number — the universal interchange format
    // JDN 0.0 = noon on January 1, 4713 BCE (Julian proleptic calendar)
    // These use the platform-provided epoch offset.
    fn to_jdn(&self): f64
    fn from_jdn(jdn: f64): Self

    // Unix timestamp (for interop)
    fn to_unix(&self): i64           // seconds since Unix epoch
    fn to_unix_nanos(&self): i128    // nanoseconds since Unix epoch
    fn from_unix(secs: i64): Self
    fn from_unix_nanos(nanos: i128): Self

    // TAI offset (leap seconds)
    fn tai_offset(&self): Duration   // difference from TAI at this instant
}

// For historical dates or far future (±10^22 years)
type Timestamp128 {
    nanos: i128,
}

// Platform trait for time — see ferrum-stdlib-platform.md
trait TimePlatform {
    fn monotonic_now(&self): Instant
    fn timestamp_now(&self): Timestamp

    /// The JDN value at the platform's epoch (nanos = 0).
    /// This is all we need to convert to/from any calendar.
    fn epoch_jdn(&self): f64
}
```

**Why JDN as the interchange format?**

Julian Day Number is the only time scale that:
- Is used by astronomers, historians, and calendar researchers worldwide
- Has no embedded calendar system (it is just a continuous day count)
- Provides a universal reference for converting between any calendar
- Is continuous (no leap seconds, no daylight saving, no missing days)

The platform tells us the JDN at its epoch. Any calendar converts through JDN.

### 20.4 Calendar Systems

Calendars are **not baked into the standard library**. The `time.calendar` module provides a trait for calendar implementations; specific calendars are separate modules.

```ferrum
// The calendar trait — implemented by each calendar system
trait Calendar {
    type Date: CalendarDate
    type DateTime: CalendarDateTime

    /// Convert a timestamp to this calendar's date representation
    fn date_from_timestamp(t: Timestamp, tz: &dyn Timezone): Self.Date

    /// Convert this calendar's date to a timestamp
    fn timestamp_from_date(d: &Self.Date, tz: &dyn Timezone): Timestamp

    /// Calendar name for display/serialization
    fn name(&self): &str
}

trait CalendarDate {
    /// Julian Day Number at midnight of this date
    fn to_jdn(&self): f64

    /// Create from Julian Day Number
    fn from_jdn(jdn: f64): Self

    /// Format according to this calendar's conventions
    fn format(&self, fmt: &str): String
}

trait CalendarDateTime: CalendarDate {
    fn hour(&self): u8
    fn minute(&self): u8
    fn second(&self): u8
    fn nanosecond(&self): u32
}

// Timezone trait — offset from UTC at a given instant
trait Timezone {
    fn offset_at(&self, t: Timestamp): Duration
    fn name(&self): &str
}

// Built-in timezones (these ARE in std, they're not calendar-specific)
struct Utc           // UTC, no offset
struct Tai           // International Atomic Time
struct Local         // System local timezone
type FixedOffset { seconds: i32 }  // Fixed UTC offset
```

### 20.5 Calendar Modules

Each calendar is a separate module. The standard library provides Gregorian as the most common; others are available as packages.

```ferrum
// std.time.calendar.gregorian — the most common, but not privileged
mod gregorian {
    type Date {
        year: i32,      // astronomical year (0 = 1 BCE, -1 = 2 BCE)
        month: u8,      // 1–12
        day: u8,        // 1–31
    }

    type DateTime {
        date: Date,
        hour: u8,       // 0–23
        minute: u8,     // 0–59
        second: u8,     // 0–60 (60 for leap second)
        nanosecond: u32,
        timezone: &dyn Timezone,
    }

    impl Calendar for Gregorian { ... }

    // Convenience for the common case
    fn now(): DateTime ! IO {
        DateTime.from_timestamp(Timestamp.now(), &Local)
    }

    enum Weekday { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday }
    enum Month { January, February, ..., December }
}

// Other calendars — separate packages, same trait
// time.calendar.islamic     — Hijri calendar
// time.calendar.hebrew      — Jewish calendar
// time.calendar.chinese     — Chinese lunisolar calendar
// time.calendar.japanese    — Japanese imperial calendar
// time.calendar.persian     — Solar Hijri calendar
// time.calendar.julian      — Julian calendar (proleptic)
// time.calendar.coptic      — Coptic calendar
// time.calendar.ethiopic    — Ethiopian calendar
// time.calendar.buddhist    — Buddhist calendar
// time.calendar.indian      — Indian national calendar
```

### 20.6 Converting Between Calendars

All calendars convert through JDN, making inter-calendar conversion trivial:

```ferrum
import time.calendar.gregorian.{Date as GregorianDate}
import time.calendar.hebrew.{Date as HebrewDate}
import time.calendar.islamic.{Date as HijriDate}

// Any date → JDN → any other date
let greg = GregorianDate { year: 2024, month: 12, day: 25 }
let jdn = greg.to_jdn()

let hebrew = HebrewDate.from_jdn(jdn)   // 23 Kislev 5785
let hijri = HijriDate.from_jdn(jdn)     // 23 Jumada al-Thani 1446

// Or via Timestamp for full datetime with timezone
let ts = Timestamp.from_jdn(jdn)
let hebrew_dt = hebrew.DateTime.from_timestamp(ts, &Utc)
```

**Why this design?**

1. **No cultural imperialism.** The Gregorian calendar is not privileged in the type system. A Hebrew date is as first-class as a Gregorian date.

2. **Correctness.** Calendar arithmetic is notoriously complex (leap years, leap months, variable-length months, intercalation rules). Each calendar module implements its own rules correctly.

3. **Interoperability.** JDN is the universal interchange format. Any calendar that can convert to/from JDN can interoperate with any other.

4. **Minimal core.** The standard library provides `Timestamp`, `Duration`, `Instant`, the `Calendar` trait, and Gregorian (because it's the most common for computing). Everything else is packages.

---

## 21. sync — Synchronization

See §13 (Concurrency) in the Language Reference for channels. This section covers shared-state primitives.

```ferrum
// Mutex — mutual exclusion
type Mutex[T]  given [A: Allocator]

impl[T] Mutex[T] {
    fn new(val: T): Self
    fn lock(&self): Result[MutexGuard[T], PoisonError[MutexGuard[T]]]  ! Sync
    fn try_lock(&self): Result[MutexGuard[T], TryLockError[MutexGuard[T]]]
    fn into_inner(self): Result[T, PoisonError[T]]
    fn is_poisoned(&self): bool
    fn clear_poison(&self)
        // NOTE: "poisoned" means a thread panicked while holding the lock.
        // clear_poison lets you recover the data. Deliberate, not an accident.
}

type MutexGuard[T]
impl[T] MutexGuard[T] {
    // Deref/DerefMut to T
    fn unlock(self): &Mutex[T]   // explicit unlock without dropping
}

// RwLock — multiple readers, one writer
type RwLock[T]  given [A: Allocator]
impl[T] RwLock[T] {
    fn new(val: T): Self
    fn read(&self): Result[RwLockReadGuard[T], PoisonError<...>]  ! Sync
    fn write(&self): Result[RwLockWriteGuard[T], PoisonError<...>]  ! Sync
    fn try_read(&self): Result[RwLockReadGuard[T], TryLockError<...>]
    fn try_write(&self): Result[RwLockWriteGuard[T], TryLockError<...>]
    fn into_inner(self): Result[T, PoisonError[T]]
    fn is_poisoned(&self): bool
    fn get_mut(&mut self): Result[&mut T, PoisonError[&mut T]]
}

// Condvar — condition variable, always paired with a Mutex
type Condvar  given [A: Allocator]
impl Condvar {
    fn new(): Self
    fn wait[T](&self, guard: MutexGuard[T]): Result[MutexGuard[T], PoisonError<...>]  ! Sync
    fn wait_while[T](&self, guard: MutexGuard[T], cond: fn(&mut T): bool)
        : Result[MutexGuard[T], PoisonError<...>]  ! Sync
    fn wait_timeout[T](&self, guard: MutexGuard[T], dur: Duration)
        : Result[(MutexGuard[T], WaitTimeoutResult), PoisonError<...>]  ! Sync
    fn notify_one(&self)
    fn notify_all(&self)
}

// Once — run initialization exactly once
type Once
impl Once {
    fn new(): Self
    fn call_once(&self, f: fn()) ! Sync
    fn call_once_force(&self, f: fn(&OnceState)) ! Sync
    fn is_completed(&self): bool
}

// LazyLock — lazily initialized static
type LazyLock[T]  given [A: Allocator]
impl[T] LazyLock[T] {
    fn new(init: fn(): T): Self
    fn get(&self): &T  ! Sync   // initializes on first call
    fn force(this: &Self): &T ! Sync
}

// OnceLock — lazily initialized, externally set
type OnceLock[T]
impl[T] OnceLock[T] {
    fn new(): Self
    fn set(&self, value: T): Result[(), T]  ! Sync   // Err if already set
    fn get(&self): Option[&T]
    fn get_or_init(&self, f: fn(): T): &T  ! Sync
    fn get_or_try_init[E](&self, f: fn(): Result[T, E]): Result[&T, E]  ! Sync
    fn into_inner(self): Option[T]
}

// Barrier
type Barrier  given [A: Allocator]
impl Barrier {
    fn new(n: usize): Self
    fn wait(&self): BarrierWaitResult  ! Sync
}

// Semaphore
type Semaphore  given [A: Allocator]
impl Semaphore {
    fn new(permits: usize): Self
    fn acquire(&self): SemaphorePermit  ! Sync
    fn try_acquire(&self): Option[SemaphorePermit]
    fn acquire_many(&self, n: usize): SemaphorePermit  ! Sync
    fn add_permits(&self, n: usize)
    fn available_permits(&self): usize
}

// Atomic types (see Language Reference §13.6)
type Atomic[T: AtomicSafe]
// AtomicSafe: bool, u8..u64, i8..i64, usize, isize, *const T, *mut T
impl[T: AtomicSafe] Atomic[T] {
    fn new(val: T): Self
    fn load(&self, ord: Ordering): T
    fn store(&self, val: T, ord: Ordering)
    fn swap(&self, val: T, ord: Ordering): T
    fn compare_exchange(
        &self,
        current: T, new: T,
        success: Ordering, failure: Ordering,
    ): Result[T, T]
    fn compare_exchange_weak(
        &self,
        current: T, new: T,
        success: Ordering, failure: Ordering,
    ): Result[T, T]
    fn fetch_add(&self, val: T, ord: Ordering): T   where T: Integer
    fn fetch_sub(&self, val: T, ord: Ordering): T   where T: Integer
    fn fetch_and(&self, val: T, ord: Ordering): T   where T: Integer
    fn fetch_or(&self, val: T, ord: Ordering): T    where T: Integer
    fn fetch_xor(&self, val: T, ord: Ordering): T   where T: Integer
    fn fetch_max(&self, val: T, ord: Ordering): T   where T: Ord + Integer
    fn fetch_min(&self, val: T, ord: Ordering): T   where T: Ord + Integer
    fn fetch_update[F: Fn(T):Option[T]](&self, fetch: Ordering, set: Ordering, f: F): Result[T, T]
    fn get_mut(&mut self): &mut T
    fn into_inner(self): T
}

enum Ordering { Relaxed, Acquire, Release, AcqRel, SeqCst }

// Fence
fn fence(ord: Ordering)
fn compiler_fence(ord: Ordering)
```

### 21.1 Protected Objects

A protected object is a monitor with automatic mutual exclusion. Functions allow concurrent reads, procedures require exclusive writes, and entries block until a barrier condition is true.

```ferrum
/// A protected object with automatic mutual exclusion.
type Protected[T] {
    data: UnsafeCell[T],
    lock: RwLock,
    condvar: Condvar,
}

impl[T] Protected[T] {
    /// Creates a new protected object.
    fn new(value: T): Self

    /// Calls a function with shared read access.
    /// Multiple readers can execute concurrently.
    fn read[R](&self, f: fn(&T): R): R ! Sync

    /// Calls a procedure with exclusive write access.
    /// No other readers or writers can execute concurrently.
    fn write[R](&self, f: fn(&mut T): R): R ! Sync

    /// Calls an entry with exclusive access, blocking until barrier is true.
    /// The barrier is re-evaluated after each write.
    fn entry[R](&self, barrier: fn(&T): bool, f: fn(&mut T): R): R ! Sync

    /// Try entry: returns None immediately if barrier is false.
    fn try_entry[R](&self, barrier: fn(&T): bool, f: fn(&mut T): R): Option[R] ! Sync

    /// Timed entry: returns None if timeout expires before barrier is true.
    fn entry_timeout[R](
        &self,
        barrier: fn(&T): bool,
        f: fn(&mut T): R,
        timeout: Duration,
    ): Option[R] ! Sync
}

// Example: Bounded Buffer (classic producer-consumer)
type BoundedBuffer[T, const N: usize] {
    data: [Option[T]; N],
    head: usize,
    tail: usize,
    count: usize,
}

impl[T, const N: usize] BoundedBuffer[T, N] {
    fn new(): Self {
        BoundedBuffer { data: [None; N], head: 0, tail: 0, count: 0 }
    }
    fn is_empty(&self): bool { self.count == 0 }
    fn is_full(&self): bool { self.count == N }
}

// Producer: blocks if buffer is full
fn put[T, const N: usize](buf: &Protected[BoundedBuffer[T, N]], item: T) ! Sync {
    buf.entry(
        |b| !b.is_full(),           // barrier: wait until not full
        |b| {
            b.data[b.tail] = Some(item)
            b.tail = (b.tail + 1) % N
            b.count += 1
        }
    )
}

// Consumer: blocks if buffer is empty
fn get[T, const N: usize](buf: &Protected[BoundedBuffer[T, N]]): T ! Sync {
    buf.entry(
        |b| !b.is_empty(),          // barrier: wait until not empty
        |b| {
            let item = b.data[b.head].take().unwrap()
            b.head = (b.head + 1) % N
            b.count -= 1
            item
        }
    )
}

// Non-blocking check
fn count[T, const N: usize](buf: &Protected[BoundedBuffer[T, N]]): usize ! Sync {
    buf.read(|b| b.count)   // read access, concurrent with other readers
}

// Why Protected objects beat raw mutexes:
// ✓ Lock scope is the closure — automatic release
// ✓ Read/write distinction — concurrent readers
// ✓ Barriers automatically re-check after writes
// ✓ No manual condvar.notify — system handles it
// ✓ Deadlock-free within a single protected object
```

### 21.2 select — Multi-Way Synchronization

Wait on multiple synchronization points and proceed with the first that becomes ready.

```ferrum
// select is a compiler intrinsic with special syntax.
// It cannot be user-defined.

// Example: receive from either channel
fn either(ch1: &Receiver[i32], ch2: &Receiver[String]): Either[i32, String] ! Sync {
    select {
        recv(ch1) -> n => Either.Left(n),
        recv(ch2) -> s => Either.Right(s),
    }
}

// Example: send with timeout
fn send_timeout(ch: &Sender[Data], data: Data, dur: Duration): bool ! Sync {
    select {
        send(ch, data) => true,
        timeout(dur) => false,
    }
}

// Example: non-blocking try
fn try_recv(ch: &Receiver[i32]): Option[i32] ! Sync {
    select {
        recv(ch) -> n => Some(n),
        default => None,
    }
}

// Guarded alternatives — conditionally enabled
fn process_commands(
    cmd_ch: &Receiver[Command],
    data_ch: &Receiver[Data],
    accepting_data: bool,
) ! Sync {
    select {
        recv(cmd_ch) -> cmd => handle_command(cmd),

        // Guard: only accept data if flag is true
        recv(data_ch) -> data if accepting_data => handle_data(data),

        // Terminate if both channels are closed
        closed(cmd_ch) && closed(data_ch) => return,
    }
}

// Select with protected object entries
fn producer_consumer(
    input: &Protected[InputBuffer],
    output: &Protected[OutputBuffer],
) ! Sync {
    select {
        // Wait for input to be non-empty
        entry(input, |b| !b.is_empty()) -> item => {
            let processed = process(item)
            output.entry(|b| !b.is_full(), |b| b.push(processed))
        }

        // Or timeout after 1 second
        timeout!(Duration.from_secs(1)) => {
            log!("No input for 1 second")
        }
    }
}
```

### 21.3 Real-Time Synchronization

For hard real-time systems with deterministic scheduling requirements.

```ferrum
// In sync.realtime

/// A mutex with priority ceiling protocol.
/// Prevents priority inversion by temporarily raising the holder's priority.
type CeilingMutex[T] {
    data: UnsafeCell[T],
    ceiling: Priority,
    state: AtomicState,
}

impl[T] CeilingMutex[T] {
    /// Creates a mutex with the specified priority ceiling.
    /// The ceiling must be >= the highest priority task that will access it.
    fn new(value: T, ceiling: Priority): Self

    /// Locks the mutex, raising the current task's priority to the ceiling.
    fn lock(&self): CeilingGuard[T] ! Sync
        requires current_priority() <= self.ceiling

    /// Tries to lock without blocking.
    fn try_lock(&self): Option[CeilingGuard[T]] ! Sync
}

type CeilingGuard[T] {
    mutex: &CeilingMutex[T],
    old_priority: Priority,
}

impl[T] Drop for CeilingGuard[T] {
    fn drop(&mut self) {
        // Restore original priority
        set_current_priority(self.old_priority)
        // Unlock
        self.mutex.unlock()
    }
}

impl[T] Deref for CeilingGuard[T] {
    type Target = T
    fn deref(&self): &T { unsafe { &*self.mutex.data.get() } }
}

impl[T] DerefMut for CeilingGuard[T] {
    fn deref_mut(&mut self): &mut T { unsafe { &mut *self.mutex.data.get() } }
}

// Execution time monitoring
/// CPU time consumed by the current task.
fn task_cpu_time(): Duration ! IO

/// A budget for CPU time.
type ExecutionTimeBudget {
    remaining: Duration,
}

impl ExecutionTimeBudget {
    fn new(budget: Duration): Self

    /// Checks remaining budget.
    fn remaining(&self): Duration ! IO

    /// Returns true if budget is exhausted.
    fn exhausted(&self): bool ! IO

    /// Blocks until budget is exhausted.
    fn wait_exhausted(&self) ! IO + Sync
}

/// Timing event — fires at a specific instant.
type TimingEvent {
    fn new(): Self
    fn set(&self, at: Instant) ! IO
    fn cancel(&self) ! IO
    fn wait(&self) ! IO + Sync       // blocks until event fires
    fn is_set(&self): bool
}
```

### 21.4 Ravenscar Profile (Safety-Critical)

A restricted concurrency model for high-integrity systems. Ravenscar restrictions enable static analysis and certification.

```ferrum
// In sync.ravenscar

/// Marker trait for Ravenscar-compliant types.
/// Ravenscar restrictions:
/// - No dynamic task creation after elaboration
/// - No task termination (tasks run forever or program ends)
/// - Single entry per protected object
/// - No select statements with multiple alternatives
/// - No abort
/// - No relative delays (only delay until)
/// - Fixed priority scheduling
trait RavenscarSafe {}

/// A Ravenscar-compliant task.
/// Must be created at elaboration time (static).
type RavenscarTask {
    priority: Priority,
    stack_size: usize,
    entry: fn() -> never,   // tasks don't return
}

/// A Ravenscar-compliant protected object.
/// Single entry only.
type RavenscarProtected[T] {
    data: UnsafeCell[T],
    ceiling: Priority,
    entry_barrier: fn(&T) -> bool,
}

impl[T] RavenscarProtected[T] {
    /// Read access (concurrent).
    fn read[R](&self, f: fn(&T): R): R ! Sync

    /// Write access (exclusive).
    fn write[R](&self, f: fn(&mut T): R): R ! Sync

    /// The single entry (blocks on barrier).
    fn entry[R](&self, f: fn(&mut T): R): R ! Sync
}

/// Compile-time verification that a module is Ravenscar-compliant.
@ravenscar
mod flight_control {
    // Compiler enforces Ravenscar restrictions in this module
    // - No dynamic allocation
    // - No dynamic task creation
    // - All tasks have static priority
    // - etc.
}
```

---

## 22. process — Process Management

```ferrum
// Spawn a child process
type Command  given [A: Allocator]

impl Command {
    fn new(program: impl AsRef[OsStr]): Self
    fn arg(&mut self, arg: impl AsRef[OsStr]): &mut Self
    fn args(&mut self, args: impl IntoIterator[Item=impl AsRef[OsStr]]): &mut Self
    fn env(&mut self, key: impl AsRef[OsStr], val: impl AsRef[OsStr]): &mut Self
    fn env_clear(&mut self): &mut Self
    fn env_remove(&mut self, key: impl AsRef[OsStr]): &mut Self
    fn current_dir(&mut self, dir: impl AsRef[Path]): &mut Self
    fn stdin(&mut self, cfg: Stdio): &mut Self
    fn stdout(&mut self, cfg: Stdio): &mut Self
    fn stderr(&mut self, cfg: Stdio): &mut Self
    fn uid(&mut self, id: u32): &mut Self   // Unix only
    fn gid(&mut self, id: u32): &mut Self   // Unix only
    fn spawn(&mut self): Result[Child, IoError] ! IO
    fn output(&mut self): Result[Output, IoError] ! IO  // waits for completion
    fn status(&mut self): Result[ExitStatus, IoError] ! IO

    // Async versions
    fn spawn_async(&mut self): Result[AsyncChild, IoError] ! IO + Async
}

enum Stdio {
    Inherit,
    Null,
    Piped,
    File(File),
    Fd(Fd),  // posix only
}

type Child {
    pub stdin:  Option[ChildStdin]
    pub stdout: Option[ChildStdout]
    pub stderr: Option[ChildStderr]

    fn id(&self): u32
    fn wait(&mut self): Result[ExitStatus, IoError] ! IO
    fn wait_with_output(self): Result[Output, IoError] ! IO
    fn try_wait(&mut self): Result[Option[ExitStatus], IoError]
    fn kill(&mut self): Result[(), IoError] ! IO
    fn send_signal(&mut self, sig: Signal): Result[(), IoError] ! IO
}

type ExitStatus { ... }
impl ExitStatus {
    fn success(&self): bool
    fn code(&self): Option[i32]   // None if terminated by signal
    fn signal(&self): Option[i32]  // None if exited normally
}

type Output {
    pub status: ExitStatus,
    pub stdout: Vec[u8],
    pub stderr: Vec[u8],
}
```

---

## 23. env — Environment

```ferrum
// Command-line arguments
fn args(): Args ! IO           // iterator over String arguments
fn args_os(): ArgsOs ! IO      // iterator over OsString (for non-UTF-8)

// Structured argument parsing is in a separate crate (clap/pico-args style)
// The stdlib provides only raw access to argv

// Environment variables
fn var(key: &str): Result[String, VarError] ! IO
fn var_os(key: &str): Option[OsString] ! IO
fn vars(): Vars ! IO       // iterator over (String, String)
fn vars_os(): VarsOs ! IO  // iterator over (OsString, OsString)

// NOTE: set_var and remove_var exist but are in sys.posix, not env.
// They mutate the process environment globally and are not thread-safe.
// Accessing them from here requires acknowledging the hazard.

enum VarError {
    NotPresent,
    NotUnicode(OsString),
}

// Paths
fn current_dir(): Result[PathBuf, IoError] ! IO
fn current_exe(): Result[PathBuf, IoError] ! IO
fn temp_dir(): PathBuf ! IO
fn home_dir(): Option[PathBuf] ! IO
```

---

## 24. backtrace — Stack Trace Capture

```ferrum
// Backtrace — captured stack trace
type Backtrace {
    inner: BacktraceInner,
}

enum BacktraceInner {
    Unsupported,
    Disabled,
    Captured(Vec[BacktraceFrame]),
}

impl Backtrace {
    /// Captures a backtrace at the current location.
    fn capture(): Self ! IO {
        match BacktraceConfig.current() {
            BacktraceConfig.Disabled => Backtrace { inner: BacktraceInner.Disabled },
            BacktraceConfig.Enabled => {
                let frames = capture_frames()
                Backtrace { inner: BacktraceInner.Captured(frames) }
            }
        }
    }

    /// Forces capture regardless of configuration.
    fn force_capture(): Self ! IO

    /// Returns an empty backtrace.
    fn disabled(): Self {
        Backtrace { inner: BacktraceInner.Disabled }
    }

    /// Returns the status of this backtrace.
    fn status(&self): BacktraceStatus {
        match self.inner {
            BacktraceInner.Unsupported => BacktraceStatus.Unsupported,
            BacktraceInner.Disabled => BacktraceStatus.Disabled,
            BacktraceInner.Captured(_) => BacktraceStatus.Captured,
        }
    }

    /// Returns an iterator over frames.
    fn frames(&self): &[BacktraceFrame]
}

enum BacktraceStatus {
    /// Backtraces are not supported on this platform.
    Unsupported,
    /// Backtraces are supported but disabled.
    Disabled,
    /// A backtrace was captured.
    Captured,
}

/// Configuration for backtrace capture.
/// Controlled by FERRUM_BACKTRACE environment variable:
///   0 or unset: disabled
///   1: enabled
///   full: enabled with full symbols
enum BacktraceConfig {
    Disabled,
    Enabled,
}

impl BacktraceConfig {
    fn current(): Self {
        // Reads FERRUM_BACKTRACE once per process, caches result
        static CONFIG: OnceLock[BacktraceConfig] = OnceLock.new()
        *CONFIG.get_or_init(|| {
            match env.var("FERRUM_BACKTRACE") {
                Ok("1") | Ok("full") => BacktraceConfig.Enabled,
                _ => BacktraceConfig.Disabled,
            }
        })
    }
}

// BacktraceFrame — a single frame in the stack trace
type BacktraceFrame {
    ip: usize,          // instruction pointer
    symbol_address: Option[usize],
}

impl BacktraceFrame {
    /// Returns the instruction pointer.
    fn ip(&self): usize

    /// Returns the symbol address (function start).
    fn symbol_address(&self): Option[usize]

    /// Resolves symbols for this frame.
    fn resolve(&self): Vec[BacktraceSymbol] ! IO
}

// BacktraceSymbol — resolved symbol information
type BacktraceSymbol {
    name: Option[String],
    filename: Option[PathBuf],
    lineno: Option[u32],
    colno: Option[u32],
}

impl BacktraceSymbol {
    fn name(&self): Option[&str]
    fn filename(&self): Option[&Path]
    fn lineno(&self): Option[u32]
    fn colno(&self): Option[u32]
}

impl Display for Backtrace {
    fn fmt(&self, f: &mut Formatter): Result[(), FmtError] {
        match self.inner {
            BacktraceInner.Unsupported => write(f, "unsupported backtrace"),
            BacktraceInner.Disabled => write(f, "disabled backtrace"),
            BacktraceInner.Captured(ref frames) => {
                for (i, frame) in frames.iter().enumerate() {
                    writeln(f, "{:4}: {:?}", i, frame)?
                }
                Ok(())
            }
        }
    }
}

impl Debug for Backtrace {
    fn fmt(&self, f: &mut Formatter): Result[(), FmtError] {
        Display.fmt(self, f)
    }
}
```

### 24.1 Backtrace Integration

```ferrum
// With the error module
@derive(Debug, Error)
@error("database query failed: {query}")
type DbError {
    query: String,

    @source
    source: IoError,

    @backtrace
    backtrace: Backtrace,  // Captured automatically
}

// Manual capture
fn risky_operation(): Result[(), MyError] ! IO {
    if something_bad() {
        return Err(MyError {
            message: "operation failed",
            backtrace: Backtrace.capture(),
        })
    }
    Ok(())
}

// Printing with backtrace
fn main() ! IO {
    match run() {
        Ok(()) => (),
        Err(e) => {
            eprintln("Error: {}", e)
            if let Some(bt) = e.backtrace() {
                if bt.status() == BacktraceStatus.Captured {
                    eprintln("\nBacktrace:\n{}", bt)
                }
            }
        }
    }
}
```

---

## 25. log — Logging Facade

A logging facade that decouples log generation from log consumption.

```ferrum
// Log levels
enum Level {
    Error = 1,
    Warn = 2,
    Info = 3,
    Debug = 4,
    Trace = 5,
}

impl Level {
    fn as_str(&self): &str {
        match self {
            Level.Error => "ERROR",
            Level.Warn => "WARN",
            Level.Info => "INFO",
            Level.Debug => "DEBUG",
            Level.Trace => "TRACE",
        }
    }
}

impl Ord for Level {
    fn cmp(&self, other: &Self): Ordering {
        (*self as u8).cmp(&(*other as u8))
    }
}

// LevelFilter — for filtering log messages
enum LevelFilter {
    Off,
    Error,
    Warn,
    Info,
    Debug,
    Trace,
}

impl LevelFilter {
    fn from_level(level: Level): Self

    /// Returns true if a message at this level would be logged.
    fn enabled(&self, level: Level): bool {
        match self {
            LevelFilter.Off => false,
            _ => level as u8 <= *self as u8,
        }
    }
}

// Record — a single log entry
type Record {
    level: Level,
    target: &str,           // typically module path
    message: String,
    file: Option[&str],
    line: Option[u32],
    module_path: Option[&str],
    key_values: Vec[(String, String)],
}

impl Record {
    fn level(&self): Level
    fn target(&self): &str
    fn message(&self): &str
    fn file(&self): Option[&str]
    fn line(&self): Option[u32]
    fn module_path(&self): Option[&str]

    /// Returns key-value pairs for structured logging.
    fn key_values(&self): &[(String, String)]
}

// Logger trait — implement to create a logging backend
trait Log: Send + Sync {
    /// Determines if a log message at this level/target should be logged.
    fn enabled(&self, level: Level, target: &str): bool

    /// Logs the record.
    fn log(&self, record: &Record)

    /// Flushes any buffered records.
    fn flush(&self)
}

// Global logger management
static LOGGER: OnceLock[&dyn Log] = OnceLock.new()
static MAX_LEVEL: AtomicU8 = AtomicU8.new(LevelFilter.Off as u8)

/// Sets the global logger. Can only be called once.
fn set_logger(logger: &'static dyn Log): Result[(), SetLoggerError] {
    LOGGER.set(logger).map_err(|_| SetLoggerError)
}

/// Sets the maximum log level.
fn set_max_level(level: LevelFilter) {
    MAX_LEVEL.store(level as u8, Ordering.Release)
}

/// Returns the current maximum log level.
fn max_level(): LevelFilter {
    // ... atomic load
}

/// Returns the current logger, or a no-op logger if none set.
fn logger(): &dyn Log {
    LOGGER.get().unwrap_or(&NOP_LOGGER)
}

type SetLoggerError {}

impl Display for SetLoggerError {
    fn fmt(&self, f: &mut Formatter): Result[(), FmtError] {
        write(f, "logger already set")
    }
}

impl Error for SetLoggerError {}

// Nop logger
struct NopLogger {}
impl Log for NopLogger {
    fn enabled(&self, _: Level, _: &str): bool { false }
    fn log(&self, _: &Record) {}
    fn flush(&self) {}
}
static NOP_LOGGER: NopLogger = NopLogger {}

// Logging intrinsics — these are compiler intrinsics that capture source location
// They look like functions but the compiler inserts file/line/module automatically

fn error(msg: &str) ! IO      // log at error level
fn warn(msg: &str) ! IO       // log at warn level
fn info(msg: &str) ! IO       // log at info level
fn debug(msg: &str) ! IO      // log at debug level
fn trace(msg: &str) ! IO      // log at trace level

fn log(level: Level, msg: &str) ! IO   // log at specified level

// These intrinsics:
// - Capture source location (file, line, module) at the call site
// - Check the log level before formatting (no cost if level is disabled)
// - Cannot be user-defined (compiler knows about them)

// Example usage:
fn process_request(req: &Request) ! IO {
    debug(format("Processing request: {}", req.id))
    // ...
    if error_condition {
        error(format("Failed to process request {}: {}", req.id, err))
    }
}

// Structured logging — use a builder pattern
fn log_kv(level: Level, msg: &str, pairs: &[(String, String)]) ! IO

// Example:
fn handle_connection(conn: &Connection) ! IO {
    log_kv(Level.Info, "Connection established", &[
        ("conn_id".to_string(), conn.id.to_string()),
        ("peer_addr".to_string(), conn.peer_addr.to_string()),
    ])
}

// Standard key names — use these for consistency across the entire stack
//
// Consistent naming enables log aggregation without custom parsing.
// These are conventions, not enforced by the type system.
mod log_keys {
    // Connection and networking
    const CONN_ID:      &str = "conn_id"       // unique connection identifier
    const PEER_ADDR:    &str = "peer_addr"     // remote IP:port
    const LOCAL_ADDR:   &str = "local_addr"    // local IP:port
    const HOSTNAME:     &str = "hostname"      // DNS hostname
    const PORT:         &str = "port"          // port number

    // TLS
    const TLS_VERSION:  &str = "tls_version"   // "1.2", "1.3"
    const CIPHER_SUITE: &str = "cipher_suite"  // negotiated cipher
    const ALPN_PROTO:   &str = "alpn_proto"    // "h2", "http/1.1"
    const CERT_SUBJECT: &str = "cert_subject"  // certificate subject
    const DANE_RESULT:  &str = "dane_result"   // "validated", "no_tlsa", "failed"
    const DNSSEC:       &str = "dnssec"        // "validated", "insecure", "bogus"

    // HTTP
    const METHOD:       &str = "method"        // "GET", "POST", etc.
    const PATH:         &str = "path"          // request path
    const STATUS_CODE:  &str = "status_code"   // response status
    const USER_AGENT:   &str = "user_agent"    // client user agent

    // Performance
    const DURATION_MS:  &str = "duration_ms"   // operation duration in milliseconds
    const DURATION_US:  &str = "duration_us"   // operation duration in microseconds
    const BYTES_READ:   &str = "bytes_read"    // bytes read
    const BYTES_WRITTEN:&str = "bytes_written" // bytes written

    // Errors
    const ERROR_CODE:   &str = "error_code"    // numeric error code
    const ERROR_DOMAIN: &str = "error_domain"  // error domain/category
    const ERROR_MSG:    &str = "error_msg"     // human-readable error

    // Application
    const REQUEST_ID:   &str = "request_id"    // unique request identifier
    const USER_ID:      &str = "user_id"       // authenticated user
    const TRACE_ID:     &str = "trace_id"      // distributed trace ID
    const SPAN_ID:      &str = "span_id"       // span within trace
}
```

### 25.1 Logger Implementation Example

```ferrum
// Simple stderr logger
type StderrLogger {
    level: LevelFilter,
}

impl StderrLogger {
    fn new(level: LevelFilter): Self {
        StderrLogger { level }
    }

    fn init(level: LevelFilter) ! IO {
        static LOGGER: OnceLock[StderrLogger] = OnceLock.new()
        let logger = LOGGER.get_or_init(|| StderrLogger.new(level))
        set_logger(logger).ok()
        set_max_level(level)
    }
}

impl Log for StderrLogger {
    fn enabled(&self, level: Level, _target: &str): bool {
        self.level.enabled(level)
    }

    fn log(&self, record: &Record) {
        let now = Timestamp.now()
        eprintln(
            "[{} {} {}] {}",
            now.format("%Y-%m-%d %H:%M:%S"),
            record.level().as_str(),
            record.target(),
            record.message()
        )
    }

    fn flush(&self) {
        // stderr is unbuffered
    }
}

// Usage
fn main() ! IO {
    StderrLogger.init(LevelFilter.Debug)

    info!("Application starting")
    debug!("Debug info: {}", some_value)

    info_kv!("Request handled",
        method = "GET",
        path = "/api/users",
        status = 200,
        duration_ms = 42,
    )

    error!("Something went wrong: {}", err)
}
```

---

*End of fmt, time, sync, process, env modules — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
