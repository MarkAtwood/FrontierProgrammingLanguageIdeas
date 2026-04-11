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

#### Python's Hidden Encoding: A Real Example

Here is what actually happens in Python when encodings go wrong:

```python
# You write this on your Mac (where locale is UTF-8):
def read_config(path):
    with open(path) as f:
        return f.read()

config = read_config("settings.txt")  # Works fine on your machine

# Your colleague runs it on their Windows box with a Latin-1 encoded file:
# UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 47
```

The problem: `open()` uses the system default encoding. On modern macOS and Linux, that is UTF-8. On Windows, it might be cp1252 (Windows Latin-1). The same code behaves differently on different machines.

```python
# "Fix" attempt 1: Force UTF-8
with open("settings.txt", encoding="utf-8") as f:
    config = f.read()
# Now it explodes on your Windows colleague's Latin-1 file

# "Fix" attempt 2: Try UTF-8, fall back
try:
    with open("settings.txt", encoding="utf-8") as f:
        config = f.read()
except UnicodeDecodeError:
    with open("settings.txt", encoding="latin-1") as f:
        config = f.read()
# Works but you opened the file twice, and the error happens at read(),
# not at open(), so you might have already done partial work
```

Even worse, the error often does not happen where you expect:

```python
# This succeeds:
with open("data.csv", encoding="utf-8") as f:
    for line in f:
        process(line)  # UnicodeDecodeError on line 4,892
        # You have already processed 4,891 lines. Partial state. Now what?
```

The fundamental issue: Python hides the encoding decision inside `open()` and the validation happens lazily as you read. By the time you discover the encoding is wrong, you may have already committed to a partial result.

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

### `&str` - Borrowed String Slice

A reference to text stored elsewhere. Does not own the data. Think of it like a `const char*` in C, but with a known length and guaranteed UTF-8 encoding.

```ferrum
fn greet(name: &str) {
    print("Hello, {name}!")
}

let s: String = String.from("Alice")
greet(&s)           // borrow from String
greet("Bob")        // borrow from string literal (stored in binary)
```

Use `&str` for function parameters when you only need to read the text.

### `String` - Owned String

Heap-allocated, growable, owned text. Like `std::string` in C++ or `str` in Python, but always UTF-8.

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

## When Bytes vs Strings Actually Matters

The distinction is not academic. Here are real scenarios where getting it wrong causes bugs:

### Scenario 1: Reading a Config File

You have a config file that might be UTF-8 or might be legacy Latin-1 from an old Windows install:

```ferrum
import fs
import text.{utf8, latin1}

fn load_config(path: &str): Result[HashMap[String, String], Error] ! IO {
    let bytes = fs.read(path)?

    // Try UTF-8 first (most common)
    let text = match utf8.validate(&bytes) {
        Ok(s) => s.to_string(),
        Err(e) => {
            // Fall back to Latin-1 for legacy files
            // Latin-1 never fails - every byte is valid
            log.warn("Config file is not UTF-8, assuming Latin-1")
            latin1.decode(&bytes)
        }
    };

    parse_key_value(&text)
}
```

In Python, you would either pick an encoding and hope, or wrap in try/except and open the file twice. In Ferrum, you read bytes once, then decide how to decode.

### Scenario 2: Parsing a CSV with Mixed Content

A CSV file contains UTF-8 text columns but also has a column with raw binary identifiers:

```ferrum
fn parse_row(line: &str): Result[Record, ParseError] {
    let fields: Vec[&str] = line.split(',').collect()

    Ok(Record {
        // Text fields - already validated UTF-8 since line is &str
        name: fields[0].to_string(),
        email: fields[1].to_string(),

        // Binary ID column is base64-encoded in the CSV
        // Decode to raw bytes, not string
        id: base64.decode(fields[2])?,  // Returns Vec[u8]
    })
}
```

### Scenario 3: Handling User Input from the Web

HTTP request bodies come in as bytes. You need to validate encoding before treating as text:

```ferrum
import text.utf8

fn handle_form_post(body: &[u8]): Result[FormData, Error] {
    // Validate that the body is actually UTF-8
    let text = utf8.validate(body)
        .map_err(|e| Error.bad_request("Body is not valid UTF-8"))?;

    // Now safe to parse as text
    parse_form_urlencoded(text)
}

fn handle_file_upload(body: &[u8]): Result[(), Error] ! IO {
    // Binary upload - do NOT validate as UTF-8
    // Just write the bytes directly
    fs.write("uploads/file.bin", body)?;
    Ok(())
}
```

### Scenario 4: Building an HTTP Response

The HTTP protocol is bytes. Headers are ASCII (technically). Body might be anything:

```ferrum
fn build_response(status: u16, body: &str): Vec[u8] {
    let mut response = Vec.new()

    // Status line and headers must be ASCII
    write!(&mut response, "HTTP/1.1 {status} OK\r\n")
    write!(&mut response, "Content-Type: text/html; charset=utf-8\r\n")
    write!(&mut response, "Content-Length: {body.len()}\r\n")
    write!(&mut response, "\r\n")

    // Body is UTF-8 text, append as bytes
    response.extend(body.as_bytes())

    response  // Vec[u8] ready to send over the socket
}

fn build_binary_response(data: &[u8]): Vec[u8] {
    let mut response = Vec.new()

    write!(&mut response, "HTTP/1.1 200 OK\r\n")
    write!(&mut response, "Content-Type: application/octet-stream\r\n")
    write!(&mut response, "Content-Length: {data.len()}\r\n")
    write!(&mut response, "\r\n")

    // Binary body - just append the bytes
    response.extend(data)

    response
}
```

### Scenario 5: Processing Binary Protocols

Network protocols often mix text and binary. DNS, for example, has length-prefixed labels:

```ferrum
fn parse_dns_name(data: &[u8]): Result[String, ParseError] {
    let mut name = String.new()
    let mut pos = 0

    while pos < data.len() && data[pos] != 0 {
        let len = data[pos] as usize
        pos += 1

        let label = &data[pos..pos + len]

        // DNS labels are ASCII-ish but validate anyway
        let label_str = utf8.validate(label)
            .map_err(|_| ParseError.new("Invalid UTF-8 in DNS label"))?;

        if !name.is_empty() {
            name.push('.')
        }
        name.push_str(label_str)

        pos += len
    }

    Ok(name)
}
```

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

// Or with lossy conversion (replaces invalid sequences with a replacement character)
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

### Latin-1 (ISO-8859-1)

Latin-1 is a single-byte encoding where every byte 0x00-0xFF maps directly to Unicode U+0000-U+00FF. Decoding never fails.

```ferrum
import text.latin1

// Decoding: bytes to UTF-8 string (never fails)
let legacy_bytes: &[u8] = &[0x43, 0x61, 0x66, 0xe9]  // "Cafe" with e-acute
let text: String = text.latin1.decode(legacy_bytes)
print(text)  // "Cafe" (with proper e-acute)

// Encoding: string to Latin-1 bytes (can fail)
let output: Result[Vec[u8], EncodeError] = text.latin1.encode("Cafe")
// Ok([0x43, 0x61, 0x66, 0xe9])

// Encoding fails if the string contains characters outside Latin-1
let result = text.latin1.encode("Hello ")
// Err(EncodeError { character: '', position: 6 })
// The emoji cannot be represented in Latin-1
```

Real-world example - converting an old log file:

```ferrum
fn convert_legacy_logs(input_path: &str, output_path: &str): Result[(), Error] ! IO {
    let bytes = fs.read(input_path)?

    // Old system wrote Latin-1
    let text = latin1.decode(&bytes)

    // Write as modern UTF-8
    fs.write_text(output_path, &text)?

    Ok(())
}
```

### ASCII

ASCII is the 7-bit subset. Bytes above 127 are invalid.

```ferrum
import text.ascii

// Validate ASCII
let bytes: &[u8] = b"Hello"
match ascii.validate(bytes) {
    Ok(s) => print("Valid ASCII: {s}"),
    Err(e) => print("Non-ASCII byte at position {e.position}"),
}

// ASCII is a subset of UTF-8, so valid ASCII is valid UTF-8
let ascii_text = ascii.validate(b"Hello")?
// ascii_text is &str, usable anywhere &str is expected
```

Real-world example - parsing HTTP headers:

```ferrum
fn parse_header(line: &[u8]): Result[(String, String), ParseError] {
    // HTTP headers must be ASCII (technically ISO-8859-1 but ASCII in practice)
    let text = ascii.validate(line)
        .map_err(|_| ParseError.new("Header contains non-ASCII bytes"))?;

    let (name, value) = text.split_once(": ")
        .ok_or(ParseError.new("Malformed header"))?;

    Ok((name.to_string(), value.trim().to_string()))
}
```

### UTF-16

Common in Windows APIs and some file formats. UTF-16 uses 2 or 4 bytes per character.

```ferrum
import text.utf16

// Decode UTF-16 Little Endian (Windows standard)
let windows_bytes: &[u8] = &[0x48, 0x00, 0x65, 0x00, 0x6c, 0x00]  // "Hel"
let text: String = text.utf16.decode_le(windows_bytes)?

// Decode UTF-16 Big Endian (network byte order, some file formats)
let be_bytes: &[u8] = &[0x00, 0x48, 0x00, 0x65, 0x00, 0x6c]
let text: String = text.utf16.decode_be(be_bytes)?

// Auto-detect from BOM (Byte Order Mark)
let with_bom: &[u8] = &[0xFF, 0xFE, 0x48, 0x00]  // BOM + "H"
let text: String = text.utf16.decode(with_bom)?

// Encode to UTF-16 LE
let output: Vec[u8] = text.utf16.encode_le("Hello")?
```

Real-world example - reading a Windows .reg file:

```ferrum
fn read_registry_export(path: &str): Result[String, Error] ! IO {
    let bytes = fs.read(path)?

    // .reg files are UTF-16 LE with BOM
    if bytes.len() >= 2 && bytes[0] == 0xFF && bytes[1] == 0xFE {
        utf16.decode_le(&bytes[2..])?
    } else {
        // Old format, ASCII
        ascii.validate(&bytes)?.to_string()
    }
}
```

### Summary of Encoding Modules

| Module | Decoding | Encoding | Notes |
|--------|----------|----------|-------|
| `text.utf8` | Validates bytes as UTF-8 | N/A (strings are already UTF-8) | Primary encoding |
| `text.ascii` | Validates bytes as ASCII | N/A (ASCII is a UTF-8 subset) | 7-bit only |
| `text.latin1` | Always succeeds | Fails if char > U+00FF | Every byte is valid |
| `text.utf16` | Can fail (odd length, invalid surrogates) | Always succeeds | LE/BE variants |

Legacy encodings (Shift-JIS, GBK, Windows-1252, KOI8-R, etc.) are in the `encoding` crate.

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
| Length | `strlen(s)` - scans every call | `s.len()` - O(1), stored |
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

### `CStr` - C Strings

C strings are null-terminated and have no encoding guarantee. `CStr` is how you safely work with them.

**Receiving strings from C code:**

```ferrum
import core.ffi.{CStr, CString, c_char}

// Suppose you are calling SQLite
extern fn sqlite3_errmsg(db: *mut sqlite3): *const c_char

fn get_error_message(db: *mut sqlite3): String {
    let ptr: *const c_char = sqlite3_errmsg(db)

    // Safety: sqlite3_errmsg returns a valid null-terminated string
    let c_str: &CStr = unsafe { CStr.from_ptr(ptr) }

    // SQLite error messages are ASCII, but use lossy just in case
    c_str.to_string_lossy().into_owned()
}
```

**Passing strings to C code:**

```ferrum
// Suppose you are calling a C logging library
extern fn log_message(msg: *const c_char)

fn log(message: &str) {
    // CString adds null terminator and checks for interior nulls
    let c_string = CString.new(message)
        .expect("log message contained null byte")

    // Pass pointer to C. c_string must outlive this call.
    unsafe { log_message(c_string.as_ptr()) }

    // c_string is dropped here, memory freed
}
```

**Using C string literals:**

```ferrum
// Compile-time C string (includes null terminator in binary)
const HELLO: &CStr = c"Hello"

extern fn puts(s: *const c_char)

fn greet() {
    // No allocation needed - literal is in the binary
    unsafe { puts(HELLO.as_ptr()) }
}
```

**Common CStr patterns:**

```ferrum
// Check if a C function returned an error string
extern fn dlerror(): *const c_char

fn check_dl_error(): Option[String] {
    let ptr = dlerror()
    if ptr.is_null() {
        return None
    }

    let c_str = unsafe { CStr.from_ptr(ptr) }
    Some(c_str.to_string_lossy().into_owned())
}
```

### `OsStr` - OS-Native Strings

File paths and OS strings may not be valid UTF-8 on all platforms:

- **Unix:** Paths are arbitrary bytes (except null). Usually UTF-8, but not guaranteed.
- **Windows:** Paths are UTF-16, but the WTF-8 representation can be invalid UTF-8.

`OsStr` handles this portably.

**Working with file paths:**

```ferrum
import core.ffi.OsStr
import fs.Path

fn list_directory(dir: &Path): Result[Vec[String], Error] ! IO {
    let mut names = Vec.new()

    for entry in fs.read_dir(dir)? {
        let path: &Path = entry.path()
        let os_str: &OsStr = path.file_name().unwrap()

        // Try to get UTF-8 name
        match os_str.to_str() {
            Some(name) => names.push(name.to_string()),
            None => {
                // Filename is not valid UTF-8
                // Use lossy conversion for display
                names.push(os_str.to_string_lossy().into_owned())
            }
        }
    }

    Ok(names)
}
```

**Environment variables:**

```ferrum
import env

fn get_home(): Option[String] {
    let os_str: OsString = env.var_os("HOME")?

    // HOME should be UTF-8 on any sane system, but check anyway
    os_str.into_string().ok()
}
```

**Creating OsStr for system calls:**

```ferrum
import core.ffi.OsString

fn set_temp_dir(path: &str) {
    // Convert UTF-8 string to OsString
    let os_path = OsString.from(path)
    env.set_var("TMPDIR", os_path)
}
```

**Real FFI example - calling stat():**

```ferrum
import core.ffi.{CString, c_char}
import libc

struct FileInfo {
    size: u64,
    is_dir: bool,
}

fn stat_file(path: &str): Result[FileInfo, Error] {
    // stat() wants a C string
    let c_path = CString.new(path)?

    let mut stat_buf: libc.stat = zeroed()

    let result = unsafe { libc.stat(c_path.as_ptr(), &mut stat_buf) }

    if result == -1 {
        return Err(Error.last_os_error())
    }

    Ok(FileInfo {
        size: stat_buf.st_size as u64,
        is_dir: (stat_buf.st_mode & libc.S_IFDIR) != 0,
    })
}
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

### Parsing a Config File

```ferrum
import fs
import text.utf8

struct Config {
    host: String,
    port: u16,
    debug: bool,
}

fn parse_config(path: &str): Result[Config, Error] ! IO {
    let text = fs.read_text(path)?

    let mut host = String.from("localhost")
    let mut port: u16 = 8080
    let mut debug = false

    for line in text.lines() {
        let line = line.trim()

        // Skip comments and blank lines
        if line.is_empty() || line.starts_with('#') {
            continue
        }

        let (key, value) = line.split_once('=')
            .ok_or(Error.new("Invalid config line: {line}"))?;

        let key = key.trim()
        let value = value.trim()

        match key {
            "host" => host = value.to_string(),
            "port" => port = value.parse()?,
            "debug" => debug = value == "true",
            _ => log.warn("Unknown config key: {key}"),
        }
    }

    Ok(Config { host, port, debug })
}
```

### Parsing CSV Data

```ferrum
fn parse_csv(text: &str): Vec[Vec[String]] {
    let mut rows = Vec.new()

    for line in text.lines() {
        let fields: Vec[String] = line.split(',')
            .map(|field| field.trim().to_string())
            .collect();
        rows.push(fields)
    }

    rows
}

// More robust: handle quoted fields
fn parse_csv_field(field: &str): String {
    let field = field.trim()

    if field.starts_with('"') && field.ends_with('"') {
        // Remove quotes and unescape doubled quotes
        field[1..field.len()-1].replace("\"\"", "\"")
    } else {
        field.to_string()
    }
}
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

### Building an HTTP Response

```ferrum
fn json_response(data: &str): Vec[u8] {
    let mut response = Vec.new()

    // Headers (ASCII)
    write!(&mut response, "HTTP/1.1 200 OK\r\n")
    write!(&mut response, "Content-Type: application/json; charset=utf-8\r\n")
    write!(&mut response, "Content-Length: {data.len()}\r\n")
    write!(&mut response, "\r\n")

    // Body (UTF-8)
    response.extend(data.as_bytes())

    response
}

fn error_response(status: u16, message: &str): Vec[u8] {
    let body = format!("{{\"error\": \"{message}\"}}")

    let mut response = Vec.new()
    write!(&mut response, "HTTP/1.1 {status} Error\r\n")
    write!(&mut response, "Content-Type: application/json\r\n")
    write!(&mut response, "Content-Length: {body.len()}\r\n")
    write!(&mut response, "\r\n")
    response.extend(body.as_bytes())

    response
}
```

---

## Compiler Error Messages

Ferrum catches encoding bugs at compile time. Here are common errors and what they mean:

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

**What it means:** You tried to pass raw bytes to a function expecting text. The compiler wants you to explicitly validate the encoding first.

**How to fix:**

```ferrum
let bytes: &[u8] = b"hello"
let text = text.utf8.validate(bytes)?  // Now it is &str
wants_str(text)
```

### Passing String Where Bytes Expected

```ferrum
fn wants_bytes(b: &[u8]) { }

let text: &str = "hello"
wants_bytes(text)
```

```
error[E0308]: mismatched types
  --> src/main.fe:5:12
   |
5  | wants_bytes(text)
   |             ^^^^ expected `&[u8]`, found `&str`
   |
   = help: use `text.as_bytes()` to get the underlying bytes
```

**How to fix:**

```ferrum
let text: &str = "hello"
wants_bytes(text.as_bytes())  // Explicit conversion
```

### Indexing a String by Integer

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

**What it means:** UTF-8 is variable-width. `s[2]` could mean "the byte at position 2" or "the character at position 2" - they are different things. Ferrum makes you be explicit.

**How to fix:**

```ferrum
// If you want the 3rd character:
let c: Option[char] = s.chars().nth(2)

// If you want bytes 2-3 (must be on character boundaries):
let slice: &str = &s[2..3]
```

### Using Result Without Handling Error

```ferrum
fn read_file(): String {
    let bytes = fs.read("data.txt")  // Returns Result
    let text = text.utf8.from_bytes(bytes)  // Also returns Result
    text  // Error: expected String, got Result[String, Error]
}
```

```
error[E0308]: mismatched types
  --> src/main.fe:4:5
   |
3  | fn read_file(): String {
   |                 ------ expected `String` because of return type
4  |     text
   |     ^^^^ expected `String`, found `Result[String, Utf8Error]`
   |
   = help: use `?` to propagate the error or handle it with `match`
```

**How to fix:**

```ferrum
fn read_file(): Result[String, Error] ! IO {
    let bytes = fs.read("data.txt")?  // ? propagates error
    let text = text.utf8.from_bytes(bytes)?
    Ok(text)
}
```

### Invalid Slice Boundary (Runtime)

The compiler cannot catch this statically (byte positions depend on content), but the runtime message is clear:

```
thread 'main' panicked at 'byte index 5 is not a char boundary;
it is inside the 2nd character which spans bytes 4..6', src/main.fe:3:15
```

**What it means:** You tried to slice a string at a byte position that lands in the middle of a multi-byte character.

**How to fix:** Use `.get()` for safe slicing, or `.char_indices()` to find valid boundaries:

```ferrum
let s = "Cafe"  // 'e' is 2 bytes

// Safe: returns None if not on boundary
if let Some(slice) = s.get(0..5) {
    print(slice)
}

// Or find character boundaries explicitly
for (byte_pos, ch) in s.char_indices() {
    print("Character '{ch}' starts at byte {byte_pos}")
}
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
