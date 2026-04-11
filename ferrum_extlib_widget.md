# Ferrum Extended Library — widget

**Module:** `extlib.ccsp.widget`
**Spec basis:** CCSP `lib_ccsp_widget` Flutter layout model in C
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.draw`, `extlib.ccsp.text_layout`, `extlib.ccsp.font`,
`extlib.ccsp.input`, `extlib.ccsp.anim`, `extlib.ccsp.a11y`, `alloc`, `sync`

---

## Table of Contents

1. [Overview and Rationale](#1-overview-and-rationale)
2. [Core Types](#2-core-types)
3. [Widget Trait — the Five Methods](#3-widget-trait--the-five-methods)
4. [Layout Widgets](#4-layout-widgets)
5. [Scrollable](#5-scrollable)
6. [Leaf Widgets](#6-leaf-widgets)
7. [Gesture Detectors](#7-gesture-detectors)
8. [Dirty Region Tracking](#8-dirty-region-tracking)
9. [WidgetTree — the Runtime](#9-widgettree--the-runtime)
10. [Accessibility Integration](#10-accessibility-integration)
11. [State Management Pattern](#11-state-management-pattern)
12. [Error Types](#12-error-types)
13. [Example Usage](#13-example-usage)
14. [Dependencies Reference](#14-dependencies-reference)

---

## 1. Overview and Rationale

### The Flutter Layout Model Is Correct

GTK, Qt, and most classical widget toolkits implement layout as a two-pass
negotiation: preferred-size queries flow up the tree, then size grants flow
back down, then position assignments flow down again. Some toolkits add a
third pass. Others make the passes bidirectional. The result is layout
complexity that is `O(n²)` in common cases, and unbounded in degenerate ones.
A text widget embedded in a scrollable column inside a flex inside a window
triggers layout re-runs on every frame because each ancestor must re-query its
children to discover whether it has changed.

Flutter's layout model is different. It is also not new: it is a clean
statement of the algorithm that cassowary-based constraint solvers and earlier
UIML research converged on, stripped of everything that made those approaches
slow. The rule is:

> **Constraints flow down. Sizes flow up. Parents set positions.**

A parent passes a `Constraints` box to each child. The child measures itself
within those constraints and returns a `Size`. The parent then positions the
child — the child does not know its own origin. This one-way data flow is not
just elegant; it is provably O(n). Each widget's layout method is called at
most once per frame. There are no re-entrant queries, no mutual dependencies,
no cycles.

GTK's layout engine was rewritten three times trying to reach this property.
Qt's layout system requires explicit `invalidate()` calls and has well-known
performance cliffs with deeply nested layouts. This module starts from the
correct algorithm and does not inherit those problems.

### Composition Over Inheritance

Classical OO GUI toolkits make a widget responsible for its own appearance,
its own layout behavior, its own event handling, and its own accessibility —
all in one class. A `Button` inherits from `Label` which inherits from `Widget`
which inherits from `Object`. Adding a border requires subclassing or a flag.
Adding padding requires subclassing again. The inheritance hierarchy grows
until it is unmaintainable, and every new requirement forces a choice between
feature creep in the base class or an explosion of leaf subclasses.

Composition solves this. A `Button` in this module is:

```
GestureDetector {
    child: ColoredBox {
        child: Padded {
            child: TextWidget { ... }
        }
    }
}
```

Each piece does one thing. `Padded` adds insets. `ColoredBox` fills a
rectangle. `GestureDetector` routes tap events. Nothing inherits from anything.
The composition is visible in the data structure. Debugging is a tree walk.

### Accessibility Is First-Class, Not an Afterthought

Most toolkits add accessibility as an afterthought: a parallel tree of `ATSPI`
nodes maintained by hand, requiring the developer to remember to call
`set_accessible_name` after every structural change. When the developer forgets,
the tool is invisible to screen readers.

This module makes accessibility a mandatory part of the widget protocol. Every
widget implements `accessibility`, just as every widget implements `paint`.
Forgetting to implement accessibility produces a compile-time trait error, not
a runtime omission that only manifests when a blind user reports a bug. The
`WidgetTree` calls `accessibility` for every widget in every frame and syncs the
result to the platform's accessibility API via `extlib.ccsp.a11y`.

---

## 2. Core Types

The geometry types `Point`, `Size`, and `Rect` are re-exported from
`extlib.ccsp.draw`. All coordinates are in logical pixels (device-independent).
The draw backend maps logical to physical pixels using the display scale factor.

```ferrum
use extlib.ccsp.draw.{ Point, Size, Rect, Color, DrawContext, DrawBackend }
use extlib.ccsp.draw.DrawError
```

### 2.1 Constraints

`Constraints` is the box that a parent passes to a child. The child must return
a `Size` where `min_width <= width <= max_width` and
`min_height <= height <= max_height`. A constraint where `min == max` on both
axes is a tight constraint: the child's size is fully determined by the parent.
A constraint where `min == 0` and `max == f32::INFINITY` is a loose constraint:
the child can be any size.

```ferrum
/// Box constraints passed from parent to child during layout.
///
/// All values are in logical pixels. min values are >= 0.0.
/// max values are >= min values, or f32::INFINITY for unconstrained.
@derive(Debug, Clone, Copy, PartialEq)
pub type Constraints {
    pub min_width:  f32,
    pub max_width:  f32,
    pub min_height: f32,
    pub max_height: f32,
}

impl Constraints {
    /// Unconstrained: any size from zero to infinity is valid.
    pub fn unbounded(): Self {
        Constraints { min_width: 0.0, max_width: f32.INFINITY,
                      min_height: 0.0, max_height: f32.INFINITY }
    }

    /// Tight: the child must be exactly this size.
    pub fn tight(size: Size): Self {
        Constraints {
            min_width:  size.width,
            max_width:  size.width,
            min_height: size.height,
            max_height: size.height,
        }
    }

    /// Loose: the child may be at most this size, and at least zero.
    pub fn loose(max: Size): Self {
        Constraints { min_width: 0.0, max_width: max.width,
                      min_height: 0.0, max_height: max.height }
    }

    /// True if width is fixed (min == max).
    pub fn has_tight_width(&self): bool { self.min_width == self.max_width }

    /// True if height is fixed (min == max).
    pub fn has_tight_height(&self): bool { self.min_height == self.max_height }

    /// Clamp a size to satisfy these constraints.
    pub fn constrain(&self, size: Size): Size {
        Size {
            width:  size.width.max(self.min_width).min(self.max_width),
            height: size.height.max(self.min_height).min(self.max_height),
        }
    }

    /// Return a new Constraints with the max tightened to the given insets removed.
    pub fn deflate(&self, insets: EdgeInsets): Constraints {
        let dw = insets.left + insets.right
        let dh = insets.top  + insets.bottom
        Constraints {
            min_width:  (self.min_width  - dw).max(0.0),
            max_width:  (self.max_width  - dw).max(0.0),
            min_height: (self.min_height - dh).max(0.0),
            max_height: (self.max_height - dh).max(0.0),
        }
    }
}
```

### 2.2 WidgetId

Every widget in a live `WidgetTree` has a stable identity used for dirty
tracking and event routing.

```ferrum
/// Opaque stable identifier for a widget node within a WidgetTree.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub type WidgetId(u64)
```

### 2.3 HitTestEntry

`hit_test` returns an entry identifying the widget that should receive input
events at a given point.

```ferrum
/// The result of a hit test: the widget that owns a point.
@derive(Debug, Clone)]
pub type HitTestEntry {
    pub widget_id: WidgetId,
    /// Position of the hit point in the widget's own local coordinate space.
    pub local_point: Point,
}
```

---

## 3. Widget Trait — the Five Methods

```ferrum
use alloc.sync.Arc
use alloc.vec.Vec

/// The core trait every widget must implement.
///
/// The five methods map directly to the five phases of the per-frame pipeline:
///   layout → paint → hit_test → accessibility → children
///
/// Implementors must satisfy:
///   - layout must be called before paint in any given frame.
///   - paint receives the Size that layout returned.
///   - hit_test and accessibility also receive that same Size.
///   - children returns the live widget children for tree traversal.
pub trait Widget {
    /// Measure this widget given parent-imposed constraints.
    ///
    /// The returned Size must satisfy the constraints:
    ///   constraints.min_width <= size.width <= constraints.max_width
    ///   constraints.min_height <= size.height <= constraints.max_height
    ///
    /// This method may call ctx.layout_child to measure children.
    /// It must not paint, mutate state, or produce side effects.
    fn layout(&self, constraints: Constraints, ctx: &mut LayoutCtx): Size

    /// Paint this widget at the origin of ctx's local coordinate system.
    ///
    /// `size` is the Size returned by the most recent layout call.
    /// The widget should paint within Rect { origin: Point.origin(), size }.
    /// Child painting is handled by iterating children and calling their
    /// paint methods; the PaintCtx translates the coordinate system for each.
    fn paint(&self, size: Size, ctx: &mut PaintCtx)

    /// Return the widget (if any) that owns the given point.
    ///
    /// `point` is in this widget's local coordinate space.
    /// Return None if the point is not inside this widget.
    /// For container widgets, recurse into children and return the deepest hit.
    fn hit_test(&self, size: Size, point: Point): Option[HitTestEntry]

    /// Emit accessibility nodes for this widget.
    ///
    /// Called after paint on every frame. The `ctx` accumulates the
    /// accessibility tree that is synced to the platform's accessibility
    /// API. Leaf widgets with visible content must emit at least one node.
    fn accessibility(&self, size: Size, ctx: &mut A11yCtx)

    /// Return this widget's live children, in paint order (back to front).
    ///
    /// The WidgetTree uses this for tree traversal, dirty propagation,
    /// and hit testing. Return an empty Vec for leaf widgets.
    fn children(&self): Vec[Arc[dyn Widget]]
}
```

### 3.1 LayoutCtx

`LayoutCtx` is the handle through which a container widget measures its
children. It encapsulates the constraint-passing protocol and layout caching.

```ferrum
/// Context available to a widget's layout method.
///
/// Provides child measurement and layout caching.
pub type LayoutCtx { ... }

impl LayoutCtx {
    /// Lay out a child widget and return its measured size.
    ///
    /// The child is identified by its Arc pointer. The caller is responsible
    /// for passing constraints that the child can satisfy. The result is
    /// cached by (child_id, constraints); a second call with the same
    /// arguments returns the cached size without re-running the child's
    /// layout method.
    pub fn layout_child(
        &mut self,
        child: &Arc[dyn Widget],
        constraints: Constraints,
    ): Size ! IO

    /// Return the WidgetId assigned to a child by the tree.
    ///
    /// Only valid for widgets that are registered children of the current
    /// widget. Returns None for widgets not in the tree.
    pub fn child_id(&self, child: &Arc[dyn Widget]): Option[WidgetId]
}
```

### 3.2 PaintCtx

`PaintCtx` wraps a `DrawContext` with the widget-local coordinate system.
When a container paints a child, it calls `paint_child`, which translates
the coordinate origin so the child paints at `(0,0)` in its own space.

```ferrum
/// Context available to a widget's paint method.
///
/// Wraps a DrawContext with widget-local coordinates and damage tracking.
pub type PaintCtx { ... }

impl PaintCtx {
    /// The underlying draw context. Widget-local coordinates: origin is the
    /// widget's top-left corner.
    pub fn draw(&mut self): &mut DrawContext

    /// Paint a child widget at the given offset from this widget's origin.
    ///
    /// Translates the coordinate system, clips to the child's bounds if
    /// clip is true, calls child.paint, and restores the transform.
    pub fn paint_child(
        &mut self,
        child: &Arc[dyn Widget],
        offset: Point,
        size:   Size,
        clip:   bool,
    )

    /// Record a damaged region (in widget-local coordinates) for partial
    /// repaint. The rect is transformed to screen coordinates and unioned
    /// with the frame's dirty region.
    pub fn add_damage(&mut self, rect: Rect)
}
```

---

## 4. Layout Widgets

Layout widgets contain children and implement the constraint-passing protocol.
None of them have inherent visual appearance — they produce no pixels of their
own except where documented (e.g., `ColoredBox`).

### 4.1 Flex

Flex distributes children along a main axis, analogous to CSS flexbox.
Children with `flex > 0` share remaining space proportionally after
fixed-size children have been measured.

```ferrum
/// Linear layout along a main axis with optional flex distribution.
@derive(Debug)]
pub type Flex {
    pub direction:             Axis,
    pub children:              Vec[FlexChild],
    pub main_axis_alignment:   MainAxisAlignment,
    pub cross_axis_alignment:  CrossAxisAlignment,
    pub main_axis_size:        MainAxisSize,
}

/// A single child in a Flex layout.
@derive(Debug)]
pub type FlexChild {
    pub widget: Arc[dyn Widget],
    /// Flex factor. 0.0 means the child is sized by its intrinsic content.
    /// Values > 0.0 receive a share of remaining main-axis space proportional
    /// to this factor relative to the sum of all flex factors.
    pub flex:   f32,
    pub fit:    FlexFit,
}

/// How a flex child fills its flex-allocated space.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum FlexFit {
    /// The child must fill the allocated space exactly (tight constraint).
    Tight,
    /// The child may be smaller than the allocated space (loose constraint).
    Loose,
}

/// Layout axis.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum Axis {
    Horizontal,
    Vertical,
}

/// How children are distributed along the main axis after sizing.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum MainAxisAlignment {
    /// Pack children at the start of the main axis.
    Start,
    /// Pack children at the end.
    End,
    /// Center children along the main axis.
    Center,
    /// Place first and last children at the edges; distribute space evenly between others.
    SpaceBetween,
    /// Distribute equal space before and after each child.
    SpaceAround,
    /// Distribute equal space between children and at both ends.
    SpaceEvenly,
}

/// How children are aligned on the cross axis.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum CrossAxisAlignment {
    /// Align children to the start of the cross axis.
    Start,
    /// Align children to the end.
    End,
    /// Center children on the cross axis.
    Center,
    /// Stretch children to fill the cross axis (tight cross-axis constraint).
    Stretch,
    /// Align children by their text baseline (requires TextWidget children).
    Baseline,
}

/// Whether the main axis should expand to fill available space or shrink to content.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum MainAxisSize {
    /// The Flex takes as much main-axis space as the constraints allow.
    Max,
    /// The Flex shrinks to the minimum space needed by its children.
    Min,
}
```

**Layout algorithm:**

1. Lay out all non-flex children (`flex == 0`) with unbounded main-axis and
   the cross-axis constraint from the parent. Sum their main-axis extents.
2. Divide remaining main-axis space among flex children according to their
   `flex` factors.
3. Lay out flex children with their allocated main-axis size (tight if `Tight`,
   loose if `Loose`) and the cross-axis constraint.
4. Compute the Flex's own size from `MainAxisSize` and `CrossAxisAlignment`.
5. Position children according to `MainAxisAlignment` and `CrossAxisAlignment`.

### 4.2 Stack

Stack renders children in z-order (first child at the back, last at the
front). Children can be positioned absolutely within the stack's bounds or
left unpositioned, in which case they are aligned by the `alignment` field.

```ferrum
/// Z-order layering of children.
@derive(Debug)]
pub type Stack {
    pub children:  Vec[StackChild],
    pub alignment: Alignment,
    pub fit:       StackFit,
}

/// A single child in a Stack.
@derive(Debug)]
pub type StackChild {
    pub widget:     Arc[dyn Widget],
    /// When Some, this child is positioned absolutely. When None, the child
    /// is an "unpositioned" child sized and aligned by the stack's rules.
    pub positioned: Option[Positioned],
}

/// Absolute positioning within a Stack.
///
/// At most one of each axis pair (top/bottom, left/right) should be set.
/// If both are set, width/height is ignored and the child fills between them.
@derive(Debug, Clone, Copy)]
pub type Positioned {
    pub top:    Option[f32],
    pub right:  Option[f32],
    pub bottom: Option[f32],
    pub left:   Option[f32],
    pub width:  Option[f32],
    pub height: Option[f32],
}

/// How the Stack sizes itself relative to its unpositioned children.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum StackFit {
    /// The Stack is sized to the largest unpositioned child.
    Loose,
    /// The Stack expands to fill the parent constraints exactly.
    Expand,
    /// The Stack passes through the parent's constraints to unpositioned children.
    Passthrough,
}
```

### 4.3 Constrained

Wraps a child with additional constraints. The child sees the intersection of
the parent's constraints and the ones specified here.

```ferrum
/// Apply additional constraints to a child.
@derive(Debug)]
pub type Constrained {
    pub constraints: Constraints,
    pub child:       Arc[dyn Widget],
}
```

The effective constraints are computed by intersecting: for each bound, the
tighter of the outer and inner constraint wins. `min` is the max of both
`min` values; `max` is the min of both `max` values.

### 4.4 Padded

Insets a child by `padding` on all sides. The child sees reduced constraints;
the `Padded` widget reports a size equal to the child's size plus the padding.

```ferrum
/// Add padding (insets) around a child.
@derive(Debug)]
pub type Padded {
    pub padding: EdgeInsets,
    pub child:   Arc[dyn Widget],
}

/// Axis-aligned insets.
@derive(Debug, Clone, Copy, PartialEq)]
pub type EdgeInsets {
    pub top:    f32,
    pub right:  f32,
    pub bottom: f32,
    pub left:   f32,
}

impl EdgeInsets {
    /// Equal insets on all sides.
    pub fn all(v: f32): Self {
        EdgeInsets { top: v, right: v, bottom: v, left: v }
    }

    /// Separate horizontal and vertical insets.
    /// `h` is applied to left and right; `v` to top and bottom.
    pub fn symmetric(h: f32, v: f32): Self {
        EdgeInsets { top: v, right: h, bottom: v, left: h }
    }

    /// Specify any subset of sides; unspecified sides default to 0.0.
    pub fn only(top: f32, right: f32, bottom: f32, left: f32): Self {
        EdgeInsets { top, right, bottom, left }
    }

    pub fn horizontal(&self): f32 { self.left + self.right }
    pub fn vertical(&self):   f32 { self.top  + self.bottom }
}
```

### 4.5 Expanded

Used inside a `Flex` to fill remaining space. `Expanded` is syntactic sugar
for `FlexChild { flex: n, fit: FlexFit.Tight, ... }` in the common case where
you do not want to construct a `FlexChild` manually. Outside a `Flex`, it
fills all available parent space.

```ferrum
/// Fill remaining flex space in a Flex, or all parent space outside one.
@derive(Debug)]
pub type Expanded {
    /// Flex factor. Must be > 0.0. Defaults to 1.0.
    pub flex:  f32,
    pub child: Arc[dyn Widget],
}
```

### 4.6 Aligned

Positions a child within its allocated space according to an `Alignment`.
The `Aligned` widget sizes itself to the maximum available space (or to the
child's size if unconstrained) and places the child at the given alignment.

```ferrum
/// Position a child within allocated space by alignment.
@derive(Debug)]
pub type Aligned {
    pub alignment: Alignment,
    pub child:     Arc[dyn Widget],
}

/// Normalized 2D alignment.
///
/// (x=-1, y=-1) is the top-left corner.
/// (x= 0, y= 0) is the center.
/// (x= 1, y= 1) is the bottom-right corner.
/// Intermediate values interpolate linearly.
@derive(Debug, Clone, Copy, PartialEq)]
pub type Alignment {
    pub x: f32,
    pub y: f32,
}

impl Alignment {
    pub const TOP_LEFT:      Alignment = Alignment { x: -1.0, y: -1.0 }
    pub const TOP_CENTER:    Alignment = Alignment { x:  0.0, y: -1.0 }
    pub const TOP_RIGHT:     Alignment = Alignment { x:  1.0, y: -1.0 }
    pub const CENTER_LEFT:   Alignment = Alignment { x: -1.0, y:  0.0 }
    pub const CENTER:        Alignment = Alignment { x:  0.0, y:  0.0 }
    pub const CENTER_RIGHT:  Alignment = Alignment { x:  1.0, y:  0.0 }
    pub const BOTTOM_LEFT:   Alignment = Alignment { x: -1.0, y:  1.0 }
    pub const BOTTOM_CENTER: Alignment = Alignment { x:  0.0, y:  1.0 }
    pub const BOTTOM_RIGHT:  Alignment = Alignment { x:  1.0, y:  1.0 }

    /// Compute the offset of a child of `child_size` within a box of `container_size`.
    pub fn resolve(&self, container_size: Size, child_size: Size): Point {
        Point {
            x: (container_size.width  - child_size.width)  * (self.x + 1.0) / 2.0,
            y: (container_size.height - child_size.height) * (self.y + 1.0) / 2.0,
        }
    }
}
```

### 4.7 SizedBox

Forces a specific width and/or height. Fields set to `None` pass through
the parent's constraints unchanged. When `child` is `None`, `SizedBox`
creates empty space (a spacer).

```ferrum
/// Force a specific size, or create empty space.
@derive(Debug)]
pub type SizedBox {
    /// If Some, the child (or empty box) is constrained to exactly this width.
    pub width:  Option[f32],
    /// If Some, the child (or empty box) is constrained to exactly this height.
    pub height: Option[f32],
    pub child:  Option[Arc[dyn Widget]],
}

impl SizedBox {
    /// A spacer with fixed width and no height constraint.
    pub fn width(w: f32): Self {
        SizedBox { width: Some(w), height: None, child: None }
    }

    /// A spacer with fixed height and no width constraint.
    pub fn height(h: f32): Self {
        SizedBox { width: None, height: Some(h), child: None }
    }

    /// A spacer with fixed width and height.
    pub fn square(side: f32): Self {
        SizedBox { width: Some(side), height: Some(side), child: None }
    }
}
```

### 4.8 AspectRatio

Constrains a child to maintain a specific width-to-height ratio. Given a
width from the parent, the height is computed as `width / ratio`. If that
height falls outside the parent's height constraints, the height is clamped
and the width is recomputed.

```ferrum
/// Maintain a fixed width/height aspect ratio.
@derive(Debug)]
pub type AspectRatio {
    /// Width divided by height. For example, 16.0/9.0 for widescreen.
    pub ratio: f32,
    pub child: Arc[dyn Widget],
}
```

### 4.9 Wrap

Flow layout: children are placed sequentially along the main axis and wrap
to a new run when they would exceed the available space. Analogous to
CSS `flex-wrap: wrap`.

```ferrum
/// Flow layout with wrapping runs.
@derive(Debug)]
pub type Wrap {
    pub direction:   Axis,
    /// Space between children on the main axis within a run.
    pub spacing:     f32,
    /// Space between runs on the cross axis.
    pub run_spacing: f32,
    pub alignment:   WrapAlignment,
    pub children:    Vec[Arc[dyn Widget]],
}

/// How runs are aligned on the cross axis.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum WrapAlignment {
    Start,
    End,
    Center,
    SpaceBetween,
    SpaceAround,
    SpaceEvenly,
}
```

---

## 5. Scrollable

`Scrollable` clips its child to its own bounds and offsets it along the
scroll axis by the controller's current offset. The child is laid out with
an unbounded constraint on the scroll axis and the parent's constraint on the
cross axis, allowing it to be larger than the viewport.

```ferrum
/// A scrollable viewport over a single child.
@derive(Debug)]
pub type Scrollable {
    pub axis:       Axis,
    pub controller: Arc[RefCell[ScrollController]],
    pub child:      Arc[dyn Widget],
    pub style:      ScrollbarStyle,
}

/// Runtime scroll position state.
///
/// This is not a widget — it is the mutable state that lives outside the
/// widget tree and drives scroll position.
pub type ScrollController {
    /// Current scroll offset in logical pixels along the scroll axis.
    pub offset:            f32,
    /// Maximum valid offset, set by WidgetTree after child layout.
    pub max_scroll_extent: f32,
}

impl ScrollController {
    pub fn new(): Self {
        ScrollController { offset: 0.0, max_scroll_extent: 0.0 }
    }

    /// Scroll to an absolute offset. Clamped to [0, max_scroll_extent].
    pub fn scroll_to(&mut self, offset: f32) {
        self.offset = offset.max(0.0).min(self.max_scroll_extent)
    }

    /// Scroll by a relative delta. Clamped.
    pub fn scroll_by(&mut self, delta: f32) {
        self.scroll_to(self.offset + delta)
    }

    /// True if the viewport is scrolled to the start.
    pub fn at_start(&self): bool { self.offset <= 0.0 }

    /// True if the viewport is scrolled to the end.
    pub fn at_end(&self): bool { self.offset >= self.max_scroll_extent }
}

/// Visual style for the scrollbar overlay.
@derive(Debug, Clone, Copy)]
pub type ScrollbarStyle {
    /// Width of the scrollbar track and thumb in logical pixels.
    pub width:       f32,
    pub thumb_color: Color,
    pub track_color: Color,
}

impl ScrollbarStyle {
    pub fn default(): Self {
        ScrollbarStyle {
            width:       8.0,
            thumb_color: Color.from_hex(0x00000080),
            track_color: Color.from_hex(0x00000020),
        }
    }
}
```

**Layout:**

1. Lay out the child with tight cross-axis constraint and unbounded main-axis
   constraint.
2. Record `child_main_extent - own_main_extent` as `max_scroll_extent`.
3. Update `controller.max_scroll_extent`.
4. The `Scrollable` sizes itself to the parent's constraint.

**Paint:**

1. Clip to own bounds.
2. Translate by `-controller.offset` along the scroll axis.
3. Call `paint_child`.
4. Restore clip.
5. If `max_scroll_extent > 0`, paint scrollbar overlay.

---

## 6. Leaf Widgets

Leaf widgets render content and have no children.

### 6.1 TextWidget

Renders shaped text using `extlib.ccsp.text_layout`. The text has already
been shaped — `TextWidget` renders glyph runs, not raw Unicode strings.
The `Shaper` is an `Arc`-shared reference to a shaping engine; the same
shaper instance can be shared across many `TextWidget` instances.

```ferrum
use extlib.ccsp.text_layout.{ AttributedString, ParagraphStyle, ParagraphLayout, Shaper }

/// Render shaped text.
@derive(Debug)]
pub type TextWidget {
    pub text:   AttributedString,
    pub style:  ParagraphStyle,
    pub shaper: Arc[Shaper],
}
```

**Layout:** Shape the text into a `ParagraphLayout` using the max-width
constraint. Return the resulting ink bounds as the size, clamped to
constraints.

**Paint:** Call `ctx.draw().draw_paragraph(layout, Point.origin())`.

### 6.2 Image

Renders an image with a specified fit mode and alignment within the allocated
space.

```ferrum
use extlib.ccsp.draw.ImageRef

/// Render an image with a fit mode.
@derive(Debug)]
pub type Image {
    pub data:      ImageRef,
    pub fit:       ImageFit,
    pub alignment: Alignment,
}

/// How an image is sized and cropped within its allocated space.
@derive(Debug, Clone, Copy, PartialEq)]
pub enum ImageFit {
    /// Stretch the image to fill the box exactly, ignoring aspect ratio.
    Fill,
    /// Scale the image to fit entirely within the box, preserving aspect ratio.
    /// May letterbox.
    Contain,
    /// Scale the image to cover the entire box, preserving aspect ratio.
    /// May crop.
    Cover,
    /// Scale the image to match the box width exactly. Height may overflow.
    FitWidth,
    /// Scale the image to match the box height exactly. Width may overflow.
    FitHeight,
    /// No scaling. Render at the image's natural size.
    None,
}
```

### 6.3 ColoredBox

Fills a rectangle with a solid color. When `child` is `Some`, the child is
laid out with the same constraints and painted on top.

```ferrum
/// A filled colored rectangle, optionally containing a child.
@derive(Debug)]
pub type ColoredBox {
    pub color: Color,
    pub child: Option[Arc[dyn Widget]],
}
```

### 6.4 CustomPaint

An escape hatch for drawing that does not fit the widget model. The painter
closure receives a `DrawContext` and the widget's size, and may issue
arbitrary draw calls. When `child` is `Some`, the child is painted after the
painter closure returns.

```ferrum
/// Custom drawing via a closure. Use sparingly; prefer composition.
pub type CustomPaint {
    pub painter: Box[dyn Fn(&mut DrawContext, Size)],
    pub child:   Option[Arc[dyn Widget]],
}
```

`CustomPaint` does not derive `Debug` because closures are not `Debug`.

---

## 7. Gesture Detectors

`GestureDetector` wraps any widget and intercepts input events that hit-test
to it or any of its descendants. Gesture recognizers run in a gesture arena:
when multiple recognizers claim the same event sequence, the arena resolves
the conflict by eliminating all but the highest-priority winner.

```ferrum
/// Wrap any widget with gesture recognition.
pub type GestureDetector {
    pub child:    Arc[dyn Widget],
    pub gestures: GestureRecognizers,
}
```

### 7.1 GestureRecognizers

```ferrum
/// Set of gesture callbacks. All fields are optional.
///
/// Unset callbacks are not registered in the gesture arena; the arena only
/// arbitrates between actually competing recognizers.
pub type GestureRecognizers {
    pub on_tap:          Option[Box[dyn Fn(TapDetails)]],
    pub on_double_tap:   Option[Box[dyn Fn(TapDetails)]],
    pub on_long_press:   Option[Box[dyn Fn(TapDetails)]],
    pub on_drag_start:   Option[Box[dyn Fn(DragStartDetails)]],
    pub on_drag_update:  Option[Box[dyn Fn(DragUpdateDetails)]],
    pub on_drag_end:     Option[Box[dyn Fn(DragEndDetails)]],
    pub on_hover:        Option[Box[dyn Fn(HoverDetails)]],
    pub on_scale_start:  Option[Box[dyn Fn(ScaleStartDetails)]],
    pub on_scale_update: Option[Box[dyn Fn(ScaleUpdateDetails)]],
    pub on_scale_end:    Option[Box[dyn Fn(ScaleEndDetails)]],
}

impl GestureRecognizers {
    /// Empty set: no gestures recognized.
    pub fn none(): Self {
        GestureRecognizers {
            on_tap:          None,
            on_double_tap:   None,
            on_long_press:   None,
            on_drag_start:   None,
            on_drag_update:  None,
            on_drag_end:     None,
            on_hover:        None,
            on_scale_start:  None,
            on_scale_update: None,
            on_scale_end:    None,
        }
    }
}
```

### 7.2 Detail Types

```ferrum
/// A completed tap or double-tap or long-press.
@derive(Debug, Clone, Copy)]
pub type TapDetails {
    /// Position in the widget's local coordinate space.
    pub position:        Point,
    /// Position in the root widget's coordinate space (screen-local).
    pub global_position: Point,
}

/// Drag sequence started.
@derive(Debug, Clone, Copy)]
pub type DragStartDetails {
    pub position:        Point,
    pub global_position: Point,
}

/// Drag moved.
@derive(Debug, Clone, Copy)]
pub type DragUpdateDetails {
    /// Motion delta since the last update, in logical pixels.
    pub delta:           Point,
    pub position:        Point,
    /// Instantaneous velocity estimate in logical pixels per second.
    pub velocity:        Point,
}

/// Drag released.
@derive(Debug, Clone, Copy)]
pub type DragEndDetails {
    /// Final velocity at release, in logical pixels per second.
    pub velocity: Point,
}

/// Pointer entered, moved within, or is hovering over the widget.
@derive(Debug, Clone, Copy)]
pub type HoverDetails {
    pub position:        Point,
    pub global_position: Point,
}

/// Scale/pinch gesture started.
@derive(Debug, Clone, Copy)]
pub type ScaleStartDetails {
    pub focal_point: Point,
}

/// Scale/pinch gesture updated.
@derive(Debug, Clone, Copy)]
pub type ScaleUpdateDetails {
    /// Current scale factor relative to the gesture start (1.0 = no change).
    pub scale:       f32,
    /// Rotation in radians relative to the gesture start.
    pub rotation:    f32,
    pub focal_point: Point,
}

/// Scale/pinch gesture ended.
@derive(Debug, Clone, Copy)]
pub type ScaleEndDetails {
    pub velocity: Point,
}
```

### 7.3 Gesture Arena

When an input event is delivered to a widget with multiple competing
recognizers — for example, a widget that handles both `on_tap` and
`on_drag_start` — the arena holds all candidates and resolves the conflict
when enough of the event sequence has arrived to disambiguate. The rules are:

- **Single candidate:** if only one recognizer is in the arena for an event
  sequence, it wins immediately.
- **Pointer up before threshold:** a tap wins over a drag if the pointer is
  released before the drag-start distance threshold (4 logical pixels by
  default).
- **Threshold exceeded:** a drag wins over a tap if the pointer moves beyond
  the threshold before being released.
- **Long press:** a long press wins if neither a tap nor a drag win within the
  long-press duration (500 ms by default).
- **Scale vs drag:** scale wins when two or more pointers are active.

Callers do not configure the arena. The resolution rules are fixed and
predictable.

---

## 8. Dirty Region Tracking

The widget system avoids full-tree relayout and full-screen repaints on every
frame. Dirty tracking limits work to the widgets that actually changed.

### 8.1 Marking Dirty

```ferrum
impl WidgetTree {
    /// Mark a widget as needing relayout and repaint.
    ///
    /// Propagates the dirty flag up the ancestor chain to the root, so the
    /// tree knows a layout pass is required. Descendants are implicitly dirty
    /// because their parent will re-layout them.
    pub fn mark_dirty(&mut self, id: WidgetId)

    /// True if any widget in the tree has been marked dirty since the last
    /// successful frame() call.
    pub fn needs_layout(&self): bool

    /// True if any widget has accumulated paint damage since the last frame.
    pub fn needs_paint(&self): bool
}
```

### 8.2 Dirty Propagation

Calling `mark_dirty(id)` sets a flag on the identified widget and on every
ancestor up to the root. The flag on ancestors is a "layout required"
signal: when `frame()` runs its layout pass, it descends through the tree and
only re-runs layout on subtrees that are flagged dirty. Siblings that are not
flagged use their cached sizes from the previous frame.

The propagation is conservative in the upward direction: marking a leaf dirty
necessarily marks its parent dirty (the parent's layout may depend on the
child's new size), which marks its grandparent dirty, and so on. In the
downward direction, only the flagged subtree is re-run; unflagged siblings are
skipped.

### 8.3 Paint Damage

```ferrum
impl PaintCtx {
    /// Add a damaged rect to the current frame's dirty region.
    ///
    /// The rect is in the widget's local coordinate space. It is transformed
    /// to screen (physical pixel) coordinates and unioned with the frame's
    /// accumulated damage region before being sent to the draw backend.
    pub fn add_damage(&mut self, rect: Rect)
}
```

The draw backend's `present_damage` method receives the union of all damaged
rects accumulated during the paint pass and repaints only those regions on
screen. Widgets that have not changed do not need to call `add_damage`, and
their pixels are not touched.

---

## 9. WidgetTree — the Runtime

`WidgetTree` owns the widget tree, drives the per-frame pipeline, and routes
input events to hit-tested widgets.

```ferrum
use extlib.ccsp.input.{ InputEvent, EventResponse }

/// The widget tree runtime.
///
/// Owns the root widget, the draw backend, and all per-frame state.
/// One WidgetTree per window.
pub type WidgetTree { ... }

impl WidgetTree {
    /// Construct a WidgetTree with a root widget and a draw backend.
    ///
    /// The backend determines where frames are rendered: Wayland surface,
    /// X11 window, framebuffer, image file, etc.
    pub fn new(
        root:    Arc[dyn Widget],
        backend: Box[dyn DrawBackend],
    ): Self ! IO

    /// Replace the root widget and schedule a full relayout on the next frame.
    pub fn set_root(&mut self, widget: Arc[dyn Widget])

    /// Notify the tree that the window has been resized.
    ///
    /// Schedules a full relayout on the next frame.
    pub fn set_size(&mut self, size: Size)

    /// Run one frame:
    ///   1. If needs_layout: run layout pass on dirty subtrees.
    ///   2. If needs_paint:  run paint pass on damaged widgets.
    ///   3. If any damage:   call backend.present_damage(damage_rect).
    ///   4. Sync accessibility tree to platform via extlib.ccsp.a11y.
    ///
    /// Returns Ok(()) if the frame was produced successfully.
    /// Returns Err if layout overflowed, a paint call failed, or the backend
    /// rejected the frame.
    pub fn frame(&mut self): Result[(), WidgetError] ! IO

    /// Deliver an input event (keyboard, pointer, scroll wheel, touch) to the
    /// widget tree.
    ///
    /// Hit-tests the event position against the current widget tree to find
    /// the target widget. Routes the event to that widget's GestureDetector
    /// (if any). Returns whether the event was consumed.
    pub fn deliver_event(&mut self, event: InputEvent): EventResponse

    /// Return the WidgetId of a widget known to be in the tree.
    ///
    /// Used to obtain an id for mark_dirty calls. Returns None if the widget
    /// is not currently in the tree.
    pub fn widget_id(&self, widget: &Arc[dyn Widget]): Option[WidgetId]

    /// Mark a widget as dirty. See Section 8.
    pub fn mark_dirty(&mut self, id: WidgetId)

    /// True if a layout pass is pending.
    pub fn needs_layout(&self): bool

    /// True if paint damage has accumulated.
    pub fn needs_paint(&self): bool
}
```

### Frame Pipeline

```
frame() call
  │
  ├── layout pass (if needs_layout)
  │     walk tree depth-first
  │     for each dirty node: call widget.layout(constraints, ctx)
  │     cache sizes; clear dirty flags
  │
  ├── paint pass (if needs_paint)
  │     walk tree depth-first in paint order
  │     for each widget: call widget.paint(size, ctx)
  │     PaintCtx accumulates damage rects
  │
  ├── present (if any damage)
  │     backend.present_damage(union_of_damage_rects)
  │
  └── accessibility pass
        walk tree depth-first
        for each widget: call widget.accessibility(size, a11y_ctx)
        a11y_ctx.flush() — sync to platform via extlib.ccsp.a11y
```

---

## 10. Accessibility Integration

Every widget implements `accessibility`. The `WidgetTree` calls it after each
paint pass. The resulting accessibility tree is synced to the platform
accessibility API (AT-SPI2 on Linux, UIAccessibility on macOS, UIA on Windows)
via `extlib.ccsp.a11y`.

```ferrum
use extlib.ccsp.a11y.{ A11yRole, A11yAction }

/// Context for building the accessibility tree during widget traversal.
///
/// Each call to node, set_*, and add_action contributes to the node currently
/// being built. The tree is assembled from the order of calls across the
/// widget traversal.
pub type A11yCtx { ... }

impl A11yCtx {
    /// Add an accessibility node for the current widget.
    ///
    /// `role` is the semantic role: Button, TextField, Image, List, etc.
    /// `label` is the accessible name read by screen readers.
    /// Must be called before set_* or add_action for this widget.
    pub fn node(&mut self, role: A11yRole, label: &str)

    /// Set the accessible value (e.g., current text in a field, slider value).
    pub fn set_value(&mut self, value: &str)

    /// Set the checked state (for checkboxes, radio buttons, toggles).
    pub fn set_checked(&mut self, checked: bool)

    /// Set whether the widget is enabled (interactable).
    pub fn set_enabled(&mut self, enabled: bool)

    /// Set whether this widget currently has keyboard focus.
    pub fn set_focused(&mut self, focused: bool)

    /// Register an action that assistive technology can invoke.
    ///
    /// `Click` triggers the widget's primary action.
    /// `Focus` moves keyboard focus to the widget.
    /// `ScrollIntoView` scrolls the widget into the viewport.
    pub fn add_action(&mut self, action: A11yAction, callback: Box[dyn Fn()])
}
```

The accessibility tree parallels the widget tree. Container widgets that
do not have semantic meaning (e.g., `Padded`, `Flex`, `Stack`) typically emit
no accessibility node and allow the traversal to descend to their children.
Leaf widgets and interactive widgets must emit at least one node.

---

## 11. State Management Pattern

Widgets in this module are immutable value types, or are shared via `Arc`.
They hold no mutable state. State lives outside the widget tree in
caller-managed cells.

This is a deliberate design choice. A built-in reactive system (like Flutter's
`StatefulWidget` or React's `useState`) adds substantial complexity: a
lifecycle protocol, a rebuild protocol, and a scheduler. The benefit is
convenience. The cost is that the system now controls the memory model,
forcing all state into framework-managed boxes. For a systems language with
explicit ownership, this is the wrong trade.

The pattern is:

```ferrum
use alloc.sync.{ Arc, Mutex }
use alloc.cell.RefCell

// 1. Define your state as a plain struct.
type CounterState {
    count: i32,
}

// 2. Wrap it in Arc<RefCell> (single-threaded) or Arc<Mutex> (multi-threaded)
//    so it can be shared between the widget builder and the event handlers.
let state = Arc.new(RefCell.new(CounterState { count: 0 }))

// 3. Capture a clone of the Arc in event handlers.
let state_for_tap = state.clone()
let tree_ref      = tree_weak_ref.clone()

let button = GestureDetector {
    child: ...,
    gestures: GestureRecognizers {
        on_tap: Some(Box.new(move |_details| {
            state_for_tap.borrow_mut().count += 1
            if let Some(tree) = tree_ref.upgrade() {
                tree.borrow_mut().mark_dirty(button_id)
            }
        })),
        ..GestureRecognizers.none()
    },
}

// 4. Widget builder reads state to construct widgets on each frame.
//    WidgetTree calls frame() which calls layout and paint.
//    Because button_id was marked dirty, the button subtree is re-laid-out
//    and repainted with the updated state.
```

`WidgetTree::mark_dirty` is the mechanism by which external state changes
schedule repaints. Call it from any event handler that mutates state. The next
`frame()` call will re-layout the dirty subtree and repaint damaged regions.

This is KISS. State is a variable. Sharing is an `Arc`. Scheduling is
`mark_dirty`. No framework magic.

---

## 12. Error Types

```ferrum
/// Errors produced by the widget system.
@derive(Debug)]
pub enum WidgetError {
    /// A widget's layout method returned a size that violates the constraints
    /// passed to it. This is a widget implementation bug.
    LayoutOverflow {
        /// The type name of the offending widget (from type_name::<W>()).
        widget_type:     &'static str,
        /// The constraints the widget received.
        constraints:     Constraints,
        /// The size the widget returned, which violated those constraints.
        computed_size:   Size,
    },

    /// A draw operation failed (backend out of memory, device lost, etc.).
    DrawError(DrawError),

    /// An accessibility tree operation failed.
    A11yError(String),
}

impl From[DrawError] for WidgetError {
    fn from(e: DrawError): WidgetError { WidgetError.DrawError(e) }
}
```

---

## 13. Example Usage

### 13.1 Button with Tap Handler

```ferrum
use extlib.ccsp.widget.*
use extlib.ccsp.draw.Color
use alloc.sync.Arc
use alloc.cell.RefCell

type AppState {
    press_count: u32,
}

fn build_button(
    state: Arc[RefCell[AppState]],
    tree:  Arc[RefCell[WidgetTree]],
): Arc[dyn Widget] {
    let state_ref = state.clone()
    let tree_ref  = tree.clone()

    let label = Arc.new(TextWidget {
        text:   AttributedString.from("Click me"),
        style:  ParagraphStyle.default(),
        shaper: shaper.clone(),
    })

    let padded = Arc.new(Padded {
        padding: EdgeInsets.symmetric(16.0, 8.0),
        child:   label,
    })

    let background = Arc.new(ColoredBox {
        color: Color.from_hex(0x2196F3FF),
        child: Some(padded),
    })

    Arc.new(GestureDetector {
        child:    background,
        gestures: GestureRecognizers {
            on_tap: Some(Box.new(move |_details| {
                state_ref.borrow_mut().press_count += 1
                let mut t = tree_ref.borrow_mut()
                // mark_dirty triggers repaint on next frame()
                if let Some(id) = t.widget_id(&background_arc) {
                    t.mark_dirty(id)
                }
            })),
            ..GestureRecognizers.none()
        },
    })
}
```

### 13.2 Scrollable List

```ferrum
fn build_list(items: &[String]): Arc[dyn Widget] {
    let controller = Arc.new(RefCell.new(ScrollController.new()))

    let rows: Vec[Arc[dyn Widget]] = items
        .iter()
        .map(|item| -> Arc[dyn Widget] {
            Arc.new(Padded {
                padding: EdgeInsets.symmetric(12.0, 8.0),
                child: Arc.new(TextWidget {
                    text:   AttributedString.from(item),
                    style:  ParagraphStyle.default(),
                    shaper: shaper.clone(),
                }),
            })
        })
        .collect()

    let column = Arc.new(Flex {
        direction:            Axis.Vertical,
        children:             rows.into_iter().map(|w| FlexChild {
            widget: w,
            flex:   0.0,
            fit:    FlexFit.Loose,
        }).collect(),
        main_axis_alignment:  MainAxisAlignment.Start,
        cross_axis_alignment: CrossAxisAlignment.Stretch,
        main_axis_size:       MainAxisSize.Min,
    })

    Arc.new(Scrollable {
        axis:       Axis.Vertical,
        controller,
        child:      column,
        style:      ScrollbarStyle.default(),
    })
}
```

### 13.3 Text Input Field with Cursor

Text input is a `CustomPaint` leaf plus a `GestureDetector` to handle focus
and tap-to-position. The cursor position is external state.

```ferrum
type TextFieldState {
    text:        String,
    cursor:      usize,     // byte offset into text
    focused:     bool,
}

fn build_text_field(
    state: Arc[RefCell[TextFieldState]],
    tree:  Arc[RefCell[WidgetTree]],
    id:    WidgetId,
): Arc[dyn Widget] {
    let state_paint = state.clone()
    let state_tap   = state.clone()
    let tree_ref    = tree.clone()

    let painter = Arc.new(CustomPaint {
        painter: Box.new(move |ctx: &mut DrawContext, size: Size| {
            let s = state_paint.borrow()
            let bg = if s.focused {
                Color.from_hex(0xFFFFFFFF)
            } else {
                Color.from_hex(0xF5F5F5FF)
            }
            ctx.fill_rect(Rect.new(0.0, 0.0, size.width, size.height), bg.into())
            // draw text and cursor via text layout (elided for brevity)
        }),
        child: None,
    })

    Arc.new(GestureDetector {
        child:    painter,
        gestures: GestureRecognizers {
            on_tap: Some(Box.new(move |details| {
                state_tap.borrow_mut().focused = true
                tree_ref.borrow_mut().mark_dirty(id)
            })),
            ..GestureRecognizers.none()
        },
    })
}
```

### 13.4 Flex Row with Spacer

```ferrum
fn build_toolbar(
    left:  Arc[dyn Widget],
    right: Arc[dyn Widget],
): Arc[dyn Widget] {
    Arc.new(Flex {
        direction:            Axis.Horizontal,
        children:             vec![
            FlexChild { widget: left,  flex: 0.0, fit: FlexFit.Loose },
            FlexChild {
                widget: Arc.new(SizedBox { width: None, height: None, child: None }),
                flex:   1.0,
                fit:    FlexFit.Tight,
            },
            FlexChild { widget: right, flex: 0.0, fit: FlexFit.Loose },
        ],
        main_axis_alignment:  MainAxisAlignment.Start,
        cross_axis_alignment: CrossAxisAlignment.Center,
        main_axis_size:       MainAxisSize.Max,
    })
}
```

---

## 14. Dependencies Reference

| Dependency | Module path | Used for |
|---|---|---|
| draw | `extlib.ccsp.draw` | `DrawContext`, `DrawBackend`, `Color`, `Point`, `Rect`, `Size`, `DrawError` |
| text_layout | `extlib.ccsp.text_layout` | `AttributedString`, `ParagraphStyle`, `ParagraphLayout`, `Shaper` |
| font | `extlib.ccsp.font` | Font loading; passed to `Shaper` |
| input | `extlib.ccsp.input` | `InputEvent`, `EventResponse` |
| anim | `extlib.ccsp.anim` | Animated property interpolation; used by widgets that animate state transitions |
| a11y | `extlib.ccsp.a11y` | `A11yRole`, `A11yAction`, platform accessibility tree sync |
| alloc | `std.alloc` | `Arc`, `Box`, `Vec`, `String`, `RefCell` |
| sync | `std.sync` | `Mutex` for multi-threaded state cells |
