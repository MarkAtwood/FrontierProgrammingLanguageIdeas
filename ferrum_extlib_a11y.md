# Ferrum Extended Library — a11y

**Module path:** `extlib::ccsp::a11y`
**Provided by:** `lib_ccsp_a11y`
**Depends on:** `extlib::ccsp::draw` (Rect), `extlib::ccsp::widget` (A11yCtx integration),
`stdlib::alloc`, `stdlib::sys`
**External dependencies (platform-conditional):** `dbus-rs` (Linux), `objc` (macOS), `windows-rs` (Windows)
**Roadmap status:** Post-1.0 (designed now, implemented after widget system stabilizes)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Accessibility Node Model](#2-accessibility-node-model)
3. [Actions](#3-actions)
4. [Accessibility Tree](#4-accessibility-tree)
5. [Platform Adapter Trait](#5-platform-adapter-trait)
6. [AT-SPI2 Adapter — Linux](#6-at-spi2-adapter--linux)
7. [NSAccessibility Adapter — macOS](#7-nsaccessibility-adapter--macos)
8. [UI Automation Adapter — Windows](#8-ui-automation-adapter--windows)
9. [A11yCtx — Widget Integration](#9-a11yctx--widget-integration)
10. [Live Regions](#10-live-regions)
11. [Keyboard Focus Management](#11-keyboard-focus-management)
12. [Error Types](#12-error-types)
13. [Example Usage](#13-example-usage)
14. [Dependencies Reference](#14-dependencies-reference)

---

## 1. Overview and Rationale

### Legal Requirements

Accessibility support for software is not optional in many jurisdictions. In the
United States, Section 508 of the Rehabilitation Act requires federal agencies and
their vendors to provide accessible electronic information and technology. The
Americans with Disabilities Act (ADA) has been interpreted by courts to apply to
software, including web applications and desktop programs offered as public
accommodations. The Communications and Video Accessibility Act (CVAA) requires
accessible user interfaces for communications software and devices.

In the European Union, EN 301 549 is the harmonised standard that defines
accessibility requirements for ICT products and services procured by public sector
bodies. Software that does not meet EN 301 549 cannot be sold to EU public
institutions. The European Accessibility Act (Directive 2019/882), which became
mandatory in June 2025, extends these requirements to a broad range of private
sector products and services including e-commerce, banking, telecommunications, and
transport booking software.

A Ferrum application that ships without accessibility support is a liability, not a
choice. This module exists to make compliance tractable.

### Moral Obligation

Approximately 15% of the world's population lives with some form of disability.
Screen readers, switch access, and magnification tools are not edge-case concerns —
they are the primary means of computer interaction for millions of people. An
application that cannot be used with a screen reader is an application that has
decided those users do not matter. That is not an acceptable position for software
built with a systems language that aspires to correctness and reliability.

### Accessibility Must Be Built In, Not Added Later

Accessibility cannot be retrofitted onto a widget system designed without it. The
experience of doing so in existing toolkits is instructive: Qt added accessibility
via a separate QAccessible layer that developers routinely forget to implement. GTK
has historically required manual AT-SPI2 wiring per widget. Web applications built
without ARIA from the beginning require large-scale remediation when audited.

Ferrum's widget system invokes `Widget::accessibility()` as a first-class method
alongside `Widget::layout()` and `Widget::paint()`. The accessibility tree is not
a post-hoc annotation layer — it is built during the same pass as layout, from the
same widget data, and kept synchronised with the visual tree automatically. A widget
author who implements `paint()` is expected to implement `accessibility()`. The
compiler can warn when `accessibility()` is missing from a type that implements
`Widget`.

### AT-SPI2 Is the Correct Linux Framework

On Linux, the Assistive Technology Service Provider Interface version 2 (AT-SPI2)
is the standard accessibility framework used by all major desktop environments:
GNOME, KDE Plasma, LXDE, and XFCE. AT-SPI2 was designed as an IPC-based protocol
from the start — it communicates over D-Bus, not via shared memory, in-process
hooks, or window system calls. This design decision has a critical consequence: the
Wayland transition did not break AT-SPI2.

When the Linux desktop moved from X11 to Wayland, accessibility frameworks that
depended on X11 extension protocols (XFixes, XRecord, XI2) broke or required
XWayland compatibility shims. AT-SPI2 was unaffected because it had no X11
dependency to begin with. Applications running natively under a Wayland compositor
expose the same D-Bus accessibility objects as applications running under X11. The
screen reader Orca, which is the primary screen reader used on Linux, consumes
AT-SPI2 events and works identically under both display servers.

### Why extlib, Not stdlib

The accessibility module has a hard dependency on D-Bus (via the accessibility
bus, `org.a11y.Bus`) on Linux, on Objective-C runtime FFI on macOS, and on COM
server registration on Windows. These platform-specific, non-trivial runtime
dependencies make this module unsuitable for the standard library. Additionally,
not every Ferrum program presents a graphical user interface: command-line tools,
daemons, embedded firmware, and network servers have no need for GUI accessibility
infrastructure. Placing it in extlib keeps the dependency opt-in.

---

## 2. Accessibility Node Model

### 2.1 Node Identity

```ferrum
/// Stable identity for an accessibility node across tree updates.
///
/// IDs are assigned by the application and must be unique within a single
/// A11yTree. Once assigned, an ID is stable for the lifetime of the node.
/// Screen readers and assistive technologies cache node IDs; reassigning IDs
/// to different nodes causes confusion in AT state machines. Do not recycle IDs.
///
/// The widget system derives IDs from widget identity (typically a stable hash
/// of the widget's position in the component tree) to ensure stability across
/// repaints and partial tree updates.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)
pub type A11yNodeId(u64)
```

### 2.2 Roles

```ferrum
/// The semantic role of an accessibility node.
///
/// Roles describe what a UI element *is*, not how it looks. A button that is
/// styled to look like a link is still role Button. An anchor that navigates
/// in-page is role Link. Correct role assignment is how screen readers announce
/// elements: "Save, button" vs "Learn more, link" vs "Search, edit text".
///
/// Roles map to platform role constants during adapter publishing:
///   AT-SPI2: Atspi.Role.*
///   macOS:   NSAccessibility*Role constants
///   Windows: UIA_*ControlTypeId constants
@derive(Debug, Clone, PartialEq)
pub enum A11yRole {
    // Top-level containers
    Window,
    Application,

    // Interactive controls
    Button,
    CheckBox,
    RadioButton,
    ComboBox,
    ListBox,
    ListItem,
    MenuItem,
    MenuBar,
    Menu,

    // Text input
    TextInput,
    PasswordInput,
    SearchBox,

    // Range controls
    Slider,
    ProgressBar,
    ScrollBar,

    // Containers and layout
    ScrollPane,
    Table,
    TableRow,
    TableCell,
    ColumnHeader,
    RowHeader,
    TreeItem,
    TreeView,
    Tab,
    TabList,
    TabPanel,
    Toolbar,
    StatusBar,

    // Dialogs and alerts
    Alert,
    Dialog,

    // Document structure
    Heading { level: u8 },  // level 1–6, maps to ARIA aria-level
    Link,
    Image,
    Canvas,
    Figure,
    Label,

    // Landmark regions (ARIA landmark roles / HTML5 sectioning elements)
    Group,
    Section,
    Navigation,
    Main,
    Complementary,
    Banner,
    ContentInfo,
    Form,
    Search,
    Article,

    // Fallback for elements that have no more specific role.
    // Use sparingly: assistive technologies announce Generic as "group" or
    // provide no role announcement, giving users no semantic information.
    Generic,
}
```

### 2.3 State

```ferrum
/// The dynamic state of an accessibility node.
///
/// State changes must be published immediately. When a checkbox is checked,
/// the checked field must be updated and A11yTree::update_node called before
/// the next frame. Stale state is a correctness defect, not a performance
/// optimisation.
///
/// All bool fields default to false. Option fields default to None, meaning
/// the property is not applicable to this node's role.
@derive(Debug, Clone, PartialEq)]
pub type A11yState {
    /// The control is enabled and can receive user input.
    /// false means the control is visually dimmed and non-interactive.
    enabled: bool,

    /// The control currently holds keyboard focus.
    /// Only one node in a tree may have focused: true at any time.
    focused: bool,

    /// For CheckBox and RadioButton: whether the control is checked.
    /// None for roles where checked state is not applicable.
    /// Some(None) represents the indeterminate/mixed state (tri-state checkbox).
    checked: Option[Option[bool]],

    /// For ComboBox, TreeItem, Menu: whether the subtree is expanded.
    /// None for roles where expand/collapse is not applicable.
    expanded: Option[bool],

    /// For ListItem, TableCell, Tab, TreeItem: whether this item is selected.
    selected: bool,

    /// For Link: whether the link destination has been visited.
    visited: bool,

    /// For TextInput and form controls: whether a value is required.
    /// Maps to ARIA aria-required and HTML required attribute semantics.
    required: bool,

    /// For form controls: whether the current value fails validation.
    /// Maps to ARIA aria-invalid. When true, the description field should
    /// contain the validation error message.
    invalid: bool,

    /// The node is intentionally hidden from assistive technologies.
    /// Use for decorative elements, duplicate labels, and visually-hidden
    /// spacers that add no semantic content. Equivalent to aria-hidden="true".
    /// Hidden nodes are excluded from the AT tree walk entirely.
    hidden: bool,

    /// The control displays a value but does not accept user edits.
    read_only: bool,

    /// For ListBox, Table, TreeView: multiple items may be selected.
    multiselectable: bool,
}
```

### 2.4 Node Descriptor

```ferrum
/// A complete description of one accessibility node.
///
/// This is the unit of communication between the application and the platform
/// adapter. Every field is optional except id and role. Supply as many fields
/// as are meaningful for the role: a ProgressBar with no value gives the user
/// no information about how much progress has been made.
@derive(Debug, Clone)]
pub type A11yNodeDesc {
    /// Stable identity. Must be unique within the tree. See A11yNodeId.
    id: A11yNodeId,

    /// Semantic role. See A11yRole.
    role: A11yRole,

    /// Short accessible name for the element.
    ///
    /// This is the primary announcement for the element. For a button, it is
    /// the button label. For an image, it is the alt text. For a text input,
    /// it is the visible label or placeholder. It must be brief — one to a
    /// few words — and must identify the element unambiguously in context.
    ///
    /// Computation order (highest wins):
    ///   1. An explicit label set here.
    ///   2. The text content of a Label node that references this node.
    ///   3. The element's visible text content (for Button, MenuItem, etc.).
    ///
    /// Never use the role name as the label. "Button" is not a label.
    label: Option[String],

    /// Longer description providing supplementary information.
    ///
    /// Use for tooltip-style text, instructions, or validation error messages.
    /// Screen readers announce this after the label and role: "Save, button,
    /// Saves the current document."  Do not duplicate the label.
    description: Option[String],

    /// Current value, as a human-readable string.
    ///
    /// For Slider: the numeric value as a string, e.g. "75".
    /// For ProgressBar: percentage, e.g. "42%".
    /// For TextInput: the current text content.
    /// For ComboBox: the selected option.
    /// Not all roles have a meaningful value.
    value: Option[String],

    /// Dynamic state. See A11yState.
    state: A11yState,

    /// Bounding rectangle in logical (device-independent) coordinates,
    /// in the same coordinate space used by extlib::ccsp::draw.
    /// Used by magnification tools to locate the element on screen.
    bounds: Rect,

    /// Ordered list of child node IDs. Order must match visual/document order.
    /// The platform adapter uses this to build the tree structure exposed to ATs.
    children: Vec[A11yNodeId],

    /// Actions the user can invoke on this node. See A11yAction.
    /// Actions are announced by screen readers ("actions: click, expand").
    actions: Vec[A11yAction],
}
```

---

## 3. Actions

```ferrum
/// An action that a user or assistive technology can invoke on a node.
///
/// Actions are declared in A11yNodeDesc::actions and invoked by the platform
/// adapter when the AT requests them. The widget system's callback mechanism
/// bridges from platform invocation back into the Ferrum event system.
///
/// Platform mappings:
///   AT-SPI2:  Atspi.Action interface, GetActionName / DoAction
///   macOS:    NSAccessibilityPerformAction (e.g. NSAccessibilityPressAction)
///   Windows:  IInvokeProvider::Invoke, IToggleProvider::Toggle, etc.
@derive(Debug, Clone, PartialEq)]
pub enum A11yAction {
    /// Activate the primary interaction for this element.
    /// Button: click. Checkbox: toggle. Link: navigate.
    Click,

    /// Move keyboard focus to this element.
    Focus,

    /// Remove keyboard focus from this element.
    Blur,

    /// Scroll the viewport so this element is fully visible.
    ScrollIntoView,

    // Scroll actions on ScrollBar and ScrollPane nodes.
    ScrollUp,
    ScrollDown,
    ScrollLeft,
    ScrollRight,

    /// Increase the value of a Slider or SpinButton.
    Increment,

    /// Decrease the value of a Slider or SpinButton.
    Decrement,

    /// Open the context menu or action menu for this element.
    ShowMenu,

    /// Expand a collapsed ComboBox, TreeItem, or disclosure widget.
    Expand,

    /// Collapse an expanded ComboBox, TreeItem, or disclosure widget.
    Collapse,

    /// Select a ListItem, Tab, TreeItem, or TableCell.
    Select,

    /// Deselect a selected item in a multiselectable container.
    Unselect,

    /// Set the value of a TextInput or ComboBox to the given string.
    /// The platform passes the new value; the widget validates and applies it.
    SetValue(String),

    /// Application-defined action with a descriptive name.
    /// Used for actions that do not map to a standard AT action vocabulary.
    /// The name should be a verb phrase, e.g. "Open in new tab".
    Custom(String),
}
```

---

## 4. Accessibility Tree

The `A11yTree` is the application-side model of the complete accessibility tree. It
is the authoritative source of accessibility information. Platform adapters derive
their platform-specific representations from this tree and push diffs to the AT
infrastructure when it changes.

```ferrum
use extlib::ccsp::a11y::{
    A11yNodeDesc, A11yNodeId, A11yTree, A11yTreeDiff, AnnouncePoliteness, A11yError,
}

pub type A11yTree
```

```ferrum
impl A11yTree {
    /// Construct an empty tree.
    ///
    /// The tree has no root until set_root is called. A tree without a root
    /// cannot be published to a platform adapter.
    pub fn new(): A11yTree

    /// Set the application root node (must have role A11yRole::Application).
    ///
    /// This is the single top-level node of the accessibility tree. Every
    /// other node in the tree is a descendant of the root. Call this once
    /// during application startup. To update the root node's properties,
    /// call update_node with the root's ID.
    pub fn set_root(mut self, node: A11yNodeDesc)

    /// Update an existing node in place, identified by node.id.
    ///
    /// Generates a diff entry of type Updated. The platform adapter will
    /// publish the changed properties to the AT infrastructure. Only changed
    /// fields are transmitted; the adapter performs field-level diffing.
    ///
    /// Panics in debug builds if node.id does not exist in the tree.
    pub fn update_node(mut self, node: A11yNodeDesc)

    /// Remove a node and all of its descendants from the tree.
    ///
    /// The removed node is detached from its parent's children list. Any
    /// A11yNodeIds referencing the removed subtree become invalid.
    /// Generates diff entries of type Removed for each affected node.
    pub fn remove_node(mut self, id: A11yNodeId)

    /// Add a new node as the last child of parent_id.
    ///
    /// The parent node's children list is updated automatically.
    /// Generates a diff entry of type Added.
    ///
    /// Panics in debug builds if parent_id does not exist in the tree.
    pub fn add_child(mut self, parent_id: A11yNodeId, node: A11yNodeDesc)

    /// Announce that keyboard focus has moved to the node with the given ID.
    ///
    /// Passing None clears focus. The platform adapter fires the appropriate
    /// focus event (AT-SPI2 StateChanged:focused, NSAccessibilityFocusedUIElementChangedNotification,
    /// UIA_AutomationFocusChangedEventId).
    ///
    /// The focused field in A11yState must be kept consistent with calls to
    /// this method: update the node's state and call set_focus together.
    pub fn set_focus(mut self, id: Option[A11yNodeId])

    /// Announce a live message to the screen reader.
    ///
    /// Use for status messages, error alerts, chat messages, and loading
    /// indicators that appear outside the normal focus path. The politeness
    /// level controls whether the announcement interrupts current speech.
    ///
    /// This is a direct announcement, not tied to any node. For node-bound
    /// live regions, see set_live_region in Section 10.
    pub fn announce(mut self, text: &str, politeness: AnnouncePoliteness)

    /// Mark a node as a live region. See Section 10.
    pub fn set_live_region(mut self, id: A11yNodeId, politeness: AnnouncePoliteness)

    /// Compute the diff since the last call to this method.
    ///
    /// Called by the platform adapter after each frame or after explicit
    /// mutation. Returns the set of changes needed to synchronise the
    /// platform's AT representation with the current tree state.
    pub fn take_diff(mut self): A11yTreeDiff
}
```

### `A11yTreeDiff`

```ferrum
/// A delta encoding of changes to an A11yTree.
///
/// Platform adapters consume diffs to minimise IPC traffic. Publishing
/// a complete tree on every frame would flood the D-Bus or COM message
/// queue with redundant data. Diffs transmit only what changed.
@derive(Debug)]
pub type A11yTreeDiff {
    /// Nodes added to the tree since the last diff.
    added: Vec[A11yNodeDesc],

    /// Nodes whose properties changed since the last diff.
    updated: Vec[A11yNodeDesc],

    /// IDs of nodes removed from the tree since the last diff.
    removed: Vec[A11yNodeId],

    /// Focus change, if any: None means focus did not change this cycle.
    focus_change: Option[Option[A11yNodeId]],

    /// Live announcements queued since the last diff.
    announcements: Vec[(String, AnnouncePoliteness)],
}
```

### `AnnouncePoliteness`

```ferrum
/// Controls when a screen reader announces a live region update.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum AnnouncePoliteness {
    /// Wait until the user is idle before announcing.
    ///
    /// Use for non-urgent status updates: "Document saved.", "3 results found.",
    /// "Sync complete." Polite announcements do not interrupt speech in progress.
    /// Maps to ARIA aria-live="polite", AT-SPI2 ATSPI_LIVE_POLITE.
    Polite,

    /// Interrupt current speech and announce immediately.
    ///
    /// Use only for urgent information: error messages that require immediate
    /// attention, session timeout warnings, security alerts. Assertive
    /// announcements are disruptive. Overuse trains users to ignore them.
    /// Maps to ARIA aria-live="assertive", AT-SPI2 ATSPI_LIVE_ASSERTIVE.
    Assertive,
}
```

---

## 5. Platform Adapter Trait

All platform-specific accessibility implementations implement `A11yAdapter`. The
application code interacts with the `A11yTree` directly; the adapter is an
implementation detail wired up during application startup.

```ferrum
/// Platform accessibility bridge.
///
/// Each platform has one implementation:
///   Linux:   AtSpi2Adapter  (D-Bus, AT-SPI2)
///   macOS:   MacA11yAdapter (NSAccessibility, Objective-C runtime)
///   Windows: WinUiaAdapter  (UI Automation, COM)
///
/// An A11yAdapter is a long-lived object for the lifetime of the application.
/// Connect once at startup, publish the initial tree, then call sync_update
/// after each A11yTree mutation.
pub trait A11yAdapter {
    /// Connect to the platform accessibility infrastructure.
    ///
    /// On Linux, this acquires the AT-SPI2 D-Bus connection.
    /// On macOS, this is a no-op (NSAccessibility is always available).
    /// On Windows, this registers with the UIAutomation COM server.
    fn connect(): Result[Self, A11yError] ! IO

    /// Publish the full tree for the first time.
    ///
    /// Call once after connect() and after the initial A11yTree is fully
    /// populated. Subsequent updates go through sync_update.
    fn publish_tree(tree: &A11yTree): Result[(), A11yError] ! IO

    /// Publish a delta update.
    ///
    /// Called after each A11yTree mutation cycle with the result of
    /// A11yTree::take_diff(). The adapter maps each diff entry to the
    /// platform's event and object model.
    fn sync_update(changes: &A11yTreeDiff): Result[(), A11yError] ! IO

    /// Notify the platform of a focus change.
    ///
    /// The adapter fires the platform focus event for the given node ID.
    /// None clears focus. Called by A11yTree::set_focus() via the adapter
    /// pipeline; applications should not call this directly.
    fn set_focus(id: Option[A11yNodeId]): Result[(), A11yError] ! IO

    /// Push a live announcement to the screen reader.
    ///
    /// Called by A11yTree::announce() via the adapter pipeline.
    fn announce(text: &str, politeness: AnnouncePoliteness): Result[(), A11yError] ! IO
}
```

---

## 6. AT-SPI2 Adapter — Linux

### Overview

AT-SPI2 exposes application accessibility trees as D-Bus objects on the
accessibility bus (`org.a11y.Bus`). The accessibility bus is a separate D-Bus
socket from the session bus, managed by `at-spi-bus-launcher`, a daemon started
automatically by systemd or the desktop session. When no AT is running, the bus
may not be present — `AtSpi2Adapter::connect()` returns `A11yError::NotAvailable`
in that case, and the application should treat accessibility as disabled.

The bridge pattern: each `A11yNodeDesc` in the tree is exposed as a D-Bus object
at a well-known path under the application's bus name. The object implements a
set of AT-SPI2 D-Bus interfaces depending on the node's role. Screen readers
like Orca enumerate these objects by traversing the tree from the root, then
subscribe to events via D-Bus signal matching.

```ferrum
use extlib::ccsp::a11y::atspi::{AtSpi2Adapter, AtSpi2Error}
use extlib::ccsp::a11y::{A11yAdapter, A11yError, A11yNodeId, AnnouncePoliteness}

pub type AtSpi2Adapter
```

```ferrum
impl AtSpi2Adapter {
    /// Connect to the AT-SPI2 accessibility bus.
    ///
    /// Steps:
    ///   1. Contact org.a11y.Status on the session bus to check whether
    ///      IsEnabled is true. If accessibility is globally disabled, returns
    ///      A11yError::NotAvailable without attempting a connection.
    ///   2. Call org.a11y.Bus::GetAddress to obtain the accessibility bus socket.
    ///   3. Open a D-Bus connection to that socket.
    ///   4. Register the application on org.a11y.atspi.Socket::Embed.
    ///
    /// This is the only operation that touches D-Bus infrastructure.
    /// All subsequent operations use the established connection.
    pub fn connect(): Result[AtSpi2Adapter, A11yError] ! IO

    /// Check whether a screen reader is currently active.
    ///
    /// Queries org.a11y.Status::ScreenReaderEnabled on the session bus.
    /// Use this to skip expensive accessibility tree population when no AT
    /// is listening. The check is cheap (one D-Bus property read).
    ///
    /// Do not use this as a condition for correctness: the tree must be kept
    /// up-to-date regardless, because a screen reader may start at any time.
    /// Use it only to gate expensive pre-computation that would be wasted
    /// when no AT is active.
    pub fn is_screen_reader_active(&self): bool ! IO
}

impl A11yAdapter for AtSpi2Adapter {
    pub fn connect(): Result[Self, A11yError] ! IO
    pub fn publish_tree(tree: &A11yTree): Result[(), A11yError] ! IO
    pub fn sync_update(changes: &A11yTreeDiff): Result[(), A11yError] ! IO
    pub fn set_focus(id: Option[A11yNodeId]): Result[(), A11yError] ! IO
    pub fn announce(text: &str, politeness: AnnouncePoliteness): Result[(), A11yError] ! IO
}
```

### D-Bus Interface Mapping

Each node is exposed as a D-Bus object implementing a set of AT-SPI2 interfaces
selected by role:

| A11yRole                         | AT-SPI2 interfaces implemented                                      |
|----------------------------------|---------------------------------------------------------------------|
| All roles                        | `org.a11y.atspi.Accessible`, `org.a11y.atspi.Component`             |
| Button, MenuItem, Link           | + `org.a11y.atspi.Action`                                           |
| TextInput, PasswordInput, SearchBox | + `org.a11y.atspi.Text`, `org.a11y.atspi.EditableText`           |
| Slider, ProgressBar, ScrollBar   | + `org.a11y.atspi.Value`                                            |
| Table                            | + `org.a11y.atspi.Table`                                            |
| TableCell                        | + `org.a11y.atspi.TableCell`                                        |
| Image, Figure, Canvas            | + `org.a11y.atspi.Image`                                            |
| Window, Dialog                   | + `org.a11y.atspi.Window`                                           |

`Accessible` provides the name, description, role, state set, parent, and child
list. `Component` provides the bounding rectangle and layer. Role constants map
from `A11yRole` to `ATSPI_ROLE_*` integer values defined in the AT-SPI2 specification.

### Events

State changes, property changes, focus changes, and live announcements are
published as D-Bus signals:

| Trigger                             | AT-SPI2 signal                                              |
|-------------------------------------|-------------------------------------------------------------|
| Node state change (checked, etc.)   | `org.a11y.atspi.Event.Object:StateChanged`                  |
| Label or description changed        | `org.a11y.atspi.Event.Object:PropertyChange:accessible-name` |
| Focus moved to a node               | `org.a11y.atspi.Event.Object:StateChanged:focused`          |
| Live region content changed         | `org.a11y.atspi.Event.Object:Announcement` (AT-SPI2 2.46+) |
| Node added to tree                  | `org.a11y.atspi.Event.Object:ChildrenChanged:add`           |
| Node removed from tree              | `org.a11y.atspi.Event.Object:ChildrenChanged:remove`        |

### X11 and Wayland Compatibility

No window-system-specific code appears in `AtSpi2Adapter`. The adapter communicates
exclusively over the D-Bus accessibility bus socket. Whether the application's window
surface is backed by a `wl_surface` (Wayland) or an `xcb_window_t` (X11) is
irrelevant to the AT-SPI2 layer. This is the defining advantage of AT-SPI2's
IPC-based design: display server transitions do not affect accessibility.

---

## 7. NSAccessibility Adapter — macOS

### Overview

macOS accessibility is built into the AppKit and Cocoa object model via the
`NSAccessibilityProtocol` Objective-C protocol. Every `NSView` and `NSWindow`
can implement accessibility methods directly. The Ferrum adapter bridges from
`A11yNodeDesc` to the Objective-C accessibility protocol without requiring
AppKit dependency: it uses the Objective-C runtime FFI (`objc` crate) to
register protocol methods and post notifications on objects that are not
`NSView` subclasses.

On macOS, no connection step is needed. The OS reads accessibility information
from the running process on demand; there is no accessibility daemon or bus.

```ferrum
use extlib::ccsp::a11y::mac::{MacA11yAdapter}
use extlib::ccsp::a11y::{A11yAdapter, A11yError, A11yNodeId, AnnouncePoliteness}

pub type MacA11yAdapter
```

```ferrum
impl MacA11yAdapter {
    /// Create the macOS adapter.
    ///
    /// No connection is established; macOS accessibility is process-internal.
    /// Returns Err only if the Objective-C runtime is unavailable, which
    /// should not occur on any supported macOS version.
    pub fn connect(): Result[MacA11yAdapter, A11yError]
}

impl A11yAdapter for MacA11yAdapter {
    pub fn connect(): Result[Self, A11yError]
    pub fn publish_tree(tree: &A11yTree): Result[(), A11yError]
    pub fn sync_update(changes: &A11yTreeDiff): Result[(), A11yError]
    pub fn set_focus(id: Option[A11yNodeId]): Result[(), A11yError]
    pub fn announce(text: &str, politeness: AnnouncePoliteness): Result[(), A11yError]
}
```

### Role Mapping

`A11yRole` values map to `NSAccessibilityRole` string constants:

| A11yRole          | NSAccessibilityRole constant                     |
|-------------------|--------------------------------------------------|
| Button            | `NSAccessibilityButtonRole`                      |
| CheckBox          | `NSAccessibilityCheckBoxRole`                    |
| RadioButton       | `NSAccessibilityRadioButtonRole`                 |
| ComboBox          | `NSAccessibilityComboBoxRole`                    |
| TextInput         | `NSAccessibilityTextFieldRole`                   |
| PasswordInput     | `NSAccessibilitySecureTextFieldRole`             |
| Slider            | `NSAccessibilitySliderRole`                      |
| ProgressBar       | `NSAccessibilityProgressIndicatorRole`           |
| Table             | `NSAccessibilityTableRole`                       |
| Heading { .. }    | `NSAccessibilityGroupRole` + `AXHeadingLevel` attribute |
| Link              | `NSAccessibilityLinkRole`                        |
| Image             | `NSAccessibilityImageRole`                       |
| Window            | `NSAccessibilityWindowRole`                      |
| Dialog            | `NSAccessibilitySheetRole` or `AXDialog`         |

### Notifications

Accessibility state changes are published via `NSAccessibilityPostNotification`:

| Trigger                       | Notification constant                                         |
|-------------------------------|---------------------------------------------------------------|
| Focus moved to a node         | `NSAccessibilityFocusedUIElementChangedNotification`          |
| Value changed (Slider, etc.)  | `NSAccessibilityValueChangedNotification`                     |
| Node added to subtree         | `NSAccessibilityUIElementDestroyedNotification` (on removal)  |
| Title/label changed           | `NSAccessibilityTitleChangedNotification`                     |

### VoiceOver Announcements

Live announcements use `NSAccessibilitySpeakAnnotation` with the politeness level
mapped to `NSAccessibilityPriorityKey`:

```ferrum
// Internal mapping used by MacA11yAdapter::announce()
fn politeness_to_priority(p: AnnouncePoliteness): i64 {
    match p {
        AnnouncePoliteness::Polite     => NSAccessibilityPriorityMedium,  // 50
        AnnouncePoliteness::Assertive  => NSAccessibilityPriorityHigh,    // 90
    }
}
```

---

## 8. UI Automation Adapter — Windows

### Overview

Windows UI Automation (UIA) is the current Windows accessibility API, introduced
in Windows Vista and mandatory for Windows Store applications since Windows 8. It
supersedes MSAA (Microsoft Active Accessibility). All major Windows screen readers
— Narrator (built in), JAWS (Freedom Scientific), and NVDA (open source) — support
UIA natively. JAWS and NVDA also support the older MSAA/IAccessible interface via a
COM IDispatch bridge; the Ferrum adapter exposes this bridge automatically.

The adapter registers the application as a UIA provider by implementing the COM
interfaces `IRawElementProviderSimple`, `IRawElementProviderFragment`, and
`IRawElementProviderFragmentRoot`. Each `A11yNodeDesc` maps to a COM object
implementing these interfaces. The `windows-rs` crate provides the safe Ferrum
bindings to the Windows COM infrastructure.

```ferrum
use extlib::ccsp::a11y::win::{WinUiaAdapter}
use extlib::ccsp::a11y::{A11yAdapter, A11yError, A11yNodeId, AnnouncePoliteness}

pub type WinUiaAdapter
```

```ferrum
impl WinUiaAdapter {
    /// Register with the UIAutomation COM server.
    ///
    /// Steps:
    ///   1. CoInitializeEx for the calling thread (STA model).
    ///   2. Register the root provider via UiaRegisterProviderCallback.
    ///   3. Store the connection for use in subsequent sync_update calls.
    ///
    /// Must be called from the UI thread. UI Automation COM objects are
    /// single-threaded apartment (STA) objects and must not be used across
    /// thread boundaries without marshalling.
    pub fn connect(): Result[WinUiaAdapter, A11yError] ! IO
}

impl A11yAdapter for WinUiaAdapter {
    pub fn connect(): Result[Self, A11yError] ! IO
    pub fn publish_tree(tree: &A11yTree): Result[(), A11yError] ! IO
    pub fn sync_update(changes: &A11yTreeDiff): Result[(), A11yError] ! IO
    pub fn set_focus(id: Option[A11yNodeId]): Result[(), A11yError] ! IO
    pub fn announce(text: &str, politeness: AnnouncePoliteness): Result[(), A11yError] ! IO
}
```

### Control Type Mapping

`A11yRole` values map to UIA Control Type IDs:

| A11yRole        | UIA Control Type constant          |
|-----------------|------------------------------------|
| Button          | `UIA_ButtonControlTypeId` (50000)  |
| CheckBox        | `UIA_CheckBoxControlTypeId` (50002)|
| RadioButton     | `UIA_RadioButtonControlTypeId` (50015) |
| ComboBox        | `UIA_ComboBoxControlTypeId` (50003)|
| ListBox         | `UIA_ListControlTypeId` (50008)    |
| ListItem        | `UIA_ListItemControlTypeId` (50007)|
| TextInput       | `UIA_EditControlTypeId` (50004)    |
| Slider          | `UIA_SliderControlTypeId` (50016)  |
| ProgressBar     | `UIA_ProgressBarControlTypeId` (50014) |
| Table           | `UIA_TableControlTypeId` (50018)   |
| TableCell       | `UIA_DataItemControlTypeId` (50029)|
| Heading { .. }  | `UIA_TextControlTypeId` (50020) + `HeadingLevel` property |
| Image           | `UIA_ImageControlTypeId` (50006)   |
| Window          | `UIA_WindowControlTypeId` (50032)  |
| Dialog          | `UIA_WindowControlTypeId` (50032)  |

### COM Interfaces Implemented per Node

Each node COM object implements the following interfaces:

- `IRawElementProviderSimple` — core properties: name, control type, help text
- `IRawElementProviderFragment` — tree navigation: parent, children, next/prev sibling
- `IRawElementProviderFragmentRoot` — root provider for the application window

Additional pattern interfaces are implemented based on role:

| A11yRole                             | UIA pattern interface(s)                                       |
|--------------------------------------|----------------------------------------------------------------|
| Button, MenuItem, Link               | `IInvokeProvider`                                              |
| CheckBox, RadioButton                | `IToggleProvider`                                              |
| ComboBox                             | `IExpandCollapseProvider`, `ISelectionProvider`                |
| ListBox, TreeView                    | `ISelectionProvider`                                           |
| ListItem, TreeItem                   | `ISelectionItemProvider`                                       |
| TextInput, PasswordInput, SearchBox  | `IValueProvider`, `ITextProvider`                              |
| Slider                               | `IRangeValueProvider`                                          |
| ScrollBar, ScrollPane                | `IScrollProvider`                                              |
| Table                                | `ITableProvider`, `IGridProvider`                              |
| TableCell                            | `ITableItemProvider`, `IGridItemProvider`                      |

### Events

Focus and property changes are raised via `UiaRaiseAutomationEvent` and
`UiaRaisePropertyChangedEvent`:

| Trigger                        | UIA event                                                       |
|--------------------------------|-----------------------------------------------------------------|
| Focus moved to a node          | `UIA_AutomationFocusChangedEventId`                             |
| Node activated (button click)  | `UIA_Invoke_InvokedEventId`                                     |
| Value changed                  | `UIA_ValueValuePropertyId` property change event               |
| State changed (checked, etc.)  | `UIA_ToggleToggleStatePropertyId` or equivalent property change |
| Node added                     | `UIA_StructureChangedEventId` (ChildAdded)                      |
| Node removed                   | `UIA_StructureChangedEventId` (ChildRemoved)                    |

Live announcements use `UiaRaiseNotificationEvent` (Windows 10 RS3+) with
`NotificationKind_ActionCompleted` for Polite and `NotificationKind_Other` for
Assertive.

---

## 9. A11yCtx — Widget Integration

`A11yCtx` is the context passed to `Widget::accessibility()`. It provides a
builder interface for registering nodes into the `A11yTree` without exposing
the tree directly. The widget system calls `accessibility()` after layout, when
widget bounds are known. Widgets declare their accessibility description using
`A11yCtx` and receive back an `A11yNodeId` they can use to register child nodes.

```ferrum
use extlib::ccsp::a11y::{A11yCtx, A11yNodeBuilder, A11yNodeId, A11yRole, A11yState, A11yAction}
use extlib::ccsp::draw::Rect

/// Context provided to Widget::accessibility().
///
/// Widgets use this to register their accessibility node(s) in the tree.
/// A11yCtx is a borrowed view into the A11yTree; nodes registered here
/// are added to the tree automatically.
pub type A11yCtx
```

```ferrum
impl A11yCtx {
    /// Begin building an accessibility node with the given role and label.
    ///
    /// role:  Semantic role. See A11yRole.
    /// label: Short accessible name. See A11yNodeDesc::label.
    ///
    /// Returns a builder. Call .build() to register the node and receive
    /// its A11yNodeId.
    pub fn node(mut self, role: A11yRole, label: &str): A11yNodeBuilder

    /// Retrieve the A11yNodeId for a node previously registered by this widget.
    ///
    /// Used when a widget needs to update its node's state without rebuilding
    /// the full descriptor. Returns None if no node with the given role has
    /// been registered by this widget in the current tree.
    pub fn existing_id(&self, role: &A11yRole): Option[A11yNodeId]
}
```

```ferrum
/// Builder for a single accessibility node.
///
/// Methods may be called in any order. Call .build() exactly once to register
/// the node.
pub type A11yNodeBuilder
```

```ferrum
impl A11yNodeBuilder {
    /// Set the longer description (supplementary information).
    /// See A11yNodeDesc::description.
    pub fn description(mut self, text: &str): Self

    /// Set the current value as a string.
    /// See A11yNodeDesc::value.
    pub fn value(mut self, text: &str): Self

    /// Set the full dynamic state.
    /// Overrides the defaults (all false / None) for this node.
    pub fn state(mut self, state: A11yState): Self

    /// Declare an action and its callback.
    ///
    /// action:   The action the AT may invoke.
    /// callback: Called when the platform invokes this action. The callback
    ///           receives no arguments; it should post a widget event to the
    ///           application's event queue.
    pub fn action[F: Fn() ! IO](mut self, action: A11yAction, callback: F): Self

    /// Override the bounding rectangle.
    ///
    /// By default, the widget's layout bounds are used. Use this only when
    /// the accessible region differs from the widget bounds (e.g., a custom
    /// hit area, or a widget that contains multiple accessible elements with
    /// different bounds).
    pub fn bounds_override(mut self, bounds: Rect): Self

    /// Register the node in the A11yTree and return its stable ID.
    ///
    /// After this call, the node exists in the tree and will be included in
    /// the next diff published to the platform adapter.
    ///
    /// The returned A11yNodeId may be passed to A11yCtx methods that accept
    /// a parent ID, or stored by the widget for use in set_focus and
    /// announce calls.
    pub fn build(self): A11yNodeId
}
```

### Widget Integration Point

The `Widget` trait in `extlib::ccsp::widget` includes:

```ferrum
pub trait Widget {
    fn layout(ctx: &mut LayoutCtx)
    fn paint(ctx: &mut PaintCtx)

    /// Build or update this widget's accessibility node(s).
    ///
    /// Called by the widget system after layout, when bounds are known.
    /// Widgets must register exactly one primary node via ctx.node(...).build()
    /// and may register additional child nodes for compound widgets.
    ///
    /// The widget system calls this method:
    ///   - Once when the widget is first added to the tree.
    ///   - Again whenever the widget's visual state changes (checked, disabled,
    ///     value update, label change).
    ///   - Again after every layout pass that changes the widget's bounds.
    fn accessibility(ctx: &mut A11yCtx)
}
```

---

## 10. Live Regions

A live region is a node whose content changes dynamically without user interaction.
The screen reader monitors live regions and announces changes automatically when
the content changes, without requiring the user to navigate to the element.

### When to Use Live Regions

| Use case                          | Politeness  |
|-----------------------------------|-------------|
| Status bar messages               | Polite      |
| Search result counts              | Polite      |
| Form validation errors            | Assertive   |
| Loading / progress messages       | Polite      |
| Chat message received             | Polite      |
| Session timeout warning           | Assertive   |
| Success confirmation ("Saved.")   | Polite      |

### API

```ferrum
// Mark a node as a live region.
// Subsequent changes to the node's label or value field will be announced
// automatically by the screen reader at the given politeness level.
//
// The node must already exist in the tree. Call this once after the node is
// first added; the live region status persists for the node's lifetime.
pub fn A11yTree::set_live_region(mut self, id: A11yNodeId, politeness: AnnouncePoliteness)
```

When a live region node's `label` or `value` changes via `update_node`, the
adapter fires the appropriate platform live region event:

- AT-SPI2: `org.a11y.atspi.Event.Object:Announcement` (AT-SPI2 2.46+); falls
  back to `TextChanged` on older AT-SPI2 versions.
- macOS: `NSAccessibilityAnnouncementRequestedNotification` with
  `NSAccessibilityAnnouncementKey` set to the new value.
- Windows: `UiaRaiseNotificationEvent` with `NotificationProcessing_MostRecent`
  for Polite and `NotificationProcessing_CurrentThenMostRecent` for Assertive.

### Important: Do Not Overuse Live Regions

Every live region announcement interrupts or queues behind current speech. An
application that marks every dynamic element as a live region produces a noisy
experience that is worse than no accessibility at all. Reserve live regions for
content the user genuinely needs to hear as soon as it changes. If the user can
navigate to the element to read it when they choose, do not mark it as a live
region.

---

## 11. Keyboard Focus Management

```ferrum
use extlib::ccsp::a11y::{A11yFocusManager, A11yNodeId, A11yTree}

/// Tracks keyboard focus and manages tab navigation order.
///
/// The widget system maintains one A11yFocusManager per window. The focus
/// manager is the authoritative source of which node holds focus; it calls
/// A11yTree::set_focus internally when focus moves.
pub type A11yFocusManager
```

```ferrum
impl A11yFocusManager {
    pub fn new(tree: &mut A11yTree): A11yFocusManager

    /// Override the tab navigation order.
    ///
    /// By default, tab order follows document order (the order nodes appear
    /// in the A11yTree children lists, depth-first). Call this to establish
    /// a custom order, e.g. for complex form layouts where visual left-to-right
    /// order differs from tree order.
    ///
    /// ids: Complete ordered list of focusable node IDs. Nodes not in this
    /// list are skipped by Tab navigation but may still receive focus via
    /// direct click or programmatic set_focus.
    pub fn set_focus_order(mut self, ids: Vec[A11yNodeId])

    /// Move focus to the next focusable node (Tab key).
    ///
    /// Wraps around at the end of the focus order.
    /// Updates A11yTree focus and fires the platform focus event.
    pub fn next_focus(mut self): Option[A11yNodeId] ! IO

    /// Move focus to the previous focusable node (Shift-Tab key).
    ///
    /// Wraps around at the beginning of the focus order.
    pub fn prev_focus(mut self): Option[A11yNodeId] ! IO

    /// Move focus to the given node directly.
    ///
    /// Returns Err if id does not exist in the tree.
    pub fn set_focus(mut self, id: A11yNodeId): Result[(), A11yError] ! IO

    /// Restrict Tab navigation to descendants of root_id.
    ///
    /// Use for modal dialogs: while a trap is active, Tab and Shift-Tab cycle
    /// only within the dialog's subtree. Focus cannot escape to the underlying
    /// window content.
    ///
    /// Call release_focus_trap() when the dialog is dismissed.
    ///
    /// Traps may be nested: each call to trap_focus pushes a new trap; each
    /// call to release_focus_trap pops the most recent one.
    pub fn trap_focus(mut self, root_id: A11yNodeId)

    /// Release the most recently installed focus trap.
    ///
    /// Does nothing if no trap is active.
    pub fn release_focus_trap(mut self)

    /// Returns the ID of the currently focused node, or None if no node is focused.
    pub fn focused(&self): Option[A11yNodeId]
}
```

---

## 12. Error Types

```ferrum
/// Errors from accessibility adapter operations.
@derive(Debug)]
pub enum A11yError {
    /// The accessibility bus is not available.
    ///
    /// On Linux: the AT-SPI2 bus is not running, or accessibility is disabled
    /// in the desktop session settings.
    /// On Windows: UIAutomation COM registration failed.
    /// Treatment: disable accessibility silently and continue. Log at info level
    /// if logging is available. Do not surface to the user.
    NotAvailable,

    /// A D-Bus error occurred (Linux AT-SPI2 adapter only).
    ///
    /// Contains the D-Bus error name and message. Typically indicates a
    /// transient bus disconnect; the adapter should attempt reconnection on
    /// the next publish_tree or sync_update call.
    DbusError(String),

    /// The requested adapter is not supported on the current platform.
    ///
    /// Returned when, e.g., AtSpi2Adapter::connect() is called on Windows.
    /// Use the platform-conditional adapter selection pattern shown in the
    /// examples rather than calling platform-specific adapters directly.
    AdapterNotSupported,

    /// The given A11yNodeId does not exist in the tree.
    ///
    /// Returned by A11yFocusManager::set_focus and A11yTree operations
    /// that reference a node by ID. Indicates a programming error: the
    /// caller is referencing a node that was removed or never added.
    NodeNotFound(A11yNodeId),

    /// A platform-specific error not covered by other variants.
    ///
    /// Contains a human-readable description. Returned for NSAccessibility
    /// failures on macOS and COM HRESULT errors on Windows.
    PlatformError(String),
}
```

---

## 13. Example Usage

### Annotating a Button Widget

```ferrum
use extlib::ccsp::a11y::{A11yCtx, A11yRole, A11yState, A11yAction}
use extlib::ccsp::widget::Widget

pub type SaveButton {
    label:    String,
    enabled:  bool,
}

impl Widget for SaveButton {
    fn layout(ctx: &mut LayoutCtx) {
        // ... layout logic ...
    }

    fn paint(ctx: &mut PaintCtx) {
        // ... drawing logic ...
    }

    fn accessibility(ctx: &mut A11yCtx) {
        ctx.node(A11yRole::Button, &self.label)
            .state(A11yState {
                enabled:  self.enabled,
                focused:  false,  // managed by A11yFocusManager
                ..A11yState::default()
            })
            .action(A11yAction::Click, || {
                // Post a save event to the application queue.
                self.on_save()
            })
            .build()
    }
}
```

### Marking an Error Message as a Live Region

```ferrum
use extlib::ccsp::a11y::{A11yCtx, A11yRole, A11yState, A11yTree, A11yNodeId, AnnouncePoliteness}

pub type FormErrorMessage {
    message:    Option[String],
    node_id:    Option[A11yNodeId],
}

impl FormErrorMessage {
    // Called once during widget initialization.
    pub fn init_accessibility(mut self, ctx: &mut A11yCtx, tree: &mut A11yTree) {
        let id = ctx.node(A11yRole::Alert, "Form error")
            .state(A11yState {
                hidden: self.message.is_none(),
                ..A11yState::default()
            })
            .build()
        tree.set_live_region(id, AnnouncePoliteness::Assertive)
        self.node_id = Some(id)
    }

    // Called when a validation error occurs or clears.
    pub fn set_error(mut self, message: Option[String], tree: &mut A11yTree) {
        self.message = message
        if let Some(id) = self.node_id {
            tree.update_node(A11yNodeDesc {
                id,
                role:  A11yRole::Alert,
                label: self.message.clone(),
                state: A11yState {
                    hidden: self.message.is_none(),
                    ..A11yState::default()
                },
                ..A11yNodeDesc::default_for(id)
            })
        }
    }
}
```

### Modal Dialog Focus Trap

```ferrum
use extlib::ccsp::a11y::{A11yCtx, A11yRole, A11yFocusManager, A11yNodeId}

pub type ConfirmDialog {
    root_id:     Option[A11yNodeId],
    ok_id:       Option[A11yNodeId],
    cancel_id:   Option[A11yNodeId],
}

impl ConfirmDialog {
    pub fn open(mut self, ctx: &mut A11yCtx, focus_manager: &mut A11yFocusManager) ! IO {
        let root = ctx.node(A11yRole::Dialog, "Confirm action")
            .description("Are you sure you want to delete this item?")
            .build()
        let ok = ctx.node(A11yRole::Button, "Delete")
            .action(A11yAction::Click, || self.on_confirm())
            .build()
        let cancel = ctx.node(A11yRole::Button, "Cancel")
            .action(A11yAction::Click, || self.on_cancel())
            .build()

        self.root_id   = Some(root)
        self.ok_id     = Some(ok)
        self.cancel_id = Some(cancel)

        // Trap focus within the dialog.
        focus_manager.trap_focus(root)
        focus_manager.set_focus_order(vec![ok, cancel])
        // Move initial focus to the less destructive action.
        focus_manager.set_focus(cancel).unwrap_or(())
    }

    pub fn close(mut self, focus_manager: &mut A11yFocusManager) {
        focus_manager.release_focus_trap()
        // Restore focus to the element that opened the dialog.
    }
}
```

### Checking Screen Reader Presence Before Expensive Work

```ferrum
use extlib::ccsp::a11y::atspi::AtSpi2Adapter
use extlib::ccsp::a11y::A11yAdapter

fn init_accessibility(adapter: &AtSpi2Adapter) ! IO {
    // Checking is cheap; skip expensive pre-computation only.
    // The tree must be maintained regardless of this check.
    if !adapter.is_screen_reader_active() {
        // Skip pre-computing extended descriptions for 500 table cells.
        // They will be computed on demand if a screen reader starts later.
        return
    }
    precompute_table_descriptions()
}
```

### Platform-Conditional Adapter Selection

```ferrum
use extlib::ccsp::a11y::{A11yAdapter, A11yError, A11yTree}

#[cfg(target_os = "linux")]
use extlib::ccsp::a11y::atspi::AtSpi2Adapter as PlatformAdapter

#[cfg(target_os = "macos")]
use extlib::ccsp::a11y::mac::MacA11yAdapter as PlatformAdapter

#[cfg(target_os = "windows")]
use extlib::ccsp::a11y::win::WinUiaAdapter as PlatformAdapter

pub fn start_accessibility(tree: &A11yTree): Result[(), A11yError] ! IO {
    let adapter = PlatformAdapter::connect()?
    adapter.publish_tree(tree)?
    Ok(())
}
```

---

## 14. Dependencies Reference

| Dependency                    | Purpose                                              | Condition           |
|-------------------------------|------------------------------------------------------|---------------------|
| `extlib::ccsp::draw`          | `Rect` type for node bounding boxes                  | All platforms       |
| `extlib::ccsp::widget`        | `A11yCtx` integration point in the Widget trait      | All platforms       |
| `stdlib::alloc`               | `String`, `Vec`                                      | All platforms       |
| `stdlib::sys`                 | Platform detection, low-level OS interfaces          | All platforms       |
| `dbus-rs`                     | D-Bus connection for AT-SPI2 accessibility bus       | Linux only          |
| `objc`                        | Objective-C runtime FFI for NSAccessibility methods  | macOS only          |
| `windows-rs`                  | COM bindings for UI Automation interfaces            | Windows only        |

Platform-conditional compilation uses `#[cfg(target_os = "...")]` attributes on
the adapter modules. The `extlib::ccsp::a11y` module re-exports only the
platform-appropriate adapter under the `PlatformAdapter` alias, so application
code that uses the alias is portable without further `cfg` guards.
