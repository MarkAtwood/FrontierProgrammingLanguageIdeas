# Ferrum Extended Library — `draw_x11` Module

**Module path:** `extlib::ccsp::draw_x11`
**Implements:** `extlib::draw::DrawBackend`
**Depends on:** `extlib::draw`, `extlib::input`, `stdlib::io`, `stdlib::alloc`
**External C libraries:** Xlib (`libX11`), XRender (`libXrender`), XShm (`libXext`), Damage (`libXdamage`)
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)

---

## 1. Overview and Rationale

### When to use this backend

| Situation | Backend to use |
|-----------|----------------|
| Wayland desktop with a compositor running | `extlib::ccsp::draw_wayland` |
| Embedded Linux with no Wayland compositor | **`extlib::ccsp::draw_x11`** |
| Running under XWayland on a Wayland desktop | **`extlib::ccsp::draw_x11`** (transparent) |
| Bare framebuffer, no display server at all | `extlib::ccsp::draw_fb` |
| Legacy Linux with X.org display server | **`extlib::ccsp::draw_x11`** |

X11 is not deprecated. Embedded Linux systems frequently run a custom display manager
on top of an X server because it is the most widely deployed, best-understood display
stack on Linux outside of bare framebuffer. Kiosks, industrial displays, medical
terminals, point-of-sale systems, and automotive head units are common examples. The
Ferrum extended library treats embedded X11 as a first-class target, not a legacy
compatibility shim.

When running under XWayland, applications using this backend are transparent to the
difference. XWayland presents a fully conformant X11 server; no application-level
changes are required.

### What this module provides

- `X11Backend` — entry point for connecting to an X display server
- `X11Connection` — a live connection to one display, holds shared state
- `X11Window` — an on-screen window that implements `DrawBackend`
- `X11WindowConfig` builder — configures window properties before creation
- XRender compositing — antialiased rendering, alpha compositing, gradients, glyphs
- XShm shared memory — zero-copy pixel upload for local displays
- Damage extension — partial repaints, reduced bandwidth on remote X11
- High-DPI support — DPI detection via Xft resources and logical/physical mapping
- Input handling — keyboard, pointer, focus, expose, configure events
- Clipboard — ICCCM/EWMH selection protocol, transparent to the caller
- EWMH/ICCCM integration — window type hints, WM state, struts for panels

### Why this is an extended library, not stdlib

The standard library does not link C libraries. This module requires Xlib, XRender,
XShm, and the Damage extension, all of which are C libraries with non-trivial
initialization and global state. Programs that do not target X11 pay no cost.

---

## 2. `X11Backend` and `X11Connection`

### Connecting to a display

```ferrum
use extlib::ccsp::draw_x11::{X11Backend, X11Connection, X11Error}

// Connect to the display named in $DISPLAY.
// Returns an error if $DISPLAY is unset, the display string is malformed,
// or the server refuses the connection.
pub fn X11Backend::connect(): Result[X11Connection, X11Error] ! IO

// Connect to an explicit display string such as ":0", ":1", or
// "hostname:0.0" for a remote X11 server over TCP.
pub fn X11Backend::connect_to(display: &str): Result[X11Connection, X11Error] ! IO
```

### `X11Connection`

`X11Connection` holds the Xlib `Display *` and all shared resources. It is not `Clone`;
only one `X11Connection` exists per display. Multiple windows share the same connection.

```ferrum
pub type X11Connection

impl X11Connection {
    // Create a window according to config.
    // The window is initially unmapped (not visible).
    // Call X11Window::show() to map it.
    pub fn create_window(
        &mut self,
        config: X11WindowConfig,
    ): Result[X11Window, X11Error] ! IO

    // Check whether the XShm extension is available on this connection.
    // XShm is only usable when the client and server are on the same host.
    // Always false for remote X11 connections.
    pub fn has_shm(&self): bool

    // Check whether the XRender extension is available and at least version 0.10.
    pub fn has_render(&self): bool

    // Check whether the Damage extension is available.
    pub fn has_damage(&self): bool

    // True when the server is on the same host as the client.
    // Derived from the display string: ":N" and "unix:N" are local;
    // "hostname:N" is remote.
    pub fn is_local(&self): bool

    // DPI of the default screen, read from the Xft.dpi X resource if present,
    // otherwise computed from the screen's physical dimensions and pixel count.
    // Falls back to 96.0 if no physical dimension information is available.
    pub fn dpi(&self): f32

    // Flush all pending X requests to the server without waiting for replies.
    // Called automatically on window operations that require it; exposed here
    // for callers that batch requests manually.
    pub fn flush(&mut self) ! IO

    // Flush all pending requests and wait until the server has processed them.
    // More expensive than flush(); use only when you need completion guarantees.
    pub fn sync(&mut self) ! IO
}
```

`X11Connection` implements `Drop`; closing the last reference calls `XCloseDisplay`.

---

## 3. `X11WindowConfig` Builder

`X11WindowConfig` is the single configuration point for window creation. Every option
has a documented default. Callers only set what they care about.

```ferrum
pub type X11WindowConfig
```

### Constructor

```ferrum
impl X11WindowConfig {
    // Start a builder with sensible defaults:
    //   title:            "" (empty, WM shows none)
    //   size:             800 × 600 pixels
    //   position:         None (WM places the window)
    //   depth:            X11Depth::FromScreen
    //   resizable:        true
    //   override_redirect: false
    //   event_mask:       X11EventMask::DEFAULT
    pub fn new(): Self
```

### Window identity

```ferrum
    // Set the WM_NAME / _NET_WM_NAME property.
    // Shown by the window manager in the title bar and task switcher.
    pub fn title(mut self, title: &str): Self

    // Set WM_CLASS (instance name and class name).
    // Used by window managers and desktop environments for grouping,
    // icon assignment, and saved session state.
    //   name  — instance name, typically the binary name (e.g., "myapp")
    //   class — class name, typically the application name (e.g., "MyApp")
    pub fn class(mut self, name: &str, class: &str): Self
```

### Geometry

```ferrum
    // Initial window size in pixels.
    // X11 operates in physical pixels; there is no logical coordinate layer.
    // Default: 800 × 600.
    pub fn size(mut self, width: u32, height: u32): Self

    // Initial position hint passed to the WM via WM_NORMAL_HINTS.
    // Most window managers honor this for the first map; subsequent moves
    // are under WM control unless override_redirect is set.
    // Default: None (WM chooses placement).
    pub fn position(mut self, x: i32, y: i32): Self

    // Minimum window size enforced by the WM via WM_NORMAL_HINTS.
    pub fn min_size(mut self, width: u32, height: u32): Self

    // Maximum window size enforced by the WM via WM_NORMAL_HINTS.
    pub fn max_size(mut self, width: u32, height: u32): Self

    // When false, sets min_size == max_size == the configured size,
    // making the window non-resizable from the WM's perspective.
    // Default: true.
    pub fn resizable(mut self, yes: bool): Self
```

### Visual depth

```ferrum
    // Select the X visual depth for the window's backing pixmap.
    // Default: X11Depth::FromScreen.
    pub fn depth(mut self, depth: X11Depth): Self
```

```ferrum
pub enum X11Depth {
    // 24-bit TrueColor: 8 bits each for R, G, B.  No per-window alpha channel.
    // XRender can still composite using source-image alpha; the window itself
    // is opaque to the compositor.
    TrueColor24,

    // 32-bit TrueColor: 8 bits each for R, G, B, A.
    // Enables per-window transparency when a compositing WM is running.
    // Requires a RGBA visual; fails at window creation if none is available.
    TrueColor32,

    // Use the default visual and depth of the screen.
    // Safe and always available; does not support per-window alpha.
    FromScreen,
}
```

### Window manager bypass

```ferrum
    // When true, set the override_redirect attribute on the window.
    // The window manager will not decorate or manage this window;
    // it appears at the exact position and size specified, on top of
    // other windows, without a title bar or border.
    //
    // Use cases: popup menus, tooltips, on-screen keyboards, splash screens,
    // and custom panel applets that must occupy exact screen coordinates.
    //
    // Misuse of override_redirect on long-lived windows is antisocial;
    // WM keyboard shortcuts (Alt+F4, etc.) will not apply to the window.
    // Default: false.
    pub fn override_redirect(mut self, yes: bool): Self
```

### Event mask

```ferrum
    // Set the X event mask for this window.
    // Only events matching the mask are delivered to poll_events() / wait_event().
    // Default: X11EventMask::DEFAULT, which includes keyboard, pointer,
    // expose, configure, focus, and enter/leave events.
    pub fn event_mask(mut self, mask: X11EventMask): Self
```

```ferrum
@derive(Clone, Copy, Debug)
pub struct X11EventMask {
    bits: u32,
}

impl X11EventMask {
    pub const KEYBOARD:        X11EventMask  // KeyPress, KeyRelease
    pub const POINTER_BUTTON:  X11EventMask  // ButtonPress, ButtonRelease
    pub const POINTER_MOTION:  X11EventMask  // MotionNotify
    pub const ENTER_LEAVE:     X11EventMask  // EnterNotify, LeaveNotify
    pub const FOCUS:           X11EventMask  // FocusIn, FocusOut
    pub const EXPOSE:          X11EventMask  // Expose
    pub const STRUCTURE:       X11EventMask  // ConfigureNotify, DestroyNotify, etc.
    pub const PROPERTY:        X11EventMask  // PropertyNotify (for clipboard)

    // Default mask: KEYBOARD | POINTER_BUTTON | POINTER_MOTION |
    //               ENTER_LEAVE | FOCUS | EXPOSE | STRUCTURE
    pub const DEFAULT:         X11EventMask

    // Bitwise OR to combine masks.
    pub fn or(self, other: X11EventMask): X11EventMask
}
```

---

## 4. `X11Window`

`X11Window` is the handle to a created window. It implements `extlib::draw::DrawBackend`.

```ferrum
pub type X11Window

impl DrawBackend for X11Window { ... }

impl X11Window {
    // Map (show) the window on screen.
    pub fn show(&mut self) ! IO

    // Unmap (hide) the window without destroying it.
    pub fn hide(&mut self) ! IO

    // Destroy the window and release all associated server-side resources.
    // The X11Window is consumed; it cannot be used after this call.
    pub fn destroy(self) ! IO

    // Current window size in physical pixels.
    // Updated after ConfigureNotify events are processed.
    pub fn width(&self): u32
    pub fn height(&self): u32

    // Set the window title at runtime (updates WM_NAME / _NET_WM_NAME).
    pub fn set_title(&mut self, title: &str) ! IO
}
```

---

## 5. XRender Compositing

XRender is the X protocol extension for antialiased, alpha-compositing 2D rendering.
It operates server-side: rendering operations execute in the X server process, which
avoids round-trips for individual draw calls. XRender is the correct layer for
antialiased shapes, font rendering with subpixel hinting, and image compositing.

### Render picture

```ferrum
impl X11Window {
    // Return the XRender Picture associated with this window's drawable.
    // The picture target is the window's backing pixmap.
    // All XRender operations are issued against this picture.
    pub fn render_picture(&self): XRenderPicture
}
```

```ferrum
pub type XRenderPicture   // opaque handle wrapping Xlib's Picture (XID)
```

### Porter-Duff compositing operators

```ferrum
pub enum PictOp {
    // Place source over destination, respecting source alpha.
    // The most common operation for drawing UI elements on a background.
    Over,

    // Copy source to destination, ignoring destination contents.
    Src,

    // Clear destination to transparent black.
    Clear,

    // Source where source alpha and destination alpha both > 0.
    In,

    // Source where destination alpha == 0.
    Out,

    // Source over destination, clipped to destination extent.
    Atop,

    // Source XOR destination.
    Xor,

    // Add source and destination, clamped.
    Add,
}
```

### Compositing operations

```ferrum
impl XRenderPicture {
    // Composite src onto dst using op.
    // src_x, src_y: top-left of the source region.
    // dst_x, dst_y: top-left of the destination region.
    // width, height: region dimensions.
    pub fn composite(
        &self,
        op:     PictOp,
        src:    &XRenderPicture,
        mask:   Option[&XRenderPicture],
        src_x:  i16, src_y: i16,
        dst_x:  i16, dst_y: i16,
        width:  u16, height: u16,
    ) ! IO

    // Fill a rectangle with a solid ARGB color.
    pub fn fill_rect(
        &self,
        op:     PictOp,
        color:  XRenderColor,
        x: i16, y: i16,
        width: u16, height: u16,
    ) ! IO
}

// 16-bit-per-channel ARGB color as used by XRender.
// Scale: 0x0000 = 0.0, 0xffff = 1.0 (pre-multiplied alpha).
pub struct XRenderColor {
    pub red:   u16,
    pub green: u16,
    pub blue:  u16,
    pub alpha: u16,
}

impl XRenderColor {
    // Construct from 8-bit-per-channel RGBA values (not pre-multiplied).
    // Pre-multiplication is applied internally.
    pub fn from_rgba8(r: u8, g: u8, b: u8, a: u8): Self
}
```

### Gradient rendering

XRender gradients are server-side; no pixel data crosses the X connection.

```ferrum
// A linear gradient from point p1 to p2 with a sequence of color stops.
pub fn create_linear_gradient(
    conn:  &X11Connection,
    p1:    (f64, f64),
    p2:    (f64, f64),
    stops: &[(f64, XRenderColor)],   // (position 0.0–1.0, color)
): Result[XRenderPicture, X11Error] ! IO

// A radial gradient centered at inner with radius 0, outer at center with outer_radius.
pub fn create_radial_gradient(
    conn:         &X11Connection,
    inner:        (f64, f64),
    outer:        (f64, f64),
    inner_radius: f64,
    outer_radius: f64,
    stops:        &[(f64, XRenderColor)],
): Result[XRenderPicture, X11Error] ! IO
```

### Glyph rendering

XRender `GlyphSet` objects hold antialiased, pre-rasterized glyphs server-side. Glyph
upload (from FreeType or another rasterizer) happens once; repeated rendering uses only
the glyph IDs and positions.

```ferrum
// A server-side glyph cache.
pub type GlyphSet

impl GlyphSet {
    // Allocate a new, empty glyph set on the server.
    pub fn create(conn: &X11Connection): Result[GlyphSet, X11Error] ! IO

    // Upload rasterized glyph data.
    // glyphs: sequence of (glyph_id, GlyphInfo, image_bytes).
    // image_bytes must match the GlyphSet's format (alpha8 or argb32).
    pub fn add_glyphs(
        &mut self,
        glyphs: &[(u32, GlyphInfo, &[u8])],
    ) ! IO

    // Render a sequence of glyphs onto dst at the given baseline origin.
    // Composites each glyph using PictOp::Over with src as the foreground color.
    pub fn render(
        &self,
        op:     PictOp,
        src:    &XRenderPicture,
        dst:    &XRenderPicture,
        x:      i32, y: i32,
        glyphs: &[u32],
    ) ! IO
}

pub struct GlyphInfo {
    pub width:   u16,
    pub height:  u16,
    pub x_off:   i16,   // bearing x
    pub y_off:   i16,   // bearing y (positive = up from baseline)
    pub x_adv:   i32,   // horizontal advance in 1/65536 pixels (26.6 fixed-point)
    pub y_adv:   i32,
}
```

---

## 6. XShm Shared Memory

The XShm extension places pixel data in a memory segment shared between the client and
the X server. A `put_shm_image` call sends only a small protocol message (not the pixel
data) across the socket, making it effectively zero-copy for local display connections.

XShm is only available when `X11Connection::is_local()` is true. Calling XShm
operations on a remote connection returns `X11Error::ExtensionNotFound { name: "MIT-SHM" }`.

### Creating and using shared memory images

```ferrum
// A pixel buffer in shared memory.  The server can read pixels from this
// buffer with a single protocol message instead of a data transfer.
pub struct ShmImage {
    pub width:  u32,
    pub height: u32,
    // Direct, mutable access to the pixel buffer.
    // Format: BGRA or BGRX depending on the window depth (32-bit or 24-bit).
    // Row stride is always width * 4 bytes (no padding).
    pub data: &mut [u8],
}

impl X11Window {
    // Allocate a shared memory image with the given dimensions.
    // Fails if XShm is unavailable or if shmget/shmat fails.
    pub fn create_shm_image(
        &self,
        width:  u32,
        height: u32,
    ): Result[ShmImage, X11Error] ! IO

    // Copy a region of a shared memory image to the window's drawable.
    // src: the region within img to copy (in pixels).
    // dst: the top-left destination in the window's coordinate space.
    // The server reads directly from the shared segment; no pixel bytes
    // cross the X socket.
    pub fn put_shm_image(
        &mut self,
        img: &ShmImage,
        src: Rect,
        dst: Point,
    ) ! IO
}
```

### Typical use pattern

```ferrum
// Render loop for a game or media application using XShm.
fn game_loop(win: &mut X11Window): Result[(), X11Error] ! IO {
    let mut img = win.create_shm_image(win.width(), win.height())?

    loop {
        // Write pixels directly into img.data; no allocation, no copy.
        render_frame(&mut img.data, img.width, img.height)

        let full = Rect { x: 0, y: 0, width: img.width, height: img.height }
        win.put_shm_image(&img, full, Point { x: 0, y: 0 })?

        match win.poll_events() {
            events if events.iter().any(|e| matches!(e, X11Event::DestroyNotify)) => break,
            _ => {}
        }
    }

    Ok(())
}
```

`ShmImage` implements `Drop`, which calls `shmdt` and `shmctl(IPC_RMID)` and
detaches the segment from the server via `XShmDetach`.

---

## 7. Damage Extension

The Damage extension tracks which regions of a drawable have been modified. On remote
X11 connections this reduces bandwidth: the application can request repaint only for
the changed areas rather than the full window.

```ferrum
// A damage tracking object associated with a drawable.
pub type X11Damage

impl X11Window {
    // Create a damage tracking object for this window.
    // Damage events will be delivered as X11Event::DamageNotify in the event stream.
    // Fails if the Damage extension is unavailable.
    pub fn create_damage(&mut self): Result[X11Damage, X11Error]
}

impl X11Damage {
    // Mark the given screen regions as damaged (dirty).
    // The window manager or compositor will schedule a repaint of these regions.
    // Useful when the application maintains its own repaint queue and wants to
    // drive damage notification itself.
    pub fn add(&mut self, regions: &[Rect]) ! IO

    // Subtract regions from the current damage set.
    // Call after repainting a region to clear its damage flag.
    pub fn subtract(&mut self, regions: &[Rect]) ! IO

    // Query the current damage region (the set of rectangles that are dirty).
    pub fn query(&self): Result[Vec[Rect], X11Error] ! IO
}
```

### Damage-driven repaint loop

```ferrum
fn repaint_loop(win: &mut X11Window): Result[(), X11Error] ! IO {
    let mut damage = win.create_damage()?

    loop {
        let event = win.wait_event()?
        match event {
            X11Event::Expose { x, y, width, height } => {
                // Repaint only the exposed rectangle.
                repaint_region(win, Rect { x, y, width, height })?
                damage.subtract(&[Rect { x, y, width, height }])?
            }
            X11Event::DestroyNotify => break,
            _ => {}
        }
    }
    Ok(())
}
```

---

## 8. High-DPI and Xft DPI

X11 does not have a built-in logical coordinate system. Physical pixel coordinates are
used throughout the protocol. HiDPI support requires the application to read the display
DPI, compute a scale factor, and render at the appropriate physical resolution.

### DPI detection

```ferrum
impl X11Connection {
    // DPI of the default screen.
    //
    // Detection order:
    //   1. Xft.dpi X resource (set by most desktop environments for HiDPI).
    //   2. Screen physical dimensions from the X server (DisplayWidthMM /
    //      DisplayWidth).  Unreliable on many embedded setups where EDID
    //      is absent or incorrect.
    //   3. Fall back to 96.0.
    pub fn dpi(&self): f32
}
```

### Scale factor

```ferrum
// Scale factor relative to the 96 dpi baseline.
// A 192 dpi display returns 2.0 (HiDPI ×2).
// A 96 dpi display returns 1.0.
// A 144 dpi display returns 1.5.
pub fn X11Connection::scale_factor(&self): f32 {
    self.dpi() / 96.0
}
```

### Coordinate mapping

```ferrum
impl X11Window {
    // Convert a logical point (in scale-independent units) to physical pixels.
    // logical_x, logical_y are in the 96-dpi coordinate space.
    // Returns (physical_x, physical_y) suitable for X11 protocol calls.
    pub fn logical_to_physical(&self, x: f32, y: f32): (i32, i32)

    // Convert physical pixels back to logical coordinates.
    pub fn physical_to_logical(&self, x: i32, y: i32): (f32, f32)
}
```

### HiDPI rendering strategies

Two approaches are viable; which to choose depends on the application:

**XRender at physical resolution.** Pass physical pixel dimensions to all XRender calls.
Font rasterization and shape rendering happen at the native display resolution.
Sharpest result; no scaling artifact.

**Offscreen pixmap at 2× then scale down.** Render to a pixmap twice the logical size,
then scale down to the window via XRender's `XRenderSetPictureFilter` with `"bilinear"`.
Useful when the rendering pipeline is expressed in logical coordinates and converting
all coordinates is impractical.

---

## 9. Input Handling

### Event polling

```ferrum
impl X11Window {
    // Return all pending events without blocking.
    // Returns an empty Vec if no events are queued.
    pub fn poll_events(&mut self): Vec[X11Event] ! IO

    // Block until at least one event arrives, then return it.
    pub fn wait_event(&mut self): X11Event ! IO
}
```

### `X11Event` enum

```ferrum
@derive(Debug, Clone)
pub enum X11Event {
    // A key was pressed.
    KeyPress {
        keycode:  u8,        // raw X keycode
        keysym:   u32,       // X keysym after modifier mapping
        string:   String,    // UTF-8 string from XLookupString / Xutf8LookupString
        mods:     Modifiers,
    },

    // A key was released.
    KeyRelease {
        keycode:  u8,
        keysym:   u32,
        mods:     Modifiers,
    },

    // A pointer button was pressed.
    ButtonPress {
        button:   u32,   // 1 = left, 2 = middle, 3 = right, 4/5 = scroll wheel
        x:        i32,   // pointer x in window coordinates
        y:        i32,   // pointer y in window coordinates
        x_root:   i32,   // pointer x in screen coordinates
        y_root:   i32,
        mods:     Modifiers,
    },

    // A pointer button was released.
    ButtonRelease {
        button:   u32,
        x:        i32,
        y:        i32,
        mods:     Modifiers,
    },

    // The pointer moved within the window.
    MotionNotify {
        x:        i32,
        y:        i32,
        x_root:   i32,
        y_root:   i32,
        mods:     Modifiers,
    },

    // The pointer entered the window area.
    EnterNotify { x: i32, y: i32 },

    // The pointer left the window area.
    LeaveNotify { x: i32, y: i32 },

    // The window received keyboard focus.
    FocusIn,

    // The window lost keyboard focus.
    FocusOut,

    // A region of the window needs to be repainted.
    // Generated when the window is first mapped and when previously obscured
    // regions become visible.
    Expose { x: i32, y: i32, width: u32, height: u32 },

    // The window was resized or moved.
    // width and height are the new physical pixel dimensions.
    ConfigureNotify { x: i32, y: i32, width: u32, height: u32 },

    // A ClientMessage arrived (WM_DELETE_WINDOW, _NET_WM_PING, etc.).
    ClientMessage { atom: u32, data: [u8; 20] },

    // The window was destroyed by the server or window manager.
    DestroyNotify,

    // A tracked damage region was reported (requires X11Damage to be created).
    DamageNotify { x: i32, y: i32, width: u32, height: u32 },
}
```

### Modifiers

```ferrum
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Modifiers {
    pub shift:    bool,
    pub control:  bool,
    pub alt:      bool,    // Mod1
    pub super_:   bool,    // Mod4 (Windows/Command key)
    pub num_lock: bool,
    pub caps_lock: bool,
}
```

### Keycode to Unicode

Ferrum's X11 input layer translates raw X keycodes to Unicode strings automatically.
`X11Event::KeyPress::string` contains the UTF-8 text produced by the key press,
respecting the input method and current modifier state, derived from
`Xutf8LookupString` (with XIM support when an input method is running) or
`XLookupString` as a fallback. Dead key composition is handled transparently.

For direct keysym access the `keysym` field is available. Application code that
needs to distinguish "press of the Left arrow key" from "press of the 'A' key" should
test `keysym`, not `string`.

### WM_DELETE_WINDOW

The X11 close-button convention requires the application to register interest in
`WM_DELETE_WINDOW` and handle the resulting `ClientMessage`. `X11Window::create_window`
registers this protocol automatically. When the user clicks the window close button,
the event stream delivers:

```ferrum
X11Event::ClientMessage {
    atom: WM_DELETE_WINDOW_ATOM,
    ..
}
```

The application is responsible for destroying the window and exiting in response.

---

## 10. Clipboard (ICCCM / EWMH)

X11 clipboard semantics differ from other platforms. Ownership of a selection is held
by a live window; the owner must respond to `SelectionRequest` events from other
clients who want to convert the selection to a specific target type (e.g., `UTF8_STRING`).
This module implements the full selection owner protocol transparently.

```ferrum
pub enum X11Selection {
    // The PRIMARY selection: set automatically when text is selected with the mouse.
    // Middle-click pastes from PRIMARY.
    Primary,

    // The CLIPBOARD selection: set explicitly by the user (Ctrl+C or Edit > Copy).
    // Ctrl+V pastes from CLIPBOARD.
    Clipboard,

    // SECONDARY: rarely used; available for completeness.
    Secondary,
}
```

### Reading the clipboard

```ferrum
impl X11Connection {
    // Retrieve the current contents of a selection as UTF-8 text.
    //
    // Sends a ConvertSelection request for the UTF8_STRING target, processes
    // the SelectionNotify event, and reads the property data.
    // Falls back to the STRING target (Latin-1) if UTF8_STRING is unavailable.
    //
    // Blocks until the selection owner responds or a timeout expires.
    // Returns X11Error::NoDisplay if no owner holds the selection.
    pub fn clipboard_get(
        &mut self,
        selection: X11Selection,
    ): Result[String, X11Error] ! IO
}
```

### Writing the clipboard

```ferrum
impl X11Connection {
    // Claim ownership of a selection and make text available to other clients.
    //
    // This module takes ownership of the X selection on behalf of the calling
    // window, stores the text, and processes incoming SelectionRequest events
    // automatically until another client claims ownership or the connection
    // closes.  The caller does not need to handle SelectionRequest events.
    //
    // Advertised targets: UTF8_STRING, STRING, TEXT, TARGETS, TIMESTAMP.
    pub fn clipboard_set(
        &mut self,
        selection: X11Selection,
        text:      &str,
    ): Result[(), X11Error] ! IO
}
```

### Selection ownership lifetime

Selection ownership is maintained by a background task tied to the `X11Connection`.
When another application claims the selection, ownership is silently released. The
stored text is dropped at that point. Applications that need to know when ownership
was lost can poll `X11Connection::owns_selection(selection)`.

---

## 11. EWMH / ICCCM Integration

### Window type hints

```ferrum
pub enum WmWindowType {
    // An ordinary application window.  Default.
    Normal,

    // A secondary window for a specific task, typically modal.
    Dialog,

    // A panel anchored to a screen edge.
    Dock,

    // A toolbar torn off from a main application window.
    Toolbar,

    // A menu window (popup or dropdown).
    Menu,

    // A small utility window (color picker, palette).
    Utility,

    // A splash screen shown during application startup.
    Splash,

    // A window that covers the entire desktop background.
    Desktop,
}

impl X11Window {
    // Set _NET_WM_WINDOW_TYPE.  Must be called before the window is mapped
    // for the hint to take effect with most window managers.
    pub fn set_window_type(&mut self, wtype: WmWindowType) ! IO
}
```

### WM state

```ferrum
pub enum WmState {
    // Window is visible and interactive.
    Normal,

    // Window is iconified (minimized) in the taskbar.
    Iconic,

    // Window is withdrawn: not managed by the WM and not visible.
    Withdrawn,
}

impl X11Window {
    pub fn set_wm_state(&mut self, state: WmState) ! IO
}
```

### `_NET_WM_STATE` atoms

```ferrum
impl X11Window {
    // Request fullscreen mode via _NET_WM_STATE_FULLSCREEN.
    pub fn set_fullscreen(&mut self, yes: bool) ! IO

    // Request maximized (horizontal and vertical) via _NET_WM_STATE_MAXIMIZED_*.
    pub fn set_maximized(&mut self, yes: bool) ! IO

    // Request always-on-top via _NET_WM_STATE_ABOVE.
    pub fn set_above(&mut self, yes: bool) ! IO

    // Request always-on-bottom via _NET_WM_STATE_BELOW.
    pub fn set_below(&mut self, yes: bool) ! IO

    // Request stickiness (visible on all desktops) via _NET_WM_STATE_STICKY.
    pub fn set_sticky(&mut self, yes: bool) ! IO
}
```

### `_NET_WM_STRUT` for panels

Panel applications (docks, taskbars) use `_NET_WM_STRUT_PARTIAL` to reserve space
at a screen edge, preventing other windows from overlapping the panel area.

```ferrum
pub struct WmStrut {
    // Reserved pixels on each screen edge (0 = no reservation).
    pub left:   u32,
    pub right:  u32,
    pub top:    u32,
    pub bottom: u32,
}

impl X11Window {
    // Set _NET_WM_STRUT and _NET_WM_STRUT_PARTIAL.
    // Call after the window is mapped; most WMs re-read struts on PropertyNotify.
    pub fn set_strut(&mut self, strut: WmStrut) ! IO
}
```

---

## 12. Remote X11 Considerations

### Is the connection local?

```ferrum
impl X11Connection {
    // True when the display string refers to the local host.
    // ":N", "unix:N", and "/tmp/.X11-unix/XN" socket paths are local.
    // "hostname:N" is remote.
    pub fn is_local(&self): bool
}
```

### Request batching

Remote X11 connections have latency measured in milliseconds per round trip. The X
protocol is asynchronous — requests are sent without waiting for replies unless a reply
is explicitly requested or `XSync` is called. Ferrum's X11 module avoids inserting
unnecessary round trips:

- `XSync` is called only at explicit `X11Connection::sync()` calls or at window creation.
- `XFlush` is called after each batch of drawing operations, not after each individual call.
- Server replies are requested only when the caller uses an API that actually needs a value
  (e.g., `X11Damage::query()`).

On remote connections XShm is unavailable (`has_shm()` returns false) and the module
falls back to `XPutImage`, which does send pixel data across the socket.

### Compression

When the X11 connection uses TCP transport, LZ4 compression is negotiated
automatically if the server supports the `BIGREQ` and `LZO`/`LZ4` extensions (as
provided by some X server configurations). Compression is transparent; no API change
is required.

---

## 13. Error Types

```ferrum
@derive(Debug, Clone)]
pub enum X11Error {
    // $DISPLAY is unset and no display string was provided.
    NoDisplay {
        // The display string that was attempted, if any.
        display: Option[String],
    },

    // The X server refused the connection (authentication failed or server
    // is not running).
    ConnectionRefused,

    // A required extension is not available on this server.
    ExtensionNotFound {
        // The extension name, e.g. "RENDER", "MIT-SHM", "DAMAGE".
        name: String,
    },

    // The window XID referenced in a request is not valid.
    BadWindow,

    // The drawable XID referenced in a request is not valid.
    BadDrawable,

    // The X server ran out of memory processing a request.
    BadAlloc,

    // A low-level X protocol error was returned.
    ProtocolError {
        // X error code (from the X error event).
        code:   u8,
        // Minor opcode of the failed request.
        detail: u16,
    },

    // An OS-level error (shmget, shmat, etc.).
    OsError {
        op:    String,
        errno: i32,
    },
}

impl Display for X11Error { ... }
impl Error   for X11Error { ... }
```

---

## 14. Example Usage

### Create a window and render with XRender

```ferrum
use extlib::ccsp::draw_x11::{
    X11Backend, X11WindowConfig, X11Depth, X11Event,
    XRenderColor, PictOp,
}

pub fn main(): Result[(), X11Error] ! IO {
    let mut conn = X11Backend::connect()?

    let config = X11WindowConfig::new()
        .title("Hello X11")
        .class("hello_x11", "HelloX11")
        .size(800, 600)
        .depth(X11Depth::TrueColor32)
        .resizable(true)

    let mut win = conn.create_window(config)?
    win.show()?

    // Register WM_DELETE_WINDOW so close button sends ClientMessage.
    // (Registered automatically by create_window; shown here for clarity.)

    let bg = XRenderColor::from_rgba8(30, 30, 30, 255)
    let fg = XRenderColor::from_rgba8(200, 220, 255, 255)

    'main: loop {
        let pic = win.render_picture()

        // Clear the window to the background color.
        pic.fill_rect(PictOp::Src, bg, 0, 0, win.width() as u16, win.height() as u16)?

        // Draw a filled rectangle in the foreground color.
        pic.fill_rect(PictOp::Over, fg, 100, 100, 200, 50)?

        conn.flush()?

        for event in win.poll_events()? {
            match event {
                X11Event::KeyPress { keysym: 0xff1b, .. } => break 'main,  // Escape
                X11Event::ClientMessage { atom, .. } if atom == WM_DELETE_WINDOW_ATOM => {
                    break 'main
                }
                X11Event::ConfigureNotify { width, height, .. } => {
                    // Window was resized; update cached dimensions.
                    // win.width() / win.height() are updated automatically.
                }
                _ => {}
            }
        }
    }

    win.destroy()?
    Ok(())
}
```

### Game rendering loop with XShm

```ferrum
use extlib::ccsp::draw_x11::{X11Backend, X11WindowConfig, X11Event}

pub fn main(): Result[(), X11Error] ! IO {
    let mut conn = X11Backend::connect()?

    if !conn.has_shm() {
        // XShm unavailable: fall back to XPutImage or XRender.
        return Err(X11Error::ExtensionNotFound { name: "MIT-SHM".to_string() })
    }

    let config = X11WindowConfig::new()
        .title("Game")
        .size(1280, 720)

    let mut win = conn.create_window(config)?
    win.show()?

    let mut img = win.create_shm_image(1280, 720)?
    let full = Rect { x: 0, y: 0, width: 1280, height: 720 }

    'game: loop {
        // Render pixels directly into the shared memory buffer.
        render_game_frame(&mut img.data, img.width, img.height)

        // Zero-copy upload to the X server.
        win.put_shm_image(&img, full, Point { x: 0, y: 0 })?
        conn.flush()?

        for event in win.poll_events()? {
            match event {
                X11Event::KeyPress { keysym: 0xff1b, .. } => break 'game,
                X11Event::ClientMessage { .. } => break 'game,
                _ => {}
            }
        }
    }

    Ok(())
}
```

### Keyboard input with modifier tracking

```ferrum
use extlib::ccsp::draw_x11::{X11Backend, X11WindowConfig, X11Event, Modifiers}

fn handle_keyboard(win: &mut X11Window): Result[(), X11Error] ! IO {
    loop {
        let event = win.wait_event()?
        match event {
            X11Event::KeyPress { keysym, string, mods, .. } => {
                if mods.control && keysym == 0x71 {  // Ctrl+Q
                    break
                }
                if !string.is_empty() {
                    // Insert the typed text into the application.
                    handle_text_input(&string)
                }
            }
            X11Event::DestroyNotify => break,
            _ => {}
        }
    }
    Ok(())
}
```

### Clipboard read and write

```ferrum
use extlib::ccsp::draw_x11::{X11Backend, X11Selection}

fn clipboard_example(): Result[(), X11Error] ! IO {
    let mut conn = X11Backend::connect()?

    // Read current clipboard contents.
    let text = conn.clipboard_get(X11Selection::Clipboard)?
    println("Clipboard: {}", text)

    // Write new contents to the clipboard.
    conn.clipboard_set(X11Selection::Clipboard, "Hello from Ferrum")?

    Ok(())
}
```

---

## 15. Dependencies

### Extended library dependencies

| Module | Why required |
|--------|-------------|
| `extlib::draw` | `DrawBackend` trait that `X11Window` implements; `Rect`, `Point`, color types shared with other backends |
| `extlib::input` | Shared `KeyEvent`, `PointerEvent` types for cross-backend input handling |

### Standard library dependencies

| Module | Used for |
|--------|---------|
| `stdlib::io` | `! IO` effect; file descriptor management |
| `stdlib::alloc` | `Vec`, `String`, heap allocation |
| `stdlib::sys` | OS-level shared memory (`shmget`, `shmat`, `shmctl`) |
| `stdlib::ffi` | C interop for Xlib, XRender, XShm, Damage |

### External C libraries

| Library | Purpose |
|---------|---------|
| `libX11` | Core X11 protocol: display connection, window creation, events, properties |
| `libXrender` | XRender extension: antialiased compositing, gradients, glyph sets |
| `libXext` | XShm extension: shared memory pixmaps and images |
| `libXdamage` | Damage extension: dirty region tracking for partial repaints |

All four libraries are dynamically linked at runtime. If a library is absent the
corresponding `X11Connection::has_*()` method returns false and operations requiring
that extension return `X11Error::ExtensionNotFound`.

### Effect requirements

| Operation | Effects |
|-----------|---------|
| `X11Backend::connect()` | `! IO` |
| `X11Connection::create_window()` | `! IO` |
| `X11Window::show()`, `hide()`, `destroy()` | `! IO` |
| `X11Window::poll_events()`, `wait_event()` | `! IO` |
| `X11Window::put_shm_image()` | `! IO` |
| `XRenderPicture::composite()`, `fill_rect()` | `! IO` |
| `GlyphSet::render()` | `! IO` |
| `X11Connection::clipboard_get()`, `clipboard_set()` | `! IO` |
| `X11Window::set_fullscreen()`, EWMH methods | `! IO` |

All public APIs that communicate with the X server carry `! IO`. Pure accessors
(`width()`, `height()`, `is_local()`, `dpi()`, `has_shm()`, etc.) do not carry
`! IO`; they read state cached at connection time or after the last event was
processed.
