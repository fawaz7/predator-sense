# Fan Control UI Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Fan Control page's overlapping controls with one mode
picker, one four-way plan selector and a graphical curve editor, so no two
controls can write the same setting.

**Architecture:** Phase 1 already made each power mode own a `FanPlan` in
config and gave a single reconciler in `window.rs` sole ownership of the fan
hardware. This plan changes only what the user sees and what the page writes
into `fan_plans`. Pure geometry lives in `ui/fan_curve_geometry.rs` and is unit
tested; drawing and dragging live in `ui/fan_curve_widget.rs`; the page wires
them to config. No hardware call and no reconciler logic changes.

**Tech Stack:** Rust 2021, gtk4-rs (`gtk4::prelude`, `DrawingArea`,
`set_draw_func`, `GestureDrag`, `GestureClick`), Cairo drawing, serde config.

**Spec:** `docs/superpowers/specs/2026-09-15-fan-control-design.md` (phases 2
and 3 of its "Suggested phasing"; phase 1 shipped and was hardware verified).

## Global Constraints

- MSRV: the sibling crates declare `rust-version = "1.80"`. Do not use
  `Option::is_none_or` (1.82) or any API newer than 1.80. Use `map_or`.
- Every new `i18n` key must exist in all nine language functions in
  `src/i18n.rs`: `t_pt`, `t_en`, `t_es`, `t_zh`, `t_ja`, `t_ru`, `t_de`,
  `t_it`, `t_tr`. A missing key falls through to `_ => key`, which renders the
  raw key string to the user.
- Settings apply live with autosave: no Apply button, no confirmation step.
- The page writes intent into `config.fan_plans` only. It must never call
  `fan::set_pwm_*`, `fan::set_fan_mode` or any other hardware write. The
  reconciler owns the hardware.
- Never write an unverified hardware claim into a comment, a commit message,
  a changelog entry or a user-visible string.
- No AI attribution anywhere in commits, changelog or PR text.
- `cargo build --release` must be warning-free. Removing the page's last call
  to a `fan.rs` function turns that function into dead code; if that happens,
  say so in the task report rather than deleting a hardware function.
- Run `cargo test` from `predator-sense-gui/`. The suite is at 260 passing
  tests before this plan. Task 1 takes it to 273; Task 3 deletes the two
  tests of the helper it makes obsolete, landing at 271. It must not drop
  below that for any other reason.

---

### Task 1: Curve geometry, and the two constants the widget needs

**Files:**
- Create: `predator-sense-gui/src/ui/fan_curve_geometry.rs`
- Modify: `predator-sense-gui/src/ui/mod.rs` (register the module)
- Modify: `predator-sense-gui/src/hardware/fan.rs:189` (make
  `FAN_CURVE_BREAKPOINTS_C` public), `:336` (`is_none_or` → `map_or`),
  `:364` (make `curve_resume_c` public)
- Test: same file, `#[cfg(test)] mod tests` at the bottom

**Interfaces:**
- Consumes: nothing.
- Produces: `ui::fan_curve_geometry::{Plot, STEP_COUNT, AXIS_MIN_C,
  AXIS_MAX_C, plot_for, column, step_at_x, y_for_pct, pct_at_y, x_for_temp,
  edited, firmware_region_end_x}`; and `fan::FAN_CURVE_BREAKPOINTS_C`,
  `fan::curve_resume_c` become `pub`.

Why an axis of 35 °C to 95 °C: the six curve bands are `<45`, `45..55`,
`55..65`, `65..75`, `75..85`, `>=85`. Drawing 35 to 95 makes all six columns
exactly ten degrees wide, so a column's width is honest and the live
temperature marker lands in the column actually in force. The two end columns
are open-ended in the code, so the marker clamps to the plot edges.

- [ ] **Step 1: Write the failing tests**

Create `predator-sense-gui/src/ui/fan_curve_geometry.rs` with only this test
module (no implementation yet):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn plot() -> Plot {
        // 6 columns of 100px, 200px tall.
        Plot { x: 0.0, y: 0.0, w: 600.0, h: 200.0 }
    }

    #[test]
    fn the_plot_sits_inside_the_widget_margins() {
        let p = plot_for(600.0 + MARGIN_LEFT + MARGIN_RIGHT, 200.0 + MARGIN_TOP + MARGIN_BOTTOM);
        assert_eq!(p.x, MARGIN_LEFT);
        assert_eq!(p.y, MARGIN_TOP);
        assert_eq!(p.w, 600.0);
        assert_eq!(p.h, 200.0);
    }

    #[test]
    fn a_widget_smaller_than_its_margins_gets_an_empty_plot() {
        let p = plot_for(4.0, 4.0);
        assert_eq!(p.w, 0.0);
        assert_eq!(p.h, 0.0);
    }

    #[test]
    fn the_columns_tile_the_plot_exactly() {
        let p = plot();
        assert_eq!(column(p, 0), (0.0, 100.0));
        assert_eq!(column(p, 5), (500.0, 600.0));
    }

    #[test]
    fn a_point_in_a_column_selects_that_step() {
        let p = plot();
        assert_eq!(step_at_x(p, 50.0), 0);
        assert_eq!(step_at_x(p, 250.0), 2);
        assert_eq!(step_at_x(p, 599.0), 5);
    }

    #[test]
    fn a_point_outside_the_plot_selects_the_nearest_step() {
        let p = plot();
        assert_eq!(step_at_x(p, -80.0), 0);
        assert_eq!(step_at_x(p, 9_000.0), 5);
    }

    #[test]
    fn full_speed_is_the_top_of_the_plot_and_zero_is_the_bottom() {
        let p = plot();
        assert_eq!(y_for_pct(p, 100), 0.0);
        assert_eq!(y_for_pct(p, 0), 200.0);
        assert_eq!(y_for_pct(p, 50), 100.0);
    }

    #[test]
    fn a_y_position_reads_back_as_the_percent_it_was_drawn_at() {
        let p = plot();
        for pct in [0u8, 25, 50, 75, 100] {
            assert_eq!(pct_at_y(p, y_for_pct(p, pct)), pct);
        }
    }

    #[test]
    fn a_drag_past_the_plot_clamps_to_zero_and_a_hundred() {
        let p = plot();
        assert_eq!(pct_at_y(p, -400.0), 100);
        assert_eq!(pct_at_y(p, 400.0), 0);
    }

    #[test]
    fn a_dragged_percent_snaps_to_the_same_five_the_spin_buttons_use() {
        let p = plot();
        // 63% of the way up the plot.
        assert_eq!(pct_at_y(p, 200.0 - 0.63 * 200.0), 65);
        assert_eq!(pct_at_y(p, 200.0 - 0.62 * 200.0), 60);
    }

    #[test]
    fn the_axis_spans_the_temperatures_the_curve_actually_switches_at() {
        let p = plot();
        assert_eq!(x_for_temp(p, AXIS_MIN_C), 0.0);
        assert_eq!(x_for_temp(p, AXIS_MAX_C), 600.0);
        // Every breakpoint the curve steps at is a column boundary, so the
        // drawing cannot disagree with `fan::fan_curve_pct`.
        for (i, &c) in crate::hardware::fan::FAN_CURVE_BREAKPOINTS_C.iter().enumerate() {
            assert_eq!(x_for_temp(p, c), column(p, i).1);
        }
    }

    #[test]
    fn a_temperature_off_the_axis_clamps_to_the_plot() {
        let p = plot();
        assert_eq!(x_for_temp(p, 12.0), 0.0);
        assert_eq!(x_for_temp(p, 120.0), 600.0);
    }

    #[test]
    fn an_edit_changes_only_the_step_it_lands_on() {
        let p = plot();
        let steps = [25u8, 35, 50, 65, 80, 100];
        let after = edited(steps, p, 250.0, y_for_pct(p, 20));
        assert_eq!(after, [25, 35, 20, 65, 80, 100]);
    }

    #[test]
    fn the_firmware_region_ends_where_the_curve_takes_over() {
        let p = plot();
        assert_eq!(firmware_region_end_x(p, None), None);
        assert_eq!(firmware_region_end_x(p, Some(55.0)), Some(x_for_temp(p, 55.0)));
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd predator-sense-gui && cargo test fan_curve_geometry`
Expected: FAIL to compile, "cannot find type `Plot`".

- [ ] **Step 3: Make the two `fan.rs` items public and fix the MSRV slip**

In `predator-sense-gui/src/hardware/fan.rs`, change these three lines, leaving
every doc comment as it is:

```rust
// was: const FAN_CURVE_BREAKPOINTS_C: [f64; 5] = [45.0, 55.0, 65.0, 75.0, 85.0];
pub const FAN_CURVE_BREAKPOINTS_C: [f64; 5] = [45.0, 55.0, 65.0, 75.0, 85.0];
```

```rust
// was: fn curve_resume_c(steps: &[u8; 6]) -> Option<f64> {
pub fn curve_resume_c(steps: &[u8; 6]) -> Option<f64> {
```

```rust
// was: && curve_resume_c(steps).is_none_or(|resume| temp_c < resume)
// `map_or`, not `is_none_or`: the sibling crates declare rust-version 1.80
// and `is_none_or` is 1.82.
&& curve_resume_c(steps).map_or(true, |resume| temp_c < resume)
```

- [ ] **Step 4: Write the geometry module**

Put this above the test module in `fan_curve_geometry.rs`:

```rust
//! Pure geometry for the fan curve chart: percent to pixel, pixel to step,
//! and the temperature axis. Split from the widget so every conversion the
//! drawing and the dragging depend on is unit tested without a display.
//!
//! The axis runs 35 C to 95 C because that makes all six of the curve's bands
//! exactly ten degrees wide (`<45`, `45..55`, `55..65`, `65..75`, `75..85`,
//! `>=85`), so a column's width on screen is the band's real width and the
//! temperature marker lands in the band actually in force. The first and last
//! bands are open-ended in the code, so anything off the axis clamps.

/// The six editable percentages, matching `config::fan_curve_points`.
pub const STEP_COUNT: usize = 6;

/// Left edge of the temperature axis, in Celsius.
pub const AXIS_MIN_C: f64 = 35.0;
/// Right edge of the temperature axis, in Celsius.
pub const AXIS_MAX_C: f64 = 95.0;

/// Room for the percent labels down the left side.
pub const MARGIN_LEFT: f64 = 34.0;
pub const MARGIN_RIGHT: f64 = 8.0;
pub const MARGIN_TOP: f64 = 10.0;
/// Room for the temperature labels along the bottom.
pub const MARGIN_BOTTOM: f64 = 20.0;

/// A percent drag snaps to this, so dragging and the spin buttons cannot
/// disagree about which values are reachable.
const SNAP_PCT: f64 = 5.0;

/// The drawable chart area inside the widget's margins.
#[derive(Clone, Copy, Debug, PartialEq)]
pub struct Plot {
    pub x: f64,
    pub y: f64,
    pub w: f64,
    pub h: f64,
}

/// The plot for a widget of this size. Never negative: a widget narrower than
/// its own margins gets an empty plot, which every function below treats as
/// "nothing to draw" rather than dividing by it.
pub fn plot_for(width: f64, height: f64) -> Plot {
    Plot {
        x: MARGIN_LEFT,
        y: MARGIN_TOP,
        w: (width - MARGIN_LEFT - MARGIN_RIGHT).max(0.0),
        h: (height - MARGIN_TOP - MARGIN_BOTTOM).max(0.0),
    }
}

fn column_width(plot: Plot) -> f64 {
    plot.w / STEP_COUNT as f64
}

/// The `(left, right)` pixel bounds of one step's column.
pub fn column(plot: Plot, step: usize) -> (f64, f64) {
    let width = column_width(plot);
    let left = plot.x + width * step as f64;
    (left, left + width)
}

/// Which step a horizontal position belongs to, clamped to the ends so a drag
/// that leaves the widget keeps editing the step it started on.
pub fn step_at_x(plot: Plot, x: f64) -> usize {
    let width = column_width(plot);
    if width <= 0.0 {
        return 0;
    }
    let index = ((x - plot.x) / width).floor();
    if index < 0.0 {
        return 0;
    }
    (index as usize).min(STEP_COUNT - 1)
}

/// The top of the bar for `pct`: 100% is the top of the plot, 0% the bottom.
pub fn y_for_pct(plot: Plot, pct: u8) -> f64 {
    plot.y + plot.h * (1.0 - f64::from(pct.min(100)) / 100.0)
}

/// The percent a vertical position means, clamped to 0..=100 and snapped to
/// the same five-percent grid the spin buttons step by.
pub fn pct_at_y(plot: Plot, y: f64) -> u8 {
    if plot.h <= 0.0 {
        return 0;
    }
    let raw = (1.0 - (y - plot.y) / plot.h) * 100.0;
    let snapped = (raw / SNAP_PCT).round() * SNAP_PCT;
    snapped.clamp(0.0, 100.0) as u8
}

/// Where a temperature sits on the axis, clamped to the plot.
pub fn x_for_temp(plot: Plot, temp_c: f64) -> f64 {
    let span = AXIS_MAX_C - AXIS_MIN_C;
    let fraction = ((temp_c - AXIS_MIN_C) / span).clamp(0.0, 1.0);
    plot.x + plot.w * fraction
}

/// `steps` with the step under `(x, y)` set to what that height means.
pub fn edited(mut steps: [u8; STEP_COUNT], plot: Plot, x: f64, y: f64) -> [u8; STEP_COUNT] {
    steps[step_at_x(plot, x)] = pct_at_y(plot, y);
    steps
}

/// Where the shaded firmware region ends, given `fan::curve_resume_c`.
///
/// Below that temperature a curve with a zero first step leaves the fans to
/// the firmware, which is a real behaviour of the reconciler and not a drawing
/// flourish: without showing it, a user reads the zero step as "off up to
/// 45 C" when it is actually "the firmware's up to the top of the first
/// non-zero band".
pub fn firmware_region_end_x(plot: Plot, resume_c: Option<f64>) -> Option<f64> {
    resume_c.map(|resume| x_for_temp(plot, resume))
}
```

Register it in `predator-sense-gui/src/ui/mod.rs`, in alphabetical order
between `fan_control_page` and `fan_page`:

```rust
pub mod fan_curve_geometry;
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `cd predator-sense-gui && cargo test fan_curve_geometry`
Expected: 13 passed.

Then run the whole suite: `cargo test`
Expected: 273 passed, 0 failed.

- [ ] **Step 6: Commit**

```bash
git add predator-sense-gui/src/ui/fan_curve_geometry.rs predator-sense-gui/src/ui/mod.rs predator-sense-gui/src/hardware/fan.rs
git commit -m "Add the curve chart's geometry, tested without a display

The axis runs 35 C to 95 C so each of the curve's six bands is one
ten-degree column and the temperature marker lands in the band actually
in force. A test asserts every breakpoint in fan.rs is a column boundary,
so the drawing cannot drift from the code that picks the step.

Also drops an is_none_or that had crept past the 1.80 floor the sibling
crates declare."
```

---

### Task 2: The curve chart widget

**Files:**
- Create: `predator-sense-gui/src/ui/fan_curve_widget.rs`
- Modify: `predator-sense-gui/src/ui/mod.rs` (register the module)

**Interfaces:**
- Consumes: `ui::fan_curve_geometry::*`, `fan::curve_resume_c`,
  `ui::brand_theme::accent()`, `i18n::t`.
- Produces:
  ```rust
  pub struct FanCurveView { pub widget: gtk::DrawingArea, /* private state */ }
  pub fn build(steps: [u8; 6]) -> FanCurveView
  impl FanCurveView {
      pub fn set_on_change(&self, f: impl Fn([u8; 6]) + 'static);
      pub fn set_steps(&self, steps: [u8; 6]);
      pub fn set_temp(&self, temp_c: Option<f64>);
      pub fn steps(&self) -> [u8; 6];
  }
  ```

Two rules the implementer must follow:

1. `set_on_change` is a separate setter, not a `build` argument, because the
   page's callback needs the page state that owns this view. The page passes a
   closure holding a `std::rc::Weak`, so the callback must tolerate being
   called after its target is gone.
2. `set_steps` must NOT call the change callback. It is how the page pushes
   config into the widget; calling back would turn every refresh into a write.
   Only a gesture calls back, and only when the value actually changed.

- [ ] **Step 1: Write the widget**

```rust
//! The fan curve as a step chart: six draggable bars, the shaded band where
//! the firmware owns the fans, and a marker at the current temperature.
//!
//! A step chart and not a smooth line, because the curve is stepped: a line
//! would draw speeds the reconciler never applies. The temperature
//! breakpoints are fixed in `fan.rs`, so dragging is vertical only.
//!
//! Every pixel-to-value conversion lives in `fan_curve_geometry`, which is
//! unit tested; what is left here is Cairo and gestures, which are not.

use gtk4::prelude::*;
use gtk4::{self as gtk, cairo};
use std::cell::RefCell;
use std::rc::Rc;

use crate::hardware::fan;
use crate::ui::fan_curve_geometry as geo;

struct State {
    steps: [u8; geo::STEP_COUNT],
    temp_c: Option<f64>,
    on_change: Option<Rc<dyn Fn([u8; geo::STEP_COUNT])>>,
}

pub struct FanCurveView {
    pub widget: gtk::DrawingArea,
    state: Rc<RefCell<State>>,
}

/// Height the chart asks for. Wide enough for six columns to be draggable
/// with a mouse without the page needing a scrollbar.
const CHART_HEIGHT: i32 = 170;
const CHART_MIN_WIDTH: i32 = 320;

pub fn build(steps: [u8; geo::STEP_COUNT]) -> FanCurveView {
    let widget = gtk::DrawingArea::new();
    widget.set_content_height(CHART_HEIGHT);
    widget.set_content_width(CHART_MIN_WIDTH);
    widget.set_hexpand(true);

    let state = Rc::new(RefCell::new(State {
        steps,
        temp_c: None,
        on_change: None,
    }));

    {
        let state = state.clone();
        widget.set_draw_func(move |_area, cr, w, h| {
            let s = state.borrow();
            draw(cr, w as f64, h as f64, s.steps, s.temp_c);
        });
    }

    // One edit path for both gestures: a click sets a value, a drag keeps
    // setting it. The callback fires only when the value really changed, so a
    // drag across one bar writes config once per five percent, not once per
    // motion event.
    let edit = {
        let state = state.clone();
        let widget = widget.clone();
        Rc::new(move |x: f64, y: f64| {
            let plot = geo::plot_for(widget.width() as f64, widget.height() as f64);
            let (changed, steps) = {
                let mut s = state.borrow_mut();
                let next = geo::edited(s.steps, plot, x, y);
                let changed = next != s.steps;
                s.steps = next;
                (changed, next)
            };
            if !changed {
                return;
            }
            widget.queue_draw();
            // The callback is cloned out and the borrow ends before it runs:
            // it reaches back into the page, which touches this same
            // RefCell. Holding the borrow across it would panic as soon as
            // the page pushed anything back.
            let callback = state.borrow().on_change.clone();
            if let Some(f) = callback {
                f(steps);
            }
        })
    };

    let click = gtk::GestureClick::new();
    {
        let edit = edit.clone();
        click.connect_pressed(move |_, _, x, y| edit(x, y));
    }
    widget.add_controller(click);

    let drag = gtk::GestureDrag::new();
    {
        let edit = edit.clone();
        drag.connect_drag_update(move |gesture, dx, dy| {
            if let Some((sx, sy)) = gesture.start_point() {
                edit(sx + dx, sy + dy);
            }
        });
    }
    widget.add_controller(drag);

    FanCurveView { widget, state }
}

impl FanCurveView {
    /// The callback a gesture fires. Never fired by `set_steps`.
    pub fn set_on_change(&self, f: impl Fn([u8; geo::STEP_COUNT]) + 'static) {
        self.state.borrow_mut().on_change = Some(Rc::new(f));
    }

    pub fn set_steps(&self, steps: [u8; geo::STEP_COUNT]) {
        if self.state.borrow().steps == steps {
            return;
        }
        self.state.borrow_mut().steps = steps;
        self.widget.queue_draw();
    }

    pub fn steps(&self) -> [u8; geo::STEP_COUNT] {
        self.state.borrow().steps
    }

    /// The live temperature marker, or `None` when no sensor answered.
    pub fn set_temp(&self, temp_c: Option<f64>) {
        if self.state.borrow().temp_c == temp_c {
            return;
        }
        self.state.borrow_mut().temp_c = temp_c;
        self.widget.queue_draw();
    }
}

fn label(cr: &cairo::Context, x: f64, y: f64, text: &str, size: f64) {
    cr.select_font_face("Sans", cairo::FontSlant::Normal, cairo::FontWeight::Normal);
    cr.set_font_size(size);
    let _ = cr.move_to(x, y);
    let _ = cr.show_text(text);
}

fn draw(
    cr: &cairo::Context,
    width: f64,
    height: f64,
    steps: [u8; geo::STEP_COUNT],
    temp_c: Option<f64>,
) {
    let plot = geo::plot_for(width, height);
    if plot.w <= 0.0 || plot.h <= 0.0 {
        return;
    }
    let accent = crate::ui::brand_theme::accent().bright;

    // Plot background.
    cr.set_source_rgba(1.0, 1.0, 1.0, 0.03);
    cr.rectangle(plot.x, plot.y, plot.w, plot.h);
    let _ = cr.fill();

    // The band the firmware owns, drawn under the bars: a zero step does not
    // mean "off until the next breakpoint", it means the firmware keeps the
    // fans until the whole first non-zero band is cleared.
    if let Some(end) = geo::firmware_region_end_x(plot, fan::curve_resume_c(&steps)) {
        if steps[0] == 0 {
            cr.set_source_rgba(0.55, 0.62, 0.72, 0.16);
            cr.rectangle(plot.x, plot.y, end - plot.x, plot.h);
            let _ = cr.fill();
            cr.set_source_rgba(0.75, 0.80, 0.88, 0.75);
            label(
                cr,
                plot.x + 4.0,
                plot.y + 12.0,
                crate::i18n::t("fan_curve_firmware_band"),
                9.0,
            );
        }
    }

    // Percent gridlines.
    for pct in [0u8, 25, 50, 75, 100] {
        let y = geo::y_for_pct(plot, pct);
        cr.set_source_rgba(1.0, 1.0, 1.0, 0.08);
        cr.set_line_width(1.0);
        cr.move_to(plot.x, y);
        cr.line_to(plot.x + plot.w, y);
        let _ = cr.stroke();
        cr.set_source_rgba(0.72, 0.76, 0.82, 0.85);
        label(cr, 4.0, y + 3.0, &format!("{pct}%"), 9.0);
    }

    // The step in force, so the marker and the bar agree.
    let live_step = temp_c.map(|t| geo::step_at_x(plot, geo::x_for_temp(plot, t)));

    for (i, &pct) in steps.iter().enumerate() {
        let (left, right) = geo::column(plot, i);
        let top = geo::y_for_pct(plot, pct);
        let lit = live_step == Some(i);
        // A zero step has no bar to draw; the shaded band already says what
        // happens there. Without this it drew a zero-height sliver that read
        // as a very low fan speed.
        if pct > 0 {
            cr.set_source_rgba(accent.0, accent.1, accent.2, if lit { 0.85 } else { 0.45 });
            cr.rectangle(left + 1.0, top, (right - left) - 2.0, plot.y + plot.h - top);
            let _ = cr.fill();
        }
        cr.set_source_rgba(accent.0, accent.1, accent.2, if lit { 1.0 } else { 0.6 });
        cr.set_line_width(if lit { 2.0 } else { 1.0 });
        cr.move_to(left + 1.0, top);
        cr.line_to(right - 1.0, top);
        let _ = cr.stroke();

        cr.set_source_rgba(0.92, 0.94, 0.97, if lit { 1.0 } else { 0.7 });
        label(cr, left + 6.0, (top - 4.0).max(plot.y + 9.0), &format!("{pct}"), 10.0);
    }

    // Column boundaries and their temperatures.
    cr.set_source_rgba(0.72, 0.76, 0.82, 0.8);
    for (i, &c) in fan::FAN_CURVE_BREAKPOINTS_C.iter().enumerate() {
        let x = geo::x_for_temp(plot, c);
        cr.set_source_rgba(1.0, 1.0, 1.0, 0.07);
        cr.set_line_width(1.0);
        cr.move_to(x, plot.y);
        cr.line_to(x, plot.y + plot.h);
        let _ = cr.stroke();
        cr.set_source_rgba(0.72, 0.76, 0.82, 0.85);
        let text = format!("{}", c as i32);
        label(cr, x - 7.0, plot.y + plot.h + 13.0, &text, 9.0);
        let _ = i;
    }

    // Live temperature marker.
    if let Some(t) = temp_c {
        let x = geo::x_for_temp(plot, t);
        cr.set_source_rgba(1.0, 0.85, 0.35, 0.95);
        cr.set_line_width(2.0);
        cr.move_to(x, plot.y);
        cr.line_to(x, plot.y + plot.h);
        let _ = cr.stroke();
        label(cr, (x + 4.0).min(plot.x + plot.w - 30.0), plot.y + plot.h - 4.0, &format!("{}C", t as i32), 9.0);
    }
}
```

Register it in `predator-sense-gui/src/ui/mod.rs`, after
`fan_curve_geometry`:

```rust
pub mod fan_curve_widget;
```

- [ ] **Step 2: Verify it builds warning-free**

Run: `cd predator-sense-gui && cargo build --release 2>&1 | grep -E 'warning|error' ; cargo test`
Expected: no warnings naming `fan_curve_widget`, and 273 tests still pass.

- [ ] **Step 3: Commit**

```bash
git add predator-sense-gui/src/ui/fan_curve_widget.rs predator-sense-gui/src/ui/mod.rs
git commit -m "Draw the fan curve as a draggable step chart

Six bars on a ten-degree grid, the firmware-owned band shaded behind
them, and a marker at the current temperature that lights the step in
force. Stepped rather than smooth because the curve is stepped: a line
would show speeds the reconciler never writes.

set_steps deliberately does not fire the change callback, so pushing
config into the widget cannot turn into a write back to it."
```

---

### Task 3: One mode picker and one plan selector

**Files:**
- Modify: `predator-sense-gui/src/ui/fan_control_page.rs` (replace everything
  from `pub fn build()` down to the `// Fan cards` comment; keep the fan
  cards, `build_fan_card`, `draw_blade_ring` and `draw_animated_fan`)
- Modify: `predator-sense-gui/src/i18n.rs` (`t_pt` and `t_en` only)

**Interfaces:**
- Consumes: `fan::{plan_for, plan_for_id, with_plan, curve_resume_c,
  DEFAULT_FAN_CURVE, get_coolboost, set_coolboost}`,
  `config::{FanPlan, load_app_config, save_app_config}`,
  `profile::{PowerProfile, get_current_profile}`,
  `ui::fan_curve_widget::FanCurveView`, `ui::background::run`.
- Produces: nothing other crates consume.

What must be gone when this task is done: the `Automatic / Max / Custom`
button row, the CPU and GPU custom sliders with their Apply button, the
`Auto curve` switch, and every write to `cfg.fan_auto_curve_enabled` and
`cfg.fan_mode`. Those two config fields stay in `config.rs` and keep their
one remaining reader, `fan::migrate_plans`, which is what carries an existing
user's old settings forward.

The bug this replaces: the `Auto curve` switch and the `Automatic` button both
wrote the active mode's plan, so clicking `Automatic` silently deleted a
running curve and every later step edit then did nothing. Verified on
hardware on 2026-09-15: the curve tracked 35% at 54 C and 65% at 66 C until
`Automatic` was clicked, after which the plan read `Automatic` and the step
edits had no curve to reach.

Structural rule for this page: the mode chips and the plan buttons are
`gtk::Button`, never `ToggleButton` or `Switch`, and selection is shown by
swapping the `accent-button` and `secondary-button` CSS classes. A
`connect_clicked` cannot fire from a programmatic refresh, which is what made
the old switch destructive. The spin buttons and the chart do need a guard:
every programmatic `set_value` / `set_steps` happens between
`state.syncing.set(true)` and `state.syncing.set(false)`, and every handler
returns early while it is set.

- [ ] **Step 1: Add the English and Portuguese strings**

In `src/i18n.rs`, add to `t_en` (near the existing `"fan_auto_curve"` line)
and to `t_pt` (at the matching place):

```rust
// t_en
"fan_editing_mode" => "Editing",
"fan_active_mode" => "Active",
"fan_plan_for" => "Fan plan for {0}",
"fan_plan_curve" => "Curve",
"fan_plan_fixed" => "Fixed",
"fan_plan_automatic_desc" => "The firmware decides. The fans can stop completely.",
"fan_plan_curve_desc" => "Your own speed for each temperature band, below.",
"fan_plan_fixed_desc" => "One speed, whatever the temperature.",
"fan_plan_max_desc" => "Full speed, all the time.",
"fan_plan_saved_other_mode" => "Saved. Takes effect when you switch to {0}.",
"fan_curve_presets" => "Presets",
"fan_curve_preset_silent" => "Silent",
"fan_curve_preset_balanced" => "Balanced",
"fan_curve_preset_aggressive" => "Aggressive",
"fan_curve_firmware_band" => "firmware",
"fan_coolboost_desc" => "Firmware fan boost. Independent of the plan above.",
"fan_fixed_percent" => "Speed",
```

```rust
// t_pt
"fan_editing_mode" => "Editando",
"fan_active_mode" => "Ativo",
"fan_plan_for" => "Plano de ventoinha para {0}",
"fan_plan_curve" => "Curva",
"fan_plan_fixed" => "Fixa",
"fan_plan_automatic_desc" => "O firmware decide. As ventoinhas podem parar completamente.",
"fan_plan_curve_desc" => "Sua própria velocidade para cada faixa de temperatura, abaixo.",
"fan_plan_fixed_desc" => "Uma velocidade só, qualquer que seja a temperatura.",
"fan_plan_max_desc" => "Velocidade máxima, o tempo todo.",
"fan_plan_saved_other_mode" => "Salvo. Vale quando você mudar para {0}.",
"fan_curve_presets" => "Predefinições",
"fan_curve_preset_silent" => "Silencioso",
"fan_curve_preset_balanced" => "Equilibrado",
"fan_curve_preset_aggressive" => "Agressivo",
"fan_curve_firmware_band" => "firmware",
"fan_coolboost_desc" => "Turbo de ventoinha do firmware. Independente do plano acima.",
"fan_fixed_percent" => "Velocidade",
```

Reuse the existing `"automatic"`, `"max"`, `"fan_title"`,
`"fan_no_pwm_note"` and `"fan_plan_applying"` keys rather than adding
synonyms.

- [ ] **Step 2: Rewrite the page**

Replace the top of `fan_control_page.rs` (the imports, the
`apply_steps_to_active_plan` helper which is no longer needed, and `build()`
up to the `// Fan cards` comment) with this. Everything from `// Fan cards`
down is unchanged except where Step 3 says otherwise.

```rust
use gtk4::prelude::*;
use gtk4::{self as gtk, glib};
use std::cell::{Cell, RefCell};
use std::f64::consts::PI;
use std::rc::{Rc, Weak};

use crate::config::{self, FanPlan};
use crate::hardware::profile::PowerProfile;
use crate::hardware::{fan, sensors};
use crate::ui::background;
use crate::ui::fan_curve_widget::{self, FanCurveView};

/// The five power modes a plan can be bound to, in the order the rest of the
/// app lists them.
const MODES: [PowerProfile; 5] = [
    PowerProfile::Eco,
    PowerProfile::Quiet,
    PowerProfile::Balanced,
    PowerProfile::Performance,
    PowerProfile::Turbo,
];

/// Starting points for someone who does not want to drag six bars. Silent
/// leans on the firmware below 55 C, which is where the reconciler hands the
/// fans back anyway when the first step is zero (see `fan::curve_resume_c`).
const PRESET_SILENT: [u8; 6] = [0, 0, 30, 45, 65, 100];
const PRESET_AGGRESSIVE: [u8; 6] = [40, 55, 70, 85, 100, 100];

/// Which of the four plan buttons a plan lights up.
#[derive(Clone, Copy, PartialEq, Eq)]
enum PlanKind {
    Automatic,
    Curve,
    Fixed,
    Max,
}

impl PlanKind {
    fn of(plan: FanPlan) -> Self {
        match plan {
            FanPlan::Automatic => PlanKind::Automatic,
            FanPlan::Curve { .. } => PlanKind::Curve,
            FanPlan::Fixed { .. } => PlanKind::Fixed,
            FanPlan::Max => PlanKind::Max,
        }
    }

    fn label(self) -> &'static str {
        match self {
            PlanKind::Automatic => crate::i18n::t("automatic"),
            PlanKind::Curve => crate::i18n::t("fan_plan_curve"),
            PlanKind::Fixed => crate::i18n::t("fan_plan_fixed"),
            PlanKind::Max => crate::i18n::t("max"),
        }
    }

    fn description(self) -> &'static str {
        match self {
            PlanKind::Automatic => crate::i18n::t("fan_plan_automatic_desc"),
            PlanKind::Curve => crate::i18n::t("fan_plan_curve_desc"),
            PlanKind::Fixed => crate::i18n::t("fan_plan_fixed_desc"),
            PlanKind::Max => crate::i18n::t("fan_plan_max_desc"),
        }
    }
}

/// Everything a refresh touches. One struct so the handlers can hold a single
/// `Weak` and nothing has to be cloned five ways.
struct Page {
    /// Mode id whose plan the controls below are editing.
    editing: RefCell<String>,
    /// While false the chips follow the machine's active mode, so opening the
    /// page always edits what is running. A chip click pins it.
    pinned: Cell<bool>,
    /// Set while a refresh is pushing config into widgets whose handlers
    /// would otherwise write it straight back.
    syncing: Cell<bool>,
    chips: Vec<(gtk::Button, String)>,
    plan_buttons: Vec<(gtk::Button, PlanKind)>,
    plan_title: gtk::Label,
    plan_desc: gtk::Label,
    fixed_box: gtk::Box,
    fixed_spin: gtk::SpinButton,
    curve_box: gtk::Box,
    curve_view: FanCurveView,
    spins: Vec<gtk::SpinButton>,
    status: gtk::Label,
}

/// The plan bound to a mode id, with `Automatic` for a mode that has none -
/// the same fallback `fan::plan_for` uses.
fn plan_of(cfg: &config::AppConfig, mode_id: &str) -> FanPlan {
    fan::plan_for_id(&cfg.fan_plans, mode_id).unwrap_or(FanPlan::Automatic)
}

/// Binds `plan` to `mode_id`. Intent only: the reconciler applies it.
fn write_plan(mode_id: &str, plan: FanPlan) -> Result<(), String> {
    let mut cfg = config::load_app_config();
    cfg.fan_plans = fan::with_plan(&cfg.fan_plans, mode_id, plan);
    config::save_app_config(&cfg)
}

/// Binds a curve and remembers its steps globally.
///
/// `fan_curve_points` no longer drives the fans - the steps inside the mode's
/// own `FanPlan::Curve` do - but it is still the seed for a mode that has
/// never had a curve, so the last curve edited anywhere is where the next
/// mode starts rather than the hardcoded default.
fn write_curve(mode_id: &str, steps: [u8; 6]) -> Result<(), String> {
    let mut cfg = config::load_app_config();
    cfg.fan_plans = fan::with_plan(&cfg.fan_plans, mode_id, FanPlan::Curve { steps });
    cfg.fan_curve_points = steps;
    config::save_app_config(&cfg)
}

fn mode_label(mode_id: &str) -> &'static str {
    MODES
        .iter()
        .find(|profile| profile.to_id() == mode_id)
        .map_or("", |profile| profile.label())
}

/// Pushes config into every control. The only place that decides what the
/// page shows, so the chips, the buttons and the editors cannot disagree.
fn refresh(page: &Rc<Page>) {
    let cfg = config::load_app_config();
    let editing = page.editing.borrow().clone();
    let plan = plan_of(&cfg, &editing);
    let kind = PlanKind::of(plan);

    for (button, id) in &page.chips {
        button.remove_css_class("accent-button");
        button.remove_css_class("secondary-button");
        button.add_css_class(if *id == editing {
            "accent-button"
        } else {
            "secondary-button"
        });
    }
    for (button, button_kind) in &page.plan_buttons {
        button.remove_css_class("accent-button");
        button.remove_css_class("secondary-button");
        button.add_css_class(if *button_kind == kind {
            "accent-button"
        } else {
            "secondary-button"
        });
    }

    page.plan_title
        .set_text(&crate::i18n::tf("fan_plan_for", &[mode_label(&editing)]));
    page.plan_desc.set_text(kind.description());
    page.fixed_box.set_visible(kind == PlanKind::Fixed);
    page.curve_box.set_visible(kind == PlanKind::Curve);

    let steps = match plan {
        FanPlan::Curve { steps } => steps,
        _ => cfg.fan_curve_points,
    };
    page.syncing.set(true);
    if let FanPlan::Fixed { percent } = plan {
        page.fixed_spin.set_value(f64::from(percent));
    }
    page.curve_view.set_steps(steps);
    for (spin, &pct) in page.spins.iter().zip(steps.iter()) {
        spin.set_value(f64::from(pct));
    }
    page.syncing.set(false);
}

/// Says what a write did, including the case that confuses people most:
/// editing a mode you are not currently in.
fn report(page: &Rc<Page>, result: Result<(), String>) {
    let editing = page.editing.borrow().clone();
    let active = crate::hardware::profile::get_current_profile().map(|p| p.to_id().to_string());
    match result {
        Ok(()) => {
            let text = if active.as_deref() == Some(editing.as_str()) {
                crate::i18n::t("fan_plan_applying").to_string()
            } else {
                crate::i18n::tf("fan_plan_saved_other_mode", &[mode_label(&editing)])
            };
            page.status.set_text(&text);
            page.status.remove_css_class("status-error");
            page.status.add_css_class("status-success");
        }
        Err(error) => {
            page.status.set_text(&error);
            page.status.remove_css_class("status-success");
            page.status.add_css_class("status-error");
        }
    }
}

pub fn build() -> gtk::Box {
    let page_box = gtk::Box::new(gtk::Orientation::Vertical, 8);
    page_box.set_margin_top(14);
    page_box.set_margin_bottom(10);
    page_box.set_margin_start(20);
    page_box.set_margin_end(20);

    let caps = crate::hardware::capabilities::get();
    let cfg = config::load_app_config();

    let header = gtk::Label::new(Some("AeroBlade™ 3D Fan"));
    header.add_css_class("fan-rpm");
    header.set_halign(gtk::Align::End);
    page_box.append(&header);

    let title = gtk::Label::new(Some(crate::i18n::t("fan_title")));
    title.add_css_class("section-title");
    title.set_halign(gtk::Align::Center);
    page_box.append(&title);

    // ---- mode chips ----
    let editing_row = gtk::Box::new(gtk::Orientation::Horizontal, 8);
    editing_row.set_halign(gtk::Align::Center);
    editing_row.set_margin_top(6);
    let editing_label = gtk::Label::new(Some(crate::i18n::t("fan_editing_mode")));
    editing_label.add_css_class("info-text-dim");
    editing_row.append(&editing_label);

    let mut chips: Vec<(gtk::Button, String)> = Vec::new();
    for profile in MODES {
        let button = gtk::Button::with_label(profile.label());
        button.add_css_class("secondary-button");
        editing_row.append(&button);
        chips.push((button, profile.to_id().to_string()));
    }
    page_box.append(&editing_row);

    // ---- plan selector ----
    let plan_title = gtk::Label::new(None);
    plan_title.add_css_class("control-label");
    plan_title.set_halign(gtk::Align::Center);
    plan_title.set_margin_top(10);
    page_box.append(&plan_title);

    let plan_row = gtk::Box::new(gtk::Orientation::Horizontal, 8);
    plan_row.set_halign(gtk::Align::Center);
    plan_row.set_margin_top(4);

    // Curve and Fixed need per-fan PWM. Without it the EC has only its own
    // Auto and Max presets, so offering the other two would be offering
    // something `fan::plan_for_hardware` immediately resolves away.
    let kinds: Vec<PlanKind> = if caps.fan_pwm {
        vec![
            PlanKind::Automatic,
            PlanKind::Curve,
            PlanKind::Fixed,
            PlanKind::Max,
        ]
    } else {
        vec![PlanKind::Automatic, PlanKind::Max]
    };
    let mut plan_buttons: Vec<(gtk::Button, PlanKind)> = Vec::new();
    for kind in kinds {
        let button = gtk::Button::with_label(kind.label());
        button.add_css_class("secondary-button");
        plan_row.append(&button);
        plan_buttons.push((button, kind));
    }
    page_box.append(&plan_row);

    let plan_desc = gtk::Label::new(None);
    plan_desc.add_css_class("info-note");
    plan_desc.set_halign(gtk::Align::Center);
    plan_desc.set_wrap(true);
    plan_desc.set_justify(gtk::Justification::Center);
    page_box.append(&plan_desc);

    // ---- Fixed ----
    let fixed_box = gtk::Box::new(gtk::Orientation::Horizontal, 8);
    fixed_box.set_halign(gtk::Align::Center);
    fixed_box.set_margin_top(6);
    let fixed_label = gtk::Label::new(Some(crate::i18n::t("fan_fixed_percent")));
    fixed_label.add_css_class("info-text-dim");
    let fixed_spin = gtk::SpinButton::with_range(0.0, 100.0, 5.0);
    fixed_box.append(&fixed_label);
    fixed_box.append(&fixed_spin);
    fixed_box.set_visible(false);
    page_box.append(&fixed_box);

    // ---- Curve ----
    let curve_box = gtk::Box::new(gtk::Orientation::Vertical, 6);
    curve_box.set_margin_top(6);
    curve_box.set_visible(false);
    let seed = match plan_of(&cfg, MODES[2].to_id()) {
        FanPlan::Curve { steps } => steps,
        _ => cfg.fan_curve_points,
    };
    let curve_view = fan_curve_widget::build(seed);
    curve_box.append(&curve_view.widget);

    let steps_row = gtk::Box::new(gtk::Orientation::Horizontal, 10);
    steps_row.set_halign(gtk::Align::Center);
    const BREAKPOINT_LABELS: [&str; 6] =
        ["< 45°C", "< 55°C", "< 65°C", "< 75°C", "< 85°C", "≥ 85°C"];
    let mut spins: Vec<gtk::SpinButton> = Vec::new();
    for (i, text) in BREAKPOINT_LABELS.iter().enumerate() {
        let column = gtk::Box::new(gtk::Orientation::Vertical, 2);
        let label = gtk::Label::new(Some(text));
        label.add_css_class("info-text-dim");
        let spin = gtk::SpinButton::with_range(0.0, 100.0, 5.0);
        spin.set_value(f64::from(seed[i]));
        column.append(&label);
        column.append(&spin);
        steps_row.append(&column);
        spins.push(spin);
    }
    curve_box.append(&steps_row);

    let presets_row = gtk::Box::new(gtk::Orientation::Horizontal, 8);
    presets_row.set_halign(gtk::Align::Center);
    presets_row.set_margin_top(4);
    let presets_label = gtk::Label::new(Some(crate::i18n::t("fan_curve_presets")));
    presets_label.add_css_class("info-text-dim");
    presets_row.append(&presets_label);
    let preset_buttons: Vec<(gtk::Button, [u8; 6])> = vec![
        (
            gtk::Button::with_label(crate::i18n::t("fan_curve_preset_silent")),
            PRESET_SILENT,
        ),
        (
            gtk::Button::with_label(crate::i18n::t("fan_curve_preset_balanced")),
            fan::DEFAULT_FAN_CURVE,
        ),
        (
            gtk::Button::with_label(crate::i18n::t("fan_curve_preset_aggressive")),
            PRESET_AGGRESSIVE,
        ),
    ];
    for (button, _) in &preset_buttons {
        button.add_css_class("secondary-button");
        presets_row.append(button);
    }
    curve_box.append(&presets_row);
    page_box.append(&curve_box);

    let status = gtk::Label::new(None);
    status.add_css_class("status-label");
    page_box.append(&status);

    if !caps.fan_pwm {
        let note = gtk::Label::new(Some(crate::i18n::t("fan_no_pwm_note")));
        note.add_css_class("info-note");
        note.set_halign(gtk::Align::Center);
        note.set_justify(gtk::Justification::Center);
        note.set_wrap(true);
        page_box.append(&note);
    }

    // ---- CoolBoost, its own section: a firmware boost that is not a plan ----
    if caps.ec {
        let cb_title = gtk::Label::new(Some("CoolBoost™"));
        cb_title.add_css_class("section-title");
        cb_title.set_halign(gtk::Align::Center);
        cb_title.set_margin_top(12);
        page_box.append(&cb_title);

        let cb_row = gtk::Box::new(gtk::Orientation::Horizontal, 10);
        cb_row.set_halign(gtk::Align::Center);
        let cb_desc = gtk::Label::new(Some(crate::i18n::t("fan_coolboost_desc")));
        cb_desc.add_css_class("info-note");
        cb_desc.set_wrap(true);
        let cb_switch = gtk::Switch::new();
        cb_switch.set_valign(gtk::Align::Center);
        cb_switch.set_sensitive(false);
        cb_row.append(&cb_desc);
        cb_row.append(&cb_switch);
        page_box.append(&cb_row);

        // The EC read costs roughly 150 ms through the helper, so it happens
        // on a worker and the switch stays insensitive until it answers.
        // Connecting the handler only after `set_active` is what keeps the
        // read from writing the value straight back.
        background::run(fan::get_coolboost, move |enabled| {
            cb_switch.set_active(enabled);
            cb_switch.connect_state_set(move |_, enabled| {
                let _ = fan::set_coolboost(enabled);
                let mut c = config::load_app_config();
                c.coolboost_enabled = enabled;
                let _ = config::save_app_config(&c);
                glib::Propagation::Proceed
            });
            cb_switch.set_sensitive(true);
        });
    }

    // ---- wire it up ----
    let initial = crate::hardware::profile::get_current_profile()
        .map_or_else(|| MODES[2].to_id().to_string(), |p| p.to_id().to_string());
    let page = Rc::new(Page {
        editing: RefCell::new(initial),
        pinned: Cell::new(false),
        syncing: Cell::new(false),
        chips,
        plan_buttons,
        plan_title,
        plan_desc,
        fixed_box,
        fixed_spin,
        curve_box,
        curve_view,
        spins,
        status,
    });

    for (button, mode_id) in &page.chips {
        let weak: Weak<Page> = Rc::downgrade(&page);
        let mode_id = mode_id.clone();
        button.connect_clicked(move |_| {
            let Some(page) = weak.upgrade() else { return };
            *page.editing.borrow_mut() = mode_id.clone();
            page.pinned.set(true);
            page.status.set_text("");
            refresh(&page);
        });
    }

    for (button, kind) in &page.plan_buttons {
        let weak: Weak<Page> = Rc::downgrade(&page);
        let kind = *kind;
        button.connect_clicked(move |_| {
            let Some(page) = weak.upgrade() else { return };
            let mode_id = page.editing.borrow().clone();
            let result = match kind {
                PlanKind::Automatic => write_plan(&mode_id, FanPlan::Automatic),
                PlanKind::Max => write_plan(&mode_id, FanPlan::Max),
                PlanKind::Fixed => write_plan(
                    &mode_id,
                    FanPlan::Fixed {
                        percent: page.fixed_spin.value() as u8,
                    },
                ),
                // Seeded from whatever the chart is showing, which `refresh`
                // set from this mode's own curve or from the last curve
                // edited anywhere.
                PlanKind::Curve => write_curve(&mode_id, page.curve_view.steps()),
            };
            report(&page, result);
            refresh(&page);
        });
    }

    {
        let weak: Weak<Page> = Rc::downgrade(&page);
        page.fixed_spin.connect_value_changed(move |spin| {
            let Some(page) = weak.upgrade() else { return };
            if page.syncing.get() {
                return;
            }
            let mode_id = page.editing.borrow().clone();
            let result = write_plan(
                &mode_id,
                FanPlan::Fixed {
                    percent: spin.value() as u8,
                },
            );
            report(&page, result);
        });
    }

    for (i, spin) in page.spins.iter().enumerate() {
        let weak: Weak<Page> = Rc::downgrade(&page);
        spin.connect_value_changed(move |spin| {
            let Some(page) = weak.upgrade() else { return };
            if page.syncing.get() {
                return;
            }
            let mut steps = page.curve_view.steps();
            steps[i] = spin.value() as u8;
            let mode_id = page.editing.borrow().clone();
            let result = write_curve(&mode_id, steps);
            page.curve_view.set_steps(steps);
            report(&page, result);
        });
    }

    {
        let weak: Weak<Page> = Rc::downgrade(&page);
        page.curve_view.set_on_change(move |steps| {
            let Some(page) = weak.upgrade() else { return };
            let mode_id = page.editing.borrow().clone();
            let result = write_curve(&mode_id, steps);
            page.syncing.set(true);
            for (spin, &pct) in page.spins.iter().zip(steps.iter()) {
                spin.set_value(f64::from(pct));
            }
            page.syncing.set(false);
            report(&page, result);
        });
    }

    for (button, steps) in preset_buttons {
        let weak: Weak<Page> = Rc::downgrade(&page);
        button.connect_clicked(move |_| {
            let Some(page) = weak.upgrade() else { return };
            let mode_id = page.editing.borrow().clone();
            let result = write_curve(&mode_id, steps);
            report(&page, result);
            refresh(&page);
        });
    }

    refresh(&page);

    // Follow the machine: the power mode changes from the Modes page, the
    // physical Predator key and the AI assistant, and a plan changes from the
    // Performance/Turbo setting in Settings. Re-reading on a timer is the same
    // approach the page used before, minus the EC round trip it no longer
    // needs: the plan is in config, not the hardware.
    {
        let weak: Weak<Page> = Rc::downgrade(&page);
        let page_widget = page_box.clone();
        glib::timeout_add_seconds_local(3, move || {
            let Some(page) = weak.upgrade() else {
                return glib::ControlFlow::Break;
            };
            if !crate::app_state::is_window_visible() || !page_widget.is_mapped() {
                return glib::ControlFlow::Continue;
            }
            if !page.pinned.get() {
                if let Some(active) = crate::hardware::profile::get_current_profile() {
                    if *page.editing.borrow() != active.to_id() {
                        *page.editing.borrow_mut() = active.to_id().to_string();
                    }
                }
            }
            refresh(&page);
            glib::ControlFlow::Continue
        });
    }
```

- [ ] **Step 3: Feed the chart, and drive the animation from the plan**

The existing sensor timer already reads temperatures every two seconds. Inside
it, after the existing `cpu_temp_l` / `gpu_temp_l` updates, add the chart feed
and replace the animation speed's dependency on the deleted `active_mode`
string:

```rust
        // The chart's marker and the fan animation both want to know what is
        // actually running, which is the active mode's plan - not the mode
        // being edited above.
        let live = crate::hardware::fan::curve_input_temp(data.cpu_temp, data.gpu_temp);
        if let Some(page) = weak_sensors.upgrade() {
            page.curve_view.set_temp(live);
        }
        let cfg = config::load_app_config();
        let running = crate::hardware::fan::plan_for(
            crate::hardware::profile::get_current_profile(),
            &cfg.fan_plans,
        );
        anim_speed.set(match running {
            config::FanPlan::Max => 0.85,
            config::FanPlan::Automatic => 0.10,
            config::FanPlan::Fixed { percent } => (f64::from(percent) / 100.0).clamp(0.05, 1.0) * 0.5,
            config::FanPlan::Curve { .. } => {
                let avg = f64::from(*cr2.borrow() + *gr2.borrow()) / 2.0;
                (avg / 6000.0).clamp(0.05, 1.0) * 0.5
            }
        });
```

Declare `let anim_speed = Rc::new(Cell::new(0.10f64));` beside the other
`Rc`s before the animation timer, clone it into both timers, and in the
animation timer replace the `match active_anim.borrow().as_str()` block with
`let speed = anim_speed_anim.get();`. `weak_sensors` is
`Rc::downgrade(&page)`.

- [ ] **Step 4: Verify it builds and the suite still passes**

Run: `cd predator-sense-gui && cargo build --release 2>&1 | grep -E '^(warning|error)' ; cargo test`
Expected: no warnings, no errors, 271 passed (273 from Task 1, less the two `plan_with_steps` tests this task deletes).

Two dead-code cases this task creates, handled differently:

- `fan::plan_with_steps` (`fan.rs:454`) existed only to carry an edit from
  `fan_curve_points` into the active mode's plan, which is what
  `apply_steps_to_active_plan` did and what the new page does directly by
  writing `FanPlan::Curve`. Delete the function and its two tests,
  `editing_the_steps_updates_a_curve_plan` and
  `editing_the_steps_leaves_a_non_curve_plan_alone`. It is a pure helper made
  obsolete by this task, not a hardware path.
- `fan::get_fan_mode` loses its last caller: this page was its only user. Do
  NOT delete it. It reads real EC state that other code and the AI assistant
  may still want, and deleting a hardware function is the human partner's
  decision. If it produces a `dead_code` warning, say so in the task report
  with the exact warning text and leave the function alone.

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/ui/fan_control_page.rs predator-sense-gui/src/i18n.rs
git commit -m "Give Fan Control one mode picker and one plan selector

The page had two controls writing the same setting: the Auto curve
switch and the Automatic button both bound the active mode's plan, so
clicking Automatic deleted a running curve and every later step edit
then had no curve to reach. Both are gone. A mode is chosen, then one of
Automatic, Curve, Fixed or Max, and the curve gets the chart plus its
six numbers and three presets.

Selection is CSS on plain buttons, not toggles: a programmatic refresh
cannot fire a click, which is what made the old switch destructive.
CoolBoost moves into its own section, since a firmware boost is not a
plan. Curve and Fixed appear only where per-fan PWM does."
```

---

### Task 4: The other seven languages

**Files:**
- Modify: `predator-sense-gui/src/i18n.rs` (`t_es`, `t_zh`, `t_ja`, `t_ru`,
  `t_de`, `t_it`, `t_tr`)

**Interfaces:**
- Consumes: the seventeen keys Task 3 added to `t_en` and `t_pt`.
- Produces: nothing.

This is transcription. Add the same seventeen keys to each of the seven
remaining language functions, placing them beside that function's existing
`"fan_auto_curve"` entry, keeping each file's existing quoting and the `{0}`
placeholders exactly as they are in the English strings. Match the register
of the surrounding strings in that language. `"fan_curve_firmware_band"`
stays the lowercase word for firmware in each language.

- [ ] **Step 1: Add the keys to all seven remaining functions**

For each of `t_es`, `t_zh`, `t_ja`, `t_ru`, `t_de`, `t_it`, `t_tr`, add
translations of these exact keys:

`fan_editing_mode`, `fan_active_mode`, `fan_plan_for` (keeps `{0}`),
`fan_plan_curve`, `fan_plan_fixed`, `fan_plan_automatic_desc`,
`fan_plan_curve_desc`, `fan_plan_fixed_desc`, `fan_plan_max_desc`,
`fan_plan_saved_other_mode` (keeps `{0}`), `fan_curve_presets`,
`fan_curve_preset_silent`, `fan_curve_preset_balanced`,
`fan_curve_preset_aggressive`, `fan_curve_firmware_band`,
`fan_coolboost_desc`, `fan_fixed_percent`.

- [ ] **Step 2: Verify every language has every key**

Run:

```bash
cd predator-sense-gui
for key in fan_editing_mode fan_active_mode fan_plan_for fan_plan_curve \
           fan_plan_fixed fan_plan_automatic_desc fan_plan_curve_desc \
           fan_plan_fixed_desc fan_plan_max_desc fan_plan_saved_other_mode \
           fan_curve_presets fan_curve_preset_silent fan_curve_preset_balanced \
           fan_curve_preset_aggressive fan_curve_firmware_band \
           fan_coolboost_desc fan_fixed_percent; do
  n=$(grep -c "\"$key\" =>" src/i18n.rs)
  printf '%-32s %s\n' "$key" "$n"
done
```

Expected: every line reads `9`.

Then: `cargo build --release 2>&1 | grep -E '^(warning|error)'` and
`cargo test`. Expected: nothing from the first, 271 passed from the second.

- [ ] **Step 3: Commit**

```bash
git add predator-sense-gui/src/i18n.rs
git commit -m "Translate the new fan plan strings into the other seven languages

A missing key renders as the key itself, so a page that only had English
strings would have shown fan_plan_curve_desc to everyone else."
```

---

## Out of scope

- Extracting the reconciler from `build_main_ui` into a testable function. It
  is a deferred review finding, not a spec requirement, and Phase 1's hardware
  ownership is verified working; the decision functions it calls
  (`plan_target`, `needs_write`, `write_for`, `may_attempt_write`) are already
  tested individually.
- Removing the "Keep fan on Auto in Performance/Turbo" switch from Settings.
  It is issue #41's feature, its description in all nine languages already
  says it rebinds exactly those two modes, and the Fan Control page's own 3s
  refresh shows its effect. Redundant with the new selector, but removing a
  maintainer-shipped setting is the maintainer's call.
- Editable temperature breakpoints, and any change to CoolBoost behaviour
  beyond moving it into its own section. Both out of scope in the spec.

## Self-Review

**Spec coverage.** UI section: mode chips (Task 3), four-way plan selector
with the Automatic/Curve/Fixed/Max descriptions (Task 3), CoolBoost in its own
section (Task 3), Auto curve toggle and Automatic/Max/Custom row gone
(Task 3), step chart with vertical-only dragging, live temperature marker and
shaded firmware region matching `curve_resume_c` (Tasks 1 and 2), presets
(Task 3), the six numbers kept (Task 3), geometry in pure tested functions
(Task 1), `DrawingArea` + `set_draw_func` + `GestureDrag` following
`tech_gauge.rs` (Task 2), editing the active mode applying live and another
mode saving quietly (Task 3's `report`). Testing section: curve geometry
covered in Task 1; plan resolution, per-mode lookup, migration and
`needs_write`/`FanState` were Phase 1 and are already tested. Not covered by
tests, as the spec says: the drawing and the drag feel, which need eyes on
hardware.

**Placeholders.** None: every step has its code or its exact command.

**Type consistency.** `FanCurveView::steps()` is read by Task 3's plan buttons
and spin handlers, and `set_steps`/`set_temp`/`set_on_change` are the only
other methods used. `geo::STEP_COUNT` is 6 and every array in Task 3 is
`[u8; 6]`; those are the same type. `fan::curve_resume_c` and
`fan::FAN_CURVE_BREAKPOINTS_C` are made public in Task 1 before Task 2 reads
them. `PowerProfile::label()` and `to_id()` both exist and return `&str`.
`crate::i18n::tf` returns `String` and takes `&[&str]`, which is how Task 3
calls it.
