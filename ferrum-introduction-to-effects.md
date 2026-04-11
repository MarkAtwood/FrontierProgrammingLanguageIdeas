# Introduction to Effects in Ferrum

**Audience:** Programmers who know C and Python, new to effect systems

---

## The Problem Effects Solve

Consider this C function:

```c
int calculate(int x, int y) {
    return x + y;
}
```

And this one:

```c
int calculate(int x, int y) {
    printf("calculating...\n");
    return x + y;
}
```

Both have the same signature: `int calculate(int, int)`. But they're fundamentally different:

- The first is **pure** — call it a million times with the same inputs, you get the same output, and nothing else happens.
- The second **does IO** — it prints to stdout, which means it touches the outside world.

C's type system can't tell them apart. Neither can Python's. You have to read the implementation to know.

This matters because:

1. **Testing.** Pure functions are trivial to test. IO functions need mocks or real file handles.
2. **Parallelism.** Pure functions can run on any thread. IO functions might race.
3. **Caching.** Pure function results can be memoized freely. IO results can't.
4. **Reasoning.** Pure functions can be inlined, reordered, or eliminated. IO functions have to happen in order.

The compiler can't help you because the type doesn't carry this information.

---

## What Effects Are

An **effect** is something a function does besides computing a return value:

| Effect | What it means |
|--------|---------------|
| `IO` | Reads or writes files, console, environment |
| `Net` | Opens sockets, makes HTTP requests |
| `Async` | Suspends and resumes (awaits) |
| `Alloc` | Allocates memory |
| `Panic` | Might panic (unwind the stack) |
| `Unsafe` | Does something the compiler can't verify |

In most languages, any function can do any of these things at any time. The caller can't know without reading the source.

---

## Effects in Ferrum

Ferrum tracks effects in the type system. A function that does IO has `! IO` in its signature:

```ferrum
fn read_file(path: &str): Result[String, IoError] ! IO {
    // ... reads from disk
}
```

The `!` means "this function has effects." The `IO` names which effect.

A pure function has no effect annotation:

```ferrum
fn add(x: i32, y: i32): i32 {
    x + y
}
```

If you try to call an IO function from a pure function, the compiler stops you:

```ferrum
fn add(x: i32, y: i32): i32 {
    println("adding...")  // ERROR: cannot perform IO in pure function
    x + y
}
```

This is the guarantee: **if a function doesn't have `! IO`, it doesn't do IO.** You can trust the signature. You don't have to read the implementation.

---

## You Don't Write Effects Everywhere

Here's the key insight that makes this usable: **you only write effect annotations at module boundaries.**

Inside your module, the compiler infers effects:

```ferrum
// Private helper — no annotation needed
fn helper(x: i32): i32 {
    x * 2
}

// Private function that does IO — compiler infers ! IO
fn log_and_compute(x: i32): i32 {
    println("computing...")  // compiler sees this
    helper(x)
}

// Public API — you must declare effects
pub fn process(x: i32): i32 ! IO {
    log_and_compute(x)
}
```

The rule is simple:
- **Private functions:** effects inferred, no annotation needed
- **Public functions (`pub`):** effects declared explicitly

This means most code looks like regular C or Python — no effect annotations cluttering every line. The annotations appear only where they document your API.

---

## Comparing to C

In C, you might document effects in comments:

```c
// Pure function - no side effects
int add(int x, int y);

// Performs IO - writes to stdout
void greet(const char* name);

// Thread-unsafe - modifies global state
int get_next_id(void);
```

The problems:
1. Comments can lie or go stale
2. The compiler doesn't check them
3. You can call `greet()` from `add()` and nothing stops you

In Ferrum:

```ferrum
fn add(x: i32, y: i32): i32
pub fn greet(name: &str) ! IO
pub fn get_next_id(): i32 ! IO  // global state is IO
```

The compiler enforces these. Call `greet()` from `add()` and you get an error.

---

## Comparing to Python

Python uses conventions and hope:

```python
def add(x, y):
    """Pure function."""
    return x + y

def fetch_user(user_id):
    """Performs network IO."""
    return requests.get(f"/users/{user_id}")
```

In practice:
- Any function can do anything
- `add()` might secretly log, cache to disk, or send telemetry
- Testing requires mocking half the universe
- Async functions are a separate color (`async def`) but regular IO isn't marked at all

Ferrum makes the implicit explicit:

```ferrum
fn add(x: i32, y: i32): i32               // pure - guaranteed
pub fn fetch_user(id: u64): User ! Net    // network IO - declared
```

---

## Multiple Effects

Functions can have multiple effects:

```ferrum
pub fn download_and_save(url: &str, path: &str): Result[(), Error] ! Net + IO {
    let data = http.get(url)?   // Net
    fs.write(path, &data)?      // IO
    Ok(())
}
```

The `+` combines effects. The function does both network and file IO.

---

## Effect Polymorphism

Sometimes you write a function that passes through whatever effects its argument has:

```ferrum
pub fn retry[F, T](times: u32, f: F): Option[T]
where
    F: Fn(): Result[T, Error] ! ?E   // ?E means "whatever effects F has"
! ?E {
    for _ in 0..times {
        if let Ok(result) = f() {
            return Some(result)
        }
    }
    None
}
```

The `?E` is an effect variable. `retry` doesn't add effects — it has whatever effects `f` has. If you pass a pure function, `retry` is pure. If you pass an IO function, `retry` does IO.

---

## What This Gets You

**1. Fearless refactoring.** Extract a function and the compiler tells you its effects. No need to trace through the implementation.

**2. Honest APIs.** When a library says `fn compute(x: i32): i32`, you know it's pure. No hidden logging, no surprise network calls, no telemetry.

**3. Better testing.** Pure functions don't need mocks. Test them with plain inputs and outputs.

**4. Safe parallelism.** The compiler knows which functions can safely run in parallel (pure ones) and which need synchronization.

**5. Optimization opportunities.** The compiler can inline, cache, or eliminate pure function calls freely.

---

## The Escape Hatch

Sometimes you need to lie. Maybe you're implementing a memoization cache that's observationally pure but internally uses mutation:

```ferrum
// This function is pure from the caller's perspective
// but uses internal mutation for the cache
@pure  // trust me
fn expensive_computation(x: i32): i32 {
    static CACHE: Mutex[HashMap[i32, i32]] = ...
    // ...
}
```

The `@pure` attribute tells the compiler "I know this looks effectful, but trust me." This is an auditable escape hatch — you can grep for `@pure` to find all the places humans overrode the compiler.

---

## Summary

| Concept | C/Python | Ferrum |
|---------|----------|--------|
| Pure function | Convention, trust | Type-checked, guaranteed |
| IO function | Same signature as pure | `! IO` in signature |
| Effect checking | None | Compiler-enforced |
| Writing annotations | N/A | Only at `pub` boundaries |
| Escape hatch | Just do it | `@pure` attribute (auditable) |

Effects are types for "what else a function does." They make the implicit explicit, let the compiler help you, and document your APIs honestly — without cluttering every line of code.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete effect system specification.*
