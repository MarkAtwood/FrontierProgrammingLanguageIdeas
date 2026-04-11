# Ferrum: Strings and Bytes

**Audience:** Developers familiar with C and Python who want to understand how Ferrum handles text.

---

## The Problem Ferrum Solves

### C: No Encoding, No Safety

In C, `char*` is just bytes with a null terminator. There is no encoding information:

```c
char* name = "Caf\xc3\xa9";  // Is this UTF-8? Latin-1? Who knows?
size_t len = strlen(name);    // Scans for null every time. O(n).

// Buffer overflow waiting to happen
char buf[8];
strcpy(buf, name);  // No bounds check. Hope it fits.
```

The encoding is implicit. The caller and callee must agree out-of-band. There is no compiler enforcement. `strlen()` scans the entire string every time because C strings do not store their length.

### Python 3: Implicit Encoding at IO Boundaries

Python 3 fixed encoding awareness but introduced a different problem: implicit encoding at IO boundaries.

```python
with open("file.txt") as f:
    text = f.read()  # What encoding? Default? System locale? PYTHONIOENCODING?
```

When does encoding happen? What encoding is used? What if it fails partway through the file? The answers are hidden behind defaults. This causes confusion:

- `UnicodeDecodeError` appears in production when file encoding does not match expectation
- Encoding behavior changes between Windows and Linux
- Environment variables and locale settings affect behavior

---

## Ferrum's Model

Ferrum separates bytes and strings at the type level:

| Type | What it is | Encoding |
|------|------------|----------|
| `&str` | Borrowed text slice | Always UTF-8 |
| `String` | Owned text | Always UTF-8 |
| `&[u8]` | Borrowed byte slice | None (raw bytes) |
| `Vec[u8]` | Owned byte buffer | None (raw bytes) |

The rule is simple:

> **IO deals in bytes. Strings are always UTF-8. Conversion is explicit.**

There is no implicit encoding. There is no default. You choose when to convert and what encoding to use.

---

## `&str` vs `String`: Borrowed vs Owned

Both `&str` and `String` are always valid UTF-8. The difference is ownership.

### `&str` — Borrowed String Slice

A reference to text stored elsewhere. Does not own the data.

```ferrum
fn greet(name: &str) {
    print("Hello, {name}!")
}

let s: String = String.from("Alice")
greet(&s)           // borrow from String
greet("Bob")        // borrow from string literal (stored in binary)
```

Use `&str` for function parameters when you only need to read the text.

### `String` — Owned String

Heap-allocated, growable, owned text.

```ferrum
let mut s = String.from("Hello")
s.push_str(", world")
s.push('!')
// s is now "Hello, world!"
```

Use `String` when you need to:
- Build text incrementally
- Store text in a struct
- Return text from a function that created it

### The Relationship

`String` dereferences to `&str`. Any function that takes `&str` accepts `&String` automatically:

```ferrum
fn word_count(text: &str): usize {
    text.split(' ').count()
}

let owned: String = String.from("one two three")
word_count(&owned)  // works: &String coerces to &str
word_count("four five")  // works: literal is &str
```

---

## `&[u8]` and `Vec[u8]`: Raw Bytes

For binary data or text in unknown/non-UTF-8 encodings, use byte slices.

```ferrum
let bytes: &[u8] = b"raw bytes"       // byte string literal
let data: Vec[u8] = vec![0x48, 0x65, 0x6c, 0x6c, 0x6f]

// No encoding assumption. Just bytes.
let jpeg_header: &[u8] = &[0xFF, 0xD8, 0xFF]
```

### Why Separate Bytes and Strings?

The compiler prevents you from mixing them accidentally:

```ferrum
fn process_text(s: &str) { ... }

let bytes: &[u8] = b"hello"
process_text(bytes)
// error[E0308]: mismatched types
//   expected `&str`, found `&[u8]`
//   note: bytes are not strings; use text.utf8.validate() to convert
```

This catches encoding bugs at compile time. You cannot accidentally pass binary data to a function expecting text.

---

## Converting Between Bytes and Strings

### Bytes to String: Validation Required

UTF-8 strings require validation. Not all byte sequences are valid UTF-8.

```ferrum
import text.utf8

let bytes: &[u8] = b"Hello, world!"

// Validate and convert (zero-copy if valid)
match text.utf8.validate(bytes) {
    Ok(s)  => print("Got text: {s}"),
    Err(e) => print("Invalid UTF-8 at byte {e.valid_up_to()}"),
}

// Or with lossy conversion (replaces invalid sequences with U+FFFD)
let s: Cow[str] = text.utf8.validate_lossy(bytes)
```

For owned conversions:

```ferrum
let bytes: Vec[u8] = read_file("data.txt")?

// Try to convert (takes ownership on success)
let text: String = text.utf8.from_bytes(bytes)?

// Or unchecked (unsafe, undefined behavior if invalid)
let text: String = unsafe { text.utf8.from_bytes_unchecked(bytes) }
```

### String to Bytes: Always Valid

Converting from string to bytes is free. UTF-8 strings are already stored as bytes internally.

```ferrum
let s: &str = "Hello"
let bytes: &[u8] = s.as_bytes()  // zero-cost, no allocation

let owned: String = String.from("World")
let byte_vec: Vec[u8] = owned.into_bytes()  // moves ownership
```

---

## Other Encodings

Ferrum does not have special readers for different encodings. Instead, you:
1. Read bytes
2. Transform to UTF-8
3. Work with the text
4. Transform back to bytes
5. Write bytes

```ferrum
import text.latin1
import text.utf16

// Latin-1: every byte is valid, maps to U+0000..U+00FF
let latin1_bytes: &[u8] = read_legacy_file()?
let text: String = text.latin1.decode(latin1_bytes)

// UTF-16 Little Endian (Windows)
let utf16_bytes: &[u8] = read_windows_file()?
let text: String = text.utf16.decode_le(utf16_bytes)?

// Writing back to Latin-1 (fails if text contains non-Latin-1 chars)
let output_bytes: Vec[u8] = text.latin1.encode(&text)?
```

Encodings available:
- `text.utf8` — UTF-8 (validation only; strings are already UTF-8)
- `text.latin1` — ISO-8859-1 (every byte valid)
- `text.ascii` — ASCII (fails on bytes > 127)
- `text.utf16` — UTF-16 LE/BE

Legacy encodings (Shift-JIS, GBK, Windows-1252, etc.) are in separate crates.

---

## String Slicing

Strings can be sliced by byte index, but the index must fall on a character boundary.

```ferrum
let s = "Hello"
let hello = &s[0..5]  // "Hello" - OK

let s = "Cafe\u{0301}"  // "Cafe" + combining accent (5 chars, 6 bytes)
let partial = &s[0..4]  // "Cafe" - OK (byte 4 is char boundary)
let bad = &s[0..5]      // panic: byte 5 is inside a multi-byte char

// Safe alternative: use .get() which returns Option
let maybe = s.get(0..5)  // Some("Cafe") or None if not on boundary
```

For character-level operations:

```ferrum
let s = "Hello"
let chars: Vec[char] = s.chars().collect()  // ['H', 'e', 'l', 'l', 'o']
let third_char: Option[char] = s.chars().nth(2)  // Some('l')
```

---

## Common String Operations

### Length

```ferrum
let s = "Cafe\u{0301}"  // "Cafe" with combining accent

s.len()        // 6 (bytes)
s.char_len()   // 5 (Unicode scalar values) - O(n), must scan
s.is_empty()   // false
```

### Searching and Checking

```ferrum
let text = "Hello, World!"

text.contains("World")          // true
text.starts_with("Hello")       // true
text.ends_with("!")             // true
text.find("World")              // Some(7) - byte index
text.rfind("o")                 // Some(8) - last occurrence
```

### Splitting

```ferrum
let text = "one,two,three"

for part in text.split(',') {
    print(part)  // "one", "two", "three"
}

let parts: Vec[&str] = text.split(',').collect()
let (first, rest) = text.split_once(',').unwrap()  // ("one", "two,three")
```

### Trimming

```ferrum
let padded = "  hello  "

padded.trim()        // "hello"
padded.trim_start()  // "hello  "
padded.trim_end()    // "  hello"

"##hello##".trim_matches('#')  // "hello"
```

### Replacing

```ferrum
let text = "Hello, World!"

text.replace("World", "Ferrum")  // "Hello, Ferrum!"
text.replacen("l", "L", 2)       // "HeLLo, World!" (first 2 only)
```

### Case Conversion

```ferrum
let s = "Hello"

s.to_uppercase()  // "HELLO"
s.to_lowercase()  // "hello"

// ASCII-only variants (faster, but only affect A-Z/a-z)
s.to_ascii_uppercase()  // "HELLO"
s.to_ascii_lowercase()  // "hello"
```

---

## Comparing to C

| Operation | C | Ferrum |
|-----------|---|--------|
| Length | `strlen(s)` — scans every call | `s.len()` — O(1), stored |
| Encoding | None, implicit | Always UTF-8, enforced |
| Bounds | None, buffer overflow | Checked, panics on OOB |
| Allocation | Manual `malloc`/`free` | Automatic with `String` |
| Null terminator | Required, interior nulls forbidden | Not used, any bytes allowed |

```ferrum
// C: strlen scans every time
for (int i = 0; i < strlen(s); i++) { ... }  // O(n^2)

// Ferrum: length is stored
for i in 0..s.len() { ... }  // O(n)
```

---

## Comparing to Python

| Aspect | Python 3 | Ferrum |
|--------|----------|--------|
| Encoding at IO | Implicit (locale/default) | Explicit (you call decode) |
| When conversion happens | Hidden in `open()` | You call `text.utf8.validate()` |
| Type safety | Runtime `TypeError` | Compile-time type mismatch |
| Error handling | `UnicodeDecodeError` at runtime | `Result` at conversion point |

```python
# Python: encoding is hidden, fails at runtime
with open("file.txt") as f:  # what encoding?
    text = f.read()  # UnicodeDecodeError if wrong
```

```ferrum
// Ferrum: explicit at every step
let bytes = fs.read("file.txt")?  // bytes, no encoding
let text = text.utf8.validate(&bytes)?  // explicit conversion point
// If invalid UTF-8, you handle it here, not later
```

---

## FFI: `CStr` and `OsStr`

When working with C code or operating system APIs, use specialized types.

### `CStr` — C Strings

For null-terminated C strings:

```ferrum
import core.ffi.{CStr, c_char}

// From C code
extern fn get_name(): *const c_char

let c_ptr: *const c_char = get_name()
let c_str: &CStr = unsafe { CStr.from_ptr(c_ptr) }

// Convert to Ferrum string (may fail if not UTF-8)
let name: &str = c_str.to_str()?

// Or lossy conversion (never fails)
let name: Cow[str] = c_str.to_string_lossy()
```

Creating C strings for FFI:

```ferrum
// Literal syntax (includes null terminator)
const HELLO: &CStr = c"Hello"

// At runtime
let s = "Hello"
let c_string = CString.new(s)?  // fails if s contains null bytes
let ptr: *const c_char = c_string.as_ptr()
// Pass ptr to C function. c_string must outlive the call.
```

### `OsStr` — OS-Native Strings

File paths and OS strings may not be valid UTF-8 on all platforms:

```ferrum
import core.ffi.OsStr
import fs.Path

let path = Path.new("/tmp/caf\u{e9}.txt")
let os_str: &OsStr = path.as_os_str()

// Try to get UTF-8 (may fail on exotic filenames)
match os_str.to_str() {
    Some(s) => print("Path is: {s}"),
    None => print("Path contains non-UTF-8 bytes"),
}

// Lossy display (always works)
print("Path is: {path.display()}")
```

---

## Practical Examples

### Reading a Text File

```ferrum
import fs
import text.utf8

fn read_text_file(path: &str): Result[String, Error] ! IO {
    // Step 1: Read bytes
    let bytes: Vec[u8] = fs.read(path)?

    // Step 2: Validate UTF-8
    let text: String = text.utf8.from_bytes(bytes)
        .map_err(|e| Error.new("Invalid UTF-8 at byte {e.valid_up_to()}"))?

    Ok(text)
}

// Or use the convenience function (does both steps)
let text = fs.read_text("config.txt")?
```

### Parsing User Input

```ferrum
fn parse_csv_line(line: &str): Vec[&str] {
    line.split(',')
        .map(|field| field.trim())
        .collect()
}

fn process_stdin() ! IO {
    let stdin = io.stdin()
    let mut buf = Vec.new()

    while let ReadUntilResult.Found(n) = stdin.read_line(&mut buf)? {
        // buf is Vec[u8] - must convert to text
        if let Ok(line) = text.utf8.validate(&buf) {
            let fields = parse_csv_line(line.trim())
            process_fields(fields)
        } else {
            print("Skipping non-UTF-8 line")
        }
        buf.clear()
    }
}
```

### Handling Mixed Encodings

```ferrum
import text.{utf8, latin1}

fn convert_legacy_file(input: &str, output: &str): Result[(), Error] ! IO {
    // Read as bytes
    let bytes = fs.read(input)?

    // Try UTF-8 first
    let text = match utf8.validate(&bytes) {
        Ok(s) => s.to_string(),
        Err(_) => {
            // Fall back to Latin-1 (never fails)
            latin1.decode(&bytes)
        }
    };

    // Write as UTF-8
    fs.write_text(output, &text)?

    Ok(())
}
```

### Building Strings Efficiently

```ferrum
fn build_report(items: &[Item]): String {
    let mut out = String.with_capacity(items.len() * 50)  // pre-allocate

    out.push_str("Report\n")
    out.push_str("======\n\n")

    for (i, item) in items.iter().enumerate() {
        // Use write! macro for formatting
        write!(&mut out, "{i}: {item.name} - ${item.price:.2}\n")
    }

    out
}
```

---

## Compiler Error Messages

Ferrum catches encoding bugs at compile time. Here are common errors:

### Mixing Bytes and Strings

```ferrum
fn wants_str(s: &str) { }

let bytes: &[u8] = b"hello"
wants_str(bytes)
```

```
error[E0308]: mismatched types
  --> src/main.fe:5:11
   |
5  | wants_str(bytes)
   |           ^^^^^ expected `&str`, found `&[u8]`
   |
   = note: bytes are not strings
   = help: use `text.utf8.validate(bytes)?` to convert bytes to &str
```

### Indexing Without Bounds Check

```ferrum
let s = "Hello"
let c = s[2]  // trying to get a character
```

```
error[E0277]: the type `str` cannot be indexed by `usize`
  --> src/main.fe:2:9
   |
2  | let c = s[2]
   |         ^^^^ string indices are byte ranges, not character positions
   |
   = note: use `s.chars().nth(2)` to get the 3rd character
   = note: use `&s[2..3]` to get a byte range (must be on char boundary)
```

### Invalid Slice Boundary

The compiler cannot catch this statically (byte positions depend on content), but the runtime message is clear:

```
thread 'main' panicked at 'byte index 5 is not a char boundary;
it is inside the 2nd character which spans bytes 4..6', src/main.fe:3:15
```

---

## Summary

| Concept | Rule |
|---------|------|
| String encoding | Always UTF-8. No exceptions. |
| IO data | Always bytes. No implicit encoding. |
| Conversion | Explicit. You call the function. |
| Type safety | Compiler enforces bytes vs strings. |
| Slicing | By byte index. Must be on char boundary. |
| Length | `len()` is bytes (O(1)). `char_len()` is chars (O(n)). |
| FFI | Use `CStr` for C strings, `OsStr` for OS strings. |

The pattern is always the same:
1. Read bytes
2. Validate/decode to UTF-8 at a deliberate point
3. Work with text
4. Encode back to bytes if needed
5. Write bytes

You control when encoding happens. The type system enforces that you do not skip steps.
