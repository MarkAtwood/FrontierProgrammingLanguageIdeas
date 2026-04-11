# Ferrum Language Reference — Modules and Packages

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Modules and Packages

### 1.1 Module System

One file = one module. The module name is the file name. No header files, no `mod.rs` convention.

```
src/
    main.fe          → module main
    parser.fe        → module parser
    net/
        http.fe      → module net.http
        tcp.fe       → module net.tcp
```

### 1.2 Importing

```ferrum
import std.collections.HashMap
import std.io.{read_line, write_line}
import std.net                        // import the module
import std.net as network             // alias
import std.prelude.*                  // glob import (discouraged outside preludes)
```

### 1.3 Visibility

```ferrum
pub fn public_function() { ... }     // visible outside the module
fn private_function() { ... }        // module-private (default)

pub type PublicType { ... }
type PrivateType { ... }

pub(crate) fn crate_visible() { ... }  // visible within the package
pub(super) fn parent_visible() { ... } // visible in parent module
```

### 1.4 Packages

A package is a directory with a `Ferrum.toml` manifest:

```toml
[package]
name    = "mypackage"
version = "0.1.0"
edition = "2025"

[dependencies]
std     = { version = "1.0" }
serde   = { version = "2.1", features = ["derive"] }

[dev-dependencies]
proptest = { version = "1.0" }
```

### 1.5 The Prelude

The following are imported in every module without explicit import:

```ferrum
// Types
Option[T]    Result[T, E]    String    Vec[T]    HashMap[K, V]
Box[T]       Arc[T]          Rc[T]

// Traits
Copy Clone Drop Display Debug
Eq PartialEq Ord PartialOrd
Iterator From Into
Send Sync

// Functions
print  println  panic  unreachable  todo  unimplemented

// Macros
assert  assert_eq  assert_ne  dbg  todo
```

---

## 2. The Standard Library

### 2.1 Core (`core`)

No allocation. No OS. Safe for `no_std` embedded targets.

```ferrum
core.ops.*          // operator traits
core.cmp.*          // Eq, Ord, etc.
core.iter.*         // Iterator and combinators
core.option         // Option[T]
core.result         // Result[T, E]
core.mem.*          // size_of, align_of, swap, replace
core.ptr.*          // raw pointer operations (unsafe)
core.slice.*        // slice operations
core.str.*          // &str operations
core.num.*          // numeric traits and conversions
core.intrinsics.*   // compiler intrinsics (unsafe)
```

### 2.2 Alloc (`alloc`)

Requires an allocator. No OS. Available in embedded with a custom allocator.

```ferrum
alloc.vec           // Vec[T]
alloc.string        // String
alloc.collections.* // HashMap, BTreeMap, BinaryHeap, etc.
alloc.boxed         // Box[T]
alloc.rc            // Rc[T]
alloc.arc           // Arc[T]
alloc.sync.*        // Mutex, RwLock, Condvar
```

### 2.3 Standard (`std`)

Full standard library. Requires OS.

```ferrum
std.io.*            // I/O traits and types  ! IO
std.fs.*            // Filesystem            ! IO
std.net.*           // Networking            ! Net
std.process.*       // Process management    ! IO + Unsafe
std.thread.*        // OS threads            ! Sync
std.time.*          // Clocks and durations
std.env.*           // Environment           ! IO
std.path.*          // Path manipulation
std.ffi.*           // C interop
std.panic           // Panic handling
```

### 2.4 Key Traits Reference

```ferrum
trait Clone {
    fn clone(self: &Self): Self
}

trait Copy: Clone {}   // marker — copy semantics

trait Drop {
    fn drop(self: &mut Self)
}

trait Default {
    fn default(): Self
}

trait Display {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] ! IO
}

trait Debug: Display {}

trait Hash {
    fn hash[H: Hasher](self: &Self, state: &mut H)
}

trait Send {}   // marker: safe to send across threads
trait Sync {}   // marker: safe to share across threads (&T: Send implies T: Sync)

trait From[T] {
    fn from(value: T): Self
}

trait Into[T] {
    fn into(self: Self): T
}
// Blanket: impl[T, U: From[T]] Into[T] for U

trait Error: Display + Debug {
    fn source(self: &Self): Option[&dyn Error]
}
```
