# Ferrum Extended Library — boring: The Default Widget Suite

**Module path:** `extlib.ccsp.boring`
**Spec basis:** CCSP `lib_ccsp_boring` — minimal honest widget rendering
**Roadmap status:** Post-1.0 (designed now, implemented after widget_suite stabilizes)
**Dependencies:** `extlib.ccsp.widget_suite` (all `*Style` traits, `WidgetSuite`),
`extlib.ccsp.draw` (DrawContext, Color, Rect, Point, EdgeInsets),
`extlib.ccsp.style` (Theme, ColorTokens, TextStyles, ShapeTokens, MotionTokens),
`extlib.ccsp.text_layout` (TextStyle)

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [What Boring Is Not](#2-what-boring-is-not)
3. [BoringTheme](#3-boringtheme)
4. [BoringWidgetSuite](#4-boringwidgetsuite)
5. [BoringButtonStyle](#5-boringbuttonstyle)
6. [BoringTextFieldStyle](#6-boringtextfieldstyle)
7. [BoringCheckboxStyle](#7-boringcheckboxstyle)
8. [BoringRadioStyle](#8-boringradiostyle)
9. [BoringToggleStyle](#9-boeringtoggles-style)
10. [BoringSliderStyle](#10-boringsliderstyle)
11. [BoringProgressStyle](#11-boringprogressstyle)
12. [BoringScrollbarStyle](#12-boringscrollbarstyle)
13. [BoringMenuStyle and BoringMenuItemStyle](#13-boringmenustyle-and-boringmenuitemstyle)
14. [BoringDialogStyle](#14-boringdialogstyle)
15. [BoringTooltipStyle](#15-boringtooltipstyle)
16. [BoringChipStyle](#16-boringchipstyle)
17. [BoringDividerStyle](#17-boringdividerstyle)
18. [BoringTabStyle](#18-boeringtabstyle)
19. [BoringListItemStyle](#19-boringlistitemstyle)
20. [Accessibility Guarantees](#20-accessibility-guarantees)
21. [Usage Examples](#21-usage-examples)
22. [Dependencies Reference](#22-dependencies-reference)

---

## 1. Philosophy

The boring suite renders every widget as the simplest thing that correctly communicates
its purpose.

**Flat fills.** Backgrounds are solid colors drawn with a single `ctx.fill_rect`. There
is no gradient, no gradient overlay, no inner shadow, no outer shadow, no blur.

**Sharp corners.** Corner radius is zero. A button is a rectangle. A text field is a
rectangle. A dialog is a rectangle. The `ShapeTokens::sharp()` configuration enforces
this: no call to `ctx.fill_rounded_rect` appears anywhere in this module.

**1-pixel borders.** Container edges are 1-pixel solid lines. There is no double border,
no border-image, no dashed-border for "empty" states, no animated border.

**Instant state transitions.** `MotionTokens::instant()` sets every duration to zero.
When a button is pressed, its background color changes on the next frame. When a dialog
opens, it is immediately visible. The `anim` extlib is available in the framework but
this suite does not use it. Animation is not disabled — it is simply absent from the
default, and can be added by any suite that needs it.

**System font.** Text is rendered in the OS default sans-serif typeface at sizes defined
by the theme's `TextStyles`. No web font, no embedded font, no font download.

**Semantic color only.** All colors come from the `Theme`'s `ColorTokens`, which follow
the Material Design 3 semantic role system. No literal hex values appear in this module.
Widget code reads `theme.colors.primary`, `theme.colors.surface`, etc. The specific
values depend on the seed color passed to `BoringTheme::new`; the boring suite itself
is color-agnostic.

### The visual vocabulary

```
Normal button:       ┌─────────────────┐
                     │   Button Label  │   surface fill, 1px outline border
                     └─────────────────┘

Pressed button:      ┌─────────────────┐
                     │   Button Label  │   primary fill, no border
                     └─────────────────┘

Focused button:      ┌═════════════════┐
                     │   Button Label  │   surface fill, 2px primary border
                     └═════════════════┘

Text field:          ┌─────────────────────┐
                     │ content here_       │   surface fill, 1px outline border
                     └─────────────────────┘

Checkbox unchecked:  ┌──┐   Checkbox checked:  ┌──┐
                     └──┘                       │╲ │
                     1px border square           └──┘

Radio unselected:    ○      Radio selected:    ●
                     1px border circle          filled circle

Toggle off:          ─────┤ │     Toggle on:   │ ├─────
                     track-[thumb]              [thumb]-track
```

---

## 2. What Boring Is Not

**Not inaccessible.** The boring suite meets WCAG 2.1 AA contrast ratios on all
color pairs. Focus indicators are visible (2px border in `colors.primary`). Interactive
states are distinguishable. "Boring" describes the absence of unnecessary decoration, not
the absence of usability. See Section 20 for the full accessibility guarantee.

**Not a placeholder.** The boring suite is a complete, production-ready default. It is
not a stub to be replaced — it is a deliberate design choice for applications where
functional clarity is preferable to visual decoration. Administrative tools, developer
UIs, data dashboards, and embedded displays are natural fits. Any suite that applies
rounded corners or drop shadows to these interfaces does not improve them.

**Not an accessibility mode.** There is a separate high-contrast theme variant (see
`BoringTheme::high_contrast`) for users who require higher contrast ratios than AA.
The standard boring suite is not that; it is AA-compliant with normal contrast, not
AAA with maximum contrast.

**Not hostile to animation.** An application using the boring suite can still use
`extlib.ccsp.anim` directly for meaningful motion: a spinner that indicates ongoing
work, a notification that slides in from the edge. The boring suite simply does not
inject animation into widget state transitions where it adds no information.

---

## 3. BoringTheme

`BoringTheme` configures the `Theme` type with the boring suite's constraints.

```ferrum
use extlib.ccsp.style.{Theme, ColorTokens, ShapeTokens, SpacingTokens,
                        MotionTokens, TextStyles}
use extlib.ccsp.draw.Color

pub type BoringTheme;

impl BoringTheme {
    /// Create a BoringTheme with a seed color.
    ///
    /// The seed drives the Material Design 3 tonal palette algorithm
    /// (OKLCh hue extraction, tonal palette generation, role assignment).
    /// The semantic color roles are populated automatically from the seed.
    ///
    /// A neutral seed (gray, white) produces a monochrome palette.
    /// A chromatic seed (blue, green, orange, etc.) produces a palette
    /// with that hue as the primary family.
    ///
    /// Calling code should pass its brand color as seed. When no brand
    /// color exists, `Color::from_srgb8(0x60, 0x60, 0x80, 0xFF)` (a neutral
    /// blue-gray) produces a legible, unassuming result.
    pub fn new(seed: Color): Theme {
        Theme {
            colors:  ColorTokens::from_seed(seed),
            shape:   ShapeTokens::sharp(),     // all corner_radius = 0.0
            spacing: SpacingTokens::compact(), // tight grid, 4px base unit
            motion:  MotionTokens::instant(),  // all durations = Duration::ZERO
            text:    TextStyles::system_ui(),  // OS default sans-serif
        }
    }

    /// Variant: dark mode with the same seed color.
    pub fn dark(seed: Color): Theme {
        Theme {
            colors:  ColorTokens::from_seed_dark(seed),
            shape:   ShapeTokens::sharp(),
            spacing: SpacingTokens::compact(),
            motion:  MotionTokens::instant(),
            text:    TextStyles::system_ui(),
        }
    }

    /// Variant: high-contrast mode. Meets WCAG 2.1 AAA (7:1 minimum contrast).
    /// Intended for users who require maximum contrast, not as the default.
    pub fn high_contrast(seed: Color): Theme {
        Theme {
            colors:  ColorTokens::from_seed_high_contrast(seed),
            shape:   ShapeTokens::sharp(),
            spacing: SpacingTokens::compact(),
            motion:  MotionTokens::instant(),
            text:    TextStyles::system_ui(),
        }
    }
}
```

### ShapeTokens::sharp

```ferrum
impl ShapeTokens {
    /// All corner radii are zero. Widgets are drawn as rectangles.
    pub fn sharp(): Self {
        ShapeTokens {
            extra_small:  0.0,
            small:        0.0,
            medium:       0.0,
            large:        0.0,
            extra_large:  0.0,
            full:         0.0,   // used for pills, chips — also 0 here
        }
    }
}
```

### MotionTokens::instant

```ferrum
impl MotionTokens {
    /// All durations are zero. State changes are instantaneous.
    /// Easing curves are defined but never produce non-zero timing.
    pub fn instant(): Self {
        MotionTokens {
            duration_short1:  Duration::ZERO,
            duration_short2:  Duration::ZERO,
            duration_medium1: Duration::ZERO,
            duration_medium2: Duration::ZERO,
            duration_long1:   Duration::ZERO,
            duration_long2:   Duration::ZERO,
            easing_standard:  EasingCurve::Linear,  // irrelevant at zero duration
            easing_decelerate: EasingCurve::Linear,
            easing_accelerate: EasingCurve::Linear,
        }
    }
}
```

---

## 4. BoringWidgetSuite

```ferrum
/// The boring widget suite. Use as the default suite for applications that
/// want functional, honest rendering without decorative overhead.
pub struct BoringWidgetSuite {
    button:     BoringButtonStyle,
    text_field: BoringTextFieldStyle,
    checkbox:   BoringCheckboxStyle,
    radio:      BoringRadioStyle,
    toggle:     BoringToggleStyle,
    slider:     BoringSliderStyle,
    progress:   BoringProgressStyle,
    scrollbar:  BoringScrollbarStyle,
    menu:       BoringMenuStyle,
    menu_item:  BoringMenuItemStyle,
    dialog:     BoringDialogStyle,
    tooltip:    BoringTooltipStyle,
    chip:       BoringChipStyle,
    divider:    BoringDividerStyle,
    tab:        BoringTabStyle,
    list_item:  BoringListItemStyle,
}

impl BoringWidgetSuite {
    pub fn new(): Self {
        BoringWidgetSuite {
            button:     BoringButtonStyle,
            text_field: BoringTextFieldStyle,
            checkbox:   BoringCheckboxStyle,
            radio:      BoringRadioStyle,
            toggle:     BoringToggleStyle,
            slider:     BoringSliderStyle,
            progress:   BoringProgressStyle,
            scrollbar:  BoringScrollbarStyle,
            menu:       BoringMenuStyle,
            menu_item:  BoringMenuItemStyle,
            dialog:     BoringDialogStyle,
            tooltip:    BoringTooltipStyle,
            chip:       BoringChipStyle,
            divider:    BoringDividerStyle,
            tab:        BoringTabStyle,
            list_item:  BoringListItemStyle,
        }
    }
}

impl WidgetSuite for BoringWidgetSuite {
    type Button    = BoringButtonStyle
    type TextField = BoringTextFieldStyle
    type Checkbox  = BoringCheckboxStyle
    type Radio     = BoringRadioStyle
    type Toggle    = BoringToggleStyle
    type Slider    = BoringSliderStyle
    type Progress  = BoringProgressStyle
    type Scrollbar = BoringScrollbarStyle
    type Menu      = BoringMenuStyle
    type MenuItem  = BoringMenuItemStyle
    type Dialog    = BoringDialogStyle
    type Tooltip   = BoringTooltipStyle
    type Chip      = BoringChipStyle
    type Divider   = BoringDividerStyle
    type Tab       = BoringTabStyle
    type ListItem  = BoringListItemStyle

    fn button(&self):     &BoringButtonStyle     { &self.button }
    fn text_field(&self): &BoringTextFieldStyle  { &self.text_field }
    fn checkbox(&self):   &BoringCheckboxStyle   { &self.checkbox }
    fn radio(&self):      &BoringRadioStyle      { &self.radio }
    fn toggle(&self):     &BoringToggleStyle     { &self.toggle }
    fn slider(&self):     &BoringSliderStyle     { &self.slider }
    fn progress(&self):   &BoringProgressStyle   { &self.progress }
    fn scrollbar(&self):  &BoringScrollbarStyle  { &self.scrollbar }
    fn menu(&self):       &BoringMenuStyle       { &self.menu }
    fn menu_item(&self):  &BoringMenuItemStyle   { &self.menu_item }
    fn dialog(&self):     &BoringDialogStyle     { &self.dialog }
    fn tooltip(&self):    &BoringTooltipStyle    { &self.tooltip }
    fn chip(&self):       &BoringChipStyle       { &self.chip }
    fn divider(&self):    &BoringDividerStyle    { &self.divider }
    fn tab(&self):        &BoringTabStyle        { &self.tab }
    fn list_item(&self):  &BoringListItemStyle   { &self.list_item }
}
```

---

## 5. BoringButtonStyle

```ferrum
pub type BoringButtonStyle;

impl ButtonStyle for BoringButtonStyle {
    fn paint(&self, ctx, bounds, label, state, theme) {
        let (fill, border_color, border_width, text_color) = match state {
            Normal          => (theme.colors.surface,
                                theme.colors.outline,         1.0,
                                theme.colors.on_surface),
            Hovered         => (theme.colors.surface_variant,
                                theme.colors.outline,         1.0,
                                theme.colors.on_surface),
            Pressed         => (theme.colors.primary,
                                theme.colors.primary,         0.0,
                                theme.colors.on_primary),
            Focused         => (theme.colors.surface,
                                theme.colors.primary,         2.0,
                                theme.colors.on_surface),
            FocusedHovered  => (theme.colors.surface_variant,
                                theme.colors.primary,         2.0,
                                theme.colors.on_surface),
            FocusedPressed  => (theme.colors.primary,
                                theme.colors.primary,         0.0,
                                theme.colors.on_primary),
            Disabled        => (theme.colors.surface,
                                theme.colors.outline_variant, 1.0,
                                theme.colors.on_surface_variant),
        }

        ctx.fill_rect(bounds, fill)
        if border_width > 0.0 {
            ctx.stroke_rect(bounds, border_color, border_width)
        }
        ctx.draw_text_centered(
            bounds,
            label,
            &theme.text_styles.label_large,
            text_color,
        )
    }

    fn paint_variant(&self, ctx, bounds, label, variant, state, theme) {
        // Primary: same as above
        // Secondary: use secondary color family on pressed
        // Ghost: no fill in any state; only text and focus border
        match variant {
            ButtonVariant::Primary   => self.paint(ctx, bounds, label, state, theme),
            ButtonVariant::Secondary => {
                let (fill, border_color, border_width, text_color) = match state {
                    Pressed | FocusedPressed =>
                        (theme.colors.secondary,
                         theme.colors.secondary, 0.0,
                         theme.colors.on_secondary),
                    other => {
                        // same as primary for other states
                        return self.paint(ctx, bounds, label, other, theme)
                    }
                }
                ctx.fill_rect(bounds, fill)
                ctx.draw_text_centered(bounds, label, &theme.text_styles.label_large, text_color)
            }
            ButtonVariant::Ghost => {
                // No background fill in any state
                let (text_color, border_color, border_width) = match state {
                    Disabled => (theme.colors.on_surface_variant,
                                 Color.TRANSPARENT, 0.0),
                    Focused | FocusedHovered | FocusedPressed =>
                                (theme.colors.primary,
                                 theme.colors.primary, 2.0),
                    _        => (theme.colors.primary,
                                 Color.TRANSPARENT, 0.0),
                }
                if border_width > 0.0 {
                    ctx.stroke_rect(bounds, border_color, border_width)
                }
                ctx.draw_text_centered(bounds, label, &theme.text_styles.label_large, text_color)
            }
        }
    }

    fn content_padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 8.0, bottom: 8.0, left: 16.0, right: 16.0 }
    }

    fn min_size(&self, _theme): Size {
        Size { width: 64.0, height: 32.0 }
    }

    fn preferred_height(&self, _theme): f32 { 32.0 }
}
```

---

## 6. BoringTextFieldStyle

```ferrum
pub type BoringTextFieldStyle;

impl TextFieldStyle for BoringTextFieldStyle {
    fn paint(&self, ctx, bounds, content, placeholder, label, helper,
             error_message, state, cursor, selection, theme) {
        // Determine colors from state
        let (fill, border_color, border_width, text_color) = match state {
            TextFieldVisualState::Normal(InteractiveState::Focused)
          | TextFieldVisualState::Normal(InteractiveState::FocusedHovered)
          | TextFieldVisualState::Normal(InteractiveState::FocusedPressed) =>
                (theme.colors.surface,
                 theme.colors.primary,         2.0,
                 theme.colors.on_surface),

            TextFieldVisualState::Normal(InteractiveState::Disabled) =>
                (theme.colors.surface,
                 theme.colors.outline_variant, 1.0,
                 theme.colors.on_surface_variant),

            TextFieldVisualState::Error(_) =>
                (theme.colors.error_container,
                 theme.colors.error,           2.0,
                 theme.colors.on_error_container),

            _ =>
                (theme.colors.surface,
                 theme.colors.outline,         1.0,
                 theme.colors.on_surface),
        }

        let input_bounds = self.input_bounds(bounds, label.is_some(), theme)

        // Optional label above the field
        if let Some(lbl) = label {
            let label_bounds = self.label_bounds(bounds, theme)
            let label_color = if state.is_error() {
                theme.colors.error
            } else if state.is_focused() {
                theme.colors.primary
            } else {
                theme.colors.on_surface_variant
            }
            ctx.draw_text(label_bounds.origin, lbl, &theme.text_styles.label_small, label_color)
        }

        // Input area background and border
        ctx.fill_rect(input_bounds, fill)
        ctx.stroke_rect(input_bounds, border_color, border_width)

        let text_area = self.text_bounds(bounds, theme)

        // Selection highlight
        if let Some(sel) = selection {
            // ... selection rectangle drawing omitted for brevity ...
        }

        // Content or placeholder
        if content.is_empty() {
            ctx.draw_text(text_area.origin, placeholder,
                          &theme.text_styles.body_large,
                          theme.colors.on_surface_variant)
        } else {
            ctx.draw_text(text_area.origin, content,
                          &theme.text_styles.body_large,
                          text_color)
        }

        // Cursor: 1-pixel vertical line, no blink (blink is optional animation)
        if let Some(cur) = cursor {
            if !state.is_disabled() {
                let cursor_x = self.cursor_x_for(content, cur, text_area, theme)
                let cursor_rect = Rect {
                    origin: Point { x: cursor_x, y: text_area.origin.y },
                    size:   Size  { width: 1.0, height: text_area.size.height },
                }
                ctx.fill_rect(cursor_rect, theme.colors.on_surface)
            }
        }

        // Helper or error text below the field
        let helper_text = match error_message {
            Some(err) => Some((err, theme.colors.error)),
            None      => helper.map(|h| (h, theme.colors.on_surface_variant)),
        }
        if let Some((text, color)) = helper_text {
            let helper_bounds = self.helper_bounds(bounds, theme)
            ctx.draw_text(helper_bounds.origin, text, &theme.text_styles.body_small, color)
        }
    }

    fn input_height(&self, _theme): f32 { 40.0 }

    fn preferred_height(&self, has_label, has_helper, theme): f32 {
        let label_h  = if has_label  { 20.0 } else { 0.0 }
        let helper_h = if has_helper { 16.0 } else { 0.0 }
        label_h + self.input_height(theme) + helper_h
    }

    fn content_padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 8.0, bottom: 8.0, left: 12.0, right: 12.0 }
    }

    fn text_bounds(&self, bounds: Rect, theme): Rect {
        // text_bounds is the content area of the input box only,
        // accounting for label above and content padding inside
        let padding = self.content_padding(theme)
        Rect {
            origin: Point {
                x: bounds.origin.x + padding.left,
                y: bounds.origin.y + padding.top,
            },
            size: Size {
                width:  bounds.size.width  - padding.horizontal(),
                height: self.input_height(theme) - padding.vertical(),
            },
        }
    }
}
```

---

## 7. BoringCheckboxStyle

```ferrum
pub type BoringCheckboxStyle;

impl CheckboxStyle for BoringCheckboxStyle {
    fn paint(&self, ctx, bounds, value, state, theme) {
        let (fill, border_color, border_width, mark_color) = match state {
            Disabled => (theme.colors.surface,
                         theme.colors.outline_variant, 1.0,
                         theme.colors.on_surface_variant),
            Focused | FocusedHovered | FocusedPressed =>
                        (theme.colors.surface,
                         theme.colors.primary,         2.0,
                         theme.colors.primary),
            _ => match value {
                TriState::Checked | TriState::Indeterminate =>
                        (theme.colors.primary,
                         theme.colors.primary,         0.0,
                         theme.colors.on_primary),
                TriState::Unchecked =>
                        (theme.colors.surface,
                         theme.colors.outline,         1.0,
                         theme.colors.on_surface),
            }
        }

        ctx.fill_rect(bounds, fill)
        if border_width > 0.0 {
            ctx.stroke_rect(bounds, border_color, border_width)
        }

        // Draw the check mark as two line segments forming a tick
        match value {
            TriState::Checked => {
                let s = bounds.size.width
                // Tick: bottom-left to middle-bottom, then middle-bottom to top-right
                let p1 = Point { x: bounds.origin.x + s * 0.15, y: bounds.origin.y + s * 0.55 }
                let p2 = Point { x: bounds.origin.x + s * 0.40, y: bounds.origin.y + s * 0.80 }
                let p3 = Point { x: bounds.origin.x + s * 0.85, y: bounds.origin.y + s * 0.20 }
                ctx.draw_line(p1, p2, mark_color, 1.5)
                ctx.draw_line(p2, p3, mark_color, 1.5)
            }
            TriState::Indeterminate => {
                // Horizontal dash
                let pad = bounds.size.width * 0.2
                let y   = bounds.origin.y + bounds.size.height * 0.5
                let p1  = Point { x: bounds.origin.x + pad, y }
                let p2  = Point { x: bounds.origin.x + bounds.size.width - pad, y }
                ctx.draw_line(p1, p2, mark_color, 1.5)
            }
            TriState::Unchecked => {}
        }
    }

    fn indicator_size(&self, _theme): f32 { 18.0 }
}
```

---

## 8. BoringRadioStyle

Radio buttons use a circle because the circle shape communicates "choose exactly one
from a group" — this is semantic, not decorative. The boring suite keeps the circle
and removes all embellishment from it.

```ferrum
pub type BoringRadioStyle;

impl RadioStyle for BoringRadioStyle {
    fn paint(&self, ctx, bounds, selected, state, theme) {
        let center = bounds.center()
        let radius = bounds.size.width / 2.0

        let (fill, border_color, border_width) = match (selected, state) {
            (_, Disabled) =>
                (theme.colors.surface,
                 theme.colors.outline_variant, 1.0),
            (true, _) =>
                (theme.colors.primary,
                 theme.colors.primary,         0.0),
            (false, Focused) | (false, FocusedHovered) =>
                (theme.colors.surface,
                 theme.colors.primary,         2.0),
            (false, _) =>
                (theme.colors.surface,
                 theme.colors.outline,         1.0),
        }

        ctx.fill_circle(center, radius, fill)
        if border_width > 0.0 {
            ctx.stroke_circle(center, radius, border_color, border_width)
        }

        // Inner dot for selected state
        if selected && state != Disabled {
            ctx.fill_circle(center, radius * 0.4, theme.colors.on_primary)
        }
    }

    fn indicator_size(&self, _theme): f32 { 18.0 }
}
```

---

## 9. BoringToggleStyle

```ferrum
pub type BoringToggleStyle;

impl ToggleStyle for BoringToggleStyle {
    fn paint(&self, ctx, bounds, on, transition, state, theme) {
        // transition is 0.0 or 1.0 (instant — no animation)
        // We use `on` directly; transition is ignored.
        let (track_fill, track_border, thumb_fill) = match (on, state) {
            (_, Disabled) =>
                (theme.colors.surface,
                 theme.colors.outline_variant,
                 theme.colors.on_surface_variant),
            (true, _) =>
                (theme.colors.primary,
                 theme.colors.primary,
                 theme.colors.on_primary),
            (false, Focused) | (false, FocusedHovered) =>
                (theme.colors.surface_variant,
                 theme.colors.primary,
                 theme.colors.on_surface_variant),
            (false, _) =>
                (theme.colors.surface_variant,
                 theme.colors.outline,
                 theme.colors.on_surface_variant),
        }

        // Track: full bounds, flat rectangle
        ctx.fill_rect(bounds, track_fill)
        ctx.stroke_rect(bounds, track_border, 1.0)

        // Thumb: square, slides to left (off) or right (on)
        let thumb_w = bounds.size.height  // square thumb
        let thumb_x = if on {
            bounds.origin.x + bounds.size.width - thumb_w
        } else {
            bounds.origin.x
        }
        let thumb_rect = Rect {
            origin: Point { x: thumb_x, y: bounds.origin.y },
            size:   Size  { width: thumb_w, height: bounds.size.height },
        }
        ctx.fill_rect(thumb_rect, thumb_fill)
    }

    fn preferred_size(&self, _theme): Size {
        Size { width: 44.0, height: 22.0 }
    }
}
```

---

## 10. BoringSliderStyle

```ferrum
pub type BoringSliderStyle;

impl SliderStyle for BoringSliderStyle {
    fn paint(&self, ctx, bounds, value, thumb_center, state, theme) {
        let track_y  = bounds.origin.y + bounds.size.height / 2.0
        let track_h  = self.track_height(theme)
        let track_rect = Rect {
            origin: Point { x: bounds.origin.x,   y: track_y - track_h / 2.0 },
            size:   Size  { width: bounds.size.width, height: track_h },
        }

        // Unfilled track (full width, background color)
        let unfilled_color = if state == Disabled {
            theme.colors.outline_variant
        } else {
            theme.colors.surface_variant
        }
        ctx.fill_rect(track_rect, unfilled_color)

        // Filled track (left portion, primary color)
        let filled_w = thumb_center.x - bounds.origin.x
        if filled_w > 0.0 && state != Disabled {
            let filled_rect = Rect {
                origin: track_rect.origin,
                size:   Size { width: filled_w, height: track_h },
            }
            ctx.fill_rect(filled_rect, theme.colors.primary)
        }

        // Thumb: square
        let ts = self.thumb_size(theme)
        let thumb_rect = Rect {
            origin: Point {
                x: thumb_center.x - ts.width  / 2.0,
                y: thumb_center.y - ts.height / 2.0,
            },
            size: ts,
        }
        let thumb_color = if state == Disabled {
            theme.colors.on_surface_variant
        } else {
            theme.colors.primary
        }
        ctx.fill_rect(thumb_rect, thumb_color)

        // Focus border on the thumb
        if state.has_focus() {
            ctx.stroke_rect(thumb_rect, theme.colors.primary, 2.0)
        }
    }

    fn track_height(&self, _theme): f32 { 4.0 }
    fn thumb_size(&self, _theme): Size  { Size { width: 16.0, height: 16.0 } }
    fn preferred_height(&self, _theme): f32 { 24.0 }
}
```

---

## 11. BoringProgressStyle

Indeterminate progress is rendered as a static filled partial bar, not an animation.
The position of the partial fill is fixed (left-aligned, 30% width). This communicates
"something is happening" without implying any direction or speed. An application that
needs an animated spinner should use `extlib.ccsp.anim` directly.

```ferrum
pub type BoringProgressStyle;

impl ProgressStyle for BoringProgressStyle {
    fn paint_bar(&self, ctx, bounds, value, _phase, theme) {
        // Track background
        ctx.fill_rect(bounds, theme.colors.surface_variant)
        ctx.stroke_rect(bounds, theme.colors.outline_variant, 1.0)

        let fill_w = match value {
            ProgressValue::Determinate(v) => bounds.size.width * v.clamp(0.0, 1.0),
            ProgressValue::Indeterminate  => bounds.size.width * 0.30,
        }
        if fill_w > 0.0 {
            let fill_rect = Rect {
                origin: bounds.origin,
                size:   Size { width: fill_w, height: bounds.size.height },
            }
            ctx.fill_rect(fill_rect, theme.colors.primary)
        }
    }

    fn paint_spinner(&self, ctx, bounds, value, _phase, theme) {
        // Spinner: square outline with a filled arc-approximated by a filled segment.
        // Boring suite uses a static arc: a square with one side highlighted.
        // Real animation is opt-in via extlib.ccsp.anim.
        let center = bounds.center()
        let radius = bounds.size.width.min(bounds.size.height) / 2.0 - 2.0
        ctx.stroke_circle(center, radius, theme.colors.surface_variant, 3.0)

        let sweep = match value {
            ProgressValue::Determinate(v) => v.clamp(0.0, 1.0),
            ProgressValue::Indeterminate  => 0.25,
        }
        ctx.stroke_arc(center, radius, 0.0, sweep * 360.0,
                       theme.colors.primary, 3.0)
    }

    fn bar_height(&self, _theme): f32     { 8.0 }
    fn spinner_size(&self, _theme): f32   { 32.0 }
}
```

---

## 12. BoringScrollbarStyle

```ferrum
pub type BoringScrollbarStyle;

impl ScrollbarStyle for BoringScrollbarStyle {
    fn paint(&self, ctx, track_bounds, thumb_bounds, _orientation, state, theme) {
        // Track background — always visible
        ctx.fill_rect(track_bounds, theme.colors.surface_variant)

        // Thumb color
        let thumb_color = match state {
            ScrollbarState::Idle     => theme.colors.outline_variant,
            ScrollbarState::Hovered  => theme.colors.outline,
            ScrollbarState::Dragging => theme.colors.primary,
        }
        ctx.fill_rect(thumb_bounds, thumb_color)
    }

    fn thickness(&self, _theme): f32         { 12.0 }
    fn min_thumb_length(&self, _theme): f32  { 24.0 }
    fn auto_hide(&self, _theme): bool        { false }
}
```

---

## 13. BoringMenuStyle and BoringMenuItemStyle

```ferrum
pub type BoringMenuStyle;

impl MenuStyle for BoringMenuStyle {
    fn paint_container(&self, ctx, bounds, theme) {
        ctx.fill_rect(bounds, theme.colors.surface)
        ctx.stroke_rect(bounds, theme.colors.outline, 1.0)
    }

    fn container_padding(&self, _theme): EdgeInsets {
        EdgeInsets.symmetric(vertical: 4.0, horizontal: 0.0)
    }
}

pub type BoringMenuItemStyle;

impl MenuItemStyle for BoringMenuItemStyle {
    fn paint(&self, ctx, bounds, label, icon, shortcut, kind, state, theme) {
        match kind {
            MenuItemKind::Separator => {
                // Horizontal divider line
                let y = bounds.origin.y + bounds.size.height / 2.0
                ctx.draw_line(
                    Point { x: bounds.origin.x + 8.0,                   y },
                    Point { x: bounds.origin.x + bounds.size.width - 8.0, y },
                    theme.colors.outline_variant, 1.0,
                )
                return
            }
            MenuItemKind::Header => {
                ctx.draw_text(
                    Point { x: bounds.origin.x + 8.0, y: bounds.origin.y + 4.0 },
                    label,
                    &theme.text_styles.label_small,
                    theme.colors.on_surface_variant,
                )
                return
            }
            _ => {}
        }

        // Hover/selected highlight
        let bg = match state {
            Hovered | FocusedHovered => theme.colors.surface_variant,
            Pressed | FocusedPressed => theme.colors.primary_container,
            Disabled                 => theme.colors.surface,
            _                        => Color.TRANSPARENT,
        }
        if bg.a > 0.0 {
            ctx.fill_rect(bounds, bg)
        }

        let text_color = if state == Disabled {
            theme.colors.on_surface_variant
        } else {
            theme.colors.on_surface
        }

        let padding = self.item_padding(theme)
        let text_x  = bounds.origin.x + padding.left

        ctx.draw_text(
            Point { x: text_x, y: bounds.origin.y + padding.top },
            label,
            &theme.text_styles.body_large,
            text_color,
        )

        // Submenu arrow at trailing edge
        if kind == MenuItemKind::Submenu {
            let arrow_x = bounds.origin.x + bounds.size.width - padding.right - 8.0
            let arrow_y = bounds.origin.y + bounds.size.height / 2.0
            ctx.draw_text(
                Point { x: arrow_x, y: arrow_y - 8.0 },
                "›",
                &theme.text_styles.body_large,
                text_color,
            )
        }

        // Keyboard shortcut at trailing edge
        if let Some(sc) = shortcut {
            let sc_x = bounds.origin.x + bounds.size.width - padding.right
                       - ctx.measure_text(sc, &theme.text_styles.label_small, f32.INFINITY).size.width
            ctx.draw_text(
                Point { x: sc_x, y: bounds.origin.y + padding.top + 2.0 },
                sc,
                &theme.text_styles.label_small,
                theme.colors.on_surface_variant,
            )
        }
    }

    fn item_height(&self, _theme): f32       { 32.0 }
    fn separator_height(&self, _theme): f32  { 9.0  }
    fn item_padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 6.0, bottom: 6.0, left: 12.0, right: 12.0 }
    }
}
```

---

## 14. BoringDialogStyle

```ferrum
pub type BoringDialogStyle;

impl DialogStyle for BoringDialogStyle {
    fn paint_container(&self, ctx, bounds, theme) {
        ctx.fill_rect(bounds, theme.colors.surface)
        ctx.stroke_rect(bounds, theme.colors.outline, 1.0)
    }

    fn paint_scrim(&self, ctx, scrim_bounds, theme) {
        // Semi-transparent overlay — communicates modal state without blurring
        let scrim_color = theme.colors.scrim.with_alpha(0.50)
        ctx.fill_rect(scrim_bounds, scrim_color)
    }

    fn content_padding(&self, _theme): EdgeInsets {
        EdgeInsets.all(24.0)
    }

    fn action_area_padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 0.0, bottom: 24.0, left: 24.0, right: 24.0 }
    }

    fn min_width(&self, _theme): f32 { 280.0 }
}
```

---

## 15. BoringTooltipStyle

```ferrum
pub type BoringTooltipStyle;

impl TooltipStyle for BoringTooltipStyle {
    fn paint(&self, ctx, bounds, text, theme) {
        ctx.fill_rect(bounds, theme.colors.inverse_surface)
        ctx.stroke_rect(bounds, theme.colors.outline, 1.0)
        let padding = self.padding(theme)
        ctx.draw_text(
            Point {
                x: bounds.origin.x + padding.left,
                y: bounds.origin.y + padding.top,
            },
            text,
            &theme.text_styles.label_small,
            theme.colors.inverse_on_surface,
        )
    }

    fn padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 4.0, bottom: 4.0, left: 8.0, right: 8.0 }
    }

    fn max_width(&self, _theme): f32 { 240.0 }
}
```

---

## 16. BoringChipStyle

```ferrum
pub type BoringChipStyle;

impl ChipStyle for BoringChipStyle {
    fn paint(&self, ctx, bounds, label, kind, has_remove, state, theme) {
        let (fill, border_color, text_color) = match (kind, state) {
            (_, Disabled) =>
                (theme.colors.surface,
                 theme.colors.outline_variant,
                 theme.colors.on_surface_variant),
            (ChipKind::Filter { selected: true }, _) =>
                (theme.colors.secondary_container,
                 theme.colors.secondary_container,
                 theme.colors.on_secondary_container),
            (_, Hovered) | (_, FocusedHovered) =>
                (theme.colors.surface_variant,
                 theme.colors.outline,
                 theme.colors.on_surface),
            _ =>
                (theme.colors.surface,
                 theme.colors.outline,
                 theme.colors.on_surface),
        }

        ctx.fill_rect(bounds, fill)
        ctx.stroke_rect(bounds, border_color, 1.0)

        let padding = self.content_padding(theme)
        ctx.draw_text(
            Point { x: bounds.origin.x + padding.left, y: bounds.origin.y + padding.top },
            label,
            &theme.text_styles.label_large,
            text_color,
        )

        // Remove button: a plain "×" at the trailing edge
        if has_remove {
            let x_x = bounds.origin.x + bounds.size.width - padding.right - 12.0
            ctx.draw_text(
                Point { x: x_x, y: bounds.origin.y + padding.top },
                "×",
                &theme.text_styles.label_large,
                text_color,
            )
        }
    }

    fn preferred_height(&self, _theme): f32 { 28.0 }
    fn content_padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 4.0, bottom: 4.0, left: 12.0, right: 12.0 }
    }
}
```

---

## 17. BoringDividerStyle

```ferrum
pub type BoringDividerStyle;

impl DividerStyle for BoringDividerStyle {
    fn paint_horizontal(&self, ctx, bounds, theme) {
        let y = bounds.origin.y + bounds.size.height / 2.0
        ctx.draw_line(
            Point { x: bounds.origin.x,                   y },
            Point { x: bounds.origin.x + bounds.size.width, y },
            theme.colors.outline_variant, 1.0,
        )
    }

    fn paint_vertical(&self, ctx, bounds, theme) {
        let x = bounds.origin.x + bounds.size.width / 2.0
        ctx.draw_line(
            Point { x, y: bounds.origin.y },
            Point { x, y: bounds.origin.y + bounds.size.height },
            theme.colors.outline_variant, 1.0,
        )
    }

    fn thickness(&self, _theme): f32        { 1.0 }
    fn vertical_margin(&self, _theme): f32  { 8.0 }
}
```

---

## 18. BoringTabStyle

```ferrum
pub type BoringTabStyle;

impl TabStyle for BoringTabStyle {
    fn paint_tab(&self, ctx, bounds, label, selected, state, theme) {
        // Selected tab: primary border on the bottom edge only
        // Unselected: no border, no fill
        let text_color = if selected {
            theme.colors.primary
        } else if state == Disabled {
            theme.colors.on_surface_variant
        } else {
            theme.colors.on_surface
        }

        // Hovered tab gets a subtle background
        if matches!(state, Hovered | FocusedHovered) && !selected {
            ctx.fill_rect(bounds, theme.colors.surface_variant)
        }

        ctx.draw_text_centered(bounds, label, &theme.text_styles.title_small, text_color)

        if selected {
            // Bottom indicator: 2px line at the bottom edge
            let indicator_rect = Rect {
                origin: Point {
                    x: bounds.origin.x,
                    y: bounds.origin.y + bounds.size.height - 2.0,
                },
                size: Size { width: bounds.size.width, height: 2.0 },
            }
            ctx.fill_rect(indicator_rect, theme.colors.primary)
        }

        if state.has_focus() {
            ctx.stroke_rect(bounds, theme.colors.primary, 2.0)
        }
    }

    fn paint_bar(&self, ctx, bounds, theme) {
        ctx.fill_rect(bounds, theme.colors.surface)
        // Bottom border separates tab bar from content
        let bottom_line = Rect {
            origin: Point {
                x: bounds.origin.x,
                y: bounds.origin.y + bounds.size.height - 1.0,
            },
            size: Size { width: bounds.size.width, height: 1.0 },
        }
        ctx.fill_rect(bottom_line, theme.colors.outline_variant)
    }

    fn bar_height(&self, _theme): f32            { 40.0 }
    fn min_tab_width(&self, _theme): f32         { 80.0 }
    fn tab_padding(&self, _theme): EdgeInsets    {
        EdgeInsets { top: 8.0, bottom: 8.0, left: 16.0, right: 16.0 }
    }
}
```

---

## 19. BoringListItemStyle

```ferrum
pub type BoringListItemStyle;

impl ListItemStyle for BoringListItemStyle {
    fn paint(&self, ctx, bounds, primary_text, secondary_text, state, theme) {
        let bg = match state {
            Hovered | FocusedHovered => theme.colors.surface_variant,
            Pressed | FocusedPressed => theme.colors.primary_container,
            Focused                  => theme.colors.surface,
            Disabled                 => theme.colors.surface,
            Normal                   => Color.TRANSPARENT,
        }
        if bg.a > 0.0 {
            ctx.fill_rect(bounds, bg)
        }
        if state.has_focus() {
            ctx.stroke_rect(bounds, theme.colors.primary, 2.0)
        }

        let text_color = if state == Disabled {
            theme.colors.on_surface_variant
        } else {
            theme.colors.on_surface
        }

        let padding = self.content_padding(theme)
        let text_x = bounds.origin.x + padding.left
        let text_y = bounds.origin.y + padding.top

        ctx.draw_text(
            Point { x: text_x, y: text_y },
            primary_text,
            &theme.text_styles.body_large,
            text_color,
        )

        if let Some(sec) = secondary_text {
            ctx.draw_text(
                Point { x: text_x, y: text_y + theme.text_styles.body_large.line_height },
                sec,
                &theme.text_styles.body_small,
                theme.colors.on_surface_variant,
            )
        }
    }

    fn single_line_height(&self, _theme): f32 { 48.0 }
    fn two_line_height(&self, _theme): f32    { 64.0 }
    fn content_padding(&self, _theme): EdgeInsets {
        EdgeInsets { top: 12.0, bottom: 12.0, left: 16.0, right: 16.0 }
    }
}
```

---

## 20. Accessibility Guarantees

The boring suite makes the following guarantees about every widget it renders:

- **Focus indicators.** Every focusable widget displays a 2-pixel border in
  `theme.colors.primary` when it has keyboard focus. This indicator is always visible:
  it is never suppressed, faded, or delayed. The 2-pixel width meets WCAG 2.2 Success
  Criterion 2.4.11 (Focus Appearance, minimum).

- **Color contrast.** The boring suite uses only the semantic color roles defined in
  `extlib.ccsp.style`. When `BoringTheme::new(seed)` generates the color palette, all
  role pairings that the boring suite uses (e.g., `on_primary` on `primary`,
  `on_surface` on `surface`) are generated with a contrast ratio ≥ 4.5:1 (WCAG 2.1 AA
  for normal text). The tonal palette algorithm enforces this; callers who override
  individual colors in `ColorTokens` are responsible for their own contrast.

- **Disabled state.** Disabled widgets are rendered with `on_surface_variant` on
  `surface`, which is intentionally lower-contrast than normal state. WCAG exempts
  disabled controls from contrast requirements; the boring suite follows this exemption.
  Disabled widgets still have a visually distinct appearance to prevent confusion.

- **No state communicated by color alone.** The boring suite does not rely on color as
  the only distinguishing factor between states. Checked checkboxes have a tick mark.
  Selected radio buttons have an inner dot. The focused state has a border (shape
  change), not merely a color change. This satisfies WCAG 1.4.1 (Use of Color).

- **Pointer cursors.** The boring suite communicates interactive regions by declaring
  the appropriate pointer cursor (hand for buttons, text cursor for text fields) to the
  platform. This is not visible in the paint methods above — it is handled by the widget
  layer — but the boring suite never suppresses or overrides this behavior.

---

## 21. Usage Examples

### Application-level default

```ferrum
fn main() ! IO {
    let app = Application::new(AppConfig {
        app_id:      "com.example.myapp",
        app_name:    "My Application",
        app_version: "1.0.0",
        ..AppConfig::default()
    })?

    // Set the default suite and theme for all windows
    app.set_default_suite(BoringWidgetSuite::new())
    app.set_default_theme(BoringTheme::new(Color::from_srgb8(0x00, 0x60, 0xC0, 0xFF)))

    app.run(|ctx| {
        ctx.push_window(AppWindow::new(MainView::new()))
    })
}
```

### Subtree override

An application that uses the boring suite by default but wants a specific
dialog to use a different suite:

```ferrum
fn build(&self, ctx: &BuildCtx): WidgetNode {
    column(vec![
        my_main_form(ctx),

        // This dialog uses a different suite — only applies within the subtree
        provide_suite(
            FancySuite::new(),
            provide_theme(
                FancySuite::theme(Color::from_hex(0x6750A4FF)),
                my_dialog(),
            ),
        ),
    ])
}
```

### Building with boring as a base

A custom suite that overrides only the button style:

```ferrum
use extlib.ccsp.boring.{BoringWidgetSuite, *}

pub struct MyBrandedSuite {
    base:          BoringWidgetSuite,
    button:        MyBrandedButtonStyle,
}

impl WidgetSuite for MyBrandedSuite {
    type Button    = MyBrandedButtonStyle
    type TextField = BoringTextFieldStyle   // all others delegate to boring
    // ...

    fn button(&self):     &MyBrandedButtonStyle { &self.button }
    fn text_field(&self): &BoringTextFieldStyle { self.base.text_field() }
    // ...
}
```

---

## 22. Dependencies Reference

```
extlib.ccsp.boring
    extlib.ccsp.widget_suite  — WidgetSuite and all *Style traits
    extlib.ccsp.draw          — DrawContext, Color, Rect, Point, EdgeInsets, Size
    extlib.ccsp.style         — Theme, ColorTokens, ShapeTokens, MotionTokens,
                                SpacingTokens, TextStyles
    extlib.ccsp.text_layout   — TextStyle (used only via Theme::text_styles)
    std.alloc                 — no heap allocation in paint methods
```

`extlib.ccsp.boring` has no external C dependencies and no platform-specific code.
It runs on every target that `extlib.ccsp.draw` supports: Wayland, X11, framebuffer,
image backend (for testing and server-side rendering), and GPU backends.
