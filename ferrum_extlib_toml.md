# Ferrum Extended Library — toml

**Module:** `extlib.toml`
**Part of:** Extended Standard Library (not `std` or `core`)
**Companion:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 1. Overview and Rationale

### TOML as the Preferred Config File Format

Configuration files are a fact of life for systems programs: servers need ports and TLS paths, CLIs need default flags, daemons need logging levels. The format question is where most languages go quietly wrong.

**YAML** is superficially appealing but catastrophic in practice. The spec is 80 pages. Indentation is semantically significant. The Norway problem (`NO` parses as `false`). Multiple incompatible document types in one stream. YAML parsers are large, complex, and have a documented history of security vulnerabilities in every language ecosystem. YAML is not a configuration file format; it is a serialization language that happens to be human-writable with great difficulty.

**JSON** was never designed for humans to write. No comments. Trailing commas are a syntax error. String-only keys. Ergonomically hostile for anything a person needs to maintain.

**INI/TOML** occupy the same conceptual space: simple key-value with optional sections. TOML wins because it is precisely specified (v1.0 is stable and frozen), has a clean type system (integers vs. floats vs. strings vs. booleans vs. datetimes are unambiguous), supports arrays and nested tables, and is genuinely human-writable. The TOML test suite (`toml-test`) provides a conformance harness that makes "does this parser actually implement TOML?" answerable.

### Why extlib, Not stdlib

The Ferrum standard library contains modules that nearly every program needs. Config file parsing does not meet this bar. A kernel module, a bootloader, a cryptographic library, a game engine, a network protocol implementation — none of these read TOML config files. Including TOML parsing in stdlib would add compile-time cost and binary size to every Ferrum program regardless of need. Programs that need config file parsing opt in explicitly via `extlib.toml`.

### TOML 1.0 as the Stable Spec

This module implements TOML 1.0, which is the stable, frozen version of the TOML specification. TOML 1.0 is complete and will not have breaking changes. Implementations targeting 1.0 will remain correct as the ecosystem matures. Pre-1.0 TOML versions are not supported; they have known ambiguities that 1.0 resolves.

### Key-Order Preservation

TOML is primarily a human-written format. When a human writes:

```toml
host     = "localhost"
port     = 8080
log_level = "info"
```

they have placed those keys in that order for readability and logical grouping. A parser that silently reorders keys to hash-bucket order and then emits them back in a different order is hostile to human maintainability. Round-trip round-tripping — parsing and re-serializing a config file — must not reorder keys.

For this reason, `extlib.toml` uses `IndexMap[String, Value]` (insertion-order-preserving map from `alloc.collections`) as its table type throughout, not `HashMap`. This is not a performance compromise: `IndexMap` provides O(1) lookup by key with the additional property that `iter()` yields entries in insertion order.

The CCSP `lib_ccsp_toml` implementation, which this module draws on, was specifically designed with this property as a first-class requirement. Configuration files maintained by humans deserve to survive their tooling.

---

## 2. Dynamic Value Type

### 2.1 The Value Enum

```ferrum
/// A TOML value. Covers all types defined in the TOML 1.0 specification.
@derive(Debug, Clone, PartialEq)
enum Value {
    /// A UTF-8 string (basic or literal).
    String(String),

    /// A 64-bit signed integer.
    /// TOML integers have no size limit in the spec, but values outside
    /// i64 range produce a parse error under this implementation.
    Integer(i64),

    /// A 64-bit IEEE 754 floating-point number.
    /// Includes the special values inf, -inf, and nan.
    Float(f64),

    /// A boolean value: true or false.
    Boolean(bool),

    /// A date/time value. TOML 1.0 specifies four subtypes.
    DateTime(TomlDateTime),

    /// A TOML array. All elements must be the same type (TOML 1.0 §4.5).
    Array(Vec[Value]),

    /// A TOML table (inline or standard). Keys are in insertion order.
    Table(Table),
}
```

### 2.2 Why Integer Is i64, Not f64

A design choice inherited from CCSP philosophy: numbers without a decimal point are integers; numbers with a decimal point are floats. `42` is an `Integer(42i64)`. `42.0` is a `Float(42.0f64)`. These are distinct types.

This is the correct choice. Config files routinely contain integer values with precise semantics: port numbers, thread pool sizes, file descriptor limits, timeouts in milliseconds. Representing `8080` as `8080.0f64` introduces a floating-point type that implies approximation where none exists. TOML 1.0 itself distinguishes integers from floats at the syntax level. Collapsing them into a single `f64` would violate TOML's own type system.

### 2.3 The Table Type Alias

```ferrum
/// An ordered TOML table: keys are strings, values are TOML values,
/// iteration order is insertion order.
type Table = IndexMap[String, Value]
```

`Table` appears throughout the API as the top-level return type and as the `Value::Table` variant. It is a type alias, not a newtype, so all `IndexMap` methods are available directly.

### 2.4 TomlDateTime

TOML 1.0 defines four datetime subtypes, and all four are meaningful. This module represents them faithfully rather than collapsing them.

```ferrum
/// A TOML date/time value. TOML 1.0 §4.6.
@derive(Debug, Clone, PartialEq)]
enum TomlDateTime {
    /// An offset date-time: a point in time with a UTC offset.
    /// Example: 1979-05-27T07:32:00+00:00
    OffsetDateTime {
        timestamp: time.Timestamp,
        offset:    time.UtcOffset,
    },

    /// A local date-time: a date and time without timezone information.
    /// Example: 1979-05-27T07:32:00
    LocalDateTime(time.NaiveDateTime),

    /// A local date: a calendar date without time or timezone.
    /// Example: 1979-05-27
    LocalDate(time.NaiveDate),

    /// A local time: a time of day without date or timezone.
    /// Example: 07:32:00
    LocalTime(time.NaiveTime),
}
```

The types `time.Timestamp`, `time.UtcOffset`, `time.NaiveDateTime`, `time.NaiveDate`, and `time.NaiveTime` are from the stdlib `time` module.

### 2.5 Display and Debug for Value

`Value` implements `Display` to produce valid TOML output for any value. This is used internally by `serialize` and `serialize_value`, and is available to callers who need to embed individual values in larger strings.

```ferrum
impl fmt.Display for Value {
    /// Formats the value as valid TOML syntax.
    /// Strings are quoted; integers have no decimal; floats always have a decimal
    /// or use special syntax (inf, -inf, nan); booleans are lowercase;
    /// arrays and tables are formatted inline.
    fn fmt(&self, f: &mut fmt.Formatter): Result[(), io.IoError] ! IO
}
```

---

## 3. Parsing

### 3.1 Primary Parse Function

```ferrum
/// Parse a TOML document from a string.
///
/// Returns an ordered table on success. Keys are in the order they appear
/// in the source document. Returns a structured error with line and column
/// on failure.
fn parse(input: &str): Result[Table, TomlError] ! Alloc
```

This is the main entry point for config file parsing. The entire document must be in memory; streaming parsing is not provided (TOML documents are config files, not data streams).

### 3.2 Single-Value Parse

```ferrum
/// Parse a single TOML value from a string.
///
/// Useful for parsing values stored in environment variables or passed
/// as CLI arguments in TOML syntax (e.g., `--set key=42`).
fn parse_value(input: &str): Result[Value, TomlError] ! Alloc
```

### 3.3 TomlError

```ferrum
/// A parse error with precise source location.
@derive(Debug)]
struct TomlError {
    /// A human-readable description of the error.
    message: String,

    /// The location in the source where the error was detected.
    span: Span,

    /// The category of error, for programmatic handling.
    kind: TomlErrorKind,
}

impl fmt.Display for TomlError {
    /// Formats as: "line 12, column 5: duplicate key 'host'"
    fn fmt(&self, f: &mut fmt.Formatter): Result[(), io.IoError] ! IO
}

impl core.error.Error for TomlError {}
```

### 3.4 Span

```ferrum
/// A source location: line, column (both 1-indexed), and byte offset.
@derive(Debug, Clone, Copy, PartialEq)]
struct Span {
    /// Line number in the source, 1-indexed.
    line:   u32,

    /// Column number in the source, 1-indexed, counting Unicode scalar values.
    column: u32,

    /// Byte offset from the start of the source string.
    offset: usize,
}
```

Line and column are 1-indexed to match what text editors and human readers expect when navigating to an error. The byte offset is provided for tools that need to underline the error region.

### 3.5 TomlErrorKind

```ferrum
/// The category of a TOML parse error.
@derive(Debug, Clone, PartialEq)]
enum TomlErrorKind {
    /// A key is syntactically invalid (e.g., contains a bare character not
    /// allowed in bare keys).
    InvalidKey,

    /// A key appears more than once at the same table level.
    DuplicateKey { key: String },

    /// A value is syntactically invalid.
    InvalidValue,

    /// The input ended before the document was complete.
    UnexpectedEof,

    /// A string contains an unrecognized or malformed escape sequence.
    InvalidEscapeSequence { sequence: String },

    /// An array contains elements of mixed types (TOML 1.0 §4.5).
    MixedArrayTypes { first: String, found: String },

    /// A datetime string is not a valid RFC 3339 or TOML datetime.
    InvalidDateTime { raw: String },

    /// A table or array-of-tables header conflicts with a previously
    /// defined key or table.
    TableConflict { key: String },

    /// A number is outside the representable range (integers outside i64,
    /// floats that overflow f64).
    NumberOutOfRange { raw: String },

    /// A Unicode escape sequence names a code point that is not a valid
    /// Unicode scalar value.
    InvalidUnicodeCodepoint { codepoint: u32 },

    /// An integer has an invalid base prefix or malformed digit sequence.
    InvalidIntegerLiteral { raw: String },

    /// Expected a specific token but found something else.
    UnexpectedToken { expected: String, found: String },
}
```

### 3.6 Conformance

This module passes the `toml-test` suite (https://github.com/toml-lang/toml-test), which is the official TOML conformance harness. The test suite covers valid documents, invalid documents that must be rejected, and value encoding. Passing `toml-test` is a normative requirement for this implementation; any divergence is a bug.

---

## 4. Typed Deserialization

Parsing produces a `Table` of dynamic `Value` nodes. Most application code wants typed structs, not dynamic maps. The deserialization traits and derive support convert `Value` trees to application types.

This is trait-based: no macros, no runtime reflection. The derive compiler intrinsic generates `FromToml` and `FromTomlTable` implementations at compile time. Hand-written implementations are equally valid.

### 4.1 Core Traits

```ferrum
/// Deserialize from a single TOML value.
trait FromToml: Sized {
    fn from_toml(value: &Value): Result[Self, TomlDeserError]
}

/// Deserialize from a TOML table.
///
/// Use this for structs that map to a TOML table (the common case).
/// `FromToml` is implemented automatically for any type implementing
/// `FromTomlTable`, extracting the `Table` variant from the `Value`.
trait FromTomlTable: Sized {
    fn from_toml_table(table: &Table): Result[Self, TomlDeserError]
}
```

### 4.2 Derive

```ferrum
/// Derive FromToml and FromTomlTable for a struct.
///
/// Each struct field maps to a TOML key with the same name by default.
/// Field attributes can override mapping, provide defaults, and rename keys.
@derive(FromToml)
struct ServerConfig {
    host:     String,
    port:     u16,
    workers:  usize,
    log_level: String,
}
```

The derive generates a `FromTomlTable` implementation that:
1. Reads each field by key from the table
2. Calls `FromToml::from_toml` on the value
3. Returns `TomlDeserError` with a key path on failure

For enums, `@derive(FromToml)` generates a string-based deserializer: the TOML string value is matched against variant names (snake_case by default).

### 4.3 Field Attributes

```ferrum
@derive(FromToml)
struct DatabaseConfig {
    /// Map TOML key "database_url" to field `url`.
    @toml(rename = "database_url")
    url: String,

    /// Field is optional; absent key uses Default::default().
    @toml(default)
    max_connections: usize,

    /// Field is optional; absent key uses the provided expression.
    @toml(default = "30")]
    timeout_seconds: u64,

    /// Skip this field during deserialization (it is always initialized
    /// from Default::default()).
    @toml(skip)
    _internal: bool,
}
```

Available field attributes:

| Attribute | Effect |
|---|---|
| `@toml(rename = "key")` | Map this field to the given TOML key name |
| `@toml(default)` | Use `Default::default()` if the key is absent |
| `@toml(default = "expr")` | Use the given literal expression if the key is absent |
| `@toml(skip)` | Do not read this field from TOML; always use `Default::default()` |

### 4.4 Standard Implementations

The following standard `FromToml` implementations are provided:

| Ferrum type | TOML type required |
|---|---|
| `bool` | `Boolean` |
| `i8`, `i16`, `i32`, `i64` | `Integer` (range-checked) |
| `u8`, `u16`, `u32`, `u64` | `Integer` (non-negative, range-checked) |
| `f32`, `f64` | `Float` |
| `String` | `String` |
| `Vec[T]` where `T: FromToml` | `Array` |
| `Option[T]` where `T: FromToml` | Any value (wraps in `Some`); absent key becomes `None` |
| `HashMap[String, T]` where `T: FromToml` | `Table` (unordered) |
| `IndexMap[String, T]` where `T: FromToml` | `Table` (insertion order preserved) |
| `Value` | Any (identity, no conversion) |

### 4.5 TomlDeserError

```ferrum
/// An error during typed deserialization: what went wrong and where.
@derive(Debug)]
struct TomlDeserError {
    /// A human-readable description of the error.
    message: String,

    /// The path from the document root to the failing key.
    /// Empty means the error is at the top level.
    /// ["database", "port"] means the error is at table.database.port.
    key_path: Vec[String],
}

impl fmt.Display for TomlDeserError {
    /// Formats as: "at database.port: expected integer, found string"
    fn fmt(&self, f: &mut fmt.Formatter): Result[(), io.IoError] ! IO
}

impl core.error.Error for TomlDeserError {}
```

The `key_path` field allows callers to display precise error messages to users without having to construct the path themselves. The deserialization machinery accumulates path components as it recurses into nested tables and arrays.

---

## 5. Serialization

### 5.1 Core Traits

```ferrum
/// Serialize to a single TOML value.
trait ToToml {
    fn to_toml(&self): Value
}

/// Serialize to a TOML table.
///
/// Implement this for structs. `ToToml` is implemented automatically for
/// any type implementing `ToTomlTable`, wrapping the result in `Value::Table`.
trait ToTomlTable {
    fn to_toml_table(&self): Table
}
```

### 5.2 Derive

```ferrum
@derive(ToToml)
struct ServerConfig {
    host:    String,
    port:    u16,
    workers: usize,
}
```

Field attributes `@toml(rename = "key")` and `@toml(skip)` apply to serialization as well as deserialization.

### 5.3 Serialization Functions

```ferrum
/// Serialize a TOML table to a string.
///
/// Produces human-readable TOML with standard formatting:
/// - Keys are not quoted unless they contain characters requiring quoting
/// - String values use basic strings (double-quoted) by default
/// - Multi-line strings are used when a string value contains newlines
/// - Inline tables are used for Value::Table entries within arrays
/// - Standard tables ([section]) are used for top-level table values
/// - Arrays of tables ([[section]]) are used for Vec[Table] values
/// - Numbers, booleans, and datetimes use their canonical TOML representations
fn serialize(table: &Table): String ! Alloc

/// Serialize a single TOML value to a string.
///
/// Produces the value's TOML inline representation. For tables and
/// arrays, this is the inline form (not the [section] form).
fn serialize_value(value: &Value): String ! Alloc
```

### 5.4 Round-Trip Guarantee

`parse(serialize(t)) == t` for all tables `t` that:
- Contain only values representable in TOML 1.0
- Have no `Float` values that are NaN (NaN != NaN, so equality is not defined)

Key order is preserved. Type identity is preserved (integers do not become floats). Numeric values round-trip exactly. String values round-trip exactly. DateTime values round-trip exactly, including the four TOML datetime subtypes.

This guarantee is normative. The implementation is tested against it.

---

## 6. Merging and Defaults

### 6.1 The merge Function

```ferrum
/// Merge two TOML tables, with override_ taking precedence over base.
///
/// Rules:
/// - Keys present only in base: kept as-is.
/// - Keys present only in override_: added to result.
/// - Keys present in both:
///   - If both values are Table: merge recursively.
///   - Otherwise: override_ value replaces base value entirely.
///     Arrays are replaced, not appended. This is deliberate: array merging
///     has no unambiguous semantics.
///
/// The result is a new Table; neither argument is modified.
fn merge(base: Table, override_: Table): Table ! Alloc
```

### 6.2 Use Case: Config File with Defaults

The idiomatic pattern is to define defaults in code, parse the user's config file, and merge with the user config taking precedence:

```ferrum
use extlib.toml.{self, Value, Table}
use extlib.toml.ToTomlTable

@derive(FromToml, ToToml)
struct ServerConfig {
    host:         String,
    port:         u16,
    workers:      usize,
    log_level:    String,
    @toml(rename = "tls_cert_path")
    tls_cert:     Option[String],
}

impl Default for ServerConfig {
    fn default(): Self {
        ServerConfig {
            host:      String.from("127.0.0.1"),
            port:      8080,
            workers:   4,
            log_level: String.from("info"),
            tls_cert:  None,
        }
    }
}

fn load_config(config_path: &str): Result[ServerConfig, Error] ! IO + Alloc {
    let defaults = ServerConfig.default().to_toml_table()
    let raw      = std.fs.read_to_string(config_path)?
    let user     = toml.parse(&raw)?
    let merged   = toml.merge(defaults, user)
    Ok(ServerConfig.from_toml_table(&merged)?)
}
```

This pattern is clean because both `defaults` and `user` are `Table` values, and `merge` operates uniformly on them. The struct does not need to represent "unset" fields — the default table fills them in before deserialization.

---

## 7. Schema Validation

For programs that expose config files to end users, schema validation provides early, precise error messages with names and descriptions of what was expected.

### 7.1 TomlSchema

```ferrum
/// A schema describing the expected structure of a TOML document.
struct TomlSchema {
    /// The fields expected at the top level of the document.
    fields: Vec[SchemaField],

    /// If true, keys not declared in fields are rejected.
    /// If false (default), undeclared keys are allowed.
    strict: bool,
}

/// A single field declaration within a schema.
struct SchemaField {
    /// The TOML key name.
    key: String,

    /// The expected TOML type.
    expected_type: SchemaType,

    /// If false, the key must be present in the document.
    /// If true, the key may be absent.
    optional: bool,

    /// Human-readable description of this field, shown in error messages
    /// and generated documentation.
    description: String,

    /// For Table fields: schema for the nested table's contents.
    nested: Option[Box[TomlSchema]],
}

/// The TOML type expected by a schema field.
@derive(Debug, Clone, PartialEq)]
enum SchemaType {
    String,
    Integer,
    Float,
    Boolean,
    DateTime,
    Array,
    Table,
    /// Accept any TOML type.
    Any,
}
```

### 7.2 validate

```ferrum
/// Validate a parsed TOML table against a schema.
///
/// Returns Ok(()) if the table satisfies the schema.
/// Returns Err with a list of all violations found (not just the first).
fn validate(table: &Table, schema: &TomlSchema): Result[(), Vec[SchemaError]] ! Alloc
```

### 7.3 SchemaError

```ferrum
/// A single schema violation.
@derive(Debug)]
enum SchemaError {
    /// A required key is absent from the table.
    MissingRequired { key: String },

    /// A key's value has the wrong type.
    TypeMismatch {
        key:      String,
        expected: SchemaType,
        found:    String,   // human-readable description of what was found
    },

    /// A key is present but not declared in the schema (only when strict = true).
    UnknownKey { key: String },
}

impl fmt.Display for SchemaError {
    fn fmt(&self, f: &mut fmt.Formatter): Result[(), io.IoError] ! IO
}

impl core.error.Error for SchemaError {}
```

Validation returns a `Vec[SchemaError]` rather than stopping at the first error. A config file with three problems should report all three so the user can fix them in one edit cycle.

---

## 8. Integration with extlib.cli

`extlib.toml` is designed to compose with `extlib.cli`, Ferrum's CLI argument parsing module. The integration is loose coupling by convention, not hard dependency.

### 8.1 Config File Loading via CLI

`extlib.cli` supports a `--config <path>` option that, when used with this module, loads a TOML config file and populates option defaults before CLI argument parsing runs. The flow:

1. `extlib.cli` recognizes `--config <path>` as a special option
2. The config file at `<path>` is loaded and parsed with `toml.parse`
3. TOML table keys are mapped to CLI option names (replacing `_` with `-` by convention: TOML `max_connections` → CLI `--max-connections`)
4. The parsed TOML values are used as defaults for their corresponding CLI options
5. Explicit CLI arguments override config file values, which override compiled-in defaults

This means users can put in a config file anything they would otherwise pass on the command line, and the config file acts as persistent defaults.

### 8.2 Default Config File Paths

When `--config` is not specified, `extlib.cli` looks for a config file in a platform-appropriate location. For a program named `myapp`:

- Linux/macOS: `$XDG_CONFIG_HOME/myapp/config.toml` or `~/.config/myapp/config.toml`
- Windows: `%APPDATA%\myapp\config.toml`

If no config file is found at the default path, the program proceeds with compiled-in defaults only — the absence of a config file is not an error.

### 8.3 Overlay Pattern

The recommended overlay pattern for a program using both `extlib.toml` and `extlib.cli`:

```ferrum
// 1. Define defaults as a typed struct (compiled-in)
let defaults = AppConfig.default()

// 2. Load and merge config file (user's persistent settings)
let config = match load_config_file() {
    Ok(file_config)  => toml.merge(defaults.to_toml_table(), file_config),
    Err(_)           => defaults.to_toml_table(),
}

// 3. Apply CLI overrides (one-time flags)
let final_config = apply_cli_overrides(config, cli_args)
```

---

## 9. Error Recovery (Lenient Mode)

For editors, language servers, and tooling that needs to display errors inline without refusing to parse a document that has problems, lenient mode returns a best-effort parse alongside all errors found.

### 9.1 parse_lenient

```ferrum
/// Parse a TOML document in lenient mode.
///
/// Returns both a best-effort table and all errors encountered.
/// On a valid document, the error list is empty and the table is identical
/// to what `parse` would return.
///
/// On an invalid document, the parser continues past errors where possible:
/// - Duplicate keys: the later value is discarded; the error is recorded.
/// - Invalid values: the key is omitted from the result; the error is recorded.
/// - Syntax errors: the parser attempts to resynchronize at the next key.
///
/// The best-effort table should not be used for production logic;
/// it exists to support tooling that wants to provide completion and
/// hover information even in documents with errors.
fn parse_lenient(input: &str): (Table, Vec[TomlError]) ! Alloc
```

### 9.2 Use Cases

Lenient mode is appropriate for:

- **Text editors** showing inline TOML error markers while the user is still typing
- **Language servers** providing hover documentation and completion in TOML files
- **Config linters** that want to report all problems in a single pass
- **Migration tools** that need to read a partially-valid old config format

Lenient mode is not appropriate for production config loading. Use `parse` for that. An application that loads a config file with errors and proceeds silently is worse than one that rejects it with a clear message.

---

## 10. Example Usage

### 10.1 Parse a Server Config File

```ferrum
use extlib.toml.{self, Value, TomlError}
use extlib.toml.FromToml
use std.fs

@derive(Debug, FromToml)]
struct TlsConfig {
    cert_path: String,
    key_path:  String,
}

@derive(Debug, FromToml)]
struct ServerConfig {
    host:    String,

    @toml(default = "8080")
    port:    u16,

    @toml(default = "4")
    workers: usize,

    @toml(rename = "log_level", default = "\"info\"")
    log_level: String,

    @toml(rename = "tls")
    tls: Option[TlsConfig],
}
```

### 10.2 Load, Merge, and Validate

```ferrum
use extlib.toml.{self, ToTomlTable, SchemaField, SchemaType, TomlSchema}

fn main(): Result[(), Error] ! IO + Alloc {
    // Read the config file.
    let raw = std.fs.read_to_string("/etc/myapp/config.toml")
        .map_err(|e| Error.from(format("cannot read config: {}", e)))?

    // Parse.
    let user_table = toml.parse(&raw)
        .map_err(|e| Error.from(format("config parse error: {}", e)))?

    // Validate against schema before merging.
    let schema = TomlSchema {
        strict: false,
        fields: [
            SchemaField {
                key:           String.from("host"),
                expected_type: SchemaType.String,
                optional:      false,
                description:   String.from("Hostname or IP address to listen on"),
                nested:        None,
            },
            SchemaField {
                key:           String.from("port"),
                expected_type: SchemaType.Integer,
                optional:      true,
                description:   String.from("TCP port to listen on (default 8080)"),
                nested:        None,
            },
            SchemaField {
                key:           String.from("tls"),
                expected_type: SchemaType.Table,
                optional:      true,
                description:   String.from("TLS certificate and key paths"),
                nested:        None,
            },
        ],
    }

    match toml.validate(&user_table, &schema) {
        Ok(())     => {},
        Err(errs)  => {
            for e in errs.iter() {
                eprintln("config error: {}", e)
            }
            return Err(Error.from("config validation failed"))
        },
    }

    // Merge user config over compiled-in defaults.
    let defaults    = ServerConfig.default().to_toml_table()
    let merged      = toml.merge(defaults, user_table)

    // Deserialize to typed struct.
    let config = ServerConfig.from_toml_table(&merged)
        .map_err(|e| Error.from(format("config type error: {}", e)))?

    println("starting on {}:{} with {} workers", config.host, config.port, config.workers)

    // ... start server ...

    Ok(())
}
```

### 10.3 Serialize Back

```ferrum
use extlib.toml.{self, ToTomlTable}

fn save_config(config: &ServerConfig, path: &str): Result[(), Error] ! IO + Alloc {
    let table = config.to_toml_table()
    let text  = toml.serialize(&table)
    std.fs.write_string(path, &text)?
    Ok(())
}
```

### 10.4 Using parse_lenient in a Config Linter

```ferrum
use extlib.toml

fn lint_config_file(path: &str) ! IO + Alloc {
    let raw = std.fs.read_to_string(path).unwrap_or_else(|e| {
        eprintln("cannot read {}: {}", path, e)
        String.new()
    })

    let (table, errors) = toml.parse_lenient(&raw)

    if errors.is_empty() {
        println("{}: OK ({} top-level keys)", path, table.len())
    } else {
        for err in errors.iter() {
            println("{}:{}:{}: error: {}", path, err.span.line, err.span.column, err.message)
        }
        println("{}: {} error(s)", path, errors.len())
    }
}
```

### 10.5 Parsing Values from CLI Arguments

```ferrum
use extlib.toml

fn parse_toml_override(key: &str, raw_value: &str): Result[Value, String] ! Alloc {
    toml.parse_value(raw_value)
        .map_err(|e| format("invalid TOML value for --set {}: {}", key, e))
}

// Usage: --set workers=8 --set log_level=\"debug\"
fn apply_overrides(
    table:     &mut Table,
    overrides: &[(String, String)],
): Result[(), String] ! Alloc {
    for (key, raw) in overrides.iter() {
        let value = parse_toml_override(key, raw)?
        table.insert(key.clone(), value)
    }
    Ok(())
}
```

---

## 11. Dependencies

`extlib.toml` depends on the following stdlib modules and nothing else. No other extlib modules are required. This is deliberate: config file parsing is a common enough primitive that minimizing its dependency footprint is worthwhile.

| Module | Used for |
|---|---|
| `alloc.string` | `String` for keys, values, error messages |
| `alloc.vec` | `Vec[T]` for arrays, error lists, key paths |
| `alloc.collections.IndexMap` | `IndexMap[String, Value]` for ordered tables |
| `alloc.collections.HashMap` | `HashMap[String, T]` in `FromToml` standard impl |
| `alloc.boxed` | `Box[TomlSchema]` for nested schema references |
| `core.error` | `Error` trait for error types |
| `fmt` | `Display`, `Debug` for all public types |
| `time` | `Timestamp`, `UtcOffset`, `NaiveDateTime`, `NaiveDate`, `NaiveTime` for `TomlDateTime` |

`extlib.toml` does not depend on:

- `io` or `fs` — all functions take `&str`; file reading is the caller's responsibility
- `net` — no network operations
- `async` — synchronous API only; TOML files are small enough that async is not warranted
- `sys` or `posix` — no OS calls
- `crypto` — no cryptographic operations
- Any other extlib module
