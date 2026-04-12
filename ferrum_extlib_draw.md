# Ferrum Extended Library — draw

**Module:** `extlib.ccsp.draw`
**Spec basis:** CCSP `lib_ccsp_draw` backend-agnostic 2D drawing specification
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `alloc`, `math` (stdlib only; intentionally foundational)

---

## 1. Overview and Rationale

### Why a Backend-Agnostic Drawing Layer

A GUI toolkit, a game engine, a terminal emulator, and a data visualization library all need to draw colored rectangles, bezier curves, gradients, and text. Without a shared abstraction, each program must pick one graphics backend at compile time and wire directly to it. Wayland programs cannot reuse the same widget code on X11. A headless test runner cannot share render logic with the GPU path. An image-export pipeline must duplicate the entire drawing layer.

`extlib.ccsp.draw` solves this by defining a **single drawing vocabulary** that all display targets implement. The widget system, the layout engine, the chart renderer — all code above this layer — speaks one language regardless of what is underneath. Backends are swappable:

- **Wayland** — native Wayland compositor client via shared memory or wl_drm
- **X11** — XCB with XRender or direct framebuffer fallback
- **Framebuffer** — software rasterizer, no GPU required, suitable for embedded displays
- **GPU** — Vulkan, Metal, or WebGPU via compute shaders
- **Image file** — PNG or JPEG output for headless rendering and testing

No display-specific API call appears above this layer. A widget that calls `ctx.fill_rect(...)` does not know whether the frame is destined for a Wayland surface, a PNG encoder, or a unit test. This is a hard invariant: the `DrawContext` and `DrawBackend` interface contains no platform-specific types, handles, or function pointers. Platform-specific initialization lives entirely below the trait boundary.

### Why Float Coordinates

Integer pixel coordinates seem simpler but produce incorrect results at fractional DPI. A display scaled at 1.5x (common on HiDPI monitors) has no integer coordinates that correctly represent "half a CSS pixel." Rounding to the nearest pixel produces misalignment, blurry text baselines, and inconsistent element spacing across screens.

Float coordinates let the backend choose how to map logical coordinates to physical pixels. At 1x DPI, `Rect { origin: Point { x: 10.0, y: 20.0 }, ... }` maps to integer pixels exactly. At 2x DPI, the same rect maps to double-density pixels, also exactly. At 1.5x DPI, the backend anti-aliases appropriately. Callers never concern themselves with the physical pixel grid: they specify geometry in logical units and the backend handles the mapping.

This is the same coordinate model used by CSS, Core Graphics (macOS/iOS), and Android's density-independent pixels. It is the correct model for portable GUI code.

### Text Is Pre-Decomposed

`extlib.ccsp.draw` does not implement text shaping, font loading, or glyph rasterization. Those responsibilities belong to `extlib.ccsp.text` (the companion text layout module). By the time a text run reaches the drawing layer, it has been decomposed into a `GlyphRun`: a list of glyph IDs, positions, and advances. The drawing backend renders positioned glyphs; it does not interpret Unicode, apply kerning, or select fonts.

This separation keeps each module small and focused. A backend that renders to a GPU texture uses the same `GlyphRun` type as a backend that outputs a PDF. Neither `extlib.ccsp.draw` nor any backend contains font-shaping logic.

---

## 2. Color Model

### 2.1 Storage: Linear Light Floats

```ferrum
/// A color in linear light RGBA space.
///
/// Components are linear light (not gamma-encoded). For standard dynamic range
/// (SDR) content, components are in [0.0, 1.0]. Values above 1.0 are valid for
/// HDR content and wide-gamut color spaces.
///
/// This is the correct representation for Porter-Duff compositing. Compositing
/// gamma-encoded values produces incorrect results (the "dark halo" artifact).
/// All arithmetic on Color values assumes linear light.
@derive(Debug, Clone, Copy, PartialEq)
struct Color {
    r: f32,
    g: f32,
    b: f32,
    a: f32,
}
```

Linear light is the physically correct representation for mixing and compositing. When two light sources overlap, their energies add linearly. When you alpha-composite two layers, the math requires linear light. Gamma-encoded values (like sRGB bytes) are a storage and transmission compression — not a computation format.

Storing gamma-encoded values in `Color` and compositing them produces the visually well-known "dark middle" artifact in gradients, the "bright halo" around anti-aliased edges, and incorrect alpha blending. `Color` stores linear light and requires callers to convert from sRGB at input time. The conversion happens once, at the boundary, and all downstream code is correct.

### 2.2 Named Constants

```ferrum
impl Color {
    /// Fully opaque white: (1.0, 1.0, 1.0, 1.0) in linear light.
    const WHITE: Color = Color { r: 1.0, g: 1.0, b: 1.0, a: 1.0 }

    /// Fully opaque black: (0.0, 0.0, 0.0, 1.0) in linear light.
    const BLACK: Color = Color { r: 0.0, g: 0.0, b: 0.0, a: 1.0 }

    /// Fully transparent black: (0.0, 0.0, 0.0, 0.0).
    /// Suitable as a "no color" sentinel or clear color.
    const TRANSPARENT: Color = Color { r: 0.0, g: 0.0, b: 0.0, a: 0.0 }
}
```

### 2.3 Input Convenience: sRGB Conversion

Most color values available to programmers are in sRGB: CSS hex codes, design tool color pickers, HTML named colors, and OS color choosers all produce sRGB values. These constructors convert from sRGB to linear light at the point of construction so that all downstream code operates on correct values.

```ferrum
impl Color {
    /// Construct from sRGB bytes.
    ///
    /// Applies the sRGB piecewise transfer function (IEC 61966-2-1) to each
    /// channel. This is NOT a divide-by-255. The sRGB curve is piecewise:
    ///   linear segment for values <= 0.04045 (sRGB-encoded)
    ///   power-law segment (gamma ~2.2) for values above that
    ///
    /// The alpha channel is linear (not gamma-encoded) per convention.
    pub fn from_srgb8(r: u8, g: u8, b: u8, a: u8): Color {
        Color {
            r: srgb_to_linear(r as f32 / 255.0),
            g: srgb_to_linear(g as f32 / 255.0),
            b: srgb_to_linear(b as f32 / 255.0),
            a: a as f32 / 255.0,
        }
    }

    /// Construct from a packed RGBA hex value.
    ///
    /// Format: 0xRRGGBBAA. Example: 0xFF8000FF is fully opaque orange.
    /// Applies the sRGB transfer function to RGB channels; alpha is linear.
    pub fn from_hex(rgba: u32): Color {
        let r = ((rgba >> 24) & 0xFF) as u8
        let g = ((rgba >> 16) & 0xFF) as u8
        let b = ((rgba >>  8) & 0xFF) as u8
        let a = ((rgba >>  0) & 0xFF) as u8
        Color.from_srgb8(r, g, b, a)
    }

    /// Construct directly from linear light float components.
    /// Use this when you already have linear light values (e.g., from HDR sources).
    pub fn from_linear(r: f32, g: f32, b: f32, a: f32): Color {
        Color { r, g, b, a }
    }
}

/// The sRGB piecewise electro-optical transfer function (IEC 61966-2-1).
/// Converts a gamma-encoded sRGB component in [0.0, 1.0] to linear light.
fn srgb_to_linear(v: f32): f32 {
    if v <= 0.04045 {
        v / 12.92
    } else {
        math.powf((v + 0.055) / 1.055, 2.4)
    }
}
```

### 2.4 Manipulation: OKLab and OKLCh

HSL and HSV are the classic "perceptual" color models, but they are not perceptually uniform. Moving 30 degrees around the HSL hue wheel changes perceived brightness dramatically depending on where you start. Two HSL colors with the same L value look very different in brightness. Mixing two HSL colors through the midpoint can produce muddy, desaturated grays or unexpectedly bright intermediate colors.

OKLab (Björn Ottosson, 2020) is a modern perceptual color space designed to fix these problems. It is approximately perceptually uniform: equal distances in OKLab correspond to approximately equal perceived color differences. Gradients mixed through OKLab stay vibrant through the midpoint. CSS Color Level 4 includes `oklab()` and `oklch()` for exactly this reason. Figma uses OKLab for gradient interpolation.

OKLCh is the cylindrical form of OKLab: L (lightness, 0..1), C (chroma, 0..~0.4), h (hue angle, 0..360 degrees). It is the intuitive manipulation space: to make a color lighter, increase L. To make it less saturated, decrease C. To shift the hue, change h.

```ferrum
/// A color in OKLCh space, the cylindrical form of OKLab.
///
/// L: lightness, perceptually uniform, range [0.0, 1.0]
/// C: chroma (colorfulness), range [0.0, ~0.4] for in-gamut sRGB colors
/// h: hue angle in degrees, range [0.0, 360.0)
/// a: alpha, linear, range [0.0, 1.0]
@derive(Debug, Clone, Copy, PartialEq)
struct OklchColor {
    l: f32,
    c: f32,
    h: f32,
    a: f32,
}

impl Color {
    /// Convert to OKLCh for perceptual manipulation.
    pub fn to_oklch(&self): OklchColor { ... }

    /// Construct a Color from OKLCh components.
    /// Values outside the sRGB gamut are clamped on conversion back to linear RGB.
    pub fn from_oklch(l: f32, c: f32, h: f32, a: f32): Color { ... }

    /// Perceptually correct color mixing through OKLab.
    ///
    /// t=0.0 returns a, t=1.0 returns b, t=0.5 returns the perceptual midpoint.
    /// Interpolation is in OKLab (Cartesian), not OKLCh (cylindrical), to avoid
    /// hue-angle wrap-around artifacts.
    pub fn mix(a: Color, b: Color, t: f32): Color { ... }

    /// Increase lightness by amount (0.0..1.0) along the OKLCh L axis.
    /// Clamped to [0.0, 1.0].
    pub fn lighten(&self, amount: f32): Color { ... }

    /// Decrease lightness by amount (0.0..1.0) along the OKLCh L axis.
    /// Clamped to [0.0, 1.0].
    pub fn darken(&self, amount: f32): Color { ... }

    /// Increase chroma by amount along the OKLCh C axis.
    pub fn saturate(&self, amount: f32): Color { ... }

    /// Decrease chroma by amount along the OKLCh C axis. Clamped to zero.
    pub fn desaturate(&self, amount: f32): Color { ... }

    /// Rotate the hue angle by degrees (positive = counterclockwise in OKLCh).
    /// Result is wrapped to [0.0, 360.0).
    pub fn rotate_hue(&self, degrees: f32): Color { ... }

    /// Return a copy of this color with alpha replaced by a.
    pub fn with_alpha(&self, a: f32): Color {
        Color { r: self.r, g: self.g, b: self.b, a }
    }
}
```

**Why OKLCh over HSL/HSV:** HSL is easy to understand but not perceptually uniform. HSL yellow and HSL blue at the same L look nothing like the same brightness. OKLCh is designed so that equal L means approximately equal perceived lightness across all hues. Color manipulations (lighten, darken, saturate) produce predictable results rather than surprising jumps.

**Why OKLCh over CIELAB/CIELCH:** CIELAB has hue linearity problems — mixing blue and yellow goes through a gray center rather than a perceptual midpoint. OKLab was specifically designed to fix hue linearity while remaining computationally simple. The conversion from XYZ to OKLab is a pair of matrix multiplications and cube roots — not expensive.

**Gradient interpolation:** `LinearGradient` and `RadialGradient` (Section 4) interpolate stops through OKLab via `Color::mix`. This is automatic: callers specify stops as `Color` values and get perceptually correct gradients without any additional code.

---

## 3. Geometry Types

All geometry uses `f32` in logical (device-independent) coordinates. The backend maps logical coordinates to physical pixels using the display's scale factor.

```ferrum
/// A 2D point in logical coordinates.
@derive(Debug, Clone, Copy, PartialEq)
struct Point {
    x: f32,
    y: f32,
}

impl Point {
    pub fn new(x: f32, y: f32): Self { Point { x, y } }

    pub fn origin(): Self { Point { x: 0.0, y: 0.0 } }

    pub fn distance(&self, other: Point): f32 {
        math.sqrt((self.x - other.x).powi(2) + (self.y - other.y).powi(2))
    }

    pub fn offset(&self, dx: f32, dy: f32): Self {
        Point { x: self.x + dx, y: self.y + dy }
    }
}

/// A 2D size in logical coordinates.
@derive(Debug, Clone, Copy, PartialEq)
struct Size {
    width:  f32,
    height: f32,
}

impl Size {
    pub fn new(width: f32, height: f32): Self { Size { width, height } }

    pub fn zero(): Self { Size { width: 0.0, height: 0.0 } }

    pub fn area(&self): f32 { self.width * self.height }

    pub fn is_empty(&self): bool { self.width <= 0.0 || self.height <= 0.0 }
}

/// An axis-aligned rectangle in logical coordinates.
///
/// Origin is the top-left corner. Positive y is downward (screen convention).
@derive(Debug, Clone, Copy, PartialEq)
struct Rect {
    origin: Point,
    size:   Size,
}

impl Rect {
    pub fn new(x: f32, y: f32, width: f32, height: f32): Self {
        Rect { origin: Point { x, y }, size: Size { width, height } }
    }

    pub fn from_points(top_left: Point, bottom_right: Point): Self {
        Rect {
            origin: top_left,
            size: Size {
                width:  bottom_right.x - top_left.x,
                height: bottom_right.y - top_left.y,
            },
        }
    }

    pub fn x(&self): f32       { self.origin.x }
    pub fn y(&self): f32       { self.origin.y }
    pub fn width(&self): f32   { self.size.width }
    pub fn height(&self): f32  { self.size.height }
    pub fn max_x(&self): f32   { self.origin.x + self.size.width }
    pub fn max_y(&self): f32   { self.origin.y + self.size.height }

    /// True if this rect contains the given point (inclusive of edges).
    pub fn contains(&self, p: Point): bool {
        p.x >= self.origin.x && p.x <= self.max_x() &&
        p.y >= self.origin.y && p.y <= self.max_y()
    }

    /// True if this rect overlaps with other (touching edges count as intersection).
    pub fn intersects(&self, other: Rect): bool {
        self.origin.x <= other.max_x() && self.max_x() >= other.origin.x &&
        self.origin.y <= other.max_y() && self.max_y() >= other.origin.y
    }

    /// The intersection of this rect and other, or None if they do not overlap.
    pub fn intersection(&self, other: Rect): Option[Rect] {
        let x0 = math.max(self.origin.x, other.origin.x)
        let y0 = math.max(self.origin.y, other.origin.y)
        let x1 = math.min(self.max_x(),  other.max_x())
        let y1 = math.min(self.max_y(),  other.max_y())
        if x1 >= x0 && y1 >= y0 {
            Some(Rect.new(x0, y0, x1 - x0, y1 - y0))
        } else {
            None
        }
    }

    /// The smallest rect that contains both this rect and other.
    pub fn union(&self, other: Rect): Rect {
        let x0 = math.min(self.origin.x, other.origin.x)
        let y0 = math.min(self.origin.y, other.origin.y)
        let x1 = math.max(self.max_x(),  other.max_x())
        let y1 = math.max(self.max_y(),  other.max_y())
        Rect.new(x0, y0, x1 - x0, y1 - y0)
    }

    /// Shrink (positive inset) or expand (negative inset) by amount on all sides.
    pub fn inset(&self, amount: f32): Rect {
        Rect.new(
            self.origin.x + amount,
            self.origin.y + amount,
            self.size.width  - 2.0 * amount,
            self.size.height - 2.0 * amount,
        )
    }

    /// Translate the rect by a point's x and y.
    pub fn offset(&self, p: Point): Rect {
        Rect { origin: Point { x: self.origin.x + p.x, y: self.origin.y + p.y }, size: self.size }
    }

    pub fn is_empty(&self): bool { self.size.is_empty() }
}
```

### 3.1 Transform

`Transform` is a 2D affine transformation represented as a 3x2 matrix (the last row is always `[0, 0, 1]` and is omitted). Transforms are applied in logical-coordinate space before the backend maps to physical pixels.

```ferrum
/// A 2D affine transform stored as a column-major 3x2 matrix.
///
/// The six components [m0..m5] map as:
///   | m0  m2  m4 |
///   | m1  m3  m5 |
///   |  0   0   1 |  (implicit)
///
/// This encodes translation (m4, m5), scale (m0, m3), rotation, and skew.
@derive(Debug, Clone, Copy, PartialEq)
struct Transform {
    m: [f32; 6],
}

impl Transform {
    /// The identity transform: no translation, scale 1, no rotation.
    pub fn identity(): Self {
        Transform { m: [1.0, 0.0, 0.0, 1.0, 0.0, 0.0] }
    }

    /// Translate by (dx, dy).
    pub fn translate(dx: f32, dy: f32): Self {
        Transform { m: [1.0, 0.0, 0.0, 1.0, dx, dy] }
    }

    /// Scale by (sx, sy) relative to the origin.
    pub fn scale(sx: f32, sy: f32): Self {
        Transform { m: [sx, 0.0, 0.0, sy, 0.0, 0.0] }
    }

    /// Rotate by radians counterclockwise (in screen space, positive y is down,
    /// so counterclockwise visually means clockwise in math convention).
    pub fn rotate(radians: f32): Self {
        let cos = math.cos(radians)
        let sin = math.sin(radians)
        Transform { m: [cos, sin, -sin, cos, 0.0, 0.0] }
    }

    /// Concatenate self with other: applies self first, then other.
    pub fn concat(&self, other: Transform): Transform {
        let a = self.m
        let b = other.m
        Transform { m: [
            a[0]*b[0] + a[2]*b[1],
            a[1]*b[0] + a[3]*b[1],
            a[0]*b[2] + a[2]*b[3],
            a[1]*b[2] + a[3]*b[3],
            a[0]*b[4] + a[2]*b[5] + a[4],
            a[1]*b[4] + a[3]*b[5] + a[5],
        ]}
    }

    /// Apply the transform to a point.
    pub fn apply(&self, p: Point): Point {
        let m = self.m
        Point {
            x: m[0]*p.x + m[2]*p.y + m[4],
            y: m[1]*p.x + m[3]*p.y + m[5],
        }
    }

    /// True if this is the identity transform (within floating-point epsilon).
    pub fn is_identity(&self): bool {
        let m = self.m
        (m[0] - 1.0).abs() < 1e-6 && m[1].abs() < 1e-6 &&
        m[2].abs() < 1e-6 && (m[3] - 1.0).abs() < 1e-6 &&
        m[4].abs() < 1e-6 && m[5].abs() < 1e-6
    }
}
```

---

## 4. Paint Types

A `Paint` describes how to fill or stroke a shape. The four variants cover the majority of GUI drawing needs.

```ferrum
/// A fill or stroke style.
@derive(Debug, Clone)]
enum Paint {
    /// A uniform color.
    Solid(Color),

    /// A linear gradient between two points.
    LinearGradient(LinearGradient),

    /// A radial gradient emanating from a center point.
    RadialGradient(RadialGradient),

    /// A tiled or stretched image pattern.
    Pattern(PatternRef),
}

impl From[Color] for Paint {
    fn from(c: Color): Paint { Paint.Solid(c) }
}
```

### 4.1 LinearGradient

```ferrum
/// A linear gradient between two points in local coordinates.
///
/// Color stops are interpolated through OKLab (via Color::mix), producing
/// perceptually correct gradients without desaturated midpoints.
@derive(Debug, Clone)
struct LinearGradient {
    /// Start point of the gradient axis.
    start: Point,
    /// End point of the gradient axis.
    end:   Point,
    /// Color stops, sorted by offset. Must contain at least two stops.
    stops: Vec[ColorStop],
}

impl LinearGradient {
    pub fn new(start: Point, end: Point, stops: Vec[ColorStop]): Self {
        LinearGradient { start, end, stops }
    }
}
```

### 4.2 RadialGradient

```ferrum
/// A radial gradient emanating from a center point.
///
/// At offset 0.0, the gradient has the color of the first stop.
/// At the circle's edge (radius), the gradient has the color of the last stop.
/// Colors between stops are interpolated through OKLab.
@derive(Debug, Clone)
struct RadialGradient {
    center: Point,
    radius: f32,
    stops:  Vec[ColorStop],
}

impl RadialGradient {
    pub fn new(center: Point, radius: f32, stops: Vec[ColorStop]): Self {
        RadialGradient { center, radius, stops }
    }
}
```

### 4.3 ColorStop

```ferrum
/// A single color stop within a gradient.
@derive(Debug, Clone, Copy)]
struct ColorStop {
    /// Position within the gradient, in [0.0, 1.0].
    /// 0.0 is the start (center for radial). 1.0 is the end (edge for radial).
    offset: f32,
    /// The color at this stop, in linear light.
    color:  Color,
}

impl ColorStop {
    pub fn new(offset: f32, color: Color): Self {
        ColorStop { offset, color }
    }
}
```

### 4.4 PatternRef

`PatternRef` is an opaque handle to a backend-owned image used as a paint pattern. It is obtained from `DrawContext::create_image` (Section 9). The pattern tiles or stretches the image to fill the painted region; tiling behavior is backend-defined.

```ferrum
/// An opaque handle to a backend-owned image used as a paint source.
/// Obtained from DrawContext::create_image. Not Clone (backend manages lifetime).
struct PatternRef { ... }
```

---

## 5. DrawBackend Trait

`DrawBackend` is the abstraction boundary. Each display target provides one implementation. Twelve required methods cover the full vocabulary of 2D drawing: frame lifecycle, filled and stroked primitives, image drawing, pre-shaped text, and transform stack management.

```ferrum
/// The low-level drawing interface implemented by each display target.
///
/// Implementors: Wayland backend, X11 backend, framebuffer software rasterizer,
/// GPU backend (Vulkan/Metal/WebGPU), PNG image encoder.
///
/// Callers use DrawContext (Section 6), not DrawBackend directly.
trait DrawBackend {
    /// The current drawable area in logical coordinates.
    /// For a window backend, this is the window's client area.
    /// For an image backend, this is the image dimensions.
    fn frame_size(&self): Size

    /// Begin a new frame. Must be called before any drawing operations.
    /// Returns Err if the backend is in an invalid state (e.g., device lost).
    fn begin_frame(&mut self): Result[(), DrawError]

    /// End the current frame. Commits all drawing operations to the backend's
    /// internal buffer or command list. Must be paired with begin_frame.
    fn end_frame(&mut self): Result[(), DrawError]

    /// Present the completed frame to the display or output target.
    /// For a window backend, this submits the frame to the compositor.
    /// For an image backend, this finalizes the encoded image.
    fn present(&mut self): Result[(), DrawError]

    /// Fill an axis-aligned rectangle with a paint.
    /// clip, if Some, restricts drawing to the intersection of rect and clip.
    fn fill_rect(&mut self, rect: Rect, paint: &Paint, clip: Option[Rect])

    /// Stroke the outline of an axis-aligned rectangle.
    fn stroke_rect(&mut self, rect: Rect, stroke: &Stroke, clip: Option[Rect])

    /// Fill the interior of a path according to fill_rule.
    fn fill_path(&mut self, path: &Path, paint: &Paint, fill_rule: FillRule, clip: Option[Rect])

    /// Stroke the outline of a path.
    fn stroke_path(&mut self, path: &Path, stroke: &Stroke, clip: Option[Rect])

    /// Draw a source image into dst, optionally sampling only src from the image.
    /// src=None means the full image. Scales to fit dst.
    fn draw_image(&mut self, image: &ImageRef, dst: Rect, src: Option[Rect], clip: Option[Rect])

    /// Render a pre-shaped glyph run with the given paint.
    /// Glyph positions in run are in logical coordinates.
    fn draw_glyph_run(&mut self, run: &GlyphRun, paint: &Paint, clip: Option[Rect])

    /// Push a transform onto the transform stack.
    /// Subsequent drawing operations are in the coordinate space of this transform
    /// concatenated with all previously pushed transforms.
    fn push_transform(&mut self, transform: Transform)

    /// Pop the most recently pushed transform.
    /// Panics if the stack is empty (unbalanced push/pop).
    fn pop_transform(&mut self)
}
```

**Design notes on the 12 operations:**

The operations are intentionally minimal. There is no `draw_text` — text is pre-decomposed to `GlyphRun` by the text library before reaching the backend. There is no `draw_circle` — circles are `Path` instances. There is no `set_global_alpha` — alpha belongs in the `Paint`'s `Color`. Each operation is composable and orthogonal.

`fill_rect` and `stroke_rect` are explicit shortcuts even though they could be expressed as paths. Backends can optimize axis-aligned filled rectangles aggressively (GPU: a quad; software: a scanline fill; compositor: a protocol-level rectangle). Making them distinct operations lets backends take the fast path without path analysis.

The clip parameter on every drawing operation, rather than a separate `set_clip` state call, avoids hidden state. A backend that renders from a display list can process each operation independently without tracking clip state. A backend that renders incrementally can avoid unnecessary clip-region push/pop pairs.

---

## 6. DrawContext

`DrawContext` is the high-level wrapper that callers use. It owns a `DrawBackend`, manages the transform stack in cooperation with the backend, and provides ergonomic scoped APIs for clips and transforms.

```ferrum
/// High-level drawing API. Owns a DrawBackend.
///
/// Callers interact with DrawContext, not DrawBackend directly.
/// DrawContext manages frame lifecycle and provides scoped transform/clip helpers.
struct DrawContext {
    backend: Box[dyn DrawBackend],
    // Internal clip stack and current transform are tracked here.
    // Details are implementation-private.
}

impl DrawContext {
    /// Construct a DrawContext from a boxed backend.
    pub fn new(backend: Box[dyn DrawBackend]): Self {
        DrawContext { backend }
    }

    /// Run a complete frame: begin_frame, call f, end_frame, present.
    ///
    /// f receives a mutable reference to this DrawContext.
    /// Returns Err if any backend call fails. If f returns early via ?,
    /// end_frame and present are still called (cleanup is best-effort).
    pub fn frame[F: FnOnce(&mut DrawContext) -> Result[(), DrawError]](
        &mut self, f: F
    ): Result[(), DrawError] {
        self.backend.begin_frame()?
        let result = f(self)
        // Attempt end_frame and present even on error; ignore their errors
        // if f already failed, so the original error propagates.
        let ef = self.backend.end_frame()
        let pr = self.backend.present()
        result.or(ef).or(pr)
    }

    /// The drawable area in logical coordinates.
    pub fn frame_size(&self): Size {
        self.backend.frame_size()
    }

    /// Fill an axis-aligned rectangle.
    pub fn fill_rect(&mut self, rect: Rect, paint: impl Into[Paint]) {
        self.backend.fill_rect(rect, &paint.into(), None)
    }

    /// Stroke the outline of an axis-aligned rectangle.
    pub fn stroke_rect(&mut self, rect: Rect, stroke: Stroke) {
        self.backend.stroke_rect(rect, &stroke, None)
    }

    /// Fill a path according to fill_rule. Default fill rule: NonZero.
    pub fn fill_path(&mut self, path: &Path, paint: impl Into[Paint], fill_rule: FillRule) {
        self.backend.fill_path(path, &paint.into(), fill_rule, None)
    }

    /// Stroke a path.
    pub fn stroke_path(&mut self, path: &Path, stroke: Stroke) {
        self.backend.stroke_path(path, &stroke, None)
    }

    /// Draw an image into dst. src=None uses the full image.
    pub fn draw_image(&mut self, image: &ImageRef, dst: Rect) {
        self.backend.draw_image(image, dst, None, None)
    }

    /// Draw an image cropped to src into dst.
    pub fn draw_image_region(&mut self, image: &ImageRef, dst: Rect, src: Rect) {
        self.backend.draw_image(image, dst, Some(src), None)
    }

    /// Render a pre-shaped glyph run.
    pub fn draw_glyph_run(&mut self, run: &GlyphRun, paint: impl Into[Paint]) {
        self.backend.draw_glyph_run(run, &paint.into(), None)
    }

    /// Execute f with a clip region active.
    ///
    /// All drawing within f is clipped to rect. The clip is automatically removed
    /// when f returns. Clips nest correctly: an inner clip is the intersection
    /// of the inner rect and the outer clip.
    pub fn clip[F: FnOnce(&mut DrawContext)](
        &mut self, rect: Rect, f: F
    ) {
        // Clips are passed per-draw-call in the backend interface.
        // DrawContext tracks the current clip stack and intersects automatically.
        // Implementation detail not exposed to callers.
        f(self)
    }

    /// Execute f with an additional transform applied.
    ///
    /// The transform is pushed before f is called and popped after it returns,
    /// even if f panics (stack is always balanced).
    pub fn with_transform[F: FnOnce(&mut DrawContext)](
        &mut self, t: Transform, f: F
    ) {
        self.backend.push_transform(t)
        f(self)
        self.backend.pop_transform()
    }

    /// Upload CPU-side image data to the backend.
    ///
    /// Returns an ImageRef that can be passed to draw_image or used as a PatternRef.
    /// The backend owns the uploaded data; dropping ImageRef releases backend resources.
    pub fn create_image(&mut self, data: &ImageData): Result[ImageRef, DrawError] {
        // Backend-specific upload (texture upload for GPU, copy for software, etc.)
        ...
    }
}
```

---

## 7. Path Builder

Paths describe arbitrary 2D shapes composed of line segments and curves. Paths are immutable after construction; `PathBuilder` is the mutable construction API.

```ferrum
/// Constructs a Path incrementally.
///
/// A PathBuilder is open (has no Fill or Stroke) — it describes geometry only.
/// Call build() to produce an immutable Path.
struct PathBuilder {
    // Internal segment list; implementation-private.
}

impl PathBuilder {
    /// Create an empty PathBuilder.
    pub fn new(): Self { PathBuilder { ... } }

    /// Move the current point without drawing. Begins a new sub-path.
    /// If the previous sub-path was open, it is left open (not closed).
    pub fn move_to(&mut self, p: Point): &mut Self { ... }

    /// Draw a straight line from the current point to p. Updates current point.
    pub fn line_to(&mut self, p: Point): &mut Self { ... }

    /// Draw a quadratic Bezier curve from the current point to end,
    /// using ctrl as the control point.
    pub fn quad_to(&mut self, ctrl: Point, end: Point): &mut Self { ... }

    /// Draw a cubic Bezier curve from the current point to end,
    /// using c1 and c2 as control points.
    pub fn cubic_to(&mut self, c1: Point, c2: Point, end: Point): &mut Self { ... }

    /// Append an arc of a circle centered at center with the given radius,
    /// sweeping from start_angle to end_angle (radians, measured from positive X axis).
    /// Positive sweep is clockwise in screen coordinates (y-down).
    pub fn arc_to(&mut self, center: Point, radius: f32, start_angle: f32, end_angle: f32): &mut Self { ... }

    /// Close the current sub-path by drawing a straight line back to its start point.
    /// Subsequent line_to or move_to begins a new sub-path.
    pub fn close(&mut self): &mut Self { ... }

    /// Consume the builder and produce an immutable Path.
    pub fn build(self): Path { ... }
}
```

### 7.1 Immutable Path

```ferrum
/// An immutable sequence of path segments.
///
/// Paths are cheap to clone (the segment list is reference-counted).
/// Paths are backend-agnostic; the backend tessellates or strokes them.
@derive(Debug, Clone)
struct Path { ... }
```

### 7.2 Convenience Constructors

These factory functions construct common shapes without requiring a `PathBuilder` at the call site.

```ferrum
impl Path {
    /// An axis-aligned rectangle (four line segments, closed).
    pub fn rect(r: Rect): Path {
        let mut b = PathBuilder.new()
        b.move_to(r.origin)
         .line_to(Point { x: r.max_x(), y: r.y() })
         .line_to(Point { x: r.max_x(), y: r.max_y() })
         .line_to(Point { x: r.x(),     y: r.max_y() })
         .close()
        b.build()
    }

    /// A rectangle with uniformly rounded corners.
    pub fn rounded_rect(r: Rect, corner_radius: f32): Path { ... }

    /// A full circle.
    pub fn circle(center: Point, radius: f32): Path {
        let mut b = PathBuilder.new()
        b.arc_to(center, radius, 0.0, 2.0 * math.PI)
         .close()
        b.build()
    }

    /// An ellipse with radii rx (horizontal) and ry (vertical).
    pub fn ellipse(center: Point, rx: f32, ry: f32): Path { ... }
}
```

### 7.3 FillRule

```ferrum
/// Determines how to fill paths with self-intersecting or nested sub-paths.
@derive(Debug, Clone, Copy, PartialEq)]
enum FillRule {
    /// Fill regions where the winding number is non-zero (SVG/PDF default).
    NonZero,
    /// Fill regions where the winding number is odd (useful for donuts, stars).
    EvenOdd,
}
```

---

## 8. Stroke

`Stroke` fully describes how to stroke a path: not just color, but line width, cap style, join style, and optional dash pattern.

```ferrum
/// A complete stroke style.
@derive(Debug, Clone)
struct Stroke {
    /// The fill used for the stroke itself (usually a solid color).
    paint:       Paint,
    /// Width of the stroke in logical coordinates.
    width:       f32,
    /// How to end open sub-paths.
    line_cap:    LineCap,
    /// How to join consecutive segments.
    line_join:   LineJoin,
    /// Miter limit: controls how far a miter join extends before it is beveled.
    /// Ignored when line_join is not Miter. Standard default: 4.0.
    miter_limit: f32,
    /// Optional dash pattern. None means a solid line.
    dash:        Option[DashPattern],
}

impl Stroke {
    /// Construct a simple solid stroke with default cap/join settings.
    pub fn solid(paint: impl Into[Paint], width: f32): Self {
        Stroke {
            paint:       paint.into(),
            width,
            line_cap:    LineCap.Butt,
            line_join:   LineJoin.Miter,
            miter_limit: 4.0,
            dash:        None,
        }
    }
}

/// Cap style for the ends of open sub-paths.
@derive(Debug, Clone, Copy, PartialEq)
enum LineCap {
    /// Square cap ending exactly at the path endpoint. (SVG: butt)
    Butt,
    /// Semicircular cap centered on the endpoint. Extends by width/2.
    Round,
    /// Square cap centered on the endpoint. Extends by width/2.
    Square,
}

/// Join style for connected path segments.
@derive(Debug, Clone, Copy, PartialEq)
enum LineJoin {
    /// Sharp pointed join. Clipped to miter_limit × width.
    Miter,
    /// Circular join centered on the vertex.
    Round,
    /// Flat join: the outer corner is cut off with a straight line.
    Bevel,
}

/// A dash pattern for stroked paths.
@derive(Debug, Clone)
struct DashPattern {
    /// Alternating lengths of dashes and gaps, in logical coordinates.
    /// Must be non-empty. [5.0, 3.0] means: 5 units on, 3 units off.
    lengths: Vec[f32],
    /// Phase offset into the pattern at the start of the stroke.
    offset:  f32,
}
```

---

## 9. Images

Images flow through two types: `ImageData` holds CPU-side pixels; `ImageRef` is a backend-owned handle after upload.

### 9.1 ImageData

```ferrum
/// CPU-side pixel data, ready for upload to a backend.
struct ImageData {
    width:  u32,
    height: u32,
    format: PixelFormat,
    /// Row-major pixel data. Length must be width * height * bytes_per_pixel(format).
    data:   Vec[u8],
}

impl ImageData {
    pub fn new(width: u32, height: u32, format: PixelFormat, data: Vec[u8]): Result[Self, DrawError] {
        let expected = (width as usize) * (height as usize) * format.bytes_per_pixel()
        if data.len() != expected {
            return Err(DrawError.InvalidPath) // misuse: data length mismatch
        }
        Ok(ImageData { width, height, format, data })
    }
}

/// The pixel format of image data.
@derive(Debug, Clone, Copy, PartialEq)
enum PixelFormat {
    /// 8 bits per channel, linear light, RGBA order. Suitable for images
    /// already linearized (e.g., decoded from a linear-light source).
    Rgba8Unorm,

    /// 8 bits per channel, sRGB gamma-encoded, RGBA order. The most common
    /// format for PNG images and CSS colors. The backend converts to linear
    /// light during upload.
    Rgba8UnormSrgb,

    /// 16 bits per channel, IEEE 754 half-float, linear light, RGBA order.
    /// Used for HDR images and wide-gamut content.
    Rgba16Float,
}

impl PixelFormat {
    pub fn bytes_per_pixel(&self): usize {
        match self {
            PixelFormat.Rgba8Unorm    => 4,
            PixelFormat.Rgba8UnormSrgb => 4,
            PixelFormat.Rgba16Float   => 8,
        }
    }
}
```

### 9.2 ImageRef

```ferrum
/// An opaque handle to a backend-owned uploaded image.
///
/// Obtained from DrawContext::create_image. Dropping an ImageRef releases
/// the backend-side resource (texture, shared memory segment, etc.).
/// ImageRef is not Clone; use Arc[ImageRef] for shared ownership.
struct ImageRef { ... }
```

---

## 10. GlyphRun

`GlyphRun` is defined by `extlib.ccsp.text` and passed through to `DrawBackend::draw_glyph_run`. The drawing layer treats it as an opaque input. No font logic appears in `extlib.ccsp.draw`.

```ferrum
/// A pre-shaped sequence of glyphs ready for rendering.
///
/// Produced by extlib.ccsp.text after Unicode shaping, kerning, and font selection.
/// The drawing layer renders positioned glyphs; it does not reshape or re-kern.
@derive(Debug, Clone)
struct GlyphRun {
    /// Identifies the font face. Opaque to the drawing layer; the backend
    /// resolves this to its internal font representation.
    font_id: FontId,

    /// Rendered size in logical points.
    size: f32,

    /// The glyphs to render, in visual order.
    glyphs: Vec[GlyphInstance],
}

/// A single glyph instance within a GlyphRun.
@derive(Debug, Clone, Copy)
struct GlyphInstance {
    /// Backend-specific glyph identifier within the font face.
    glyph_id: u32,

    /// Position of the glyph origin (baseline-left) in logical coordinates.
    position: Point,

    /// Horizontal advance to the next glyph's origin, in logical coordinates.
    advance: f32,
}

/// An opaque font face identifier. Produced by extlib.ccsp.text.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)
struct FontId { id: u32 }
```

**Layering invariant:** No code in `extlib.ccsp.draw` loads fonts, applies Unicode BiDi, selects typeface variants, applies kerning pairs, or resolves font fallback chains. All of that happens in `extlib.ccsp.text`. The drawing layer receives the fully resolved glyph list and renders it. This invariant keeps the drawing layer portable across font systems (FreeType, CoreText, DirectWrite, web fonts) without any change to the drawing API.

---

## 11. Error Types

```ferrum
/// An error from a drawing operation.
@derive(Debug)]
enum DrawError {
    /// The backend returned an implementation-specific error.
    /// The string contains a human-readable description for diagnostics.
    BackendError(String),

    /// The image data is too large for this backend.
    /// Common limits: GPU texture size (4096×4096 or 8192×8192 depending on hardware).
    ImageTooLarge,

    /// A PathBuilder was used incorrectly (e.g., line_to without a prior move_to).
    InvalidPath,

    /// The underlying device has been lost (GPU reset, Wayland compositor disconnect).
    /// The application must destroy and recreate the backend.
    DeviceLost,
}

impl fmt.Display for DrawError {
    fn fmt(&self, f: &mut fmt.Formatter): fmt.Result {
        match self {
            DrawError.BackendError(msg) => write(f, "backend error: {}", msg)
            DrawError.ImageTooLarge     => write(f, "image too large for backend")
            DrawError.InvalidPath       => write(f, "invalid path construction")
            DrawError.DeviceLost        => write(f, "device lost; backend must be recreated")
        }
    }
}

impl core.error.Error for DrawError {}
```

`DeviceLost` warrants special attention. GPU resets and Wayland compositor reconnection are real events in long-running GUI applications. When `DeviceLost` is returned, all `ImageRef` handles are invalid and the backend must be recreated from scratch. The application is responsible for re-uploading textures and re-initializing backend state. This is not recoverable by retrying the last draw call.

---

## 12. Example Usage

### 12.1 Solid Color Box

```ferrum
use extlib.ccsp.draw.{DrawContext, Color, Rect}

fn draw_ui(ctx: &mut DrawContext) {
    let bg = Color.from_srgb8(30, 30, 30, 255)
    ctx.fill_rect(ctx.frame_size().into(), bg)

    let box_color = Color.from_hex(0x4A90E2FF)  // blue
    ctx.fill_rect(Rect.new(50.0, 50.0, 200.0, 100.0), box_color)
}
```

### 12.2 Gradient Background

```ferrum
use extlib.ccsp.draw.{DrawContext, Color, ColorStop, LinearGradient, Paint, Point, Rect}

fn draw_gradient_background(ctx: &mut DrawContext) {
    let size = ctx.frame_size()
    let gradient = LinearGradient.new(
        Point.new(0.0, 0.0),
        Point.new(0.0, size.height),
        Vec.from([
            ColorStop.new(0.0, Color.from_hex(0x1A1A2EFF)),
            ColorStop.new(1.0, Color.from_hex(0x16213EFF)),
        ])
    )
    ctx.fill_rect(Rect.new(0.0, 0.0, size.width, size.height), gradient)
}
```

### 12.3 Rounded Rect Button

```ferrum
use extlib.ccsp.draw.{DrawContext, Color, Path, Rect, Stroke, LineCap, LineJoin}

fn draw_button(ctx: &mut DrawContext, rect: Rect, label_run: &GlyphRun) {
    let fill_color  = Color.from_srgb8(72, 144, 226, 255)
    let hover_color = fill_color.lighten(0.1)     // perceptually 10% lighter
    let border      = Color.from_srgb8(50, 110, 190, 255)

    let path = Path.rounded_rect(rect, 8.0)

    // Fill
    ctx.fill_path(&path, fill_color, FillRule.NonZero)

    // Border stroke
    let stroke = Stroke {
        paint:       Paint.Solid(border),
        width:       1.5,
        line_cap:    LineCap.Butt,
        line_join:   LineJoin.Miter,
        miter_limit: 4.0,
        dash:        None,
    }
    ctx.stroke_path(&path, stroke)

    // Label
    let label_color = Color.WHITE
    ctx.draw_glyph_run(label_run, label_color)
}
```

### 12.4 Perceptual Color Mixing

```ferrum
use extlib.ccsp.draw.Color

fn palette_between(a: Color, b: Color): [Color; 5] {
    // Five-step gradient through OKLab. The midpoint stays vibrant.
    [
        Color.mix(a, b, 0.0),
        Color.mix(a, b, 0.25),
        Color.mix(a, b, 0.5),
        Color.mix(a, b, 0.75),
        Color.mix(a, b, 1.0),
    ]
}

fn main() ! IO {
    // Mix orange and blue through OKLab — the midpoint is a warm neutral,
    // not the muddy brown HSL would produce.
    let orange = Color.from_hex(0xFF8C00FF)
    let blue   = Color.from_hex(0x1E90FFFF)
    let steps  = palette_between(orange, blue)
    // steps[2] is a warm beige, not a desaturated gray
}
```

### 12.5 Image Upload and Draw

```ferrum
use extlib.ccsp.draw.{DrawContext, ImageData, PixelFormat, Rect}

fn load_and_draw_image(ctx: &mut DrawContext, png_pixels: Vec[u8], w: u32, h: u32)
    : Result[(), DrawError]
{
    let data = ImageData.new(w, h, PixelFormat.Rgba8UnormSrgb, png_pixels)?
    let image = ctx.create_image(&data)?
    ctx.draw_image(&image, Rect.new(10.0, 10.0, w as f32, h as f32))
    Ok(())
}
```

### 12.6 Scoped Transform and Clip

```ferrum
use extlib.ccsp.draw.{DrawContext, Transform, Rect, Color, Path}

fn draw_rotated_panel(ctx: &mut DrawContext, center_x: f32, center_y: f32) {
    // Translate to center, rotate 45 degrees, translate back
    let t = Transform.translate(center_x, center_y)
             .concat(Transform.rotate(math.PI / 4.0))
             .concat(Transform.translate(-50.0, -50.0))

    ctx.with_transform(t, fn(ctx: &mut DrawContext) {
        // This fill_rect is in the rotated coordinate space
        ctx.fill_rect(Rect.new(0.0, 0.0, 100.0, 100.0), Color.from_hex(0xE74C3CFF))
    })
}

fn draw_clipped_content(ctx: &mut DrawContext) {
    let clip_region = Rect.new(20.0, 20.0, 300.0, 200.0)
    ctx.clip(clip_region, fn(ctx: &mut DrawContext) {
        // Only pixels within clip_region are drawn
        ctx.fill_rect(Rect.new(0.0, 0.0, 1000.0, 1000.0), Color.from_hex(0x2ECC71FF))
    })
}
```

### 12.7 Rendering to Multiple Backends

```ferrum
use extlib.ccsp.draw.{DrawContext, DrawBackend}
// hypothetical backend constructors:
use extlib.ccsp.draw.backends.{WaylandBackend, PngBackend}

fn render_frame(ctx: &mut DrawContext) {
    ctx.frame(fn(ctx: &mut DrawContext): Result[(), DrawError] {
        draw_gradient_background(ctx)
        draw_button(ctx, Rect.new(100.0, 200.0, 120.0, 40.0), &make_label())
        Ok(())
    }).expect("frame failed")
}

fn main() ! IO + Alloc {
    // Render to a live Wayland window
    let mut wayland_ctx = DrawContext.new(Box.new(WaylandBackend.new()))
    render_frame(&mut wayland_ctx)

    // Render the identical frame to a PNG file for testing
    let mut png_ctx = DrawContext.new(Box.new(PngBackend.new(800, 600)))
    render_frame(&mut png_ctx)
    // png_ctx.backend now holds the encoded PNG bytes
}
```

The key point: `render_frame` is written once. It calls only `DrawContext` methods. It does not contain any Wayland calls, any PNG calls, or any backend conditionals. The backend selection is entirely at the `DrawContext.new(...)` call site.

### 12.8 Dashed Path

```ferrum
use extlib.ccsp.draw.{DrawContext, Path, Stroke, Paint, Color, DashPattern, LineCap, LineJoin, Point}

fn draw_dashed_border(ctx: &mut DrawContext, rect: Rect) {
    let path = Path.rect(rect)
    let stroke = Stroke {
        paint:       Paint.Solid(Color.from_hex(0xAAAAAA80)),
        width:       1.0,
        line_cap:    LineCap.Round,
        line_join:   LineJoin.Round,
        miter_limit: 4.0,
        dash:        Some(DashPattern { lengths: Vec.from([4.0, 4.0]), offset: 0.0 }),
    }
    ctx.stroke_path(&path, stroke)
}
```

---

## 13. Dependencies

`extlib.ccsp.draw` depends only on:

| Module | Used for |
|---|---|
| `alloc.vec` | `Vec[T]` for path segments, gradient stops, image pixel data |
| `alloc.string` | `String` in `DrawError::BackendError` |
| `alloc.boxed` | `Box[dyn DrawBackend]` in `DrawContext` |
| `math` | `sin`, `cos`, `sqrt`, `pow`, `PI` for geometry and color math |
| `fmt` | `Display`, `Debug` for `DrawError` |
| `core.error` | `Error` trait for `DrawError` |

`extlib.ccsp.draw` does **not** depend on:

- `io` or `fs` — no file I/O (image encoding belongs to backend crates)
- `net` — no network operations
- `async` — all drawing calls are synchronous; async frame scheduling belongs to the application
- `sys` or `posix` — no OS calls (platform calls are in backend implementations, not this crate)
- `extlib.ccsp.text` — text shaping is a caller responsibility; `GlyphRun` is a data type only

This intentional narrowness makes `extlib.ccsp.draw` foundational: other extlib modules can depend on it without pulling in transitive dependencies. The text layout module (`extlib.ccsp.text`), the widget toolkit (`extlib.ccsp.widgets`), and chart libraries all depend on `extlib.ccsp.draw`, not the other way around.

The effect annotations on `DrawContext` methods are absent by design: individual draw calls (`fill_rect`, `stroke_path`, etc.) are not `! IO` at the type level — they write into an internal buffer, not to a system resource directly. The `! IO` effect surfaces at frame boundaries: `begin_frame`, `end_frame`, and `present` communicate with the display system and are where the `! IO` effect would appear in a complete production implementation. The `frame(f)` wrapper carries `! IO` implicitly through its backend calls.
