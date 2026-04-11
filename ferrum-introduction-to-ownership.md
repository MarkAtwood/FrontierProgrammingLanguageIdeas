# Introduction to Ownership in Ferrum

**Audience:** Programmers who know C and Python, new to ownership-based memory management

---

## The Problem Ownership Solves

Every program allocates memory. The question is: who frees it, and when?

If you've written C, you've hit bugs where you freed memory twice, or forgot to free it, or used it after freeing. If you've written Python, you've hit moments where the garbage collector paused at the worst time, or a file handle stayed open longer than you wanted.

Ownership is a third approach. It gives you C's control over when memory is freed, with Python's guarantee that you won't screw it up.

### C: Manual Memory, No Safety Net

In C, you allocate and free memory yourself:

```c
char* greet(const char* name) {
    char* buf = malloc(64);
    sprintf(buf, "Hello, %s!", name);
    return buf;
}

int main() {
    char* msg = greet("Alice");
    printf("%s\n", msg);
    free(msg);  // you must remember to free
    return 0;
}
```

This works when you're careful. But C gives you no help when you're not.

**Use-after-free** - you've probably hit this one:

```c
char* msg = greet("Alice");
free(msg);
printf("%s\n", msg);  // BUG: msg is freed, but compiler doesn't stop you
```

What happens? Maybe it prints garbage. Maybe it crashes. Maybe it prints "Hello, Alice!" and works fine in testing, then crashes in production when a different allocation reuses that memory. You've probably spent hours debugging something like this.

**Double-free** - subtle and dangerous:

```c
char* msg = greet("Alice");
free(msg);
// ... 100 lines of code later, in a different function ...
free(msg);  // BUG: double free — heap corruption
```

Double-frees don't always crash immediately. Sometimes they corrupt the heap in ways that cause crashes much later, in completely unrelated code. Security researchers love these bugs because they're exploitable.

**Memory leak** - the slow death:

```c
void process() {
    char* msg = greet("Alice");
    if (error_condition) {
        return;  // BUG: forgot to free msg
    }
    free(msg);
}
```

Your program works fine for an hour, then the server runs out of memory and gets OOM-killed at 3am. You've seen this movie.

The compiler sees nothing wrong with any of these. The bugs show up at runtime - or worse, in production, or in a security audit.

### Python: Garbage Collection, No Control

Python takes the opposite approach: you never free anything.

```python
def greet(name):
    return f"Hello, {name}!"

msg = greet("Alice")
print(msg)
# no free() needed — garbage collector handles it
```

This is convenient. You can't double-free or use-after-free because you never free. But you pay for it:

**Unpredictable pauses.** The garbage collector runs when it decides to, not when you want. If you're writing a game that needs to hit 60fps, or a trading system where microseconds matter, GC pauses are a problem.

```python
# This might pause for 50ms in the middle of your tight loop
for i in range(1000000):
    process_frame()  # GC could kick in here, any iteration
```

**No control over when resources are released.** This is the one that bites Python programmers most often:

```python
def process():
    f = open("data.txt")
    data = f.read()
    # f is still open here
    # GC might close it in 1ms, or 10 seconds, or never
    return data
```

You've probably learned to write `with open(...) as f:` to avoid this. But that's a convention you have to remember. The language doesn't force you to do it right.

Worse, some resources don't work with `with`:

```python
def get_connection():
    conn = database.connect()
    return conn
    # When does conn get closed? When GC feels like it.
    # Hope you don't run out of database connections first.
```

### The Trade-off

C gives you control but no safety. Python gives you safety but no control.

Ownership gives you both: you control exactly when things are freed, and the compiler guarantees you can't screw it up.

---

## What Ownership Is

The rule is simple: **every value has exactly one owner.**

```ferrum
fn main() {
    let msg = String.from("hello")  // msg owns the string
    print(msg)                       // msg is still the owner
}   // msg goes out of scope — string is freed here, automatically
```

When the owner goes out of scope, the value is freed. The compiler inserts the cleanup code. You don't write `free()`. You don't wait for a garbage collector. The memory is freed *exactly* when the owner's scope ends.

Think of it like this: in C, `free()` is something you call. In Ferrum, `free()` is something the compiler writes for you, in exactly the right place, every time.

**What does "goes out of scope" mean?**

Scope is the region of code where a variable is valid. In most cases, it's the block (the `{ }`) where the variable was created:

```ferrum
fn main() {
    let x = 1           // x comes into scope
    {
        let y = 2       // y comes into scope
        print(x + y)    // both x and y are valid here
    }                   // y goes out of scope, y is freed
    print(x)            // x is still valid
}                       // x goes out of scope, x is freed
```

If `y` owned heap memory, that memory would be freed at the inner `}`. If `x` owned heap memory, it would be freed at the outer `}`. No guessing. No garbage collector deciding when. The scope determines when.

---

## Moving: Transferring Ownership

What happens when you assign one variable to another?

```ferrum
fn main() {
    let a = String.from("hello")
    let b = a           // What happens here?
    print(b)
}
```

In C, `b = a` would make `b` point to the same memory as `a`. Both variables would be valid, both would point to the same string, and you'd have to remember not to free it twice.

In Ferrum, `b = a` **moves** ownership from `a` to `b`. After this line, `a` is no longer valid:

```ferrum
fn main() {
    let a = String.from("hello")
    let b = a           // ownership moves from a to b
    // print(a)         // ERROR: a is no longer valid
    print(b)            // fine — b is the owner now
}
```

The compiler enforces this. If you uncomment `print(a)`, you get a compile error, not a runtime bug.

### Why Does Moving Exist?

Moves solve the double-free problem.

If two variables owned the same value, they'd both try to free it when they went out of scope:

```ferrum
let a = String.from("hello")
let b = a
// If both a and b were valid owners...
}   // b goes out of scope, frees the string
}   // a goes out of scope, tries to free the same string — DOUBLE FREE
```

Ferrum prevents this by making `a` invalid after the move. Only `b` is an owner. Only `b` will free the string.

Compare to the C bug you've probably written:

```c
char* a = strdup("hello");
char* b = a;  // both point to same memory — C allows this
// ... later ...
free(a);
free(b);      // BUG: double free — but C compiles this happily
```

In Ferrum, this situation is impossible. The compiler tracks ownership and prevents you from using a moved value.

### What Does the Compiler Error Look Like?

When you try to use a moved value, the compiler tells you exactly what happened:

```
error: use of moved value: `a`
  --> src/main.fe:4:11
   |
2  |     let a = String.from("hello")
   |         - move occurs because `a` has type `String`
3  |     let b = a
   |             - value moved here
4  |     print(a)
   |           ^ value used here after move
```

The error shows you where the move happened and where you tried to use the moved value. You don't have to hunt through a debugger to find the bug.

### Moving in Function Calls

Passing a value to a function also moves it:

```ferrum
fn consume(s: String) {
    print(s)
}   // s goes out of scope, string is freed here

fn main() {
    let msg = String.from("hello")
    consume(msg)        // ownership moves into consume()
    // print(msg)       // ERROR: msg was moved into consume()
}
```

When `consume(msg)` is called, ownership of the string transfers to the `s` parameter inside `consume`. When `consume` returns, `s` goes out of scope and the string is freed.

Back in `main`, `msg` is no longer valid. You gave it away.

**In C, this is ambiguous:**

```c
void consume(char* s) {
    printf("%s\n", s);
}   // nothing happens to s

int main() {
    char* msg = strdup("hello");
    consume(msg);   // passes pointer — but who owns the memory?
    printf("%s\n", msg);  // still valid? depends on what consume() does
    free(msg);      // you have to know consume() didn't free it
    return 0;
}
```

In C, you have to read the documentation (if there is any) to know whether a function takes ownership or just borrows. In Ferrum, the type signature tells you. `fn consume(s: String)` takes ownership. You can see it.

---

## Copy Types: When Moving Doesn't Happen

Some types are so small and cheap that copying them is trivial. For these types, assignment copies instead of moves:

```ferrum
fn main() {
    let x: i32 = 5
    let y = x       // copy, not move
    print(x)        // fine — x is still valid
    print(y)        // also fine — y has its own copy
}
```

This makes sense when you think about what's happening at the machine level. An `i32` is 4 bytes. Copying 4 bytes is as fast as copying a pointer would be. There's no benefit to moving.

Types that copy instead of moving:
- All integer types (`i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`)
- All floating-point types (`f32`, `f64`)
- Booleans (`bool`)
- Characters (`char`)
- Tuples containing only copyable types, like `(i32, i32)`
- Fixed-size arrays of copyable types, like `[i32; 4]`

### Why Can't String Be Copied?

A `String` owns heap-allocated memory. If `String` were copyable, what would `let b = a` do?

Option 1: Copy the pointer. Now `a` and `b` both point to the same heap memory. Double-free bug when both go out of scope.

Option 2: Copy the heap memory. Allocate a new buffer, copy all the bytes. This is expensive — what if the string is 10MB?

Ferrum chooses neither. It moves. If you actually want to copy the heap data, you have to say so explicitly:

```ferrum
let a = String.from("hello")
let b = a.clone()   // explicit copy: allocates new memory, copies the bytes
print(a)            // fine — a still owns its copy
print(b)            // fine — b owns a separate copy
```

`.clone()` tells the reader "yes, I know this might be expensive, I want it anyway."

---

## Borrowing: Using Without Owning

Moving is too restrictive for many cases. What if you want to use a value, but not consume it?

```ferrum
fn print_length(s: String) {
    print(s.len())
}   // s is dropped here — the string is freed!

fn main() {
    let msg = String.from("hello")
    print_length(msg)    // ownership moves to print_length
    // We can't use msg anymore! But we just wanted to print its length.
}
```

This would be annoying. You'd have to clone everything, or have functions return the values they received.

Instead, Ferrum lets you **borrow** a value. Borrowing gives you access to a value without taking ownership.

### Shared Borrows: `&T`

A shared borrow lets you read a value:

```ferrum
fn print_length(s: &String) {   // note the &
    print(s.len())
}   // s goes out of scope, but the String is NOT freed (we don't own it)

fn main() {
    let msg = String.from("hello")
    print_length(&msg)   // lend msg to print_length
    print(msg)           // still valid — we still own it
}
```

The `&` creates a reference. Think of it as a pointer that the compiler tracks. The function borrows the value, uses it, and gives it back when it returns.

**In C, this is similar to passing a pointer:**

```c
void print_length(const char* s) {
    printf("%zu\n", strlen(s));
}

int main() {
    char* msg = strdup("hello");
    print_length(msg);  // passes pointer
    printf("%s\n", msg); // still valid — we still own it
    free(msg);
    return 0;
}
```

The difference is what the compiler guarantees. C's `const char*` is a hint. You can cast it away and modify the string:

```c
void sneaky(const char* s) {
    char* mutable = (char*)s;  // C allows this cast
    mutable[0] = 'X';          // modifies the original string!
}
```

You've probably been bitten by code that does this, or by code that accidentally passes a non-const pointer to a function that modifies it.

Ferrum's `&T` is a guarantee. A shared borrow cannot modify the value. Period. There's no escape hatch.

### Exclusive Borrows: `&mut T`

What if you need to modify the borrowed value? Use an exclusive borrow:

```ferrum
fn add_exclamation(s: &mut String) {
    s.push('!')
}

fn main() {
    let mut msg = String.from("hello")  // note: msg must be declared mut
    add_exclamation(&mut msg)
    print(msg)  // prints "hello!"
}
```

`&mut` creates an exclusive borrow. While this borrow exists, you have full read-write access to the value. But no one else can access it at all.

---

## The Borrowing Rules

Ferrum enforces two rules about borrows. These rules catch bugs at compile time that would be runtime nightmares in C.

### Rule 1: Shared OR Exclusive, Never Both

At any moment, a value can have:
- Any number of shared borrows (`&T`), OR
- Exactly one exclusive borrow (`&mut T`)

Never both at the same time.

**Multiple shared borrows - fine:**

```ferrum
let s = String.from("hello")
let r1 = &s
let r2 = &s
let r3 = &s
print(r1)  // fine
print(r2)  // fine
print(r3)  // fine
```

Everyone's just reading. No conflict.

**One exclusive borrow - fine:**

```ferrum
let mut s = String.from("hello")
let r = &mut s
r.push('!')
print(r)  // fine
```

One writer, no readers. No conflict.

**Shared and exclusive at the same time - compiler error:**

```ferrum
let mut s = String.from("hello")
let r1 = &s           // shared borrow
let r2 = &mut s       // ERROR: can't borrow as exclusive while shared borrow exists
print(r1)
```

The compiler catches this:

```
error: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.fe:3:10
  |
2 | let r1 = &s
  |          -- immutable borrow occurs here
3 | let r2 = &mut s
  |          ^^^^^^ mutable borrow occurs here
4 | print(r1)
  |       -- immutable borrow later used here
```

### Why Does This Rule Exist?

This rule prevents a whole class of bugs: reading data while someone else is modifying it.

**Here's a C bug you might have written:**

```c
void process_items(struct Vector* v) {
    for (int i = 0; i < v->length; i++) {
        if (should_add_more(v->items[i])) {
            vector_push(v, new_item());  // BUG: modifies v while we're iterating
        }
    }
}
```

Pushing to a vector might reallocate its internal buffer. Now the loop is reading from freed memory. This is undefined behavior.

**Or this Python surprise:**

```python
items = [1, 2, 3, 4, 5]
for item in items:
    if item % 2 == 0:
        items.remove(item)  # BUG: modifying list while iterating
print(items)  # prints [1, 3, 5]? [1, 2, 3, 5]? depends on Python version
```

Python doesn't crash, but you get wrong results. The iterator gets confused when the list changes underneath it.

**In Ferrum, the compiler prevents this:**

```ferrum
let mut items = vec[1, 2, 3, 4, 5]
for item in &items {           // shared borrow of items
    if item % 2 == 0 {
        items.remove(item)     // ERROR: can't mutate items while iterating
    }
}
```

The `for` loop holds a shared borrow of `items`. You can't mutate `items` while that borrow exists. The compiler forces you to find a different approach (like collecting indices first, then removing).

### Rule 2: Borrows Can't Outlive the Owner

A borrow can't exist longer than the thing it borrows from:

```ferrum
fn dangle(): &String {
    let s = String.from("hello")
    &s  // ERROR: s will be dropped, but we're returning a reference to it
}
```

The compiler catches this:

```
error: `s` does not live long enough
 --> src/main.fe:3:5
  |
2 |     let s = String.from("hello")
  |         - binding `s` declared here
3 |     &s
  |     ^^ borrowed value does not live long enough
4 | }
  | - `s` dropped here while still borrowed
```

**This is the dangling pointer bug you've debugged in C:**

```c
char* dangle() {
    char buf[64] = "hello";
    return buf;  // returns pointer to stack memory that's about to be freed
}

int main() {
    char* s = dangle();
    printf("%s\n", s);  // undefined behavior: reading freed stack memory
}
```

GCC might warn you about this specific case, but C can't catch it in general. Ferrum catches all cases.

---

## A Complete Example

Let's put it together with a more realistic example. Say you're building a function to format a user's full name:

```ferrum
struct User {
    first_name: String,
    last_name: String,
}

// Takes ownership of first_name and last_name
fn create_user(first_name: String, last_name: String): User {
    User { first_name, last_name }
}

// Borrows the user, doesn't consume it
fn format_name(user: &User): String {
    format("{} {}", user.first_name, user.last_name)
}

// Exclusively borrows the user to modify it
fn set_last_name(user: &mut User, new_name: String) {
    user.last_name = new_name
}

fn main() {
    // create_user takes ownership of the strings we pass
    let mut user = create_user(
        String.from("Alice"),
        String.from("Smith")
    )

    // format_name borrows user, so we can use user again after
    let name = format_name(&user)
    print(name)  // prints "Alice Smith"

    // set_last_name exclusively borrows user to modify it
    set_last_name(&mut user, String.from("Jones"))

    // user is still valid, just modified
    print(format_name(&user))  // prints "Alice Jones"
}
```

The function signatures tell you everything:
- `create_user(first_name: String, ...)` - takes ownership, consumes the strings
- `format_name(user: &User)` - borrows, doesn't modify, you keep ownership
- `set_last_name(user: &mut User, ...)` - borrows exclusively, will modify

No documentation needed. The types are the documentation.

---

## Comparing to C and Python

### Memory Allocation and Deallocation

| Scenario | C | Python | Ferrum |
|----------|---|--------|--------|
| Allocate | `malloc()` | automatic | automatic |
| Deallocate | `free()` (manual) | GC (unpredictable) | scope exit (deterministic) |
| Use-after-free | Runtime bug | Impossible | Compile error |
| Double-free | Runtime bug | Impossible | Compile error |
| Memory leak | Common | Rare (GC) | Rare (ownership) |
| Dangling pointer | Common | Impossible | Compile error |

### Reference Semantics

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Shared access | `const T*` (advisory, can cast away) | Default | `&T` (enforced by compiler) |
| Exclusive access | `T*` (nothing prevents aliasing) | No concept | `&mut T` (enforced) |
| Aliasing | Unrestricted | Unrestricted | Controlled |
| Iterator invalidation | Your problem | Sometimes wrong results | Compile error |
| Data races | Your problem | GIL hides most | Compile error |

### Ownership Tracking

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Who frees memory? | You decide (and hope you're right) | GC decides (eventually) | Owner, at scope exit |
| Transfer ownership | Convention and comments | N/A (GC) | Move semantics (enforced) |
| Temporary access | Raw pointer (unrestricted) | N/A | Borrowing (tracked) |
| Compiler help | Almost none | None | Full verification |

---

## Lifetime Annotations: When You Need Them

Most of the time, Ferrum figures out lifetime relationships automatically. But sometimes you need to help.

### When the Compiler Can Figure It Out

```ferrum
fn first_word(s: &str): &str {
    match s.find(' ') {
        Some(i) => &s[0..i],
        None => s,
    }
}
```

The compiler sees: you take a reference in (`&str`), you return a reference out (`&str`). The output must come from the input. So the output can't outlive the input. Done.

No annotations needed.

### When the Compiler Needs Help

What if you have two input references?

```ferrum
fn choose(a: &str, b: &str, pick_first: bool): &str {
    if pick_first { a } else { b }
}
```

The compiler can't figure this out. The return value could come from `a` or `b`. How long does the result live? As long as `a`? As long as `b`? The compiler doesn't know.

You tell it with a lifetime annotation:

```ferrum
fn choose<'a>(a: &'a str, b: &'a str, pick_first: bool): &'a str {
    if pick_first { a } else { b }
}
```

The `'a` (read as "lifetime a") is a label. It says: "both inputs have the same lifetime, and the output has that lifetime too."

In practice, this means: the result is valid as long as *both* inputs are valid. If either input goes away, the result becomes invalid.

**You can read `'a` as "I'm telling the compiler these things are connected."**

### When Do You Actually Need This?

In practice, lifetime annotations show up in a few places:

1. **Functions that return references and have multiple reference inputs** - like `choose` above
2. **Structs that hold references** - the struct needs to know how long the reference lives
3. **Complex API boundaries** - when the relationship isn't obvious from context

In everyday code - reading files, processing data, handling requests - you rarely write lifetime annotations. The compiler figures it out.

---

## Common Questions

### "What if I Need Two Mutable References?"

You can't have two `&mut` to the same data at the same time. But you often don't need to.

**Instead of:**
```ferrum
// This doesn't work
let r1 = &mut data
let r2 = &mut data
modify_both(r1, r2)
```

**Think about what you're actually doing:**
```ferrum
// Often you want sequential mutations
modify_first(&mut data)
modify_second(&mut data)
```

Or if you really need two mutable views, you might need different data structures (like splitting a vector into two halves, each with its own mutable reference).

### "Can I Opt Out of This?"

Ferrum has an `unsafe` escape hatch for low-level code that needs to bypass the rules. But if you're asking this question as a beginner, the answer is: you don't need it yet.

The ownership rules aren't arbitrary restrictions. They're catching real bugs. If the compiler is complaining, there's usually a better way to structure your code.

### "Is This Slower Than C?"

No. Ownership is purely a compile-time concept. The generated machine code doesn't know about ownership, borrows, or lifetimes. It's just loads, stores, and frees - exactly like hand-written C, but with the compiler having proven there are no bugs.

There's no reference counting. No garbage collector. No runtime bookkeeping.

---

## Summary

| Concept | What it means |
|---------|---------------|
| Ownership | Every value has exactly one owner |
| Drop | Value is freed when owner goes out of scope |
| Move | Ownership transfers; old variable becomes invalid |
| Copy | Small, simple types are duplicated instead of moved |
| Shared borrow (`&T`) | Read-only access, doesn't take ownership |
| Exclusive borrow (`&mut T`) | Read-write access, no other references allowed |
| Borrowing rules | Many `&T` OR one `&mut T`, never both |
| Lifetime | Compiler tracks how long references are valid |

The ownership system prevents at compile time:
- Use-after-free
- Double-free
- Dangling pointers
- Data races on borrowed data
- Iterator invalidation bugs

The cost is learning to think about ownership. The benefit is never debugging these bugs again.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete ownership and borrowing specification.*
