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

### What Is an Iterator?

An iterator is an object that produces a sequence of values one at a time. Think of it like a cursor moving through a collection—each time you ask for the next value, it gives you one and advances.

In C, you'd manually manage an index variable:

```c
int i = 0;              // Your "cursor"
while (i < n) {
    process(arr[i]);    // Get current value
    i++;                // Advance cursor
}
```

An iterator wraps this pattern into an object. You don't manage the index—you just keep asking "give me the next item" until there are no more.

### Getting an Iterator

Every collection in Ferrum has an `.iter()` method that returns an iterator:

```ferrum
let numbers = [1, 2, 3, 4, 5];
let iter = numbers.iter();  // Iterator over &i32 references
```

Three variations exist:
- `.iter()` — iterates over references (`&T`) — you can look but not touch
- `.iter_mut()` — iterates over mutable references (`&mut T`) — you can modify in place
- `.into_iter()` — consumes the collection, iterates over owned values (`T`) — you take the values out

```ferrum
let mut scores = [10, 20, 30];

// Read-only access
for score in scores.iter() {
    println("{}", score);
}

// Modify in place
for score in scores.iter_mut() {
    *score += 5;  // The * dereferences the mutable reference
}

// Consume the array (can't use scores after this)
let doubled: Vec[i32] = scores.into_iter().map(|x| x * 2).collect();
```

**When to use which:**
- Use `.iter()` most of the time—it's the safest and most common
- Use `.iter_mut()` when you need to modify elements without creating a new collection
- Use `.into_iter()` when you're done with the original collection and want to transform it into something else

### The `for` Loop

The `for` loop automatically uses iterators under the hood:

```ferrum
// This clean syntax...
for x in collection {
    println("{}", x);
}

// ...is equivalent to this explicit version:
let mut iter = collection.into_iter();
loop {
    match iter.next() {
        Some(x) => println("{}", x),
        None => break,
    }
}
```

The `.next()` method returns `Some(value)` if there's another item, or `None` when the iterator is exhausted. The `for` loop hides this bookkeeping from you.

### Iterators Wait Until You Need Them

A key concept: iterator operations don't run immediately. They set up a *pipeline* that only executes when you actually consume the results.

```ferrum
let numbers = [1, 2, 3, 4, 5];

// This line does NOTHING yet—no computation happens
let iter = numbers.iter()
    .filter(|x| **x > 2)
    .map(|x| x * 10);

// Only NOW does computation happen, as we pull values out
for n in iter {
    println("{}", n);  // Prints 30, 40, 50
}
```

Why does this matter? Two reasons:

**1. Memory efficiency:** You never build intermediate collections.

```ferrum
// This processes one element at a time through the entire chain
// No intermediate Vec[i32] is ever created
let result: i32 = huge_data.iter()
    .filter(|x| x.is_valid())
    .map(|x| x.compute_expensive())
    .sum();
```

**2. You can work with infinite sequences:**

```ferrum
// This would be impossible if filter() tried to process everything upfront
let first_10_evens: Vec[i32] = (0..)   // 0, 1, 2, 3, ... forever
    .filter(|x| x % 2 == 0)             // Keep only evens
    .take(10)                           // Stop after 10
    .collect();                         // Now compute and collect
// [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

Compare to Python, where a list comprehension runs immediately:

```python
# Python: this runs RIGHT NOW and allocates a list
result = [x * 10 for x in numbers if x > 2]  # [30, 40, 50] created immediately
```

Python generators (`(x for x in ...)`) are similar to Ferrum iterators, but they're less common and can't be chained as cleanly.

---

## Closures: Anonymous Functions That Remember Things

Before we dive into iterator methods, you need to understand closures. A closure is a small, unnamed function that you can write inline. The special thing about closures is they can "remember" variables from the surrounding code.

### Reading the Syntax

If you've never seen `|x| x * 2` before, here's how to read it:

```
|x| x * 2
 │  └────── The body: what to do with x
 └──────── The parameter list: this closure takes one argument called x
```

The pipes `| |` replace the parentheses you'd use in a regular function. Think of `|x|` as meaning "given x".

```ferrum
// Regular function
fn double(x: i32) -> i32 {
    x * 2
}

// Same thing as a closure
let double = |x| x * 2;

// Calling them looks identical
double(5)  // Returns 10 either way
```

### Multiple Parameters and No Parameters

```ferrum
// No parameters: empty pipes
let say_hello = || println("Hello!");
say_hello();  // Prints: Hello!

// Two parameters
let add = |a, b| a + b;
add(3, 4);  // Returns 7

// Three parameters with explicit types (usually optional)
let clamp = |value: i32, min: i32, max: i32| -> i32 {
    if value < min { min }
    else if value > max { max }
    else { value }
};
```

### Multi-Line Closures

For anything beyond a simple expression, use braces:

```ferrum
let process = |x| {
    let doubled = x * 2;
    let incremented = doubled + 1;
    incremented * incremented  // Last expression is the return value
};

process(3);  // (3 * 2 + 1)² = 49
```

### Capturing Variables: The Magic Part

Unlike C function pointers, closures can grab variables from the surrounding scope:

```ferrum
let multiplier = 3;
let scale = |x| x * multiplier;  // Uses `multiplier` from outside

println("{}", scale(10));  // 30
println("{}", scale(20));  // 60
```

The closure "captures" `multiplier` and carries it along. You don't pass it as a parameter—it just works.

In C, you'd need to manually pass context through a void pointer:

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

### How Capture Works

Ferrum decides how to capture variables based on what you do with them:

**Reading a variable? Capture by reference (borrow it):**

```ferrum
let name = String.from("Alice");
let greet = || println("Hello, {}", name);  // Just reads `name`

greet();           // Works
println("{}", name);  // Still works—we only borrowed it
```

**Modifying a variable? Capture by mutable reference:**

```ferrum
let mut count = 0;
let mut increment = || {
    count += 1;  // Modifies `count`
};

increment();  // count is now 1
increment();  // count is now 2
```

Note: the closure itself must be declared `mut` because calling it modifies state.

**Need the closure to own the value? Use `move`:**

```ferrum
let name = String.from("Alice");
let greet = move || println("Hello, {}", name);  // Takes ownership

greet();           // Works
// println("{}", name);  // ERROR: name was moved into the closure
```

The `move` keyword is essential when the closure needs to outlive the current scope:

```ferrum
fn make_greeter(name: String) -> impl Fn() {
    // Without `move`, the closure would try to reference `name`
    // But `name` gets dropped when make_greeter returns!
    move || println("Hello, {}", name)
}

let greet = make_greeter(String.from("Bob"));
greet();  // Prints: Hello, Bob
```

### A Practical Capture Example

Here's a real use case—counting how many items pass a filter:

```ferrum
fn count_matching[T](items: &[T], predicate: impl Fn(&T) -> bool): usize {
    let mut count = 0;
    for item in items {
        if predicate(item) {
            count += 1;
        }
    }
    count
}

let threshold = 50;
let scores = [85, 42, 91, 38, 67];

// The closure captures `threshold` from the surrounding scope
let high_scores = count_matching(&scores, |score| *score > threshold);
// high_scores = 3
```

---

## Iterator Methods

Now let's look at the methods that make iterators powerful.

### Transforming: `map()`

Apply a function to each element, producing a new iterator of transformed values:

```ferrum
let numbers = [1, 2, 3, 4];
let doubled: Vec[i32] = numbers.iter()
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

**Practical example—parsing log timestamps:**

```ferrum
let log_lines = [
    "2024-01-15 10:30:00 INFO Starting",
    "2024-01-15 10:30:01 ERROR Failed",
    "2024-01-15 10:30:02 INFO Recovered",
];

// Extract just the timestamps
let timestamps: Vec[&str] = log_lines.iter()
    .map(|line| &line[0..19])  // First 19 chars are the timestamp
    .collect();
// ["2024-01-15 10:30:00", "2024-01-15 10:30:01", "2024-01-15 10:30:02"]
```

### Filtering: `filter()`

Keep only elements that satisfy a condition:

```ferrum
let numbers = [1, 2, 3, 4, 5, 6];
let evens: Vec[i32] = numbers.iter()
    .filter(|x| *x % 2 == 0)
    .copied()  // Convert &i32 to i32
    .collect();
// [2, 4, 6]
```

Note: `filter()` gives you references to the original elements, so we use `.copied()` to get owned values. This makes sense—filtering doesn't change values, so why copy them until necessary?

**Practical example—filtering API response data:**

```ferrum
struct User {
    id: u64,
    name: String,
    email: String,
    active: bool,
    last_login_days_ago: u32,
}

let users: Vec[User] = fetch_users_from_api();

// Find users who haven't logged in for 30+ days and are still active
let inactive_actives: Vec[&User] = users.iter()
    .filter(|u| u.active && u.last_login_days_ago >= 30)
    .collect();

// Generate reminder emails
for user in inactive_actives {
    send_reminder_email(&user.email);
}
```

### Aggregating: `fold()`

Combine all elements into a single value. You provide:
1. A starting value (the "accumulator")
2. A function that takes (accumulator, current_element) and returns the new accumulator

```ferrum
let numbers = [1, 2, 3, 4, 5];

// Sum: start at 0, keep adding
let sum = numbers.iter().fold(0, |acc, x| acc + x);
// Step by step: 0 → 0+1=1 → 1+2=3 → 3+3=6 → 6+4=10 → 10+5=15
// Result: 15

// Product: start at 1, keep multiplying
let product = numbers.iter().fold(1, |acc, x| acc * x);
// Result: 120

// Build a comma-separated string
let csv = numbers.iter()
    .map(|x| x.to_string())
    .fold(String.new(), |acc, s| {
        if acc.is_empty() { s } else { format("{},{}", acc, s) }
    });
// "1,2,3,4,5"
```

**C equivalent:**

```c
int sum = 0;  // Starting value
for (int i = 0; i < 5; i++) {
    sum = sum + numbers[i];  // Accumulate
}
```

**Practical example—aggregating sales data:**

```ferrum
struct Sale {
    product_id: u64,
    quantity: u32,
    price_cents: u64,
}

let sales: Vec[Sale] = load_daily_sales();

// Calculate total revenue
let total_cents: u64 = sales.iter()
    .fold(0, |total, sale| total + (sale.quantity as u64 * sale.price_cents));

println("Total revenue: ${:.2}", total_cents as f64 / 100.0);
```

Common aggregations have shortcuts so you don't need `fold`:

```ferrum
let numbers = [1, 2, 3, 4, 5];
let sum: i32 = numbers.iter().sum();           // 15
let product: i32 = numbers.iter().product();   // 120
let count = numbers.iter().count();            // 5
let max = numbers.iter().max();                // Some(&5)
let min = numbers.iter().min();                // Some(&1)
```

Note that `max()` and `min()` return `Option` because the collection might be empty.

### Collecting: `collect()`

Turn an iterator back into a concrete collection:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// Into a Vec
let vec: Vec[i32] = numbers.iter().copied().collect();

// Into a HashSet (removes duplicates)
let set: HashSet[i32] = numbers.iter().copied().collect();

// Into a String (for char iterators)
let s: String = ['h', 'e', 'l', 'l', 'o'].iter().collect();
```

The type annotation tells `collect()` what to produce.

**Practical example—building a lookup table:**

```ferrum
struct Config {
    key: String,
    value: String,
}

let configs: Vec[Config] = load_config_file();

// Build a HashMap for O(1) lookup
let config_map: HashMap[String, String] = configs
    .into_iter()
    .map(|c| (c.key, c.value))
    .collect();

// Now we can quickly look up any config value
let timeout = config_map.get("timeout").unwrap_or(&"30".to_string());
```

### Finding: `find()`, `position()`

Search for elements:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// Find first even number (returns Option)
let first_even = numbers.iter().find(|x| *x % 2 == 0);
// Some(&2)

// Find index of first even number
let pos = numbers.iter().position(|x| x % 2 == 0);
// Some(1)

// Find with transformation
let first_big_square = numbers.iter()
    .map(|x| x * x)
    .find(|x| *x > 10);
// Some(16)
```

These stop as soon as they find a match—they don't scan the whole collection.

### Testing: `any()`, `all()`

Check conditions across a collection:

```ferrum
let numbers = [1, 2, 3, 4, 5];

let has_even = numbers.iter().any(|x| x % 2 == 0);      // true
let all_positive = numbers.iter().all(|x| *x > 0);      // true
let all_even = numbers.iter().all(|x| x % 2 == 0);      // false
```

These are "short-circuit" operations:
- `any()` stops at the first `true` (no need to check the rest)
- `all()` stops at the first `false` (one failure means the whole thing fails)

---

## Chaining Operations

The real power comes from chaining operations into a pipeline:

```ferrum
struct User {
    name: String,
    age: u32,
    active: bool,
}

let users = vec[
    User { name: "Alice".into(), age: 30, active: true },
    User { name: "Bob".into(), age: 25, active: false },
    User { name: "Carol".into(), age: 35, active: true },
    User { name: "Dave".into(), age: 28, active: true },
];

// Names of active users over 27
let names: Vec[&str] = users.iter()
    .filter(|u| u.active)
    .filter(|u| u.age > 27)
    .map(|u| u.name.as_str())
    .collect();
// ["Alice", "Carol", "Dave"]
```

Read it top-to-bottom: start with users, keep active ones, keep those over 27, extract names, collect into a vector.

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

Python's comprehension is concise for simple cases but doesn't chain as well. When things get complex:

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

The Ferrum version reads like a recipe: filter, transform, filter, transform, collect.

---

## Real-World Examples

### Processing Log Files

```ferrum
let log_contents = fs.read_to_string("server.log")?;

// Count error messages per hour
let error_counts: HashMap[String, usize] = log_contents
    .lines()
    .filter(|line| line.contains("ERROR"))
    .map(|line| {
        // Extract the hour: "2024-01-15 10:30:00" -> "10"
        let hour = &line[11..13];
        hour.to_string()
    })
    .fold(HashMap.new(), |mut counts, hour| {
        *counts.entry(hour).or_insert(0) += 1;
        counts
    });

// Print results
for (hour, count) in &error_counts {
    println("Hour {}: {} errors", hour, count);
}
```

### Transforming API Responses

```ferrum
#[derive(Deserialize)]
struct ApiUser {
    id: u64,
    username: String,
    email: String,
    created_at: String,
    deleted: bool,
}

struct DisplayUser {
    id: u64,
    display_name: String,
}

fn fetch_active_users(): Result[Vec[DisplayUser], Error] {
    let response: Vec[ApiUser] = http.get("https://api.example.com/users")
        .json()?;

    let active_users: Vec[DisplayUser] = response
        .into_iter()
        .filter(|u| !u.deleted)
        .map(|u| DisplayUser {
            id: u.id,
            display_name: format("{} <{}>", u.username, u.email),
        })
        .collect();

    Ok(active_users)
}
```

### Aggregating Data from Multiple Sources

```ferrum
struct Order {
    user_id: u64,
    product_id: u64,
    quantity: u32,
    total_cents: u64,
}

// Calculate total spent per user
fn spending_by_user(orders: &[Order]): HashMap[u64, u64] {
    orders.iter()
        .fold(HashMap.new(), |mut totals, order| {
            *totals.entry(order.user_id).or_insert(0) += order.total_cents;
            totals
        })
}

// Find top 10 spenders
fn top_spenders(orders: &[Order], n: usize): Vec[(u64, u64)] {
    let mut spending: Vec[(u64, u64)] = spending_by_user(orders)
        .into_iter()
        .collect();

    spending.sort_by(|a, b| b.1.cmp(&a.1));  // Sort by amount descending

    spending.into_iter().take(n).collect()
}
```

### Parsing Configuration Files

```ferrum
// Parse a KEY=VALUE config file, skipping comments and empty lines
fn parse_config(content: &str): HashMap[String, String] {
    content
        .lines()
        .map(|line| line.trim())
        .filter(|line| !line.is_empty() && !line.starts_with('#'))
        .filter_map(|line| {
            let mut parts = line.splitn(2, '=');
            let key = parts.next()?.trim().to_string();
            let value = parts.next()?.trim().to_string();
            Some((key, value))
        })
        .collect()
}

// Usage:
// DATABASE_URL=postgres://localhost/mydb
// # This is a comment
// TIMEOUT=30
let config = parse_config(&fs.read_to_string("app.conf")?);
let db_url = config.get("DATABASE_URL").expect("DATABASE_URL required");
```

---

## Common Patterns

### Transform and Collect

The most common pattern: transform each element and collect results.

```ferrum
// Parse strings to integers
let strings = ["1", "2", "3", "4"];
let numbers: Vec[i32] = strings.iter()
    .map(|s| s.parse().unwrap())
    .collect();

// Extract fields from structs
let names: Vec[&str] = users.iter()
    .map(|u| u.name.as_str())
    .collect();
```

### Filter-Map in One Step

When you want to filter *and* transform at once, `filter_map` is your friend:

```ferrum
let strings = ["1", "two", "3", "four", "5"];

// Parse only the valid integers, skip the rest
let numbers: Vec[i32] = strings.iter()
    .filter_map(|s| s.parse().ok())  // ok() converts Result to Option; type inferred from Vec[i32]
    .collect();
// [1, 3, 5]
```

This is cleaner than:

```ferrum
// More verbose equivalent
let numbers: Vec[i32] = strings.iter()
    .map(|s| s.parse())
    .filter(|r| r.is_ok())
    .map(|r| r.unwrap())
    .collect();
```

### Partition into Two Groups

```ferrum
let numbers = [1, 2, 3, 4, 5, 6];
let (evens, odds): (Vec[i32], Vec[i32]) = numbers.iter()
    .copied()
    .partition(|x| x % 2 == 0);
// evens = [2, 4, 6]
// odds = [1, 3, 5]
```

### Find or Use Default

```ferrum
let settings = vec[
    ("timeout", 30),
    ("retries", 3),
];

let timeout = settings.iter()
    .find(|(key, _)| *key == "timeout")
    .map(|(_, value)| *value)
    .unwrap_or(60);  // Default to 60 if not found
```

### Enumerate for Index Access

When you need both index and value:

```ferrum
for (index, value) in items.iter().enumerate() {
    println("{}: {}", index, value);
}

// Find index of maximum value
let max_index = numbers.iter()
    .enumerate()
    .max_by_key(|(_, v)| *v)
    .map(|(i, _)| i);
```

### Zip for Parallel Iteration

Process two collections together, element by element:

```ferrum
let names = ["Alice", "Bob", "Carol"];
let scores = [95, 87, 92];

for (name, score) in names.iter().zip(scores.iter()) {
    println("{}: {}", name, score);
}

// Dot product of two vectors
let dot: i32 = a.iter()
    .zip(b.iter())
    .map(|(x, y)| x * y)
    .sum();
```

### Flatten Nested Structures

```ferrum
let nested = vec[vec[1, 2], vec[3, 4], vec[5]];
let flat: Vec[i32] = nested.iter()
    .flatten()
    .copied()
    .collect();
// [1, 2, 3, 4, 5]

// flat_map = map + flatten in one step
let words: Vec[&str] = lines.iter()
    .flat_map(|line| line.split_whitespace())
    .collect();
```

### Take and Skip

```ferrum
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// First 3 elements
let first_three: Vec[i32] = numbers.iter().copied().take(3).collect();
// [1, 2, 3]

// Skip first 3, take rest
let rest: Vec[i32] = numbers.iter().copied().skip(3).collect();
// [4, 5, 6, 7, 8, 9, 10]

// Pagination: page 2, 3 items per page
let page_2: Vec[i32] = numbers.iter()
    .copied()
    .skip(3)      // Skip page 1
    .take(3)      // Take page 2
    .collect();
// [4, 5, 6]
```

---

## Common Mistakes

### Mistake 1: Forgetting That Iterators Are Consumed

An iterator can only be used once. After you iterate through it, it's empty.

```ferrum
let numbers = [1, 2, 3];
let iter = numbers.iter();

let sum: i32 = iter.sum();      // Consumes the iterator
let count = iter.count();        // ERROR: iter is already consumed
```

**Compiler error:**

```
error[E0382]: use of moved value: `iter`
 --> src/main.rs:5:13
  |
3 |     let iter = numbers.iter();
  |         ---- move occurs because `iter` has type `Iter<'_, i32>`
4 |     let sum: i32 = iter.sum();
  |                    ---- `iter` moved due to this method call
5 |     let count = iter.count();
  |                 ^^^^ value used here after move
```

**Fix:** Create a new iterator, or collect into a Vec first if you need multiple passes:

```ferrum
let numbers = [1, 2, 3];
let sum: i32 = numbers.iter().sum();      // Fresh iterator
let count = numbers.iter().count();        // Another fresh iterator
```

### Mistake 2: Reference vs. Value Confusion

`.iter()` yields references, which can cause confusion:

```ferrum
let numbers = [1, 2, 3, 4, 5];

// WRONG: comparing &i32 with i32
let evens: Vec[i32] = numbers.iter()
    .filter(|x| x % 2 == 0)   // x is &&i32 here!
    .collect();
```

**Compiler error:**

```
error[E0369]: cannot mod `&&i32` by `{integer}`
 --> src/main.rs:4:20
  |
4 |     .filter(|x| x % 2 == 0)
  |                 - ^ - {integer}
  |                 |
  |                 &&i32
```

**Fix:** Dereference with `*` or use `.copied()`:

```ferrum
// Option 1: Dereference in the closure
let evens: Vec[&i32] = numbers.iter()
    .filter(|x| *x % 2 == 0)
    .collect();

// Option 2: Use copied() to work with values
let evens: Vec[i32] = numbers.iter()
    .copied()                    // Now we have i32, not &i32
    .filter(|x| x % 2 == 0)      // x is i32
    .collect();
```

### Mistake 3: Not Consuming the Iterator

Iterator chains are lazy—nothing happens until you consume them.

```ferrum
let numbers = [1, 2, 3, 4, 5];

// This does NOTHING!
numbers.iter()
    .map(|x| {
        println("Processing {}", x);  // Never printed
        x * 2
    });

println("Done");  // Prints "Done" but nothing was processed
```

**Compiler warning:**

```
warning: unused `Map` that must be used
 --> src/main.rs:3:1
  |
3 | /     numbers.iter()
4 | |         .map(|x| {
5 | |             println("Processing {}", x);
6 | |             x * 2
7 | |         });
  | |__________^
  |
  = note: iterators are lazy and do nothing unless consumed
```

**Fix:** Add a consuming operation:

```ferrum
// Option 1: Collect the results
let doubled: Vec[i32] = numbers.iter()
    .map(|x| x * 2)
    .collect();

// Option 2: Use for_each for side effects
numbers.iter()
    .for_each(|x| println("Processing {}", x));
```

### Mistake 4: Mutating While Iterating

You can't modify a collection while iterating over it:

```ferrum
let mut numbers = vec[1, 2, 3, 4, 5];

// WRONG: Can't modify vec while iterating
for x in &numbers {
    if *x > 3 {
        numbers.push(*x * 2);  // ERROR!
    }
}
```

**Compiler error:**

```
error[E0502]: cannot borrow `numbers` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:9
  |
4 |     for x in &numbers {
  |              --------
  |              |
  |              immutable borrow occurs here
  |              immutable borrow later used here
5 |         if *x > 3 {
6 |             numbers.push(*x * 2);
  |             ^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

**Fix:** Collect first, then modify:

```ferrum
let mut numbers = vec[1, 2, 3, 4, 5];

// Collect what to add first
let to_add: Vec[i32] = numbers.iter()
    .filter(|x| **x > 3)
    .map(|x| *x * 2)
    .collect();

// Then modify
numbers.extend(to_add);
```

### Mistake 5: Using `unwrap()` in Chains Without Thinking

Calling `.unwrap()` inside a chain will panic if any element fails:

```ferrum
let strings = ["1", "2", "three", "4"];

// This panics on "three"
let numbers: Vec[i32] = strings.iter()
    .map(|s| s.parse().unwrap())  // PANIC on "three"; type inferred from Vec[i32]
    .collect();
```

**Fix:** Use `filter_map()` to skip failures, or handle errors explicitly:

```ferrum
// Skip failures
let numbers: Vec[i32] = strings.iter()
    .filter_map(|s| s.parse().ok())
    .collect();
// [1, 2, 4]

// Or collect Results to handle errors
let results: Result[Vec[i32], _] = strings.iter()
    .map(|s| s.parse())
    .collect();
// Err(ParseIntError { kind: InvalidDigit })
```

### Mistake 6: Closure Capture Surprises

Closures capture by reference by default, which can cause issues:

```ferrum
fn make_closures(): Vec[impl Fn() -> i32] {
    let mut closures = vec[];
    for i in 0..3 {
        closures.push(|| i);  // ERROR: `i` doesn't live long enough
    }
    closures
}
```

**Fix:** Use `move` to take ownership:

```ferrum
fn make_closures(): Vec[impl Fn() -> i32] {
    let mut closures = vec[];
    for i in 0..3 {
        closures.push(move || i);  // Each closure owns its own copy of i
    }
    closures
}
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
| `.iter_mut()` | Get iterator over mutable references | `vec.iter_mut()` |
| `.into_iter()` | Get iterator, consuming collection | `vec.into_iter()` |
| `.map(f)` | Transform each element | `.map(\|x\| x * 2)` |
| `.filter(p)` | Keep elements where p returns true | `.filter(\|x\| x > 0)` |
| `.filter_map(f)` | Map then filter out None values | `.filter_map(\|x\| x.parse().ok())` |
| `.fold(init, f)` | Reduce to single value | `.fold(0, \|a, x\| a + x)` |
| `.collect()` | Gather into collection | `let v: Vec[_] = iter.collect()` |
| `.find(p)` | First element where p returns true | `.find(\|x\| x.id == 5)` |
| `.position(p)` | Index of first match | `.position(\|x\| x == 5)` |
| `.any(p)` | True if any element matches | `.any(\|x\| x < 0)` |
| `.all(p)` | True if all elements match | `.all(\|x\| x > 0)` |
| `.count()` | Number of elements | `.count()` |
| `.sum()` | Sum of elements | `let s: i32 = iter.sum()` |
| `.product()` | Product of elements | `let p: i32 = iter.product()` |
| `.max()` / `.min()` | Maximum / minimum element | `.max()` |
| `.take(n)` | First n elements | `.take(5)` |
| `.skip(n)` | Skip first n elements | `.skip(2)` |
| `.enumerate()` | Pair with indices | `.enumerate()` |
| `.zip(other)` | Pair with another iterator | `.zip(other.iter())` |
| `.flatten()` | Flatten nested iterators | `.flatten()` |
| `.flat_map(f)` | Map then flatten | `.flat_map(\|x\| x.chars())` |
| `.cloned()` | Clone referenced elements | `.cloned()` |
| `.copied()` | Copy referenced elements (for Copy types) | `.copied()` |
| `.partition(p)` | Split into two collections | `.partition(\|x\| x % 2 == 0)` |
| `.for_each(f)` | Execute for side effects | `.for_each(\|x\| println("{}", x))` |

---

## Summary

Ferrum iterators give you:

- **Readability**: Chain operations in a clear pipeline that reads like a recipe
- **Safety**: No index errors, no iterator invalidation, the compiler catches mistakes
- **Performance**: Zero-cost—compiles to tight loops identical to hand-written C
- **Composability**: Build complex transforms from simple, reusable parts
- **Laziness**: Process huge or infinite sequences without allocating intermediate storage

Coming from C, you get the same performance with less boilerplate and fewer bugs. Coming from Python, you get the same expressiveness without the memory overhead of intermediate lists.

The learning curve is modest: master `map`, `filter`, `fold`, and `collect`, and you can handle 90% of collection processing. The rest you'll pick up as needed.
