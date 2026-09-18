# GPU power headroom: 88 W observed against a 140 W rating

**Status:** open question, nothing implemented. Found while measuring something
else (CHANGELOG §30 and the Turbo pinning question); recorded here rather than
chased, because it is a bigger lead than what it interrupted.

**Machine:** Acer Predator Helios 16 PH16-71, i7-13700HX, RTX 4060 Laptop,
BIOS V1.18, CachyOS kernel 7.2.5-1-cachyos, NVIDIA driver 615.71.09, on AC.

## The observation

`nvidia-smi -q -d POWER` reports:

```
Current Power Limit : 80.00 W      Default Power Limit : 80.00 W
Min Power Limit     :  5.00 W      Max Power Limit     : 140.00 W
```

Under a GPU-saturating load (hashcat MD5, 100% utilisation), with the firmware
thermal profile at the Turbo index, the GPU never exceeded **88.5 W**:

| fans | GPU power | GPU temp | SM clock |
|---|---|---|---|
| firmware curve | 75-85 W | 83-85 C | 2285-2413 MHz |
| forced to maximum (~6050 rpm) | steady 87-88 W | 83-84 C | 2490 MHz |

So roughly 7 W of the 60 W gap between the 80 W base and the 140 W ceiling is
ever used, and better cooling buys about 10 W without lifting the ceiling.

The PH16-71's 4060 is advertised at up to 140 W total graphics power.

## What is already ruled out

- **Not the CPU policy.** Fifteen paired measurement windows across four
  experiments varied `min_perf_pct` between 100 and 17 under idle, partial CPU
  load and full CPU load, with fans on the firmware curve and forced to
  maximum. GPU power deltas were within +/-0.7 W and changed sign. See
  CHANGELOG §30.
- **Not CPU contention generally.** Adding 25 W of CPU load costs the GPU about
  3 W, measured against bracketing controls that themselves drifted 3.1 W. Weak
  coupling, most likely thermal through shared heatpipes.
- **Not a dead Dynamic Boost.** `nvidia-powerd` 2.0 is active and was caught
  working: three seconds into a GPU load the P-core `scaling_max_freq` was
  pulled down to ~3.7 GHz and later released back to 4.8 and 5.0 GHz while GPU
  power climbed 81.9 W to 86.8 W. NVIDIA's driver README confirms that
  `scaling_max_freq` is the only interface it uses.
- **Only partly thermal.** Max fans lifted the steady figure from 76-85 W to
  87-88 W and held 83 C, which is not obviously a thermal wall.

## Open questions for a future session

1. **Is the 80 W base limit meant to move?** These measurements were taken with
   the firmware thermal profile at the Turbo index (raw index 5, the 115 W PL1
   tier by the calibration in `thermal_profiles.json`), and the GPU's
   `Current Power Limit` stayed at 80.00 W throughout. If the Acer firmware
   profile is supposed to raise the GPU budget, it is not doing so, or not
   through a channel `nvidia-smi` reports.
2. **Is this the known Linux limitation?** NVIDIA's own forums carry a report of
   `nvidia-powerd` giving only about +5 W above an 80 W default on a 4070
   laptop, which matches the ~7 W seen here. If it is that, the ceiling is not
   this project's to move.
3. **Does the SBIOS power budget table read as zeros?** A related forum report
   traces a stuck Dynamic Boost to an `NPCF` SSDT method returning a zeroed
   power budget. This is greppable offline: the workspace dumps from this
   machine already contain `NPCF`, verified 2026-09-18, in
   `firmware/dsdt.dsl` (31 occurrences), `firmware/ssdt11.dsl` and
   `firmware/ssdt13.dsl`. Deliberately not opened yet, so the next session gets
   an unprejudiced read.
4. **What does Windows reach on the same chassis?** Reviews of the PH16-71 quote
   much higher sustained GPU power. A number measured on this specific machine
   under Windows PredatorSense would say whether the gap is Linux-side or the
   unit's own configuration.
5. **Can `nvidia-powerd` be made to explain itself?** It logs to the journal and
   may have a verbose mode; its view of the available budget would answer
   question 1 directly.

## Reproducing the measurements

```console
# GPU load that actually saturates power (vkmark does not: 25 W, 24% utilisation)
hashcat -a 3 -m 0 -D 2 -w 4 --runtime 40 --potfile-disable --quiet \
  7d0bd9b3b8a5f6e2c1a4f09e3b7d2c65 '?a?a?a?a?a?a?a?a'

# telemetry while it runs
nvidia-smi --query-gpu=power.draw,clocks.sm,utilization.gpu,temperature.gpu --format=csv
nvidia-smi -q -d POWER | grep -i "power limit"

# the lever Dynamic Boost uses
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_max_freq | sort -u

# CPU package power, for the other side of the budget
sudo cat /sys/class/powercap/intel-rapl:0/energy_uj   # delta over a known interval

# fans to maximum, to separate thermal limits from budget limits
sudo sh -c 'echo 1 > /sys/class/hwmon/hwmon5/pwm1_enable; echo 255 > /sys/class/hwmon/hwmon5/pwm1'
```

Stop the app first (`pkill -x predator-sense`) when the fan state must stay put,
or its reconciler will take the fans back within a few seconds.

## Caveats on the evidence

- hashcat is a compute load. A game mixes CPU and GPU work differently and may
  reach a different equilibrium; nothing here measures frame pacing.
- `Max Power Limit: 140.00 W` is what the vBIOS reports as its ceiling. It is
  not a promise that this chassis will ever deliver it.
- The GPU's own power limit could not be set from userspace at all on this
  machine: `nvidia-smi -pl` fails with "not supported in current scope", which
  is a long-standing finding in this project and is why the app treats GPU
  wattage as best-effort.

## References

- [Dynamic Boost on Linux, NVIDIA driver README](https://download.nvidia.com/XFree86/Linux-x86_64/580.126.18/README/dynamicboost.html)
- [nvidia-powerd gives only +5W above the default 80W limit, NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/nvidia-dynamic-boost-nvidia-powerd-gives-only-5w-boost-above-default-power-limit-80w-4070-laptop/327208)
- [RTX 5050 Mobile Dynamic Boost stuck, NPCF SSDT returns zeroed power budget](https://forums.developer.nvidia.com/t/bug-rtx-5050-mobile-dynamic-boost-stuck-at-35w-npcf-ssdt-returns-zeroed-power-budget/382919)
- [Acer Predator Helios 16 PH16-71 review, LaptopMedia](https://laptopmedia.com/review/acer-predator-helios-16-ph16-71-review-liquid-metal-meets-a-huge-power-draw/)
