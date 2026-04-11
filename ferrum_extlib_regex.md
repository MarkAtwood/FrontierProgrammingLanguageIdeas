# Ferrum Extended Library — regex

**Module:** `extlib.regex`
**Part of:** Extended Standard Library (not `std` or `core`)
**Companion:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 1. Overview and Rationale

### Why extlib, Not stdlib

The Ferrum standard library (`std`) contains modules that nearly every program needs: strings, collections, I/O, HTTP, and crypto primitives. Regular expressions do not meet this bar. Most systems programs never use them; parsers write grammars, protocols write state machines, and log processors use byte-pattern search. Putting regex in stdlib would add compile-time cost, binary size, and surface area to every Ferrum program regardless of need.

More importantly, regex has a safety property that requires deliberate design: **catastrophic backtracking**. Backtracking NFA and DFA engines can exhibit O(2^n) behavior on adversarial input — ReDoS (Regular Expression Denial of Service). This is a class of vulnerability that has caused real outages. Ferrum's stdlib design philosophy is that footguns belong either in `unsafe` or in modules that require explicit opt-in. A regex module that could silently expose programs to ReDoS belongs in the extended library, where its safety policy can be documented and enforced as a first-class contract.

### Safety Philosophy: NFA by Default

`extlib.regex` uses a **Thompson NFA** engine by default. The NFA engine provides:

- **O(n) worst-case time** in the length of the input string, regardless of pattern
- **No catastrophic backtracking** — ReDoS is structurally impossible with the default engine
- **Predictable latency** — suitable for processing untrusted input in servers and parsers

A DFA cache is layered on top for acceleration: when the NFA simulation produces a DFA state that has been seen before, the cache short-circuits future simulation. This accelerates common patterns without changing worst-case complexity.

Features that require backtracking — lookahead, lookbehind, and backreferences — are **opt-in** via `RegexBuilder.backtracking(true)`. Enabling backtracking emits a compiler warning, making the elevated risk visible at the call site. Programs that process untrusted input should not set this flag.

This design follows the approach of CCSP's `lib_ccsp_regex`: backtracking is never implicit, the safe path is the default path, and the elevated-risk path requires affirmative action.

---

## 2. Core Types

### 2.1 Regex

`Regex` is an immutable compiled regular expression. It is `Send + Sync`: multiple threads can share a reference to the same `Regex` without synchronization.

```ferrum
/// A compiled regular expression.
///
/// Compiled from a pattern string at construction time. Immutable after construction.
/// Thread-safe: impl Send + Sync.
type Regex { ... }

impl Regex {
    // Constructors — see Section 3
    // Matching API — see Section 4
    // Replacement API — see Section 5
    // Splitting API — see Section 7
}
```

### 2.2 RegexBuilder

`RegexBuilder` configures a `Regex` before compilation. All flags have safe defaults.

```ferrum
/// Builder for configuring a Regex before compilation.
type RegexBuilder {
    fn new(pattern: &str): Self

    /// Case-insensitive matching. Default: false.
    fn case_insensitive(self, yes: bool): Self

    /// `^` and `$` match line boundaries, not just string boundaries. Default: false.
    fn multiline(self, yes: bool): Self

    /// `.` matches `\n`. Default: false.
    fn dot_all(self, yes: bool): Self

    /// Enable Unicode mode: character classes and case folding are Unicode-aware.
    /// Default: true. Set false only for ASCII-only patterns on byte slices.
    fn unicode(self, yes: bool): Self

    /// Reject patterns whose NFA exceeds this many states. Default: 10_000.
    /// Prevents pathological patterns from consuming excessive memory.
    fn size_limit(self, bytes: usize): Self

    /// Limit the DFA cache to this many bytes. Default: 2_097_152 (2 MiB).
    /// When the cache is full, the engine falls back to NFA simulation.
    fn dfa_size_limit(self, bytes: usize): Self

    /// Opt in to backtracking. Enables lookahead, lookbehind, backreferences.
    ///
    /// WARNING: backtracking can exhibit O(2^n) behavior on adversarial input.
    /// Do not enable for programs that process untrusted input.
    /// Emits a compiler warning at the call site.
    fn backtracking(self, yes: bool): Self

    /// Compile the regex. Returns Err if the pattern is invalid or exceeds size_limit.
    fn build(self): Result[Regex, RegexError] ! Alloc
}
```

### 2.3 Match

`Match` represents a contiguous region of an input string that was matched. It borrows from the input string with lifetime `'input`.

```ferrum
/// A single match: a contiguous region of an input string.
/// Borrows from the input string; does not allocate.
type Match['input] {
    /// The matched substring.
    fn as_str(&self): &'input str

    /// The byte offset of the start of the match.
    fn start(&self): usize

    /// The byte offset one past the end of the match (exclusive).
    fn end(&self): usize

    /// The byte range of the match.
    fn range(&self): core.ops.Range[usize]
}
```

`Match` does not allocate. It is a pair of byte indices into the original input.

### 2.4 Captures

`Captures` holds the result of a successful capture match: the overall match plus zero or more captured groups, each a `Match` or `None` for non-participating groups.

```ferrum
/// The result of a capturing match.
/// Group 0 is always the overall match. Groups 1..n are capturing groups.
type Captures['input] {
    /// The overall match (group 0). Always present on success.
    fn get(&self, i: usize): Option[Match['input]]

    /// Access a named capture group by name.
    fn name(&self, name: &str): Option[Match['input]]

    /// The number of capture groups (including group 0).
    fn len(&self): usize

    /// Iterate over all capture groups. Non-participating groups yield None.
    fn iter(&self): impl Iterator[Item=Option[Match['input]]]

    /// The full match region (equivalent to self.get(0).unwrap()).
    fn full_match(&self): Match['input]
}

// Index by group number: captures[0] panics if group 0 is absent (i.e., no match),
// but Captures is only constructed on success, so group 0 is always present.
// Out-of-bounds index panics.
impl[I: Into[usize]] core.ops.Index[I] for Captures['input] {
    type Output = Match['input]
}
```

### 2.5 CaptureNames

`CaptureNames` is an iterator over the names of all capture groups in a compiled `Regex`. Unnamed groups yield `None`.

```ferrum
/// Iterator over capture group names. Unnamed groups yield None.
/// Indices correspond to capture group numbers (0-indexed).
type CaptureNames { ... }

impl Iterator for CaptureNames {
    type Item = Option[&'static str]
}
```

`Regex` exposes this via:

```ferrum
impl Regex {
    fn capture_names(&self): CaptureNames
    fn captures_len(&self): usize   // total number of capture groups (including group 0)
}
```

### 2.6 RegexSet

`RegexSet` compiles multiple patterns and matches them simultaneously against an input in a single pass. This is more efficient than compiling each pattern separately and running each in sequence.

```ferrum
/// A set of patterns matched simultaneously in a single pass over the input.
type RegexSet { ... }

impl RegexSet {
    /// Compile multiple patterns. Fails if any pattern is invalid.
    fn new(patterns: impl IntoIterator[Item=impl AsRef[str]]): Result[Self, RegexError] ! Alloc

    /// Returns a SetMatches indicating which patterns matched.
    fn matches(&self, text: &str): SetMatches

    /// True if any pattern matches.
    fn is_match(&self, text: &str): bool

    /// The number of patterns in the set.
    fn len(&self): usize

    /// The original pattern strings, in order.
    fn patterns(&self): &[String]
}

/// The result of a RegexSet match: which patterns matched.
type SetMatches { ... }

impl SetMatches {
    /// True if any pattern matched.
    fn matched_any(&self): bool

    /// True if pattern i matched.
    fn matched(&self, i: usize): bool

    /// Iterate over the indices of patterns that matched.
    fn iter(&self): impl Iterator[Item=usize]
}
```

`RegexSet` does not return the positions of matches — only which patterns matched. To find positions, compile each matching pattern individually and run `find` on it. This is a deliberate limitation that preserves the single-pass O(n) guarantee.

---

## 3. Compilation

### 3.1 Regex::new

```ferrum
impl Regex {
    /// Compile a pattern. Returns Err if the pattern is syntactically invalid
    /// or exceeds the default size limit.
    fn new(pattern: &str): Result[Self, RegexError] ! Alloc
}
```

This is the standard runtime compilation path. Use it when the pattern is not known at compile time.

### 3.2 Compile-Time Validation via regex()

Ferrum has no user-defined macros. However, when `regex(...)` is called with a **string literal**, the compiler validates the pattern at compile time and reports any syntax error as a compile error rather than a runtime `Err`. At runtime, the compiled state is initialized once and cached.

```ferrum
// pattern is a string literal — validated at compile time, error is a compile error
let re = regex("[a-z]+@[a-z]+\\.[a-z]{2,6}")

// pattern is a runtime string — equivalent to Regex.new(), can return Err
let pat: String = get_pattern_from_config()
let re = regex(pat)   // type: Result[Regex, RegexError]
```

`regex(...)` is a compiler intrinsic, not a user-callable function. Its return type depends on whether the argument is a literal:

| Argument | Return type | Failure mode |
|---|---|---|
| String literal | `Regex` | Compile error |
| Runtime `&str` or `String` | `Result[Regex, RegexError]` | Runtime `Err` |

This mirrors how `format(...)` works: a literal format string is validated at compile time; the compiler intrinsic has special handling for literal arguments.

### 3.3 RegexBuilder for Advanced Configuration

When default flags are insufficient, use `RegexBuilder`:

```ferrum
let re = RegexBuilder.new("(?i)foo|bar")
    .case_insensitive(true)
    .multiline(true)
    .size_limit(50_000)
    .build()?
```

All `RegexBuilder` methods return `Self`, enabling method chaining. `build()` is the only method that allocates and returns `Result`.

---

## 4. Matching API

All matching methods take `&str` (UTF-8). Byte-slice matching is available via the `bytes` sub-module (Section 11).

```ferrum
impl Regex {
    /// Returns true if the pattern matches anywhere in text.
    fn is_match(&self, text: &str): bool

    /// Returns the leftmost match, or None.
    fn find<'t>(&self, text: &'t str): Option[Match['t]]

    /// Returns an iterator over all non-overlapping matches, left to right.
    fn find_iter<'t>(&self, text: &'t str): impl Iterator[Item=Match['t]]

    /// Returns the captures for the leftmost match, or None.
    fn captures<'t>(&self, text: &'t str): Option[Captures['t]]

    /// Returns an iterator over captures for all non-overlapping matches.
    fn captures_iter<'t>(&self, text: &'t str): impl Iterator[Item=Captures['t]]

    /// Returns the end byte offset of the shortest match, or None.
    /// Useful when you only need to know whether a match exists and where
    /// it ends, without locating its start.
    fn shortest_match(&self, text: &str): Option[usize]

    /// Like is_match, but anchored: the pattern must match starting at byte
    /// offset `start`. Does not search the full string.
    fn is_match_at(&self, text: &str, start: usize): bool

    /// Like find, but anchored at byte offset `start`.
    fn find_at<'t>(&self, text: &'t str, start: usize): Option[Match['t]]
}
```

**Iteration semantics:** `find_iter` and `captures_iter` never return overlapping matches. After each match, the search resumes after the end of the previous match. An empty match at position `i` advances the cursor to `i+1` to prevent infinite loops.

**No panic on invalid UTF-8 positions:** All byte offsets returned by `Match` are guaranteed to be on UTF-8 character boundaries.

---

## 5. Replacement API

Replacement methods return a new owned `String`. They require `! Alloc` because they allocate the result.

```ferrum
impl Regex {
    /// Replace the leftmost match with replacement.
    /// If no match, returns a copy of text.
    fn replace<'t, R: Replacer>(&self, text: &'t str, rep: R): String ! Alloc

    /// Replace all non-overlapping matches with replacement.
    fn replace_all<'t, R: Replacer>(&self, text: &'t str, rep: R): String ! Alloc

    /// Replace the first n non-overlapping matches with replacement.
    fn replacen<'t, R: Replacer>(&self, text: &'t str, n: usize, rep: R): String ! Alloc
}
```

### 5.1 The Replacer Trait

```ferrum
/// Anything that can produce replacement text.
trait Replacer {
    fn replace_append(&mut self, caps: &Captures, dst: &mut String)
}
```

The library provides blanket implementations so callers rarely implement `Replacer` directly:

| Type | Behavior |
|---|---|
| `&str` | Literal replacement, with `$name` and `$1` substitution |
| `String` | Same as `&str` |
| `fn(&Captures): String` | Computed replacement |

### 5.2 Capture References in Replacement Strings

Within a `&str` replacement, the following substitutions are performed:

| Syntax | Replaced with |
|---|---|
| `$0` | The entire match (same as `${0}`) |
| `$1`, `$2`, … | Capture group by number |
| `$name` | Named capture group |
| `${name}` | Named capture group (unambiguous form) |
| `$$` | A literal `$` |

A non-participating group referenced by `$1` substitutes as an empty string. A reference to a group index that does not exist substitutes as an empty string (not an error).

```ferrum
let re = regex("(?P<year>\\d{4})-(?P<month>\\d{2})-(?P<day>\\d{2})")
let result = re.replace_all("2026-04-10", "$day/$month/$year")
// result == "10/04/2026"
```

### 5.3 Computed Replacements

```ferrum
let re = regex("\\b\\w+\\b")
let result = re.replace_all("hello world", fn(caps: &Captures): String {
    caps[0].as_str().to_ascii_uppercase()
})
// result == "HELLO WORLD"
```

---

## 6. Named Captures

Named capture groups use the `(?P<name>...)` syntax, following the Python/PCRE convention.

```ferrum
let re = regex("(?P<proto>https?)://(?P<host>[^/]+)(?P<path>/[^?]*)?")
let text = "https://example.com/index.html"

match re.captures(text) {
    None    => println("no match")
    Some(caps) => {
        let proto = caps.name("proto").map(fn(m) { m.as_str() }).unwrap_or("")
        let host  = caps.name("host").map(fn(m) { m.as_str() }).unwrap_or("")
        let path  = caps.name("path").map(fn(m) { m.as_str() }).unwrap_or("/")
        println("proto={} host={} path={}", proto, host, path)
    }
}
```

Named and numbered access are equivalent. `caps.name("host")` and `caps.get(2)` refer to the same group if `host` is group 2.

Group 0 is the overall match. `caps[0]` and `caps.get(0)` and `caps.full_match()` are all equivalent.

To enumerate all group names in a compiled pattern:

```ferrum
let re = regex("(?P<year>\\d{4})-(?P<month>\\d{2})")
for (i, name) in re.capture_names().enumerate() {
    match name {
        None       => println("group {} is unnamed", i)
        Some(name) => println("group {} is named '{}'", i, name)
    }
}
```

---

## 7. Splitting

```ferrum
impl Regex {
    /// Split text on each match. Returns an iterator of the substrings between matches.
    /// Captures within the pattern are not included in the output.
    fn split<'t>(&self, text: &'t str): impl Iterator[Item=&'t str]

    /// Split at most n times. The last element is the remainder.
    fn splitn<'t>(&self, text: &'t str, n: usize): impl Iterator[Item=&'t str]
}
```

If the pattern matches at the start of the string, the first element is an empty string. If it matches at the end, the last element is an empty string. This is consistent with how string splitting works elsewhere in the standard library.

```ferrum
let re = regex("\\s+")
let words: Vec[&str] = re.split("one   two\tthree").collect()
// words == ["one", "two", "three"]

let parts: Vec[&str] = re.splitn("a b c d", 3).collect()
// parts == ["a", "b", "c d"]
```

---

## 8. RegexSet

`RegexSet` matches multiple patterns in a single O(n) pass. It is more efficient than looping over a `Vec[Regex]` and is the correct tool for classifying input against a fixed set of patterns (routing tables, log classifiers, protocol demultiplexers).

```ferrum
let set = RegexSet.new([
    "\\d+",
    "[a-z]+",
    "[A-Z]+",
])?

let matches = set.matches("hello123")
// matches.matched(0) == true   (\d+ matched "123")
// matches.matched(1) == true   ([a-z]+ matched "hello")
// matches.matched(2) == false

for i in matches.iter() {
    println("pattern {} matched", i)
}
```

`RegexSet` does not return the positions or contents of matches — only which pattern indices matched. To retrieve match positions after a `RegexSet` match, compile the relevant patterns individually and call `find` on them. This is the cost of the single-pass guarantee.

### 8.1 RegexSetBuilder

```ferrum
/// Builder for RegexSet with shared flags across all patterns.
type RegexSetBuilder {
    fn new(patterns: impl IntoIterator[Item=impl AsRef[str]]): Self
    fn case_insensitive(self, yes: bool): Self
    fn multiline(self, yes: bool): Self
    fn dot_all(self, yes: bool): Self
    fn unicode(self, yes: bool): Self
    fn size_limit(self, bytes: usize): Self
    fn dfa_size_limit(self, bytes: usize): Self
    fn build(self): Result[RegexSet, RegexError] ! Alloc
}
```

---

## 9. Backtracking Opt-In

Backtracking enables features that the NFA engine cannot express:

- **Lookahead:** `(?=...)`, `(?!...)`
- **Lookbehind:** `(?<=...)`, `(?<!...)`
- **Backreferences:** `\1`, `\2`, etc.

To enable backtracking:

```ferrum
// COMPILER WARNING: backtracking enabled — O(2^n) worst case on adversarial input
let re = RegexBuilder.new("(?<=foo)bar")
    .backtracking(true)
    .build()?
```

The compiler warning is **not suppressible** without a blanket lint suppression. This is intentional: every use of backtracking in a codebase should be visible in a grep for the warning suppression annotation. Programs that process untrusted input (HTTP servers, log processors, parsers) should treat any backtracking regex as a code review finding.

Backtracking is implemented as a separate engine path. The NFA engine is always tried first; the backtracking engine is used only when the pattern contains constructs the NFA engine cannot handle. A pattern with backtracking enabled but no backtracking constructs is identical in behavior and performance to a pattern without backtracking — the flag controls availability, not enforcement.

There is no API to check at runtime whether a pattern uses backtracking. The distinction is static, visible in the builder call.

---

## 10. Error Types

```ferrum
/// An error compiling or running a regex.
/// Never used for match failure — no match is None, not Err.
@derive(Debug)
enum RegexError {
    /// The pattern contains a syntax error.
    Syntax {
        /// The byte offset in the pattern where the error was detected.
        pos:     usize,
        /// A human-readable description of the error.
        message: String,
        /// The original pattern, for display.
        pattern: String,
    }

    /// The compiled NFA exceeded the configured size limit.
    TooLarge {
        limit:   usize,
        actual:  usize,
        pattern: String,
    }

    /// A named capture group was referenced but does not exist in the pattern.
    UnknownGroup {
        name:    String,
        pattern: String,
    }
}

impl fmt.Display for RegexError {
    // Formats as:
    //   Syntax error at byte 12 in pattern `foo(?P<`: unexpected end of group name
    //   Pattern too large: NFA has 15432 states, limit is 10000
    //   Unknown capture group 'year' in pattern `(?P<month>\d+)`
}

impl core.error.Error for RegexError {}
```

`RegexError` uses string descriptions, not error codes. There is no numeric error code API. If you need to programmatically distinguish error kinds, match on the enum variant. If you need the position of a syntax error, read `Syntax.pos`.

---

## 11. Byte-Slice Matching (bytes sub-module)

The default API operates on `&str` (UTF-8). For binary data or when you need to match on raw bytes rather than Unicode characters, use `extlib.regex.bytes`:

```ferrum
// extlib.regex.bytes — all types and methods mirror the str API,
// but input and output are &[u8] instead of &str.

use extlib.regex.bytes.{Regex as ByteRegex, RegexBuilder as ByteRegexBuilder}

let re = ByteRegexBuilder.new(b"foo\\x00bar")
    .unicode(false)
    .build()?

let input: &[u8] = b"foo\x00bar baz"
match re.find(input) {
    None    => println("not found")
    Some(m) => println("found at {}..{}", m.start(), m.end())
}
```

The `bytes` sub-module exports: `Regex`, `RegexBuilder`, `Match`, `Captures`, `RegexSet`, `RegexSetBuilder`. All types are analogous to the `str` versions with `&[u8]` substituted for `&str`.

---

## 12. Performance Notes

### NFA Guarantee

The default engine is a Thompson NFA simulation. Time complexity is O(n * m) where n is the input length and m is the number of NFA states. State count is bounded by the pattern size. The `size_limit` flag (default 10,000 states) prevents patterns that are syntactically valid but computationally pathological.

There is no way to construct a ReDoS vulnerability with the default engine, regardless of input. This holds for all patterns that can be expressed without backtracking constructs.

### DFA Acceleration

A lazy DFA cache sits above the NFA. For patterns with bounded state space, the DFA is constructed on demand as new input characters arrive. Cache hits are O(1) per character. The DFA cache is bounded by `dfa_size_limit`; when the cache fills, the engine falls back to NFA simulation without any behavioral change.

Most patterns with fixed character classes and no large alternations will run in DFA mode for the majority of their input. Patterns with many Unicode character classes may fall back to NFA mode more frequently.

### Prefilter Optimization

When a pattern requires a literal byte sequence to match (e.g., `foo\w+` must start with `foo`), the engine uses SIMD byte search to skip non-matching positions before starting NFA simulation. This makes literal-anchored patterns very fast on long inputs that mostly do not match.

### Clone and Arc

`Regex` and `RegexSet` are both `Clone`. Cloning is inexpensive: the compiled NFA is reference-counted internally (`Arc`). Cloning a `Regex` does not recompile the pattern.

For sharing a `Regex` across threads, `Arc[Regex]` or direct sharing via references both work. `Regex` is `Send + Sync`.

---

## 13. Example Usage

### 13.1 Basic Matching

```ferrum
use extlib.regex.Regex

fn count_words(text: &str): usize {
    let re = regex("\\b\\w+\\b")
    re.find_iter(text).count()
}
```

### 13.2 Named Capture Extraction

```ferrum
use extlib.regex.{Regex, Captures}

type LogEntry {
    timestamp: String,
    level:     String,
    message:   String,
}

fn parse_log_line(line: &str): Option[LogEntry] ! Alloc {
    let re = regex(
        r"^(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z)\s+\[(?P<level>\w+)\]\s+(?P<msg>.+)$"
    )
    let caps = re.captures(line)?
    Some(LogEntry {
        timestamp: caps.name("ts").unwrap().as_str().to_owned(),
        level:     caps.name("level").unwrap().as_str().to_owned(),
        message:   caps.name("msg").unwrap().as_str().to_owned(),
    })
}
```

### 13.3 Replacement with Capture References

```ferrum
use extlib.regex.Regex

fn normalize_date(text: &str): String ! Alloc {
    let re = regex("(?P<y>\\d{4})-(?P<m>\\d{2})-(?P<d>\\d{2})")
    re.replace_all(text, "$d/$m/$y")
}

fn main() ! IO + Alloc {
    println("{}", normalize_date("Meeting on 2026-04-10 and 2026-05-01"))
    // "Meeting on 10/04/2026 and 01/05/2026"
}
```

### 13.4 Computed Replacement

```ferrum
use extlib.regex.Regex

fn redact_emails(text: &str): String ! Alloc {
    let re = regex("[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}")
    re.replace_all(text, fn(caps: &extlib.regex.Captures): String {
        let addr = caps[0].as_str()
        let at = addr.find('@').unwrap_or(addr.len())
        format("[REDACTED@{}]", &addr[at+1..])
    })
}
```

### 13.5 RegexSet for Classification

```ferrum
use extlib.regex.RegexSet

fn classify_input(s: &str): &'static str {
    let set = RegexSet.new([
        r"^\d+$",               // 0: integer
        r"^\d+\.\d+$",          // 1: decimal
        r"^[a-zA-Z_]\w*$",      // 2: identifier
        r"^https?://\S+$",      // 3: URL
    ]).expect("patterns are valid")

    let m = set.matches(s)
    if m.matched(3)      { return "url" }
    if m.matched(1)      { return "decimal" }
    if m.matched(0)      { return "integer" }
    if m.matched(2)      { return "identifier" }
    "unknown"
}
```

### 13.6 Splitting

```ferrum
use extlib.regex.Regex

fn tokenize_csv_line(line: &str): Vec[&str] {
    let re = regex(",\\s*")
    re.split(line).collect()
}
```

### 13.7 Builder Flags

```ferrum
use extlib.regex.RegexBuilder

fn find_case_insensitive(pattern: &str, text: &str): bool ! Alloc {
    let re = RegexBuilder.new(pattern)
        .case_insensitive(true)
        .multiline(false)
        .build()?
    Ok(re.is_match(text))
}
```

### 13.8 Error Handling

```ferrum
use extlib.regex.{Regex, RegexError}

fn compile_user_pattern(pattern: &str): Result[Regex, String] ! Alloc {
    Regex.new(pattern).map_err(fn(e: RegexError): String {
        match e {
            RegexError.Syntax { pos, message, .. } =>
                format("invalid pattern at byte {}: {}", pos, message)
            RegexError.TooLarge { limit, actual, .. } =>
                format("pattern too complex: {} states (limit {})", actual, limit)
            RegexError.UnknownGroup { name, .. } =>
                format("unknown capture group '{}'", name)
        }
    })
}
```

---

## 14. Dependencies

`extlib.regex` depends on the following Ferrum standard library modules:

| Module | Used for |
|---|---|
| `core.str` | `&str`, UTF-8 validation, `char` iteration |
| `core.ops` | `Index`, `Range`, `RangeBounds` |
| `core.error` | `Error` trait for `RegexError` |
| `alloc.string` | `String` for owned results and error messages |
| `alloc.vec` | `Vec[T]` for collecting iterators |
| `alloc.arc` | `Arc` for internal reference-counting of compiled NFA |
| `fmt` | `Display`, `Debug` for `RegexError` and `Match` |

`extlib.regex` does not depend on:

- `io` or `fs` — no file operations
- `net` — no network operations
- `async` — no async operations
- `sys` or `posix` — no OS calls
- `crypto` — no cryptographic operations

The `! Alloc` effect appears on methods that return `String` or compile patterns. Matching and iteration over an already-compiled `Regex` against a pre-existing `&str` carry no effect annotations — they are pure computations.
