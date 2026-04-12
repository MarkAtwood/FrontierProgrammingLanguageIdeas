# Ferrum Introduction: Modules and Packages

**For:** Developers familiar with C and Python

---

## The Problem

If you have written C, you know header files. If you have written Python, you know `import`. Both have problems Ferrum solves.

### C: The Header File Tax

In C, every public function needs two declarations: one in the header, one in the source file. You maintain both. When they diverge, you get linker errors or worse, silent ABI mismatches.

```c
// math_utils.h
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

double vector_length(double x, double y);  // declaration

#endif

// math_utils.c
#include "math_utils.h"

double vector_length(double x, double y) {  // definition (must match!)
    return sqrt(x*x + y*y);
}
```

**Include guards** (`#ifndef MATH_UTILS_H`) are boilerplate you write hundreds of times. Forget them and you get "redefinition" errors:

```c
// Forgot include guards in config.h
// config.h
struct Config {
    int port;
    char* host;
};

// server.c
#include "config.h"
#include "database.h"  // database.h also includes config.h

// Compiler error:
// error: redefinition of 'struct Config'
```

Put them wrong and you get silent ODR (One Definition Rule) violations:

```c
// utils.h - different definitions in different translation units
#ifndef UTILS_H
#define UTILS_H

#ifdef USE_FAST_MODE
struct Buffer { char data[64]; };    // 64 bytes
#else
struct Buffer { char data[256]; };   // 256 bytes
#endif

#endif

// One .c file compiled with -DUSE_FAST_MODE, another without.
// Linker silently links them. Program crashes mysteriously.
```

**Forward declarations** are another tax. To use a type before it is defined, you declare it again:

```c
// parser.h
struct Lexer;  // forward declaration - duplicated knowledge

struct Parser {
    struct Lexer* lexer;  // can only use pointer, not value
};

void parser_init(struct Parser* p, struct Lexer* l);

// Now in lexer.h, you need the same dance if Lexer refers to Parser
```

This gets worse with **circular includes**:

```c
// a.h
#ifndef A_H
#define A_H
#include "b.h"  // needs B

struct A {
    struct B* b_ref;
};

#endif

// b.h
#ifndef B_H
#define B_H
#include "a.h"  // needs A

struct B {
    struct A* a_ref;
};

#endif

// This works only because of include guards + forward declarations.
// But try to use A by value in B, or vice versa, and you're stuck.
```

And there is **no namespacing**. Two libraries with a `parse()` function? Collision. The fix is naming conventions: `json_parse()`, `xml_parse()`, `mycompany_json_parse()`. Your code becomes a monument to the lack of modules.

### Python: Runtime Surprises

Python has clean import syntax. But it is checked at runtime:

```python
from utils import hleper  # typo: "hleper" instead of "helper"

def main():
    hleper()  # ImportError here, at runtime

# If main() is only called in production under certain conditions,
# you might not discover this until 3am on a Saturday.
```

**Circular imports** are a runtime puzzle:

```python
# a.py
from b import B  # runs b.py, which tries to import A

class A:
    def make_b(self):
        return B()

# b.py
from a import A  # runs a.py, which tries to import B - but B doesn't exist yet!

class B:
    def make_a(self):
        return A()

# ImportError: cannot import name 'A' from partially initialized module 'a'
```

The fix is usually moving imports inside functions, or restructuring your code. Neither is satisfying.

Dynamic imports mean your IDE cannot always tell what exists. And `import *` pollutes your namespace in ways you discover by accident:

```python
from utils import *  # brings in... what exactly?
from helpers import *  # hope nothing conflicts!

# Six months later:
# "Where did calculate_tax come from? utils? helpers? Did someone add it last week?"
```

---

## Getting Started: Your First Ferrum Project

Let's create a project from scratch.

### Step 1: Create the Project

```bash
$ ferrum new myapp
Created binary package `myapp`

$ cd myapp
$ tree
.
├── Ferrum.toml
└── src
    └── main.fe

1 directory, 2 files
```

**`Ferrum.toml`:**
```toml
[package]
name    = "myapp"
version = "0.1.0"
edition = "2025"

[dependencies]
```

**`src/main.fe`:**
```ferrum
fn main() {
    println("Hello, world!")
}
```

### Step 2: Build and Run

```bash
$ ferrum build
   Compiling myapp v0.1.0 (/home/dev/myapp)
    Finished dev [unoptimized + debuginfo] target in 0.34s

$ ferrum run
Hello, world!
```

### Step 3: Add Some Code

Let's add a second module. Create `src/greeter.fe`:

```ferrum
// src/greeter.fe

pub fn greet(name: &str) {
    println("Hello, {name}!")
}

fn format_greeting(name: &str): String {
    // Private helper - only visible in this file
    "Hello, ".to_owned() + name + "!"
}
```

Update `src/main.fe`:

```ferrum
// src/main.fe
import greeter.greet

fn main() {
    greet("Ferrum")
}
```

```bash
$ ferrum run
Hello, Ferrum!
```

That's it. No header file. No makefile changes. No `mod greeter;` declaration. You created a file, marked a function `pub`, and imported it.

### Step 4: Add a Dependency

Let's add JSON support. Edit `Ferrum.toml`:

```toml
[package]
name    = "myapp"
version = "0.1.0"
edition = "2025"

[dependencies]
json = "^1.0"
```

```bash
$ ferrum build
    Updating package index
   Compiling json v1.0.3
   Compiling myapp v0.1.0 (/home/dev/myapp)
    Finished dev [unoptimized + debuginfo] target in 1.23s
```

Now use it:

```ferrum
// src/main.fe
import json.{to_json, from_json}

struct User {
    name: String,
    age: u32,
}

fn main(): Result[()]  {
    let user = User { name: "Alice".into(), age: 30 }
    let s = to_json(&user)?
    println("{s}")

    let parsed: User = from_json(&s)?
    println("Parsed: {parsed.name}, {parsed.age}")
    Ok(())
}
```

---

## One File, One Module

In Ferrum, a source file is a module. The filename is the module name. No separate header file. No declaration/definition split.

```
src/
    main.fe          -> module main
    parser.fe        -> module parser
    net/
        http.fe      -> module net.http
        tcp.fe       -> module net.tcp
```

If you want a function to be public, mark it `pub`. Otherwise it is private to the module. That is the entire system.

```ferrum
// parser.fe

pub fn parse(input: &str): Result[Ast] {
    let tokens = tokenize(input)?
    build_ast(tokens)
}

fn tokenize(input: &str): Result[Vec[Token]] {
    // private: only visible within parser.fe
    ...
}

fn build_ast(tokens: Vec[Token]): Result[Ast] {
    // also private
    ...
}
```

Other modules see `parse`. They do not see `tokenize` or `build_ast`. No header file declares this; the `pub` keyword is the declaration and documentation of the interface.

---

## Importing Names

### Basic Import

Bring a name into scope with `import`:

```ferrum
import std.collections.HashMap

fn main() {
    let mut map = HashMap.new()
    map.insert("key", 42)
}
```

The `HashMap` name is now available. No prefix required.

### Import the Module Itself

If you want to use the module as a namespace:

```ferrum
import std.collections

fn main() {
    let mut map = collections.HashMap.new()
    let mut set = collections.HashSet.new()
}
```

### Selective Imports

Import multiple names from one module:

```ferrum
import std.io.{Read, Write, BufReader}
```

This is common for trait imports. You pull in `Read` and `Write` so you can call `.read()` and `.write()` on types that implement them.

### Aliases

Give a module a shorter or clearer name:

```ferrum
import std.collections as col
import std.net as network

fn main() {
    let map = col.HashMap.new()
    let conn = network.TcpStream.connect("example.com:80")?
}
```

Useful when module names are long, or when you want to make the code clearer:

```ferrum
import company.legacy.old_parser as old
import company.v2.new_parser as new

fn migrate(input: &str): Result[Ast] {
    let old_ast = old.parse(input)?
    new.from_legacy(old_ast)
}
```

### Glob Imports (Use Sparingly)

You can import everything from a module:

```ferrum
import std.prelude.*
```

This is discouraged outside of preludes and tests. It hides where names come from and can cause conflicts. When you read code with glob imports, you cannot tell at a glance which module provides which name.

---

## Visibility: A Step-by-Step Walkthrough

Let's trace through exactly how visibility works, using a real scenario.

### Scenario: Building a Database Connection Pool

You're building a connection pool library. You want users to see `Pool` and `Connection`, but not your internal retry logic or connection health checks.

```
mypool/
    Ferrum.toml
    src/
        lib.fe           <- package entry point for libraries
        pool.fe          <- main Pool type
        connection.fe    <- Connection type
        internal/
            health.fe    <- health checks (internal)
            retry.fe     <- retry logic (internal)
```

### Level 1: Private (default) - Same Module Only

```ferrum
// src/internal/retry.fe

const MAX_RETRIES: u32 = 3          // private constant
const BACKOFF_MS: u64 = 100         // private constant

fn calculate_backoff(attempt: u32): Duration {
    // private function - only visible in this file
    Duration.from_millis(BACKOFF_MS * (1 << attempt))
}

pub(crate) fn with_retry[T, F](f: F): Result[T]
where
    F: Fn(): Result[T]
{
    let mut attempt = 0
    loop {
        match f() {
            Ok(v) => return Ok(v),
            Err(e) if attempt < MAX_RETRIES => {
                std.thread.sleep(calculate_backoff(attempt))
                attempt += 1
            }
            Err(e) => return Err(e),
        }
    }
}
```

- `calculate_backoff` is private. Only `retry.fe` can call it.
- `MAX_RETRIES` and `BACKOFF_MS` are private. Implementation details.
- `with_retry` is `pub(crate)` - we'll cover that next.

### Level 2: pub(crate) - Anywhere in the Package

```ferrum
// src/internal/health.fe

import internal.retry.with_retry   // can import pub(crate) items from same package

pub(crate) struct HealthStatus {
    is_healthy: bool,
    last_check: Instant,
    consecutive_failures: u32,
}

pub(crate) fn check_connection(conn: &Connection): HealthStatus {
    // can be called from pool.fe, connection.fe, etc.
    // but NOT from code that depends on this package
    ...
}
```

**Why `pub(crate)`?** Your `Pool` implementation needs to call `check_connection`. But users of your library shouldn't call it directly - it's an implementation detail. They should just call `pool.get()` and trust that unhealthy connections are handled.

### Level 3: pub(super) - Parent Module Only

```ferrum
// src/internal/health.fe

pub(super) fn diagnostic_dump(): String {
    // Only visible to the parent module (src/internal)
    // Not visible to src/pool.fe or src/connection.fe
    ...
}
```

This is less common, but useful when a submodule wants to expose something only to its immediate parent.

### Level 4: pub - World-Visible

```ferrum
// src/pool.fe

import internal.health.check_connection
import internal.retry.with_retry

/// A thread-safe connection pool.
///
/// # Example
/// ```
/// let pool = Pool.new("postgres://localhost/mydb", 10)?
/// let conn = pool.get()?
/// conn.execute("SELECT 1")?
/// ```
pub struct Pool {
    connections: Vec[Connection],    // private field
    max_size: usize,                 // private field
}

impl Pool {
    /// Create a new pool with the given connection string and max size.
    pub fn new(conn_str: &str, max_size: usize): Result[Pool] ! IO + Net {
        ...
    }

    /// Get a connection from the pool, waiting if necessary.
    pub fn get(self: &Self): Result[Connection] ! IO {
        with_retry(|| self.try_get())
    }

    fn try_get(self: &Self): Result[Connection] {
        // private - implementation detail
        ...
    }

    fn return_connection(self: &Self, conn: Connection) {
        // private - called when Connection is dropped
        ...
    }
}
```

Users of your library see:
- `Pool` type (but not its fields)
- `Pool.new()` constructor
- `Pool.get()` method

Users do not see:
- `try_get()` or `return_connection()` methods
- Any types or functions from `internal/`

### What the Compiler Tells You

If someone tries to access something they shouldn't:

```ferrum
// In code that depends on mypool
import mypool.Pool
import mypool.internal.health.check_connection  // trying to import internal API

fn main() {
    let pool = Pool.new("...", 10)?
    let status = check_connection(&pool.connections[0])  // trying to access private field
}
```

Compiler errors:

```
error[E0603]: function `check_connection` is private
 --> main.fe:2:8
  |
2 | import mypool.internal.health.check_connection
  |        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ private function
  |
  = note: this function is marked `pub(crate)`, so it can only be accessed
          within the `mypool` package

error[E0616]: field `connections` of type `Pool` is private
 --> main.fe:6:28
  |
6 |     let status = check_connection(&pool.connections[0])
  |                                        ^^^^^^^^^^^ private field
  |
  = note: consider using a public method to access this field
```

### Visibility Summary Table

| Visibility | Who Can See It | Use Case |
|-----------|----------------|----------|
| (none) | Same module only | Implementation details, helpers |
| `pub(super)` | Parent module | Submodule exposing to its parent |
| `pub(crate)` | Anywhere in the package | Shared internal APIs |
| `pub` | Anyone who imports it | Public API |

---

## Error Messages for Common Mistakes

Ferrum catches module errors at compile time. Here's what the errors look like.

### Importing a Private Item

```ferrum
// math.fe
pub fn add(a: i32, b: i32): i32 { a + b }
fn subtract(a: i32, b: i32): i32 { a - b }  // private!

// main.fe
import math.{add, subtract}  // error!
```

```
error[E0603]: function `subtract` is private
 --> main.fe:1:18
  |
1 | import math.{add, subtract}
  |                   ^^^^^^^^ private function
  |
note: the function is defined here
 --> math.fe:2:1
  |
2 | fn subtract(a: i32, b: i32): i32 { a - b }
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: consider marking it `pub` if it should be accessible from other modules
```

### Importing Something That Does Not Exist

```ferrum
import math.multiply  // math.fe doesn't have this
```

```
error[E0432]: unresolved import `math.multiply`
 --> main.fe:1:8
  |
1 | import math.multiply
  |        ^^^^^^^^^^^^^ no `multiply` in `math`
  |
  = help: available public items in `math`: `add`
```

### Typo in Import Path

```ferrum
import std.collectoins.HashMap  // typo: "collectoins"
```

```
error[E0433]: failed to resolve: could not find `collectoins` in `std`
 --> main.fe:1:12
  |
1 | import std.collectoins.HashMap
  |            ^^^^^^^^^^^ not found in `std`
  |
  = help: did you mean `collections`?
```

### Circular Type Dependencies

Ferrum allows circular imports between modules. You can have `A imports B` and `B imports A`. However, you cannot have circular type dependencies at the definition level:

```ferrum
// a.fe
import b.B

pub struct A {
    b: B,      // A contains B by value
}

// b.fe
import a.A

pub struct B {
    a: A,      // B contains A by value - impossible!
}
```

```
error[E0072]: recursive type `A` has infinite size
 --> a.fe:3:1
  |
3 | pub type A {
  | ^^^^^^^^^^ recursive type has infinite size
  |
note: type `A` contains `B` which contains `A`
 --> a.fe:4:5
  |
4 |     b: B,
  |     ^^^^ contains `B`
  |
note: type `B` contains `A`
 --> b.fe:4:5
  |
4 |     a: A,
  |     ^^^^ contains `A`
  |
  = help: insert indirection (e.g., `Box[A]`) to break the cycle
```

The fix:

```ferrum
// b.fe
import a.A

pub struct B {
    a: Box[A],   // Box breaks the cycle - B has fixed size
}
```

This is the same constraint C has (you can't have `struct A { struct B b; }` and `struct B { struct A a; }`). Ferrum just gives you a clearer error message.

---

## Comparing to C: What Changes

Let's translate a typical C project structure to Ferrum.

### C Version

```
mylib/
    include/
        mylib/
            parser.h
            lexer.h
            ast.h
    src/
        parser.c
        lexer.c
        ast.c
    Makefile
```

**`include/mylib/parser.h`:**
```c
#ifndef MYLIB_PARSER_H
#define MYLIB_PARSER_H

#include "mylib/ast.h"
#include "mylib/lexer.h"

typedef struct Parser Parser;

Parser* parser_new(void);
void parser_free(Parser* p);
Ast* parser_parse(Parser* p, const char* source);

#endif
```

**`src/parser.c`:**
```c
#include "mylib/parser.h"
#include <stdlib.h>
#include <string.h>

struct Parser {
    Lexer* lexer;
    Token current;
};

// Now implement all the functions declared in the header...
Parser* parser_new(void) {
    Parser* p = malloc(sizeof(Parser));
    p->lexer = lexer_new();
    return p;
}

// ... repeat for every function
```

**`Makefile`:**
```makefile
CC = gcc
CFLAGS = -I include -Wall -Wextra
SRCS = src/parser.c src/lexer.c src/ast.c
OBJS = $(SRCS:.c=.o)

libmylib.a: $(OBJS)
    ar rcs $@ $^

%.o: %.c
    $(CC) $(CFLAGS) -c $< -o $@

# Don't forget to list header dependencies...
src/parser.o: include/mylib/parser.h include/mylib/lexer.h include/mylib/ast.h
src/lexer.o: include/mylib/lexer.h
src/ast.o: include/mylib/ast.h
```

### Ferrum Version

```
mylib/
    Ferrum.toml
    src/
        lib.fe
        parser.fe
        lexer.fe
        ast.fe
```

**`Ferrum.toml`:**
```toml
[package]
name = "mylib"
version = "0.1.0"
edition = "2025"
```

**`src/parser.fe`:**
```ferrum
import lexer.{Lexer, Token}
import ast.Ast

pub struct Parser {
    lexer: Lexer,
    current: Token,
}

impl Parser {
    pub fn new(): Parser {
        Parser {
            lexer: Lexer.new(),
            current: Token.Eof,
        }
    }

    pub fn parse(self: &mut Self, source: &str): Result[Ast] {
        self.lexer.set_source(source)
        self.current = self.lexer.next_token()?
        self.parse_expression()
    }

    fn parse_expression(self: &mut Self): Result[Ast] {
        // private helper
        ...
    }
}
```

**That's it.** No header. No include guards. No makefile rules for header dependencies. No forward declarations.

### Side-by-Side Comparison

| C Pain Point | Ferrum Solution |
|--------------|-----------------|
| Include guards (`#ifndef`) | None needed. One file = one module. No multiple inclusion problem. |
| Declaration/definition split | Write once. The compiler extracts the interface. |
| Forward declarations | None needed. Order does not matter; the compiler resolves all names in the module before checking uses. |
| Manual header dependencies in Makefile | Automatic. The compiler tracks dependencies. |
| No namespaces (`json_parse`, `xml_parse`) | Module paths are namespaces: `json.parse`, `xml.parse`. |
| Fragile `-I` include paths | Dependencies declared in `Ferrum.toml`. Resolved automatically. |
| Circular include hell | Just import what you need. Circular imports work (if types don't have circular by-value containment). |

---

## Comparing to Python: What's Different

The import syntax feels similar, but the semantics differ.

### What's Similar

```python
# Python
from collections import defaultdict
import json
from mypackage.utils import helper
```

```ferrum
// Ferrum
import std.collections.HashMap
import json
import mypackage.utils.helper
```

Both use dot-separated paths. Both let you import specific items or whole modules.

### What's Different

| Python | Ferrum |
|--------|--------|
| `ImportError` at runtime | Compile-time error |
| `from x import *` common | Glob imports discouraged |
| Circular imports are runtime errors | Circular imports work (compiler handles them) |
| Dynamic imports (`__import__()`) | No runtime import. All resolved at compile time. |
| No enforced visibility (`_private` is convention) | `pub` / `pub(crate)` / private enforced by compiler |
| `requirements.txt` can drift from actual deps | Lockfile pins exact versions and hashes |
| Virtual environments for isolation | Dependencies are per-package, not global |

### Circular Imports: Ferrum Wins

In Python:
```python
# a.py
from b import B
class A:
    def method(self): return B()

# b.py
from a import A
class B:
    def method(self): return A()

# ImportError at runtime
```

In Ferrum:
```ferrum
// a.fe
import b.B
pub struct A { }
impl A {
    pub fn method(self: &Self): B { B { } }
}

// b.fe
import a.A
pub struct B { }
impl B {
    pub fn method(self: &Self): A { A { } }
}

// Works fine! The compiler handles the circular reference.
```

Ferrum compiles the entire package at once, so it can resolve circular imports that would fail in Python's sequential import system.

### The Typo That Ships to Production

Python:
```python
# This file runs fine on import
from utils import proccess_data  # typo: "proccess"

def handle_request(data):
    return proccess_data(data)  # ImportError here, at runtime
```

You might not discover this until the code path is executed. If `handle_request` is only called for certain edge cases, the bug ships to production.

Ferrum:
```ferrum
import utils.proccess_data  // typo: "proccess"

fn handle_request(data: &Data): Result[Response] {
    proccess_data(data)
}
```

```
error[E0432]: unresolved import `utils.proccess_data`
 --> handler.fe:1:14
  |
1 | import utils.proccess_data
  |              ^^^^^^^^^^^^^ no `proccess_data` in `utils`
  |
  = help: did you mean `process_data`?
```

Build fails. The typo never reaches production.

---

## Packages and Dependencies

### Package Structure

A package is a directory with a `Ferrum.toml` manifest. This is the unit of compilation and distribution.

```
myproject/
    Ferrum.toml
    src/
        main.fe      <- binary entry point
        lib.fe       <- library entry point (if both exist)
        utils.fe
        parser/
            mod.fe   <- not needed! parser.fe or parser/*.fe both work
            lexer.fe
```

### Ferrum.toml Reference

```toml
[package]
name    = "myproject"
version = "0.1.0"
edition = "2025"
authors = ["Your Name <you@example.com>"]
description = "A short description"
license = "MIT"
repository = "https://github.com/you/myproject"

[dependencies]
http = "^2.1"
json = "^1.0"
log  = "^0.5"

[dev-dependencies]
test-utils = "^1.0"
```

### Version Syntax

Ferrum uses semantic versioning:

| Syntax | Meaning | When to Use |
|--------|---------|-------------|
| `"^2.1"` | `>=2.1.0, <3.0.0` | Default. Accept compatible updates. |
| `"~2.1"` | `>=2.1.0, <2.2.0` | Conservative. Patch updates only. |
| `"=2.1.5"` | Exactly this version | Pinning for reproducibility. |
| `">=2.1, <2.5"` | Explicit range | When you need specific bounds. |

The `^` caret is most common: you accept any compatible update within the major version.

---

## Dependency Sources: Practical Scenarios

Dependencies can come from multiple places. Here are real-world scenarios.

### Scenario 1: Using a Published Package

The normal case. Someone published a library, you use it.

```toml
[dependencies]
http = "^2.1"
```

This fetches from the default package registry.

### Scenario 2: Using a Git Repository (Not Yet Published)

Your company has an internal library that isn't on the public registry.

```toml
[dependencies]
# Company's internal auth library
internal-auth = { git = "https://github.com/mycompany/internal-auth" }
```

You can pin to a branch, tag, or commit:

```toml
[dependencies]
# Default branch (usually main)
mylib = { git = "https://github.com/org/mylib" }

# Specific branch (for testing a feature)
mylib = { git = "https://github.com/org/mylib", branch = "feature-x" }

# Specific tag (stable release)
mylib = { git = "https://github.com/org/mylib", tag = "v1.2.3" }

# Specific commit (maximum reproducibility)
mylib = { git = "https://github.com/org/mylib", rev = "a1b2c3d4e5f6" }
```

Use `rev` for reproducibility. Branches can change; commits cannot.

### Scenario 3: Using Your Fork to Fix a Bug

You found a bug in a library. The maintainer hasn't merged your fix yet. Use your fork:

```toml
[dependencies]
# Temporary: using my fork until PR #123 is merged
http = { git = "https://github.com/myname/http", branch = "fix-timeout-bug" }
```

When the fix is merged upstream, switch back:

```toml
[dependencies]
http = "^2.1.1"  # version that includes the fix
```

### Scenario 4: Using a Patched Version Globally

You need to patch a transitive dependency (a dependency of a dependency):

```toml
[dependencies]
web-framework = "^3.0"  # this depends on http

# Patch http wherever it's used
[patch.registry.default]
http = { git = "https://github.com/myname/http", branch = "fix-timeout-bug" }
```

This replaces `http` for your entire dependency tree, not just direct dependencies.

### Scenario 5: Local Development (Multi-Package)

You're developing a library and an app that uses it, side by side:

```
workspace/
    my-lib/
        Ferrum.toml
        src/
            lib.fe
    my-app/
        Ferrum.toml
        src/
            main.fe
```

In `my-app/Ferrum.toml`:

```toml
[dependencies]
my-lib = { path = "../my-lib" }
```

Changes to `my-lib` are picked up immediately on rebuild. No version bump needed. No `ferrum publish` needed.

When you're ready to release:

```toml
[dependencies]
# Change from path to version
my-lib = "^1.0.0"
```

### Scenario 6: Monorepo Package

The library is in a subdirectory of a larger git repo:

```toml
[dependencies]
utils = { git = "https://github.com/org/monorepo", path = "packages/utils" }
```

You can combine with tags:

```toml
[dependencies]
utils = { git = "https://github.com/org/monorepo", tag = "v2.0", path = "packages/utils" }
```

---

## The Lockfile

When you run `ferrum build`, the compiler generates `Ferrum.lock`. This pins exact versions and content hashes:

```toml
# Ferrum.lock - DO NOT EDIT

[[package]]
name = "http"
version = "2.1.5"
source = "registry+https://packages.ferrum-lang.org"
sha256 = "a1b2c3d4e5f6789..."

[[package]]
name = "json"
version = "1.0.3"
source = "registry+https://packages.ferrum-lang.org"
sha256 = "b2c3d4e5f6a7890..."

[[package]]
name = "my-lib"
version = "0.0.0"
source = "git+https://github.com/org/my-lib?rev=deadbeef"
sha256 = "c3d4e5f6g7h8901..."
```

**Commit the lockfile.** It ensures everyone on your team, and your CI server, builds with identical dependencies.

To update dependencies:

```bash
# Update all dependencies to latest compatible versions
$ ferrum update

# Update a specific dependency
$ ferrum update http
```

---

## The Prelude

Every module automatically imports common types and traits. You do not write these imports:

```ferrum
// Automatically available in every module:

// Types
Option[T]    Result[T, E]    String    Vec[T]

// Traits
Copy Clone Drop Default
Display Debug
Eq PartialEq Ord PartialOrd
Iterator
From Into

// Functions
print println panic todo unreachable

// Macros
assert assert_eq assert_ne dbg
```

You can use `Option`, `Vec`, and `println` without any import statement. This keeps common code concise.

If the prelude conflicts with a name you define, your local definition wins. You can always use the full path (`std.option.Option`) to disambiguate.

---

## A Realistic Multi-File Project

Here's a complete project structure for a REST API server:

```
userservice/
    Ferrum.toml
    src/
        main.fe              <- entry point
        config.fe            <- configuration
        server.fe            <- HTTP server setup
        routes/
            mod.fe           <- re-exports public route handlers
            users.fe         <- /users endpoints
            health.fe        <- /health endpoint
        db/
            mod.fe           <- re-exports public db interface
            connection.fe    <- connection pool
            queries.fe       <- SQL queries
            models.fe        <- database models
        middleware/
            auth.fe          <- authentication
            logging.fe       <- request logging
```

**`Ferrum.toml`:**
```toml
[package]
name    = "userservice"
version = "0.1.0"
edition = "2025"

[dependencies]
http   = "^2.0"
sql    = "^1.5"
json   = "^1.0"
log    = "^0.5"
env    = "^0.3"

[dev-dependencies]
test-server = "^1.0"
```

**`src/main.fe`:**
```ferrum
import config.Config
import server.Server

fn main(): Result[()] ! IO + Net {
    log.init()?

    let cfg = Config.from_env()?
    log.info("Starting server on {cfg.host}:{cfg.port}")

    let server = Server.new(&cfg)?
    server.run()
}
```

**`src/config.fe`:**
```ferrum
import env

pub struct Config {
    pub host: String,
    pub port: u16,
    pub database_url: String,
    pub jwt_secret: String,
}

impl Config {
    pub fn from_env(): Result[Config] ! IO {
        Ok(Config {
            host:         env.var("HOST").unwrap_or("0.0.0.0".into()),
            port:         env.var("PORT").unwrap_or("8080".into()).parse()?,
            database_url: env.var("DATABASE_URL")?,
            jwt_secret:   env.var("JWT_SECRET")?,
        })
    }
}
```

**`src/server.fe`:**
```ferrum
import http.{HttpServer, Request, Response}
import config.Config
import routes
import middleware.{auth, logging}
import db.connection.Pool

pub struct Server {
    http: HttpServer,
    db: Pool,
}

impl Server {
    pub fn new(cfg: &Config): Result[Server] ! IO + Net {
        let db = Pool.connect(&cfg.database_url, 10)?
        let http = HttpServer.bind(&cfg.host, cfg.port)?
        Ok(Server { http, db })
    }

    pub fn run(self: &mut Self): Result[()] ! IO + Net {
        loop {
            let (req, responder) = self.http.accept()?

            // Apply middleware
            let req = logging.log_request(req)

            // Route the request
            let response = self.route(&req)?

            // Send response
            responder.send(response)?
        }
    }

    fn route(self: &Self, req: &Request): Result[Response] ! IO {
        match (req.method, req.path.as_str()) {
            (Get, "/health")     => routes.health.check(),
            (Get, "/users")      => routes.users.list(&self.db, req),
            (Get, "/users/{id}") => routes.users.get(&self.db, req),
            (Post, "/users")     => routes.users.create(&self.db, req),
            _                    => Ok(Response.not_found()),
        }
    }
}
```

**`src/routes/users.fe`:**
```ferrum
import http.{Request, Response}
import json
import db.{Pool, queries, models.User}
import middleware.auth

pub fn list(db: &Pool, req: &Request): Result[Response] ! IO {
    let _user = auth.require_auth(req)?

    let users = queries.all_users(db)?
    let body = json.to_string(&users)?

    Ok(Response.ok(body).with_content_type("application/json"))
}

pub fn get(db: &Pool, req: &Request): Result[Response] ! IO {
    let _user = auth.require_auth(req)?
    let id: i64 = req.param("id")?.parse()?

    match queries.user_by_id(db, id)? {
        Some(user) => {
            let body = json.to_string(&user)?
            Ok(Response.ok(body).with_content_type("application/json"))
        }
        None => Ok(Response.not_found()),
    }
}

pub fn create(db: &Pool, req: &Request): Result[Response] ! IO {
    let _user = auth.require_auth(req)?
    let new_user: CreateUser = json.from_str(&req.body)?

    let user = queries.insert_user(db, &new_user)?
    let body = json.to_string(&user)?

    Ok(Response.created(body).with_content_type("application/json"))
}

// Request body type - only used in this module
struct CreateUser {
    name: String,
    email: String,
}
```

**`src/db/models.fe`:**
```ferrum
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
    pub created_at: DateTime,
}

pub(crate) struct UserRow {
    // Internal representation matching database schema
    // Not exposed to users of this library
    id: i64,
    name: String,
    email: String,
    created_at_unix: i64,
}

impl From[UserRow] for User {
    fn from(row: UserRow): User {
        User {
            id: row.id,
            name: row.name,
            email: row.email,
            created_at: DateTime.from_unix(row.created_at_unix),
        }
    }
}
```

**`src/db/queries.fe`:**
```ferrum
import sql.{Query, Row}
import db.connection.Pool
import db.models.{User, UserRow}

pub fn all_users(db: &Pool): Result[Vec[User]] ! IO {
    let rows: Vec[UserRow] = db.query("SELECT * FROM users")?
    Ok(rows.into_iter().map(User.from).collect())
}

pub fn user_by_id(db: &Pool, id: i64): Result[Option[User]] ! IO {
    let row: Option[UserRow] = db.query_one(
        "SELECT * FROM users WHERE id = ?",
        (id,)
    )?
    Ok(row.map(User.from))
}

pub fn insert_user(db: &Pool, name: &str, email: &str): Result[User] ! IO {
    let row: UserRow = db.query_one(
        "INSERT INTO users (name, email) VALUES (?, ?) RETURNING *",
        (name, email)
    )?
    Ok(User.from(row))
}
```

**Notice the patterns:**

1. **Imports are paths from the package root**: `import config.Config` means `src/config.fe`, type `Config`.
2. **Nested modules use directory structure**: `import routes.users` means `src/routes/users.fe`.
3. **External packages use their package name**: `import http.HttpServer` means the `http` dependency from `Ferrum.toml`.
4. **`pub(crate)` for internal APIs**: `UserRow` is shared between `models.fe` and `queries.fe`, but not exposed outside the package.
5. **Private by default**: `CreateUser` in `routes/users.fe` is not visible anywhere else.

---

## Workspaces: Multiple Packages in One Repo

For larger projects, you might have multiple packages:

```
myorg/
    Ferrum.toml          <- workspace root
    packages/
        core/
            Ferrum.toml
            src/
                lib.fe
        cli/
            Ferrum.toml
            src/
                main.fe
        server/
            Ferrum.toml
            src/
                main.fe
```

**Workspace root `Ferrum.toml`:**
```toml
[workspace]
members = [
    "packages/core",
    "packages/cli",
    "packages/server",
]

[workspace.dependencies]
# Shared versions for all members
http = "^2.1"
json = "^1.0"
log  = "^0.5"
```

**`packages/cli/Ferrum.toml`:**
```toml
[package]
name = "myorg-cli"
version = "0.1.0"

[dependencies]
core = { path = "../core" }
http = { workspace = true }   # uses version from workspace root
```

Workspaces let you:
- Share dependency versions across packages
- Build everything together (`ferrum build --workspace`)
- Run tests for all packages with one command (`ferrum test --workspace`)
- Share a single `Ferrum.lock` file

---

## Quick Reference

```ferrum
// Import a single item
import std.collections.HashMap

// Import multiple items
import std.io.{Read, Write, BufReader}

// Import the module itself
import std.net

// Alias
import std.collections as col

// Visibility
pub fn public_api() { }           // anyone can call
pub(crate) fn internal() { }      // package only
fn private_helper() { }           // this module only

// Type visibility
pub struct PublicStruct {
    pub visible_field: i32,
    hidden_field: i32,            // private even though type is pub
}
```

**`Ferrum.toml` quick reference:**
```toml
[package]
name    = "mypackage"
version = "0.1.0"
edition = "2025"

[dependencies]
name = "^1.0"                                      # registry
name = { git = "https://...", tag = "v1.0" }       # git
name = { path = "../local" }                       # local

[dev-dependencies]
test-lib = "^1.0"

[patch.registry.default]
name = { git = "https://...", branch = "fix" }     # global patch
```

---

## Summary

- One file is one module. The filename is the module name.
- No header files, no include guards, no forward declarations.
- `import` brings names into scope. Imports are checked at compile time.
- `pub` makes things public. Default is private. `pub(crate)` for package-internal APIs.
- Packages are defined by `Ferrum.toml`. Dependencies come from registries, git, or local paths.
- The lockfile (`Ferrum.lock`) pins exact versions for reproducibility.
- The prelude provides common types so you do not need to import them.

**If you're coming from C:** No more maintaining two copies of every declaration. No more include guard boilerplate. No more linker errors from header/source mismatches. No more naming everything `mylib_mymodule_myfunction` to avoid collisions.

**If you're coming from Python:** Same clean syntax, but the compiler catches import errors before you ship. Circular imports just work. Visibility is enforced, not just a `_underscore` convention.
