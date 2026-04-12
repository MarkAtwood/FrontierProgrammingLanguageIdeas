# Ferrum Extended Library — `draw_wayland` Module

**Module path:** `extlib::draw_wayland`
**Provided by:** `lib_ccsp_draw_wayland`
**Depends on:** `extlib::draw` (DrawBackend trait), `extlib::draw_gpu` (optional, Vulkan path),
`extlib::input`, `stdlib::sys::posix` (mmap, file descriptors)
**External C libraries:** `libwayland-client`

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [WaylandBackend and Connection](#2-waylandbackend-and-connection)
3. [WindowConfig Builder](#3-windowconfig-builder)
4. [xdg-shell Window Management](#4-xdg-shell-window-management)
5. [High-DPI Support](#5-high-dpi-support)
6. [Software Rendering Path — wl_shm](#6-software-rendering-path--wl_shm)
7. [GPU Rendering Path — EGL and Vulkan](#7-gpu-rendering-path--egl-and-vulkan)
8. [Input Handling via wl_seat](#8-input-handling-via-wl_seat)
9. [IME Support via zwp-text-input-v3](#9-ime-support-via-zwp-text-input-v3)
10. [Clipboard and Drag-and-Drop](#10-clipboard-and-drag-and-drop)
11. [wlr-layer-shell — Panels and Overlays](#11-wlr-layer-shell--panels-and-overlays)
12. [Event Loop Integration](#12-event-loop-integration)
13. [Error Types](#13-error-types)
14. [Example Usage](#14-example-usage)
15. [Dependencies Reference](#15-dependencies-reference)

---

## 1. Overview and Rationale

### Why Wayland Is the Primary Linux Display Backend

Wayland is the default display server protocol on every major Linux desktop distribution
in 2026. Fedora switched to Wayland by default in Fedora 25 (2016). Ubuntu made it the
default in Ubuntu 22.04 LTS. GNOME has defaulted to Wayland since GNOME 3.36 (2020). KDE
Plasma followed. As of 2026, a Ferrum program targeting a fresh Linux desktop installation
will run on Wayland unless the user has explicitly opted out.

X11 is still present on these systems via XWayland — an X11 server implemented as a
Wayland client. XWayland provides compatibility for legacy X11 applications. Writing new
application code that targets the X11 protocol directly means targeting XWayland, which
adds an indirection layer, loses access to Wayland-native features (fractional scaling,
compositor-side decorations, damage tracking, GPU zero-copy), and carries the full weight
of the X11 protocol surface. There is no engineering reason to do this for a new
application.

**Use this module (`draw_wayland`) when:** you are writing a Linux desktop application,
GUI toolkit, game, creative tool, or any program that opens windows and renders content
on a Linux desktop. This is the right default for interactive Linux applications.

**Use `draw_x11` when:** you need to support Linux systems without a Wayland compositor
(headless servers, very old distributions, some embedded Linux configurations), or when
you need X11 extension protocols (XInput2 for tablet pressure, XRandR for explicit
display management) that have no Wayland equivalent yet.

**Use `draw_fb` when:** you are running on bare Linux with no display server (framebuffer
console, embedded kiosk, recovery console), or on a platform where neither Wayland nor
X11 is available.

### What Protocols This Module Covers

| Protocol | Role |
|----------|------|
| `libwayland-client` | Core Wayland protocol — object lifecycle, event dispatch, `wl_surface`, `wl_compositor` |
| `xdg-shell` | Standard window management — toplevel windows, popups, configure/configure-ack cycle |
| `xdg-decoration-unstable-v1` | Negotiate client-side vs. server-side window decorations |
| `wp-viewporter` | Logical-to-physical coordinate mapping for non-integer scale factors |
| `wp-fractional-scale-v1` | Fractional display scale factors (e.g., 1.5x, 1.25x) |
| `wl_shm` | Software rendering via shared memory buffers |
| `linux-dmabuf-unstable-v1` | GPU zero-copy: share DMA-BUF textures from GPU to compositor |
| `wl_seat`, `wl_pointer`, `wl_keyboard`, `wl_touch` | Pointer, keyboard, and touch input |
| `wl_data_device` | Clipboard and drag-and-drop |
| `zwp-text-input-v3` | IME protocol for CJK and other complex input methods |
| `zwlr-layer-shell-v1` | Layer surfaces for panels, overlays, notification windows (wlroots compositors) |

EGL and Vulkan integration (for GPU rendering) uses `wl_egl_window` and
`VK_KHR_wayland_surface` respectively, both of which operate over the standard
`wl_surface` Wayland object.

---

## 2. WaylandBackend and Connection

### `WaylandBackend`

The module entry point. Connects to a running Wayland compositor.

```ferrum
use extlib::draw_wayland::{WaylandBackend, WaylandConnection, WaylandError}
use stdlib::fs::Path

pub type WaylandBackend
```

```ferrum
impl WaylandBackend {
    // Connect to the compositor identified by $WAYLAND_DISPLAY.
    // $WAYLAND_DISPLAY is typically "wayland-0" or "wayland-1";
    // the compositor socket is found under $XDG_RUNTIME_DIR.
    //
    // Returns an error if $WAYLAND_DISPLAY is unset, the socket does not
    // exist, or the initial protocol handshake fails.
    pub fn connect(): Result[WaylandConnection, WaylandError] ! IO

    // Connect to a specific compositor socket path.
    // Use this when $WAYLAND_DISPLAY is not set or when testing against
    // a nested compositor socket.
    pub fn connect_to(socket: &Path): Result[WaylandConnection, WaylandError] ! IO
}
```

### `WaylandConnection`

Represents an active connection to a Wayland compositor. Owns the Wayland display
object and the global registry. Multiple windows can be created from a single connection.

```ferrum
pub type WaylandConnection
```

```ferrum
impl WaylandConnection {
    // Create a new top-level window.
    // The window is not visible until the first commit following a
    // configure-ack cycle with the compositor.
    pub fn create_window(
        config: WindowConfig
    ): Result[WaylandWindow, WaylandError] ! IO

    // Create a layer surface (panels, overlays).
    // Requires zwlr-layer-shell-v1 support from the compositor.
    // Returns WaylandError::GlobalNotFound("zwlr_layer_shell_v1") when absent.
    pub fn create_layer_surface(
        config: LayerSurfaceConfig
    ): Result[LayerSurface, WaylandError] ! IO

    // Access the clipboard.
    pub fn clipboard(&self): &Clipboard

    // File descriptor that becomes readable when Wayland events are pending.
    // Use this to integrate Wayland event dispatch into a poll-based
    // or async event loop alongside other file descriptors.
    pub fn event_fd(&self): RawFd

    // Dispatch all pending events, invoking registered callbacks.
    // Should be called whenever event_fd is readable, or in a tight loop
    // for simple synchronous programs.
    pub fn dispatch_events(&mut self): Result[(), WaylandError] ! IO

    // Enumerate connected outputs (monitors).
    pub fn outputs(&self): &[WaylandOutput]
}
```

### `WaylandOutput`

Describes a connected display output (monitor).

```ferrum
pub struct WaylandOutput {
    // Compositor-assigned name, e.g. "HDMI-A-1", "eDP-1".
    pub name:             String,

    // Human-readable description, e.g. "Dell U2720Q".
    pub description:      String,

    // Current scale factor as advertised by wp-fractional-scale-v1.
    // 1.0 for a 96 dpi display, 2.0 for HiDPI, 1.5 for fractional.
    pub scale:            f32,

    // Refresh rate in millihertz (e.g., 60000 for 60 Hz, 144000 for 144 Hz).
    pub refresh_rate_mhz: u32,

    // Physical dimensions of the panel in millimetres.
    // (0, 0) when the compositor does not report physical size.
    pub physical_size_mm: (u32, u32),

    // Current mode resolution in physical pixels.
    pub resolution_px:    (u32, u32),
}
```

### `WaylandWindow` implements `DrawBackend`

```ferrum
pub type WaylandWindow

impl DrawBackend for WaylandWindow
```

All drawing operations from `extlib::draw` are available through this implementation.
Physical pixel dimensions for the canvas are `logical_size * scale_factor`.

---

## 3. WindowConfig Builder

`WindowConfig` configures a window before creation. Every option has a documented
default. Only set what you need.

```ferrum
use extlib::draw_wayland::{
    WindowConfig, DecorationMode, RenderBackend, Size
}

pub type WindowConfig
```

```ferrum
impl WindowConfig {
    // Start a builder with sensible defaults:
    //   title:       ""
    //   app_id:      ""  (should be set for desktop integration)
    //   size:        800 x 600 logical pixels
    //   resizable:   true
    //   decorations: DecorationMode::ServerSide
    //   render:      RenderBackend::Software
    //   fullscreen:  false
    //   maximized:   false
    pub fn new(): Self

    // Window title shown in the compositor's title bar or task switcher.
    pub fn title(mut self, title: &str): Self

    // Application identifier in reverse-DNS format, e.g. "com.example.MyApp".
    // Used by the compositor for desktop integration: task switcher grouping,
    // .desktop file matching, window rules, and persistence of window geometry.
    // Every deployed application should set this.
    pub fn app_id(mut self, id: &str): Self

    // Initial window size in logical pixels.
    // The compositor may send a configure event with different dimensions
    // before the first frame is displayed.  The application must handle this.
    pub fn size(mut self, width: f32, height: f32): Self

    // Minimum window size in logical pixels.  The compositor will not allow
    // the user to resize below this.  Default: no minimum.
    pub fn min_size(mut self, size: Size): Self

    // Maximum window size in logical pixels.  Default: no maximum.
    pub fn max_size(mut self, size: Size): Self

    // When false, the compositor does not offer resize handles and the
    // XDG toplevel is configured as non-resizable.  Default: true.
    pub fn resizable(mut self, resizable: bool): Self

    // How window decorations (title bar, borders, close/minimize buttons)
    // are drawn.  See DecorationMode.
    pub fn decorations(mut self, mode: DecorationMode): Self

    // Select the rendering backend.  See RenderBackend.
    pub fn render_backend(mut self, backend: RenderBackend): Self

    // Start the window in fullscreen mode on the given output.
    // None means the compositor chooses the output.
    pub fn fullscreen(mut self, output: Option[&WaylandOutput]): Self

    // Start the window maximized.  Default: false.
    pub fn maximized(mut self, maximized: bool): Self
}
```

### `Size`

```ferrum
pub struct Size {
    pub width:  f32,
    pub height: f32,
}

impl Size {
    pub fn new(width: f32, height: f32): Self
}
```

### `DecorationMode`

```ferrum
pub enum DecorationMode {
    // Request that the compositor draw the title bar, resize handles, and
    // close/minimize buttons.  This is the default and is preferred when
    // available: it gives a native look consistent with other windows,
    // works correctly with compositor effects (blur, shadow, rounded corners),
    // and requires no drawing code in the application.
    //
    // Not all compositors support server-side decorations.  When the compositor
    // does not support xdg-decoration-unstable-v1, Ferrum falls back to
    // ClientSide automatically and reports DecorationMode::ClientSide via
    // WaylandWindow::active_decoration_mode().
    ServerSide,

    // The application draws its own title bar and borders inside the window
    // surface.  Required for custom-styled applications (e.g., a game or a
    // design tool that embeds the title bar in its own chrome).
    ClientSide,

    // No decorations at all.  The window has no title bar or borders.
    // Useful for splash screens, fullscreen applications, and custom-shaped
    // overlay windows.
    None,
}
```

### `RenderBackend`

```ferrum
pub enum RenderBackend {
    // CPU-rendered pixels via wl_shm shared memory.
    // Always available.  Suitable for UI-heavy applications, text editors,
    // and any workload where GPU overhead exceeds the drawing cost.
    Software,

    // OpenGL ES 3.0 via EGL.  Requires a compositor that supports EGL
    // and a GPU with GLES3 drivers.
    OpenGlEs3,

    // Vulkan via VK_KHR_wayland_surface.  Requires extlib::draw_gpu and
    // a Vulkan-capable GPU.  Provides access to the full draw_gpu extlib API.
    Vulkan,
}
```

---

## 4. xdg-shell Window Management

xdg-shell is the stable Wayland extension protocol for application windows. It defines
the `xdg_surface` and `xdg_toplevel` objects that implement the window management
contract between the application and compositor: configure events (resize, maximize,
fullscreen), acknowledgement, and commit.

### Window Management Methods

```ferrum
impl WaylandWindow {
    // Update the window title at runtime.
    pub fn set_title(&mut self, title: &str): () ! IO

    // Enter fullscreen on the specified output.
    // None lets the compositor choose the output (typically the one the
    // window currently occupies).
    pub fn set_fullscreen(
        &mut self,
        output: Option[&WaylandOutput]
    ): () ! IO

    // Exit fullscreen and return to the previous windowed state.
    pub fn unset_fullscreen(&mut self): () ! IO

    // Request that the compositor maximize the window.
    pub fn maximize(&mut self): () ! IO

    // Restore the window from maximized state.
    pub fn unmaximize(&mut self): () ! IO

    // Request that the compositor minimize (iconify) the window.
    // The compositor may ignore this on some desktop environments.
    pub fn minimize(&mut self): () ! IO

    // The decoration mode currently in effect.
    // May differ from the requested mode if the compositor fell back
    // from ServerSide to ClientSide (see DecorationMode::ServerSide).
    pub fn active_decoration_mode(&self): DecorationMode

    // Current logical size in logical pixels.
    // This is the size most recently acknowledged in the configure cycle.
    pub fn logical_size(&self): Size

    // Register a callback to be invoked whenever the compositor sends a
    // configure event requesting a resize or state change.
    // The callback receives the new logical size and a ConfigureState.
    // The application must call acknowledge_configure() and then redraw
    // and commit before the compositor will display the new frame.
    pub fn on_configure(
        &mut self,
        f: impl Fn(Size, ConfigureState) + 'static
    )

    // Acknowledge the most recent configure event.
    // Must be called before committing a frame in response to a configure.
    pub fn acknowledge_configure(&mut self): () ! IO
}
```

### `ConfigureState`

```ferrum
pub struct ConfigureState {
    pub maximized:   bool,
    pub fullscreen:  bool,
    pub activated:   bool,   // window has keyboard focus
    pub resizing:    bool,   // user is actively dragging a resize handle
}
```

### xdg-decoration Protocol Detail

The `xdg-decoration-unstable-v1` protocol negotiation works as follows:

1. When `DecorationMode::ServerSide` is requested, the module sends
   `zxdg_toplevel_decoration_v1.set_mode(server_side)` to the compositor.

2. The compositor responds with a `configure` event carrying its preferred mode.
   Some compositors (particularly those running tiling window managers like sway)
   always respond with `client_side` regardless of the request.

3. The module stores the negotiated mode and reports it via
   `active_decoration_mode()`. The application's `on_configure` callback fires
   so the application can adjust its layout (e.g., add title bar padding when
   the compositor forces client-side decorations).

4. When `DecorationMode::None` is requested, the module does not send the
   `set_mode` request and instead uses `xdg_surface`'s `set_window_geometry`
   to set a zero-margin input region, giving an undecorated surface.

---

## 5. High-DPI Support

### wp-fractional-scale-v1 and wp-viewporter

Modern displays use non-integer scale factors — a 4K laptop screen at 200% reports
scale 2.0, but a 2560x1440 monitor configured at 150% reports scale 1.5. Wayland's
original integer-only `wl_output.scale` cannot express fractional values. The
`wp-fractional-scale-v1` protocol extension adds fractional scale reporting.
`wp-viewporter` provides the coordinate mapping that allows the application to render
to a physical-pixel-sized buffer and have the compositor display it at the logical
size.

This module uses both extensions together:

- `wp_fractional_scale_v1` subscribes to scale change events per surface.
- `wp_viewport` maps the physical-pixel surface to the logical pixel dimensions that
  the compositor uses for layout.

The result: application code works in logical pixels throughout. The module translates
to and from physical pixels transparently.

### Scale Factor API

```ferrum
impl WaylandWindow {
    // The current effective scale factor for this window.
    // This is the scale factor of the output the window primarily occupies.
    // Multiply logical sizes by this value to get physical pixel counts.
    // Example: a 200x150 logical window at scale 1.5 has a 300x225 physical buffer.
    pub fn scale_factor(&self): f32

    // Register a callback invoked when the scale factor changes.
    // This happens when the window moves to a different output (monitor),
    // when the user changes the display scale setting, or when a new
    // monitor is connected.
    // The callback receives the new scale factor.
    // The application should re-render at the new scale and commit.
    pub fn on_scale_change(
        &mut self,
        f: impl Fn(f32) + 'static
    )
}
```

### Coordinate System

All public API coordinates are in **logical pixels**. Physical pixels are an
implementation detail of the rendering path.

| Quantity | Unit |
|----------|------|
| `WindowConfig::size`, `min_size`, `max_size` | Logical pixels |
| `ConfigureState` dimensions | Logical pixels |
| `PointerEvent::position` | Logical pixels |
| `Rect` in damage tracking | Physical pixels (see §6) |
| Software frame buffer stride | Physical pixels |
| EGL/Vulkan surface dimensions | Physical pixels |

The one exception — physical-pixel damage rects — matches the Wayland protocol
requirement: `wl_surface.damage_buffer` takes physical pixel coordinates.

---

## 6. Software Rendering Path — wl_shm

`wl_shm` shared memory rendering is the universally available fallback. No GPU, EGL,
or Vulkan support is required. The compositor and application share anonymous memory
(`memfd_create` on Linux); the application writes pixel data and the compositor reads
it during composition.

### Double Buffering

The module maintains two `wl_shm_pool` buffers per window (front and back). While the
compositor is reading the front buffer to display the current frame, the application
writes the next frame into the back buffer. On `commit()`, the buffers are swapped.
The module never writes to a buffer currently held by the compositor.

### Frame Buffer Access

```ferrum
impl WaylandWindow {
    // Request a writable view of the back buffer.
    // The slice covers the full window in RGBA8 format (R at byte 0,
    // A at byte 3) in row-major order, physical pixels.
    //
    // Width in bytes = physical_width * 4.
    // Total length   = physical_width * physical_height * 4.
    //
    // This call is only valid when RenderBackend::Software is active.
    // Calling it on a GPU-backed window returns WaylandError::WrongBackend.
    pub fn software_frame_buffer(
        &mut self
    ): Result[&mut [u8], WaylandError]

    // Physical pixel dimensions of the frame buffer.
    // Equals logical_size() * scale_factor(), rounded up to the nearest integer.
    pub fn physical_size(&self): (u32, u32)
}
```

### Damage Tracking

Wayland requires the application to declare which regions of the surface have changed
since the last committed frame. The compositor uses this information to skip
recompositing unchanged regions, which saves GPU bandwidth and improves performance.

```ferrum
impl WaylandWindow {
    // Declare damaged (changed) regions in physical pixels before committing.
    // wl_surface.damage_buffer() takes physical pixel coordinates per the
    // Wayland protocol specification; logical-pixel damage is not accepted.
    //
    // Calling commit() without calling damage() first damages the full surface
    // (equivalent to damage_all()), which is correct but inefficient.
    pub fn damage(&mut self, rects: &[Rect])

    // Damage the entire surface.  Equivalent to passing the full surface
    // bounding rectangle to damage().  Use when a full redraw has occurred.
    pub fn damage_all(&mut self)

    // Commit the current frame.  Swaps the back buffer to front, sends the
    // damage regions to the compositor, and queues the buffer for display.
    // After commit() returns, the back buffer is writable again for the
    // next frame.
    pub fn commit(&mut self): Result[(), WaylandError] ! IO

    // Register a callback invoked when the compositor signals it is ready
    // for the next frame (wl_surface.frame callback).  Use this instead of
    // rendering as fast as possible to avoid producing frames faster than
    // the compositor can display them.
    pub fn on_frame_ready(
        &mut self,
        f: impl Fn() + 'static
    )
}
```

### `Rect`

```ferrum
pub struct Rect {
    pub x:      i32,
    pub y:      i32,
    pub width:  i32,
    pub height: i32,
}
```

---

## 7. GPU Rendering Path — EGL and Vulkan

### EGL / OpenGL ES 3 Path

EGL is the API that connects OpenGL ES contexts to Wayland surfaces. The module creates
a `wl_egl_window` object wrapping the `wl_surface`, then uses EGL to create a
`EGLSurface` and `EGLContext` targeting GLES 3.0.

```ferrum
impl WaylandWindow {
    // Obtain the EGL surface for this window.
    // Only valid when RenderBackend::OpenGlEs3 is active.
    //
    // The returned EglSurface is ready for use with eglMakeCurrent and
    // eglSwapBuffers.  Physical pixel dimensions follow scale_factor().
    // Call resize_egl_surface() after scale changes.
    pub fn egl_surface(&self): Result[&EglSurface, WaylandError]

    // Resize the underlying wl_egl_window to match the current physical
    // pixel dimensions.  Must be called in response to configure events
    // and scale changes before the next swap.
    pub fn resize_egl_surface(&mut self): Result[(), WaylandError] ! IO
}

// An EGL surface and context pair bound to a Wayland window.
pub struct EglSurface {
    // Raw EGLSurface handle.  Type-erased pointer; cast to EGLSurface
    // when passing to raw EGL function calls via FFI.
    pub raw_surface: *mut (),

    // Raw EGLContext handle.
    pub raw_context: *mut (),

    // Raw EGLDisplay handle.
    pub raw_display: *mut (),
}

impl EglSurface {
    // Make this context current on the calling thread.
    pub fn make_current(&self): Result[(), WaylandError] ! Unsafe

    // Swap front and back buffers, presenting the rendered frame.
    // Respects the compositor's frame pacing if swap interval is 1
    // (the default).
    pub fn swap_buffers(&self): Result[(), WaylandError] ! IO
}
```

### Vulkan Path

Vulkan integration works through the `VK_KHR_wayland_surface` instance extension.
The module extracts the raw `wl_display` and `wl_surface` pointers needed for
`vkCreateWaylandSurfaceKHR` and packages them as a `VulkanSurface` for use with
`extlib::draw_gpu`.

```ferrum
impl WaylandWindow {
    // Obtain a Vulkan surface descriptor for this window.
    // Only valid when RenderBackend::Vulkan is active.
    //
    // The VulkanSurface carries the raw wl_display* and wl_surface*
    // pointers needed by VK_KHR_wayland_surface.  Pass this to
    // extlib::draw_gpu::VulkanDevice::create_swapchain().
    pub fn vulkan_surface(&self): Result[VulkanSurface, WaylandError]
}

// Wayland-specific Vulkan surface descriptor.
// Consumed by extlib::draw_gpu.
pub struct VulkanSurface {
    // Raw pointer to the wl_display.  Valid for the lifetime of
    // the WaylandConnection this window was created from.
    pub display: *mut (),

    // Raw pointer to the wl_surface.  Valid for the lifetime of
    // the WaylandWindow.
    pub surface: *mut (),

    // Required Vulkan instance extensions to enable:
    //   "VK_KHR_surface"
    //   "VK_KHR_wayland_surface"
    pub required_extensions: &'static [&'static str],
}
```

### wl_dmabuf — Zero-Copy GPU Textures

The `linux-dmabuf-unstable-v1` protocol allows a GPU to export a rendered frame as a
DMA-BUF file descriptor and hand it directly to the compositor without a readback
through CPU memory. The compositor imports the DMA-BUF and composites it on the GPU.

```ferrum
impl WaylandWindow {
    // Commit a DMA-BUF buffer as the next displayed frame.
    //
    // The DmaBufFrame is obtained from the GPU driver via
    // extlib::draw_gpu and carries the DMA-BUF file descriptors,
    // format (DRM fourcc), and modifier.
    //
    // The compositor takes temporary ownership of the buffer until
    // the next commit, at which point the release event fires and the
    // buffer can be reused.
    //
    // Returns WaylandError::GlobalNotFound("linux_dmabuf_v1") when
    // the compositor does not support the protocol.
    pub fn commit_dmabuf(
        &mut self,
        frame: DmaBufFrame
    ): Result[(), WaylandError] ! IO
}

pub struct DmaBufFrame {
    // File descriptors for each plane.  Maximum four planes.
    pub fds:      [Option[RawFd]; 4],

    // Byte offsets for each plane.
    pub offsets:  [u32; 4],

    // Row stride in bytes for each plane.
    pub strides:  [u32; 4],

    // Number of planes in use.
    pub n_planes: u32,

    // DRM fourcc pixel format code (e.g., DRM_FORMAT_ARGB8888 = 0x34325241).
    pub format:   u32,

    // DRM format modifier (e.g., DRM_FORMAT_MOD_LINEAR = 0).
    pub modifier: u64,

    // Physical pixel dimensions.
    pub width:    u32,
    pub height:   u32,
}
```

---

## 8. Input Handling via wl_seat

Wayland input is delivered through `wl_seat`, which aggregates all input devices
associated with a logical seat (typically one seat per user session). A seat may have
a pointer, keyboard, and touch device. The module subscribes to all available
capabilities.

### Event Types

```ferrum
pub enum WindowEvent {
    Pointer(PointerEvent),
    Key(KeyEvent),
    Touch(TouchEvent),
    TextInput(TextInputEvent),   // from zwp-text-input-v3; see §9
    CloseRequested,              // user clicked the close button
    FocusGained,
    FocusLost,
    Configure { size: Size, state: ConfigureState },
    ScaleChanged(f32),
    OutputEntered(WaylandOutput),
    OutputLeft(WaylandOutput),
}
```

### Pointer Events

```ferrum
pub struct PointerEvent {
    // Cursor position in logical pixels relative to the window's top-left corner.
    pub position: Point,

    // Button state change, if this event is a button press or release.
    // None for motion events.
    pub button:   Option[ButtonEvent],

    // Scroll delta, if this event is a scroll wheel or touchpad scroll.
    // None for motion and button events.
    pub scroll:   Option[ScrollDelta],
}

pub struct ButtonEvent {
    pub button: MouseButton,
    pub state:  ButtonState,
}

pub enum MouseButton {
    Left,
    Right,
    Middle,
    Back,
    Forward,
    Other(u32),
}

pub enum ButtonState { Pressed, Released }

pub struct ScrollDelta {
    // Discrete scroll in notches (mouse wheel).  None for smooth scroll.
    pub discrete: Option[(i32, i32)],   // (horizontal_notches, vertical_notches)

    // Continuous scroll in logical pixels (touchpad, high-resolution wheel).
    pub continuous: Option[(f32, f32)], // (horizontal_px, vertical_px)
}
```

### Keyboard Events

```ferrum
pub struct KeyEvent {
    // Platform-independent key code.  See extlib::input::KeyCode.
    pub keycode:   KeyCode,

    pub state:     KeyState,

    // Active modifier keys at the time of this event.
    pub modifiers: Modifiers,

    // UTF-8 text produced by this keypress after applying the keyboard layout
    // and modifiers.  None for non-printing keys (arrows, F-keys, modifiers).
    // When IME is active, use TextInputEvent::Commit instead; raw key text
    // is suppressed for pre-edit sequences.
    pub text:      Option[String],
}

pub enum KeyState { Pressed, Released, Repeat }

pub struct Modifiers {
    pub shift:   bool,
    pub ctrl:    bool,
    pub alt:     bool,
    pub logo:    bool,   // Super / Windows / Command key
    pub capslock: bool,
    pub numlock:  bool,
}
```

### Touch Events

```ferrum
pub struct TouchEvent {
    // Touch point identifier, stable across Down → Motion → Up for one finger.
    pub id:       u32,

    // Touch position in logical pixels relative to window top-left.
    pub position: Point,

    pub phase:    TouchPhase,
}

pub enum TouchPhase {
    Started,
    Moved,
    Ended,
    Cancelled,
}
```

### `Point`

```ferrum
pub struct Point {
    pub x: f32,
    pub y: f32,
}
```

### Polling and Callback APIs

```ferrum
impl WaylandWindow {
    // Collect all events queued since the last call.
    // Requires dispatch_events() to have been called on the owning
    // WaylandConnection to populate the queue.
    pub fn poll_events(&mut self): Vec[WindowEvent]

    // Register a callback invoked for each event as it is dispatched.
    // The callback and the poll_events() queue are independent; using
    // both simultaneously is allowed.
    pub fn on_event(
        &mut self,
        f: impl Fn(&WindowEvent) + 'static
    )
}
```

---

## 9. IME Support via zwp-text-input-v3

Raw keyboard events cannot represent CJK input correctly. When a user types Chinese,
Japanese, or Korean characters, the input method editor (IME) intercepts keystrokes,
shows a candidate window, and commits the final character sequence as a single unit.
The `zwp-text-input-v3` protocol provides the Wayland-native IME interface.

### Activating IME

```ferrum
impl WaylandWindow {
    // Declare the type of text input expected at the current focus point.
    // This hint allows the compositor and IME to configure themselves
    // appropriately (e.g., show a numeric keypad on mobile, disable spell
    // checking in a URL field).
    pub fn set_input_type(&mut self, input_type: InputType): () ! IO

    // Provide the IME with a hint about where the cursor is on screen,
    // in logical pixels relative to the window's top-left.
    // The compositor uses this to position the IME candidate window so
    // it does not overlap the text being edited.
    pub fn set_cursor_rect(&mut self, rect: Rect): () ! IO

    // Notify the IME of the text surrounding the cursor, so it can offer
    // context-aware suggestions.
    // text: up to 1000 bytes of UTF-8 surrounding text.
    // cursor_pos: byte offset of the cursor within text.
    // anchor_pos: byte offset of the selection anchor (equal to cursor_pos
    //             when there is no selection).
    pub fn set_surrounding_text(
        &mut self,
        text: &str,
        cursor_pos: u32,
        anchor_pos: u32,
    ): () ! IO
}
```

### `InputType`

```ferrum
pub enum InputType {
    // General text input.  Spell checking and auto-correction active.
    Text,

    // Multi-line text input (text editor, terminal).
    Multiline,

    // Numeric input only.
    Number,

    // Email address.  @ and . keys promoted on virtual keyboards.
    Email,

    // URL input.  / and . keys promoted; auto-correction suppressed.
    Url,

    // Search field.  Search suggestions from IME where supported.
    Search,

    // Password field.  IME candidate window suppressed; no surrounding text.
    Password,
}
```

### IME Events

IME events arrive as `WindowEvent::TextInput(TextInputEvent)`. They are delivered via
the same `poll_events()` queue and `on_event()` callback as other window events.

```ferrum
pub enum TextInputEvent {
    // A composition sequence is in progress.
    // text:         The current pre-edit string.  Display it at the cursor
    //               position with an underline or highlight to distinguish
    //               it from committed text.
    // cursor_begin: Byte offset of the cursor within text.
    // cursor_end:   Byte offset of the cursor end within text.
    //               Equals cursor_begin when there is no in-composition selection.
    //
    // A PreEdit with an empty text string clears the current composition.
    PreEdit {
        text:         String,
        cursor_begin: i32,
        cursor_end:   i32,
    },

    // The IME has finalized the composition and produced committed text.
    // Insert committed_text at the current cursor position and clear any
    // displayed pre-edit string.
    Commit {
        text: String,
    },

    // The IME requests deletion of text surrounding the cursor.
    // before_length: number of bytes to delete before the cursor.
    // after_length:  number of bytes to delete after the cursor.
    // This arrives before the corresponding Commit event when the IME
    // replaces text (e.g., autocorrect).
    DeleteSurrounding {
        before_length: u32,
        after_length:  u32,
    },
}
```

### IME Protocol Sequence

A typical CJK input sequence:

1. User focuses a text field → application calls `set_input_type(InputType::Text)` and `set_cursor_rect(...)`.
2. User types a consonant or syllable character.
3. Compositor delivers `TextInputEvent::PreEdit { text: "あ", ... }` — application renders the underlined pre-edit text.
4. User selects a kanji from the candidate window.
5. Compositor delivers `TextInputEvent::Commit { text: "東京" }` — application inserts the committed text and clears the pre-edit.

---

## 10. Clipboard and Drag-and-Drop

### Clipboard

```ferrum
use extlib::draw_wayland::Clipboard

pub type Clipboard
```

```ferrum
impl Clipboard {
    // Place UTF-8 text onto the clipboard selection.
    // The connection must remain alive and dispatch events while the text
    // is on the clipboard; Wayland clipboard ownership is maintained by
    // responding to compositor data requests.
    pub fn set_text(&mut self, text: &str): Result[(), WaylandError] ! IO

    // Read UTF-8 text from the clipboard.
    // Blocks (or suspends in async context) until the owning application
    // responds to the data request.
    // Returns WaylandError::NoData when the clipboard is empty or
    // does not contain text.
    pub fn get_text(&self): Result[String, WaylandError] ! IO

    // Place arbitrary MIME-typed data onto the clipboard.
    // The data argument is the raw bytes the compositor will forward to
    // the receiving application when it requests this MIME type.
    pub fn set_data(&mut self, mime_type: &str, data: Vec[u8]): () ! IO

    // Read data from the clipboard for the given MIME type.
    // Returns WaylandError::NoData when no selection offers that type.
    pub fn get_data(
        &self,
        mime_type: &str
    ): Result[Vec[u8], WaylandError] ! IO

    // List the MIME types currently available on the clipboard.
    // Returns an empty slice when the clipboard is empty.
    pub fn available_types(&self): Vec[String]
}
```

### Drag-and-Drop

```ferrum
// A drag source: the application initiates a drag carrying data.
pub type DragSource

impl DragSource {
    // Create a new drag source advertising the given MIME types.
    pub fn new(mime_types: Vec[String]): Self

    // Register a callback invoked when the compositor requests the data
    // for a specific MIME type (i.e., the user has dropped onto a target
    // that accepted the type).
    pub fn on_data_requested(
        &mut self,
        f: impl Fn(&str) -> Vec[u8] + 'static
    )

    // Begin the drag-and-drop operation.  Must be called in response to a
    // mouse button press event (the serial from that event is required by
    // the protocol).
    pub fn start_drag(
        &mut self,
        window: &WaylandWindow,
        button_serial: u32
    ): Result[(), WaylandError] ! IO
}

// A drop target: the application receives data from a drag.
pub trait DropTarget {
    // Called when a drag enters the window's surface.
    // mime_types: the MIME types offered by the drag source.
    // position:   logical pixel position within the window.
    //
    // Return Some(mime_type) to accept the drag for that type,
    // or None to reject.
    fn on_drag_enter(
        &mut self,
        mime_types: &[String],
        position: Point
    ): Option[String]

    // Called as the drag moves within the window.
    // Return Some(mime_type) to continue accepting, None to reject.
    fn on_drag_motion(
        &mut self,
        position: Point
    ): Option[String]

    // Called when the drag leaves the window.
    fn on_drag_leave(&mut self)

    // Called when the user drops onto this window.
    // data: the raw bytes for the accepted MIME type.
    fn on_drop(&mut self, data: Vec[u8], position: Point)
}

impl WaylandWindow {
    // Register a drop target for this window.
    // Replaces any previously registered target.
    pub fn set_drop_target(
        &mut self,
        target: impl DropTarget + 'static
    )
}
```

### `DropEvent` (polling alternative to the `DropTarget` trait)

For applications that prefer a polling model over trait dispatch:

```ferrum
pub enum DropEvent {
    Enter {
        mime_types: Vec[String],
        position:   Point,
    },
    Motion {
        position: Point,
    },
    Leave,
    Drop {
        mime_type: String,
        data:      Vec[u8],
        position:  Point,
    },
}
```

These are delivered in the `WindowEvent` queue as `WindowEvent::Drop(DropEvent)`
when no `DropTarget` trait object is registered.

---

## 11. wlr-layer-shell — Panels and Overlays

`zwlr-layer-shell-v1` is a wlroots compositor extension that allows applications to
place surfaces at fixed positions relative to the screen rather than as managed
application windows. This is the mechanism used by Wayland-native status bars (Waybar,
yambar), notification daemons, lock screen overlays, and desktop widgets.

**Availability:** wlr-layer-shell is not part of the core Wayland protocol. It is
supported by wlroots-based compositors (sway, Hyprland, labwc, river) and by GNOME
Shell since version 45. KDE Plasma does not support it as of 2026. Applications should
handle `WaylandError::GlobalNotFound("zwlr_layer_shell_v1")` gracefully.

### `LayerSurfaceConfig` Builder

```ferrum
pub type LayerSurfaceConfig

impl LayerSurfaceConfig {
    pub fn new(): Self

    // The compositor layer to place this surface on.
    pub fn layer(mut self, layer: Layer): Self

    // Application-defined namespace string identifying the type of surface.
    // E.g., "statusbar", "notification", "lockscreen".
    // Used by compositor window rules.
    pub fn namespace(mut self, ns: &str): Self

    // Which screen edges to anchor this surface to.
    // Anchoring to two opposite edges (e.g., Left | Right) causes the surface
    // to stretch to fill the width.
    pub fn anchor(mut self, anchor: Anchor): Self

    // Exclusive zone in logical pixels.
    //   Positive value: reserve this many pixels from the anchored edge for
    //     this surface; other surfaces will not be placed in that zone.
    //   Zero: surface does not reserve space (floating overlay).
    //  -1: surface participates in the exclusive zone of other surfaces
    //     (i.e., avoids them).
    pub fn exclusive_zone(mut self, zone: i32): Self

    // Margins from each edge in logical pixels.
    pub fn margin(mut self, edges: Edges): Self

    // Initial surface size in logical pixels.
    // Dimensions along a stretched axis are ignored.
    pub fn size(mut self, width: u32, height: u32): Self
}
```

### `Layer`

```ferrum
pub enum Layer {
    // Below desktop wallpaper.
    Background,

    // Above wallpaper, below windows.
    Bottom,

    // Above all normal windows.
    Top,

    // Above everything including the lock screen (compositors may
    // restrict access to this layer).
    Overlay,
}
```

### `Anchor`

```ferrum
// Bitflag: combine edges with | to anchor to multiple sides.
pub struct Anchor {
    pub const NONE:   Anchor = Anchor(0)
    pub const TOP:    Anchor = Anchor(1)
    pub const BOTTOM: Anchor = Anchor(2)
    pub const LEFT:   Anchor = Anchor(4)
    pub const RIGHT:  Anchor = Anchor(8)
}

impl std::ops::BitOr for Anchor { ... }
```

### `Edges`

```ferrum
pub struct Edges {
    pub top:    i32,
    pub bottom: i32,
    pub left:   i32,
    pub right:  i32,
}
```

### `LayerSurface`

```ferrum
pub type LayerSurface

impl LayerSurface {
    // Current logical dimensions of the layer surface.
    pub fn logical_size(&self): Size

    // Access the software frame buffer.  Same API as WaylandWindow.
    pub fn software_frame_buffer(
        &mut self
    ): Result[&mut [u8], WaylandError]

    pub fn damage(&mut self, rects: &[Rect])
    pub fn damage_all(&mut self)
    pub fn commit(&mut self): Result[(), WaylandError] ! IO
    pub fn scale_factor(&self): f32
    pub fn egl_surface(&self): Result[&EglSurface, WaylandError]
    pub fn vulkan_surface(&self): Result[VulkanSurface, WaylandError]

    // Register configure callback.
    // Fires when the compositor assigns a size to the surface.
    pub fn on_configure(
        &mut self,
        f: impl Fn(Size) + 'static
    )

    pub fn acknowledge_configure(&mut self): () ! IO
}
```

---

## 12. Event Loop Integration

### Poll-Based Integration

The most direct integration approach: call `dispatch_events()` whenever the Wayland
file descriptor becomes readable.

```ferrum
use stdlib::sys::posix::{poll, PollFd, PollEvents}

fn run_event_loop(
    conn: &mut WaylandConnection,
    window: &mut WaylandWindow,
): Result[(), WaylandError] ! IO {
    let wayland_fd = conn.event_fd()
    let mut running = true

    while running {
        let mut fds = [PollFd::new(wayland_fd, PollEvents::IN)]
        poll(&mut fds, timeout_ms: -1)?

        if fds[0].revents().contains(PollEvents::IN) {
            conn.dispatch_events()?
        }

        for event in window.poll_events() {
            match event {
                WindowEvent::CloseRequested => { running = false }
                WindowEvent::Key(k) => handle_key(k),
                WindowEvent::Pointer(p) => handle_pointer(p),
                _ => {}
            }
        }
    }
    Ok(())
}
```

### Async Runtime Integration

`WaylandConnection` implements `IoPollable` from `stdlib::async`, making it usable
directly in any Ferrum async runtime that supports I/O readiness polling (epoll, kqueue).

```ferrum
impl IoPollable for WaylandConnection {
    fn io_fd(&self): RawFd { self.event_fd() }
}
```

In an async context, await on the connection directly:

```ferrum
fn run_async(
    conn: &mut WaylandConnection,
    window: &mut WaylandWindow,
): Result[(), WaylandError] ! Async + IO {
    loop {
        // Suspends until event_fd is readable; does not block the async thread.
        conn.wait_readable().await?
        conn.dispatch_events()?

        for event in window.poll_events() {
            match event {
                WindowEvent::CloseRequested => return Ok(()),
                other => handle_event(other).await,
            }
        }
    }
}
```

### Integration with `std.async` Reactor

When `stdlib::async` runtime is in use, register the Wayland connection with the
reactor once during initialization:

```ferrum
use stdlib::async::reactor::Reactor

fn setup(conn: &WaylandConnection, reactor: &mut Reactor): Result[(), WaylandError] ! IO {
    reactor.register_io(conn.event_fd(), IoInterest::Readable)?
    Ok(())
}
```

The reactor will call `dispatch_events()` automatically on each iteration when the
Wayland fd is readable.

---

## 13. Error Types

```ferrum
pub enum WaylandError {
    // $WAYLAND_DISPLAY is not set, or the compositor socket does not exist.
    NoDisplay,

    // The compositor sent a protocol error on an object.
    // This typically indicates a programming error (bad protocol sequence).
    ProtocolError {
        interface: String,   // e.g., "xdg_surface"
        code:      u32,
        message:   String,
    },

    // wl_shm buffer allocation or mmap failed.
    ShmFailed {
        reason: String,
    },

    // EGL API call returned an error.
    // The i32 is the raw EGL error code (e.g., EGL_BAD_SURFACE = 0x300D).
    EglError(i32),

    // Vulkan API call returned an error.
    // The u32 is the raw VkResult value (e.g., VK_ERROR_OUT_OF_HOST_MEMORY = -1).
    VulkanError(u32),

    // A required Wayland global (protocol extension) was not advertised
    // by the compositor.  The string is the interface name.
    // E.g., GlobalNotFound("zwlr_layer_shell_v1") on a compositor that
    // does not support wlr-layer-shell.
    GlobalNotFound(&'static str),

    // A method was called on a window whose render backend does not
    // support the operation (e.g., software_frame_buffer() on a Vulkan window).
    WrongBackend,

    // The Wayland connection was closed unexpectedly (compositor crashed,
    // display server restarted).
    ConnectionLost,

    // No data is available (clipboard empty, no selection for that MIME type).
    NoData,

    // I/O error on the Wayland socket.
    Io(stdlib::io::IoError),
}

impl Display for WaylandError { ... }
impl Error   for WaylandError { ... }
```

---

## 14. Example Usage

### Create a Window and Render a Gradient with Software Rendering

```ferrum
use extlib::draw_wayland::{
    WaylandBackend, WindowConfig, DecorationMode, RenderBackend,
    WaylandError, WindowEvent,
}

fn main(): Result[(), WaylandError] ! IO {
    let mut conn = WaylandBackend::connect()?

    let config = WindowConfig::new()
        .title("Gradient Demo")
        .app_id("com.example.gradient_demo")
        .size(800.0, 600.0)
        .decorations(DecorationMode::ServerSide)
        .render_backend(RenderBackend::Software)

    let mut window = conn.create_window(config)?

    let mut running = true
    window.on_event(move |event| {
        match event {
            WindowEvent::CloseRequested => { running = false }
            _ => {}
        }
    })

    while running {
        conn.dispatch_events()?

        let (pw, ph) = window.physical_size()
        let buf = window.software_frame_buffer()?

        // Render a simple red-to-blue horizontal gradient.
        for y in 0..ph {
            for x in 0..pw {
                let i = ((y * pw + x) * 4) as usize
                let t = x as f32 / pw as f32
                buf[i]     = ((1.0 - t) * 255.0) as u8  // R
                buf[i + 1] = 0                            // G
                buf[i + 2] = (t * 255.0) as u8           // B
                buf[i + 3] = 255                          // A
            }
        }

        window.damage_all()
        window.commit()?
    }

    Ok(())
}
```

### Keyboard and IME Input

```ferrum
use extlib::draw_wayland::{
    WaylandBackend, WindowConfig, RenderBackend, WindowEvent,
    TextInputEvent, InputType, KeyCode, KeyState, WaylandError,
}

fn main(): Result[(), WaylandError] ! IO {
    let mut conn = WaylandBackend::connect()?

    let config = WindowConfig::new()
        .title("Text Input Demo")
        .app_id("com.example.text_input")
        .size(640.0, 480.0)
        .render_backend(RenderBackend::Software)

    let mut window = conn.create_window(config)?

    // Declare that the window accepts text input.
    window.set_input_type(InputType::Text)?
    window.set_cursor_rect(extlib::draw_wayland::Rect {
        x: 50, y: 200, width: 540, height: 24
    })?

    let mut text_buffer = String::new()
    let mut preedit     = String::new()

    loop {
        conn.dispatch_events()?

        for event in window.poll_events() {
            match event {
                WindowEvent::CloseRequested => return Ok(()),

                // IME pre-edit: display underlined composition text.
                WindowEvent::TextInput(TextInputEvent::PreEdit { text, .. }) => {
                    preedit = text
                }

                // IME commit: insert final text, clear pre-edit.
                WindowEvent::TextInput(TextInputEvent::Commit { text }) => {
                    text_buffer.push_str(&text)
                    preedit.clear()
                }

                // Delete surrounding text (IME replacement).
                WindowEvent::TextInput(TextInputEvent::DeleteSurrounding {
                    before_length, after_length
                }) => {
                    // Remove before_length bytes before cursor and
                    // after_length bytes after cursor from text_buffer.
                    // (Implementation elided for brevity.)
                }

                // Raw key events for non-IME keys (arrows, backspace, enter).
                WindowEvent::Key(k) if k.state == KeyState::Pressed => {
                    match k.keycode {
                        KeyCode::Backspace => {
                            text_buffer.pop()
                        }
                        KeyCode::Enter => {
                            // Submit text_buffer.
                        }
                        _ => {
                            // When IME is active, printable text arrives via
                            // TextInput::Commit, not here.  For keyboards
                            // without IME, k.text carries the character.
                            if let Some(ch) = k.text {
                                text_buffer.push_str(&ch)
                            }
                        }
                    }
                }

                _ => {}
            }
        }

        // Redraw text_buffer + preedit.
        render_text(&mut window, &text_buffer, &preedit)?
    }
}
```

### Layer Surface for a Status Bar

```ferrum
use extlib::draw_wayland::{
    WaylandBackend, LayerSurfaceConfig, Layer, Anchor, Edges, WaylandError,
}

fn main(): Result[(), WaylandError] ! IO {
    let mut conn = WaylandBackend::connect()?

    let config = LayerSurfaceConfig::new()
        .layer(Layer::Top)
        .namespace("statusbar")
        .anchor(Anchor::LEFT | Anchor::RIGHT | Anchor::TOP)
        .exclusive_zone(30)      // reserve 30 logical pixels at the top edge
        .margin(Edges { top: 0, bottom: 0, left: 0, right: 0 })
        .size(0, 30)             // width is ignored (stretched), height is 30

    let mut bar = conn.create_layer_surface(config)?

    loop {
        conn.dispatch_events()?

        let (pw, ph) = (
            (bar.logical_size().width  * bar.scale_factor()) as u32,
            (bar.logical_size().height * bar.scale_factor()) as u32,
        )
        let buf = bar.software_frame_buffer()?

        // Fill bar with dark grey.
        for pixel in buf.chunks_exact_mut(4) {
            pixel[0] = 32   // R
            pixel[1] = 32   // G
            pixel[2] = 32   // B
            pixel[3] = 255  // A
        }

        bar.damage_all()
        bar.commit()?
    }
}
```

---

## 15. Dependencies Reference

### Extended Library Dependencies

| Module | Why required |
|--------|-------------|
| `extlib::draw` | Defines the `DrawBackend` trait that `WaylandWindow` and `LayerSurface` implement. Required for use with the rest of the draw ecosystem. |
| `extlib::draw_gpu` | Optional. Required only when `RenderBackend::Vulkan` is selected. Provides `VulkanDevice`, swapchain management, and `DmaBufFrame` construction. |
| `extlib::input` | Provides `KeyCode` enum used in `KeyEvent`. Shared across all platform backends for portable key handling. |

### Standard Library Dependencies

| Module | Used for |
|--------|---------|
| `stdlib::sys::posix` | `mmap` and `munmap` for `wl_shm` buffer allocation; `memfd_create` for anonymous shared memory; `RawFd` type; `poll` for synchronous event loop integration |
| `stdlib::fs` | `Path` type used in `WaylandBackend::connect_to()` |
| `stdlib::async` | `IoPollable` trait for async reactor integration; `wait_readable()` future |
| `stdlib::io` | `IoError` wrapped inside `WaylandError::Io` |
| `stdlib::alloc` | `Vec`, `String`, `Box` |

### External C Library Dependencies

| Library | Symbols used |
|---------|-------------|
| `libwayland-client.so` | All core Wayland protocol objects (`wl_display`, `wl_surface`, `wl_compositor`, `wl_shm`, `wl_seat`, `wl_data_device`), xdg-shell, xdg-decoration, wp-viewporter, wp-fractional-scale-v1, linux-dmabuf-v1, zwp-text-input-v3, zwlr-layer-shell-v1 |
| `libEGL.so` | EGL context and surface management; only linked when `RenderBackend::OpenGlEs3` is active at runtime |

Vulkan surface creation uses the Vulkan loader (`libvulkan.so`) which is linked
transitively through `extlib::draw_gpu`. This module does not link `libvulkan` directly.

### Effect Requirements

| Function category | Effects |
|-------------------|---------|
| `WaylandBackend::connect`, `connect_to` | `! IO` |
| `WaylandConnection::dispatch_events` | `! IO` |
| `WaylandWindow::commit` | `! IO` |
| `WaylandWindow::set_title`, `set_fullscreen`, etc. | `! IO` |
| `WaylandWindow::set_input_type`, `set_cursor_rect` | `! IO` |
| `Clipboard::get_text`, `get_data` | `! IO` |
| `EglSurface::make_current` | `! Unsafe` |
| `EglSurface::swap_buffers` | `! IO` |
| `DragSource::start_drag` | `! IO` |
| Async variants (`wait_readable`) | `! Async + IO` |
| `WaylandWindow::software_frame_buffer` | None — returns a slice reference, no effects |
| `WaylandWindow::scale_factor`, `logical_size`, `physical_size` | None — pure reads of cached state |
