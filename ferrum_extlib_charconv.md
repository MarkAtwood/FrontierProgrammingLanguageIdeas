# Ferrum Extended Library — charconv: Character Encoding Conversion

**Module path:** `extlib.charconv`
**Layer:** Extended standard library (not stdlib; requires optional dependencies)
**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)

---

## 1. Overview and Rationale

Ferrum's internal text representation is UTF-8, always. `&str` and `String` are guaranteed to be valid UTF-8; the compiler enforces this at the type level. There is no way to construct a `String` containing invalid UTF-8 in safe Ferrum code.

The rest of the world is not UTF-8. Legacy files arrive in Shift-JIS, GB18030, Windows-1252, or ISO-8859-15. Network protocols specify charset headers. OS APIs on Windows return UTF-16. Email bodies are Q-encoded Latin-1. These encodings do not disappear because we prefer UTF-8.

`charconv` is the bridge. It converts between the external byte world and the internal UTF-8 world. It lives at the boundary — reading external data in, writing UTF-8 out, and vice versa. Once data has crossed the boundary and become a `String`, `charconv` is no longer involved.

### Why extlib, Not stdlib

The stdlib includes `text.latin1`, `text.ascii`, and `text.utf16` because these are simple, self-contained, and widely needed (OS APIs, network protocols, and the C ABI use them constantly). Their implementations are a few hundred lines each.

`charconv` is not in stdlib for three reasons:

- **ICU is large.** The full International Components for Unicode library is 25–35 MB of compiled code. Most programs — command-line tools, embedded systems, network daemons — never need Devanagari-to-Braille transliteration or locale-aware Thai word breaking. Pulling ICU into every Ferrum binary by default would be hostile to constrained environments.

- **Most programs don't need legacy encodings.** A web server processing UTF-8 JSON, a systems daemon talking to POSIX APIs, a game engine — none of these need Shift-JIS. Making them pay for it silently is wrong.

- **libiconv is a system dependency.** The conversion engine depends on libiconv (or an equivalent), which is a shared system library, not pure Ferrum. Extlib dependencies are explicitly opted into; stdlib dependencies are implicit.

Programs that do need comprehensive encoding support — email clients, document converters, terminal emulators, database drivers for legacy databases — opt in by depending on `extlib.charconv`.

---

## 2. Design Principle: Conversion Only at System Boundaries

The hard rule: **encoding conversion happens at the boundary where external data enters or leaves the program.** Inside Ferrum code, everything is UTF-8.

```
┌─────────────────────────────────────────────────────────┐
│                     Ferrum program                      │
│                                                         │
│   &str / String (UTF-8, guaranteed)                     │
│                                                         │
│   ┌──────────────────────────────────────────────────┐  │
│   │             charconv boundary                    │  │
│   │                                                  │  │
│   │   bytes → decode() → String                     │  │
│   │   String → encode() → bytes                     │  │
│   └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
          ↑                              ↓
   &[u8] from file/net/OS          &[u8] to file/net/OS
   (Shift-JIS, Latin-1, etc.)     (target encoding)
```

This means:
- You never pass a "Shift-JIS string" around inside your program. There is no such type. You have bytes.
- You decode at the point of ingestion — reading from a file, receiving a network packet, calling an OS API.
- You encode at the point of emission — writing to a file, sending a network packet, calling an OS API that expects a specific encoding.
- Between those two points, you work with `String` and `&str` like any other Ferrum code.

This design eliminates an entire class of bugs: the implicit re-encoding, the "is this string already converted?", the double-conversion, the encoding metadata carried alongside the string.

---

## 3. The `Encoding` Enum

```ferrum
pub enum Encoding {
    // Unicode
    Utf8,
    Utf16Le,
    Utf16Be,
    Utf32Le,
    Utf32Be,

    // ASCII and 8-bit Western
    Ascii,
    Latin1,           // ISO-8859-1. Identical to Windows-1252 for bytes 0x20–0x7E.
    Windows1252,      // Windows Western European (superset of Latin-1 in 0x80–0x9F)
    Windows1250,      // Windows Central European
    Windows1251,      // Windows Cyrillic
    Windows1253,      // Windows Greek
    Windows1254,      // Windows Turkish
    Windows1255,      // Windows Hebrew
    Windows1256,      // Windows Arabic
    Windows1257,      // Windows Baltic
    Windows1258,      // Windows Vietnamese

    // ISO-8859 family
    Iso8859_2,        // Central European
    Iso8859_3,        // South European / Esperanto
    Iso8859_4,        // North European
    Iso8859_5,        // Cyrillic
    Iso8859_6,        // Arabic
    Iso8859_7,        // Greek
    Iso8859_8,        // Hebrew (visual order — legacy)
    Iso8859_9,        // Turkish
    Iso8859_10,       // Nordic
    Iso8859_13,       // Baltic Rim
    Iso8859_14,       // Celtic
    Iso8859_15,       // Western European (ISO-8859-1 + euro sign)
    Iso8859_16,       // South-Eastern European

    // Japanese
    ShiftJis,         // Windows Shift-JIS (CP932 superset)
    EucJp,            // EUC-JP

    // Korean
    EucKr,            // EUC-KR / KS X 1001

    // Chinese Simplified
    Gb18030,          // GB 18030 — superset of GBK and GB2312; mandated in China
    Gbk,              // GBK — subset of GB18030, most common in practice

    // Chinese Traditional
    Big5,             // Big5 — Taiwan, Hong Kong

    // Cyrillic
    Koi8R,            // KOI8-R — Russian
    Koi8U,            // KOI8-U — Ukrainian
    Ibm866,           // CP866 — DOS Cyrillic

    // Thai
    Windows874,       // Windows Thai (TIS-620 superset)

    // Japanese IBM
    Ibm850,           // CP850 — DOS Western European
    Ibm852,           // CP852 — DOS Central European

    // Mac encodings
    MacRoman,         // Mac OS Roman
    MacCyrillic,      // Mac OS Cyrillic

    // EBCDIC
    Ebcdic037,        // IBM EBCDIC US-Canada
    Ebcdic500,        // IBM EBCDIC International

    // Other
    Utf7,             // RFC 2152, for IMAP and legacy email. Decode only.
}
```

### `Encoding::from_label`

MIME charset labels are not consistent with each other. `"utf-8"`, `"UTF-8"`, `"utf8"`, and `"csUTF8"` all mean the same thing. `from_label` implements the WHATWG Encoding Standard label mapping — the same lookup web browsers use:

```ferrum
impl Encoding {
    /// Map an IANA charset label or MIME charset parameter to an Encoding.
    /// Case-insensitive. Trims ASCII whitespace. Returns None for unknown labels.
    /// Follows the WHATWG Encoding Standard label table.
    pub fn from_label(label: &str): Option[Encoding]

    /// Return the canonical IANA name for this encoding.
    pub fn name(self: &Self): &'static str

    /// Return the WHATWG-canonical label (preferred MIME charset parameter value).
    pub fn canonical_label(self: &Self): &'static str

    /// True if this encoding is ASCII-compatible in the range 0x00–0x7F.
    pub fn is_ascii_compatible(self: &Self): bool
}
```

Example:

```ferrum
let enc = Encoding.from_label("windows-1252")  // Some(Encoding.Windows1252)
let enc = Encoding.from_label("x-sjis")        // Some(Encoding.ShiftJis)
let enc = Encoding.from_label("bogus")         // None
let enc = Encoding.from_label("  UTF-8  ")     // Some(Encoding.Utf8)

let name = Encoding.ShiftJis.name()            // "Shift_JIS"
let label = Encoding.Utf8.canonical_label()    // "UTF-8"
```

---

## 4. Encoding Detection

Detection is best-effort. There is no algorithm that can determine encoding with certainty from bytes alone — a byte sequence that is valid UTF-8 might have been intended as Windows-1252 and happened to also be valid UTF-8. Detection returns a confidence score so callers can decide whether to trust it, prompt the user, or fall back to a default.

```ferrum
pub struct DetectionResult {
    pub encoding:   Encoding,
    pub confidence: f32,          // 0.0 (no confidence) to 1.0 (certain)
    pub language:   Option[&'static str],  // BCP 47 language tag if detected, e.g. "ja", "zh-Hans"
}

/// Attempt to detect the encoding of a byte slice.
/// Examines BOMs, byte frequency distributions, and multibyte sequence validity.
/// Short inputs (< 16 bytes) return lower confidence.
/// A Utf8 result with confidence 1.0 means the bytes are valid UTF-8 (not a guess).
pub fn detect(bytes: &[u8]): DetectionResult ! Alloc

/// Return the top N candidate encodings with confidence scores, in descending order.
/// Useful when you want to present choices to a user or try multiple decodings.
pub fn detect_candidates(bytes: &[u8], max: usize): Vec[DetectionResult] ! Alloc
```

Confidence thresholds are a matter of application policy, not library policy. Some guidance:

| Confidence | Interpretation |
|---|---|
| 1.0 | Structurally certain (e.g. valid UTF-8, BOM present) |
| 0.9–1.0 | High confidence, typically safe to use automatically |
| 0.7–0.9 | Reasonable guess; log and proceed, or prompt on mismatch |
| < 0.7 | Weak signal; prefer a user-specified override or documented default |

Detection is performed in memory; the function does not do IO. Pass it the first 4–8 KB of a file for best results without reading the whole thing.

---

## 5. One-Shot Decode API

One-shot decoding reads all bytes at once and returns a `String`. Use this when you have the full buffer in memory.

```ferrum
/// Decode bytes from the given encoding to UTF-8.
/// Returns Err if the bytes contain sequences invalid in that encoding.
pub fn decode(bytes: &[u8], encoding: Encoding): Result[String, DecodeError] ! Alloc

/// Decode bytes, replacing invalid sequences with U+FFFD REPLACEMENT CHARACTER.
/// Never returns an error. The result is always valid UTF-8.
pub fn decode_lossy(bytes: &[u8], encoding: Encoding): String ! Alloc

/// Decode bytes into an existing String buffer.
/// More efficient when decoding many small buffers — avoids repeated allocation.
pub fn decode_into(
    bytes:    &[u8],
    encoding: Encoding,
    output:   &mut String,
): Result[(), DecodeError] ! Alloc
```

Examples:

```ferrum
// Read a Shift-JIS file and decode to UTF-8
let bytes = fs.read("/data/sjis_file.txt")?
let text = charconv.decode(&bytes, Encoding.ShiftJis)?

// Tolerate bad bytes in untrusted network data
let text = charconv.decode_lossy(&packet_payload, Encoding.Windows1252)

// Reuse a buffer across many small decodes
let mut buf = String.with_capacity(4096)
for chunk in chunks {
    charconv.decode_into(&chunk, Encoding.EucJp, &mut buf)?
    process(&buf)
    buf.clear()
}
```

---

## 6. Streaming Decoder

For large inputs that should not be fully buffered, or for data that arrives in pieces (network streams, chunked file reads), the streaming decoder maintains state across calls.

```ferrum
pub struct Decoder {
    // opaque fields
}

impl Decoder {
    /// Create a new streaming decoder for the given encoding.
    pub fn new(encoding: Encoding): Self ! Alloc

    /// Feed a chunk of input bytes to the decoder.
    /// Appends decoded text to `output`.
    /// Set `last` to true on the final chunk to flush any incomplete multibyte sequence.
    /// Returns DecodeResult describing what happened.
    pub fn decode_to(
        self:   &mut Self,
        input:  &[u8],
        output: &mut String,
        last:   bool,
    ): DecodeResult

    /// Reset the decoder to its initial state, keeping the allocated encoding tables.
    pub fn reset(self: &mut Self)

    /// The encoding this decoder was created for.
    pub fn encoding(self: &Self): Encoding
}

pub enum DecodeResult {
    /// All input consumed, output appended. Continue feeding.
    Ok,
    /// Encountered an invalid byte sequence at the given offset.
    /// The decoder has stopped; call reset() or discard the decoder.
    InvalidSequence {
        byte_offset: usize,
        sequence:    [u8; 4],
        len:         u8,   // how many bytes of sequence are valid
    },
    /// Input ended mid-multibyte-sequence and last=true was set.
    /// The incomplete sequence starts at byte_offset within the final chunk.
    IncompleteAtEnd { byte_offset: usize },
}
```

Example: streaming conversion of a large GB18030 network response:

```ferrum
let mut decoder = charconv.Decoder.new(Encoding.Gb18030)
let mut output  = String.new()
let mut buf     = [0u8; 8192]

loop {
    match socket.read(&mut buf)? {
        ReadResult.Eof => {
            match decoder.decode_to(&[], &mut output, true) {
                DecodeResult.Ok                    => break,
                DecodeResult.IncompleteAtEnd { .. } => return Err(Error.TruncatedInput),
                DecodeResult.InvalidSequence { .. } => return Err(Error.InvalidEncoding),
            }
        },
        ReadResult.Data(n) => {
            match decoder.decode_to(&buf[..n], &mut output, false) {
                DecodeResult.Ok                    => (),
                DecodeResult.InvalidSequence { .. } => return Err(Error.InvalidEncoding),
                DecodeResult.IncompleteAtEnd { .. } => unreachable(),
            }
        },
        ReadResult.Err(e) => return Err(Error.Io(e)),
    }
}

process(&output)
```

---

## 7. One-Shot Encode API

One-shot encoding converts a `&str` (always UTF-8) to bytes in the target encoding.

```ferrum
/// Encode a UTF-8 string to the target encoding.
/// Returns Err if any character in `s` cannot be represented in the target encoding.
pub fn encode(s: &str, encoding: Encoding): Result[Vec[u8], EncodeError] ! Alloc

/// Encode, replacing unencodable characters with the encoding's best-available
/// replacement (e.g. '?' for ASCII-only targets, the encoding's own substitution
/// character where one is defined).
pub fn encode_lossy(s: &str, encoding: Encoding): Vec[u8] ! Alloc

/// Encode into an existing byte buffer.
pub fn encode_into(
    s:        &str,
    encoding: Encoding,
    output:   &mut Vec[u8],
): Result[(), EncodeError] ! Alloc
```

Examples:

```ferrum
// Produce a Shift-JIS byte string for a legacy API
let sjis_bytes = charconv.encode(&text, Encoding.ShiftJis)?

// Write a Windows-1252 file, substituting '?' for any unmappable chars
let bytes = charconv.encode_lossy(&text, Encoding.Windows1252)
fs.write("output.txt", &bytes)?
```

### Streaming Encoder

```ferrum
pub struct Encoder {
    // opaque fields
}

impl Encoder {
    pub fn new(encoding: Encoding): Self ! Alloc

    /// Encode a chunk of UTF-8 text, appending encoded bytes to `output`.
    /// Set `last` to true on the final chunk.
    pub fn encode_to(
        self:   &mut Self,
        input:  &str,
        output: &mut Vec[u8],
        last:   bool,
    ): EncodeResult

    pub fn reset(self: &mut Self)
    pub fn encoding(self: &Self): Encoding
}

pub enum EncodeResult {
    Ok,
    /// A character at char_offset cannot be represented in the target encoding.
    Unencodable { char_value: char, char_offset: usize },
}
```

---

## 8. BOM Handling

A Byte Order Mark is a U+FEFF codepoint at the start of a stream that indicates encoding and byte order. BOM stripping is separated from decoding because not all inputs have BOMs, and whether to strip a BOM is a policy decision (some formats require it; others forbid it).

```ferrum
/// Examine the first bytes of a buffer for a BOM.
/// Returns the remaining bytes (after the BOM, if any) and the detected encoding.
/// If no BOM is found, returns the full slice and None.
///
/// Detected BOMs:
///   EF BB BF          → Encoding.Utf8
///   FF FE             → Encoding.Utf16Le
///   FE FF             → Encoding.Utf16Be
///   FF FE 00 00       → Encoding.Utf32Le  (checked before Utf16Le)
///   00 00 FE FF       → Encoding.Utf32Be
///
/// BOM stripping is zero-copy: returns a sub-slice of the input.
pub fn strip_bom(bytes: &[u8]): (&[u8], Option[Encoding])

/// Like strip_bom, but if no BOM is present, fall back to `default`.
pub fn strip_bom_or(bytes: &[u8], default: Encoding): (&[u8], Encoding)
```

Example: reading a file that might be UTF-8, UTF-16 LE, or UTF-16 BE:

```ferrum
let raw = fs.read("possibly_utf16.txt")?
let (payload, bom_encoding) = charconv.strip_bom(&raw)

let encoding = match bom_encoding {
    Some(enc) => enc,
    None      => Encoding.Utf8,   // assume UTF-8 if no BOM
}

let text = charconv.decode(payload, encoding)?
```

---

## 9. Unicode Operations (ICU-backed, Optional Feature)

These functions require the `icu` feature flag. They are compiled out when `icu` is not enabled. The ICU library is dynamically linked when the feature is active.

```ferrum
// Available only when compiled with feature "icu"
#[cfg(feature = "icu")]
pub mod unicode {

    pub enum NormalizationForm {
        Nfc,    // Canonical Decomposition, followed by Canonical Composition
        Nfd,    // Canonical Decomposition
        Nfkc,   // Compatibility Decomposition, followed by Canonical Composition
        Nfkd,   // Compatibility Decomposition
    }

    /// Normalize a string to the given Unicode normalization form.
    /// Returns a new String. The input is unchanged.
    /// Required for correct string comparison in many scripts and protocols.
    pub fn normalize(s: &str, form: NormalizationForm): String ! Alloc

    /// Compare two strings according to locale-specific collation rules.
    /// `locale` is a BCP 47 language tag: "en", "de", "zh-Hans", "ja", "tr", etc.
    /// Returns std.cmp.Ordering.
    ///
    /// Warning: locale-aware collation is expensive. Cache results if comparing
    /// a large set against a fixed key.
    pub fn collate(a: &str, b: &str, locale: &str): std.cmp.Ordering ! Alloc

    /// Case-fold a string to uppercase using locale-sensitive rules.
    /// Essential for Turkish (dotted/dotless I) and Lithuanian.
    pub fn to_uppercase_locale(s: &str, locale: &str): String ! Alloc

    /// Case-fold a string to lowercase using locale-sensitive rules.
    pub fn to_lowercase_locale(s: &str, locale: &str): String ! Alloc

    /// Titlecase a string (first letter of each word uppercased) per locale rules.
    pub fn to_titlecase_locale(s: &str, locale: &str): String ! Alloc

    /// Break a string into grapheme clusters — what a user perceives as "one character."
    /// Emoji sequences, combining marks, and Hangul syllables are single grapheme clusters.
    pub fn grapheme_clusters(s: &str): impl Iterator[&str]

    /// Break a string into words per Unicode word-break rules for the given locale.
    pub fn word_segments(s: &str, locale: &str): impl Iterator[&str] ! Alloc

    /// True if the two strings are canonically equivalent (NFC-compare).
    /// More correct than byte comparison for accented characters.
    pub fn canonical_equal(a: &str, b: &str): bool ! Alloc

    /// Transliterate text from one script to another, per ICU transliterator ID.
    /// Example IDs: "Latin-ASCII", "Any-Latin", "Hiragana-Katakana", "zh-Latn/BGN"
    pub fn transliterate(s: &str, transliterator_id: &str): Result[String, TranslitError] ! Alloc

}
```

Note: `to_uppercase` and `to_lowercase` without a locale are provided by Ferrum's stdlib `String` methods and use Unicode's locale-independent case mapping, which is correct for most uses. The locale-sensitive variants here are needed only when locale matters — Turkish `i`/`I`, Lithuanian dotted-I, and a handful of other cases.

---

## 10. Error Types

```ferrum
/// Error returned when bytes cannot be decoded in the stated encoding.
pub struct DecodeError {
    pub encoding:    Encoding,
    pub byte_offset: usize,            // offset of the first invalid byte in the input
    pub sequence:    [u8; 4],          // the offending byte(s), zero-padded
    pub sequence_len: u8,              // how many bytes of sequence are meaningful (1–4)
}

impl Display for DecodeError {
    fn fmt(&self, f: &mut Formatter): Result[(), IoError] ! IO {
        write(f, "invalid {:?} sequence at byte {}: {:02x?}",
            self.encoding,
            self.byte_offset,
            &self.sequence[..self.sequence_len as usize])
    }
}

impl Error for DecodeError {}

/// Error returned when a Unicode character cannot be encoded in the target encoding.
pub struct EncodeError {
    pub encoding:    Encoding,
    pub char_value:  char,             // the Unicode code point that could not be encoded
    pub char_offset: usize,           // byte offset within the source UTF-8 string
}

impl Display for EncodeError {
    fn fmt(&self, f: &mut Formatter): Result[(), IoError] ! IO {
        write(f, "cannot encode U+{:04X} ({}) in {:?} at byte {}",
            self.char_value as u32,
            self.char_value,
            self.encoding,
            self.char_offset)
    }
}

impl Error for EncodeError {}

/// Error returned by ICU-backed transliteration.
#[cfg(feature = "icu")]
pub struct TranslitError {
    pub transliterator_id: String,
    pub message:           String,
}

impl Display for TranslitError { ... }
impl Error for TranslitError {}
```

---

## 11. Example Usage

### Reading a Shift-JIS File

```ferrum
use extlib.charconv.{self, Encoding}
use std.fs

fn read_sjis_file(path: &str): Result[String, Error] ! IO + Alloc {
    let bytes = fs.read(path)?
    let text  = charconv.decode(&bytes, Encoding.ShiftJis)?
    Ok(text)
}

fn main(): Result[(), Error] ! IO + Alloc {
    let text = read_sjis_file("/data/legacy.txt")?
    print("{}", text)
    Ok(())
}
```

### Detecting the Encoding of Network Data

```ferrum
use extlib.charconv.{self, Encoding}

fn decode_response_body(
    body:            &[u8],
    content_type:    Option[&str],  // e.g. "text/html; charset=windows-1251"
): Result[String, Error] ! Alloc {
    // Try to parse a charset from Content-Type first.
    let declared = content_type
        .and_then(|ct| parse_charset_param(ct))
        .and_then(|label| Encoding.from_label(label))

    let encoding = match declared {
        Some(enc) => enc,
        None => {
            // No declared charset. Detect from content.
            let result = charconv.detect(body)
            if result.confidence < 0.80 {
                // Low confidence: fall back to the web's default.
                Encoding.Windows1252
            } else {
                result.encoding
            }
        },
    }

    Ok(charconv.decode(body, encoding)?)
}
```

### Streaming Conversion from EUC-KR

```ferrum
use extlib.charconv.{self, Encoding, DecodeResult}
use std.io.{Read, ReadResult}

fn convert_euckr_stream[R: Read](
    source: &mut R,
    output: &mut String,
): Result[(), Error] ! IO + Alloc {
    let mut decoder = charconv.Decoder.new(Encoding.EucKr)
    let mut buf = [0u8; 4096]

    loop {
        match source.read(&mut buf)? {
            ReadResult.Eof => {
                match decoder.decode_to(&[], output, true) {
                    DecodeResult.Ok                    => return Ok(()),
                    DecodeResult.IncompleteAtEnd { .. } => return Err(Error.TruncatedInput),
                    DecodeResult.InvalidSequence { byte_offset, .. } =>
                        return Err(Error.BadEncoding(byte_offset)),
                }
            },
            ReadResult.Data(n) => {
                match decoder.decode_to(&buf[..n], output, false) {
                    DecodeResult.Ok => (),
                    DecodeResult.InvalidSequence { byte_offset, .. } =>
                        return Err(Error.BadEncoding(byte_offset)),
                    DecodeResult.IncompleteAtEnd { .. } => unreachable(),
                }
            },
            ReadResult.Err(e) => return Err(Error.Io(e)),
        }
    }
}
```

### Writing a Latin-1 File for a Legacy System

```ferrum
use extlib.charconv.{self, Encoding}
use std.fs

fn write_latin1(path: &str, text: &str): Result[(), Error] ! IO + Alloc {
    // encode() fails if any character cannot be represented in Latin-1.
    // Use encode_lossy() if substitution is acceptable.
    let bytes = charconv.encode(text, Encoding.Latin1)?
    fs.write(path, &bytes)?
    Ok(())
}
```

### Locale-Aware Sort (ICU feature)

```ferrum
use extlib.charconv.unicode

fn sort_names_for_locale(names: &mut Vec[String], locale: &str) ! Alloc {
    names.sort_by(|a, b| unicode.collate(a, b, locale))
}

fn normalize_for_lookup(query: &str): String ! Alloc {
    unicode.normalize(query, unicode.NormalizationForm.Nfkc)
}
```

---

## 12. Dependencies

### Required: libiconv

The conversion engine for non-Unicode encodings is backed by `libiconv` (GNU iconv) or the platform's equivalent:

- **Linux:** GNU libiconv or glibc's built-in iconv (glibc iconv covers the full set of encodings in the `Encoding` enum)
- **macOS / BSD:** libiconv is part of the base system (`/usr/lib/libiconv.dylib`)
- **Windows:** Win32 `MultiByteToWideChar` / `WideCharToMultiByte` for the encodings Windows supports; libiconv static link for others
- **WASI / embedded:** A vendored subset of libiconv with the most common encodings; configure with `charconv_encodings` build flag to control which are included

`charconv` wraps the libiconv C API through Ferrum's FFI layer. The wrapper is `! Unsafe` internally (crossing the C boundary); the public API is safe Ferrum.

### Optional: ICU (feature `"icu"`)

The `unicode` submodule (section 9) requires ICU4C. ICU is not linked by default. To enable:

```toml
# In your project's build config:
[dependencies]
extlib.charconv = { version = "1.0", features = ["icu"] }
```

This links `libicuuc` and `libicudata`. ICU data files are large (~25 MB for full coverage). For size-constrained builds, ICU can be built with a reduced dataset covering only the locales you need.

The `"icu"` feature enables:
- `unicode.normalize`
- `unicode.collate`
- `unicode.to_uppercase_locale` / `to_lowercase_locale` / `to_titlecase_locale`
- `unicode.grapheme_clusters`
- `unicode.word_segments`
- `unicode.canonical_equal`
- `unicode.transliterate`

Without the `"icu"` feature, these functions do not exist; code that references them will not compile, producing a clear diagnostic.

### No Runtime Encoding Tables

`charconv` does not ship its own encoding tables independent of libiconv. The encoding tables are libiconv's — widely tested, maintained, and already on most systems. Shipping a parallel set would create a divergence hazard.

---

## 13. Relationship to stdlib `text` Module

The stdlib `text` module (see `ferrum-stdlib-io.md` §7) handles three encodings:

| Module | Encodings | Notes |
|---|---|---|
| `text.utf8` | UTF-8 | Zero-copy validation, in `core` |
| `text.latin1` | ISO-8859-1 | Infallible decode (every byte valid); in stdlib |
| `text.ascii` | ASCII | Subset of Latin-1; in stdlib |
| `text.utf16` | UTF-16 LE/BE | Common in Windows and Java APIs; in stdlib |

`extlib.charconv` covers everything else. If you need only Latin-1 or UTF-16, use the stdlib modules directly — no dependency on `extlib.charconv` required.

`extlib.charconv.decode` with `Encoding.Latin1`, `Encoding.Ascii`, or `Encoding.Utf16Le`/`Utf16Be` delegates to the stdlib implementations. This means switching from a stdlib call to a charconv call (when the encoding is determined at runtime) has no behavioral difference for those encodings.
