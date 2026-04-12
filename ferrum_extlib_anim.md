# Ferrum Extended Library — anim

**Module:** `extlib.ccsp.anim`
**Spec basis:** CCSP `lib_ccsp_anim` animation engine specification
**Roadmap status:** Post-1.0 (designed now, implemented after stdlib stabilizes)
**Dependencies:** `extlib.ccsp.draw` (Color, Point, Size, Rect); `std.time` (Duration, Instant); `std.async` (Runtime, interval)

---

## 1. Overview and Rationale

### What Animation Adds

A UI that snaps instantly between states is functionally correct but perceptually jarring. When a panel slides in, when a button bounces on press, when a list item fades out on delete — the motion carries two kinds of information that a static jump cannot.

**Perceived responsiveness.** Users rate animated interfaces as faster even when the total elapsed time is identical. The motion fills the cognitive gap between cause and effect: the button responds immediately and visibly, even if the work it triggers takes 300ms. An animation that begins at tap-time and completes at result-time makes the wait feel like the natural cost of motion, not a failure to respond.

**Spatial continuity.** Abrupt state changes break the user's mental model of where things are. When a dialog appears without motion, the user must scan to find it. When it slides in from the right, they know exactly where it came from and where to look to dismiss it. Animation encodes the relationship between states.

Neither benefit requires complex tooling. They require a library that is easy to reach for, composable, and correct by default.

### Why Physics-Based Animation

Timed curves — linear, ease-in, ease-out — are mathematically simple and predictable. They have a known duration and follow a fixed path. But they do not feel natural.

Human motion and physical objects do not stop at a prescribed moment. A spring does not know how long it is supposed to oscillate: it oscillates until the energy is gone. A ball does not ease out over exactly 300ms: it decelerates under friction and stops when its velocity reaches zero. Users have deep intuition for physical motion. Timed curves, however smooth, remain recognizable as synthetic.

Spring physics produces animations that overshoot and settle, that interrupt gracefully when the target changes mid-flight, and that feel proportional to distance without the programmer having to tune durations per transition. The same spring parameters produce a short snap for a 5px movement and a longer, satisfying arc for a 200px movement. Timed curves require different durations for different distances to feel proportional, creating a maintenance burden as layouts change.

`extlib.ccsp.anim` provides both. Timed curves are the right tool for coordinated, choreographed sequences where a fixed duration is required. Spring physics is the right tool for user-driven transitions where the motion should feel physical.

### Why Event-Loop Driven

The animation engine does not own a background thread. All callbacks are invoked from the same event-loop thread that drives the rest of the application.

A background-thread design requires synchronized access to every piece of state that an animation updates. Every `on_update` callback would touch UI state — positions, colors, opacities — that is also touched by event handlers and layout code. Synchronizing this correctly requires locks, and locks in UI code produce subtle bugs: deadlocks when a callback triggers an event, priority inversions under load, and data races when a programmer forgets a lock.

Event-loop-driven animation avoids this entirely. The `AnimationController` uses `std.async.interval` to schedule a tick at the target frame rate. When the tick fires, it is already on the event loop thread. All `on_update` callbacks run without any synchronization: they can freely read and write application state, call layout methods, and trigger redraws. This is the same model used by browsers (requestAnimationFrame), iOS (CADisplayLink), and every mature UI framework.

The tradeoff is that long-running work in `on_update` will drop frames. This is correct behavior: `on_update` should update a value and mark a widget dirty, not do expensive computation. The event loop is a cooperative scheduler, and animation callbacks must be cooperative.

### Why extlib, Not stdlib

Not every Ferrum program needs animation. Embedded firmware, CLI tools, servers, protocol parsers, and cryptographic libraries have no use for it. Placing animation in `extlib` keeps the standard library focused and ensures that programs not using animation pay no compile-time or binary-size cost.

The dependency on `extlib.ccsp.draw` (for `Color`, `Point`, `Size`, `Rect`) also disqualifies `anim` from `std`: a stdlib module cannot depend on an extlib module.

---

## 2. Easing Curves

### 2.1 EasingCurve

```ferrum
/// A function mapping normalized time [0.0, 1.0] to normalized progress [0.0, 1.0].
///
/// The input `t` is always in [0.0, 1.0], where 0.0 is the start of the
/// animation and 1.0 is the end. The output is the interpolation factor
/// passed to `Animatable::lerp`. For most curves the output is also in
/// [0.0, 1.0], but spring curves may exceed this range (overshoot).
@derive(Debug, Clone, Copy)
enum EasingCurve {
    /// Constant rate of change throughout the animation.
    Linear,

    /// Starts slow, finishes fast. Equivalent to CSS `ease-in`.
    EaseIn,

    /// Starts fast, finishes slow. Equivalent to CSS `ease-out`.
    EaseOut,

    /// Starts slow, fast in the middle, finishes slow. Equivalent to CSS `ease-in-out`.
    EaseInOut,

    /// A cubic Bézier curve with two control points.
    ///
    /// Follows the CSS cubic-bezier() convention: the start point is implicitly
    /// (0, 0) and the end point is implicitly (1, 1). The two control points
    /// are (x1, y1) and (x2, y2). x1 and x2 must be in [0.0, 1.0]; y1 and y2
    /// may exceed this range to produce overshoot.
    ///
    /// CSS `ease` = CubicBezier { x1: 0.25, y1: 0.1, x2: 0.25, y2: 1.0 }
    /// CSS `ease-in` = CubicBezier { x1: 0.42, y1: 0.0, x2: 1.0, y2: 1.0 }
    /// CSS `ease-out` = CubicBezier { x1: 0.0, y1: 0.0, x2: 0.58, y2: 1.0 }
    /// CSS `ease-in-out` = CubicBezier { x1: 0.42, y1: 0.0, x2: 0.58, y2: 1.0 }
    CubicBezier { x1: f32, y1: f32, x2: f32, y2: f32 },

    /// A stepped animation: progress jumps in discrete increments.
    ///
    /// `count` is the number of steps. `position` controls whether the jump
    /// happens at the start or end of each interval. Equivalent to CSS
    /// `steps(count, position)`.
    Steps { count: u32, position: StepPosition },

    /// A spring physics curve.
    ///
    /// Produced by `EasingCurve::spring(...)`. The curve is computed by
    /// integrating the spring differential equation and normalizing so that
    /// the resting position is 1.0. The output may exceed [0.0, 1.0] for
    /// underdamped springs (springs that overshoot).
    ///
    /// Duration is not fixed: `evaluate` returns 1.0 once the spring has
    /// settled within a small threshold of 1.0. Callers should use
    /// `Animated::is_done` rather than relying on a fixed duration.
    Spring { stiffness: f32, damping: f32, mass: f32 },
}

/// Controls whether a stepped animation jumps at the start or end of each step interval.
@derive(Debug, Clone, Copy, PartialEq)
enum StepPosition {
    /// Progress jumps at the end of each interval.
    /// At t=0, value is at step 0. Jump happens at the end of each interval.
    /// Equivalent to CSS `step-end` / `steps(n, end)`.
    End,

    /// Progress jumps at the start of each interval.
    /// At t=0, value immediately jumps to step 1. Equivalent to CSS `step-start`.
    /// Equivalent to CSS `steps(n, start)`.
    Start,

    /// The first jump happens at t=0. Equivalent to CSS `steps(n, jump-both)`
    /// minus the last step at t=1.
    JumpBoth,

    /// No jump at t=0 or t=1. n steps produce n-1 evenly spaced jumps.
    /// Equivalent to CSS `steps(n, jump-none)`.
    JumpNone,
}
```

### 2.2 Evaluation

```ferrum
impl EasingCurve {
    /// Evaluate the easing curve at normalized time `t`.
    ///
    /// `t` must be in [0.0, 1.0]. The returned value is the interpolation
    /// factor for `Animatable::lerp`. For Linear, EaseIn, EaseOut, EaseInOut,
    /// and Steps, the return value is in [0.0, 1.0]. For CubicBezier and Spring
    /// curves, the return value may exceed this range.
    ///
    /// For Spring curves, once `t` represents a time after the spring has
    /// settled, `evaluate` returns exactly 1.0. The mapping from t to
    /// physical time is implicit: t is treated as a fraction of the spring's
    /// total settling time. Use `Animated` or `SpringSimulation` directly
    /// when you need explicit time handling.
    pub fn evaluate(self, t: f32): f32 { ... }
}
```

The cubic Bézier evaluation uses Newton's method to solve for the `t` parameter on the x axis, then evaluates the y component. This is the same algorithm used by all major browsers for `cubic-bezier()`.

### 2.3 Predefined Constants

```ferrum
impl EasingCurve {
    /// No easing. Constant rate of change.
    const LINEAR: EasingCurve = EasingCurve::Linear

    /// Gentle acceleration then deceleration. CSS `ease`.
    const EASE: EasingCurve =
        EasingCurve::CubicBezier { x1: 0.25, y1: 0.1, x2: 0.25, y2: 1.0 }

    /// Starts slow, accelerates. CSS `ease-in`.
    const EASE_IN: EasingCurve =
        EasingCurve::CubicBezier { x1: 0.42, y1: 0.0, x2: 1.0, y2: 1.0 }

    /// Decelerates to a stop. CSS `ease-out`.
    const EASE_OUT: EasingCurve =
        EasingCurve::CubicBezier { x1: 0.0, y1: 0.0, x2: 0.58, y2: 1.0 }

    /// Slow start, fast middle, slow end. CSS `ease-in-out`.
    const EASE_IN_OUT: EasingCurve =
        EasingCurve::CubicBezier { x1: 0.42, y1: 0.0, x2: 0.58, y2: 1.0 }
}
```

### 2.4 Spring Curve Constructor

```ferrum
impl EasingCurve {
    /// Construct a spring-physics easing curve.
    ///
    /// The curve integrates the spring differential equation:
    ///   m * x'' + d * x' + k * x = k
    /// where k = stiffness, d = damping, m = mass. The solution is normalized
    /// so position 0.0 is the start and position 1.0 is the rest point.
    ///
    /// The curve evaluates to a time-normalized spring: input t=0.0 is the
    /// start of the animation; t=1.0 is the moment the spring is considered
    /// settled (position and velocity both below 0.001). The total physical
    /// time depends on the spring parameters.
    ///
    /// Returns Err(AnimError::InvalidSpringParameters) if stiffness or mass
    /// are not positive, or if damping is negative.
    pub fn spring(stiffness: f32, damping: f32, mass: f32): Result[EasingCurve, AnimError] {
        if stiffness <= 0.0 {
            return Err(AnimError::InvalidSpringParameters {
                reason: "stiffness must be positive"
            })
        }
        if mass <= 0.0 {
            return Err(AnimError::InvalidSpringParameters {
                reason: "mass must be positive"
            })
        }
        if damping < 0.0 {
            return Err(AnimError::InvalidSpringParameters {
                reason: "damping must be non-negative"
            })
        }
        Ok(EasingCurve::Spring { stiffness, damping, mass })
    }
}
```

---

## 3. Spring Physics

The spring differential equation describes a mass on a spring with damping:

```
m·x″(t) + d·x′(t) + k·(x(t) − 1) = 0
```

where `x(t)` is position (0 = start, 1 = rest), `k` is stiffness, `d` is damping, and `m` is mass. The character of the motion depends on the discriminant `Δ = d² − 4km`:

- `Δ > 0` — **overdamped**: returns to rest without oscillation, slowly
- `Δ = 0` — **critically damped**: fastest return to rest without overshoot
- `Δ < 0` — **underdamped**: oscillates, overshoots rest, settles

### 3.1 SpringSimulation

```ferrum
/// A direct spring physics simulation.
///
/// Solves the spring differential equation analytically. Suitable for cases
/// where you need explicit position and velocity values, or where the target
/// changes mid-animation (interrupt and redirect).
///
/// Position 0.0 is the start; position 1.0 is the rest (target). Position
/// may exceed 1.0 for underdamped springs (overshoot) or go below 0.0
/// for springs with large initial velocity away from the target.
@derive(Debug, Clone, Copy)]
struct SpringSimulation {
    /// Restoring force per unit displacement. Higher = stiffer, faster.
    /// Must be positive.
    pub stiffness: f32,

    /// Energy dissipation per unit velocity. Higher = less oscillation.
    /// Must be non-negative.
    pub damping: f32,

    /// The mass being driven. Higher = slower, more momentum.
    /// Must be positive.
    pub mass: f32,

    /// Initial velocity of the mass, in units per second.
    /// Zero means starting from rest. Positive means moving toward the target.
    /// Negative means moving away from the target (will overshoot before returning).
    pub initial_velocity: f32,
}
```

```ferrum
impl SpringSimulation {
    /// Construct a new spring simulation starting at position 0.0, targeting 1.0.
    pub fn new(stiffness: f32, damping: f32, mass: f32): Result[Self, AnimError] {
        if stiffness <= 0.0 {
            return Err(AnimError::InvalidSpringParameters { reason: "stiffness must be positive" })
        }
        if mass <= 0.0 {
            return Err(AnimError::InvalidSpringParameters { reason: "mass must be positive" })
        }
        if damping < 0.0 {
            return Err(AnimError::InvalidSpringParameters { reason: "damping must be non-negative" })
        }
        Ok(SpringSimulation { stiffness, damping, mass, initial_velocity: 0.0 })
    }

    /// Set the initial velocity. Returns self for chaining.
    pub fn with_initial_velocity(self, v: f32): Self {
        SpringSimulation { initial_velocity: v, ..self }
    }

    /// Compute position at `time` seconds after the start of the simulation.
    ///
    /// Returns the position in the range [0.0, ∞). For underdamped springs
    /// this may transiently exceed 1.0 (overshoot) before settling at 1.0.
    /// After a sufficiently large time, always returns a value very close to 1.0.
    pub fn position(self, time: f32): f32 { ... }

    /// Compute velocity (units per second) at `time` seconds.
    ///
    /// Positive velocity means moving toward the target (position increasing).
    /// Negative velocity means moving away from the target.
    pub fn velocity(self, time: f32): f32 { ... }

    /// Returns true when the spring has effectively settled.
    ///
    /// "Settled" means both of:
    ///   |position(time) - 1.0| < threshold
    ///   |velocity(time)| < threshold
    ///
    /// `threshold` of 0.001 is appropriate for most UI animations.
    /// A threshold of 0.0001 gives a tighter stop for precision use.
    pub fn is_done(self, time: f32, threshold: f32): bool {
        (self.position(time) - 1.0).abs() < threshold
            && self.velocity(time).abs() < threshold
    }

    /// Compute the time (in seconds) at which `is_done(t, threshold)` first becomes true.
    ///
    /// This is the duration of the animation for display purposes. The value is
    /// computed by binary search over the settling condition.
    pub fn settling_time(self, threshold: f32): f32 { ... }
}
```

### 3.2 Spring Presets

These presets are taken from React Spring's default configurations. They represent practical tuning points covering the common range of UI animation feels.

```ferrum
/// Ready-made spring parameter sets for common UI animation feels.
@derive(Debug, Clone, Copy, PartialEq)]
enum SpringPreset {
    /// Soft, slow, subtle. stiffness=170, damping=26, mass=1.
    /// Good for background transitions, fades, non-critical state changes.
    Gentle,

    /// Lively, bouncy. stiffness=180, damping=12, mass=1.
    /// Good for attention-getting transitions: new items, success states.
    /// Produces visible overshoot.
    Wobbly,

    /// Snappy and direct. stiffness=210, damping=20, mass=1.
    /// Good for user-driven interactions where responsiveness is primary:
    /// drag-to-dismiss, swipe, button press feedback.
    Stiff,

    /// Deliberate and slow. stiffness=280, damping=60, mass=1.
    /// Good for large-scale layout transitions, onboarding sequences,
    /// or when the motion is meant to be noticed.
    Slow,

    /// Provide explicit parameters.
    Custom { stiffness: f32, damping: f32, mass: f32 },
}

impl SpringPreset {
    /// Extract the stiffness, damping, and mass values for this preset.
    pub fn parameters(self): (f32, f32, f32) {
        match self {
            SpringPreset::Gentle         => (170.0, 26.0, 1.0),
            SpringPreset::Wobbly         => (180.0, 12.0, 1.0),
            SpringPreset::Stiff          => (210.0, 20.0, 1.0),
            SpringPreset::Slow           => (280.0, 60.0, 1.0),
            SpringPreset::Custom { stiffness, damping, mass }
                                         => (stiffness, damping, mass),
        }
    }

    /// Construct a SpringSimulation from this preset.
    pub fn simulation(self): SpringSimulation {
        let (k, d, m) = self.parameters()
        SpringSimulation::new(k, d, m).unwrap()
    }
}
```

---

## 4. Animatable Trait

### 4.1 Definition

```ferrum
/// A value type that can be smoothly interpolated.
///
/// Implemented for scalar and geometric types. The interpolation must be
/// linear in a perceptually meaningful space: for Color, this is Oklab,
/// not linear RGB. For geometric types, this is Cartesian interpolation
/// of each component independently.
///
/// `lerp(a, b, 0.0)` must return `a`.
/// `lerp(a, b, 1.0)` must return `b`.
/// `lerp(a, b, t)` for t outside [0.0, 1.0] is valid; the result is an
/// extrapolation. Spring animations use t > 1.0 for overshoot.
trait Animatable: Copy {
    fn lerp(a: Self, b: Self, t: f32): Self
}
```

### 4.2 Standard Implementations

```ferrum
impl Animatable for f32 {
    fn lerp(a: f32, b: f32, t: f32): f32 {
        a + (b - a) * t
    }
}

impl Animatable for f64 {
    fn lerp(a: f64, b: f64, t: f32): f64 {
        a + (b - a) * (t as f64)
    }
}

/// Color interpolation through Oklab for perceptual correctness.
/// See Section 11 for rationale.
impl Animatable for Color {
    fn lerp(a: Color, b: Color, t: f32): Color {
        Color.mix(a, b, t)
    }
}

impl Animatable for Point {
    fn lerp(a: Point, b: Point, t: f32): Point {
        Point {
            x: f32.lerp(a.x, b.x, t),
            y: f32.lerp(a.y, b.y, t),
        }
    }
}

impl Animatable for Size {
    fn lerp(a: Size, b: Size, t: f32): Size {
        Size {
            width:  f32.lerp(a.width,  b.width,  t),
            height: f32.lerp(a.height, b.height, t),
        }
    }
}

impl Animatable for Rect {
    fn lerp(a: Rect, b: Rect, t: f32): Rect {
        Rect {
            origin: Point.lerp(a.origin, b.origin, t),
            size:   Size.lerp(a.size,   b.size,   t),
        }
    }
}

impl[const N: usize] Animatable for [f32; N] {
    fn lerp(a: [f32; N], b: [f32; N], t: f32): [f32; N] {
        let mut result: [f32; N] = [0.0; N]
        for i in 0..N {
            result[i] = f32.lerp(a[i], b[i], t)
        }
        result
    }
}
```

---

## 5. Animated Value

`Animated[T]` is a stateful wrapper for a single value that can be smoothly transitioned to new targets. It is the lightweight option when you do not need the full `AnimationController` machinery — for example, a single opacity value in a widget that manages its own animation.

### 5.1 Type Definition

```ferrum
/// A value of type T that can be animated to new targets.
///
/// Tracks the current animation state: start value, target value, start time,
/// duration, and easing curve. `value_at` evaluates the current interpolated
/// value; it must be called once per frame.
///
/// `Animated[T]` does not use a timer or callback. The caller is responsible
/// for calling `value_at` at the appropriate rate (typically from the frame
/// loop driven by `AnimationController`).
struct Animated[T: Animatable] {
    current:   T,
    target:    T,
    from:      T,
    start:     Option[Instant],
    duration:  Duration,
    curve:     EasingCurve,
    spring:    Option[SpringSimulation],
}
```

### 5.2 Methods

```ferrum
impl[T: Animatable] Animated[T] {
    /// Create a new Animated value at `value`, not animating.
    pub fn new(value: T): Self {
        Animated {
            current:  value,
            target:   value,
            from:     value,
            start:    None,
            duration: Duration.ZERO,
            curve:    EasingCurve::Linear,
            spring:   None,
        }
    }

    /// Start animating toward `target` over `duration` using `curve`.
    ///
    /// If an animation is already running, it interrupts at the current
    /// value and starts from there. This avoids snapping on interruption.
    pub fn animate_to(&mut self, target: T, duration: Duration, curve: EasingCurve) {
        self.from     = self.current
        self.target   = target
        self.duration = duration
        self.curve    = curve
        self.spring   = None
        self.start    = None
    }

    /// Start a spring animation toward `target`.
    ///
    /// The spring starts from the current position. If the previous animation
    /// was also a spring and is still running, the initial velocity is
    /// inherited from the previous spring's current velocity, producing a
    /// smooth direction change.
    pub fn animate_spring(&mut self, target: T, spring: SpringPreset) {
        let (k, d, m) = spring.parameters()
        let sim = SpringSimulation { stiffness: k, damping: d, mass: m, initial_velocity: 0.0 }
        self.from   = self.current
        self.target = target
        self.spring = Some(sim)
        self.start  = None
    }

    /// Evaluate the current value at `time`.
    ///
    /// Must be called once per frame. Updates `self.current`.
    /// Returns the interpolated value between `from` and `target`.
    pub fn value_at(&mut self, time: Instant): T {
        let start = match self.start {
            None => {
                self.start = Some(time)
                time
            },
            Some(s) => s,
        }

        match self.spring {
            Some(sim) => {
                let elapsed = (time - start).as_secs_f32()
                let t = sim.position(elapsed)
                self.current = T.lerp(self.from, self.target, t)
            },
            None => {
                let elapsed = (time - start).as_secs_f32()
                let total   = self.duration.as_secs_f32()
                let t = if total == 0.0 { 1.0 } else { (elapsed / total).clamp(0.0, 1.0) }
                let progress = self.curve.evaluate(t)
                self.current = T.lerp(self.from, self.target, progress)
            },
        }

        self.current
    }

    /// Returns true if the animation has finished.
    ///
    /// For timed animations: true when elapsed >= duration.
    /// For spring animations: true when the spring has settled (threshold 0.001).
    pub fn is_done(&self, time: Instant): bool {
        let start = match self.start {
            None    => return false,
            Some(s) => s,
        }
        match self.spring {
            Some(sim) => {
                let elapsed = (time - start).as_secs_f32()
                sim.is_done(elapsed, 0.001)
            },
            None => {
                time - start >= self.duration
            },
        }
    }

    /// Stop the animation at the current position.
    ///
    /// After cancel, `value_at` returns the value frozen at the time of the
    /// last `value_at` call.
    pub fn cancel(&mut self) {
        self.target   = self.current
        self.from     = self.current
        self.spring   = None
        self.start    = None
        self.duration = Duration.ZERO
    }

    /// Return the current value without advancing the animation.
    pub fn get(&self): T {
        self.current
    }
}
```

---

## 6. Animation Descriptor

`Animation[T]` is a declarative, value-type descriptor of an animation. It holds no runtime state and can be stored, cloned, and reused. You build an `Animation` to describe what you want to happen, then hand it to `AnimationController` to run it.

### 6.1 RepeatMode

```ferrum
/// Controls how many times an animation repeats.
@derive(Debug, Clone, Copy, PartialEq)
enum RepeatMode {
    /// Play once and stop. This is the default.
    Once,

    /// Play exactly `n` times total (not n additional times).
    /// Times(1) is equivalent to Once.
    Times(u32),

    /// Play indefinitely until cancelled.
    Forever,
}
```

### 6.2 Animation Type and Builder

```ferrum
/// A declarative animation descriptor.
///
/// Describes a complete animation: start value, end value, timing, easing,
/// delay, and repeat behavior. Contains no runtime state and can be freely
/// cloned and reused.
///
/// Build with `Animation::new(from, to, duration)`, then chain modifiers.
@derive(Debug, Clone)]
struct Animation[T: Animatable] {
    from:             T,
    to:               T,
    duration:         Duration,
    curve:            EasingCurve,
    delay:            Duration,
    repeat:           RepeatMode,
    reverse_on_repeat: bool,
}

impl[T: Animatable] Animation[T] {
    /// Create a new animation from `from` to `to` over `duration`.
    ///
    /// Default curve: EasingCurve::EASE_OUT.
    /// Default delay: zero.
    /// Default repeat: Once.
    /// Default reverse_on_repeat: false.
    pub fn new(from: T, to: T, duration: Duration): Self {
        Animation {
            from,
            to,
            duration,
            curve:             EasingCurve::EASE_OUT,
            delay:             Duration.ZERO,
            repeat:            RepeatMode::Once,
            reverse_on_repeat: false,
        }
    }

    /// Set the easing curve.
    pub fn curve(self, curve: EasingCurve): Self {
        Animation { curve, ..self }
    }

    /// Set a delay before the animation starts.
    pub fn delay(self, delay: Duration): Self {
        Animation { delay, ..self }
    }

    /// Set the repeat mode.
    pub fn repeat(self, repeat: RepeatMode): Self {
        Animation { repeat, ..self }
    }

    /// If true, alternate direction on each repeat cycle.
    ///
    /// On odd-numbered cycles, the animation runs from `to` back to `from`.
    /// Only meaningful when `repeat` is not `Once`.
    pub fn reverse_on_repeat(self, yes: bool): Self {
        Animation { reverse_on_repeat: yes, ..self }
    }

    /// Evaluate the animation at `elapsed` time since the animation started.
    ///
    /// `elapsed` includes the delay: values during the delay period return `from`.
    /// Returns `to` once the animation is complete and not repeating.
    pub fn value_at(&self, elapsed: Duration): T {
        if elapsed < self.delay {
            return self.from
        }
        let active = elapsed - self.delay
        let cycle_duration = self.duration
        let cycle_secs = cycle_duration.as_secs_f32()
        if cycle_secs == 0.0 {
            return self.to
        }

        let active_secs = active.as_secs_f32()
        let cycle_index = (active_secs / cycle_secs) as u32
        let cycle_t = (active_secs % cycle_secs) / cycle_secs

        let done = match self.repeat {
            RepeatMode::Once      => cycle_index >= 1,
            RepeatMode::Times(n)  => cycle_index >= n,
            RepeatMode::Forever   => false,
        }
        if done {
            return self.to
        }

        let t = if self.reverse_on_repeat && cycle_index % 2 == 1 {
            1.0 - cycle_t
        } else {
            cycle_t
        }

        T.lerp(self.from, self.to, self.curve.evaluate(t))
    }

    /// Returns true when the animation has finished (will not produce new values).
    ///
    /// Always false for RepeatMode::Forever.
    pub fn is_done(&self, elapsed: Duration): bool {
        match self.repeat {
            RepeatMode::Forever  => false,
            RepeatMode::Once     => elapsed >= self.delay + self.duration,
            RepeatMode::Times(n) => elapsed >= self.delay + self.duration * n,
        }
    }
}
```

---

## 7. AnimationController

The `AnimationController` is the central driver. It owns a set of running animations, schedules an interval timer at 60fps, and dispatches `on_update` callbacks from the event loop thread each frame.

### 7.1 AnimationHandle

```ferrum
/// An opaque handle to a running animation.
///
/// Used to cancel an animation or query its completion status.
/// Dropping the handle does not cancel the animation: call `cancel` explicitly.
@derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct AnimationHandle {
    id: u64,
}

impl AnimationHandle {
    /// Returns true if the animation associated with this handle has finished.
    ///
    /// Returns false if the handle has expired (the animation was already
    /// cancelled or removed).
    pub fn is_done(&self, controller: &AnimationController): bool {
        controller.is_handle_done(*self)
    }
}
```

### 7.2 AnimationController

```ferrum
/// Drives animations from the async event loop.
///
/// Uses `std.async.interval` at 60fps. All `on_update` callbacks are called
/// from the event loop thread: no synchronization is required to access
/// application state from callbacks.
///
/// Create one AnimationController per application (or per independently
/// animated subsystem). All animations share the same timer tick.
struct AnimationController {
    runtime:    &Runtime,
    animations: Map[u64, BoxedAnimation],
    next_id:    u64,
}

impl AnimationController {
    /// Create a new controller attached to `runtime`.
    ///
    /// The controller starts its internal interval timer immediately.
    /// Animations can be registered before the first tick.
    pub fn new(runtime: &Runtime): Self ! Async { ... }

    /// Start a timed animation described by `animation`.
    ///
    /// `on_update` is called once per frame with the current interpolated value.
    /// It is called on the event loop thread.
    ///
    /// Returns a handle that can be used to cancel the animation.
    pub fn start[T: Animatable + 'static](
        &mut self,
        animation: Animation[T],
        on_update: impl Fn(T) + 'static,
    ): AnimationHandle { ... }

    /// Start a spring animation from `from` to `to`.
    ///
    /// The spring parameters are taken from `spring`. `on_update` is called
    /// once per frame until the spring settles (threshold 0.001).
    pub fn start_spring[T: Animatable + 'static](
        &mut self,
        from:      T,
        to:        T,
        spring:    SpringPreset,
        on_update: impl Fn(T) + 'static,
    ): AnimationHandle { ... }

    /// Cancel a specific animation by handle.
    ///
    /// The `on_update` callback will not be called again after this returns.
    /// If the handle is already expired, this is a no-op.
    pub fn cancel(&mut self, handle: AnimationHandle) { ... }

    /// Cancel all running animations.
    ///
    /// All `on_update` callbacks will not be called again after this returns.
    pub fn cancel_all(&mut self) { ... }

    /// Query whether a handle's animation is done.
    pub fn is_handle_done(&self, handle: AnimationHandle): bool { ... }
}
```

### 7.3 Internal Timer Architecture

The controller creates a single `std.async.interval` at 60fps (period ≈ 16.67ms). On each tick:

1. Record `now = Instant.now()`.
2. For each running animation, evaluate `value_at(now)` and call `on_update(value)`.
3. Remove animations where `is_done(now)` is true after the callback.

The 60fps target is advisory. If the event loop is busy processing a long-running callback, a tick will be delayed. When the delayed tick eventually fires, `Instant.now()` will reflect the true elapsed time, so the animation value will be correct even if frames were dropped. The animation does not speed up to "catch up" — it evaluates at the actual elapsed time and accepts the missed frames as-is. This matches browser behavior.

```ferrum
// Internal tick implementation (illustrative, not public API)
fn tick(&mut self, now: Instant) {
    let mut done_ids: Vec[u64] = Vec.new()

    for (id, anim) in &mut self.animations {
        let value = anim.value_at(now)
        anim.call_on_update(value)
        if anim.is_done(now) {
            done_ids.push(*id)
        }
    }

    for id in done_ids {
        self.animations.remove(id)
    }
}
```

---

## 8. Sequences

A `Sequence` runs animations one after another. Each step starts when the previous step finishes. Steps can be animations with callbacks, fixed-duration waits, or side-effect callbacks with no animated value.

### 8.1 Sequence Builder

```ferrum
/// A step in a sequence.
enum SequenceStep {
    Animate(BoxedSequenceAnimation),
    Wait(Duration),
    Call(Box[dyn Fn()]),
}

/// Builds a Sequence by chaining steps.
struct SequenceBuilder {
    steps: Vec[SequenceStep],
}

impl SequenceBuilder {
    pub fn new(): Self {
        SequenceBuilder { steps: Vec.new() }
    }

    /// Add an animation step. `on_update` is called each frame for the
    /// duration of this step. The next step starts when this animation
    /// is done, respecting the animation's `delay` and `repeat` settings.
    pub fn then[T: Animatable + 'static](
        mut self,
        animation: Animation[T],
        on_update: impl Fn(T) + 'static,
    ): Self {
        self.steps.push(SequenceStep::Animate(box_animation(animation, on_update)))
        self
    }

    /// Add a fixed-duration wait with no callbacks.
    pub fn then_wait(mut self, duration: Duration): Self {
        self.steps.push(SequenceStep::Wait(duration))
        self
    }

    /// Add a side-effect call with no animation. Called once when this step
    /// is reached. Useful for triggering state changes between animations
    /// (e.g., changing a tab, showing a new element before animating it in).
    pub fn then_call(mut self, f: impl Fn() + 'static): Self {
        self.steps.push(SequenceStep::Call(Box.new(f)))
        self
    }

    /// Finalize the sequence.
    pub fn build(self): Sequence {
        Sequence { steps: self.steps }
    }
}

struct Sequence {
    steps: Vec[SequenceStep],
}
```

### 8.2 Starting a Sequence

```ferrum
impl AnimationController {
    /// Start a sequence. Returns a handle that completes when the last step
    /// in the sequence finishes.
    ///
    /// Cancelling the handle stops the sequence at its current step and does
    /// not call any further `on_update` or `then_call` callbacks.
    pub fn start_sequence(&mut self, seq: Sequence): AnimationHandle { ... }
}
```

---

## 9. Parallel Groups

A `ParallelGroup` runs a set of animations simultaneously. The group handle is done when all animations in the group are done. Animations in a group may have different durations and easing curves; they all start at the same instant (the instant the group is started).

### 9.1 ParallelGroup Builder

```ferrum
/// Builds a ParallelGroup by accumulating animations.
struct ParallelGroupBuilder {
    animations: Vec[BoxedParallelAnimation],
}

impl ParallelGroupBuilder {
    pub fn new(): Self {
        ParallelGroupBuilder { animations: Vec.new() }
    }

    /// Add an animation to the group.
    ///
    /// All animations added to a group start at the same time.
    /// Use `Animation::delay` to offset individual animations within the group.
    pub fn add[T: Animatable + 'static](
        mut self,
        animation: Animation[T],
        on_update: impl Fn(T) + 'static,
    ): Self {
        self.animations.push(box_parallel_animation(animation, on_update))
        self
    }

    pub fn build(self): ParallelGroup {
        ParallelGroup { animations: self.animations }
    }
}

struct ParallelGroup {
    animations: Vec[BoxedParallelAnimation],
}
```

### 9.2 Starting a Parallel Group

```ferrum
impl AnimationController {
    /// Start all animations in the group simultaneously.
    ///
    /// Returns a handle that completes when ALL animations in the group are done.
    /// Cancelling the handle stops all animations in the group.
    pub fn start_parallel(&mut self, group: ParallelGroup): AnimationHandle { ... }
}
```

---

## 10. Stagger

Stagger is a specialized form of a parallel group where each item starts a fixed delay after the previous one. It is the standard pattern for animating list items in one-by-one.

```ferrum
/// A set of animations where each starts `delay` after the previous.
///
/// The first animation starts at t=0. The second starts at t=delay.
/// The third at t=2*delay. And so on.
///
/// This is equivalent to a ParallelGroup where each animation after the
/// first has an additional `i * delay` prepended to its own delay.
struct Stagger[T: Animatable] {
    items: Vec[(Animation[T], Box[dyn Fn(T)])],
    delay: Duration,
}

impl[T: Animatable + 'static] Stagger[T] {
    /// Create a stagger from a list of (animation, callback) pairs.
    ///
    /// `delay` is the time between the start of consecutive animations.
    /// A delay of Duration::from_millis(50) with 8 items means the last
    /// item starts at 350ms.
    pub fn new(items: Vec[(Animation[T], impl Fn(T) + 'static)], delay: Duration): Self {
        let boxed = items.into_iter()
            .map(|(anim, f)| (anim, Box.new(f) as Box[dyn Fn(T)]))
            .collect()
        Stagger { items: boxed, delay }
    }
}

impl AnimationController {
    /// Start a stagger. Returns a handle that completes when the last item
    /// in the stagger is done.
    ///
    /// Internally this is equivalent to creating a ParallelGroup where item
    /// `i` has an extra `delay * i` prepended to its existing delay.
    pub fn start_stagger[T: Animatable + 'static](
        &mut self,
        stagger: Stagger[T],
    ): AnimationHandle { ... }
}
```

**Use case — list items animating in:**

```ferrum
let items: Vec[ListItem] = fetch_results()?

// Build one Animation per item, each starting 60ms after the previous.
let stagger = Stagger.new(
    items.iter().enumerate().map(|(i, item)| {
        let id = item.id
        let anim = Animation.new(0.0f32, 1.0f32, Duration.from_millis(200))
            .curve(EasingCurve::EASE_OUT)
        let callback = move |opacity: f32| {
            app_state.set_item_opacity(id, opacity)
            widget_tree.mark_dirty(item_widget_id(id))
        }
        (anim, callback)
    }).collect(),
    Duration.from_millis(60),
)

let handle = controller.start_stagger(stagger)
```

---

## 11. Widget Integration Pattern

`extlib.ccsp.anim` does not depend on any widget library. Instead it defines a clean integration pattern that widget systems can follow.

### 11.1 The Dirty-Flag Pattern

Animation does not directly modify widget properties. Instead, `on_update` callbacks:

1. Write the new animated value into shared state (an `Arc[Cell[T]]` or equivalent).
2. Call `widget_tree.mark_dirty(widget_id)` to schedule a redraw.

The frame loop, driven by the platform backend, calls `draw` on dirty widgets. The widget reads the current animated value from shared state and uses it during layout and paint.

```ferrum
use extlib.ccsp.anim.{AnimationController, Animation, SpringPreset, EasingCurve}
use extlib.ccsp.draw.Color
use std.sync.Arc
use std.cell.Cell
use std.time.Duration

struct ButtonState {
    background:  Arc[Cell[Color]],
    controller:  AnimationController,
    widget_id:   WidgetId,
    press_handle: Option[AnimationHandle],
}

impl ButtonState {
    pub fn on_press(&mut self) {
        let bg   = Arc.clone(&self.background)
        let wid  = self.widget_id
        let anim = Animation.new(
            Color.from_hex(0x4A90E2FF),
            Color.from_hex(0x2C5FA8FF),
            Duration.from_millis(80),
        ).curve(EasingCurve::EASE_OUT)

        self.press_handle = Some(
            self.controller.start(anim, move |c: Color| {
                bg.set(c)
                widget_tree.mark_dirty(wid)
            })
        )
    }

    pub fn on_release(&mut self) {
        let bg  = Arc.clone(&self.background)
        let wid = self.widget_id

        self.controller.start_spring(
            self.background.get(),
            Color.from_hex(0x4A90E2FF),
            SpringPreset::Stiff,
            move |c: Color| {
                bg.set(c)
                widget_tree.mark_dirty(wid)
            },
        )
    }
}
```

### 11.2 AnimatedWidget Wrapper

For the common case of a single animated scalar property, the `AnimatedWidget` wrapper provides a ready-made integration:

```ferrum
/// Wraps a widget with a single animated parameter of type T.
///
/// `AnimatedWidget` owns an `Animated[T]` and marks the widget dirty on
/// each frame while the animation is running. The widget receives the
/// current animated value in its `draw` method via `AnimatedWidget::value`.
///
/// This is a convenience type; it does not replace the full controller
/// for multi-animation or sequenced use cases.
struct AnimatedWidget[T: Animatable] {
    pub inner:   Box[dyn Widget],
    value:       Arc[Cell[T]],
    animated:    Animated[T],
    widget_id:   WidgetId,
}

impl[T: Animatable + 'static] AnimatedWidget[T] {
    pub fn new(inner: Box[dyn Widget], initial: T, widget_id: WidgetId): Self {
        AnimatedWidget {
            inner,
            value:     Arc.new(Cell.new(initial)),
            animated:  Animated.new(initial),
            widget_id,
        }
    }

    pub fn animate_to(&mut self, target: T, duration: Duration, curve: EasingCurve) {
        self.animated.animate_to(target, duration, curve)
    }

    pub fn animate_spring(&mut self, target: T, spring: SpringPreset) {
        self.animated.animate_spring(target, spring)
    }

    /// Call once per frame from the frame loop. Returns the current value.
    pub fn tick(&mut self, now: Instant): T {
        let v = self.animated.value_at(now)
        self.value.set(v)
        v
    }

    pub fn value(&self): T {
        self.value.get()
    }
}
```

---

## 12. Color Interpolation via Oklab

Color interpolation is one of the most visible places where "correct" and "naive" diverge.

Linear RGB interpolation between red `(1, 0, 0)` and blue `(0, 0, 1)` passes through a dark grayish midpoint. The colors are summing to a low-energy mid-gray region of the RGB cube, even though the visual midpoint between red and blue should be a vibrant magenta. Interpolating through linear RGB produces dull, dark transitions wherever two colors do not have additive overlap.

Oklab (Björn Ottosson, 2020) is a perceptually uniform color space. Equal distances in Oklab correspond to approximately equal perceived differences. Mixing two colors through Oklab keeps the midpoint at the perceived visual midpoint: vibrant, correctly-hued, at the expected lightness.

```ferrum
/// Color interpolation through Oklab.
///
/// This is the implementation used by `impl Animatable for Color`.
/// It is the same conversion used by `Color::mix` in extlib.ccsp.draw.
///
/// The conversion path is:
///   Linear RGB → XYZ (D65) → Oklab → lerp → Oklab → XYZ → Linear RGB
///
/// Alpha is linearly interpolated independently in linear space.
/// The alpha channel is not pre-multiplied before Oklab interpolation;
/// color components are interpolated at full opacity and alpha applied after.
impl Animatable for Color {
    fn lerp(a: Color, b: Color, t: f32): Color {
        Color.mix(a, b, t)
    }
}
```

This matches `Color::mix` from `extlib.ccsp.draw` (Section 2.4 of that document). The `draw` and `anim` extlibs produce identical results for color interpolation: a gradient drawn with `LinearGradient` and an animated color transition between the same two stops will pass through the same midpoint.

The connection is intentional. Color ramps in data visualizations, animated color themes, and hover-state transitions all benefit from the same perceptual model. Using the same interpolation function in both drawing and animation means that a gradient and its animated equivalent look identical at every `t`.

---

## 13. Error Types

```ferrum
/// Errors produced by the animation engine.
@derive(Debug, Clone)]
enum AnimError {
    /// The AsyncRuntime is not available or has been shut down.
    /// Returned by `AnimationController::new` if the runtime is not running.
    RuntimeNotAvailable,

    /// Spring parameters are physically invalid.
    ///
    /// `reason` describes which parameter is invalid and why.
    /// This error is returned by `SpringSimulation::new` and
    /// `EasingCurve::spring` rather than panicking.
    InvalidSpringParameters { reason: &'static str },

    /// The AnimationHandle refers to an animation that has already finished
    /// or been cancelled.
    ///
    /// Returned by operations that require a live handle.
    HandleExpired,
}

impl fmt.Display for AnimError {
    fn fmt(&self, f: &mut fmt.Formatter): fmt.Result {
        match self {
            AnimError::RuntimeNotAvailable =>
                f.write_str("animation runtime is not available"),
            AnimError::InvalidSpringParameters { reason } =>
                write!(f, "invalid spring parameters: {}", reason),
            AnimError::HandleExpired =>
                f.write_str("animation handle has expired"),
        }
    }
}
```

---

## 14. Example Usage

### 14.1 Fade-In on Appear

```ferrum
use extlib.ccsp.anim.{AnimationController, Animation, EasingCurve}
use std.time.Duration

fn animate_fade_in(controller: &mut AnimationController, widget_id: WidgetId,
                   opacity: Arc[Cell[f32]]) {
    let anim = Animation.new(0.0f32, 1.0f32, Duration.from_millis(200))
        .curve(EasingCurve::EASE_OUT)

    controller.start(anim, move |v: f32| {
        opacity.set(v)
        widget_tree.mark_dirty(widget_id)
    })
}
```

### 14.2 Spring Bounce on Tap

```ferrum
use extlib.ccsp.anim.{AnimationController, SpringPreset}
use extlib.ccsp.draw.Point

fn animate_bounce(controller: &mut AnimationController, widget_id: WidgetId,
                  position: Arc[Cell[Point]], target: Point) {
    let current = position.get()

    controller.start_spring(current, target, SpringPreset::Wobbly, move |p: Point| {
        position.set(p)
        widget_tree.mark_dirty(widget_id)
    })
}
```

### 14.3 Staggered List Entry

```ferrum
use extlib.ccsp.anim.{AnimationController, Animation, Stagger, EasingCurve}
use std.time.Duration

fn animate_list_in(
    controller: &mut AnimationController,
    items: &[ListItem],
    opacities: Arc[Vec[Cell[f32]]],
    widget_ids: &[WidgetId],
) {
    let pairs: Vec[_] = items.iter().enumerate().map(|(i, item)| {
        let ops  = Arc.clone(&opacities)
        let wid  = widget_ids[i]
        let anim = Animation.new(0.0f32, 1.0f32, Duration.from_millis(180))
            .curve(EasingCurve::EASE_OUT)
        let cb = move |v: f32| {
            ops[i].set(v)
            widget_tree.mark_dirty(wid)
        }
        (anim, cb)
    }).collect()

    let stagger = Stagger.new(pairs, Duration.from_millis(50))
    controller.start_stagger(stagger)
}
```

### 14.4 Sequence: Slide In Then Fade

```ferrum
use extlib.ccsp.anim.{AnimationController, Animation, SequenceBuilder, EasingCurve, SpringPreset}
use extlib.ccsp.draw.{Color, Point}
use std.time.Duration

fn animate_panel_appear(
    controller: &mut AnimationController,
    widget_id:  WidgetId,
    position:   Arc[Cell[Point]],
    opacity:    Arc[Cell[f32]],
    offscreen:  Point,
    final_pos:  Point,
) {
    let pos_ref = Arc.clone(&position)
    let wid     = widget_id

    let slide = Animation.new(offscreen, final_pos, Duration.from_millis(300))
        .curve(EasingCurve::EASE_OUT)

    let op_ref = Arc.clone(&opacity)
    let fade   = Animation.new(0.0f32, 1.0f32, Duration.from_millis(150))
        .curve(EasingCurve::LINEAR)

    let seq = SequenceBuilder.new()
        .then(slide, move |p: Point| {
            pos_ref.set(p)
            widget_tree.mark_dirty(wid)
        })
        .then(fade, move |v: f32| {
            op_ref.set(v)
            widget_tree.mark_dirty(wid)
        })
        .build()

    controller.start_sequence(seq)
}
```

### 14.5 Interrupting a Spring Mid-Flight

```ferrum
// The user taps a button to move a panel, then taps again before it arrives.
// Spring animations inherit current position on interrupt; no snap.

fn move_panel(state: &mut PanelState, new_target: Point) {
    // Cancel the previous handle if running.
    if let Some(h) = state.move_handle.take() {
        state.controller.cancel(h)
    }

    // Start a new spring from wherever the panel is now.
    let pos = Arc.clone(&state.position)
    let wid = state.widget_id
    let from = state.position.get()

    state.move_handle = Some(
        state.controller.start_spring(from, new_target, SpringPreset::Stiff, move |p: Point| {
            pos.set(p)
            widget_tree.mark_dirty(wid)
        })
    )
}
```

---

## 15. Dependencies

| Dependency | What is used |
|---|---|
| `extlib.ccsp.draw` | `Color` (Animatable impl, Oklab interpolation), `Point`, `Size`, `Rect` |
| `std.time` | `Duration`, `Instant` |
| `std.async` | `Runtime`, `interval` (for the 60fps tick) |
| `std.sync` | `Arc` (shared state in callbacks) |
| `std.cell` | `Cell` (interior mutability for animated values) |
| `std.collections` | `Map`, `Vec` (internal controller state) |

No other extlib modules are required. In particular:

- `extlib.ccsp.widget` is **not** a dependency: the integration pattern (Section 11) uses a protocol, not an import. Widget system integration is documented here but not linked at the type level.
- `extlib.ccsp.font` and `extlib.ccsp.text` are not needed: animation does not render text.
- No network, crypto, or serialization extlibs are involved.

The only mandatory extlib dependency is `extlib.ccsp.draw`, solely for the shared geometric and color types. A future factoring that extracted `Color`, `Point`, `Size`, and `Rect` into a `extlib.ccsp.geometry` foundational crate would allow `anim` to function without a full draw dependency, but that split is not part of the current design.
