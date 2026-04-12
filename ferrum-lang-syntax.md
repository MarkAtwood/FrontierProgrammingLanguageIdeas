# Ferrum Language Reference — Syntax

**Part of:** [Ferrum Language Reference](ferrum-language-reference.md)

---

## 1. Expressions and Statements

### 1.1 Everything is an Expression

In Ferrum, most constructs are expressions and return a value. Statements are expressions whose value is `()` (unit).

```ferrum
let x = if condition { 1 } else { 2 }

let y = match input {
    Some(v) => v * 2,
    None    => 0,
}

let z = {
    let a = compute()
    let b = transform(a)
    b + 1   // last expression is the block's value
}
```

### 1.2 Variable Binding

```ferrum
let x = 42               // immutable binding, type inferred
let x: i32 = 42          // explicit type
let mut x = 42           // mutable binding
let (a, b) = (1, 2)      // destructuring
let Point { x, y } = pt  // struct destructuring
let [first, ..rest] = arr // slice pattern (see §2)
```

Bindings are immutable by default. `mut` is required for mutation.

Variables are initialized at the point of declaration. Uninitialized variable use is a compile error (tracked by dataflow analysis).

### 1.3 Assignment

```ferrum
x = 42           // requires mut binding
x += 1           // compound assignment: += -= *= /= %= &= |= ^= <<= >>=
x +%= 1          // wrapping compound
x +|= 1          // saturating compound
*ptr = value     // through a raw pointer (unsafe)
arr[i] = value   // indexed (bounds checked in debug)
struct.field = v // field assignment (requires &mut)
```

### 1.4 Arithmetic and Logic

```ferrum
// Arithmetic
a + b    a - b    a * b    a / b    a % b
-a       // negation

// Wrapping variants
a +% b   a -% b   a *% b

// Saturating variants
a +| b   a -| b   a *| b

// Bitwise
a & b    a | b    a ^ b    !a    a << n    a >> n
```

**Shift amount type safety:**

Shift operations require the shift amount to fit in `log2(bitwidth)` bits.
This prevents undefined behavior from over-shifting and catches bugs at
compile time.

```ferrum
let x: u32 = 1
let n: u8  = 5

x << n           // error: shift amount type u8 too wide for u32 (needs u5)
x << (n as u5)   // ok: explicit conversion acknowledges the truncation
x << 5           // ok: literal fits in u5
x << 5u5         // ok: explicit u5 suffix

// For any integer type T with BITS bits, the shift amount type is:
//   @Int(false, log2(T.BITS))
// Examples:
//   u8   → shift amount is u3  (0..7)
//   u16  → shift amount is u4  (0..15)
//   u32  → shift amount is u5  (0..31)
//   u64  → shift amount is u6  (0..63)
//   u128 → shift amount is u7  (0..127)

// This catches over-shift bugs at compile time:
let y: u8 = 1 << 8u4    // error: 8 exceeds u8 shift range (0..7)
let z: u8 = 1 << 7      // ok: shifts to 0x80
```

The right-shift operator `>>` performs arithmetic shift (sign-extending) on
signed types and logical shift (zero-extending) on unsigned types.

```ferrum
// Comparison (return bool)
a == b   a != b   a < b   a <= b   a > b   a >= b

// Logical (short-circuit)
a && b
a || b
!a

// Range expressions
0..10      // exclusive: 0, 1, ..., 9   (type Range[i32])
0..=10     // inclusive: 0, 1, ..., 10  (type RangeInclusive[i32])
..10       // from start to 10 (exclusive)
0..        // from 0 to end
..         // full range
```

### 1.5 Control Flow

**`if` / `else`:**
```ferrum
if condition {
    ...
} else if other {
    ...
} else {
    ...
}

// As expression:
let x = if flag { 1 } else { 0 }
```

**`loop`:**
```ferrum
loop {
    if done { break result_value }
}
// loop is an expression; break carries the value
let x = loop { if cond { break 42 } }
```

**`while`:**
```ferrum
while condition {
    ...
}
// while does not return a value (returns ())
```

**`for`:**
```ferrum
for item in collection {
    ...
}

for (i, item) in collection.enumerate() {
    ...
}

for i in 0..10 {
    ...
}
```

**`break` and `continue`:**
```ferrum
break           // exit current loop
break value     // exit loop with value (loop only)
continue        // next iteration

// Labeled:
'outer: for i in 0..10 {
    for j in 0..10 {
        if condition { break 'outer }
    }
}
```

**`return`:**
```ferrum
return value    // early return
return          // early return of ()
```

**`defer`:**
```ferrum
defer { expr }   // runs on any exit from the enclosing block
```

`defer` binds to the innermost enclosing block, not the function. It runs unconditionally on:
- Normal block completion
- `return` from the enclosing function
- `?` propagation (before the error leaves the block)
- Panic unwind

Multiple `defer` expressions in one block run in **reverse declaration order** — last declared, first executed. `defer` captures variables by reference.

```ferrum
fn push_batch(q: &mut BoundedQueue[T], items: &[T]) ! IO {
    trusted("batch push — truncate restores invariant on any exit") {
        defer { q.data.truncate(q.limit) }   // guaranteed to run

        for item in items {
            q.data.push(item.clone())
            log_progress(q)?   // early return here: defer fires, truncate runs,
                               // then Err propagates — invariant holds for caller
        }
    }
}
```

`defer` does not suppress or catch errors — it runs cleanup and then allows the exit to proceed normally. A panic inside a `defer` expression behaves the same as a panic anywhere else.

### 1.6 The `?` Operator

Propagates `Err` or `None` from the current function:

```ferrum
fn read_and_parse(path: &str): Result[Data] ! IO {
    let raw = read_file(path)?    // returns Err if read_file returns Err
    let parsed = parse(&raw)?     // returns Err if parse returns Err
    Ok(transform(parsed))
}
```

`?` works on `Result[T, E]` and `Option[T]`. For `Option`, `None` propagates. For `Result`, `Err(e)` propagates directly — no implicit type conversion occurs.

The function's return error type must accommodate the propagated error. There are two ways to achieve this:

1. **Error union (preferred):** The return type is `Result[T, E1 | E2 | ...]`. The compiler widens the union automatically as `?` expressions are encountered. Private functions may write `Result[T, _]` and let the compiler infer the full union. `pub` functions must write the union explicitly — same rule as effect annotations.

2. **Explicit conversion:** Use `.map_err(|e| ...)` at the call site to convert before propagating:
   ```ferrum
   let x = fallible().map_err(MyError::Wrap)?
   ```

There is no implicit `From`-trait coercion. If a `?` expression's error type does not fit the return type, it is a compile error with a message naming the mismatch.

`?` also propagates the effect: using `?` on a `! IO` function inside a function makes that function `! IO`.

> **Design decision:** `?` does not coerce through a `From` trait. The `From`-based approach (as in
> Rust) leads to god error types, coherence conflicts, and invisible information loss at every `?`
> site. Error unions make the accumulated error types visible in the signature, preserve all error
> information, and require no trait impls. Explicit `.map_err()` is available when a caller
> genuinely wants to collapse to a named type.

### 1.7 Closures

```ferrum
let add = |a, b| a + b           // inferred types
let add = |a: i32, b: i32|: i32 { a + b }   // explicit types
let double = |x| x * 2           // single expression
let process = |x| {              // block body
    let y = transform(x)
    y + 1
}

// Closures capture by the minimum required:
// - shared reference if the captured variable is only read
// - exclusive reference if it is mutated
// - move if move is required (e.g., crossing task boundary)

// Force move capture:
let s = String.from("hello")
let f = move || print(s)   // s is moved into the closure
```

### 1.8 `select` Expression

For concurrent channel operations. This is a compiler intrinsic with special syntax, not a user-definable construct:

```ferrum
let result = select {
    ch1 -> msg  => process(msg),
    ch2 -> msg  => fallback(msg),
    timeout(1s) => Err(TimeoutError),
}
```

`select` blocks until one branch is ready, then evaluates that branch. All branches must have the same type. `timeout(duration)` creates a timer that fires after the specified duration.

---

## 2. Pattern Matching

### 2.1 `match` Expression

```ferrum
match value {
    pattern1          => expr1,
    pattern2 if guard => expr2,
    _                 => default_expr,
}
```

`match` is **exhaustive** — the compiler verifies all cases are covered. A non-exhaustive match is a compile error.

### 2.2 Pattern Forms

**Literal patterns:**
```ferrum
match x {
    0     => "zero",
    1..=9 => "single digit",
    _     => "other",
}
```

**Enum patterns:**
```ferrum
match opt {
    Some(v) => use(v),
    None    => default(),
}

match result {
    Ok(v)    => v,
    Err(e)   => handle(e),
}
```

**Struct patterns:**
```ferrum
match point {
    Point { x: 0, y } => print("on y-axis at {y}"),
    Point { x, y: 0 } => print("on x-axis at {x}"),
    Point { x, y }    => print("at ({x}, {y})"),
}

// Ignore remaining fields
match msg {
    Message { kind: MsgKind.Error, text, .. } => log_error(text),
    _ => (),
}
```

**Tuple patterns:**
```ferrum
match pair {
    (0, y) => print("x is zero, y = {y}"),
    (x, 0) => print("y is zero, x = {x}"),
    (x, y) => print("({x}, {y})"),
}
```

**Slice patterns:**
```ferrum
match slice {
    []           => "empty",
    [x]          => "one element",
    [x, y]       => "two elements",
    [first, ..rest] => "first is {first}, rest has {rest.len()} elements",
    [.., last]   => "last is {last}",
}
```

**Guard clauses:**
```ferrum
match x {
    n if n < 0  => "negative",
    n if n == 0 => "zero",
    n           => "positive",
}
```

**Binding with `@`:**
```ferrum
match x {
    n @ 1..=9 => print("single digit: {n}"),
    _         => print("other"),
}
```

**Or patterns:**
```ferrum
match x {
    0 | 1 | 2 => "small",
    _         => "large",
}
```

### 2.3 `if let` and `while let`

```ferrum
if let Some(v) = maybe_value {
    use(v)
}

if let Ok(data) = result {
    process(data)
} else {
    handle_error()
}

while let Some(item) = iter.next() {
    process(item)
}
```

### 2.4 Destructuring in `let`

```ferrum
let (x, y) = compute_pair()
let Point { x, y } = get_point()
let [a, b, c] = three_element_array
let [head, ..tail] = slice   // tail is &[T]
```

---

## 3. Functions

### 3.1 Function Declarations

```ferrum
fn name[T: Bound](param: Type, ...): ReturnType ! Effects
    given [Cap: CapBound]
    requires precondition
    ensures  postcondition
{
    body
}
```

All parts except `fn`, the name, parameter list, and body are optional.

```ferrum
// Minimal
fn greet() {
    print("hello")
}

// Full
fn binary_search[T: Ord](xs: &[T], key: &T): Option[usize]
    given [A: Allocator]
    requires xs.is_sorted()
    ensures match result {
        Some(i) => xs[i] == *key,
        None    => forall i in 0..xs.len() => xs[i] != *key,
    }
{
    ...
}
```

### 3.2 Parameters

```ferrum
fn f(x: i32)            // by value (copy or move)
fn f(x: &i32)           // shared reference
fn f(x: &mut i32)       // exclusive reference
fn f(mut x: i32)        // local mutation of a by-value parameter
fn f(x: impl Trait)     // any type implementing Trait (sugar for generic)
fn f(x: &dyn Trait)     // dynamic dispatch
```

**Variadic parameters** are not supported. Use slices or iterators.

### 3.3 Methods

Methods are functions defined in `impl` blocks:

```ferrum
impl Type {
    fn method(self: &Self, ...): ... { ... }       // shared receiver
    fn method_mut(self: &mut Self, ...): ... { }  // mutable receiver
    fn consume(self: Self, ...): ... { ... }        // consuming receiver
    fn static_fn(...): ... { ... }                  // no receiver (static method)
}

// Call syntax
value.method()          // method call
Type.static_fn()        // static call
```

`self` in method position is sugar: `self: &Self` can be written `&self`, `self: &mut Self` as `&mut self`, `self: Self` as `self`.

### 3.4 Operator Overloading

Implemented via traits:

```ferrum
trait Add[Rhs = Self] {
    type Output
    fn add(self: Self, rhs: Rhs): Self.Output
}

impl Add for Vec2 {
    type Output = Vec2
    fn add(self: Vec2, rhs: Vec2): Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}
```

Overloadable operators and their traits:

| Operator | Trait |
|---|---|
| `+` `-` `*` `/` `%` | `Add` `Sub` `Mul` `Div` `Rem` |
| `&` `|` `^` `!` `<<` `>>` | `BitAnd` `BitOr` `BitXor` `Not` `Shl` `Shr` |
| `+=` etc. | `AddAssign` etc. |
| `==` `!=` | `Eq` `PartialEq` |
| `<` `<=` `>` `>=` | `Ord` `PartialOrd` |
| `[]` | `Index` `IndexMut` |
| unary `-` | `Neg` |
| `..` `..=` | (not overloadable) |

### 3.5 Constants

```ferrum
const PI: f64 = 3.14159265358979
const BITS: usize = 64
const BYTES: usize = BITS / 8           // constant expression
const MAX: u64 = (1 << 63) - 1          // constant expression
const TABLE: [u8; 4] = [0, 1, 2, 3]     // constant array
```

Constants must be initialized with constant expressions. A constant expression is:
- A literal
- A reference to another constant
- Arithmetic, bitwise, or comparison operators on constant expressions
- Array literals where all elements are constant expressions

**No user code executes at compile time.** The compiler evaluates constant expressions internally using its own evaluator. There is no `const fn` — if you need computed values, compute them at runtime or use an external build tool.

---

## 4. Types in Detail

### 4.1 Struct Types

```ferrum
type Point {
    x: f32,
    y: f32,
}

// Generic struct
type Pair[A, B] {
    first: A,
    second: B,
}

// Tuple struct
type Meters(f32)
type Rgb(u8, u8, u8)

// Unit struct
type Marker

// Struct with visibility
type Config {
    pub host: String,
    pub port: Port,
    timeout: Duration,    // private by default
}
```

**Struct construction:**
```ferrum
let p = Point { x: 1.0, y: 2.0 }
let m = Meters(3.5)
let c = Rgb(255, 0, 128)

// Update syntax (copies/moves fields from another instance)
let p2 = Point { x: 3.0, ..p }
```

**Field access:**
```ferrum
p.x
p.y
m.0     // tuple struct field
c.2
```

### 4.2 Enum Types

```ferrum
enum Direction {
    North,
    South,
    East,
    West,
}

enum Shape {
    Circle(f32),               // radius
    Rectangle(f32, f32),       // width, height
    Triangle { base: f32, height: f32 },
}

// Generic enum
enum Option[T] {
    Some(T),
    None,
}

enum Result[T, E] {
    Ok(T),
    Err(E),
}
```

Enums have no implicit integer conversion. To get a discriminant:
```ferrum
Direction.North as u32   // requires @repr(u32) on the enum
```

### 4.3 Type Aliases

```ferrum
type Meters = f32
type Result[T] = std.Result[T, Error]   // alias with different arity
type Callback = fn(i32): bool
```

Type aliases are transparent — `Meters` and `f32` are the same type. For a distinct type use a newtype (tuple struct with one field).

### 4.4 Newtype Pattern

```ferrum
type Meters(f32)
type Seconds(f32)

// Meters and Seconds are different types — cannot be accidentally mixed
fn distance_in_time(d: Meters, s: Seconds): f32 {
    d.0 / s.0
}

// Implement Display for a foreign type
type Wrapper(Vec[i32])
impl Display for Wrapper { ... }
```

### 4.5 Type Invariants

Invariants declared on a type are enforced at every point where the type's internal state could be observed externally:

```ferrum
type SortedVec[T: Ord] {
    inner: Vec[T],

    invariant forall i, j where i < j => self.inner[i] <= self.inner[j]
}
```

The invariant is checked:
- After every `&mut self` method returns (in debug and proof mode).
- After every construction (in all modes unless `release`-only).
- Before any `&self` method is called in proof mode (entry assertion).

In release mode, invariants are elided unless annotated `safe invariant`.

```ferrum
type SafeCounter {
    value: u64,
    safe invariant self.value < u64.MAX   // kept even in release
}
```

---
