# Ferrum Extended Library — text_layout: Text Shaping and Layout

**Module path:** `extlib.ccsp.text_layout`
**Layer:** Extended standard library (not stdlib; requires optional dependencies)
**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)
**Spec basis:** CCSP `lib_ccsp_text` text shaping and layout specification
**Companion modules:**
- [extlib.ccsp.draw](ferrum_extlib_draw.md) — `DrawContext`, `Color`, `Point`, `Rect`
- [extlib.font](ferrum_extlib_font.md) — `FontCollection`, `FontFace`, `GlyphId`

---

## 1. Overview and Rationale

### Text Is the Hardest Part of UI

Rendering a rectangle or a bezier curve is deterministic: geometry in, pixels out. Rendering text is not. The sequence of Unicode codepoints `U+0645 U+0631 U+062D U+0628 U+0627` is visually `مرحبا` — Arabic for "hello" — and it runs right-to-left, uses contextual letter forms that change based on position in the word, and may contain optional diacritical marks that float above or below base characters without advancing the cursor. The sequence `U+0915 U+094D U+0937` is the Hindi conjunct consonant क्ष, formed from two consonants and a virama that suppresses the inherent vowel and triggers ligature formation. None of this is optional: an Arabic renderer that skips contextual shaping produces unreadable results. An Indic renderer that ignores conjuncts produces the wrong words.

Text layout therefore involves several distinct, well-specified algorithms:

1. **Script detection** — determining which writing system each run of text belongs to, so the correct shaping rules are applied.
2. **Shaping** — converting a sequence of Unicode codepoints into a sequence of positioned glyph IDs, applying ligature substitution, contextual forms, mark placement, kerning, and OpenType feature lookups. This is script-specific and font-specific.
3. **Bidirectional layout** — Unicode Standard Annex #9 defines the algorithm for mixing left-to-right and right-to-left text on the same line. An English sentence containing an Arabic phrase must have the Arabic fragment run right-to-left while the English context runs left-to-right.
4. **Line breaking** — Unicode Standard Annex #14 defines which positions in a string are legal break points. Breaking at arbitrary byte offsets produces incorrect results (breaking in the middle of a grapheme cluster, between a base character and a combining mark, or at a position prohibited by language rules).
5. **Paragraph layout** — distributing shaped runs across lines at legal break points, aligning lines horizontally, computing line metrics, and optionally justifying or truncating.
6. **Hit testing and selection** — converting a display coordinate to a text position (for cursor placement), and converting a text range to display rectangles (for selection highlight). Both must handle bidi text correctly.

### Why HarfBuzz

HarfBuzz is the only open-source text shaper that correctly handles all major writing systems in production use. It implements the OpenType specification for shaping (GSUB for glyph substitution, GPOS for glyph positioning), the Unicode Bidirectional Algorithm, and script-specific shaping for Arabic, Indic scripts (Devanagari, Bengali, Tamil, Telugu, Kannada, Malayalam, Sinhala, and more), Tibetan, Khmer, Myanmar, and dozens of others. It is the shaping engine used by every major Linux desktop (GTK, Qt, Pango), Android, Firefox, Chrome, LibreOffice, and XeTeX.

No alternative provides equivalent correctness across this range of scripts. A custom shaper that handles Latin, Greek, and Cyrillic but not Arabic or Devanagari is not a portable text renderer — it is a Western-text renderer. `extlib.ccsp.text_layout` uses HarfBuzz as its shaping backend.

### Why UAX #9 and UAX #14

The Unicode Bidirectional Algorithm (UAX #9) and the Unicode Line Breaking Algorithm (UAX #14) are Unicode Consortium standards. They define the correct behavior for their respective problems across all scripts. Implementations that deviate from these standards produce text that is rendered differently from every other conformant platform. Ferrum text layout is UAX #9 and UAX #14 conformant.

### The Abstraction Invariant

No type or constant from HarfBuzz, ICU, or any other underlying library appears in the public API of `extlib.ccsp.text_layout`. The widget system, the text editor, the PDF renderer — any code layered above this module — speaks only the types defined here. The shaping backend is an implementation detail. A future implementation could substitute a platform-native shaping API (Core Text, DirectWrite) or a GPU-side shaper without changing any call site. The abstraction boundary is a hard guarantee, not a convention.

All `! Unsafe` operations are confined to the implementation. Every public function is safe to call.

---

## 2. TextStyle — Style Applied to a Run of Text

`TextStyle` carries the typographic attributes that apply to a contiguous run of text. A run inherits a base style and may override any subset of attributes.

### 2.1 FontWeight

```ferrum
/// The weight (stroke thickness) of a font.
///
/// Numeric values follow the CSS font-weight scale and the OpenType
/// `usWeightClass` field. Named constants cover the conventional stops;
/// Custom allows any value in the valid range [1, 1000].
@derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)
pub enum FontWeight {
    /// Weight 100. Hairline stroke.
    Thin,

    /// Weight 200.
    ExtraLight,

    /// Weight 300.
    Light,

    /// Weight 400. Default body text weight.
    Regular,

    /// Weight 500.
    Medium,

    /// Weight 600.
    SemiBold,

    /// Weight 700. Default bold weight.
    Bold,

    /// Weight 800.
    ExtraBold,

    /// Weight 900. Maximum named weight.
    Black,

    /// An explicit numeric weight in [1, 1000].
    ///
    /// Useful for variable fonts with continuous weight axes.
    /// FontWeight::Custom(400) is equivalent to FontWeight::Regular.
    Custom(u16),
}

impl FontWeight {
    /// Returns the numeric weight value (1–1000).
    pub fn value(self: Self): u16
}
```

### 2.2 VariationAxisValue

Variable fonts expose named design axes — weight, width, optical size, and custom axes. Each axis is identified by a four-character OpenType tag and takes a continuous floating-point value within the axis range.

```ferrum
/// A single variable font axis setting.
///
/// `tag` is a four-character OpenType axis tag, e.g. b"wght" for weight,
/// b"wdth" for width, b"ital" for italic, b"opsz" for optical size.
/// Custom axes defined by the font designer use private-use tags.
///
/// `value` is within the axis range declared in the font's `fvar` table.
/// Out-of-range values are clamped to the declared minimum or maximum.
@derive(Debug, Clone, Copy, PartialEq)]
pub struct VariationAxisValue {
    /// Four-byte OpenType axis tag. ASCII characters only.
    pub tag: [u8; 4],

    /// Axis value. Must be within the axis range; clamped if out of range.
    pub value: f32,
}
```

### 2.3 TextStyle

```ferrum
/// Typographic style for a run of text.
///
/// TextStyle is applied to a byte range within an AttributedString. Ranges
/// that have no explicit style inherit the AttributedString's base style.
/// All fields contribute to font selection, glyph shaping, and rendering.
@derive(Debug, Clone, PartialEq)]
pub struct TextStyle {
    /// Font family name, e.g. "Noto Sans", "Helvetica Neue", "Source Serif 4".
    ///
    /// The font collection is queried for a face matching this family, the
    /// requested weight, and the italic flag. If the exact face is not found
    /// the collection performs best-match substitution.
    pub font_family: String,

    /// Font size in logical units (points at 1x DPI; CSS pixels at screen DPI).
    ///
    /// Must be positive. Values <= 0.0 are treated as 12.0.
    pub font_size: f32,

    /// Stroke weight of the font.
    pub weight: FontWeight,

    /// Whether to request an italic face.
    ///
    /// If no italic face exists in the family, the renderer may apply
    /// a synthetic oblique slant. The font collection reports whether
    /// synthesized slant was applied via FontFace::is_synthetic_italic.
    pub italic: bool,

    /// Foreground color. Applied when rendering each glyph run.
    pub color: Color,

    /// Draw an underline beneath this run.
    ///
    /// Underline thickness and offset are taken from the font's `post` table
    /// (underlineThickness, underlinePosition). If the font does not specify
    /// these, reasonable defaults (1px thick, 1px below baseline) are used.
    pub underline: bool,

    /// Draw a strikethrough through the vertical midpoint of this run.
    ///
    /// Thickness is taken from the font's `OS/2` table (strikeoutSize,
    /// strikeoutPosition). If absent, defaults to 1px at x-height midpoint.
    pub strikethrough: bool,

    /// Additional horizontal space between letters, in logical units.
    ///
    /// Positive values expand; negative values condense. Applied after
    /// shaping, so it does not affect ligature formation or kerning lookup —
    /// it is added uniformly after all OpenType processing.
    ///
    /// Default: 0.0
    pub letter_spacing: f32,

    /// Additional horizontal space added after each space character (U+0020).
    ///
    /// Applied after shaping. Does not affect non-space word separators.
    ///
    /// Default: 0.0
    pub word_spacing: f32,

    /// Variable font axis overrides.
    ///
    /// Overrides the named instance defaults for the matched face. If the face
    /// is not a variable font, this list is ignored. The weight axis (b"wght")
    /// is set automatically from the `weight` field unless also present here,
    /// in which case the explicit axis value takes precedence.
    pub variation_axes: Vec[VariationAxisValue],
}

impl TextStyle {
    /// Construct a minimal style: family, size, Regular weight, no decoration.
    ///
    /// color defaults to Color::BLACK (opaque black in linear light).
    /// All spacing values default to 0.0. No variation axes are set.
    pub fn new(font_family: String, font_size: f32): TextStyle

    /// Returns a clone of this style with the given font family substituted.
    pub fn with_family(self: &Self, family: String): TextStyle

    /// Returns a clone of this style with the given color substituted.
    pub fn with_color(self: &Self, color: Color): TextStyle

    /// Returns a clone of this style with the given weight substituted.
    pub fn with_weight(self: &Self, weight: FontWeight): TextStyle

    /// Returns a clone of this style with italic toggled.
    pub fn with_italic(self: &Self, italic: bool): TextStyle
}
```

---

## 3. AttributedString — Text with Per-Range Styles

`AttributedString` pairs a UTF-8 string with a set of style annotations keyed by byte ranges. It is the primary input to the shaping and layout pipeline. Style ranges may overlap: the later `add_style` call for any given byte wins field-by-field over an earlier one (last-writer-wins per field, not per range).

```ferrum
/// A UTF-8 string annotated with typographic style ranges.
///
/// The text is stored as a single UTF-8 String. Styles are applied as
/// byte-range annotations. Byte ranges must be aligned to UTF-8 character
/// boundaries; misaligned offsets cause TextLayoutError::InvalidByteOffset.
///
/// An AttributedString is immutable after construction via the builder
/// methods. Pass it to Shaper::shape to begin layout.
pub struct AttributedString { ... }

impl AttributedString {
    /// Construct an attributed string from a UTF-8 string slice.
    ///
    /// The text is copied into an internal buffer. The base style is
    /// initially unset; call set_base_style before shaping or a default
    /// 12pt "sans-serif" black Regular style is applied.
    pub fn new(text: &str): AttributedString

    /// Set the default style applied to byte ranges with no explicit style.
    ///
    /// Must be called before shaping if any range has no explicit style
    /// annotation. Replaces any previously set base style.
    pub fn set_base_style(self: &mut Self, style: TextStyle)

    /// Apply a style to a byte range in the text.
    ///
    /// `range` is a half-open byte range [start, end) into the UTF-8 text.
    /// Both `start` and `end` must lie on UTF-8 character boundaries.
    /// Ranges may overlap; the field values of later calls win.
    ///
    /// Returns Err(TextLayoutError::InvalidByteOffset) if either endpoint
    /// is not on a UTF-8 character boundary or is beyond text length.
    pub fn add_style(
        self: &mut Self,
        range: Range[usize],
        style: TextStyle,
    ): Result[(), TextLayoutError]

    /// Set the BCP 47 language tag for the entire string.
    ///
    /// Language affects shaping: some OpenType features are language-specific
    /// (e.g. Turkish dotless-i, Serbian Cyrillic forms, Dutch IJ ligature).
    /// Example values: "en", "ar", "hi", "tr-TR", "sr-Latn".
    ///
    /// If not set, language is inferred from the Unicode script of each run.
    /// Explicit language tags produce more correct shaping for language-specific
    /// OpenType features.
    pub fn set_language(self: &mut Self, lang: &str)

    /// Set a BCP 47 language tag for a specific byte range.
    ///
    /// Overrides the string-level language for this range only.
    /// Useful for multilingual paragraphs where different runs are in
    /// different languages.
    pub fn set_language_range(
        self: &mut Self,
        range: Range[usize],
        lang: &str,
    ): Result[(), TextLayoutError]

    /// The underlying UTF-8 text.
    pub fn text(self: &Self): &str

    /// The total byte length of the text.
    pub fn len(self: &Self): usize

    /// Returns true if the text is empty.
    pub fn is_empty(self: &Self): bool

    /// The effective style at a given byte offset.
    ///
    /// Returns the base style with all range-style overrides applied for
    /// every style range that covers `byte_offset`.
    ///
    /// Returns Err if byte_offset is beyond text length.
    pub fn style_at(self: &Self, byte_offset: usize): Result[TextStyle, TextLayoutError]

    /// Iterator over (byte_range, style) pairs in logical order.
    ///
    /// Yields contiguous ranges where the effective style is uniform.
    /// Adjacent ranges with identical styles are merged. Covers the entire
    /// text from byte 0 to the last byte.
    pub fn style_runs(self: &Self): impl Iterator[Item=(Range[usize], &TextStyle)]
}
```

---

## 4. Text Shaping — HarfBuzz Wrapper

The shaper converts an `AttributedString` into a `ShapedText`: a sequence of glyph runs, each run containing a list of glyph IDs with their positions and advances, ready for rendering.

### 4.1 Core Types

```ferrum
/// A Unicode script identifier.
///
/// Identifies the writing system for a run of text, used to select the
/// correct OpenType shaping tables. Based on ISO 15924 script codes.
/// The variants cover scripts in common use; Script::Unknown covers
/// characters not assigned to a known script.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Script {
    Arabic,
    Armenian,
    Bengali,
    Bopomofo,
    Cyrillic,
    Devanagari,
    Ethiopic,
    Georgian,
    Greek,
    Gujarati,
    Gurmukhi,
    Han,
    Hangul,
    Hebrew,
    Hiragana,
    Kannada,
    Katakana,
    Khmer,
    Latin,
    Malayalam,
    Myanmar,
    Oriya,
    Sinhala,
    Tamil,
    Telugu,
    Thaana,
    Thai,
    Tibetan,
    /// Text with no dominant script, or a script not listed here.
    Unknown,
}

/// A single shaped glyph with its position and cluster mapping.
///
/// `cluster` links the glyph back to the source text for hit testing and
/// selection. Multiple glyphs may share the same cluster (ligatures). A
/// cluster may have no corresponding glyph (deleted characters). Always
/// iterate clusters via ShapedRun::glyph_clusters, not raw glyph indices.
@derive(Debug, Clone, Copy)]
pub struct ShapedGlyph {
    /// The glyph index in the font face.
    pub glyph_id: GlyphId,

    /// Byte offset in the original AttributedString text of the first
    /// codepoint that produced this glyph. Used for hit testing.
    pub cluster: u32,

    /// Horizontal advance: how far the cursor moves right after this glyph.
    /// Negative in RTL runs (cursor moves left).
    pub x_advance: f32,

    /// Vertical advance: non-zero only in vertical writing mode scripts.
    pub y_advance: f32,

    /// Horizontal offset from the cursor position to draw this glyph.
    /// Used for mark positioning, kerning pair adjustments, and GPOS lookups.
    pub x_offset: f32,

    /// Vertical offset from the baseline to draw this glyph.
    /// Positive is up. Used for superscripts, subscripts, and mark positioning.
    pub y_offset: f32,
}

/// A run of glyphs that share a font face, size, color, bidi level, and script.
///
/// Runs are the atomic rendering unit. A run is guaranteed homogeneous: every
/// glyph uses the same font face at the same size. Runs do not span bidi level
/// boundaries, script boundaries, or style boundaries that change font or size.
///
/// Glyphs within a run are in visual order for the run's direction: left-to-right
/// for LTR runs, right-to-left for RTL runs. The run's position in a LayoutLine
/// is determined by paragraph layout and bidi reordering.
@derive(Debug, Clone)]
pub struct ShapedRun {
    /// The font face used for every glyph in this run.
    pub font_face: Arc[FontFace],

    /// Font size in logical units.
    pub font_size: f32,

    /// Foreground color for rendering this run.
    pub color: Color,

    /// Glyphs in this run, in logical order (not yet bidi-reordered to visual).
    pub glyphs: Vec[ShapedGlyph],

    /// Unicode Bidirectional Algorithm embedding level for this run.
    /// Even levels are LTR; odd levels are RTL.
    pub bidi_level: u8,

    /// Writing system detected for this run.
    pub script: Script,

    /// BCP 47 language tag for this run. Empty string if undetected.
    pub language: String,

    /// Whether this run carries underline decoration.
    pub underline: bool,

    /// Whether this run carries strikethrough decoration.
    pub strikethrough: bool,
}

impl ShapedRun {
    /// The total advance width of this run in logical units.
    ///
    /// Sum of x_advance over all glyphs. For RTL runs the value is positive;
    /// direction is encoded in bidi_level, not the sign of advance.
    pub fn advance_width(self: &Self): f32

    /// Returns the ascending height above the baseline for this run's font
    /// at its size. From the font's `hhea` or `OS/2` ascender.
    pub fn ascender(self: &Self): f32

    /// Returns the descending depth below the baseline (positive value).
    pub fn descender(self: &Self): f32
}

/// The fully shaped representation of an AttributedString.
///
/// Contains all runs in logical order. Paragraph layout (Paragraph::layout)
/// distributes these runs into LayoutLines at legal break points.
@derive(Debug)]
pub struct ShapedText {
    /// All shaped runs, in logical (not visual) order.
    pub runs: Vec[ShapedRun],

    /// The UTF-8 text that was shaped. Kept for hit testing and selection.
    pub text: String,
}
```

### 4.2 Shaper

```ferrum
/// Text shaper. Wraps HarfBuzz.
///
/// A Shaper holds the HarfBuzz buffer and font cache state. It is not
/// Send; create one Shaper per render thread. The underlying FontCollection
/// is shared (Arc) and may be used from multiple threads.
///
/// Shaping is the most computationally expensive step in text layout.
/// Cache ShapedText results when the input AttributedString has not changed.
pub struct Shaper { ... }

impl Shaper {
    /// Construct a new Shaper backed by the given font collection.
    ///
    /// The font collection is queried when shaping to resolve font families
    /// to FontFace instances, perform font fallback for codepoints not
    /// covered by the primary face, and apply variable-font axis settings.
    ///
    /// Returns Err if HarfBuzz initialization fails (out of memory).
    pub fn new(font_collection: Arc[FontCollection]): Result[Shaper, TextLayoutError]

    /// Shape an AttributedString into positioned glyph runs.
    ///
    /// Performs in order:
    ///   1. Unicode script detection per codepoint (Unicode Script property).
    ///   2. Bidirectional analysis (UAX #9) to assign embedding levels.
    ///   3. Run segmentation: split the string at every boundary where
    ///      font family, font size, script, or bidi level changes.
    ///   4. Font resolution: for each run, select the best matching FontFace
    ///      from the collection, with fallback for missing codepoints.
    ///   5. HarfBuzz shaping for each run: ligature substitution (GSUB),
    ///      mark and kern positioning (GPOS), contextual forms.
    ///   6. Assembly of ShapedRun values in logical order.
    ///
    /// The returned ShapedText contains runs in logical (reading) order.
    /// Call Paragraph::layout to distribute runs into lines.
    ///
    /// Returns Err if font resolution or shaping fails for any run.
    pub fn shape(
        self: &mut Self,
        text: &AttributedString,
    ): Result[ShapedText, TextLayoutError]

    /// Enable or disable a global OpenType feature for subsequent shape calls.
    ///
    /// Features are identified by four-character OpenType tags, e.g.
    /// b"liga" (standard ligatures), b"kern" (kerning), b"calt" (contextual
    /// alternates), b"smcp" (small capitals), b"frac" (diagonal fractions).
    ///
    /// By default, the standard set of features enabled by HarfBuzz is used:
    /// kern, liga, calt, clig, locl, mark, mkmk, and script-required features.
    /// Disable kern or liga only for specific rendering needs (e.g. code text).
    pub fn set_feature(self: &mut Self, tag: [u8; 4], enabled: bool)

    /// Access the font collection backing this shaper.
    pub fn font_collection(self: &Self): &Arc[FontCollection]
}
```

---

## 5. Bidirectional Text — UAX #9

The Unicode Bidirectional Algorithm (UAX #9) determines the visual order of characters in text that mixes left-to-right and right-to-left scripts. `Shaper::shape` calls the bidi analyzer internally. The types below are exposed so that callers who need raw bidi data (e.g. for custom rendering, accessibility metadata, or selection highlight geometry) can access it directly.

```ferrum
/// Base paragraph direction for bidirectional analysis.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Direction {
    /// Left-to-right paragraph direction.
    Ltr,

    /// Right-to-left paragraph direction.
    Rtl,

    /// Determine direction from the first strong directional character in
    /// the text (Unicode rule P2/P3). Falls back to Ltr if no strong
    /// directional character is found.
    Auto,
}

/// The result of UAX #9 analysis for one paragraph.
///
/// `levels[i]` is the UAX #9 embedding level for byte `i` in the analyzed
/// string. Even levels are LTR; odd levels are RTL. The maximum level is 125.
///
/// `visual_order[i]` is the visual position of logical run `i` after bidi
/// reordering. Run 0 is the first run in logical order; its visual_order value
/// is its index in a left-to-right visual sequence.
@derive(Debug, Clone)]
pub struct BidiResult {
    /// Per-byte embedding level. Parallel to the input string's bytes.
    ///
    /// Not all byte positions are the start of a character; levels for
    /// continuation bytes of multibyte UTF-8 sequences equal the level
    /// of the leading byte.
    pub levels: Vec[u8],

    /// Mapping from logical run index to visual (display) order index.
    pub visual_order: Vec[usize],

    /// The resolved paragraph base direction (Ltr or Rtl, never Auto).
    pub base_direction: Direction,
}

/// Bidirectional text analyzer.
///
/// Implements Unicode Standard Annex #9 (The Unicode Bidirectional Algorithm).
/// Produces embedding levels and visual ordering for a paragraph of text.
///
/// Shaper::shape calls BidiAnalyzer internally. Direct use is for callers
/// that need raw bidi data independent of shaping.
pub struct BidiAnalyzer { ... }

impl BidiAnalyzer {
    /// Construct a new BidiAnalyzer.
    pub fn new(): BidiAnalyzer

    /// Analyze a paragraph of UTF-8 text.
    ///
    /// `text` must be a single paragraph (no hard newlines). If the text
    /// contains U+000A (LF) or U+000D (CR) line separators, call this
    /// function once per paragraph, splitting at those characters.
    ///
    /// `base_direction` is the paragraph embedding level. Pass Direction::Auto
    /// when the direction is unknown and should be detected from content.
    ///
    /// The returned BidiResult is valid for the lifetime of `text` as long
    /// as `text` is not modified.
    pub fn analyze(self: &Self, text: &str, base_direction: Direction): BidiResult
}
```

---

## 6. Line Breaking — UAX #14

The Unicode Line Breaking Algorithm (UAX #14) classifies every position in a string as a mandatory break, an allowed break, or a prohibited break. `Paragraph::layout` calls the line breaker internally. The types are exposed for callers implementing custom layout.

```ferrum
/// Classification of a line break opportunity.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BreakOpportunity {
    /// A line break must occur here (paragraph separator, hard newline).
    Mandatory,

    /// A line break is allowed here but not required.
    /// The layout engine breaks here when the current line is full.
    Allowed,

    /// A line break is prohibited here (within a no-break sequence).
    /// The layout engine must not break here regardless of line length.
    Never,
}

/// A single line break opportunity position.
///
/// `byte_offset` is the byte position in the source string immediately
/// after which the break opportunity exists. A break at this position
/// means the line ends before the character at `byte_offset`.
@derive(Debug, Clone, Copy)]
pub struct LineBreakOpportunity {
    /// Byte offset in the source string of this opportunity.
    pub byte_offset: usize,

    /// The type of break opportunity at this position.
    pub opportunity: BreakOpportunity,
}

/// UAX #14 line break analyzer.
///
/// Computes line break opportunities for a UTF-8 string. Opportunities
/// are based on the Unicode Line_Break property of each character, plus
/// the pair-table rules of UAX #14.
///
/// Paragraph::layout calls LineBreaker internally. Direct use is for
/// callers implementing custom wrapping logic.
pub struct LineBreaker { ... }

impl LineBreaker {
    /// Construct a new LineBreaker.
    pub fn new(): LineBreaker

    /// Compute all line break opportunities in `text`.
    ///
    /// Returns opportunities in byte offset order. The list always includes
    /// an entry at the last byte of the string (end-of-paragraph).
    ///
    /// Callers selecting where to break a line scan the list for the last
    /// Allowed opportunity before the line overflows, or the first Mandatory
    /// opportunity, whichever comes first.
    pub fn breaks(self: &Self, text: &str): Vec[LineBreakOpportunity]
}
```

---

## 7. Paragraph Layout

Paragraph layout distributes `ShapedRun` values across `LayoutLine` values at UAX #14 break points, applies bidi reordering within each line, computes line metrics, and optionally applies alignment, justification, or overflow handling.

### 7.1 ParagraphStyle

```ferrum
/// Line height specification.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum LineHeight {
    /// 1.2 times the font size of the dominant run on each line.
    ///
    /// This is the CSS `normal` value and the default for body text.
    /// The multiplier 1.2 matches the typographic convention of most
    /// print and screen fonts for comfortable single-spaced reading.
    Normal,

    /// A multiplier applied to the font size of the dominant run.
    ///
    /// Factor(1.5) gives "one-and-a-half spacing". Factor(2.0) gives
    /// "double spacing". Values below 1.0 produce overlapping lines.
    Factor(f32),

    /// An explicit line height in logical units, independent of font size.
    ///
    /// All lines have exactly this height regardless of the glyphs on them.
    /// Use when precise vertical rhythm is required.
    Px(f32),
}

/// Horizontal alignment of lines within the paragraph's available width.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TextAlignment {
    /// Align to the start edge (left for LTR paragraphs, right for RTL).
    Start,

    /// Align to the end edge (right for LTR paragraphs, left for RTL).
    End,

    /// Center each line within the available width.
    Center,

    /// Expand word and letter spacing on all lines except the last so
    /// each line fills the available width exactly.
    ///
    /// The last line (or the only line) is Start-aligned.
    Justify,
}

/// Behavior when text overflows the available number of lines or width.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TextOverflow {
    /// Clip content that exceeds the constraint. No ellipsis is inserted.
    Clip,

    /// Replace the end of the last visible line with `…` (U+2026 HORIZONTAL
    /// ELLIPSIS) such that the resulting line fits within max_width.
    ///
    /// The ellipsis is rendered in the same style as the last glyph it
    /// replaces.
    Ellipsis,

    /// Like Ellipsis, but the break falls on a word boundary rather than
    /// a glyph boundary. The last whole word before overflow is kept; the
    /// partial word is replaced with `…`. Preferred for most body text.
    EllipsisWord,
}

/// Typographic style for a paragraph of text.
///
/// ParagraphStyle controls how ShapedText is distributed into lines,
/// how lines are aligned, and what happens when content overflows.
@derive(Debug, Clone)]
pub struct ParagraphStyle {
    /// Horizontal alignment of lines in the paragraph.
    pub alignment: TextAlignment,

    /// Line height specification.
    pub line_height: LineHeight,

    /// Maximum number of lines to produce.
    ///
    /// None means no limit. Some(n) means the layout stops after n lines;
    /// remaining content is subject to the `overflow` setting.
    pub max_lines: Option[u32],

    /// What to do when content overflows `max_lines` or `max_width`.
    pub overflow: TextOverflow,

    /// Width of a tab stop in logical units.
    ///
    /// Tab characters (U+0009) advance the cursor to the next multiple of
    /// this value. Default: 32.0 (four em-widths at 8pt body text).
    pub tab_size: f32,
}

impl ParagraphStyle {
    /// Default paragraph style: Start alignment, Normal line height,
    /// no max_lines limit, Clip overflow, 32px tab size.
    pub fn default(): ParagraphStyle
}
```

### 7.2 Layout Output Types

```ferrum
/// A glyph run placed at an absolute position within a LayoutLine.
@derive(Debug, Clone)]
pub struct PositionedRun {
    /// The shaped run to render.
    pub run: ShapedRun,

    /// Horizontal offset from the line's left edge to this run's origin.
    ///
    /// This is the x position of the first glyph's advance origin.
    /// For RTL runs, glyphs are drawn with decreasing x; this is still
    /// the left edge of the run's bounding box after bidi reordering.
    pub x: f32,

    /// Vertical offset from the paragraph origin to this run's baseline.
    ///
    /// Equals the enclosing LayoutLine::baseline_y.
    pub y: f32,
}

/// A single line in a laid-out paragraph.
///
/// Runs are in visual (display) order: run[0] is the leftmost visible run
/// regardless of the logical direction of the text.
@derive(Debug, Clone)]
pub struct LayoutLine {
    /// Runs in visual (left-to-right display) order.
    pub runs: Vec[PositionedRun],

    /// Y coordinate of the text baseline from the paragraph origin.
    ///
    /// Glyphs sit on this baseline. Descenders extend below it.
    pub baseline_y: f32,

    /// Height above the baseline (positive, in logical units).
    pub ascender: f32,

    /// Depth below the baseline (positive, in logical units).
    pub descender: f32,

    /// The total advance width of all runs on this line.
    ///
    /// For Start-aligned text this equals the width of the content.
    /// For Justify-aligned text this equals `max_width`.
    pub width: f32,
}

impl LayoutLine {
    /// Total line height: ascender + descender.
    pub fn height(self: &Self): f32

    /// Bounding rectangle of this line, in paragraph-local coordinates.
    /// Origin is (0, baseline_y - ascender); size is (width, height).
    pub fn bounding_rect(self: &Self): Rect
}

/// A fully laid-out paragraph ready for rendering and hit testing.
@derive(Debug)]
pub struct ParagraphLayout {
    /// Lines in top-to-bottom order.
    pub lines: Vec[LayoutLine],

    /// Total height of all lines including inter-line spacing.
    pub total_height: f32,

    /// The max_width passed to layout. Used by hit_test and selection_rects.
    pub max_width: f32,

    /// The UTF-8 text from the ShapedText. Kept for hit testing.
    pub text: String,
}
```

### 7.3 Paragraph — Layout Functions

```ferrum
/// Paragraph layout engine.
///
/// Paragraph holds a LineBreaker and performs line breaking, bidi reordering,
/// alignment, and overflow handling. It is stateless between calls; create one
/// Paragraph and reuse it across multiple layout operations.
pub struct Paragraph { ... }

impl Paragraph {
    /// Construct a new Paragraph layout engine.
    pub fn new(): Paragraph

    /// Lay out a ShapedText at a given maximum width.
    ///
    /// Distributes runs across lines at UAX #14 break points such that no
    /// line exceeds max_width. Applies bidi reordering within each line.
    /// Uses ParagraphStyle::default() for alignment and overflow.
    ///
    /// For full control over style, use layout_styled.
    pub fn layout(
        self: &mut Self,
        shaped: &ShapedText,
        max_width: f32,
    ): Result[ParagraphLayout, TextLayoutError]

    /// Lay out a ShapedText with explicit style control.
    ///
    /// Performs the full layout pipeline:
    ///   1. UAX #14 line break analysis.
    ///   2. Greedy line-filling: accumulate runs until the next break
    ///      would overflow max_width, then start a new line.
    ///   3. Bidi reordering within each line (visual order).
    ///   4. Horizontal alignment per ParagraphStyle::alignment.
    ///      For Justify, adjusts word_spacing and letter_spacing on
    ///      all non-last lines to reach exactly max_width.
    ///   5. Vertical placement: baseline_y for each line derived from
    ///      ParagraphStyle::line_height and the line's font metrics.
    ///   6. Overflow handling: if max_lines is Some(n), lines beyond n
    ///      are dropped and TextOverflow is applied to line n.
    ///
    /// Justification does not modify the ShapedRun values; adjusted spacing
    /// is stored only in the PositionedRun positions.
    pub fn layout_styled(
        self: &mut Self,
        attributed: &AttributedString,
        max_width: f32,
        style: &ParagraphStyle,
        shaper: &mut Shaper,
    ): Result[ParagraphLayout, TextLayoutError]
}
```

---

## 8. Hit Testing — Display Coordinates to Text Positions

Hit testing converts a display coordinate (e.g. a mouse click position relative to the paragraph origin) to a byte offset in the source text, and the inverse: a byte offset to a caret rectangle.

### 8.1 Types

```ferrum
/// Whether a caret position is before or after the character at a given offset.
///
/// Affinity matters at bidi run boundaries where two adjacent visual positions
/// correspond to the same byte offset (the boundary between an LTR run and an
/// RTL run immediately to its right). Upstream means the caret is logically
/// before the character at byte_offset (associated with the run to the left);
/// Downstream means it is logically after.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TextAffinity {
    /// Caret is associated with the character preceding byte_offset
    /// (the end of the run to the upper/left in visual order).
    Upstream,

    /// Caret is associated with the character at byte_offset
    /// (the start of the run to the lower/right in visual order).
    Downstream,
}

/// The result of a hit test at a display coordinate.
@derive(Debug, Clone, Copy)]
pub struct HitTestResult {
    /// Byte offset in the source text of the character nearest the hit point.
    pub byte_offset: usize,

    /// Index of the Unicode scalar value (character) nearest the hit point.
    ///
    /// char_index counts Unicode scalar values, not bytes. Useful for
    /// logical cursor movement (next/prev character) in editors.
    pub char_index: usize,

    /// Whether the hit fell before or after the character at byte_offset.
    pub affinity: TextAffinity,

    /// Index of the line (0-based) in ParagraphLayout::lines that the hit
    /// fell on.
    pub line_index: usize,
}

/// The display rectangle for a text insertion caret.
@derive(Debug, Clone, Copy)]
pub struct CaretPosition {
    /// Horizontal position of the caret, in paragraph-local logical units.
    pub x: f32,

    /// Y coordinate of the top of the caret rectangle (baseline - ascender).
    pub line_y: f32,

    /// Height of the caret rectangle (ascender + descender of the line).
    pub height: f32,
}
```

### 8.2 ParagraphLayout Hit Test Methods

```ferrum
impl ParagraphLayout {
    /// Hit test a display coordinate.
    ///
    /// `point` is in paragraph-local coordinates (origin at the top-left of
    /// the paragraph bounding box, y increases downward).
    ///
    /// If the point is above all lines, returns the start of the first line.
    /// If the point is below all lines, returns the end of the last line.
    /// If the point is to the left of a line, returns the start of that line.
    /// If the point is to the right of a line, returns the end of that line.
    ///
    /// For bidi text, the returned byte_offset is the logical character
    /// nearest the visual hit position. Affinity resolves ambiguity at
    /// bidi run boundaries.
    pub fn hit_test(self: &Self, point: Point): HitTestResult

    /// Compute the caret rectangle for a byte offset.
    ///
    /// `byte_offset` must be on a Unicode scalar value boundary (i.e. the
    /// start of a UTF-8 encoded character or the end of the string).
    ///
    /// Returns the visual caret position for inserting text before the
    /// character at `byte_offset`. In bidi text this may differ from the
    /// logical position; the caret is always placed visually.
    ///
    /// Returns Err if byte_offset is not on a character boundary.
    pub fn position_of(
        self: &Self,
        byte_offset: usize,
    ): Result[CaretPosition, TextLayoutError]
}
```

---

## 9. Selection Rectangles

In bidi text, a logically contiguous selection may not correspond to a visually contiguous range. An English sentence with an embedded Arabic phrase, when selected from an English word through the Arabic phrase to another English word, produces up to three visual rectangles: one at the start of the English text, one covering the Arabic fragment (which is visually between the English portions), and one at the end of the English text. `selection_rects` handles this correctly.

```ferrum
impl ParagraphLayout {
    /// Compute display rectangles covering the selection [start_byte, end_byte).
    ///
    /// `start_byte` and `end_byte` are byte offsets into the source text.
    /// They must be on Unicode scalar value boundaries. The range is
    /// half-open: start_byte is included, end_byte is excluded.
    ///
    /// Returns a list of non-overlapping Rects in paragraph-local coordinates.
    /// The list may contain multiple rectangles when the selection spans a
    /// bidi run boundary (RTL text embedded in LTR context or vice versa),
    /// or spans multiple lines.
    ///
    /// Rectangles are ordered top-to-bottom, then left-to-right within a line.
    /// Each rectangle covers the full line height (ascender + descender) of
    /// the line it belongs to.
    ///
    /// Returns an empty Vec if start_byte == end_byte.
    /// Returns Err if either offset is not on a character boundary.
    pub fn selection_rects(
        self: &Self,
        start_byte: usize,
        end_byte: usize,
    ): Result[Vec[Rect], TextLayoutError]

    /// Bounding rectangles for each line in the paragraph.
    ///
    /// Returns one Rect per entry in `self.lines`, in top-to-bottom order.
    /// Each Rect spans the full line height and the full paragraph width
    /// (not just the content width), suitable for line-level selection
    /// highlighting in editors.
    pub fn line_rects(self: &Self): Vec[Rect]
}
```

---

## 10. Rendering Glyph Runs to DrawContext

The drawing layer (`extlib.ccsp.draw`) renders positioned glyphs via `DrawContext::draw_glyph_run`. The functions below bridge the layout layer to the drawing layer.

```ferrum
/// Render a fully laid-out paragraph to a DrawContext.
///
/// Iterates all lines and runs in `layout` in visual order, calling
/// `ctx.draw_glyph_run` for each PositionedRun. The glyph run's color
/// from the ShapedRun is used as a solid paint. No additional clipping
/// is applied; if the paragraph overflows the DrawContext surface, glyphs
/// outside the surface are clipped by the backend.
///
/// `origin` is the top-left corner of the paragraph in the DrawContext's
/// coordinate space.
///
/// Decoration (underline, strikethrough) is drawn separately for each run
/// via `ctx.fill_rect`, using the run's color and font metrics for position
/// and thickness.
pub fn render_paragraph(
    layout: &ParagraphLayout,
    ctx: &mut DrawContext,
    origin: Point,
)

/// Render a selection highlight beneath a paragraph.
///
/// Draws filled rectangles for each Rect in `rects` (as returned by
/// ParagraphLayout::selection_rects) using `highlight_color` as fill.
/// Call this before render_paragraph so that glyphs draw on top of the
/// highlight.
///
/// `origin` is the same paragraph origin passed to render_paragraph.
pub fn render_selection(
    rects: &[Rect],
    ctx: &mut DrawContext,
    origin: Point,
    highlight_color: Color,
)
```

Callers that need custom rendering — per-run effects, GPU instancing, PDF output — can iterate `layout.lines` and `line.runs` directly and submit glyph runs to their own pipeline without using these convenience functions.

---

## 11. Grapheme Cluster Navigation

A text editor or input method must move the cursor by grapheme clusters, not by bytes or Unicode scalar values. The string "é" may be encoded as a single precomposed codepoint (U+00E9, 2 bytes) or as a base character and combining acute accent (U+0065 U+0301, 3 bytes): either way it is one grapheme cluster and the cursor should jump over it as a unit. The Devanagari syllable क़ is three Unicode scalar values (base + nukta + combining vowel) but one grapheme cluster. `GraphemeCursor` implements the Unicode Text Segmentation algorithm (UAX #29) for grapheme cluster boundaries.

```ferrum
/// Grapheme cluster boundary cursor.
///
/// Implements UAX #29 grapheme cluster boundaries. Operates on a UTF-8
/// string slice. All byte offsets refer to positions in that string.
pub struct GraphemeCursor { ... }

impl GraphemeCursor {
    /// Construct a cursor for `text`.
    ///
    /// `text` is borrowed; the cursor is valid as long as `text` is
    /// unchanged. Creating a cursor is O(1); boundary queries are O(1)
    /// amortized.
    pub fn new(text: &str): GraphemeCursor

    /// Return the byte offset of the next grapheme cluster boundary after
    /// `byte_offset`.
    ///
    /// Returns None if `byte_offset` is at or past the end of the text.
    /// `byte_offset` need not itself be a boundary.
    pub fn next_boundary(self: &mut Self, byte_offset: usize): Option[usize]

    /// Return the byte offset of the previous grapheme cluster boundary
    /// before `byte_offset`.
    ///
    /// Returns None if `byte_offset` is at or before the start of the text.
    pub fn prev_boundary(self: &mut Self, byte_offset: usize): Option[usize]

    /// Return true if `byte_offset` is a grapheme cluster boundary.
    ///
    /// Byte offset 0 and text length are always boundaries.
    /// Returns false if byte_offset falls inside a UTF-8 multibyte sequence.
    pub fn is_boundary(self: &mut Self, byte_offset: usize): bool
}

/// Word boundary utilities.
///
/// Word boundaries follow UAX #29 word break rules.
pub struct WordBreaker { ... }

impl WordBreaker {
    /// Construct a new WordBreaker.
    pub fn new(): WordBreaker

    /// Return the byte offset of the start of the next word after
    /// `byte_offset`.
    ///
    /// "Word" means the start of the next sequence of letter or number
    /// characters, skipping spaces and punctuation. Returns text length if
    /// there is no next word.
    pub fn next_word(self: &Self, text: &str, byte_offset: usize): usize

    /// Return the byte offset of the start of the word at or before
    /// `byte_offset`.
    ///
    /// Returns 0 if there is no previous word.
    pub fn prev_word(self: &Self, text: &str, byte_offset: usize): usize

    /// Return the byte range [start, end) of the word that contains
    /// `byte_offset`.
    ///
    /// Returns an empty range (byte_offset..byte_offset) if the offset
    /// falls on a non-word character (space, punctuation).
    pub fn word_at(self: &Self, text: &str, byte_offset: usize): Range[usize]
}
```

---

## 12. Error Types

```ferrum
/// Errors produced by text layout operations.
@derive(Debug, Clone)]
pub enum TextLayoutError {
    /// No font face matching the requested family name was found in the
    /// font collection, even after fallback substitution.
    FontNotFound {
        family: String,
    },

    /// HarfBuzz shaping failed for a run of text.
    ///
    /// The String contains a diagnostic message. Shaping failure is rare
    /// and usually indicates an invalid or malformed font file.
    ShapingFailed(String),

    /// A byte offset is not on a UTF-8 character boundary, or is beyond
    /// the length of the text.
    InvalidByteOffset {
        /// The invalid offset.
        offset: usize,
        /// Total byte length of the text at the time of the call.
        text_len: usize,
    },

    /// A style range has start > end, or end > text length.
    InvalidRange {
        start: usize,
        end: usize,
        text_len: usize,
    },

    /// HarfBuzz or the line breaker ran out of memory.
    OutOfMemory,
}
```

---

## 13. Example Usage

The following example shapes a multilingual paragraph (English and Arabic), wraps it at 400 logical units, performs hit testing for cursor placement, renders a selection highlight, and draws the paragraph.

```ferrum
use extlib.ccsp.draw.{ DrawContext, Color, Point, Rect }
use extlib.ccsp.text_layout.{
    AttributedString, TextStyle, FontWeight, ParagraphStyle,
    TextAlignment, TextOverflow, LineHeight, Shaper, Paragraph,
    GraphemeCursor, render_paragraph, render_selection,
}
use extlib.font.{ FontCollection }
use alloc.sync.Arc

pub fn render_multilingual_example(
    ctx: &mut DrawContext,
    font_collection: Arc[FontCollection],
): Result[(), TextLayoutError] {
    // 1. Build the attributed string with two languages.
    //    The paragraph starts in English, switches to Arabic, then back.
    let text = "Hello, "مرحبا" world!"
    let mut astr = AttributedString::new(text)

    // English style: 16pt Noto Sans Regular, dark text.
    let en_style = TextStyle::new("Noto Sans", 16.0)
        .with_color(Color::from_srgb8(30, 30, 30, 255))

    // Arabic style: same family and size, same color.
    // HarfBuzz selects Arabic-script shaping rules based on codepoints.
    let ar_style = en_style.clone()

    // Bold "world" to demonstrate per-range weight.
    let bold_style = en_style.clone().with_weight(FontWeight::Bold)

    // Byte offsets must be on UTF-8 character boundaries.
    // "Hello, " = 7 bytes (ASCII).
    // "مرحبا" = 10 bytes (each Arabic letter is 2 bytes in UTF-8).
    // " world!" = 7 bytes.
    astr.add_style(0..7, en_style.clone())?
    astr.add_style(7..17, ar_style)?
    astr.add_style(18..24, bold_style)?  // "world!"
    astr.set_base_style(en_style)
    astr.set_language("en")
    astr.set_language_range(7..17, "ar")?

    // 2. Shape the attributed string.
    let mut shaper = Shaper::new(font_collection)?
    let shaped = shaper.shape(&astr)?

    // 3. Lay out the shaped text at 400px wide.
    let para_style = ParagraphStyle {
        alignment: TextAlignment::Start,
        line_height: LineHeight::Normal,
        max_lines: Option::None,
        overflow: TextOverflow::EllipsisWord,
        tab_size: 32.0,
    }
    let mut paragraph = Paragraph::new()
    let layout = paragraph.layout_styled(&astr, 400.0, &para_style, &mut shaper)?

    // 4. Hit test at a display coordinate (simulating a mouse click at x=120, y=8).
    let click = Point { x: 120.0, y: 8.0 }
    let hit = layout.hit_test(click)
    // hit.byte_offset is the byte position nearest the click.
    // Use GraphemeCursor to snap to the nearest grapheme cluster boundary
    // for safe cursor placement.
    let mut gcursor = GraphemeCursor::new(layout.text.as_str())
    let cursor_offset = if gcursor.is_boundary(hit.byte_offset) {
        hit.byte_offset
    } else {
        gcursor.next_boundary(hit.byte_offset).unwrap_or(layout.text.len())
    }
    let caret = layout.position_of(cursor_offset)?
    // caret.x, caret.line_y, caret.height describe the caret rectangle.

    // 5. Compute selection rectangles for the Arabic fragment.
    let sel_rects = layout.selection_rects(7, 17)?

    // 6. Render: selection highlight first, then text on top.
    let origin = Point { x: 20.0, y: 40.0 }
    let highlight = Color::from_srgb8(173, 214, 255, 180)  // translucent blue
    render_selection(&sel_rects, ctx, origin, highlight)
    render_paragraph(&layout, ctx, origin)

    // 7. Draw the caret.
    let caret_rect = Rect {
        origin: Point {
            x: origin.x + caret.x,
            y: origin.y + caret.line_y,
        },
        size: extlib.ccsp.draw.Size { width: 1.5, height: caret.height },
    }
    ctx.fill_rect(caret_rect, &extlib.ccsp.draw.Paint::Solid(Color::BLACK))

    Result::Ok(())
}
```

---

## 14. Dependencies

### Ferrum Extended Library

| Module | Types used |
|---|---|
| `extlib.ccsp.draw` | `DrawContext`, `Color`, `Point`, `Rect`, `Paint` |
| `extlib.font` | `FontCollection`, `FontFace`, `GlyphId` |

### Ferrum Standard Library

| Module | Purpose |
|---|---|
| `alloc` | `String`, `Vec`, `Arc` for owned data and shared font state |
| `alloc.sync` | `Arc[FontCollection]`, `Arc[FontFace]` — shared across the shaper |
| `core.ops` | `Range[usize]` for style and selection byte ranges |

### External C Libraries

| Library | Version | Purpose |
|---|---|---|
| HarfBuzz | ≥ 8.0 | Text shaping: GSUB/GPOS, complex script rules, bidi embedding |

HarfBuzz is linked via Ferrum's FFI layer. All calls to HarfBuzz are marked `! Unsafe` internally and are not visible in the public API. HarfBuzz is the only external dependency: Unicode bidi analysis (UAX #9) and line breaking (UAX #14) are provided by HarfBuzz's `hb-unicode` and by a bundled UAX #14 implementation that requires no additional library.

### Optional Runtime Dependencies

| Library | Condition | Purpose |
|---|---|---|
| fontconfig | Linux, if `extlib.font` uses system fonts | Font discovery |

These are resolved at link time by `extlib.font`; `extlib.ccsp.text_layout` does not depend on them directly.
