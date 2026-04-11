# Introduction to Ownership in Ferrum

**Audience:** Programmers who know C and Python, new to ownership-based memory management

---

## The Problem Ownership Solves

Every program allocates memory. The question is: who frees it, and when?

### C: Manual Memory, No Safety

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

This works when you're careful. But C gives you no help when you're not:

**Use-after-free:**
```c
char* msg = greet("Alice");
free(msg);
printf("%s\n", msg);  // BUG: msg is freed, but compiler doesn't stop you
```

**Double-free:**
```c
char* msg = greet("Alice");
free(msg);
free(msg);  // BUG: double free — heap corruption, security vulnerability
```

**Memory leak:**
```c
void process() {
    char* msg = greet("Alice");
    if (error_condition) {
        return;  // BUG: forgot to free msg
    }
    free(msg);
}
```

The compiler sees nothing wrong with any of these. The bugs show up at runtime — or worse, they show up in production, or in a security audit.

### Python: Garbage Collection, No Control

Python takes the opposite approach: you never free anything.

```python
def greet(name):
    return f"Hello, {name}!"

msg = greet("Alice")
print(msg)
# no free() needed — garbage collector handles it
```

This is convenient. But you pay for it:

**Unpredictable pauses:** The garbage collector runs when it wants, not when you want. In latency-sensitive code, this is a problem.

**No control over memory layout:** You can't say "allocate this in a contiguous block" or "free this right now."

**No deterministic destruction:** If `msg` holds a file handle or a network connection, you don't know when it will be closed.

```python
def process():
    f = open("data.txt")
    data = f.read()
    # f is still open here — GC might close it eventually
    return data
```

Python programmers work around this with `with` statements, but that's convention, not enforcement.

### The Trade-off

C gives you control but no safety. Python gives you safety but no control.

Ownership gives you both.

---

## What Ownership Is

The rule is simple: **every value has exactly one owner.**

```ferrum
fn main() {
    let msg = String.from("hello")  // msg owns the string
    print(msg)                       // msg is still the owner
}   // msg goes out of scope — string is freed here, automatically
```

When the owner goes out of scope, the value is dropped. The compiler inserts the cleanup code for you. You don't write `free()`. You don't wait for a garbage collector. The memory is freed exactly when the owner's scope ends.

This is deterministic. This is automatic. And it's checked at compile time.

---

## Moving: Transferring Ownership

Ownership can be transferred from one variable to another. This is called a **move**.

```ferrum
fn main() {
    let a = String.from("hello")
    let b = a           // ownership moves from a to b
    // print(a)         // ERROR: a no longer owns the string
    print(b)            // fine — b is the owner now
}
```

After the move, `a` is no longer valid. The compiler enforces this. You cannot use a value after you've given it away.

### Why Moves Exist

Moves prevent double-free. If two variables owned the same value, they'd both try to free it when they went out of scope.

```ferrum
let a = String.from("hello")
let b = a   // move, not copy
// If this were a copy, both a and b would free the same memory
```

In C, this is a bug you have to avoid by convention:

```c
char* a = strdup("hello");
char* b = a;  // both point to same memory
free(a);
free(b);      // BUG: double free
```

In Ferrum, the compiler makes this impossible. After `let b = a`, the variable `a` doesn't exist anymore (from the compiler's perspective).

### Moves in Function Calls

Passing a value to a function moves it:

```ferrum
fn consume(s: String) {
    print(s)
}   // s is dropped here

fn main() {
    let msg = String.from("hello")
    consume(msg)    // ownership moves into consume()
    // print(msg)   // ERROR: msg was moved
}
```

This is different from C, where passing a pointer doesn't transfer ownership:

```c
void consume(char* s) {
    printf("%s\n", s);
}   // nothing happens to s

int main() {
    char* msg = strdup("hello");
    consume(msg);   // passes pointer, doesn't transfer ownership
    printf("%s\n", msg);  // still valid — but who frees it?
    free(msg);
    return 0;
}
```

In C, you have to remember who owns what. In Ferrum, the type system tracks it for you.

---

## Copy Types: When Moving Doesn't Happen

Some types are cheap to copy: integers, floats, booleans, characters. For these, Ferrum copies instead of moves:

```ferrum
fn main() {
    let x: i32 = 5
    let y = x       // copy, not move
    print(x)        // fine — x is still valid
    print(y)        // also fine
}
```

Types that implement the `Copy` trait are duplicated on assignment. This includes:
- All integer and floating-point types
- Booleans and characters
- References (`&T`)
- Tuples and arrays of `Copy` types

Types that own heap memory (like `String` or `Vec`) cannot be `Copy` — copying them would mean copying the heap allocation, which is expensive and must be explicit.

```ferrum
let a = String.from("hello")
let b = a.clone()   // explicit copy — you asked for it
print(a)            // fine
print(b)            // also fine
```

---

## Borrowing: Using Without Owning

Often you want to use a value without taking ownership. This is **borrowing**.

### Shared References: `&T`

A shared reference lets you read a value without owning it:

```ferrum
fn print_length(s: &String) {
    print(s.len())
}   // s goes out of scope, but the String is NOT dropped (we don't own it)

fn main() {
    let msg = String.from("hello")
    print_length(&msg)   // borrow msg
    print(msg)           // still valid — we only lent it
}
```

The `&` creates a reference. The function borrows the value, uses it, and returns. The original owner keeps ownership.

Compare to C:

```c
void print_length(const char* s) {
    printf("%zu\n", strlen(s));
}

int main() {
    char* msg = strdup("hello");
    print_length(msg);  // passes pointer
    printf("%s\n", msg); // still valid
    free(msg);
    return 0;
}
```

This looks similar, but C's `const char*` is a suggestion. You can cast it away:

```c
void sneaky(const char* s) {
    char* mutable = (char*)s;  // C allows this
    mutable[0] = 'X';          // modifies the original
}
```

Ferrum's `&T` is a guarantee. You cannot get a mutable reference from a shared reference.

### Exclusive References: `&mut T`

An exclusive reference lets you modify a value:

```ferrum
fn add_exclamation(s: &mut String) {
    s.push('!')
}

fn main() {
    let mut msg = String.from("hello")
    add_exclamation(&mut msg)
    print(msg)  // prints "hello!"
}
```

The `&mut` creates an exclusive reference. While it exists, no other reference (shared or exclusive) to that value can exist.

---

## The Borrowing Rules

Ferrum enforces two rules at compile time:

1. **You can have many shared references, OR one exclusive reference — not both.**
2. **References cannot outlive their owner.**

### Rule 1: Shared XOR Exclusive

This is valid — multiple shared references:

```ferrum
let s = String.from("hello")
let r1 = &s
let r2 = &s
print(r1)  // fine
print(r2)  // fine
```

This is valid — one exclusive reference:

```ferrum
let mut s = String.from("hello")
let r = &mut s
r.push('!')
print(r)  // fine
```

This is invalid — shared and exclusive at the same time:

```ferrum
let mut s = String.from("hello")
let r1 = &s
let r2 = &mut s   // ERROR: cannot borrow as mutable while borrowed as shared
print(r1)
```

### Why This Rule Exists

This prevents data races at compile time.

In C, nothing stops you from doing this:

```c
void reader(const char* s) {
    while (*s) {
        putchar(*s++);
    }
}

void writer(char* s) {
    strcpy(s, "CHANGED!");
}

// What if reader and writer run on different threads?
// Data race. Undefined behavior.
```

In Ferrum, the compiler prevents this. If someone has a shared reference, no one can modify the value. If someone has an exclusive reference, no one else can access the value at all.

### Rule 2: References Can't Outlive Owners

This is invalid — reference outlives owner:

```ferrum
fn dangle(): &String {
    let s = String.from("hello")
    &s  // ERROR: s is dropped at end of function, reference would dangle
}
```

In C, this compiles and creates a dangling pointer:

```c
char* dangle() {
    char buf[64] = "hello";
    return buf;  // WARNING (maybe), but compiles
}
// The returned pointer points to freed stack memory
```

Ferrum makes this a compile error. The reference cannot outlive the value it points to.

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
| Shared access | `const T*` (advisory) | Default behavior | `&T` (enforced) |
| Exclusive access | `T*` | No concept | `&mut T` (enforced) |
| Aliasing | Unrestricted | Unrestricted | Controlled |
| Data races | Your problem | GIL hides them | Compile error |

### Ownership Tracking

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Who frees memory? | You decide | GC decides | Owner, at scope exit |
| Transfer ownership | Convention | N/A (GC) | Move semantics |
| Temporary access | Raw pointer | N/A | Borrowing |
| Compiler help | None | None | Full verification |

---

## Lifetime Annotations: Usually Not Needed

You might have heard that ownership systems require complex lifetime annotations everywhere. In Ferrum, this is mostly false.

The compiler infers lifetimes in the common cases:

```ferrum
// No annotations needed — compiler infers the relationship
fn first_word(s: &str): &str {
    match s.find(' ') {
        Some(i) => &s[0..i],
        None => s,
    }
}
```

The compiler sees that the returned reference comes from `s`, so it must not outlive `s`. This is inferred automatically.

**When do you need annotations?** When the relationship between input and output lifetimes is genuinely ambiguous:

```ferrum
// Ambiguous: does result come from a, b, or either?
// Compiler can't know without annotation
fn choose<'a>(a: &'a str, b: &'a str, pick_first: bool): &'a str {
    if pick_first { a } else { b }
}
```

The `'a` is a lifetime annotation. It says: "the return value lives as long as both inputs do."

**The 90% rule:** Ferrum's inference eliminates annotations in at least 90% of practical code. The remaining 10% — complex borrowing patterns at API boundaries — requires annotations. And those annotations carry genuine semantic content that a reader needs.

---

## What This Gets You

**No memory bugs.** Use-after-free, double-free, dangling pointers — these become compile errors, not runtime surprises.

**Deterministic cleanup.** Resources are freed exactly when their owner goes out of scope. Files are closed. Connections are dropped. Memory is reclaimed. No GC pause, no `with` statement, no manual `free()`.

**Fearless concurrency.** The borrowing rules prevent data races at compile time. If your code compiles, it doesn't have data races on borrowed data.

**Zero runtime cost.** Ownership is a compile-time concept. There's no reference counting, no garbage collector, no runtime bookkeeping. The generated code is as efficient as hand-written C.

**Clear APIs.** Function signatures tell you who owns what. `fn consume(s: String)` takes ownership. `fn borrow(s: &String)` borrows temporarily. `fn modify(s: &mut String)` borrows exclusively. The intent is in the type.

---

## Summary

| Concept | What it means |
|---------|---------------|
| Ownership | Every value has exactly one owner |
| Drop | Value is freed when owner goes out of scope |
| Move | Ownership transfers; old variable becomes invalid |
| Copy | Small types are duplicated instead of moved |
| Shared borrow (`&T`) | Read-only access without taking ownership |
| Exclusive borrow (`&mut T`) | Read-write access, no other references allowed |
| Borrowing rules | Many `&T` OR one `&mut T`, never both |
| Lifetime inference | Compiler figures out lifetimes in most cases |

Ownership is Ferrum's answer to the memory management question. It gives you C's control with Python's safety — and the compiler checks it all at compile time.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete ownership and borrowing specification.*
