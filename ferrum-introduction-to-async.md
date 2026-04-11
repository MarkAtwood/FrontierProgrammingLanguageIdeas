# Introduction to Async in Ferrum

**Audience:** Programmers who know C and Python, new to Ferrum's concurrency model

---

## The Problem Async Solves

You want to handle 10,000 network connections. How do you do it?

### The C Way: Threads + Manual Everything

```c
// One thread per connection — doesn't scale
void* handle_client(void* arg) {
    int fd = *(int*)arg;
    char buf[1024];
    while (1) {
        int n = read(fd, buf, sizeof(buf));  // blocks the thread
        if (n <= 0) break;
        write(fd, buf, n);
    }
    close(fd);
    return NULL;
}

int main() {
    // ...
    while (1) {
        int client = accept(server, NULL, NULL);
        pthread_t t;
        pthread_create(&t, NULL, handle_client, &client);
        pthread_detach(t);  // leak if we forget
    }
}
```

Problems with this approach:
1. **Memory:** Each thread needs a stack (usually 1-8 MB). 10,000 threads = 10-80 GB of stack space.
2. **Scheduling:** The OS scheduler wasn't designed for 10,000 threads. Context switches become expensive.
3. **Error handling:** What if `pthread_create` fails? What happens to in-flight connections?
4. **Cleanup:** Detached threads are fire-and-forget. You can't wait for them to finish gracefully.

The "solution" in C is to use `select`, `poll`, or `epoll`:

```c
// Event loop — manual, error-prone
int epoll_fd = epoll_create1(0);
struct epoll_event ev, events[MAX_EVENTS];

// Add server socket
ev.events = EPOLLIN;
ev.data.fd = server;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server, &ev);

while (1) {
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == server) {
            // Accept new connection, add to epoll...
        } else {
            // Handle client, manage state machine manually...
        }
    }
}
```

Now you can handle 10,000 connections with one thread. But you've traded one problem for another:
- **State machines everywhere:** You can't block, so every operation becomes a state machine.
- **Callback spaghetti:** Logic that was sequential is now scattered across event handlers.
- **Manual resource tracking:** You must remember to remove fds from epoll, close them, free buffers.
- **Platform-specific:** `epoll` is Linux. macOS uses `kqueue`. Windows uses `IOCP`. Each has different semantics.

### The Python Way: Colored Functions

Python's `asyncio` makes async code look sequential again:

```python
async def handle_client(reader, writer):
    while True:
        data = await reader.read(1024)  # doesn't block the thread
        if not data:
            break
        writer.write(data)
        await writer.drain()
    writer.close()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8080)
    await server.serve_forever()

asyncio.run(main())
```

This is much better. But there's a catch: **the color problem.**

```python
def compute():
    # This is a "sync" function
    return expensive_calculation()

async def fetch():
    # This is an "async" function
    return await http_client.get(url)

async def process():
    result1 = compute()      # OK: sync from async
    result2 = await fetch()  # OK: async from async
    return result1 + result2

def main():
    result = compute()       # OK: sync from sync
    result = await fetch()   # ERROR: can't await in sync function
    result = asyncio.run(fetch())  # Workaround: but now you're in a new event loop
```

Functions are "colored" — sync or async — and the colors don't mix easily:
- You **can** call sync from async (just call it).
- You **can't** call async from sync without ceremony (`asyncio.run()`, `loop.run_until_complete()`).
- Libraries split into sync and async versions (`requests` vs `aiohttp`).
- If you call a sync function that blocks (like `time.sleep()`), you block the entire event loop.

---

## Ferrum's Async Model

Ferrum solves these problems with three ideas:

1. **Futures:** Async operations return values you can wait on.
2. **Structured concurrency:** Tasks are scoped — they can't outlive their parent.
3. **Effects, not colors:** Async is tracked in the type system like any other effect.

### Futures and Await

A `Future` is a value that will be available later. When you `await` it, you get the value (or suspend until it's ready).

```ferrum
async fn fetch(url: &str): Result[Response, HttpError] ! Net + Async {
    let response = http.get(url).await?
    Ok(response)
}
```

The `.await` doesn't block a thread — it suspends this task and lets other tasks run.

Compare to Python:
- Python: `async def` creates a coroutine, `await` suspends.
- Ferrum: Same, but the `! Async` effect makes it explicit in the type.

Compare to C:
- C: You'd use `epoll` + callbacks + state machines.
- Ferrum: Same performance, sequential code style.

### The Runtime

The runtime manages the event loop and thread pool. It's a capability, not a global:

```ferrum
fn main() ! IO + Async {
    let runtime = Runtime.new()?

    runtime.block_on(async {
        let response = fetch("https://example.com").await?
        println!("{}", response.text().await?)
        Ok(())
    })
}
```

`block_on` runs an async computation to completion, blocking the current thread until done. Inside the async block, `.await` suspends the task, not the thread.

### Spawning Tasks

Use `spawn` to run tasks concurrently:

```ferrum
scope s {
    let handle1 = s.spawn(fetch("https://api.example.com/users"))
    let handle2 = s.spawn(fetch("https://api.example.com/posts"))

    // Both requests run concurrently
    let users = handle1.await?
    let posts = handle2.await?

    process(users, posts)
}
```

Compare to C pthreads:

| C (pthreads) | Ferrum |
|--------------|--------|
| `pthread_create(&t, NULL, func, arg)` | `s.spawn(func())` |
| `pthread_join(t, &result)` | `handle.await` |
| `pthread_detach(t)` | Not possible — tasks are always scoped |
| Stack: 1-8 MB per thread | Stack: kilobytes per task |
| OS scheduling | User-space scheduling |

Compare to Python asyncio:

| Python asyncio | Ferrum |
|----------------|--------|
| `asyncio.create_task(coro)` | `s.spawn(future)` |
| `await task` | `handle.await` |
| Tasks can outlive their creator | Tasks scoped to `scope` block |
| `async def` vs `def` | Same `fn`, different effects |

---

## Structured Concurrency

The most important difference from pthreads and asyncio: **tasks can't outlive their scope.**

```ferrum
scope s {
    s.spawn(task1())
    s.spawn(task2())
    s.spawn(task3())
}
// <- All three tasks have finished here. Guaranteed.
//    Even if one panics. Even if we return early.
```

This is called **structured concurrency**. The scope owns its tasks. When the scope exits — normally, early return, or panic — it waits for all tasks to complete.

### Why This Matters

**C problem:** Detached threads are orphaned. If `main()` exits, they're killed mid-operation. If they hold resources, those leak.

```c
pthread_detach(t);  // "I don't care when this finishes"
// If main() exits, t is killed without cleanup
```

**Python problem:** Tasks can escape their creator:

```python
async def start_work():
    task = asyncio.create_task(background_job())
    return  # We return, but task keeps running

# Later: where did that task go? Is it still running?
# If it fails, who handles the exception?
```

**Ferrum solution:** The scope block enforces a parent-child relationship:

```ferrum
fn start_work() ! Async {
    scope s {
        s.spawn(background_job())
    }
    // background_job() is done here
}
```

You can't "fire and forget" accidentally. If you want a task to outlive a function, you must pass it up explicitly:

```ferrum
fn create_worker(): Task[Result[(), Error>] ! Async {
    // Return the task handle — caller becomes responsible
    // But even this task is bounded by *their* scope
}
```

### Cancellation

When a scope is cancelled (via early return or panic), its tasks are cancelled:

```ferrum
fn download_first(urls: &[String]): Result[Response, Error] ! Net + Async {
    scope s {
        for url in urls {
            s.spawn(fetch(url))
        }

        // Wait for first success
        for handle in s.handles() {
            if let Ok(response) = handle.await {
                return Ok(response)  // <- Early return
            }
        }

        Err(AllFailed)
    }
    // All other in-flight requests are cancelled
}
```

Compare to C: You'd need to track every in-flight request, send cancellation signals, wait for acknowledgment. Miss one and you have a resource leak.

---

## No Colored Functions

Python's color problem stems from `async def` being syntactically different from `def`. You can't call `async def` without `await`, and you can't `await` in `def`.

Ferrum doesn't have colored functions. It has **effects.**

```ferrum
// Blocking IO
fn read_file(path: &str): Result[String, IoError] ! IO {
    fs.read_to_string(path)
}

// Async IO
fn read_file_async(path: &str): Result[String, IoError] ! IO + Async {
    fs.read_to_string_async(path).await
}

// Both are just `fn`. The difference is in the effects.
```

The `! IO` means "performs IO." The `! IO + Async` means "performs IO and may suspend."

### Calling Across Boundaries

You can call a non-async function from an async context (just call it):

```ferrum
async fn process() ! IO + Async {
    let config = read_config()  // synchronous, blocks briefly, that's fine
    let data = fetch_data().await  // async, suspends
    transform(config, data)
}
```

You can call an async function from a sync context using `block_on`:

```ferrum
fn main() ! IO {
    let runtime = Runtime.new()?
    let result = runtime.block_on(async_operation())
    println!("{result}")
}
```

This is explicit — you're choosing to block the current thread until the async operation completes. Unlike Python, there's no magic implicit event loop.

### The IO Traits Work Both Ways

The same `Read` and `Write` traits work for both blocking and async:

```ferrum
trait Read {
    fn read(&mut self, buf: &mut [u8]): Result[usize, IoError]
}

trait AsyncRead {
    fn poll_read(&mut self, cx: &mut Context, buf: &mut [u8]): Poll[Result[usize, IoError]]
}

// TcpStream implements both
impl Read for TcpStream { ... }
impl AsyncRead for TcpStream { ... }
```

You write generic code over `Read` and it works with files, sockets, and buffers — both blocking and async. No library schism.

---

## Channels

Tasks communicate through channels, not shared memory:

```ferrum
// Create a bounded channel
let (tx, rx) = Chan.bounded[Message](capacity: 32)

scope s {
    // Producer
    s.spawn(async {
        for i in 0..100 {
            tx.send(Message { id: i }).await?
        }
    })

    // Consumer
    s.spawn(async {
        while let Some(msg) = rx.recv().await {
            process(msg)
        }
    })
}
```

### Channel Types

```ferrum
Chan.bounded[T](n)   // Blocks sender when full
Chan.unbounded[T]()  // Never blocks sender (can use unbounded memory)
```

### Select

Wait on multiple channels:

```ferrum
let result = select! {
    rx1 -> msg => handle_type1(msg),
    rx2 -> msg => handle_type2(msg),
    tx  <- outgoing => (),  // Send when ready
    timeout!(500ms) => Err(Timeout),
}
```

Compare to C:
- C: You'd use `select()` or `poll()` on file descriptors, converting channel operations into fd operations.
- Ferrum: First-class channel support in `select!`.

Compare to Python:
- Python: `asyncio.wait()` or `asyncio.gather()` for similar patterns.
- Ferrum: More ergonomic syntax, integrated with the type system.

---

## Comparing to C Concurrency

| Concept | C | Ferrum |
|---------|---|--------|
| Thread creation | `pthread_create` | `scope { s.spawn(...) }` |
| Thread join | `pthread_join` | Automatic at scope exit |
| Detached threads | `pthread_detach` | Not possible (by design) |
| Event loop | `epoll`/`kqueue`/`IOCP` (manual) | Runtime (automatic) |
| Async operation | Callback + state machine | `future.await` |
| Channel | Implement yourself | `Chan.bounded()` |
| Select | `select()`/`poll()` | `select!` macro |
| Memory per task | 1-8 MB (thread stack) | Kilobytes (future state) |
| Scheduling | OS kernel | User-space runtime |

### What C Gets Right

- **Control:** You can tune thread affinity, stack size, scheduling policy.
- **Predictability:** No runtime, no GC, no surprises.
- **Interop:** pthreads is universal.

Ferrum preserves these:
- Platform-specific thread control via `sys.posix.pthread` when needed.
- No GC. Predictable memory.
- C FFI for interop.

### What C Gets Wrong

- **Safety:** Data races are undefined behavior. No compiler help.
- **Ergonomics:** Manual state machines for async code.
- **Portability:** `epoll` vs `kqueue` vs `IOCP`.

Ferrum fixes these:
- Data races are compile-time errors (`Send` and `Sync` bounds).
- Sequential async code with `await`.
- Unified `scope` and `select!` across platforms.

---

## Comparing to Python Concurrency

| Concept | Python | Ferrum |
|---------|--------|--------|
| Async function | `async def` | `fn ... ! Async` |
| Await | `await` | `.await` |
| Task spawn | `asyncio.create_task()` | `s.spawn()` |
| Structured concurrency | `asyncio.TaskGroup` (3.11+) | `scope` (always) |
| Sync from async | Just call it (may block event loop!) | Just call it (explicit blocking) |
| Async from sync | `asyncio.run()` | `runtime.block_on()` |
| Colored functions | Yes (`async def` vs `def`) | No (effects, not syntax) |
| Channels | `asyncio.Queue` | `Chan` |
| Select | `asyncio.wait()` | `select!` |

### What Python Gets Right

- **Syntax:** `async`/`await` is readable and familiar.
- **Ecosystem:** Rich async libraries (aiohttp, asyncpg, etc.).
- **Batteries included:** `asyncio` is in the standard library.

Ferrum adopts these:
- Similar `async`/`await` syntax.
- Async traits for ecosystem compatibility.
- Full async runtime in the standard library.

### What Python Gets Wrong

- **The color problem:** Async and sync don't mix.
- **Accidental blocking:** Call `time.sleep()` in async code and you freeze the event loop.
- **Task lifecycle:** Tasks can outlive their creators, leading to hard-to-debug issues.

Ferrum fixes these:
- Effects instead of colors — the type system tracks what suspends.
- Blocking calls are explicit (`block_on`), not accidental.
- Structured concurrency by default — tasks can't escape their scope.

---

## A Complete Example

Here's a concurrent HTTP scraper in Ferrum:

```ferrum
fn scrape_urls(urls: &[String]): Result[Vec[Page], Error] ! Net + Async {
    let (tx, rx) = Chan.bounded[Result[Page, Error]](32)

    scope s {
        // Spawn a worker for each URL
        for url in urls {
            let tx = tx.clone()
            s.spawn(async {
                let result = fetch_and_parse(&url).await
                tx.send(result).await.ok()
            })
        }

        // Drop our sender so rx knows when to stop
        drop(tx)

        // Collect results
        let mut pages = Vec.new()
        while let Some(result) = rx.recv().await {
            pages.push(result?)
        }

        Ok(pages)
    }
}

async fn fetch_and_parse(url: &str): Result[Page, Error] ! Net {
    let response = http.get(url).await?
    let html = response.text().await?
    let page = parse_html(&html)?
    Ok(page)
}
```

Compare this to the equivalent in C (100+ lines of epoll, callbacks, and state machines) or Python (similar length, but with colored function constraints).

---

## Summary

| Problem | C | Python | Ferrum |
|---------|---|--------|--------|
| Scaling to 10k connections | epoll + manual state machines | asyncio | scope + spawn + await |
| Thread safety | Undefined behavior (hope) | GIL (limited parallelism) | Type-checked (Send/Sync) |
| Task lifecycle | Manual (easy to leak) | Unstructured (tasks escape) | Structured (scope-bounded) |
| Colored functions | N/A | Yes (async def vs def) | No (effects) |
| Cancellation | Manual | Tricky | Automatic at scope exit |
| Channel communication | Build it yourself | asyncio.Queue | Chan |

Ferrum's async model gives you:
- **C-level performance:** No GC, predictable memory, user-space scheduling.
- **Python-level ergonomics:** Sequential async code with `await`.
- **Safety guarantees:** Data races caught at compile time, structured concurrency by default.

---

*See also: [Ferrum Language Reference — Concurrency](ferrum-lang-concurrency.md) for complete specification.*
