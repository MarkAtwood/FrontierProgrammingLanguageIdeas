# Ferrum for Rust Programmers

**Audience:** Experienced Rust developers learning Ferrum

You already understand ownership, borrowing, lifetimes, traits, and why they matter. Ferrum shares these foundations — you won't need to relearn them. This guide focuses on what's different: where Ferrum changes syntax, where it adds features Rust doesn't have, and where it makes different tradeoffs.

---

## The Short Version

If you're impatient, here's Ferrum in 30 seconds:

```ferrum
// Ferrum
fn process[T: Display](items: &[T]): Result[usize, Error] ! IO {
    for item in items {
        println("{}", item)
    }
    Ok(items.len())
}
```

```rust
// Rust equivalent
fn process<T: Display>(items: &[T]) -> Result<usize, Error> {
    for item in items {
        println!("{}", item);
    }
    Ok(items.len())
}
```

Key differences visible here:
- `[T]` not `<T>` for generics
- `:` not `->` for return types
- `! IO` declares effects (Ferrum tracks IO in the type system)
- No semicolons (optional, usually omitted)
- `println()` not `println!()` (it's a function, not a macro)

Now let's go deeper.

---

## Syntax Changes

These are mechanical — your fingers will adapt in a day.

### Generics: Square Brackets

```rust
// Rust
fn map<T, U, F>(opt: Option<T>, f: F) -> Option<U>
where
    F: FnOnce(T) -> U
```

```ferrum
// Ferrum
fn map[T, U, F](opt: Option[T], f: F): Option[U]
where
    F: FnOnce(T): U
```

**Why the change:** `<>` creates parsing ambiguity with comparison operators. Every language that uses `<>` for generics has grammar complications. Square brackets don't.

```ferrum
// Unambiguous in Ferrum
let x = a < b && c > d       // comparisons
let y: Vec[i32] = Vec.new()  // generic
```

### Paths: Dots Instead of Double Colon

```rust
// Rust
use std::collections::HashMap;
let map = HashMap::<String, i32>::new();
std::io::stdout().write_all(b"hello")?;
```

```ferrum
// Ferrum
use std.collections.HashMap
let map = HashMap[String, i32].new()
std.io.stdout().write_all(b"hello")?
```

**Why:** `::` is two characters for no benefit. Every other mainstream language uses `.` for namespacing.

### Attributes: @ Instead of #[]

```rust
// Rust
#[derive(Debug, Clone)]
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
}
```

```ferrum
// Ferrum
@derive(Debug, Clone)
@repr(C)
struct Point {
    x: f64,
    y: f64,
}
```

**Why:** Slightly cleaner, one less bracket pair. Also `@` reads as "at" which suggests metadata/annotation.

### Return Types: Colon

```rust
// Rust
fn foo() -> i32 { 42 }
fn bar() -> Result<String, Error> { Ok("hello".into()) }
```

```ferrum
// Ferrum
fn foo(): i32 { 42 }
fn bar(): Result[String, Error] { Ok("hello".into()) }
```

**Why:** Consistency with variable declarations (`let x: i32`). The arrow `->` is reserved for closure types.

### No Semicolons (Usually)

```ferrum
fn example() {
    let x = 5
    let y = 10
    println("sum = {}", x + y)
}
```

Semicolons are optional statement terminators. You can use them if you want, but idiomatic Ferrum omits them. The grammar is designed to be unambiguous without them.

**When you need semicolons:** To suppress the value of an expression when you want the block to return unit:

```ferrum
fn returns_unit() {
    compute_something();  // semicolon discards the return value
}
```

### Macros Are Just Functions

Rust's `println!`, `vec!`, `format!` are macros because Rust's type system can't express variadic functions cleanly. Ferrum handles this differently:

```ferrum
// These are functions, not macros
println("Hello, {}!", name)
let v = vec(1, 2, 3, 4, 5)
let s = format("Value: {}", x)
```

Ferrum still has macros for compile-time code generation, but you use them less often because more things are expressible as regular functions.

---

## Lifetimes: Inferred, Not Annotated

This is the biggest ergonomic difference.

### Rust: Explicit Lifetimes

```rust
struct Parser<'input> {
    source: &'input str,
    position: usize,
}

impl<'input> Parser<'input> {
    fn new(source: &'input str) -> Parser<'input> {
        Parser { source, position: 0 }
    }

    fn peek(&self) -> Option<&'input str> {
        self.source.get(self.position..self.position + 1)
    }
}

fn parse_all<'a, 'b>(parser: &'a mut Parser<'b>, limit: usize) -> Vec<&'b str> {
    // ...
}
```

### Ferrum: Region Inference

```ferrum
struct Parser {
    source: &str,
    position: usize,
}

impl Parser {
    fn new(source: &str): Parser {
        Parser { source, position: 0 }
    }

    fn peek(&self): Option[&str] {
        self.source.get(self.position..self.position + 1)
    }
}

fn parse_all(parser: &mut Parser, limit: usize): Vec[&str] {
    // ...
}
```

**What's happening:** Ferrum's compiler infers region (lifetime) relationships automatically using the same rules Rust uses — it just doesn't require you to write them out.

**The tradeoff:** You can't *name* lifetimes, so certain advanced patterns require different approaches. But 95% of Rust lifetime annotations are just satisfying the borrow checker, not conveying information to humans. Ferrum drops that ceremony.

**When it matters:** If you're writing self-referential structures or complex lifetime relationships, Ferrum provides escape hatches (explicit region annotations in `trusted` code). But the common cases — returning references from structs, iterators over borrowed data — just work.

---

## Effects: Tracked in the Type System

This is a feature Rust doesn't have.

### The Problem Effects Solve

In Rust, these functions have the same signature:

```rust
fn compute(x: i32) -> i32 {
    x * 2
}

fn compute_with_logging(x: i32) -> i32 {
    println!("computing...");  // IO!
    x * 2
}
```

You can't tell from the signature which one does IO. You have to read the implementation.

### Ferrum: Effects in Signatures

```ferrum
fn compute(x: i32): i32 {
    x * 2
}

fn compute_with_logging(x: i32): i32 ! IO {
    println("computing...")
    x * 2
}
```

The `! IO` declares that this function performs IO. If you try to call an IO function from a pure function, the compiler stops you:

```ferrum
fn pure_function(x: i32): i32 {
    compute_with_logging(x)  // ERROR: cannot perform IO in pure function
}
```

### Effect Types

| Effect | Meaning |
|--------|---------|
| `IO` | File, console, environment access |
| `Net` | Network operations |
| `Async` | Can suspend/await |
| `Alloc` | Allocates memory |
| `Panic` | Might panic |
| `Unsafe` | Does something the compiler can't verify |

Combine with `+`:

```ferrum
pub fn download_and_save(url: &str, path: &str): Result[(), Error] ! Net + IO {
    let data = http.get(url)?
    fs.write(path, &data)?
    Ok(())
}
```

### You Don't Write Effects Everywhere

Effects are **inferred within modules** and **required at `pub` boundaries**:

```ferrum
// Private function — effects inferred, no annotation needed
fn helper(x: i32): i32 {
    println("debug: {}", x)  // compiler infers ! IO
    x * 2
}

// Public function — must declare effects
pub fn api_function(x: i32): i32 ! IO {
    helper(x)
}
```

This means internal code looks like regular Rust. Effects only appear in your public API, where they serve as documentation and contract.

### Why This Matters

1. **API honesty.** When a library says `fn compute(x: i32): i32`, you know it's pure. No hidden logging, no surprise network calls, no telemetry.

2. **Testing.** Pure functions don't need mocks.

3. **Parallelism.** The compiler knows which functions can safely run in parallel.

4. **Optimization.** Pure functions can be memoized, reordered, or eliminated.

---

## Safety Levels: Four, Not One

Rust has one escape hatch: `unsafe`. Ferrum has four levels, because not all unsafe operations are equally dangerous.

### The Four Levels

| Level | Trust | Example | Auditing |
|-------|-------|---------|----------|
| `unchecked` | Skips runtime checks | `arr.get_unchecked(i)` | Low risk — correctness, not memory |
| `trusted` | I know what I'm doing | Internal invariant maintenance | Medium — local reasoning |
| `extern` | External code is correct | FFI calls | High — depends on external code |
| `unsafe` | Raw pointer manipulation | `*ptr = value` | Highest — memory safety at stake |

### Why Four Levels?

Consider these Rust `unsafe` blocks:

```rust
// Rust — all just "unsafe"
unsafe { slice.get_unchecked(index) }      // skip bounds check
unsafe { my_internal_sort(data) }           // maintain invariant
unsafe { libc::write(fd, buf.as_ptr(), len) } // call C
unsafe { *raw_ptr = value }                 // raw pointer write
```

These have very different risk profiles, but Rust treats them identically for auditing purposes.

```ferrum
// Ferrum — distinguished by risk
unchecked { slice.get_unchecked(index) }     // just a bounds check skip
trusted { my_internal_sort(data) }           // I maintain the invariant
extern { libc.write(fd, buf.as_ptr(), len) } // calling into C
unsafe { *raw_ptr = value }                  // actual memory unsafety
```

**Auditing benefit:** When reviewing a codebase, you can:
- Ignore `unchecked` blocks (just performance optimization)
- Locally verify `trusted` blocks (check the invariant)
- Flag `extern` blocks for FFI review
- Carefully scrutinize `unsafe` blocks (memory safety)

---

## Contracts: requires/ensures/invariant

Ferrum has built-in design-by-contract, something Rust only has through macros and external tools.

### Preconditions and Postconditions

```ferrum
fn binary_search[T: Ord](arr: &[T], target: &T): Option[usize]
    requires arr.is_sorted()
    ensures match result {
        Some(i) => arr[i] == *target,
        None => !arr.contains(target),
    }
{
    // implementation
}
```

**`requires`** — what must be true when calling this function. Calling with an unsorted array is a bug in the *caller*.

**`ensures`** — what the function guarantees about its return value. The compiler verifies this (statically where possible, runtime checks otherwise).

### Struct Invariants

```ferrum
struct SortedVec[T: Ord] {
    inner: Vec[T],
}
invariant self.inner.is_sorted()

impl SortedVec[T: Ord] {
    fn push(&mut self, value: T) {
        let pos = self.inner.binary_search(&value).unwrap_or_else(|i| i)
        self.inner.insert(pos, value)
        // invariant automatically checked here in debug builds
    }
}
```

The invariant is checked at the end of every `pub` method (in debug builds). It documents and enforces the struct's correctness condition.

### Comparing to Rust

In Rust, you'd write:

```rust
/// Binary search over a sorted slice.
///
/// # Preconditions
/// - `arr` must be sorted in ascending order
///
/// # Returns
/// - `Some(index)` where `arr[index] == target`
/// - `None` if target is not in `arr`
fn binary_search<T: Ord>(arr: &[T], target: &T) -> Option<usize> {
    // ...
}
```

Problems:
1. Comments aren't checked
2. Nothing stops you from passing an unsorted array
3. The postcondition isn't even expressible in a comment (what does "where" mean exactly?)

Ferrum contracts are part of the type system. The compiler enforces them.

---

## Proof Functions: SMT-Backed Verification

For properties the SMT solver can handle, Ferrum can verify contracts at compile time:

```ferrum
fn abs(x: i32): i32
    ensures result >= 0
    ensures result == x || result == -x
{
    if x >= 0 { x } else { -x }
}
// Compiler proves these postconditions hold for all x
```

For properties that need induction or complex reasoning, you write proof functions:

```ferrum
proof fn reverse_involutive[T](xs: &[T])
    ensures xs.reverse().reverse() == xs
{
    match xs {
        [] => {}
        [head, ..tail] => {
            reverse_involutive(tail)
            // algebraic reasoning the compiler checks
        }
    }
}
```

**`proof fn`** compiles to nothing — it's erased after verification. It exists only to guide the SMT solver.

**When you need this:** Cryptography, aerospace, financial systems — anywhere you need mathematical certainty, not just testing confidence.

**When you don't:** Most code. Contracts are checked at runtime in debug builds, which catches most bugs. Proofs are for when "most" isn't good enough.

---

## Allocators: Heap by Default

Rust requires threading allocators through generics:

```rust
fn process<A: Allocator>(data: Vec<u8, A>, allocator: A) -> Vec<u8, A> {
    // ...
}
```

Ferrum defaults to `Heap` and only requires explicit allocators when you want something else:

```ferrum
// Uses Heap implicitly
fn process(data: Vec[u8]): Vec[u8] {
    // ...
}

// Explicit allocator when needed
fn process_with_allocator[A: Allocator](data: Vec[u8, A]): Vec[u8, A]
    given A
{
    // ...
}
```

The `given A` clause says "this function needs an allocator of type A provided by the caller." It's like an implicit parameter.

---

## Modules: Different Organization

### No mod.rs

Rust's module system has two ways to declare a module:

```
src/
  foo.rs          # mod foo
  bar/
    mod.rs        # mod bar
    baz.rs        # mod bar::baz
```

Ferrum simplifies this:

```
src/
  foo.fe          # mod foo
  bar.fe          # mod bar (can contain submodules)
  bar/
    baz.fe        # mod bar.baz
```

A module is either a single file `foo.fe` or a directory `foo/` with files inside. No `mod.rs` ambiguity.

### Visibility

```ferrum
pub struct Foo { ... }        // visible outside this module
pub(package) struct Bar { ... }  // visible within this package
struct Baz { ... }            // private to this module
```

Ferrum uses `pub(package)` where Rust uses `pub(crate)`. Same concept, different name.

### use Syntax

```rust
// Rust
use std::collections::{HashMap, HashSet};
use crate::utils::helper;
```

```ferrum
// Ferrum
use std.collections.{HashMap, HashSet}
use .utils.helper
```

The leading `.` means "this package" (like `crate::`).

---

## Async: Not Colored

Rust async functions are "colored" — async and sync functions have different types:

```rust
// Rust: two different worlds
fn sync_read(path: &str) -> Result<String, Error> { ... }
async fn async_read(path: &str) -> Result<String, Error> { ... }
```

Ferrum uses effects instead:

```ferrum
// Ferrum: same signature, different effects
fn sync_read(path: &str): Result[String, Error] ! IO { ... }
fn async_read(path: &str): Result[String, Error] ! IO + Async { ... }
```

The `Async` effect marks suspendable functions. You can call sync functions from async contexts freely. The effect system tracks what can suspend.

```ferrum
async fn process() ! IO + Async {
    let config = sync_read("config.toml")?   // sync IO, fine
    let data = async_read("data.txt").await? // async IO, fine
    // ...
}
```

**Structured concurrency** is built-in:

```ferrum
async fn fetch_all(urls: &[String]): Vec[Response] ! Net + Async {
    scope |s| {
        urls.iter()
            .map(|url| s.spawn(|| http.get(url)))
            .collect()
    }.await
}
```

The `scope` ensures all spawned tasks complete before returning. No unstructured spawning into the void.

---

## Capabilities: First-Class Authority

Ferrum has a capability system for controlling ambient authority. Instead of any code being able to do `fs.read("file")`, capabilities make authority explicit:

```ferrum
fn process_file(fs: FsCap, path: &str): Result[String, Error] ! IO {
    fs.read(path)
}
```

The `FsCap` is a capability token. You can only do filesystem operations if someone gave you the capability.

**Why this matters:**
- Sandboxing is built into the type system
- You can audit what authority a function needs by looking at its parameters
- Libraries can't secretly phone home or write to disk

In practice, capabilities are often implicit in the runtime context:

```ferrum
fn main() ! IO + Net {
    // main() has full capabilities by default
    let data = fs.read("input.txt")?
}
```

You pass restricted capabilities when you want to sandbox:

```ferrum
fn run_plugin(plugin: Plugin, fs: FsCap.read_only("/plugins/")) ! IO {
    plugin.run(fs)  // plugin can only read from /plugins/
}
```

---

## What's the Same

Don't relearn these — they work like Rust:

- **Ownership and borrowing** — same rules
- **Move semantics** — same rules
- **Pattern matching** — same syntax and semantics
- **Result and Option** — same types, same `?` operator
- **Traits** — same concept (syntax differs slightly)
- **Enums with data** — same
- **Iterators** — same lazy, zero-cost model
- **Closures** — same (`Fn`, `FnMut`, `FnOnce`)
- **Smart pointers** — `Box`, `Rc`, `Arc`, `Weak` (with `[T]` syntax)
- **Interior mutability** — `Cell`, `RefCell` work the same
- **Thread safety** — `Send` and `Sync` traits
- **Drop** — same destructor trait

---

## Migration Checklist

When porting Rust to Ferrum:

1. **Syntax transforms** (mechanical):
   - `<T>` → `[T]`
   - `::` → `.`
   - `#[attr]` → `@attr`
   - `->` → `:`
   - Remove semicolons
   - `crate::` → `.`

2. **Remove lifetime annotations** — Ferrum infers them

3. **Add effects to `pub` functions**:
   - Uses `println`/files? → `! IO`
   - Uses network? → `! Net`
   - Can panic? → `! Panic` (optional, for documentation)

4. **Map `unsafe` blocks** to appropriate level:
   - Bounds check skips → `unchecked`
   - Invariant maintenance → `trusted`
   - FFI calls → `extern`
   - Pointer manipulation → `unsafe`

5. **Add contracts** where appropriate:
   - Document preconditions with `requires`
   - Document postconditions with `ensures`
   - Add `invariant` to types with internal invariants

6. **Review macros** — many can become regular functions

---

## Quick Reference

| Rust | Ferrum |
|------|--------|
| `Vec<T>` | `Vec[T]` |
| `std::io::Result` | `std.io.Result` |
| `#[derive(Debug)]` | `@derive(Debug)` |
| `fn foo() -> T` | `fn foo(): T` |
| `'a` (lifetime) | (inferred) |
| `unsafe { }` | `unchecked/trusted/extern/unsafe { }` |
| `println!("{}", x)` | `println("{}", x)` |
| `pub(crate)` | `pub(package)` |
| `crate::foo` | `.foo` |
| `mod foo;` | (automatic from file structure) |
| (none) | `requires`/`ensures`/`invariant` |
| (none) | `! IO`, `! Net` (effects) |
| (none) | `proof fn` |
| (none) | Capabilities |

---

## What Ferrum Asks of You

Ferrum's additions aren't free. Here's what you commit to:

1. **Effect annotations at API boundaries.** You'll annotate `pub` functions with their effects. It's like Rust's `async` but for all side effects.

2. **Thinking about safety levels.** Instead of one `unsafe` bucket, you'll categorize unsafe operations. This is more work upfront, less work when auditing.

3. **Contracts for complex invariants.** You can ignore contracts entirely, but you'll get more value from Ferrum by documenting preconditions and postconditions where they matter.

The return: APIs that can't lie about their effects, finer-grained unsafe auditing, and optional but powerful verification tools when you need certainty beyond testing.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete specification.*
