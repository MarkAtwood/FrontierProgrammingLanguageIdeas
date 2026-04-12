# Ferrum Extended Library — widget_suite: Visual Style Protocol

**Module path:** `extlib.ccsp.widget_suite`
**Spec basis:** CCSP `lib_ccsp_widget_suite` — semantic/visual separation protocol
**Roadmap status:** Post-1.0 (designed now, implemented after widget system stabilizes)
**Dependencies:** `extlib.ccsp.widget` (Widget, Constraints, WidgetTree), `extlib.ccsp.draw`
(DrawContext, Color, Rect, Point, EdgeInsets), `extlib.ccsp.style` (Theme, TextStyle),
`extlib.ccsp.text_layout` (TextMeasure), `extlib.ccsp.a11y` (A11yRole)

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [The Layout Algorithm](#2-the-layout-algorithm)
3. [InteractiveState](#3-interactivestate)
4. [ButtonStyle](#4-buttonstyle)
5. [TextFieldStyle](#5-textfieldstyle)
6. [CheckboxStyle and RadioStyle](#6-checkboxstyle-and-radiostyle)
7. [ToggleStyle](#7-togglestyle)
8. [SliderStyle](#8-sliderstyle)
9. [ProgressStyle](#9-progressstyle)
10. [ScrollbarStyle](#10-scrollbarstyle)
11. [MenuStyle and MenuItemStyle](#11-menustyle-and-menuitemstyle)
12. [DialogStyle](#12-dialogstyle)
13. [TooltipStyle](#13-tooltipstyle)
14. [ChipStyle](#14-chipstyle)
15. [DividerStyle](#15-dividerstyle)
16. [TabStyle](#16-tabstyle)
17. [ListItemStyle](#17-listitemstyle)
18. [WidgetSuite — the Umbrella Trait](#18-widgetsuite--the-umbrella-trait)
19. [BuildCtx Integration](#19-buildctx-integration)
20. [Subtree Overrides with provide_suite](#20-subtree-overrides-with-provide_suite)
21. [Writing a Custom Suite](#21-writing-a-custom-suite)
22. [Error Types](#22-error-types)
23. [Dependencies Reference](#23-dependencies-reference)

---

## 1. Overview and Rationale

### The Core Separation

A button is a semantic region that accepts a tap, responds to keyboard Enter, announces
itself to screen readers, and fires an `on_press` callback. How it looks — color, border,
radius, shadow, press animation — is orthogonal to all of that.

This module enforces that separation structurally. Every built-in widget type defines a
`*Style` trait that describes the visual contract for that widget type. The widget
implementation contains layout logic and event logic. It contains no rendering code.
The `*Style` implementation does the rendering.

Consequences:

- Dropping in a different suite changes the appearance of every widget in an application
  without touching any widget code.
- Writing a new widget that fits into a suite requires implementing `*Style`, not
  subclassing a base widget class.
- The boring default (see `extlib.ccsp.boring`) and a Material Design suite (or Fluent,
  or macOS HIG, or a game UI skin) are interchangeable at the call site.
- Widget code is testable independently of rendering: layout tests never touch a
  `DrawContext`.

### What the `*Style` traits do NOT control

`*Style` traits control painting only. They do not:

- Define widget structure or widget composition
- Override layout (except through the sizing hints they provide — `preferred_height`,
  `padding`, `min_size` — which the widget uses as inputs to its layout pass)
- Handle events
- Produce accessibility nodes

Those responsibilities belong to the widget implementation, which delegates to
`extlib.ccsp.widget` and `extlib.ccsp.a11y`. The style layer is purely visual.

---

## 2. The Layout Algorithm

Every built-in widget in `extlib.ccsp.widget` uses the **same layout algorithm that
Flutter's RenderBox system uses**. This is documented here because the `*Style` sizing
hints feed directly into it.

### The three rules

**Rule 1 — Constraints flow down.**
When a parent lays out a child, it calls:

```ferrum
let child_size = child.layout(constraints, ctx)
```

where `constraints: Constraints` is a bounding box of allowed sizes:

```ferrum
@derive(Debug, Clone, Copy, PartialEq)
pub type Constraints {
    pub min_width:  f32,
    pub max_width:  f32,   // f32::INFINITY means unconstrained
    pub min_height: f32,
    pub max_height: f32,
}
```

The child must return a `Size` where
`min_width <= size.width <= max_width` and
`min_height <= size.height <= max_height`.
Violating this contract is a debug assertion failure.

**Rule 2 — Sizes flow up.**
`child.layout(constraints, ctx)` returns `Size`. The parent uses the returned size to
position the child and compute its own size. The child does not decide its own position.

**Rule 3 — Parents set positions.**
A child widget does not know its own position on screen. The parent stores each child's
offset and passes it at paint time:

```ferrum
child.paint(ctx, offset)
```

This one-way data flow makes layout O(n) in the number of widgets. Each widget's
`layout` method is called **at most once per frame**. There are no re-entrant queries,
no bidirectional negotiations, no cycles. This is the property that makes deeply nested
layouts fast.

### How `*Style` sizing hints fit in

A `ButtonStyle` provides:

```ferrum
fn preferred_height(&self, theme: &Theme): f32
fn padding(&self, theme: &Theme): EdgeInsets
fn min_size(&self, theme: &Theme): Size
```

The `Button` widget's `layout` implementation calls these to determine its intrinsic
size, then clamps against the incoming `constraints`:

```ferrum
fn layout(&mut self, constraints: Constraints, ctx: &LayoutCtx): Size {
    let style     = ctx.suite().button()
    let padding   = style.padding(ctx.theme())
    let min_size  = style.min_size(ctx.theme())
    let label_w   = ctx.measure_text(&self.label, &ctx.theme().text_styles.label_large,
                                     constraints.max_width - padding.horizontal()).width
    let w = (label_w + padding.horizontal()).max(min_size.width)
    let h = style.preferred_height(ctx.theme()).max(min_size.height)
    constraints.constrain(Size { width: w, height: h })
}
```

The style tells the widget what it needs; the widget enforces the constraints.

### The LayoutCtx

Every `layout` call receives a `LayoutCtx`:

```ferrum
pub type LayoutCtx

impl LayoutCtx {
    /// The active widget suite for this subtree.
    pub fn suite(&self): &dyn WidgetSuiteErased

    /// The active theme for this subtree.
    pub fn theme(&self): &Theme

    /// Measure text without rendering it.
    pub fn measure_text(
        &self,
        text:      &str,
        style:     &TextStyle,
        max_width: f32,
    ): TextMeasurement

    /// Whether this layout pass is triggered by a resize (vs. first layout).
    pub fn is_resize(&self): bool
}

pub type TextMeasurement {
    pub size:     Size,
    pub baseline: f32,   // distance from top to text baseline
    pub lines:    u32,
}
```

`LayoutCtx` is available only during `layout`. It is not available during `paint`
(paint receives a `DrawContext`), `handle_event`, or `accessibility`. This is the
correct boundary: layout is about size, paint is about pixels, they do not share state.

---

## 3. InteractiveState

Every interactive widget can be in one of these visual states. `*Style` paint methods
receive the current state and choose colors, borders, and indicators accordingly.

```ferrum
/// The visual interaction state of a widget at the time of painting.
///
/// States are mutually exclusive. Focused+Hovered and Focused+Pressed
/// are separate values because focus-ring rendering while hovered is a
/// meaningful visual case (the ring must remain visible).
///
/// Disabled takes precedence over all other states: a disabled widget
/// cannot be hovered, pressed, or focused.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)
pub enum InteractiveState {
    /// Default state. No pointer interaction, no keyboard focus.
    Normal,

    /// Pointer is over the widget. No button pressed.
    Hovered,

    /// Primary pointer button is held down over the widget.
    Pressed,

    /// Widget has keyboard focus. No pointer interaction.
    Focused,

    /// Widget has keyboard focus AND pointer is over it.
    FocusedHovered,

    /// Widget has keyboard focus AND primary pointer button is held.
    FocusedPressed,

    /// Widget is not interactive. Pointer and focus events are ignored.
    Disabled,
}

impl InteractiveState {
    pub fn is_disabled(self): bool { self == Self.Disabled }
    pub fn has_focus(self): bool {
        matches!(self, Self.Focused | Self.FocusedHovered | Self.FocusedPressed)
    }
    pub fn is_pressed(self): bool {
        matches!(self, Self.Pressed | Self.FocusedPressed)
    }
}
```

---

## 4. ButtonStyle

```ferrum
/// Visual style contract for a push button.
///
/// A Button widget calls these methods during layout and paint.
/// The style does not define what happens when the button is pressed;
/// that is the widget's responsibility.
pub trait ButtonStyle: Send + Sync {
    /// Paint the button background, border, and label text.
    ///
    /// `bounds` is the full bounding box of the button in local widget coordinates.
    /// The style is responsible for all visible rendering within this rectangle.
    /// It must not draw outside bounds.
    fn paint(
        &self,
        ctx:    &mut DrawContext,
        bounds: Rect,
        label:  &str,
        state:  InteractiveState,
        theme:  &Theme,
    )

    /// Paint a button that contains an icon and an optional label.
    ///
    /// Default implementation calls paint() with the label; override for
    /// icon-specific treatment (e.g., icon-only buttons with square aspect).
    fn paint_icon_button(
        &self,
        ctx:    &mut DrawContext,
        bounds: Rect,
        icon:   &dyn DrawIcon,
        label:  Option[&str],
        state:  InteractiveState,
        theme:  &Theme,
    ) {
        // default: paint normally; icon rendering delegated to DrawIcon
        if let Some(lbl) = label {
            self.paint(ctx, bounds, lbl, state, theme)
        }
        icon.draw(ctx, bounds.icon_region(), state, theme)
    }

    /// Inset from the outer bounds to the content (label) region.
    fn content_padding(&self, theme: &Theme): EdgeInsets

    /// Minimum size. Layout will not return a size smaller than this.
    fn min_size(&self, theme: &Theme): Size

    /// Preferred height when the widget has unconstrained height.
    fn preferred_height(&self, theme: &Theme): f32
}
```

### ButtonVariant

Buttons come in three semantic variants. A suite may render them identically or
differently; what matters is that the widget declares its intent.

```ferrum
/// The semantic weight of a button.
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ButtonVariant {
    /// The primary action on this surface. One per dialog or major section.
    Primary,

    /// Secondary or alternative actions.
    Secondary,

    /// Low-emphasis action; no background fill.
    Ghost,
}
```

The `Button` widget stores a `ButtonVariant`. The `ButtonStyle::paint` method receives
it indirectly because the widget passes all relevant context. If a suite wants separate
rendering per variant, it inspects the label text's semantic context or the variant
flag carried in the paint call. For variants to be style-transparent, the widget calls:

```ferrum
// variant is stored on the widget; paint receives it as part of the call
fn paint_variant(
    &self,
    ctx:     &mut DrawContext,
    bounds:  Rect,
    label:   &str,
    variant: ButtonVariant,
    state:   InteractiveState,
    theme:   &Theme,
)
```

Suites that do not distinguish variants may implement `paint_variant` by delegating
to `paint` and ignoring the variant.

---

## 5. TextFieldStyle

```ferrum
/// Visual state of a text field beyond the base interactive state.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum TextFieldVisualState {
    /// Normal editing state.
    Normal(InteractiveState),
    /// The field's content has failed validation. Error indicator visible.
    Error(InteractiveState),
    /// Field content is being validated asynchronously. Spinner visible.
    Validating(InteractiveState),
    /// Field content has passed validation. Success indicator visible.
    Valid(InteractiveState),
}

/// A half-open byte range into the content string.
pub type Selection { pub start: usize, pub end: usize }

/// Cursor position as a byte offset into the content string.
pub type CursorPos(usize)

pub trait TextFieldStyle: Send + Sync {
    /// Paint the text field container, content, placeholder, cursor, and selection.
    ///
    /// `content` is the current text. `placeholder` is shown when content is empty.
    /// `cursor` is Some when the field has keyboard focus.
    /// `selection` is Some when text is selected; takes priority over cursor for rendering.
    /// `label` is the field label (shown above or floating inside, per suite).
    /// `helper` is optional helper text shown below the field.
    /// `error_message` is shown instead of helper when state is Error.
    fn paint(
        &self,
        ctx:           &mut DrawContext,
        bounds:        Rect,
        content:       &str,
        placeholder:   &str,
        label:         Option[&str],
        helper:        Option[&str],
        error_message: Option[&str],
        state:         TextFieldVisualState,
        cursor:        Option[CursorPos],
        selection:     Option[Selection],
        theme:         &Theme,
    )

    /// Height of the input area only (not including label above or helper below).
    fn input_height(&self, theme: &Theme): f32

    /// Total preferred height including label and helper regions.
    fn preferred_height(
        &self,
        has_label:  bool,
        has_helper: bool,
        theme:      &Theme,
    ): f32

    /// Padding inside the input area, from the field border to the text.
    fn content_padding(&self, theme: &Theme): EdgeInsets

    /// Rectangle within `bounds` where text content is rendered.
    /// The widget uses this to place the text layout and hit-test the cursor.
    fn text_bounds(&self, bounds: Rect, theme: &Theme): Rect
}
```

---

## 6. CheckboxStyle and RadioStyle

### TriState

```ferrum
/// Three-valued checkbox state.
/// Indeterminate is used when the checkbox represents a group where some
/// but not all members are checked (a "select all" checkbox, for instance).
@derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TriState {
    Checked,
    Unchecked,
    Indeterminate,
}
```

### CheckboxStyle

```ferrum
pub trait CheckboxStyle: Send + Sync {
    /// Paint the checkbox indicator.
    ///
    /// The indicator is always square. `bounds` is a square of size
    /// returned by `indicator_size()`.
    fn paint(
        &self,
        ctx:   &mut DrawContext,
        bounds: Rect,
        value: TriState,
        state: InteractiveState,
        theme: &Theme,
    )

    /// Side length of the checkbox square in logical pixels.
    fn indicator_size(&self, theme: &Theme): f32
}
```

### RadioStyle

```ferrum
pub trait RadioStyle: Send + Sync {
    /// Paint the radio button indicator.
    ///
    /// `selected` is true when this radio button is the active choice in its group.
    /// `bounds` is a square of size returned by `indicator_size()`.
    fn paint(
        &self,
        ctx:      &mut DrawContext,
        bounds:   Rect,
        selected: bool,
        state:    InteractiveState,
        theme:    &Theme,
    )

    /// Side length of the bounding square for the radio indicator.
    fn indicator_size(&self, theme: &Theme): f32
}
```

---

## 7. ToggleStyle

A toggle (on/off switch) is visually distinct from a checkbox even though both are
boolean. A checkbox is for "include this option"; a toggle is for "this system is
active right now."

```ferrum
pub trait ToggleStyle: Send + Sync {
    /// Paint the toggle track and thumb.
    ///
    /// `on` is the logical value; `transition` is the current animation
    /// progress in [0.0, 1.0] from off to on (or 1.0 to 0.0 from on to off).
    /// Suites that use no animation always receive transition == 0.0 (off)
    /// or transition == 1.0 (on); boring suite always passes the logical value
    /// directly with no animation.
    fn paint(
        &self,
        ctx:        &mut DrawContext,
        bounds:     Rect,
        on:         bool,
        transition: f32,
        state:      InteractiveState,
        theme:      &Theme,
    )

    /// Preferred size of the toggle track+thumb assembly.
    fn preferred_size(&self, theme: &Theme): Size
}
```

---

## 8. SliderStyle

```ferrum
pub trait SliderStyle: Send + Sync {
    /// Paint the slider track, fill, and thumb.
    ///
    /// `value` is normalized to [0.0, 1.0].
    /// `thumb_center` is the computed center point of the thumb in local coordinates,
    /// derived from `value` and the track bounds. The style uses this to draw the thumb
    /// at the correct position without recomputing the track geometry.
    fn paint(
        &self,
        ctx:          &mut DrawContext,
        bounds:       Rect,
        value:        f32,
        thumb_center: Point,
        state:        InteractiveState,
        theme:        &Theme,
    )

    /// Height of the track line (not the thumb).
    fn track_height(&self, theme: &Theme): f32

    /// Size of the thumb indicator.
    fn thumb_size(&self, theme: &Theme): Size

    /// Total preferred height of the slider widget, including thumb overhang.
    fn preferred_height(&self, theme: &Theme): f32
}
```

---

## 9. ProgressStyle

```ferrum
/// Whether the progress amount is known.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum ProgressValue {
    /// Known fraction in [0.0, 1.0].
    Determinate(f32),
    /// Amount unknown; show an indeterminate indicator.
    Indeterminate,
}

pub trait ProgressStyle: Send + Sync {
    /// Paint a horizontal progress bar.
    ///
    /// For Indeterminate, `phase` is a value in [0.0, 1.0) that advances each frame,
    /// used to animate the indeterminate pattern. The boring suite ignores phase
    /// and renders a static filled rectangle for indeterminate state.
    fn paint_bar(
        &self,
        ctx:   &mut DrawContext,
        bounds: Rect,
        value: ProgressValue,
        phase: f32,
        theme: &Theme,
    )

    /// Paint a circular progress indicator.
    fn paint_spinner(
        &self,
        ctx:   &mut DrawContext,
        bounds: Rect,
        value: ProgressValue,
        phase: f32,
        theme: &Theme,
    )

    /// Preferred height of a horizontal bar.
    fn bar_height(&self, theme: &Theme): f32

    /// Preferred diameter of a spinner.
    fn spinner_size(&self, theme: &Theme): f32
}
```

---

## 10. ScrollbarStyle

```ferrum
pub trait ScrollbarStyle: Send + Sync {
    /// Paint a vertical or horizontal scrollbar.
    ///
    /// `track_bounds` is the full scrollbar track area.
    /// `thumb_bounds` is the computed thumb rectangle within the track.
    /// The style paints both.
    fn paint(
        &self,
        ctx:          &mut DrawContext,
        track_bounds: Rect,
        thumb_bounds: Rect,
        orientation:  ScrollbarOrientation,
        state:        ScrollbarState,
        theme:        &Theme,
    )

    /// Width (for vertical) or height (for horizontal) of the scrollbar track.
    fn thickness(&self, theme: &Theme): f32

    /// Minimum length of the thumb regardless of content ratio.
    fn min_thumb_length(&self, theme: &Theme): f32

    /// True if the scrollbar should be hidden when not scrolling.
    /// Boring suite returns false (always visible when needed).
    fn auto_hide(&self, theme: &Theme): bool
}

@derive(Debug, Clone, Copy, PartialEq)]
pub enum ScrollbarOrientation { Vertical, Horizontal }

@derive(Debug, Clone, Copy, PartialEq)]
pub enum ScrollbarState {
    /// Not being interacted with. Some suites hide or fade the bar.
    Idle,
    /// Pointer is over the scrollbar track or thumb.
    Hovered,
    /// Thumb is being dragged.
    Dragging,
}
```

---

## 11. MenuStyle and MenuItemStyle

```ferrum
pub trait MenuStyle: Send + Sync {
    /// Paint the menu container (background, border, shadow if the suite uses one).
    ///
    /// `bounds` is the full menu rectangle. Individual items are painted by
    /// MenuItemStyle; this method paints only the container.
    fn paint_container(
        &self,
        ctx:    &mut DrawContext,
        bounds: Rect,
        theme:  &Theme,
    )

    /// Padding between the menu border and the first/last item.
    fn container_padding(&self, theme: &Theme): EdgeInsets
}

@derive(Debug, Clone, Copy, PartialEq)]
pub enum MenuItemKind {
    /// A normal selectable item.
    Normal,
    /// An item that opens a submenu.
    Submenu,
    /// A visual separator between groups of items.
    Separator,
    /// A header label (non-interactive, names a group).
    Header,
}

pub trait MenuItemStyle: Send + Sync {
    /// Paint a single menu item.
    ///
    /// `icon` is Some if the item has a leading icon. `shortcut` is the
    /// keyboard shortcut hint shown at the trailing edge.
    fn paint(
        &self,
        ctx:      &mut DrawContext,
        bounds:   Rect,
        label:    &str,
        icon:     Option[&dyn DrawIcon],
        shortcut: Option[&str],
        kind:     MenuItemKind,
        state:    InteractiveState,
        theme:    &Theme,
    )

    /// Height of one normal menu item row.
    fn item_height(&self, theme: &Theme): f32

    /// Height of a separator item.
    fn separator_height(&self, theme: &Theme): f32

    /// Horizontal padding inside item bounds.
    fn item_padding(&self, theme: &Theme): EdgeInsets
}
```

---

## 12. DialogStyle

```ferrum
pub trait DialogStyle: Send + Sync {
    /// Paint the dialog container (background, border, optional title bar).
    ///
    /// Dialog contents (body, action buttons) are positioned by the layout
    /// engine and painted by their own widgets; this paints only the shell.
    fn paint_container(
        &self,
        ctx:    &mut DrawContext,
        bounds: Rect,
        theme:  &Theme,
    )

    /// Paint the backdrop scrim behind the dialog.
    ///
    /// `scrim_bounds` is the full window/screen area. The boring suite
    /// draws a semi-transparent rectangle.
    fn paint_scrim(
        &self,
        ctx:          &mut DrawContext,
        scrim_bounds: Rect,
        theme:        &Theme,
    )

    /// Padding between the dialog border and its content area.
    fn content_padding(&self, theme: &Theme): EdgeInsets

    /// Padding between content area and action button row.
    fn action_area_padding(&self, theme: &Theme): EdgeInsets

    /// Minimum dialog width.
    fn min_width(&self, theme: &Theme): f32
}
```

---

## 13. TooltipStyle

```ferrum
pub trait TooltipStyle: Send + Sync {
    /// Paint a tooltip.
    ///
    /// `bounds` is the computed tooltip rectangle. The style paints the
    /// background and then the text within the content padding.
    fn paint(
        &self,
        ctx:    &mut DrawContext,
        bounds: Rect,
        text:   &str,
        theme:  &Theme,
    )

    /// Padding inside the tooltip from border to text.
    fn padding(&self, theme: &Theme): EdgeInsets

    /// Maximum width before text wraps.
    fn max_width(&self, theme: &Theme): f32
}
```

---

## 14. ChipStyle

A chip is a compact, optionally-removable label or filter token.

```ferrum
@derive(Debug, Clone, Copy, PartialEq)]
pub enum ChipKind {
    /// Informational label; not interactive.
    Label,
    /// Can be toggled on/off to filter a list.
    Filter { selected: bool },
    /// Triggers an action when tapped.
    Action,
}

pub trait ChipStyle: Send + Sync {
    fn paint(
        &self,
        ctx:          &mut DrawContext,
        bounds:       Rect,
        label:        &str,
        kind:         ChipKind,
        has_remove:   bool,
        state:        InteractiveState,
        theme:        &Theme,
    )

    fn preferred_height(&self, theme: &Theme): f32
    fn content_padding(&self, theme: &Theme): EdgeInsets
}
```

---

## 15. DividerStyle

```ferrum
pub trait DividerStyle: Send + Sync {
    /// Paint a horizontal dividing line.
    fn paint_horizontal(&self, ctx: &mut DrawContext, bounds: Rect, theme: &Theme)

    /// Paint a vertical dividing line.
    fn paint_vertical(&self, ctx: &mut DrawContext, bounds: Rect, theme: &Theme)

    /// Thickness of the divider line.
    fn thickness(&self, theme: &Theme): f32

    /// Vertical space above and below a horizontal divider.
    fn vertical_margin(&self, theme: &Theme): f32
}
```

---

## 16. TabStyle

```ferrum
pub trait TabStyle: Send + Sync {
    /// Paint a single tab label.
    fn paint_tab(
        &self,
        ctx:      &mut DrawContext,
        bounds:   Rect,
        label:    &str,
        selected: bool,
        state:    InteractiveState,
        theme:    &Theme,
    )

    /// Paint the tab bar container background.
    fn paint_bar(
        &self,
        ctx:    &mut DrawContext,
        bounds: Rect,
        theme:  &Theme,
    )

    /// Height of the tab bar.
    fn bar_height(&self, theme: &Theme): f32

    /// Minimum width of one tab.
    fn min_tab_width(&self, theme: &Theme): f32

    /// Padding inside a single tab cell.
    fn tab_padding(&self, theme: &Theme): EdgeInsets
}
```

---

## 17. ListItemStyle

```ferrum
pub trait ListItemStyle: Send + Sync {
    /// Paint one row in a list.
    ///
    /// `leading` and `trailing` are optional icon regions, pre-sized by
    /// the widget. The style draws the background and the text; the widget
    /// draws the icons via DrawIcon.
    fn paint(
        &self,
        ctx:          &mut DrawContext,
        bounds:       Rect,
        primary_text: &str,
        secondary_text: Option[&str],
        state:        InteractiveState,
        theme:        &Theme,
    )

    /// Height when the item has only primary text.
    fn single_line_height(&self, theme: &Theme): f32

    /// Height when the item has primary and secondary text.
    fn two_line_height(&self, theme: &Theme): f32

    /// Horizontal insets from list edge to text content.
    fn content_padding(&self, theme: &Theme): EdgeInsets
}
```

---

## 18. WidgetSuite — the Umbrella Trait

`WidgetSuite` groups all `*Style` implementations into one injectable unit. A module that
wants to provide a complete visual style implements this trait. An incomplete suite — one
that overrides only some styles — wraps a base suite and delegates the rest.

```ferrum
/// A complete set of visual style implementations for all built-in widget types.
///
/// Implement this trait to provide a custom visual style. Drop it into an
/// application with `provide_suite(suite, root_widget)` or set it globally
/// with `Application::set_default_suite(suite)`.
///
/// All associated types must implement their respective `*Style` trait.
/// The `suite()` accessor methods return references to those implementations.
pub trait WidgetSuite: Send + Sync + 'static {
    type Button:    ButtonStyle
    type TextField: TextFieldStyle
    type Checkbox:  CheckboxStyle
    type Radio:     RadioStyle
    type Toggle:    ToggleStyle
    type Slider:    SliderStyle
    type Progress:  ProgressStyle
    type Scrollbar: ScrollbarStyle
    type Menu:      MenuStyle
    type MenuItem:  MenuItemStyle
    type Dialog:    DialogStyle
    type Tooltip:   TooltipStyle
    type Chip:      ChipStyle
    type Divider:   DividerStyle
    type Tab:       TabStyle
    type ListItem:  ListItemStyle

    fn button(&self):     &Self.Button
    fn text_field(&self): &Self.TextField
    fn checkbox(&self):   &Self.Checkbox
    fn radio(&self):      &Self.Radio
    fn toggle(&self):     &Self.Toggle
    fn slider(&self):     &Self.Slider
    fn progress(&self):   &Self.Progress
    fn scrollbar(&self):  &Self.Scrollbar
    fn menu(&self):       &Self.Menu
    fn menu_item(&self):  &Self.MenuItem
    fn dialog(&self):     &Self.Dialog
    fn tooltip(&self):    &Self.Tooltip
    fn chip(&self):       &Self.Chip
    fn divider(&self):    &Self.Divider
    fn tab(&self):        &Self.Tab
    fn list_item(&self):  &Self.ListItem
}
```

### Object-safe erasure

`WidgetSuite` is not object-safe as written (associated types). For dynamic dispatch
across subtrees, the runtime uses an erased version:

```ferrum
/// Object-safe version of WidgetSuite, used by LayoutCtx and BuildCtx.
///
/// This is an implementation detail. Callers interact with WidgetSuite directly;
/// the framework erases it internally via WidgetSuiteErased::from_suite().
pub trait WidgetSuiteErased: Send + Sync {
    fn button_erased(&self):    &dyn ButtonStyle
    fn text_field_erased(&self): &dyn TextFieldStyle
    fn checkbox_erased(&self):  &dyn CheckboxStyle
    fn radio_erased(&self):     &dyn RadioStyle
    fn toggle_erased(&self):    &dyn ToggleStyle
    fn slider_erased(&self):    &dyn SliderStyle
    fn progress_erased(&self):  &dyn ProgressStyle
    fn scrollbar_erased(&self): &dyn ScrollbarStyle
    fn menu_erased(&self):      &dyn MenuStyle
    fn menu_item_erased(&self): &dyn MenuItemStyle
    fn dialog_erased(&self):    &dyn DialogStyle
    fn tooltip_erased(&self):   &dyn TooltipStyle
    fn chip_erased(&self):      &dyn ChipStyle
    fn divider_erased(&self):   &dyn DividerStyle
    fn tab_erased(&self):       &dyn TabStyle
    fn list_item_erased(&self): &dyn ListItemStyle
}

// Blanket implementation: any WidgetSuite becomes WidgetSuiteErased automatically.
impl[S: WidgetSuite] WidgetSuiteErased for S { ... }
```

---

## 19. BuildCtx Integration

`BuildCtx` (defined in `extlib.ccsp.widget`) exposes the active suite:

```ferrum
impl BuildCtx {
    /// The active widget suite for this subtree.
    ///
    /// Widgets do not call this directly during build — they call it during
    /// layout (via LayoutCtx) and paint (by calling suite methods on the
    /// DrawContext extension). However it is exposed here for widget authors
    /// who need to inspect suite properties during the build pass (e.g., to
    /// decide whether to include an animation child).
    pub fn suite(&self): &dyn WidgetSuiteErased
}
```

During `layout`, widgets access the suite via `LayoutCtx::suite()`.
During `paint`, the suite is accessed via the widget's stored `Arc[dyn WidgetSuiteErased]`
snapshot taken at last layout. The snapshot is updated on suite changes before the next
layout pass, ensuring paint always uses a consistent suite.

---

## 20. Subtree Overrides with provide_suite

A single application can mix suites in different subtrees. The outer application uses
`BoringTheme`; a settings panel uses a higher-density compact suite; a modal dialog uses
a Material-styled suite.

```ferrum
/// Provide a suite for a subtree, overriding the parent's suite.
///
/// All widgets within `child`'s subtree see the provided suite via
/// BuildCtx::suite() and LayoutCtx::suite(). Widgets outside the
/// subtree are unaffected.
///
/// This uses the same inherited-data mechanism as provide_theme:
/// the suite is an InheritedData value injected into the BuildCtx.
pub fn provide_suite(suite: impl WidgetSuite + 'static, child: WidgetNode): WidgetNode

/// Retrieve the active suite from BuildCtx.
/// Provided for widget authors who need explicit suite access during build.
pub fn use_suite(ctx: &BuildCtx): &dyn WidgetSuiteErased
```

Example:

```ferrum
fn build(&self, ctx: &BuildCtx): WidgetNode {
    column(vec![
        // Main UI — uses the application's default suite (boring)
        my_main_content(),

        // Settings panel — uses a compact suite
        provide_suite(
            CompactSuite::new(),
            settings_panel(),
        ),
    ])
}
```

---

## 21. Writing a Custom Suite

A complete minimal skeleton:

```ferrum
use extlib.ccsp.widget_suite.*
use extlib.ccsp.draw.{DrawContext, Rect, Color}
use extlib.ccsp.style.Theme

// Step 1: implement each *Style trait
pub type MyButtonStyle;

impl ButtonStyle for MyButtonStyle {
    fn paint(&self, ctx, bounds, label, state, theme) {
        // ... your rendering code ...
    }
    fn content_padding(&self, theme) -> EdgeInsets { EdgeInsets.all(12.0) }
    fn min_size(&self, theme) -> Size { Size { width: 80.0, height: 40.0 } }
    fn preferred_height(&self, theme) -> f32 { 40.0 }
}

// Step 2: implement WidgetSuite
pub type MySuite {
    button: MyButtonStyle,
    // ... other styles ...
}

impl MySuite {
    pub fn new(): Self {
        MySuite {
            button: MyButtonStyle,
            // ...
        }
    }
}

impl WidgetSuite for MySuite {
    type Button    = MyButtonStyle
    // ... other associated types ...

    fn button(&self):     &MyButtonStyle  { &self.button }
    // ...
}
```

### Delegating suite

A suite that overrides only some styles and delegates the rest:

```ferrum
pub type PartialOverrideSuite[Base: WidgetSuite] {
    base:          Base,
    custom_button: MyButtonStyle,
}

impl[Base: WidgetSuite] WidgetSuite for PartialOverrideSuite[Base] {
    type Button    = MyButtonStyle
    type TextField = Base.TextField   // delegates
    // ... etc

    fn button(&self):     &MyButtonStyle        { &self.custom_button }
    fn text_field(&self): &Base.TextField       { self.base.text_field() }
    // ...
}
```

---

## 22. Error Types

`WidgetSuite` implementations do not return errors from paint methods — paint is
infallible by contract. However, suite construction and registration can fail:

```ferrum
@derive(Debug)]
pub enum SuiteError {
    /// Required resource not found during suite initialization (e.g., a font family
    /// named by the suite is not available on this system).
    ResourceNotFound { resource: String },

    /// Suite is incompatible with the running platform (e.g., a GPU suite on a
    /// system with no GPU).
    PlatformIncompatible { reason: String },
}

impl core.error.Error for SuiteError {}
impl fmt.Display for SuiteError { ... }
```

---

## 23. Dependencies Reference

```
extlib.ccsp.widget_suite
    extlib.ccsp.widget        — Widget, Constraints, LayoutCtx, BuildCtx, WidgetNode
    extlib.ccsp.draw          — DrawContext, Color, Rect, Point, EdgeInsets, Size
    extlib.ccsp.style         — Theme, TextStyle, ColorTokens, SpacingTokens
    extlib.ccsp.text_layout   — TextMeasure, TextMeasurement
    extlib.ccsp.a11y          — A11yRole (referenced by widget implementations)
    std.alloc                 — Arc, Box, Vec
    std.sync                  — Send, Sync
```

`extlib.ccsp.widget_suite` has no external C dependencies. All rendering is expressed
through `DrawContext` operations defined in `extlib.ccsp.draw`. Suites that require
platform-specific rendering (GPU effects, platform blur) depend on the relevant draw
backend modules (`extlib.ccsp.draw_gpu`, etc.) and declare those as additional
dependencies in their own `Cargo.toml`.
