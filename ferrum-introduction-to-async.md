# Introduction to Async in Ferrum

**Audience:** Programmers who know C and Python, new to Ferrum's concurrency model

---

## The Problem Async Solves

You're building a web server. Or a chat application. Or a scraper. You need to handle many network connections at once — maybe hundreds, maybe thousands. How do you do it?

The fundamental challenge: network I/O is slow. A database query takes milliseconds. A remote API call takes tens or hundreds of milliseconds. If your program waits for each operation to complete before starting the next one, you waste most of your time doing nothing.

Here's the core insight: **while one connection waits for data, you could be handling other connections.** The question is how to structure your code to make that possible.

---

## The C Way: Threads + Manual Everything

The obvious approach: one thread per connection. Each thread handles its own client, blocking when it needs to wait.

```c
void* handle_client(void* arg) {
    int fd = *(int*)arg;
    char buf[1024];
    while (1) {
        int n = read(fd, buf, sizeof(buf));  // This blocks the thread
        if (n <= 0) break;
        write(fd, buf, n);
    }
    close(fd);
    return NULL;
}

int main() {
    int server = setup_server();
    while (1) {
        int client = accept(server, NULL, NULL);
        pthread_t t;
        int* arg = malloc(sizeof(int));  // Who frees this?
        *arg = client;
        pthread_create(&t, NULL, handle_client, arg);
        pthread_detach(t);  // "I'll never join this thread"
    }
}
```

This works for 10 connections. Maybe 100. But it falls apart at scale.

### Problem 1: Memory

Each thread needs its own stack — a block of memory for function calls, local variables, and return addresses. On Linux, the default is 8 MB per thread. On macOS, it's 512 KB to 8 MB depending on configuration.

Do the math:
- 100 threads = 800 MB just for stacks
- 1,000 threads = 8 GB
- 10,000 threads = 80 GB

You'll run out of memory before you run out of connections to serve.

### Problem 2: Context Switching

The operating system's scheduler decides which thread runs. When you have 10,000 threads, the scheduler spends significant time just deciding who gets to run next. Each context switch costs microseconds of CPU time plus cache thrashing.

Your server spends more time managing threads than doing actual work.

### Problem 3: Resource Leaks

Look at that `pthread_detach` call. It means "I'm never going to wait for this thread to finish." The thread runs until it exits, and the OS cleans up.

But what if:
- The thread is holding a file descriptor when the program exits?
- The thread is in the middle of writing to a database?
- The thread allocated memory that never gets freed?

Detached threads are fire-and-forget. You lose visibility into what they're doing. When something goes wrong, you have no way to know which threads are still running or what state they're in.

And that `malloc` for the argument — if `pthread_create` fails, you leak memory. If the thread crashes before freeing it, you leak memory. This is tedious bookkeeping that's easy to get wrong.

### The C "Solution": Event Loops

The alternative in C is to not use threads at all. Instead, you use a single thread that monitors all connections at once using `select`, `poll`, or `epoll`:

```c
int epoll_fd = epoll_create1(0);
struct epoll_event ev, events[MAX_EVENTS];

// Register the server socket
ev.events = EPOLLIN;
ev.data.fd = server;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server, &ev);

while (1) {
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == server) {
            // New connection: accept it, add to epoll
            int client = accept(server, NULL, NULL);
            ev.events = EPOLLIN;
            ev.data.fd = client;
            epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client, &ev);
        } else {
            // Existing connection has data: handle it
            int fd = events[i].data.fd;
            char buf[1024];
            int n = read(fd, buf, sizeof(buf));
            if (n <= 0) {
                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
            } else {
                write(fd, buf, n);
            }
        }
    }
}
```

Now one thread handles 10,000 connections. The `epoll_wait` call blocks until *any* of the monitored file descriptors has activity, then you process whichever ones are ready.

But you've traded one set of problems for another:

**State machines everywhere.** In the threaded version, each connection's state is implicit in where the thread is executing. In the event loop version, you can't "wait" — the loop must keep running. So every multi-step operation becomes an explicit state machine: "I've received 3 bytes of the header, waiting for 5 more."

**Callback spaghetti.** Logic that was sequential (read request, query database, send response) is now scattered across multiple event handler iterations. Following the flow of a single request through your code becomes a nightmare.

**Manual cleanup.** You have to remember to remove file descriptors from epoll before closing them. Miss it and you get undefined behavior. You have to track which connections have which allocated buffers. It's all on you.

**Platform lock-in.** `epoll` is Linux-only. macOS uses `kqueue`. Windows uses `IOCP`. They have different APIs and different semantics. Cross-platform code needs to abstract over all of them.

---

## The Python Way: asyncio

Python's `asyncio` gives you the best of both worlds, kind of. You write sequential-looking code, and the runtime handles the event loop:

```python
async def handle_client(reader, writer):
    while True:
        data = await reader.read(1024)  # Suspends, doesn't block
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

That `await reader.read(1024)` doesn't block the thread. It tells the event loop "I'm waiting for data; go do something else and come back to me when it's ready." The code reads top-to-bottom, but under the hood it's an event loop.

This is much better than C's manual epoll wrangling. But Python has its own problems.

### The GIL Problem

Python has a Global Interpreter Lock (GIL). Only one thread can execute Python bytecode at a time. If you use `threading` to parallelize CPU work, the threads take turns — you don't get true parallelism.

`asyncio` doesn't help here. It's for I/O-bound work (waiting on network, disk, databases). If you have CPU-bound work, you need `multiprocessing`, which has its own overhead and complexity.

Ferrum doesn't have a GIL. Multiple threads can execute Ferrum code simultaneously. Async is for I/O-bound work; threads and parallelism work as you'd expect.

### The Color Problem

This is the killer. In Python, functions are "colored" — they're either sync or async:

```python
def compute():           # Sync function (no async/await)
    return expensive_calculation()

async def fetch():       # Async function (must be awaited)
    return await http_client.get(url)
```

The colors have strict rules:
- **Sync can't call async** (directly). You can't `await` in a regular function.
- **Async can call sync**, but if the sync function blocks (like `time.sleep(10)`), you freeze the entire event loop.

This leads to practical headaches:

```python
def process_data(data):
    # I need to make an HTTP request here...
    # But I can't use await because this is a sync function!

    # Option 1: Rewrite this function and all callers to be async
    # Option 2: Use asyncio.run(), but you can't nest event loops
    # Option 3: Use run_in_executor() to run sync code in a thread pool

    # None of these are good.
    pass

async def async_process_data(data):
    # Now this is async, but...
    # Every function that calls this must also be async
    # And every function that calls those must also be async
    # The color spreads like a virus
    pass
```

Libraries split into sync and async versions. You want to make HTTP requests? There's `requests` (sync) and `aiohttp` (async). They have different APIs. You can't mix them easily. Your dependency list doubles.

The most common bug: accidentally calling a blocking function from async code.

```python
async def fetch_users():
    # This looks harmless...
    response = requests.get('https://api.example.com/users')  # WRONG!
    # requests.get() blocks. The entire event loop freezes.
    # All other tasks stop making progress until this returns.
    # Should be: response = await aiohttp_session.get(url)
```

Nothing warns you. It just silently degrades performance.

---

## Ferrum's Approach

Ferrum solves these problems with three ideas that work together:

1. **Lightweight tasks** instead of threads — thousands of concurrent operations without memory bloat
2. **Structured concurrency** — tasks can't escape their scope, so you always know what's running
3. **Effects instead of colors** — the type system tracks what can suspend, not special syntax

Let's look at each.

---

## Futures and Await

A `Future` represents a value that will be available later. When you call an async function, you get a Future. When you `.await` it, you either get the result immediately (if it's ready) or your task suspends until it is.

```ferrum
async fn fetch(url: &str): Result[Response, HttpError] ! Net + Async {
    let response = http.get(url).await?
    Ok(response)
}
```

That `.await` doesn't block a thread. It tells the runtime "I'm waiting for this HTTP response; run other tasks until it's ready." The thread can go handle hundreds of other tasks while this request is in flight.

Compare to the three approaches:

| Approach | What happens during the wait |
|----------|------------------------------|
| C (pthread) | Thread is blocked. OS manages it. Costs megabytes. |
| C (epoll) | You return to the event loop. Manual state tracking. |
| Python (asyncio) | Task suspends. Event loop runs other tasks. |
| Ferrum | Task suspends. Runtime runs other tasks. Minimal memory per task. |

The key difference from Python: Ferrum's effects tell you exactly what a function does.

```ferrum
fn pure_compute(x: i32): i32 {
    x * x  // No effects. Can't do I/O, can't suspend.
}

fn read_config(): Result[Config, IoError] ! IO {
    fs.read_to_string("config.toml")  // Has IO effect. Might block briefly.
}

async fn fetch_data(): Result[Data, Error] ! Net + Async {
    http.get(url).await  // Has Net and Async effects. Will suspend.
}
```

When you see `! Async` in the type signature, you know this function can suspend. When you don't see it, you know it can't freeze your event loop accidentally.

---

## The Runtime

The runtime is the engine that drives async execution. It's not a hidden global like Python's event loop — you create it explicitly.

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

`block_on` runs an async computation on the current thread, blocking until it completes. Inside the async block, `.await` suspends the task (not the thread). The runtime multiplexes tasks onto threads.

Think of it like this:
- **Thread**: OS-managed, megabytes of memory, expensive to create
- **Task**: Runtime-managed, kilobytes of memory, cheap to create

You might have 4 OS threads running 10,000 tasks. The runtime schedules which task runs on which thread, switching between them when they suspend.

---

## Spawning Tasks

Use `spawn` to run multiple tasks concurrently. Here's a concrete example — fetching multiple URLs at once:

```ferrum
async fn fetch_all_users(): Result[Vec[User], Error] ! Net + Async {
    let urls = [
        "https://api.example.com/users/1",
        "https://api.example.com/users/2",
        "https://api.example.com/users/3",
    ]

    scope s {
        // Spawn a task for each URL — they all run concurrently
        let handles: Vec[_] = urls.iter()
            .map(|url| s.spawn(fetch_user(url)))
            .collect()

        // Wait for all of them and collect results
        let mut users = Vec.new()
        for handle in handles {
            users.push(handle.await?)
        }

        Ok(users)
    }
}

async fn fetch_user(url: &str): Result[User, Error] ! Net {
    let response = http.get(url).await?
    let user: User = response.json().await?
    Ok(user)
}
```

Without concurrency, fetching 3 users at 100ms each takes 300ms. With concurrent spawning, it takes ~100ms (they all run in parallel).

Compare this to C pthreads:

| C (pthreads) | Ferrum |
|--------------|--------|
| `pthread_create(&t, NULL, func, arg)` | `s.spawn(func())` |
| `pthread_join(t, &result)` | `handle.await` |
| `pthread_detach(t)` — dangerous | Not possible by design |
| 1-8 MB memory per thread | Kilobytes per task |
| Must manually track all threads | Scope tracks everything |
| Manual argument passing, freeing | Return values work naturally |

Compare to Python asyncio:

| Python asyncio | Ferrum |
|----------------|--------|
| `asyncio.create_task(coro)` | `s.spawn(future)` |
| `await task` | `handle.await` |
| Tasks can outlive their creator | Tasks scoped to `scope` block |
| `asyncio.gather(*tasks)` | `scope` + spawn + await |
| `async def` vs `def` split | Same `fn`, different effects |

---

## Structured Concurrency

This is the most important concept in Ferrum's async model. It's worth understanding deeply.

**Structured concurrency means: tasks can't outlive the scope that created them.**

```ferrum
scope s {
    s.spawn(task_a())
    s.spawn(task_b())
    s.spawn(task_c())
}
// <- When we reach here, ALL THREE tasks have finished. Guaranteed.
//    Even if task_b panicked.
//    Even if we returned early.
//    Even if someone called cancel.
```

The `scope` block owns its tasks. When control leaves the scope — for any reason — the scope waits for all its tasks to complete. If any are still running, they're cancelled and the scope waits for the cancellation to finish.

### Why Fire-and-Forget Is Dangerous

Here's a real bug that happens in unstructured systems. Imagine you're writing a web service that processes orders:

```python
# Python — the bug
async def process_order(order):
    # Save to database
    await db.insert(order)

    # Send confirmation email in the background
    asyncio.create_task(send_confirmation_email(order))

    # Update inventory in the background
    asyncio.create_task(update_inventory(order))

    # Return success to the client
    return {"status": "ok"}
```

What's wrong? Those `create_task` calls spawn tasks that run independently. The function returns before they finish. The client gets "ok" but:

1. **The email might fail silently.** The task crashes, logs an error somewhere, and nobody notices.

2. **Inventory update might fail.** Now your stock counts are wrong. Customer orders an item that's actually out of stock.

3. **Shutdown is broken.** When you restart the server, those background tasks are killed mid-operation. The email was half-sent? The inventory was partially updated? You don't know.

4. **Debugging is hell.** An exception in a background task doesn't propagate to the caller. It's logged (maybe) and swallowed. You find out from customer complaints, not error monitoring.

5. **Testing is flaky.** Your test finishes and asserts success. But the background tasks are still running. They might fail after your test passes. Or they might interfere with the next test.

This happens constantly in production systems. The fix in Python is `asyncio.TaskGroup` (added in 3.11), but it's optional — you have to remember to use it.

### How Ferrum Prevents This

In Ferrum, you can't accidentally fire-and-forget:

```ferrum
fn process_order(order: Order): Result[Response, Error] ! DB + Net + Async {
    // Save to database
    db.insert(&order).await?

    scope s {
        // These run concurrently with each other
        let email_handle = s.spawn(send_confirmation_email(&order))
        let inventory_handle = s.spawn(update_inventory(&order))

        // Both MUST complete before we return
        email_handle.await?   // If email fails, we know immediately
        inventory_handle.await?  // If inventory fails, we know immediately
    }
    // <- We only reach here if BOTH tasks succeeded

    Ok(Response { status: "ok" })
}
```

There's no way to return while tasks are still running. The scope enforces it. If you want faster response to the client, you structure it explicitly:

```ferrum
fn process_order(order: Order): Result[Response, Error] ! DB + Net + Async {
    // Save to database (must complete before returning)
    db.insert(&order).await?

    // Respond to client immediately
    let response = Response { status: "accepted", order_id: order.id }

    // Background work — but it's still scoped to THIS function
    scope s {
        s.spawn(send_confirmation_email(&order))
        s.spawn(update_inventory(&order))
        // If the server shuts down, these tasks are cancelled cleanly
        // If they fail, we can log/handle it here
    }

    Ok(response)
}
```

Or if you really need work to continue after returning, you make that explicit in the function's interface:

```ferrum
// The caller spawns this and is responsible for it
async fn background_order_processing(order: Order): Result[(), Error] ! DB + Net {
    send_confirmation_email(&order).await?
    update_inventory(&order).await?
    Ok(())
}

// In main or a long-lived scope:
scope main_scope {
    // ... handle request ...
    main_scope.spawn(background_order_processing(order))
    // This task is explicitly owned by main_scope
    // When the server shuts down, main_scope waits for it
}
```

You can't accidentally orphan a task. You have to explicitly choose who owns it.

### Cancellation

When a scope exits early — via `return`, `break`, `?` propagation, or panic — it cancels all its tasks:

```ferrum
async fn download_first(urls: &[String]): Result[Response, Error] ! Net + Async {
    scope s {
        // Start downloading all URLs concurrently
        for url in urls {
            s.spawn(fetch(url))
        }

        // Wait for the first one to succeed
        for handle in s.handles() {
            if let Ok(response) = handle.await {
                return Ok(response)  // Early return!
            }
        }

        Err(Error::AllFailed)
    }
    // <- Early return causes all remaining downloads to be cancelled
}
```

In C, you'd have to track every in-flight request, send them each a cancellation signal, handle partial states, and wait for acknowledgment. Miss one and you leak resources.

In Python, task cancellation exists but is tricky — `task.cancel()` doesn't immediately stop the task, it raises `CancelledError` at the next await point, which the task can catch and ignore.

In Ferrum, cancellation is cooperative but enforced. A cancelled task can run cleanup code, but it can't refuse to stop.

---

## No Colored Functions

Let's revisit Python's color problem and see how Ferrum avoids it.

Python:
```python
def sync_function():       # Can't use await
    pass

async def async_function(): # Must use await to call
    pass

# These are different "colors" — they don't mix easily
```

Ferrum:
```ferrum
fn regular_function(): i32 {
    42
}

fn io_function(): Result[String, IoError] ! IO {
    fs.read_to_string("file.txt")
}

async fn async_function(): Result[Response, Error] ! Net + Async {
    http.get(url).await
}

// All are `fn`. The difference is in the effects (! IO, ! Async, etc.)
```

The `! Effect` syntax declares what side effects a function can have:
- No effects = pure computation, no I/O, no suspension
- `! IO` = can do I/O (might block briefly)
- `! Async` = can suspend (must be awaited or spawned)
- `! Net` = can do network operations
- `! IO + Async` = can do both

### Calling Across Boundaries

Here's how mixing works:

**Non-async calling non-async:** Just call it.

```ferrum
fn compute_total(items: &[Item]): i32 {
    items.iter().map(|i| i.price).sum()
}

fn process() ! IO {
    let items = load_items()?
    let total = compute_total(&items)  // Fine
    println!("Total: {total}")
}
```

**Async calling non-async:** Just call it.

```ferrum
async fn fetch_and_compute() ! Net + Async {
    let response = http.get(url).await?
    let items: Vec[Item] = response.json().await?
    let total = compute_total(&items)  // Fine, it's sync and fast
    Ok(total)
}
```

**Non-async calling async:** Use `block_on` (explicit).

```ferrum
fn main() ! IO {
    let runtime = Runtime.new()?

    // Explicitly block this thread until the async work completes
    let result = runtime.block_on(fetch_and_compute())

    println!("Got: {result:?}")
}
```

The difference from Python: in Ferrum, if you call a slow sync function from async code, the type system tells you. A function that takes 10 seconds of CPU time doesn't have `! Async` — it doesn't suspend, it just runs. You can see that in the signature and decide if you want to spawn it on a blocking thread pool:

```ferrum
async fn process_with_heavy_compute() ! Async + IO {
    let data = fetch_data().await?

    // This is CPU-bound, would block the async runtime
    // So we run it on a separate thread pool
    let result = spawn_blocking(|| heavy_computation(data)).await?

    Ok(result)
}
```

No silent footguns. The type system helps you see what suspends and what blocks.

---

## Channels

Tasks communicate by sending messages through channels, not by sharing memory directly. This avoids data races by design.

```ferrum
let (tx, rx) = Chan.bounded[Message](capacity: 32)

scope s {
    // Producer task
    s.spawn(async {
        for i in 0..100 {
            tx.send(Message { id: i }).await?
        }
        drop(tx)  // Signal we're done sending
    })

    // Consumer task
    s.spawn(async {
        while let Some(msg) = rx.recv().await {
            process_message(msg)
        }
    })
}
```

### A Practical Example: Worker Pool

Here's a pattern for processing work with limited concurrency — a worker pool:

```ferrum
async fn process_urls_with_pool(
    urls: &[String],
    max_workers: usize
): Result[Vec[Page], Error] ! Net + Async {
    let (work_tx, work_rx) = Chan.bounded[String](urls.len())
    let (result_tx, result_rx) = Chan.bounded[Result[Page, Error]](urls.len())

    scope s {
        // Spawn worker tasks (limited concurrency)
        for _ in 0..max_workers {
            let work_rx = work_rx.clone()
            let result_tx = result_tx.clone()
            s.spawn(async {
                while let Some(url) = work_rx.recv().await {
                    let result = fetch_and_parse(&url).await
                    result_tx.send(result).await.ok()
                }
            })
        }

        // Feed work into the queue
        for url in urls {
            work_tx.send(url.clone()).await?
        }
        drop(work_tx)  // Signal no more work
        drop(result_tx)  // Our handle; workers have their copies

        // Collect results
        let mut pages = Vec.new()
        while let Some(result) = result_rx.recv().await {
            pages.push(result?)
        }

        Ok(pages)
    }
}
```

This fetches URLs with at most `max_workers` concurrent requests — useful when you have rate limits or want to avoid overwhelming a server.

### Channel Types

```ferrum
Chan.bounded[T](n)   // Holds up to n items. send() suspends when full.
Chan.unbounded[T]()  // No limit. send() never suspends. Can use unbounded memory.
```

Use bounded channels by default. They provide backpressure — if the consumer is slow, the producer slows down too. Unbounded channels can grow forever if the consumer can't keep up.

### Select: Waiting on Multiple Things

Sometimes you need to wait for whichever happens first — a message from any of several channels, or a timeout:

```ferrum
let result = select! {
    rx1 -> msg => {
        // Received from rx1
        handle_message(msg)
    },
    rx2 -> msg => {
        // Received from rx2
        handle_other(msg)
    },
    timeout!(Duration::from_secs(5)) => {
        // 5 seconds passed with no messages
        Err(Error::Timeout)
    },
}
```

Compare to C's `select()`:
- C: You're working with file descriptors. Channels must be implemented as pipes or sockets. Tedious.
- Ferrum: Channels are first-class. `select!` works directly with them.

Compare to Python's `asyncio.wait()`:
- Python: `done, pending = await asyncio.wait(tasks, return_when=FIRST_COMPLETED)` — returns sets, you fish out results.
- Ferrum: Pattern matching syntax, type-checked, integrates with timeouts.

---

## Practical Example: A TCP Echo Server

Here's a complete example — a server that accepts connections and echoes back whatever clients send:

```ferrum
async fn run_server(addr: &str): Result[(), Error] ! Net + Async {
    let listener = TcpListener.bind(addr).await?
    println!("Listening on {addr}")

    scope s {
        loop {
            let (socket, client_addr) = listener.accept().await?
            println!("Client connected: {client_addr}")

            s.spawn(handle_client(socket, client_addr))
        }
    }
}

async fn handle_client(
    mut socket: TcpStream,
    addr: SocketAddr
): Result[(), Error] ! Net {
    let mut buf = [0u8; 1024]

    loop {
        let n = socket.read(&mut buf).await?
        if n == 0 {
            println!("Client disconnected: {addr}")
            return Ok(())
        }

        socket.write_all(&buf[..n]).await?
    }
}

fn main() ! IO + Async {
    let runtime = Runtime.new()?
    runtime.block_on(run_server("127.0.0.1:8080"))
}
```

Compare the same server in C with epoll — it would be 100+ lines of state management, buffer tracking, and error handling. The Ferrum version reads sequentially, like the threaded version, but scales like the epoll version.

---

## Practical Example: Parallel HTTP Requests with Timeout

A common pattern — make multiple requests, but give up if it takes too long:

```ferrum
async fn fetch_with_timeout(
    urls: &[String],
    timeout_secs: u64
): Result[Vec[Response], Error] ! Net + Async {
    let deadline = Instant.now() + Duration::from_secs(timeout_secs)

    scope s {
        let handles: Vec[_] = urls.iter()
            .map(|url| s.spawn(http.get(url)))
            .collect()

        let mut responses = Vec.new()

        for handle in handles {
            let remaining = deadline.saturating_duration_since(Instant.now())

            let result = select! {
                handle -> response => response,
                timeout!(remaining) => return Err(Error::Timeout),
            }

            responses.push(result?)
        }

        Ok(responses)
    }
}
```

If any request takes too long, we return `Error::Timeout`. All in-flight requests are cancelled automatically when we return early.

---

## Summary: Ferrum vs C vs Python

| Problem | C | Python | Ferrum |
|---------|---|--------|--------|
| **10k connections** | epoll + manual state machines | asyncio | scope + spawn + await |
| **Memory per task** | 1-8 MB (thread stack) | ~2 KB (coroutine) | Kilobytes (task state) |
| **Thread safety** | Undefined behavior if you mess up | GIL prevents true parallelism | Compile-time checked (Send/Sync) |
| **Task lifecycle** | Manual tracking, easy to leak | Fire-and-forget possible | Structured, scope-bounded |
| **Function colors** | N/A | Yes (async def vs def) | No (effects system) |
| **Blocking in async** | You manage it | Silent bug | Type system catches it |
| **Cancellation** | Manual, error-prone | Tricky (CancelledError) | Automatic at scope exit |
| **Cross-platform** | epoll/kqueue/IOCP split | Handled by asyncio | Handled by runtime |

### What You Get with Ferrum

**C-level performance.** No garbage collector. Predictable memory usage. Tasks are cheap. The runtime uses modern OS primitives (epoll, kqueue, IOCP) under the hood.

**Python-level ergonomics.** Sequential code that reads top-to-bottom. No callback spaghetti. No manual state machines.

**Safety guarantees.** Data races caught at compile time. Tasks can't escape their scope. Cancellation is automatic and correct. No silent blocking bugs.

---

## Quick Reference

```ferrum
// Create a runtime
let runtime = Runtime.new()?

// Run async code from sync context
runtime.block_on(async { ... })

// Spawn concurrent tasks
scope s {
    let handle = s.spawn(async_operation())
    let result = handle.await?
}

// Channels
let (tx, rx) = Chan.bounded[T](capacity)
tx.send(value).await?
let value = rx.recv().await

// Select on multiple operations
select! {
    rx -> msg => handle(msg),
    timeout!(duration) => handle_timeout(),
}

// Effects in signatures
fn sync_pure(): T { ... }
fn sync_io(): T ! IO { ... }
async fn async_net(): T ! Net + Async { ... }
```

---

*See also: [Ferrum Language Reference - Concurrency](ferrum-lang-concurrency.md) for the complete specification.*
