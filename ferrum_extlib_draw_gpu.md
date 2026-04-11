# Ferrum Extended Library — `draw_gpu` Module

**Module path:** `extlib::draw_gpu`
**Provided by:** `lib_ccsp_draw_gpu`
**Depends on:** `extlib::draw` (DrawBackend trait), `extlib::font` (glyph rasterization), `extlib::draw_wayland` / `extlib::draw_x11` (surface creation); `stdlib::alloc`, `stdlib::sys`
**External C dependencies:** Vulkan headers (1.2+), EGL/GLES3 headers, libtess2

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [GpuBackend](#2-gpubackend)
3. [Vulkan Path](#3-vulkan-path)
4. [OpenGL ES 3.x Fallback Path](#4-opengl-es-3x-fallback-path)
5. [Glyph Atlas Management](#5-glyph-atlas-management)
6. [Path Tessellation](#6-path-tessellation)
7. [Texture Cache](#7-texture-cache)
8. [Frame Lifecycle — Vulkan](#8-frame-lifecycle--vulkan)
9. [Surface Types](#9-surface-types)
10. [Debugging and Profiling](#10-debugging-and-profiling)
11. [Error Types](#11-error-types)
12. [Example Usage](#12-example-usage)
13. [Dependencies](#13-dependencies)

---

## 1. Overview and Rationale

### Why GPU rendering

Software rasterization (the `draw_fb` backend) is correct and portable, but it scales
poorly with UI complexity. On a 4K display rendering a document with dozens of layered
text runs, drop shadows, rounded rectangles, and animated transitions, the CPU becomes
the bottleneck: rasterizing each pixel in software on every frame is simply too much
work for 60 fps.

GPU rendering solves this structurally. The GPU contains hundreds to thousands of
shader cores that rasterize triangles in parallel. Once geometry has been tessellated
on the CPU and uploaded to GPU memory, the actual pixel fill — the expensive part —
runs on the GPU asynchronously while the CPU prepares the next frame. Hardware
compositing layers mean that a scrolling region does not require re-rasterizing the
static background. Animation at 60 fps or 120 fps on high-refresh displays becomes
feasible even for complex UIs.

The `draw_gpu` backend achieves this by implementing `DrawBackend` (from `extlib::draw`)
entirely in terms of GPU commands. All path geometry is tessellated to triangles on the
CPU, uploaded once per frame into a per-frame vertex buffer, and then drawn by a small
set of cached shader pipelines. Text is rendered via a glyph atlas: glyphs are
CPU-rasterized once (by `extlib::font`) and uploaded as a texture; subsequent uses are
pure GPU quad draws at negligible cost.

### Vulkan as the primary backend

Vulkan (Khronos, released 2016) is the primary GPU API. It is available on:

- Linux — via Mesa (RADV for AMD, ANV for Intel, NVK for NVIDIA, plus many others)
- Windows — vendor drivers and ANGLE
- macOS and iOS — via MoltenVK (Vulkan over Metal)
- Android — native in Android 7.0+
- Embedded Linux — Mesa on ARM Mali, Imagination PowerVR, Vivante

Vulkan exposes the GPU explicitly. Command buffers, render passes, pipeline state
objects, descriptor sets, and synchronization are all visible to the application.
This explicitness is a feature, not a burden: it allows the backend to eliminate the
implicit state-machine overhead that OpenGL imposes, batch all draw calls into a single
command buffer per frame, pipeline shader compilation, and control memory layout. The
result is lower and more predictable frame latency.

### OpenGL ES 3.x as the fallback

OpenGL ES 3.0 (GLES3) is available on hardware that predates Vulkan support: Raspberry
Pi 3 (Broadcom VideoCore IV), older Android devices (pre-Android 7), and some
industrial SoCs with GLES3 drivers that predate the Khronos Vulkan push. GLES3
provides VAOs, VBOs, UBOs, instanced rendering, and sRGB framebuffers — enough to
implement the same drawing model as the Vulkan path.

### No OpenGL 2.1 path

OpenGL 2.1 (released 2006) is explicitly not supported. The reasons are structural:

- **No VAOs.** Without Vertex Array Objects, vertex attribute setup must be repeated
  every draw call. This is performance-prohibitive for a UI renderer.
- **No compute shaders.** Future backend features (GPU-side path flattening, glyph SDF
  generation) require compute. GL 2.1 has no compute stage.
- **No sRGB framebuffer objects.** Correct color handling (linear light compositing,
  sRGB output) requires `GL_FRAMEBUFFER_SRGB` from OpenGL 3.0 / EXT_framebuffer_sRGB.
  Without it the backend would need to do manual gamma correction in every shader, which
  is error-prone and slower.
- **Too old in practice.** Hardware limited to GL 2.1 is either too slow for a UI
  renderer, or has a GLES3 driver available alongside the legacy GL 2.1 driver.

If `GpuBackend::auto()` cannot find Vulkan and cannot find GLES3, it returns
`GpuError::NoSuitableDevice`. The caller may then fall back to `draw_fb`.

---

## 2. GpuBackend

`GpuBackend` is the top-level type. It implements `DrawBackend` from `extlib::draw` and
manages all GPU resources for the lifetime of a rendering surface.

```ferrum
use extlib::draw_gpu::{GpuBackend, GpuConfig, GpuError, VsyncMode}
use extlib::draw_gpu::surface::{VulkanSurface, GlesSurface, AnySurface}

pub type GpuBackend
```

### 2.1 Constructors

```ferrum
impl GpuBackend {
    // Create a Vulkan-backed GpuBackend from a platform-provided VkSurfaceKHR handle.
    //
    // Selects the first physical device that supports the required features:
    // geometry shaders are not required; compute is optional at this stage.
    // Fails with GpuError::NoSuitableDevice when no Vulkan-capable device is found.
    //
    // Effects: IO (enumerates Vulkan instance, allocates GPU memory)
    pub fn new_vulkan(
        surface: VulkanSurface,
        config:  GpuConfig,
    ): Result[GpuBackend, GpuError] ! IO

    // Create an OpenGL ES 3.x-backed GpuBackend from an EGL surface handle.
    //
    // Requires EGL 1.4+ and a context supporting GLES 3.0.
    // Fails with GpuError::NoSuitableDevice when EGL context creation fails or the
    // driver does not expose GLES 3.0.
    //
    // Effects: IO (creates EGL context, compiles shaders)
    pub fn new_gles3(
        surface: GlesSurface,
        config:  GpuConfig,
    ): Result[GpuBackend, GpuError] ! IO

    // Attempt Vulkan first; fall back to GLES3 if Vulkan is unavailable.
    //
    // The AnySurface enum carries whichever surface type the platform backend
    // created. If the surface is Vulkan(v) but Vulkan init fails, auto() does
    // not retry with GLES3 from the same surface — the caller must provide a
    // GlesSurface explicitly for the GLES3 path. When the surface is
    // AnySurface::Vulkan and Vulkan init succeeds, GLES3 is never attempted.
    //
    // Returns GpuError::NoSuitableDevice only when every applicable path fails.
    //
    // Effects: IO
    pub fn auto(
        surface: AnySurface,
        config:  GpuConfig,
    ): Result[GpuBackend, GpuError] ! IO
}
```

### 2.2 `GpuConfig`

```ferrum
pub type GpuConfig {
    // MSAA sample count. Must be a power of two. 1 = no MSAA.
    // Vulkan: checked against VkSampleCountFlags; unsupported values fall back
    // to the next lower supported count.
    // GLES3: checked against GL_MAX_SAMPLES; same fallback.
    pub msaa_samples: u32,

    // Request an HDR swapchain when the display reports HDR capability.
    // Vulkan: uses VK_FORMAT_R16G16B16A16_SFLOAT when true and the display supports it.
    // GLES3: no HDR support; this field is ignored on the GLES3 path.
    pub hdr: bool,

    // Swapchain presentation timing mode.
    pub vsync: VsyncMode,

    // Maximum GPU memory to dedicate to the texture cache (uploaded ImageRef values).
    // When the cache exceeds this limit, least-recently-used textures are evicted.
    // Does not include the glyph atlas, which is managed separately.
    pub max_texture_cache_bytes: u64,
}

impl GpuConfig {
    // Sensible defaults: 4x MSAA, no HDR, vsync on, 256 MiB texture cache.
    pub fn default(): Self
}
```

### 2.3 `VsyncMode`

```ferrum
pub enum VsyncMode {
    // Present immediately; no wait for vblank. Tearing possible.
    // Vulkan: VK_PRESENT_MODE_IMMEDIATE_KHR
    // GLES3: eglSwapInterval(0)
    Off,

    // Wait for vblank. No tearing. Frame rate capped to display refresh rate.
    // Vulkan: VK_PRESENT_MODE_FIFO_KHR (always available per spec)
    // GLES3: eglSwapInterval(1)
    On,

    // Present immediately if the previous frame is late; otherwise wait for
    // vblank. Reduces stutter under load without consistent tearing.
    // Vulkan: VK_PRESENT_MODE_FIFO_RELAXED_KHR (falls back to FIFO if unsupported)
    // GLES3: not available; treated as On.
    Adaptive,
}
```

### 2.4 `DrawBackend` implementation

```ferrum
impl DrawBackend for GpuBackend {
    fn begin_frame(&mut self): Result[(), GpuError] ! IO
    fn end_frame(&mut self):   Result[(), GpuError] ! IO
    fn present(&mut self):     Result[(), GpuError] ! IO

    fn fill_path(
        &mut self,
        path:       &Path,
        fill_rule:  FillRule,
        paint:      &Paint,
    ): Result[(), GpuError] ! IO

    fn stroke_path(
        &mut self,
        path:       &Path,
        stroke:     &StrokeStyle,
        paint:      &Paint,
    ): Result[(), GpuError] ! IO

    fn draw_image(
        &mut self,
        image:  &ImageRef,
        dst:    Rect,
        src:    Rect,
        filter: ImageFilter,
    ): Result[(), GpuError] ! IO

    fn draw_glyph_run(
        &mut self,
        run:    &GlyphRun,
        origin: Point,
        paint:  &Paint,
    ): Result[(), GpuError] ! IO
}
```

---

## 3. Vulkan Path

### 3.1 Command buffer model

One `VkCommandBuffer` is recorded per frame. All draw calls issued between
`begin_frame()` and `end_frame()` append commands to this buffer; no GPU work executes
until the buffer is submitted at `end_frame()`. This eliminates the per-draw CPU-GPU
round trips that plague immediate-mode GL code.

The backend maintains two or three command buffers in rotation (double or triple
buffering controlled by swapchain image count). A `VkFence` per frame-in-flight
prevents the CPU from overwriting a command buffer that the GPU is still consuming.

```
Frame N:   CPU records → submit → GPU executes
Frame N+1: CPU records → submit → GPU executes
           (fence ensures N's buffer is free before N+1 reuses it)
```

### 3.2 Render pass

A single `VkRenderPass` with one subpass is used. Layout:

| Attachment | Format | Load op | Store op |
|---|---|---|---|
| Color | `VK_FORMAT_B8G8R8A8_SRGB` (standard) or `VK_FORMAT_R16G16B16A16_SFLOAT` (HDR) | `CLEAR` | `STORE` |
| Depth/stencil | `VK_FORMAT_D24_UNORM_S8_UINT` (when MSAA > 1) | `CLEAR` | `DONT_CARE` |
| MSAA resolve (when MSAA > 1) | Same as color | N/A | `STORE` |

The sRGB swapchain format (`VK_FORMAT_B8G8R8A8_SRGB`) means the GPU applies the sRGB
transfer function at scan-out automatically. Shaders write linear light values; no
manual gamma correction is needed in any shader.

When HDR is enabled and the display reports `VK_COLOR_SPACE_HDR10_ST2084_EXT` or
`VK_COLOR_SPACE_EXTENDED_SRGB_LINEAR_EXT`, the backend selects
`VK_FORMAT_R16G16B16A16_SFLOAT`. Tone mapping is the caller's responsibility; the
backend produces linear scRGB values.

### 3.3 Pipeline state objects

Pipeline state objects (PSOs) are created at backend initialization and cached for the
lifetime of the backend. The cache key is `(DrawMode, BlendMode)`.

```ferrum
pub enum DrawMode {
    Fill,
    Stroke,
    Image,
    GlyphGrayscale,
    GlyphSubpixel,
}

pub enum BlendMode {
    // Standard Porter-Duff source-over compositing.
    SourceOver,

    // Replace destination pixels entirely.
    Copy,

    // Multiply source and destination. Useful for shadows.
    Multiply,

    // Screen blend mode.
    Screen,
}
```

Each `(DrawMode, BlendMode)` pair maps to one `VkPipeline`. All pipelines share a
single `VkPipelineLayout` with two descriptor set layouts: set 0 for per-frame
uniforms (transform matrix, viewport), set 1 for per-draw resources (texture/glyph
atlas sampler).

### 3.4 `VkPipelineCache` persistence

On first run, `VkPipelineCache` is created empty. On clean shutdown, the cache blob is
written to disk (path: `$XDG_CACHE_HOME/ferrum/draw_gpu/pipeline_cache.bin` on Linux,
platform equivalent elsewhere). On subsequent runs, the blob is loaded and passed to
`vkCreatePipelineCache`. This eliminates shader recompilation time on startup, which
matters most on embedded hardware with slow CPUs.

The cache blob is treated as opaque. If the file is corrupt or was produced by a
different driver version (detected via `VkPhysicalDeviceProperties::pipelineCacheUUID`),
it is discarded and the cache rebuilt from scratch.

### 3.5 Vertex buffer

Dynamic geometry (tessellated paths, image quads, glyph quads) is uploaded into a
per-frame vertex buffer. The buffer is allocated once with
`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` and
persistently mapped. On each frame the write cursor resets to zero; draw calls append
vertices sequentially. The buffer is sized to `GpuConfig`-derived capacity (default:
4 MiB per frame-in-flight).

```ferrum
pub type Vertex {
    // Position in normalized device coordinates.
    pub pos:      [f32; 2],

    // UV coordinates into the glyph atlas or image texture.
    // (0.0, 0.0) for solid-color draws.
    pub uv:       [f32; 2],

    // RGBA color in linear light, premultiplied alpha.
    pub color:    [f32; 4],
}
```

### 3.6 Glyph atlas — Vulkan

The glyph atlas is a `VkImage` allocated at backend initialization:

- Format: `VK_FORMAT_R8_UNORM` for grayscale glyphs, `VK_FORMAT_R8G8B8A8_UNORM` for
  subpixel (RGB) glyphs. Two separate atlas images are maintained.
- Dimensions: 2048×2048 initially; doubled (up to `VkPhysicalDeviceLimits::maxImageDimension2D`)
  when the packer is full and eviction does not free enough space.
- A `VkDescriptorSet` is bound once per frame via `vkCmdBindDescriptorSets` before the
  glyph draw calls, referencing a combined image sampler with `VK_FILTER_LINEAR`.

Atlas updates (new glyph uploads) go through a staging buffer:
1. Write glyph bitmap into staging buffer (host-visible).
2. Record `vkCmdCopyBufferToImage` into the current command buffer.
3. Insert a pipeline barrier transitioning the image from transfer-dst to shader-read
   layout.

### 3.7 Frame-in-flight tracking

```ferrum
// Conceptual internal structure — not public API
type FrameSlot {
    command_buffer: VkCommandBuffer,
    fence:          VkFence,          // signaled when GPU finishes this frame
    vertex_buffer:  MappedBuffer,
    staging_buffer: MappedBuffer,
}
```

`begin_frame()` waits on the fence for the oldest in-flight slot before reusing it,
then calls `vkResetCommandBuffer`. The fence is reset and re-submitted with
`vkQueueSubmit` at `end_frame()`.

---

## 4. OpenGL ES 3.x Fallback Path

### 4.1 Geometry setup

Each draw call uses a VAO (Vertex Array Object) that captures the vertex attribute
layout. One shared VAO is created at initialization and rebound at each draw. Geometry
is uploaded into a single VBO (Vertex Buffer Object) using
`GL_DYNAMIC_DRAW` (re-uploaded each frame, same as the Vulkan vertex buffer).

Per-frame uniform data (transform matrix, viewport size, time) is stored in a UBO
(Uniform Buffer Object):

```glsl
// GLSL ES 3.0 — embedded as a Ferrum compile-time constant
layout(std140) uniform PerFrame {
    mat4 transform;
    vec2 viewport_size;
    float time;
};
```

### 4.2 sRGB framebuffer

`GL_FRAMEBUFFER_SRGB` is enabled for the default framebuffer. Shaders write linear
light RGBA values; the driver applies the sRGB transfer function before writing to the
display. This matches the Vulkan path behavior exactly.

When rendering to an intermediate framebuffer object (e.g., for layer compositing),
`GL_SRGB8_ALPHA8` is used as the internal format. `GL_FRAMEBUFFER_SRGB` is toggled
appropriately.

### 4.3 Instanced rendering

Glyph runs and arrays of axis-aligned rectangles are batched using instanced rendering
(`glDrawArraysInstanced`). Each instance encodes one glyph quad or one rect via a per-
instance VBO:

```ferrum
pub type GlyphInstance {
    // Top-left corner of the glyph quad in screen coordinates.
    pub screen_pos:  [f32; 2],

    // UV rectangle in the atlas (min_u, min_v, max_u, max_v).
    pub atlas_uv:    [f32; 4],

    // Glyph color in linear light, premultiplied alpha.
    pub color:       [f32; 4],
}
```

### 4.4 Shader compilation

Shaders are authored in a subset of GLSL that can be compiled to SPIR-V via
`glslang`. The SPIR-V is then transpiled to GLSL ES 3.0 via `naga` (or `spirv-cross`
where naga lacks coverage). The GLSL ES 3.0 source is embedded as a compile-time
`&str` constant in the Ferrum binary.

At backend initialization, `glCreateShader` / `glCompileShader` / `glLinkProgram` runs
on the embedded GLSL. Compilation errors return `GpuError::ShaderCompileFailed` with
the driver's info log. In correct builds this error path is never taken — it exists
as a defensive check for driver bugs.

### 4.5 Tessellator

The GLES3 path uses the same CPU-side tessellator as the Vulkan path (libtess2). The
tessellated triangle list is uploaded into the shared VBO. There is no difference in
geometry between the two paths; only the command recording API differs.

---

## 5. Glyph Atlas Management

### 5.1 Overview

The glyph atlas is a packed texture of rasterized glyph bitmaps. It eliminates
repeated CPU rasterization of the same glyph at the same size: once a glyph is
uploaded, all subsequent uses are pure GPU quad draws referencing UV coordinates into
the atlas.

```ferrum
pub type GlyphAtlas {
    // The GPU texture (opaque handle valid for the backend's lifetime).
    pub texture: GpuTexture,

    // Rectangle packer tracking free regions within the atlas.
    packer: RectPacker,

    // LRU map: (glyph_id, font_id, size_px, subpixel_offset) → GlyphAtlasEntry
    lru: LruMap[GlyphKey, GlyphAtlasEntry],

    // Total bytes currently allocated in the atlas.
    used_bytes: u64,
}

pub type GlyphKey {
    pub glyph_id:        u32,
    pub font_id:         u64,
    pub size_px:         u16,
    pub subpixel_offset: u8,   // 0..3, for subpixel positioning
}

pub type GlyphAtlasEntry {
    // UV rectangle within the atlas texture (normalized 0.0..1.0).
    pub uv_min: [f32; 2],
    pub uv_max: [f32; 2],

    // Glyph metrics needed to position the quad.
    pub advance_x:  f32,
    pub bearing_x:  f32,
    pub bearing_y:  f32,
    pub width_px:   u16,
    pub height_px:  u16,
}
```

### 5.2 Upload path

```ferrum
impl GlyphAtlas {
    // Upload a rasterized glyph bitmap into the atlas.
    //
    // bitmap: row-major, top-down.
    //   Grayscale glyphs: 1 byte per pixel (GL_R8 / VK_FORMAT_R8_UNORM).
    //   Subpixel glyphs:  3 bytes per pixel (RGB, GL_RGB8 / VK_FORMAT_R8G8B8_UNORM).
    //
    // If the atlas is full, LRU entries are evicted until enough space is free.
    // If a single glyph is too large for the atlas (>256×256), returns
    // GpuError::TextureTooLarge.
    //
    // Effects: IO (may trigger GPU texture upload)
    pub fn upload_glyph(
        &mut self,
        key:     GlyphKey,
        bitmap:  &[u8],
        metrics: GlyphMetrics,
    ): Result[GlyphAtlasEntry, GpuError] ! IO

    // Look up a cached entry without uploading.
    // Returns None when the glyph is not in the atlas (must be rasterized first).
    pub fn lookup(&self, key: &GlyphKey): Option[&GlyphAtlasEntry]
}
```

### 5.3 Rasterization delegation

`GlyphAtlas` does not rasterize glyphs itself. The caller (typically `DrawBackend::draw_glyph_run`)
queries `extlib::font` for a rasterized bitmap:

```ferrum
// Pseudocode — internal to GpuBackend::draw_glyph_run
fn draw_glyph_run(&mut self, run: &GlyphRun, origin: Point, paint: &Paint) ! IO {
    for glyph in run.glyphs() {
        let key = GlyphKey { glyph_id: glyph.id, font_id: run.font_id(), ... }
        let entry = match self.glyph_atlas.lookup(&key) {
            Some(e) => {
                self.frame_stats.glyph_cache_hits += 1
                e.clone()
            }
            None => {
                self.frame_stats.glyph_cache_misses += 1
                let bitmap = extlib::font::rasterize(glyph.id, run.font(), run.size_px())?
                self.glyph_atlas.upload_glyph(key, &bitmap.data, bitmap.metrics)?
            }
        }
        self.emit_glyph_quad(glyph, &entry, origin, paint)
    }
}
```

### 5.4 Grayscale and subpixel separation

Grayscale glyphs and subpixel (LCD) glyphs require different shader paths:
- Grayscale: alpha-blend the glyph color over the destination using the R8 value as alpha.
- Subpixel: per-channel alpha blend using the RGB values as separate alpha masks for
  R, G, B channels (FreeType-style LCD rendering). Requires compositing over the
  background color, so the background must be known at draw time.

The two glyph modes use distinct `DrawMode` variants (`GlyphGrayscale` and
`GlyphSubpixel`) and therefore distinct PSOs. The glyph atlas maintains two textures
— one R8 and one RGB8 — so both can be bound simultaneously without switching
descriptor sets between glyphs in the same run.

---

## 6. Path Tessellation

### 6.1 Overview

All filled and stroked paths are converted to triangle geometry on the CPU before being
uploaded to the GPU. The tessellator runs synchronously during the draw call and the
result is written directly into the current frame's vertex buffer region.

```ferrum
pub fn tessellate(
    path:      &Path,
    fill_rule: FillRule,
): Result[Vec[Vertex], GpuError]
```

### 6.2 Convex path fast path

Before invoking libtess2, `tessellate` performs a quick convexity check on the input
path:

1. Walk all contour segments and compute the cross products of consecutive edge
   directions.
2. If all cross products have the same sign (all left turns or all right turns), the
   path is convex.

Convex paths are tessellated as a triangle fan from the first vertex: zero extra
allocation, no libtess2 overhead. This handles the common cases — circles, rounded
rectangles, convex polygons — without invoking the general tessellator.

```
Convex polygon with N vertices → N-2 triangles
Triangle fan: (v0, v1, v2), (v0, v2, v3), ..., (v0, v_{N-2}, v_{N-1})
```

### 6.3 Concave paths — libtess2

Concave paths and paths with self-intersections are handled by libtess2 (extracted
from the Google Skia project, MIT license, approximately 2,000 lines of C). libtess2
implements the Hertel-Mehlhorn monotone polygon decomposition followed by triangle
extraction, supporting both even-odd and non-zero winding fill rules.

The C library is called via Ferrum's FFI (`! Unsafe`). The FFI wrapper is encapsulated
inside the tessellator module; no `! Unsafe` leaks into the `tessellate` public API.

```ferrum
// Internal — not public
fn tessellate_concave(
    path:      &Path,
    fill_rule: FillRule,
) : Result[Vec[Vertex], GpuError] ! Unsafe {
    let tess = libtess2::TessContext::new()
    for contour in path.contours() {
        tess.add_contour(contour.vertices())
    }
    let output = tess.tesselate(fill_rule_to_libtess2(fill_rule))?
    Ok(output.triangles().map(vertex_from_libtess2).collect())
}
```

If libtess2 returns an error (degenerate geometry, numerical instability),
`GpuError::TessellationFailed` is returned. The draw call is skipped; other draw calls
in the frame are unaffected.

### 6.4 Tessellation cache

Paths that do not change between frames (static UI elements) are cached by path hash.
The cache key is a 64-bit hash of the path's control points and the fill rule.
Cache entries store the pre-tessellated `Vec[Vertex]` and are evicted when memory
pressure exceeds a threshold (default: 8 MiB of cached tessellation data).

Dynamic paths (animated, data-driven) bypass the cache via a `Path::dynamic()` hint.

### 6.5 Stroke expansion

Stroked paths are converted to filled paths before tessellation. The stroke expansion
algorithm:

1. For each segment, compute offset curves at distance `stroke_width / 2` on both sides.
2. Connect offset curves at joins using the configured `StrokeJoin` (miter, round, bevel).
3. Cap open contour endpoints using the configured `StrokeCap` (butt, round, square).
4. The result is a filled polygon that represents the stroke region.
5. This filled polygon is then tessellated by the normal convex/libtess2 path.

Round joins and round caps require approximating circular arcs with cubic Bezier curves
(four arcs per full circle, standard approximation). The Bezier curves are then
flattened to line segments before tessellation.

### 6.6 Anti-aliasing

- **MSAA (primary):** When `GpuConfig::msaa_samples > 1`, the render pass uses a
  multisampled attachment and the GPU resolves to single-sample at frame end.
  Path edges receive coverage anti-aliasing automatically at the cost of
  `msaa_samples`× memory for the color attachment.
- **Analytical AA (alternative, no MSAA required):** For strokes and simple fills,
  the vertex shader can output a coverage value that fades the alpha at the edge of the
  primitive. This is narrower (1 pixel) but costs no extra memory and is useful on
  memory-constrained embedded targets with `msaa_samples = 1`.

---

## 7. Texture Cache

### 7.1 Overview

Images uploaded via `DrawBackend::draw_image` are cached on the GPU to avoid
re-uploading on every frame. The cache is an LRU map from `ImageRef` (a reference-
counted handle) to `GpuTexture`.

```ferrum
pub type TextureCache {
    // LRU map: ImageRef → GpuTexture
    entries:      LruMap[ImageId, GpuTexture],

    // Current total GPU memory used by cached textures.
    used_bytes:   u64,

    // Limit from GpuConfig.
    max_bytes:    u64,
}

pub type GpuTexture {
    // Opaque handle — either VkImage or GL texture name, depending on backend.
    pub handle:     TextureHandle,
    pub width:      u32,
    pub height:     u32,
    pub format:     GpuTextureFormat,
    pub size_bytes: u64,
}

pub enum GpuTextureFormat {
    Rgba8Unorm,
    Rgba8Srgb,
    R8Unorm,
    Rgba16Float,
}
```

### 7.2 Upload

```ferrum
impl GpuBackend {
    // Upload image data to the GPU, returning a reference-counted handle.
    //
    // If the same image content was previously uploaded (identified by ImageData's
    // content hash), the cached GpuTexture is returned immediately without
    // re-uploading. If the cache is full, LRU entries are evicted until the new
    // image fits within max_texture_cache_bytes.
    //
    // Returns GpuError::TextureTooLarge when the image exceeds the hardware
    // maximum texture dimension (query with GpuBackend::max_texture_size()).
    //
    // Effects: IO (may upload to GPU)
    pub fn create_image(
        &mut self,
        data: &ImageData,
    ): Result[ImageRef, GpuError] ! IO

    // Maximum supported texture dimension on the current device.
    pub fn max_texture_size(&self): u32
}
```

### 7.3 `ImageRef` lifetime

`ImageRef` is an `Arc[TextureCacheEntry]`. The `TextureCache` holds one weak reference
to each entry. When the caller drops all `ImageRef` handles, the strong count reaches
zero and the `TextureCacheEntry` destructor is called, which marks the GPU memory for
reclamation. This means callers that hold an `ImageRef` across frames do not pay
re-upload cost; callers that drop it allow the memory to be reclaimed under pressure.

```ferrum
pub type ImageRef = Arc[TextureCacheEntry]

pub type TextureCacheEntry {
    pub id:      ImageId,
    pub texture: GpuTexture,
}
```

---

## 8. Frame Lifecycle — Vulkan

The Vulkan backend follows a strict per-frame protocol. Calling methods out of order
returns `GpuError::InvalidState`.

### 8.1 `begin_frame()`

```ferrum
pub fn begin_frame(&mut self): Result[(), GpuError] ! IO
```

Steps:
1. Wait on the in-flight `VkFence` for the current frame slot (blocks until the GPU
   has finished consuming last frame's command buffer for this slot).
2. Reset the fence.
3. Call `vkAcquireNextImageKHR` to obtain the next swapchain image index.
   - `VK_ERROR_OUT_OF_DATE_KHR` → return `GpuError::SwapchainOutOfDate` (caller must
     call `rebuild_swapchain()` and retry).
   - `VK_SUBOPTIMAL_KHR` → continue; `present()` will return `SwapchainOutOfDate`
     to prompt a rebuild after the frame completes.
4. `vkBeginCommandBuffer` on the current frame slot's command buffer.
5. Insert a pipeline barrier transitioning the swapchain image from
   `UNDEFINED` / `PRESENT_SRC_KHR` to `COLOR_ATTACHMENT_OPTIMAL`.
6. `vkCmdBeginRenderPass` with the acquired image as the framebuffer attachment,
   clearing to the configured background color.
7. Bind the shared descriptor set (per-frame uniforms, glyph atlas sampler).
8. Reset the vertex buffer write cursor to zero.

### 8.2 Draw calls

Between `begin_frame()` and `end_frame()`, draw calls (fill, stroke, image, glyph run)
append commands to the open command buffer. No GPU work executes during this phase.
The CPU thread that calls draw methods must be the same thread that called
`begin_frame()` — the command buffer is not internally synchronized.

### 8.3 `end_frame()`

```ferrum
pub fn end_frame(&mut self): Result[(), GpuError] ! IO
```

Steps:
1. `vkCmdEndRenderPass`.
2. Insert a pipeline barrier transitioning the swapchain image from
   `COLOR_ATTACHMENT_OPTIMAL` to `PRESENT_SRC_KHR`.
3. `vkEndCommandBuffer`.
4. `vkQueueSubmit` with the command buffer and the frame fence.
   - Wait semaphore: image-available semaphore from `vkAcquireNextImageKHR`.
   - Signal semaphore: render-finished semaphore for `present()`.
5. Advance the frame slot index (modulo frame-in-flight count).

### 8.4 `present()`

```ferrum
pub fn present(&mut self): Result[(), GpuError] ! IO
```

Calls `vkQueuePresentKHR` with the render-finished semaphore as a wait semaphore.

- `VK_ERROR_OUT_OF_DATE_KHR` or `VK_SUBOPTIMAL_KHR` → return
  `GpuError::SwapchainOutOfDate`.
- `VK_ERROR_DEVICE_LOST` → return `GpuError::DeviceLost`.

### 8.5 Swapchain rebuild

When `begin_frame()` or `present()` returns `GpuError::SwapchainOutOfDate`, the caller
must rebuild the swapchain:

```ferrum
impl GpuBackend {
    // Rebuild the swapchain after a resize or OS-level surface change.
    // Waits for the GPU to be idle before destroying the old swapchain.
    // All framebuffers and swapchain image views are recreated.
    // The pipeline cache, glyph atlas, and texture cache are unaffected.
    //
    // Effects: IO
    pub fn rebuild_swapchain(
        &mut self,
        new_width:  u32,
        new_height: u32,
    ): Result[(), GpuError] ! IO
}
```

`DeviceLost` is more severe: the entire backend must be dropped and recreated. The
glyph atlas and texture cache must be re-uploaded from scratch. The caller is
responsible for re-uploading all persistent GPU resources after recreating the backend.

---

## 9. Surface Types

Platform backends create surfaces; `draw_gpu` consumes them. The surface types are thin
wrappers around platform API handles.

```ferrum
pub mod surface {
    // A VkSurfaceKHR handle created by a platform backend.
    // Created by WaylandWindow::vulkan_surface() or Win32Window::vulkan_surface() etc.
    // The surface is valid as long as the originating window is alive.
    pub type VulkanSurface {
        pub instance: VkInstance,    // the VkInstance the surface belongs to
        pub surface:  VkSurfaceKHR,  // the platform-specific surface handle
    }

    // An EGL surface handle created by a platform backend.
    // Created by WaylandWindow::egl_surface() or similar.
    pub type GlesSurface {
        pub display: EGLDisplay,
        pub surface: EGLSurface,
        pub context: EGLContext,
    }

    // Union of the two surface types, used by GpuBackend::auto().
    pub enum AnySurface {
        Vulkan(VulkanSurface),
        Gles(GlesSurface),
    }
}
```

### 9.1 Surface creation — platform backend methods

```ferrum
// extlib::draw_wayland
impl WaylandWindow {
    // Create a VkSurfaceKHR for this window via VK_KHR_wayland_surface.
    // Returns None when Vulkan is not available on this system.
    pub fn vulkan_surface(&self): Option[VulkanSurface] ! IO

    // Create an EGL surface for this window.
    // Returns None when EGL/GLES3 is not available.
    pub fn egl_surface(&self): Option[GlesSurface] ! IO
}

// extlib::draw_x11
impl X11Window {
    // Create a VkSurfaceKHR via VK_KHR_xcb_surface or VK_KHR_xlib_surface.
    pub fn vulkan_surface(&self): Option[VulkanSurface] ! IO

    pub fn egl_surface(&self): Option[GlesSurface] ! IO
}
```

---

## 10. Debugging and Profiling

### 10.1 Validation layers

```ferrum
impl GpuBackend {
    // Enable Vulkan validation layers for this backend.
    //
    // Must be called before the first draw call. Has no effect on the GLES3 path
    // (use GL_KHR_debug instead, which is enabled automatically in debug builds).
    //
    // Validation layer messages are forwarded to the Ferrum logging system at
    // the appropriate severity (error → log::error, warning → log::warn, etc.).
    //
    // This method is only available in debug builds. In release builds it is
    // a no-op that the compiler eliminates. Validation layers impose significant
    // CPU overhead and must not be used in production.
    pub fn enable_validation_layers(&mut self) ! IO
}
```

### 10.2 GPU profiler markers

```ferrum
impl GpuBackend {
    // Insert a named debug region marker into the GPU command stream.
    //
    // Vulkan: vkCmdBeginDebugUtilsLabelEXT (VK_EXT_debug_utils).
    // GLES3:  glPushDebugGroup (GL_KHR_debug).
    //
    // Markers appear in GPU profilers (RenderDoc, NSight, AMD RGP, Perfetto).
    // In release builds these are no-ops.
    pub fn begin_debug_region(&mut self, label: &str)
    pub fn end_debug_region(&mut self)
}
```

### 10.3 Frame statistics

```ferrum
pub type FrameStats {
    // Number of GPU draw calls issued in this frame.
    pub draw_calls:        u32,

    // Total triangles submitted (tessellated geometry + glyph/image quads).
    pub triangles:         u64,

    // Number of texture uploads to the GPU (new images and glyph bitmaps combined).
    pub texture_uploads:   u32,

    // Glyph atlas cache hits (glyph found in atlas, no rasterization needed).
    pub glyph_cache_hits:  u32,

    // Glyph atlas cache misses (glyph rasterized and uploaded this frame).
    pub glyph_cache_misses: u32,
}

impl GpuBackend {
    // Return statistics for the most recently completed frame.
    // Valid to call after present() returns.
    // Returns zeroed stats before the first completed frame.
    pub fn frame_stats(&self): FrameStats
}
```

---

## 11. Error Types

```ferrum
pub enum GpuError {
    // The GPU device was lost (driver crash, physical removal, power event).
    // The backend is unrecoverable. Drop it and recreate from scratch.
    // All GPU resources (atlas, texture cache, pipelines) must be re-uploaded.
    DeviceLost,

    // GPU or CPU memory allocation failed.
    // May be transient; freeing resources and retrying may succeed.
    OutOfMemory,

    // The swapchain is out of date — the window was resized or the surface
    // changed. Call rebuild_swapchain() and retry the frame.
    SwapchainOutOfDate,

    // Shader compilation failed. The embedded string is the driver's info log.
    // This indicates a driver bug in release builds; treat as fatal.
    ShaderCompileFailed(String),

    // libtess2 could not tessellate the path (degenerate geometry or
    // numerical instability). The affected draw call is skipped.
    TessellationFailed,

    // The requested image or glyph exceeds the hardware maximum texture size.
    TextureTooLarge {
        max_size: u32,
    },

    // No Vulkan-capable or GLES3-capable device was found.
    // The caller should fall back to a software renderer (draw_fb).
    NoSuitableDevice,

    // A backend method was called in an invalid order (e.g., draw_image
    // called before begin_frame, or begin_frame called twice).
    InvalidState,
}

impl Display for GpuError { ... }
impl Error   for GpuError { ... }
```

---

## 12. Example Usage

### 12.1 Create a Vulkan backend from a Wayland window

```ferrum
use extlib::draw_gpu::{GpuBackend, GpuConfig, GpuError, VsyncMode}

fn create_gpu_backend(window: &WaylandWindow): Result[GpuBackend, GpuError] ! IO {
    let config = GpuConfig {
        msaa_samples:            4,
        hdr:                     false,
        vsync:                   VsyncMode::Adaptive,
        max_texture_cache_bytes: 256 * 1024 * 1024,  // 256 MiB
    }

    // Try Vulkan first (the Wayland backend creates VK_KHR_wayland_surface).
    match window.vulkan_surface() {
        Some(surface) => GpuBackend::new_vulkan(surface, config),
        None => {
            // Vulkan unavailable — try EGL/GLES3.
            let surface = window.egl_surface()
                .ok_or(GpuError::NoSuitableDevice)?
            GpuBackend::new_gles3(surface, config)
        }
    }
}
```

### 12.2 Animated UI render loop at 60 fps

```ferrum
use extlib::draw::{DrawBackend, Path, Paint, Color}
use extlib::draw_gpu::GpuError

fn render_loop(backend: &mut GpuBackend, ui: &Ui): Result[(), GpuError] ! IO {
    loop {
        match backend.begin_frame() {
            Ok(()) => {}
            Err(GpuError::SwapchainOutOfDate) => {
                let (w, h) = ui.window_size()
                backend.rebuild_swapchain(w, h)?
                backend.begin_frame()?
            }
            Err(e) => return Err(e),
        }

        // Draw background.
        let bg = Path::rect(ui.bounds())
        backend.fill_path(&bg, FillRule::NonZero, &Paint::color(Color::BLACK))?

        // Draw animated UI elements.
        for widget in ui.visible_widgets() {
            widget.draw(backend)?
        }

        backend.end_frame()?

        match backend.present() {
            Ok(()) => {}
            Err(GpuError::SwapchainOutOfDate) => {
                // Rebuild at the top of the next iteration.
            }
            Err(GpuError::DeviceLost) => {
                // Device lost: backend is unrecoverable.
                return Err(GpuError::DeviceLost)
            }
            Err(e) => return Err(e),
        }

        let stats = backend.frame_stats()
        if stats.glyph_cache_misses > 50 {
            // Log unexpected miss rate (may indicate atlas thrashing).
            log::warn("high glyph cache miss rate: {}", stats.glyph_cache_misses)
        }
    }
}
```

### 12.3 Handle DeviceLost — full backend recreation

```ferrum
fn run_with_recovery(window: &WaylandWindow, ui: &mut Ui) ! IO {
    loop {
        let mut backend = match create_gpu_backend(window) {
            Ok(b)  => b,
            Err(GpuError::NoSuitableDevice) => {
                // Fall back to software renderer.
                log::warn("no GPU device; falling back to draw_fb")
                return run_software(window, ui)
            }
            Err(e) => panic("unrecoverable GPU init error: {}", e),
        }

        match render_loop(&mut backend, ui) {
            Ok(()) => break,   // clean exit
            Err(GpuError::DeviceLost) => {
                log::warn("GPU device lost; recreating backend")
                // Drop backend here — all GPU resources are freed.
                // ui retains its CPU-side image/font state; the new backend
                // will re-upload textures on first use.
                drop(backend)
                // Loop and recreate.
            }
            Err(e) => panic("render error: {}", e),
        }
    }
}
```

---

## 13. Dependencies

### Extended library dependencies

| Module | Why required |
|---|---|
| `extlib::draw` | Provides the `DrawBackend` trait, `Path`, `Paint`, `GlyphRun`, `ImageRef`, `FillRule`, `StrokeStyle` types that `GpuBackend` implements and consumes. |
| `extlib::font` | Glyph rasterization: `extlib::font::rasterize()` produces bitmaps that the glyph atlas uploads. `draw_gpu` does not contain a rasterizer. |
| `extlib::draw_wayland` | `WaylandWindow::vulkan_surface()` and `WaylandWindow::egl_surface()` for Linux/Wayland targets. |
| `extlib::draw_x11` | `X11Window::vulkan_surface()` and `X11Window::egl_surface()` for Linux/X11 targets. |

### External C library dependencies

| Library | Version | License | Role |
|---|---|---|---|
| Vulkan headers | 1.2+ | Apache 2.0 | GPU command recording, pipeline management, swapchain |
| EGL headers | 1.4+ | MIT | GLES3 context creation and surface management |
| GLES3 headers | 3.0+ | MIT | OpenGL ES 3.0 draw commands |
| libtess2 | commit `83b6f72` (Skia fork) | MIT | Concave path tessellation |

### Standard library dependencies

| Module | Used for |
|---|---|
| `stdlib::alloc` | `Vec`, `Arc`, `Box`, `LruMap`, `HashMap` |
| `stdlib::sys` | FFI bindings to Vulkan, EGL, and libtess2 C APIs |
| `stdlib::io` | Persistent pipeline cache blob read/write |
| `stdlib::hash` | Path tessellation cache key hashing, image content hashing |

### Effect requirements

| Public function | Effects |
|---|---|
| `GpuBackend::new_vulkan()`, `::new_gles3()`, `::auto()` | `! IO` — allocates GPU memory, opens device |
| `DrawBackend::begin_frame()`, `::end_frame()`, `::present()` | `! IO` — submits GPU commands |
| `DrawBackend::fill_path()`, `::stroke_path()`, `::draw_image()`, `::draw_glyph_run()` | `! IO` — may trigger GPU texture upload |
| `GpuBackend::rebuild_swapchain()` | `! IO` — destroys and recreates GPU resources |
| `GpuBackend::frame_stats()` | none — pure read of CPU-side counters |
| Internal tessellator FFI | `! Unsafe` — encapsulated; not visible in public API |

All GPU resource allocation and submission carries `! IO` because it interacts with
kernel drivers and GPU hardware outside the program's pure computation. The `! Unsafe`
from the libtess2 FFI call is encapsulated inside the tessellator module and does not
propagate to callers of `DrawBackend`.
