# Fan control: one owner, per-mode plans, a visible curve

Date: 2026-09-15
Status: approved in brainstorming, not yet planned
Scope: this fork first. Offer upstream once it has proven itself on hardware.

## Why

Three problems, all measured on a PH16-71 on 2026-09-15.

**The page contradicts itself.** Fan Control offers a CoolBoost toggle, a Fan
Speed selector (Automatic / Max / Custom), an Auto curve toggle and a six-step
curve editor, with no defined relationship between them. "Automatic" means the
firmware curve; "Auto curve" means the app's software curve, which writes
*manual* PWM, so choosing Automatic with Auto curve on displays a mode the
machine is not in. Max plus Auto curve is undefined: the curve overwrites Max
within three seconds. Nothing in the UI says these are exclusive.

**Nothing owns fan state.** The curve tick, `profile.rs` (which forces Max on
Performance/Turbo), the turbo-key watcher, the AI assistant and the page's own
buttons all write fan hardware independently. Instrumentation caught them
disagreeing inside a single startup sequence: `set_fan_mode(Auto)` hands the
fans to the firmware at t=0, and the curve takes them back four seconds later.
An earlier unexplained manual write at 48 C could not be attributed afterwards
because failures were discarded by `let _ =`.

**Fan behaviour cannot follow the power mode.** There is one global fan
setting. Quiet-when-Balanced and aggressive-when-Turbo is not expressible,
even though lighting already binds per mode.

## Decisions

| Question | Decision |
|---|---|
| Audience | This fork first; offer upstream later |
| Mode relationship | Per-mode fan settings, mirroring lighting schemes |
| What a setting holds | Four-way: Automatic, Curve, Fixed percent, Max |
| Placement | Fan Control page with a mode picker at the top |
| Ownership | A single reconciler performs every fan write |
| Curve editing | Graphical step chart, draggable, with the numbers kept |

## Data model

```rust
pub enum FanPlan {
    Automatic,                 // the firmware curve owns the fans
    Curve { steps: [u8; 6] },  // the six fixed breakpoints, percent each
    Fixed { percent: u8 },     // hold one speed
    Max,
}

pub struct FanBinding { pub mode: String, pub plan: FanPlan }  // mode = PowerProfile::to_id()
pub fan_plans: Vec<FanBinding>                                 // in AppConfig
```

Migration, on first run: the existing `fan_auto_curve_enabled`,
`fan_curve_points` and `fan_mode` collapse into one plan applied to every
mode, so behaviour does not change until a mode is deliberately given its own.
The old fields stay readable through serde defaults and stop being written,
and the migration is their only remaining reader: every other consumer (the AI
assistant, the tray, the pages) moves to `fan_plans`, or the two would drift
apart into exactly the disagreement this design removes.

## The reconciler

One tick is the only code in the app that calls `set_pwm_percent`,
`set_pwm_auto` or `set_fan_mode`. It runs on the existing three-second
timer. Its inputs are the active power mode, that mode's plan, and the current
CPU and GPU temperatures. It resolves a target,
compares against `FanState` (what it last successfully applied), and writes
only when `needs_write` says the hardware would actually change.

Everyone else writes intent, never hardware:

| Writer today | After |
|---|---|
| `profile.rs` forcing Max on Performance/Turbo | removed; the mode's plan decides |
| AI assistant calling `set_fan_mode` | writes a plan into config |
| Fan page buttons | write a plan into config |
| Turbo-key watcher | sets the profile only; the reconciler follows |
| Curve tick | becomes the reconciler |

`Curve` keeps the rules built and verified on 2026-09-15: a 0% step hands the
fans to the firmware, because manual `pwm 0` is not off (the EC holds a floor,
~1670 RPM measured), and manual control resumes only at the top of the first
non-zero band, because resuming at the boundary produced a measured
one-minute oscillation between 0 and 2500 RPM.

Non-PWM models: the reconciler translates `Automatic` and `Max` to the EC
preset path. A `Curve` or `Fixed` plan is unreachable through the UI there, but
can still arrive in a config copied from another machine, so the reconciler
resolves either to `Automatic` rather than doing nothing silently, and says so
once in the log. The UI therefore never needs to know which hardware path is
in play.

Failure handling: a failed write sets `FanState::Unknown` and logs, so the
reconciler reasserts rather than trusting a state it never applied. After three
consecutive failures it backs off to one *attempt* every 30 s, not merely one
log line every 30 s, so a dead helper does not mean a privileged call every
three seconds forever. A success resets the back-off.

## UI

```
Fan Control
Editing:   [ Eco ]  [ Quiet ]  [* Balanced ]  [ Performance ]  [ Turbo ]

Fan plan for Balanced
  ( ) Automatic     firmware decides; fans can stop completely
  (*) Curve         graph and steps, below
  ( ) Fixed         [ 40 ]%
  ( ) Max

  [ graphical step chart, draggable, live temperature marker,
    shaded firmware region, presets: Silent / Balanced / Aggressive / Reset ]
  <45C [ 0]  <55C [35]  <65C [50]  <75C [65]  <85C [80]  >=85C [100]

CoolBoost                                                [ off ]
  Firmware fan boost. Independent of the plan above.

CPU fan  2492 RPM  44C              GPU fan  2518 RPM  43C
```

The Auto curve toggle and the Automatic/Max/Custom row are both gone: the plan
selector absorbs them, which is what removes the undefined combinations.

Mode chips behave like lighting schemes: editing the active mode applies live
with autosave, editing another mode saves quietly and takes effect on switch.

The curve widget is a step chart, not a smooth line, because the curve is
stepped and a smooth line would depict behaviour the code does not have.
Boundaries are fixed in code, so dragging is vertical only. The live
temperature marker highlights the step in force. The shaded region shows where
the firmware owns the fans, matching `curve_resume_c` exactly.

Implementation: `ui/fan_curve_widget.rs`, a `DrawingArea` with `set_draw_func`
and a `GestureDrag`, following `ui/tech_gauge.rs`. Geometry (percent to pixel,
hit testing, clamping) lives in pure functions so it can be unit tested.

## Testing

Unit, written test-first: plan resolution for all four variants; per-mode
lookup and fallback; config migration from the four real starting points
(curve on, curve off with `fan_mode: auto`, `fan_mode: max`, nothing set);
capability translation on a non-PWM model; curve geometry; and the existing
`needs_write`/`FanState` pair.

On hardware: each plan applied and read back (`pwm_enable=2`; the step for the
current temperature; `pwm 102` for Fixed 40%; `pwm 255` for Max); a mode switch
applying the new plan within a tick; zero writes across idle minutes; exactly
one writer in the log during a profile switch; and the 0% regression staying a
single transition over several minutes.

Not automatable: the drawing and the drag feel. Those get eyes.

## Suggested phasing

Three phases, each shippable and verifiable on hardware on its own:

1. **Per-mode plans and the reconciler.** Config shape, migration, plan
   resolution, single ownership, other writers converted to intent. No visible
   UI change beyond the existing controls being driven through the new path.
2. **UI consolidation.** Mode picker, four-way plan selector, CoolBoost moved
   to its own section, the Auto curve toggle and the Automatic/Max/Custom row
   removed.
3. **The graphical curve widget**, replacing nothing: the numbers stay.

Phase 1 carries the risk (it touches fan hardware ownership), so it gets the
hardware verification table in full before phase 2 begins.

## Out of scope

A "copy this plan to every mode" button (migration already makes them
identical), editable temperature breakpoints (fixed in code upstream), and any
change to CoolBoost behaviour beyond moving it into its own section.
