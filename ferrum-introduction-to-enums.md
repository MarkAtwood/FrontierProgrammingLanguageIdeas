# Introduction to Enums in Ferrum

**Audience:** Programmers who know C and Python, learning Ferrum

---

## What Problem Are We Solving?

You've written code like this a hundred times. You have a value that can be one of several things, and each thing has different data attached to it.

In C, you reach for an enum and a union:

```c
enum MessageType { TEXT, IMAGE, FILE };

struct Message {
    enum MessageType type;
    union {
        char* text;                    // for TEXT
        struct { int w, h; char* url; } image;  // for IMAGE
        struct { char* name; size_t size; } file;   // for FILE
    };
};

void print_message(struct Message* m) {
    switch (m->type) {
        case TEXT:
            printf("Text: %s\n", m->text);
            break;
        case IMAGE:
            printf("Image: %dx%d at %s\n", m->image.w, m->image.h, m->image.url);
            break;
        case FILE:
            printf("File: %s (%zu bytes)\n", m->file.name, m->file.size);
            break;
    }
}
```

This works, but it has problems that will bite you:

- **Problem 1: The enum is just a number.** Nothing stops you from writing `m->type = 99` or `m->type = -1`. The compiler doesn't know those are invalid.
- **Problem 2: The union doesn't know which variant is active.** If `m->type` is `TEXT` but you read `m->image.w`, you get garbage. The compiler can't warn you.
- **Problem 3: The switch isn't checked.** Add `VIDEO` to the enum next month, and `print_message()` silently does nothing for videos. No warning. No error. You'll find out in production.
- **Problem 4: It's a lot of code.** You need an enum, a union, a struct wrapper, and careful discipline to keep them all in sync.

In Python, you might try dataclasses and isinstance:

```python
from dataclasses import dataclass

@dataclass
class TextMessage:
    text: str

@dataclass
class ImageMessage:
    width: int
    height: int
    url: str

@dataclass
class FileMessage:
    name: str
    size: int

def print_message(m):
    if isinstance(m, TextMessage):
        print(f"Text: {m.text}")
    elif isinstance(m, ImageMessage):
        print(f"Image: {m.width}x{m.height} at {m.url}")
    elif isinstance(m, FileMessage):
        print(f"File: {m.name} ({m.size} bytes)")
    else:
        raise ValueError(f"Unknown message type: {m}")
```

This is cleaner, but still has issues:

- **Problem 1: No compile-time checking.** You can pass any object to `print_message`. You'll find out at runtime.
- **Problem 2: isinstance chains are easy to forget.** Add `VideoMessage` next month, and the `else` branch catches it with a runtime error. Better than C's silent bug, but you still don't find out until someone hits that code path.
- **Problem 3: The type hint `TextMessage | ImageMessage | FileMessage` exists, but Python doesn't enforce it.** It's documentation, not a guarantee.

The core problem in both languages: **"which variant" and "what data" are separate things.** You have to manually keep them synchronized. The compiler can't help you because it doesn't understand the relationship.

---

## Enums in Ferrum: Tag and Data Together

Ferrum solves this by combining the tag and data into one type:

```ferrum
enum Message {
    Text(String),
    Image { width: i32, height: i32, url: String },
    File { name: String, size: u64 },
}
```

This says: a `Message` is **exactly one of** these three things:
- `Text` with a string
- `Image` with a width, height, and URL
- `File` with a name and size

Not both. Not neither. Not something else. The compiler knows all the possibilities.

Creating values:

```ferrum
let m1 = Message.Text("Hello, world!".to_string())
let m2 = Message.Image { width: 800, height: 600, url: "cat.jpg".to_string() }
let m3 = Message.File { name: "report.pdf".to_string(), size: 1024 }
```

The type of `m1`, `m2`, and `m3` is all `Message`. The compiler tracks which variant each is.

---

## Using Enums with match

To do something with a `Message`, you use `match`:

```ferrum
fn print_message(m: Message) {
    match m {
        Message.Text(text) => println("Text: {text}"),
        Message.Image { width, height, url } => println("Image: {width}x{height} at {url}"),
        Message.File { name, size } => println("File: {name} ({size} bytes)"),
    }
}
```

The `match` expression does three things at once:

1. **Checks which variant you have.** Is it `Text`, `Image`, or `File`?
2. **Extracts the data.** Binds `text`, or `width`/`height`/`url`, or `name`/`size` to the actual values inside.
3. **Runs the right code.** Each "arm" (the part after `=>`) handles one variant.

Compare this to C and Python:

| C | Python | Ferrum |
|---|--------|--------|
| Check `m->type` manually | `isinstance()` checks | `match` checks automatically |
| Access `m->text` manually | Access `m.text` | Pattern binds `text` for you |
| Forget a case? Silent bug | Forget a case? Runtime error | Forget a case? **Won't compile** |
| Tag/data can get out of sync | Object might not be a message | Impossible by construction |

---

## The Compiler Catches Missing Cases

Here's where Ferrum really shines. What happens if you forget a case?

```ferrum
fn print_message(m: Message) {
    match m {
        Message.Text(text) => println("Text: {text}"),
        Message.Image { width, height, url } => println("Image: {width}x{height} at {url}"),
        // oops, forgot File
    }
}
```

The compiler refuses to build your code:

```
error[E0004]: non-exhaustive match
  --> messages.fe:12:5
   |
12 |     match m {
   |     ^^^^^^^ pattern `Message.File { .. }` not covered
   |
   = note: the matched value is of type `Message`
   = help: ensure all variants are matched, or add a wildcard pattern `_`
```

This isn't a warning you can ignore. Your code does not compile until you handle every variant.

Now imagine you're six months into a project. You add `VideoMessage` to handle video uploads:

```ferrum
enum Message {
    Text(String),
    Image { width: i32, height: i32, url: String },
    File { name: String, size: u64 },
    Video { duration_secs: u32, url: String },  // NEW
}
```

The compiler immediately tells you every place in your codebase that needs updating:

```
error[E0004]: non-exhaustive match
  --> messages.fe:12:5
   |
12 |     match m {
   |     ^^^^^^^ pattern `Message.Video { .. }` not covered

error[E0004]: non-exhaustive match
  --> handlers.fe:45:9
   |
45 |         match msg {
   |         ^^^^^^^^^ pattern `Message.Video { .. }` not covered

error[E0004]: non-exhaustive match
  --> api.fe:78:5
   |
78 |     match response {
   |     ^^^^^^^^^^^^^^ pattern `Message.Video { .. }` not covered
```

In C, you'd grep for `switch` and hope you found them all. In Python, you'd discover missing cases when users hit them in production. In Ferrum, the compiler finds them all before your code even runs.

---

## Simple Enums (No Data)

Not every variant needs data. Sometimes you just want a fixed set of choices:

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

fn main() {
    let heading = Direction.North
    let behind = opposite(heading)  // Direction.South
}
```

This looks like a C enum, but it's type-safe:

```ferrum
let d: Direction = Direction.North  // OK
let d: Direction = 0                // ERROR: expected Direction, found integer
let d: Direction = "North"          // ERROR: expected Direction, found String
```

You can only put a `Direction` value where a `Direction` is expected. No accidental integers.

---

## Practical Example: Parsing Commands

Let's say you're building a CLI tool that accepts commands like `add 5`, `subtract 3`, `multiply 2`, or `quit`. Here's how you'd model and parse them:

```ferrum
enum Command {
    Add(i64),
    Subtract(i64),
    Multiply(i64),
    Divide(i64),
    Quit,
}

fn parse_command(input: &str): Option[Command] {
    let parts: Vec[&str] = input.trim().split_whitespace().collect()

    match parts.as_slice() {
        ["add", n] => n.parse().ok().map(Command.Add),
        ["subtract", n] => n.parse().ok().map(Command.Subtract),
        ["multiply", n] => n.parse().ok().map(Command.Multiply),
        ["divide", n] => n.parse().ok().map(Command.Divide),
        ["quit"] => Some(Command.Quit),
        _ => None,
    }
}

fn execute(cmd: Command, current: i64): Option[i64] {
    match cmd {
        Command.Add(n) => Some(current + n),
        Command.Subtract(n) => Some(current - n),
        Command.Multiply(n) => Some(current * n),
        Command.Divide(n) if n != 0 => Some(current / n),
        Command.Divide(_) => None,  // division by zero
        Command.Quit => None,
    }
}
```

Notice how each command variant carries exactly the data it needs. `Add` needs a number. `Quit` needs nothing. The enum captures this precisely.

---

## Practical Example: API Responses

APIs often return responses with different shapes depending on success or failure. Here's how to model that:

```ferrum
enum ApiResponse {
    Success { data: UserData, cached: bool },
    NotFound { resource_id: String },
    RateLimited { retry_after_secs: u32 },
    ServerError { message: String, code: i32 },
}

fn handle_response(resp: ApiResponse) {
    match resp {
        ApiResponse.Success { data, cached } => {
            if cached {
                println("(from cache)")
            }
            display_user(data)
        }
        ApiResponse.NotFound { resource_id } => {
            println("Resource not found: {resource_id}")
        }
        ApiResponse.RateLimited { retry_after_secs } => {
            println("Too many requests. Try again in {retry_after_secs} seconds.")
        }
        ApiResponse.ServerError { message, code } => {
            println("Server error {code}: {message}")
            report_to_monitoring(code, message)
        }
    }
}
```

Compare this to Python, where you might check `response["status"]` and hope the right fields exist, or C, where you'd have a union and manually check a status code. Here, each response type carries exactly its relevant data, and the compiler ensures you handle all cases.

---

## Wildcards and Ignoring Fields

Sometimes you don't need all the data from a variant.

The `_` wildcard matches anything and discards it:

```ferrum
fn is_error(resp: ApiResponse): bool {
    match resp {
        ApiResponse.Success { .. } => false,
        _ => true,  // any other variant is an error
    }
}
```

The `..` in `{ .. }` means "there are fields here, but I don't care about them":

```ferrum
fn get_retry_time(resp: ApiResponse): Option[u32] {
    match resp {
        ApiResponse.RateLimited { retry_after_secs } => Some(retry_after_secs),
        _ => None,
    }
}
```

You can also ignore specific fields:

```ferrum
match resp {
    ApiResponse.Success { data, cached: _ } => display_user(data),  // ignore cached flag
    // ...
}
```

---

## Combining Variants with |

When multiple variants should have the same behavior, use `|` to combine them:

```ferrum
fn should_retry(resp: ApiResponse): bool {
    match resp {
        ApiResponse.RateLimited { .. } | ApiResponse.ServerError { .. } => true,
        ApiResponse.Success { .. } | ApiResponse.NotFound { .. } => false,
    }
}
```

This is cleaner than repeating the same code for each variant.

---

## Guards: Adding Conditions to Patterns

Sometimes a pattern needs an extra condition. Use `if` after the pattern:

```ferrum
fn describe_response(resp: ApiResponse): String {
    match resp {
        ApiResponse.Success { cached: true, .. } => "Cache hit".to_string(),
        ApiResponse.Success { cached: false, .. } => "Fresh data".to_string(),
        ApiResponse.RateLimited { retry_after_secs } if retry_after_secs > 60 =>
            "Rate limited (long wait)".to_string(),
        ApiResponse.RateLimited { .. } =>
            "Rate limited (short wait)".to_string(),
        ApiResponse.ServerError { code, .. } if code >= 500 =>
            "Server error".to_string(),
        ApiResponse.ServerError { .. } =>
            "Client error".to_string(),
        ApiResponse.NotFound { .. } =>
            "Not found".to_string(),
    }
}
```

The `if` after a pattern is called a guard. The arm only matches if:
1. The pattern matches, AND
2. The guard condition is true

Guards are checked in order. The first matching arm wins.

**Important:** When using guards, the compiler can't always prove exhaustiveness. You may need a catch-all arm:

```ferrum
fn categorize(n: i32): &str {
    match n {
        x if x < 0 => "negative",
        x if x > 0 => "positive",
        _ => "zero",  // needed because compiler can't prove x < 0 || x == 0 || x > 0
    }
}
```

---

## if let: When You Only Care About One Variant

Full `match` is overkill when you only want to handle one case:

```ferrum
// Full match - verbose when you only care about Success
match response {
    ApiResponse.Success { data, .. } => process(data),
    _ => {}  // do nothing
}

// if let - cleaner
if let ApiResponse.Success { data, .. } = response {
    process(data)
}
```

You can add an `else` for everything else:

```ferrum
if let ApiResponse.Success { data, .. } = response {
    process(data)
} else {
    println("Request failed")
}
```

This is especially useful for `Option` and `Result` (covered below).

---

## while let: Looping on a Pattern

Similar to `if let`, but loops while the pattern matches:

```ferrum
// Process items from a queue until it's empty
while let Some(item) = queue.pop() {
    process(item)
}
```

---

## Option: Values That Might Not Exist

Ferrum has no null. Instead, it uses `Option` to represent values that might not exist:

```ferrum
enum Option[T] {
    Some(T),   // there's a value
    None,      // there's no value
}
```

The `[T]` means `Option` works with any type. `Option[String]` is an optional string. `Option[i32]` is an optional integer.

Here's how it compares to C and Python:

**C:**
```c
// Might return NULL
User* find_user(int id);

void greet(int id) {
    User* user = find_user(id);
    if (user != NULL) {
        printf("Hello, %s\n", user->name);
    }
    // Forget the NULL check? Segfault.
}
```

**Python:**
```python
def find_user(id):
    # Returns User or None
    ...

def greet(id):
    user = find_user(id)
    if user is not None:
        print(f"Hello, {user.name}")
    # Forget the None check? AttributeError at runtime.
```

**Ferrum:**
```ferrum
fn find_user(id: u64): Option[User]

fn greet(id: u64) {
    match find_user(id) {
        Some(user) => println("Hello, {}", user.name),
        None => println("User not found"),
    }
}
```

What happens if you forget to handle `None`?

```ferrum
fn greet(id: u64) {
    match find_user(id) {
        Some(user) => println("Hello, {}", user.name),
        // forgot None
    }
}
```

Compiler error:

```
error[E0004]: non-exhaustive match
 --> greet.fe:3:5
  |
3 |     match find_user(id) {
  |     ^^^^^^^^^^^^^^^^^^^ pattern `None` not covered
  |
  = help: ensure all variants are matched
```

The compiler forces you to decide what to do when there's no value. You can't forget.

`if let` is common with `Option`:

```ferrum
if let Some(user) = find_user(id) {
    println("Hello, {}", user.name)
}
// If None, nothing happens - and that's clearly intentional
```

---

## Result: Operations That Might Fail

`Result` represents an operation that can either succeed with a value or fail with an error:

```ferrum
enum Result[T, E] {
    Ok(T),    // success, here's the value
    Err(E),   // failure, here's the error
}
```

**C:**
```c
// Returns bytes read, or -1 on error (check errno)
int read_file(const char* path, char* buf, size_t len);

void process_config() {
    char buf[1024];
    int n = read_file("config.txt", buf, sizeof(buf));
    if (n < 0) {
        perror("read_file");
        return;
    }
    // Easy to forget the error check. Code silently continues with garbage.
}
```

**Python:**
```python
def read_file(path):
    with open(path) as f:
        return f.read()
    # Raises FileNotFoundError, PermissionError, etc.

def process_config():
    try:
        contents = read_file("config.txt")
        # process it
    except Exception as e:
        print(f"Error: {e}")
    # Forget the try/except? Exception propagates up silently.
```

**Ferrum:**
```ferrum
fn read_file(path: &str): Result[String, IoError]

fn process_config() {
    match read_file("config.txt") {
        Ok(contents) => {
            // process contents
        }
        Err(e) => println("Error reading config: {e}"),
    }
}
```

What if you forget to handle the error?

```ferrum
fn process_config() {
    match read_file("config.txt") {
        Ok(contents) => {
            // process contents
        }
        // forgot Err
    }
}
```

Compiler error:

```
error[E0004]: non-exhaustive match
 --> config.fe:3:5
  |
3 |     match read_file("config.txt") {
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ pattern `Err(_)` not covered
  |
  = help: ensure all variants are matched
```

### The ? Operator: Propagating Errors

When you're in a function that also returns `Result`, you can use `?` to propagate errors:

```ferrum
fn load_config(): Result[Config, IoError] {
    let contents = read_file("config.txt")?  // If Err, return it immediately
    let config = parse_config(&contents)?    // Same here
    Ok(config)
}
```

The `?` operator means: "If this is `Err`, return from the function with that error. If it's `Ok`, unwrap the value and continue."

It's shorthand for:

```ferrum
fn load_config(): Result[Config, IoError] {
    let contents = match read_file("config.txt") {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    let config = match parse_config(&contents) {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    Ok(config)
}
```

Much cleaner with `?`.

---

## State Machines with Enums

Enums are perfect for state machines because each state can carry exactly the data relevant to that state.

Let's build a simple TCP connection state machine:

```ferrum
enum ConnectionState {
    Disconnected,
    Connecting { address: String, attempts: u32 },
    Connected { socket: Socket, connected_at: Timestamp },
    Disconnecting { reason: String },
}
```

Notice:
- `Disconnected` has no data - there's nothing to track
- `Connecting` tracks the address and retry count
- `Connected` has the actual socket and when we connected
- `Disconnecting` knows why we're disconnecting

Now the event handler:

```ferrum
enum Event {
    Connect(String),       // user wants to connect to this address
    ConnectionSuccess(Socket),
    ConnectionFailed,
    Timeout,
    Disconnect(String),    // user wants to disconnect, here's why
    SocketClosed,
}

fn handle_event(state: ConnectionState, event: Event): ConnectionState {
    match (state, event) {
        // From Disconnected, we can start connecting
        (ConnectionState.Disconnected, Event.Connect(addr)) =>
            ConnectionState.Connecting { address: addr, attempts: 1 },

        // While connecting, we might succeed
        (ConnectionState.Connecting { .. }, Event.ConnectionSuccess(socket)) =>
            ConnectionState.Connected {
                socket,
                connected_at: Timestamp.now(),
            },

        // While connecting, we might fail - retry up to 3 times
        (ConnectionState.Connecting { address, attempts }, Event.ConnectionFailed)
            if attempts < 3 =>
            ConnectionState.Connecting { address, attempts: attempts + 1 },

        // Too many failures - give up
        (ConnectionState.Connecting { .. }, Event.ConnectionFailed) =>
            ConnectionState.Disconnected,

        // Timeout while connecting - same as failure
        (ConnectionState.Connecting { address, attempts }, Event.Timeout)
            if attempts < 3 =>
            ConnectionState.Connecting { address, attempts: attempts + 1 },

        (ConnectionState.Connecting { .. }, Event.Timeout) =>
            ConnectionState.Disconnected,

        // From Connected, user can request disconnect
        (ConnectionState.Connected { .. }, Event.Disconnect(reason)) =>
            ConnectionState.Disconnecting { reason },

        // Disconnecting completes when socket closes
        (ConnectionState.Disconnecting { .. }, Event.SocketClosed) =>
            ConnectionState.Disconnected,

        // Ignore events that don't make sense in the current state
        (state, _) => state,
    }
}
```

Why is this better than a C struct with a state field?

1. **You can't access socket when disconnected.** In C, you could accidentally read `connection.socket` when `connection.state == DISCONNECTED`. Here, `Disconnected` simply doesn't have a socket field.

2. **You can't forget to initialize state-specific data.** When you write `ConnectionState.Connecting { address, attempts: 1 }`, you must provide both fields. In C, you might forget to initialize `attempts`.

3. **Adding a new state is safe.** If you add `Reconnecting`, the compiler tells you every match that needs to handle it.

---

## Full Comparison: C, Python, and Ferrum

Let's see the same problem solved in all three languages.

**Problem:** Model shapes (circles, rectangles, triangles) and compute their areas.

### C Version

```c
#include <stdio.h>
#include <math.h>

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

// Constructor functions
struct Shape make_circle(double r) {
    struct Shape s;
    s.type = CIRCLE;
    s.circle.radius = r;
    return s;
}

struct Shape make_rectangle(double w, double h) {
    struct Shape s;
    s.type = RECTANGLE;
    s.rect.width = w;
    s.rect.height = h;
    return s;
}

struct Shape make_triangle(double a, double b, double c) {
    struct Shape s;
    s.type = TRIANGLE;
    s.tri.a = a;
    s.tri.b = b;
    s.tri.c = c;
    return s;
}

double area(struct Shape s) {
    switch (s.type) {
        case CIRCLE:
            return 3.14159 * s.circle.radius * s.circle.radius;
        case RECTANGLE:
            return s.rect.width * s.rect.height;
        case TRIANGLE: {
            double p = (s.tri.a + s.tri.b + s.tri.c) / 2;
            return sqrt(p * (p - s.tri.a) * (p - s.tri.b) * (p - s.tri.c));
        }
        default:
            return 0;  // What if someone passes type=99?
    }
}
```

**Problems:**
- 50+ lines of boilerplate
- Can create invalid shapes (`s.type = 99`)
- Can access wrong variant (`s.rect.width` when `s.type == CIRCLE`)
- Adding a shape requires updating multiple places, no compiler help
- The `default` case is defensive code for "impossible" states

### Python Version

```python
from dataclasses import dataclass
from typing import Union
import math

@dataclass
class Circle:
    radius: float

@dataclass
class Rectangle:
    width: float
    height: float

@dataclass
class Triangle:
    a: float
    b: float
    c: float

Shape = Union[Circle, Rectangle, Triangle]  # Type hint, not enforced

def area(shape: Shape) -> float:
    if isinstance(shape, Circle):
        return 3.14159 * shape.radius ** 2
    elif isinstance(shape, Rectangle):
        return shape.width * shape.height
    elif isinstance(shape, Triangle):
        p = (shape.a + shape.b + shape.c) / 2
        return math.sqrt(p * (p - shape.a) * (p - shape.b) * (p - shape.c))
    else:
        raise ValueError(f"Unknown shape: {shape}")
```

**Problems:**
- `isinstance` chains are manual and error-prone
- The `else: raise` is defensive code you hope never runs
- Type hints are documentation, not enforcement
- `area("not a shape")` fails at runtime, not compile time
- Adding a shape requires hunting down all isinstance chains

### Ferrum Version

```ferrum
enum Shape {
    Circle(f64),
    Rectangle { width: f64, height: f64 },
    Triangle { a: f64, b: f64, c: f64 },
}

fn area(shape: Shape): f64 {
    match shape {
        Shape.Circle(r) => 3.14159 * r * r,
        Shape.Rectangle { width, height } => width * height,
        Shape.Triangle { a, b, c } => {
            let p = (a + b + c) / 2.0
            (p * (p - a) * (p - b) * (p - c)).sqrt()
        }
    }
}
```

**Advantages:**
- 14 lines total
- Cannot create an invalid `Shape`
- Cannot access wrong variant's data
- Adding `Ellipse` causes compile errors everywhere it needs handling
- No defensive `else` case - there are no other cases, provably
- Type checked at compile time

---

## Summary

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Enum with data | Manual tag + union | isinstance chains | `enum` with variants |
| Type safety | None (enums are integers) | Runtime only | Compile-time |
| Missing case check | Not checked | Runtime error | Compile error |
| Pattern matching | switch (no data extraction) | if/elif (verbose) | `match` (concise) |
| Null handling | NULL pointers | None + checking | `Option[T]` |
| Error handling | Return codes + errno | Exceptions | `Result[T, E]` |
| State machines | Structs + discipline | Classes + discipline | Enums (type-enforced) |

The key insight: Ferrum enums unify the tag ("which variant") and the data ("what's inside") into a single type. The compiler tracks both, so:

- You can't create invalid variants
- You can't access the wrong variant's data
- You can't forget to handle a variant
- Adding new variants is safe - the compiler finds every place that needs updating

This isn't magic or new theory. Languages like ML have had this since the 1970s. Rust brought it to systems programming. Ferrum adopts it because it eliminates entire categories of bugs that plague C and Python code - the kind of bugs you've probably debugged at 2am.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete enum syntax and semantics.*
