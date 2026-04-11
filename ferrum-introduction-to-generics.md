# Introduction to Generics in Ferrum

**Audience:** Programmers who know C and Python, new to parametric polymorphism

---

## The Problem Generics Solve

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

Three functions that do the same thing, differing only in type. If you have 10 types, you write 10 functions. This is tedious, error-prone, and bloats your codebase.

C offers two escape hatches. Both have serious problems.

---

## C's Approach 1: void*

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

// But also:
swap(&x, &d1, sizeof(int));  // compiles! corrupts memory at runtime
```

Problems with `void*`:

1. **No type safety.** The compiler can't check that `a` and `b` have the same type. Pass an `int*` and a `double*` and it compiles silently.

2. **Manual size tracking.** You must pass `sizeof` everywhere. Get it wrong and you corrupt memory or crash.

3. **No IDE help.** Autocomplete doesn't know what type you're working with. Refactoring tools can't help.

4. **Runtime overhead.** The `memcpy` calls happen at runtime. The compiler can't optimize because it doesn't know the type.

---

## C's Approach 2: Macros

```c
#define SWAP(type, a, b) do { \
    type tmp = (a);           \
    (a) = (b);                \
    (b) = tmp;                \
} while(0)

// Usage
int x = 1, y = 2;
SWAP(int, x, y);  // works

// But:
SWAP(int, x, 1.5);  // silently truncates the double
SWAP(int, x, "hello");  // cryptic error message
```

Problems with macros:

1. **No type checking.** Macros are text substitution. The preprocessor doesn't understand types.

2. **Terrible error messages.** When something goes wrong, the error points to the expanded code, not your macro call. Good luck debugging.

3. **No debugger support.** You can't step through a macro. Breakpoints don't work inside them.

4. **Name collisions.** Macros pollute the namespace. `SWAP` might conflict with another library's `SWAP`.

5. **No scoping.** Macros ignore function boundaries. They're globally visible.

C11 added `_Generic`, which helps a bit:

```c
#define swap(a, b) _Generic((a), \
    int*: swap_int, \
    double*: swap_double \
)(a, b)
```

But you still have to write each type-specific function. `_Generic` just picks which one to call.

---

## Python's Approach: Dynamic Typing

```python
def swap(a, b):
    return b, a

x, y = 1, 2
x, y = swap(x, y)  # works

x, y = "hello", "world"
x, y = swap(x, y)  # works
```

This works because Python doesn't check types until runtime. Every variable can hold any type.

Problems:

1. **Runtime errors.** Type mismatches become crashes in production, not compile errors during development.

    ```python
    def process(items):
        for item in items:
            print(item.upper())

    process(["hello", "world"])  # works
    process([1, 2, 3])  # crashes at runtime: AttributeError
    ```

2. **No IDE help.** Without type information, autocomplete doesn't know what methods are available. You have to remember or check the docs.

3. **Performance overhead.** Every operation requires runtime type dispatch. Python can't optimize because it doesn't know types until the code runs.

4. **Refactoring is scary.** Rename a method and you won't know if you broke something until you run every code path.

Python's type hints (`def swap(a: T, b: T) -> tuple[T, T]`) help with IDE support but don't enforce anything at runtime. They're comments that tools can read.

---

## Generics: Type-Safe Abstraction

Generics let you write one function that works with any type, with full type safety and no runtime cost.

In Ferrum:

```ferrum
fn swap[T](a: &mut T, b: &mut T) {
    let tmp = *a;
    *a = *b;
    *b = tmp;
}
```

The `[T]` declares a **type parameter**. `T` is a placeholder that gets filled in when you call the function:

```ferrum
let mut x: i32 = 1;
let mut y: i32 = 2;
swap(&mut x, &mut y);  // T = i32

let mut s1: String = "hello".to_string();
let mut s2: String = "world".to_string();
swap(&mut s1, &mut s2);  // T = String
```

The compiler figures out what `T` is from context. You don't have to write `swap[i32](&mut x, &mut y)` (though you can if you want to be explicit).

**Type safety is enforced:**

```ferrum
let mut x: i32 = 1;
let mut y: f64 = 2.0;
swap(&mut x, &mut y);  // ERROR: expected &mut i32, found &mut f64
```

The compiler catches the error. No runtime crash, no memory corruption.

---

## Generic Types

Types can have type parameters too. Here's a growable array:

```ferrum
type Vec[T] {
    data: *mut T,
    len: usize,
    cap: usize,
}

impl[T] Vec[T] {
    fn new(): Self {
        Vec { data: null_mut(), len: 0, cap: 0 }
    }

    fn push(&mut self, value: T) {
        // ... grow if needed, then store value
    }

    fn get(&self, index: usize): Option[&T] {
        if index < self.len {
            Some(&self.data[index])
        } else {
            None
        }
    }
}
```

Usage:

```ferrum
let mut numbers: Vec[i32] = Vec.new();
numbers.push(1);
numbers.push(2);
numbers.push(3);

let mut names: Vec[String] = Vec.new();
names.push("Alice".to_string());
names.push("Bob".to_string());
```

**Compare to C:**

```c
// C: void* based "generic" vector
typedef struct {
    void* data;
    size_t len;
    size_t cap;
    size_t elem_size;  // must track element size manually
} Vec;

void vec_push(Vec* v, void* elem);  // no type safety
void* vec_get(Vec* v, size_t i);    // returns void*, must cast

// Usage
Vec numbers;
int x = 42;
vec_push(&numbers, &x);
int* p = (int*)vec_get(&numbers, 0);  // manual cast, no checking
```

**Compare to Python:**

```python
numbers = []
numbers.append(1)
numbers.append(2)
numbers.append("oops")  # no error - mixed types allowed

# Later:
total = sum(numbers)  # crashes at runtime
```

---

## Multiple Type Parameters

Types can have multiple parameters. A hash map needs both a key type and a value type:

```ferrum
type HashMap[K, V] {
    // ...
}

let ages: HashMap[String, u32] = HashMap.new();
ages.insert("Alice".to_string(), 30);
ages.insert("Bob".to_string(), 25);

let scores: HashMap[u64, f64] = HashMap.new();
scores.insert(12345, 98.5);
```

The parameters `K` and `V` are independent. You can use any types for either.

---

## The Problem with Unconstrained Generics

Here's a function that finds the maximum of two values:

```ferrum
fn max[T](a: T, b: T): T {
    if a > b { a } else { b }  // ERROR: cannot compare T with >
}
```

This doesn't compile. Why? Because not every type can be compared with `>`. What does `"hello" > "world"` mean? What about `Vec[i32] > Vec[i32]`? The compiler doesn't know that `T` supports comparison.

---

## Trait Bounds

A **trait bound** constrains a type parameter to types that have certain capabilities. The `Ord` trait means "can be ordered" (supports `<`, `>`, `<=`, `>=`):

```ferrum
fn max[T: Ord](a: T, b: T): T {
    if a > b { a } else { b }
}
```

Now the compiler knows `T` supports `>`. This compiles.

```ferrum
max(1, 2)                          // works: i32 implements Ord
max("apple", "banana")             // works: &str implements Ord
max(Vec.from([1,2]), Vec.from([3,4]))  // ERROR: Vec[i32] does not implement Ord
```

You can require multiple traits with `+`:

```ferrum
fn print_max[T: Ord + Debug](a: T, b: T) ! IO {
    let m = max(a, b);
    println("{:?}", m);
}
```

This function requires `T` to be both orderable (`Ord`) and printable (`Debug`).

---

## Common Trait Bounds

| Trait | Meaning | Examples |
|-------|---------|----------|
| `Copy` | Can be copied bit-for-bit | `i32`, `f64`, `bool`, `(u8, u8)` |
| `Clone` | Can be duplicated (possibly expensive) | `String`, `Vec[T]`, all `Copy` types |
| `Eq` | Can be compared for equality | `i32`, `String`, `bool` |
| `Ord` | Can be ordered | `i32`, `String` (lexicographic) |
| `Hash` | Can be hashed | `i32`, `String`, `(i32, i32)` |
| `Debug` | Can be printed for debugging | Almost everything |
| `Display` | Can be printed for users | `i32`, `String`, `f64` |
| `Default` | Has a default value | `i32` (0), `String` (empty), `Vec[T]` (empty) |

---

## Trait Bounds in Practice

Sorting requires ordering:

```ferrum
fn sort[T: Ord](slice: &mut [T]) {
    // ... implementation uses < and > on elements
}

let mut nums = [3, 1, 4, 1, 5];
sort(&mut nums);  // [1, 1, 3, 4, 5]
```

A hash map requires keys to be hashable and comparable:

```ferrum
type HashMap[K: Hash + Eq, V] {
    // ...
}
```

Serialization requires the `Serialize` trait:

```ferrum
fn to_json[T: Serialize](value: &T): String {
    // ...
}
```

---

## Where Clauses

For complex bounds, use a `where` clause:

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

This is clearer than cramming everything into the function signature.

---

## How Generics Compile: Monomorphization

Here's the magic: generics have **zero runtime cost**.

When you write:

```ferrum
fn max[T: Ord](a: T, b: T): T {
    if a > b { a } else { b }
}

max(1, 2);        // T = i32
max(1.0, 2.0);    // T = f64
```

The compiler generates specialized versions:

```ferrum
// Generated by compiler (you never see this)
fn max_i32(a: i32, b: i32): i32 {
    if a > b { a } else { b }
}

fn max_f64(a: f64, b: f64): f64 {
    if a > b { a } else { b }
}
```

This is called **monomorphization**: each concrete type gets its own specialized code. The generic syntax is a convenience for you; the compiled code is fully specialized.

**Benefits:**

1. **No runtime dispatch.** Calls go directly to the specialized function.
2. **Full optimization.** The compiler can inline, vectorize, and optimize each specialization.
3. **No type information at runtime.** No vtables, no type tags, no overhead.

**Compare to C's void*:**

- C's `void*` approach has runtime overhead (memcpy, size calculations).
- Ferrum's generics compile to code as fast as hand-written type-specific functions.

**Compare to Python:**

- Python dispatches every operation through runtime type lookups.
- Ferrum's generics compile to direct calls with no dispatch overhead.

---

## Generics vs Trait Objects

Generics are resolved at compile time. But sometimes you need runtime flexibility: a collection that holds different types implementing the same trait.

**Trait objects** provide runtime polymorphism:

```ferrum
trait Draw {
    fn draw(&self) ! IO;
}

impl Draw for Circle {
    fn draw(&self) ! IO { /* ... */ }
}

impl Draw for Rectangle {
    fn draw(&self) ! IO { /* ... */ }
}

// Generic: all elements must be the same type
fn draw_all_generic[T: Draw](shapes: &[T]) ! IO {
    for shape in shapes {
        shape.draw();
    }
}

// Trait object: elements can be different types
fn draw_all_dynamic(shapes: &[&dyn Draw]) ! IO {
    for shape in shapes {
        shape.draw();
    }
}
```

Usage:

```ferrum
let circles: Vec[Circle] = vec![Circle.new(), Circle.new()];
draw_all_generic(&circles);  // all Circles

let shapes: Vec[&dyn Draw] = vec![&circle, &rectangle, &triangle];
draw_all_dynamic(&shapes);  // mixed types
```

**When to use each:**

| Use generics when... | Use trait objects when... |
|---------------------|---------------------------|
| All elements are the same type | Elements have different types |
| You want maximum performance | You need runtime flexibility |
| The type is known at compile time | The type is determined at runtime |
| Binary size is not a concern | You want smaller binaries |

Trait objects have a small runtime cost: an extra pointer indirection through a vtable. Generics have no runtime cost but may increase binary size (each specialization is separate code).

---

## impl Trait

When you don't want to name the concrete type, use `impl Trait`:

```ferrum
// Returns "some type that implements Iterator"
fn evens(n: usize): impl Iterator[Item = usize] {
    (0..n).filter(|x| x % 2 == 0)
}
```

The caller knows it gets an `Iterator` but doesn't need to know the exact type (which is some complicated filter-chain type). This is still monomorphized; no runtime cost.

In function parameters, `impl Trait` is shorthand for a generic:

```ferrum
// These are equivalent:
fn process(iter: impl Iterator[Item = i32]) { ... }
fn process[I: Iterator[Item = i32]](iter: I) { ... }
```

---

## Summary

| Approach | Type safety | Performance | Error messages | IDE support |
|----------|-------------|-------------|----------------|-------------|
| C `void*` | None | Runtime overhead | N/A (no checking) | None |
| C macros | Limited | Good | Terrible | Limited |
| C `_Generic` | Good | Good | Decent | Limited |
| Python dynamic | None (runtime) | Slow | Runtime only | Limited |
| Python type hints | Documentation only | Slow | Tooling only | Good |
| Ferrum generics | Full | Zero overhead | Clear | Full |

Generics give you:
- **Type safety:** Errors caught at compile time, not runtime
- **Code reuse:** One function works with many types
- **Zero cost:** Compiles to specialized code, no runtime dispatch
- **Clear errors:** The compiler tells you exactly what's wrong
- **IDE support:** Autocomplete, refactoring, and navigation all work

The syntax is simple: `[T]` declares a type parameter, `T: Trait` constrains what types are allowed. The compiler handles the rest.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete generics specification.*
