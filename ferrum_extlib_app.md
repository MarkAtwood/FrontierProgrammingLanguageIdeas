# Ferrum Extended Library — app

**Module:** `extlib.ccsp.app`
**Spec basis:** CCSP `lib_ccsp_app` — application lifecycle, multi-window management, system integration
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.draw` (DrawBackend), `extlib.ccsp.widget` (Widget, WidgetTree), `extlib.ccsp.input` (InputEvent), `extlib.ccsp.style` (Theme); platform backends: `extlib.ccsp.draw_wayland`, `extlib.ccsp.draw_x11`; stdlib: `async`, `net`, `fs`, `sys`; external: `dbus-rs` (Linux), `objc` (macOS), `windows-rs` (Windows)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Application — Top-Level Object](#2-application--top-level-object)
3. [AppContext — Event Callback Handle](#3-appcontext--event-callback-handle)
4. [AppWindow — Top-Level Window](#4-appwindow--top-level-window)
5. [Menu System](#5-menu-system)
6. [System Notifications](#6-system-notifications)
7. [File Picker — Sandboxed-Safe File Access](#7-file-picker--sandboxed-safe-file-access)
8. [Single-Instance Enforcement](#8-single-instance-enforcement)
9. [System Tray and Menu Bar Icon](#9-system-tray-and-menu-bar-icon)
10. [About Dialog](#10-about-dialog)
11. [Error Types](#11-error-types)
12. [Example Usage](#12-example-usage)
13. [Dependencies Reference](#13-dependencies-reference)

---

## 1. Overview and Rationale

### What This Module Adds Above the Draw Backends

`extlib.ccsp.draw`, `extlib.ccsp.draw_wayland`, and `extlib.ccsp.draw_x11` provide the
vocabulary for rendering pixels onto surfaces. They do not provide application lifecycle,
window management, system menu integration, OS notifications, or sandboxed file access.
A program using only the draw layer must manage its own event loop, create platform
surfaces directly, and call OS APIs manually for every service that sits above pixel
output.

`extlib.ccsp.app` fills that gap. It wraps the platform-specific machinery of application
startup, window creation, multi-window management, system menus, desktop notifications,
file pickers, single-instance coordination, and system tray icons behind a single
portable API. Application code calls `Application::run` and works with `AppWindow`,
`NotificationService`, and `FilePickerService` regardless of whether the underlying
platform is Wayland, macOS Cocoa, or Win32.

The module deliberately sits above the draw backends and below the widget layer. It owns
window lifetime and widget tree attachment. It does not implement widgets itself.

### Why Sandboxed-Friendly

Modern application distribution uses sandboxed packaging: Flatpak and Snap on Linux,
the macOS App Sandbox on Apple platforms, and the Windows App Container model. Each
sandbox restricts what a process may do directly:

- A Flatpak application cannot open a native file dialog using GTK's internal calls
  because the sandboxed process does not have access to the file system paths the dialog
  would traverse.
- A sandboxed macOS application cannot send desktop notifications by calling POSIX IPC
  directly; it must go through `UNUserNotificationCenter`.
- A Snap cannot listen on a D-Bus name not declared in its snap interface manifest without
  a portal.

In every case, the correct mechanism is a **portal**: a well-known D-Bus service that runs
outside the sandbox with the necessary privileges and brokers the operation on the
application's behalf. On Linux this is the XDG Desktop Portal
(`org.freedesktop.portal.*`). On macOS it is the system frameworks (`NSOpenPanel`,
`UNUserNotificationCenter`). On Windows it is WinRT APIs (`IFileOpenDialog`,
`Windows.UI.Notifications`).

`extlib.ccsp.app` routes every privileged operation through the correct portal on each
platform. An application written against this API works inside a Flatpak, inside a Snap,
inside a macOS sandbox, and outside all of them, without conditional code.

### Why extlib

This module has a large platform-specific dependency set: D-Bus bindings (`dbus-rs`),
Objective-C runtime bindings (`objc`), and Windows Runtime bindings (`windows-rs`).
Every application requires exactly one platform's bindings, but all three are required
at the crate level. Placing this dependency weight in the standard library would impose
it on programs that never open a window. It belongs in extlib, opt-in.

Additionally, this module ties together several other extlib modules (`draw`, `widget`,
`input`, `style`, `font`, platform draw backends). A module with cross-cutting extlib
dependencies cannot be in stdlib.

---

## 2. Application — Top-Level Object

`Application` is the singleton entry point. There is exactly one per process. It
initializes the platform event loop, registers the application identity with the desktop
environment, and owns the connection to the display server.

### `AppConfig`

```ferrum
/// Configuration for the application, provided at startup.
///
/// `app_id` is the most important field. It must be set for correct desktop
/// integration on every platform.
@derive(Debug, Clone)
pub type AppConfig {
    /// Reverse-DNS application identifier.
    ///
    /// Used as:
    ///   - Wayland xdg_toplevel app_id (window grouping, .desktop file matching)
    ///   - D-Bus well-known name prefix (e.g., "com.example.myapp.SingleInstance")
    ///   - macOS CFBundleIdentifier (must match the bundle if distributing via App Store)
    ///   - Windows application user model ID (taskbar grouping, jump lists)
    ///   - Desktop file ID on Linux (links window to launcher icon)
    ///
    /// Required format: reverse DNS, all lowercase, no hyphens in components.
    /// Good: "com.example.myapp"
    /// Bad:  "MyApp", "my-app", "com.example.My-App"
    pub app_id:        String,

    /// Human-readable application name, shown in About dialogs and window lists.
    pub app_name:      String,

    /// Version string, shown in About dialogs.
    /// Convention: semver string, e.g. "1.4.2" or "2.0.0-beta.1".
    pub app_version:   &'static str,

    /// When true, only one instance of the application may run at a time.
    /// A second launch sends its arguments to the running instance and exits.
    /// See Section 8 for details.
    pub single_instance: bool,
}
```

### `Application::new`

```ferrum
impl Application {
    /// Initialize the application and connect to the platform display service.
    ///
    /// On Linux: connects to the Wayland compositor ($WAYLAND_DISPLAY) or falls
    /// back to X11 ($DISPLAY). Acquires D-Bus session connection.
    ///
    /// On macOS: initializes NSApp, sets activation policy.
    ///
    /// On Windows: registers the window class, initializes COM.
    ///
    /// Returns AppError::PlatformInitFailed if no display is available.
    /// Returns AppError::SandboxRestricted if the sandbox blocks display access
    /// (unusual but possible in headless CI environments).
    ///
    /// Must be called on the main thread. Calling from any other thread returns
    /// AppError::PlatformInitFailed("must be called on main thread").
    pub fn new(config: AppConfig): Result[Application, AppError] ! IO
}
```

### `Application::run`

```ferrum
impl Application {
    /// Start the application event loop.
    ///
    /// Calls `event_loop` once immediately to let the application create its
    /// initial windows, then enters the platform event loop. The event loop
    /// dispatches OS events to windows, calls registered callbacks, and
    /// processes widget updates until `Application::quit()` is called.
    ///
    /// This function never returns normally. When `quit()` is called, the process
    /// exits after the current event has been processed. The `!` return type
    /// reflects this: there is no value to return.
    ///
    /// The `! IO` effect reflects that the event loop performs I/O: reading from
    /// the Wayland socket, dispatching D-Bus signals, posting WM_QUIT, etc.
    pub fn run(app: Application, event_loop: impl Fn(&AppContext)): ! ! IO
}
```

### `Application::quit`

```ferrum
impl Application {
    /// Schedule application exit at the end of the current event.
    ///
    /// Safe to call from within any event callback. The event loop finishes
    /// processing the current event, then exits cleanly. In-flight async tasks
    /// attached to the application executor are cancelled.
    pub fn quit()
}
```

### `Application::set_badge_count`

```ferrum
impl Application {
    /// Set the dock or taskbar badge count.
    ///
    /// A count of 0 clears the badge. Negative values are not accepted.
    ///
    /// Platform behavior:
    ///   Linux: sends org.freedesktop.UrgencyHint via D-Bus to the launcher
    ///          (Unity, GNOME Shell extensions, KDE). Not all desktops support
    ///          numeric badges; those that do not silently ignore this call.
    ///   macOS: sets the NSDockTile badgeLabel.
    ///   Windows: sets the taskbar overlay icon with an accessibility label.
    ///
    /// Returns AppError::NotSupported on platforms or desktop environments that
    /// do not implement badge counts.
    pub fn set_badge_count(n: u32): Result[(), AppError]
}
```

---

## 3. AppContext — Event Callback Handle

`AppContext` is passed to the event loop callback and to every event handler registered
on an `AppWindow`. It provides access to all application-level services and is the
factory for new windows. It is not `Send`; all operations must occur on the main thread.

```ferrum
/// Handle to application-level services, passed to event callbacks.
///
/// Not Send — must be used only on the main thread.
/// Not Clone — there is one AppContext per running application.
pub type AppContext { ... }
```

### Window Creation

```ferrum
impl AppContext {
    /// Create a new top-level window.
    ///
    /// The window is created in the hidden state. Call AppWindow::show() to
    /// make it visible. This allows the application to configure the window
    /// fully before it appears on screen.
    ///
    /// On Wayland, this allocates an xdg_toplevel surface.
    /// On macOS, this allocates an NSWindow.
    /// On Windows, this calls CreateWindowEx.
    pub fn create_window(
        &self,
        config: WindowConfig
    ): Result[AppWindow, AppError] ! IO
}
```

### Service Accessors

```ferrum
impl AppContext {
    /// Access the desktop notification service.
    /// The service is always available; individual send operations may fail
    /// with AppError::NotSupported on environments with no notification daemon.
    pub fn notifications(&self): &NotificationService

    /// Access the sandboxed file picker service.
    /// On Linux, routes through org.freedesktop.portal.FileChooser.
    /// On macOS, routes through NSOpenPanel / NSSavePanel.
    /// On Windows, routes through IFileOpenDialog / IFileSaveDialog.
    pub fn file_picker(&self): &FilePickerService

    /// Access the clipboard service.
    /// Text and arbitrary MIME-typed data are supported.
    pub fn clipboard(&self): &ClipboardService

    /// Open a URL in the system default handler.
    ///
    /// Platform routing:
    ///   Linux:   xdg-open (outside sandbox) or
    ///            org.freedesktop.portal.OpenURI (inside Flatpak/Snap)
    ///   macOS:   NSWorkspace.open(_:)
    ///   Windows: ShellExecute with "open" verb
    ///
    /// Returns AppError::PermissionDenied if the sandbox blocks URL opening
    /// and no portal is available.
    pub fn open_url(&self, url: &str): Result[(), AppError]

    /// Set the application-global menu bar.
    ///
    /// On macOS: this is the NSMenuBar displayed at the top of the screen
    /// when the application is frontmost. Required for a native macOS experience.
    ///
    /// On Windows: this sets the system menu available via the application icon
    /// in the taskbar. Per-window menus are set with AppWindow::set_menu_bar.
    ///
    /// On Linux: this publishes the menu via the org.freedesktop.menus D-Bus
    /// interface. Desktop environments that support the Global Menu protocol
    /// (e.g., KDE Plasma with the global menu applet) display it in the panel.
    /// On desktops without global menu support, this call has no effect.
    pub fn set_global_menu(&self, menu: Menu)
}
```

### `WindowConfig`

```ferrum
/// Configuration for a new top-level window.
///
/// Use WindowConfig::new() to start a builder. Every field has a documented
/// default. Only set what you need.
@derive(Debug, Clone)]
pub type WindowConfig {
    /// Window title shown in the title bar and task switcher.
    /// Default: "" (empty string)
    pub title: String,

    /// Initial window size in logical pixels.
    /// Default: Size { width: 800.0, height: 600.0 }
    pub size: Size,

    /// Minimum allowed window size. None means no minimum.
    pub min_size: Option[Size],

    /// Maximum allowed window size. None means no maximum.
    pub max_size: Option[Size],

    /// Whether the user can resize the window. Default: true.
    pub resizable: bool,

    /// Start the window maximized. Default: false.
    pub maximized: bool,

    /// Start the window fullscreen. Default: false.
    pub fullscreen: bool,
}

impl WindowConfig {
    // Create a WindowConfig with all defaults.
    pub fn new(): Self

    pub fn title(mut self, title: &str): Self
    pub fn size(mut self, width: f32, height: f32): Self
    pub fn min_size(mut self, size: Size): Self
    pub fn max_size(mut self, size: Size): Self
    pub fn resizable(mut self, resizable: bool): Self
    pub fn maximized(mut self, maximized: bool): Self
    pub fn fullscreen(mut self, fullscreen: bool): Self
}
```

### `Size`

```ferrum
pub type Size {
    pub width:  f32,
    pub height: f32,
}

impl Size {
    pub fn new(width: f32, height: f32): Self
}
```

---

## 4. AppWindow — Top-Level Window

`AppWindow` represents a single top-level window. Multiple `AppWindow` instances may be
created from the same `AppContext`, enabling multi-window applications. Each window has
its own widget tree, menu bar (on platforms where that is per-window), and lifecycle.

```ferrum
/// A top-level application window.
///
/// Created via AppContext::create_window. Starts hidden; call show() to display.
/// Dropping an AppWindow is equivalent to calling close().
pub type AppWindow { ... }
```

### Title and Geometry

```ferrum
impl AppWindow {
    // Set the window title shown in the title bar and task switcher.
    pub fn set_title(&self, title: &str)

    // Set the window size in logical pixels.
    // The platform may adjust this to satisfy min/max constraints.
    pub fn set_size(&self, size: Size)

    // Set the minimum allowed window size in logical pixels.
    // Constrains user resize and programmatic set_size.
    pub fn set_min_size(&self, size: Size)

    // Set the maximum allowed window size in logical pixels.
    pub fn set_max_size(&self, size: Size)

    // Enter or exit fullscreen mode.
    // On Wayland: requests xdg_toplevel.set_fullscreen / unset_fullscreen.
    // On macOS: calls toggleFullScreen.
    // On Windows: adjusts window style and calls SetWindowPos.
    pub fn set_fullscreen(&self, fullscreen: bool)

    // Move the window to the center of the screen it currently occupies.
    // On multi-monitor systems, centers on the monitor with the largest
    // intersection with the current window position.
    pub fn center_on_screen(&self)

    // Query the current window size in logical pixels.
    pub fn size(&self): Size

    // Query the current window position in logical pixels relative to
    // the top-left corner of the containing screen.
    pub fn position(&self): Point
}
```

### Widget Tree

```ferrum
impl AppWindow {
    // Set the root widget of this window's widget tree.
    //
    // The widget tree is traversed for layout, paint, and input dispatch on
    // every frame. Replacing the root replaces the entire tree.
    // Passing an Arc[dyn Widget] allows the widget to be shared with other
    // parts of the application if needed.
    pub fn set_widget_root(&self, widget: Arc[dyn Widget])
}
```

### Visibility and Lifecycle

```ferrum
impl AppWindow {
    // Make the window visible.
    // Has no effect if the window is already visible.
    pub fn show(&self)

    // Hide the window without destroying it.
    // The window's state (size, widget tree, title) is preserved.
    // Widgets remain in memory and continue to receive timer events.
    pub fn hide(&self)

    // Close and destroy the window.
    // All resources associated with the window are freed.
    // If this is the last open window and no other reason to stay alive exists,
    // the application exits. Call Application::quit() explicitly if you want
    // controlled shutdown.
    pub fn close(&self)

    // Query whether the window is currently visible.
    pub fn is_visible(&self): bool
}
```

### Close-Request Handler

```ferrum
/// The action to take in response to a user-initiated window close request.
pub enum CloseAction {
    /// Allow the window to close.
    Close,

    /// Keep the window open. Use this for "unsaved changes" dialogs that ask
    /// the user whether to save, discard, or cancel.
    Keep,
}

impl AppWindow {
    // Register a callback invoked when the user attempts to close the window
    // (clicks the close button, presses Alt+F4, etc.).
    //
    // Return CloseAction::Close to allow the close.
    // Return CloseAction::Keep to suppress the close (e.g., show a save dialog).
    //
    // If no handler is registered, the window closes by default.
    pub fn set_on_close_request(&self, f: impl Fn() -> CloseAction + 'static)
}
```

### Menu Bar

```ferrum
impl AppWindow {
    // Set the menu bar for this window.
    //
    // On Windows and Linux, each window has its own menu bar displayed
    // immediately below the title bar.
    //
    // On macOS, this call is ignored: macOS has a single application-wide
    // menu bar. Use AppContext::set_global_menu() on macOS.
    pub fn set_menu_bar(&self, menu: Menu)
}
```

### Screenshot

```ferrum
impl AppWindow {
    // Capture the current rendered frame as an ImageData.
    //
    // The returned ImageData contains the last committed frame in RGBA8 format,
    // at the window's physical pixel dimensions. Useful for generating task
    // switcher thumbnails or saving snapshots.
    //
    // Returns the image at physical pixel resolution (logical size * scale factor).
    pub fn screenshot(&self): ImageData
}
```

### `ImageData`

```ferrum
/// A CPU-side RGBA image buffer.
///
/// Width and height are in physical pixels.
/// Pixel data is stored in row-major order, 4 bytes per pixel: R, G, B, A.
/// Total length: width * height * 4 bytes.
pub type ImageData {
    pub width:  u32,
    pub height: u32,
    pub pixels: Vec[u8],
}
```

### `Point`

```ferrum
pub type Point {
    pub x: f32,
    pub y: f32,
}
```

---

## 5. Menu System

Menus are built using a builder pattern, then attached to windows or the global menu
bar. The `MenuAction` enum names standard edit and application actions; the platform maps
these to the correct system accelerators and menu item behaviors automatically.

### `Menu` and `MenuBuilder`

```ferrum
/// An immutable, fully-constructed menu tree ready to attach to a window or
/// the global menu bar.
pub type Menu { ... }

impl Menu {
    // Start building a new menu.
    pub fn new(): MenuBuilder
}
```

```ferrum
pub type MenuBuilder { ... }

impl MenuBuilder {
    // Add a menu item with a label and action.
    // No keyboard shortcut is assigned; the user must click the item.
    pub fn item(
        &mut self,
        label: &str,
        action: MenuAction
    ): &mut MenuBuilder

    // Add a menu item with a label, action, and keyboard shortcut.
    // The platform displays the shortcut next to the label.
    // On macOS, Cmd is the standard modifier; on Windows and Linux, Ctrl.
    pub fn item_with_shortcut(
        &mut self,
        label:    &str,
        action:   MenuAction,
        shortcut: KeyShortcut
    ): &mut MenuBuilder

    // Add a horizontal separator line between groups of related items.
    pub fn separator(&mut self): &mut MenuBuilder

    // Add a submenu with a label and a pre-built child Menu.
    pub fn submenu(&mut self, label: &str, submenu: Menu): &mut MenuBuilder

    // Finalize the builder and produce a Menu.
    pub fn build(self): Menu
}
```

### `MenuAction`

```ferrum
/// The action attached to a menu item.
///
/// Standard actions (Cut, Copy, Paste, etc.) are mapped to platform-native
/// behavior automatically: macOS uses the responder chain; Windows and Linux
/// dispatch to the focused widget via the widget tree's action protocol.
///
/// Custom application logic uses Callback.
pub enum MenuAction {
    /// Run arbitrary application code.
    Callback(Box[dyn Fn()]),

    /// Request application exit. Equivalent to Application::quit().
    Quit,

    // Standard text editing actions — dispatched to the focused widget.
    Cut,
    Copy,
    Paste,
    SelectAll,
    Undo,
    Redo,

    // View actions — dispatched to the focused or frontmost window.
    ZoomIn,
    ZoomOut,
    FullScreen,

    // Application actions — handled by the application layer.

    /// Show the About dialog. Requires AppContext::show_about to have been called
    /// or a handler to be registered. Automatically wired on macOS.
    About,

    /// Show the application preferences window.
    /// The application must register a handler; this action routes to it.
    Preferences,
}
```

### `KeyShortcut`

```ferrum
/// A keyboard shortcut to display alongside a menu item.
pub type KeyShortcut {
    pub key:       KeyCode,
    pub modifiers: Modifiers,
}

impl KeyShortcut {
    // Convenience: Cmd+key on macOS, Ctrl+key on Windows and Linux.
    pub fn primary(key: KeyCode): Self
}
```

`KeyCode` and `Modifiers` are defined in `extlib.ccsp.input`.

### Platform Mapping

| Platform | Implementation |
|----------|---------------|
| macOS | `NSMenu` / `NSMenuItem` hierarchy attached to `[NSApp mainMenu]` |
| Windows | `HMENU` attached to the window via `SetMenu`; system menu accessible via taskbar icon |
| Linux | `org.freedesktop.menus` D-Bus interface (Global Menu); per-window menu bar widget rendered by the widget layer as fallback on desktops without Global Menu support |

---

## 6. System Notifications

Desktop notifications appear in the system notification center or notification tray,
outside the application's windows. They are appropriate for background-task completion,
incoming messages, and alerts that need user attention even when the application is
not focused.

### `NotificationService`

```ferrum
pub type NotificationService { ... }

impl NotificationService {
    // Send a desktop notification.
    //
    // On Linux: sends a method call to org.freedesktop.Notifications via D-Bus.
    //           Works inside Flatpak and Snap via the portal.
    // On macOS: requests authorization if not yet granted, then schedules via
    //           UNUserNotificationCenter. First call may trigger a permission
    //           prompt; subsequent calls are silent.
    // On Windows: posts to the WinRT Windows.UI.Notifications toast API.
    //
    // The ! Net effect reflects the D-Bus call on Linux. On macOS and Windows,
    // the underlying call is synchronous IPC which is classified as ! Net here
    // for consistency — the effect propagates up to callers regardless of
    // platform.
    pub fn send(
        &self,
        notification: Notification
    ): Result[NotificationHandle, AppError] ! Net
}
```

### `Notification`

```ferrum
/// A notification to send to the desktop notification system.
@derive(Debug, Clone)]
pub type Notification {
    /// Short summary line. Shown in the notification bubble and notification center.
    pub title:   String,

    /// Body text. May be empty. Some platforms render Markdown-lite markup;
    /// plain text is always safe.
    pub body:    String,

    /// Icon shown in the notification.
    /// None uses the application icon registered with AppConfig::app_id.
    pub icon:    Option[NotificationIcon],

    /// Urgency hint.
    /// Low:      shown quietly, may be aggregated or hidden by the OS.
    /// Normal:   standard notification.
    /// Critical: shown immediately and persistently until dismissed.
    pub urgency: Urgency,

    /// Action buttons shown in the notification body.
    /// Maximum number supported varies by platform (Linux: typically 2-3;
    /// macOS: 1; Windows: up to 5 in an adaptive toast).
    /// Extra actions beyond the platform maximum are silently dropped.
    pub actions: Vec[NotificationAction],

    /// How long to display the notification before auto-dismissal.
    pub timeout: NotificationTimeout,
}

impl Notification {
    // Construct a minimal notification with just a title and body.
    // Sets urgency Normal, no actions, timeout Default.
    pub fn new(title: &str, body: &str): Self
}
```

### `NotificationIcon`

```ferrum
pub enum NotificationIcon {
    /// Use the application's registered icon (the default if icon is None).
    AppIcon,

    /// An icon from the system icon theme (e.g., "mail-unread", "dialog-warning").
    /// On platforms without icon themes, falls back to AppIcon.
    NamedIcon(String),

    /// An icon loaded from a file on disk.
    /// Formats: PNG, JPEG, SVG (platform support for SVG varies).
    File(PathBuf),
}
```

### `Urgency`

```ferrum
pub enum Urgency {
    /// Low-priority notification. May be batched, hidden, or shown only in
    /// notification center on Do Not Disturb systems.
    Low,

    /// Standard notification. Default.
    Normal,

    /// High-priority notification. Bypasses Do Not Disturb on most platforms.
    /// Use only for genuinely urgent conditions (security alerts, data loss risk).
    Critical,
}
```

### `NotificationAction`

```ferrum
/// An action button that appears in the notification body.
pub type NotificationAction {
    /// Machine-readable identifier. Returned by NotificationHandle::on_action
    /// when the user clicks this button.
    pub id:    String,

    /// Human-readable label shown on the button.
    pub label: String,
}
```

### `NotificationTimeout`

```ferrum
pub enum NotificationTimeout {
    /// Platform default duration (typically 5-10 seconds for Normal urgency).
    Default,

    /// Notification remains until the user explicitly dismisses it.
    Never,

    /// Auto-dismiss after the given number of milliseconds.
    Ms(u32),
}
```

### `NotificationHandle`

```ferrum
/// A handle to a sent notification. Used to close the notification programmatically
/// or register action callbacks.
pub type NotificationHandle { ... }

impl NotificationHandle {
    // Dismiss the notification before its timeout expires.
    // Has no effect if the notification has already been dismissed.
    pub fn close(&self)

    // Register a callback invoked when the user clicks an action button.
    // The &str argument is the NotificationAction::id of the clicked action.
    // Also called with the id "default" if the user clicks the notification
    // body itself (outside any action button).
    pub fn on_action(&self, f: impl Fn(&str) + 'static)
}
```

### Platform Notes

On macOS, the application must include `NSUserNotificationUsageDescription` in its
`Info.plist` for the permission prompt to display correctly. `extlib.ccsp.app` surfaces
`AppError::PermissionDenied` if the user has denied notification access.

On Windows, the WinRT toast API requires the application to have a Start Menu shortcut
with the AppUserModelID set. Applications installed via the Windows Package Manager or
MSIX satisfy this automatically. Portable EXEs (no installer) may receive
`AppError::NotSupported { feature: "notifications", platform: "windows-portable" }`.

---

## 7. File Picker — Sandboxed-Safe File Access

The file picker service routes all file selection dialogs through the platform portal,
making them work correctly inside Flatpak, Snap, and macOS sandboxed distributions.
Sandboxed applications MUST use this service for user-initiated file access. Calling
`stdlib::fs::read_dir` on an arbitrary path from a sandboxed application will fail
with permission errors; the portal provides the only sanctioned path.

```ferrum
pub type FilePickerService { ... }
```

### Open File

```ferrum
impl FilePickerService {
    // Show a file open dialog and return the selected file path.
    //
    // Returns Ok(Some(path)) if the user selected a file.
    // Returns Ok(None) if the user cancelled.
    // Returns Err(AppError) if the portal call fails.
    //
    // Inside a Flatpak or Snap: routes through
    //   org.freedesktop.portal.FileChooser.OpenFile
    // On macOS: runs NSOpenPanel on the main thread.
    // On Windows: runs IFileOpenDialog.
    pub fn open_file(
        &self,
        config: FilePickerConfig
    ): Result[Option[PathBuf], AppError] ! IO

    // Show a multi-select file open dialog.
    // Returns the list of selected paths (empty Vec if the user cancelled).
    pub fn open_files(
        &self,
        config: FilePickerConfig
    ): Result[Vec[PathBuf], AppError] ! IO

    // Show a save-file dialog.
    //
    // Returns Ok(Some(path)) with the chosen save path.
    // Returns Ok(None) if the user cancelled.
    // The file is NOT created by this call. The application is responsible
    // for writing to the returned path.
    pub fn save_file(
        &self,
        config: SavePickerConfig
    ): Result[Option[PathBuf], AppError] ! IO

    // Show a folder selection dialog.
    //
    // Returns Ok(Some(path)) with the chosen folder path.
    // Returns Ok(None) if the user cancelled.
    pub fn pick_folder(
        &self,
        config: FolderPickerConfig
    ): Result[Option[PathBuf], AppError] ! IO
}
```

### `FilePickerConfig`

```ferrum
/// Configuration for file open dialogs.
@derive(Debug, Clone)]
pub type FilePickerConfig {
    /// Title shown in the dialog's title bar.
    /// None uses the platform default ("Open" or equivalent).
    pub title:       Option[String],

    /// File type filters presented in the dialog's filter dropdown.
    /// An empty Vec shows all files.
    pub filters:     Vec[FileFilter],

    /// The directory to show when the dialog first opens.
    /// None lets the platform choose (typically the last-used directory).
    pub initial_dir: Option[PathBuf],
}

impl FilePickerConfig {
    pub fn new(): Self
    pub fn title(mut self, title: &str): Self
    pub fn filter(mut self, filter: FileFilter): Self
    pub fn initial_dir(mut self, path: PathBuf): Self
}
```

### `SavePickerConfig`

```ferrum
@derive(Debug, Clone)]
pub type SavePickerConfig {
    pub title:            Option[String],
    pub filters:          Vec[FileFilter],
    pub initial_dir:      Option[PathBuf],
    /// Suggested filename in the dialog's filename field.
    pub suggested_name:   Option[String],
}

impl SavePickerConfig {
    pub fn new(): Self
    pub fn title(mut self, title: &str): Self
    pub fn filter(mut self, filter: FileFilter): Self
    pub fn initial_dir(mut self, path: PathBuf): Self
    pub fn suggested_name(mut self, name: &str): Self
}
```

### `FolderPickerConfig`

```ferrum
@derive(Debug, Clone)]
pub type FolderPickerConfig {
    pub title:       Option[String],
    pub initial_dir: Option[PathBuf],
}

impl FolderPickerConfig {
    pub fn new(): Self
    pub fn title(mut self, title: &str): Self
    pub fn initial_dir(mut self, path: PathBuf): Self
}
```

### `FileFilter`

```ferrum
/// A named set of file extensions for the dialog's filter dropdown.
///
/// Example:
///   FileFilter { name: "Images", extensions: vec!["png", "jpg", "webp"] }
///   FileFilter { name: "All Files", extensions: vec![] }
///
/// Extensions are specified without the leading dot.
/// An empty extensions list matches all files.
@derive(Debug, Clone)]
pub type FileFilter {
    pub name:       String,
    pub extensions: Vec[String],
}

impl FileFilter {
    pub fn new(name: &str, extensions: &[&str]): Self
}
```

### Sandbox Semantics

Inside a Flatpak or Snap, the portal returns a path under `/run/user/<uid>/doc/`
(the document portal). The returned path is valid for reading and writing via normal
`stdlib::fs` calls from within the sandbox. It is a bind-mount the portal has
granted to this process specifically. The application should not store these paths
persistently (they change across portal sessions); use the document portal's
`org.freedesktop.portal.Documents` interface if persistent access is required. That
level of document portal management is out of scope for `extlib.ccsp.app` and is
addressed in a separate extlib module.

---

## 8. Single-Instance Enforcement

When `AppConfig::single_instance` is `true`, the application ensures that only one
instance runs at a time. A second launch detected by the already-running instance causes
the running instance to be brought to the foreground and the second process to exit.

The arguments passed to the second invocation are forwarded to the running instance,
allowing the running instance to open files or navigate to content.

### `Application::set_on_second_instance`

```ferrum
impl Application {
    // Register a callback invoked when a second instance of the application is
    // launched while this instance is running.
    //
    // The &[String] argument contains the command-line arguments passed to the
    // second instance (argv[1..]).
    //
    // Typical response: bring the main window to the foreground, open the
    // file paths in the args if any are present.
    //
    // This callback has no effect if AppConfig::single_instance is false.
    pub fn set_on_second_instance(
        &self,
        f: impl Fn(&[String]) + 'static
    )
}
```

### Platform Implementation

| Platform | Mechanism |
|----------|-----------|
| Linux | First instance acquires the well-known D-Bus name `<app_id>.SingleInstance`. Second instance calls `<app_id>.SingleInstance.Activate(argv)` on that name and exits. D-Bus name acquisition is atomic; the race is handled by the session bus. |
| macOS | The OS delivers `applicationShouldHandleReopen` and `application(_:openFiles:)` to the `NSApplicationDelegate` of the running instance via the app's registered bundle ID. No second process launches. |
| Windows | First instance creates a named mutex `Local\<app_id>`. Second instance detects the mutex, sends the args via a hidden window message (`WM_COPYDATA`) to the first instance's HWND, then exits. |

On Linux, this requires a D-Bus session bus. If no session bus is available
(`DBUS_SESSION_BUS_ADDRESS` is unset), `single_instance` enforcement is skipped with a
warning logged to stderr, and both instances run concurrently.

---

## 9. System Tray and Menu Bar Icon

A system tray icon (called a "status item" on macOS) places a small icon in the system
notification area or menu bar. It is optional and must be explicitly created. Tray icons
are appropriate for applications that run in the background and provide quick access to
controls without opening a window.

```ferrum
pub type TrayIcon { ... }

impl TrayIcon {
    // Create a system tray icon with the given icon and tooltip text.
    //
    // The icon appears in:
    //   Linux:   the system tray via the StatusNotifierItem D-Bus protocol
    //            (org.kde.StatusNotifierItem). Supported by KDE Plasma, GNOME
    //            Shell (with AppIndicator extension), Xfce, and most panels.
    //   macOS:   the macOS menu bar via NSStatusItem.
    //   Windows: the notification area (system tray) via Shell_NotifyIcon.
    //
    // Returns AppError::NotSupported on Linux desktops that do not have a
    // StatusNotifierItem-compatible system tray (rare in 2026 but possible
    // in minimal desktop environments).
    pub fn new(icon: &Icon, tooltip: &str): Result[TrayIcon, AppError] ! IO

    // Replace the tray icon image.
    pub fn set_icon(&self, icon: &Icon)

    // Update the tooltip text shown on hover.
    pub fn set_tooltip(&self, text: &str)

    // Attach a context menu shown when the user right-clicks (Windows/Linux)
    // or clicks (macOS) the tray icon.
    pub fn set_menu(&self, menu: Menu)

    // Register a callback for a left-click (Windows/Linux) or main-click
    // (macOS) on the tray icon.
    // Typical use: show or focus the main window.
    pub fn on_click(&self, f: impl Fn() + 'static)

    // Remove the tray icon from the system tray.
    // After this call the TrayIcon is inert; further calls to set_icon,
    // set_menu, etc. are no-ops.
    pub fn remove(&self)
}
```

### `Icon`

```ferrum
/// An image used as a tray icon, window icon, or notification icon.
///
/// Prefer SVG or multi-resolution ICNS/ICO for correct rendering at all
/// screen densities. PNG is acceptable for fixed-resolution icons.
pub enum Icon {
    /// PNG or JPEG image data from memory.
    Bytes {
        data:   Vec[u8],
        width:  u32,
        height: u32,
    },

    /// An ImageData (from AppWindow::screenshot or similar).
    Image(ImageData),

    /// An icon from the system icon theme by name.
    /// Falls back to a generic application icon on platforms without themes.
    Named(String),
}
```

### Platform Notes

On Linux, the StatusNotifierItem protocol requires the application to export a D-Bus
object at a path of the form `/StatusNotifierItem` and register it with the
`org.kde.StatusNotifierWatcher` service. `extlib.ccsp.app` handles this registration
automatically. The application does not interact with D-Bus directly.

On GNOME Shell without the AppIndicator extension, the SNI tray icon is not visible.
`TrayIcon::new` succeeds (the D-Bus registration is accepted) but the icon does not
appear to the user. There is no reliable way to detect this condition from within the
sandbox; this is a known limitation of GNOME Shell's default configuration.

---

## 10. About Dialog

The About dialog displays application identity information in a platform-native style.
macOS has a well-defined `NSApp orderFrontStandardAboutPanelWithOptions:` call; Windows
and Linux use a dialog built from the widget layer.

```ferrum
impl AppContext {
    // Show the platform About dialog for this application.
    //
    // On macOS: calls [NSApp orderFrontStandardAboutPanelWithOptions:] with
    //           the provided fields mapped to NSAboutPanelOption keys.
    // On Windows and Linux: opens a modal dialog window built from the
    //           widget layer, styled by the current application Theme.
    //
    // The dialog is non-modal on macOS (standard macOS convention) and
    // modal on Windows and Linux.
    pub fn show_about(&self, info: AboutInfo): Result[(), AppError] ! IO
}
```

### `AboutInfo`

```ferrum
@derive(Debug, Clone)]
pub type AboutInfo {
    /// Application name displayed as the dialog's title.
    pub app_name:    &'static str,

    /// Version string (e.g., "1.4.2").
    pub version:     &'static str,

    /// One or two sentence description of what the application does.
    /// None omits the description field.
    pub description: Option[&'static str],

    /// URL of the application's home page or documentation.
    /// Rendered as a clickable link on all platforms.
    pub website:     Option[&'static str],

    /// SPDX license identifier or short license description.
    /// None omits the license field.
    pub license:     Option[&'static str],

    /// List of author names or "Name <email>" strings.
    pub authors:     Vec[&'static str],

    /// Application logo displayed in the dialog.
    /// None uses the application icon registered with app_id.
    pub logo:        Option[ImageData],
}
```

---

## 11. Error Types

```ferrum
/// Errors produced by application lifecycle and system service operations.
@derive(Debug)]
pub enum AppError {
    /// The platform display system failed to initialize.
    ///
    /// Payload: human-readable reason, e.g.:
    ///   "WAYLAND_DISPLAY is not set"
    ///   "NSApp initialization failed: cannot connect to window server"
    ///   "CreateWindowEx failed: error 5 (access denied)"
    PlatformInitFailed(String),

    /// A D-Bus method call or name acquisition failed.
    ///
    /// Linux only. Payload: the D-Bus error name and message.
    /// e.g.: "org.freedesktop.DBus.Error.ServiceUnknown: The name
    ///        org.freedesktop.Notifications was not provided by any .service files"
    DbusError(String),

    /// The user or system denied a permission required for the operation.
    ///
    /// Examples:
    ///   - macOS notification permission denied by user in System Settings.
    ///   - Windows notification access blocked by group policy.
    ///   - File picker returned access denied from the portal.
    PermissionDenied,

    /// The operation was blocked because the application is running inside a
    /// sandbox and the required portal is not available.
    ///
    /// `operation`: the name of the restricted operation.
    ///   e.g.: "open_url", "set_badge_count", "notification_send"
    SandboxRestricted {
        operation: &'static str,
    },

    /// The requested feature is not supported on the current platform or
    /// desktop environment.
    ///
    /// `feature`:  what was requested (e.g., "tray_icon", "badge_count")
    /// `platform`: a short platform identifier (e.g., "linux-gnome-no-sni",
    ///             "windows-portable", "wayland-no-xdg-decoration")
    NotSupported {
        feature:  &'static str,
        platform: &'static str,
    },

    /// Window creation failed.
    ///
    /// Payload: human-readable reason.
    WindowCreateFailed(String),

    /// An I/O error occurred at the platform IPC boundary.
    Io(stdlib::io::IoError),
}

impl Display for AppError { ... }
impl Error   for AppError { ... }
```

---

## 12. Example Usage

### 12.1 Simple Single-Window Application

A minimal application: one window, a widget tree, clean shutdown when the window is
closed.

```ferrum
use extlib::ccsp::app::{
    Application, AppConfig, AppContext, AppWindow,
    WindowConfig, CloseAction, AppError,
}
use extlib::ccsp::widget::TextLabel

fn main(): Result[(), AppError] ! IO {
    let config = AppConfig {
        app_id:          "com.example.hello".into(),
        app_name:        "Hello".into(),
        app_version:     "0.1.0",
        single_instance: false,
    }

    let app = Application::new(config)?

    Application::run(app, |ctx| {
        let window = ctx.create_window(
            WindowConfig::new()
                .title("Hello, Ferrum")
                .size(640.0, 480.0)
        ).expect("window creation failed")

        let label = Arc::new(
            TextLabel::new("Hello, world!")
                .font_size(24.0)
                .center()
        )
        window.set_widget_root(label)
        window.center_on_screen()

        // Close the app when the window is closed.
        window.set_on_close_request(|| {
            Application::quit()
            CloseAction::Close
        })

        window.show()
    })
}
```

### 12.2 Multi-Window Application with Dark Mode Toggle

Two windows sharing a toggle: clicking the button in one window affects the theme of
both.

```ferrum
use extlib::ccsp::app::{Application, AppConfig, AppContext, AppWindow,
    WindowConfig, CloseAction, AppError}
use extlib::ccsp::widget::{Button, VStack}
use extlib::ccsp::style::{Theme, ColorScheme}
use stdlib::sync::Arc
use stdlib::sync::Mutex

fn build_menu(ctx: &AppContext): Menu {
    Menu::new()
        .submenu("File",
            Menu::new()
                .item_with_shortcut("Quit", MenuAction::Quit,
                    KeyShortcut::primary(KeyCode::Q))
                .build()
        )
        .submenu("View",
            Menu::new()
                .item("Toggle Dark Mode", MenuAction::Callback(Box::new(|| {
                    // Handled via the shared theme state below.
                })))
                .build()
        )
        .build()
}

fn main(): Result[(), AppError] ! IO {
    let config = AppConfig {
        app_id:          "com.example.multiwin".into(),
        app_name:        "Multi-Window Demo".into(),
        app_version:     "0.1.0",
        single_instance: true,
    }

    let app = Application::new(config)?

    app.set_on_second_instance(|args| {
        // A second launch: bring the first window to the foreground.
        // (Window handle retrieval from a registry is application-specific.)
    })

    Application::run(app, |ctx| {
        let dark_mode = Arc::new(Mutex::new(false))

        let make_window = |ctx: &AppContext, title: &str| -> AppWindow {
            let dm = Arc::clone(&dark_mode)
            let toggle = Button::new("Toggle Dark Mode")
                .on_click(move || {
                    let mut flag = dm.lock()
                    *flag = !*flag
                    let scheme = if *flag {
                        ColorScheme::Dark
                    } else {
                        ColorScheme::Light
                    }
                    Theme::set_global_color_scheme(scheme)
                })

            let root = Arc::new(VStack::new().child(toggle))
            let win = ctx.create_window(
                WindowConfig::new()
                    .title(title)
                    .size(400.0, 300.0)
            ).expect("window creation failed")

            win.set_widget_root(root)
            win.set_menu_bar(build_menu(ctx))
            win.set_on_close_request(|| CloseAction::Close)
            win.show()
            win
        }

        let _win1 = make_window(ctx, "Window One")
        let _win2 = make_window(ctx, "Window Two")

        ctx.set_global_menu(build_menu(ctx))
    })
}
```

### 12.3 Desktop Notification with Action Button

Send a notification with a "View" action; when clicked, bring the window to focus.

```ferrum
use extlib::ccsp::app::{
    Application, AppConfig, AppContext, Notification,
    NotificationAction, NotificationTimeout, AppError,
}

fn notify_task_complete(ctx: &AppContext) {
    let notif = Notification {
        title:   "Export complete".into(),
        body:    "report_q4.pdf has been saved to Downloads.".into(),
        icon:    None,
        urgency: Urgency::Normal,
        actions: vec![
            NotificationAction { id: "view".into(), label: "View".into() },
        ],
        timeout: NotificationTimeout::Ms(8000),
    }

    match ctx.notifications().send(notif) {
        Ok(handle) => {
            handle.on_action(|action_id| {
                if action_id == "view" || action_id == "default" {
                    ctx.open_url("file:///home/user/Downloads/report_q4.pdf")
                        .unwrap_or_else(|e| {
                            // Log and continue; notification handler must not panic.
                            eprintln!("open_url failed: {}", e)
                        })
                }
            })
        }
        Err(e) => {
            // Non-fatal: notification is a convenience, not a requirement.
            eprintln!("notification failed: {}", e)
        }
    }
}
```

### 12.4 Sandboxed File Open Dialog

Open a user-selected image file, reading it via `stdlib::fs` after the portal grants
access.

```ferrum
use extlib::ccsp::app::{FilePickerConfig, FileFilter, AppContext, AppError}
use stdlib::fs

fn open_image(ctx: &AppContext): Result[Option[Vec[u8]], AppError] ! IO {
    let config = FilePickerConfig::new()
        .title("Open Image")
        .filter(FileFilter::new("Images", &["png", "jpg", "jpeg", "webp", "tiff"]))
        .filter(FileFilter::new("All Files", &[]))

    let path = ctx.file_picker().open_file(config)?

    match path {
        None => Ok(None),   // user cancelled
        Some(p) => {
            let bytes = fs::read(&p)
                .map_err(|e| AppError::Io(e))?
            Ok(Some(bytes))
        }
    }
}
```

### 12.5 System Tray Icon with Context Menu

A background application with a tray icon that shows a menu and opens the main window
on click.

```ferrum
use extlib::ccsp::app::{
    Application, AppConfig, AppContext, TrayIcon, Icon,
    Menu, MenuAction, KeyShortcut, AppError,
}

fn setup_tray(ctx: &AppContext) -> Result[TrayIcon, AppError] ! IO {
    let icon = Icon::Named("application-status".into())

    let tray = TrayIcon::new(&icon, "My Background App")?

    let menu = Menu::new()
        .item("Show Window", MenuAction::Callback(Box::new(|| {
            // Show main window (application-specific retrieval).
        })))
        .separator()
        .item("Quit", MenuAction::Quit)
        .build()

    tray.set_menu(menu)

    tray.on_click(|| {
        // Show the main window on left-click.
    })

    Ok(tray)
}

fn main(): Result[(), AppError] ! IO {
    let config = AppConfig {
        app_id:          "com.example.bgapp".into(),
        app_name:        "Background App".into(),
        app_version:     "1.0.0",
        single_instance: true,
    }

    let app = Application::new(config)?

    Application::run(app, |ctx| {
        let _tray = setup_tray(ctx)
            .expect("tray icon setup failed")

        // No window shown at startup. The tray menu is the entry point.
    })
}
```

### 12.6 About Dialog

```ferrum
use extlib::ccsp::app::{AppContext, AboutInfo, AppError}

fn show_about(ctx: &AppContext): Result[(), AppError] ! IO {
    ctx.show_about(AboutInfo {
        app_name:    "Ferrum Editor",
        version:     "2.1.0",
        description: Some("A fast, keyboard-driven text editor."),
        website:     Some("https://example.com/ferrum-editor"),
        license:     Some("MIT"),
        authors:     vec!["Alice Example", "Bob Sample <bob@example.com>"],
        logo:        None,
    })
}
```

---

## 13. Dependencies Reference

### Extended Library Dependencies

| Module | Why required |
|--------|-------------|
| `extlib::ccsp::draw` | `DrawBackend` trait; `ImageData` type used by `AppWindow::screenshot` and `AboutInfo::logo` |
| `extlib::ccsp::widget` | `Widget` trait; `AppWindow::set_widget_root` accepts `Arc[dyn Widget]`; the About dialog on Windows and Linux is built from the widget layer |
| `extlib::ccsp::input` | `KeyCode` and `Modifiers` types used in `KeyShortcut`; `InputEvent` dispatched through widget trees |
| `extlib::ccsp::style` | `Theme` type; the About dialog and per-window menus on Linux are styled via the active theme |
| `extlib::ccsp::draw_wayland` | Linux Wayland display backend; used when `$WAYLAND_DISPLAY` is set |
| `extlib::ccsp::draw_x11` | Linux X11/XWayland fallback; used when `$DISPLAY` is set and Wayland is unavailable |
| `extlib::ccsp::a11y` | Accessibility tree construction wired through `AppWindow` widget layout; optional but strongly recommended |

### Standard Library Dependencies

| Module | Used for |
|--------|---------|
| `stdlib::async` | Application event loop integration; async tasks spawned by the widget layer |
| `stdlib::net` | `! Net` effect on notification send; D-Bus socket I/O on Linux |
| `stdlib::fs` | `PathBuf` type used in file picker results; `stdlib::fs::read` usable on portal-granted paths |
| `stdlib::sys` | Platform detection; POSIX fd handling for Wayland socket; named mutex on Windows |
| `stdlib::io` | `IoError` wrapped inside `AppError::Io` |
| `stdlib::alloc` | `Vec`, `String`, `Box`, `Arc` |
| `stdlib::sync` | `Mutex`, `Arc` for shared state across window callbacks |

### External Library Dependencies

| Library | Platform | Used for |
|---------|----------|---------|
| `dbus-rs` | Linux | D-Bus session bus connection; `org.freedesktop.Notifications`, `org.freedesktop.portal.FileChooser`, `org.freedesktop.portal.OpenURI`, `org.freedesktop.portal.Documents`, `org.kde.StatusNotifierItem`, `<app_id>.SingleInstance` |
| `objc` | macOS | `NSApp`, `NSWindow`, `NSMenu`, `NSStatusItem`, `UNUserNotificationCenter`, `NSOpenPanel`, `NSSavePanel`, `NSWorkspace` Objective-C message sends via Ferrum FFI |
| `windows-rs` | Windows | `CreateWindowEx`, `Shell_NotifyIcon`, `IFileOpenDialog`, `IFileSaveDialog`, `Windows.UI.Notifications`, named mutex, `WM_COPYDATA` IPC |

### Effect Requirements

| Function category | Effects |
|-------------------|---------|
| `Application::new` | `! IO` |
| `Application::run` | `! IO` (diverges; return type `!`) |
| `AppContext::create_window` | `! IO` |
| `AppContext::open_url` | none (schedules IPC; side effect is deferred) |
| `AppContext::show_about` | `! IO` |
| `NotificationService::send` | `! Net` |
| `FilePickerService::open_file`, `open_files`, `save_file`, `pick_folder` | `! IO` |
| `TrayIcon::new` | `! IO` |
| `AppWindow::screenshot` | none (reads last committed frame from memory) |
| `AppWindow::show`, `hide`, `close`, `set_title`, `set_size` | none (queued platform messages; flushed on next event loop iteration) |
