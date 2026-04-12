# Ferrum for C and Python Programmers

**Audience:** SDE2 who knows C and Python, learning Ferrum for the first time

This guide covers the basics of Ferrum — enough to read and write simple programs. We assume you're comfortable with pointers, memory allocation, static typing (C), and dynamic typing with duck typing (Python). We'll use your existing knowledge as scaffolding and point out where Ferrum does things differently.

---

## What You Don't Need to Know

You don't need to know Rust, Haskell, or OCaml to understand Ferrum. If you know C and Python, you have everything you need. Ferrum borrows ideas from many languages, but we explain each concept from first principles using C and Python as reference points.

This is the only time we'll mention Haskell or OCaml in these introduction documents.

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
- No `#include`, no headers — the standard library is available automatically
- `println` is a built-in, not `printf` — no format specifiers like `%d`, just use `{}`
- No return type means returns nothing (like `void`)
- No forward declarations needed — functions can be defined in any order

**What's different from Python:**
- Curly braces for blocks, not indentation
- `fn` keyword for functions
- Compiled, not interpreted — you get a native binary
- Static typing — types are checked at compile time, not runtime

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

**Why does this matter?** When you read code, immutability tells you what can change and what can't:

```ferrum
fn process_data(data: &[u8]) {
    let header = parse_header(data)      // header is fixed once set
    let mut buffer = Vec.new()           // buffer will be modified
    let mut offset = 0                   // offset changes as we read

    // When reading this function, you immediately know:
    // - header never changes after line 2
    // - buffer and offset will change — pay attention to them
}
```

This isn't just style — the compiler enforces it. If you write `let x = 5` and later try `x = 6`, you get a compile error, not a runtime surprise.

**Comparing to C:**
```c
int x = 5;           // mutable by default
const int y = 10;    // immutable is opt-in
```
Ferrum flips this: immutable is default, mutable is opt-in. In C, `const` is often forgotten. In Ferrum, you have to actively choose `mut`.

**Comparing to Python:**
```python
x = 5    # everything is reassignable, no way to prevent it
x = 6    # Python can't stop you
```
Python has no equivalent to immutable bindings. You can set conventions, but nothing enforces them.

**Shadowing — reusing a name with a new value:**

```ferrum
fn main() {
    let x = "hello"       // x is a string
    let x = x.len()       // x is now a number (shadows the previous x)
    println("{}", x)      // prints 5
}
```

This isn't mutation — you're creating a new variable that happens to have the same name. The old `x` is gone. This is useful when transforming data:

```ferrum
let input = "  42  "
let input = input.trim()           // still &str, but trimmed
let input: i32 = input.parse()?    // now i32
```

Each `let` creates a new binding. This is different from `mut`, where you're changing the same variable.

---

## Types

Ferrum is statically typed. The compiler knows the type of every expression at compile time. You can usually omit type annotations because the compiler infers them, but when you need to be explicit:

```ferrum
let x: i32 = 42          // explicit type
let y = 42               // inferred as i32
```

### Integers

```ferrum
let a: i32 = 42       // signed 32-bit (like C's int32_t)
let b: u64 = 100      // unsigned 64-bit
let c: isize = -1     // pointer-sized signed (like C's intptr_t)
let d: usize = 50     // pointer-sized unsigned (like C's size_t)
let e = 42            // type inferred as i32
```

| Type | Bits | C Equivalent | Typical Use |
|------|------|--------------|-------------|
| `i8`, `u8` | 8 | `int8_t`, `uint8_t` | Bytes, small counters |
| `i16`, `u16` | 16 | `int16_t`, `uint16_t` | Less common, audio samples |
| `i32`, `u32` | 32 | `int32_t`, `uint32_t` | Default integer, most math |
| `i64`, `u64` | 64 | `int64_t`, `uint64_t` | Large values, timestamps |
| `i128`, `u128` | 128 | (no equivalent) | Cryptography, UUIDs |
| `isize`, `usize` | ptr | `intptr_t`, `size_t` | Array indices, sizes |

**Why so many types?** In Python, integers grow automatically. In C and Ferrum, you choose the size. This matters for:
- Memory layout (structs, arrays)
- Performance (32-bit math is often faster than 64-bit)
- Overflow behavior (what happens at the boundary)

**No implicit conversions.** This won't compile:

```ferrum
let a: i32 = 42
let b: i64 = a    // ERROR: expected i64, found i32
```

You must be explicit:

```ferrum
let b: i64 = a.into()     // safe conversion (i32 fits in i64)
let c: i32 = big.try_into()?  // fallible conversion (might not fit)
let d: i16 = x as i16     // cast — truncates if too big, use sparingly
```

**Why no implicit conversions?** C's implicit conversions are a source of bugs:

```c
// C — compiles without warning
unsigned int a = 4000000000;
int b = a;  // silent overflow, b is negative

// C — the classic comparison bug
size_t len = get_length();
int offset = -1;
if (offset < len) { ... }  // WRONG: offset becomes huge positive number
```

Ferrum makes you acknowledge every conversion. It's more typing, but it makes overflow bugs visible in the code.

**Literal suffixes for clarity:**

```ferrum
let x = 42_i64      // i64 literal
let y = 1_000_000   // underscores for readability
let z = 0xff_u8     // hex literal, u8
let w = 0b1010      // binary literal
let o = 0o755       // octal literal
```

### Floats

```ferrum
let x: f32 = 3.14      // 32-bit float (like C's float)
let y: f64 = 2.718     // 64-bit float (like C's double)
let z = 1.5            // inferred as f64 — the default
```

Same as C: `f32` is `float`, `f64` is `double`. Use `f64` unless you have a reason not to (GPU work, memory constraints).

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

**Why require explicit booleans?** C's "truthiness" enables bugs:

```c
// C — did you mean ==?
if (x = 5) { ... }   // assignment, always true

// C — checking the wrong thing
char* p = get_data();
if (p) { ... }       // checks if non-NULL, but what about empty string?
```

Ferrum requires a real boolean. `if pointer != null` is explicit about what you're checking.

### Characters

```ferrum
let c: char = 'A'      // Unicode scalar value, 4 bytes
let emoji: char = '🦀'
```

**`char` is NOT a byte.** In C, `char` is 1 byte and you use it for both characters and raw bytes. Ferrum separates these:
- `char` — a Unicode code point (4 bytes, can represent any character)
- `u8` — a byte (use for raw binary data)

```ferrum
let byte: u8 = b'A'    // ASCII byte literal
let ch: char = 'A'     // Unicode character
```

---

## Strings

Quick overview (see [ferrum-introduction-to-strings.md](ferrum-introduction-to-strings.md) for details):

```ferrum
let s: &str = "hello"           // string slice — borrowed, immutable
let owned: String = "hello".to_string()   // owned string — can grow
```

**Two string types? Why?**

Think of it like C, where you have:
- `const char*` — a pointer to string data owned elsewhere
- `char*` with `malloc` — a heap-allocated string you own

Ferrum has:
- `&str` — like `const char*` but better: known length, guaranteed UTF-8
- `String` — like a heap-allocated `char*`: you own it, can grow it

```ferrum
// &str — lightweight view into string data
let name: &str = "Alice"              // points to static data
let slice: &str = &my_string[0..5]    // points into a String

// String — owned, heap-allocated
let mut greeting = String.new()
greeting.push_str("Hello, ")
greeting.push_str(name)               // String can grow
```

**Comparing to Python:** Python strings are immutable and handle memory automatically. Ferrum's `String` is mutable (can append), and you're aware of ownership. But like Python (and unlike C), Ferrum strings are always valid UTF-8 — no worrying about encoding.

```ferrum
let name = "Alice"
println("Hello, {}!", name)

// String formatting is similar to Python's f-strings conceptually
let msg = format("Hello, {}!", name)
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

**Parameter types are required.** Ferrum doesn't infer function signatures. Your function signature is your API — it should be explicit about what types it accepts and returns.

**Return type** comes after `:`. Omit it for functions that return nothing (like `void` in C).

**Implicit return — this is important:** The last expression in a function (without a semicolon) is automatically returned. This is different from C and Python where you always write `return`.

```ferrum
fn square(x: i32): i32 {
    x * x        // returned — no semicolon, no return keyword
}

fn square_explicit(x: i32): i32 {
    return x * x   // also works, same thing
}

fn oops(x: i32): i32 {
    x * x;       // WRONG: semicolon makes this a statement, returns nothing
}
```

**When to use explicit `return`:** Use it for early returns:

```ferrum
fn abs(x: i32): i32 {
    if x < 0 {
        return -x    // early return
    }
    x                // implicit return
}

fn validate(input: &str): Result[Data, Error] {
    if input.is_empty() {
        return Err(Error.InvalidInput)  // early return on error
    }
    // ... normal processing ...
    Ok(data)
}
```

**Comparing to C:**
```c
int add(int x, int y) {
    return x + y;    // return keyword always required
}
```

**Comparing to Python:**
```python
def add(x, y):
    return x + y     # return keyword always required
```

The implicit return takes getting used to, but it leads to cleaner code once you're familiar with it.

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

**No parentheses around the condition** (unlike C). Braces are required (unlike Python's indentation).

```ferrum
// Ferrum
if x > 0 { ... }

// C — parentheses required
if (x > 0) { ... }

// Python — no braces
if x > 0:
    ...
```

**If is an expression** — it returns a value. This is one of the bigger mental shifts from C:

```ferrum
let sign = if x > 0 { "positive" } else { "negative" }
```

Both branches must return the same type. This replaces C's ternary operator:

```c
// C ternary
const char* sign = x > 0 ? "positive" : "negative";

// Ferrum if-expression — same idea, clearer syntax
let sign = if x > 0 { "positive" } else { "negative" }
```

**Multi-line if expressions:**

```ferrum
let result = if condition {
    let temp = compute_something()
    temp * 2       // last expression in block is the value
} else {
    default_value()
}
```

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

This is clearer than C's `while (1)` or `for (;;)`.

**`loop` can return a value:**

```ferrum
let result = loop {
    let attempt = try_something()
    if attempt.is_ok() {
        break attempt.unwrap()   // break with a value
    }
}
```

**`while` — conditional loop:**
```ferrum
let mut count = 5
while count > 0 {
    println("{}", count)
    count = count - 1
}
```

Same as C and Python, just without parentheses around the condition.

**`for` — iterate over a collection:**
```ferrum
for i in 0..5 {
    println("{}", i)    // prints 0, 1, 2, 3, 4
}

for item in items {
    println("{}", item)
}
```

`0..5` is a range (exclusive end, like Python's `range(5)`).
`0..=5` includes 5 (like Python's `range(6)`).

**Comparing to C:**
```c
for (int i = 0; i < 5; i++) { ... }   // index manipulation
```
Ferrum's `for` is iteration over a sequence, not index manipulation. You can't forget to increment or mess up the bounds.

**Comparing to Python:**
```python
for i in range(5): ...    # same idea!
for item in items: ...    # same
```

If you're comfortable with Python's `for`, Ferrum's version is almost identical.

**Iterating with index:**

```ferrum
for (i, item) in items.iter().enumerate() {
    println("{}: {}", i, item)
}
```

### Match

Pattern matching — like a powerful switch statement:

```ferrum
match x {
    0 => println("zero"),
    1 => println("one"),
    2..=9 => println("small"),
    _ => println("large"),
}
```

**Comparing to C's switch:**

```c
// C switch
switch (x) {
    case 0:
        printf("zero\n");
        break;    // don't forget this!
    case 1:
        printf("one\n");
        break;
    default:
        printf("other\n");
}
```

```ferrum
// Ferrum match — no break needed, no fallthrough bugs
match x {
    0 => println("zero"),
    1 => println("one"),
    _ => println("other"),
}
```

**Key differences from C's switch:**
- No `break` needed — each arm is independent
- No fallthrough bugs
- Must be exhaustive — you must handle every possible value
- Can match ranges, patterns, not just constants
- It's an expression — returns a value

**Match is exhaustive:** The compiler ensures you handle every case. This catches bugs at compile time:

```ferrum
enum Status { Active, Inactive, Pending }

fn describe(s: Status): &str {
    match s {
        Status.Active => "running",
        Status.Inactive => "stopped",
        // ERROR: non-exhaustive — Pending not handled
    }
}
```

**Match as an expression:**

```ferrum
let description = match x {
    0 => "zero",
    1..=9 => "single digit",
    _ => "large",
}
```

**Comparing to Python:** Python's `match` (3.10+) is similar:

```python
match x:
    case 0:
        print("zero")
    case _:
        print("other")
```

Ferrum's match is similar but with stronger compile-time guarantees.

---

## Structs

```ferrum
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 }
    println("({}, {})", p.x, p.y)
}
```

**Note:** Ferrum uses `type` not `struct`. Think of it as "define a type" rather than "define a struct."

**Comparing to C:**

```c
// C
typedef struct {
    double x;
    double y;
} Point;

Point p = { .x = 3.0, .y = 4.0 };
```

**Comparing to Python:**

```python
# Python dataclass
@dataclass
class Point:
    x: float
    y: float

p = Point(x=3.0, y=4.0)
```

Ferrum's syntax is between C and Python — cleaner than C's `typedef struct`, similar to Python's dataclass.

### Methods

In C, you write functions that take a struct pointer. In Ferrum, you attach methods to types:

```ferrum
struct Circle {
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

    // Method — takes &self (read-only access)
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

**What's `impl`?** It's where you define methods for a type. Think of it as "implement functionality for Circle."

**The `self` parameter:**
- **`&self`** — borrow the instance for reading (like `const Circle*` in C)
- **`&mut self`** — borrow mutably, can modify (like `Circle*` in C)
- **`self`** — take ownership, consume the instance (more on this in ownership docs)
- **`Self`** — refers to the type being implemented (here, `Circle`)

**Static methods vs instance methods:**

```ferrum
impl Circle {
    // No self parameter — call as Circle.new(...)
    fn new(x: f64, y: f64, r: f64): Self { ... }

    // Has &self — call on an instance: c.area()
    fn area(&self): f64 { ... }
}

let c = Circle.new(0.0, 0.0, 5.0)   // static method
let a = c.area()                     // instance method
```

**Comparing to C:**
```c
// C — explicit pointer passing
double circle_area(const Circle* self) {
    return 3.14159 * self->radius * self->radius;
}

// Called as
double a = circle_area(&c);
```

Ferrum's `c.area()` is syntactic sugar for essentially the same thing, but cleaner.

**Comparing to Python:**
```python
class Circle:
    def area(self):
        return 3.14159 * self.radius ** 2

# Called as
a = c.area()
```

Similar to Python, but Ferrum's `self` has explicit mutability (`&self` vs `&mut self`).

---

## Tuples and Arrays

### Tuples

Fixed-size, mixed types — useful for grouping a few values together:

```ferrum
let pair: (i32, &str) = (42, "answer")
let (num, text) = pair          // destructuring
println("{}: {}", num, text)

let first = pair.0              // index access (0-based)
let second = pair.1
```

**Why tuples instead of a struct?** Tuples are anonymous — use them when you don't want to define a named type for a quick grouping.

**Good for returning multiple values:**

```ferrum
fn min_max(items: &[i32]): (i32, i32) {
    let mut min = items[0]
    let mut max = items[0]
    for item in items {
        if *item < min { min = *item }
        if *item > max { max = *item }
    }
    (min, max)
}

let (lo, hi) = min_max(&numbers)
```

**Comparing to C:** C doesn't have tuples. You'd use a struct or out-parameters:

```c
// C — out-parameters
void min_max(int* items, int len, int* min_out, int* max_out) { ... }

// Or return a struct
struct MinMax { int min; int max; };
struct MinMax min_max(int* items, int len) { ... }
```

**Comparing to Python:** Python tuples work the same way:

```python
def min_max(items):
    return min(items), max(items)

lo, hi = min_max(numbers)
```

### Arrays

Fixed-size, same type, stack-allocated:

```ferrum
let arr: [i32; 5] = [1, 2, 3, 4, 5]    // 5 elements of i32
let first = arr[0]
let len = arr.len()                     // 5

let zeros: [i32; 100] = [0; 100]        // 100 zeros (fill syntax)
```

**The size is part of the type.** `[i32; 5]` and `[i32; 10]` are different types:

```ferrum
fn takes_five(arr: [i32; 5]) { ... }

let a: [i32; 5] = [1, 2, 3, 4, 5]
let b: [i32; 10] = [0; 10]

takes_five(a)    // ok
takes_five(b)    // ERROR: expected [i32; 5], found [i32; 10]
```

**Comparing to C:**
```c
int arr[5] = {1, 2, 3, 4, 5};
```
Similar, but C arrays decay to pointers. Ferrum arrays keep their size.

For dynamic size, use `Vec` (see [ferrum-introduction-to-smart-pointers.md](ferrum-introduction-to-smart-pointers.md)):

```ferrum
let mut v: Vec[i32] = Vec.new()
v.push(1)
v.push(2)
v.push(3)
println("length: {}", v.len())    // 3
```

`Vec` is like C++'s `std::vector` or Python's `list` — heap-allocated, grows as needed.

### Slices

A view into a contiguous sequence — doesn't own the data:

```ferrum
let arr = [1, 2, 3, 4, 5]
let slice: &[i32] = &arr[1..4]    // [2, 3, 4] — no copy, just a view
```

Slices are powerful because a function can accept `&[i32]` and work with data from arrays, Vecs, or other slices:

```ferrum
fn sum(items: &[i32]): i32 {
    let mut total = 0
    for item in items {
        total = total + *item
    }
    total
}

let arr = [1, 2, 3]
let vec = vec[4, 5, 6]

sum(&arr)         // works with array
sum(&vec)         // works with Vec
sum(&arr[0..2])   // works with partial slice
```

**Comparing to C:** This is like passing a pointer and length:

```c
int sum(const int* items, size_t len) { ... }
```

But Ferrum's slice carries the length with it — no separate parameter, no buffer overruns.

---

## References and Borrowing

Quick overview (see [ferrum-introduction-to-ownership.md](ferrum-introduction-to-ownership.md) for the full story):

**The problem Ferrum solves:** In C, you can have dangling pointers, double frees, and data races. Ferrum prevents these at compile time.

**References are safe pointers:**

```ferrum
fn main() {
    let x = 5
    let r = &x         // borrow x (read-only reference)
    println("{}", r)   // use the reference
    println("{}", x)   // x is still valid — we only borrowed it
}
```

**Mutable references let you modify:**

```ferrum
fn main() {
    let mut x = 5
    let r = &mut x     // mutable borrow
    *r = 10            // modify through reference (dereference with *)
    println("{}", x)   // prints 10
}
```

**The borrowing rules:**
1. You can have many `&T` (shared/read-only borrows) OR one `&mut T` (exclusive/mutable borrow), but not both at the same time
2. References can't outlive what they point to

These rules prevent bugs:

```ferrum
let mut data = vec[1, 2, 3]

let first = &data[0]     // borrow data
data.push(4)             // ERROR: can't modify while borrowed
println("{}", first)     // the borrow is used here
```

Why is this an error? Because `push` might reallocate the vector, making `first` a dangling pointer. In C, this would compile and crash at runtime. Ferrum catches it at compile time.

**Comparing to C:**

```c
// C — compiles fine, undefined behavior at runtime
int* first = &data[0];
push(&data, 4);           // might realloc, first now dangles
printf("%d\n", *first);   // boom
```

**This takes time to internalize.** If you're used to C, you're not used to the compiler checking your pointer usage this carefully. It will reject code that you know is safe. As you learn Ferrum's patterns, you'll find ways to express what you want. See the ownership introduction for strategies.

---

## Error Handling

Ferrum doesn't have exceptions (like Python) or error codes with manual checking (like C). It has `Option` and `Result` — types that force you to handle errors.

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

**Comparing to C:** In C, you'd return a sentinel value like `-1` and hope the caller checks:

```c
int find(int* items, int len, int target) {
    for (int i = 0; i < len; i++) {
        if (items[i] == target) return i;
    }
    return -1;  // sentinel — caller might forget to check
}
```

With `Option`, you can't ignore the possibility of `None` — the compiler forces you to handle it.

**Comparing to Python:** Python raises an exception or returns `None`. Ferrum's `Option` is explicit — you can see from the type signature that this function might not return a value.

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

`Result[T, E]` is either `Ok(value)` or `Err(error)`. The types tell you what can go wrong.

**Comparing to C error codes:**

```c
// C — error code pattern
int divide(int a, int b, int* result) {
    if (b == 0) return -1;  // error code
    *result = a / b;
    return 0;               // success
}
```

Ferrum's `Result` carries both the success value and the error in one type. You can't access the result without checking for error first.

### The `?` operator

Propagates errors up the call stack — this is Ferrum's answer to exception propagation:

```ferrum
fn do_math(): Result[i32, &str] {
    let a = divide(10, 2)?    // if Err, return early with that error
    let b = divide(a, 2)?     // same
    Ok(b)
}
```

The `?` is roughly equivalent to:

```ferrum
fn do_math(): Result[i32, &str] {
    let a = match divide(10, 2) {
        Ok(val) => val,
        Err(e) => return Err(e),
    }
    let b = match divide(a, 2) {
        Ok(val) => val,
        Err(e) => return Err(e),
    }
    Ok(b)
}
```

Much cleaner. The `?` is the idiomatic way to handle errors when you want to propagate them up.

**Comparing to Python exceptions:**

```python
def do_math():
    a = divide(10, 2)   # might raise
    b = divide(a, 2)    # might raise
    return b
```

Python's exceptions propagate automatically but invisibly. With `?`, you see exactly where error propagation happens.

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

Doc comments (`///`) are extracted by documentation tools and rendered as formatted text.

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

// Debug print (shows file:line and value)
dbg(some_value)
```

`dbg` is especially useful during development:

```ferrum
let x = 5
let y = dbg(x * 2)    // prints [src/main.fe:3] x * 2 = 10, returns 10
```

**Format placeholders:**
- `{}` — default display (human-readable)
- `{:?}` — debug format (shows structure)
- `{:#?}` — pretty debug (multi-line, indented)
- `{:04}` — pad with zeros: `0042`
- `{:.2}` — 2 decimal places: `3.14`
- `{:x}` — hexadecimal: `2a`
- `{:b}` — binary: `101010`

```ferrum
let point = Point { x: 3.0, y: 4.0 }
println("{:?}", point)     // Point { x: 3.0, y: 4.0 }
println("{:#?}", point)    // Pretty-printed across multiple lines
```

---

## Assertions

```ferrum
assert(x > 0)                          // panics if false
assert(x > 0, "x must be positive")    // with message
assert_eq(a, b)                        // panics if a != b
assert_ne(a, b)                        // panics if a == b
```

Assertions always run (unlike C's `assert` which can be disabled with `NDEBUG`). For release-only checks, use `debug_assert`.

```ferrum
assert(valid)              // always runs
debug_assert(valid)        // only in debug builds
```

---

## Type Inference

Ferrum infers types when it can, so you don't have to write them everywhere:

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

Sometimes the compiler can't figure out what type you want:

```ferrum
let x = "42".parse()?          // ERROR: parse to what type?
```

Solutions:

```ferrum
let x: i32 = "42".parse()?     // annotate the variable — inference fills in the type
```

---

## Constants

```ferrum
const MAX_SIZE: usize = 1024
const PI: f64 = 3.14159265359
const GREETING: &str = "Hello"
```

Constants are:
- Always immutable (no `mut` allowed)
- Must have explicit type (no inference)
- Must be compile-time evaluable (no runtime computation)
- Inlined at every use site (like `#define` in C, but type-safe)

**Comparing to C:**

```c
#define MAX_SIZE 1024           // no type checking
const size_t MAX_SIZE = 1024;   // can't use in array sizes
```

Ferrum's `const` is like a type-safe `#define` — you can use it anywhere a literal would work.

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
struct Point { x: f64, y: f64 } // struct
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
