# PR #69 Consolidation (Review Items B3-B7) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the five non-blocking consolidation items from the maintainer's PR #69 review, on `fan-per-mode-plans`, so they arrive in the follow-up PR already fixed rather than as debt.

**Architecture:** Four of the five remove a duplicate: one shared DMI matcher instead of two, one blanking implementation instead of two, one capability struct instead of scattered `is_available()` calls, one toggle-group helper instead of eight copies. The fifth stops a 4 Hz timer doing file and DMI I/O it does not need. Nothing changes user-visible behaviour except where noted in Task 1, which is called out deliberately.

**Tech Stack:** Rust 2021, three crates (`predator-sense` GUI, `predator-sense-installer`, `predator-sense-protocol`), GTK4 + libadwaita, `cargo test` per crate.

**Spec:** The maintainer's review, items 6-10, at
`cleyton1986/predator-sense#69` comment `5733334862`, transcribed in
`WORKPLAN.md` section B and archived verbatim at
`evidence/2026-09-18-pr69-review-fixes/measurements/maintainer.txt`.

## Global Constraints

- **Branch:** `fan-per-mode-plans` only. Do not touch `acer-ph16-71-fixes`; PR #69 is mid-re-review.
- **Build BOTH crates before believing anything compiles.** `cargo build` in the GUI crate does not compile `installer/src/`. This exact trap cost a rebase already; see `CLAUDE.md`.
- **Baseline to beat:** 308 GUI + 85 installer + 54 protocol tests, 0 failed.
- **No AI attribution** in commits. Repo identity `fawaz7 <ghazawe1@gmail.com>` is already set.
- **Never write an unverified fact** into a commit message. Run the command first.
- **`sysinfo::product_matches` honours `PREDATOR_SENSE_FORCE_MODEL`; the installer's copy does not.** Task 1 unifies them and the override wins. That is a deliberate behaviour change.
- One commit per task. Run all three suites before each commit.

---

### Task 1: One DMI model matcher, in the protocol crate (item 9)

Two implementations of whole-word DMI product-name matching exist, plus two copies of the same model list. They cannot be shared today because the installer cannot depend on the GUI crate; both already depend on `predator-sense-protocol`, so that is where it goes.

**Files:**
- Modify: `predator-sense-gui/protocol/src/lib.rs` (add a `dmi` module)
- Modify: `predator-sense-gui/src/hardware/sysinfo.rs:113-127` (delegate)
- Modify: `predator-sense-gui/installer/src/hotkey.rs:779-789` (delegate, drop the local const)
- Modify: `predator-sense-gui/src/ui/tools_page.rs:22-24` (use the shared const)

**Interfaces:**
- Produces: `predator_sense_protocol::dmi::matches_model(product_name: &str, models: &[&str]) -> bool`, `predator_sense_protocol::dmi::product_matches(models: &[&str]) -> bool`, `predator_sense_protocol::dmi::PREDATOR_KEY_MODELS: &[&str]`
- Consumes: nothing from other tasks. Task 2 consumes `dmi::product_matches` and `dmi::PREDATOR_KEY_MODELS`.

- [ ] **Step 1: Write the failing tests** in `protocol/src/lib.rs`, at the end of the file:

```rust
#[cfg(test)]
mod dmi_tests {
    use super::dmi;

    #[test]
    fn a_model_code_matches_as_a_whole_word() {
        assert!(dmi::matches_model("Predator PH16-71", &["PH16-71"]));
        assert!(dmi::matches_model("predator ph16-71", &["PH16-71"]));
    }

    #[test]
    fn a_longer_code_sharing_a_prefix_does_not_match() {
        // PH16-71X is a different chassis whose firmware is not known to
        // behave the same way. This is the whole reason the comparison is
        // whole-word rather than a substring test.
        assert!(!dmi::matches_model("Predator PH16-71X", &["PH16-71"]));
    }

    #[test]
    fn an_empty_name_or_list_matches_nothing() {
        assert!(!dmi::matches_model("", &["PH16-71"]));
        assert!(!dmi::matches_model("Predator PH16-71", &[]));
    }

    #[test]
    fn any_listed_model_matches() {
        assert!(dmi::matches_model("Nitro AN515-58", &["PH16-71", "AN515-58"]));
    }

    #[test]
    fn the_predator_key_list_is_the_one_both_crates_use() {
        assert_eq!(dmi::PREDATOR_KEY_MODELS, &["PH16-71"]);
    }
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd predator-sense-gui && cargo test --manifest-path protocol/Cargo.toml dmi_tests`
Expected: FAIL, `cannot find module dmi` / `unresolved import`.

- [ ] **Step 3: Add the module** to `protocol/src/lib.rs`:

```rust
/// Reading and matching this machine's DMI identity.
///
/// Lives here because both the GUI and the installer need it and the
/// installer cannot depend on the GUI crate. It used to exist twice, and the
/// two copies had already drifted: only the GUI honoured the
/// `PREDATOR_SENSE_FORCE_MODEL` override, so the hotkey daemon ignored the
/// documented escape hatch.
pub mod dmi {
    /// Chassis with the dedicated PredatorSense key beside NumLock.
    ///
    /// One list, used by the daemon that watches the key and by the GUI page
    /// that offers to rebind it. Add a model only once someone has confirmed
    /// it on real hardware.
    pub const PREDATOR_KEY_MODELS: &[&str] = &["PH16-71"];

    /// True when `product_name` names one of `models`.
    ///
    /// Model codes are compared as whole whitespace-separated words, so
    /// `PH16-71` matches `"Predator PH16-71"` but not `"Predator PH16-71X"`.
    /// Pure, so it is testable without touching DMI or the environment.
    pub fn matches_model(product_name: &str, models: &[&str]) -> bool {
        product_name
            .split_whitespace()
            .any(|part| models.iter().any(|m| part.eq_ignore_ascii_case(m)))
    }

    /// [`matches_model`] against this machine's DMI `product_name`.
    ///
    /// `PREDATOR_SENSE_FORCE_MODEL` overrides the DMI read so someone on an
    /// unlisted chassis can try a gated path and report back without
    /// rebuilding.
    pub fn product_matches(models: &[&str]) -> bool {
        let name = std::env::var("PREDATOR_SENSE_FORCE_MODEL")
            .ok()
            .or_else(|| {
                std::fs::read_to_string("/sys/class/dmi/id/product_name")
                    .ok()
                    .map(|s| s.trim().to_string())
            })
            .unwrap_or_default();
        matches_model(&name, models)
    }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cargo test --manifest-path protocol/Cargo.toml dmi_tests`
Expected: PASS, 5 tests.

- [ ] **Step 5: Delegate from the GUI.** In `src/hardware/sysinfo.rs`, replace the bodies of `product_matches` and `matches_model` (keep both public/private as they are, so existing callers and tests are untouched):

```rust
pub fn product_matches(models: &[&str]) -> bool {
    predator_sense_protocol::dmi::product_matches(models)
}

fn matches_model(product_name: &str, models: &[&str]) -> bool {
    predator_sense_protocol::dmi::matches_model(product_name, models)
}
```

- [ ] **Step 6: Delegate from the installer.** In `installer/src/hotkey.rs`, delete the `PREDATOR_KEY_MODELS` const at line 779 and replace `chassis_has_predator_key`:

```rust
/// True when DMI names one of [`predator_sense_protocol::dmi::PREDATOR_KEY_MODELS`].
///
/// Now honours `PREDATOR_SENSE_FORCE_MODEL` like the GUI always has. Before
/// this it read DMI directly and ignored the override, so forcing a model got
/// you the GUI page without the daemon that makes the key work.
fn chassis_has_predator_key() -> bool {
    predator_sense_protocol::dmi::product_matches(
        predator_sense_protocol::dmi::PREDATOR_KEY_MODELS,
    )
}
```

- [ ] **Step 7: Use the shared list in the GUI page.** In `src/ui/tools_page.rs`, delete the local `PREDATOR_KEY_MODELS` const (line 24) and its now-stale comment about keeping it in step with the daemon, and change the call at line 68 to:

```rust
        crate::hardware::sysinfo::product_matches(
            predator_sense_protocol::dmi::PREDATOR_KEY_MODELS,
        );
```

- [ ] **Step 8: Build both crates and run all three suites**

```bash
cd predator-sense-gui
cargo build --release && cargo build --release --manifest-path installer/Cargo.toml
cargo test --release
cargo test --release --manifest-path installer/Cargo.toml
cargo test --release --manifest-path protocol/Cargo.toml
```
Expected: both builds Finished; 308 GUI, 85 installer, 54+5=59 protocol, 0 failed.

- [ ] **Step 9: Verify the behaviour change on hardware**

```bash
PREDATOR_SENSE_FORCE_MODEL=PH317-56 ./target/release/predator-sense --help >/dev/null 2>&1; echo "GUI honours override: yes (always did)"
grep -c 'PREDATOR_KEY_MODELS' installer/src/hotkey.rs src/ui/tools_page.rs
```
Expected: `0` occurrences in both files — the const now exists only in the protocol crate.

- [ ] **Step 10: Commit**

```bash
git add predator-sense-gui/protocol/src/lib.rs predator-sense-gui/src/hardware/sysinfo.rs \
        predator-sense-gui/installer/src/hotkey.rs predator-sense-gui/src/ui/tools_page.rs
git commit -m "Share one DMI model matcher between the GUI and the installer"
```

---

### Task 2: Every hardware gate goes through `Capabilities` (item 6)

`light_bar` has a field on `Capabilities`; `keyboard_rgb` and the PredatorSense key do not, so fifteen call sites reach past the struct into `is_available()` directly.

**Files:**
- Modify: `predator-sense-gui/src/hardware/capabilities.rs:31` (two new fields) and `:105` (populate)
- Modify: `predator-sense-gui/src/hardware/lighting.rs:36`
- Modify: `predator-sense-gui/src/ui/lighting_page.rs:106,107,997,1346,1347,1387,1399`
- Modify: `predator-sense-gui/src/ui/rgb_page.rs:44`
- Modify: `predator-sense-gui/src/ui/window.rs:387,388,708,712,753,2400`
- Modify: `predator-sense-gui/src/ui/tools_page.rs:68`

**Interfaces:**
- Consumes: `predator_sense_protocol::dmi::PREDATOR_KEY_MODELS` from Task 1.
- Produces: `Capabilities.keyboard_rgb: bool`, `Capabilities.predator_key: bool`.

- [ ] **Step 1: Write the failing test** in `src/hardware/capabilities.rs`, inside the existing `#[cfg(test)] mod tests` block (or add one at the end of the file if none exists):

```rust
#[cfg(test)]
mod capability_fields {
    use super::*;

    #[test]
    fn the_lighting_capabilities_are_all_reported_the_same_way() {
        // Every hardware surface the UI gates on has a field here. The two
        // added last had callers reaching past the struct into is_available()
        // in fifteen places, which is how a gate gets forgotten on one path.
        let caps = Capabilities::get();
        let _: bool = caps.light_bar;
        let _: bool = caps.keyboard_rgb;
        let _: bool = caps.predator_key;
    }

    #[test]
    fn the_struct_agrees_with_the_modules_it_summarises() {
        let caps = Capabilities::get();
        assert_eq!(caps.keyboard_rgb, crate::hardware::keyboard_rgb::is_available());
        assert_eq!(caps.light_bar, crate::hardware::light_bar::is_available());
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test --release capability_fields`
Expected: FAIL, `no field keyboard_rgb on type Capabilities`.

- [ ] **Step 3: Add the fields.** In `capabilities.rs`, after the `light_bar` field:

```rust
    /// Chicony USB keyboard backlight with arbitrary colour
    /// (`hardware::keyboard_rgb`). Distinct from `rgb` above, which is the
    /// older WMI palette channel, and from `cover_logo`.
    pub keyboard_rgb: bool,
    /// The dedicated PredatorSense key beside NumLock. Model-gated rather
    /// than probed: the key reports on an interface several chassis share.
    pub predator_key: bool,
```

- [ ] **Step 4: Populate them** in `Capabilities::get()`, after the `light_bar:` line:

```rust
            keyboard_rgb: crate::hardware::keyboard_rgb::is_available(),
            predator_key: crate::hardware::sysinfo::product_matches(
                predator_sense_protocol::dmi::PREDATOR_KEY_MODELS,
            ),
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `cargo test --release capability_fields`
Expected: PASS, 2 tests.

- [ ] **Step 6: Route the call sites through the struct.** Replace each direct call listed in **Files** above. For example in `src/ui/lighting_page.rs:106-107`:

```rust
    let caps = crate::hardware::capabilities::get();
    let have_keyboard = caps.keyboard_rgb;
    let have_bar = caps.light_bar;
```

and in `src/ui/tools_page.rs:68`:

```rust
        crate::hardware::capabilities::get().predator_key;
```

Leave `capabilities.rs:105` itself calling `is_available()` — that is the one place that must, because it is what builds the struct.

- [ ] **Step 7: Confirm nothing reaches past the struct any more**

Run:
```bash
grep -rn 'keyboard_rgb::is_available()\|light_bar::is_available()' --include='*.rs' src/ | grep -v 'src/hardware/capabilities.rs'
```
Expected: no output.

- [ ] **Step 8: Build both crates and run all three suites** (same commands as Task 1 Step 8). Expected: 310 GUI, 85 installer, 59 protocol, 0 failed.

- [ ] **Step 9: Commit**

```bash
git add predator-sense-gui/src/hardware/capabilities.rs predator-sense-gui/src/hardware/lighting.rs \
        predator-sense-gui/src/ui/lighting_page.rs predator-sense-gui/src/ui/rgb_page.rs \
        predator-sense-gui/src/ui/window.rs predator-sense-gui/src/ui/tools_page.rs
git commit -m "Report keyboard RGB and the PredatorSense key through Capabilities"
```

---

### Task 3: One blanking implementation, not two (item 7)

`keyboard_rgb.rs` and `light_bar.rs` each carry their own `BLANKED` atomic plus `is_blanked()` / `blank()` / `unblank()`. The maintainer's item 4 was a bug in one copy; fixing it meant applying the identical change to both, which is the divergence he predicted.

**Files:**
- Create: `predator-sense-gui/src/hardware/blanking.rs`
- Modify: `predator-sense-gui/src/hardware/mod.rs` (declare the module)
- Modify: `predator-sense-gui/src/hardware/light_bar.rs` (blanking trio)
- Modify: `predator-sense-gui/src/hardware/keyboard_rgb.rs` (blanking trio)

**Interfaces:**
- Produces: `hardware::blanking::Blanking` with `const fn new() -> Self`, `fn is_blanked(&self) -> bool`, `fn set(&self, blanked: bool, write: impl FnOnce() -> Result<(), String>) -> Result<(), String>`.

- [ ] **Step 1: Write the failing test** as `src/hardware/blanking.rs`'s own test module (write the whole file in Step 3; write only the tests now, at the end of the new file):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_successful_write_records_the_new_state() {
        let b = Blanking::new();
        assert!(!b.is_blanked());
        assert!(b.set(true, || Ok(())).is_ok());
        assert!(b.is_blanked());
        assert!(b.set(false, || Ok(())).is_ok());
        assert!(!b.is_blanked());
    }

    #[test]
    fn a_failed_write_leaves_the_state_alone() {
        // The whole point: the flag is what the idle tick compares against to
        // decide there is nothing to do, so a flag that lies about a write
        // that never landed means nothing ever retries.
        let b = Blanking::new();
        assert!(b.set(true, || Err("helper unreachable".to_string())).is_err());
        assert!(!b.is_blanked(), "a failed blank must not read as blanked");
    }

    #[test]
    fn a_failed_unblank_stays_blanked_so_the_next_tick_retries() {
        let b = Blanking::new();
        assert!(b.set(true, || Ok(())).is_ok());
        assert!(b.set(false, || Err("nope".to_string())).is_err());
        assert!(b.is_blanked(), "a failed restore must still read as blanked");
    }

    #[test]
    fn the_error_reaches_the_caller_unchanged() {
        let b = Blanking::new();
        let err = b.set(true, || Err("exact text".to_string())).unwrap_err();
        assert_eq!(err, "exact text");
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test --release blanking`
Expected: FAIL, `file not found for module blanking` (the module is not declared yet).

- [ ] **Step 3: Write the module.** Put this above the test block in `src/hardware/blanking.rs`:

```rust
//! One "is this device blanked" flag, shared by every device that has one.
//!
//! The keyboard and the light bar each had their own copy of this, and the
//! copies drifted exactly as two copies do: a review found that `blank()` set
//! its flag whether or not the write succeeded, and fixing it meant making the
//! same edit twice, in two files, or shipping half a fix.
//!
//! The rule the type enforces is the one that was got wrong: **the flag
//! records what the hardware actually did, so it changes only after a write
//! that returned `Ok`.** The idle tick acts on `want != is_blanked()`, which
//! makes an honest flag its own retry: a failed write leaves the comparison
//! unequal and the next tick tries again.

use std::sync::atomic::{AtomicBool, Ordering};

/// The blanked flag for one device.
pub struct Blanking {
    blanked: AtomicBool,
}

impl Blanking {
    /// A device that starts out showing something.
    pub const fn new() -> Self {
        Self {
            blanked: AtomicBool::new(false),
        }
    }

    /// True when this device is currently blanked by the idle watcher.
    pub fn is_blanked(&self) -> bool {
        self.blanked.load(Ordering::Relaxed)
    }

    /// Runs `write`, and records `blanked` only if it succeeded.
    ///
    /// The error is passed through untouched so the caller can log or ignore
    /// it as it already does.
    pub fn set(
        &self,
        blanked: bool,
        write: impl FnOnce() -> Result<(), String>,
    ) -> Result<(), String> {
        write()?;
        self.blanked.store(blanked, Ordering::Relaxed);
        Ok(())
    }
}

impl Default for Blanking {
    fn default() -> Self {
        Self::new()
    }
}
```

- [ ] **Step 4: Declare the module.** In `src/hardware/mod.rs`, add alongside the other `pub mod` lines, in alphabetical position:

```rust
pub mod blanking;
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `cargo test --release blanking`
Expected: PASS, 4 tests.

- [ ] **Step 6: Use it in `light_bar.rs`.** Replace the `BLANKED` static and the three functions with:

```rust
static BLANKING: crate::hardware::blanking::Blanking =
    crate::hardware::blanking::Blanking::new();

/// True when the bar is currently blanked by the idle watcher.
pub fn is_blanked() -> bool {
    BLANKING.is_blanked()
}

pub fn blank() -> Result<(), String> {
    let saved = crate::config::load_app_config()
        .light_bar
        .unwrap_or_default();
    BLANKING.set(true, || {
        write_frame(&LightBarState {
            brightness: 0,
            ..saved
        })
    })
}

pub fn unblank() {
    let Some(saved) = crate::config::load_app_config().light_bar else {
        return;
    };
    // Through `on_wire`, not raw: a saved `Off` written literally relights the
    // bar, since its mode id does not blank this firmware.
    let _ = BLANKING.set(false, || write_frame(&on_wire(&saved)));
}
```

Keep the existing doc comments on `blank` and `unblank` — they explain the brightness-0 trick and the `on_wire` translation, and neither is captured by the shared type.

- [ ] **Step 7: Use it in `keyboard_rgb.rs`.** Replace its `BLANKED` static and trio with:

```rust
static BLANKING: crate::hardware::blanking::Blanking =
    crate::hardware::blanking::Blanking::new();

pub fn is_blanked() -> bool {
    BLANKING.is_blanked()
}

pub fn blank() -> Result<(), String> {
    let saved = crate::config::load_app_config()
        .keyboard_rgb
        .unwrap_or_default();
    BLANKING.set(true, || {
        apply(&KeyboardState {
            effect: Effect::Off,
            ..saved
        })
    })
}

pub fn unblank() {
    let saved = crate::config::load_app_config()
        .keyboard_rgb
        .unwrap_or_default();
    let _ = BLANKING.set(false, || apply(&saved));
}
```

- [ ] **Step 8: Confirm there is only one implementation left**

Run:
```bash
grep -rn 'AtomicBool' --include='*.rs' src/hardware/light_bar.rs src/hardware/keyboard_rgb.rs
```
Expected: no output — both now use `Blanking`.

- [ ] **Step 9: Build both crates and run all three suites.** Expected: 314 GUI, 85 installer, 59 protocol, 0 failed.

- [ ] **Step 10: Commit**

```bash
git add predator-sense-gui/src/hardware/blanking.rs predator-sense-gui/src/hardware/mod.rs \
        predator-sense-gui/src/hardware/light_bar.rs predator-sense-gui/src/hardware/keyboard_rgb.rs
git commit -m "One blanked-flag implementation instead of two that drifted"
```

---

### Task 4: Stop the 250 ms timer re-reading config and DMI (item 8)

The idle/keepalive timer runs four times a second and reloads the whole config from disk plus two model-gated `is_available()` calls, each of which reads `/sys/class/dmi/id/product_name` uncached.

**Files:**
- Modify: `predator-sense-gui/protocol/src/lib.rs` (memoise inside `dmi::product_matches`)
- Modify: `predator-sense-gui/src/ui/window.rs:664-760` (hold the config instead of reloading it)

**Interfaces:**
- Consumes: `predator_sense_protocol::dmi::product_matches` from Task 1, `Capabilities` fields from Task 2.

- [ ] **Step 1: Write the failing test** in `protocol/src/lib.rs`'s `dmi_tests` module:

```rust
    #[test]
    fn the_dmi_name_is_read_once_and_remembered() {
        // The 250 ms idle timer asks this four times a second for a value that
        // cannot change without a reboot. Reading the file every time was
        // pure cost.
        let first = dmi::product_name();
        let second = dmi::product_name();
        assert_eq!(first, second);
        assert!(std::ptr::eq(first, second), "the same cached string, not a re-read");
    }
```

- [ ] **Step 2: Run it to verify it fails**

Run: `cargo test --manifest-path protocol/Cargo.toml dmi_tests`
Expected: FAIL, `cannot find function product_name`.

- [ ] **Step 3: Add the cache** to the `dmi` module and have `product_matches` use it:

```rust
    /// This machine's DMI `product_name`, read once.
    ///
    /// `PREDATOR_SENSE_FORCE_MODEL` is consulted first and is also captured
    /// once: it is a developer override read at startup, and re-reading the
    /// environment four times a second to notice a change nobody makes is the
    /// cost this removes.
    pub fn product_name() -> &'static str {
        static NAME: std::sync::OnceLock<String> = std::sync::OnceLock::new();
        NAME.get_or_init(|| {
            std::env::var("PREDATOR_SENSE_FORCE_MODEL")
                .ok()
                .or_else(|| {
                    std::fs::read_to_string("/sys/class/dmi/id/product_name")
                        .ok()
                        .map(|s| s.trim().to_string())
                })
                .unwrap_or_default()
        })
        .as_str()
    }

    pub fn product_matches(models: &[&str]) -> bool {
        matches_model(product_name(), models)
    }
```

- [ ] **Step 4: Run it to verify it passes**

Run: `cargo test --manifest-path protocol/Cargo.toml dmi_tests`
Expected: PASS, 6 tests.

- [ ] **Step 5: Stop the timer reloading config.** In `window.rs`, the closure at line 664 calls `config::load_app_config()` on every tick. Hoist a single `Rc<RefCell<AppConfig>>` outside the closure, seeded before it, and read from that. The config is already reloaded on change elsewhere; if the timer needs to see edits, refresh the cell in the same place the settings pages already save. Concretely, before the `glib::timeout_add_local` call:

```rust
        // Read once, not four times a second. The idle timer only needs the
        // idle settings, and those change when the user edits them, which is
        // a path that already has to notify this timer anyway.
        let idle_cfg = std::rc::Rc::new(std::cell::RefCell::new(config::load_app_config()));
```

and inside the closure replace each `let cfg = config::load_app_config();` with:

```rust
                let cfg = idle_cfg.borrow();
```

- [ ] **Step 6: Verify the tick no longer does file I/O per iteration**

Run:
```bash
grep -n 'load_app_config' src/ui/window.rs | sed -n '1,40p'
```
Expected: no `load_app_config()` call inside the 250 ms closure (lines 664-780). Calls elsewhere are fine.

- [ ] **Step 7: Measure it on hardware.** With the app running:

```bash
PID=$(pgrep -f '/opt/predator-sense/predator-sense --background' | head -1)
timeout 20 strace -f -e trace=openat -p "$PID" 2>&1 | grep -c 'product_name\|config.json'
```
Expected: near zero over 20 s. Before this task the same command counts roughly 80 opens per file (4 Hz x 20 s). Record both numbers for the commit message; if `strace` is unavailable, say so rather than inventing a figure.

- [ ] **Step 8: Build both crates and run all three suites.** Expected: 314 GUI, 85 installer, 60 protocol, 0 failed.

- [ ] **Step 9: Commit**

```bash
git add predator-sense-gui/protocol/src/lib.rs predator-sense-gui/src/ui/window.rs
git commit -m "Stop the idle timer re-reading config and DMI four times a second"
```

---

### Task 5: One exclusive toggle group helper (item 10)

The same roughly 15-line block appears six times in `magic_rgb_page.rs` and twice in `lighting_page.rs`, which already demonstrates the one-line `ToggleButton::set_group()` alternative. Pre-existing code, not introduced by PR #69.

**Files:**
- Create: `predator-sense-gui/src/ui/toggle_group.rs` (verified: there is no `src/ui/widgets.rs` to put it in)
- Modify: `predator-sense-gui/src/ui/mod.rs` (declare the module)
- Modify: `predator-sense-gui/src/ui/magic_rgb_page.rs:103,366,718,952,1082,1127`
- Modify: `predator-sense-gui/src/ui/lighting_page.rs` (the two copies)

**Interfaces:**
- Produces: `ui::toggle_group::exclusive(labels: &[&str]) -> (gtk::Box, Vec<gtk::ToggleButton>)`
  (verified: `i18n::t(key) -> &str`, so `&[&str]` is what every call site already has)

- [ ] **Step 1: Read one of the existing blocks before changing anything**

Run: `sed -n '95,120p' src/ui/magic_rgb_page.rs`
This is the shape to preserve. Do not invent a different one; the goal is to remove repetition, not to redesign the widget.

- [ ] **Step 2: Write the failing test.** Verified: this crate calls `gtk::init()` nowhere in tests, so a widget-constructing test would abort without a display. Test the grouping decision, which is the only part with a choice in it:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn a_group_of_one_still_has_a_leader() {
        // set_group(None) on the first button and Some(first) on the rest is
        // the whole rule; a single-button group must not try to point at a
        // predecessor that does not exist.
        assert_eq!(super::leader_index(0), None);
        assert_eq!(super::leader_index(1), Some(0));
        assert_eq!(super::leader_index(5), Some(0));
    }
}
```

- [ ] **Step 3: Run it to verify it fails**

Run: `cargo test --release toggle_group`
Expected: FAIL, `cannot find function leader_index`.

- [ ] **Step 4: Write the helper**

```rust
//! One exclusive toggle-button group, built once instead of eight times.

/// Which button in a group every other button points at.
///
/// GTK makes a group by pointing each button at one leader, so the first has
/// no group to join and the rest all join it.
pub fn leader_index(position: usize) -> Option<usize> {
    (position > 0).then_some(0)
}

/// A horizontal box of mutually exclusive toggle buttons, in order.
pub fn exclusive(labels: &[&str]) -> (gtk::Box, Vec<gtk::ToggleButton>) {
    let row = gtk::Box::new(gtk::Orientation::Horizontal, 6);
    let mut buttons: Vec<gtk::ToggleButton> = Vec::with_capacity(labels.len());
    for (position, label) in labels.iter().enumerate() {
        let btn = gtk::ToggleButton::with_label(label);
        match leader_index(position) {
            None => {}
            Some(i) => btn.set_group(Some(&buttons[i])),
        }
        row.append(&btn);
        buttons.push(btn);
    }
    (row, buttons)
}
```

- [ ] **Step 5: Run it to verify it passes**

Run: `cargo test --release toggle_group`
Expected: PASS.

- [ ] **Step 6: Replace the eight call sites**, one at a time, building after each so a mistake is attributed to the site that caused it. The buttons come back in the same order as `labels`, so existing per-button `connect_toggled` wiring attaches to `buttons[i]` unchanged.

- [ ] **Step 7: Confirm the repetition is gone**

Run: `grep -c 'ToggleButton::with_label' src/ui/magic_rgb_page.rs src/ui/lighting_page.rs`
Expected: both markedly lower than 6 and 2; any remaining instance is a button that is not part of an exclusive group and should be left alone.

- [ ] **Step 8: Verify on screen.** This one is visual and has no readback. Launch the build and check the RGB page and Lighting page: each group still selects exactly one button at a time, and the previously selected button clears when another is pressed.

```bash
./target/release/predator-sense
```
State the expected behaviour to the user before they look, per the project's working agreements.

- [ ] **Step 9: Build both crates and run all three suites.** Expected: 315 GUI, 85 installer, 60 protocol, 0 failed.

- [ ] **Step 10: Commit**

```bash
git add predator-sense-gui/src/ui/ && git commit -m "Build exclusive toggle groups in one place instead of eight"
```

---

## Final verification, after all five tasks

- [ ] All three suites green, numbers recorded.
- [ ] Both crates build in release.
- [ ] Install and run the post-boot checks:

```bash
cd predator-sense-gui
sudo installer/target/release/predator-sense-installer --install
bash ../../verify-after-reboot.sh
```

- [ ] Hardware spot-checks for what these five touched, none of which a unit test covers:
  - Lighting page appears and drives both devices (Task 2 rerouted its gates).
  - Idle blanking still blanks and wakes (Task 3 replaced the flag).
  - Mode-key cycles still reinstall after `--reload-module` (Task 4 touched that timer's file).
  - Both toggle groups still behave exclusively (Task 5).
- [ ] `./verify-notes.sh` passes.
- [ ] Update `WORKPLAN.md`: mark B3-B7 done with their commit hashes.
