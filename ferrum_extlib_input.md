# Ferrum Extended Library — input

**Module:** `extlib.ccsp.input`
**Spec basis:** CCSP `lib_ccsp_input` unified input event normalization specification
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.draw` (Point, Size, Rect), `extlib.ccsp.draw_wayland` / `extlib.ccsp.draw_x11` / `extlib.ccsp.draw_fb` (platform adapters, feature-gated), `stdlib.alloc`

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [InputEvent — the Unified Type](#2-inputevent--the-unified-type)
3. [PointerEvent](#3-pointerevent)
4. [KeyboardEvent](#4-keyboardevent)
5. [TextInputEvent](#5-textinputevent)
6. [TouchEvent](#6-touchevent)
7. [ScrollEvent](#7-scrollevent)
8. [FocusEvent](#8-focusevent)
9. [Input Source Adapters](#9-input-source-adapters)
10. [IME Control](#10-ime-control)
11. [Keyboard Layout and Character Mapping](#11-keyboard-layout-and-character-mapping)
12. [Clipboard](#12-clipboard)
13. [Error Types](#13-error-types)
14. [Example Usage](#14-example-usage)
15. [Dependencies Reference](#15-dependencies-reference)

---

## 1. Overview and Rationale

### Why Normalize Input

Every display platform delivers input through a different event model.

Wayland sends pointer events through `wl_pointer`, keyboard events through `wl_keyboard`
with XKB state, touch through `wl_touch`, and IME input through `zwp-text-input-v3`. The
keyboard protocol delivers raw keysyms, not characters; characters come separately from
the text-input protocol.

X11 delivers all input through `XEvent` unions — `XKeyEvent`, `XButtonEvent`,
`XMotionEvent`, `XFocusChangeEvent` — each with platform-specific fields. Key-to-character
mapping goes through `XmbLookupString` or `Xutf8LookupString` depending on locale. IME
uses X Input Method (XIM), an entirely separate protocol predating modern IME requirements.

evdev (Linux's raw kernel input layer) delivers `input_event` structs with type/code/value
triples. It has no concept of windows, no coordinate system, and no character composition.
It is used for windowless kiosk applications, game input polling, and accessibility tools.

Win32 delivers `WM_MOUSEMOVE`, `WM_KEYDOWN`, `WM_CHAR`, `WM_INPUT`, and `WM_IME_COMPOSITION`
messages to a window procedure. The split between `WM_KEYDOWN` (physical key) and `WM_CHAR`
(character produced) is the correct model, but the surrounding protocol is entirely
platform-specific.

A widget system should not contain four diverging code paths for "the user pressed a
button." `extlib.ccsp.input` normalizes all four platforms into a single `InputEvent`
type. The widget system, the text field implementation, the gesture recognizer, the
keyboard shortcut handler — all code above this layer — deals with one event vocabulary
regardless of what is underneath. Platform-specific initialization lives entirely in the
adapter implementations below the `InputAdapter` trait boundary.

### Why Float Coordinates

Physical pixel coordinates are wrong for portable GUI code. A display scaled at 1.5x has
no integer coordinate that correctly represents "the pointer is between two physical
pixels." Rounding to the nearest pixel produces jitter at sub-pixel positions, incorrect
hit testing near widget boundaries, and inconsistent behavior across DPI settings.

Float coordinates in logical (device-independent) pixels let the widget system express
geometry without knowing the physical pixel grid. At 1x DPI, `Point { x: 42.0, y: 17.0 }`
maps exactly to physical pixel (42, 17). At 2x DPI, it maps to physical pixel (84, 34). At
1.5x DPI, the compositor handles the fractional mapping. Widget hit-test code is correct at
all DPI values without modification.

This is the same coordinate model used in `extlib.ccsp.draw`, CSS logical pixels, Core
Graphics (macOS), and Android density-independent pixels. Pointer and touch positions
delivered by this module are already in the same logical coordinate space as drawing
operations, so hit testing requires no coordinate conversion.

### Text Input Is Not Key Events

There is a fundamental distinction between physical key events and text input. They are
different things that happen to correlate in some circumstances.

A `KeyboardEvent` describes a physical action: a key with identity `KeyA` was pressed.
This is the correct event for keyboard shortcuts (`Ctrl+S`, `Ctrl+Z`), game movement
(`W`/`A`/`S`/`D`), and focus navigation (`Tab`, `Escape`, `ArrowDown`). It says nothing
about what character the user intends to input.

A `TextInputEvent::Commit` describes the character(s) the user has chosen to insert: the
string `"a"`, or `"A"`, or `"は"`, or `"한글"`. This is the correct event for any text
field. It comes from the platform's input method, which may involve:

- Direct keystroke-to-character mapping for ASCII (the trivial case)
- Dead keys for accented Latin characters (`dead_acute` + `a` → `á`)
- IME composition for CJK scripts: the user types phonetic input across several keystrokes,
  sees a candidate list, and selects a character — all of which is invisible to the
  application until `Commit`
- Voice input, on-screen keyboards, handwriting recognition, autocorrect — none of which
  produces key events at all

An application that derives text from `KeyboardEvent` will:
- Break on any non-US-ASCII keyboard layout
- Break for any CJK user
- Break for voice input and accessibility input methods
- Implement caps lock and shift inconsistently
- Produce wrong output for dead key sequences

`TextInputEvent` is the only correct source of text. `KeyboardEvent` is for keys, not
characters. This module enforces the distinction structurally: `KeyboardEvent` has no
field for a character, and `TextInputEvent` has no field for a key code.

---

## 2. InputEvent — the Unified Type

```ferrum
/// A normalized input event, independent of platform source.
///
/// Produced by a platform adapter implementing InputAdapter.
/// All pointer and touch coordinates are in logical widget-space pixels,
/// matching the coordinate system of extlib.ccsp.draw.
@derive(Debug, Clone)
pub enum InputEvent {
    /// Pointer (mouse, stylus) motion and button state.
    Pointer(PointerEvent),

    /// Physical key press or release. Use for shortcuts and navigation.
    /// Do NOT use for text input — use TextInput instead.
    Keyboard(KeyboardEvent),

    /// Text produced by keyboard + IME. Use for all text field input.
    /// This is the ONLY correct source of input characters.
    TextInput(TextInputEvent),

    /// Touch screen contact.
    Touch(TouchEvent),

    /// Scroll wheel or trackpad scroll.
    Scroll(ScrollEvent),

    /// Window-level focus change.
    Focus(FocusEvent),

    /// The window has been resized. Size is in logical pixels.
    WindowResize(Size),

    /// The user has requested the window be closed (title bar X, Alt+F4, etc.).
    /// The application must decide whether to close; this event alone does nothing.
    WindowClose,
}
```

`InputEvent` is the single type that traverses the entire stack from platform adapter to
widget system. No platform-specific type appears above the `InputAdapter` trait boundary.
Wrapping variants allow exhaustive pattern matching; every platform combination falls into
one of these eight cases.

---

## 3. PointerEvent

### `PointerEvent`

```ferrum
/// A pointer (mouse or stylus) input event.
///
/// `position` is in logical widget-space coordinates.
/// `global_position` is the pointer position in logical screen coordinates.
///
/// For mouse buttons, `pressure` is 0.0 when no button is held and 1.0 when
/// any button is held. For stylus input, `pressure` is the physical pen
/// pressure in [0.0, 1.0].
@derive(Debug, Clone)]
pub struct PointerEvent {
    /// What happened.
    pub kind:            PointerEventKind,

    /// Pointer position in logical widget-space coordinates.
    /// Origin is the top-left corner of the receiving widget.
    pub position:        Point,

    /// Pointer position in logical screen coordinates.
    /// Origin is the top-left corner of the primary monitor.
    pub global_position: Point,

    /// Which button changed state, for ButtonDown and ButtonUp events.
    /// None for Enter, Leave, and Move events.
    pub button:          Option[MouseButton],

    /// Keyboard modifier state at the time of this event.
    pub modifiers:       Modifiers,

    /// Contact pressure. Range [0.0, 1.0].
    /// Mouse: 0.0 when no button pressed, 1.0 when any button pressed.
    /// Stylus: physical pen pressure from hardware.
    /// Touch: reported by hardware if available, otherwise 1.0 on contact.
    pub pressure:        f32,

    /// Stylus tilt angle. None for mouse input.
    /// x component: tilt toward/away from the right edge, in degrees [-90, 90].
    /// y component: tilt toward/away from the bottom edge, in degrees [-90, 90].
    /// (0.0, 0.0) means the stylus is perpendicular to the surface.
    pub tilt:            Option[Point],
}
```

### `PointerEventKind`

```ferrum
/// What kind of pointer event occurred.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PointerEventKind {
    /// The pointer entered the widget's area.
    Enter,

    /// The pointer left the widget's area.
    Leave,

    /// The pointer moved within the widget's area.
    /// Also sent when the pointer moves outside the widget area while a button
    /// is held (pointer capture).
    Move,

    /// A mouse button was pressed.
    ButtonDown,

    /// A mouse button was released.
    ButtonUp,
}
```

### `MouseButton`

```ferrum
/// Identity of a mouse button.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum MouseButton {
    /// Primary button. Left button on a right-handed mouse.
    Left,

    /// Secondary button. Right button on a right-handed mouse.
    Right,

    /// Middle button or scroll wheel click.
    Middle,

    /// Back (browser back / thumb button 1).
    Back,

    /// Forward (browser forward / thumb button 2).
    Forward,

    /// Any additional button not covered by the named variants.
    /// The value is the platform button index (0-based, raw from driver).
    Other(u8),
}
```

### `Modifiers`

```ferrum
/// Keyboard modifier key state at the time of an input event.
///
/// Reflects the logical modifier state, not the physical key state.
/// `shift` is true when either ShiftLeft or ShiftRight is held.
/// `control` is true when either ControlLeft or ControlRight is held.
/// `alt` is true when either AltLeft or AltRight is held.
/// `super_` is true when either MetaLeft or MetaRight is held
/// (the Windows key on PC keyboards, Command on macOS keyboards).
@derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct Modifiers {
    pub shift:     bool,
    pub control:   bool,
    pub alt:       bool,
    /// The Super/Windows/Command key. Named `super_` to avoid reserved word clash.
    pub super_:    bool,
    pub caps_lock: bool,
    pub num_lock:  bool,
}
```

```ferrum
impl Modifiers {
    /// No modifiers held, both lock keys off.
    pub fn none(): Self {
        Modifiers {
            shift: false, control: false, alt: false, super_: false,
            caps_lock: false, num_lock: false,
        }
    }

    /// True if any of Shift, Control, Alt, or Super is held.
    pub fn any_held(&self): bool {
        self.shift || self.control || self.alt || self.super_
    }
}
```

---

## 4. KeyboardEvent

### `KeyboardEvent`

```ferrum
/// A physical key press or release event.
///
/// `key` identifies the physical key by its USB HID position, not by the
/// character it produces. Layout-independence is intentional: keyboard
/// shortcuts must work regardless of which layout is active.
///
/// Do NOT use KeyboardEvent to obtain text. Use TextInputEvent instead.
/// `KeyboardEvent` has no character field by design.
@derive(Debug, Clone)]
pub struct KeyboardEvent {
    /// Whether the key was pressed or released.
    pub kind:      KeyEventKind,

    /// Physical key identity, layout-independent.
    pub key:       KeyCode,

    /// Hardware scan code. Platform-specific.
    /// Use this only for keybinding rebinding UI (e.g., "press the key you
    /// want to assign"). Do not use it to derive characters.
    /// Value encoding: evdev codes on Linux, virtual-key codes on Win32,
    /// key codes from the HID usage table on macOS.
    pub scan_code: u32,

    /// Keyboard modifier state at the time of this event.
    pub modifiers: Modifiers,

    /// True if this is a synthetic repeat event generated by the OS key-repeat
    /// mechanism. The key was not physically pressed again.
    /// Typically starts after ~500 ms held, repeats at ~30 Hz. Platform-defined.
    pub repeat:    bool,
}
```

### `KeyEventKind`

```ferrum
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum KeyEventKind {
    /// The key was pressed (or is repeating — check `repeat` field).
    Down,

    /// The key was released.
    Up,
}
```

### `KeyCode`

Physical key identities. These correspond to USB HID usage page 0x07 positions and match
the W3C UI Events KeyboardEvent.code values. Layout-independent: `KeyA` is always the key
in the A position on a QWERTY keyboard, regardless of whether the active layout maps it to
`a`, `q` (Dvorak home row), or `の` (Japanese Kana).

```ferrum
/// Physical key identity, independent of keyboard layout.
///
/// Names refer to the key's position on a standard US ANSI keyboard.
/// On non-ANSI keyboards (ISO, JIS), some keys are physically different
/// but are mapped to the closest equivalent code.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum KeyCode {
    // --- Letter keys (A-Z position) ---
    KeyA, KeyB, KeyC, KeyD, KeyE, KeyF, KeyG, KeyH, KeyI, KeyJ,
    KeyK, KeyL, KeyM, KeyN, KeyO, KeyP, KeyQ, KeyR, KeyS, KeyT,
    KeyU, KeyV, KeyW, KeyX, KeyY, KeyZ,

    // --- Digit row (0-9, above letters) ---
    Digit0, Digit1, Digit2, Digit3, Digit4,
    Digit5, Digit6, Digit7, Digit8, Digit9,

    // --- Function keys ---
    F1,  F2,  F3,  F4,  F5,  F6,
    F7,  F8,  F9,  F10, F11, F12,
    F13, F14, F15, F16, F17, F18,
    F19, F20, F21, F22, F23, F24,

    // --- Navigation cluster ---
    ArrowLeft, ArrowRight, ArrowUp, ArrowDown,
    Home, End, PageUp, PageDown,

    // --- Editing cluster ---
    Backspace, Delete, Insert,
    Return, Tab, Escape,

    // --- Whitespace ---
    Space,

    // --- Modifier keys ---
    ShiftLeft,   ShiftRight,
    ControlLeft, ControlRight,
    AltLeft,     AltRight,
    MetaLeft,    MetaRight,   // Windows key / Command key

    // --- Punctuation (US ANSI positions) ---
    Comma,       // ,<
    Period,      // .>
    Slash,       // /?
    Backslash,   // \|
    BracketLeft, // [{
    BracketRight,// ]}
    Quote,       // '"
    Backquote,   // `~
    Minus,       // -_
    Equal,       // =+
    Semicolon,   // ;:

    // --- Numpad ---
    Numpad0, Numpad1, Numpad2, Numpad3, Numpad4,
    Numpad5, Numpad6, Numpad7, Numpad8, Numpad9,
    NumpadAdd, NumpadSubtract, NumpadMultiply, NumpadDivide,
    NumpadEnter, NumpadDecimal,

    // --- System / media ---
    CapsLock, NumLock, ScrollLock,
    PrintScreen, Pause,
    MediaPlayPause, MediaStop, MediaNextTrack, MediaPrevTrack,
    VolumeUp, VolumeDown, VolumeMute,

    // --- Catch-all for keys not listed above ---
    // Value is the raw platform scan code.
    Unknown(u32),
}
```

---

## 5. TextInputEvent

### `TextInputEvent`

```ferrum
/// Text produced by keyboard input, dead keys, or IME composition.
///
/// This is the ONLY correct source of characters for text field input.
/// KeyboardEvent delivers physical key identities; TextInputEvent delivers
/// the characters the user intends to insert.
///
/// For plain ASCII input without an active IME, the platform generates
/// Commit events automatically. No special handling is required for the
/// common ASCII case; the same code handles ASCII and CJK correctly.
@derive(Debug, Clone)]
pub struct TextInputEvent {
    pub kind: TextInputKind,
}
```

### `TextInputKind`

```ferrum
/// What kind of text input event occurred.
@derive(Debug, Clone)]
pub enum TextInputKind {
    /// The user has committed final text to be inserted at the cursor.
    ///
    /// For ASCII input: a single character, e.g. "a", "A", "3", "!".
    /// For IME input: a completed character sequence, e.g. "は", "한", "你好".
    /// For dead keys: the composed character, e.g. "á" (dead_acute + a).
    ///
    /// The application must insert `text` at the current cursor position.
    Commit {
        text: String,
    },

    /// IME composition is in progress. The text is a preview, not yet committed.
    ///
    /// Display `text` inline at the cursor position in a visually distinct style
    /// (typically underlined). `cursor` is the byte range within `text` where the
    /// IME wants the visual cursor placed during composition.
    ///
    /// A new PreEdit replaces the previous PreEdit. An empty `text` string
    /// cancels the current composition (same as the user pressing Escape in
    /// the IME candidate window).
    PreEdit {
        text:   String,
        cursor: Range[usize],
    },

    /// The IME is requesting deletion of surrounding text.
    ///
    /// `before` is the number of UTF-8 bytes to delete before the cursor.
    /// `after` is the number of UTF-8 bytes to delete after the cursor.
    ///
    /// Some IMEs (notably certain Korean and Japanese IMEs) need to delete
    /// previously committed characters to re-enter composition state.
    DeleteSurrounding {
        before: u32,
        after:  u32,
    },
}
```

### The Text Input Invariant

`TextInputEvent::Commit` is the only correct way to obtain characters from keyboard input.

An application that pattern-matches on `KeyboardEvent` to convert `KeyA` to `'a'` will
fail for every non-trivial input case: Shift for uppercase, Caps Lock state, AltGr for
third-level characters, dead key sequences, any IME, voice input, on-screen keyboards.
This class of bug is not theoretical — it affects every non-English keyboard layout and
every CJK user.

For ASCII text input without an active IME, the platform generates `Commit` events
automatically. No IME-specific code is required; the same text field implementation that
handles `Commit { text: "a" }` handles `Commit { text: "は" }` without modification.

---

## 6. TouchEvent

### `TouchEvent`

```ferrum
/// A touch screen contact event.
///
/// Each contact point (finger) has a unique `id` that is stable across the
/// Down → Move* → Up sequence for that contact. When a second finger touches
/// the screen while the first is still down, it receives a different `id`.
/// `id` values are not reused until all fingers are lifted.
///
/// `position` and `global_position` are in logical pixels, matching the
/// coordinate system of extlib.ccsp.draw and PointerEvent.
@derive(Debug, Clone)]
pub struct TouchEvent {
    /// What happened to this contact point.
    pub kind:            TouchEventKind,

    /// Unique identifier for this touch contact, stable within a sequence.
    pub id:              u64,

    /// Contact position in logical widget-space coordinates.
    pub position:        Point,

    /// Contact position in logical screen coordinates.
    pub global_position: Point,

    /// Contact pressure in [0.0, 1.0], if reported by hardware.
    /// Hardware that does not report pressure delivers 1.0 on contact.
    pub pressure:        f32,

    /// Contact area as an ellipse, if reported by hardware.
    /// x = half-width of contact ellipse in logical pixels.
    /// y = half-height of contact ellipse in logical pixels.
    /// None when the hardware does not report contact area.
    pub radius:          Option[Point],
}
```

### `TouchEventKind`

```ferrum
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TouchEventKind {
    /// A new contact point appeared (finger touched the screen).
    Down,

    /// An existing contact point moved.
    Move,

    /// A contact point was lifted (finger left the screen).
    Up,

    /// The contact sequence was cancelled.
    ///
    /// Sent when the OS takes over the touch stream (e.g., swipe-from-edge
    /// system gesture intercepted, phone call overlay, accessibility interrupt).
    /// The application must abandon any gesture in progress for this `id` as
    /// if the touch had never occurred.
    Cancel,
}
```

### Touch Identity Semantics

Each `id` tracks a single finger from `Down` through all subsequent `Move` events to
either `Up` or `Cancel`. When two fingers are on screen simultaneously, they have distinct
`id` values. After `Up` or `Cancel`, that `id` may be reused in a future touch sequence
(after all contacts are lifted), so gesture recognizers must key their state on `id` only
within an active sequence, not across sequences.

---

## 7. ScrollEvent

### `ScrollEvent`

```ferrum
/// A scroll event from a mouse wheel or trackpad.
///
/// Mouse wheel and trackpad are distinguished by `delta` variant:
/// - Mouse wheel delivers Lines delta (one detent = 1.0)
/// - Trackpad delivers Pixels delta (continuous, sub-pixel precision)
///
/// `phase` is relevant for trackpad momentum scrolling; mouse wheels
/// do not have meaningful phase information and deliver MayBegin / Changed
/// for each detent click.
@derive(Debug, Clone)]
pub struct ScrollEvent {
    /// Pointer position at the time of the scroll, in logical widget-space coordinates.
    pub position:  Point,

    /// How much to scroll.
    pub delta:     ScrollDelta,

    /// Keyboard modifier state. Shift+scroll is conventionally horizontal scroll.
    pub modifiers: Modifiers,

    /// Phase of a trackpad scroll gesture.
    pub phase:     ScrollPhase,
}
```

### `ScrollDelta`

```ferrum
/// The amount to scroll, in one of two units depending on the input device.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum ScrollDelta {
    /// Continuous pixel-precise scroll from a trackpad or smooth-scroll mouse.
    ///
    /// dx is positive for scroll right, negative for scroll left.
    /// dy is positive for scroll down, negative for scroll up.
    /// Values are in logical pixels.
    Pixels { dx: f32, dy: f32 },

    /// Discrete line-based scroll from a mouse wheel with physical detents.
    ///
    /// dx and dy are in units of "one detent click." 1.0 = one detent.
    /// How many pixels to scroll per line is the application's decision
    /// (conventionally 3 lines of text or ~40-60 logical pixels per detent).
    /// Fractional values are possible on some platforms with high-resolution wheels.
    Lines { dx: f32, dy: f32 },
}
```

### `ScrollPhase`

```ferrum
/// Phase of a trackpad scroll gesture.
///
/// Trackpad scroll gestures have a lifecycle: the user places two fingers
/// on the pad (MayBegin), begins moving (Began), continues (Changed),
/// lifts their fingers (Ended), and the content continues to scroll with
/// decelerating momentum (more Changed events). Cancel occurs if the OS
/// interrupts the gesture.
///
/// Mouse wheel events do not have meaningful phases; each detent click
/// produces a standalone Changed event.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ScrollPhase {
    /// Two fingers are on the trackpad but no scroll motion has been detected yet.
    MayBegin,

    /// Scroll motion has begun. First event with actual movement.
    Began,

    /// Scroll motion is ongoing (finger moving, or momentum decelerating after lift).
    Changed,

    /// The user's fingers have left the pad. Momentum Changed events may follow.
    Ended,

    /// The scroll gesture was cancelled before completing.
    Cancelled,
}
```

### Momentum Scrolling

On platforms with momentum scrolling (macOS trackpad, some Wayland compositors with
`wp-pointer-gestures`), a sequence of `ScrollPhase::Changed` events continues after
`Ended`, with `Pixels` deltas that decelerate to zero. Applications implementing
scroll views should apply these events normally; the deceleration curve is computed
by the platform, not the application. Applications that want to interrupt momentum
(e.g., the user clicks a scrollbar) should call `ImeController` and re-engage the
scroll position — the momentum events simply stop when the platform detects the
next user input.

---

## 8. FocusEvent

### `FocusEvent`

```ferrum
/// A window-level focus change event.
///
/// This is window focus, not widget focus. Widget focus (which text field
/// is active, which button is highlighted) is managed by the widget system
/// above this layer.
///
/// When a window loses focus, the application should:
/// - Cancel any active drag operations
/// - Release any captured pointer
/// - Pause animations if appropriate
///
/// When a window gains focus, the application should:
/// - Re-read clipboard availability if it depends on focus
/// - Resume any paused animations
@derive(Debug, Clone)]
pub struct FocusEvent {
    pub kind: FocusKind,
}
```

### `FocusKind`

```ferrum
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FocusKind {
    /// This window became the foreground window.
    Gained,

    /// This window lost foreground status (another window was activated,
    /// or the user switched to another application).
    Lost,
}
```

---

## 9. Input Source Adapters

Platform adapters translate native events into `InputEvent` values. Each adapter
implements the `InputAdapter` trait. No `InputEvent` field contains a platform type;
the conversion is complete at the adapter boundary.

### `InputAdapter` trait

```ferrum
/// Common interface for all platform input adapters.
///
/// Implementations are provided for Wayland, X11, evdev, and Win32.
/// The widget system depends only on this trait, not on any concrete adapter.
pub trait InputAdapter {
    /// Return the next pending input event, or None if the event queue is empty.
    ///
    /// Does not block. Returns None immediately when no events are available.
    /// The caller is responsible for blocking on the platform event fd (see
    /// event_fd() on the relevant connection type) when it needs to wait.
    fn next_event(&mut self): Option[InputEvent] ! IO
}
```

### `WaylandInputAdapter`

Wraps `wl_seat` events from an active Wayland connection. Pointer events come from
`wl_pointer`, keyboard events from `wl_keyboard` (with XKB state for modifier tracking),
touch events from `wl_touch`, and text input events from `zwp-text-input-v3`. Scroll
events from `wl_pointer` axis events use `wl_pointer.axis_source` to distinguish trackpad
(finger source → `Pixels`) from mouse wheel (wheel source → `Lines`).

```ferrum
pub type WaylandInputAdapter

impl WaylandInputAdapter {
    /// Create an input adapter for the given Wayland window.
    ///
    /// The adapter registers `wl_seat` listeners on the window's connection.
    /// Events are translated to InputEvent and buffered internally until
    /// next_event() is called.
    pub fn new(window: &WaylandWindow): WaylandInputAdapter
}

impl InputAdapter for WaylandInputAdapter
```

### `X11InputAdapter`

Wraps X11 events from an active X11 connection. Key events come through XCB's
`xcb_key_press_event_t`; character mapping uses XKB via `xkbcommon` to handle dead keys
and compose sequences. Text input uses XIM (`XOpenIM` / `XCreateIC`) with over-the-spot
or root-window preedit style. Pointer events from `xcb_motion_notify_event_t` and
`xcb_button_press_event_t`.

```ferrum
pub type X11InputAdapter

impl X11InputAdapter {
    /// Create an input adapter for the given X11 window.
    pub fn new(window: &X11Window): X11InputAdapter
}

impl InputAdapter for X11InputAdapter
```

### `EvdevInputAdapter`

Reads raw kernel input events from an evdev device node (e.g., `/dev/input/event0`).
For windowless applications: kiosk displays, game input polling, accessibility tools.
Has no window coordinate system; all pointer coordinates are relative to the device's
reported absolute axes. Does not produce `TextInput` events (no compositor, no IME).
Requires read permission on the device file; typically `input` group membership or
a `udev` rule granting access.

```ferrum
pub type EvdevInputAdapter

impl EvdevInputAdapter {
    /// Open an evdev device and create an input adapter for it.
    ///
    /// Returns InputError::DeviceNotFound if the path does not exist.
    /// Returns InputError::PermissionDenied if the process lacks read access.
    pub fn new(device: &Path): Result[EvdevInputAdapter, InputError] ! IO
}

impl InputAdapter for EvdevInputAdapter
```

### `Win32InputAdapter`

Wraps Win32 window messages. Pointer events come from `WM_MOUSEMOVE` /
`WM_LBUTTONDOWN` / etc. Key events from `WM_KEYDOWN` / `WM_KEYUP`. Text input from
`WM_CHAR` and `WM_IME_COMPOSITION` / `WM_IME_ENDCOMPOSITION`. Touch events from
`WM_TOUCH` (legacy) or `WM_POINTER` (Windows 8+, preferred). Scroll from
`WM_MOUSEWHEEL` / `WM_MOUSEHWHEEL`.

```ferrum
pub type Win32InputAdapter

impl Win32InputAdapter {
    /// Create an input adapter for the given Win32 window handle.
    pub fn new(hwnd: Win32Hwnd): Win32InputAdapter
}

impl InputAdapter for Win32InputAdapter
```

---

## 10. IME Control

An `ImeController` communicates back to the platform's input method framework. The
application tells the IME where the text cursor is (so the candidate window appears near
the insertion point), what kind of input is expected, and what text already surrounds the
cursor (so the IME can offer context-aware suggestions).

Without IME control, the candidate window appears in the top-left corner of the screen, or
wherever the platform defaults it. Applications that display a text field should create an
`ImeController` for that field and keep its cursor rectangle up to date.

```ferrum
pub type ImeController

impl ImeController {
    /// Create an IME controller linked to the given adapter.
    ///
    /// The adapter must remain alive at least as long as the controller.
    pub fn new[A: InputAdapter](adapter: &mut A): ImeController

    /// Enable or disable text input mode for the associated window.
    ///
    /// When enabled, the platform activates the input method and begins
    /// delivering TextInput events. When disabled, TextInput events stop
    /// and the IME candidate window is hidden.
    ///
    /// Call with `true` when a text field gains focus, `false` when it loses focus.
    pub fn set_input_enabled(&mut self, enabled: bool)

    /// Inform the IME where the text insertion cursor is.
    ///
    /// `rect` is in logical widget-space coordinates and should describe the
    /// visual cursor rectangle (position and line height). The platform uses
    /// this to position the IME candidate window near the cursor, avoiding
    /// overlap with the text being composed.
    ///
    /// Call whenever the cursor moves, the field scrolls, or the window moves.
    pub fn set_cursor_rect(&mut self, rect: Rect)

    /// Inform the IME of the expected input type.
    ///
    /// Some IMEs adapt their behavior to the input type: suppressing emoji
    /// for phone number fields, offering URL completion for URL fields,
    /// hiding the candidate window for password fields.
    pub fn set_input_type(&mut self, input_type: InputType)

    /// Provide text surrounding the cursor to the IME.
    ///
    /// `text` is a window of the document text around the cursor (need not be
    /// the full document). `cursor` is the byte range of the selection within
    /// `text` (collapsed cursor = empty range).
    ///
    /// Some IMEs use surrounding text for context-aware prediction and
    /// backspace behavior during composition. Providing it improves IME quality.
    /// Callers that cannot efficiently extract surrounding text may omit this call.
    pub fn set_surrounding_text(&mut self, text: &str, cursor: Range[usize])
}
```

### `InputType`

```ferrum
/// The type of input expected by the focused text field.
///
/// Used as a hint to the platform input method. The IME may adapt its
/// keyboard layout, candidate list, or correction behavior accordingly.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum InputType {
    /// General text. Default. IME fully active.
    Text,

    /// Multi-line text. Same as Text but signals newline is a valid input character.
    Multiline,

    /// Password. IME should suppress candidate window and autocorrect.
    /// Platform may also mask key-repeat timing to reduce side-channel leakage.
    Password,

    /// Numeric input. IME may show a numeric keypad.
    Number,

    /// Telephone number. IME may show a dial pad layout.
    Tel,

    /// Email address. IME may suppress autocorrect and show @ symbol prominently.
    Email,

    /// URL. IME may suppress autocorrect and show / and . prominently.
    Url,

    /// Search query. IME may show a search/go action button.
    Search,
}
```

---

## 11. Keyboard Layout and Character Mapping

### Purpose and Scope

`KeyCode::to_display_char` answers the question: "On the user's current keyboard layout,
what character does this physical key position produce?" This is used exclusively to
display keybinding labels in user interfaces, such as "Press Ctrl+S to save" or
"Shortcut: Ctrl+Z."

It is not used for text input. Text input comes from `TextInputEvent`. Using
`to_display_char` for text input will produce incorrect results for any user whose
keyboard layout differs from the assumptions in the calling code.

### API

```ferrum
/// A snapshot of the currently active keyboard layout.
///
/// Obtained from the platform via KeyboardLayout::current().
/// Does not update automatically when the user switches layouts;
/// call current() again after receiving a layout-change notification
/// from the platform (delivered as a platform-specific event outside
/// the InputEvent model).
pub type KeyboardLayout

impl KeyboardLayout {
    /// Query the current keyboard layout from the platform.
    ///
    /// On Wayland, reads the XKB keymap from wl_keyboard::keymap.
    /// On X11, queries XKB via xkbcommon.
    /// On Win32, queries GetKeyboardLayout().
    pub fn current(): KeyboardLayout ! IO

    /// Short name of the layout as reported by the platform.
    ///
    /// Examples: "us", "de", "fr", "dvorak", "colemak", "jp", "kr".
    /// Format is platform-defined; do not parse or compare across platforms.
    pub fn name(&self): &str
}

impl KeyCode {
    /// The character this key produces on the given layout with the given modifiers.
    ///
    /// Returns None for keys that do not produce a printable character:
    /// modifier keys, function keys, navigation keys, media keys.
    ///
    /// Returns None for dead keys (keys that begin a compose sequence).
    /// Dead key sequences are handled internally by the platform text input
    /// protocol and surface as TextInputEvent::Commit; they cannot be
    /// previewed character-by-character through this API.
    ///
    /// Use this only for keybinding label display, not for text input.
    pub fn to_display_char(
        self,
        modifiers: Modifiers,
        layout:    &KeyboardLayout,
    ): Option[char]
}
```

---

## 12. Clipboard

Platform clipboard access. The clipboard is a shared OS-level resource; operations may
fail if the clipboard is owned by another process that has exited, if the requested
format is not available, or if the platform denies access.

On X11, this module uses the `CLIPBOARD` selection (not `PRIMARY`). The `PRIMARY`
selection (middle-click paste) is outside the scope of this module.

On Wayland, clipboard access uses `wl_data_device`. Clipboard reads require a round-trip
to the data source process; they are asynchronous at the protocol level and are wrapped
synchronously here with a short timeout.

On Win32, this module uses the standard Win32 clipboard (`OpenClipboard` /
`GetClipboardData` / `SetClipboardData`).

```ferrum
pub type Clipboard

impl Clipboard {
    /// Write UTF-8 text to the clipboard.
    ///
    /// Replaces any previous clipboard content.
    pub fn set_text(&self, text: &str): Result[(), InputError]

    /// Read UTF-8 text from the clipboard.
    ///
    /// Returns InputError::ClipboardEmpty if the clipboard contains no text.
    /// Returns InputError::ClipboardFormatUnavailable if the clipboard contains
    /// only non-text data.
    pub fn get_text(&self): Result[String, InputError] ! IO

    /// Write arbitrary binary data to the clipboard with the given MIME type.
    ///
    /// `mime_type` should be a standard MIME type string, e.g. "image/png",
    /// "text/html", "application/json".
    /// Multiple MIME types can be set by calling set_data multiple times
    /// before relinquishing clipboard ownership.
    pub fn set_data(&self, mime_type: &str, data: Vec[u8]): Result[(), InputError]

    /// Read binary data from the clipboard for the given MIME type.
    ///
    /// Returns InputError::ClipboardFormatUnavailable if the requested MIME type
    /// is not present on the clipboard.
    pub fn get_data(&self, mime_type: &str): Result[Vec[u8], InputError] ! IO

    /// List the MIME types currently available on the clipboard.
    ///
    /// Returns an empty Vec if the clipboard is empty.
    /// The caller can use this to check availability before calling get_data.
    pub fn available_types(&self): Vec[String]
}
```

---

## 13. Error Types

```ferrum
/// Error type for input adapter initialization, IME operations, and clipboard access.
@derive(Debug, Clone)]
pub enum InputError {
    /// Platform adapter could not be initialized.
    /// The String contains a platform-specific description of the failure
    /// (e.g., "wl_seat global not advertised", "XOpenIM returned null").
    AdapterInitFailed(String),

    /// Text input was requested but no IME framework is available.
    ///
    /// On Wayland: zwp-text-input-v3 was not advertised by the compositor.
    /// On X11: XOpenIM failed (no input method running).
    /// This is a soft failure: basic ASCII input still works via key events;
    /// only IME-based input (CJK, dead keys via IM) is unavailable.
    ImeNotAvailable,

    /// The clipboard was read but contained no data.
    ClipboardEmpty,

    /// The clipboard contains data, but not in the requested format.
    ClipboardFormatUnavailable,

    /// The evdev device path does not exist.
    DeviceNotFound { path: String },

    /// The process lacks permission to open the evdev device.
    ///
    /// Typically resolved by adding the process user to the `input` group
    /// or adding a udev rule: `KERNEL=="event*", GROUP="input", MODE="0660"`.
    PermissionDenied,
}
```

---

## 14. Example Usage

### Keyboard Shortcut Handling

Use `KeyboardEvent` for shortcuts. Match on `KeyCode` and `Modifiers`. Do not involve
`TextInputEvent` in shortcut handling.

```ferrum
use extlib::ccsp::input::{InputEvent, KeyboardEvent, KeyEventKind, KeyCode, Modifiers}

fn handle_event(event: InputEvent, doc: &mut Document) {
    match event {
        InputEvent::Keyboard(KeyboardEvent { kind: KeyEventKind::Down, key, modifiers, repeat, .. }) => {
            // Standard save shortcut: Ctrl+S
            if key == KeyCode::KeyS && modifiers.control && !modifiers.shift && !modifiers.alt {
                doc.save()
            }
            // Undo: Ctrl+Z
            if key == KeyCode::KeyZ && modifiers.control && !modifiers.shift {
                doc.undo()
            }
            // Redo: Ctrl+Shift+Z or Ctrl+Y
            if (key == KeyCode::KeyZ && modifiers.control && modifiers.shift)
                || (key == KeyCode::KeyY && modifiers.control)
            {
                doc.redo()
            }
            // Navigation: arrow keys, allow auto-repeat
            match key {
                KeyCode::ArrowLeft  => doc.cursor_left(repeat),
                KeyCode::ArrowRight => doc.cursor_right(repeat),
                KeyCode::Home       => doc.cursor_line_start(),
                KeyCode::End        => doc.cursor_line_end(),
                _                   => {}
            }
        }
        _ => {}
    }
}
```

### Text Field Input

Use `TextInputEvent` for all character insertion. Key events are used only for cursor
movement, deletion, and confirming or cancelling composition.

```ferrum
use extlib::ccsp::input::{InputEvent, KeyboardEvent, KeyEventKind, KeyCode}
use extlib::ccsp::input::{TextInputEvent, TextInputKind}

fn handle_text_field(event: InputEvent, field: &mut TextField) {
    match event {
        // Text input: the ONLY correct source of characters.
        InputEvent::TextInput(TextInputEvent { kind }) => {
            match kind {
                TextInputKind::Commit { text } => {
                    field.insert_text(&text)
                }
                TextInputKind::PreEdit { text, cursor } => {
                    field.set_composition_preview(&text, cursor)
                }
                TextInputKind::DeleteSurrounding { before, after } => {
                    field.delete_surrounding(before, after)
                }
            }
        }
        // Key events: structural editing only, no character derivation.
        InputEvent::Keyboard(KeyboardEvent { kind: KeyEventKind::Down, key, .. }) => {
            match key {
                KeyCode::Backspace => field.delete_backward(),
                KeyCode::Delete    => field.delete_forward(),
                KeyCode::Return    => field.submit(),
                KeyCode::Escape    => field.cancel_composition(),
                KeyCode::ArrowLeft => field.move_cursor_left(),
                KeyCode::ArrowRight=> field.move_cursor_right(),
                _                  => {}
            }
        }
        _ => {}
    }
}
```

### Touch Gesture Recognizer

Track active touch contacts by `id`. Use `Cancel` to abandon gestures when the OS
interrupts the touch stream.

```ferrum
use extlib::ccsp::input::{InputEvent, TouchEvent, TouchEventKind}
use stdlib::collections::HashMap

struct TapRecognizer {
    active: HashMap[u64, Point],
}

impl TapRecognizer {
    fn handle(&mut self, event: InputEvent): Option[Point] {
        match event {
            InputEvent::Touch(TouchEvent { kind, id, position, .. }) => {
                match kind {
                    TouchEventKind::Down => {
                        self.active.insert(id, position);
                        None
                    }
                    TouchEventKind::Move => {
                        // Check for movement beyond tap threshold.
                        if let Some(start) = self.active.get(&id) {
                            if position.distance(*start) > 8.0 {
                                self.active.remove(&id);
                            }
                        }
                        None
                    }
                    TouchEventKind::Up => {
                        if self.active.remove(&id).is_some() {
                            Some(position)  // Tap confirmed.
                        } else {
                            None
                        }
                    }
                    TouchEventKind::Cancel => {
                        self.active.remove(&id);
                        None
                    }
                }
            }
            _ => None
        }
    }
}
```

### Clipboard Paste

```ferrum
use extlib::ccsp::input::{Clipboard, InputError}

fn paste_text(clipboard: &Clipboard, field: &mut TextField): Result[(), InputError] ! IO {
    let text = clipboard.get_text()?
    field.insert_text(&text)
    Ok(())
}

fn paste_image(clipboard: &Clipboard, canvas: &mut Canvas): Result[(), InputError] ! IO {
    let available = clipboard.available_types()
    if available.iter().any(|t| t == "image/png") {
        let data = clipboard.get_data("image/png")?
        canvas.load_png_bytes(&data)
    } else {
        Err(InputError::ClipboardFormatUnavailable)
    }
}
```

---

## 15. Dependencies Reference

| Dependency | Role |
|---|---|
| `extlib.ccsp.draw` | `Point`, `Size`, `Rect` — geometry types shared with the drawing layer |
| `extlib.ccsp.draw_wayland` | `WaylandWindow` type consumed by `WaylandInputAdapter::new` |
| `extlib.ccsp.draw_x11` | `X11Window` type consumed by `X11InputAdapter::new` |
| `extlib.ccsp.draw_fb` | No direct input adapter; evdev adapter works alongside fb |
| `stdlib.alloc` | `String`, `Vec`, `HashMap` — heap allocation for event payloads |

Platform adapter modules (`draw_wayland`, `draw_x11`) are feature-gated. A program
targeting only Win32 does not link against libwayland-client. A program targeting only
Wayland does not link against XCB. The `EvdevInputAdapter` depends only on Linux kernel
headers and `stdlib.sys.posix`; it is available on any Linux target regardless of display
server.
