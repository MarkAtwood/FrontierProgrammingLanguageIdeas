# Ferrum Extended Library — font: Font Rendering

**Module path:** `extlib.font`
**Layer:** Extended standard library (not stdlib; requires optional dependency)
**Part of:** [Ferrum Standard Library](ferrum-stdlib.md)
**Companion:** [extlib.draw](ferrum_extlib_draw.md) (for `Rect`, `Color` types)

---

## 1. Overview and Rationale

### Why a Font Extlib

Font rendering is a complex, self-contained subsystem. Correct handling of OpenType, TrueType, PostScript Type 1, and bitmap fonts requires glyph outline decomposition, scan-line rasterization, hint bytecode interpretation, subpixel LCD filtering, kerning table lookup, and — for variable fonts — interpolation across continuous design-space axes. This is a substantial body of code with a substantial dependency (FreeType 2, a C library). Most Ferrum programs — firmware, protocol daemons, database engines, CLI tools — never render text glyphs. Placing this module in the standard library would impose the FreeType dependency and all its surface area on every binary regardless of need.

Programs that do render text opt in explicitly:

```toml
[dependencies]
extlib.font = { version = "1.0" }
```

### Why FreeType

FreeType 2 is the industry-standard font rasterizer. It is correct, battle-hardened, handles every practically relevant font format (OpenType/TrueType, PostScript Type 1, CFF, WOFF, bitmap strikes, color emoji), and is the engine behind the text you are reading on Linux, Android, and most embedded displays. It implements the OpenType Font Variations specification, which is the standard for variable fonts. No alternative open-source rasterizer covers this breadth with comparable correctness and platform deployment history.

`extlib.font` wraps FreeType via Ferrum's FFI layer (`! Unsafe` internally). All unsafe operations are confined to the implementation; the public API is entirely safe.

### Abstraction Rationale

The public API exposes no FreeType types, no FreeType error codes (they are translated to `FontError`), and no FreeType memory model. A caller using `extlib.font` does not know FreeType exists. This means the underlying rasterizer is swappable — a future implementation could substitute HarfBuzz's rasterizer, a GPU-side signed-distance-field renderer, or a platform-native API — without changing any call site. The abstraction is not indirection for its own sake: it is the guarantee that font rendering is a module boundary, not a FreeType binding.

### What This Module Does Not Cover

Text shaping (Unicode bidirectional algorithm, ligature substitution, mark positioning, complex script layout) is a separate concern handled by a shaping engine such as HarfBuzz. `extlib.font` provides glyph rasterization and metrics. It provides `FontCollection::script_for_char` for Unicode script detection as a convenience for shaping integration, but does not perform shaping itself. A full text layout stack would layer a shaping extlib on top of this one.

---

## 2. FontLibrary — Global Context

`FontLibrary` holds the FreeType library instance. FreeType requires global initialization before any font operations. A program typically creates one `FontLibrary` at startup, keeps it alive for the duration of text rendering, and drops it at shutdown.

```ferrum
/// The FreeType library context.
///
/// Initializes FreeType on construction. Holds shared state required by all
/// FontFace and ScaledFont instances created from it.
///
/// Send + Sync: may be shared across threads. FreeType operations are
/// serialized internally where required.
pub type FontLibrary { ... }

impl FontLibrary {
    /// Initialize the FreeType library.
    ///
    /// Returns Err on FreeType initialization failure (out of memory, missing
    /// shared library). In practice this succeeds on any correctly installed
    /// system.
    pub fn new(): Result[FontLibrary, FontError]

    /// Load a font face from a file on disk.
    ///
    /// For .ttc (TrueType Collection) files containing multiple faces, loads
    /// face index 0. Use load_font_file_index to select a specific face.
    pub fn load_font_file(self: &Self, path: &Path): Result[FontFace, FontError] ! IO

    /// Load a font face from a file, selecting a specific face index.
    ///
    /// Use for .ttc collections. face_index 0 is the first (or only) face.
    pub fn load_font_file_index(
        self: &Self,
        path: &Path,
        face_index: u32,
    ): Result[FontFace, FontError] ! IO

    /// Load a font face from in-memory bytes.
    ///
    /// The Arc keeps the data alive for the lifetime of the FontFace.
    /// FreeType reads directly from this buffer; no copy is made.
    pub fn load_font_bytes(
        self: &Self,
        data: Arc[Vec[u8]],
    ): Result[FontFace, FontError]

    /// Load a specific face from an in-memory font collection.
    pub fn load_font_bytes_index(
        self: &Self,
        data: Arc[Vec[u8]],
        face_index: u32,
    ): Result[FontFace, FontError]

    /// Enumerate fonts available on this system.
    ///
    /// On Linux: queries fontconfig.
    /// On macOS: queries CoreText.
    /// On Windows: queries DirectWrite.
    ///
    /// Returns FontFaceDesc records describing discovered fonts. The fonts
    /// are not loaded; call load_font_file to open one.
    pub fn system_fonts(self: &Self): Result[Vec[FontFaceDesc], FontError] ! IO

    /// Find system fonts matching a query.
    ///
    /// Returns matching FontFaceDesc records sorted by quality of match,
    /// best first. The list may be empty if no font matches.
    pub fn match_font(
        self: &Self,
        query: FontQuery,
    ): Result[Vec[FontFaceDesc], FontError] ! IO
}
```

`FontLibrary` is `Send + Sync`. Font operations on separate `FontFace` and `ScaledFont` instances may proceed concurrently. The library serializes operations that FreeType requires to be single-threaded; callers do not need to coordinate.

---

## 3. FontFace — Loaded Typeface

`FontFace` represents a single loaded font face: one file (or one index within a collection file), fully parsed and ready for metric queries and rasterization setup. A `FontFace` is not yet at a specific size; it is the face in its native design-space units.

```ferrum
/// A fully-parsed font face, not yet sized.
///
/// Wraps a FreeType FT_Face. Lifetime is tied to the FontLibrary that
/// created it. Clone is O(1): the underlying face is reference-counted.
pub type FontFace { ... }

impl FontFace {
    /// The font family name as declared in the name table.
    /// e.g. "Inter", "Noto Sans", "Source Code Pro"
    pub fn family_name(self: &Self): &str

    /// The style name as declared in the name table.
    /// e.g. "Regular", "Bold", "Bold Italic", "Light Condensed"
    pub fn style_name(self: &Self): &str

    /// Units per em. Typically 1000 (PostScript/CFF) or 2048 (TrueType).
    /// Design-space metrics (glyph outlines, metrics tables) are in these units.
    pub fn units_per_em(self: &Self): u32

    /// True if the font contains scalable outlines (TrueType, CFF, Type 1).
    pub fn is_scalable(self: &Self): bool

    /// True if the font contains embedded bitmap strikes at fixed sizes.
    pub fn is_fixed_sizes(self: &Self): bool

    /// True if the font contains color glyph data (COLR, CBDT/CBLC, sbix).
    /// Emoji fonts and color icon fonts set this.
    pub fn has_color_glyphs(self: &Self): bool

    /// Number of glyphs in the font.
    pub fn glyph_count(self: &Self): u32

    /// Map a Unicode code point to a glyph ID.
    ///
    /// Returns None if the font has no glyph for this character.
    /// The GlyphId 0 (.notdef) is never returned here; None is returned instead.
    pub fn char_to_glyph_id(self: &Self, ch: char): Option[GlyphId]

    /// Look up a glyph by PostScript name.
    ///
    /// Returns None if no glyph has this name. Useful for accessing named
    /// glyphs such as ".notdef", "space", "uni0041", or ligature names.
    pub fn glyph_id_for_name(self: &Self, name: &str): Option[GlyphId]

    /// The variation axes defined in this font's fvar table.
    ///
    /// Returns an empty Vec for non-variable fonts.
    pub fn variable_axes(self: &Self): Vec[VariationAxis]

    /// Named instances defined in the font's fvar/STAT tables.
    ///
    /// Named instances are predefined axis coordinate sets such as
    /// "Light", "Regular", "Bold", "Black" in a variable font.
    /// Returns an empty Vec for non-variable fonts.
    pub fn named_instances(self: &Self): Vec[NamedInstance]

    /// Create a ScaledFont at a specific pixel size.
    ///
    /// size_px is the desired em square height in pixels at 1:1 display scale.
    /// For HiDPI displays, multiply by the device pixel ratio before calling.
    pub fn scale(self: &Self, size_px: f32): ScaledFont

    /// Create a ScaledFont at a specific pixel size with variation axis values.
    ///
    /// axes need not cover all axes; axes not listed use their default value.
    /// For non-variable fonts, the axes slice is ignored.
    pub fn scale_with_axes(
        self: &Self,
        size_px: f32,
        axes: &[VariationAxisValue],
    ): ScaledFont
}
```

### GlyphId

```ferrum
/// An opaque glyph index within a specific FontFace.
///
/// GlyphId values are only meaningful within the face that produced them.
/// Do not mix GlyphIds across faces.
@derive(Copy, Clone, Eq, Hash, Debug)
pub type GlyphId(u32)

impl GlyphId {
    /// The .notdef glyph, shown when no other glyph is available.
    pub const NOTDEF: GlyphId = GlyphId(0)
}
```

---

## 4. ScaledFont — FontFace at a Specific Size

`ScaledFont` fixes a `FontFace` at a specific pixel size and optional variation axis coordinates. All metric values from a `ScaledFont` are in pixels at that size.

```ferrum
/// A font face sized to specific pixel dimensions.
///
/// All metrics are in pixels. Subpixel values are in fractional pixels
/// (f32) to support subpixel positioning.
///
/// ScaledFont is lightweight. Creating multiple ScaledFonts from one
/// FontFace at different sizes is inexpensive.
pub type ScaledFont { ... }

impl ScaledFont {
    /// Line-level metrics for this face at this size.
    pub fn metrics(self: &Self): FontMetrics

    /// Per-glyph advance and bounding-box metrics.
    ///
    /// Returns Err(GlyphNotFound) if id is out of range.
    pub fn glyph_metrics(self: &Self, id: GlyphId): Result[GlyphMetrics, FontError]

    /// Kerning advance between a left and right glyph pair, in pixels.
    ///
    /// Prefers GPOS kerning (OpenType); falls back to kern table (legacy TrueType).
    /// Returns 0.0 if the pair has no kerning entry or if the font has no kern data.
    ///
    /// The returned value is additive: add it to the advance_x of the left glyph
    /// when positioning the right glyph.
    pub fn kerning_pair(self: &Self, left: GlyphId, right: GlyphId): f32

    /// Rasterize a glyph to a pixel bitmap.
    ///
    /// hint controls grid-fitting. See HintMode for trade-offs.
    /// Returns Err(GlyphNotFound) if id is not in the face.
    pub fn rasterize(
        self: &Self,
        id: GlyphId,
        hint: HintMode,
    ): Result[GlyphBitmap, FontError]

    /// Set the LCD subpixel filter applied during subpixel rasterization.
    ///
    /// Only affects rasterize() calls that produce SubpixelRgb or SubpixelBgr
    /// output. The default is LcdFilter.Default, which matches FreeType's
    /// recommended filter for most LCD panels.
    pub fn set_lcd_filter(self: &mut Self, filter: LcdFilter)
}
```

### FontMetrics

```ferrum
/// Line-level typographic metrics for a ScaledFont.
///
/// All values are in pixels at the ScaledFont's size.
/// Values follow typographic sign conventions: ascender is positive
/// (above baseline), descender is negative (below baseline).
@derive(Clone, Debug)
pub type FontMetrics {
    /// Distance from baseline to top of tallest glyph (positive).
    pub ascender: f32,

    /// Distance from baseline to bottom of lowest glyph (negative).
    pub descender: f32,

    /// Recommended line spacing (ascender - descender + line gap).
    /// Use this as the vertical advance between lines of text.
    pub height: f32,

    /// Recommended underline position (typically negative; below baseline).
    pub underline_position: f32,

    /// Recommended underline stroke thickness.
    pub underline_thickness: f32,

    /// Recommended strikethrough position (typically near x-height midpoint).
    pub strikethrough_position: f32,

    /// Maximum advance width across all glyphs in the face.
    pub max_advance_width: f32,
}
```

### GlyphMetrics

```ferrum
/// Per-glyph advance and bounding-box metrics.
///
/// All values are in pixels at the ScaledFont's size.
@derive(Clone, Debug)
pub type GlyphMetrics {
    /// Horizontal advance: cursor moves this far right after this glyph.
    pub advance_x: f32,

    /// Vertical advance: for top-to-bottom scripts, cursor moves this far down.
    /// Zero for horizontal-only fonts.
    pub advance_y: f32,

    /// Horizontal bearing: pixel offset from cursor to left edge of glyph bitmap.
    pub bearing_x: f32,

    /// Vertical bearing: pixel offset from baseline to top edge of glyph bitmap.
    pub bearing_y: f32,

    /// Bitmap width in pixels.
    pub width: f32,

    /// Bitmap height in pixels.
    pub height: f32,
}
```

### HintMode

```ferrum
/// Glyph hinting mode for rasterization.
///
/// Hinting adjusts glyph outlines to align with the pixel grid, improving
/// sharpness at small sizes. The trade-off is between sharpness (Full) and
/// faithful outline reproduction (None).
@derive(Copy, Clone, Eq, Debug)
pub enum HintMode {
    /// No hinting. Outlines are rendered exactly as defined.
    ///
    /// Best for large sizes (> 40px), printing, and rendering pipelines
    /// that apply their own grid-fitting (e.g. distance-field rendering).
    None,

    /// FreeType light hinting. Adjusts vertical stems only.
    ///
    /// Preserves horizontal proportions and spacing while improving
    /// vertical sharpness. Recommended for most screen rendering on
    /// modern high-density displays.
    Light,

    /// Full grid-fitting. Adjusts both horizontal and vertical stems.
    ///
    /// Produces the sharpest result at small sizes on low-density displays
    /// (96 dpi). May alter glyph proportions to fit the grid.
    Full,

    /// FreeType auto-hinter. Applies hinting to fonts with no hint bytecode.
    ///
    /// Equivalent to Light for fonts that have TrueType bytecode hints;
    /// applies FreeType's own hinting algorithm for fonts without hints.
    AutoHint,
}
```

---

## 5. Glyph Rasterization

### GlyphBitmap

```ferrum
/// A rasterized glyph bitmap.
///
/// bearing_x and bearing_y give the pixel offset from the cursor position
/// (baseline, left edge of advance) to the top-left corner of this bitmap.
/// These match the FreeType slot->bitmap_left / slot->bitmap_top convention.
@derive(Debug)
pub type GlyphBitmap {
    /// Horizontal offset from cursor to left edge of bitmap (signed pixels).
    pub bearing_x: i32,

    /// Vertical offset from baseline to top edge of bitmap (signed pixels, positive up).
    pub bearing_y: i32,

    /// Bitmap width in pixels.
    pub width: u32,

    /// Bitmap height in pixels.
    pub height: u32,

    /// Pixel format of the data buffer.
    pub format: GlyphBitmapFormat,

    /// Raw pixel data. Length is format-dependent:
    ///   Grayscale:    width * height bytes (one byte per pixel, alpha mask)
    ///   SubpixelRgb:  width * height * 3 bytes (R, G, B per pixel)
    ///   SubpixelBgr:  width * height * 3 bytes (B, G, R per pixel)
    ///   Color:        width * height * 4 bytes (R, G, B, A per pixel, premultiplied)
    ///   Bitmap1bpp:   ceil(width / 8) * height bytes (1 bit per pixel, MSB first)
    pub data: Vec[u8],
}
```

### GlyphBitmapFormat

```ferrum
/// Pixel format of a GlyphBitmap's data buffer.
@derive(Copy, Clone, Eq, Debug)]
pub enum GlyphBitmapFormat {
    /// 8-bit grayscale alpha mask. One byte per pixel.
    ///
    /// The canonical format for scalable font rendering. Composite onto
    /// a background color using the glyph byte as the alpha channel.
    /// FreeType FT_PIXEL_MODE_GRAY.
    Grayscale,

    /// RGB subpixel rendering, one byte per sub-pixel (R, G, B order).
    ///
    /// For LCD panels with RGB stripe layout. Bytes per pixel: 3.
    /// Requires per-channel compositing with gamma-corrected blending.
    /// FreeType FT_PIXEL_MODE_LCD.
    SubpixelRgb,

    /// BGR subpixel rendering, one byte per sub-pixel (B, G, R order).
    ///
    /// For LCD panels with BGR stripe layout (some older panels and
    /// vertical-stripe configurations). Bytes per pixel: 3.
    SubpixelBgr,

    /// RGBA color bitmap, 4 bytes per pixel (R, G, B, A), premultiplied alpha.
    ///
    /// Produced by color emoji fonts (COLR v0/v1, CBDT/CBLC, sbix).
    /// Composite using premultiplied alpha blending.
    Color,

    /// 1-bit monochrome bitmap, MSB first, padded to byte boundaries per row.
    ///
    /// Produced for bitmap-only fonts and when hinting produces exactly
    /// on-pixel outlines. Row stride is ceil(width / 8) bytes.
    Bitmap1bpp,
}
```

### LcdFilter

```ferrum
/// Subpixel LCD filter applied during SubpixelRgb/SubpixelBgr rasterization.
///
/// LCD rendering decomposes each pixel into R, G, B sub-pixel contributions.
/// Without filtering, this causes color fringing. The filter blurs the
/// sub-pixel weights horizontally to reduce fringing at the cost of slight
/// sharpness reduction.
@derive(Copy, Clone, Eq, Debug)
pub enum LcdFilter {
    /// No filtering. Maximum sharpness, visible color fringing.
    None,

    /// Light filter. Weights: [1/4, 1/2, 1/4]. Minimal fringing reduction.
    Light,

    /// FreeType default filter. Weights: [3/16, 4/16, 5/16, 4/16, 3/16] (5-tap).
    ///
    /// Recommended for most LCD panels. Matches the filter used by most
    /// Linux desktop environments.
    Default,

    /// Legacy filter (pre-2.4.0 FreeType behavior). Weights: [1/4, 2/4, 1/4].
    /// Included for compatibility with rendering pipelines that were tuned to
    /// the old output. Prefer Default for new code.
    Legacy,
}
```

---

## 6. Variable Fonts (OpenType Font Variations)

Variable fonts (OpenType Font Variations, ISO/IEC 14496-22) encode a continuous design space across one or more named axes. A single variable font file can represent the full range from Thin to Black weight, from Condensed to Extended width, and from upright to italic, without requiring separate files for each style. The font stores master outlines and delta tables; FreeType interpolates at any axis coordinate within the defined range.

`extlib.font` exposes variation axes through `FontFace::variable_axes()` and applies axis coordinates via `FontFace::scale_with_axes()`. The rest of the API is identical to non-variable font use; rasterization, metrics, and kerning all reflect the current axis state.

### VariationAxis

```ferrum
/// A single variation axis defined in a variable font's fvar table.
@derive(Clone, Debug)]
pub type VariationAxis {
    /// The four-byte OpenType axis tag.
    ///
    /// Well-known tags (as ASCII bytes):
    ///   b"wght"  Weight:       100 (Thin) to 900 (Black), default usually 400
    ///   b"wdth"  Width:        75 (Condensed) to 125 (Extended), default 100
    ///   b"ital"  Italic:       0 (upright) to 1 (italic)
    ///   b"slnt"  Slant:        negative values lean right (italic direction)
    ///   b"opsz"  Optical size: display size in points for optical-size tuning
    pub tag: [u8; 4],

    /// Human-readable axis name from the font's name table.
    pub name: String,

    /// Minimum allowed coordinate value.
    pub minimum: f32,

    /// Default coordinate value (used when this axis is not specified).
    pub default: f32,

    /// Maximum allowed coordinate value.
    pub maximum: f32,
}
```

### VariationAxisValue

```ferrum
/// A specific coordinate on one variation axis.
@derive(Copy, Clone, Debug)
pub type VariationAxisValue {
    /// The four-byte axis tag this value applies to.
    pub tag: [u8; 4],

    /// The axis coordinate. Must be within [minimum, maximum] for the axis.
    /// Values outside the range are clamped by FreeType.
    pub value: f32,
}
```

### NamedInstance

```ferrum
/// A predefined named variation instance from the font's fvar table.
///
/// Named instances correspond to the discrete named styles a variable font
/// advertises: "Thin", "Light", "Regular", "Medium", "Bold", "Black",
/// and any combination the designer chose to name.
///
/// Using a named instance is equivalent to calling scale_with_axes with
/// the instance's coordinates.
@derive(Clone, Debug)
pub type NamedInstance {
    /// The name of this instance, from the font's name table.
    pub name: String,

    /// The axis coordinates that define this instance.
    /// Covers all axes the designer chose to specify for this instance.
    pub coordinates: Vec[VariationAxisValue],
}
```

### Well-Known Axis Tags

```ferrum
/// Well-known OpenType variation axis tags.
pub mod axis_tag {
    /// Weight axis. Range typically 100–900. Default 400 (Regular).
    pub const WEIGHT:        [u8; 4] = *b"wght"
    /// Width axis. Range typically 75–125. Default 100 (Normal).
    pub const WIDTH:         [u8; 4] = *b"wdth"
    /// Italic axis. Range 0–1. 0 = upright, 1 = italic.
    pub const ITALIC:        [u8; 4] = *b"ital"
    /// Slant axis. Negative values lean right. 0 = upright.
    pub const SLANT:         [u8; 4] = *b"slnt"
    /// Optical size axis. Value is the intended display size in points.
    pub const OPTICAL_SIZE:  [u8; 4] = *b"opsz"
}
```

---

## 7. Glyph Atlas

The glyph atlas packs rasterized glyphs into a single large texture for efficient GPU upload. Instead of uploading one texture per glyph (thousands of draw calls), the renderer uploads one atlas texture and references sub-regions by UV coordinates.

```ferrum
/// A packed texture atlas of rasterized glyphs.
///
/// Packs GlyphBitmaps onto a fixed-size canvas using shelf packing.
/// Tracks which glyphs are present and evicts least-recently-used
/// entries when capacity is exceeded.
///
/// Not Send: intended for single-threaded use on the render thread.
/// Create one atlas per render thread.
pub type GlyphAtlas { ... }

impl GlyphAtlas {
    /// Create a new empty atlas with the given pixel dimensions.
    ///
    /// width and height should be powers of two for GPU compatibility
    /// (e.g. 1024, 2048, 4096). The atlas allocates width * height bytes
    /// for grayscale, or width * height * 4 bytes for color (RGBA) storage,
    /// depending on the format of glyphs inserted.
    pub fn new(width: u32, height: u32): GlyphAtlas

    /// Set the maximum memory budget for cached glyph data.
    ///
    /// When the total rasterized glyph data exceeds this budget, the atlas
    /// evicts least-recently-used entries to make room. The atlas texture
    /// is not resized; evicted regions are available for re-use.
    ///
    /// Default: no limit (eviction occurs only when the texture area is full).
    pub fn set_max_bytes(self: &mut Self, n: u64)

    /// Look up a glyph in the atlas, rasterizing and packing it if absent.
    ///
    /// On hit: returns the cached AtlasEntry without rasterizing.
    /// On miss: calls font.rasterize(glyph_id, hint), packs the resulting
    ///          bitmap into the atlas, updates the dirty flag, and returns
    ///          the new AtlasEntry.
    ///
    /// Returns Err(AtlasFull) if the glyph cannot be packed and no eviction
    /// candidate is available. The caller should either grow the atlas or
    /// flush and clear it.
    pub fn get_or_rasterize(
        self: &mut Self,
        font: &ScaledFont,
        glyph_id: GlyphId,
        hint: HintMode,
    ): Result[AtlasEntry, FontError]

    /// Raw texture data for GPU upload.
    ///
    /// Layout: top-left origin, row-major, no padding between rows.
    /// Format: R8 (grayscale atlas) or R8G8B8A8 (color atlas).
    ///
    /// Upload to a GPU texture whenever is_dirty() returns true, then call
    /// clear_dirty().
    pub fn texture_data(self: &Self): &[u8]

    /// True if new glyphs have been packed since the last clear_dirty() call.
    ///
    /// Check this before each frame. If true, re-upload texture_data() to
    /// the GPU texture before drawing.
    pub fn is_dirty(self: &Self): bool

    /// Clear the dirty flag after the texture has been uploaded to the GPU.
    pub fn clear_dirty(self: &mut Self)

    /// Remove all cached glyphs and reset the atlas to empty.
    ///
    /// Call when switching fonts, changing size, or when the texture is
    /// no longer valid (e.g. GPU context loss).
    pub fn clear(self: &mut Self)

    /// Width of the atlas texture in pixels.
    pub fn width(self: &Self): u32

    /// Height of the atlas texture in pixels.
    pub fn height(self: &Self): u32
}
```

### AtlasEntry

```ferrum
/// Location of a rasterized glyph within the atlas texture.
///
/// uv_rect gives the glyph's position in UV space ([0.0, 1.0] in both axes).
/// bearing_x and bearing_y give the pen-relative offset for positioning the
/// quad: same convention as GlyphBitmap bearings.
@derive(Copy, Clone, Debug)
pub type AtlasEntry {
    /// UV coordinates of this glyph's sub-region in the atlas texture.
    ///
    /// uv_rect.x, uv_rect.y: top-left corner (UV origin at top-left).
    /// uv_rect.width, uv_rect.height: extent in UV space.
    /// Multiply by atlas width/height to get pixel coordinates.
    pub uv_rect: Rect,

    /// Horizontal bearing: offset from cursor to left edge of glyph (pixels).
    pub bearing_x: i32,

    /// Vertical bearing: offset from baseline to top edge of glyph (pixels, positive up).
    pub bearing_y: i32,

    /// Glyph bitmap width in pixels (for sizing the rendered quad).
    pub width: u32,

    /// Glyph bitmap height in pixels (for sizing the rendered quad).
    pub height: u32,
}
```

### Shelf Packing

The atlas uses shelf packing: glyphs are placed left-to-right on horizontal shelves. When a glyph is too tall for the current shelf, a new shelf begins at the bottom of the previous one. The shelf height is set to the height of the first glyph placed on it. This algorithm is simple, cache-friendly, and produces low fragmentation for the roughly uniform glyph heights produced by a single font at a single size.

For workloads mixing many font sizes or many faces, the caller may maintain one atlas per size class (e.g. small/medium/large) to reduce wasted shelf space.

---

## 8. Font Collection (Font Fallback)

Real text rendering requires multiple faces: a primary Latin face, an emoji face, a CJK fallback, a symbol face. The `FontCollection` type handles character-level fallback: given a character, it finds the first face in priority order that covers it.

```ferrum
/// An ordered collection of font faces providing character coverage fallback.
///
/// When looking up a glyph, FontCollection iterates faces in priority order
/// (highest priority first) and returns the first face that has a glyph
/// for the requested character.
///
/// Typical setup: primary typeface at priority 100, emoji font at priority 50,
/// CJK fallback at priority 30, last-resort font at priority 0.
pub type FontCollection { ... }

impl FontCollection {
    /// Create an empty font collection.
    pub fn new(): FontCollection

    /// Add a font face to the collection.
    ///
    /// Higher priority values are tried before lower ones.
    /// Faces with equal priority are tried in insertion order.
    pub fn add_face(self: &mut Self, face: FontFace, priority: u32)

    /// Remove all faces from the collection.
    pub fn clear(self: &mut Self)

    /// Find the highest-priority face that has a glyph for ch.
    ///
    /// Returns (FontFace, GlyphId) for the first face that maps ch,
    /// or None if no face in the collection covers the character.
    pub fn char_to_glyph(self: &Self, ch: char): Option[(FontFace, GlyphId)]

    /// Detect the Unicode script of a character.
    ///
    /// Returns the primary Unicode script for ch (Latin, Han, Arabic,
    /// Devanagari, etc.). Useful for selecting a shaping engine or
    /// for per-script face selection before calling char_to_glyph.
    ///
    /// Characters with multiple possible scripts (common and inherited
    /// characters such as punctuation and digits) return Script.Common.
    pub fn script_for_char(ch: char): Script

    /// Iterate over all registered faces in priority order.
    pub fn faces(self: &Self): impl Iterator[Item=&FontFace]
}
```

### Script

```ferrum
/// Unicode script identifier for a character.
///
/// Values correspond to Unicode script property values (UAX #24).
/// This list covers the most commonly encountered scripts. The full
/// Unicode script list is available in the extlib.unicode module.
@derive(Copy, Clone, Eq, Hash, Debug)
pub enum Script {
    Common,       // Digits, punctuation, symbols shared across scripts
    Inherited,    // Characters that inherit the script of surrounding text
    Latin,
    Greek,
    Cyrillic,
    Arabic,
    Hebrew,
    Devanagari,
    Bengali,
    Han,          // CJK unified ideographs
    Hiragana,
    Katakana,
    Hangul,
    Thai,
    Tibetan,
    // ... additional scripts accessible via extlib.unicode
}
```

---

## 9. System Font Discovery

Platform-specific font discovery is exposed through `FontLibrary::system_fonts()` and `FontLibrary::match_font()`, backed by the platform's native font registry. The Ferrum implementation uses platform-specific backends; callers see only the portable `FontFaceDesc` and `FontQuery` types.

### FontFaceDesc

```ferrum
/// Metadata describing a system font, before it is loaded.
///
/// Returned by system_fonts() and match_font(). To use the font,
/// call FontLibrary::load_font_file_index(path, face_index).
@derive(Clone, Debug)
pub type FontFaceDesc {
    /// Absolute path to the font file.
    pub path: PathBuf,

    /// Face index within the file (0 for single-face files).
    pub face_index: u32,

    /// Font family name (e.g. "Noto Sans", "Helvetica Neue").
    pub family: String,

    /// Style name (e.g. "Regular", "Bold Italic", "Condensed Light").
    pub style: String,

    /// Nominal weight class.
    pub weight: FontWeight,

    /// True if the face is italic or oblique.
    pub is_italic: bool,

    /// True if the face is a monospace (fixed-pitch) face.
    pub is_monospace: bool,
}
```

### FontWeight

```ferrum
/// CSS/OpenType font weight classification.
@derive(Copy, Clone, Eq, Ord, Debug)
pub enum FontWeight {
    Thin,         // 100
    ExtraLight,   // 200
    Light,        // 300
    Regular,      // 400
    Medium,       // 500
    SemiBold,     // 600
    Bold,         // 700
    ExtraBold,    // 800
    Black,        // 900
}

impl FontWeight {
    /// Numeric weight value (100–900).
    pub fn value(self: Self): u32
}
```

### FontQuery

```ferrum
/// Criteria for matching system fonts.
///
/// Fields set to None are unconstrained. The platform backend translates
/// these criteria into the native matching API (fontconfig pattern,
/// CoreText descriptor, DirectWrite font family query).
@derive(Clone, Debug)
pub type FontQuery {
    /// Preferred font family name, or None to match any family.
    pub family: Option[String],

    /// Required weight, or None to match any weight.
    pub weight: Option[FontWeight],

    /// If true, require an italic/oblique face.
    pub italic: bool,

    /// If true, require a monospace face.
    pub monospace: bool,
}

impl FontQuery {
    pub fn new(): Self

    pub fn family(self, name: &str): Self
    pub fn weight(self, w: FontWeight): Self
    pub fn italic(self, yes: bool): Self
    pub fn monospace(self, yes: bool): Self
}
```

### Platform Backends

The platform-specific discovery code is compiled conditionally. Callers do not invoke these directly; they use `FontLibrary::system_fonts()` and `FontLibrary::match_font()`.

**Linux — fontconfig:**
The backend issues `fc_match` and `fc_list` queries via the fontconfig C API. The `extlib.font` FFI binding declares the necessary fontconfig functions with `! Unsafe` internally. Fontconfig is an optional dependency; on systems without fontconfig, `system_fonts()` returns an empty list and `match_font()` returns `Err(FontError::UnsupportedPlatform)`.

**macOS — CoreText:**
The backend uses `CTFontDescriptorCreateMatchingFontDescriptors` to enumerate and match fonts. CoreText is always available on macOS.

**Windows — DirectWrite:**
The backend uses `IDWriteFactory::GetSystemFontCollection` and `IDWriteFontFamily` to enumerate fonts. DirectWrite is always available on Windows Vista and later.

---

## 10. Error Types

```ferrum
/// Errors produced by extlib.font operations.
@derive(Debug)]
pub enum FontError {
    /// FreeType returned an error. The i32 is the raw FT_Error code.
    ///
    /// Most callers do not need to inspect the code; it is included for
    /// diagnostics and logging. Check the FreeType documentation for
    /// FT_Error values.
    FreeTypeError(i32),

    /// The requested glyph ID does not exist in the face.
    GlyphNotFound(GlyphId),

    /// The glyph atlas has no space for a new glyph and no eviction candidate
    /// is available. The caller should grow the atlas or call clear().
    AtlasFull,

    /// An I/O error occurred reading a font file.
    Io(IoError),

    /// The font file is not a recognized font format or is corrupt.
    InvalidFontFile,

    /// The font format is recognized but not supported by this build.
    ///
    /// Example: a font requiring a FreeType module that was not compiled in.
    UnsupportedFormat,

    /// A variation axis value is outside the axis's allowed range.
    InvalidAxisValue {
        /// The axis tag for which the value was rejected.
        axis: [u8; 4],
        /// The value that was rejected.
        value: f32,
        /// The allowed range [minimum, maximum].
        range: (f32, f32),
    },

    /// The platform font discovery backend is not available.
    ///
    /// Returned by system_fonts() and match_font() on platforms where
    /// the backend was not compiled in (e.g. no fontconfig on Linux).
    UnsupportedPlatform,
}

impl Display for FontError { ... }
```

---

## 11. Example Usage

### Load a System Font and Rasterize "Hello"

```ferrum
use extlib.font.{FontLibrary, FontQuery, FontWeight, HintMode, GlyphAtlas}
use std.path.Path

fn render_hello() ! IO {
    // Initialize FreeType once.
    let lib = FontLibrary::new().unwrap()

    // Find a regular sans-serif font from the system.
    let descs = lib.match_font(
        FontQuery::new()
            .family("Inter")
            .weight(FontWeight.Regular)
    ).unwrap()

    let desc = descs.first().expect("no matching font found")
    let face = lib.load_font_file_index(&desc.path, desc.face_index).unwrap()

    // Scale to 24px for a 1:1 display (multiply by device pixel ratio for HiDPI).
    let font = face.scale(24.0)

    let metrics = font.metrics()
    // metrics.height gives the recommended line spacing in pixels.

    // Rasterize each character in "Hello".
    let mut atlas = GlyphAtlas::new(1024, 1024)
    let text = "Hello"
    let mut cursor_x: f32 = 0.0
    let mut prev_glyph: Option[GlyphId] = None

    for ch in text.chars() {
        let glyph_id = match face.char_to_glyph_id(ch) {
            Some(id) => id,
            None     => continue,
        }

        // Apply kerning from previous glyph.
        if let Some(prev) = prev_glyph {
            cursor_x += font.kerning_pair(prev, glyph_id)
        }

        let entry = atlas.get_or_rasterize(&font, glyph_id, HintMode.Light).unwrap()

        // entry.uv_rect: sub-region of the atlas texture to sample.
        // entry.bearing_x / bearing_y: pen-relative offset for quad placement.
        // Submit a textured quad at (cursor_x + entry.bearing_x, baseline - entry.bearing_y).

        let glyph_metrics = font.glyph_metrics(glyph_id).unwrap()
        cursor_x += glyph_metrics.advance_x
        prev_glyph = Some(glyph_id)
    }

    // Upload to GPU if new glyphs were packed.
    if atlas.is_dirty() {
        // gpu_upload_texture(atlas.texture_data(), atlas.width(), atlas.height())
        atlas.clear_dirty()
    }
}
```

### Variable Font Weight Animation

```ferrum
use extlib.font.{FontLibrary, VariationAxisValue, axis_tag, HintMode}

fn animate_weight(lib: &FontLibrary, face_path: &Path) ! IO {
    let face = lib.load_font_file(face_path).unwrap()

    // Inspect available axes.
    for axis in face.variable_axes() {
        // e.g. axis.tag == *b"wght", axis.minimum == 100.0, axis.maximum == 900.0
    }

    // Animate weight from 100 to 900 across 60 frames.
    for frame in 0..60_u32 {
        let weight = 100.0 + (800.0 * (frame as f32) / 59.0)

        let font = face.scale_with_axes(32.0, &[
            VariationAxisValue { tag: axis_tag.WEIGHT, value: weight },
        ])

        // Rasterize glyphs with this weight variant.
        // font.rasterize(glyph_id, HintMode.Light) ...
    }
}
```

### Latin + Emoji Fallback Collection

```ferrum
use extlib.font.{FontLibrary, FontCollection, FontQuery, FontWeight, HintMode, GlyphAtlas}

fn build_collection(lib: &FontLibrary) ! IO {
    let query_latin = FontQuery::new().family("Noto Sans").weight(FontWeight.Regular)
    let query_emoji = FontQuery::new().family("Noto Color Emoji")
    let query_cjk   = FontQuery::new().family("Noto Sans CJK SC")

    let latin_desc = lib.match_font(query_latin).unwrap().into_iter().next().unwrap()
    let emoji_desc = lib.match_font(query_emoji).unwrap().into_iter().next().unwrap()
    let cjk_desc   = lib.match_font(query_cjk).unwrap().into_iter().next().unwrap()

    let latin = lib.load_font_file(&latin_desc.path).unwrap()
    let emoji = lib.load_font_file(&emoji_desc.path).unwrap()
    let cjk   = lib.load_font_file(&cjk_desc.path).unwrap()

    let mut collection = FontCollection::new()
    collection.add_face(latin, 100)   // tried first
    collection.add_face(emoji, 50)    // fallback for emoji characters
    collection.add_face(cjk, 30)      // fallback for CJK

    // Look up any character, including emoji and CJK, across the collection.
    let ch = '\u{1F600}'   // grinning face emoji
    if let Some((face, glyph_id)) = collection.char_to_glyph(ch) {
        let font = face.scale(24.0)
        let bitmap = font.rasterize(glyph_id, HintMode.None).unwrap()
        // bitmap.format == GlyphBitmapFormat.Color for color emoji
    }
}
```

---

## 12. Dependencies

### External

| Dependency | Required | Notes |
|---|---|---|
| FreeType 2 | Yes | Font rasterizer. Linked as a C library via Ferrum FFI. |
| fontconfig | Optional | Linux system font discovery. Compiled in when `features = ["fontconfig"]`. |
| CoreText | Optional | macOS system font discovery. Compiled in on macOS targets automatically. |
| DirectWrite | Optional | Windows system font discovery. Compiled in on Windows targets automatically. |

### Ferrum Stdlib

| Module | Used for |
|---|---|
| `std.alloc` | `Vec[u8]`, `Arc[Vec[u8]]`, `String` allocation |
| `std.fs` | `Path`, `PathBuf`, file reading in `load_font_file` |
| `std.sync` | `Arc`, internal `Mutex` for FreeType serialization |

### Ferrum Extlib

| Module | Used for |
|---|---|
| `extlib.draw` | `Rect` (UV rectangle in `AtlasEntry`) |

The `extlib.draw` dependency is for the `Rect` type only. Programs that use `extlib.font` without `extlib.draw` can define a compatible `Rect` or use the UV coordinates as raw `(f32, f32, f32, f32)` tuples via the `AtlasEntry::uv_raw()` accessor (returns `(u: f32, v: f32, w: f32, h: f32)` without the `Rect` dependency).

---

## Implementation Notes

### FreeType Threading Model

FreeType's `FT_Library` is not thread-safe for concurrent calls on the same library instance. The `FontLibrary` implementation wraps the FT_Library in a `Mutex` and acquires it for each operation. This serializes FreeType calls across threads. In practice, font loading and rasterization are called infrequently enough that mutex contention is not a bottleneck; if a program requires concurrent rasterization at high throughput, it should create one `FontLibrary` per thread.

### Memory Ownership for `load_font_bytes`

When loading from bytes, FreeType needs the buffer to remain valid for the lifetime of the face (FreeType reads glyph data on demand, not eagerly). The `Arc[Vec[u8]]` passed to `load_font_bytes` is cloned into the `FontFace` and kept alive until the face is dropped. No copy of the font data is made; the `Arc` reference count is the only overhead.

### Shelf Packing Eviction

When LRU eviction is enabled via `set_max_bytes`, eviction operates at the shelf level: when a shelf's glyphs are all evicted, the shelf row is reclaimed for new allocations. Glyphs within a shelf are not individually reclaimed (doing so would require compaction, which invalidates UV coordinates held by callers). This means eviction is coarse-grained; callers with highly varied glyph sets should use larger atlases.

### `Rect` Type

`Rect` is defined in `extlib.draw`:

```ferrum
// From extlib.draw — included here for reference.
@derive(Copy, Clone, Debug)
pub type Rect {
    pub x:      f32,
    pub y:      f32,
    pub width:  f32,
    pub height: f32,
}
```

UV coordinates in `AtlasEntry.uv_rect` are in [0.0, 1.0] in both axes, with the origin at the top-left of the atlas texture, consistent with the texture coordinate convention used by Vulkan, Metal, and Direct3D (top-left origin).
