# Introduction to Enums in Ferrum

**Audience:** Programmers who know C and Python, new to algebraic data types

---

## The Problem Enums Solve

Consider this C code:

```c
enum Shape { CIRCLE, RECTANGLE };

struct ShapeData {
    enum Shape type;
    union {
        double radius;           // for CIRCLE
        struct { double w, h; }; // for RECTANGLE
    };
};

double area(struct ShapeData s) {
    switch (s.type) {
        case CIRCLE:    return 3.14159 * s.radius * s.radius;
        case RECTANGLE: return s.w * s.h;
    }
}
```

Problems with this:

1. **The enum is just an integer.** Nothing stops you from writing `s.type = 42` or `s.type = -1`. The compiler won't complain.

2. **The union is untagged.** If `s.type` is `CIRCLE` but you read `s.w`, you get garbage. The compiler can't help.

3. **The switch isn't checked.** Add `TRIANGLE` to the enum, and `area()` silently does nothing for triangles. No warning. No error.

4. **Boilerplate.** You need an enum, a union, a struct wrapper, and discipline to keep them in sync.

Python's `enum` module is better but still limited:

```python
from enum import Enum

class Shape(Enum):
    CIRCLE = 1
    RECTANGLE = 2

# But how do you attach data?
# Option 1: Separate dict/object — same sync problem as C
# Option 2: Tuple values — awkward, untyped
class Shape(Enum):
    CIRCLE = ("circle",)
    RECTANGLE = ("rectangle",)

# You end up with isinstance() chains anyway:
def area(shape, data):
    if isinstance(data, Circle):
        return 3.14159 * data.radius ** 2
    elif isinstance(data, Rectangle):
        return data.w * data.h
    # Easy to forget a case. No compiler help.
```

The core problem: **C and Python separate "which variant" from "what data."** You have to manually keep them synchronized. Add a variant, and you have to hunt down every switch statement and isinstance chain.

---

## Enums in Ferrum

Ferrum enums combine the tag and data into one type:

```ferrum
enum Shape {
    Circle(f64),              // radius
    Rectangle(f64, f64),      // width, height
}
```

This says: a `Shape` is either a `Circle` with a radius, or a `Rectangle` with width and height. Not both. Not neither. Exactly one.

Creating values:

```ferrum
let c = Shape.Circle(3.0)
let r = Shape.Rectangle(4.0, 5.0)
```

The type of both `c` and `r` is `Shape`. The compiler knows which variant each is, and it knows what data each carries.

---

## Pattern Matching with `match`

To use a `Shape`, you pattern match on it:

```ferrum
fn area(shape: Shape): f64 {
    match shape {
        Shape.Circle(radius) => 3.14159 * radius * radius,
        Shape.Rectangle(w, h) => w * h,
    }
}
```

The `match` expression does three things:

1. **Checks which variant.** Is it a `Circle` or `Rectangle`?
2. **Extracts the data.** Binds `radius` or `w, h` to the contained values.
3. **Returns a value.** Each arm produces the result.

Compare to C:

| C | Ferrum |
|---|--------|
| Check tag manually | `match` checks for you |
| Access union fields manually | Pattern binds the data |
| Forget a case? Silent bug | Forget a case? Compile error |
| Tag/data can get out of sync | Impossible by construction |

---

## Exhaustiveness Checking

Here's the killer feature. Try this:

```ferrum
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
    Triangle(f64, f64, f64),  // NEW: three sides
}

fn area(shape: Shape): f64 {
    match shape {
        Shape.Circle(r) => 3.14159 * r * r,
        Shape.Rectangle(w, h) => w * h,
        // forgot Triangle
    }
}
```

Compiler error:

```
error: non-exhaustive match
  --> shapes.fe:10:5
   |
10 |     match shape {
   |     ^^^^^ pattern Shape.Triangle not covered
   |
   = help: add a match arm for Shape.Triangle
```

The compiler ensures you handle every variant. Add a new variant anywhere in your codebase, and the compiler tells you every place that needs updating.

In C, you'd have to grep for every switch statement and hope you found them all. In Python, you'd discover the missing case at runtime, in production, when a user hits the new variant.

---

## Enums Without Data

Not every variant needs data:

```ferrum
enum Direction {
    North,
    South,
    East,
    West,
}

fn opposite(d: Direction): Direction {
    match d {
        Direction.North => Direction.South,
        Direction.South => Direction.North,
        Direction.East => Direction.West,
        Direction.West => Direction.East,
    }
}
```

This is like a C enum, but type-safe. You can't assign an arbitrary integer to a `Direction`.

---

## Enums with Named Fields

For clarity, variants can have named fields:

```ferrum
enum Event {
    Click { x: i32, y: i32 },
    KeyPress { key: char, modifiers: u8 },
    Resize { width: u32, height: u32 },
}

fn handle(event: Event) {
    match event {
        Event.Click { x, y } => println!("clicked at ({x}, {y})"),
        Event.KeyPress { key, .. } => println!("pressed {key}"),
        Event.Resize { width, height } => resize_window(width, height),
    }
}
```

The `..` in `KeyPress { key, .. }` means "ignore the other fields." Useful when you don't need all the data.

---

## Guards

Sometimes you need to refine a match with a condition:

```ferrum
fn describe(shape: Shape): &str {
    match shape {
        Shape.Circle(r) if r < 1.0 => "small circle",
        Shape.Circle(r) if r > 100.0 => "huge circle",
        Shape.Circle(_) => "medium circle",
        Shape.Rectangle(w, h) if w == h => "square",
        Shape.Rectangle(_, _) => "rectangle",
    }
}
```

The `if` after a pattern is a guard. The arm only matches if the pattern matches AND the guard is true.

Guards are checked in order. The first matching arm wins.

---

## Nested Patterns

Patterns can be nested:

```ferrum
enum Expr {
    Num(i64),
    Add(Box[Expr], Box[Expr]),
    Mul(Box[Expr], Box[Expr]),
}

fn simplify(expr: Expr): Expr {
    match expr {
        // 0 + x => x
        Expr.Add(box Expr.Num(0), box x) => x,
        // x + 0 => x
        Expr.Add(box x, box Expr.Num(0)) => x,
        // 0 * x => 0
        Expr.Mul(box Expr.Num(0), _) => Expr.Num(0),
        // 1 * x => x
        Expr.Mul(box Expr.Num(1), box x) => x,
        // no simplification
        other => other,
    }
}
```

The `box` pattern matches through a `Box`, binding the contained value. You can nest patterns arbitrarily deep.

---

## The Wildcard and Or Patterns

`_` matches anything and discards it:

```ferrum
match shape {
    Shape.Circle(_) => "it's round",
    _ => "it's not round",
}
```

`|` combines alternatives:

```ferrum
fn is_horizontal(d: Direction): bool {
    match d {
        Direction.East | Direction.West => true,
        Direction.North | Direction.South => false,
    }
}
```

---

## Comparing to C

Here's the same code in both languages.

**C:**
```c
// Define the types
enum ShapeType { CIRCLE, RECTANGLE, TRIANGLE };

struct Circle { double radius; };
struct Rectangle { double width, height; };
struct Triangle { double a, b, c; };

struct Shape {
    enum ShapeType type;
    union {
        struct Circle circle;
        struct Rectangle rect;
        struct Triangle tri;
    };
};

// Create values
struct Shape make_circle(double r) {
    struct Shape s;
    s.type = CIRCLE;
    s.circle.radius = r;
    return s;
}

// Use values
double area(struct Shape s) {
    switch (s.type) {
        case CIRCLE:
            return 3.14159 * s.circle.radius * s.circle.radius;
        case RECTANGLE:
            return s.rect.width * s.rect.height;
        case TRIANGLE: {
            // Heron's formula
            double p = (s.tri.a + s.tri.b + s.tri.c) / 2;
            return sqrt(p * (p - s.tri.a) * (p - s.tri.b) * (p - s.tri.c));
        }
        default:
            return 0;  // what if someone passes type=99?
    }
}
```

**Ferrum:**
```ferrum
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
    Triangle(f64, f64, f64),
}

fn area(shape: Shape): f64 {
    match shape {
        Shape.Circle(r) => 3.14159 * r * r,
        Shape.Rectangle(w, h) => w * h,
        Shape.Triangle(a, b, c) => {
            let p = (a + b + c) / 2.0
            (p * (p - a) * (p - b) * (p - c)).sqrt()
        }
    }
}
```

The Ferrum version:
- Has no way to create an invalid `Shape`
- Has no way to access the wrong variant's data
- Will refuse to compile if you forget a case
- Is half the code

---

## Comparing to Python

**Python:**
```python
from dataclasses import dataclass

@dataclass
class Circle:
    radius: float

@dataclass
class Rectangle:
    width: float
    height: float

Shape = Circle | Rectangle  # type hint, not enforced at runtime

def area(shape):
    if isinstance(shape, Circle):
        return 3.14159 * shape.radius ** 2
    elif isinstance(shape, Rectangle):
        return shape.width * shape.height
    else:
        raise ValueError(f"Unknown shape: {shape}")
```

**Ferrum:**
```ferrum
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}

fn area(shape: Shape): f64 {
    match shape {
        Shape.Circle(r) => 3.14159 * r * r,
        Shape.Rectangle(w, h) => w * h,
    }
}
```

The Python version:
- Can receive any object at runtime, even non-shapes
- Requires isinstance() chains that are easy to forget
- The `else: raise` is defensive coding because Python can't prove exhaustiveness
- Type hints exist but aren't enforced

The Ferrum version:
- Only accepts `Shape` values — enforced by the type system
- The `match` is proven exhaustive at compile time
- There is no "else" case because there are no other cases

---

## Common Pattern: Option

`Option` represents a value that might not exist:

```ferrum
enum Option[T] {
    Some(T),
    None,
}
```

This replaces null pointers. Instead of a pointer that might be null:

```c
// C: might return NULL
char* find_user(int id);

// Caller must check
char* name = find_user(42);
if (name != NULL) {
    printf("Hello, %s\n", name);
}
// Forget to check? Segfault.
```

Ferrum:

```ferrum
fn find_user(id: u64): Option[String]

// Caller must match
match find_user(42) {
    Some(name) => println!("Hello, {name}"),
    None => println!("User not found"),
}
// Forget to handle None? Won't compile.
```

Shorthand with `if let`:

```ferrum
if let Some(name) = find_user(42) {
    println!("Hello, {name}")
}
// else branch optional if you just want to skip None
```

---

## Common Pattern: Result

`Result` represents an operation that might fail:

```ferrum
enum Result[T, E] {
    Ok(T),
    Err(E),
}
```

This replaces error codes and exceptions:

```c
// C: returns -1 on error, sets errno
int read_file(const char* path, char* buf, size_t len);

// Caller must check return AND errno
int n = read_file("data.txt", buf, sizeof(buf));
if (n < 0) {
    perror("read_file");  // easy to forget
}
```

```python
# Python: raises exception
def read_file(path):
    with open(path) as f:
        return f.read()
# Caller might not catch it. Exception propagates silently.
```

Ferrum:

```ferrum
fn read_file(path: &str): Result[String, IoError] ! IO

match read_file("data.txt") {
    Ok(contents) => process(contents),
    Err(e) => println!("Error: {e}"),
}

// Or use ? to propagate errors
fn load_config(): Result[Config, IoError] ! IO {
    let text = read_file("config.toml")?  // returns Err early if failed
    parse_config(&text)
}
```

The `?` operator is shorthand for "if this is `Err`, return it immediately."

---

## Common Pattern: State Machines

Enums naturally model state machines:

```ferrum
enum ConnectionState {
    Disconnected,
    Connecting { address: String, attempts: u32 },
    Connected { socket: Socket },
    Disconnecting { reason: String },
}

fn handle_event(state: ConnectionState, event: Event): ConnectionState {
    match (state, event) {
        (ConnectionState.Disconnected, Event.Connect(addr)) =>
            ConnectionState.Connecting { address: addr, attempts: 1 },

        (ConnectionState.Connecting { address, attempts }, Event.Timeout) if attempts < 3 =>
            ConnectionState.Connecting { address, attempts: attempts + 1 },

        (ConnectionState.Connecting { .. }, Event.Timeout) =>
            ConnectionState.Disconnected,

        (ConnectionState.Connecting { .. }, Event.Success(socket)) =>
            ConnectionState.Connected { socket },

        (ConnectionState.Connected { .. }, Event.Disconnect(reason)) =>
            ConnectionState.Disconnecting { reason },

        (ConnectionState.Disconnecting { .. }, Event.Closed) =>
            ConnectionState.Disconnected,

        (state, _) => state,  // ignore invalid transitions
    }
}
```

Each state carries exactly the data relevant to that state. A `Disconnected` connection doesn't have a socket. A `Connected` connection doesn't have a retry counter. The type system enforces this.

---

## Summary

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Enum + data | Manual tag/union | isinstance chains | `enum` with variants |
| Type safety | None (enums are ints) | Runtime only | Compile-time enforced |
| Exhaustiveness | Not checked | Not checked | Compiler-enforced |
| Pattern matching | switch (no destructuring) | if/elif (verbose) | `match` (concise, complete) |
| Null handling | NULL pointers | None + hope | `Option[T]` |
| Error handling | Return codes | Exceptions | `Result[T, E]` |
| State machines | Structs + discipline | Classes + discipline | Enums (type-enforced) |

Enums in Ferrum unify the tag and the data. The compiler knows which variant you have and what data it contains. Pattern matching lets you work with variants safely and concisely. Exhaustiveness checking ensures you never forget a case.

This isn't a new concept — ML had it in the 1970s, Haskell refined it, Rust popularized it. Ferrum adopts it because it eliminates entire categories of bugs that plague C and Python code.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete enum specification.*
