# Zinc Language Specification

**Version:** 0.3
**Status:** Design specification
**Elevator pitch:** Zig, but the allocator is ambient, functions have
                    pre/post conditions, the binary layout is verified,
                    the numeric tower is complete, and concurrency
                    primitives have typed wrappers.
**Base language:** Zig 0.13. Everything not mentioned here is unchanged.

---

## What This Is

Zinc makes six additions to Zig:

1. **Ambient allocators.** The allocator stops being a function parameter
   you thread through every call. It becomes a scoped capability declared
   at API boundaries and resolved automatically inside function bodies.

2. **Design by contract.** Functions can declare preconditions (`requires`),
   postconditions (`ensures`), and types can declare invariants. These are
   checked at runtime in debug mode and stripped in release mode.

3. **Verified binary layout declarations.** Structs that represent hardware
   registers, network packets, or binary file formats get a `layout` block
   that specifies exact bit positions, byte order, and total size. The
   compiler verifies correctness and generates read/write/view codecs.

4. **Extended numeric types.** `bf16` for ML work, `d32`/`d64`/`d128`
   decimal floats for exact decimal arithmetic, `Fixed(I, F)` fixed-point
   for DSP and embedded, `BigDecimal` and `Rational` in the standard library.

5. **Numeric comptime interfaces.** Structured constraints (`Integer`,
   `Float`, `Numeric`, `SignedInt`) replacing ad-hoc `@typeInfo` checks in
   generic numeric functions.

6. **Concurrency and pointer stdlib additions.** `TaggedPtr(T, n_bits)` as
   a typed, contract-verified wrapper. `SyncGuard` as a lint convention
   that warns on lock-holding across suspension points. A CAS-based STM
   library. Hardware TM shims in platform namespaces.

Everything else is Zig. Comptime. Tagged unions. `!T` error unions. `defer`.
No garbage collector. No hidden control flow. No borrow checker. No runtime.
C interop. Cross-compilation.

If you know Zig, you know 94% of Zinc already.

---

## The Problem With Zig's Allocator Model

In Zig, every function that allocates memory takes an `std.mem.Allocator`
parameter. This is honest — you can see where allocation happens — but it
generates noise:

```zig
fn parse(allocator: std.mem.Allocator, input: []const u8) !Value {
    var tokens = std.ArrayList(Token).init(allocator);
    defer tokens.deinit();
    var nodes = std.ArrayList(Node).init(allocator);
    defer nodes.deinit();
    const result = try buildAst(allocator, tokens.items, nodes.items);
    return result;
}

fn buildAst(allocator: std.mem.Allocator, tokens: []Token, nodes: []Node) !Ast {
    ...
    const child = try parseChild(allocator, tokens[i..]);
    ...
}

fn parseChild(allocator: std.mem.Allocator, tokens: []Token) !Node {
    ...
}
```

The allocator travels through every function in the call chain. It is
rarely the interesting part of any function signature. In a large codebase,
this is significant noise.

The Zig community is aware of this. The explicit allocator is an intentional
design decision — Zig values explicitness and rejects hidden costs. Zinc
respects this: the allocator remains explicit at function API boundaries.
It just stops being a parameter.

---

## Ambient Allocators

### The `using` clause

A function that requires an allocator declares it with `using`:

```zig
fn parse(input: []const u8) !Value  using(allocator) {
    var tokens = std.ArrayList(Token).init(allocator);
    defer tokens.deinit();
    var nodes = std.ArrayList(Node).init(allocator);
    defer nodes.deinit();
    const result = try buildAst(tokens.items, nodes.items);
    return result;
}

fn buildAst(tokens: []Token, nodes: []Node) !Ast  using(allocator) {
    ...
    const child = try parseChild(tokens[i..]);
    ...
}

fn parseChild(tokens: []Token) !Node  using(allocator) {
    ...
}
```

`using(allocator)` means: "this function requires an ambient allocator
named `allocator` of type `std.mem.Allocator`." The allocator is visible
inside the function body by that name. It is not a parameter — it does not
appear in the function's parameter list and callers do not pass it.

The function signature communicates that allocation happens. The call sites
do not carry the noise.

### The `with` block

The ambient allocator is set by a `with` block:

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    with (allocator: gpa.allocator()) {
        const value = try parse(input);
        defer value.deinit();
        try process(value);
    }
}
```

`with (allocator: expr)` sets the ambient allocator for the lexical scope
of the block. Every call to a `using(allocator)` function inside that block
resolves to `expr`.

**Nesting:** Inner `with` blocks override outer ones for their scope.

```zig
with (allocator: gpa.allocator()) {
    const long_lived = try loadConfig(path);  // uses gpa

    with (allocator: arena.allocator()) {
        const tmp = try parseRequest(data);   // uses arena
        defer arena.reset();
    }  // arena scope ends

    try process(long_lived);  // uses gpa again
}
```

**Tests:** The test allocator is set in the same way.

```zig
test "parse round-trips" {
    with (allocator: std.testing.allocator) {
        const result = try parse("hello world");
        defer result.deinit();
        try std.testing.expectEqualStrings("hello world", result.text);
    }
}
```

This replaces the current pattern of passing `std.testing.allocator`
explicitly to every function under test.

### Naming the allocator

The conventional name is `allocator`. When a function uses the conventional
name, the `with` block can use a positional shorthand:

```zig
with (gpa.allocator()) {    // shorthand: sets the default 'allocator'
    ...
}
```

This is equivalent to:

```zig
with (allocator: gpa.allocator()) {
    ...
}
```

### Two allocators simultaneously

When a function needs two allocators with different lifetimes or purposes:

```zig
fn compile(source: []const u8) !Artifact
    using(scratch, persistent)  // two named allocators
{
    // Scratch: temporary parse data, freed after compilation
    const ast = try scratch.create(Ast);
    defer scratch.destroy(ast);

    // Persistent: the output artifact, outlives this function
    const artifact = try persistent.create(Artifact);
    try buildArtifact(artifact, ast);
    return artifact;
}

// Call site:
with (scratch: arena.allocator(), persistent: gpa.allocator()) {
    const artifact = try compile(source);
}
```

Both names are visible in the function body and must be provided by the
`with` block. The compiler errors if a `using(name)` function is called
and no enclosing `with` provides that name.

### Explicit override at call site

When you need to call a `using(allocator)` function with a specific
allocator outside a `with` block — in FFI callbacks, in `comptime` contexts,
or when crossing a task boundary — you can pass it explicitly:

```zig
// Explicit override syntax
const result = try parse(input) with(allocator: special_allocator);
```

This is the escape hatch for cases where lexical scoping does not fit.
It should be rare.

### What `using` does NOT do

`using` is not:

- A type parameter. There is no `Alloc[Arena]` or polymorphism over allocator
  types. `std.mem.Allocator` is the single interface, same as today.
- An effect annotation. There is no effect system tracking which functions
  allocate.
- A capability in the Ferrum sense. There is no ambient capability
  resolution system beyond lexical `with` scoping.

It is simply a scoped implicit parameter resolved by name at the call site.
The implementation is a small compiler pass, not a type system feature.

---

## Design by Contract

### Overview

Zinc adds Eiffel-style design by contract. Contracts are:

- **Checked at runtime in debug mode** (`-Ddebug` or `builtin.mode == .Debug`)
- **Stripped entirely in release mode** (`-Drelease-fast`, `-Drelease-safe`,
  `-Drelease-small`)
- **Not formal proofs.** No SMT solver. No proof terms. Just assertions that
  run.

The value is documentation that the compiler enforces and tests can exercise.

### `requires` — preconditions

A precondition is the caller's responsibility. If a `requires` clause fires,
the bug is at the call site.

```zig
fn binarySearch(items: []const i32, target: i32) ?usize
    requires items.len == 0 or isSorted(items)
{
    // Implementation can assume items is sorted
    ...
}

fn divide(a: f64, b: f64) f64
    requires b != 0.0
{
    return a / b;
}

fn getIndex(items: []const u8, idx: usize) u8
    requires idx < items.len
{
    return items[idx];
}
```

In debug mode, `requires P` compiles to:

```zig
std.debug.assert(P);  // or: if (!P) @panic("precondition violated: " ++ @src().fn_name);
```

at the top of the function body, before any other code.

Multiple `requires` clauses are allowed and all are checked:

```zig
fn insertSorted(list: *SortedList, item: i32) !void
    requires list.items.len < list.capacity
    requires item >= 0
{
    ...
}
```

### `ensures` — postconditions

A postcondition is the implementer's responsibility. If an `ensures` clause
fires, the bug is in the function body.

```zig
fn push(self: *ArrayList(T), item: T) !void
    ensures self.items.len == old(self.items.len) + 1
    ensures self.items[self.items.len - 1] == item
{
    try self.items.append(item);
}
```

`result` refers to the return value and is available in `ensures` clauses:

```zig
fn max(a: i32, b: i32) i32
    ensures result >= a
    ensures result >= b
    ensures result == a or result == b
{
    return if (a > b) a else b;
}

fn sqrt(x: f64) f64
    requires x >= 0.0
    ensures result >= 0.0
    ensures @abs(result * result - x) < 1e-10
{
    return std.math.sqrt(x);
}
```

**Error returns:** `ensures` clauses only apply on non-error return paths.
If a function returns `!T` and returns an error, postconditions are not
checked. This matches Eiffel's model — the postcondition defines what holds
when the operation succeeds.

```zig
fn readLine(reader: anytype, buf: []u8) ![]u8
    requires buf.len > 0
    ensures result.len > 0             // only checked on Ok return
    ensures result.len <= buf.len      // only checked on Ok return
{
    const n = try reader.read(buf);
    if (n == 0) return error.EndOfFile;
    return buf[0..n];
}
```

### `old()` — pre-call snapshots

`old(expr)` captures the value of `expr` at function entry, for use in
`ensures` clauses.

```zig
fn pop(self: *ArrayList(T)) !T
    requires self.items.len > 0
    ensures self.items.len == old(self.items.len) - 1
{
    return self.items.pop();
}

fn transfer(from: *Account, to: *Account, amount: u64) !void
    requires from.balance >= amount
    ensures from.balance == old(from.balance) - amount
    ensures to.balance == old(to.balance) + amount
{
    from.balance -= amount;
    to.balance += amount;
}
```

The compiler generates a snapshot variable at function entry for each
expression referenced in `old()`:

```zig
// Desugaring of the pop() function above:
fn pop(self: *ArrayList(T)) !T {
    if (builtin.mode == .Debug) {
        std.debug.assert(self.items.len > 0);
        const __old_items_len = self.items.len;
        const result = self.items.pop();
        std.debug.assert(self.items.len == __old_items_len - 1);
        return result;
    }
    return self.items.pop();
}
```

Only expressions that actually appear in `old()` are snapshotted. The
compiler does not copy the entire state of all arguments.

For types that require allocation to copy (slices, complex structs), `old()`
triggers a shallow copy via the ambient allocator if one is in scope, or a
compile error if no allocator is available and the type is not `Copy`-able.

### `invariant` — struct invariants

A struct invariant declares a property that must hold after every method
that takes a mutable pointer to the struct.

```zig
const SortedList = struct {
    items: []i32,
    len: usize,

    invariant self.len <= self.items.len
    invariant isSorted(self.items[0..self.len])

    pub fn insert(self: *SortedList, item: i32) !void  using(allocator) {
        // grow if needed
        if (self.len == self.items.len) {
            self.items = try allocator.realloc(self.items, self.items.len * 2);
        }
        // find insertion point and shift
        var i = self.len;
        while (i > 0 and self.items[i-1] > item) : (i -= 1) {
            self.items[i] = self.items[i-1];
        }
        self.items[i] = item;
        self.len += 1;
        // invariant checked here automatically in debug mode
    }

    pub fn get(self: *const SortedList, idx: usize) i32 {
        // no invariant check — *const method cannot violate invariant
        return self.items[idx];
    }
};
```

The invariant is checked automatically after the body of any method that
takes `*Self` (mutable pointer). Methods taking `*const Self` do not trigger
the check — they cannot modify the struct and therefore cannot violate the
invariant.

Multiple invariants are allowed and all are checked:

```zig
const BoundedQueue = struct {
    data: []u8,
    head: usize,
    tail: usize,
    count: usize,

    invariant self.count <= self.data.len
    invariant self.head < self.data.len
    invariant self.tail < self.data.len
    invariant self.count == (self.tail - self.head + self.data.len) % self.data.len
        or (self.count == 0)
};
```

### Contract interaction with `comptime`

Contracts are runtime assertions. They do not participate in comptime
evaluation. A `comptime` expression that calls a function with a `requires`
clause does not trigger the check — comptime execution is already verified
by the type system and comptime evaluation semantics.

This is a deliberate simplification. Runtime contracts and comptime
verification are separate mechanisms for separate purposes.

### Contract enforcement modes

| Build mode | `requires` | `ensures` | `invariant` |
|------------|------------|-----------|-------------|
| Debug | runtime assert | runtime assert | checked after `*Self` methods |
| ReleaseSafe | runtime assert | stripped | stripped |
| ReleaseFast | stripped | stripped | stripped |
| ReleaseSmall | stripped | stripped | stripped |

`ReleaseSafe` keeps `requires` checks because precondition violations are
caller bugs — you want to catch those even in production. Postconditions
and invariants are stripped because they add overhead proportional to
function complexity.

Override per-function if needed:

```zig
// Keep this check even in release — security-critical precondition
fn authenticate(token: []const u8) !Claims
    requires(always) token.len == 32
{
    ...
}

// Strip even in debug — too expensive for this hot loop
fn innerKernel(data: []const f32) f32
    requires(never) data.len % 8 == 0
{
    ...
}
```

`requires(always)` keeps the check in all build modes.
`requires(never)` strips the check in all build modes (documentation only).

### Contract expressions

Contract expressions are Zig expressions. They can call any pure function
visible at the call site:

```zig
fn isSorted(items: []const i32) bool {
    if (items.len < 2) return true;
    for (items[0..items.len-1], items[1..]) |a, b| {
        if (a > b) return false;
    }
    return true;
}

fn binarySearch(items: []const i32, target: i32) ?usize
    requires isSorted(items)  // calls a regular Zig function
{
    ...
}
```

Contract expressions should be pure — no side effects, no allocation (unless
using `old()` with an ambient allocator). The compiler does not enforce this
in v1, but a contract that allocates or has side effects in debug mode will
produce confusing behavior.

### `forall` in contracts

For contracts that range over a collection, a limited `forall` expression
is available:

```zig
fn sort(items: []i32) void
    ensures forall (i in 0..items.len-1) items[i] <= items[i+1]
    ensures forall (i in 0..items.len) old(containsElement(items, items[i]))
    // every element in output was in input
{
    std.sort.heap(i32, items, {}, std.sort.asc(i32));
}
```

`forall (binding in range) expr` iterates `range` and asserts `expr` for
each element. In debug mode this is an O(n) loop. In release mode it is
stripped.

---

## Interaction Between the Two Features

Allocators and contracts interact in one specific place: `old()` for
non-Copy types.

When a contract uses `old(expr)` and `expr` has a type that cannot be
trivially copied (a slice, a pointer to heap data, a struct containing
pointers), the compiler needs to make a copy at function entry.

If the function has `using(allocator)` in scope, the copy uses that
allocator. The snapshot is freed at the end of the function regardless
of return path.

```zig
fn sort(items: []i32) void
    using(allocator)
    ensures isPermutation(items, old(items))
    // old(items) is a slice — needs a copy
{
    // Compiler generates at entry:
    //   const __old_items = try allocator.dupe(i32, items);
    //   defer allocator.free(__old_items);
    std.sort.heap(i32, items, {}, std.sort.asc(i32));
    // Compiler generates at exit:
    //   assert(isPermutation(items, __old_items));
}
```

If no allocator is in scope and `old(expr)` requires one, the compiler
emits an error:

```
error: `old(items)` requires a copy of type `[]i32`, but no ambient
       allocator is in scope. Add `using(allocator)` to this function.
```

---

## Example: A complete data structure

```zig
const MinHeap = struct {
    data: []i32,
    len: usize,

    invariant self.len <= self.data.len
    invariant isHeap(self.data[0..self.len])

    pub fn init(capacity: usize) !MinHeap  using(allocator) {
        return .{
            .data = try allocator.alloc(i32, capacity),
            .len = 0,
        };
    }

    pub fn deinit(self: *MinHeap) void  using(allocator) {
        allocator.free(self.data);
        self.* = undefined;
    }

    pub fn push(self: *MinHeap, value: i32) !void  using(allocator)
        ensures self.len == old(self.len) + 1
        ensures containsElement(self.data[0..self.len], value)
    {
        if (self.len == self.data.len) {
            self.data = try allocator.realloc(self.data, self.data.len * 2);
        }
        self.data[self.len] = value;
        self.len += 1;
        siftUp(self.data, self.len - 1);
    }

    pub fn pop(self: *MinHeap) ?i32
        requires self.len > 0
        ensures result != null
        ensures self.len == old(self.len) - 1
        ensures result.? <= old(self.data[0])
    {
        if (self.len == 0) return null;
        const min = self.data[0];
        self.len -= 1;
        if (self.len > 0) {
            self.data[0] = self.data[self.len];
            siftDown(self.data, 0, self.len);
        }
        return min;
    }

    pub fn peek(self: *const MinHeap) ?i32 {
        // *const method — no invariant check, no contract needed
        if (self.len == 0) return null;
        return self.data[0];
    }

    fn siftUp(data: []i32, idx: usize) void { ... }
    fn siftDown(data: []i32, idx: usize, len: usize) void { ... }
};

fn isHeap(items: []const i32) bool {
    var i: usize = 1;
    while (i < items.len) : (i += 1) {
        if (items[(i-1)/2] > items[i]) return false;
    }
    return true;
}

fn containsElement(items: []const i32, value: i32) bool {
    for (items) |item| if (item == value) return true;
    return false;
}

// Usage:
pub fn heapSort(items: []i32) !void  using(allocator) {
    var heap = try MinHeap.init(items.len);
    defer heap.deinit();

    for (items) |item| try heap.push(item);

    var i: usize = 0;
    while (heap.pop()) |min| : (i += 1) {
        items[i] = min;
    }
}
```

The `with` block that the caller provides:

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var data = [_]i32{ 5, 3, 8, 1, 9, 2, 7, 4, 6 };

    with (gpa.allocator()) {
        try heapSort(&data);
    }

    // data is now sorted
    std.debug.print("{any}\n", .{data});
}
```

---

## What Is Explicitly Not Added

- **No borrow checker.** Zinc is manual memory management, same as Zig. Contracts help document ownership assumptions but do not enforce them.
- **No effect system.** There is no tracking of which functions do I/O, allocate, or have side effects beyond what is visible in the signature.
- **No type classes or traits.** Comptime duck typing handles generics, same as Zig.
- **No region inference.** The `with` block is the region. It is lexical and explicit.
- **No formal verification.** Contracts are runtime assertions in debug mode. They are not SMT-discharged. They are not proof obligations. A `requires` clause that is satisfied by all inputs is documentation, not a theorem.
- **No safety level ladder.** Zig's existing `unsafe` story (pointer arithmetic, `@ptrCast`, etc.) is unchanged.
- **No sum types with exhaustive matching beyond tagged unions.** Zig's tagged unions are the sum type story.
- **No hardware transactional memory in the core stdlib.** HTM (Intel TSX, ARM TME, IBM POWER HTM) lives in platform-specific namespaces. See §Concurrency below.
- **No CHERI capability pointer support.** CHERI requires ABI-level changes to pointer representation that are out of scope for Zinc. If CHERI support lands in upstream Zig it will be inherited.

---

## Concurrency and Pointer Annotations

### What Zig already has — and it is complete

Zig's atomic primitives are first-class builtins that map directly to LLVM
atomic instructions with no overhead:

```zig
// All C11 memory orderings: Unordered, Monotonic, Acquire, Release, AcqRel, SeqCst
const val  = @atomicLoad(u32, &shared, .Acquire);
@atomicStore(u32, &shared, new_val, .Release);
const old  = @atomicRmw(u32, &counter, .Add, 1, .SeqCst);
const rslt = @cmpxchg(u32, &shared, expected, desired, .AcqRel, .Monotonic);
@fence(.SeqCst);
```

`std.atomic.Value(T)` provides a method-call wrapper. `std.Thread.Mutex`,
`std.Thread.RwLock`, `std.Thread.Semaphore`, and `std.Thread.Condition` all
exist and work. Zinc inherits all of this unchanged.

Zig also has three pointer qualifiers already:

- `*volatile T` — volatile access, correct for MMIO, unchanged in Zinc
- `*allowzero T` — opt-in nullable pointer, correct for kernel work at
  address 0, unchanged in Zinc
- `*align(N) T` — alignment in the type, used for SIMD, unchanged in Zinc

Zinc does not touch any of these.

### What Zinc adds: four stdlib additions

#### 1. `std.atomic.Value(T)` ergonomic improvements

Zig's `std.atomic.Value(T)` exists but the community is still settling
conventions. Zinc standardizes the method names and adds `fetchMax`,
`fetchMin`, and `compareExchangeLoop`:

```zig
var counter = std.atomic.Value(u64).init(0);

_ = counter.fetchAdd(1, .monotonic);
_ = counter.fetchSub(1, .release);
_ = counter.fetchAnd(mask, .seq_cst);
_ = counter.fetchOr(flags, .acq_rel);
_ = counter.fetchXor(bits, .monotonic);
_ = counter.fetchMax(val, .monotonic);   // added in Zinc
_ = counter.fetchMin(val, .monotonic);   // added in Zinc

const old = counter.load(.acquire);
counter.store(new_val, .release);
const swapped = counter.swap(new_val, .seq_cst);

// Spin until CAS succeeds — common pattern, made explicit
counter.compareExchangeLoop(expected, desired, .acq_rel, .monotonic,
    struct {
        fn onFail(actual: u64) void { _ = actual; }  // called on each retry
    }
);
```

No compiler changes. Pure stdlib work.

#### 2. `SyncGuard` marker interface

A comptime interface that Mutex guards, RwLock guards, and Semaphore
permits implement. The Zinc linter warns when a `SyncGuard` value is
live across an async suspension point — the most common deadlock pattern
in async code.

```zig
// In std.sync:
pub fn isSyncGuard(comptime T: type) bool {
    return @hasDecl(T, "__sync_guard_marker");
}

// MutexGuard, RwLockReadGuard, RwLockWriteGuard, SemaphorePermit
// all declare:
pub const __sync_guard_marker = {};
```

The linter check (in `zinc check` or `zinc build` with warnings enabled):

```
warning: SyncGuard 'MutexGuard' is live across async suspension point
  --> src/server.zig:47:5
   |
44 |     const guard = mutex.lock();
   |                   ------------ guard created here
...
47 |     try conn.readLine(&buf);  // async suspension
   |     ^^^^^^^^^^^^^^^^^^^^^^^^ guard still live here
   |
   = note: holding a lock across a suspension point will deadlock if
           another task tries to acquire the same lock
   = help: release the guard before the suspension point
```

This is a lint pass over the function's control flow graph, not a type
system feature. It checks whether any local variable implementing
`isSyncGuard()` is live at any `async`/`await` point. No compiler
changes required — implementable as a Zinc-specific analysis pass.

#### 3. `std.mem.TaggedPtr(T, n_bits)` — typed tagged pointers

Tagged pointers store small integers in the low bits of a pointer,
exploiting the fact that aligned pointers have zero in their low
alignment bits. Zig supports this via convention — store as `usize`,
mask manually — but there is no typed wrapper and no enforcement that
the pointer type is actually aligned enough.

```zig
// std.mem.TaggedPtr(PointeeType, tag_bits)
// Requires: @alignOf(PointeeType) >= (1 << tag_bits)
// Verified at comptime for types with known alignment.

const TaggedNode = std.mem.TaggedPtr(*Node, 2);
// Compile error if Node is not at least 4-byte aligned.

var tp = TaggedNode.init(node_ptr, 0b01)
    requires @alignOf(Node) >= 4;   // contract, checked at debug runtime for
                                     // pointers from runtime allocation

const ptr: *Node = tp.ptr();        // strips tag bits, returns typed pointer
const tag: u2    = tp.tag();        // extracts 2-bit tag
tp.setTag(0b10);                     // replaces tag, preserves pointer
const raw: usize = tp.raw();        // exposes underlying usize if needed

// Atomic operations on tagged pointers
var atomic_tp = std.atomic.Value(TaggedNode).init(tp);
const old = atomic_tp.compareExchangeWeak(expected, desired, .acq_rel, .monotonic);
```

The implementation:

```zig
pub fn TaggedPtr(comptime T: type, comptime n_bits: comptime_int) type {
    comptime {
        if (@alignOf(T) < (@as(usize, 1) << n_bits))
            @compileError(std.fmt.comptimePrint(
                "TaggedPtr: {s} has alignment {d}, needs at least {d} for {d} tag bits",
                .{ @typeName(T), @alignOf(T), @as(usize, 1) << n_bits, n_bits }
            ));
    }
    const TagType = std.meta.Int(.unsigned, n_bits);
    const mask: usize = (@as(usize, 1) << n_bits) - 1;

    return struct {
        raw_: usize,

        pub fn init(ptr: T, tag: TagType) @This() {
            const addr = @intFromPtr(ptr);
            std.debug.assert(addr & mask == 0);  // debug: verify alignment
            return .{ .raw_ = addr | @as(usize, tag) };
        }
        pub fn ptr(self: @This()) T {
            return @ptrFromInt(self.raw_ & ~mask);
        }
        pub fn tag(self: @This()) TagType {
            return @truncate(self.raw_ & mask);
        }
        pub fn setTag(self: *@This(), t: TagType) void {
            self.raw_ = (self.raw_ & ~mask) | @as(usize, t);
        }
        pub fn raw(self: @This()) usize { return self.raw_; }
    };
}
```

Pure comptime struct. No compiler changes. Uses the `requires` contract
system naturally for the alignment debug check.

#### 4. Software Transactional Memory: `std.stm`

A CAS-based STM library. No language primitives. No compiler support. No
scheduler integration. Convention-enforced transactional semantics using
Zig's existing atomic builtins.

The limitation relative to Haskell's STM: Zig has no effect system, so
the compiler cannot enforce that `TVar` reads only happen inside
`atomically`. This is enforced by convention — `TVar.read()` returns an
error if called outside a transaction context — rather than by the type
system.

```zig
const stm = std.stm;

// Transactional variable — wraps any Copy-able type
var balance_a = stm.TVar(u64).init(1000);
var balance_b = stm.TVar(u64).init(500);

// Transaction: runs, retries on conflict, commits atomically
try stm.atomically(struct {
    fn run(tx: *stm.Tx) !void {
        const a = try tx.read(&balance_a);
        const b = try tx.read(&balance_b);
        if (a < 100) return error.InsufficientFunds;
        try tx.write(&balance_a, a - 100);
        try tx.write(&balance_b, b + 100);
    }
}.run);
```

- **`stm.TVar(T)`** — a transactional variable. Wraps a value and a version counter. Thread-safe reads outside transactions return a snapshot. Writes outside transactions are a runtime error in debug mode.
- **`stm.Tx`** — a transaction context. Records all reads (with version at read time) and all intended writes. `tx.read()` reads the current value and logs the version. `tx.write()` records an intended write without applying it.
- **`stm.atomically(fn(*Tx) !void)`** — commits the transaction. Validates that all read versions are still current (no concurrent modification). If valid, applies all writes atomically using a global lock for the commit phase. If invalid, clears the transaction log and retries from the start.
- **`tx.retry()`** — abort the current transaction and block until at least one read `TVar` changes. Implemented with condition variables. Equivalent to Haskell's `retry`.

**`tx.orElse(first, second)`** — try `first`; if it calls `retry()`, try
`second` instead. Composable alternative.

```zig
// Transfer with retry if insufficient funds
try stm.atomically(struct {
    fn run(tx: *stm.Tx) !void {
        const balance = try tx.read(&account_balance);
        if (balance < amount) {
            tx.retry();  // block until balance changes
        }
        try tx.write(&account_balance, balance - amount);
    }
}.run);
```

**Performance:** STM has higher overhead than a well-designed mutex for
low-contention workloads. It shines for composability — you can compose
two transactions without either knowing about the other's locking strategy.
For raw performance use `std.Thread.Mutex` or `std.atomic.Value`. For
correct composition of complex shared state, use `std.stm`.

**Implementation:** Pure Zig stdlib. Uses `std.atomic.Value` for version
counters, `std.Thread.Mutex` for commit-phase serialization, and
`std.Thread.Condition` for `retry()` blocking. Approximately 800–1,200
lines. No compiler changes required.

### Hardware Transactional Memory — platform namespaces

Hardware TM (Intel RTM/TSX, ARM TME, IBM POWER HTM) lives in
platform-specific namespaces. Not in core stdlib. Using these namespaces
is an explicit declaration of non-portability.

```zig
// x86-64 only
const htm = @import("std").os.arch.x86_64.htm;

fn transferHtm(from: *u64, to: *u64, amount: u64) bool {
    switch (htm.begin()) {
        .started => {
            if (from.* < amount) htm.abort(1);
            from.* -= amount;
            to.*   += amount;
            htm.commit();
            return true;
        },
        .aborted  => return false,
        .unsupported => return false,
    }
}
// Caller must provide a non-HTM fallback path.
// HTM never guarantees progress. This is not optional.
```

The HTM shims are thin wrappers over the relevant intrinsics (`XBEGIN`,
`XEND`, `XABORT` on x86; `TSTART`, `TCOMMIT`, `TCANCEL` on ARM). They
provide a typed result enum and nothing else. Retry logic, fallback paths,
and conflict avoidance are the caller's responsibility.

Intel TSX was disabled by microcode on most CPUs due to speculative
execution vulnerabilities (TAA, MDS). Any code using
`std.os.arch.x86_64.htm` should check `htm.isSupported()` at runtime.
The shim returns `.unsupported` on CPUs where TSX has been disabled.

---

## Binary Layout Declarations

### What Zig already has

`packed struct` packs fields with no padding in declaration order, LSB-first.
`@bitSizeOf` queries sizes at comptime. `std.mem.readInt`/`writeInt` handle
endianness explicitly. Arbitrary-width integer types (`u3`, `u24`, etc.) give
you the building blocks.

This covers roughly 70% of what Ada's representation clauses do. What is
missing: bit-position declarations, total-size assertions, per-field byte
order, gap/overlap detection, and auto-generated codecs.

### The `layout` block

A `layout` block is a separate declaration attached to a struct. It specifies
the physical binary representation and causes the compiler to generate
read/write/view functions.

```zig
const StatusRegister = packed struct {
    ready:    bool,
    mode:     u3,
    reserved: u4,
    error_:   bool,
    _pad:     u7,
};

layout StatusRegister {
    .total_size = 16,           // bits — compile error if fields don't sum here
    .byte_order = .little,
    .bit_order  = .lsb_first,

    .ready    = .{ .byte = 0, .bits = 0..0 },
    .mode     = .{ .byte = 0, .bits = 1..3 },
    .reserved = .{ .byte = 0, .bits = 4..7 },
    .error_   = .{ .byte = 1, .bits = 0..0 },
    ._pad     = .{ .byte = 1, .bits = 1..7 },
}
```

### Compiler verifications

The compiler verifies at compile time:

- Every declared bit position is within `[0, total_size)`.
- No two fields overlap — any overlapping ranges are a compile error naming
  the exact conflicting bits.
- Every bit in `[0, total_size)` is accounted for — gaps are a compile error
  naming the unaccounted bit range.
- Each field's declared bit width is compatible with its type — a `u3` field
  in a 2-bit position is a compile error.
- Multi-byte fields on non-natural alignment require an explicit
  `.unaligned = true` marker — missing marker is a compile error.

```zig
// Compile error example:
layout Broken {
    .total_size = 16,
    .a = .{ .byte = 0, .bits = 0..3 },
    .b = .{ .byte = 0, .bits = 4..7 },
    // byte 1 unaccounted for
}
// error: layout 'Broken': bits 8..15 are unaccounted for
//        declare a padding field or reduce total_size to 8
```

### Padding fields

Fields whose names start with `_pad` or `_reserved` are padding:

- Inaccessible from user code
- Zeroed in generated `write()` codec
- Ignored (but verified present) in `read()` codec

Anonymous padding without a struct field uses the `_` entry in the layout:

```zig
layout Packet {
    .total_size = 32,
    .version  = .{ .byte = 0, .bits = 0..3 },
    ._        = .{ .byte = 0, .bits = 4..7 },   // anonymous pad
    .length   = .{ .bytes = 1..2 },
    ._        = .{ .byte = 3 },                  // anonymous pad
}
```

### Generated codec

For each `layout` declaration the compiler generates three functions:

- **`TypeName.read(bytes: []const u8) TypeName`** — extracts fields from raw bytes per the layout. Applies byte/bit order conversion. Verifies the input slice is long enough. Returns the typed struct.
- **`TypeName.write(self: TypeName, buf: []u8) void`** — packs fields into raw bytes per the layout. Applies byte/bit order conversion. Zeroes all padding fields. Asserts `buf.len >= total_size / 8`.
- **`TypeName.view(ptr: *volatile [N]u8) TypeNameView`** — returns a zero-copy view over memory-mapped bytes. Field accessors (`view.ready()`, `view.set_mode(3)`) read and write individual fields using volatile loads and stores. Suitable for MMIO register access.

```zig
// Read from network packet
const hdr = EthernetHeader.read(packet_bytes);
std.debug.print("ethertype: 0x{x}\n", .{hdr.ethertype});

// Write to buffer
var buf: [14]u8 = undefined;
EthernetHeader.write(.{
    .dst_mac   = dest,
    .src_mac   = src,
    .ethertype = 0x0800,
}, &buf);

// Memory-mapped I/O
const reg = StatusRegister.view(@as(*volatile [2]u8, @ptrCast(MMIO_BASE)));
if (reg.ready()) {
    reg.set_mode(3);
}
```

### Byte order

Layout-level byte order applies to all multi-byte fields unless overridden:

```zig
layout EthernetHeader {
    .total_size = 112,
    .byte_order = .big,   // network byte order

    .dst_mac   = .{ .bytes = 0..5  },
    .src_mac   = .{ .bytes = 6..11 },
    .ethertype = .{ .bytes = 12..13 },
}
```

Per-field override for genuinely mixed-endian hardware:

```zig
layout WeirdRegister {
    .total_size = 32,
    .upper = .{ .bytes = 0..1, .byte_order = .big    },
    .lower = .{ .bytes = 2..3, .byte_order = .little },
}
```

Byte order values: `.big`, `.little`, `.native`.
Bit order values: `.lsb_first` (default, x86 convention), `.msb_first`.

### Multi-byte field spanning

A field can span multiple bytes by specifying a byte range:

```zig
layout IpHeader {
    .total_size = 160,
    .byte_order = .big,

    .version  = .{ .byte = 0, .bits = 0..3  },
    .ihl      = .{ .byte = 0, .bits = 4..7  },
    .tos      = .{ .byte = 1               },
    .length   = .{ .bytes = 2..3           },
    .id       = .{ .bytes = 4..5           },
    .flags    = .{ .byte = 6, .bits = 0..2 },
    .frag_off = .{ .byte = 6, .bits = 3..7,
                   .byte = 7              },   // spans two bytes
    .ttl      = .{ .byte = 8               },
    .protocol = .{ .byte = 9               },
    .checksum = .{ .bytes = 10..11         },
    .src      = .{ .bytes = 12..15         },
    .dst      = .{ .bytes = 16..19         },
}
```

### Implementation cost

Pure code generation. No new type system concepts. The verification is
arithmetic over ranges. The codec generation is bit manipulation with
well-defined semantics.

Estimated compiler delta: 4,000–6,000 lines.

---

## Extended Numeric Types

### What Zig already has — and it is more than most languages

Zig has arbitrary-width integers as first-class types: `u1` through `u65535`,
`i1` through `i65535`. Not library types. Not macros. Real compiler-supported
types. `u3`, `u24`, `u48`, `u127` — all valid Zig types today. This is
Zig's killer feature for systems and embedded work and Zinc inherits it
unchanged.

Zig also has `f16`, `f32`, `f64`, `f80` (x87 extended), and `f128` (quad
precision). Five float types, all first-class. `comptime_int` and
`comptime_float` provide arbitrary precision at compile time.

So Zig's integer story is essentially complete. The gaps are in floats
(missing `bf16`, missing decimal floats), fixed-point (no stdlib type),
and exact arithmetic (`BigDecimal`, `Rational` not in stdlib).

### `bf16` — brain float 16

```zig
const x: bf16 = 3.14;
const y: f32  = x;              // lossless: bf16 is truncated f32
const z: bf16 = @floatCast(y);  // lossy: truncates lower 16 mantissa bits
```

Brain float 16 uses the same exponent range as `f32` (8 exponent bits) but
only 7 mantissa bits. The result: the same numeric range as `f32` with less
precision. The tradeoff makes sense for ML training where gradient updates
tolerate low precision but need to survive the same dynamic range as `f32`.

Hardware support: Google TPU (all generations), Intel AMX (Sapphire Rapids+),
NVIDIA A100+, AMD MI200+, Apple Silicon (ANE). Soft-float on all other
targets.

**`f32` ↔ `bf16` conversion is trivially cheap** because `bf16` is simply
`f32` with the lower 16 bits dropped. Converting `f32` to `bf16` is a
right-shift. Converting `bf16` to `f32` is a left-shift (zero-extend). The
compiler generates this without calling any library.

Literal suffix: `3.14bf16`

**`bf16` and `f16` are distinct types.** `f16` has more mantissa precision
but a smaller exponent range. They are not interchangeable. Converting between
them requires passing through `f32`.

---

### Decimal floats: `d32`, `d64`, `d128`

IEEE 754-2008 decimal floating-point. The defining property:

```zig
// Binary float: wrong
const a: f64 = 0.1 + 0.2;
std.debug.assert(a == 0.3);   // FAILS — binary rounding

// Decimal float: correct
const b: d64 = 0.1 + 0.2;
std.debug.assert(b == 0.3);   // passes — exact decimal arithmetic
```

Decimal floats are for financial calculations, accounting, tax, and any
domain where decimal fractional values must be exact. They are not for
scientific computing — use binary floats there.

| Type | Standard | Significant digits | Exponent range |
|------|----------|--------------------|----------------|
| `d32` | IEEE 754-2008 | 7 | −95 … +96 |
| `d64` | IEEE 754-2008 | 16 | −383 … +384 |
| `d128` | IEEE 754-2008 | 34 | −6143 … +6144 |

`d64` is the standard choice for financial work. 16 significant digits handles
all practical monetary values.

```zig
const price:    d64 = 1234567.89;
const tax_rate: d64 = 0.0825;
const tax:      d64 = price * tax_rate;       // exact
const total:    d64 = price + tax;            // exact
const cents:    i64 = @intFromDecimal(total * 100.0, .round_half_even);
```

Literal suffix: `0.1d32`, `0.1d64`, `0.1d128`.

**Critical implementation detail:** The lexer stores decimal literals as
their original string representation and converts to IEEE 754-2008 decimal
encoding. The literal `0.1d64` is never converted to binary float at any
point. If you write `0.1d64` you get the exact decimal value 0.1.

This is different from `@as(d64, 0.1)` which would convert the binary float
`0.1` (which is `0.1000000000000000055511...`) to decimal. Do not do that.
Always use decimal literals for decimal values.

**Operations:** Arithmetic operators `+`, `-`, `*`, `/` work on decimal
floats. Comparison operators work. Transcendental functions (`sin`, `exp`,
etc.) are not defined on decimal types — convert to `f64` first if you
need them. Decimal floats are for exact decimal arithmetic, not scientific
computing.

**Rounding modes** are explicit, not ambient:

```zig
const x: d64 = 2.5;
const a = std.math.decimal.round(x, .half_even);    // 2.0 (banker's rounding)
const b = std.math.decimal.round(x, .half_away);    // 3.0 (school rounding)
const c = std.math.decimal.round(x, .toward_zero);  // 2.0 (truncation)
```

Default arithmetic rounding is `half_even` (IEEE 754-2008 default).

**Hardware support:** IBM POWER6+ and IBM z10+ have native hardware decimal
float. All other targets use the Intel DFP Math Library or equivalent
software implementation. The compiler links this automatically when `d32`,
`d64`, or `d128` types are used.

**Performance:** Software decimal float is roughly 10–50× slower than
hardware binary float. This is the correct tradeoff for financial code
where correctness is non-negotiable.

---

### Fixed-point: `Fixed(I, F)` and `UFixed(I, F)`

Fixed-point arithmetic represents fractional values as scaled integers.
Zero hardware overhead — maps directly to integer instructions. No rounding
during arithmetic except division.

```zig
// Fixed(integer_bits, fractional_bits) — signed
// UFixed(integer_bits, fractional_bits) — unsigned

const Q16_16 = Fixed(16, 16);   // 32-bit, range ±32767.9999..., 1/65536 precision
const Q1_15  = Fixed(1, 15);    // 16-bit, range [-1, 1), standard audio DSP
const Q8_8   = Fixed(8, 8);     // 16-bit, range ±127.996..., 1/256 precision
const UQ8_8  = UFixed(8, 8);    // 16-bit unsigned, range [0, 255.996...)

var x: Q16_16 = 3.75;     // compile-time exact conversion from float literal
var y: Q16_16 = 1.5;
var z: Q16_16 = x + y;    // 5.25, exact, uses integer ADD
```

The value represented is `stored_integer / 2^F`. Stored in the smallest
containing integer type:

| Total bits (I+F) | Storage type |
|-----------------|--------------|
| 1–8 | `i8` / `u8` |
| 9–16 | `i16` / `u16` |
| 17–32 | `i32` / `u32` |
| 33–64 | `i64` / `u64` |
| 65–128 | `i128` / `u128` |

**Arithmetic:**

```zig
var a: Q16_16 = 3.75;
var b: Q16_16 = 1.5;

// Addition/subtraction: exact, same Q format
var sum: Q16_16 = a + b;    // 5.25

// Multiplication: result has doubled fractional bits — must rescale
// mul() returns Fixed(I*2, F*2) to preserve precision
var wide = a.mul(b);           // Fixed(32, 32): 5.625 exactly
var prod: Q16_16 = wide.rescale(Q16_16);  // back to Q16_16

// Or use mulSaturate for one-step saturating multiply
var sat: Q16_16 = a.mulSaturate(b);  // saturates instead of overflowing

// Widening multiply — critical for DSP chains
var wide2: Fixed(32, 32) = a.mulWide(b);

// Division
var quot: Q16_16 = a.div(b);  // b must be non-zero
```

**Audio DSP example:**

```zig
// FIR filter — standard embedded audio pattern
fn firFilter(samples: []const i16, coeffs: []const Q1_15) i16
    requires samples.len == coeffs.len
{
    var acc: Fixed(32, 30) = Fixed(32, 30).zero();  // accumulator with headroom
    for (samples, coeffs) |s, c| {
        const sample = Q1_15.fromRaw(s);
        acc = acc.addWide(sample.mulWide(c));
    }
    return acc.rescale(Q1_15).toRaw();
}
```

**`Fixed` is a standard library type, not a compiler primitive.** It is
a `comptime`-parameterized struct in `std.math.fixed`, consistent with how
Zig handles `std.ArrayList(T)` and other parameterized types. No compiler
changes required.

Common aliases in `std.math.fixed`:

```zig
pub const Q8_8   = Fixed(8, 8);
pub const Q16_16 = Fixed(16, 16);
pub const Q32_32 = Fixed(32, 32);
pub const Q1_15  = Fixed(1, 15);
pub const Q1_31  = Fixed(1, 31);
pub const Q0_15  = UFixed(0, 16);
pub const Q0_31  = UFixed(0, 32);
```

---

### `BigDecimal` and `Rational` in stdlib

Zig already has `std.math.big.Int` (arbitrary precision integer). Zinc adds
`std.math.big.Decimal` and `std.math.big.Rational`.

**`BigDecimal`** — arbitrary precision decimal arithmetic. Stored as
`BigInt × 10^scale`. Exact decimal arithmetic with no precision limit.
Uses the ambient allocator.

```zig
const BigDecimal = std.math.big.Decimal;

var x = try BigDecimal.parse("3.141592653589793238462643383279", allocator);
defer x.deinit();

var y = try x.mul(x, allocator);   // exact to full precision
defer y.deinit();

// Rounding only when explicitly requested
var z = try x.sqrt(50, allocator); // 50 significant digits
defer z.deinit();
```

**`Rational`** — exact rational arithmetic. Stored as `BigInt / BigUint`
in lowest terms. Never rounds.

```zig
const Rational = std.math.big.Rational;

var one_third = try Rational.init(1, 3, allocator);
defer one_third.deinit();

var two_thirds = try one_third.add(one_third, allocator);
defer two_thirds.deinit();

var one = try two_thirds.add(one_third, allocator);
defer one.deinit();
// one.numerator == 1, one.denominator == 1 — always reduced to lowest terms

// Series that binary float gets wrong:
var sum = try Rational.zero(allocator);
var i: u32 = 1;
while (i <= 100) : (i += 1) {
    const term = try Rational.init(1, i * i, allocator);
    sum = try sum.add(term, allocator);
}
// exact sum of 1/n² for n=1..100
```

Both types use `using(allocator)` naturally.

---

### Numeric comptime interfaces

Structured replacements for ad-hoc `@typeInfo` checks in generic functions.
Defined in `std.meta` as comptime validation functions.

```zig
// Currently in Zig:
fn popcount(value: anytype) u32 {
    comptime {
        if (@typeInfo(@TypeOf(value)) != .Int)
            @compileError("popcount requires an integer type");
    }
    return @popCount(value);
}

// In Zinc with numeric interfaces:
fn popcount(value: std.meta.Integer) u32 {
    return @popCount(value);
}
```

The interfaces:

```zig
// In std.meta:

/// Any integer type: u1..u65535, i1..i65535, usize, isize, comptime_int
pub fn Integer(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Int, .ComptimeInt => {},
        else => @compileError(@typeName(T) ++ " is not an integer type"),
    }
}

/// Unsigned integer only
pub fn UnsignedInt(comptime T: type) void {
    const info = @typeInfo(T);
    if (info != .Int or info.Int.signedness != .unsigned)
        @compileError(@typeName(T) ++ " is not an unsigned integer");
}

/// Signed integer only
pub fn SignedInt(comptime T: type) void {
    const info = @typeInfo(T);
    if (info != .Int or info.Int.signedness != .signed)
        @compileError(@typeName(T) ++ " is not a signed integer");
}

/// Any binary float type: f16, bf16, f32, f64, f80, f128
pub fn Float(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Float, .ComptimeFloat => {},
        else => @compileError(@typeName(T) ++ " is not a float type"),
    }
}

/// Any decimal float type: d32, d64, d128
pub fn DecimalFloat(comptime T: type) void {
    switch (T) {
        d32, d64, d128 => {},
        else => @compileError(@typeName(T) ++ " is not a decimal float type"),
    }
}

/// Any numeric type: integer, float, or decimal float
pub fn Numeric(comptime T: type) void {
    switch (@typeInfo(T)) {
        .Int, .Float, .ComptimeInt, .ComptimeFloat => {},
        else => switch (T) {
            d32, d64, d128 => {},
            else => @compileError(@typeName(T) ++ " is not a numeric type"),
        },
    }
}
```

Used as parameter constraints — the function only compiles when called with
a conforming type:

```zig
fn clamp(value: std.meta.Numeric, lo: @TypeOf(value), hi: @TypeOf(value))
    @TypeOf(value)
    requires lo <= hi
{
    if (value < lo) return lo;
    if (value > hi) return hi;
    return value;
}

fn abs(value: std.meta.SignedInt) @TypeOf(value) {
    return if (value < 0) -value else value;
}

fn bitWidth(comptime T: std.meta.Integer) comptime_int {
    return @typeInfo(T).Int.bits;
}
```

**This is a pure standard library addition.** No compiler changes required.
The `std.meta.Integer` usage as a parameter type is sugar over comptime
function calls that Zig already supports.

---

### Literal suffix summary

| Suffix | Type | Example |
|--------|------|---------|
| `u8`..`u65535` | Unsigned integer | `42u24`, `255u8` |
| `i8`..`i65535` | Signed integer | `-1i32`, `127i7` |
| `usize`, `isize` | Pointer-width integer | `0usize` |
| `f16` | IEEE half precision | `3.14f16` |
| `bf16` | Brain float 16 | `3.14bf16` |
| `f32` | IEEE single precision | `3.14f32` |
| `f64` | IEEE double precision | `3.14f64` |
| `f80` | x87 extended precision | `3.14f80` |
| `f128` | IEEE quad precision | `3.14f128` |
| `d32` | IEEE decimal32 | `0.1d32` |
| `d64` | IEEE decimal64 | `0.1d64` |
| `d128` | IEEE decimal128 | `0.1d128` |

Untyped integer literals are `comptime_int` (unchanged from Zig).
Untyped float literals are `comptime_float` (unchanged from Zig).
Decimal literals with a `d` suffix are always stored as exact decimal
strings by the lexer — never converted through binary float.

---

### What Zig already has that Zinc inherits unchanged

This section exists to prevent confusion. Zinc does not touch:

- Arbitrary-width integers `u1`..`u65535`, `i1`..`i65535` — already in Zig,
  already better than most languages including Ferrum
- `f16`, `f32`, `f64`, `f80`, `f128` — already in Zig
- `comptime_int`, `comptime_float` — already in Zig, arbitrary precision
- `std.math.big.Int` — already in Zig stdlib
- All integer arithmetic operators and builtins — unchanged

The gap Zinc fills is specifically: `bf16`, decimal floats, `Fixed(I,F)`,
`BigDecimal`, `Rational`, and the numeric comptime interface conventions.

---

## Implementation Cost Summary

**Ambient allocators:** 3,000–5,000 lines. Lexical scoping pass for `using`
declarations. No new type system concepts.

**Contracts:** 5,000–8,000 lines. AST transformation pass. `assert` insertion,
`old()` snapshot generation, invariant checks. No new type system concepts.

**Binary layout declarations:** 4,000–6,000 lines. Layout verification pass
(arithmetic on ranges), codec generation pass (bit manipulation). No new
type system concepts.

**`bf16`:** 500–1,000 lines. Follow the same path as `f16`. Backend
integration for targets with hardware support, soft-float linkage elsewhere.

**Decimal floats `d32`/`d64`/`d128`:** 2,000–4,000 lines in the compiler
(new types through the type system and codegen), plus the Intel DFP Math
Library or equivalent as a runtime dependency. The lexer extension for
decimal literals is small but critical — decimal strings must never touch
binary float conversion.

**`Fixed(I, F)` and `UFixed(I, F)`:** 1,000–2,000 lines. Pure stdlib
addition, no compiler changes. Comptime-parameterized struct using existing
Zig machinery.

**`BigDecimal`, `Rational`:** 3,000–5,000 lines. Pure stdlib additions
building on the existing `std.math.big.Int`. No compiler changes.

**Numeric comptime interfaces:** 200–400 lines. Pure stdlib addition in
`std.meta`. No compiler changes.

**`std.atomic.Value(T)` improvements:** 200–400 lines. Pure stdlib additions.
`fetchMax`, `fetchMin`, `compareExchangeLoop`. No compiler changes.

**`SyncGuard` marker and linter pass:** 500–1,000 lines. Comptime interface
declaration in stdlib plus a control-flow analysis pass in the linter.
No compiler changes to the type system.

**`std.mem.TaggedPtr(T, n_bits)`:** 200–300 lines. Pure comptime struct.
No compiler changes.

**`std.stm`:** 800–1,200 lines. Pure Zig using existing atomic builtins,
Mutex, and Condition. No compiler changes.

**Platform HTM shims:** 300–500 lines per platform. Thin intrinsic wrappers.
No compiler changes.

**Total compiler delta:** Roughly 15,000–24,000 lines on top of the Zig
compiler. Standard library additions are roughly 8,000–13,000 lines.
No new type system concepts anywhere in the concurrency additions.
No research problems anywhere. Engineering work throughout.

---

## Relationship to Ferrum

Ferrum has ambient allocators and design by contract too — along with a
full effect system, region inference, a borrow checker, GADTs, a proof
system, and much more.

Zinc is not Ferrum-lite. Zinc is Zig-plus. The design philosophy is
different: Zig values simplicity and explicitness above all. Zinc adds the
minimum to fix specific ergonomic gaps without disturbing the rest of the
design.

If Zig is the right level of abstraction for your problem, Zinc is the
right language. If you need the full Ferrum machinery — effect tracking,
verified invariants, the proof system, the borrow checker — use Ferrum.

The two languages share ideas but not a type system, not a runtime, and
not a standard library.

---

## Open Questions

**Should `with` be a keyword or a library function?**
Keyword is cleaner syntactically. Library function is more Zig-like
(everything is explicit). Current leaning: keyword, because it changes
name resolution semantics in a way a library function cannot.

**Should `requires(always)` be the default for security-critical annotations?**
Probably not default — opt-in is more explicit. A future linter rule
could flag functions that look security-critical without `requires(always)`.

**Should the compiler warn on `old()` expressions that are expensive to copy?**
Yes. Snapshotting a large struct in a hot loop will silently destroy
performance in debug mode. The compiler should warn when `old()` copies
more than N bytes.

**Should contracts be visible in generated documentation?**
Yes, always. A function's `requires` and `ensures` clauses are the most
important documentation it has. They should appear prominently in `zinc doc`
output, more prominent than the description comment.

**Should `@as(d64, some_f64_value)` be a compile error?**
Leaning yes. Converting a binary float to decimal float almost always
indicates a bug — the programmer meant to write a decimal literal but
wrote a binary float literal instead. The cast should require an explicit
`std.math.decimal.fromBinaryFloat(val)` call that makes the lossiness
visible.

**Should `Fixed(I, F)` use operator overloading or explicit methods?**
Zig does not have operator overloading for user types. `Fixed` arithmetic
uses explicit methods (`.add()`, `.mul()`, `.rescale()`). This is consistent
with Zig's philosophy — arithmetic that might overflow or require rescaling
should not look like trivial arithmetic. The verbosity is intentional.

**Should `d64` decimal literals without a suffix be supported in financial
code contexts?**
A `#[decimal_context]` attribute on a block or function that makes untyped
decimal literals (`0.1`, `19.99`) resolve to `d64` rather than `comptime_float`
would reduce annotation burden in financial code. Not in v1 — requires
context-sensitive literal resolution. File for future consideration.

**Should the layout `view()` type support atomic field access for
multi-threaded MMIO?**
Some hardware registers require atomic read-modify-write. The current
`view()` generates plain volatile accesses. An `.atomic = true` field
annotation in the layout block that generates atomic load/store for
specific fields would cover this. Not in v1 but the layout block syntax
has room for it.

**Should `std.stm` enforce transactional context via the type system?**
Haskell's STM enforces that `TVar` reads only happen inside `atomically`
through the type system (`STM a` monad). Zig has no equivalent mechanism.
Zinc's STM enforces this at runtime in debug mode — `tx.read()` panics if
called without a transaction context. Full compile-time enforcement would
require either an effect system or a significant stdlib design change.
Accepted limitation for v1.

**Should `TaggedPtr` support tags wider than the alignment bits?**
Some use cases want to store more bits than the pointer alignment provides,
using a separate word alongside the pointer. This is a different pattern
(fat pointer) rather than tagged pointer. Out of scope for `TaggedPtr`.
A separate `std.mem.FatPtr(T, MetaType)` could address this.

**Should hardware TM shims expose abort codes?**
Intel RTM's `XABORT` instruction takes an 8-bit immediate. ARM TME's
`TCANCEL` takes a 64-bit immediate. These codes are visible in the abort
status. The current shim exposes them as an opaque integer. A typed abort
reason enum per platform would be cleaner but requires per-platform
maintenance. Deferred to platform maintainers.

**Should `SyncGuard` warnings be errors by default?**
Holding a lock across a suspension point is almost always wrong. Making it
an error rather than a warning would be stricter but might break valid
patterns (e.g., a guard held across a `@asyncCall` where the programmer
has verified no contention is possible). Current position: warning by
default, `--deny sync-guard-suspension` to promote to error.

---

*End of Zinc Language Specification 0.3*
