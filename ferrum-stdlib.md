# Ferrum Standard Library

**Audience:** Standard library authors, compiler implementers, language users
**Companion:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## Table of Contents

This document has been split into multiple files for manageability:

1. [Design Principles](#1-design-principles)
2. [Library Layers](#2-library-layers)

**Module Documentation:**

- [ferrum-stdlib-core.md](ferrum-stdlib-core.md) — `core` — No-alloc primitives (ops, cmp, option, result, iter, mem, ptr, slice, str, ffi/CStr/OsStr, cell, pin, error, NonZero, Bounded/Discrete/Named, BoundedString)
- [ferrum-stdlib-alloc.md](ferrum-stdlib-alloc.md) — `alloc` + `collections` — Allocation-dependent types, CString/OsString, Cow, Rc/Weak, data structures
- [ferrum-stdlib-io.md](ferrum-stdlib-io.md) — `io`, `binary`, `text`, `fs` — The IO model, encoding, and filesystem
- [ferrum-stdlib-async-net.md](ferrum-stdlib-async-net.md) — `async`, `net`, `http`, `poll`, `lineproto`, `dgram`, `cbor` — Event loop, readiness polling, networking, HTTP stack, protocol frameworks, and binary serialization
- [ferrum-stdlib-sys.md](ferrum-stdlib-sys.md) — `sys`, `posix`, `windows` — Cross-platform sys, POSIX/Windows threading APIs
- [ferrum-stdlib-platform.md](ferrum-stdlib-platform.md) — Platform abstraction meta-spec (POSIX, Linux, BSD, Windows, WASI, Zephyr)
- [ferrum-stdlib-math.md](ferrum-stdlib-math.md) — `math`, `linalg`, `simd` — Mathematics, linear algebra, ElementaryFloat trait, and vectorization
- [ferrum-stdlib-numeric.md](ferrum-stdlib-numeric.md) — `alloc.bigint`, `alloc.bigdecimal`, `alloc.rational`, `core.complex` — Arbitrary precision numerics
- [ferrum-stdlib-util.md](ferrum-stdlib-util.md) — `fmt`, `time`, `sync`, `process`, `env`, `backtrace`, `log` — Formatting, time (with PeriodicTimer), synchronization (Protected objects, select, real-time, Ravenscar), logging, and utilities
- [ferrum-stdlib-crypto-testing.md](ferrum-stdlib-crypto-testing.md) — `crypto`, `testing` — Cryptographic primitives, test framework, and C mistakes reference

---

## 1. Design Principles

### What C Got Wrong (and We Fix)

Every design decision in this library is tested against this list.

| C mistake | Root cause | Ferrum fix |
|---|---|---|
| `errno` global error state | No return channel for errors | All errors are `Result` values |
| `strlen` must scan every time | Null-terminated strings | `&str` carries length always |
| `fread` returns `size_t`, EOF via `feof()` | Confused return type | `read` returns `Result[usize]`; EOF is `Ok(0)` or explicit `Eof` variant |
| Text/binary mode confusion on Windows | `FILE*` conflates both | Separate `TextReader`/`BinaryReader` types |
| `printf` format strings are untrusted | Strings as format specs | Format is a compile-time construct, never a runtime string |
| `malloc` returns uninitialized memory | Performance shortcut | `alloc` returns zeroed by default; `alloc_uninit` is explicit `unsafe` |
| `time_t` is 32-bit on some platforms | Premature optimization | `Instant` and `Timestamp` use 64-bit nanoseconds always |
| `signal` handlers can do almost nothing safely | Async-signal-safety | Signals convert to channel messages; no raw signal handlers |
| Locale-dependent `ctype.h` | Global mutable state | All locale operations are explicit and parameterized |
| `read` conflates EOF and 0-byte read | Overloaded return value | `ReadResult` is a proper enum |
| No ownership of returned pointers | C has no ownership | All allocated returns are owned values |
| Unsigned arithmetic in sizes with signed comparisons | Historical accident | `usize` everywhere for sizes; `isize` for offsets; never mixed silently |
| Socket API requires 15 steps for HTTP | Historical layering | HTTP is a first-class type, not a pile of socket calls |
| `strtol` requires a sentinel, not a Result | No error mechanism | All parse operations return `Result` |
| `memcpy` on overlapping regions is UB | No precondition checking | `copy` vs `copy_overlapping` are distinct functions; contracts enforced |
| Header files create implicit interfaces | Build model artifact | No header files; types are self-describing |
| `int` is platform-width | Historical artifact | Explicit-width types everywhere; `int` does not exist |

### Structural Principles

**Layering is absolute.** `core` never depends on `alloc`. `alloc` never depends on `std`. `std` never depends on platform-specific crates. Each layer is usable independently.

**Effects are honest.** Every operation that touches OS resources carries an effect annotation. A pure computation annotated pure is verified pure. Effects are inferred within modules — you only write annotations at `pub` boundaries.

**Allocators default to Heap.** Types like `Vec`, `String`, and `HashMap` are allocator-generic, but `Heap` is the default. Most code never mentions allocators. Custom allocators are opt-in for specialized use cases.

**Encoding is explicit.** Text is always `&str` (UTF-8) or an explicitly-typed alternative. There is no "system encoding." There is no "default charset." The type tells you what you have.

**Time is calendar-agnostic.** The core time types (`Timestamp`, `Instant`, `Duration`) have no embedded calendar. Calendars are separate modules that convert through Julian Day Number. Gregorian is common, not privileged.

**Async and sync are unified.** Blocking and non-blocking IO share the same trait hierarchy. A function that works on a `Read` works on both a blocking file and a non-blocking socket buffer. The difference is in the concrete type, not the trait.

---

## 2. Library Layers

```
┌──────────────────────────────────────────────────────────────────────┐
│  Application layer                                                    │
│  http · rpc · tls · ws · grpc  (separate crates, not stdlib)         │
├──────────────────────────────────────────────────────────────────────┤
│  std  (requires OS or WASI)                                           │
│  io · fs · net · http · sys · process · env · async · time           │
├──────────────────────────────────────────────────────────────────────┤
│  platform shims (see ferrum-stdlib-platform.md)                       │
│  linux · bsd · darwin · windows · fuchsia · ohos · wasi · zephyr      │
├──────────────────────────────────────────────────────────────────────┤
│  alloc  (requires an allocator, no OS)                                │
│  collections · string · fmt · sync · math · linalg · crypto          │
├──────────────────────────────────────────────────────────────────────┤
│  core  (no allocator, no OS, always available)                        │
│  primitives · ops · cmp · iter · option · result · mem · ptr         │
│  simd · binary · text::core · math::core                              │
└──────────────────────────────────────────────────────────────────────┘
```

**Import convention:**
```ferrum
import core.iter.Iterator
import alloc.collections.HashMap
import std.net.http.{Client, Request, Response}
import std.sys.posix                  // opt-in POSIX namespace
import std.sys.linux.io_uring         // opt-in Linux-specific
import std.sys.wasi.filesystem        // opt-in WASI (see platform spec)
```

The prelude imports a curated subset automatically (see §2.1).

### 2.1 The Prelude

Imported into every module automatically:

```ferrum
// From core
Option[T]         Result[T, E]       never
Copy Clone Drop Default
Eq PartialEq Ord PartialOrd
Iterator IntoIterator
From Into TryFrom TryInto
Send Sync

// From alloc
String Vec[T] Box[T] Arc[T]
HashMap[K,V] HashSet[T]

// From std
Error
io.Read io.Write io.Seek
fmt.Display fmt.Debug

// Intrinsics (compiler built-ins)
assert assert_eq assert_ne
panic unreachable todo unimplemented
dbg print println eprint eprintln
format
```

---

*End of Ferrum Standard Library Index — see linked files for module documentation.*
