# Ferrum Smart Pointers: A Practical Introduction

**Target audience:** Software engineers familiar with C's manual memory management and Python's garbage collection. No type theory background required.

---

## The Problem

You know both extremes:

**C gives you full control:**
```c
char *buf = malloc(1024);
// ... use buf ...
free(buf);
// Oops, forgot to free on the error path
// Oops, freed twice
// Oops, used after free
```

Manual memory management is fast and predictable, but error-prone. You spend brain cycles tracking ownership instead of solving problems.

**Python gives you no control:**
```python
buf = bytearray(1024)
# ... use buf ...
# GC frees it... eventually
# Maybe during your latency-sensitive operation
# Maybe never if there's a cycle
```

Garbage collection eliminates memory bugs but takes away determinism. You can't predict when memory is freed, and you can't force it to happen *now*.

**What's in between?**

Ferrum's answer: smart pointers with compile-time ownership tracking. The compiler proves your memory management is correct. You get the control of C with the safety of Python — and better performance than both in many cases.

---

## `Box[T]`: Single Owner, Heap Allocated

`Box[T]` is the simplest smart pointer. It allocates a value on the heap and owns it exclusively. When the `Box` goes out of scope, the value is dropped and memory is freed. No manual `free()` required.

```ferrum
fn process_data() {
    let data = Box.new([0u8; 1_000_000])  // 1MB on the heap

    use_data(&data)

}   // data dropped here, memory freed automatically
```

Think of `Box` as `malloc` with an automatic `free` at scope exit — except the compiler guarantees you can't use it after free, because the value is moved or dropped.

### When to Use Box

**Recursive types.** Structs can't contain themselves directly (infinite size), but they can contain a `Box` to themselves:

```ferrum
type LinkedList[T] {
    value: T,
    next: Option[Box[LinkedList[T]]],  // Box provides indirection
}

fn build_list(): LinkedList[i32] {
    LinkedList {
        value: 1,
        next: Some(Box.new(LinkedList {
            value: 2,
            next: None,
        })),
    }
}
```

Without `Box`, the compiler would reject this — `LinkedList` would have infinite size.

**Large data you don't want on the stack.** Stack space is limited (typically 1-8 MB). Large arrays belong on the heap:

```ferrum
// Bad: might overflow the stack
fn bad(): [u8; 10_000_000] {
    [0; 10_000_000]
}

// Good: heap allocated
fn good(): Box[[u8; 10_000_000]] {
    Box.new([0; 10_000_000])
}
```

**Trait objects.** When you need a value of "some type that implements this trait" but don't know the concrete type at compile time:

```ferrum
trait Drawable {
    fn draw(&self)
}

type Circle { radius: f32 }
type Square { side: f32 }

impl Drawable for Circle { fn draw(&self) { /* ... */ } }
impl Drawable for Square { fn draw(&self) { /* ... */ } }

fn make_shape(use_circle: bool): Box[dyn Drawable] {
    if use_circle {
        Box.new(Circle { radius: 5.0 })
    } else {
        Box.new(Square { side: 10.0 })
    }
}
```

The `Box[dyn Drawable]` can hold any `Drawable` — the actual type is determined at runtime.

---

## `Rc[T]`: Reference Counting for Shared Ownership

Sometimes one owner isn't enough. Multiple parts of your program need access to the same data, and you can't determine at compile time which one will be done with it last.

`Rc[T]` (reference-counted pointer) solves this. Multiple `Rc`s can point to the same value. Each clone increments a counter. When an `Rc` is dropped, the counter decrements. When the counter hits zero, the value is dropped.

```ferrum
import alloc.rc.Rc

fn shared_config() {
    let config = Rc.new(Config.load())

    let handler1 = Handler.new(config.clone())  // count: 2
    let handler2 = Handler.new(config.clone())  // count: 3

    drop(handler1)  // count: 2
    drop(handler2)  // count: 1

}   // count: 0, Config dropped here
```

### When to Use Rc

**Shared ownership in trees or graphs.** A tree node might have multiple parents, or a graph node multiple incoming edges:

```ferrum
import alloc.rc.{Rc, Weak}

type TreeNode {
    value: i32,
    children: Vec[Rc[TreeNode]],
}

fn build_tree(): Rc[TreeNode] {
    let leaf1 = Rc.new(TreeNode { value: 1, children: Vec.new() })
    let leaf2 = Rc.new(TreeNode { value: 2, children: Vec.new() })

    // Both leaves shared by the root
    Rc.new(TreeNode {
        value: 0,
        children: vec![leaf1.clone(), leaf2.clone()],
    })
}
```

**Caches and interned data.** When multiple parts of your program should share the same instance:

```ferrum
type StringInterner {
    strings: HashMap[String, Rc[String]],
}

impl StringInterner {
    fn intern(&mut self, s: &str): Rc[String] {
        if let Some(existing) = self.strings.get(s) {
            existing.clone()  // share existing allocation
        } else {
            let new = Rc.new(String.from(s))
            self.strings.insert(s.to_owned(), new.clone())
            new
        }
    }
}
```

**When you genuinely can't determine a single owner.** Sometimes the ownership structure is dynamic — determined by runtime conditions rather than code structure. `Rc` handles this gracefully.

### The Single-Threaded Limitation

`Rc` is **not thread-safe**. The reference count is a plain integer, not atomic. If you try to send an `Rc` to another thread, the compiler stops you:

```ferrum
import alloc.rc.Rc

fn broken() {
    let shared = Rc.new(42)

    scope s {
        s.spawn(|| {
            print(*shared)  // ERROR: Rc[i32] is not Send
        })
    }
}
```

The error message is clear:

```
error: `Rc[i32]` cannot be sent between threads safely
  --> src/main.rs:6:17
   |
6  |         s.spawn(|| {
   |                 ^^ Rc[i32] is not Send
   |
   = help: the trait `Send` is not implemented for `Rc[i32]`
   = note: use `Arc[i32]` for thread-safe reference counting
```

This isn't a runtime check that might fail. The compiler won't let you compile this code. Data races are impossible.

---

## `Arc[T]`: Atomic Reference Counting for Threads

`Arc[T]` is `Rc[T]`'s thread-safe sibling. The only difference: `Arc` uses atomic operations for the reference count, so it's safe to share across threads.

```ferrum
import sync.Arc

fn thread_safe_config() {
    let config = Arc.new(Config.load())

    scope s {
        for i in 0..4 {
            let cfg = config.clone()  // atomic increment
            s.spawn(|| {
                process_with_config(&cfg)
            })  // atomic decrement when task ends
        }
    }   // all tasks joined, config dropped
}
```

### When to Use Arc

**Shared data across threads.** Any time multiple threads need read access to the same data:

```ferrum
import sync.{Arc, Mutex}

type ThreadSafeCache {
    data: Arc[Mutex[HashMap[String, String]]],
}

impl ThreadSafeCache {
    fn new(): Self {
        ThreadSafeCache {
            data: Arc.new(Mutex.new(HashMap.new())),
        }
    }

    fn get(&self, key: &str): Option[String] {
        let guard = self.data.lock()
        guard.get(key).cloned()
    }

    fn set(&self, key: String, value: String) {
        let mut guard = self.data.lock()
        guard.insert(key, value)
    }
}

fn use_cache() {
    let cache = ThreadSafeCache.new()

    scope s {
        // Multiple threads can hold Arc clones
        let c1 = cache.data.clone()
        let c2 = cache.data.clone()

        s.spawn(|| { /* use c1 */ })
        s.spawn(|| { /* use c2 */ })
    }
}
```

### Arc vs Rc: The Cost

`Arc` is slightly slower than `Rc` because atomic operations have overhead. On x86-64, an atomic increment is about 10-20x slower than a regular increment. This sounds scary but usually doesn't matter — you're not incrementing reference counts in a tight loop.

Rule of thumb:
- Single-threaded code: use `Rc`
- Multi-threaded code: use `Arc`
- Don't optimize prematurely; profile first

---

## `Weak[T]`: Breaking Cycles

Reference counting has a weakness: cycles. If A points to B and B points to A, both reference counts stay at 1 forever. Memory leaks.

```ferrum
// DON'T DO THIS - creates a cycle
type Parent {
    children: Vec[Rc[Child]],
}

type Child {
    parent: Rc[Parent],  // Cycle! Parent -> Child -> Parent
}
```

`Weak[T]` solves this. A `Weak` reference doesn't keep the value alive — it's an observer, not an owner. When all strong references (`Rc` or `Arc`) are gone, the value is dropped, even if `Weak` references remain.

```ferrum
import alloc.rc.{Rc, Weak}

type Parent {
    children: Vec[Rc[Child]],
}

type Child {
    parent: Weak[Parent],  // Weak! No cycle.
}

fn build_family(): Rc[Parent] {
    let parent = Rc.new(Parent { children: Vec.new() })

    let child = Rc.new(Child {
        parent: Rc.downgrade(&parent),  // create Weak from Rc
    })

    Rc.get_mut(&mut parent).unwrap().children.push(child)

    parent
}   // When parent is dropped, children are dropped, no cycle
```

### Using Weak References

A `Weak` might point to a dropped value. You must check before using:

```ferrum
fn access_parent(child: &Child) {
    match child.parent.upgrade() {
        Some(parent) => {
            // parent is a valid Rc[Parent]
            print("Parent exists")
        }
        None => {
            // Parent was already dropped
            print("Parent is gone")
        }
    }
}
```

`upgrade()` returns `Option[Rc[T]]` (or `Option[Arc[T]]` for `sync.Weak`). If the value still exists, you get a strong reference. If not, you get `None`.

---

## Comparing to C: Automating What You'd Do Manually

In C, you'd implement reference counting yourself:

```c
struct RefCounted {
    int ref_count;
    void *data;
};

void retain(struct RefCounted *rc) {
    rc->ref_count++;
}

void release(struct RefCounted *rc) {
    if (--rc->ref_count == 0) {
        free(rc->data);
        free(rc);
    }
}
```

This is error-prone:
- Forget to call `retain`? Use-after-free.
- Forget to call `release`? Memory leak.
- Call `release` twice? Double-free.
- Use the wrong atomic operations? Data race.

Ferrum's smart pointers do the same thing, but the compiler:
1. Inserts the `retain`/`release` calls automatically
2. Proves you can't use a value after it's freed
3. Refuses to compile code that would create a data race

You get the same performance (or better — no virtual dispatch overhead) with none of the bugs.

---

## Comparing to Python: Deterministic Instead of "Eventually"

Python's garbage collector frees objects "eventually":

```python
def process():
    huge_data = load_huge_dataset()  # 10 GB
    result = analyze(huge_data)
    # huge_data is "no longer needed" but...

    # GC might not run here
    other_work()  # might OOM because huge_data is still in memory

    return result
```

You can call `gc.collect()`, but it's not guaranteed to free your object if there's a reference cycle.

Ferrum is deterministic:

```ferrum
fn process(): Result {
    let huge_data = load_huge_dataset()?  // 10 GB
    let result = analyze(&huge_data)?

    drop(huge_data)  // freed NOW, guaranteed

    other_work()?    // 10 GB already freed

    Ok(result)
}
```

When a value goes out of scope or is explicitly `drop()`ed, it's freed immediately. No GC pause, no "maybe later," no hoping the reference count reaches zero.

---

## Interior Mutability: When You Need to Mutate Through Shared References

Ferrum's borrowing rules say: either many shared references OR one mutable reference, never both. But sometimes you need to mutate data that's shared.

Enter interior mutability — types that let you mutate through a shared reference, with the safety checks moved from compile time to runtime.

### `RefCell[T]`: Runtime Borrow Checking

`RefCell` enforces borrowing rules at runtime:

```ferrum
import core.cell.RefCell

type Counter {
    value: RefCell[i32],
}

impl Counter {
    fn new(): Self {
        Counter { value: RefCell.new(0) }
    }

    fn increment(&self) {  // Note: &self, not &mut self
        let mut val = self.value.borrow_mut()
        *val += 1
    }

    fn get(&self): i32 {
        *self.value.borrow()
    }
}
```

With `RefCell`, the compiler lets you call `borrow_mut()` through a shared reference. But if you violate the borrowing rules at runtime — say, calling `borrow_mut()` while a `borrow()` is active — the program panics:

```ferrum
fn broken() {
    let cell = RefCell.new(42)

    let borrowed = cell.borrow()      // shared borrow
    let mutable = cell.borrow_mut()   // PANIC: already borrowed
}
```

Use `try_borrow()` and `try_borrow_mut()` if you want to handle conflicts gracefully.

### `Mutex[T]`: Thread-Safe Interior Mutability

`RefCell` is single-threaded. For multi-threaded interior mutability, use `Mutex`:

```ferrum
import sync.{Arc, Mutex}

fn shared_counter() {
    let counter = Arc.new(Mutex.new(0))

    scope s {
        for _ in 0..10 {
            let c = counter.clone()
            s.spawn(|| {
                let mut guard = c.lock()  // blocks until lock acquired
                *guard += 1
            })
        }
    }

    print("Final count: {}", *counter.lock())
}
```

`Mutex.lock()` returns a guard that:
1. Holds the lock until dropped
2. Dereferences to the inner value
3. Automatically unlocks when it goes out of scope

No manual unlock, no forgetting to release.

### `RwLock[T]`: Multiple Readers or One Writer

`RwLock` allows either multiple readers OR one writer:

```ferrum
import sync.{Arc, RwLock}

fn concurrent_read_write() {
    let data = Arc.new(RwLock.new(vec![1, 2, 3]))

    scope s {
        // Many readers can proceed simultaneously
        for i in 0..10 {
            let d = data.clone()
            s.spawn(|| {
                let guard = d.read()  // shared lock
                print("Reader {}: {:?}", i, *guard)
            })
        }

        // Writer blocks until all readers are done
        let d = data.clone()
        s.spawn(|| {
            let mut guard = d.write()  // exclusive lock
            guard.push(4)
        })
    }
}
```

---

## Practical Example: A Thread-Safe LRU Cache

Let's put it together. Here's a thread-safe LRU cache using `Arc`, `Mutex`, and smart interior design:

```ferrum
import sync.{Arc, Mutex}
import collections.LinkedHashMap

type LruCache[K: Eq + Hash + Clone, V: Clone] {
    inner: Arc[Mutex[LruInner[K, V]]],
    capacity: usize,
}

type LruInner[K, V] {
    map: LinkedHashMap[K, V],
}

impl[K: Eq + Hash + Clone, V: Clone] LruCache[K, V] {
    fn new(capacity: usize): Self {
        LruCache {
            inner: Arc.new(Mutex.new(LruInner {
                map: LinkedHashMap.new(),
            })),
            capacity,
        }
    }

    fn get(&self, key: &K): Option[V] {
        let mut inner = self.inner.lock()

        // Move to end (most recently used)
        if let Some(value) = inner.map.remove(key) {
            inner.map.insert(key.clone(), value.clone())
            Some(value)
        } else {
            None
        }
    }

    fn put(&self, key: K, value: V) {
        let mut inner = self.inner.lock()

        // Remove if exists (to update position)
        inner.map.remove(&key)

        // Evict oldest if at capacity
        while inner.map.len() >= self.capacity {
            if let Some((oldest_key, _)) = inner.map.iter().next() {
                let k = oldest_key.clone()
                inner.map.remove(&k)
            }
        }

        inner.map.insert(key, value)
    }

    // Clone the Arc to share with another thread
    fn share(&self): Self {
        LruCache {
            inner: self.inner.clone(),
            capacity: self.capacity,
        }
    }
}

// Usage across threads
fn main() {
    let cache = LruCache.new(100)

    scope s {
        let c1 = cache.share()
        let c2 = cache.share()

        s.spawn(|| {
            c1.put("key1".to_string(), "value1".to_string())
        })

        s.spawn(|| {
            if let Some(v) = c2.get(&"key1".to_string()) {
                print("Got: {}", v)
            }
        })
    }
}
```

---

## Quick Reference

| Type | Ownership | Thread-Safe | Use Case |
|------|-----------|-------------|----------|
| `Box[T]` | Single owner | Yes (if T is) | Heap allocation, recursive types, trait objects |
| `Rc[T]` | Shared (counted) | No | Shared ownership, single-threaded |
| `Arc[T]` | Shared (atomic counted) | Yes | Shared ownership, multi-threaded |
| `Weak[T]` | Non-owning | Matches Rc/Arc | Breaking cycles, optional references |
| `RefCell[T]` | N/A (interior mut) | No | Mutate through &self, single-threaded |
| `Mutex[T]` | N/A (interior mut) | Yes | Mutate through &self, multi-threaded |
| `RwLock[T]` | N/A (interior mut) | Yes | Many readers OR one writer |

---

## The Key Insight

Smart pointers aren't magic. They're the same patterns you'd implement manually in C — single ownership, reference counting, locking. The difference is:

1. **The compiler inserts the bookkeeping.** No forgotten `free()` or `release()`.
2. **The type system prevents misuse.** No use-after-free, no data races.
3. **Cleanup is deterministic.** No GC pause, no "eventually."

You're not giving up control. You're getting the compiler to check your work.
