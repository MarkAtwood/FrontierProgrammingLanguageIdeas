# Ferrum Smart Pointers: A Practical Introduction

**Target audience:** Software engineers familiar with C's manual memory management and Python's garbage collection. No type theory background required.

---

## The Problem: Two Extremes, Neither Great

You've probably worked with both ends of the spectrum.

**C gives you full control, but you carry the burden:**

```c
char *buf = malloc(1024);
process(buf);
if (error_occurred) {
    return -1;  // Oops, forgot to free buf
}
free(buf);

// Later, in another function...
free(buf);  // Double-free: undefined behavior, possibly a security hole

// Even later...
printf("%s", buf);  // Use-after-free: reading garbage, or crashing
```

Every `malloc` needs exactly one `free`. Not zero, not two. On every code path. Including error paths. Including early returns. Including exceptions (if you're using C++). You spend mental energy tracking who "owns" memory instead of solving your actual problem.

**Python takes the burden away, but also takes control:**

```python
def process():
    huge_data = load_huge_dataset()  # 10 GB in RAM
    result = analyze(huge_data)

    # huge_data is "done" here, but...
    # When does Python actually free those 10 GB?

    do_more_work()  # This might run out of memory waiting for GC

    return result
```

The garbage collector frees memory "eventually." You can't say "free this now." You can call `gc.collect()`, but it might not free your object if there's a reference cycle. GC pauses can hit at bad times. And you have no visibility into when memory actually gets reclaimed.

**What we really want:**

- Memory freed automatically (like Python)
- Memory freed *immediately* when we're done (not "eventually")
- Compiler catches use-after-free at compile time (unlike C)
- No runtime garbage collector (unlike Python)

Ferrum's smart pointers give you all four.

---

## The Core Idea: Automatic Cleanup at Scope Exit

Here's the key insight that makes smart pointers work:

```ferrum
fn example() {
    let data = Box.new([0u8; 1_000_000]);  // allocate 1 MB on heap

    use_data(&data);

    if something_failed {
        return;  // data is automatically freed here
    }

    more_work(&data);

}   // data is automatically freed here too
```

When a value goes "out of scope" (the `}` at the end of the block, or an early `return`), Ferrum automatically runs cleanup code. For `Box`, that cleanup code frees the memory.

This pattern is sometimes called "RAII" in C++ circles, but the name is confusing. Think of it simply as: **the compiler inserts `free()` for you, at every exit point, and guarantees you can't use the value after it's freed.**

In C terms, it's as if the compiler rewrites your code to:

```c
void example() {
    char *data = malloc(1000000);

    use_data(data);

    if (something_failed) {
        free(data);  // compiler inserted this
        return;
    }

    more_work(data);

    free(data);  // compiler inserted this too
}
```

Except the compiler also checks that you don't access `data` after it's freed. Try it and you get a compile error, not a runtime crash.

---

## Choosing the Right Smart Pointer: A Decision Tree

Start here when deciding which pointer type to use:

```
Do you need HEAP allocation?
├── NO → Just use the value directly (stack allocated)
└── YES → Continue...

Does exactly ONE thing own this data?
├── YES → Use Box[T]
└── NO (multiple owners) → Continue...

Do multiple THREADS need access?
├── NO (single thread) → Use Rc[T]
└── YES (multi-threaded) → Use Arc[T]

Do you have CYCLES (A points to B, B points to A)?
└── YES → Use Weak[T] for the "back reference"

Do you need to MUTATE through a shared reference?
├── Single-threaded → Combine with RefCell[T]
└── Multi-threaded → Combine with Mutex[T] or RwLock[T]
```

Let's go through each type with concrete examples.

---

## `Box[T]`: Single Owner, Heap Allocated

`Box[T]` is the simplest smart pointer. It puts a value on the heap and owns it exclusively. When the `Box` goes out of scope, the memory is freed. One owner, one lifetime, no sharing.

```ferrum
fn process_image() {
    // Allocate a large buffer on the heap, not the stack
    let buffer = Box.new([0u8; 10_000_000]);  // 10 MB

    load_image_into(&buffer);
    process(&buffer);

}   // buffer freed automatically here
```

Think of `Box.new(value)` as `malloc` + assignment, and the closing `}` as an automatic `free`.

### When You Need Box

**Problem 1: Recursive types have infinite size**

This won't compile:

```ferrum
// ERROR: recursive type has infinite size
type LinkedList[T] {
    value: T,
    next: Option[LinkedList[T]],  // LinkedList contains LinkedList contains...
}
```

The compiler error:

```
error[E0072]: recursive type `LinkedList` has infinite size
 --> src/main.fe:2:1
  |
2 | type LinkedList[T] {
  | ^^^^^^^^^^^^^^^^^^ recursive type has infinite size
3 |     value: T,
4 |     next: Option[LinkedList[T]],
  |           --------------------- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`) to make `LinkedList` representable
  |
4 |     next: Option[Box[LinkedList[T]]],
  |           ++++++++               +
```

The fix: use `Box` for indirection. `Box` has a fixed size (one pointer), even though what it points to can vary:

```ferrum
type LinkedList[T] {
    value: T,
    next: Option[Box[LinkedList[T]]],  // Box is just a pointer - fixed size
}

fn build_list(): LinkedList[i32] {
    LinkedList {
        value: 1,
        next: Some(Box.new(LinkedList {
            value: 2,
            next: Some(Box.new(LinkedList {
                value: 3,
                next: None,
            })),
        })),
    }
}
```

**Problem 2: Stack space is limited**

Stack size is typically 1-8 MB. Try to put a huge array on the stack and you'll overflow it:

```ferrum
fn dangerous(): [u8; 50_000_000] {  // 50 MB on stack - will overflow!
    [0; 50_000_000]
}
```

The fix: heap allocate with Box:

```ferrum
fn safe(): Box[[u8; 50_000_000]] {  // 50 MB on heap - fine
    Box.new([0; 50_000_000])
}
```

**Problem 3: You need a value of "any type implementing trait X"**

Sometimes you don't know the concrete type at compile time:

```ferrum
trait Drawable {
    fn draw(&self);
}

type Circle { radius: f32 }
type Square { side: f32 }
type Triangle { base: f32, height: f32 }

impl Drawable for Circle { fn draw(&self) { /* ... */ } }
impl Drawable for Square { fn draw(&self) { /* ... */ } }
impl Drawable for Triangle { fn draw(&self) { /* ... */ } }

// What's the return type here? Circle? Square? Triangle?
// We don't know at compile time - depends on user input
fn shape_from_user_input(input: &str): Box[dyn Drawable] {
    match input {
        "circle" => Box.new(Circle { radius: 5.0 }),
        "square" => Box.new(Square { side: 10.0 }),
        _ => Box.new(Triangle { base: 3.0, height: 4.0 }),
    }
}

fn main() {
    let shapes: Vec[Box[dyn Drawable]] = vec![
        Box.new(Circle { radius: 1.0 }),
        Box.new(Square { side: 2.0 }),
    ];

    for shape in &shapes {
        shape.draw();  // Works! Calls the right implementation
    }
}
```

The `dyn Drawable` part means "some type that implements Drawable, figured out at runtime." The `Box` is needed because different types have different sizes, but `Box` (a pointer) has a fixed size.

---

## `Rc[T]`: Multiple Owners, Single Thread

Sometimes one owner isn't enough. Consider a tree where a node can have multiple parents, or a cache where multiple parts of your code hold references to the same entry.

`Rc[T]` (reference-counted pointer) solves this. Here's how it works:

1. `Rc.new(value)` creates the value and sets a counter to 1
2. `rc.clone()` increments the counter (doesn't copy the data!)
3. When an `Rc` is dropped, the counter decrements
4. When the counter hits 0, the value is actually freed

**It's just counting how many things point to the value.** When nothing points to it anymore, it's freed.

```ferrum
import alloc.rc.Rc

fn demonstrate_reference_counting() {
    let shared = Rc.new(ExpensiveData.load());  // count = 1
    print("Count: {}", Rc.strong_count(&shared));  // prints: Count: 1

    let copy1 = shared.clone();  // count = 2 (same data, new pointer)
    print("Count: {}", Rc.strong_count(&shared));  // prints: Count: 2

    let copy2 = shared.clone();  // count = 3
    print("Count: {}", Rc.strong_count(&shared));  // prints: Count: 3

    drop(copy1);  // count = 2 (copy1 goes away)
    print("Count: {}", Rc.strong_count(&shared));  // prints: Count: 2

    drop(copy2);  // count = 1
    print("Count: {}", Rc.strong_count(&shared));  // prints: Count: 1

}   // count = 0, ExpensiveData finally freed here
```

Note: `clone()` on an `Rc` is cheap! It just increments a counter and copies a pointer. It does NOT copy the underlying data.

### Concrete Example: Shared Configuration

Multiple handlers need access to the same config. You don't want to copy the config, and you can't easily determine which handler outlives the others:

```ferrum
import alloc.rc.Rc

type AppConfig {
    database_url: String,
    max_connections: u32,
    debug_mode: bool,
}

type DatabaseHandler {
    config: Rc[AppConfig],
}

type CacheHandler {
    config: Rc[AppConfig],
}

type LogHandler {
    config: Rc[AppConfig],
}

fn setup_app(): (DatabaseHandler, CacheHandler, LogHandler) {
    // Load config once
    let config = Rc.new(AppConfig {
        database_url: "postgres://localhost/mydb".to_string(),
        max_connections: 100,
        debug_mode: true,
    });

    // Share it with all handlers - each gets its own Rc pointing to same data
    let db = DatabaseHandler { config: config.clone() };
    let cache = CacheHandler { config: config.clone() };
    let log = LogHandler { config: config.clone() };

    // Original `config` can even be dropped now - handlers keep data alive
    drop(config);

    (db, cache, log)
    // Config stays alive until ALL handlers are dropped
}

impl DatabaseHandler {
    fn connect(&self) {
        // All handlers see the same config
        print("Connecting to: {}", self.config.database_url);
    }
}
```

### Concrete Example: Building a Graph

Graphs have nodes that can be reached from multiple other nodes:

```ferrum
import alloc.rc.Rc
import core.cell.RefCell

type GraphNode[T] {
    value: T,
    // Multiple nodes can point to the same neighbor
    neighbors: RefCell[Vec[Rc[GraphNode[T]]]],
}

fn build_graph(): Vec[Rc[GraphNode[i32]]] {
    // Create nodes
    let node_a = Rc.new(GraphNode {
        value: 1,
        neighbors: RefCell.new(Vec.new())
    });
    let node_b = Rc.new(GraphNode {
        value: 2,
        neighbors: RefCell.new(Vec.new())
    });
    let node_c = Rc.new(GraphNode {
        value: 3,
        neighbors: RefCell.new(Vec.new())
    });

    // A -> B, A -> C
    node_a.neighbors.borrow_mut().push(node_b.clone());
    node_a.neighbors.borrow_mut().push(node_c.clone());

    // B -> C (now C is reachable from both A and B)
    node_b.neighbors.borrow_mut().push(node_c.clone());

    // C -> B (creates a cycle B <-> C, see Weak section for how to handle)
    node_c.neighbors.borrow_mut().push(node_b.clone());

    vec![node_a, node_b, node_c]
}
```

### The Single-Thread Limitation: Rc Is Not Thread-Safe

`Rc`'s counter is a plain integer. If two threads try to increment it at the same time, you get a data race. Ferrum prevents this at compile time:

```ferrum
import alloc.rc.Rc

fn this_will_not_compile() {
    let shared = Rc.new(42);

    scope s {
        let copy = shared.clone();
        s.spawn(|| {
            print("{}", *copy);  // Try to use Rc in another thread
        });
    }
}
```

The compiler error:

```
error[E0277]: `Rc[i32]` cannot be sent between threads safely
  --> src/main.fe:8:17
   |
8  |         s.spawn(|| {
   |           ----- ^^ `Rc[i32]` cannot be sent between threads safely
   |           |
   |           required by a bound introduced by this call
9  |             print("{}", *copy);
   |                          ---- `Rc[i32]` is captured here
   |
   = help: the trait `Send` is not implemented for `Rc[i32]`
   = note: required for `&Rc[i32]` to implement `Send`
note: required because it's used within this closure
  --> src/main.fe:8:17
   |
8  |         s.spawn(|| {
   |                 ^^
help: consider using `Arc[i32]` instead, which is thread-safe
   |
4  |     let shared = Arc.new(42);
   |                  ~~~
```

This isn't a runtime crash. The code never compiles. Data races are impossible because you can't even build the broken program.

---

## `Arc[T]`: Multiple Owners, Thread-Safe

`Arc[T]` is `Rc[T]` with atomic operations. The only difference: the counter uses CPU atomic instructions that are safe across threads.

```ferrum
import sync.Arc

fn share_across_threads() {
    let config = Arc.new(AppConfig.load());

    scope s {
        for i in 0..4 {
            let cfg = config.clone();  // Atomic increment
            s.spawn(|| {
                // Each thread has its own Arc pointing to the same config
                print("Thread {}: max_conn = {}", i, cfg.max_connections);
            });  // Atomic decrement when thread ends
        }
    }   // All threads joined, config dropped if this was last reference
}
```

### Arc vs Rc: When to Use Which

| Situation | Use |
|-----------|-----|
| Single-threaded program | `Rc` (faster) |
| Data shared across threads | `Arc` (required) |
| Not sure / might add threads later | `Arc` (safe default) |

The performance difference is small in most real code. An atomic increment is about 10-20x slower than a regular increment, but you're incrementing reference counts, not doing math in a tight loop. Unless profiling shows it matters, just use `Arc` if there's any chance of threading.

### Concrete Example: Thread-Safe Cache

```ferrum
import sync.{Arc, Mutex}
import collections.HashMap

type Cache[K, V] {
    data: Arc[Mutex[HashMap[K, V]]],
}

impl[K: Eq + Hash + Clone, V: Clone] Cache[K, V] {
    fn new(): Self {
        Cache {
            data: Arc.new(Mutex.new(HashMap.new())),
        }
    }

    fn get(&self, key: &K): Option[V] {
        let guard = self.data.lock();  // Blocks until lock acquired
        guard.get(key).cloned()
    }   // Lock released when guard goes out of scope

    fn set(&self, key: K, value: V) {
        let mut guard = self.data.lock();
        guard.insert(key, value);
    }

    // Get a new handle to the same cache (for passing to another thread)
    fn share(&self): Self {
        Cache { data: self.data.clone() }
    }
}

fn use_shared_cache() {
    let cache = Cache.new();
    cache.set("key1".to_string(), "value1".to_string());

    scope s {
        let c1 = cache.share();
        let c2 = cache.share();

        s.spawn(|| {
            c1.set("from_thread_1".to_string(), "data".to_string());
        });

        s.spawn(|| {
            if let Some(v) = c2.get(&"key1".to_string()) {
                print("Thread 2 got: {}", v);
            }
        });
    }
}
```

---

## `Weak[T]`: Breaking Reference Cycles

Here's a problem that bites everyone eventually:

```ferrum
import alloc.rc.Rc
import core.cell.RefCell

type Parent {
    name: String,
    children: RefCell[Vec[Rc[Child]]],
}

type Child {
    name: String,
    parent: Rc[Parent],  // PROBLEM: Child holds strong reference to Parent
}

fn create_family() {
    let parent = Rc.new(Parent {
        name: "Alice".to_string(),
        children: RefCell.new(Vec.new())
    });

    let child = Rc.new(Child {
        name: "Bob".to_string(),
        parent: parent.clone(),  // parent count = 2
    });

    parent.children.borrow_mut().push(child.clone());  // child count = 2

    // At end of scope:
    // - We drop `child` variable: child count = 1 (parent still holds it)
    // - We drop `parent` variable: parent count = 1 (child still holds it)
    // - Both counts stuck at 1 forever!
    // - MEMORY LEAK
}
```

The cycle: Parent -> Child -> Parent. Neither count ever reaches 0.

### The Fix: Use Weak for Back-References

`Weak[T]` is a reference that doesn't increment the count. It doesn't keep the value alive. It just lets you get to the value *if it's still alive*.

Think of it as: "I want to know about this thing, but I don't need it to stick around just for me."

```ferrum
import alloc.rc.{Rc, Weak}
import core.cell.RefCell

type Parent {
    name: String,
    children: RefCell[Vec[Rc[Child]]],
}

type Child {
    name: String,
    parent: Weak[Parent],  // FIXED: Weak doesn't keep Parent alive
}

fn create_family(): Rc[Parent] {
    let parent = Rc.new(Parent {
        name: "Alice".to_string(),
        children: RefCell.new(Vec.new())
    });

    let child = Rc.new(Child {
        name: "Bob".to_string(),
        parent: Rc.downgrade(&parent),  // Create Weak from Rc
    });

    parent.children.borrow_mut().push(child);

    parent  // Return parent; when caller drops it, everything is freed
}

// Now the ownership is clear:
// - Parent owns Children (strong references)
// - Children reference Parent weakly (don't keep it alive)
// When Parent is dropped, Children are dropped, no cycle.
```

### Using a Weak Reference

Since a `Weak` doesn't keep the value alive, the value might already be gone when you try to use it. You must check:

```ferrum
impl Child {
    fn get_parent_name(&self): Option[String] {
        // upgrade() returns Some(Rc) if value exists, None if it's been freed
        match self.parent.upgrade() {
            Some(parent_rc) => Some(parent_rc.name.clone()),
            None => {
                print("Parent was already dropped!");
                None
            }
        }
    }
}

fn demonstrate_weak() {
    let child: Rc[Child];

    {
        let parent = Rc.new(Parent {
            name: "Alice".to_string(),
            children: RefCell.new(Vec.new()),
        });

        child = Rc.new(Child {
            name: "Bob".to_string(),
            parent: Rc.downgrade(&parent),
        });

        parent.children.borrow_mut().push(child.clone());

        // Parent still alive here
        print("{:?}", child.get_parent_name());  // prints: Some("Alice")

    }   // Parent dropped here (child's Weak doesn't keep it alive)

    // Parent is gone now
    print("{:?}", child.get_parent_name());  // prints: None
}
```

### The Pattern: Strong References for Ownership, Weak for Back-References

```
Owner (strong) → Owned (strong) → Back-reference (WEAK)
     └──────────────────────────────────────┘

     The cycle is broken because Weak doesn't count as ownership
```

Common examples:
- **Tree nodes**: Parent → Children (Rc), Child → Parent (Weak)
- **Observer pattern**: Subject → Observers (Rc), Observer → Subject (Weak)
- **Cache with expiration**: Index → Entry (Weak), so entries can be dropped when nothing else uses them

---

## Interior Mutability: Mutating Shared Data

Ferrum's normal rule: many shared references (`&T`) OR one mutable reference (`&mut T`), never both at the same time.

But sometimes you need to mutate data that multiple things reference. That's where "interior mutability" comes in: types that let you mutate through a shared `&T` reference.

### `RefCell[T]`: Runtime Borrow Checking

`RefCell` moves borrow checking from compile time to runtime:

```ferrum
import core.cell.RefCell

type Counter {
    value: RefCell[i32],
}

impl Counter {
    fn new(): Self {
        Counter { value: RefCell.new(0) }
    }

    fn increment(&self) {  // Note: &self (shared), not &mut self
        let mut val = self.value.borrow_mut();  // Runtime check
        *val += 1;
    }   // borrow_mut guard dropped here

    fn get(&self): i32 {
        *self.value.borrow()  // Runtime check
    }
}

fn use_counter() {
    let counter = Counter.new();

    // Multiple things can have &Counter at the same time
    let ref1 = &counter;
    let ref2 = &counter;

    ref1.increment();  // Works: borrow_mut() succeeds
    print("{}", ref2.get());  // Works: borrow() succeeds (no overlap)
}
```

**Important**: RefCell panics if you violate borrow rules at runtime:

```ferrum
fn this_panics() {
    let cell = RefCell.new(42);

    let borrowed = cell.borrow();       // Shared borrow active
    let mutable = cell.borrow_mut();    // PANIC! Can't mutably borrow while shared borrow exists
}
```

Runtime error:

```
thread 'main' panicked at 'already borrowed: BorrowMutError'
```

Use `try_borrow()` and `try_borrow_mut()` if you want to handle this gracefully:

```ferrum
fn safe_approach() {
    let cell = RefCell.new(42);

    let borrowed = cell.borrow();

    match cell.try_borrow_mut() {
        Ok(mut val) => *val += 1,
        Err(_) => print("Couldn't get mutable access, already borrowed"),
    }
}
```

### Common Pattern: Rc + RefCell

Since `Rc` gives you shared ownership and `RefCell` gives you interior mutability, they're often combined:

```ferrum
import alloc.rc.Rc
import core.cell.RefCell

type SharedVec[T] {
    data: Rc[RefCell[Vec[T]]],
}

impl[T: Clone] SharedVec[T] {
    fn new(): Self {
        SharedVec { data: Rc.new(RefCell.new(Vec.new())) }
    }

    fn push(&self, value: T) {
        self.data.borrow_mut().push(value);
    }

    fn get(&self, index: usize): Option[T] {
        self.data.borrow().get(index).cloned()
    }

    fn share(&self): Self {
        SharedVec { data: self.data.clone() }
    }
}
```

### `Mutex[T]`: Thread-Safe Interior Mutability

`RefCell` is not thread-safe. For multi-threaded code, use `Mutex`:

```ferrum
import sync.{Arc, Mutex}

fn concurrent_counter() {
    let counter = Arc.new(Mutex.new(0));

    scope s {
        for _ in 0..10 {
            let c = counter.clone();
            s.spawn(|| {
                let mut guard = c.lock();  // Blocks until lock acquired
                *guard += 1;
            });  // Lock released when guard dropped
        }
    }

    print("Final count: {}", *counter.lock());  // prints: 10
}
```

Key differences from RefCell:
- `lock()` blocks (waits) instead of panicking
- Thread-safe
- Slightly more overhead

### `RwLock[T]`: Many Readers, One Writer

If you have mostly reads and few writes, `RwLock` is more efficient than `Mutex`:

```ferrum
import sync.{Arc, RwLock}

fn mostly_reads() {
    let data = Arc.new(RwLock.new(vec![1, 2, 3, 4, 5]));

    scope s {
        // Many readers can proceed simultaneously
        for i in 0..10 {
            let d = data.clone();
            s.spawn(|| {
                let guard = d.read();  // Shared lock - multiple allowed
                print("Reader {}: first = {}", i, guard[0]);
            });
        }

        // Writer waits for all readers to finish
        let d = data.clone();
        s.spawn(|| {
            let mut guard = d.write();  // Exclusive lock - waits for readers
            guard.push(6);
        });
    }
}
```

---

## Common Mistakes and How to Fix Them

### Mistake 1: Using Rc Across Threads

**The problem:**
```ferrum
let shared = Rc.new(data);
s.spawn(|| use(shared.clone()));  // Won't compile
```

**The error:**
```
error[E0277]: `Rc[Data]` cannot be sent between threads safely
```

**The fix:** Use `Arc` instead of `Rc`:
```ferrum
let shared = Arc.new(data);
s.spawn(|| use(shared.clone()));  // Works
```

### Mistake 2: Creating Reference Cycles

**The problem:**
```ferrum
type Node {
    next: Option[Rc[Node]],
    prev: Option[Rc[Node]],  // Creates cycle: A -> B -> A
}
```

**What happens:** Memory leaks. Nodes never freed.

**The fix:** Use `Weak` for back-references:
```ferrum
type Node {
    next: Option[Rc[Node]],   // Strong: I own my successor
    prev: Option[Weak[Node]], // Weak: I know my predecessor, but don't keep it alive
}
```

### Mistake 3: Holding RefCell Borrows Too Long

**The problem:**
```ferrum
fn bad() {
    let cell = RefCell.new(vec![1, 2, 3]);

    let borrowed = cell.borrow();  // Borrowed here...

    // ... lots of code ...

    cell.borrow_mut().push(4);  // PANIC! `borrowed` still alive

    drop(borrowed);  // Too late
}
```

**The fix:** Limit borrow scope:
```ferrum
fn good() {
    let cell = RefCell.new(vec![1, 2, 3]);

    {
        let borrowed = cell.borrow();
        print("{:?}", *borrowed);
    }   // borrowed dropped here

    cell.borrow_mut().push(4);  // Works now
}
```

Or use block expressions:
```ferrum
fn also_good() {
    let cell = RefCell.new(vec![1, 2, 3]);

    let first = { cell.borrow()[0] };  // Borrow ends at }

    cell.borrow_mut().push(4);  // Works
}
```

### Mistake 4: Forgetting to Clone Rc/Arc

**The problem:**
```ferrum
let shared = Rc.new(data);
let handler1 = Handler { config: shared };      // Moves shared
let handler2 = Handler { config: shared };      // ERROR: shared already moved
```

**The error:**
```
error[E0382]: use of moved value: `shared`
```

**The fix:** Clone before moving:
```ferrum
let shared = Rc.new(data);
let handler1 = Handler { config: shared.clone() };  // Clone, count = 2
let handler2 = Handler { config: shared.clone() };  // Clone, count = 3
// original `shared` still valid, count = 3
```

### Mistake 5: Deadlocks with Multiple Locks

**The problem:**
```ferrum
// Thread 1:
let a = lock_a.lock();
let b = lock_b.lock();  // Waits for lock_b

// Thread 2:
let b = lock_b.lock();
let a = lock_a.lock();  // Waits for lock_a

// Both threads wait forever!
```

**The fix:** Always lock in the same order:
```ferrum
// Both threads:
let a = lock_a.lock();  // Always lock_a first
let b = lock_b.lock();  // Then lock_b
```

### Mistake 6: Upgrading a Weak That's Gone

**The problem:**
```ferrum
fn access_parent(weak: &Weak[Parent]) {
    let parent = weak.upgrade().unwrap();  // Panics if parent is gone!
}
```

**The fix:** Handle the `None` case:
```ferrum
fn access_parent(weak: &Weak[Parent]) {
    match weak.upgrade() {
        Some(parent) => {
            // Use parent
        }
        None => {
            // Handle the case where parent was already dropped
            print("Parent no longer exists");
        }
    }
}
```

---

## Practical Patterns: "Here's What You Do When..."

### "I need to allocate large data"

```ferrum
// Put it on the heap with Box
let big_data = Box.new([0u8; 10_000_000]);
```

### "I need a recursive data structure"

```ferrum
// Use Box for indirection
type Tree[T] {
    value: T,
    children: Vec[Box[Tree[T]]],
}
```

### "Multiple parts of my code need the same data"

```ferrum
// Use Rc (single-threaded) or Arc (multi-threaded)
let shared = Rc.new(expensive_computation());
let copy1 = shared.clone();
let copy2 = shared.clone();
```

### "I have a parent/child relationship"

```ferrum
// Parents own children (Rc), children reference parents (Weak)
type Parent { children: Vec[Rc[Child]] }
type Child { parent: Weak[Parent] }
```

### "I need to modify shared data (single thread)"

```ferrum
// Use Rc<RefCell<T>>
let shared = Rc.new(RefCell.new(data));
shared.borrow_mut().modify();
```

### "I need to modify shared data (multiple threads)"

```ferrum
// Use Arc<Mutex<T>>
let shared = Arc.new(Mutex.new(data));
shared.lock().modify();
```

### "I have mostly reads, few writes (multiple threads)"

```ferrum
// Use Arc<RwLock<T>>
let shared = Arc.new(RwLock.new(data));
shared.read();   // Many simultaneous readers OK
shared.write();  // Exclusive access for writing
```

---

## Quick Reference

| Type | What It Does | Thread-Safe | When to Use |
|------|--------------|-------------|-------------|
| `Box[T]` | Single owner, heap allocated | Yes | Large data, recursive types, trait objects |
| `Rc[T]` | Counts how many things point to it | **No** | Shared ownership, single thread |
| `Arc[T]` | Same as Rc but atomic | **Yes** | Shared ownership across threads |
| `Weak[T]` | Doesn't keep value alive | Same as Rc/Arc | Breaking cycles, optional references |
| `RefCell[T]` | Mutate through shared reference | **No** | Interior mutability, single thread |
| `Mutex[T]` | Lock for exclusive access | **Yes** | Interior mutability across threads |
| `RwLock[T]` | Many readers OR one writer | **Yes** | Read-heavy shared data |

---

## The Big Picture

Smart pointers in Ferrum aren't magic. They're the same patterns you'd implement manually in C:

- **Box** = malloc + guaranteed free at scope exit
- **Rc/Arc** = reference counting (like COM's AddRef/Release, or Python's refcount)
- **RefCell/Mutex** = runtime checked borrows / locks

The difference is that the compiler:

1. **Inserts the cleanup automatically** — no forgotten `free()` or `release()`
2. **Checks your work at compile time** — use-after-free is a compile error, not a crash
3. **Enforces thread safety** — data races are impossible because unsafe code won't compile
4. **Makes cleanup deterministic** — values are freed immediately when the last reference goes away, not "eventually" by a GC

You're not giving up control. You're getting the compiler to verify that your manual memory management is correct — and getting it to write the boring parts for you.
