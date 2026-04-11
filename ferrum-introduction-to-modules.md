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

Include guards (`#ifndef`) are boilerplate you write hundreds of times. Forget them and you get "redefinition" errors. Put them wrong and you get silent ODR violations.

Forward declarations are another tax: to use a type before it is defined, you declare it again. More duplication, more divergence risk.

And there is no namespacing. Two libraries with a `parse()` function? Collision. The fix is naming conventions: `json_parse()`, `xml_parse()`, `mycompany_json_parse()`. Your code becomes a monument to the lack of modules.

### Python: Runtime Surprises

Python has clean import syntax. But it is checked at runtime:

```python
from utils import hleper  # typo: "hleper" instead of "helper"

# This runs fine until you call hleper()
# Then: ImportError at runtime, possibly in production
```

Circular imports are a runtime puzzle. Dynamic imports mean your IDE cannot always tell what exists. And `import *` pollutes your namespace in ways you discover by accident.

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

## Visibility

### Default: Private

Everything is private by default. If you do not write `pub`, the item is visible only within its module.

```ferrum
// internal.fe

type InternalState {        // private type
    buffer: Vec[u8],
    cursor: usize,
}

fn reset_buffer(s: &mut InternalState) {  // private function
    s.buffer.clear()
    s.cursor = 0
}
```

Other modules cannot see `InternalState` or `reset_buffer`. They do not exist outside this file.

### `pub`: World-Visible

Mark something `pub` and it is visible to any module that imports it:

```ferrum
// api.fe

pub type Config {
    pub host: String,
    pub port: u16,
    timeout: Duration,   // private field!
}

pub fn connect(cfg: &Config): Result[Connection] ! Net {
    ...
}
```

Note: `Config` is public, but its `timeout` field is private. Outside code can create a `Config` only through a constructor you provide, or by using struct update syntax from an existing `Config`.

### `pub(crate)`: Package-Visible

Sometimes you need internal sharing across modules within your package, but not exposure to the outside world:

```ferrum
// internal/cache.fe

pub(crate) type CacheEntry {
    key: String,
    value: Vec[u8],
    expires: Instant,
}

pub(crate) fn evict_expired(cache: &mut Cache) {
    ...
}
```

Other modules in your package can use `CacheEntry` and `evict_expired`. But if someone depends on your package as a library, they cannot see these items.

### `pub(super)`: Parent Module Visible

For finer-grained control, `pub(super)` makes an item visible to the parent module:

```ferrum
// net/internal.fe

pub(super) fn raw_socket_operation() {
    // visible to net.fe, but not to net.http or outside code
}
```

### Visibility Summary

| Visibility | Who Can See It |
|-----------|----------------|
| (none) | Same module only |
| `pub(super)` | Parent module |
| `pub(crate)` | Anywhere in the package |
| `pub` | Anyone who imports it |

---

## Packages

A package is a directory with a `Ferrum.toml` manifest. This is the unit of compilation and distribution.

### Minimal Package

```
myproject/
    Ferrum.toml
    src/
        main.fe
```

```toml
# Ferrum.toml
[package]
name    = "myproject"
version = "0.1.0"
edition = "2025"
```

Run `ferrum build` and you get a binary.

### Package with Dependencies

```toml
[package]
name    = "webapp"
version = "1.0.0"
edition = "2025"

[dependencies]
http = "^2.1"
json = "^1.0"
log  = "^0.5"

[dev-dependencies]
test-utils = "^1.0"
```

`[dependencies]` are linked into your binary. `[dev-dependencies]` are only for tests and benchmarks.

### Version Syntax

Ferrum uses semantic versioning:

| Syntax | Meaning |
|--------|---------|
| `"^2.1"` | `>=2.1.0, <3.0.0` (compatible updates) |
| `"~2.1"` | `>=2.1.0, <2.2.0` (patch updates only) |
| `"=2.1.5"` | Exactly this version |
| `">=2.1, <2.5"` | Explicit range |

The `^` caret is most common: you accept any compatible update within the major version.

---

## Dependency Sources

Dependencies can come from multiple places.

### Registry (Default)

```toml
[dependencies]
http = "^2.1"
```

This fetches from the default package registry. Most dependencies look like this.

### Git Repository

```toml
[dependencies]
# Default branch
mylib = { git = "https://github.com/org/mylib" }

# Specific branch
mylib = { git = "https://github.com/org/mylib", branch = "develop" }

# Specific tag
mylib = { git = "https://github.com/org/mylib", tag = "v1.2.3" }

# Specific commit (reproducible)
mylib = { git = "https://github.com/org/mylib", rev = "a1b2c3d" }
```

Use `rev` for reproducibility. Branches can change; commits cannot.

### Path (Local Development)

```toml
[dependencies]
my-local-lib = { path = "../my-local-lib" }
```

This is for developing multiple packages together. Changes to the path dependency are picked up immediately, no version bump needed.

Common pattern: develop with `path`, publish with a version:

```toml
[dependencies]
# During development:
my-lib = { path = "../my-lib" }

# For release, change to:
# my-lib = "^1.0"
```

### Monorepo Packages

For packages in a subdirectory of a git repo:

```toml
[dependencies]
utils = { git = "https://github.com/org/monorepo", path = "packages/utils" }
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
```

Commit the lockfile. It ensures everyone on your team, and your CI server, builds with identical dependencies.

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

## A Multi-File Project Example

Here is a realistic project structure:

```
myapp/
    Ferrum.toml
    src/
        main.fe
        config.fe
        server/
            http.fe
            handlers.fe
        db/
            connection.fe
            queries.fe
```

**`Ferrum.toml`:**
```toml
[package]
name    = "myapp"
version = "0.1.0"
edition = "2025"

[dependencies]
http   = "^2.0"
sql    = "^1.5"
json   = "^1.0"
log    = "^0.5"
```

**`src/main.fe`:**
```ferrum
import config.Config
import server.http.Server

fn main(): Result[()] ! IO + Net {
    let cfg = Config.from_env()?
    let server = Server.new(&cfg)?
    server.run()
}
```

**`src/config.fe`:**
```ferrum
pub type Config {
    pub host: String,
    pub port: u16,
    pub db_url: String,
}

impl Config {
    pub fn from_env(): Result[Config] ! IO {
        Ok(Config {
            host:   std.env.var("HOST")?.unwrap_or("localhost".into()),
            port:   std.env.var("PORT")?.unwrap_or("8080".into()).parse()?,
            db_url: std.env.var("DATABASE_URL")?,
        })
    }
}
```

**`src/server/http.fe`:**
```ferrum
import http.{HttpServer, Request, Response}
import config.Config
import server.handlers

pub type Server {
    inner: HttpServer,
}

impl Server {
    pub fn new(cfg: &Config): Result[Server] ! Net {
        let inner = HttpServer.bind(&cfg.host, cfg.port)?
        Ok(Server { inner })
    }

    pub fn run(self: &mut Self): Result[()] ! IO + Net {
        loop {
            let (req, responder) = self.inner.accept()?
            let response = handlers.route(&req)?
            responder.send(response)?
        }
    }
}
```

**`src/server/handlers.fe`:**
```ferrum
import http.{Request, Response}
import db.queries

pub fn route(req: &Request): Result[Response] ! IO {
    match req.path.as_str() {
        "/health" => Ok(Response.ok("healthy")),
        "/users"  => list_users(req),
        _         => Ok(Response.not_found()),
    }
}

fn list_users(req: &Request): Result[Response] ! IO {
    let users = queries.all_users()?
    let body = json.to_string(&users)?
    Ok(Response.ok(body).with_content_type("application/json"))
}
```

Notice:

1. **Imports are paths from the package root**: `import config.Config` means `src/config.fe`, type `Config`.
2. **Nested modules use directory structure**: `import server.handlers` means `src/server/handlers.fe`.
3. **External crates use their package name**: `import http.HttpServer` means the `http` dependency from `Ferrum.toml`.
4. **Visibility controls the API**: `Config` is `pub`, so `main.fe` can use it. The handlers module keeps `list_users` private; only `route` is exposed.

---

## Comparing to C

| C Problem | Ferrum Solution |
|-----------|-----------------|
| Include guards (`#ifndef`) | None needed. One file = one module. |
| Forward declarations | None needed. Order does not matter within a module; the compiler resolves names across all items. |
| Header/source split | None. Write the function once. |
| No namespaces | Module paths are namespaces: `net.http.Server`, `db.queries.all_users`. |
| Fragile include paths (`-I` flags) | Dependencies declared in `Ferrum.toml`. The build system resolves them. |
| Manual dependency management | `Ferrum.toml` + lockfile. Reproducible, versioned, hash-verified. |

---

## Comparing to Python

| Python Behavior | Ferrum Behavior |
|-----------------|-----------------|
| `ImportError` at runtime | Compile-time error. If the module or name does not exist, the build fails. |
| `from x import *` hides origins | Glob imports exist but are discouraged. Explicit imports are preferred. |
| Circular import issues | The compiler handles cycles. You can have `A imports B` and `B imports A` as long as there is no type-level cycle. |
| Dynamic imports (`__import__`) | No runtime import. All imports resolved at compile time. |
| No visibility control | `pub` / `pub(crate)` / private. Encapsulation is enforced. |
| `requirements.txt` drift | Lockfile pins exact versions and hashes. |

The import syntax is similar (`import x.y.z`), so the transition is easy. The difference is that Ferrum checks everything at compile time. A typo in an import is a build error, not a production incident.

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
- Build everything together
- Run tests for all packages with one command
- Publish packages independently with consistent versioning

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
pub type PublicStruct {
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

If you are coming from C: no more maintaining two copies of every declaration. If you are coming from Python: same clean syntax, but the compiler catches import errors before you ship.
