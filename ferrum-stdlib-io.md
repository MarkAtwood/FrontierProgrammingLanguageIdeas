# Ferrum Standard Library — io, binary, text, fs

**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 5. io — The IO Model

`io` defines the trait hierarchy for all IO. Concrete types live in `fs`, `net`, etc.

### Bytes and Strings Are Separate

**IO deals in bytes.** When you read from a file or socket, you get `&[u8]`. When you write, you provide `&[u8]`. There is no implicit encoding or decoding.

**Strings require explicit conversion.** To interpret bytes as text:
- Use `Utf8.validate(bytes)` to check and convert to `&str`
- Use `String.from_utf8(vec)` to take ownership
- Use `TextReader` for streaming text with any encoding

To write text as bytes:
- Use `str.as_bytes()` — free, zero-cost (always valid UTF-8)
- Use `TextWriter` for streaming text with any encoding

**Why?** Python 3's implicit encoding/decoding at IO boundaries caused confusion: when exactly does conversion happen? What encoding? What if it fails mid-stream? Ferrum makes you choose the conversion point deliberately. The type tells you whether you have bytes or text — there is no ambiguity.

### 5.1 Core IO Traits

```ferrum
// The read result — distinct EOF from error
enum ReadResult {
    Data(usize),    // bytes read (may be less than requested — partial read is OK)
    Eof,            // clean end of stream
    Err(IoError),   // actual error
}

trait Read {
    // Read up to buf.len() bytes. Partial reads are NOT errors.
    // Returns ReadResult, never conflates EOF with error.
    fn read(&mut self, buf: &mut [u8]): ReadResult ! IO

    // Provided:
    fn read_exact(&mut self, buf: &mut [u8]): Result[(), IoError] ! IO
        // Loops until buf is full or EOF/Err - Err(UnexpectedEof) if EOF before full
    fn read_to_end(&mut self, buf: &mut Vec[u8]): Result[usize, IoError] ! IO
    fn bytes(&mut self): Bytes[Self]          // iterator over bytes
    fn chain[R: Read](self, next: R): Chain[Self, R]
    fn take(self, limit: u64): Take[Self]
    fn by_ref(&mut self): &mut Self
}

// NOTE: There is no read_to_string on Read.
// Read deals in bytes. If you want text, use TextReader explicitly.
// This is intentional — bytes are not strings, and the encoding
// conversion must be explicit and happen at a deliberate point.

trait Write {
    // Write some or all of buf. Returns bytes written.
    // The caller must check and retry - partial writes are NOT errors.
    fn write(&mut self, buf: &[u8]): Result[usize, IoError] ! IO

    fn flush(&mut self): Result[(), IoError] ! IO

    // Provided:
    fn write_all(&mut self, buf: &[u8]): Result[(), IoError] ! IO
        // Loops until all bytes written
    fn write_fmt(&mut self, fmt: Arguments): Result[(), IoError] ! IO
    fn by_ref(&mut self): &mut Self
}

enum SeekFrom {
    Start(u64),
    End(i64),
    Current(i64),
}

trait Seek {
    fn seek(&mut self, pos: SeekFrom): Result[u64, IoError] ! IO
    fn stream_position(&mut self): Result[u64, IoError] ! IO
    fn stream_len(&mut self): Result[u64, IoError] ! IO
    fn rewind(&mut self): Result[(), IoError] ! IO
}

// BufRead - for line-oriented and buffered reading
// All methods work with BYTES, not strings. Use TextReader for text.
trait BufRead: Read {
    fn fill_buf(&mut self): Result[&[u8], IoError] ! IO
    fn consume(&mut self, amt: usize)

    // Provided:
    fn read_until(&mut self, byte: u8, buf: &mut Vec[u8]): Result[ReadUntilResult, IoError] ! IO
    fn read_line(&mut self, buf: &mut Vec[u8]): Result[ReadUntilResult, IoError] ! IO
        // Equivalent to read_until(b'\n', buf)
    fn split(&mut self, byte: u8): Split[Self]   // iterator of Vec[u8]
    fn lines(&mut self): Lines[Self]             // iterator of Vec[u8], splits on \n
}

enum ReadUntilResult {
    Found(usize),   // delimiter found, bytes read
    Eof(usize),     // EOF before delimiter, bytes read
}
```

### 5.2 IoError

Not an integer. Not errno. A structured type.

```ferrum
type IoError {
    pub kind:    IoErrorKind,
    pub message: String,
    source:      Option[Box[dyn Error]],
}

enum IoErrorKind {
    NotFound,
    PermissionDenied,
    ConnectionRefused,
    ConnectionReset,
    ConnectionAborted,
    AddrInUse,
    AddrNotAvailable,
    BrokenPipe,
    AlreadyExists,
    WouldBlock,
    InvalidInput,
    InvalidData,
    TimedOut,
    WriteZero,
    Interrupted,
    UnexpectedEof,
    OutOfMemory,
    Unsupported,
    Other(String),

    // NOT a catch-all. "Other" is for genuinely uncategorizable errors.
    // New variants are added to this enum in minor versions when needed.
    // Exhaustive matching requires a wildcard for forward compatibility:
    //   _ => handle_unknown_io_error(e),
}

impl IoError {
    fn new(kind: IoErrorKind, message: impl Into[String]): Self
    fn kind(&self): IoErrorKind
    fn from_os_error(code: i32): Self   // converts OS errno codes
    fn raw_os_error(&self): Option[i32] // back to OS code if available
}
```

### 5.3 Buffered Wrappers

```ferrum
// Buffered reader — adds BufRead to any Read
type BufReader[R: Read]  given [A: Allocator] {
    inner: R,
    buf:   Box[[u8]],
    // ...
}

impl[R: Read] BufReader[R] {
    fn new(inner: R): Self                         // default 8KB buffer
    fn with_capacity(cap: usize, inner: R): Self
    fn buffer(&self): &[u8]
    fn capacity(&self): usize
    fn into_inner(self): R
    fn get_ref(&self): &R
    fn get_mut(&mut self): &mut R
}

// Buffered writer — batches writes
type BufWriter[W: Write]  given [A: Allocator]

impl[W: Write] BufWriter[W] {
    fn new(inner: W): Self
    fn with_capacity(cap: usize, inner: W): Self
    fn into_inner(self): Result[W, IntoInnerError[W]]
    fn into_parts(self): (W, Result[Vec[u8], WriterPanicked])
    fn get_ref(&self): &W
    fn get_mut(&mut self): &mut W
    fn buffer(&self): &[u8]
    fn capacity(&self): usize
}

// Line-buffered writer — flushes on \n
type LineWriter[W: Write]

// Cursor — in-memory Read+Write+Seek
type Cursor[T]
impl[T: AsRef[[u8]]] Cursor[T]: Read + Seek
impl Cursor[Vec[u8]]: Write + Read + Seek

// Sink — discards all writes (like /dev/null)
type Sink
impl Write for Sink { fn write(&mut self, buf: &[u8]): Result[usize] { Ok(buf.len()) } }

// Empty — reads return EOF immediately
type Empty
impl Read for Empty { fn read(&mut self, _: &mut [u8]): ReadResult { ReadResult.Eof } }
```

### 5.4 Standard Streams

```ferrum
fn stdin(): Stdin   ! IO     // buffered stdin
fn stdout(): Stdout ! IO     // buffered stdout
fn stderr(): Stderr ! IO     // unbuffered stderr (always unbuffered - this is intentional)

type Stdin  { ... }
type Stdout { ... }
type Stderr { ... }

impl BufRead for Stdin
impl Write for Stdout
impl Write for Stderr

// Lock for exclusive access across threads
impl Stdin  { fn lock(&self): StdinLock ! Sync }
impl Stdout { fn lock(&self): StdoutLock ! Sync }
impl Stderr { fn lock(&self): StderrLock ! Sync }
```

---

## 6. binary — Binary IO

Binary IO is a separate module from text IO. The distinction is first-class, not a flag.

### 6.1 Endian-aware Primitives

```ferrum
// The core trait: read/write a typed value in a specific byte order
trait BinaryRead {
    fn read_u8(&mut self): Result[u8,  IoError] ! IO
    fn read_u16_le(&mut self): Result[u16, IoError] ! IO
    fn read_u16_be(&mut self): Result[u16, IoError] ! IO
    fn read_u32_le(&mut self): Result[u32, IoError] ! IO
    fn read_u32_be(&mut self): Result[u32, IoError] ! IO
    fn read_u64_le(&mut self): Result[u64, IoError] ! IO
    fn read_u64_be(&mut self): Result[u64, IoError] ! IO
    fn read_u128_le(&mut self): Result[u128, IoError] ! IO
    fn read_u128_be(&mut self): Result[u128, IoError] ! IO
    fn read_i8(&mut self): Result[i8,  IoError] ! IO
    // ... signed variants follow same pattern
    fn read_f32_le(&mut self): Result[f32, IoError] ! IO
    fn read_f32_be(&mut self): Result[f32, IoError] ! IO
    fn read_f64_le(&mut self): Result[f64, IoError] ! IO
    fn read_f64_be(&mut self): Result[f64, IoError] ! IO
    fn read_bytes_exact(&mut self, buf: &mut [u8]): Result[(), IoError] ! IO
    fn read_array[const N: usize](&mut self): Result[[u8; N], IoError] ! IO
}

trait BinaryWrite {
    fn write_u8(&mut self, v: u8): Result[(), IoError] ! IO
    fn write_u16_le(&mut self, v: u16): Result[(), IoError] ! IO
    fn write_u16_be(&mut self, v: u16): Result[(), IoError] ! IO
    // ... etc.
    fn write_f64_le(&mut self, v: f64): Result[(), IoError] ! IO
    fn write_f64_be(&mut self, v: f64): Result[(), IoError] ! IO
    fn write_bytes(&mut self, buf: &[u8]): Result[(), IoError] ! IO
    fn write_padding(&mut self, n: usize): Result[(), IoError] ! IO
        // writes n zero bytes - explicit, not implicit
}

// Blanket impl: any Read/Write gets BinaryRead/BinaryWrite
impl[R: Read] BinaryRead for R { ... }
impl[W: Write] BinaryWrite for W { ... }
```

### 6.2 Layout-integrated Binary IO

Types with `layout` declarations get automatic serialization/deserialization (see Language Reference §15):

```ferrum
// Generated automatically from layout declaration:
trait BinarySerialize {
    fn serialize[W: BinaryWrite](&self, w: &mut W): Result[(), IoError] ! IO
}

trait BinaryDeserialize: Sized {
    fn deserialize[R: BinaryRead](r: &mut R): Result[Self, IoError] ! IO
}

// Derive macro for types without layout declarations
@derive(BinarySerialize, BinaryDeserialize)
@binary(byte_order = little_endian)
type Record {
    id:    u32,
    flags: u16,
    data:  [u8; 64],
}
```

### 6.3 Bit-level IO

For protocols and formats that pack fields below byte boundaries:

```ferrum
type BitReader[R: Read] {
    inner:     R,
    bit_buf:   u64,
    bits_left: u8,
}

impl[R: Read] BitReader[R] {
    fn new(inner: R): Self
    fn read_bits(&mut self, n: u8): Result[u64, IoError] ! IO
        requires n <= 64
    fn read_bit(&mut self): Result[bool, IoError] ! IO
    fn align_to_byte(&mut self)   // discard remaining bits in current byte
    fn into_inner(self): R
}

type BitWriter[W: Write] { ... }
impl[W: Write] BitWriter[W] {
    fn write_bits(&mut self, val: u64, n: u8): Result[(), IoError] ! IO
        requires n <= 64
    fn write_bit(&mut self, val: bool): Result[(), IoError] ! IO
    fn flush_bits(&mut self): Result[(), IoError] ! IO  // pad to byte boundary with zeros
}
```

### 6.4 Hexdump and Debug

```ferrum
// Format bytes as hex for debugging
fn hexdump(data: &[u8]): String
fn hexdump_annotated(data: &[u8], annotations: &[(Range[usize], &str)]): String

// Parse hex strings
fn from_hex(s: &str): Result[Vec[u8], HexError]
fn to_hex(data: &[u8]): String
fn to_hex_upper(data: &[u8]): String
```

---

## 7. text — Text IO and Encoding

### 7.1 Encoding Types

There is no "default encoding." Every text operation specifies its encoding.

```ferrum
// The encoding trait
trait Encoding {
    type Error: Error
    fn name(&self): &str

    fn decode(&self, bytes: &[u8]): Result[String, Self.Error]
    fn encode(&self, s: &str): Result[Vec[u8], Self.Error]

    // Streaming versions for large inputs
    fn decoder(&self): Box[dyn Decoder]
    fn encoder(&self): Box[dyn Encoder]
}

// Built-in encodings (not all — just the stdlib ones)
struct Utf8      // the default — &str is always UTF-8
struct Utf16Le
struct Utf16Be
struct Latin1    // ISO-8859-1 — byte value = Unicode codepoint
struct Ascii     // strict ASCII — non-ASCII bytes are errors

// Windows codepages and other legacy encodings are in a separate crate
// (encoding_rs is the reference implementation to wrap)

impl Utf8 {
    fn validate(bytes: &[u8]): Result[&str, Utf8Error]
    fn validate_lossy(bytes: &[u8]): &str  // replaces invalid with U+FFFD
}
```

### 7.2 Text Readers and Writers

This is where bytes become strings. You choose when this happens.

```ferrum
// Wraps a byte reader, decodes on the fly
// The encoding is explicit — there is no default.
type TextReader[R: Read, E: Encoding]  given [A: Allocator]

impl[R: Read, E: Encoding] TextReader[R, E] {
    fn new(inner: R, encoding: E): Self
    fn read_char(&mut self): Result[Option[char], TextError] ! IO
    fn read_line(&mut self): Result[Option[String], TextError] ! IO  // None at EOF
    fn read_to_string(&mut self): Result[String, TextError] ! IO
    fn lines(&mut self): TextLines[Self]  // iterator of Result[String, TextError]
}

// Wraps a byte writer, encodes on the fly
type TextWriter[W: Write, E: Encoding]

impl[W: Write, E: Encoding] TextWriter[W, E] {
    fn new(inner: W, encoding: E): Self
    fn write_str(&mut self, s: &str): Result[(), TextError] ! IO
    fn write_char(&mut self, c: char): Result[(), TextError] ! IO
    fn writeln(&mut self, s: &str): Result[(), TextError] ! IO
    fn flush(&mut self): Result[(), IoError] ! IO
}

// Example: reading a UTF-8 file line by line
let file = File.open("data.txt")?
let reader = TextReader.new(BufReader.new(file), Utf8)
for line in reader.lines() {
    let line = line?    // TextError if invalid UTF-8
    println!("{}", line)
}

// Example: reading a Latin-1 file
let file = File.open("legacy.txt")?
let reader = TextReader.new(BufReader.new(file), Latin1)
let content = reader.read_to_string()?
```

### 7.3 Line Ending Handling

Line endings are explicit, not implicit.

```ferrum
enum LineEnding {
    Lf,     // Unix: \n
    CrLf,   // Windows: \r\n
    Cr,     // Old Mac: \r
    Native, // platform default (Lf on Unix/WASM, CrLf on Windows)
}

// BufRead.lines() always strips the ending.
// If you need the ending preserved, use read_until(b'\n').
// If you need CRLF normalization, use a NormalizingReader.

type NormalizingReader[R: Read] {
    // Normalizes all line endings to Lf on read
}
```

### 7.4 Formatting

No `printf`. No format-string vulnerabilities. Format strings are compile-time constructs.

```ferrum
// Format a value to a string
fn format(args: Arguments): String   // used by format!() macro

// fmt! macro — compile-time format string
let s = fmt!("{name} is {age} years old", name = person.name, age = person.age)
let s = fmt!("{:>10}", value)          // right-align, width 10
let s = fmt!("{:#010x}", value)        // hex, 10 wide, leading zeros, 0x prefix
let s = fmt!("{:.3}", 3.14159)         // 3 decimal places

// Format spec syntax:
// {:}          — use Display trait
// {:#?}        — use Debug trait, pretty-printed
// {:>N}        — right-align, width N
// {:<N}        — left-align, width N
// {:^N}        — center, width N
// {:0N}        — zero-pad, width N
// {:.P}        — precision P (floats: decimal places; strings: max chars)
// {:b}         — binary
// {:o}         — octal
// {:x}         — hex lowercase
// {:X}         — hex uppercase
// {:#x}        — hex with 0x prefix
// {:e}         — scientific notation
// {:+}         — always show sign

// The format string is validated at compile time.
// Mismatched argument count or type is a compile error.
// There is no runtime format string interpretation.
```

### 7.5 Unicode Utilities

```ferrum
// In core.unicode
fn is_alphabetic(c: char): bool
fn is_numeric(c: char): bool
fn is_alphanumeric(c: char): bool
fn is_whitespace(c: char): bool
fn is_control(c: char): bool
fn is_uppercase(c: char): bool
fn is_lowercase(c: char): bool
fn to_uppercase(c: char): ToUppercase   // iterator: some chars expand to multiple
fn to_lowercase(c: char): ToLowercase
fn is_ascii(c: char): bool
fn to_ascii_uppercase(c: char): Option[char]
fn to_ascii_lowercase(c: char): Option[char]

// Unicode normalization - in alloc.unicode
enum NfcNormalization  {}
enum NfdNormalization  {}
enum NfkcNormalization {}
enum NfkdNormalization {}

fn normalize_nfc(s: &str): String
fn normalize_nfd(s: &str): String
fn normalize_nfkc(s: &str): String
fn normalize_nfkd(s: &str): String

// Grapheme cluster iteration (for display width, not char count)
fn grapheme_clusters(s: &str): GraphemeClusters
fn display_width(s: &str): usize   // terminal display columns (CJK = 2)
```

---

## 8. fs — Filesystem

### 8.1 Files

```ferrum
type File

impl File {
    fn open(path: impl AsRef[Path]): Result[Self, IoError] ! IO
        // Opens for reading only
    fn create(path: impl AsRef[Path]): Result[Self, IoError] ! IO
        // Creates or truncates, opens for writing
    fn options(): OpenOptions
        // Builder for full control

    fn metadata(&self): Result[Metadata, IoError] ! IO
    fn set_permissions(&self, perms: Permissions): Result[(), IoError] ! IO
    fn sync_all(&self): Result[(), IoError] ! IO  // flush to OS + durable storage
    fn sync_data(&self): Result[(), IoError] ! IO // flush data only (not metadata)
    fn set_len(&self, size: u64): Result[(), IoError] ! IO  // truncate or extend
    fn try_clone(&self): Result[File, IoError] ! IO
}

impl Read for File
impl Write for File
impl Seek for File

type OpenOptions {
    // builder pattern
    fn new(): Self
    fn read(&mut self, read: bool): &mut Self
    fn write(&mut self, write: bool): &mut Self
    fn append(&mut self, append: bool): &mut Self
    fn truncate(&mut self, truncate: bool): &mut Self
    fn create(&mut self, create: bool): &mut Self
    fn create_new(&mut self, create_new: bool): &mut Self
        // Fails if file exists - no TOCTOU race
    fn mode(&mut self, mode: u32): &mut Self  // Unix permissions
    fn open(&self, path: impl AsRef[Path]): Result[File, IoError] ! IO
}

// No global current-directory mutation.
// Relative paths are resolved against an explicit base or the process cwd (once, at startup).
```

### 8.2 Paths

```ferrum
// Path — a borrowed path slice (like &str for strings)
// PathBuf — an owned path (like String)
// Paths are NOT necessarily UTF-8 on all platforms.
// Use path.to_str(): Option[&str] for UTF-8, or path.display() for lossy display.

type PathBuf  given [A: Allocator]
type Path     // unsized, always behind a reference

impl Path {
    fn new(s: &str): &Self
    fn to_str(&self): Option[&str]
    fn display(&self): Display            // lossy UTF-8 for display only
    fn to_path_buf(&self): PathBuf
    fn is_absolute(&self): bool
    fn is_relative(&self): bool
    fn parent(&self): Option[&Path]
    fn file_name(&self): Option[&OsStr]
    fn file_stem(&self): Option[&OsStr]
    fn extension(&self): Option[&OsStr]
    fn join(&self, other: impl AsRef[Path]): PathBuf
    fn with_extension(&self, ext: impl AsRef[OsStr]): PathBuf
    fn with_file_name(&self, name: impl AsRef[OsStr]): PathBuf
    fn components(&self): Components      // iterator over path components
    fn starts_with(&self, prefix: impl AsRef[Path]): bool
    fn ends_with(&self, suffix: impl AsRef[Path]): bool
    fn strip_prefix(&self, prefix: impl AsRef[Path]): Result[&Path, StripPrefixError]
    fn exists(&self): bool ! IO
    fn is_file(&self): bool ! IO
    fn is_dir(&self): bool ! IO
    fn is_symlink(&self): bool ! IO
    fn canonicalize(&self): Result[PathBuf, IoError] ! IO
    fn metadata(&self): Result[Metadata, IoError] ! IO
}
```

### 8.3 Directory Operations

```ferrum
fn read_dir(path: impl AsRef[Path]): Result[ReadDir, IoError] ! IO
fn create_dir(path: impl AsRef[Path]): Result[(), IoError] ! IO
fn create_dir_all(path: impl AsRef[Path]): Result[(), IoError] ! IO
fn remove_dir(path: impl AsRef[Path]): Result[(), IoError] ! IO
fn remove_dir_all(path: impl AsRef[Path]): Result[(), IoError] ! IO
fn remove_file(path: impl AsRef[Path]): Result[(), IoError] ! IO
fn rename(from: impl AsRef[Path], to: impl AsRef[Path]): Result[(), IoError] ! IO
fn copy(from: impl AsRef[Path], to: impl AsRef[Path]): Result[u64, IoError] ! IO
fn hard_link(src: impl AsRef[Path], dst: impl AsRef[Path]): Result[(), IoError] ! IO
fn symlink(src: impl AsRef[Path], dst: impl AsRef[Path]): Result[(), IoError] ! IO
fn read_link(path: impl AsRef[Path]): Result[PathBuf, IoError] ! IO

// Metadata
type Metadata {
    fn file_type(&self): FileType
    fn len(&self): u64
    fn is_dir(&self): bool
    fn is_file(&self): bool
    fn is_symlink(&self): bool
    fn permissions(&self): Permissions
    fn modified(&self): Result[Timestamp, IoError]
    fn accessed(&self): Result[Timestamp, IoError]
    fn created(&self): Result[Timestamp, IoError]
}

// Directory iterator
type ReadDir  // implements Iterator[Item = Result[DirEntry, IoError]]

type DirEntry {
    fn path(&self): PathBuf
    fn file_name(&self): OsString
    fn file_type(&self): Result[FileType, IoError] ! IO
    fn metadata(&self): Result[Metadata, IoError] ! IO
}

// Convenience functions — read entire files
fn read(path: impl AsRef[Path]): Result[Vec[u8], IoError] ! IO
    // Reads entire file as bytes. Use this.

fn read_text(path: impl AsRef[Path]): Result[String, IoError] ! IO
    // Reads file as UTF-8. Fails if not valid UTF-8.
    // Equivalent to: read(path).and_then(|b| String.from_utf8(b))

fn write(path: impl AsRef[Path], contents: &[u8]): Result[(), IoError] ! IO
    // Creates or truncates, writes all bytes.

fn write_text(path: impl AsRef[Path], contents: &str): Result[(), IoError] ! IO
    // Creates or truncates, writes UTF-8.
    // Equivalent to: write(path, contents.as_bytes())
```

### 8.4 Temporary Files

```ferrum
type TempFile  given [A: Allocator]
    // Deleted when dropped. Guaranteed.

impl TempFile {
    fn new(): Result[Self, IoError] ! IO
    fn new_in(dir: impl AsRef[Path]): Result[Self, IoError] ! IO
    fn path(&self): &Path
    fn persist(self, path: impl AsRef[Path]): Result[File, PersistError] ! IO
        // Keep the file - won't be deleted at drop
    fn keep(self): Result[(File, PathBuf), PersistError] ! IO
}

type TempDir
impl TempDir {
    fn new(): Result[Self, IoError] ! IO
    fn new_in(dir: impl AsRef[Path]): Result[Self, IoError] ! IO
    fn path(&self): &Path
    fn persist(self): Result[PathBuf, PersistError] ! IO
    fn into_path(self): PathBuf  // won't be deleted
    fn close(self): Result[(), IoError] ! IO
}
```

---

*End of io, binary, text, fs modules — see [ferrum-stdlib.md](ferrum-stdlib.md) for index.*
