# Ferrum Extended Library — cli

**Module:** `extlib.cli`
**Part of:** Extended Standard Library (not `std` or `core`)
**Companion:** [Ferrum Standard Library](ferrum-stdlib.md), [extlib.toml](ferrum_extlib_toml.md)

---

## 1. Overview and Rationale

### The Drift Problem

Every CLI program has two sources of truth for its options: the parsing code and the help text. In hand-written parsers these are separate artifacts. The `--workers` flag is parsed in one function, described in a string three hundred lines away, and read from an environment variable in a third place that nobody remembered to document. Over the life of a project, these drift. The help text describes `--timeout` in seconds; the code now interprets it in milliseconds. The environment variable `APP_WORKERS` works on one deployment and not another because one engineer remembered the prefix and one didn't.

Declarative CLI parsing solves drift by construction. A `CliSchema` is the single source of truth for every option: its name, its type, its default, its environment variable name, its config file key, and its help text. Help cannot drift from parsing because they are generated from the same data structure. Environment variable lookup cannot silently vary because the variable name is in the schema.

### Source Tracking for Debugging

Distributed systems have a production-debugging problem: a binary is running with unexpected behavior, and you cannot tell whether it came from a flag someone passed on the command line, a stale environment variable in a `systemd` unit file, or a value in a config file that predates the current team. Without source tracking, the only recourse is to read all three.

`extlib.cli` records where every option value came from. `.source("workers")` returns `OptionSource::EnvVar { name: "APP_WORKERS" }` or `OptionSource::ConfigFile { path: ..., key: "server.workers" }` or `OptionSource::CommandLine { position: 3 }`. `.print_effective_config()` dumps all options and their sources to a writer in one call. This turns a production mystery into a one-line diagnosis.

### Uniform CLI / Env / Config Integration

The source priority is fixed and unambiguous: command-line arguments override environment variables, which override config file values, which override compiled-in defaults. This order is what users expect and what is correct. The schema declares all three lookup paths per option; the parser implements the priority automatically.

### Why extlib, Not stdlib

CLI argument parsing depends on `extlib.toml` for config file loading. A module with a non-trivial extlib dependency does not belong in the standard library. Additionally, not every Ferrum program is a CLI binary: embedded firmware, library crates, kernel modules, and protocol implementations have no need for CLI parsing. Placing it in extlib keeps it opt-in.

### CCSP Lineage

`extlib.cli` is based on the CCSP `lib_ccsp_cli` design, which was specifically built to enforce the schema-is-truth principle and source tracking as first-class features, not afterthoughts.

---

## 2. Option Types

### 2.1 CliType

```ferrum
/// The type of a CLI option's value.
@derive(Debug, Clone)
enum CliType {
    /// A boolean flag. Presence means true; --no-<name> means false.
    /// On the command line: --verbose / --no-verbose
    /// In config/env: "true" / "false" / "1" / "0" / "yes" / "no"
    Bool,

    /// A UTF-8 string value.
    String,

    /// A 64-bit signed integer.
    /// Accepts decimal, 0x hex, 0o octal, 0b binary prefixes.
    Int,

    /// A 64-bit unsigned integer.
    /// Accepts decimal, 0x hex, 0o octal, 0b binary prefixes. Rejects negative.
    Uint,

    /// A 64-bit IEEE 754 floating-point number.
    Float,

    /// A byte size with optional SI suffix.
    ///
    /// Suffixes (case-insensitive): none = bytes, k/kb = 1024, m/mb = 1024^2,
    /// g/gb = 1024^3, t/tb = 1024^4. Decimal prefixes (1000-based) are not
    /// accepted; SI binary suffixes (kib, mib, gib) are synonyms for k/m/g.
    ///
    /// Examples: "512", "10k", "2m", "1g", "1gb", "256KiB"
    /// Stored as u64 bytes.
    Size,

    /// A duration with mandatory unit suffix.
    ///
    /// Suffixes: ms (milliseconds), s (seconds), m (minutes), h (hours).
    /// Fractional values are accepted: "1.5s" = 1500ms.
    /// Stored as std.time.Duration (nanosecond precision).
    ///
    /// Examples: "100ms", "5s", "2m", "1h", "1.5s", "500ms"
    Duration,

    /// A filesystem path. Stored as PathBuf.
    /// Tilde expansion (~/) is performed relative to the user's home directory.
    Path,

    /// A value from a fixed set of string variants.
    /// The value is validated against the variants list at parse time.
    /// Stored as a string (validated to be one of the declared variants).
    Enum {
        variants: &'static [&'static str],
    },

    /// A list of values of a given element type.
    /// On the command line: --flag value1 --flag value2, or --flag value1,value2
    /// In config: TOML array or comma-separated string
    /// Stored as Vec[String] (each element validated/converted per element_type).
    List {
        element_type: Box[CliType],
    },

    /// A counted flag. Each occurrence increments a u32 counter.
    /// -v = 1, -v -v = 2, -vvv = 3 (short flag stacking)
    /// Cannot be set from env or config as a count; those sources use Uint instead.
    Count,
}
```

### 2.2 Size Parsing

`Size` parsing is strict about ambiguity. The input must be a non-negative integer followed by an optional suffix with no whitespace between them. `"10 k"` is rejected; `"10k"` is accepted. Overflow — values that exceed `u64::MAX` bytes after scaling — produces `CliError::SizeOverflow`.

| Input | Result (bytes) |
|---|---|
| `"512"` | 512 |
| `"10k"` | 10240 |
| `"2m"` | 2097152 |
| `"1g"` | 1073741824 |
| `"1gb"` | 1073741824 |
| `"256KiB"` | 262144 |
| `"1t"` | 1099511627776 |

### 2.3 Duration Parsing

`Duration` parsing maps to `std.time.Duration` (nanosecond-precision unsigned). Fractional values are accepted where the result is representable in nanoseconds.

| Input | Result |
|---|---|
| `"100ms"` | `Duration.from_millis(100)` |
| `"5s"` | `Duration.from_secs(5)` |
| `"2m"` | `Duration.from_secs(120)` |
| `"1h"` | `Duration.from_secs(3600)` |
| `"1.5s"` | `Duration.from_millis(1500)` |
| `"0.5ms"` | `Duration.from_micros(500)` |

Values without a unit suffix are rejected. `"5"` is not a valid duration; `"5s"` is.

---

## 3. Option Schema Definition

### 3.1 CliOption

```ferrum
/// The default value for a CLI option. Stored as the serialized string form
/// so that it can be displayed in help text without type-specific formatting.
@derive(Debug, Clone)]
struct CliDefault {
    /// The default value as it would appear on the command line.
    /// For Bool: "true" or "false". For Size: "10m". For Duration: "5s".
    display: &'static str,
}

/// A single option declaration within a CliSchema.
///
/// All fields are `'static` — schemas are defined at compile time as
/// `static` constants and referenced by the parse functions.
@derive(Debug)]
struct CliOption {
    /// The long option name, without the -- prefix.
    /// Conventionally kebab-case: "log-level", "max-connections".
    name: &'static str,

    /// The short single-character flag, without the - prefix.
    /// None if this option has no short form.
    short: Option[char],

    /// The type of value this option accepts.
    type_: CliType,

    /// The compiled-in default value, if any.
    /// None means the option is unset unless the user provides a value.
    /// Use required: true for options that must be provided.
    default: Option[CliDefault],

    /// The environment variable name to check if the option is not on
    /// the command line. None means no environment variable fallback.
    /// Example: "APP_LOG_LEVEL"
    env_var: Option[&'static str],

    /// The TOML config file key for this option.
    /// Dot-separated for nested tables: "server.port" reads from
    /// the [server] table's port key.
    /// None means this option cannot be set via config file.
    config_key: Option[&'static str],

    /// If true, parsing fails with MissingRequired if no value is provided
    /// from any source (CLI, env, config, or default).
    required: bool,

    /// Help text displayed in --help output.
    /// Should be a complete sentence or short phrase.
    help: &'static str,

    /// Metavar name for the value placeholder in help output.
    /// Displayed as: --name <METAVAR>
    /// None uses the type name: STRING, INT, UINT, FLOAT, SIZE, DURATION, PATH.
    /// Not displayed for Bool (no value) or Count (no value).
    metavar: Option[&'static str],
}
```

### 3.2 Example Schema Declaration

```ferrum
static LOG_LEVEL_OPTION: CliOption = CliOption {
    name:       "log-level",
    short:      Some('l'),
    type_:      CliType.Enum { variants: &["trace", "debug", "info", "warn", "error"] },
    default:    Some(CliDefault { display: "info" }),
    env_var:    Some("APP_LOG_LEVEL"),
    config_key: Some("log.level"),
    required:   false,
    help:       "Minimum log level to emit.",
    metavar:    Some("LEVEL"),
}
```

---

## 4. Declarative Schema

### 4.1 CliSchema

```ferrum
/// The top-level schema for a CLI program.
///
/// Define as a `static` constant; pass a reference to `parse` or `parse_from`.
/// All data is `'static` — no heap allocation at schema definition time.
@derive(Debug)]
struct CliSchema {
    /// The options accepted by this program (or this subcommand).
    options: &'static [CliOption],

    /// Subcommands, if this is a multi-command program.
    /// None for programs with a single command.
    subcommands: Option[&'static [CliSubcommand]],

    /// Version string, printed by --version.
    /// Example: "myapp 1.4.2"
    version: &'static str,

    /// One-line description of the program, shown at the top of --help.
    description: &'static str,

    /// Optional text shown at the bottom of --help, after the option table.
    /// Useful for examples or references to further documentation.
    epilog: Option[&'static str],

    /// Platform-specific default config file paths, tried in order when
    /// --config is not given. The first path that exists is loaded.
    ///
    /// Paths may contain the placeholder `{appname}` which is replaced
    /// by the program name at parse time. Tilde expansion is applied.
    ///
    /// Typical values:
    ///   Linux/macOS: "~/.config/{appname}/config.toml"
    ///   Windows:     "%APPDATA%/{appname}/config.toml"
    ///
    /// If the slice is empty, config file loading is disabled by default
    /// (only enabled if --config is explicitly passed).
    default_config_paths: &'static [&'static str],
}
```

### 4.2 CliSubcommand

```ferrum
/// A subcommand within a multi-command CLI program.
///
/// Example: `git commit`, `cargo build`, `kubectl apply`
@derive(Debug)]
struct CliSubcommand {
    /// The subcommand name as typed on the command line.
    name: &'static str,

    /// One-line description shown in the parent command's --help.
    description: &'static str,

    /// Options accepted by this subcommand.
    /// These are in addition to (not replacing) any global options
    /// defined in the parent schema.
    options: &'static [CliOption],

    /// Handler function invoked when this subcommand is selected.
    /// Receives the fully parsed arguments for this subcommand.
    ///
    /// The handler's effect set is open; callers supply the concrete
    /// implementation at registration time.
    handler: fn(args: &ParsedArgs): Result[(), CliError] ! IO,
}
```

### 4.3 Schema is Static

Schemas are defined as `static` values — they exist for the entire program lifetime and require no heap allocation. This means:

- Schema definition has zero runtime cost beyond the initial startup data layout.
- `parse` takes `&'static CliSchema`, not `CliSchema`, because the schema outlives the parsed args.
- Generated help text borrows directly from schema strings without copying.

---

## 5. Parsing

### 5.1 Primary Parse Function

```ferrum
/// Parse arguments from the process environment.
///
/// Reads from:
/// 1. env::args() — the process command-line arguments
/// 2. Environment variables — for options with env_var set
/// 3. TOML config file — if --config is passed or a default config path exists
///
/// Priority: CLI > env var > config file > default
///
/// Automatically handles --help (prints help and exits), --version (prints
/// version and exits), and --config <path> (loads a TOML config file).
///
/// Returns Err for any parse, type, or constraint violation.
pub fn parse(schema: &'static CliSchema): Result[ParsedArgs, CliError] ! IO + Alloc
```

`parse` reads `env::args()` directly. The first element (the program name) is consumed but not treated as an option. Subcommand dispatch is handled internally; if the schema has subcommands and a valid subcommand name is the first non-option argument, parsing continues with that subcommand's option set.

### 5.2 Test Parse Function

```ferrum
/// Parse arguments from a provided slice.
///
/// Identical to `parse` in all respects except that it reads from `args`
/// rather than `env::args()`. Environment variable lookup and config file
/// loading still occur unless suppressed by the caller.
///
/// Used in tests. Does not exit the process on --help or --version;
/// instead returns CliError::HelpRequested or CliError::VersionRequested.
pub fn parse_from(
    schema: &'static CliSchema,
    args:   &[&str],
): Result[ParsedArgs, CliError] ! Alloc
```

`parse_from` is the testing entry point. Because it does not call `env::args()`, it requires only `! Alloc` rather than `! IO + Alloc`. Environment variables are still read during `parse_from` unless the caller sets `NO_ENV=1` or similar — tests that need isolation should set the relevant environment variables to known values or use a mock env (see Section 13).

### 5.3 Config File Loading

When a config file is found — either via `--config <path>` or by discovering a default config path — it is loaded and parsed as TOML using `extlib.toml::parse`. The resulting table is consulted for each option that has a `config_key`. Dotted keys (`"server.port"`) navigate nested tables.

Config file errors are fatal: a config file that exists but is unparseable returns `CliError::ConfigParseError`. A config file that does not exist at any default path is silently ignored.

---

## 6. ParsedArgs

### 6.1 Typed Accessors

```ferrum
/// The result of a successful parse. Holds all option values and their sources.
struct ParsedArgs { ... }

impl ParsedArgs {
    /// Returns the bool value for the named option.
    /// Panics if name is not a Bool option in the schema.
    pub fn get_bool(&self, name: &str): bool

    /// Returns the string value for the named option.
    /// Panics if name is not a String option in the schema.
    pub fn get_string(&self, name: &str): &str

    /// Returns the i64 value for the named option.
    /// Panics if name is not an Int option in the schema.
    pub fn get_int(&self, name: &str): i64

    /// Returns the u64 value for the named option.
    /// Panics if name is not a Uint option in the schema.
    pub fn get_uint(&self, name: &str): u64

    /// Returns the f64 value for the named option.
    /// Panics if name is not a Float option in the schema.
    pub fn get_float(&self, name: &str): f64

    /// Returns the byte count for the named option.
    /// Panics if name is not a Size option in the schema.
    pub fn get_size(&self, name: &str): u64

    /// Returns the duration for the named option.
    /// Panics if name is not a Duration option in the schema.
    pub fn get_duration(&self, name: &str): std.time.Duration

    /// Returns the path for the named option.
    /// Panics if name is not a Path option in the schema.
    pub fn get_path(&self, name: &str): &std.fs.Path

    /// Returns the enum variant string for the named option.
    /// The returned string is guaranteed to be one of the declared variants.
    /// Panics if name is not an Enum option in the schema.
    pub fn get_enum(&self, name: &str): &str

    /// Returns the list of values for the named option.
    /// Panics if name is not a List option in the schema.
    pub fn get_list(&self, name: &str): &[String]

    /// Returns the count for the named option.
    /// Panics if name is not a Count option in the schema.
    pub fn get_count(&self, name: &str): u32

    /// Returns where the value for this option came from.
    /// Panics if name is not in the schema.
    pub fn source(&self, name: &str): OptionSource

    /// Returns the name of the subcommand selected by the user, if any.
    pub fn subcommand(&self): Option[&str]

    /// Returns the remaining positional arguments after all option parsing.
    pub fn positional(&self): &[String]
}
```

### 6.2 Panics vs. Errors

The typed accessors panic on wrong-type access (e.g., calling `get_bool` on a `String` option) because this is always a programmer error: the schema is `'static` and known at compile time. A `get_bool` call on a `String` option would only happen if the caller passed the wrong name string, which is a coding mistake, not a user error. Panicking surfaces it immediately during development.

User errors — invalid values, missing required options, unknown flags — are all `Result[_, CliError]` from `parse`.

---

## 7. Source Tracking

### 7.1 OptionSource

```ferrum
/// Where a parsed option value came from.
@derive(Debug, Clone)]
enum OptionSource {
    /// No value was provided; the compiled-in default was used.
    Default,

    /// The value came from a TOML config file.
    ConfigFile {
        /// The path of the config file that provided the value.
        path: std.fs.PathBuf,
        /// The TOML key that was read.
        key:  String,
    },

    /// The value came from an environment variable.
    EnvVar {
        /// The environment variable name.
        name: String,
    },

    /// The value came from the command-line arguments.
    CommandLine {
        /// The zero-based index into the original argv slice at which
        /// the option name was found.
        position: usize,
    },
}
```

### 7.2 print_effective_config

```ferrum
impl ParsedArgs {
    /// Print all options with their current values and sources.
    ///
    /// Output format (one option per line):
    ///   log-level       = "info"    [default]
    ///   port            = 8080      [env: APP_PORT]
    ///   workers         = 4         [config: ~/.config/myapp/config.toml key=server.workers]
    ///   host            = "0.0.0.0" [cli: argv[3]]
    ///
    /// This is intended for operator debugging, not for machine parsing.
    /// Use source() for programmatic source inspection.
    pub fn print_effective_config(
        &self,
        writer: &mut impl std.io.Write,
    ): Result[(), std.io.IoError] ! IO
}
```

`print_effective_config` produces human-readable, column-aligned output. Values are printed in their parsed form, not as raw strings. A `Duration` option shows `"5s"`, not `"5000000000"`. A `Size` option shows `"10m"`, not `"10485760"`. This makes the output immediately useful to a human reading a support ticket or debugging a deployment.

---

## 8. Typed Struct Extraction

### 8.1 FromArgs Trait

```ferrum
/// A type that can be extracted from a ParsedArgs.
///
/// Implement manually or derive with @derive(FromArgs).
pub trait FromArgs: Sized {
    fn from_args(args: &ParsedArgs): Result[Self, CliError]
}
```

### 8.2 Derive

```ferrum
/// Derive FromArgs for a struct.
///
/// Each field maps to a CLI option by field name (snake_case maps to kebab-case
/// option name: field `log_level` maps to option "log-level").
///
/// Field types must implement conversion from the corresponding CliType:
///   bool      ← Bool
///   String    ← String
///   i64       ← Int
///   u64       ← Uint or Size
///   f64       ← Float
///   Duration  ← Duration
///   PathBuf   ← Path
///   String    ← Enum (caller validates against known variants)
///   Vec[T]    ← List (T must be parseable from String)
///   u32       ← Count
@derive(FromArgs)
struct ServerConfig {
    host:      String,
    port:      u64,
    workers:   u64,
    log_level: String,
}
```

The generated implementation calls `get_*` on `ParsedArgs` for each field, using the field name with underscores replaced by hyphens as the option name. If the option name does not exist in `ParsedArgs`, the derive generates a `CliError::UnknownOption` at field-extraction time — this catches schema/struct mismatches at runtime on first use.

### 8.3 Field Attributes

```ferrum
@derive(FromArgs)
struct AppConfig {
    /// Map CLI option "bind-address" to field `host`.
    @cli(name = "bind-address")
    host: String,

    /// Use "http-port" option for this field.
    @cli(name = "http-port")
    port: u64,

    /// Inline the fields of a sub-struct, all drawing from the same ParsedArgs.
    @cli(flatten)
    tls: TlsConfig,
}

@derive(FromArgs)
struct TlsConfig {
    /// Maps to option "tls-cert".
    @cli(name = "tls-cert")
    cert_path: std.fs.PathBuf,

    /// Maps to option "tls-key".
    @cli(name = "tls-key")
    key_path: std.fs.PathBuf,
}
```

Available field attributes:

| Attribute | Effect |
|---|---|
| `@cli(name = "option-name")` | Map this field to the named CLI option instead of using the field name |
| `@cli(flatten)` | Extract the sub-struct's fields from the same `ParsedArgs`, recursively |

`@cli(flatten)` composes cleanly: a flattened sub-struct calls `from_args` on the same `ParsedArgs` instance, so all option lookups go to the same parsed argument map. Nested flattening is supported to any depth.

---

## 9. Help Generation

### 9.1 print_help

```ferrum
/// Print formatted help text for the given schema to a writer.
///
/// Format:
///   Usage: <appname> [OPTIONS] [SUBCOMMAND]
///
///   <description>
///
///   Options:
///     -h, --help             Print this help and exit
///         --version          Print version and exit
///     -p, --port <PORT>      Port to listen on. [default: 8080] [env: APP_PORT]
///     -w, --workers <N>      Worker thread count. [default: 4] [env: APP_WORKERS]
///     -l, --log-level <LEVEL>
///                            Minimum log level to emit. [default: info]
///                            Variants: trace, debug, info, warn, error
///
///   Subcommands:
///     serve     Start the HTTP server
///     migrate   Run database migrations
///
///   <epilog>
///
/// Help cannot drift from the option set because all text is generated from
/// the schema. The only text not in the schema is the Usage line, which
/// is constructed from the schema's structure.
pub fn print_help(
    schema:  &'static CliSchema,
    writer:  &mut impl std.io.Write,
) : Result[(), std.io.IoError] ! IO
```

### 9.2 Automatic Handling

`parse` and `parse_from` automatically handle `--help` and `-h`. When encountered:

- `parse`: prints help to stdout and exits with code 0.
- `parse_from`: returns `Err(CliError::HelpRequested)` without printing. The caller is responsible for printing if desired. This allows tests to verify that `--help` is recognized without the test process exiting.

`--version` is handled the same way: `parse` prints the schema's `version` string and exits; `parse_from` returns `Err(CliError::VersionRequested)`.

### 9.3 Formatting Rules

The help formatter:

- Aligns option columns so all help text starts at the same column across all options.
- Wraps long help text at 80 characters, indented to the help text column.
- Shows `[default: <value>]` for options with a default, appended to help text.
- Shows `[env: <VAR>]` for options with `env_var` set, appended to help text.
- Shows `Variants: a, b, c` on a separate line for `Enum` options.
- Does not show `[required]` tags; omitting the default is the signal that an option is required.
- For `Bool` options, shows `--[no-]<name>` to indicate both forms are accepted.
- For `Count` options, shows `-v, -v -v, -v -v -v` in the metavar position to communicate the counting behavior.

---

## 10. Config File Integration

### 10.1 Platform Default Paths

`CliSchema.default_config_paths` holds a list of path templates tried in order when no `--config` flag is provided. The `{appname}` placeholder is substituted with the program name derived from `env::current_exe()` (basename only, without extension on Windows).

Recommended values by platform:

```ferrum
static MY_SCHEMA: CliSchema = CliSchema {
    // ... other fields ...
    default_config_paths: &[
        // Linux and macOS (XDG-compliant)
        "~/.config/{appname}/config.toml",
        // macOS Application Support alternative
        "~/Library/Application Support/{appname}/config.toml",
        // Windows %APPDATA%
        "%APPDATA%\\{appname}\\config.toml",
        // System-wide (Linux, requires appropriate permissions)
        "/etc/{appname}/config.toml",
    ],
}
```

At runtime, only the paths appropriate for the current platform are tried. Paths beginning with `%APPDATA%` are skipped on non-Windows; paths beginning with `~/Library/` are skipped on non-macOS.

### 10.2 Config Key Mapping

A `config_key` of `"server.port"` reads from:

```toml
[server]
port = 8080
```

A `config_key` of `"port"` reads from:

```toml
port = 8080
```

Keys are case-sensitive and must match the TOML key exactly. Hyphens in option names are not automatically mapped to underscores in config keys; the `config_key` field is the exact TOML path. By convention, TOML keys use underscores and option names use hyphens, so `config_key` typically differs from `name`:

```ferrum
CliOption {
    name:       "log-level",
    config_key: Some("log.level"),    // TOML: [log]\nlevel = "info"
    // ...
}
```

### 10.3 Config File Type Coercion

TOML values are coerced to the declared `CliType` before storage:

| CliType | Accepted TOML types |
|---|---|
| `Bool` | `Boolean`, or `String` ("true"/"false"/"1"/"0") |
| `String` | `String` |
| `Int` | `Integer` |
| `Uint` | `Integer` (must be non-negative) |
| `Float` | `Float`, `Integer` (promoted) |
| `Size` | `String` (e.g. `"10m"`), `Integer` (bytes) |
| `Duration` | `String` (e.g. `"5s"`), `Float` (seconds) |
| `Path` | `String` |
| `Enum` | `String` (validated against variants) |
| `List` | `Array` of the element type, or `String` (comma-separated) |
| `Count` | `Integer` (non-negative, fits u32) |

Type mismatches produce `CliError::InvalidValue` with the config file path as context.

---

## 11. Error Types

### 11.1 CliError

```ferrum
/// An error during CLI argument parsing.
@derive(Debug)]
enum CliError {
    /// An option name was not found in the schema.
    UnknownOption {
        /// The option name as typed (with -- prefix stripped).
        name: String,
    },

    /// A required option had no value from any source.
    MissingRequired {
        name: String,
    },

    /// An option's value could not be parsed as the declared type.
    InvalidValue {
        name:          String,
        value:         String,
        expected_type: String,   // human-readable, e.g. "unsigned integer"
    },

    /// An Enum option's value is not one of the declared variants.
    InvalidEnumVariant {
        name:  String,
        value: String,
        valid: Vec[String],
    },

    /// An option was specified more than once on the command line where
    /// only one value is accepted (not applicable to List or Count options).
    DuplicateOption {
        name: String,
    },

    /// A Size option's value parses but its scaled byte count exceeds u64::MAX.
    SizeOverflow {
        name:  String,
        value: String,
    },

    /// Two options were specified that cannot be used together.
    ConflictingOptions {
        a: String,
        b: String,
    },

    /// The TOML config file could not be parsed.
    ConfigParseError(extlib.toml.TomlError),

    /// --help was requested (only from parse_from; parse exits instead).
    HelpRequested,

    /// --version was requested (only from parse_from; parse exits instead).
    VersionRequested,
}

impl fmt.Display for CliError {
    fn fmt(&self, f: &mut fmt.Formatter): Result[(), std.io.IoError] ! IO
}

impl core.error.Error for CliError {}
```

### 11.2 Error Messages

`CliError` formats as a human-readable message suitable for printing to stderr before exiting:

```
error: unknown option: --verbos (did you mean --verbose?)
error: missing required option: --database-url
error: invalid value for --workers: "four" (expected unsigned integer)
error: --workers: invalid enum variant "critical" (valid: trace, debug, info, warn, error)
error: --max-memory: size overflow: "9999999t" exceeds maximum representable value
error: config file parse error at line 12, column 5: duplicate key 'port'
```

The "did you mean" suggestion for `UnknownOption` uses edit-distance comparison against the schema's declared option names.

---

## 12. Example Usage

### 12.1 Schema and Parsing

```ferrum
use extlib.cli.{self, CliSchema, CliOption, CliType, CliDefault}

static SCHEMA: CliSchema = CliSchema {
    description: "A high-performance HTTP server.",
    version:     "myserver 2.1.0",
    epilog:      Some("See https://example.com/docs for full documentation."),
    default_config_paths: &[
        "~/.config/{appname}/config.toml",
        "/etc/{appname}/config.toml",
    ],
    subcommands: None,
    options: &[
        CliOption {
            name:       "host",
            short:      Some('H'),
            type_:      CliType.String,
            default:    Some(CliDefault { display: "127.0.0.1" }),
            env_var:    Some("APP_HOST"),
            config_key: Some("server.host"),
            required:   false,
            help:       "IP address or hostname to bind.",
            metavar:    Some("HOST"),
        },
        CliOption {
            name:       "port",
            short:      Some('p'),
            type_:      CliType.Uint,
            default:    Some(CliDefault { display: "8080" }),
            env_var:    Some("APP_PORT"),
            config_key: Some("server.port"),
            required:   false,
            help:       "TCP port to listen on.",
            metavar:    Some("PORT"),
        },
        CliOption {
            name:       "workers",
            short:      Some('w'),
            type_:      CliType.Uint,
            default:    Some(CliDefault { display: "4" }),
            env_var:    Some("APP_WORKERS"),
            config_key: Some("server.workers"),
            required:   false,
            help:       "Number of worker threads.",
            metavar:    Some("N"),
        },
        CliOption {
            name:       "log-level",
            short:      Some('l'),
            type_:      CliType.Enum {
                            variants: &["trace", "debug", "info", "warn", "error"],
                        },
            default:    Some(CliDefault { display: "info" }),
            env_var:    Some("APP_LOG_LEVEL"),
            config_key: Some("log.level"),
            required:   false,
            help:       "Minimum log level to emit.",
            metavar:    Some("LEVEL"),
        },
        CliOption {
            name:       "max-body-size",
            short:      None,
            type_:      CliType.Size,
            default:    Some(CliDefault { display: "1m" }),
            env_var:    Some("APP_MAX_BODY_SIZE"),
            config_key: Some("server.max_body_size"),
            required:   false,
            help:       "Maximum request body size.",
            metavar:    Some("SIZE"),
        },
        CliOption {
            name:       "request-timeout",
            short:      None,
            type_:      CliType.Duration,
            default:    Some(CliDefault { display: "30s" }),
            env_var:    Some("APP_REQUEST_TIMEOUT"),
            config_key: Some("server.request_timeout"),
            required:   false,
            help:       "Timeout for a single HTTP request.",
            metavar:    Some("DURATION"),
        },
        CliOption {
            name:       "database-url",
            short:      None,
            type_:      CliType.String,
            default:    None,
            env_var:    Some("DATABASE_URL"),
            config_key: Some("database.url"),
            required:   true,
            help:       "PostgreSQL connection URL.",
            metavar:    Some("URL"),
        },
        CliOption {
            name:       "verbose",
            short:      Some('v'),
            type_:      CliType.Count,
            default:    None,
            env_var:    None,
            config_key: None,
            required:   false,
            help:       "Increase verbosity. Repeat for more: -v, -vv, -vvv.",
            metavar:    None,
        },
    ],
}
```

### 12.2 Parsing and Effective Config Dump

```ferrum
use extlib.cli
use std.io

fn main(): Result[(), Error] ! IO + Alloc {
    let args = cli.parse(&SCHEMA)?

    // Print effective configuration for debugging or --debug-config flag.
    if args.get_count("verbose") >= 2 {
        let stderr = io.stderr()
        args.print_effective_config(&mut stderr.lock())?
    }

    let host    = args.get_string("host")
    let port    = args.get_uint("port")
    let workers = args.get_uint("workers")
    let level   = args.get_enum("log-level")
    let max_body = args.get_size("max-body-size")
    let timeout  = args.get_duration("request-timeout")
    let db_url   = args.get_string("database-url")

    println(
        "starting on {}:{} workers={} log={} max_body={}b timeout={}ms",
        host, port, workers, level, max_body,
        timeout.as_millis(),
    )

    // ... start server ...

    Ok(())
}
```

### 12.3 Typed Struct Extraction with @derive(FromArgs)

```ferrum
use extlib.cli.{self, FromArgs}
use std.fs.PathBuf
use std.time.Duration

@derive(Debug, FromArgs)
struct ServerConfig {
    host:    String,
    port:    u64,
    workers: u64,

    @cli(name = "log-level")
    log_level: String,

    @cli(name = "max-body-size")
    max_body_size: u64,

    @cli(name = "request-timeout")
    request_timeout: Duration,

    @cli(name = "database-url")
    database_url: String,
}

fn main(): Result[(), Error] ! IO + Alloc {
    let args   = cli.parse(&SCHEMA)?
    let config = ServerConfig.from_args(&args)?

    println("host={} port={}", config.host, config.port)
    println(
        "request timeout: {}ms",
        config.request_timeout.as_millis()
    )

    Ok(())
}
```

### 12.4 Source Inspection

```ferrum
fn log_option_source(args: &cli.ParsedArgs, name: &str) {
    use cli.OptionSource

    match args.source(name) {
        OptionSource.Default =>
            println!("{}: using default", name),

        OptionSource.EnvVar { name: var } =>
            println("{}: from env ${}", name, var),

        OptionSource.ConfigFile { path, key } =>
            println("{}: from {} (key={})", name, path.display(), key),

        OptionSource.CommandLine { position } =>
            println("{}: from command line (argv[{}])", name, position),
    }
}
```

### 12.5 Test Usage

```ferrum
use extlib.cli

#[test]
fn test_port_override(): Result[(), cli.CliError] {
    let args = cli.parse_from(&SCHEMA, &["--port", "9090"])?
    assert_eq(args.get_uint("port"), 9090)
    assert!(matches!(
        args.source("port"),
        cli.OptionSource.CommandLine { position: 1 }
    ))
    Ok(())
}

#[test]
fn test_missing_required_returns_error() {
    let result = cli.parse_from(&SCHEMA, &[])
    match result {
        Err(cli.CliError.MissingRequired { name }) =>
            assert_eq(name, "database-url"),
        other =>
            panic("expected MissingRequired, got {:?}", other),
    }
}

#[test]
fn test_help_returns_error_not_exit() {
    let result = cli.parse_from(&SCHEMA, &["--help"])
    assert!(matches!(result, Err(cli.CliError.HelpRequested)))
}

#[test]
fn test_size_parsing() {
    let args = cli.parse_from(
        &SCHEMA,
        &["--max-body-size", "2m", "--database-url", "postgres://localhost/test"],
    ).expect("should parse")
    assert_eq(args.get_size("max-body-size"), 2 * 1024 * 1024)
}

#[test]
fn test_duration_parsing() {
    let args = cli.parse_from(
        &SCHEMA,
        &["--request-timeout", "1500ms", "--database-url", "postgres://localhost/test"],
    ).expect("should parse")
    assert_eq(
        args.get_duration("request-timeout"),
        std.time.Duration.from_millis(1500)
    )
}

#[test]
fn test_count_stacking() {
    let args = cli.parse_from(
        &SCHEMA,
        &["-vvv", "--database-url", "postgres://localhost/test"],
    ).expect("should parse")
    assert_eq(args.get_count("verbose"), 3)
}
```

### 12.6 Example --help Output

```
Usage: myserver [OPTIONS]

A high-performance HTTP server.

Options:
  -h, --help                   Print this help and exit.
      --version                Print version and exit.
  -H, --host <HOST>            IP address or hostname to bind.
                               [default: 127.0.0.1] [env: APP_HOST]
  -p, --port <PORT>            TCP port to listen on.
                               [default: 8080] [env: APP_PORT]
  -w, --workers <N>            Number of worker threads.
                               [default: 4] [env: APP_WORKERS]
  -l, --log-level <LEVEL>      Minimum log level to emit.
                               [default: info] [env: APP_LOG_LEVEL]
                               Variants: trace, debug, info, warn, error
      --max-body-size <SIZE>   Maximum request body size.
                               [default: 1m] [env: APP_MAX_BODY_SIZE]
      --request-timeout <DURATION>
                               Timeout for a single HTTP request.
                               [default: 30s] [env: APP_REQUEST_TIMEOUT]
      --database-url <URL>     PostgreSQL connection URL. [env: DATABASE_URL]
  -v, --verbose                Increase verbosity. Repeat for more: -v, -vv, -vvv.

See https://example.com/docs for full documentation.
```

---

## 13. Dependencies

`extlib.cli` depends on the following modules:

| Module | Used for |
|---|---|
| `extlib.toml` | Config file loading and parsing |
| `std.env` | `env::args()`, `env::var()`, `env::current_exe()`, `home_dir()` |
| `std.fs` | Config file reading, `Path`, `PathBuf` |
| `std.io` | `Write` trait for `print_help`, `print_effective_config` |
| `std.time` | `Duration` for the `Duration` CLI type |
| `alloc.string` | `String` for parsed values and error messages |
| `alloc.vec` | `Vec[T]` for parsed lists and error variant lists |
| `alloc.collections.HashMap` | Internal option value storage by name |
| `alloc.boxed` | `Box[CliType]` for `List { element_type }` |
| `core.error` | `Error` trait for `CliError` |
| `fmt` | `Display`, `Debug` for all public types |

`extlib.cli` does not depend on:

- `net` — no network operations
- `async` — synchronous parse only; CLI parsing is not a performance bottleneck
- `crypto` — no cryptographic operations
- `sys` or `posix` — no direct OS calls beyond what `std.env` and `std.fs` provide
