# Fan per-mode plans and single ownership: Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every power mode holds its own fan plan, and exactly one reconciler writes fan hardware.

**Architecture:** A `FanPlan` enum per power mode lives in `AppConfig::fan_plans`, mirroring how `ModeBinding` binds lighting schemes to modes. Pure functions in `hardware/fan.rs` resolve (plan, temperatures, current state) into a `CurveAction`, and the existing three-second tick in `ui/window.rs` becomes the only code that calls `set_pwm_percent`, `set_pwm_auto` or `set_fan_mode`. Every other writer is converted to storing a plan in config.

**Tech Stack:** Rust 2021, GTK4 via gtk4-rs, serde/serde_json for config, existing `predator-sense-helper` for privileged writes.

**Spec:** `docs/superpowers/specs/2026-09-15-fan-control-design.md`

## Global Constraints

- No UI change in this phase. The existing Fan Control page keeps its controls; they write plans instead of hardware.
- Tests run with `cargo test` from `predator-sense-gui/`. All three crates must stay green: 228 GUI, 84 installer, 54 protocol as of the starting commit.
- `cargo build --release` must exit 0 before any commit.
- Author every commit as the repo identity already configured (`fawaz7 <ghazawe1@gmail.com>`). No AI attribution lines anywhere.
- No em dashes in code comments, commit messages or docs.
- Hardware claims require a measurement in the same session. A zero exit code is not evidence: USB writes have returned success while changing nothing, and the EC has accepted fan bytes it then ignored.
- Existing names this plan builds on, do not rename them: `FanState::{Unknown, Firmware, Manual}`, `needs_write(CurveAction, FanState) -> bool`, `CurveAction::{Firmware, Manual, Hold}`, `curve_action(f64, &[u8;6], bool) -> CurveAction`, `curve_input_temp(Option<f64>, Option<f64>) -> Option<f64>`, `fan_curve_pct(f64, &[u8;6]) -> u8`, `DEFAULT_FAN_CURVE: [u8;6]`, `set_pwm_percent(u8,u8) -> Result<(),String>`, `set_pwm_auto() -> Result<(),String>`, `set_fan_mode(FanMode) -> Result<(),String>`, `FanMode::{Auto,Max,Custom}`, `PowerProfile::{Quiet,Balanced,Performance,Turbo,Eco}` with `to_id()`/`from_id()`, `capabilities::get().fan_pwm`, `sensors::read_critical_temps()`, `profile::get_current_profile()`.

---

### Task 1: The FanPlan type

**Files:**
- Modify: `src/config.rs` (add the type next to `ModeBinding`, and the `AppConfig` field)
- Test: `src/config.rs` (inline `#[cfg(test)] mod` at the end of the file, matching the crate's convention)

**Interfaces:**
- Consumes: nothing
- Produces: `config::FanPlan` (`Automatic`, `Curve { steps: [u8; 6] }`, `Fixed { percent: u8 }`, `Max`), `config::FanBinding { mode: String, plan: FanPlan }`, `AppConfig::fan_plans: Vec<FanBinding>`

- [ ] **Step 1: Write the failing test**

Add at the end of `src/config.rs`:

```rust
#[cfg(test)]
mod fan_plan_tests {
    use super::*;

    #[test]
    fn a_curve_plan_survives_a_config_round_trip() {
        let binding = FanBinding {
            mode: "balanced".into(),
            plan: FanPlan::Curve { steps: [0, 35, 50, 65, 80, 100] },
        };
        let json = serde_json::to_string(&binding).expect("serialize");
        let back: FanBinding = serde_json::from_str(&json).expect("deserialize");
        assert_eq!(back.mode, "balanced");
        assert_eq!(back.plan, FanPlan::Curve { steps: [0, 35, 50, 65, 80, 100] });
    }

    #[test]
    fn every_plan_variant_round_trips() {
        for plan in [
            FanPlan::Automatic,
            FanPlan::Max,
            FanPlan::Fixed { percent: 40 },
            FanPlan::Curve { steps: DEFAULT_PLAN_STEPS },
        ] {
            let json = serde_json::to_string(&plan).expect("serialize");
            let back: FanPlan = serde_json::from_str(&json).expect("deserialize");
            assert_eq!(back, plan);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --lib fan_plan_tests 2>&1 | tail -20` (the crate is a binary; use `cargo test fan_plan_tests`)
Expected: FAIL to compile with `cannot find type FanPlan in this scope`

- [ ] **Step 3: Write minimal implementation**

In `src/config.rs`, directly after the `ModeBinding` struct:

```rust
/// The six step percentages a `FanPlan::Curve` holds, in the order of the
/// fixed breakpoints in `hardware::fan` (<45, <55, <65, <75, <85, >=85 C).
pub const DEFAULT_PLAN_STEPS: [u8; 6] = [25, 35, 50, 65, 80, 100];

/// What the fans should do while a given power mode is active.
///
/// `Automatic` leaves them to the firmware, which is the only thing that can
/// stop them: a manual pwm of 0 is not off, the EC holds a floor.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum FanPlan {
    Automatic,
    Curve { steps: [u8; 6] },
    Fixed { percent: u8 },
    Max,
}

/// Binds a fan plan to a power mode, keyed by `PowerProfile::to_id()` for the
/// same reason `ModeBinding` is: the map stays readable in config.json and
/// survives an unknown mode.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FanBinding {
    pub mode: String,
    pub plan: FanPlan,
}
```

And in `AppConfig`, next to the other fan fields:

```rust
    /// One fan plan per power mode. Empty means "not migrated yet", which
    /// `hardware::fan::migrate_plans` fills from the pre-existing global fan
    /// settings on first run.
    #[serde(default)]
    pub fan_plans: Vec<FanBinding>,
```

And in `impl Default for AppConfig`, next to `fan_auto_curve_enabled`:

```rust
            fan_plans: Vec::new(),
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test fan_plan_tests`
Expected: PASS, 2 tests

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/config.rs
git commit -m "Add a per-mode fan plan to the config

One plan per power mode, keyed the way ModeBinding already keys lighting
schemes. Empty means not yet migrated; the migration in the next commit
fills it from the existing global fan settings so behaviour does not change."
```

---

### Task 2: Resolve a plan into a target

**Files:**
- Modify: `src/hardware/fan.rs`
- Test: `src/hardware/fan.rs` (the existing `#[cfg(test)] mod tests`)

**Interfaces:**
- Consumes: `config::FanPlan`, existing `CurveAction`, `curve_action`, `fan_curve_pct`
- Produces: `fan::plan_target(plan: FanPlan, temp_c: Option<f64>, handed_off: bool) -> CurveAction`

- [ ] **Step 1: Write the failing test**

Add inside `mod tests` in `src/hardware/fan.rs`:

```rust
    #[test]
    fn automatic_leaves_the_fans_to_the_firmware() {
        use crate::config::FanPlan;
        assert_eq!(
            plan_target(FanPlan::Automatic, Some(70.0), false),
            CurveAction::Firmware
        );
        // Already handed off: nothing to write every tick.
        assert_eq!(
            plan_target(FanPlan::Automatic, Some(70.0), true),
            CurveAction::Hold
        );
    }

    #[test]
    fn fixed_and_max_are_flat_percentages() {
        use crate::config::FanPlan;
        assert_eq!(
            plan_target(FanPlan::Fixed { percent: 40 }, Some(70.0), false),
            CurveAction::Manual(40)
        );
        assert_eq!(
            plan_target(FanPlan::Max, Some(30.0), false),
            CurveAction::Manual(100)
        );
    }

    #[test]
    fn a_curve_plan_uses_the_curve_rules() {
        use crate::config::FanPlan;
        let plan = FanPlan::Curve { steps: [0, 35, 50, 65, 80, 100] };
        // Below the zero step: hand off, exactly as curve_action decides.
        assert_eq!(plan_target(plan, Some(40.0), false), CurveAction::Firmware);
        // Inside the firmware band after a handoff: hold.
        assert_eq!(plan_target(plan, Some(50.0), true), CurveAction::Hold);
        // Above the band: the curve takes over.
        assert_eq!(plan_target(plan, Some(70.0), true), CurveAction::Manual(65));
    }

    #[test]
    fn no_temperature_reading_means_write_nothing() {
        use crate::config::FanPlan;
        // A curve cannot be evaluated without a sensor, and guessing a
        // percentage for hardware whose temperature is unknown is how fans end
        // up pinned. Flat plans do not need one.
        assert_eq!(
            plan_target(FanPlan::Curve { steps: DEFAULT_FAN_CURVE }, None, false),
            CurveAction::Hold
        );
        assert_eq!(
            plan_target(FanPlan::Max, None, false),
            CurveAction::Manual(100)
        );
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test plan_target`
Expected: FAIL to compile, `cannot find function plan_target in this scope`

- [ ] **Step 3: Write minimal implementation**

In `src/hardware/fan.rs`, after `curve_action`:

```rust
/// Resolves a mode's plan into what the reconciler should do this tick.
///
/// `temp_c` is the hotter die from `curve_input_temp`. `handed_off` is whether
/// the firmware currently holds the fans, which only `Curve` and `Automatic`
/// care about.
pub fn plan_target(
    plan: crate::config::FanPlan,
    temp_c: Option<f64>,
    handed_off: bool,
) -> CurveAction {
    use crate::config::FanPlan;
    match plan {
        FanPlan::Automatic => {
            if handed_off {
                CurveAction::Hold
            } else {
                CurveAction::Firmware
            }
        }
        FanPlan::Max => CurveAction::Manual(100),
        FanPlan::Fixed { percent } => CurveAction::Manual(percent.min(100)),
        FanPlan::Curve { steps } => match temp_c {
            Some(temp_c) => curve_action(temp_c, &steps, handed_off),
            // No sensor: writing a guessed percentage is how fans get pinned.
            None => CurveAction::Hold,
        },
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test plan_target && cargo test 2>&1 | grep 'test result'`
Expected: 4 new tests PASS, whole suite green

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/hardware/fan.rs
git commit -m "Resolve a fan plan into a target for the tick

Automatic hands off once and then holds, Max and Fixed are flat
percentages, and Curve defers to the rules already measured on hardware. A
Curve with no sensor reading holds rather than guessing a percentage, which
is how fans end up pinned at a speed nothing chose."
```

---

### Task 3: Find the plan for the active mode

**Files:**
- Modify: `src/hardware/fan.rs`
- Test: `src/hardware/fan.rs` (`mod tests`)

**Interfaces:**
- Consumes: `config::{FanBinding, FanPlan}`, `PowerProfile`
- Produces: `fan::plan_for(profile: Option<PowerProfile>, plans: &[FanBinding]) -> FanPlan`

- [ ] **Step 1: Write the failing test**

```rust
    #[test]
    fn the_plan_for_the_active_mode_is_used() {
        use crate::config::{FanBinding, FanPlan};
        use crate::hardware::profile::PowerProfile;
        let plans = vec![
            FanBinding { mode: "balanced".into(), plan: FanPlan::Automatic },
            FanBinding { mode: "turbo".into(), plan: FanPlan::Max },
        ];
        assert_eq!(plan_for(Some(PowerProfile::Balanced), &plans), FanPlan::Automatic);
        assert_eq!(plan_for(Some(PowerProfile::Turbo), &plans), FanPlan::Max);
    }

    #[test]
    fn a_mode_with_no_plan_falls_back_to_automatic() {
        use crate::config::{FanBinding, FanPlan};
        use crate::hardware::profile::PowerProfile;
        let plans = vec![FanBinding { mode: "turbo".into(), plan: FanPlan::Max }];
        // Automatic is the safe fallback: the firmware is always able to cool
        // the machine, and it is what an un-migrated config means.
        assert_eq!(plan_for(Some(PowerProfile::Quiet), &plans), FanPlan::Automatic);
        assert_eq!(plan_for(None, &plans), FanPlan::Automatic);
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test plan_for`
Expected: FAIL to compile, `cannot find function plan_for`

- [ ] **Step 3: Write minimal implementation**

```rust
/// The plan bound to `profile`, or `Automatic` when there is none.
///
/// `Automatic` is the safe fallback in every unknown case: the firmware can
/// always cool the machine, and it is also what an un-migrated config means.
pub fn plan_for(
    profile: Option<crate::hardware::profile::PowerProfile>,
    plans: &[crate::config::FanBinding],
) -> crate::config::FanPlan {
    let Some(profile) = profile else {
        return crate::config::FanPlan::Automatic;
    };
    plans
        .iter()
        .find(|binding| binding.mode == profile.to_id())
        .map(|binding| binding.plan)
        .unwrap_or(crate::config::FanPlan::Automatic)
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test plan_for && cargo test 2>&1 | grep 'test result'`
Expected: 2 new tests PASS, suite green

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/hardware/fan.rs
git commit -m "Look up the fan plan bound to the active power mode

Automatic is the fallback for an unbound mode, an unknown profile and an
un-migrated config alike: the firmware can always cool the machine."
```

---

### Task 4: Migrate the old global settings

**Files:**
- Modify: `src/hardware/fan.rs`
- Test: `src/hardware/fan.rs` (`mod tests`)

**Interfaces:**
- Consumes: `config::{AppConfig, FanBinding, FanPlan, DEFAULT_PLAN_STEPS}`
- Produces: `fan::migrate_plans(cfg: &crate::config::AppConfig) -> Vec<FanBinding>`

- [ ] **Step 1: Write the failing test**

```rust
    #[test]
    fn the_curve_migrates_to_every_mode() {
        use crate::config::{AppConfig, FanPlan};
        let mut cfg = AppConfig::default();
        cfg.fan_auto_curve_enabled = true;
        cfg.fan_curve_points = [0, 35, 50, 65, 80, 100];
        let plans = migrate_plans(&cfg);
        assert_eq!(plans.len(), 5);
        for binding in &plans {
            assert_eq!(binding.plan, FanPlan::Curve { steps: [0, 35, 50, 65, 80, 100] });
        }
    }

    #[test]
    fn a_disabled_curve_migrates_to_the_saved_fan_mode() {
        use crate::config::{AppConfig, FanPlan};
        let mut cfg = AppConfig::default();
        cfg.fan_auto_curve_enabled = false;
        cfg.fan_mode = Some("max".into());
        for binding in migrate_plans(&cfg) {
            assert_eq!(binding.plan, FanPlan::Max);
        }

        let mut cfg = AppConfig::default();
        cfg.fan_auto_curve_enabled = false;
        cfg.fan_mode = Some("auto".into());
        for binding in migrate_plans(&cfg) {
            assert_eq!(binding.plan, FanPlan::Automatic);
        }
    }

    #[test]
    fn nothing_configured_migrates_to_automatic() {
        use crate::config::{AppConfig, FanPlan};
        let cfg = AppConfig::default();
        let plans = migrate_plans(&cfg);
        assert_eq!(plans.len(), 5);
        for binding in &plans {
            assert_eq!(binding.plan, FanPlan::Automatic);
        }
    }

    #[test]
    fn migration_covers_every_mode_exactly_once() {
        use crate::config::AppConfig;
        let plans = migrate_plans(&AppConfig::default());
        let mut modes: Vec<&str> = plans.iter().map(|b| b.mode.as_str()).collect();
        modes.sort_unstable();
        assert_eq!(modes, vec!["balanced", "eco", "performance", "quiet", "turbo"]);
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test migrate_plans`
Expected: FAIL to compile, `cannot find function migrate_plans`

- [ ] **Step 3: Write minimal implementation**

```rust
/// Builds the initial per-mode plans from the pre-existing global fan
/// settings, so migrating changes nothing about how the machine behaves until
/// a mode is deliberately given its own plan.
///
/// This is the only remaining reader of `fan_auto_curve_enabled`,
/// `fan_curve_points` and `fan_mode`. Every other consumer uses `fan_plans`,
/// or the two would drift into exactly the disagreement this design removes.
pub fn migrate_plans(cfg: &crate::config::AppConfig) -> Vec<crate::config::FanBinding> {
    use crate::config::{FanBinding, FanPlan};
    use crate::hardware::profile::PowerProfile;

    let plan = if cfg.fan_auto_curve_enabled {
        FanPlan::Curve { steps: cfg.fan_curve_points }
    } else {
        match cfg.fan_mode.as_deref() {
            Some("max") => FanPlan::Max,
            // "auto", anything unrecognised, and never-set all mean the
            // firmware was in charge, which is Automatic.
            _ => FanPlan::Automatic,
        }
    };

    [
        PowerProfile::Eco,
        PowerProfile::Quiet,
        PowerProfile::Balanced,
        PowerProfile::Performance,
        PowerProfile::Turbo,
    ]
    .into_iter()
    .map(|profile| FanBinding {
        mode: profile.to_id().to_string(),
        plan,
    })
    .collect()
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test migrate_plans && cargo test 2>&1 | grep 'test result'`
Expected: 4 new tests PASS, suite green

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/hardware/fan.rs
git commit -m "Migrate the global fan settings into per-mode plans

Every mode starts with the plan the machine was already following, so
migration is invisible until a mode is given its own. This is the last
reader of fan_auto_curve_enabled, fan_curve_points and fan_mode."
```

---

### Task 5: Refuse PWM plans on hardware without PWM

**Files:**
- Modify: `src/hardware/fan.rs`
- Test: `src/hardware/fan.rs` (`mod tests`)

**Interfaces:**
- Consumes: `config::FanPlan`
- Produces: `fan::plan_for_hardware(plan: FanPlan, pwm_available: bool) -> FanPlan`

- [ ] **Step 1: Write the failing test**

```rust
    #[test]
    fn pwm_plans_are_unavailable_without_pwm() {
        use crate::config::FanPlan;
        // Reachable from a config copied off a machine that does have PWM.
        // Resolving to Automatic beats doing nothing silently.
        assert_eq!(
            plan_for_hardware(FanPlan::Curve { steps: DEFAULT_FAN_CURVE }, false),
            FanPlan::Automatic
        );
        assert_eq!(
            plan_for_hardware(FanPlan::Fixed { percent: 40 }, false),
            FanPlan::Automatic
        );
        // These two work through the EC preset path on any model.
        assert_eq!(plan_for_hardware(FanPlan::Max, false), FanPlan::Max);
        assert_eq!(plan_for_hardware(FanPlan::Automatic, false), FanPlan::Automatic);
    }

    #[test]
    fn every_plan_survives_on_hardware_with_pwm() {
        use crate::config::FanPlan;
        for plan in [
            FanPlan::Automatic,
            FanPlan::Max,
            FanPlan::Fixed { percent: 40 },
            FanPlan::Curve { steps: DEFAULT_FAN_CURVE },
        ] {
            assert_eq!(plan_for_hardware(plan, true), plan);
        }
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test plan_for_hardware`
Expected: FAIL to compile, `cannot find function plan_for_hardware`

- [ ] **Step 3: Write minimal implementation**

```rust
/// Narrows a plan to what this machine can actually do.
///
/// `Curve` and `Fixed` need per-fan PWM. Without `ACER_CAP_PWM` there is no
/// `pwm1` to write, so they are unreachable through the UI, but they can still
/// arrive in a config copied from a machine that has it. Resolving to
/// `Automatic` keeps the fans governed by something rather than leaving a plan
/// that silently does nothing.
pub fn plan_for_hardware(
    plan: crate::config::FanPlan,
    pwm_available: bool,
) -> crate::config::FanPlan {
    use crate::config::FanPlan;
    match plan {
        FanPlan::Curve { .. } | FanPlan::Fixed { .. } if !pwm_available => FanPlan::Automatic,
        other => other,
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test plan_for_hardware && cargo test 2>&1 | grep 'test result'`
Expected: 2 new tests PASS, suite green

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/hardware/fan.rs
git commit -m "Narrow fan plans to what the hardware can do

Curve and Fixed need per-fan PWM. On a model without it they are
unreachable in the UI but can still arrive in a copied config, so they
resolve to Automatic instead of silently doing nothing."
```

---

### Task 6: The reconciler

**Files:**
- Modify: `src/ui/window.rs` (the three-second fan tick added by commit `d5f1131`)
- Test: hardware verification; the decision logic is already covered by Tasks 2 to 5

**Interfaces:**
- Consumes: `fan::{plan_for, plan_for_hardware, plan_target, needs_write, FanState, CurveAction, curve_input_temp, migrate_plans}`, `profile::get_current_profile`, `sensors::read_critical_temps`, `capabilities::get().fan_pwm`, `config::{load_app_config, save_app_config}`
- Produces: no new public interface; this is the only caller of `set_pwm_percent`/`set_pwm_auto` after Task 7

- [ ] **Step 1: Replace the tick body**

In `src/ui/window.rs`, the tick currently reads `cfg.fan_auto_curve_enabled` and calls `curve_action` directly. Replace its body with:

```rust
                let cfg = config::load_app_config();
                // Migrate on first sight, then use plans only.
                if cfg.fan_plans.is_empty() {
                    let mut migrated = cfg.clone();
                    migrated.fan_plans = crate::hardware::fan::migrate_plans(&cfg);
                    if config::save_app_config(&migrated).is_ok() {
                        crate::hardware::applog::info(
                            "fan: migrated the global fan settings to per-mode plans",
                        );
                    }
                    return glib::ControlFlow::Continue;
                }

                if applying.get() {
                    return glib::ControlFlow::Continue;
                }

                let plan = crate::hardware::fan::plan_for_hardware(
                    crate::hardware::fan::plan_for(
                        crate::hardware::profile::get_current_profile(),
                        &cfg.fan_plans,
                    ),
                    crate::hardware::capabilities::get().fan_pwm,
                );
                let (cpu, gpu) = sensors::read_critical_temps();
                let temp = crate::hardware::fan::curve_input_temp(cpu, gpu);
                let applied = fan_state.get();
                let target = crate::hardware::fan::plan_target(
                    plan,
                    temp,
                    applied == crate::hardware::fan::FanState::Firmware,
                );
                if !crate::hardware::fan::needs_write(target, applied) {
                    return glib::ControlFlow::Continue;
                }
                // ... existing Firmware / Manual(pct) / Hold match, unchanged
```

- [ ] **Step 2: Build and run the whole suite**

Run: `cargo build --release && cargo test 2>&1 | grep 'test result'`
Expected: exit 0, all three suites green

- [ ] **Step 3: Verify on hardware, one plan at a time**

For each plan, set it for the active mode by editing `~/.config/predator-sense/config.json` with the app stopped, then start the app and read the hardware back:

```bash
H=/sys/class/hwmon/hwmon5
echo "pwm1=$(cat $H/pwm1) enable=$(cat $H/pwm1_enable) fan1=$(cat $H/fan1_input)"
```

Expected: `Automatic` gives `pwm1_enable=2`; `Max` gives `pwm1=255`; `Fixed { percent: 40 }` gives `pwm1=102`; `Curve` gives the step for the current hotter die.

- [ ] **Step 4: Verify write volume and single ownership**

Run, with the app running and idle for three minutes:

```bash
grep -c 'fan: manual pwm\|fan: firmware curve' ~/.local/share/predator-sense/app.log
```

Expected: no growth once the target is reached. Then switch power modes and confirm exactly one write follows.

- [ ] **Step 5: Commit**

```bash
git add predator-sense-gui/src/ui/window.rs
git commit -m "Drive the fans from the active mode's plan, in one place

The tick now resolves (active mode, its plan, temperatures) into a target
and is the only code that writes fan hardware. Migration runs on first
sight of an empty plan list.

Verified on a PH16-71: Automatic gives pwm_enable=2, Max gives pwm 255,
Fixed 40% gives pwm 102, Curve gives the step for the hotter die, and an
idle machine produces no writes at all once its target is reached."
```

---

### Task 7: Stop the other writers

**Files:**
- Modify: `src/hardware/profile.rs:998` (remove the `set_fan_mode` call in `set_profile`)
- Modify: `src/ui/fan_control_page.rs` (buttons write plans instead of hardware)
- Modify: `src/hardware/ai_assistant.rs` (the `set_fan_mode` action writes a plan)
- Test: `src/hardware/fan.rs` (`mod tests`) for the helper the page and assistant share

**Interfaces:**
- Consumes: `config::{FanPlan, FanBinding}`
- Produces: `fan::set_plan_for(profile: Option<PowerProfile>, plan: FanPlan) -> Result<(), String>`

- [ ] **Step 1: Write the failing test for the shared helper**

```rust
    #[test]
    fn setting_a_plan_replaces_only_that_modes_binding() {
        use crate::config::{FanBinding, FanPlan};
        let existing = vec![
            FanBinding { mode: "balanced".into(), plan: FanPlan::Automatic },
            FanBinding { mode: "turbo".into(), plan: FanPlan::Max },
        ];
        let updated = with_plan(&existing, "balanced", FanPlan::Max);
        assert_eq!(updated.len(), 2);
        assert_eq!(plan_for_id(&updated, "balanced"), Some(FanPlan::Max));
        assert_eq!(plan_for_id(&updated, "turbo"), Some(FanPlan::Max));
    }

    #[test]
    fn setting_a_plan_for_an_unbound_mode_adds_it() {
        use crate::config::{FanBinding, FanPlan};
        let existing = vec![FanBinding { mode: "turbo".into(), plan: FanPlan::Max }];
        let updated = with_plan(&existing, "eco", FanPlan::Automatic);
        assert_eq!(updated.len(), 2);
        assert_eq!(plan_for_id(&updated, "eco"), Some(FanPlan::Automatic));
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test with_plan`
Expected: FAIL to compile, `cannot find function with_plan`

- [ ] **Step 3: Write minimal implementation**

```rust
/// The plan bound to a raw mode id, for callers that already have the id.
pub fn plan_for_id(
    plans: &[crate::config::FanBinding],
    mode: &str,
) -> Option<crate::config::FanPlan> {
    plans
        .iter()
        .find(|binding| binding.mode == mode)
        .map(|binding| binding.plan)
}

/// `plans` with `mode` bound to `plan`, adding the binding if it is new.
pub fn with_plan(
    plans: &[crate::config::FanBinding],
    mode: &str,
    plan: crate::config::FanPlan,
) -> Vec<crate::config::FanBinding> {
    let mut updated: Vec<crate::config::FanBinding> = plans
        .iter()
        .filter(|binding| binding.mode != mode)
        .cloned()
        .collect();
    updated.push(crate::config::FanBinding {
        mode: mode.to_string(),
        plan,
    });
    updated
}

/// Writes a plan for `profile` into the config. Intent only: the reconciler
/// notices on its next tick and performs any hardware write.
pub fn set_plan_for(
    profile: Option<crate::hardware::profile::PowerProfile>,
    plan: crate::config::FanPlan,
) -> Result<(), String> {
    let Some(profile) = profile else {
        return Err("fan plan: no active power mode".into());
    };
    let mut cfg = crate::config::load_app_config();
    cfg.fan_plans = with_plan(&cfg.fan_plans, profile.to_id(), plan);
    crate::config::save_app_config(&cfg)
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test with_plan && cargo test 2>&1 | grep 'test result'`
Expected: 2 new tests PASS, suite green

- [ ] **Step 5: Convert the three writers**

In `src/hardware/profile.rs`, delete the `set_fan_mode(fan_mode)` call in `set_profile` and the now-unused `fan_mode_for` helper it feeds. Leave a comment saying the mode's plan decides and the reconciler applies it.

In `src/ui/fan_control_page.rs`, replace each `set_fan_mode`/`set_pwm_percent`/`set_pwm_auto` call in a button handler with the matching `fan::set_plan_for(profile::get_current_profile(), plan)`: Automatic to `FanPlan::Automatic`, Max to `FanPlan::Max`, Custom to `FanPlan::Fixed { percent }`, the auto-curve toggle to `FanPlan::Curve { steps: cfg.fan_curve_points }` when switched on and `FanPlan::Automatic` when switched off.

In `src/hardware/ai_assistant.rs`, replace each `set_fan_mode(..)` with `set_plan_for(profile::get_current_profile(), ..)` using the same mapping.

- [ ] **Step 6: Verify no writer remains outside the reconciler**

Run:

```bash
grep -rn 'set_pwm_percent\|set_pwm_auto\|set_fan_mode' src/ --include='*.rs' | grep -v '^src/hardware/fan.rs'
```

Expected: exactly one group of hits, all in `src/ui/window.rs` inside the reconciler.

- [ ] **Step 7: Verify on hardware**

Switch power modes through the UI and the mode key, run the AI assistant's fan action if enabled, and press the page's buttons. After each, check the log:

```bash
grep 'fan: ' ~/.local/share/predator-sense/app.log | tail -5
```

Expected: every hardware write is preceded by a reconciler decision; no write appears immediately after a profile change without one.

- [ ] **Step 8: Commit**

```bash
git add predator-sense-gui/src/hardware/profile.rs predator-sense-gui/src/ui/fan_control_page.rs predator-sense-gui/src/hardware/ai_assistant.rs predator-sense-gui/src/hardware/fan.rs
git commit -m "Route every fan change through the reconciler

Applying a power profile no longer forces a fan mode, the page's buttons
and the AI assistant's fan action write a plan into config, and the
reconciler is left as the only code that touches fan hardware.

Instrumentation on 2026-09-15 caught the old arrangement disagreeing inside
one startup sequence: set_fan_mode(Auto) handed the fans to the firmware and
the curve took them back four seconds later."
```

---

### Task 8: Apply plans on hardware without per-fan PWM

Added mid-execution by controller ruling. The spec requires the reconciler to
translate `Automatic` and `Max` to the EC preset path on models without PWM
(`docs/superpowers/specs/2026-09-15-fan-control-design.md`, "The reconciler"),
and no task implemented it. Task 7 removed the last direct `set_fan_mode`
callers, so on EC-only hardware fan intent now reaches config and stops there:
the tick that would apply it is gated behind `caps.fan_pwm` at
`src/ui/window.rs:598`. Most Acer models this app supports are EC-only.

**Files:**
- Modify: `src/hardware/fan.rs` (the new function and its tests)
- Modify: `src/ui/window.rs` (remove the capability gate, dispatch on the result)

**Interfaces:**
- Consumes: `CurveAction`, `FanMode`, `capabilities::get().fan_pwm`
- Produces: `fan::FanWrite` and `fan::write_for(target: CurveAction, pwm_available: bool) -> Option<FanWrite>`

- [ ] **Step 1: Write the failing test**

Add inside `mod tests` in `src/hardware/fan.rs`:

```rust
    #[test]
    fn pwm_hardware_gets_pwm_writes() {
        assert_eq!(write_for(CurveAction::Firmware, true), Some(FanWrite::PwmAuto));
        assert_eq!(
            write_for(CurveAction::Manual(35), true),
            Some(FanWrite::PwmPercent(35))
        );
    }

    #[test]
    fn ec_only_hardware_gets_preset_writes() {
        // plan_for_hardware narrows Curve and Fixed to Automatic when there is
        // no PWM, so only Automatic and Max can reach here, which is why
        // Manual maps to the Max preset rather than a duty cycle.
        assert_eq!(write_for(CurveAction::Firmware, false), Some(FanWrite::PresetAuto));
        assert_eq!(write_for(CurveAction::Manual(100), false), Some(FanWrite::PresetMax));
    }

    #[test]
    fn hold_writes_nothing_on_either_kind_of_hardware() {
        assert_eq!(write_for(CurveAction::Hold, true), None);
        assert_eq!(write_for(CurveAction::Hold, false), None);
    }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test write_for`
Expected: FAIL to compile, `cannot find function write_for` / `cannot find type FanWrite`

- [ ] **Step 3: Write minimal implementation**

In `src/hardware/fan.rs`, after `plan_for_hardware`:

```rust
/// Which hardware call a resolved target needs on this machine.
///
/// Two families of Acer hardware reach this point. Models with
/// `ACER_CAP_PWM` take a duty cycle per fan; everything else has only the EC's
/// own Auto and Max presets. Keeping the choice in one pure function means the
/// reconciler does not grow a second decision path, and it stays testable
/// without hardware.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FanWrite {
    /// `pwm_enable = 2` on both fans: hand them to the firmware curve.
    PwmAuto,
    /// A manual duty on both fans.
    PwmPercent(u8),
    /// The EC's Auto preset, for models with no per-fan PWM.
    PresetAuto,
    /// The EC's Max preset.
    PresetMax,
}

pub fn write_for(target: CurveAction, pwm_available: bool) -> Option<FanWrite> {
    match (target, pwm_available) {
        (CurveAction::Hold, _) => None,
        (CurveAction::Firmware, true) => Some(FanWrite::PwmAuto),
        (CurveAction::Manual(percent), true) => Some(FanWrite::PwmPercent(percent)),
        // Without per-fan PWM, plan_for_hardware has already narrowed every
        // plan to Automatic or Max, so a manual target can only have come from
        // Max. There is no EC preset for an arbitrary percentage.
        (CurveAction::Firmware, false) => Some(FanWrite::PresetAuto),
        (CurveAction::Manual(_), false) => Some(FanWrite::PresetMax),
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test write_for && cargo test 2>&1 | grep 'test result'`
Expected: 3 new tests PASS, whole suite green

- [ ] **Step 5: Remove the capability gate and dispatch on FanWrite**

In `src/ui/window.rs`, the fan tick sits inside `if crate::hardware::capabilities::get().fan_pwm {` (around line 598). That gate is what strands EC-only hardware: delete it so the tick always runs, keeping the block's contents at the outer level. Then replace the `match action` arms so the hardware call comes from `write_for`:

```rust
                let pwm = crate::hardware::capabilities::get().fan_pwm;
                let Some(write) = crate::hardware::fan::write_for(target, pwm) else {
                    return glib::ControlFlow::Continue;
                };
                applying.set(true);
                let applying_done = applying.clone();
                let state = fan_state.clone();
                let intended = target;
                background::run(
                    move || match write {
                        crate::hardware::fan::FanWrite::PwmAuto => {
                            crate::hardware::fan::set_pwm_auto()
                        }
                        crate::hardware::fan::FanWrite::PwmPercent(percent) => {
                            crate::hardware::fan::set_pwm_percent(percent, percent)
                        }
                        crate::hardware::fan::FanWrite::PresetAuto => {
                            crate::hardware::fan::set_fan_mode(crate::hardware::fan::FanMode::Auto)
                        }
                        crate::hardware::fan::FanWrite::PresetMax => {
                            crate::hardware::fan::set_fan_mode(crate::hardware::fan::FanMode::Max)
                        }
                    },
                    move |result| {
                        applying_done.set(false);
                        match result {
                            // FanState tracks the target, not the call: a
                            // preset Auto and a pwm_enable=2 are both "the
                            // firmware has them" as far as needs_write cares.
                            Ok(()) => state.set(match intended {
                                CurveAction::Firmware => FanState::Firmware,
                                CurveAction::Manual(percent) => FanState::Manual(percent),
                                CurveAction::Hold => FanState::Unknown,
                            }),
                            Err(error) => {
                                state.set(FanState::Unknown);
                                crate::hardware::applog::error(&format!(
                                    "fan: {write:?} not applied: {error}"
                                ));
                            }
                        }
                    },
                );
```

Keep the `needs_write` check ahead of this, unchanged. Keep the migration branch and the `applying` guard unchanged.

- [ ] **Step 6: Build and run every suite**

Run: `cargo build --release && cargo test 2>&1 | grep 'test result' && cargo test --manifest-path installer/Cargo.toml 2>&1 | grep 'test result'`
Expected: exit 0; 248 GUI (245 plus 3 new); 84 installer.

- [ ] **Step 7: Verify no writer remains outside the reconciler**

Run:

```bash
grep -rn 'set_pwm_percent\|set_pwm_auto\|set_fan_mode' src/ --include='*.rs' | grep -v '^\S*:[0-9]*: *//' | grep -v '^src/hardware/fan.rs'
```

Expected: hits in `src/ui/window.rs` only, and `set_fan_mode` among them (it is no longer dead).

- [ ] **Step 8: Commit**

```bash
git add predator-sense-gui/src/hardware/fan.rs predator-sense-gui/src/ui/window.rs
git commit -m "Apply fan plans on hardware without per-fan PWM

The reconciler only ran on models with ACER_CAP_PWM, which was harmless
while other code wrote fans directly. Once those writers became intent
only, EC-only hardware, which is most of the models this app supports, had
its fan intent written to config and applied by nobody.

write_for() picks the hardware call from the resolved target and the
machine's capability: a duty cycle where there is PWM, the EC's Auto and Max
presets where there is not. plan_for_hardware has already narrowed Curve and
Fixed to Automatic on those models, so a manual target there can only mean
Max.

The capability gate around the tick is gone, since the tick now knows what
to do on both families of hardware."
```

## Self-Review

**Spec coverage:** data model (Task 1), reconciler and its inputs (Task 6), plan resolution including the curve rules (Task 2), per-mode lookup and fallback (Task 3), migration from all four starting points (Task 4), capability narrowing (Task 5), converting every other writer (Task 7), failure handling and back-off (already shipped in `d5f1131`, exercised by Task 6 step 4). Phase 1 of the spec's phasing is fully covered. Phases 2 (UI consolidation) and 3 (curve widget) get their own plans.

**Placeholders:** none. Every code step carries the code.

**Type consistency:** `FanPlan`, `FanBinding`, `DEFAULT_PLAN_STEPS` defined in Task 1 and used unchanged after; `plan_target`, `plan_for`, `plan_for_hardware`, `migrate_plans`, `with_plan`, `plan_for_id`, `set_plan_for` each defined once with the signature later tasks call. `CurveAction`, `FanState`, `needs_write`, `curve_input_temp`, `DEFAULT_FAN_CURVE` are pre-existing and listed in Global Constraints.

**Gap found and fixed during review:** Task 6's original draft read `cfg.fan_auto_curve_enabled` to decide whether to run at all, which would have left the curve toggle as a second source of truth beside the plans. The tick now depends on plans only, and Task 7 maps the toggle onto `FanPlan::Curve`/`FanPlan::Automatic`.
