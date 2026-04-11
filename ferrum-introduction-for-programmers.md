# Ferrum for C and Python Programmers

**Audience:** SDE2 who knows C and Python, learning Ferrum for the first time

This guide covers the basics of Ferrum — enough to read and write simple programs. For deeper topics, see the specialized introductions linked at the end.

---

## Your First Program

```ferrum
fn main() {
    println("Hello, world!")
}
```

Save as `hello.fe`, run with `ferrum run hello.fe`.

**What's different from C:**
- No semicolons (they're optional, usually omitted)
- No `#include`, no headers
- `println` is a built-in, not `printf`
- No return type means returns nothing (like `void`)

**What's different from Python:**
- Curly braces for blocks, not indentation
- `fn` keyword for functions
- Compiled, not interpreted

---

## Variables and Mutability

```ferrum
fn main() {
    let x = 5           // immutable by default
    let mut y = 10      // mutable — can be changed

    y = y + x           // ok
    // x = 6            // ERROR: cannot assign to immutable variable

    println("x = {}, y = {}", x, y)
}
```

**The rule:** Variables are immutable by default. Add `mut` when you need to change them.

**Why?** Immutable is safer. When you see `let x = ...`, you know `x` never changes. When you see `let mut x = ...`, you know to watch for modifications.

**Comparing to C:**
```c
int x = 5;           // mutable by default
const int y = 10;    // immutable is opt-in
```
Ferrum flips this: immutable is default, mutable is opt-in.

**Comparing to Python:**
```python
x = 5    # everything is mutable, no way to prevent reassignment
x = 6    # Python can't stop you
```
Ferrum lets you say "this doesn't change" and the compiler enforces it.

---

## Types

### Integers

```ferrum
let a: i32 = 42       // signed 32-bit (like C's int32_t)
let b: u64 = 100      // unsigned 64-bit
let c: isize = -1     // pointer-sized signed (like C's intptr_t)
let d: usize = 50     // pointer-sized unsigned (like C's size_t)
let e = 42            // type inferred as i32
```

| Type | Bits | Range |
|------|------|-------|
| `i8`, `u8` | 8 | -128..127, 0..255 |
| `i16`, `u16` | 16 | ±32K, 0..65K |
| `i32`, `u32` | 32 | ±2B, 0..4B |
| `i64`, `u64` | 64 | ±9×10¹⁸, 0..18×10¹⁸ |
| `i128`, `u128` | 128 | huge |
| `isize`, `usize` | ptr | platform-dependent |

**No implicit conversions.** This won't compile:

```ferrum
let a: i32 = 42
let b: i64 = a    // ERROR: expected i64, found i32
```

You must be explicit:

```ferrum
let b: i64 = a.into()     // explicit conversion
let b: i64 = a as i64     // or cast (use sparingly)
```

**Why?** C's implicit conversions cause bugs:
```c
unsigned int a = 4000000000;
int b = a;  // silent overflow, b is negative
```
Ferrum makes you acknowledge every conversion.

### Floats

```ferrum
let x: f32 = 3.14      // 32-bit float
let y: f64 = 2.718     // 64-bit float (default)
let z = 1.5            // inferred as f64
```

### Booleans

```ferrum
let yes: bool = true
let no: bool = false
```

No implicit bool conversions. This won't compile:

```ferrum
let x = 5
if x {          // ERROR: expected bool, found i32
    // ...
}
```

You must be explicit:

```ferrum
if x != 0 {     // ok
    // ...
}
```

**Why?** C's "truthiness" causes bugs:
```c
if (x = 5) { ... }   // assignment, always true — did you mean ==?
```
Ferrum requires a real boolean.

### Characters

```ferrum
let c: char = 'A'      // Unicode scalar value, 4 bytes
let emoji: char = '🦀'
```

`char` is NOT a byte. It's a Unicode code point. For bytes, use `u8`.

---

## Strings

Quick overview (see [ferrum-introduction-to-strings.md](ferrum-introduction-to-strings.md) for details):

```ferrum
let s: &str = "hello"           // string slice — borrowed, immutable
let owned: String = "hello".to_string()   // owned string — can grow
```

- `&str` is like `const char*` but with known length and guaranteed UTF-8
- `String` is like a `std::string` — heap-allocated, growable
- Both are always valid UTF-8

```ferrum
let name = "Alice"
println("Hello, {}!", name)

let mut greeting = String.new()
greeting.push_str("Hello, ")
greeting.push_str(name)
```

---

## Functions

```ferrum
fn add(x: i32, y: i32): i32 {
    x + y    // last expression is the return value — no semicolon
}

fn greet(name: &str) {
    println("Hello, {}!", name)
}

fn main() {
    let sum = add(3, 4)
    greet("World")
}
```

**Parameter types are required.** Ferrum doesn't infer function signatures — that's your API, it should be explicit.

**Return type** comes after `:`. Omit it for functions that return nothing.

**Implicit return:** The last expression (without semicolon) is returned. Or use `return` explicitly:

```ferrum
fn abs(x: i32): i32 {
    if x < 0 {
        return -x
    }
    x
}
```

**Comparing to C:**
```c
int add(int x, int y) {
    return x + y;    // return required
}
```

**Comparing to Python:**
```python
def add(x, y):
    return x + y     # no type annotations (unless you add them)
```

---

## Control Flow

### If/Else

```ferrum
if x > 0 {
    println("positive")
} else if x < 0 {
    println("negative")
} else {
    println("zero")
}
```

**No parentheses around the condition** (unlike C). Braces are required (unlike Python).

**If is an expression** — it returns a value:

```ferrum
let sign = if x > 0 { "positive" } else { "negative" }
```

Both branches must return the same type.

### Loops

**`loop` — infinite loop:**
```ferrum
loop {
    println("forever")
    if done {
        break
    }
}
```

**`while` — conditional loop:**
```ferrum
while count > 0 {
    println("{}", count)
    count = count - 1
}
```

**`for` — iterate over a collection:**
```ferrum
for i in 0..5 {
    println("{}", i)    // prints 0, 1, 2, 3, 4
}

for item in items {
    println("{}", item)
}
```

`0..5` is a range (exclusive end). `0..=5` includes 5.

**Comparing to C:**
```c
for (int i = 0; i < 5; i++) { ... }   // C-style for
```
Ferrum's `for` is iteration, not index manipulation.

**Comparing to Python:**
```python
for i in range(5): ...    # similar!
for item in items: ...    # same idea
```

### Match

Pattern matching (see [ferrum-introduction-to-enums.md](ferrum-introduction-to-enums.md) for details):

```ferrum
match x {
    0 => println("zero"),
    1 => println("one"),
    2..=9 => println("small"),
    _ => println("large"),
}
```

`match` must be exhaustive — every possible value must be handled. The compiler checks this.

---

## Structs

```ferrum
type Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 }
    println("({}, {})", p.x, p.y)
}
```

**Note:** Ferrum uses `type` not `struct`. (It defines a type.)

### Methods

```ferrum
type Circle {
    center: Point,
    radius: f64,
}

impl Circle {
    // Constructor (by convention called "new")
    fn new(x: f64, y: f64, r: f64): Self {
        Circle {
            center: Point { x, y },   // shorthand: x: x becomes just x
            radius: r,
        }
    }

    // Method — takes &self
    fn area(&self): f64 {
        3.14159 * self.radius * self.radius
    }

    // Method that modifies — takes &mut self
    fn grow(&mut self, factor: f64) {
        self.radius = self.radius * factor
    }
}

fn main() {
    let mut c = Circle.new(0.0, 0.0, 5.0)
    println("Area: {}", c.area())
    c.grow(2.0)
    println("New area: {}", c.area())
}
```

**`&self`** — borrow the instance (read-only)
**`&mut self`** — borrow mutably (can modify)
**`self`** — take ownership (consumes the instance)
**`Self`** — the type being implemented (here, `Circle`)

**Comparing to C:**
```c
double circle_area(const Circle* self) {
    return 3.14159 * self->radius * self->radius;
}
```
Same idea, but in Ferrum the `self` parameter is special syntax.

**Comparing to Python:**
```python
def area(self):
    return 3.14159 * self.radius ** 2
```
Similar, but Ferrum's `self` has explicit mutability.

---

## Tuples and Arrays

### Tuples

Fixed-size, mixed types:

```ferrum
let pair: (i32, &str) = (42, "answer")
let (num, text) = pair          // destructuring
println("{}: {}", num, text)

let first = pair.0              // index access
let second = pair.1
```

Good for returning multiple values:

```ferrum
fn min_max(items: &[i32]): (i32, i32) {
    // ... find min and max ...
    (min, max)
}

let (lo, hi) = min_max(&numbers)
```

### Arrays

Fixed-size, same type:

```ferrum
let arr: [i32; 5] = [1, 2, 3, 4, 5]    // 5 elements
let first = arr[0]
let len = arr.len()                     // 5

let zeros: [i32; 100] = [0; 100]        // 100 zeros
```

Arrays are stack-allocated with fixed size known at compile time.

For dynamic size, use `Vec` (see [ferrum-introduction-to-smart-pointers.md](ferrum-introduction-to-smart-pointers.md)):

```ferrum
let mut v: Vec[i32] = Vec.new()
v.push(1)
v.push(2)
v.push(3)
```

### Slices

A view into a contiguous sequence:

```ferrum
let arr = [1, 2, 3, 4, 5]
let slice: &[i32] = &arr[1..4]    // [2, 3, 4] — no copy
```

Slices are borrowed — they don't own the data.

---

## References and Borrowing

Quick overview (see [ferrum-introduction-to-ownership.md](ferrum-introduction-to-ownership.md) for details):

```ferrum
fn main() {
    let x = 5
    let r = &x         // borrow x (read-only reference)
    println("{}", r)   // ok
    println("{}", x)   // x is still valid
}
```

```ferrum
fn main() {
    let mut x = 5
    let r = &mut x     // mutable borrow
    *r = 10            // modify through reference
    println("{}", x)   // prints 10
}
```

**The rules:**
1. You can have many `&T` (shared borrows) OR one `&mut T` (exclusive borrow), not both
2. References can't outlive what they point to

The compiler enforces these at compile time — no dangling pointers, no data races.

---

## Error Handling

Quick overview (see [ferrum-introduction-to-option-result.md](ferrum-introduction-to-option-result.md) for details):

### Option — for values that might not exist

```ferrum
fn find(items: &[i32], target: i32): Option[usize] {
    for (i, item) in items.iter().enumerate() {
        if *item == target {
            return Some(i)
        }
    }
    None
}

fn main() {
    let items = [10, 20, 30]

    match find(&items, 20) {
        Some(index) => println("Found at {}", index),
        None => println("Not found"),
    }
}
```

### Result — for operations that might fail

```ferrum
fn divide(a: i32, b: i32): Result[i32, &str] {
    if b == 0 {
        Err("division by zero")
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10, 2) {
        Ok(result) => println("Result: {}", result),
        Err(msg) => println("Error: {}", msg),
    }
}
```

### The `?` operator

Propagates errors up the call stack:

```ferrum
fn do_math(): Result[i32, &str] {
    let a = divide(10, 2)?    // if Err, return early
    let b = divide(a, 2)?
    Ok(b)
}
```

---

## Comments

```ferrum
// Single line comment

/* Multi-line
   comment */

/// Doc comment for the next item
/// Supports **markdown**
fn documented_function() {
    // ...
}
```

---

## Printing and Debugging

```ferrum
// Print to stdout
print("no newline")
println("with newline")
println("formatted: {} + {} = {}", 1, 2, 3)

// Print to stderr
eprint("error, no newline")
eprintln("error with newline")

// Debug print (uses Debug trait, shows structure)
dbg!(some_value)    // prints file:line and value
```

Format placeholders:
- `{}` — default display
- `{:?}` — debug format
- `{:#?}` — pretty debug (multi-line)
- `{:04}` — pad with zeros
- `{:.2}` — 2 decimal places

---

## Assertions

```ferrum
assert!(x > 0)                          // panics if false
assert!(x > 0, "x must be positive")    // with message
assert_eq!(a, b)                        // panics if a != b
assert_ne!(a, b)                        // panics if a == b
```

Assertions always run (unlike C where `assert` can be disabled).

---

## Type Inference

Ferrum infers types when it can:

```ferrum
let x = 42              // i32 (default integer)
let y = 3.14            // f64 (default float)
let v = Vec.new()       // type inferred from usage

v.push(1_i32)           // now v is Vec[i32]
```

You can annotate when needed:

```ferrum
let x: i64 = 42
let v: Vec[String] = Vec.new()
```

**When inference needs help:**

```ferrum
let x = "42".parse()?          // parse to what?
let x: i32 = "42".parse()?     // ok
let x = "42".parse[i32]()?     // also ok (turbofish)
```

---

## Constants

```ferrum
const MAX_SIZE: usize = 1024
const PI: f64 = 3.14159265359
```

Constants are:
- Always immutable
- Must have explicit type
- Must be compile-time evaluable
- Inlined at every use site

---

## What's Next

This covered the basics. For deeper topics, see:

| Topic | Guide |
|-------|-------|
| Ownership, borrowing, moves | [ferrum-introduction-to-ownership.md](ferrum-introduction-to-ownership.md) |
| Null safety, error handling | [ferrum-introduction-to-option-result.md](ferrum-introduction-to-option-result.md) |
| Polymorphism, interfaces | [ferrum-introduction-to-traits.md](ferrum-introduction-to-traits.md) |
| Sum types, pattern matching | [ferrum-introduction-to-enums.md](ferrum-introduction-to-enums.md) |
| Generic types and functions | [ferrum-introduction-to-generics.md](ferrum-introduction-to-generics.md) |
| Lazy iteration, lambdas | [ferrum-introduction-to-iterators.md](ferrum-introduction-to-iterators.md) |
| UTF-8, encoding, text | [ferrum-introduction-to-strings.md](ferrum-introduction-to-strings.md) |
| Heap allocation, Rc, Arc | [ferrum-introduction-to-smart-pointers.md](ferrum-introduction-to-smart-pointers.md) |
| Project structure, dependencies | [ferrum-introduction-to-modules.md](ferrum-introduction-to-modules.md) |
| Tracking side effects | [ferrum-introduction-to-effects.md](ferrum-introduction-to-effects.md) |
| Contracts, verification | [ferrum-introduction-to-proofs.md](ferrum-introduction-to-proofs.md) |
| Concurrent programming | [ferrum-introduction-to-async.md](ferrum-introduction-to-async.md) |
| Calling C code | [ferrum-introduction-to-ffi.md](ferrum-introduction-to-ffi.md) |
| Breaking the rules safely | [ferrum-introduction-to-unsafe.md](ferrum-introduction-to-unsafe.md) |

For complete language specification, see [ferrum.md](ferrum.md).

---

## Quick Reference

### Declaring things
```ferrum
let x = 5                    // immutable variable
let mut y = 10               // mutable variable
const C: i32 = 100           // constant

fn foo(x: i32): i32 { x }    // function
type Point { x: f64, y: f64 } // struct
enum Color { Red, Green, Blue } // enum

impl Point { ... }           // methods
trait Display { ... }        // trait
impl Display for Point { ... } // trait implementation
```

### Types
```ferrum
i8 i16 i32 i64 i128 isize    // signed integers
u8 u16 u32 u64 u128 usize    // unsigned integers
f32 f64                       // floats
bool                          // boolean
char                          // Unicode character
&str String                   // strings
[T; N]                        // array (fixed size)
Vec[T]                        // vector (dynamic)
(T, U, V)                     // tuple
Option[T]                     // Some(T) or None
Result[T, E]                  // Ok(T) or Err(E)
&T &mut T                     // references
Box[T] Rc[T] Arc[T]           // smart pointers
```

### Control flow
```ferrum
if cond { } else { }
match x { pat => expr, _ => expr }
loop { break }
while cond { }
for x in iter { }
return value
```

### Operators
```ferrum
+ - * / %                     // arithmetic
== != < > <= >=               // comparison
&& || !                       // logical
& | ^ << >>                   // bitwise
= += -= *= /= ...             // assignment
```

---

*End of Ferrum for C and Python Programmers*
