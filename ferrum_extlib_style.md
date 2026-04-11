# Ferrum Extended Library — style: Design Tokens and Theming

**Module path:** `extlib.ccsp.style`
**Layer:** Extended standard library (not stdlib; requires optional dependency)
**Spec basis:** CCSP `lib_ccsp_style` — design tokens and theming
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.draw` (Color, Point, EdgeInsets), `extlib.ccsp.text_layout` (TextStyle), `extlib.ccsp.anim` (EasingCurve), `extlib.ccsp.widget` (Widget trait, WidgetTree); stdlib `sync` (RwLock, Arc)

---

## 1. Overview and Rationale

### Why Design Tokens

A design token is a named design decision: a specific color, a spacing value, a corner radius, a motion duration. Instead of scattering literal values through widget code (`Color.from_hex(0x6750A4FF)`, `border_radius: 12.0`, `duration: 200`), tokens give each decision a name and a single authoritative source. The widget code says `theme.colors.primary` and `theme.geometry.border_radius_medium`; the token values live in the theme.

This separation has three concrete consequences:

**Systematic consistency.** Every button, card, chip, and text field that uses `theme.colors.primary` changes together when the primary color changes. There is no spreadsheet of places to update. Consistency is structural, not disciplinary.

**Easy dark mode and high contrast.** Dark mode is a different `Theme` value, not a mirror image maintained in parallel CSS. Switching from light to dark is `handle.set(Theme.dark())`. High-contrast mode is `handle.set(Theme.high_contrast())`. Widget code is unchanged; it reads whatever theme the handle currently provides.

**Widget code that is readable.** `ctx.fill_rect(rect, theme.colors.surface)` communicates intent. `ctx.fill_rect(rect, Color.from_hex(0xFFFBFEFF))` communicates nothing except a hex value that may or may not be current.

### Why NOT CSS

CSS is the dominant theming mechanism in web browsers. It is not the right model for a native widget system.

**Specificity and cascade create implicit dependencies.** In CSS, every style rule can potentially affect every element. A rule added to solve one problem silently changes an unrelated component. The interaction between a component's own styles and styles inherited from ancestors is non-local: you cannot understand a component's appearance by reading only its own code. These interactions become bugs that are difficult to isolate and reproduce.

**`!important` is an arms race.** The specificity system inevitably produces `!important` declarations to win specificity contests. `!important` in a component's stylesheet can then be overridden only by `!important` in a more specific rule, creating an escalation that ends in unmaintainable style sheets.

**Selector matching has runtime cost.** A CSS engine must evaluate every rule against every element on every style recalculation. Even with incremental invalidation, selector matching is work proportional to rule count times element count in the worst case. A native widget system that owns the widget tree can apply tokens at layout time with zero matching cost: each widget reads named tokens from its `Theme` reference. There are no selectors.

**No layout engine, no inheritance by default.** CSS assumes a document model where properties like `font-family` and `color` inherit from parent elements. Native widget layout is a tree of positioned boxes; there is no inherited property mechanism. Simulating CSS inheritance in a native widget system requires either introducing the inheritance machinery (expensive) or documenting which properties inherit and which do not (ambiguous and surprising).

**Overrides are explicit code, not specificity games.** In `extlib.ccsp.style`, a widget override is a `WidgetOverrides` value passed to the widget constructor. You can read the widget construction site and immediately know what is overridden. There is no need to search for rules that might win the specificity contest.

### Why an Extlib

`extlib.ccsp.style` depends on `extlib.ccsp.draw` for `Color`, `Point`, and `EdgeInsets`; on `extlib.ccsp.text_layout` for `TextStyle`; on `extlib.ccsp.anim` for `EasingCurve`; and on `extlib.ccsp.widget` for the `Widget` trait. These are all optional extlibs. A program that uses only network I/O and JSON parsing never pays for any of this. Placing style in the standard library would impose all four extlib dependencies on every binary.

Programs that build UIs opt in explicitly:

```toml
[dependencies]
extlib.ccsp.style = { version = "1.0" }
```

---

## 2. Color Tokens

### 2.1 Color Role System

`ColorTokens` encodes the Material Design 3 color role system. Each role is a named semantic slot, not a raw color value. A widget uses `colors.primary` rather than an OKLCh triple. The semantic name communicates purpose; the value communicates appearance; they are decoupled.

```ferrum
/// The complete set of color roles for a theme.
///
/// All colors are in linear-light RGBA (extlib.ccsp.draw Color type).
/// Semantic naming follows Material Design 3 color role conventions.
///
/// Every role comes in a family:
///   <role>           — the container or interactive surface color
///   on_<role>        — text and icons drawn on top of <role>
///   <role>_container — a less prominent surface in the same hue
///   on_<role>_container — text and icons on top of <role>_container
///
/// This four-way structure guarantees legible text on every surface
/// without ad-hoc contrast adjustments.
@derive(Debug, Clone)]
pub type ColorTokens {
    // Primary family — the brand color, used for key interactive elements
    primary:              Color,
    primary_container:    Color,
    on_primary:           Color,
    on_primary_container: Color,

    // Secondary family — less prominent accents
    secondary:              Color,
    secondary_container:    Color,
    on_secondary:           Color,
    on_secondary_container: Color,

    // Tertiary family — contrasting accent for differentiation
    tertiary:              Color,
    tertiary_container:    Color,
    on_tertiary:           Color,
    on_tertiary_container: Color,

    // Error family — destructive actions, validation errors
    error:              Color,
    error_container:    Color,
    on_error:           Color,
    on_error_container: Color,

    // Background and surface — the canvas roles
    background:    Color,
    on_background: Color,
    surface:         Color,
    surface_variant: Color,
    on_surface:         Color,
    on_surface_variant: Color,

    // Outline — borders, dividers, focus rings
    outline:         Color,
    outline_variant: Color,

    // Utility roles
    shadow: Color,   // drop shadow color
    scrim:  Color,   // modal backdrop color

    // Inverse roles — used on snackbars, tooltips (inverted surface)
    inverse_surface:    Color,
    inverse_on_surface: Color,
    inverse_primary:    Color,
}
```

### 2.2 Tonal Palette Generation

`ColorTokens::from_seed` generates a complete M3 color scheme from a single brand color. The algorithm works in OKLCh:

1. Extract the hue angle from the seed color.
2. Generate a tonal palette for the primary hue: 13 tones at fixed lightness stops (0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 95, 99, 100).
3. Derive secondary hue by rotating +60 degrees (analogous, warmer).
4. Derive tertiary hue by rotating -60 degrees (analogous, cooler).
5. Derive neutral hue by desaturating the primary (chroma reduced to ~0.02).
6. Assign M3 role tone mappings to light-mode slots (primary = tone 40, on_primary = tone 100, primary_container = tone 90, etc.).

All arithmetic uses OKLCh from `extlib.ccsp.draw`'s `Color::to_oklch` and `Color::from_oklch`.

```ferrum
impl ColorTokens {
    /// Generate a complete M3 color scheme from a single seed color.
    ///
    /// The seed color's hue drives the primary family. Secondary and tertiary
    /// hues are derived by rotating ±60° in OKLCh. Neutral hues are desaturated
    /// variants of the primary.
    ///
    /// The resulting scheme is a light-mode scheme. Pass to Theme::dark() or
    /// Theme::high_contrast() to derive dark and high-contrast variants.
    pub fn from_seed(seed: Color): ColorTokens { ... }

    /// Construct from fully explicit role values.
    ///
    /// No validation is performed at construction time. Call
    /// Theme::validate_contrast() to check WCAG ratios.
    pub fn new(
        primary: Color,
        primary_container: Color,
        on_primary: Color,
        on_primary_container: Color,
        secondary: Color,
        secondary_container: Color,
        on_secondary: Color,
        on_secondary_container: Color,
        tertiary: Color,
        tertiary_container: Color,
        on_tertiary: Color,
        on_tertiary_container: Color,
        error: Color,
        error_container: Color,
        on_error: Color,
        on_error_container: Color,
        background: Color,
        on_background: Color,
        surface: Color,
        surface_variant: Color,
        on_surface: Color,
        on_surface_variant: Color,
        outline: Color,
        outline_variant: Color,
        shadow: Color,
        scrim: Color,
        inverse_surface: Color,
        inverse_on_surface: Color,
        inverse_primary: Color,
    ): ColorTokens { ... }
}
```

---

## 3. Typography Tokens

### 3.1 Type Scale

`TypographyTokens` encodes the Material Design 3 type scale: five categories (display, headline, title, label, body) each with three sizes (large, medium, small). Each slot is a `TextStyle` from `extlib.ccsp.text_layout`.

```ferrum
/// The complete M3 type scale.
///
/// Each field is a TextStyle (font_family, font_size, weight, line_height,
/// letter_spacing, etc.) from extlib.ccsp.text_layout.
///
/// Intended use by category:
///   display  — short, expressive text at large sizes (hero text, splash screens)
///   headline — section headings, dialogs titles
///   title    — list items, app bar, card headers
///   label    — button labels, tabs, navigation items (prominent small text)
///   body     — body copy, descriptions, input field text
@derive(Debug, Clone)]
pub type TypographyTokens {
    display_large:  TextStyle,
    display_medium: TextStyle,
    display_small:  TextStyle,

    headline_large:  TextStyle,
    headline_medium: TextStyle,
    headline_small:  TextStyle,

    title_large:  TextStyle,
    title_medium: TextStyle,
    title_small:  TextStyle,

    label_large:  TextStyle,
    label_medium: TextStyle,
    label_small:  TextStyle,

    body_large:  TextStyle,
    body_medium: TextStyle,
    body_small:  TextStyle,
}
```

### 3.2 M3 Baseline Type Scale Values

The M3 baseline scale, using the system default sans-serif font:

| Token           | Size (sp) | Weight | Line height | Letter spacing |
|-----------------|-----------|--------|-------------|----------------|
| display_large   | 57        | 400    | 64          | -0.25          |
| display_medium  | 45        | 400    | 52          | 0.0            |
| display_small   | 36        | 400    | 44          | 0.0            |
| headline_large  | 32        | 400    | 40          | 0.0            |
| headline_medium | 28        | 400    | 36          | 0.0            |
| headline_small  | 24        | 400    | 32          | 0.0            |
| title_large     | 22        | 400    | 28          | 0.0            |
| title_medium    | 16        | 500    | 24          | +0.15          |
| title_small     | 14        | 500    | 20          | +0.1           |
| label_large     | 14        | 500    | 20          | +0.1           |
| label_medium    | 12        | 500    | 16          | +0.5           |
| label_small     | 11        | 500    | 16          | +0.5           |
| body_large      | 16        | 400    | 24          | +0.5           |
| body_medium     | 14        | 400    | 20          | +0.25          |
| body_small      | 12        | 400    | 16          | +0.4           |

Sizes are in scale-independent pixels (sp), which on most platforms equal logical pixels at 1x density.

```ferrum
impl TypographyTokens {
    /// M3 baseline type scale using the system default sans-serif font.
    pub fn default(): TypographyTokens { ... }

    /// M3 baseline type scale with a specific font family applied to all slots.
    ///
    /// Use this to apply a custom brand typeface while keeping all M3
    /// size, weight, and spacing values unchanged.
    pub fn default_with_font(family: &str): TypographyTokens { ... }

    /// Construct from fully explicit slot values.
    pub fn new(
        display_large:  TextStyle,
        display_medium: TextStyle,
        display_small:  TextStyle,
        headline_large:  TextStyle,
        headline_medium: TextStyle,
        headline_small:  TextStyle,
        title_large:  TextStyle,
        title_medium: TextStyle,
        title_small:  TextStyle,
        label_large:  TextStyle,
        label_medium: TextStyle,
        label_small:  TextStyle,
        body_large:  TextStyle,
        body_medium: TextStyle,
        body_small:  TextStyle,
    ): TypographyTokens { ... }
}
```

---

## 4. Geometry Tokens

### 4.1 Shape and Spacing

`GeometryTokens` collects shape, elevation, spacing, and icon size tokens. Spacing uses a 4 logical-pixel base unit, matching the M3 specification.

```ferrum
/// Combined shadow and tinted surface for a Material elevation level.
///
/// Material Design 3 expresses elevation via two simultaneous effects:
///   1. A drop shadow (color, offset, blur radius).
///   2. A primary-color tint applied to the surface at a given opacity.
///      Higher elevation = more tint = the surface appears slightly colored.
///
/// shadow_color is pre-multiplied into the shadow; callers do not need to
/// alpha-blend separately. surface_tint is an opacity in [0.0, 1.0] applied
/// to the theme's primary color before compositing onto the surface.
@derive(Debug, Clone, Copy)]
pub type ElevationStyle {
    shadow_color:  Color,
    shadow_offset: Point,
    shadow_blur:   f32,
    surface_tint:  f32,   // opacity of primary-color overlay on the surface
}

/// Shape, elevation, spacing, and icon size tokens.
///
/// All linear dimensions are in logical pixels (device-independent).
/// Spacing uses a 4 px base unit:
///   spacing_1  =  4 px
///   spacing_2  =  8 px
///   spacing_3  = 12 px
///   spacing_4  = 16 px
///   spacing_6  = 24 px
///   spacing_8  = 32 px
///   spacing_12 = 48 px
///   spacing_16 = 64 px
///   spacing_24 = 96 px
@derive(Debug, Clone)]
pub type GeometryTokens {
    // Corner radii — M3 shape scale
    border_radius_small:       f32,   // 4 px  — chips, text fields
    border_radius_medium:      f32,   // 12 px — cards, dialogs
    border_radius_large:       f32,   // 16 px — bottom sheets
    border_radius_extra_large: f32,   // 28 px — FABs, large dialogs
    border_radius_full:        f32,   // 9999 px — pills, switches

    // Elevation levels — M3 tonal elevation
    elevation_0: ElevationStyle,   // flat — no shadow, no tint
    elevation_1: ElevationStyle,   // cards (resting)
    elevation_2: ElevationStyle,   // elevated cards, hovered buttons
    elevation_3: ElevationStyle,   // navigation drawers
    elevation_4: ElevationStyle,   // modal sheets
    elevation_5: ElevationStyle,   // dialogs

    // Spacing — 4 px base unit
    spacing_1:  f32,   //  4 px
    spacing_2:  f32,   //  8 px
    spacing_3:  f32,   // 12 px
    spacing_4:  f32,   // 16 px
    spacing_6:  f32,   // 24 px
    spacing_8:  f32,   // 32 px
    spacing_12: f32,   // 48 px
    spacing_16: f32,   // 64 px
    spacing_24: f32,   // 96 px

    // Icon sizes
    icon_size_small:  f32,   // 16 px
    icon_size_medium: f32,   // 24 px — default
    icon_size_large:  f32,   // 48 px
}
```

### 4.2 Default Values

The M3 baseline geometry tokens:

```ferrum
impl GeometryTokens {
    /// M3 baseline geometry tokens.
    pub fn default(): GeometryTokens {
        GeometryTokens {
            border_radius_small:       4.0,
            border_radius_medium:      12.0,
            border_radius_large:       16.0,
            border_radius_extra_large: 28.0,
            border_radius_full:        9999.0,

            elevation_0: ElevationStyle {
                shadow_color:  Color.from_srgb8(0, 0, 0, 0),
                shadow_offset: Point.new(0.0, 0.0),
                shadow_blur:   0.0,
                surface_tint:  0.0,
            },
            elevation_1: ElevationStyle {
                shadow_color:  Color.from_srgb8(0, 0, 0, 38),   // 15% opacity
                shadow_offset: Point.new(0.0, 1.0),
                shadow_blur:   2.0,
                surface_tint:  0.05,
            },
            elevation_2: ElevationStyle {
                shadow_color:  Color.from_srgb8(0, 0, 0, 51),   // 20% opacity
                shadow_offset: Point.new(0.0, 2.0),
                shadow_blur:   4.0,
                surface_tint:  0.08,
            },
            elevation_3: ElevationStyle {
                shadow_color:  Color.from_srgb8(0, 0, 0, 64),   // 25% opacity
                shadow_offset: Point.new(0.0, 4.0),
                shadow_blur:   8.0,
                surface_tint:  0.11,
            },
            elevation_4: ElevationStyle {
                shadow_color:  Color.from_srgb8(0, 0, 0, 77),   // 30% opacity
                shadow_offset: Point.new(0.0, 6.0),
                shadow_blur:   10.0,
                surface_tint:  0.12,
            },
            elevation_5: ElevationStyle {
                shadow_color:  Color.from_srgb8(0, 0, 0, 89),   // 35% opacity
                shadow_offset: Point.new(0.0, 8.0),
                shadow_blur:   12.0,
                surface_tint:  0.14,
            },

            spacing_1:  4.0,
            spacing_2:  8.0,
            spacing_3:  12.0,
            spacing_4:  16.0,
            spacing_6:  24.0,
            spacing_8:  32.0,
            spacing_12: 48.0,
            spacing_16: 64.0,
            spacing_24: 96.0,

            icon_size_small:  16.0,
            icon_size_medium: 24.0,
            icon_size_large:  48.0,
        }
    }
}
```

**Why 4 px base unit:** 4 divides evenly into 8, 12, 16, 24, 32, 48. These are the dominant spacing values in modern UI systems (M3, Apple HIG, Fluent). A 4 px grid keeps components visually aligned without requiring per-widget pixel-pushing. Naming tokens by multiplier (`spacing_4` = 4 × 4 = 16 px) makes the relationship to the base unit explicit.

**Why explicit elevation levels rather than a formula:** The five M3 elevation levels are specified by the M3 system with known shadow and tint values. Encoding them as explicit tokens means widgets reference `geometry.elevation_2` and the designer can adjust what level 2 looks like without changing widget code. A formula would couple widget code to shadow parameters.

---

## 5. Motion Tokens

### 5.1 Duration and Easing

`MotionTokens` encodes the M3 motion system: four duration categories (short, medium, long, extra-long) at four sub-steps each, and six named easing curves.

```ferrum
/// The complete M3 motion token set.
///
/// Duration naming:
///   short_1..short_4   — 50..200 ms  — micro-interactions (checkbox, ripple)
///   medium_1..medium_4 — 250..400 ms — component transitions (expand, collapse)
///   long_1..long_4     — 450..600 ms — full-screen transitions
///   extra_long_1..extra_long_4 — 700..1000 ms — complex, deliberate transitions
///
/// Easing curve naming follows M3 conventions:
///   standard           — default for most transitions (enters and exits)
///   standard_accelerate — elements leaving the screen (start slow, end fast)
///   standard_decelerate — elements entering the screen (start fast, end slow)
///   emphasized          — for expressive, attention-drawing transitions
///   emphasized_accelerate — emphasized exit
///   emphasized_decelerate — emphasized entrance
///
/// EasingCurve is the cubic Bezier type from extlib.ccsp.anim.
@derive(Debug, Clone)]
pub type MotionTokens {
    duration_short_1: Duration,   // 50 ms
    duration_short_2: Duration,   // 100 ms
    duration_short_3: Duration,   // 150 ms
    duration_short_4: Duration,   // 200 ms

    duration_medium_1: Duration,  // 250 ms
    duration_medium_2: Duration,  // 300 ms
    duration_medium_3: Duration,  // 350 ms
    duration_medium_4: Duration,  // 400 ms

    duration_long_1: Duration,    // 450 ms
    duration_long_2: Duration,    // 500 ms
    duration_long_3: Duration,    // 550 ms
    duration_long_4: Duration,    // 600 ms

    duration_extra_long_1: Duration,  // 700 ms
    duration_extra_long_2: Duration,  // 800 ms
    duration_extra_long_3: Duration,  // 900 ms
    duration_extra_long_4: Duration,  // 1000 ms

    easing_standard:            EasingCurve,  // cubic-bezier(0.2, 0.0, 0.0, 1.0)
    easing_standard_accelerate: EasingCurve,  // cubic-bezier(0.3, 0.0, 1.0, 1.0)
    easing_standard_decelerate: EasingCurve,  // cubic-bezier(0.0, 0.0, 0.0, 1.0)
    easing_emphasized:            EasingCurve, // M3 path easing (approximated cubic)
    easing_emphasized_accelerate: EasingCurve, // cubic-bezier(0.3, 0.0, 0.8, 0.15)
    easing_emphasized_decelerate: EasingCurve, // cubic-bezier(0.05, 0.7, 0.1, 1.0)
}
```

### 5.2 Default Motion Tokens

```ferrum
impl MotionTokens {
    /// M3 baseline motion tokens.
    pub fn default(): MotionTokens {
        MotionTokens {
            duration_short_1: Duration.from_millis(50),
            duration_short_2: Duration.from_millis(100),
            duration_short_3: Duration.from_millis(150),
            duration_short_4: Duration.from_millis(200),

            duration_medium_1: Duration.from_millis(250),
            duration_medium_2: Duration.from_millis(300),
            duration_medium_3: Duration.from_millis(350),
            duration_medium_4: Duration.from_millis(400),

            duration_long_1: Duration.from_millis(450),
            duration_long_2: Duration.from_millis(500),
            duration_long_3: Duration.from_millis(550),
            duration_long_4: Duration.from_millis(600),

            duration_extra_long_1: Duration.from_millis(700),
            duration_extra_long_2: Duration.from_millis(800),
            duration_extra_long_3: Duration.from_millis(900),
            duration_extra_long_4: Duration.from_millis(1000),

            easing_standard:            EasingCurve.cubic(0.2, 0.0, 0.0, 1.0),
            easing_standard_accelerate: EasingCurve.cubic(0.3, 0.0, 1.0, 1.0),
            easing_standard_decelerate: EasingCurve.cubic(0.0, 0.0, 0.0, 1.0),
            easing_emphasized:            EasingCurve.cubic(0.2, 0.0, 0.0, 1.0),
            easing_emphasized_accelerate: EasingCurve.cubic(0.3, 0.0, 0.8, 0.15),
            easing_emphasized_decelerate: EasingCurve.cubic(0.05, 0.7, 0.1, 1.0),
        }
    }
}
```

**Why named duration steps rather than a single value:** Widget code that says `motion.duration_short_3` communicates the motion's intention: it is a micro-interaction, on the fast end, but not instantaneous. A literal `150` communicates none of that. If the designer decides micro-interactions should be faster, changing `duration_short_3` to 120 ms updates every widget that uses it.

**Why cubic Bezier easing:** Cubic Bezier curves with four control-point parameters cover the full range of standard easing profiles (ease-in, ease-out, ease-in-out, linear). The M3 "emphasized" easing is technically a path-based easing that cubic Bezier approximates well within 2% error over the 0..1 range. `EasingCurve` from `extlib.ccsp.anim` stores the curve and provides an `evaluate(t: f32): f32` method that the animation system calls during frame updates.

---

## 6. Theme — The Complete Design System

### 6.1 Theme Type

```ferrum
/// The complete design system for a UI.
///
/// A Theme is an immutable snapshot of all design tokens. It is cheaply
/// cloneable (all fields are Clone). The ThemeHandle wraps a Theme in an
/// Arc[RwLock[Theme]] for shared mutable access across the widget tree.
///
/// Construct with Theme::default() for the M3 baseline blue theme, or with
/// Theme::from_seed(color) for a brand-specific theme.
@derive(Debug, Clone)]
pub type Theme {
    pub colors:     ColorTokens,
    pub typography: TypographyTokens,
    pub geometry:   GeometryTokens,
    pub motion:     MotionTokens,
}
```

### 6.2 Construction and Variants

```ferrum
impl Theme {
    /// Material Design 3 baseline light theme with blue-violet primary.
    ///
    /// Primary seed: #6750A4 (M3 reference primary).
    pub fn default(): Theme { ... }

    /// Generate a complete M3 theme from a single brand color.
    ///
    /// Derives a full ColorTokens via ColorTokens::from_seed, then combines
    /// with the M3 baseline TypographyTokens, GeometryTokens, and MotionTokens.
    pub fn from_seed(seed: Color): Theme {
        Theme {
            colors:     ColorTokens.from_seed(seed),
            typography: TypographyTokens.default(),
            geometry:   GeometryTokens.default(),
            motion:     MotionTokens.default(),
        }
    }

    /// Dark variant of this theme.
    ///
    /// Derives dark color roles from the same hues, following the M3 dark scheme
    /// tone-mapping table (primary becomes tone 80, on_primary becomes tone 20,
    /// surface becomes tone 6, background becomes tone 6, etc.).
    ///
    /// Typography, geometry, and motion tokens are unchanged: dark mode is a
    /// color change, not a layout or motion change.
    pub fn dark(&self): Theme {
        Theme {
            colors:     self.colors.to_dark(),
            typography: self.typography.clone(),
            geometry:   self.geometry.clone(),
            motion:     self.motion.clone(),
        }
    }

    /// WCAG AAA high-contrast variant of this theme.
    ///
    /// Adjusts color token values to meet:
    ///   - Minimum 4.5:1 contrast ratio for body text (WCAG AA)
    ///   - Minimum 7:1 contrast ratio for all on_* tokens against their
    ///     corresponding surface tokens (WCAG AAA for normal text)
    ///   - Minimum 4.5:1 for UI component boundaries (focus rings, input borders)
    ///
    /// Hues are preserved; lightness is adjusted in OKLCh to meet thresholds.
    /// Theme::validate_contrast() on the returned Theme will produce no warnings.
    pub fn high_contrast(&self): Theme { ... }

    /// Override only the color tokens, keeping typography, geometry, and motion.
    pub fn with_colors(&self, colors: ColorTokens): Theme {
        Theme {
            colors,
            typography: self.typography.clone(),
            geometry:   self.geometry.clone(),
            motion:     self.motion.clone(),
        }
    }

    /// Apply a new typeface to all typography tokens, keeping all other tokens.
    ///
    /// Replaces the font_family on every TextStyle in typography.
    /// All size, weight, and spacing values are unchanged.
    pub fn with_font(&self, family: &str): Theme {
        Theme {
            colors:     self.colors.clone(),
            typography: TypographyTokens.default_with_font(family),
            geometry:   self.geometry.clone(),
            motion:     self.motion.clone(),
        }
    }
}
```

### 6.3 Dark Scheme Tone Mapping

The `to_dark()` method on `ColorTokens` applies the M3 dark scheme tone table. The hues and chromas are the same as the light scheme; only the tone (lightness) assignments change. Key mappings:

| Role                 | Light tone | Dark tone |
|----------------------|-----------|-----------|
| primary              | 40        | 80        |
| on_primary           | 100       | 20        |
| primary_container    | 90        | 30        |
| on_primary_container | 10        | 90        |
| surface              | 99        | 6         |
| on_surface           | 10        | 90        |
| background           | 99        | 6         |
| on_background        | 10        | 90        |
| surface_variant      | 90        | 30        |
| on_surface_variant   | 30        | 80        |
| outline              | 50        | 60        |

```ferrum
impl ColorTokens {
    /// Derive the dark variant by remapping tones per the M3 dark scheme table.
    /// Hues and chromas are preserved from the light scheme.
    pub fn to_dark(&self): ColorTokens { ... }
}
```

---

## 7. Runtime Theme Switching

### 7.1 ThemeHandle

```ferrum
/// A shared, runtime-switchable reference to a Theme.
///
/// ThemeHandle wraps Arc[RwLock[Theme]]. All widgets that receive this handle
/// share the same underlying Theme snapshot. Calling set() on any clone of
/// the handle updates all of them simultaneously.
///
/// Clone is cheap: it clones the Arc, not the Theme.
@derive(Debug, Clone)]
pub type ThemeHandle {
    inner: Arc[RwLock[Theme]],
}

impl ThemeHandle {
    /// Create a new ThemeHandle initialized with the given theme.
    pub fn new(theme: Theme): ThemeHandle {
        ThemeHandle { inner: Arc.new(RwLock.new(theme)) }
    }

    /// Replace the current theme.
    ///
    /// All widgets reading this handle will see the new theme on their next
    /// draw call. The widget system marks affected subtrees dirty and schedules
    /// a repaint; no manual invalidation is required.
    pub fn set(&self, theme: Theme) {
        let mut guard = self.inner.write()
        *guard = theme
    }

    /// Read the current theme.
    ///
    /// Returns a clone of the current Theme snapshot. The clone is immediately
    /// owned by the caller; no lock is held after this call returns.
    pub fn current(&self): Theme {
        self.inner.read().clone()
    }

    /// Toggle between light and dark variants.
    ///
    /// If the current theme was constructed with Theme::dark(), switches to
    /// Theme::default(). Otherwise, switches to the dark variant of the
    /// current theme. Uses is_dark() to determine the current state.
    pub fn toggle_dark(&self) {
        let current = self.current()
        if current.is_dark() {
            self.set(current.light())
        } else {
            self.set(current.dark())
        }
    }
}
```

### 7.2 ThemedWidget — Theme Injection

`ThemedWidget` injects a `ThemeHandle` into a widget subtree. All descendants that accept a `ThemeHandle` should be constructed with the same handle. The tree is not walked automatically; handle propagation is explicit at widget construction time.

```ferrum
/// Wraps a widget subtree and provides a ThemeHandle to it.
///
/// ThemedWidget itself is a transparent layout container: it contributes no
/// visual appearance and passes all layout constraints directly to its child.
///
/// Purpose: place a ThemedWidget at the root of a window or a subtree to
/// establish the theme for that subtree. When the handle is updated via
/// ThemeHandle::set(), the ThemedWidget schedules a repaint of its subtree.
pub type ThemedWidget {
    pub theme: ThemeHandle,
    pub child: Arc[dyn Widget],
}

impl Widget for ThemedWidget { ... }
```

### 7.3 System Theme Detection

```ferrum
impl Theme {
    /// Query the platform for the user's dark mode preference.
    ///
    /// Platform sources:
    ///   Linux (freedesktop): org.freedesktop.appearance.color-scheme via dbus.
    ///     Reads setting 1 (dark) or 2 (light). Falls back to false if dbus
    ///     is unavailable or the property is not set.
    ///   macOS: [NSApp appearance] or NSAppearanceNameAqua/DarkAqua via Cocoa.
    ///   Windows: reads Software\Microsoft\Windows\CurrentVersion\Themes\Personalize
    ///     AppsUseLightTheme from the registry (0 = dark, 1 = light).
    ///
    /// Returns true if the system prefers dark mode, false otherwise.
    /// Returns false if the preference cannot be determined.
    pub fn system_prefers_dark(): bool ! IO { ... }

    /// True if this Theme was derived via Theme::dark() or has dark surface tones.
    ///
    /// Determined by comparing the OKLCh lightness of colors.surface against
    /// 0.5: values below 0.5 indicate a dark theme.
    pub fn is_dark(&self): bool { ... }

    /// Return the light variant of this theme.
    ///
    /// If the theme was generated from a seed, re-derives the light color
    /// scheme from the same hues. If derived from Theme::default(), returns
    /// Theme::default(). Preserves typography, geometry, and motion tokens.
    pub fn light(&self): Theme { ... }
}
```

### 7.4 Adapting to System Theme at Startup

```ferrum
// Typical startup pattern: match system preference, listen for changes.
let base_theme = Theme.from_seed(Color.from_hex(0x1B6EF3FF))
let initial_theme = if Theme.system_prefers_dark() ! IO {
    base_theme.dark()
} else {
    base_theme
}
let handle = ThemeHandle.new(initial_theme)
```

---

## 8. Widget Override Model

### 8.1 Explicit Overrides, No Cascade

Every design system must allow exceptions: the delete button is red, not primary; the hero card has a larger corner radius; a specific text field has extra-wide padding for emphasis. In CSS, these exceptions are additional selectors with higher specificity. In `extlib.ccsp.style`, they are explicit `WidgetOverrides` values passed at widget construction.

```ferrum
/// Per-widget token overrides.
///
/// Each field is an Option. None means "use the theme token". Some(value)
/// means "use this value instead, ignoring the theme token for this property."
///
/// Overrides are applied at draw time. They do not affect other widgets.
/// There is no inheritance: a Card with a background override does not cause
/// its Text child to use a different color unless the Text widget also receives
/// an override.
///
/// This is intentional. Implicit inheritance across widget boundaries is the
/// source of cascade surprises in CSS. Explicit overrides are local: you can
/// read the widget construction site and know exactly what is overridden.
@derive(Debug, Clone, Default)]
pub type WidgetOverrides {
    pub border_radius: Option[f32],
    pub background:    Option[Color],
    pub text_color:    Option[Color],
    pub padding:       Option[EdgeInsets],
}
```

### 8.2 Override Resolution

Override resolution is a two-step lookup with no recursion and no ancestor search:

1. If `overrides.field` is `Some(value)`, use `value`.
2. Otherwise, use `theme.colors.<relevant_token>` (or the appropriate geometry/typography token).

There are exactly two possible outcomes per property per widget. No specificity score, no rule ordering, no inheritance chain.

```ferrum
// Illustrative: how a button resolves its background color.
fn resolve_background(theme: &Theme, overrides: &WidgetOverrides, style: ButtonStyle): Color {
    match overrides.background {
        Some(c) => c,
        None    => match style {
            ButtonStyle.Filled  => theme.colors.primary,
            ButtonStyle.Tonal   => theme.colors.secondary_container,
            ButtonStyle.Outlined => Color.TRANSPARENT,
            ButtonStyle.Text     => Color.TRANSPARENT,
            ButtonStyle.Elevated => theme.colors.surface,
        },
    }
}
```

---

## 9. Built-in Themed Widgets

All built-in widgets implement `extlib.ccsp.widget::Widget`. All draw operations use theme tokens exclusively; no literal color, size, or duration values appear in widget code.

### 9.1 Button

```ferrum
/// M3 button styles.
pub enum ButtonStyle {
    Filled,       // primary background, on_primary text
    FilledTonal,  // secondary_container background, on_secondary_container text
    Outlined,     // transparent background, primary border and text
    Text,         // transparent background, primary text, no border
    Elevated,     // surface background with elevation_1 shadow
}

/// A Material Design 3 button.
pub type Button {
    pub label:     String,
    pub on_tap:    Box[dyn Fn()],
    pub theme:     ThemeHandle,
    pub style:     ButtonStyle,
    pub overrides: WidgetOverrides,
    pub enabled:   bool,
}

impl Widget for Button { ... }
```

### 9.2 TextField

```ferrum
/// A single-line text input field.
///
/// value is shared mutable state; the widget reads and writes through it.
/// on_change is called after every keystroke with the new string value.
pub type TextField {
    pub value:       Arc[RefCell[String]],
    pub label:       String,
    pub placeholder: Option[String],
    pub theme:       ThemeHandle,
    pub overrides:   WidgetOverrides,
    pub on_change:   Box[dyn Fn(&str)],
    pub enabled:     bool,
    pub error_text:  Option[String],   // if Some, draws error state
}

impl Widget for TextField { ... }
```

### 9.3 Checkbox

```ferrum
/// A labeled checkbox.
pub type Checkbox {
    pub checked:   Arc[Cell[bool]],
    pub label:     String,
    pub theme:     ThemeHandle,
    pub overrides: WidgetOverrides,
    pub on_change: Box[dyn Fn(bool)],
    pub enabled:   bool,
}

impl Widget for Checkbox { ... }
```

### 9.4 RadioButton

```ferrum
/// A single radio button within a group.
///
/// Grouping is the caller's responsibility: the caller tracks which value
/// is selected and passes selected=true to at most one RadioButton per group.
pub type RadioButton {
    pub selected:  bool,
    pub label:     String,
    pub theme:     ThemeHandle,
    pub overrides: WidgetOverrides,
    pub on_tap:    Box[dyn Fn()],
    pub enabled:   bool,
}

impl Widget for RadioButton { ... }
```

### 9.5 Switch

```ferrum
/// A toggle switch.
pub type Switch {
    pub on:        Arc[Cell[bool]],
    pub theme:     ThemeHandle,
    pub overrides: WidgetOverrides,
    pub on_change: Box[dyn Fn(bool)],
    pub enabled:   bool,
}

impl Widget for Switch { ... }
```

### 9.6 Chip

```ferrum
/// An M3 chip: compact interactive element for filters, choices, or actions.
pub type Chip {
    pub label:     String,
    pub theme:     ThemeHandle,
    pub overrides: WidgetOverrides,
    pub on_tap:    Box[dyn Fn()],
    pub selected:  bool,
    pub enabled:   bool,
}

impl Widget for Chip { ... }
```

### 9.7 Card

```ferrum
/// An M3 card: a surface container with optional elevation.
///
/// elevation is 0..5, indexing into theme.geometry.elevation_0..elevation_5.
/// Elevation 1 is the M3 default for filled cards.
pub type Card {
    pub child:     Arc[dyn Widget],
    pub theme:     ThemeHandle,
    pub overrides: WidgetOverrides,
    pub elevation: u8,   // 0..=5
}

impl Widget for Card { ... }
```

### 9.8 Divider

```ferrum
/// A horizontal rule using theme.colors.outline_variant.
pub type Divider {
    pub theme: ThemeHandle,
}

impl Widget for Divider { ... }
```

### 9.9 ProgressIndicator

```ferrum
/// A linear progress indicator.
///
/// value: Some(v) where v is in [0.0, 1.0] for determinate progress.
/// value: None for indeterminate (animated cycling).
pub type ProgressIndicator {
    pub value:     Option[f32],
    pub theme:     ThemeHandle,
    pub overrides: WidgetOverrides,
}

impl Widget for ProgressIndicator { ... }
```

---

## 10. Accessibility Integration

### 10.1 Contrast Validation

`Theme::validate_contrast` checks every (on_X, X) pair in the color token set against WCAG contrast thresholds. It returns a list of warnings for any pair that falls below threshold. Widgets that want to validate their specific token pairs can also call `contrast_ratio` directly.

```ferrum
/// A WCAG contrast warning: a token pair whose contrast ratio falls below
/// the required threshold.
@derive(Debug, Clone)]
pub type ContrastWarning {
    /// Name of the foreground color token (e.g., "on_primary").
    pub foreground_token:    String,
    /// Name of the background color token (e.g., "primary").
    pub background_token:    String,
    /// Measured contrast ratio (WCAG formula, 1.0..21.0).
    pub ratio:               f32,
    /// Required minimum ratio for this pair's usage category.
    pub required_ratio:      f32,
    /// The WCAG level this pair fails to meet.
    pub fails_level:         WcagLevel,
}

pub enum WcagLevel {
    AA,    // 4.5:1 for normal text, 3.0:1 for large text and UI components
    AAA,   // 7.0:1 for normal text, 4.5:1 for large text and UI components
}

impl Theme {
    /// Validate all color token pairs against WCAG contrast requirements.
    ///
    /// Checks the following pairs:
    ///   (on_primary,            primary)
    ///   (on_primary_container,  primary_container)
    ///   (on_secondary,          secondary)
    ///   (on_secondary_container, secondary_container)
    ///   (on_tertiary,           tertiary)
    ///   (on_tertiary_container, tertiary_container)
    ///   (on_error,              error)
    ///   (on_error_container,    error_container)
    ///   (on_background,         background)
    ///   (on_surface,            surface)
    ///   (on_surface_variant,    surface_variant)
    ///
    /// Each on_X token is evaluated as foreground against its paired X token
    /// as background. body text pairs use 4.5:1 for AA and 7:1 for AAA.
    /// UI component and large text pairs use 3:1 for AA and 4.5:1 for AAA.
    ///
    /// Returns an empty Vec if all pairs pass WCAG AA. Returns warnings for
    /// any pair that fails AA or AAA.
    pub fn validate_contrast(&self): Vec[ContrastWarning] { ... }
}
```

### 10.2 WCAG Contrast Ratio Formula

```ferrum
/// Compute the WCAG 2.1 relative luminance of a linear-light color.
///
/// Input is a Color in linear light (the extlib.ccsp.draw Color type). Because
/// Color already stores linear light values, no gamma conversion is needed.
/// The formula is simply the standard luminance coefficients.
pub fn relative_luminance(c: Color): f32 {
    0.2126 * c.r + 0.7152 * c.g + 0.0722 * c.b
}

/// Compute the WCAG 2.1 contrast ratio between two colors.
///
/// Returns a value in [1.0, 21.0]. 1.0 is no contrast (identical colors);
/// 21.0 is maximum contrast (black on white).
pub fn contrast_ratio(foreground: Color, background: Color): f32 {
    let l1 = relative_luminance(foreground)
    let l2 = relative_luminance(background)
    let lighter = math.max(l1, l2)
    let darker  = math.min(l1, l2)
    (lighter + 0.05) / (darker + 0.05)
}
```

### 10.3 High-Contrast Guarantee

`Theme::high_contrast` guarantees that `validate_contrast` on the returned theme produces no `WcagLevel::AAA` warnings for body text pairs and no `WcagLevel::AA` warnings for any pair. The guarantee is enforced by the algorithm: it adjusts lightness in OKLCh until every required ratio is met, then performs a final validation pass and panics in debug builds if the threshold is not achieved. A passing `high_contrast()` theme is a correct input for accessibility-sensitive deployments.

---

## 11. Example Usage

### 11.1 Create a Branded Theme from a Seed Color

```ferrum
import extlib.ccsp.draw.{Color}
import extlib.ccsp.style.{Theme, ThemeHandle, ThemedWidget}

// Derive a full M3 theme from a brand color.
let brand_color = Color.from_hex(0x1B6EF3FF)   // brand blue
let light_theme = Theme.from_seed(brand_color)
let dark_theme  = light_theme.dark()

// Start in the system-preferred mode.
let initial_theme = if Theme.system_prefers_dark() ! IO {
    dark_theme.clone()
} else {
    light_theme.clone()
}

let theme_handle = ThemeHandle.new(initial_theme)
```

### 11.2 Dark Mode Toggle

```ferrum
// A toggle button that switches the entire UI between light and dark.
let toggle = Button {
    label:     "Toggle dark mode".to_string(),
    theme:     theme_handle.clone(),
    style:     ButtonStyle.Text,
    overrides: WidgetOverrides.default(),
    enabled:   true,
    on_tap:    Box.new(|| {
        // Switch between the two pre-built themes.
        let current = theme_handle.current()
        if current.is_dark() {
            theme_handle.set(light_theme.clone())
        } else {
            theme_handle.set(dark_theme.clone())
        }
    }),
}
```

### 11.3 Override Button Color for a Danger Action

```ferrum
// A delete button: same shape and typography as a filled button, but red.
// Override is explicit code — no specificity, no cascade.
let current_theme = theme_handle.current()
let delete_button = Button {
    label:     "Delete account".to_string(),
    theme:     theme_handle.clone(),
    style:     ButtonStyle.Filled,
    enabled:   true,
    on_tap:    Box.new(|| { handle_delete() }),
    overrides: WidgetOverrides {
        background:  Some(current_theme.colors.error),
        text_color:  Some(current_theme.colors.on_error),
        padding:     None,
        border_radius: None,
    },
}
```

### 11.4 Display Contrast Warnings

```ferrum
// Validate a custom theme before deploying.
let warnings = my_theme.validate_contrast()
if warnings.is_empty() {
    log.info("Theme passes WCAG AA contrast for all token pairs.")
} else {
    for w in &warnings {
        log.warn(
            "{}/{}: ratio {:.2} (required {:.2}, fails {:?})",
            w.foreground_token,
            w.background_token,
            w.ratio,
            w.required_ratio,
            w.fails_level,
        )
    }
}
```

### 11.5 Custom Font with Existing Color Scheme

```ferrum
// Apply a brand typeface to the default blue theme.
let brand_theme = Theme.default().with_font("Inter")
let handle = ThemeHandle.new(brand_theme)
```

### 11.6 Widget Tree with Themed Widgets

```ferrum
import extlib.ccsp.style.{Button, ButtonStyle, Card, Checkbox, WidgetOverrides}
import extlib.ccsp.widget.{Column, Padding}
import extlib.ccsp.draw.{EdgeInsets}
import sync.{Arc, Cell}

let checked = Arc.new(Cell.new(false))

let content = Column {
    children: vec[
        Arc.new(Card {
            elevation: 1,
            theme:     theme_handle.clone(),
            overrides: WidgetOverrides.default(),
            child:     Arc.new(Padding {
                insets: EdgeInsets.all(16.0),
                child:  Arc.new(Checkbox {
                    checked:   checked.clone(),
                    label:     "Accept terms".to_string(),
                    theme:     theme_handle.clone(),
                    overrides: WidgetOverrides.default(),
                    enabled:   true,
                    on_change: Box.new(|v| { log.debug("Checked: {}", v) }),
                }),
            }),
        }),
        Arc.new(Button {
            label:     "Continue".to_string(),
            theme:     theme_handle.clone(),
            style:     ButtonStyle.Filled,
            enabled:   true,
            overrides: WidgetOverrides.default(),
            on_tap:    Box.new(|| { handle_continue() }),
        }),
    ],
}

let root = ThemedWidget {
    theme: theme_handle,
    child: Arc.new(content),
}
```

---

## 12. Dependencies

### Direct Dependencies

| Extlib | Used for |
|---|---|
| `extlib.ccsp.draw` | `Color`, `Point`, `EdgeInsets` types |
| `extlib.ccsp.text_layout` | `TextStyle` (font_family, font_size, weight, letter_spacing, line_height) |
| `extlib.ccsp.anim` | `EasingCurve` (cubic Bezier, `evaluate(t)`) |
| `extlib.ccsp.widget` | `Widget` trait, `WidgetTree`, layout containers (`Column`, `Row`, `Padding`) |

### Stdlib Dependencies

| Module | Used for |
|---|---|
| `sync` | `Arc[T]`, `RwLock[T]` for `ThemeHandle` |
| `alloc` | `Vec[T]`, `String`, `Box[T]` |
| `core` | `Cell[T]`, `RefCell[T]` for mutable widget state |
| `math` | `math.max`, `math.min`, `math.powf` for contrast and color calculations |

### Dependency Rationale

`extlib.ccsp.style` is a leaf in the extlib dependency graph: nothing in `draw`, `text_layout`, `anim`, or `widget` depends on `style`. The dependency flows upward from primitives to the design system. This means a program can use the drawing layer or the animation system without pulling in the style token system, and conversely, a program can use only the token definitions and contrast tools without instantiating any widgets.

The style extlib does not depend on a rendering backend (`draw_wayland`, `draw_x11`, `draw_gpu`). Backend selection remains the caller's responsibility and does not affect token values or widget structure.
