# Introduction to Traits in Ferrum

**Audience:** Programmers who know C and Python, new to trait-based polymorphism

---

## The Problem Traits Solve

You want to write a function that sorts things. Not just integers — any kind of thing that can be compared.

### In C: Function Pointers and void*

```c
void sort(void* arr, size_t n, size_t elem_size,
          int (*cmp)(const void*, const void*)) {
    // ... qsort implementation
}

int compare_ints(const void* a, const void* b) {
    return *(int*)a - *(int*)b;
}

int main() {
    int nums[] = {3, 1, 4, 1, 5};
    sort(nums, 5, sizeof(int), compare_ints);
}
```

Problems:

1. **No type safety.** The compiler can't check that `compare_ints` actually compares `int`s. Pass the wrong comparator and you get silent corruption.
2. **Manual vtables.** If you want multiple operations (compare, hash, print), you build a struct of function pointers by hand.
3. **void* everywhere.** Types disappear. The compiler can't help you.
4. **Runtime overhead.** Every call goes through a function pointer — no inlining, no optimization.

### In Python: Duck Typing and Inheritance

```python
def sort(items):
    # Just call < and hope it works
    for i in range(len(items)):
        for j in range(i + 1, len(items)):
            if items[j] < items[i]:
                items[i], items[j] = items[j], items[i]

sort([3, 1, 4])       # works
sort(["b", "a", "c"]) # works
sort([1, "a", None])  # TypeError at runtime, maybe
```

Or with inheritance:

```python
from abc import ABC, abstractmethod

class Comparable(ABC):
    @abstractmethod
    def __lt__(self, other): pass

class Person(Comparable):
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __lt__(self, other):
        return self.age < other.age
```

Problems:

1. **Runtime errors.** Duck typing fails when you actually call the method, not when you pass the wrong type.
2. **No guarantees.** Nothing ensures `items[j] < items[i]` will work until you try it.
3. **Fragile base class.** Change `Comparable` and every subclass might break.
4. **Diamond problem.** Multiple inheritance creates ambiguity about which method to call.
5. **Inheritance couples types.** `Person` must know about `Comparable` at definition time.

---

## What Traits Are

A **trait** defines a set of methods that a type can implement:

```ferrum
trait Ord {
    fn cmp(self: &Self, other: &Self): Ordering
}
```

This says: "Any type that implements `Ord` must have a `cmp` method that compares two values."

A trait is not a type. It's a contract. It says what a type can do, not what it is.

---

## Implementing Traits

You implement a trait for a type using an `impl` block:

```ferrum
type Point {
    x: i32,
    y: i32,
}

impl Ord for Point {
    fn cmp(self: &Self, other: &Self): Ordering {
        // Compare by x first, then by y
        match self.x.cmp(&other.x) {
            Ordering.Equal => self.y.cmp(&other.y),
            ord => ord,
        }
    }
}
```

Now `Point` satisfies the `Ord` contract. You can compare points.

The key insight: **the trait and the type are defined separately.** You can implement traits for types you didn't write, and you can implement your traits for types others wrote.

```ferrum
// Implement Display for a type from another library
impl Display for ThirdPartyWidget {
    fn fmt(self: &Self, f: &mut Formatter): Result[()] {
        write(f, "Widget({})", self.id)
    }
}
```

In Python, you'd have to subclass or monkey-patch. In C, you'd have to write a wrapper. In Ferrum, you just write an impl block.

---

## Trait Bounds: Requiring Capabilities

Now you can write that sort function:

```ferrum
fn sort[T: Ord](items: &mut [T]) {
    for i in 0..items.len() {
        for j in (i + 1)..items.len() {
            if items[j].cmp(&items[i]) == Ordering.Less {
                items.swap(i, j)
            }
        }
    }
}
```

The `[T: Ord]` is a **trait bound**. It says: "T can be any type, as long as it implements `Ord`."

If you try to sort something that doesn't implement `Ord`:

```ferrum
type Blob { data: Vec[u8] }

let mut blobs = [Blob { data: vec![1, 2] }, Blob { data: vec![3] }];
sort(&mut blobs);  // ERROR: Blob does not implement Ord
```

The compiler tells you exactly what's missing:

```
error: trait bound not satisfied
  --> example.fe:10:5
   |
10 |     sort(&mut blobs)
   |     ^^^^ the trait `Ord` is not implemented for `Blob`
   |
   = help: implement `Ord` for `Blob`, or use a different sorting function
```

Compare this to:
- **C:** Silent corruption at runtime
- **Python:** `TypeError: '<' not supported between instances of 'Blob' and 'Blob'` when the sort runs

Ferrum catches it before the program runs.

---

## Multiple Trait Bounds

You can require multiple capabilities:

```ferrum
fn print_sorted[T: Ord + Display](items: &mut [T]) ! IO {
    sort(items);
    for item in items {
        println!("{}", item);
    }
}
```

`T: Ord + Display` means T must be both orderable and printable.

For complex bounds, use `where`:

```ferrum
fn merge[K, V](a: HashMap[K, V], b: HashMap[K, V]): HashMap[K, V]
where
    K: Eq + Hash,
    V: Clone,
{
    // ...
}
```

---

## Static vs Dynamic Dispatch

Ferrum gives you two ways to use traits, with different tradeoffs.

### Static Dispatch (Generics)

```ferrum
fn print_it[T: Display](x: &T) ! IO {
    println!("{}", x);
}
```

The compiler generates a separate version of `print_it` for each type you call it with. `print_it::<i32>`, `print_it::<String>`, `print_it::<Point>`. This is **monomorphization**.

- **Pro:** No runtime overhead. The compiler can inline and optimize.
- **Pro:** Each version is specialized for its type.
- **Con:** Code bloat if you use many types.
- **Con:** The type must be known at compile time.

### Dynamic Dispatch (dyn Trait)

```ferrum
fn print_it(x: &dyn Display) ! IO {
    println!("{}", x);
}
```

One function, one copy of the code. At runtime, a vtable lookup finds the right `fmt` method.

- **Pro:** One copy of the code, no bloat.
- **Pro:** Can store different types in the same collection.
- **Con:** Vtable lookup on every call (small overhead).
- **Con:** Compiler can't inline or specialize.

### When to Use Which

**Use generics (static dispatch) when:**
- Performance matters
- You know the types at compile time
- You're writing library code that should be maximally flexible

**Use `dyn Trait` (dynamic dispatch) when:**
- You need heterogeneous collections (different types together)
- You're building plugin systems
- Code size matters more than call speed

Example — heterogeneous collection:

```ferrum
// Can't do this with generics — all elements must be the same type
let mut loggers: Vec[Box[dyn Logger]] = Vec.new();
loggers.push(Box.new(FileLogger.new("app.log")));
loggers.push(Box.new(ConsoleLogger.new()));
loggers.push(Box.new(NetworkLogger.new("logs.example.com")));

for logger in &loggers {
    logger.log("Application started");
}
```

---

## Comparing to C

### C: Manual vtables

```c
typedef struct {
    void (*write)(void* self, const char* msg);
    void (*flush)(void* self);
} LoggerVtable;

typedef struct {
    LoggerVtable* vtable;
    void* data;
} Logger;

void log_message(Logger* logger, const char* msg) {
    logger->vtable->write(logger->data, msg);
}
```

You build and maintain the vtable by hand. Nothing checks that `write` and `flush` have the right signatures. Nothing stops you from passing garbage.

### Ferrum: Compiler-managed vtables

```ferrum
trait Logger {
    fn write(self: &mut Self, msg: &str) ! IO
    fn flush(self: &mut Self) ! IO
}

fn log_message(logger: &mut dyn Logger, msg: &str) ! IO {
    logger.write(msg);
}
```

The compiler generates the vtable, checks all the types, and guarantees safety. You get the same runtime representation, but you can't mess it up.

---

## Comparing to Python

### Python: ABC (Abstract Base Classes)

```python
from abc import ABC, abstractmethod

class Logger(ABC):
    @abstractmethod
    def write(self, msg: str) -> None: pass

    @abstractmethod
    def flush(self) -> None: pass

class FileLogger(Logger):
    def __init__(self, path):
        self.file = open(path, 'w')

    def write(self, msg):
        self.file.write(msg)

    def flush(self):
        self.file.flush()
```

Problems:

1. **Runtime checking.** Forget to implement `flush` and you get an error when you instantiate, not when you define.
2. **Inheritance required.** `FileLogger` must inherit from `Logger`. You can't make someone else's class a Logger.
3. **No static dispatch.** Every method call is dynamic.

### Ferrum: Traits are separate from types

```ferrum
trait Logger {
    fn write(self: &mut Self, msg: &str) ! IO
    fn flush(self: &mut Self) ! IO
}

impl Logger for File {
    fn write(self: &mut Self, msg: &str) ! IO {
        self.write_str(msg)
    }

    fn flush(self: &mut Self) ! IO {
        self.sync()
    }
}
```

You can implement `Logger` for `File` even though you didn't write either one. The `impl` is checked at compile time. You choose static or dynamic dispatch at the call site.

---

## Why Traits Beat Inheritance

### 1. Composition Over Inheritance

Inheritance says: "A Dog IS-A Animal."

Traits say: "A Dog CAN-DO these things."

```ferrum
// Traits compose behaviors
type Dog {
    name: String,
    breed: Breed,
}

impl Walk for Dog { ... }
impl Bark for Dog { ... }
impl Pet for Dog { ... }

// A robot dog can walk but not bark
type RobotDog {
    model: String,
}

impl Walk for RobotDog { ... }
impl Pet for RobotDog { ... }
// No impl Bark — it's a robot
```

With inheritance, `RobotDog` would inherit `bark()` from `Dog` and you'd have to override it to throw an error. With traits, you just don't implement `Bark`.

### 2. No Fragile Base Class

In Python:

```python
class Animal:
    def speak(self):
        print(self.sound())  # calls subclass method

    def sound(self):
        return "..."

class Dog(Animal):
    def sound(self):
        return "woof"
```

If `Animal` changes how `speak()` works, `Dog` might break. The base class and derived class are tightly coupled.

In Ferrum, traits define a contract. The implementation is local to the type. Changing one impl doesn't affect others.

### 3. No Diamond Problem

Python:

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B"

class C(A):
    def method(self): return "C"

class D(B, C):
    pass

D().method()  # "B"? "C"? Depends on MRO rules
```

Ferrum doesn't have multiple inheritance. A type implements traits, and each trait has exactly one implementation for that type. No ambiguity.

### 4. Retroactive Implementation

You can implement traits for existing types:

```ferrum
// Make i32 loggable, even though you didn't write i32
impl Logger for i32 {
    fn write(self: &mut Self, msg: &str) ! IO {
        println!("[{}] {}", self, msg);  // use self as log level
    }
    fn flush(self: &mut Self) ! IO { }
}
```

With inheritance, you'd have to wrap `i32` in a new class. With traits, you just add the capability.

---

## Default Method Implementations

Traits can provide default implementations:

```ferrum
trait Iterator {
    type Item

    fn next(self: &mut Self): Option[Self.Item]

    // Provided methods — implementations get these for free
    fn count(self: Self): usize {
        let mut n = 0;
        while self.next().is_some() {
            n += 1;
        }
        n
    }

    fn map[U](self: Self, f: fn(Self.Item): U): Map[Self, U] {
        Map { iter: self, f }
    }

    fn filter(self: Self, pred: fn(&Self.Item): bool): Filter[Self] {
        Filter { iter: self, pred }
    }
}
```

Implement `next`, get `count`, `map`, `filter`, and dozens more for free. Override any of them if you have a more efficient version.

---

## Associated Types

Traits can have associated types — types that are determined by the implementation:

```ferrum
trait Iterator {
    type Item  // each Iterator decides what Item is

    fn next(self: &mut Self): Option[Self.Item]
}

impl Iterator for Range[i32] {
    type Item = i32  // Range[i32] yields i32s

    fn next(self: &mut Self): Option[i32] {
        if self.current < self.end {
            let val = self.current;
            self.current += 1;
            Some(val)
        } else {
            None
        }
    }
}
```

Associated types avoid parameter explosion. Instead of `Iterator[T]` everywhere, you write `Iterator` and let the impl decide what `Item` is.

---

## Summary

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Polymorphism | void*, function pointers | Duck typing, inheritance | Traits |
| Type checking | None | Runtime | Compile time |
| Define behavior | Struct of function pointers | ABC or convention | `trait` definition |
| Implement behavior | Fill in function pointers | Inheritance or duck typing | `impl` blocks |
| Static dispatch | Manual specialization | Not available | Generics `[T: Trait]` |
| Dynamic dispatch | Manual vtables | Always (implicit) | `dyn Trait` (explicit) |
| Add behavior to existing type | Wrapper struct | Monkey-patching | `impl Trait for Type` |
| Multiple behaviors | Multiple structs | Multiple inheritance (diamond!) | Multiple impl blocks |

Traits are contracts that types fulfill. They give you the flexibility of duck typing with the safety of static types. They give you the power of inheritance without the coupling. And they let you choose between compile-time and runtime dispatch based on your needs.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete trait system specification.*
