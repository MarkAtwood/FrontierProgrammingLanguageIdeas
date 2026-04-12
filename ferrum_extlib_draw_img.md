# Ferrum Extended Library — draw.img

**Module:** `extlib.draw.img`
**Part of:** Extended Standard Library (not `std` or `core`)
**Companion:** [Ferrum Standard Library](ferrum-stdlib.md), `extlib.draw` (DrawBackend trait and DrawContext)
**CCSP basis:** `lib_ccsp_draw_img` — software rasterizer to memory buffer

---

## 1. Overview and Rationale

`extlib.draw.img` is a CPU-only, software-rasterized backend for the Ferrum drawing library (`extlib.draw`). It renders drawing operations into an in-memory RGBA pixel buffer and can write the result as PNG, PPM, raw RGBA, or single-page PDF.

This module serves three distinct use cases.

### 1.1 Reference Implementation for Visual Regression Testing

GPU-accelerated draw backends — Metal, Vulkan, WebGPU — produce pixels through hardware pipelines that have driver bugs, rounding differences, and platform-specific behavior. When the question "which backend is correct?" arises, `extlib.draw.img` is the answer by definition.

The reference status is not aspirational; it is contractual. If the software rasterizer and a GPU backend disagree on a pixel, the software rasterizer's output is correct. GPU backends are tested against it, not the other way around. This gives the test suite a stable, portable oracle that runs identically on every CI machine regardless of GPU availability.

Visual regression testing works by rendering a scene with `ImageBackend`, saving the result as a reference PNG, and on subsequent runs comparing the new render against the reference pixel-by-pixel with a configurable tolerance. The `diff` and `assert_image_eq` functions in this module support exactly this workflow.

### 1.2 Documentation Generation

Ferrum's documentation tooling renders code examples in documentation comments by actually executing them. The image in the documentation IS the running code: if the example is wrong, the image is wrong, and the documentation build fails. `extlib.draw.img` is the backend that makes this possible — it has no display requirements, no GPU requirements, and no windowing system requirements. It can run in a documentation build pipeline on any host.

The `render_to_file` convenience function and the `set_label` method on `ImageBackend` exist specifically to support this workflow.

### 1.3 Server-Side Image Generation

Servers that generate charts, report thumbnails, map tiles, or PDF output cannot assume a GPU or display. `extlib.draw.img` provides a complete drawing backend that operates entirely in memory, writes PNG or PDF output, and carries the `! IO` effect only at the point of writing to disk — the rendering itself is pure computation over the pixel buffer.

### Why a CPU Rasterizer Is the Right Reference

Hardware rasterizers are non-deterministic across platforms, driver versions, and GPU models. Floating-point rounding in GPU shaders differs between vendors. A reference implementation must be: deterministic (same input always produces the same output), reproducible (the same on every machine), and inspectable (the algorithm is readable code, not a driver black box). A software rasterizer satisfies all three. The CCSP `lib_ccsp_draw_img` was designed with these properties as explicit requirements.

---

## 2. ImageBackend

`ImageBackend` is the central type of this module. It holds an RGBA working buffer and implements the `DrawBackend` trait from `extlib.draw`, which means a `DrawContext` can be constructed from it and all draw operations can be issued against it.

### 2.1 Constructors

```ferrum
use extlib.draw.img.{ImageBackend, ImageBackendConfig, SubpixelMode}
use extlib.draw.Color

impl ImageBackend {
    /// Create a new backend with a transparent black background.
    /// Width and height are in physical pixels.
    pub fn new(width: u32, height: u32): ImageBackend

    /// Create a backend with full configuration control.
    pub fn with_config(config: ImageBackendConfig): ImageBackend
}
```

`new` is the common path. It produces a backend filled with transparent black (`Color { r: 0.0, g: 0.0, b: 0.0, a: 0.0 }` in linear light). All subsequent draw operations composite over this initial state using Porter-Duff Over.

### 2.2 ImageBackendConfig

```ferrum
/// Full configuration for ImageBackend construction.
struct ImageBackendConfig {
    width:             u32,
    height:            u32,
    /// Initial background color. Default: transparent black.
    background:        Color,
    /// Enable anti-aliasing. Default: true.
    antialias:         bool,
    /// LCD subpixel rendering mode. Default: SubpixelMode.None.
    subpixel_rendering: SubpixelMode,
}

impl ImageBackendConfig {
    pub fn default(width: u32, height: u32): Self
}
```

`background` is in the same linear-light color space used throughout the draw extlib. If you want an opaque white background, use `Color::from_srgb(1.0, 1.0, 1.0, 1.0)` (which converts sRGB white to linear light).

### 2.3 SubpixelMode

```ferrum
/// LCD subpixel rendering hint for glyph rasterization.
@derive(Debug, Clone, PartialEq)
enum SubpixelMode {
    /// No subpixel rendering. Safe for any output (PNG, PDF, unknown display).
    None,
    /// Horizontal RGB stripe order (most common: left-to-right R, G, B).
    HorizontalRgb,
    /// Horizontal BGR stripe order (some laptop panels, most Windows default ClearType).
    HorizontalBgr,
}
```

Subpixel rendering triples effective horizontal resolution for text by treating each color channel of an LCD pixel as a separate sample. The result looks wrong in any context other than the specific display it was tuned for. For reference rendering, visual regression testing, and PDF output, use `SubpixelMode.None`.

### 2.4 DrawBackend Implementation

```ferrum
impl extlib.draw.DrawBackend for ImageBackend {
    // All draw operations from extlib.draw are implemented here in software.
    // See Section 3 for the rasterization details.
}
```

Constructing a `DrawContext` from an `ImageBackend`:

```ferrum
use extlib.draw.DrawContext
use extlib.draw.img.ImageBackend

let mut backend = ImageBackend.new(800, 600)
let mut ctx = DrawContext.new(&mut backend)
// All draw operations on ctx are now CPU-rasterized into backend's pixel buffer.
```

---

## 3. CPU Rasterizer

The rasterizer is the core of this module. Every draw operation that `DrawContext` can issue is implemented entirely in software, operating on the in-memory RGBA pixel buffer. There is no GPU involvement at any stage.

### 3.1 Path Rasterization

Filled paths use a scan-line algorithm: for each pixel row intersected by the path, the fill algorithm computes the x-intercepts of all edges, sorts them, and fills the spans between pairs of intercepts. The fill rule (even-odd or non-zero winding) follows the setting on the fill paint passed through `extlib.draw`.

Stroked paths are rendered using a widened stroke: the stroke is expanded into a filled outline, then the filled outline is rasterized with the scan-line algorithm. This produces correct joins and caps (miter, round, bevel; butt, round, square) matching the draw extlib specification.

Bresenham line drawing is used only for non-anti-aliased single-pixel strokes where exactness and speed are both needed. In all other cases, the analytical path rasterizer is used.

### 3.2 Anti-Aliasing

When `antialias: true` (the default):

- **Axis-aligned rectangles:** analytical AA computes the exact coverage fraction for each edge pixel. This is exact and inexpensive — no supersampling.
- **Arbitrary paths:** 4×4 MSAA (16 samples per pixel) by default. For paths where an analytical formula is tractable (circles, rounded rects), analytical AA is used instead.
- **Single-pixel strokes:** Wu's algorithm for lines; analytical coverage for 1px axis-aligned rects.

When `antialias: false`, all edges are snapped to integer pixel boundaries. Use this for pixel-art, test fixtures where exact pixel matches are required, or performance-sensitive offline rendering where AA is not needed.

### 3.3 Gradient Rendering

Linear, radial, and conic gradients are computed per-pixel. Gradient color stops are interpolated in **Oklab color space**, matching the behavior specified in `extlib.draw`. Interpolation in Oklab avoids the muddy colors that appear when interpolating in sRGB or linear RGB through intermediate hues.

The interpolation pipeline for each pixel:

1. Compute the gradient parameter t for the pixel's position.
2. Find the surrounding color stops.
3. Interpolate in Oklab (L, a, b channels independently, with linear t).
4. Convert from Oklab to linear-light RGBA.
5. Composite using Porter-Duff Over.

This ensures that a gradient rendered by `ImageBackend` and a gradient rendered by a GPU backend (which should also interpolate in Oklab, per the draw extlib spec) produce the same perceptual result, and any discrepancy is attributable to the GPU backend.

### 3.4 Compositing

All compositing uses **Porter-Duff Over** in **linear light** (linear RGB, not gamma-encoded sRGB). This matches the compositing model specified in `extlib.draw`.

The compositing pipeline:

1. All source colors are stored in linear-light RGBA internally.
2. Per-pixel: `result = src + dst * (1 - src.a)` (standard Over formula), applied channel-wise in linear light.
3. The final buffer contains linear-light RGBA. Conversion to sRGB happens only at output time, in the format writers (PNG, PPM, PDF), not during compositing.

Premultiplied alpha is used internally for compositing correctness. Source colors from the draw extlib are in straight alpha; they are premultiplied on input to the compositing operation and un-premultiplied on output to pixel access.

### 3.5 Glyph Rendering

Glyph bitmaps are provided by `extlib.font`. The font extlib rasterizes each glyph into a grayscale coverage bitmap at a requested pixel size. `extlib.draw.img` blits these bitmaps into the pixel buffer:

- For `SubpixelMode.None`: the grayscale coverage is used as an alpha mask, composited over the destination with the text fill color.
- For `SubpixelMode.HorizontalRgb` and `SubpixelMode.HorizontalBgr`: the font extlib provides a per-channel coverage bitmap at 3× horizontal resolution. The three per-channel coverages are applied independently to the R, G, B channels of the destination.

Glyph bitmaps are cached by the font extlib, not by `extlib.draw.img`. `ImageBackend` does not own a glyph cache; it receives pre-rasterized bitmaps from the draw extlib layer.

### 3.6 Image Sampling

When a draw operation paints a source image (loaded via `DrawContext.create_image`), the image is sampled during rasterization. Three sampling modes are supported:

| Mode | Quality | Cost |
|---|---|---|
| `Nearest` | Blocky at upscale, aliased at downscale | O(1) per pixel |
| `Bilinear` | Smooth interpolation; some blur | 4 samples per pixel |
| `Lanczos3` | High quality; correct downscaling | ~36 samples per pixel |

The default is `Nearest`. The draw extlib specifies which sampling mode to use per image paint; `ImageBackend` respects that specification.

All source images are converted to linear-light RGBA before sampling (see Section 9). Sampling is performed in linear light, not in sRGB, to avoid brightness artifacts in bilinear and Lanczos3 kernels.

---

## 4. Output Formats

All output functions take a reference to an `ImageBackend` and write the pixel buffer in the specified format. They do not mutate the backend. The `! IO` effect appears on functions that perform I/O; `to_vec` and `to_image_data` do not perform I/O and carry no effect annotation.

### 4.1 PNG

```ferrum
/// Write the backend's pixel buffer as RGBA PNG.
/// Lossless. Alpha channel is preserved.
pub fn write_png(
    backend: &ImageBackend,
    writer:  &mut impl extlib.io.Write,
): Result[(), ImgError] ! IO

/// Convenience form: open path for writing, call write_png, close.
pub fn write_png_file(
    backend: &ImageBackend,
    path:    &std.fs.Path,
): Result[(), ImgError] ! IO
```

PNG output applies the sRGB transfer function to the linear-light buffer before encoding (gamma-encodes to sRGB). The PNG file is tagged with the sRGB color space chunk so conformant viewers display it correctly. Alpha is straight (not premultiplied) in the output file, per PNG convention.

### 4.2 PPM

```ferrum
/// Write the backend's pixel buffer as RGB PPM (P6 binary format).
/// No alpha channel. Simple format: no external library required to parse.
pub fn write_ppm(
    backend: &ImageBackend,
    writer:  &mut impl extlib.io.Write,
): Result[(), ImgError] ! IO
```

PPM is a trivially simple format: a short ASCII header followed by raw RGB bytes. It has no alpha channel. The alpha from the backend is composited against opaque black before writing. PPM output is intended for unit tests and debugging where an external PNG library may not be available.

The format is P6 (binary), 8 bits per channel, sRGB-encoded (the same gamma encoding as PNG).

### 4.3 Raw RGBA

```ferrum
/// Write raw RGBA bytes with no header or metadata.
/// Row-major, top-to-bottom, 4 bytes per pixel (R, G, B, A), sRGB-encoded.
pub fn write_raw_rgba(
    backend: &ImageBackend,
    writer:  &mut impl extlib.io.Write,
): Result[(), ImgError] ! IO
```

Raw output is for cases where the caller controls the framing: embedding in a custom file format, writing to a shared memory region, feeding into an image processing pipeline. The buffer is sRGB-encoded (not linear light) for compatibility with conventional image consumers.

### 4.4 PDF

```ferrum
/// Write a single-page PDF containing the rasterized image.
pub fn write_pdf(
    backend:  &ImageBackend,
    writer:   &mut impl extlib.io.Write,
    metadata: &PdfMetadata,
): Result[(), ImgError] ! IO
```

The image is embedded as a rasterized bitmap inside the PDF — this is not a vector PDF. The `dpi` field in `PdfMetadata` controls the physical page size: a 1000×1000 pixel image at 96 DPI produces a page approximately 10.4 × 10.4 inches; at 300 DPI, approximately 3.3 × 3.3 inches. Vector PDF output (embedding draw operations as PDF graphics operators instead of a bitmap) is a potential future extension but is not part of this module.

### 4.5 In-Memory Access

```ferrum
/// Return the pixel buffer as RGBA bytes (sRGB-encoded).
/// Length is width × height × 4.
/// Allocates a new Vec[u8].
pub fn to_vec(backend: &ImageBackend): Vec[u8]

/// Return the pixel buffer as ImageData for passing to other backends
/// or to DrawContext.create_image.
pub fn to_image_data(backend: &ImageBackend): extlib.draw.ImageData
```

`to_vec` applies the sRGB transfer function before returning, matching the encoding used by all file format writers. The internal buffer is linear-light; callers who want linear-light values should use the pixel access API in Section 10.

`to_image_data` returns the linear-light buffer wrapped in an `ImageData` struct, suitable for use as a source image in another draw context (including a GPU-backed one).

---

## 5. PDF Output Details

### 5.1 PdfMetadata

```ferrum
/// Metadata embedded in the PDF document information dictionary.
struct PdfMetadata {
    /// Document title (optional).
    title:   Option[String],
    /// Author name (optional).
    author:  Option[String],
    /// Subject or description (optional).
    subject: Option[String],
    /// Creator application string. Included in PDF metadata.
    /// Typically the name of the Ferrum program that produced the file.
    creator: &'static str,
    /// Pixels per inch. Controls physical page dimensions.
    /// Default: 96.0. Use 150.0 or 300.0 for print-quality output.
    dpi:     f32,
}

impl PdfMetadata {
    pub fn default(creator: &'static str): Self
}
```

### 5.2 PDF Structure

The output is a minimal conformant PDF 1.4 document:

- One page object, size derived from `width / dpi` × `height / dpi` in inches.
- One image XObject: the rasterized RGBA bitmap, JPEG-compressed for document size (or deflate-compressed if the image has significant transparency).
- Document information dictionary with any non-`None` metadata fields populated.
- No JavaScript, no forms, no interactive features.

The resulting PDF is not signed, not encrypted, and not linearized. It is intended for documentation generation and report output, not secure document distribution.

### 5.3 Physical Size Calculation

Page dimensions in PDF user units (points, 1/72 inch):

```
page_width_pts  = (backend.width()  as f32 / metadata.dpi) * 72.0
page_height_pts = (backend.height() as f32 / metadata.dpi) * 72.0
```

At `dpi: 96.0`, a 960×540 pixel image produces a 720×405 point page (10 × 5.625 inches).

### 5.4 Future Extension

Vector PDF output — representing draw operations as PDF graphics operators (paths, text, gradients) rather than embedding a rasterized bitmap — would allow PDFs that scale to any print resolution. This is out of scope for the current module, which focuses on raster output. A future `extlib.draw.pdf` module would implement vector PDF.

---

## 6. Visual Regression Testing

### 6.1 Pixel Diff

```ferrum
/// Compare two images pixel by pixel. Both must have the same dimensions.
/// Returns Err(ImgError.DimensionMismatch) if dimensions differ.
pub fn diff(
    a: &ImageBackend,
    b: &ImageBackend,
): Result[ImageDiff, ImgError]

/// The result of a pixel-by-pixel comparison.
struct ImageDiff {
    /// Maximum delta across all pixels and channels (0.0 = identical, 1.0 = maximally different).
    max_delta:         f32,
    /// Mean delta across all pixels and channels.
    mean_delta:        f32,
    /// Number of pixels where any channel differs by more than floating-point epsilon.
    different_pixels:  u32,
    /// Total number of pixels (width × height).
    total_pixels:      u32,
    /// Visualization of differences. None if images are identical.
    /// Black = no difference. Red channel = normalized delta magnitude.
    delta_image:       Option[ImageBackend],
}

impl ImageDiff {
    /// True if all pixels are identical (different_pixels == 0).
    pub fn is_identical(&self): bool

    /// Fraction of pixels that differ (0.0 .. 1.0).
    pub fn different_fraction(&self): f32
}
```

Delta values are computed in linear light, channel-wise, as `|a_channel - b_channel|`, then averaged across channels. The `max_delta` and `mean_delta` fields are in the [0.0, 1.0] range.

The `delta_image` visualization maps each pixel's mean channel delta to a color: zero delta produces pure black; delta 1.0 produces pure red. Intermediate values are linearly interpolated. This makes it easy to see at a glance whether differences are scattered random noise (ReDoS-style: many small red dots) or systematic shifts (a large solid red region).

### 6.2 Test Assertion

```ferrum
/// Assert that actual and expected are within tolerance.
///
/// On failure: prints a diff summary (max_delta, mean_delta, different_pixels/total_pixels),
/// saves the delta image to /tmp/ferrum_test_diff_<timestamp>.png,
/// and panics with the diff summary as the panic message.
///
/// tolerance is compared against max_delta (the strictest measure).
/// Pass 0.0 for exact pixel equality; pass a small value like 0.01 for
/// tolerance against floating-point rounding differences between platforms.
pub fn assert_image_eq(
    actual:    &ImageBackend,
    expected:  &ImageBackend,
    tolerance: f32,
) ! IO
```

The saved delta PNG path includes a timestamp to avoid collisions when multiple tests fail in parallel. The path is printed to stderr before the panic, so it is visible in test output even when the terminal discards the panic message.

`assert_image_eq` carries `! IO` because it writes the delta PNG to `/tmp` on failure. Tests that call it must declare `! IO` in their signature.

### 6.3 Loading Reference Images

```ferrum
/// Load a PNG file from disk as an ImageBackend (linear-light RGBA, sRGB decoded).
/// The backend can be used directly as the expected argument to assert_image_eq.
pub fn load_png(path: &std.fs.Path): Result[ImageBackend, ImgError] ! IO

/// Load a PNG from an in-memory byte slice.
/// Intended for embedding reference images as test fixtures with include_bytes!().
/// Does not carry ! IO — the bytes are already in memory.
pub fn load_png_bytes(data: &[u8]): Result[ImageBackend, ImgError]
```

Reference images for tests are typically committed to the repository as PNG files and loaded with `load_png` or embedded with `include_bytes!()` and loaded with `load_png_bytes`. Embedded fixtures make tests fully self-contained with no filesystem dependency at test runtime.

All loaded PNGs are decoded to linear-light RGBA (the inverse sRGB transfer function is applied on load). This matches the internal representation of `ImageBackend` constructed from scratch.

---

## 7. Documentation Rendering

### 7.1 Image Label

```ferrum
impl ImageBackend {
    /// Attach a human-readable label to this backend.
    /// Stored in the PNG tEXt chunk under key "Comment" when writing PNG.
    /// Has no effect on rendering. Useful for identifying images in debug output.
    pub fn set_label(&mut self, text: &str)

    pub fn label(&self): Option[&str]
}
```

The label survives a round-trip through `write_png` / `load_png`. When the Ferrum documentation tool renders a code example, it sets the label to the fully-qualified name of the example function, making it easy to correlate a PNG file in the docs output directory with the code that generated it.

### 7.2 Render to File Convenience

```ferrum
/// Render a draw function to a PNG file.
/// Creates an ImageBackend of the given dimensions, constructs a DrawContext,
/// calls draw_fn, then writes the result to path.
///
/// This is the canonical way to produce a documentation screenshot from a
/// draw function.
pub fn render_to_file(
    draw_fn: impl Fn(&mut extlib.draw.DrawContext),
    path:    &std.fs.Path,
    width:   u32,
    height:  u32,
) : Result[(), ImgError] ! IO
```

Example use in documentation:

```ferrum
/// Renders a gradient button. The image below is generated from this exact code.
///
/// ![gradient button](gradient_button.png)
pub fn draw_gradient_button(ctx: &mut DrawContext) {
    // ... draw operations ...
}

// In the documentation build script:
render_to_file(draw_gradient_button, Path.new("gradient_button.png"), 200, 60)?
```

---

## 8. Multi-Frame and Animation Support

`extlib.draw.img` supports Animated PNG (APNG) for creating animated examples in documentation or animated UI test recordings.

### 8.1 AnimatedPngWriter

```ferrum
/// Writer for Animated PNG (APNG) output.
///
/// Frames are added sequentially. Each frame must have the same dimensions
/// as the first frame.
struct AnimatedPngWriter { ... }

impl AnimatedPngWriter {
    /// Create a new APNG writer.
    /// writer: destination for the APNG bytes.
    /// delay_ms: display duration for each frame in milliseconds.
    ///           All frames share the same delay; per-frame delay is a future extension.
    pub fn new(writer: impl extlib.io.Write, delay_ms: u32): Self

    /// Append a frame. The backend's current pixel buffer is captured at call time.
    /// Returns Err if this frame's dimensions differ from the first frame.
    pub fn add_frame(&mut self, backend: &ImageBackend): Result[(), ImgError] ! IO

    /// Finalize the APNG and flush the writer.
    /// Must be called to produce a valid APNG file.
    /// After finish(), no more frames may be added.
    pub fn finish(self): Result[(), ImgError] ! IO
}
```

### 8.2 Usage Pattern

```ferrum
use extlib.draw.img.{ImageBackend, AnimatedPngWriter}
use std.fs.File

let file = File.create(Path.new("animation.png"))?
let mut apng = AnimatedPngWriter.new(file, 100)   // 100ms per frame = 10 fps

for frame_index in 0..60 {
    let mut backend = ImageBackend.new(400, 300)
    let mut ctx = DrawContext.new(&mut backend)
    draw_frame(&mut ctx, frame_index)
    apng.add_frame(&backend)?
}

apng.finish()?
```

APNG is a backward-compatible extension of PNG: viewers that do not support APNG display only the first frame without error.

---

## 9. Image Loading

`extlib.draw.img` provides decoders for common image formats, producing `ImageData` suitable for use with `DrawContext.create_image`.

### 9.1 Image Loading Functions

```ferrum
use extlib.draw.img.{load_png, load_png_bytes, load_jpeg, load_webp}
use extlib.draw.ImageData

/// Load a PNG from a file path. Decodes to linear-light RGBA.
pub fn load_png(path: &std.fs.Path): Result[ImageData, ImgError] ! IO

/// Load a PNG from a byte slice (e.g., include_bytes!()).
/// No ! IO — data is already in memory.
pub fn load_png_bytes(data: &[u8]): Result[ImageData, ImgError]

/// Decode a JPEG to linear-light RGBA.
/// JPEG has no alpha channel; the A channel is set to 1.0.
pub fn load_jpeg(path: &std.fs.Path): Result[ImageData, ImgError] ! IO

/// Decode a WebP to linear-light RGBA.
/// Supports both lossy and lossless WebP. Alpha is preserved if present.
pub fn load_webp(path: &std.fs.Path): Result[ImageData, ImgError] ! IO
```

### 9.2 ImageData Structure

All loaders produce:

```ferrum
use extlib.draw.{ImageData, PixelFormat}

struct ImageData {
    width:  u32,
    height: u32,
    /// Always Rgba8Unorm after loading. The 8-bit values are linear-light, not sRGB.
    format: PixelFormat,
    /// Row-major, top-to-bottom, 4 bytes per pixel (R, G, B, A) in linear light.
    data:   Vec[u8],
}
```

### 9.3 Color Space Handling on Load

All loaders apply the inverse sRGB transfer function (gamma decoding) to convert from the stored sRGB encoding to linear light. This is applied unconditionally:

- PNG with sRGB chunk: decoded per the sRGB transfer function.
- PNG without color space metadata: assumed sRGB and decoded the same way.
- JPEG: assumed sRGB (the overwhelmingly common case); decoded to linear light.
- WebP: assumed sRGB; decoded to linear light.

Images are stored in linear light internally because compositing and gradient interpolation must occur in linear light to be physically correct. Conversion to sRGB happens only at output time.

The `data` field of `ImageData` is therefore in linear light, even though the file on disk was sRGB-encoded. Callers who read raw bytes from `ImageData` will get linear-light values, not sRGB values.

---

## 10. Pixel Access

Pixel access functions return colors in **linear light** — the same representation used internally by the rasterizer. This is suitable for analysis, test assertions, and passing colors to other linear-light computations.

```ferrum
impl ImageBackend {
    /// Get the color of a single pixel at (x, y).
    /// x must be in [0, width), y in [0, height). Panics otherwise.
    /// Color is in linear-light RGBA, straight alpha.
    pub fn pixel(&self, x: u32, y: u32): extlib.draw.Color

    /// Get a row of pixels as a slice.
    /// y must be in [0, height). Panics otherwise.
    /// Each Color is in linear-light RGBA, straight alpha.
    pub fn row(&self, y: u32): &[extlib.draw.Color]

    /// Get all pixels as a flat slice, row-major, top-to-bottom.
    /// Length is width × height.
    pub fn pixels(&self): &[extlib.draw.Color]

    /// Image width in pixels.
    pub fn width(&self): u32

    /// Image height in pixels.
    pub fn height(&self): u32

    /// Image dimensions as a Size.
    pub fn size(&self): extlib.draw.Size
}
```

These methods do not allocate. They return references into the backend's internal buffer.

For callers who want sRGB-encoded bytes (for comparison with file output), use `to_vec()` (Section 4.5), which applies the sRGB transfer function before returning.

---

## 11. Error Types

```ferrum
use extlib.draw.Size

/// Errors produced by extlib.draw.img operations.
@derive(Debug)
enum ImgError {
    /// An I/O error occurred while reading or writing.
    Io(std.io.IoError),

    /// PNG encode or decode failed (e.g., invalid signature, corrupted IDAT chunk).
    PngError(String),

    /// JPEG decode failed (e.g., corrupted DCT data, unsupported encoding).
    JpegError(String),

    /// WebP decode failed.
    WebpError(String),

    /// Width or height is zero, or exceeds the platform maximum (2^16 - 1 pixels per dimension).
    InvalidDimensions,

    /// Two images were compared or combined but their dimensions differ.
    DimensionMismatch { a: Size, b: Size },

    /// Pixel format not supported by this operation (e.g., non-RGBA format passed to write_ppm).
    UnsupportedFormat,
}

impl std.fmt.Display for ImgError {
    // Formats as:
    //   I/O error: <IoError>
    //   PNG error: <message>
    //   JPEG error: <message>
    //   WebP error: <message>
    //   Invalid dimensions
    //   Dimension mismatch: a is 800x600, b is 1024x768
    //   Unsupported pixel format
}

impl std.error.Error for ImgError {}
```

`ImgError` uses string descriptions for codec errors because codec libraries produce string diagnostics, not typed error codes. The `Io` variant wraps `std.io.IoError` directly so callers can use `?` to propagate I/O errors in functions that return `Result[_, ImgError]`.

---

## 12. Example Usage

### 12.1 Visual Regression Test for a Widget

```ferrum
use extlib.draw.{DrawContext, Color}
use extlib.draw.img.{ImageBackend, assert_image_eq, load_png_bytes}

fn draw_checkbox(ctx: &mut DrawContext, checked: bool) {
    // ... draw the checkbox widget ...
}

#[test]
fn test_checkbox_checked() ! IO {
    let mut backend = ImageBackend.new(32, 32)
    let mut ctx = DrawContext.new(&mut backend)
    draw_checkbox(&mut ctx, true)

    let reference = load_png_bytes(include_bytes!("fixtures/checkbox_checked.png"))
        .expect("fixture must be valid PNG")
    assert_image_eq(&backend, &reference, 0.005)
}
```

The `include_bytes!()` intrinsic embeds the PNG fixture at compile time, making the test fully self-contained. The tolerance of `0.005` permits rounding differences across platforms while catching any meaningful regression.

### 12.2 Generate a Documentation Screenshot

```ferrum
use extlib.draw.DrawContext
use extlib.draw.img.render_to_file
use std.fs.Path

fn draw_progress_bar(ctx: &mut DrawContext) {
    // ... draw a 75% full progress bar ...
}

fn main() ! IO {
    render_to_file(
        draw_progress_bar,
        Path.new("docs/images/progress_bar.png"),
        300,
        24,
    )?
}
```

### 12.3 Render a Chart to PNG

```ferrum
use extlib.draw.{DrawContext, Color, Rect, Paint}
use extlib.draw.img::{ImageBackend, write_png_file}
use std.fs.Path

fn render_bar_chart(data: &[f32], path: &Path): Result[(), extlib.draw.img.ImgError] ! IO {
    let mut backend = ImageBackend.new(640, 480)
    let mut ctx = DrawContext.new(&mut backend)

    // white background
    ctx.fill_rect(Rect.new(0.0, 0.0, 640.0, 480.0), Paint.solid(Color::from_srgb(1.0, 1.0, 1.0, 1.0)))

    let bar_width = 640.0 / data.len() as f32
    for (i, &value) in data.iter().enumerate() {
        let height = value * 400.0
        let x = i as f32 * bar_width + 4.0
        let y = 440.0 - height
        ctx.fill_rect(
            Rect.new(x, y, bar_width - 8.0, height),
            Paint.solid(Color::from_srgb(0.2, 0.5, 0.9, 1.0)),
        )
    }

    write_png_file(&backend, path)
}
```

### 12.4 Compare GPU Backend Against Reference

```ferrum
use extlib.draw.{DrawContext, DrawBackend}
use extlib.draw.img::{ImageBackend, diff}
use extlib.draw.gpu.GpuBackend   // hypothetical GPU backend

fn assert_gpu_matches_reference(
    draw_fn: impl Fn(&mut DrawContext),
    width:   u32,
    height:  u32,
) ! IO {
    // Render with the reference CPU backend
    let mut ref_backend = ImageBackend.new(width, height)
    let mut ref_ctx = DrawContext.new(&mut ref_backend)
    draw_fn(&mut ref_ctx)

    // Render with the GPU backend, retrieve pixels as ImageBackend
    let mut gpu_backend = GpuBackend.new(width, height)
    let mut gpu_ctx = DrawContext.new(&mut gpu_backend)
    draw_fn(&mut gpu_ctx)
    let gpu_image = gpu_backend.to_image_backend()   // GPU backend's readback method

    let result = diff(&gpu_image, &ref_backend).expect("same dimensions")
    if result.max_delta > 0.01 {
        println("GPU vs reference: max_delta={:.4} mean_delta={:.4} different={}/{}",
            result.max_delta,
            result.mean_delta,
            result.different_pixels,
            result.total_pixels,
        )
        panic("GPU backend differs from reference beyond tolerance")
    }
}
```

---

## 13. Dependencies

### 13.1 Ferrum Extended Library

| Module | Used for |
|---|---|
| `extlib.draw` | `DrawBackend` trait, `DrawContext`, `Color`, `ImageData`, `Size`, `Rect`, `Paint`, `PixelFormat` |
| `extlib.font` | Glyph rasterization: the font extlib provides glyph coverage bitmaps on request |

### 13.2 External Libraries (Optional, Feature-Gated)

| Library | Feature flag | Used for |
|---|---|---|
| `libpng` | `feature = "png"` (default on) | PNG encode and decode |
| `libjpeg-turbo` | `feature = "jpeg"` (default on) | JPEG decode |
| `libwebp` | `feature = "webp"` (default off) | WebP decode |

When a feature is disabled, the corresponding load and write functions return `Err(ImgError.UnsupportedFormat)` at runtime. PNG is on by default because it is the primary output format for testing and documentation. JPEG is on by default because it is the most common source image format.

The PDF writer (`write_pdf`) has no external library dependency: the minimal PDF structure required for embedded raster images can be produced using only `std.io` and `std.fmt`.

The APNG writer has no external library dependency beyond the PNG encoder.

### 13.3 Standard Library

| Module | Used for |
|---|---|
| `std.fs` | `Path`, `File` for file-based load and write functions |
| `std.io` | `Write` trait, `IoError` |
| `std.fmt` | `Display`, `Debug` for `ImgError` |
| `alloc.vec` | `Vec[u8]` for pixel buffers and output staging |
| `alloc.string` | `String` for error messages and labels |
