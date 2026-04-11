# Ferrum Iterators and Closures

## The Problem

You've written this loop a thousand times in C:

```c
// Sum of squares of even numbers
int sum = 0;
for (int i = 0; i < n; i++) {
    if (nums[i] % 2 == 0) {
        sum += nums[i] * nums[i];
    }
}
```

It works. It's also easy to mess up: off-by-one errors, forgetting to increment, accidentally modifying `i` inside the loop. The *intent* (filter evens, square them, sum) is buried in boilerplate.

Python cleans this up:

```python
sum(x * x for x in nums if x % 2 == 0)
```

Beautiful. But what if you want to do something more complex? Maybe you need intermediate steps, or you want to chain operations. Python's list comprehensions get unwieldy, and if you use `[...]` instead of `(...)`, you're allocating intermediate lists:

```python
# This allocates two intermediate lists
evens = [x for x in nums if x % 2 == 0]
squared = [x * x for x in evens]
result = sum(squared)
```

Ferrum gives you the best of both worlds: readable, chainable operations that compile down to the same machine code as your hand-written C loop.

```ferrum
let sum = nums.iter()
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();
```

No intermediate allocations. No off-by-one errors. The compiler generates the same tight loop C would.

---

## Iterator Basics

### Getting an Iterator

Every collection in Ferrum has an `.iter()` method that returns an iterator:

```ferrum
let numbers = [1, 2, 3, 4, 5];
let iter = numbers.iter();  // Iterator over &i32 references
```

Three variations exist:
- `.iter()` — iterates over references (`&T`)
- `.iter_mut()` — iterates over mutable references (`&mut T`)
- `.into_iter()` — consumes the collection, iterates over owned values (`T`)

```ferrum
let mut scores = [10, 20, 30];

// Read-only access
for score in scores.iter() {
    println("{}", score);
}

// Modify in place
for score in scores.iter_mut() {
    *score += 5;
}

// Consume the array (can't use scores after this)
let doubled: Vec<i32> = scores.into_iter().map(|x| x * 2).collect();
```

### The `for` Loop

The `for` loop is syntactic sugar for iterators:

```ferrum
// These are equivalent
for x in collection {
    println("{}", x);
}

// Desugars to roughly:
let mut iter = collection.into_iter();
while let Some(x) = iter.next() {
    println("{}", x);
}
```

### Lazy Evaluation

Iterators are lazy. Nothing happens until you consume them:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// This does NOTHING yet
let iter = numbers.iter()
    .filter(|x| **x > 2)
    .map(|x| x * 10);

// Only now does computation happen
for n in iter {
    println("{}", n);  // Prints 30, 40, 50
}
```

Compare to Python, where a list comprehension executes immediately:

```python
# Python: this runs RIGHT NOW and allocates a list
result = [x * 10 for x in numbers if x > 2]  # [30, 40, 50] created immediately
```

Ferrum's laziness means you can work with huge datasets or infinite sequences without blowing up memory:

```ferrum
// First 10 even numbers starting from 0
let evens: Vec<i32> = (0..)          // Infinite range!
    .filter(|x| x % 2 == 0)
    .take(10)
    .collect();
// [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

---

## Closures

Before we dive into iterator methods, you need to understand closures. A closure is an anonymous function that can capture variables from its environment.

### Basic Syntax

```ferrum
// Function
fn double(x: i32) -> i32 {
    x * 2
}

// Closure (anonymous function)
let double = |x| x * 2;

// With explicit types
let double = |x: i32| -> i32 { x * 2 };
```

The `|x|` part is the parameter list. The body can be a single expression or a block:

```ferrum
// Single expression (no braces needed)
let add_one = |x| x + 1;

// Block with multiple statements
let complex = |x| {
    let y = x * 2;
    let z = y + 1;
    z * z
};
```

### Capturing Variables

Unlike C function pointers, closures capture their environment:

```ferrum
let multiplier = 3;
let scale = |x| x * multiplier;  // Captures `multiplier`

println("{}", scale(10));  // 30
```

In C, you'd need to pass context explicitly:

```c
// C: Verbose and error-prone
typedef struct {
    int multiplier;
} ScaleContext;

int scale(int x, void* ctx) {
    ScaleContext* c = (ScaleContext*)ctx;
    return x * c->multiplier;
}

// Usage
ScaleContext ctx = { .multiplier = 3 };
int result = scale(10, &ctx);  // 30
```

Every C callback that needs state requires this dance: define a struct, cast void pointers, hope you got the types right. Ferrum closures handle this automatically and type-safely.

### Capture Modes

Ferrum closures capture variables in three ways, chosen automatically based on usage:

1. **By reference** (`&T`) — when you only read the variable
2. **By mutable reference** (`&mut T`) — when you modify the variable
3. **By move** (`T`) — when you take ownership

```ferrum
let text = String::from("hello");
let numbers = vec![1, 2, 3];

// Captures `text` by reference (just reading)
let print_text = || println("{}", text);

// Captures `numbers` by mutable reference (modifying)
let mut push_num = || numbers.push(4);

// Force capture by move (takes ownership)
let consume = move || {
    drop(text);  // We own it now
};
// text is no longer valid here
```

The `move` keyword forces all captures to take ownership. This is essential when the closure outlives the current scope:

```ferrum
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    // Without `move`, this would try to capture `n` by reference
    // But `n` goes out of scope when make_adder returns!
    move |x| x + n
}

let add_five = make_adder(5);
println("{}", add_five(10));  // 15
```

---

## Iterator Methods

Now let's look at the methods that make iterators powerful.

### Transforming: `map()`

Apply a function to each element:

```ferrum
let numbers = [1, 2, 3, 4];
let doubled: Vec<i32> = numbers.iter()
    .map(|x| x * 2)
    .collect();
// [2, 4, 6, 8]
```

**C equivalent:**

```c
int doubled[4];
for (int i = 0; i < 4; i++) {
    doubled[i] = numbers[i] * 2;
}
```

**Python equivalent:**

```python
doubled = [x * 2 for x in numbers]  # Allocates immediately
# or
doubled = list(map(lambda x: x * 2, numbers))
```

### Filtering: `filter()`

Keep only elements matching a predicate:

```ferrum
let numbers = [1, 2, 3, 4, 5, 6];
let evens: Vec<i32> = numbers.iter()
    .filter(|x| *x % 2 == 0)
    .copied()  // Convert &i32 to i32
    .collect();
// [2, 4, 6]
```

Note: `filter()` yields references, so we use `.copied()` to get owned values. This is because the original collection still owns the data.

### Aggregating: `fold()`

Reduce a collection to a single value:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// Sum
let sum = numbers.iter().fold(0, |acc, x| acc + x);
// 15

// Product
let product = numbers.iter().fold(1, |acc, x| acc * x);
// 120

// Build a string
let csv = numbers.iter()
    .map(|x| x.to_string())
    .fold(String::new(), |acc, s| {
        if acc.is_empty() { s } else { acc + "," + &s }
    });
// "1,2,3,4,5"
```

**C equivalent:**

```c
int sum = 0;
for (int i = 0; i < 5; i++) {
    sum += numbers[i];
}
```

Common aggregations have shortcuts:

```ferrum
let numbers = [1, 2, 3, 4, 5];
let sum: i32 = numbers.iter().sum();       // 15
let product: i32 = numbers.iter().product(); // 120
let count = numbers.iter().count();        // 5
let max = numbers.iter().max();            // Some(&5)
let min = numbers.iter().min();            // Some(&1)
```

### Collecting: `collect()`

Turn an iterator back into a collection:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// Into a Vec
let vec: Vec<i32> = numbers.iter().copied().collect();

// Into a HashSet
let set: HashSet<i32> = numbers.iter().copied().collect();

// Into a String (for char iterators)
let s: String = ['h', 'e', 'l', 'l', 'o'].iter().collect();
```

The type annotation tells `collect()` what to produce. This is called "type-directed" collection.

### Finding: `find()`, `position()`

Locate elements:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// Find first even number
let first_even = numbers.iter().find(|x| *x % 2 == 0);
// Some(&2)

// Find position of first even
let pos = numbers.iter().position(|x| x % 2 == 0);
// Some(1)

// Find with transformation
let first_big_square = numbers.iter()
    .map(|x| x * x)
    .find(|x| *x > 10);
// Some(16)
```

### Testing: `any()`, `all()`

Check conditions across a collection:

```ferrum
let numbers = [1, 2, 3, 4, 5];

let has_even = numbers.iter().any(|x| x % 2 == 0);      // true
let all_positive = numbers.iter().all(|x| *x > 0);      // true
let all_even = numbers.iter().all(|x| x % 2 == 0);      // false
```

These short-circuit: `any()` stops at the first `true`, `all()` stops at the first `false`.

---

## Chaining

The real power comes from chaining operations:

```ferrum
struct User {
    name: String,
    age: u32,
    active: bool,
}

let users = vec![
    User { name: "Alice".into(), age: 30, active: true },
    User { name: "Bob".into(), age: 25, active: false },
    User { name: "Carol".into(), age: 35, active: true },
    User { name: "Dave".into(), age: 28, active: true },
];

// Names of active users over 27, sorted
let names: Vec<&str> = users.iter()
    .filter(|u| u.active)
    .filter(|u| u.age > 27)
    .map(|u| u.name.as_str())
    .collect();
// ["Alice", "Carol", "Dave"]
```

**C equivalent:**

```c
// Manual, error-prone, allocates intermediate storage
char* names[MAX_USERS];
int count = 0;
for (int i = 0; i < num_users; i++) {
    if (users[i].active && users[i].age > 27) {
        names[count++] = users[i].name;
    }
}
```

**Python equivalent:**

```python
names = [u.name for u in users if u.active and u.age > 27]
```

Python's comprehension is concise for simple cases but doesn't chain well:

```python
# Gets ugly fast
result = [
    process(x)
    for x in (
        transform(y)
        for y in items
        if condition1(y)
    )
    if condition2(x)
]

# vs Ferrum's linear chain
let result = items.iter()
    .filter(condition1)
    .map(transform)
    .filter(condition2)
    .map(process)
    .collect();
```

---

## Common Patterns

### Transform and Collect

The most common pattern: transform each element and collect results.

```ferrum
// Parse strings to integers
let strings = ["1", "2", "3", "4"];
let numbers: Vec<i32> = strings.iter()
    .map(|s| s.parse().unwrap())
    .collect();

// Extract fields
let names: Vec<&str> = users.iter()
    .map(|u| u.name.as_str())
    .collect();
```

### Filter and Collect

Keep only elements matching criteria.

```ferrum
// Keep only valid results
let results: Vec<i32> = inputs.iter()
    .filter_map(|x| x.parse::<i32>().ok())  // filter + map in one
    .collect();

// Partition into two collections
let (evens, odds): (Vec<i32>, Vec<i32>) = numbers.iter()
    .partition(|x| *x % 2 == 0);
```

### Find or Default

Search with a fallback.

```ferrum
let config = settings.iter()
    .find(|s| s.key == "timeout")
    .map(|s| s.value)
    .unwrap_or(30);  // Default to 30 if not found
```

### Enumerate for Index Access

When you need both index and value:

```ferrum
for (index, value) in items.iter().enumerate() {
    println("{}: {}", index, value);
}

// Find index of maximum
let max_index = numbers.iter()
    .enumerate()
    .max_by_key(|(_, v)| *v)
    .map(|(i, _)| i);
```

### Zip for Parallel Iteration

Process two collections together:

```ferrum
let names = ["Alice", "Bob", "Carol"];
let scores = [95, 87, 92];

for (name, score) in names.iter().zip(scores.iter()) {
    println("{}: {}", name, score);
}

// Dot product
let dot: i32 = a.iter()
    .zip(b.iter())
    .map(|(x, y)| x * y)
    .sum();
```

### Flatten Nested Structures

```ferrum
let nested = vec![vec![1, 2], vec![3, 4], vec![5]];
let flat: Vec<i32> = nested.iter()
    .flatten()
    .copied()
    .collect();
// [1, 2, 3, 4, 5]

// flat_map combines map + flatten
let words: Vec<&str> = lines.iter()
    .flat_map(|line| line.split_whitespace())
    .collect();
```

---

## Zero-Cost Abstraction

Here's the key insight: Ferrum's iterator chains compile to the same machine code as hand-written loops.

```ferrum
// This high-level code...
let sum: i32 = numbers.iter()
    .filter(|x| *x % 2 == 0)
    .map(|x| x * x)
    .sum();

// ...compiles to essentially this:
let mut sum = 0;
for x in numbers {
    if x % 2 == 0 {
        sum += x * x;
    }
}
```

The compiler:
1. Inlines all the closure calls
2. Eliminates iterator object creation
3. Fuses the filter/map/sum into a single loop
4. Produces the same assembly as C

This is called "zero-cost abstraction": you pay nothing at runtime for the higher-level syntax. Profile it yourself—the iterator version matches hand-written loops cycle for cycle.

---

## Quick Reference

| Method | Purpose | Example |
|--------|---------|---------|
| `.iter()` | Get iterator over references | `vec.iter()` |
| `.into_iter()` | Get iterator, consuming collection | `vec.into_iter()` |
| `.map(f)` | Transform each element | `.map(\|x\| x * 2)` |
| `.filter(p)` | Keep elements where p is true | `.filter(\|x\| x > 0)` |
| `.filter_map(f)` | Map + filter None results | `.filter_map(\|x\| x.parse().ok())` |
| `.fold(init, f)` | Reduce to single value | `.fold(0, \|a, x\| a + x)` |
| `.collect()` | Gather into collection | `.collect::<Vec<_>>()` |
| `.find(p)` | First element where p is true | `.find(\|x\| x.id == 5)` |
| `.any(p)` | True if any element matches | `.any(\|x\| x < 0)` |
| `.all(p)` | True if all elements match | `.all(\|x\| x > 0)` |
| `.count()` | Number of elements | `.count()` |
| `.sum()` | Sum of elements | `.sum::<i32>()` |
| `.take(n)` | First n elements | `.take(5)` |
| `.skip(n)` | Skip first n elements | `.skip(2)` |
| `.enumerate()` | Pair with indices | `.enumerate()` |
| `.zip(other)` | Pair with another iterator | `.zip(other.iter())` |
| `.flatten()` | Flatten nested iterators | `.flatten()` |
| `.cloned()` | Clone referenced elements | `.cloned()` |
| `.copied()` | Copy referenced elements | `.copied()` |

---

## Summary

Ferrum iterators give you:

- **Readability**: Chain operations in a clear pipeline
- **Safety**: No index errors, no iterator invalidation
- **Performance**: Zero-cost—compiles to tight loops
- **Composability**: Build complex transforms from simple parts
- **Laziness**: Process huge or infinite sequences efficiently

Coming from C, you get the same performance with less boilerplate and fewer bugs. Coming from Python, you get the same expressiveness without the memory overhead of intermediate lists.

The learning curve is modest: master `map`, `filter`, `fold`, and `collect`, and you can handle 90% of collection processing. The rest you'll pick up as needed.
