# Introduction to Proofs in Ferrum

**Audience:** Programmers who know C and Python, new to formal verification

---

## The Problem Proofs Solve

Here's a binary search in C:

```c
int binary_search(int* arr, int len, int target) {
    int lo = 0;
    int hi = len - 1;
    while (lo <= hi) {
        int mid = (lo + hi) / 2;  // BUG: integer overflow
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

This code shipped in the JDK for nine years before anyone noticed the bug. `(lo + hi)` overflows when both are large. The fix is `lo + (hi - lo) / 2`.

Testing didn't catch it because:
- It only fails with arrays larger than a billion elements
- Most tests use small arrays
- The failure is silent corruption, not a crash

Code review didn't catch it because the code *looks* right. The bug is in what you're not thinking about.

This is the class of bug that proofs catch: **things that are true for all inputs, not just the ones you tested.**

---

## What Proofs Are (and Aren't)

A proof is a statement about your code that the compiler verifies mathematically. Not by running tests. Not by fuzzing. By reasoning about the code itself.

In Ferrum, proofs come in layers. You use the layer you need.

---

## Layer 1: Contracts

Contracts are preconditions and postconditions. The compiler checks them where it can; they become runtime checks otherwise.

```ferrum
fn binary_search(arr: &[i32], target: i32): Option[usize]
    requires arr.is_sorted()
    ensures match result {
        Some(i) => arr[i] == target,
        None => !arr.contains(target),
    }
{
    // implementation
}
```

**`requires`** — what must be true when you call the function. If you call `binary_search` on an unsorted array, that's your bug.

**`ensures`** — what the function guarantees about its result. If it returns `Some(i)`, then `arr[i] == target`. If it returns `None`, the target isn't in the array.

### Comparing to C

In C, you'd write:

```c
// Precondition: arr must be sorted
// Postcondition: returns index of target, or -1 if not found
int binary_search(int* arr, int len, int target);
```

Problems:
1. Comments are not checked
2. "Sorted" is vague — ascending? descending? by what comparator?
3. The postcondition doesn't say the returned index actually contains the target
4. Nothing stops you from passing an unsorted array

In Ferrum, `requires arr.is_sorted()` is checked. The compiler may verify it statically, or insert a runtime check, or require you to prove it at the call site.

### Comparing to Python

Python has assertions:

```python
def binary_search(arr, target):
    assert arr == sorted(arr), "arr must be sorted"
    # ...
```

Problems:
1. Assertions run at runtime — you pay the cost
2. `arr == sorted(arr)` is O(n log n) — you might skip it in production
3. Assertions are often disabled with `-O`
4. No postcondition at all

Ferrum contracts are part of the type system. They're documentation, specification, and verification in one.

---

## Layer 2: SMT Verification

For simple arithmetic contracts, the compiler can verify them automatically using an SMT solver.

```ferrum
fn abs(x: i32): i32
    ensures result >= 0
    ensures result == x || result == -x
{
    if x >= 0 { x } else { -x }
}
```

The compiler feeds this to an SMT solver (think: a very good equation solver) and proves the postconditions hold for all possible inputs. No testing needed. It's mathematically certain.

### What SMT Can Handle

- Arithmetic: `x + y > z`, `a * b <= c`
- Comparisons: `result >= 0`
- Array bounds: `index < arr.len()`
- Simple conditionals: `if x > 0 then result > 0`

### What SMT Can't Handle

- Loops (usually)
- Recursive functions
- Complex data structures
- "This sorting algorithm actually sorts"

When SMT can't verify something, the contract becomes a runtime check or you write a proof.

---

## Layer 3: Proof Functions

For properties SMT can't handle, you write a proof — a function that exists only to convince the compiler.

```ferrum
/// Proof that reversing a list twice gives back the original
proof fn reverse_involutive[T](xs: &[T])
    ensures xs.reverse().reverse() == xs
{
    match xs {
        [] => {}  // empty list: trivial
        [head, ..tail] => {
            // inductive case: assume it works for tail, prove for whole list
            reverse_involutive(tail);
            // ... algebraic reasoning the compiler checks
        }
    }
}
```

**`proof fn`** — this function has no runtime cost. It's erased after verification. It exists only to guide the compiler's reasoning.

Most programmers never write proof functions. They use contracts, and the proofs live in libraries.

---

## The Overflow Bug, Fixed

Back to binary search. Here's how Ferrum prevents the overflow:

```ferrum
fn binary_search(arr: &[i32], target: i32): Option[usize]
    requires arr.is_sorted()
{
    let mut lo: usize = 0;
    let mut hi: usize = arr.len();

    while lo < hi {
        // This line is checked for overflow
        let mid = lo + (hi - lo) / 2;

        match arr[mid].cmp(&target) {
            Less => lo = mid + 1,
            Greater => hi = mid,
            Equal => return Some(mid),
        }
    }
    None
}
```

The compiler knows:
- `lo` and `hi` are `usize` (unsigned)
- `hi - lo` can't underflow because `lo < hi` (from the loop condition)
- `lo + (hi - lo) / 2` can't overflow because `(hi - lo) / 2 <= hi - lo`
- `mid < arr.len()` so `arr[mid]` is in bounds

These aren't runtime checks. The compiler proves them from the code structure.

If you wrote `(lo + hi) / 2`, the compiler would say:

```
error: potential overflow in addition
  --> search.fe:8:19
   |
 8 |         let mid = (lo + hi) / 2;
   |                   ^^^^^^^^^
   |
   = note: `lo + hi` may exceed usize::MAX when both are large
   = help: use `lo + (hi - lo) / 2` to avoid overflow
```

---

## Contracts as Documentation

Contracts serve as precise documentation:

```ferrum
/// Sorts the slice in ascending order.
fn sort[T: Ord](slice: &mut [T])
    ensures slice.is_sorted()
    ensures slice.is_permutation_of(old(slice))
{
    // ...
}
```

This says exactly what `sort` does:
1. The result is sorted
2. The result contains the same elements as the input (no elements lost or duplicated)

Compare to a typical docstring: "Sorts the slice." That doesn't tell you it's stable, or that it doesn't lose elements, or what order.

---

## What You Actually Write

Most code uses simple contracts or none at all:

```ferrum
// No contracts needed — types are enough
fn add(x: i32, y: i32): i32 {
    x + y
}

// Simple bounds check
fn get(arr: &[T], index: usize): Option[&T]
    requires index < arr.len()  // or just use arr.get() which returns Option
{
    Some(&arr[index])
}

// Domain constraint
fn sqrt(x: f64): f64
    requires x >= 0.0
    ensures result >= 0.0
    ensures (result * result - x).abs() < 1e-10
{
    // ...
}
```

You escalate to more complex contracts when:
- The function has subtle preconditions
- The correctness property isn't obvious
- You're writing a library others will depend on

You write proof functions when:
- SMT can't verify the contract
- You're implementing core data structures
- You need mathematical certainty (cryptography, aerospace, finance)

---

## The Layers

| Layer | What it is | Who uses it | Compiler does |
|-------|-----------|-------------|---------------|
| Types | `i32`, `Option[T]` | Everyone | Full verification |
| Contracts | `requires`, `ensures` | Library authors, careful code | SMT where possible, runtime checks otherwise |
| Proofs | `proof fn` | Data structure implementers, high-assurance code | Full verification, then erases |

Each layer is optional. You move up when your problem requires it.

---

## Runtime Checks When Needed

When the compiler can't prove a contract statically, it becomes a runtime check:

```ferrum
fn process(data: &[u8])
    requires data.len() >= 4
{
    let header = &data[0..4];
    // ...
}

// At the call site:
process(&input);  // compiler inserts: assert(input.len() >= 4)
```

You can control this:

```ferrum
@contracts(runtime = "debug")   // check in debug builds only
@contracts(runtime = "always")  // always check
@contracts(runtime = "never")   // trust the caller (unsafe territory)
```

---

## Summary

| Concept | C | Python | Ferrum |
|---------|---|--------|--------|
| Preconditions | Comments | `assert` (runtime) | `requires` (verified or checked) |
| Postconditions | Comments | None | `ensures` (verified or checked) |
| Overflow checks | None (UB) | Automatic (slow) | Verified at compile time |
| Bounds checks | None (UB) | Automatic (runtime) | Verified where possible |
| Mathematical proofs | Not possible | Not possible | `proof fn` |

Proofs in Ferrum aren't academic exercises. They're tools for catching bugs that testing misses — the overflow that happens at scale, the bounds error in the edge case, the invariant that holds "by construction" until someone changes the construction.

Most code uses types and simple contracts. That catches most bugs. When you need more certainty, the tools are there.

---

*See also: [Ferrum Language Reference](ferrum-language-reference.md) for complete verification specification.*
