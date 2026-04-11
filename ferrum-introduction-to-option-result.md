# Introduction to Option and Result in Ferrum

**Audience:** Programmers who know C and Python, new to algebraic error handling

---

## The Problem

### C: Null Pointers and Error Codes

In C, "no value" is represented by `NULL`:

```c
User* find_user(int id) {
    // returns NULL if not found
}

void greet_user(int id) {
    User* user = find_user(id);
    printf("Hello, %s\n", user->name);  // BUG: crashes if user is NULL
}
```

Tony Hoare, who invented null references in 1965, called it his "billion-dollar mistake." The problem: **nothing forces you to check.** The type `User*` doesn't tell you it might be null. You have to remember, and humans forget.

C handles errors with return codes:

```c
int read_config(const char* path, Config* out) {
    FILE* f = fopen(path, "r");
    if (f == NULL) return -1;  // error
    // ...read file...
    if (parse_failed) return -2;  // different error
    *out = config;
    return 0;  // success
}

void init() {
    Config cfg;
    read_config("app.conf", &cfg);  // BUG: ignored return value
    use_config(&cfg);  // cfg is garbage
}
```

The problems:
1. **Easy to ignore.** The compiler doesn't care if you check the return code.
2. **Out-of-band signaling.** The error code is separate from the actual result. You have to remember which is which.
3. **Magic numbers.** What does `-1` mean? What about `-2`? You have to read the docs.

### Python: Exceptions and None

Python replaces error codes with exceptions:

```python
def read_config(path):
    with open(path) as f:
        return parse(f.read())  # might raise FileNotFoundError, ParseError, ...

def init():
    cfg = read_config("app.conf")  # might explode
    use_config(cfg)
```

The problems:
1. **Invisible control flow.** Any function can raise any exception. You can't see it in the signature.
2. **Easy to forget.** Which exceptions should you catch? The type system doesn't tell you.
3. **Broad catches hide bugs.** `except Exception:` catches everything, including bugs.

Python uses `None` for "no value":

```python
def find_user(user_id):
    # returns None if not found
    ...

def greet_user(user_id):
    user = find_user(user_id)
    print(f"Hello, {user.name}")  # BUG: crashes if user is None
```

Same problem as C's `NULL` — the type doesn't distinguish "User" from "User or nothing."

---

## Option: A Value That Might Not Exist

Ferrum's `Option[T]` is a type that explicitly represents "a value of type T, or nothing":

```ferrum
enum Option[T] {
    Some(T),
    None,
}
```

That's the entire definition. An `Option[User]` is either `Some(user)` containing a `User`, or `None` containing nothing.

```ferrum
fn find_user(id: u64): Option[User] {
    if let Some(user) = database.get(id) {
        Some(user)
    } else {
        None
    }
}

fn greet_user(id: u64) {
    let user = find_user(id);
    println!("Hello, {}", user.name)  // ERROR: Option[User] has no field 'name'
}
```

The compiler stops you. You wrote `Option[User]`, not `User`. You have to handle the `None` case:

```ferrum
fn greet_user(id: u64) {
    match find_user(id) {
        Some(user) => println!("Hello, {}", user.name),
        None => println!("User not found"),
    }
}
```

**The key insight:** The possibility of "no value" is encoded in the type. You can't forget to check because the compiler won't let you treat an `Option[User]` as a `User`.

---

## Result: Success or Failure

`Result[T, E]` represents an operation that either succeeds with a value of type `T`, or fails with an error of type `E`:

```ferrum
enum Result[T, E] {
    Ok(T),
    Err(E),
}
```

Here's the config-reading example in Ferrum:

```ferrum
fn read_config(path: &str): Result[Config, ConfigError] {
    let contents = fs.read_to_string(path)?;  // might fail
    let config = parse(&contents)?;           // might fail
    Ok(config)
}

fn init(): Result[(), ConfigError] {
    let cfg = read_config("app.conf")?;
    use_config(&cfg);
    Ok(())
}
```

The `?` operator is shorthand for "if this is `Err`, return early with that error; otherwise, unwrap the `Ok` value." We'll cover it in detail below.

The type signature tells the whole story: `read_config` returns either a `Config` on success, or a `ConfigError` on failure. **The error channel is part of the type.** You can't ignore it.

```ferrum
fn init() {
    let cfg = read_config("app.conf");  // cfg is Result[Config, ConfigError]
    use_config(&cfg);  // ERROR: expected Config, got Result[Config, ConfigError]
}
```

You have to handle the error:

```ferrum
fn init() {
    match read_config("app.conf") {
        Ok(cfg) => use_config(&cfg),
        Err(e) => eprintln!("Failed to read config: {}", e),
    }
}
```

---

## The `?` Operator: Propagating Errors

Writing `match` everywhere is verbose. The `?` operator handles the common case: "if this failed, pass the error up to my caller."

```ferrum
fn process_file(path: &str): Result[Data, Error] {
    let contents = fs.read_to_string(path)?;  // returns Err early if read fails
    let parsed = parse(&contents)?;           // returns Err early if parse fails
    let validated = validate(&parsed)?;       // returns Err early if validate fails
    Ok(validated)
}
```

The `?` operator:
1. If the value is `Ok(x)`, unwraps it to `x`
2. If the value is `Err(e)`, returns `Err(e)` from the current function

This is equivalent to:

```ferrum
fn process_file(path: &str): Result[Data, Error] {
    let contents = match fs.read_to_string(path) {
        Ok(c) => c,
        Err(e) => return Err(e),
    };
    let parsed = match parse(&contents) {
        Ok(p) => p,
        Err(e) => return Err(e),
    };
    let validated = match validate(&parsed) {
        Ok(v) => v,
        Err(e) => return Err(e),
    };
    Ok(validated)
}
```

The `?` operator also works on `Option`:

```ferrum
fn get_user_email(id: u64): Option[String] {
    let user = find_user(id)?;        // returns None if user not found
    let profile = user.profile()?;    // returns None if no profile
    let email = profile.email()?;     // returns None if no email
    Some(email)
}
```

---

## Pattern Matching

The `match` expression handles all cases explicitly:

```ferrum
fn describe(opt: Option[i32]): String {
    match opt {
        Some(n) if n > 0 => format!("positive: {}", n),
        Some(n) if n < 0 => format!("negative: {}", n),
        Some(0) => "zero".to_string(),
        None => "nothing".to_string(),
    }
}
```

For `Result`:

```ferrum
fn handle(result: Result[Data, Error]) {
    match result {
        Ok(data) => process(data),
        Err(IoError(e)) => eprintln!("IO failed: {}", e),
        Err(ParseError(e)) => eprintln!("Parse failed: {}", e),
        Err(e) => eprintln!("Unknown error: {}", e),
    }
}
```

**The compiler enforces exhaustiveness.** If you forget a case, it's a compile error:

```ferrum
fn describe(opt: Option[i32]): String {
    match opt {
        Some(n) => format!("got {}", n),
        // ERROR: non-exhaustive match, missing `None`
    }
}
```

For quick checks, use `if let`:

```ferrum
if let Some(user) = find_user(id) {
    println!("Found: {}", user.name);
}

if let Err(e) = write_file(path, data) {
    eprintln!("Write failed: {}", e);
}
```

---

## Comparing to C

### Null Handling

**C:**
```c
User* user = find_user(id);
if (user != NULL) {
    printf("%s\n", user->name);
}
// Easy to forget the check. Compiler doesn't help.
```

**Ferrum:**
```ferrum
let user = find_user(id);
if let Some(u) = user {
    println!("{}", u.name);
}
// Can't forget. user is Option[User], not User.
```

### Error Handling

**C:**
```c
int result = do_operation(&data);
if (result < 0) {
    // handle error... but what error?
}
// Easy to ignore. Error type is just int.
```

**Ferrum:**
```ferrum
match do_operation() {
    Ok(data) => use(data),
    Err(IoError(e)) => handle_io(e),
    Err(ParseError(e)) => handle_parse(e),
}
// Can't ignore. Error type is explicit.
```

### Errno Pattern

**C:**
```c
FILE* f = fopen(path, "r");
if (f == NULL) {
    // check errno to find out what went wrong
    if (errno == ENOENT) { ... }
    else if (errno == EACCES) { ... }
}
```

**Ferrum:**
```ferrum
match fs.open(path) {
    Ok(f) => ...,
    Err(NotFound) => ...,
    Err(PermissionDenied) => ...,
    Err(e) => ...,
}
// Error is part of the return value, not a global.
```

---

## Comparing to Python

### None Handling

**Python:**
```python
user = find_user(id)
print(user.name)  # AttributeError if user is None
# Type checker (mypy) can help, but it's optional.
```

**Ferrum:**
```ferrum
let user = find_user(id);
println!("{}", user.name);  // Compile error: Option[User] has no field 'name'
// Type system is not optional.
```

### Exception Handling

**Python:**
```python
try:
    data = read_file(path)
    parsed = parse(data)
    validated = validate(parsed)
except FileNotFoundError:
    ...
except ParseError:
    ...
except ValidationError:
    ...
# Which line raised which exception? Have to trace through.
# What if read_file raises ParseError? Silent bug.
```

**Ferrum:**
```ferrum
let data = read_file(path)?;          // returns FileError
let parsed = parse(&data)?;           // returns ParseError
let validated = validate(&parsed)?;   // returns ValidationError
// Each operation's error type is explicit. No confusion.
```

### The "Catch Everything" Anti-Pattern

**Python:**
```python
try:
    do_stuff()
except Exception as e:
    log(e)
# This catches KeyboardInterrupt, bugs, everything. Bad.
```

**Ferrum:**
```ferrum
match do_stuff() {
    Ok(result) => result,
    Err(e) => {
        log(&e);
        // You explicitly handle the error type.
        // Panics (the Ferrum equivalent of bugs) are separate.
    }
}
```

---

## Common Methods

`Option` and `Result` have methods for common transformations:

### Option Methods

```ferrum
let opt: Option[i32] = Some(5);

opt.unwrap()              // returns 5, panics if None
opt.unwrap_or(0)          // returns 5, or 0 if None
opt.unwrap_or_else(|| compute_default())  // lazy default

opt.map(|n| n * 2)        // Some(10)
opt.and_then(|n| if n > 0 { Some(n) } else { None })

opt.is_some()             // true
opt.is_none()             // false

opt.ok_or(MyError)        // converts Option to Result
```

### Result Methods

```ferrum
let res: Result[i32, Error] = Ok(5);

res.unwrap()              // returns 5, panics if Err
res.unwrap_or(0)          // returns 5, or 0 if Err
res.expect("should work") // returns 5, panics with message if Err

res.map(|n| n * 2)        // Ok(10)
res.map_err(|e| wrap(e))  // transforms error type

res.is_ok()               // true
res.is_err()              // false

res.ok()                  // converts Result to Option, discarding error
```

---

## Why This Is Better

### 1. The Compiler Forces You to Handle Errors

In C and Python, error handling is optional. You can ignore return codes, skip null checks, and forget try/except. The compiler doesn't care.

In Ferrum, `Option` and `Result` are types. You can't use an `Option[User]` where a `User` is expected. You can't use a `Result[Data, Error]` where `Data` is expected. The compiler makes you handle the absence/error case.

### 2. No Hidden Control Flow

Python exceptions can jump from anywhere to anywhere. A function five levels deep can throw, and the exception bubbles up invisibly.

Ferrum's `?` operator is explicit. You see it in the code. It only propagates errors from that exact expression. No hidden jumps.

### 3. Errors Are Values, Not Magic

C's `errno` is a global. Python's exceptions are a separate language feature with special syntax.

Ferrum's `Result` is just a type. An enum with two variants. You can store it in a variable, put it in a collection, pass it to a function. No special rules.

### 4. Pattern Matching Is Exhaustive

When you `match` on an `Option` or `Result`, the compiler checks that you handled every case. Add a new error variant? The compiler tells you everywhere you need to update.

### 5. Documentation in the Type

```ferrum
fn read_file(path: &str): Result[String, IoError]
```

The signature tells you: this function might fail with an `IoError`. You don't have to read the implementation or hope the docs are up to date.

---

## When to Use Which

| Situation | Use |
|-----------|-----|
| Value might not exist | `Option[T]` |
| Lookup that might fail to find | `Option[T]` |
| Operation that can fail | `Result[T, E]` |
| Operation with multiple failure modes | `Result[T, E]` with enum `E` |
| Must never be null | Just `T` |

```ferrum
fn get(key: &str): Option[Value]           // might not exist
fn parse(s: &str): Result[Data, ParseError] // might fail
fn compute(x: i32): i32                     // always succeeds
```

---

## Summary

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| No value | `NULL` (unchecked) | `None` (unchecked) | `Option[T]` (must handle) |
| Errors | Return codes (easy to ignore) | Exceptions (invisible) | `Result[T, E]` (must handle) |
| Propagation | Check and return manually | Implicit throw | `?` operator (explicit) |
| Exhaustiveness | None | None | Compiler-enforced |
| Error info | Magic numbers, errno | Exception type | Error type in signature |

The fundamental improvement: **the type system encodes the possibility of absence or failure.** You can't accidentally ignore it. The compiler is your safety net.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete type system specification.*
