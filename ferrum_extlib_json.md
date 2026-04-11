# Ferrum Extended Library — JSON

**Module:** `extlib.ccsp.json`
**RFC basis:** RFC 8259 (JSON), RFC 6901 (JSON Pointer), RFC 7396 (JSON Merge Patch)
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `alloc`, `io`, `collections` (stdlib only; intentionally standalone)

---

## 1. Overview and Rationale

### Why JSON

JSON (JavaScript Object Notation, RFC 8259) is the lingua franca of data exchange on the modern internet. It is the wire format for REST APIs, the dominant configuration file format outside of TOML and YAML, the serialization layer for message queues, the storage format for document databases, and the output format of command-line tools. A systems language that cannot parse and produce JSON fluently cannot participate in the ecosystem that most programs live in.

Ferrum's standard library includes CBOR (`std.data.cbor`) because CBOR is the binary serialization format used in CoAP, COSE, FIDO2/WebAuthn, and other constrained-device protocols that are core to Ferrum's embedded and IoT targets. JSON is not in the standard library. The rationale is practical: a JSON implementation that is correct, secure, and fast is not a trivially small module. DOM parsing, streaming SAX-style parsing, typed deserialization with derive macros, JSON Pointer traversal, and streaming serialization represent substantial code and design surface. Bundling this with every Ferrum program — including no-alloc embedded programs that will never see a REST API — is the wrong default.

Programs that need JSON opt in via `extlib.ccsp.json`. Programs that do not need it pay nothing.

### Integer Correctness

JSON was designed in a JavaScript context. JavaScript has only one numeric type — IEEE 754 double-precision float — and JSON was specified to match it. The practical consequence: a JSON number with no decimal point or exponent (e.g., `42`, `-7`, `1000000`) is represented internally as `f64` in naive implementations, which silently loses precision for integers larger than 2^53. This is the JavaScript mistake.

`extlib.ccsp.json` corrects it. Integer-valued JSON numbers — those with no decimal point and no exponent in their source representation — are parsed as `i64`. The JSON value `9007199254740993` is not the same as `9007199254740992` in this library. Applications that handle financial amounts, database primary keys, Unix timestamps in milliseconds, or 64-bit identifiers get the right answer.

The rules:

- `42` → `Value.Integer(42i64)`
- `-7` → `Value.Integer(-7i64)`
- `42.0` → `Value.Float(42.0f64)`
- `1e10` → `Value.Float(1e10f64)` — exponent notation → float
- `9999999999999999999` → `Value.Float(...)` — too large for i64, promotes to float with a diagnostic warning; precision may be lost

This matches the approach of CCSP's `lib_ccsp_json` reference implementation.

### Security Limits and the JSON Parser Amplification Problem

Deeply nested JSON documents are the ReDoS equivalent for JSON parsers. A document with 100,000 nested arrays — `[[[[...]]]]` — can consume stack space proportional to its depth. Without a limit, a single 200-byte document can crash a server via stack overflow. This is not hypothetical: it has caused real service outages.

This module enforces hard limits by default and makes all limits configurable via `ParseConfig`. The defaults are:

| Limit | Default | Rationale |
|---|---|---|
| `max_depth` | 64 | Deeper nesting is pathological in practice; RFC 8259 recommends implementations impose limits |
| `max_string_len` | 1 MiB | Single JSON strings should not be megabytes in normal use |
| `max_document_len` | 16 MiB | Documents larger than this are datasets, not API responses |

All limits are checked before allocation, not after. A crafted document that would exceed a limit returns `Err(JsonError)` immediately; it does not consume proportional memory first.

### JSONTestSuite Compliance

This module passes all mandatory cases from Nicolas Seriot's JSONTestSuite:

- All `y_*` test cases (must accept) are accepted
- All `n_*` test cases (must reject) are rejected
- `i_*` test cases (implementation-defined) have documented behavior

---

## 2. Dynamic Value Type

The `Value` type represents any JSON value in memory. It is the core type for DOM-style (full-document) parsing and construction.

```ferrum
use extlib.ccsp.json.Value

@derive(Debug, Clone, PartialEq)
enum Value {
    Null,
    Bool(bool),
    Integer(i64),
    Float(f64),
    String(String),
    Array(Vec[Value]),
    Object(IndexMap[String, Value]),
}
```

### 2.1 Object Key Ordering

`Object` uses `IndexMap[String, Value]` — an insertion-order-preserving map from `alloc.collections`. RFC 8259 §4 states that JSON objects are unordered, and that implementations should not depend on order. However, well-behaved parsers preserve insertion order because:

- Many real-world JSON producers write keys in a meaningful order (e.g., `id` before `name` before `data`)
- Round-tripping a parsed document should produce semantically identical output
- Diffing JSON in version control is dramatically more readable when order is stable

`IndexMap` gives O(1) lookup by key and O(1) index access, preserving insertion order at no asymptotic cost.

### 2.2 Accessor Methods

```ferrum
impl Value {
    // Type-testing predicates
    fn is_null(&self): bool
    fn is_bool(&self): bool
    fn is_integer(&self): bool
    fn is_float(&self): bool
    fn is_number(&self): bool     // true for both Integer and Float
    fn is_string(&self): bool
    fn is_array(&self): bool
    fn is_object(&self): bool

    // Type-converting accessors — return None if the value is a different variant
    fn as_bool(&self): Option[bool]
    fn as_i64(&self): Option[i64]
    fn as_f64(&self): Option[f64]   // also coerces Integer to f64
    fn as_str(&self): Option[&str]
    fn as_array(&self): Option[&Vec[Value]]
    fn as_array_mut(&mut self): Option[&mut Vec[Value]]
    fn as_object(&self): Option[&IndexMap[String, Value]]
    fn as_object_mut(&mut self): Option[&mut IndexMap[String, Value]]

    // Key lookup in an Object — returns None if not an Object or key absent
    fn get(&self, key: &str): Option[&Value]
    fn get_mut(&mut self, key: &str): Option[&mut Value]

    // Index lookup in an Array — returns None if not an Array or index out of bounds
    fn get_index(&self, index: usize): Option[&Value]

    // Take ownership of an Object field or Array element
    fn remove(&mut self, key: &str): Option[Value]   // Object only
}

// Indexing operators — panic on wrong variant or absent key/index
// Use .get() and .get_index() for non-panicking access
impl core.ops.Index[&str] for Value {
    type Output = Value
    // Panics if not an Object or if key is absent
}

impl core.ops.Index[usize] for Value {
    type Output = Value
    // Panics if not an Array or if index is out of bounds
}
```

### 2.3 Display and Serialization

```ferrum
impl fmt.Display for Value {
    // Produces compact JSON output (no extra whitespace).
    // Equivalent to serialize(&self).
    fn fmt(&self, f: &mut Formatter): Result[(), IoError] ! IO
}

impl fmt.Debug for Value {
    // Produces pretty-printed JSON (indent = 2).
    // Useful for logging and debugging.
    fn fmt(&self, f: &mut Formatter): Result[(), IoError] ! IO
}
```

### 2.4 Construction Convenience

```ferrum
impl Value {
    // Literal constructors
    fn null(): Self         { Value.Null }
    fn bool_(v: bool): Self { Value.Bool(v) }
    fn integer(v: i64): Self { Value.Integer(v) }
    fn float_(v: f64): Self  { Value.Float(v) }
    fn string_(v: impl Into[String]): Self { Value.String(v.into()) }
    fn array(items: Vec[Value]): Self { Value.Array(items) }
    fn object(pairs: Vec[(String, Value)]): Self {
        // Constructs an IndexMap from the pairs, preserving order
        Value.Object(IndexMap.from_iter(pairs))
    }
}
```

---

## 3. Parsing (DOM)

### 3.1 Top-Level Functions

```ferrum
// Parse a JSON document from a UTF-8 string.
// Uses default ParseConfig: max_depth=64, max_string=1MiB, max_document=16MiB.
fn parse(input: &str): Result[Value, JsonError]

// Parse a JSON document from a byte slice.
// Validates UTF-8; returns JsonError if the input is not valid UTF-8.
// Rejects a leading UTF-8 BOM (0xEF 0xBB 0xBF) — RFC 8259 §8.1 prohibits it.
fn parse_bytes(input: &[u8]): Result[Value, JsonError]

// Parse with explicit configuration.
fn parse_with_config(input: &str, config: &ParseConfig): Result[Value, JsonError]

// Parse from an io.Read stream.
// Reads only as many bytes as needed to complete the document.
fn parse_reader(reader: &mut impl Read, config: &ParseConfig): Result[Value, JsonError] ! IO
```

### 3.2 ParseConfig

```ferrum
@derive(Debug, Clone)
type ParseConfig {
    // Maximum nesting depth of arrays and objects.
    // Each opening '[' or '{' increments the depth counter.
    // Exceeding this limit returns JsonError immediately.
    // Default: 64
    pub max_depth: usize,

    // Maximum length in bytes of any single JSON string value (after unescaping).
    // Default: 1_048_576 (1 MiB)
    pub max_string_len: usize,

    // Maximum total length in bytes of the input document.
    // Default: 16_777_216 (16 MiB)
    pub max_document_len: usize,

    // Accept C-style // line comments and /* block */ comments.
    // Not valid per RFC 8259; useful for human-written config files.
    // Default: false
    pub allow_comments: bool,

    // Accept trailing commas in arrays and objects: [1, 2,] and {"a": 1,}.
    // Not valid per RFC 8259; useful for human-written config files.
    // Default: false
    pub allow_trailing_commas: bool,
}

impl ParseConfig {
    fn default(): Self {
        ParseConfig {
            max_depth:            64,
            max_string_len:       1_048_576,
            max_document_len:     16_777_216,
            allow_comments:       false,
            allow_trailing_commas: false,
        }
    }

    // Convenience constructor for parsing config files.
    // Enables comments and trailing commas; other limits unchanged.
    fn for_config_files(): Self {
        ParseConfig {
            allow_comments:        true,
            allow_trailing_commas: true,
            ..Self.default()
        }
    }
}
```

### 3.3 Error Type

```ferrum
@derive(Debug, Clone)
type JsonError {
    // Human-readable description of the parse failure.
    pub message: String,
    // 1-based line number of the offending character in the input.
    pub line:    u32,
    // 1-based column number (in Unicode scalar values, not bytes).
    pub column:  u32,
    // Byte offset of the offending character in the input.
    pub offset:  usize,
}

impl fmt.Display for JsonError {
    // "JSON parse error at line 3, column 12: expected ',' or ']'"
}

impl core.error.Error for JsonError {}
```

### 3.4 Parser Behavior

**UTF-8 and BOM.** The parser requires valid UTF-8. A byte sequence that is not valid UTF-8 is a parse error, not a replacement-character substitution. A UTF-8 BOM at the start of the document is also a parse error. RFC 8259 §8.1 explicitly states that implementations must not add a BOM.

**Number parsing.** Numbers with no `.` or `e`/`E` are parsed as `i64` if they fit in `[-2^63, 2^63-1]`. Numbers that do not fit are parsed as `f64` with a diagnostic (a warning, not an error — the parse succeeds). Numbers with a decimal point or exponent are always `f64`.

**Whitespace.** The four whitespace characters defined by RFC 8259 — space (0x20), tab (0x09), newline (0x0A), carriage return (0x0D) — are accepted and ignored. No other characters are treated as whitespace.

**Duplicate keys.** RFC 8259 says object key uniqueness is undefined. This implementation retains the last value for duplicate keys, matching the behavior of most JSON parsers. The order of retained keys in the `IndexMap` reflects their position of last occurrence.

**JSONTestSuite compliance.** All `y_*` (must-accept) and `n_*` (must-reject) cases pass. Selected `i_*` behavior:

| Case | Behavior |
|---|---|
| `i_number_double_huge_neg_exp` | Parsed as `Float`, value may be 0.0 or -0.0 per IEEE 754 |
| `i_number_huge_exp` | Parsed as `Float(Infinity)` — not an error |
| `i_number_too_big_neg_int` | Parsed as `Float` with precision warning |
| `i_number_too_big_pos_int` | Parsed as `Float` with precision warning |
| `i_object_key_lone_2nd_surrogate` | Parse error — invalid UTF-8 in key |
| `i_string_lone_surrogate_utf8` | Parse error — invalid UTF-8 |

---

## 4. Serialization

### 4.1 DOM Serialization

```ferrum
// Compact output — no spaces, no newlines.
// "{"name":"Alice","age":42}"
fn serialize(value: &Value): String

// Pretty-printed output.
// indent is the number of spaces per nesting level; typical values: 2 or 4.
fn serialize_pretty(value: &Value, indent: usize): String

// Write to any io.Write — no intermediate String allocation.
// Effect ! IO is required because writing to a writer is an I/O operation.
fn serialize_to(value: &Value, writer: &mut impl Write): Result[(), IoError] ! IO

// Write pretty-printed to any io.Write.
fn serialize_pretty_to(
    value: &Value,
    writer: &mut impl Write,
    indent: usize,
): Result[(), IoError] ! IO
```

### 4.2 Serialization Guarantees

- Output is always valid UTF-8.
- Control characters (0x00–0x1F, 0x7F) in string values are escaped as `\uXXXX`.
- Forward slashes `/` are not escaped (RFC 8259 permits but does not require it).
- Non-ASCII Unicode characters are written as their UTF-8 representation, not `\uXXXX` escapes.
- `Float` values: `NaN` and `Infinity` are not representable in JSON. Serializing them returns `Err(IoError)` from `serialize_to`, or panics in `serialize` and `serialize_pretty`. Callers must sanitize `f64` values before serialization.
- Integer values are written without decimal point or exponent: `Value.Integer(42)` → `42`, never `42.0`.
- No trailing commas.
- Output is RFC 8259 compliant.

---

## 5. Typed Deserialization

### 5.1 The FromJson Trait

```ferrum
trait FromJson: Sized {
    // Deserialize from a borrowed Value.
    // Returns Err if the value's structure does not match the expected type.
    fn from_json(value: &Value): Result[Self, JsonDeserError]
}
```

Top-level convenience functions:

```ferrum
// Parse and immediately deserialize.
fn from_str[T: FromJson](input: &str): Result[T, JsonError]

// Deserialize from an already-parsed Value.
fn from_value[T: FromJson](value: &Value): Result[T, JsonDeserError]
```

### 5.2 Standard Implementations

| Ferrum type | JSON source | Notes |
|---|---|---|
| `bool` | `Bool` | |
| `i8`, `i16`, `i32`, `i64` | `Integer` | Range-checked; returns `JsonDeserError` on overflow |
| `u8`, `u16`, `u32`, `u64` | `Integer` | Range-checked; rejects negative values |
| `f32`, `f64` | `Float` or `Integer` | `Integer` is coerced to float |
| `String` | `String` | |
| `Option[T]` | `Null` → `None`, anything → `Some(T::from_json(...))` | |
| `Vec[T]` | `Array` | Each element deserialized as `T` |
| `HashMap[String, T]` | `Object` | Key order not preserved |
| `IndexMap[String, T]` | `Object` | Key order preserved |
| `Value` | any | Identity; always succeeds |

### 5.3 Derive Macro for Structs

```ferrum
@derive(FromJson)
type User {
    pub id:    i64,
    pub name:  String,
    pub email: String,
}
// Deserializes from: {"id": 1, "name": "Alice", "email": "alice@example.com"}
// All fields are required unless marked #[json(default)].
// Unknown fields in the JSON object are ignored by default.
```

### 5.4 Field Attributes

```ferrum
@derive(FromJson)
type ApiResponse {
    // Rename: the JSON key is "userId", the Ferrum field is user_id
    #[json(rename = "userId")]
    pub user_id: i64,

    // Default: if the key is absent, use Default.default() instead of failing
    #[json(default)]
    pub page_size: usize,

    // Flatten: the fields of nested_obj are deserialized from the parent object,
    // not from a nested JSON object.
    // The flattened type must also implement FromJson from an Object.
    #[json(flatten)]
    pub metadata: RequestMetadata,
}
```

Attribute reference:

| Attribute | Effect |
|---|---|
| `#[json(rename = "key")]` | Use `"key"` as the JSON field name instead of the Ferrum field name |
| `#[json(default)]` | If the key is absent, use `T::default()` instead of returning an error |
| `#[json(default = "path::to::fn")]` | If the key is absent, call the specified function to produce the default |
| `#[json(flatten)]` | Deserialize this field's fields from the parent object |
| `#[json(skip)]` | Never deserialize this field; always use `T::default()` |
| `#[json(deny_unknown_fields)]` | On the struct: fail if the JSON object contains any key not in the struct |

### 5.5 Derive Macro for Enums

```ferrum
@derive(FromJson)
#[json(tag = "type")]
enum Shape {
    // Deserializes when JSON has "type": "circle"
    Circle { radius: f64 },
    // Deserializes when JSON has "type": "rect"
    Rect { width: f64, height: f64 },
}
// {"type": "circle", "radius": 1.5} → Shape.Circle { radius: 1.5 }
// {"type": "rect", "width": 10.0, "height": 5.0} → Shape.Rect { width: 10.0, height: 5.0 }
```

Enum representation options:

| Attribute | Description | Example |
|---|---|---|
| `#[json(tag = "t")]` | Internally tagged: discriminant is a field named `t` | `{"t": "Foo", "x": 1}` |
| `#[json(tag = "t", content = "c")]` | Adjacently tagged: tag and content are separate fields | `{"t": "Foo", "c": {"x": 1}}` |
| `#[json(untagged)]` | Try each variant in order; first that succeeds wins | `{"x": 1}` |

The default (no attribute) is externally tagged: `{"VariantName": {...payload...}}`.

### 5.6 Deserialization Error Type

```ferrum
@derive(Debug, Clone)
type JsonDeserError {
    // Human-readable description of the type mismatch or missing field.
    pub message: String,
    // JSONPath-style path to the failing location.
    // Examples: "$", "$.users", "$.users[0]", "$.users[0].name"
    pub path: JsonPath,
}

type JsonPath(String)

impl fmt.Display for JsonDeserError {
    // "type error at $.users[0].age: expected Integer, got String"
}

impl core.error.Error for JsonDeserError {}
```

---

## 6. Typed Serialization

### 6.1 The ToJson Trait

```ferrum
trait ToJson {
    fn to_json(&self): Value
}
```

Top-level convenience functions:

```ferrum
// Serialize a typed value to a compact JSON string.
fn to_string[T: ToJson](value: &T): String

// Serialize a typed value to a pretty-printed JSON string.
fn to_string_pretty[T: ToJson](value: &T, indent: usize): String

// Serialize to a Value — equivalent to value.to_json().
fn to_value[T: ToJson](value: &T): Value
```

### 6.2 Standard Implementations

`ToJson` is implemented for the same types as `FromJson`:

```
bool, i8–i64, u8–u64, f32, f64, String, &str,
Option[T: ToJson], Vec[T: ToJson],
HashMap[String, T: ToJson], IndexMap[String, T: ToJson],
Value (identity)
```

`f32` and `f64` serialization: `NaN` and `Infinity` are not representable in JSON. `ToJson` for floats produces `Value.Null` for non-finite values and emits a diagnostic warning. This is a lossy conversion; callers that cannot tolerate this should validate their floats before calling `to_json()`.

### 6.3 Derive Macro for Structs

```ferrum
@derive(ToJson)
type User {
    pub id:    i64,
    pub name:  String,
    pub email: String,
}
// User { id: 1, name: "Alice", email: "alice@example.com" }
// → {"id": 1, "name": "Alice", "email": "alice@example.com"}
```

The same `#[json(...)]` field attributes from `FromJson` apply:

- `#[json(rename = "key")]` — write this key name in the output object
- `#[json(skip)]` — omit this field from the output entirely
- `#[json(skip_if = "Option.is_none")]` — omit this field if the predicate returns true
- `#[json(flatten)]` — merge this field's key-value pairs into the parent object

### 6.4 Derive Macro for Enums

```ferrum
@derive(ToJson)
#[json(tag = "type")]
enum Shape {
    Circle { radius: f64 },
    Rect { width: f64, height: f64 },
}
// Shape.Circle { radius: 1.5 } → {"type": "circle", "radius": 1.5}
```

The variant name is converted to lowercase by default. Use `#[json(rename = "Circle")]` on the variant to override.

---

## 7. Streaming Parser (SAX-Style)

The DOM parser loads the entire document into memory as a `Value` tree. For documents that are too large to fit in memory — log files, data exports, multi-gigabyte datasets — the streaming parser processes the document event by event without accumulating a tree.

### 7.1 Event Type

```ferrum
@derive(Debug, Clone, PartialEq)
enum JsonEvent {
    // Start of a JSON object: the next events will be alternating Key and value events
    StartObject,
    // End of the current JSON object
    EndObject,
    // Start of a JSON array: the next events will be value events
    StartArray,
    // End of the current JSON array
    EndArray,
    // A key within an object — always followed immediately by a value event
    Key(String),
    // Value events — one of these follows each Key in an object, or appears in an array
    ValueNull,
    ValueBool(bool),
    ValueInteger(i64),
    ValueFloat(f64),
    ValueString(String),
}
```

### 7.2 JsonParser Type

```ferrum
type JsonParser[R: Read]

impl[R: Read] JsonParser[R] {
    // Create a streaming parser from any io.Read.
    fn new(reader: R): Self

    // Create with explicit limits.
    fn with_config(reader: R, config: ParseConfig): Self

    // Advance to the next event.
    // Returns Ok(Some(event)) for each event in the document.
    // Returns Ok(None) after the root value is complete.
    // Returns Err on parse error or I/O error.
    fn next_event(&mut self): Result[Option[JsonEvent], JsonError] ! IO

    // Current nesting depth (number of open arrays/objects not yet closed).
    fn depth(&self): usize
}
```

### 7.3 Usage: Processing a Large JSON Array

The common pattern for large files is a streaming array of records:

```ferrum
use extlib.ccsp.json.{JsonParser, JsonEvent}
use std.io.BufReader
use std.fs.File

fn process_large_log(path: &str): Result[(), JsonError] ! IO {
    let file = File.open(path)?
    let reader = BufReader.new(file)
    let mut parser = JsonParser.new(reader)

    // Expect the root to be an array
    match parser.next_event()? {
        Some(JsonEvent.StartArray) => {}
        other => return Err(JsonError {
            message: format("expected array, got {:?}", other),
            line: 1, column: 1, offset: 0,
        })
    }

    let mut count: u64 = 0
    loop {
        match parser.next_event()? {
            Some(JsonEvent.EndArray) | None => break,
            Some(JsonEvent.StartObject) => {
                // Consume one object worth of events
                let record = read_log_record(&mut parser)?
                process_record(record)?
                count += 1
            }
            Some(other) => return Err(JsonError {
                message: format("expected object in array, got {:?}", other),
                line: 0, column: 0, offset: 0,
            })
        }
    }

    println("processed {} records", count)
    Ok(())
}

fn read_log_record[R: Read](
    parser: &mut JsonParser[R],
): Result[LogRecord, JsonError] ! IO {
    let mut level = String.new()
    let mut message = String.new()
    let mut timestamp: i64 = 0

    // The caller already consumed StartObject; consume key-value pairs until EndObject
    loop {
        match parser.next_event()? {
            Some(JsonEvent.EndObject) => break,
            Some(JsonEvent.Key(k)) => {
                match k.as_str() {
                    "level" => {
                        if let Some(JsonEvent.ValueString(v)) = parser.next_event()? {
                            level = v
                        }
                    }
                    "message" => {
                        if let Some(JsonEvent.ValueString(v)) = parser.next_event()? {
                            message = v
                        }
                    }
                    "ts" => {
                        if let Some(JsonEvent.ValueInteger(v)) = parser.next_event()? {
                            timestamp = v
                        }
                    }
                    _ => {
                        // Skip unknown keys by consuming the value
                        skip_value(parser)?
                    }
                }
            }
            _ => break,
        }
    }

    Ok(LogRecord { level, message, timestamp })
}

fn skip_value[R: Read](parser: &mut JsonParser[R]): Result[(), JsonError] ! IO {
    let mut depth: usize = 0
    loop {
        match parser.next_event()? {
            Some(JsonEvent.StartObject) | Some(JsonEvent.StartArray) => depth += 1,
            Some(JsonEvent.EndObject)   | Some(JsonEvent.EndArray)   => {
                if depth == 0 { return Ok(()) }
                depth -= 1
            }
            Some(_) => { if depth == 0 { return Ok(()) } }
            None => return Ok(()),
        }
    }
}
```

### 7.4 Security Limits

The streaming parser enforces the same limits as the DOM parser:

- `max_depth`: `StartObject` and `StartArray` increment the depth counter. Exceeding `max_depth` returns `Err(JsonError)`.
- `max_string_len`: Applies to both `Key(String)` and `ValueString(String)` events.
- `max_document_len`: The parser tracks total bytes consumed; exceeding this limit returns `Err(JsonError)`.

---

## 8. Streaming Serializer

The streaming serializer writes JSON directly to an `io.Write` without building an intermediate `Value` tree. Use it when constructing large JSON outputs that would be expensive to accumulate in memory.

```ferrum
type JsonSerializer[W: Write]

impl[W: Write] JsonSerializer[W] {
    // Create a compact (no whitespace) streaming serializer.
    fn new(writer: W): Self

    // Create a pretty-printing serializer.
    // indent: spaces per nesting level (typically 2 or 4)
    fn pretty(writer: W, indent: usize): Self

    // Begin a JSON object. Must be followed by alternating .key() and value calls,
    // then ended by .end_object().
    fn begin_object(&mut self): Result[(), IoError] ! IO

    // End the current JSON object.
    fn end_object(&mut self): Result[(), IoError] ! IO

    // Begin a JSON array. Must be followed by value calls, then .end_array().
    fn begin_array(&mut self): Result[(), IoError] ! IO

    // End the current JSON array.
    fn end_array(&mut self): Result[(), IoError] ! IO

    // Write an object key. Must be called inside begin_object/end_object.
    fn key(&mut self, k: &str): Result[(), IoError] ! IO

    // Write individual value types.
    fn value_null(&mut self): Result[(), IoError] ! IO
    fn value_bool(&mut self, v: bool): Result[(), IoError] ! IO
    fn value_i64(&mut self, v: i64): Result[(), IoError] ! IO
    fn value_f64(&mut self, v: f64): Result[(), IoError] ! IO
        // NaN and Infinity: returns Err(IoError) rather than producing invalid JSON
    fn value_str(&mut self, v: &str): Result[(), IoError] ! IO
    fn value_string(&mut self, v: &String): Result[(), IoError] ! IO

    // Write a dynamic Value (DOM value) into the stream.
    fn value(&mut self, v: &Value): Result[(), IoError] ! IO

    // Flush and close. Must be called exactly once when done.
    // Returns Err if the document is structurally incomplete (e.g., unclosed array).
    fn finish(self): Result[(), IoError] ! IO
}
```

### 8.1 Usage: Building a JSON Response

```ferrum
use extlib.ccsp.json.JsonSerializer
use std.io.StringWriter

fn build_user_list_response(users: &[User]): Result[String, IoError] ! IO {
    let mut writer = StringWriter.new()
    let mut ser = JsonSerializer.new(&mut writer)

    ser.begin_object()?
    ser.key("users")?
    ser.begin_array()?

    for user in users {
        ser.begin_object()?
        ser.key("id")?
        ser.value_i64(user.id)?
        ser.key("name")?
        ser.value_str(&user.name)?
        ser.key("email")?
        ser.value_str(&user.email)?
        ser.end_object()?
    }

    ser.end_array()?
    ser.key("count")?
    ser.value_i64(users.len() as i64)?
    ser.end_object()?
    ser.finish()?

    Ok(writer.into_string())
}
```

### 8.2 Separator and Structural Tracking

The serializer tracks structural state internally. It inserts commas between array elements and between object key-value pairs automatically. Callers do not write commas. The sequence `.key("a")` followed by `.key("b")` (without an intervening value) returns `Err(IoError)` with a message describing the structural violation.

---

## 9. JSON Pointer (RFC 6901)

JSON Pointer defines a string syntax for identifying a specific value within a JSON document. A pointer is a sequence of reference tokens separated by `/`. Each token is either an object key or an array index.

Examples:

| Pointer | Selects |
|---|---|
| `""` | The root document |
| `"/foo"` | `root["foo"]` |
| `"/foo/0"` | `root["foo"][0]` |
| `"/a~1b"` | `root["a/b"]` — `~1` is the escaped `/` |
| `"/m~0n"` | `root["m~n"]` — `~0` is the escaped `~` |

```ferrum
@derive(Debug, Clone, PartialEq)]
type JsonPointer(Vec[String])   // each element is a decoded reference token

impl JsonPointer {
    // Parse a JSON Pointer string.
    // The empty string "" is valid and refers to the root.
    // Returns Err if the string is non-empty and does not start with '/'.
    fn parse(ptr: &str): Result[Self, JsonPointerError]

    // Return the number of reference tokens.
    fn len(&self): usize

    // True if this is the root pointer (zero tokens).
    fn is_root(&self): bool

    // Get the value at the location this pointer identifies.
    // Returns None if any step along the path does not exist.
    fn get<'a>(&self, value: &'a Value): Option[&'a Value]

    // Get a mutable reference to the value at this pointer's location.
    fn get_mut<'a>(&self, value: &'a mut Value): Option[&'a mut Value]

    // Serialize back to a JSON Pointer string (with ~ and / escaped).
    fn to_string(&self): String
}

impl fmt.Display for JsonPointer {
    // Same as to_string()
}

@derive(Debug, Clone)]
type JsonPointerError {
    pub message: String,
    pub offset:  usize,   // byte offset in the pointer string where parsing failed
}

impl fmt.Display for JsonPointerError { ... }
impl core.error.Error for JsonPointerError {}
```

### 9.1 Usage

```ferrum
use extlib.ccsp.json.{parse, JsonPointer}

fn extract_user_name(doc: &str): Result[Option[String], JsonError] {
    let value = parse(doc)?
    let ptr = JsonPointer.parse("/users/0/name").unwrap()
    let name = ptr.get(&value)
        .and_then(fn(v) { v.as_str() })
        .map(fn(s) { s.to_owned() })
    Ok(name)
}
```

---

## 10. JSON Merge Patch (RFC 7396)

JSON Merge Patch defines a format and algorithm for describing changes to a JSON document. The patch is itself a JSON value. The algorithm:

- If the patch is not an object, the target is replaced entirely by the patch.
- If the patch is an object, for each key in the patch:
  - If the patch value is `null`, remove the key from the target.
  - Otherwise, recursively merge the patch value into the target at that key.

```ferrum
// Apply a merge patch to a target document, modifying target in place.
// RFC 7396 §2 algorithm.
fn apply_merge_patch(target: &mut Value, patch: &Value)

// Produce a merge patch that transforms `source` into `target`.
// The result can be applied with apply_merge_patch(source, result) to reproduce target.
fn diff_merge_patch(source: &Value, target: &Value): Value
```

### 10.1 Usage

```ferrum
use extlib.ccsp.json.{parse, apply_merge_patch}

fn apply_config_patch(config_json: &str, patch_json: &str): Result[String, JsonError] {
    let mut config = parse(config_json)?
    let patch = parse(patch_json)?
    apply_merge_patch(&mut config, &patch)
    Ok(config.to_string())
}

// config: {"host": "localhost", "port": 5432, "debug": true}
// patch:  {"debug": null, "port": 5433}
// result: {"host": "localhost", "port": 5433}
// — "debug" was null in the patch, so it was removed
// — "port" was updated to 5433
// — "host" was not in the patch, so it was left unchanged
```

---

## 11. Security Limits and JSONTestSuite Compliance

### 11.1 Security Limits Summary

| Limit | Config field | Default | What it prevents |
|---|---|---|---|
| Nesting depth | `max_depth` | 64 | Stack overflow via deeply nested `[[[[...]]]]` |
| String length | `max_string_len` | 1 MiB | Memory exhaustion from a single huge string value |
| Document length | `max_document_len` | 16 MiB | Memory exhaustion from a huge document |

All limits apply to both the DOM parser (`parse`, `parse_bytes`) and the streaming parser (`JsonParser`). The limits are enforced as the document is consumed, before allocation proportional to the excess occurs.

### 11.2 Limit Adjustment Guidance

Applications that process JSON from untrusted sources (public REST APIs, user file uploads, network peers) should use the defaults or tighten them. Applications that process trusted internal data (config files authored by developers, messages from co-located services) may raise limits or disable them:

```ferrum
let config = ParseConfig {
    max_depth:        256,
    max_string_len:   64 * 1024 * 1024,   // 64 MiB for trusted large data
    max_document_len: 512 * 1024 * 1024,  // 512 MiB
    ..ParseConfig.default()
};
```

There is no `max_depth: 0` meaning "unlimited." To effectively disable depth limiting, set `max_depth: usize.MAX`.

### 11.3 JSONTestSuite

The test suite is maintained at <https://github.com/nst/JSONTestSuite>. This module's test suite includes all 318 test inputs from the suite as committed byte-exact fixtures. Tests are fully offline — no network access required. The fixture files are committed to the `extlib.ccsp.json` test directory alongside the source.

Mandatory conformance:

- **`y_*` cases (must accept):** All 95 cases accepted.
- **`n_*` cases (must reject):** All 188 cases rejected.
- **`i_*` cases (implementation-defined):** 35 cases; behavior documented in §3.4 above.

---

## 12. Example Usage

### 12.1 Parse a REST API Response

```ferrum
use extlib.ccsp.json.{parse, Value}

fn fetch_user_name(json_response: &str): Result[String, JsonError] {
    let value = parse(json_response)?
    let name = value
        .get("user")
        .and_then(fn(u) { u.get("name") })
        .and_then(fn(n) { n.as_str() })
        .ok_or_else(fn() {
            JsonError { message: "missing $.user.name".to_owned(), line: 0, column: 0, offset: 0 }
        })?
        .to_owned()
    Ok(name)
}

fn main() ! IO {
    // Simulated API response
    let response = r#"{"user": {"id": 42, "name": "Alice", "active": true}}"#
    let name = fetch_user_name(response).unwrap()
    println("user: {}", name)  // "user: Alice"
}
```

### 12.2 Typed Deserialization

```ferrum
use extlib.ccsp.json.{from_str, to_string}

@derive(FromJson, ToJson, Debug)
type GithubRepo {
    pub id:          i64,
    #[json(rename = "full_name")]
    pub full_name:   String,
    pub description: Option[String],
    #[json(rename = "stargazers_count")]
    pub stars:       u32,
    pub private:     bool,
}

fn print_repo_info(json: &str) ! IO {
    match from_str::[GithubRepo](json) {
        Ok(repo) => {
            println("repo: {}", repo.full_name)
            println("stars: {}", repo.stars)
            match repo.description {
                Some(d) => println("description: {}", d),
                None    => println("(no description)"),
            }
        }
        Err(e) => println("parse error: {}", e),
    }
}
```

### 12.3 Streaming a Large File

```ferrum
use extlib.ccsp.json.{JsonParser, JsonEvent, ParseConfig}
use std.fs.File
use std.io.BufReader

// Count total records and sum a numeric field in a large JSON array
// without loading the whole document into memory.
fn sum_amounts(path: &str): Result[i64, JsonError] ! IO {
    let config = ParseConfig {
        max_document_len: usize.MAX,  // trusted internal data, no size cap
        ..ParseConfig.default()
    }
    let reader = BufReader.new(File.open(path)?)
    let mut parser = JsonParser.with_config(reader, config)

    // Expect root array
    match parser.next_event()? {
        Some(JsonEvent.StartArray) => {}
        _ => return Err(JsonError {
            message: "expected root array".to_owned(),
            line: 1, column: 1, offset: 0,
        })
    }

    let mut total: i64 = 0
    loop {
        match parser.next_event()? {
            Some(JsonEvent.StartObject) => {}
            Some(JsonEvent.EndArray) | None => break,
            _ => continue,
        }

        // Inside one record: scan for "amount" key
        loop {
            match parser.next_event()? {
                Some(JsonEvent.Key(k)) if k == "amount" => {
                    match parser.next_event()? {
                        Some(JsonEvent.ValueInteger(v)) => total += v,
                        Some(JsonEvent.ValueFloat(v))   => total += v as i64,
                        _ => {}
                    }
                }
                Some(JsonEvent.EndObject) | None => break,
                _ => {}
            }
        }
    }

    Ok(total)
}
```

### 12.4 Building a JSON Response with the Streaming Serializer

```ferrum
use extlib.ccsp.json.JsonSerializer
use std.io.StringWriter

type HealthStatus {
    pub service: String,
    pub ok:      bool,
    pub latency_ms: i64,
}

fn build_health_response(statuses: &[HealthStatus]): Result[String, IoError] ! IO {
    let mut writer = StringWriter.new()
    let mut ser = JsonSerializer.new(&mut writer)

    ser.begin_object()?
    ser.key("status")?
    let all_ok = statuses.iter().all(fn(s) { s.ok })
    ser.value_str(if all_ok { "ok" } else { "degraded" })?
    ser.key("services")?
    ser.begin_array()?

    for status in statuses {
        ser.begin_object()?
        ser.key("service")?
        ser.value_str(&status.service)?
        ser.key("ok")?
        ser.value_bool(status.ok)?
        ser.key("latency_ms")?
        ser.value_i64(status.latency_ms)?
        ser.end_object()?
    }

    ser.end_array()?
    ser.end_object()?
    ser.finish()?

    Ok(writer.into_string())
}
```

### 12.5 JSON Pointer Navigation

```ferrum
use extlib.ccsp.json.{parse, JsonPointer}

fn get_nested(doc: &str, pointer: &str): Option[String] {
    let value = parse(doc).ok()?
    let ptr = JsonPointer.parse(pointer).ok()?
    ptr.get(&value)?.as_str().map(fn(s) { s.to_owned() })
}

fn main() ! IO {
    let doc = r#"{"config": {"database": {"host": "db.internal", "port": 5432}}}"#
    println("{:?}", get_nested(doc, "/config/database/host"))
    // Some("db.internal")
    println("{:?}", get_nested(doc, "/config/database/timeout"))
    // None
}
```

### 12.6 JSON Merge Patch

```ferrum
use extlib.ccsp.json.{parse, apply_merge_patch, serialize}

fn main() ! IO {
    let mut config = parse(r#"
        {"host": "localhost", "port": 5432, "debug": true, "pool_size": 10}
    "#).unwrap()

    let patch = parse(r#"
        {"debug": null, "port": 5433, "pool_size": 20}
    "#).unwrap()

    apply_merge_patch(&mut config, &patch)

    println("{}", serialize(&config))
    // {"host":"localhost","port":5433,"pool_size":20}
    // — "debug" was removed (null in patch)
    // — "port" and "pool_size" were updated
    // — "host" was unchanged (absent from patch)
}
```

---

## 13. Dependencies

`extlib.ccsp.json` depends only on Ferrum standard library modules. It has no extlib dependencies. This is a deliberate design constraint: JSON is ubiquitous enough that adding extlib transitive dependencies would be surprising, and the functionality needed (allocation, I/O, collections) is entirely within stdlib.

| Module | Used for |
|---|---|
| `alloc.string` | `String` — owned string storage for values and error messages |
| `alloc.vec` | `Vec[T]` — dynamic arrays for `Value.Array` and internal buffers |
| `alloc.box_` | `Box[T]` — not used directly; pulled in via alloc |
| `collections.indexmap` | `IndexMap[String, Value]` — insertion-order-preserving object storage |
| `collections.hashmap` | `HashMap[String, T]` — for `FromJson`/`ToJson` impls on `HashMap` |
| `core.str` | `&str`, UTF-8 validation |
| `core.ops` | `Index`, `Range` |
| `core.error` | `Error` trait for `JsonError` and `JsonDeserError` |
| `core.fmt` | `Display`, `Debug` formatting |
| `io` | `Read`, `Write` — for streaming parser and serializer |

`extlib.ccsp.json` does not depend on:

- Any other extlib module
- `net`, `async`, `http` — no networking
- `sys`, `posix`, `windows` — no OS calls
- `crypto` — no cryptographic operations
- `std.data.cbor` — CBOR and JSON are independent modules with no shared code

The `! IO` effect appears only on functions that accept or produce `impl Read` / `impl Write`. Parsing from `&str` and serializing to `String` carry no effect annotations — they are pure computations that happen to allocate.

---

*Part of the CCSP (Constrained and Cryptographic Systems Protocol) extended library suite.*
*See also: `std.data.cbor`, `extlib.ccsp.coap`, `extlib.regex`*
