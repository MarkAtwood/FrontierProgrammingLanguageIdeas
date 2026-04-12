# Ferrum Extended Library — Draw: Linux Framebuffer Backend

**Module:** `extlib.ccsp.draw.fb`
**Spec basis:** Linux kernel framebuffer API (`linux/fb.h`), POSIX terminal ioctls
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.draw` (DrawBackend trait); `std.sys.posix` (ioctl, mmap); no other extlib deps

---

## 1. Overview and Rationale

The framebuffer backend provides direct pixel access to a display without a windowing system. It targets the environments where there is no Wayland compositor, no X11 server, and no display server process at all — only a kernel device node and a block of memory mapped to the screen.

### When to Use the Framebuffer Backend

- **Embedded Linux.** A Raspberry Pi running a custom image, an industrial HMI, or an OpenWrt router with an attached LCD panel. The Linux kernel exposes `/dev/fb0`. No compositor is present, and spawning one is not an option. The framebuffer backend writes pixels directly.

- **Microcontrollers with custom displays.** A bare-metal or RTOS-based target with an SPI LCD or I2C OLED. There is no `/dev/fb0` at all. The flush-callback path in `FramebufferBackend::from_raw()` accepts any write function — SPI, MMIO, DMA — and drives it with the same drawing API as a Linux desktop backend.

- **Kiosk and signage systems.** A full-screen application running as the only GUI process. The framebuffer backend occupies the virtual terminal and renders directly, eliminating the latency, memory, and attack surface of a compositor.

- **Recovery and installer screens.** Early-boot environments where the display stack has not started. GRUB and many Linux installers use the framebuffer for exactly this reason. Ferrum code running in a recovery initramfs can draw UI without a running systemd or display manager.

- **Headless rendering and testing.** The `from_raw()` constructor with a no-op flush callback runs the software rasterizer entirely in memory with no hardware dependency. Integration tests can render scenes and compare pixel output without a display.

### No Windowing System Required

This backend has no dependency on a display server, compositor, or graphics library. It does not link against libwayland, libX11, SDL, or Qt. It communicates with the display through three mechanisms:

- `open("/dev/fb0")` — a plain file open
- `ioctl()` — device queries and control
- `mmap()` — pixel buffer access

On custom targets the device file is replaced by a user-supplied flush callback. The dependency surface is a POSIX file descriptor and optionally `mmap`.

---

## 2. FramebufferBackend

`FramebufferBackend` is a concrete type that implements the `DrawBackend` trait defined in `extlib.ccsp.draw`. All drawing operations reach this type through that trait interface; the framebuffer-specific methods are for setup, configuration, and flush-callback control.

```ferrum
use extlib.ccsp.draw.{DrawBackend, Color, Rect, ImageRef, GlyphRun}
use std.path.Path
use std.sys.posix.{Fd, MmapPtr}

// Open the Linux framebuffer device at the given path.
// Queries FBIOGET_VSCREENINFO and FBIOGET_FSCREENINFO.
// Returns FbError.DeviceOpen if the path cannot be opened (no device,
// permission denied) and FbError.IoctlFailed if the info queries fail.
pub fn FramebufferBackend::open(device: &Path): Result[FramebufferBackend, FbError] ! IO

// Construct a framebuffer backend for targets with no device file.
// The flush callback in config is called by present() with the pixel buffer
// and the list of dirty rectangles. See §6 for callback details.
pub fn FramebufferBackend::from_raw(
    config: RawFbConfig,
): Result[FramebufferBackend, FbError]

impl DrawBackend for FramebufferBackend { ... }
```

The type owns the back buffer (`Vec[u8]` in RGBA8888 working format) and, for the `open()` path, the mmap of the front buffer. When the `FramebufferBackend` is dropped, the mmap is unmapped and the file descriptor is closed. For the `from_raw()` path, the caller owns any hardware handles; the type only holds the pixel buffer and the flush closure.

---

## 3. Framebuffer Info

On Linux, the kernel exposes two info structures via ioctl. `FBIOGET_VSCREENINFO` returns the virtual screen geometry and pixel format. `FBIOGET_FSCREENINFO` returns physical memory layout including `line_length`. This module queries both at `open()` time and exposes the result as `FbInfo`.

```ferrum
pub struct FbInfo {
    pub width:          u32,   // visible pixel columns
    pub height:         u32,   // visible pixel rows
    pub virtual_width:  u32,   // virtual framebuffer columns (>= width; used for panning)
    pub virtual_height: u32,   // virtual framebuffer rows (>= height; used for double-buffering with FBIOPAN_DISPLAY)
    pub bits_per_pixel: u32,   // bits per pixel in the hardware framebuffer
    pub line_length:    u32,   // bytes per row in the mmap (may include padding)
    pub pixel_format:   PixelFormat,
}

impl FramebufferBackend {
    pub fn info(&self): &FbInfo
}

// All pixel formats that this backend can convert to or from.
// The internal working format is always Rgba8888 (see §4).
pub enum PixelFormat {
    Rgb565,       // 16 bpp; common on small LCDs (ILI9341, ST7789)
    Rgb888,       // 24 bpp packed; desktop LCDs
    Argb8888,     // 32 bpp; most Linux framebuffers in ARGB byte order
    Rgba8888,     // 32 bpp; RGBA byte order (internal working format)
    Bgr565,       // 16 bpp; BGR variant (some Samsung panels)
    Bgr888,       // 24 bpp packed BGR
    Abgr8888,     // 32 bpp; ABGR byte order
    Grayscale8,   // 8 bpp luminance
    Grayscale1,   // 1 bpp packed monochrome; used by SSD1306 OLED and e-ink
}
```

`PixelFormat` is inferred automatically from the `red`, `green`, `blue`, and `transp` bitfield offsets and lengths in `FBIOGET_VSCREENINFO`. If the hardware reports a layout that does not match any known variant, `open()` returns `FbError.UnsupportedPixelFormat`.

For `from_raw()` targets the caller specifies `format` directly in `RawFbConfig`.

---

## 4. Pixel Format Conversion

The draw extlib's `Color` type is RGBA8888: four bytes, red first. The back buffer stores pixels in this format. At `present()` time the back buffer is converted to the hardware pixel format before being written to the front buffer or passed to the flush callback.

```ferrum
// Convert a back buffer (RGBA8888) to the hardware format in-place into dst.
// src and dst must each contain exactly width * height pixels in their
// respective formats. SIMD acceleration is applied automatically where the
// target supports NEON or SSE4.1.
pub fn convert_rgba_to_format(
    src:    &[u8],
    dst:    &mut [u8],
    format: PixelFormat,
    width:  u32,
    height: u32,
)

impl FramebufferBackend {
    // Set the background color used when compositing RGBA pixels onto
    // hardware formats that have no alpha channel (Rgb565, Rgb888, etc.).
    // Default: opaque black (0, 0, 0, 255).
    pub fn set_background_color(&mut self, color: Color)
}
```

Alpha compositing for non-alpha formats uses the Porter-Duff "src over" formula against the configured background color. This is a pure CPU operation applied once per `present()` call. On SIMD-capable targets (ARM NEON, x86 SSE4.1) the conversion loop is vectorized; the fallback is a portable scalar loop.

`Grayscale1` packing is row-major, MSB-first per byte, which matches the SSD1306 horizontal addressing mode. Conversion from RGBA8888 applies luminance weighting (0.299 R + 0.587 G + 0.114 B) and then applies the threshold or dither configured via `MonochromeConfig` (see §11).

---

## 5. Double Buffering

Double buffering prevents tearing by ensuring that the display always reads from a complete, consistent frame.

```ferrum
impl FramebufferBackend {
    // Enable or disable double buffering. Default: enabled.
    // When enabled, drawing goes to the back buffer; present() commits it.
    // When disabled, drawing goes directly to the front buffer (mmap).
    // Disabling is useful only for diagnostics or on targets where the
    // back buffer allocation is too expensive.
    pub fn set_double_buffer(&mut self, enabled: bool)
}
```

### Back Buffer

The back buffer is a `Vec[u8]` allocated on the heap, sized `width * height * 4` bytes, in RGBA8888 format. All draw extlib operations write to this buffer via the CPU rasterizer.

### Front Buffer

On the `open()` path, the front buffer is the framebuffer mmap: the kernel maps the hardware video RAM directly into the process address space. Writing to this memory updates the display.

### Page Flip

When `virtual_height >= 2 * height`, the kernel supports hardware page flipping via `FBIOPAN_DISPLAY`. The front buffer mmap covers both display pages. `present()` converts the back buffer into the inactive page and then calls `FBIOPAN_DISPLAY` to atomically switch the hardware's scan-out pointer. This is a true zero-copy flip: no `memcpy` from back to front; the conversion is done directly into the inactive hardware page.

When `FBIOPAN_DISPLAY` is not available (virtual height too small, or driver does not support it), `present()` converts the back buffer into a temporary staging buffer and copies it to the mmap with `memcpy`. This is the memcpy fallback path. The staging buffer is reused across frames to avoid allocation.

### from_raw Path

`from_raw()` targets have no mmap. The back buffer is the only buffer. At `present()` the back buffer is converted to the target format and passed to the flush callback. See §6.

---

## 6. Flush Callback

For custom and embedded targets, the flush callback is the only path from pixel buffer to hardware. There is no kernel framebuffer, no mmap, and no `FBIOPAN_DISPLAY`.

```ferrum
// Configuration for a custom/embedded framebuffer target.
pub struct RawFbConfig {
    pub width:  u32,
    pub height: u32,
    pub format: PixelFormat,

    // Called by present() after pixel format conversion.
    // Arguments:
    //   pixels: the full pixel buffer in the configured format,
    //           row-major, width * height pixels
    //   dirty:  the list of rectangles that changed since the last present().
    //           May be empty if damage tracking is disabled or if a full
    //           refresh was forced. The backend guarantees that dirty
    //           rectangles are non-overlapping and clipped to the display bounds.
    pub flush: Box[dyn FnMut(&[u8], &[DirtyRect]) + Send],
}

// A rectangle of changed pixels, in display coordinates.
// x, y are the top-left corner; width and height are the extent.
pub struct DirtyRect {
    pub x:      u32,
    pub y:      u32,
    pub width:  u32,
    pub height: u32,
}
```

### SPI LCD Use Case

An ILI9341 controller on SPI requires a `CASET`/`RASET`/`RAMWR` command sequence per dirty region. The flush callback receives the dirty rectangles and iterates over them, sending only the changed pixel data. A full-screen refresh sends one rectangle covering the entire display.

```ferrum
let mut spi = SpiDevice.open("/dev/spidev0.0")?

let config = RawFbConfig {
    width:  240,
    height: 320,
    format: PixelFormat.Rgb565,
    flush: Box.new(move |pixels, dirty| {
        for rect in dirty {
            ili9341_set_window(&mut spi, rect.x, rect.y, rect.width, rect.height)
            let row_bytes = (rect.width * 2) as usize
            for row in 0..rect.height {
                let y = (rect.y + row) as usize
                let x = rect.x as usize
                let src_offset = (y * 240 + x) * 2
                spi.write(&pixels[src_offset .. src_offset + row_bytes]).ok()
            }
        }
    }),
}

let mut backend = FramebufferBackend::from_raw(config)?
```

### DMA Transfer Use Case

A target with a hardware DMA controller can initiate a DMA transfer from the pixel buffer address. The flush callback captures the DMA handle and starts a transfer, blocking until completion or accepting a completion callback depending on the DMA API.

### MMIO Framebuffer Use Case

Some SoCs expose display controller registers at a fixed physical address. After `mmap()`-ing the MMIO region the flush callback writes pixels directly into the memory-mapped register window, which the display controller reads on each scan line.

---

## 7. Damage Tracking

Damage tracking allows `present()` to send only the regions that changed since the last frame. This is critical for slow buses: writing a 320x240 RGB565 display over SPI at 40 MHz takes approximately 3.7 ms for a full frame. If only a 40x20 button label changed, sending that region takes 0.05 ms — 74× less bus time.

```ferrum
impl FramebufferBackend {
    // Mark a rectangle as dirty. Dirty rectangles accumulate until present()
    // is called. Multiple overlapping calls are unioned into the dirty set.
    // If damage tracking is not used — i.e., damage() is never called —
    // present() treats the entire display as dirty.
    pub fn damage(&mut self, rect: Rect)

    // Force a full-display flush every N frames regardless of damage tracking.
    // Used for e-ink displays that accumulate ghosting over partial updates
    // and need periodic full refresh to clear artifacts.
    // Setting n to 0 disables periodic full refresh (default).
    pub fn set_full_refresh(&mut self, every_n_frames: u32)
}
```

Internally, the dirty set is maintained as a list of non-overlapping rectangles. When `damage()` is called, the new rectangle is merged with any existing dirty rectangles it intersects. At `present()` time, the dirty set is passed to the flush callback and then cleared.

When using the `FBIOPAN_DISPLAY` path (§5), dirty tracking still controls which regions are converted and written to the inactive hardware page before the page flip.

---

## 8. VSync and Frame Timing

### VSync via Ioctl

The Linux framebuffer driver exposes `FBIO_WAITFORVSYNC` on many platforms. Calling this ioctl blocks the calling thread until the next vertical blanking interval.

```ferrum
impl FramebufferBackend {
    // Block until the next vertical blanking interval.
    // Returns FbError.IoctlFailed { request: FBIO_WAITFORVSYNC, ... }
    // if the driver does not support this ioctl. The caller may fall back
    // to target_fps() in that case.
    pub fn wait_vsync(&self): Result[(), FbError] ! IO
}
```

A typical render loop on a vsync-capable device:

```ferrum
loop {
    draw_frame(&mut ctx)
    backend.present()?
    backend.wait_vsync()?
}
```

### Software Frame Limiting

When vsync is unavailable — common on embedded targets and virtual framebuffers — software frame limiting caps the frame rate to avoid burning CPU.

```ferrum
impl FramebufferBackend {
    // Set a target frame rate in frames per second. present() will sleep
    // if the frame completed faster than 1/fps seconds. Does not block
    // waiting for hardware; it is purely a software delay.
    // Set to 0.0 to disable frame limiting (default).
    pub fn target_fps(&mut self, fps: f32)
}
```

The sleep is implemented via `std.time.sleep()`. On targets with real-time scheduling constraints, the caller should manage frame timing explicitly and not use this method.

---

## 9. POSIX Terminal Interaction

Full-screen framebuffer applications need to manage the terminal: hide the cursor, suppress echo, prevent the terminal from interpreting keystrokes, and handle virtual terminal switching.

### FbTerminal

```ferrum
// Acquire exclusive use of the current terminal for full-screen rendering.
// On success:
//   - The text cursor is hidden (DECTCEM off via VT100 escape)
//   - Terminal echo and canonical mode are disabled (cfmakeraw equivalent)
//   - The process is set as the foreground process group on the tty
// Dropping FbTerminal restores all terminal state.
// Returns FbError.DeviceOpen if no controlling terminal is available.
pub fn FbTerminal::acquire(): Result[FbTerminal, FbError] ! IO

pub struct FbTerminal { ... }   // owns the saved termios state; restores on drop

impl FbTerminal {
    // Switch to virtual terminal n (1-based, e.g. n=1 for VT1).
    // Requires CAP_SYS_TTY_CONFIG or running as root on most systems.
    // Uses VT_ACTIVATE + VT_WAITACTIVE ioctls.
    pub fn switch_vt(&self, n: u32): Result[(), FbError] ! IO

    // Register this process as a VT-switching client using VT_SETMODE ioctl.
    // The process will receive SIGUSR1 when asked to release the VT (another
    // VT is being switched to) and SIGUSR2 when the VT is being restored.
    // The caller must install signal handlers and call signal_vt_switch_grant()
    // in response. Without this, VT switching hangs until the process exits.
    pub fn enable_vt_switch_signals(&self): Result[(), FbError] ! IO

    // Acknowledge a VT release request (call from SIGUSR1 handler).
    // Sends VT_RELDISP 1 to the kernel.
    pub fn signal_vt_switch_grant(&self): Result[(), FbError] ! IO

    // Acknowledge a VT acquire (call from SIGUSR2 handler).
    // Sends VT_RELDISP VT_ACKACQ to the kernel.
    pub fn signal_vt_acquire(&self): Result[(), FbError] ! IO
}
```

VT switch signals use `SIGUSR1` and `SIGUSR2` by convention with the `VT_SETMODE` ioctl in `VT_PROCESS` mode. An application that does not call `enable_vt_switch_signals()` will still work but switching away from its VT will be delayed or will not complete until the application exits.

---

## 10. Software Rasterizer Integration

The framebuffer backend is a purely CPU-side renderer. No GPU is involved at any point.

### DrawContext and Back Buffer

The draw extlib's `DrawContext` holds a reference to the `FramebufferBackend` via the `DrawBackend` trait. All drawing calls — `fill_rect()`, `stroke_path()`, `fill_path()`, `draw_image()`, `draw_glyph_run()` — execute the draw extlib's software rasterizer, which writes RGBA8888 pixels into the back buffer.

The rasterizer is the same code regardless of whether the final output is a framebuffer, a PNG encoder, or a Wayland shared memory buffer. The backend is a pixel sink; it does not participate in rasterization decisions.

### Image Uploads

```ferrum
// ImageRef is an opaque handle returned by DrawBackend::create_image().
// On the framebuffer backend, it is a reference-counted byte buffer.
// There is no GPU texture allocation, no driver upload, no DMA.
// The pixel data stays in normal heap memory and is blitted by the
// rasterizer at draw time.
//
// type alias shown for clarity; the exact type is DrawBackend-defined:
type ImageRef = Arc[Vec[u8]]   // RGBA8888, row-major
```

Creating an image does not copy to a GPU. Blitting an image scales and composites it in the rasterizer's inner loop. This is simple and memory-efficient but slower than a GPU blit for large images. The tradeoff is appropriate for embedded displays where images are small (icons, splash screens) and the CPU is the only processor available.

### Text Rendering

`GlyphRun` rendering uses the font extlib's glyph cache, which stores pre-rasterized glyph bitmaps (grayscale8 coverage maps). The draw extlib blits each glyph bitmap into the back buffer using alpha compositing with the text color. No GPU shader, no font atlas texture — just a CPU blit loop.

---

## 11. Embedded and Custom Display Adapters

This section describes how the `from_raw()` path maps to common embedded display interfaces. In each case the application code provides a `RawFbConfig` with the appropriate `flush` callback and the rest of the draw extlib API is identical.

### SPI LCD — ILI9341 / ST7789

These controllers use RGB565 (16 bpp) natively. The flush callback receives `PixelFormat.Rgb565` pixels and dirty rectangles, issues window set commands (`CASET`, `RASET`, `RAMWR`), and writes pixel bytes over SPI.

The SPI bus is typically 40–80 MHz, giving 5–10 MB/s throughput. A 320×240 frame is 150 KB, requiring 15–30 ms for a full refresh. Damage tracking (§7) reduces this to the changed region only.

### E-ink Display

E-ink panels are slow (100–500 ms per full refresh) but retain the image with no power. They require special handling:

```ferrum
// E-ink targets typically use Grayscale8 or Rgb888 working format.
// The flush callback selects full or partial refresh mode based on the
// dirty region coverage ratio.

let mut last_full_refresh = 0u32
let config = RawFbConfig {
    width:  400,
    height: 300,
    format: PixelFormat.Grayscale8,
    flush: Box.new(move |pixels, dirty| {
        let total_pixels = 400 * 300
        let dirty_pixels: u32 = dirty.iter().map(|r| r.width * r.height).sum()
        let coverage = dirty_pixels as f32 / total_pixels as f32

        if coverage > 0.5 || last_full_refresh > 50 {
            eink_full_refresh(pixels)
            last_full_refresh = 0
        } else {
            for rect in dirty {
                eink_partial_refresh(pixels, rect)
            }
        }
        last_full_refresh += 1
    }),
}
```

`set_full_refresh(every_n_frames: u32)` (§7) provides the same periodic full-refresh behavior without the application writing it manually.

### OLED — SSD1306

The SSD1306 uses a 1-bit packed monochrome format with page-based addressing: each byte encodes 8 vertical pixels, MSB at bottom. The `Grayscale1` pixel format in this module uses horizontal row-major packing (MSB-first per byte, MSB at the top), which differs from the SSD1306 page format. The flush callback performs the transposition.

```ferrum
// Transpose from row-major 1bpp to SSD1306 page format in the flush callback.
// Each SSD1306 page is 8 rows; each byte in the page covers one column,
// bits 0–7 mapping to rows 0–7 within the page.
let config = RawFbConfig {
    width:  128,
    height: 64,
    format: PixelFormat.Grayscale1,
    flush: Box.new(move |pixels, _dirty| {
        let mut page_buf = [0u8; 128 * 8]   // 8 pages of 128 bytes
        for page in 0..8u32 {
            for col in 0..128u32 {
                let mut byte = 0u8
                for bit in 0..8u32 {
                    let row = page * 8 + bit
                    let src_byte = (row * 128 + col) / 8
                    let src_bit = 7 - ((row * 128 + col) % 8)
                    if (pixels[src_byte as usize] >> src_bit) & 1 == 1 {
                        byte |= 1 << bit
                    }
                }
                page_buf[(page * 128 + col) as usize] = byte
            }
        }
        i2c_write_ssd1306(&mut i2c, &page_buf).ok()
    }),
}
```

### Monochrome Pixel Format Configuration

For 1-bit displays, the `Grayscale1` conversion from RGBA8888 applies a configurable luminance threshold or dithering.

```ferrum
// Configures the RGBA8888 -> Grayscale1 conversion used when
// PixelFormat.Grayscale1 is selected.
pub struct MonochromeConfig {
    // Threshold in the range [0.0, 1.0]. Pixels with luminance above
    // this value become white (1); pixels at or below become black (0).
    // Default: 0.5
    pub threshold: f32,

    // If true, use Floyd-Steinberg error diffusion dithering instead
    // of hard threshold. Produces smoother gradients on 1-bit displays.
    // Default: false
    pub dither: bool,
}

impl FramebufferBackend {
    // Set the monochrome conversion parameters. Only takes effect when
    // pixel_format is Grayscale1. Must be called before from_raw().
    pub fn set_monochrome_config(&mut self, config: MonochromeConfig)
}
```

---

## 12. Error Types

```ferrum
pub enum FbError {
    // The device path could not be opened.
    // Typical causes: no device node (/dev/fb0 absent), permission denied,
    // device busy (another process has it open exclusively).
    DeviceOpen { path: String, error: IoError },

    // A kernel ioctl failed.
    // `request` is the ioctl number (e.g., FBIOGET_VSCREENINFO).
    // `error` is the errno value from the failed call.
    IoctlFailed { request: u32, error: IoError },

    // The framebuffer mmap failed.
    // `error` is the errno from the failed mmap(2) call.
    MmapFailed { error: IoError },

    // The hardware framebuffer reported a pixel format that this module
    // does not recognize. `format` is a human-readable description of
    // the reported bitfield layout from FBIOGET_VSCREENINFO.
    UnsupportedPixelFormat { format: String },

    // The width or height reported by the device is zero or exceeds
    // the maximum supported size (65535 x 65535).
    InvalidDimensions { width: u32, height: u32 },

    // The flush callback provided to from_raw() panicked.
    // The panic message is captured and returned here. The backend
    // is in an undefined state after this error; it should be dropped.
    FlushCallbackPanic { message: String },
}

impl Display for FbError { ... }
impl From[FbError] for IoError { ... }
```

---

## 13. Example Usage

### Raspberry Pi Framebuffer Application

A minimal full-screen framebuffer application on a Raspberry Pi running Raspberry Pi OS Lite (no desktop environment).

```ferrum
use extlib.ccsp.draw.fb.{FramebufferBackend, FbTerminal}
use extlib.ccsp.draw.{DrawContext, Color, Rect}
use std.path.Path

fn main(): Result[(), FbError] ! IO {
    // Acquire the terminal: hide cursor, disable echo, enter raw mode.
    // Terminal state is restored when _term is dropped.
    let _term = FbTerminal::acquire()?

    // Open the framebuffer.
    let mut backend = FramebufferBackend::open(Path.new("/dev/fb0"))?
    let info = backend.info()
    let (w, h) = (info.width, info.height)

    // Wrap in DrawContext from the draw extlib.
    let mut ctx = DrawContext.new(&mut backend)

    let mut frame: u32 = 0
    loop {
        // Draw a frame: clear to dark blue, draw a white rectangle.
        ctx.clear(Color.from_rgb(0x00, 0x00, 0x33))
        ctx.fill_rect(
            Rect { x: 40.0, y: 40.0, width: (w - 80) as f32, height: (h - 80) as f32 },
            Color.from_rgb(0xFF, 0xFF, 0xFF),
        )

        // Mark the entire screen as dirty and present.
        backend.damage(Rect.from_wh(w, h))
        backend.present()?
        backend.wait_vsync().unwrap_or(())

        frame += 1
        if frame >= 300 { break }
    }

    Ok(())
}
```

### SPI LCD with Flush Callback

An ILI9341 320×240 LCD driven over `/dev/spidev0.0` on a custom embedded board.

```ferrum
use extlib.ccsp.draw.fb.{FramebufferBackend, RawFbConfig, DirtyRect, PixelFormat}
use extlib.ccsp.draw.{DrawContext, Color, Rect}

fn main(): Result[(), FbError] ! IO {
    let mut spi = open_spi("/dev/spidev0.0", 40_000_000)?

    let config = RawFbConfig {
        width:  320,
        height: 240,
        format: PixelFormat.Rgb565,
        flush: Box.new(move |pixels: &[u8], dirty: &[DirtyRect]| {
            for rect in dirty {
                // Send CASET (column address set) then RASET (row address set).
                ili9341_set_window(&mut spi, rect.x, rect.y, rect.width, rect.height)
                // Write RGB565 pixels row by row.
                let bytes_per_pixel = 2usize
                for row in 0..rect.height {
                    let y = (rect.y + row) as usize
                    let x = rect.x as usize
                    let offset = (y * 320 + x) * bytes_per_pixel
                    let row_bytes = rect.width as usize * bytes_per_pixel
                    spi.write(&pixels[offset .. offset + row_bytes]).ok()
                }
            }
        }),
    }

    let mut backend = FramebufferBackend::from_raw(config)?
    backend.target_fps(30.0)

    let mut ctx = DrawContext.new(&mut backend)

    // Draw a gradient test pattern.
    for y in 0..240u32 {
        for x in 0..320u32 {
            let r = (x * 255 / 319) as u8
            let g = (y * 255 / 239) as u8
            ctx.fill_rect(
                Rect { x: x as f32, y: y as f32, width: 1.0, height: 1.0 },
                Color.from_rgb(r, g, 0x80),
            )
        }
    }

    backend.damage(Rect.from_wh(320, 240))
    backend.present()?
    Ok(())
}
```

### E-ink Partial Update

A 400×300 e-ink display that renders a clock face. The time digits update every second; the border and labels update only when the application starts.

```ferrum
use extlib.ccsp.draw.fb.{FramebufferBackend, RawFbConfig, PixelFormat}
use extlib.ccsp.draw.{DrawContext, Color, Rect}
use std.time.Duration

fn run_clock(eink: &mut EinkDriver): Result[(), FbError] ! IO {
    let config = RawFbConfig {
        width:  400,
        height: 300,
        format: PixelFormat.Grayscale8,
        flush: Box.new(move |pixels, dirty| {
            eink.flush_regions(pixels, dirty)
        }),
    }

    let mut backend = FramebufferBackend::from_raw(config)?
    // Full refresh every 60 frames (60 seconds) to clear ghosting.
    backend.set_full_refresh(60)

    let mut ctx = DrawContext.new(&mut backend)

    // Draw the static border once.
    ctx.stroke_rect(
        Rect { x: 5.0, y: 5.0, width: 390.0, height: 290.0 },
        Color.from_rgb(0, 0, 0),
        2.0,
    )
    backend.damage(Rect { x: 0, y: 0, width: 400, height: 300 })
    backend.present()?

    loop {
        let now = std.time.now()

        // Clear only the time region, redraw the digits.
        let time_region = Rect { x: 80, y: 100, width: 240, height: 80 }
        ctx.fill_rect(
            Rect { x: 80.0, y: 100.0, width: 240.0, height: 80.0 },
            Color.from_rgb(0xFF, 0xFF, 0xFF),
        )
        draw_time_digits(&mut ctx, now)

        // Only the time region is dirty.
        backend.damage(time_region)
        backend.present()?

        std.time.sleep(Duration.from_secs(1))
    }
}
```

---

## 14. Dependencies

| Dependency | Role |
|---|---|
| `extlib.ccsp.draw` | `DrawBackend` trait, `Color`, `Rect`, `ImageRef`, `GlyphRun` — the drawing API this backend implements |
| `std.sys.posix` | `ioctl()`, `mmap()`, `munmap()`, `open()`, `close()`, `tcgetattr()`, `tcsetattr()`, `cfmakeraw()` |
| `std.path` | `Path` — used in `FramebufferBackend::open()` parameter type |
| `std.sync` | `Arc` — used internally for `ImageRef` pixel buffer reference counting |

No other extlib dependencies are taken. There is no dependency on `extlib.ccsp.dtls`, `extlib.ccsp.dns_secure`, or any network module. This is intentional: embedded targets that use the framebuffer backend typically have no network stack, and adding unneeded dependencies would bloat the binary and increase the porting surface.

The SIMD conversion routines in §4 use `std.arch` intrinsics guarded by compile-time feature detection. Targets without SIMD (e.g., soft-float ARMv6 or RISC-V without V extension) compile the scalar fallback. No runtime detection; the choice is made at compile time based on the target triple.

---

*Part of the CCSP (Constrained and Cryptographic Systems Protocol) extended library suite.*
*See also: `extlib.ccsp.draw` (DrawBackend trait and software rasterizer), `extlib.ccsp.font` (glyph cache and text layout)*
