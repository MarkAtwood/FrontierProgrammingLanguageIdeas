# Introduction to Generics in Ferrum

**Audience:** Programmers who know C and Python, but haven't used a language with generics before.

---

## What Are Generics?

Generics let you write code that works with any type, while still catching type errors at compile time. You write one function or data structure, and the compiler generates specialized versions for each type you actually use.

If you've ever:
- Written the same function three times for `int`, `double`, and `char`
- Debugged a crash caused by casting `void*` to the wrong type
- Had Python code work perfectly in testing, then blow up in production with an `AttributeError`

...then generics solve your problem.

---

## The Problem: Code Duplication in C

You need a function that swaps two values. In C:

```c
void swap_int(int* a, int* b) {
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

void swap_double(double* a, double* b) {
    double tmp = *a;
    *a = *b;
    *b = tmp;
}

void swap_char(char* a, char* b) {
    char tmp = *a;
    *a = *b;
    *b = tmp;
}
```

Three functions that do the exact same thing, differing only in type. If you have 10 types, you write 10 functions. Copy, paste, change the type, hope you didn't make a typo. Every time you fix a bug, you fix it in 10 places.

Now imagine a more complex data structure: a resizable array, a hash table, a binary tree. In a large C codebase, you'll find `IntList`, `StringList`, `NodeList`, `ConnectionList`... each a separate copy-paste of the same logic.

C offers two escape hatches. Both have serious problems.

---

## C's Approach 1: void* (Type Erasure)

You can use `void*` to write a single function that works with any type:

```c
void swap(void* a, void* b, size_t size) {
    char tmp[size];  // VLA or alloca
    memcpy(tmp, a, size);
    memcpy(a, b, size);
    memcpy(b, tmp, size);
}

// Usage
int x = 1, y = 2;
swap(&x, &y, sizeof(int));  // works

double d1 = 1.0, d2 = 2.0;
swap(&d1, &d2, sizeof(double));  // works
```

So far so good. But here's where it gets dangerous:

```c
// These all compile without warnings:
swap(&x, &d1, sizeof(int));     // int and double - corrupts memory
swap(&x, &y, sizeof(double));   // wrong size - corrupts adjacent memory
swap(&x, &y, 1);                // too small - partial swap, garbage result
```

The compiler can't help you because `void*` erases all type information. You're back to the 1970s: "here's a pointer to some bytes, good luck."

**Real-world pain with void*:**

```c
// A "generic" linked list
typedef struct Node {
    void* data;
    struct Node* next;
} Node;

void list_append(Node** head, void* data);
void* list_get(Node* head, int index);

// Usage
Node* users = NULL;
User* alice = create_user("Alice");
list_append(&users, alice);

// Later, someone writes:
char* name = (char*)list_get(users, 0);  // WRONG! It's a User*, not char*
printf("Name: %s\n", name);              // Undefined behavior, possible crash
```

The bug might not crash immediately. It might corrupt memory silently, then crash hours later in unrelated code. Good luck debugging that.

**The void* pain checklist:**
1. **No type safety.** The compiler can't check that you're using the right type. Bugs become runtime crashes (or worse, silent corruption).
2. **Manual size tracking.** You have to pass `sizeof(T)` everywhere and get it right every time.
3. **Casts everywhere.** Every time you get data back, you cast it. Every cast is a potential bug.
4. **No IDE help.** Autocomplete shows `void*`. Refactoring tools can't track what type you meant.
5. **Performance overhead.** The `memcpy` calls happen at runtime. The compiler can't optimize because it doesn't know the actual types.

---

## C's Approach 2: Macros (Text Substitution)

Macros let you generate type-specific code:

```c
#define SWAP(type, a, b) do { \
    type tmp = (a);           \
    (a) = (b);                \
    (b) = tmp;                \
} while(0)

// Usage
int x = 1, y = 2;
SWAP(int, x, y);  // expands to: do { int tmp = (x); (x) = (y); (y) = tmp; } while(0)
```

This avoids the `void*` overhead, but introduces new problems:

```c
SWAP(int, x, 1.5);  // silently truncates the double to int

SWAP(int, x, arr[i++]);  // BUG: i++ happens twice because 'a' appears twice in macro

SWAP(int, x, "hello");   // error message will be incomprehensible
```

**What macro error messages actually look like:**

```c
SWAP(int, x, "hello");
```

Compiler output:
```
main.c:47:5: error: incompatible types when assigning to type 'int' from type 'char *'
   47 |     SWAP(int, x, "hello");
      |     ^~~~
main.c:12:14: note: in definition of macro 'SWAP'
   12 |     (a) = (b);                \
      |              ^
```

You have to mentally expand the macro, figure out which `(a)` or `(b)` it's complaining about, and map that back to your code. With nested macros, this becomes a nightmare.

**Debugging macros is worse:**
- You can't step through a macro in gdb/lldb
- Breakpoints don't work inside macro bodies
- Error line numbers point to the expansion, not your code

**Macros for data structures are even worse:**

```c
#define DEFINE_VECTOR(type, name) \
    typedef struct {              \
        type* data;               \
        size_t len;               \
        size_t cap;               \
    } name##_Vec;                 \
    \
    void name##_push(name##_Vec* v, type value) { \
        /* ... 20 lines of memory management ... */ \
    } \
    /* ... more functions ... */

DEFINE_VECTOR(int, Int)
DEFINE_VECTOR(double, Double)
DEFINE_VECTOR(User, User)
```

Now you have to debug macro-generated code. When `Int_push` crashes, the stack trace shows line numbers inside the macro definition, not where you called it.

C11 added `_Generic` for type-based dispatch:

```c
#define swap(a, b) _Generic((a), \
    int*: swap_int, \
    double*: swap_double \
)(a, b)
```

But you still have to write each type-specific function. `_Generic` just picks which one to call.

---

## Python's Approach: Dynamic Typing (Check Types at Runtime)

In Python, you don't declare types, so everything is "generic" by default:

```python
def swap(a, b):
    return b, a

x, y = 1, 2
x, y = swap(x, y)  # works

x, y = "hello", "world"
x, y = swap(x, y)  # works
```

This is convenient. But the flexibility comes with a cost.

**The "it works until it doesn't" problem:**

```python
def process_users(users):
    for user in users:
        print(f"Welcome, {user.name}!")

# Works in testing
process_users([User("Alice"), User("Bob")])

# Production data has a bug - one element is a dict instead of User
process_users([User("Alice"), {"name": "Bob"}])  # works by accident!

# Until someone adds:
def process_users(users):
    for user in users:
        print(f"Welcome, {user.name}!")
        user.update_last_login()  # AttributeError: 'dict' has no attribute 'update_last_login'
```

That `AttributeError` happens at 3am when your on-call engineer is trying to figure out why half the users didn't get their welcome emails.

**Python bugs hide until they execute:**

```python
def calculate_discount(price, discount_percent):
    return price * (1 - discount_percent / 100)

# This bug won't be caught until someone calls it with strings:
calculate_discount("100", "20")  # TypeError at runtime
```

Your test suite might not cover this code path. Your type checker (if you use one) might not be configured to catch it. The bug ships to production.

**Python's type hints help, but don't enforce:**

```python
def calculate_discount(price: float, discount_percent: float) -> float:
    return price * (1 - discount_percent / 100)

calculate_discount("100", "20")  # Still runs! Type hints are just comments.
```

Type checkers like mypy can catch this, but they're optional, often not configured strictly, and don't run at runtime.

---

## Generics: The Best of Both Worlds

Generics give you:
- Write code once, works with any type (like Python)
- Full type safety at compile time (better than C)
- Zero runtime cost (faster than Python, as fast as hand-written C)

Here's the swap function in Ferrum:

```ferrum
fn swap[T](a: &mut T, b: &mut T) {
    let tmp = *a;
    *a = *b;
    *b = tmp;
}
```

The `[T]` declares a **type parameter** named `T`. Think of it as a placeholder: "this function works for any type, call it T."

When you call the function, the compiler figures out what `T` should be:

```ferrum
let mut x: i32 = 1;
let mut y: i32 = 2;
swap(&mut x, &mut y);  // compiler infers: T = i32

let mut s1: String = "hello".to_string();
let mut s2: String = "world".to_string();
swap(&mut s1, &mut s2);  // compiler infers: T = String
```

You don't have to write `swap[i32](&mut x, &mut y)`. The compiler figures it out from the types of `x` and `y`. (You can be explicit if you want, but usually you don't need to.)

**Type safety is enforced at compile time:**

```ferrum
let mut x: i32 = 1;
let mut y: f64 = 2.0;
swap(&mut x, &mut y);
```

Compiler output:
```
error[E0308]: mismatched types
 --> src/main.fe:4:17
  |
4 |     swap(&mut x, &mut y);
  |          ------  ^^^^^^ expected `&mut i32`, found `&mut f64`
  |          |
  |          expected because this is `&mut i32`
  |
  = note: in call to `swap[T]`, T was inferred as `i32` from the first argument
```

The compiler catches the error. No runtime crash, no memory corruption. You fix it before the code ever runs.

---

## Practical Example: A Generic Stack

Let's build something more substantial: a stack (last-in, first-out data structure).

```ferrum
struct Stack[T] {
    items: Vec[T],
}

impl[T] Stack[T] {
    /// Create a new empty stack
    fn new(): Self {
        Stack { items: Vec.new() }
    }

    /// Push a value onto the stack
    fn push(&mut self, value: T) {
        self.items.push(value);
    }

    /// Pop the top value off the stack, if any
    fn pop(&mut self): Option[T] {
        self.items.pop()
    }

    /// Look at the top value without removing it
    fn peek(&self): Option[&T] {
        self.items.last()
    }

    /// Check if the stack is empty
    fn is_empty(&self): bool {
        self.items.len() == 0
    }
}
```

Now you can use it with any type:

```ferrum
// Stack of integers
let mut numbers: Stack[i32] = Stack.new();
numbers.push(1);
numbers.push(2);
numbers.push(3);
println("{}", numbers.pop());  // Some(3)
println("{}", numbers.pop());  // Some(2)

// Stack of strings
let mut commands: Stack[String] = Stack.new();
commands.push("open file".to_string());
commands.push("edit line 5".to_string());
commands.push("save".to_string());

// Undo the last command
let last_command = commands.pop();  // Some("save")

// Stack of your own types
struct Task {
    id: u64,
    name: String,
    priority: u8,
}

let mut task_stack: Stack[Task] = Stack.new();
task_stack.push(Task { id: 1, name: "Fix bug".to_string(), priority: 1 });
```

**Compare this to C:**

In C, you'd either:
1. Write `IntStack`, `StringStack`, `TaskStack` separately (copy-paste hell)
2. Use `void*` and lose all type safety
3. Use macros and lose your sanity debugging

In Ferrum, you write `Stack[T]` once, and it works for every type.

---

## Practical Example: A Generic Cache

Here's a cache that stores computed values to avoid recomputing them:

```ferrum
struct Cache[K, V] {
    map: HashMap[K, V],
    max_size: usize,
}

impl[K: Hash + Eq, V] Cache[K, V] {
    fn new(max_size: usize): Self {
        Cache {
            map: HashMap.new(),
            max_size,
        }
    }

    fn get(&self, key: &K): Option[&V] {
        self.map.get(key)
    }

    fn insert(&mut self, key: K, value: V) {
        if self.map.len() >= self.max_size {
            // Remove oldest entry (simplified - real LRU is more complex)
            if let Some(old_key) = self.map.keys().next().cloned() {
                self.map.remove(&old_key);
            }
        }
        self.map.insert(key, value);
    }

    fn get_or_compute(&mut self, key: K, compute: fn(&K) -> V): &V
    where K: Clone
    {
        if !self.map.contains_key(&key) {
            let value = compute(&key);
            self.insert(key.clone(), value);
        }
        self.map.get(&key).unwrap()
    }
}
```

Usage:

```ferrum
// Cache expensive database lookups
let mut user_cache: Cache[u64, User] = Cache.new(1000);
user_cache.insert(12345, load_user_from_db(12345));

// Cache file contents
let mut file_cache: Cache[String, Vec[u8]] = Cache.new(100);
let contents = file_cache.get_or_compute("config.json".to_string(), |path| {
    fs.read(path).unwrap()
});

// Cache computed results
let mut fib_cache: Cache[u64, u64] = Cache.new(100);
```

Notice the `K: Hash + Eq` in the impl. That's a **trait bound** - we'll explain those next.

---

## Practical Example: Functions That Work on Any Collection

You can write functions that work on any collection that supports iteration:

```ferrum
/// Find the sum of any collection of numbers
fn sum[T, I](items: I): T
where
    T: Add[Output = T] + Default,
    I: Iterator[Item = T],
{
    let mut total = T.default();
    for item in items {
        total = total + item;
    }
    total
}

// Works with arrays
let arr = [1, 2, 3, 4, 5];
println("{}", sum(arr.iter()));  // 15

// Works with vectors
let vec = Vec.from([1.0, 2.5, 3.5]);
println("{}", sum(vec.iter()));  // 7.0

// Works with ranges
println("{}", sum(1..=100));  // 5050
```

Or find the first element matching a condition:

```ferrum
/// Find the first element matching a predicate
fn find[T, I](items: I, predicate: fn(&T) -> bool): Option[T]
where
    I: Iterator[Item = T],
{
    for item in items {
        if predicate(&item) {
            return Some(item);
        }
    }
    None
}

let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let first_even = find(numbers.iter(), |n| n % 2 == 0);  // Some(2)

let users = get_all_users();
let admin = find(users.iter(), |u| u.role == Role.Admin);
```

---

## The Problem with Unconstrained Generics

Here's a function that finds the maximum of two values:

```ferrum
fn max[T](a: T, b: T): T {
    if a > b { a } else { b }
}
```

This doesn't compile. Why?

```
error[E0369]: binary operation `>` cannot be applied to type `T`
 --> src/main.fe:2:10
  |
2 |     if a > b { a } else { b }
  |        - ^ - T
  |        |
  |        T
  |
  = note: `T` might not implement `Ord`
  = help: consider adding a trait bound: `fn max[T: Ord](a: T, b: T): T`
```

The compiler is saying: "I don't know that `T` supports the `>` operator. What if someone calls `max(some_file, other_file)`? What does it mean to compare two files?"

Not every type can be compared. The compiler needs you to say which types are allowed.

---

## Trait Bounds: Saying What Types Are Allowed

A **trait** is a set of capabilities. For example:
- `Ord` means "can be compared with `<`, `>`, `<=`, `>=`"
- `Clone` means "can be duplicated"
- `Debug` means "can be printed for debugging"

A **trait bound** constrains a type parameter to types that have those capabilities:

```ferrum
fn max[T: Ord](a: T, b: T): T {
    if a > b { a } else { b }
}
```

The `T: Ord` says: "T must implement the `Ord` trait." Now the compiler knows that `>` works on `T`.

```ferrum
max(1, 2);              // OK: i32 implements Ord
max("apple", "banana"); // OK: &str implements Ord (alphabetical order)
max(3.14, 2.72);        // OK: f64 implements Ord
```

But:

```ferrum
struct Point { x: f64, y: f64 }

let p1 = Point { x: 1.0, y: 2.0 };
let p2 = Point { x: 3.0, y: 4.0 };
max(p1, p2);
```

Compiler output:
```
error[E0277]: the trait bound `Point: Ord` is not satisfied
 --> src/main.fe:12:5
  |
12|     max(p1, p2);
  |     ^^^ the trait `Ord` is not implemented for `Point`
  |
  = help: the following implementations are available:
            impl Ord for i32
            impl Ord for f64
            impl Ord for String
            ... and 42 others
  = note: required by a bound in `max`: `fn max[T: Ord](a: T, b: T) -> T`
```

The error is clear: `Point` doesn't implement `Ord`, so you can't use it with `max`. If you want to compare points, you need to decide *how* to compare them (by distance from origin? by x-coordinate?) and implement `Ord`.

---

## Multiple Trait Bounds

You can require multiple traits with `+`:

```ferrum
fn print_sorted[T: Ord + Debug](items: &mut [T]) ! IO {
    items.sort();
    for item in items {
        println("{:?}", item);
    }
}
```

This function requires `T` to be:
- `Ord` (so we can sort)
- `Debug` (so we can print)

If you call it with a type that's `Ord` but not `Debug`:

```ferrum
struct SecretKey { bytes: [u8; 32] }
impl Ord for SecretKey { /* ... */ }
// Note: we deliberately don't implement Debug to avoid accidental logging

let mut keys: Vec[SecretKey] = get_keys();
print_sorted(&mut keys);
```

Compiler output:
```
error[E0277]: `SecretKey` doesn't implement `Debug`
 --> src/main.fe:15:18
  |
15|     print_sorted(&mut keys);
  |                  ^^^^^^^^^ `SecretKey` cannot be formatted using `{:?}`
  |
  = note: required by a bound in `print_sorted`: `T: Ord + Debug`
  = help: the trait `Debug` is not implemented for `SecretKey`
  = help: consider implementing `Debug` for `SecretKey`, or use a different function
```

---

## Common Trait Bounds You'll Use

| Trait | What it means | Types that have it |
|-------|---------------|-------------------|
| `Copy` | Can be copied by just copying bytes (cheap) | `i32`, `f64`, `bool`, `char`, small structs |
| `Clone` | Can be duplicated (might allocate memory) | `String`, `Vec[T]`, plus all `Copy` types |
| `Eq` | Can be compared for equality with `==` | `i32`, `String`, `bool`, most types |
| `Ord` | Can be ordered with `<`, `>`, `<=`, `>=` | `i32`, `String` (alphabetical), `f64` |
| `Hash` | Can be turned into a hash code | `i32`, `String`, `(i32, i32)`, most types |
| `Debug` | Can be printed with `{:?}` for debugging | Almost everything |
| `Display` | Can be printed with `{}` for users | `i32`, `String`, `f64` |
| `Default` | Has a sensible default value | `i32` (0), `String` ("")), `Vec[T]` ([]) |

**When to use which:**

```ferrum
// Hash map keys need Hash + Eq
struct HashMap[K: Hash + Eq, V] { ... }

// Sorting needs Ord
fn sort[T: Ord](items: &mut [T]) { ... }

// Printing needs Debug or Display
fn log[T: Debug](value: &T) ! IO { println("{:?}", value); }

// Storing in a container often needs Clone
fn duplicate[T: Clone](item: &T): T { item.clone() }
```

---

## Where Clauses: Complex Bounds Made Readable

When you have many bounds, the function signature gets cluttered:

```ferrum
// Hard to read
fn merge_maps[K: Hash + Eq + Clone, V: Clone, M: Map[K, V] + Default](a: M, b: M): M {
```

Use a `where` clause to put bounds after the signature:

```ferrum
fn merge_maps[K, V, M](a: M, b: M): M
where
    K: Hash + Eq + Clone,
    V: Clone,
    M: Map[K, V] + Default,
{
    let mut result = M.default();
    for (k, v) in a.iter() {
        result.insert(k.clone(), v.clone());
    }
    for (k, v) in b.iter() {
        result.insert(k.clone(), v.clone());
    }
    result
}
```

The `where` clause also lets you express more complex relationships:

```ferrum
fn process[I](iter: I)
where
    I: Iterator,
    I.Item: Debug + Clone,  // bounds on the iterator's item type
{
    for item in iter {
        println("{:?}", item.clone());
    }
}
```

---

## How Generics Compile: Zero Runtime Cost

Here's the key insight: **generics have no runtime overhead**.

When you write:

```ferrum
fn max[T: Ord](a: T, b: T): T {
    if a > b { a } else { b }
}

let x = max(1, 2);        // T = i32
let y = max(1.0, 2.0);    // T = f64
let z = max("a", "b");    // T = &str
```

The compiler generates separate versions of the function for each type you use:

```ferrum
// The compiler generates these (you never see them):
fn max_i32(a: i32, b: i32): i32 {
    if a > b { a } else { b }
}

fn max_f64(a: f64, b: f64): f64 {
    if a > b { a } else { b }
}

fn max_str(a: &str, b: &str): &str {
    if a > b { a } else { b }
}
```

This is called **monomorphization**: the compiler turns your one generic function into multiple specialized functions, one for each concrete type. "Mono" (one) + "morph" (form) = each type gets its own form of the function.

**Why this matters:**

1. **No runtime dispatch.** When you call `max(1, 2)`, it compiles to a direct call to `max_i32`. No looking up which function to call at runtime.

2. **Full optimization.** The compiler can inline, vectorize, and optimize each specialization. `max_i32` might compile to a single CPU instruction.

3. **Same speed as hand-written code.** The generated `max_i32` is identical to what you'd write by hand. Generics are syntactic sugar, not a runtime abstraction.

**Compare to C's void*:**

```c
// C: memcpy at runtime, no optimization possible
void* max(void* a, void* b, size_t size, int (*cmp)(void*, void*)) {
    return cmp(a, b) > 0 ? a : b;
}
```

Every call does a function pointer call (`cmp`), which the compiler can't inline. The CPU can't predict where the call goes, so it stalls the pipeline.

**Compare to Python:**

```python
def max(a, b):
    return a if a > b else b
```

Every `>` comparison does: look up `a`'s type, find its `__gt__` method, call it, handle the result. That's dozens of operations for a simple comparison.

Ferrum's generics: one or two CPU instructions.

---

## The Tradeoff: Binary Size

There's one cost to monomorphization: code size.

If you use `Vec[i32]`, `Vec[String]`, `Vec[User]`, and `Vec[Connection]`, the compiler generates four separate implementations of every `Vec` method. For a large program using many generic types, this can add up.

In practice:
- For most programs, this isn't a problem
- The linker removes unused functions
- Modern CPUs have large instruction caches
- You can use trait objects (covered below) when code size matters more than speed

---

## Generic Types

Types themselves can have type parameters. Here's a simplified vector:

```ferrum
struct Vec[T] {
    data: *mut T,
    len: usize,
    cap: usize,
}

impl[T] Vec[T] {
    fn new(): Self {
        Vec { data: null_mut(), len: 0, cap: 0 }
    }

    fn push(&mut self, value: T) {
        if self.len == self.cap {
            self.grow();
        }
        unsafe {
            self.data.add(self.len).write(value);
        }
        self.len += 1;
    }

    fn get(&self, index: usize): Option[&T] {
        if index < self.len {
            Some(unsafe { &*self.data.add(index) })
        } else {
            None
        }
    }

    fn len(&self): usize {
        self.len
    }
}
```

The `impl[T] Vec[T]` says: "for any type T, here are methods on Vec[T]".

Usage:

```ferrum
let mut numbers: Vec[i32] = Vec.new();
numbers.push(1);
numbers.push(2);
numbers.push(3);
println("{}", numbers.len());  // 3

let mut names: Vec[String] = Vec.new();
names.push("Alice".to_string());
names.push("Bob".to_string());

// Type mismatch is caught at compile time:
numbers.push("oops".to_string());  // ERROR: expected i32, found String
```

**Compare to C:**

```c
typedef struct {
    void* data;
    size_t len;
    size_t cap;
    size_t elem_size;  // must track this manually!
} Vec;

void vec_push(Vec* v, void* elem);
void* vec_get(Vec* v, size_t i);

// Usage
Vec numbers = vec_new(sizeof(int));
int x = 42;
vec_push(&numbers, &x);

// No type safety:
double d = 3.14;
vec_push(&numbers, &d);  // Compiles! Corrupts your data silently.

// Manual casting on every access:
int* p = (int*)vec_get(&numbers, 0);  // Hope you got the cast right!
```

**Compare to Python:**

```python
numbers = []
numbers.append(1)
numbers.append(2)
numbers.append("oops")  # No error - Python lists hold anything

total = sum(numbers)  # TypeError at runtime when it hits "oops"
```

---

## Multiple Type Parameters

Types can have multiple type parameters:

```ferrum
struct Pair[A, B] {
    first: A,
    second: B,
}

let pair: Pair[String, i32] = Pair {
    first: "Alice".to_string(),
    second: 30,
};

struct HashMap[K, V] {
    // ...
}

let ages: HashMap[String, u32] = HashMap.new();
ages.insert("Alice".to_string(), 30);
ages.insert("Bob".to_string(), 25);

let scores: HashMap[u64, f64] = HashMap.new();
scores.insert(12345, 98.5);
```

Each parameter is independent. You can use any types for K and V.

---

## Generics vs Trait Objects: Compile-Time vs Runtime

Generics are resolved at compile time. But sometimes you need runtime flexibility: a list that holds different types that all implement the same trait.

**Trait objects** provide runtime polymorphism:

```ferrum
trait Draw {
    fn draw(&self) ! IO;
}

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }

impl Draw for Circle {
    fn draw(&self) ! IO {
        println("Drawing circle with radius {}", self.radius);
    }
}

impl Draw for Rectangle {
    fn draw(&self) ! IO {
        println("Drawing {}x{} rectangle", self.width, self.height);
    }
}
```

**With generics (all elements same type):**

```ferrum
fn draw_all[T: Draw](shapes: &[T]) ! IO {
    for shape in shapes {
        shape.draw();
    }
}

// Must use all the same type:
let circles: Vec[Circle] = vec[Circle { radius: 1.0 }, Circle { radius: 2.0 }];
draw_all(&circles);  // OK

// Can't mix types:
let mixed = vec[Circle { radius: 1.0 }, Rectangle { width: 2.0, height: 3.0 }];
// ERROR: expected Circle, found Rectangle
```

**With trait objects (different types allowed):**

```ferrum
fn draw_all_dynamic(shapes: &[&dyn Draw]) ! IO {
    for shape in shapes {
        shape.draw();
    }
}

// Can mix types:
let circle = Circle { radius: 1.0 };
let rect = Rectangle { width: 2.0, height: 3.0 };
let shapes: Vec[&dyn Draw] = vec[&circle, &rect];
draw_all_dynamic(&shapes);  // OK - draws both
```

The `&dyn Draw` is a trait object: a fat pointer containing (1) a pointer to the data and (2) a pointer to a vtable with the methods.

**When to use each:**

| Use generics when... | Use trait objects when... |
|---------------------|---------------------------|
| All items are the same type | Items have different concrete types |
| You want maximum performance | You need runtime flexibility |
| Types are known at compile time | Types are determined at runtime |
| You're okay with larger binaries | You want smaller binary size |

**Performance comparison:**

- Generics: direct function call, can be inlined, no overhead
- Trait objects: indirect call through vtable (pointer chase), can't be inlined, small overhead

For most code, the trait object overhead is negligible. But in tight loops processing millions of items, generics can be significantly faster.

---

## impl Trait: Hide the Concrete Type

Sometimes you want to return "some type that implements this trait" without naming the specific type:

```ferrum
fn evens(limit: usize): impl Iterator[Item = usize] {
    (0..limit).filter(|x| x % 2 == 0)
}
```

The actual return type is something like `Filter[Range[usize], fn(&usize) -> bool]` - a complicated type you don't want to write out. With `impl Iterator`, you just say "this returns something you can iterate over."

The caller knows they get an `Iterator`, but doesn't need to know the exact type. This is still monomorphized (zero runtime cost) - the compiler knows the concrete type, you just don't have to spell it out.

**In function arguments, `impl Trait` is shorthand:**

```ferrum
// These are equivalent:
fn process(iter: impl Iterator[Item = i32]) { ... }
fn process[I: Iterator[Item = i32]](iter: I) { ... }
```

Use `impl Trait` for simpler signatures when you don't need to refer to the type parameter elsewhere.

---

## Error Messages: What Violations Look Like

One of the biggest benefits of generics is clear error messages. Let's see some examples.

**Missing trait bound:**

```ferrum
fn find_max[T](items: &[T]): Option[&T] {
    if items.is_empty() {
        return None;
    }
    let mut max = &items[0];
    for item in &items[1..] {
        if item > max {  // ERROR
            max = item;
        }
    }
    Some(max)
}
```

```
error[E0369]: binary operation `>` cannot be applied to type `&T`
 --> src/main.fe:7:17
  |
7 |         if item > max {
  |            ---- ^ --- &T
  |            |
  |            &T
  |
  = note: `T` might not implement `Ord`
  = help: consider restricting type parameter `T`:
  |
1 | fn find_max[T: Ord](items: &[T]): Option[&T] {
  |              +++++
```

**Type mismatch in generic call:**

```ferrum
let mut stack: Stack[i32] = Stack.new();
stack.push(42);
stack.push("hello");  // ERROR
```

```
error[E0308]: mismatched types
 --> src/main.fe:4:12
  |
4 |     stack.push("hello");
  |           ---- ^^^^^^^ expected `i32`, found `&str`
  |           |
  |           arguments to this method are incorrect
  |
note: method defined here
 --> src/stack.fe:12:5
  |
12|     fn push(&mut self, value: T) {
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

**Type doesn't implement required trait:**

```ferrum
struct SecretData {
    content: Vec[u8],
}

let secrets: HashMap[SecretData, String] = HashMap.new();
```

```
error[E0277]: the trait bound `SecretData: Hash` is not satisfied
 --> src/main.fe:6:18
  |
6 |     let secrets: HashMap[SecretData, String] = HashMap.new();
  |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Hash` is not implemented for `SecretData`
  |
note: required by a bound in `HashMap`
 --> std/collections/hash_map.fe:5:18
  |
5 | type HashMap[K: Hash + Eq, V] {
  |                 ^^^^ required by this bound in `HashMap`
  = help: consider implementing `Hash` for `SecretData`:
  |
1 + impl Hash for SecretData {
2 +     fn hash(&self, state: &mut Hasher) {
3 +         self.content.hash(state);
4 +     }
5 + }
  |
```

Notice how the compiler:
1. Tells you exactly what went wrong
2. Points to the line with the problem
3. Shows you what trait is missing
4. Suggests how to fix it

This is much better than C's "segmentation fault" or Python's "AttributeError: 'NoneType' has no attribute 'foo'" at 3am.

---

## Summary

| Approach | Type safety | Performance | Error quality | IDE support |
|----------|-------------|-------------|---------------|-------------|
| C `void*` | None | Runtime overhead | No checking | None |
| C macros | None | Good | Terrible | None |
| Python dynamic | Runtime only | Slow | Runtime only | Limited |
| **Ferrum generics** | **Compile-time** | **Zero overhead** | **Clear, helpful** | **Full** |

**Generics give you:**

1. **Type safety:** Errors caught at compile time, not in production
2. **Code reuse:** Write once, works with any type
3. **Zero cost:** Compiles to specialized code, as fast as hand-written
4. **Clear errors:** The compiler tells you exactly what's wrong and how to fix it
5. **IDE support:** Autocomplete, refactoring, go-to-definition all work

**The syntax summary:**

```ferrum
// Generic function
fn foo[T](x: T): T { ... }

// With trait bound
fn foo[T: Clone](x: T): T { ... }

// Multiple bounds
fn foo[T: Clone + Debug](x: T): T { ... }

// Multiple type parameters
fn foo[K, V](key: K, value: V): Pair[K, V] { ... }

// Where clause for complex bounds
fn foo[K, V](key: K, value: V)
where
    K: Hash + Eq,
    V: Clone,
{ ... }

// Generic type
struct Container[T] { value: T }

// impl block for generic type
impl[T] Container[T] { ... }

// impl block with bounds
impl[T: Clone] Container[T] { ... }

// Trait object (runtime polymorphism)
fn foo(x: &dyn Trait) { ... }

// impl Trait (hides concrete type)
fn foo(): impl Iterator[Item = i32] { ... }
```

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for the complete generics specification.*
