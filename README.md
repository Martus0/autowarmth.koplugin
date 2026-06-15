# AutoWarmth — Enhanced Fork

A modified version of KOReader's built-in `autowarmth` plugin with independent dawn/dusk controls, a configurable transition mechanism, and several bug fixes.

## Changes vs original

### Bug fixes

**Schedule interpolation skipped most hardware steps on non-round devices**
The original loop iterated in 1% warmth increments and filtered with `frac(n * scale) == 0`, which only passes integer hardware boundaries. On a Kindle (24 steps), this means only steps 6, 12, 18, 24 (multiples of 25%) were ever used — 2 changes instead of 14 for a 0→64% ramp. Fixed to iterate over hardware steps directly, using all available levels.

**Warmth always set to 100% when dark mode enabled**
The original encoded dark mode as `warmth + 1000` but then decoded it with `math.min(val, 100)`, which always produced 100. Fixed to `val - 1000`.

**Schedule interpolation ignored actual warmth when dark mode active**
The warmth ramp between two dark-mode keyframes used `math.min(val, 100)` as the base, producing a 100→100 difference of zero (no steps) or incorrectly starting from 100 instead of the real warmth value.

**Dark mode turned off mid-ramp**
Intermediate warmth steps between two keyframes were stored as plain values (no dark mode flag), causing `setWarmth` to call `onSetNightMode(false)` during a ramp between two dark-mode keyframes.

**UI displayed 100% instead of actual warmth**
The menu label and "Currently active parameters" summary both hardcoded `"100 %"` for any dark-mode entry.

### New features

**Independent sunrise/sunset and civil dawn/civil dusk controls**
The original plugin mirrored sunrise↔sunset and civil dawn↔civil dusk, forcing them to always have the same warmth and dark mode setting. These four keyframes are now independently controllable.

**Configurable transition window per keyframe**
See below.

---

## Transition mechanism

By default (transition = 0 min) warmth changes gradually across the entire time window between two keyframes — the original behaviour.

When a transition duration is set, the warmth ramp is compressed into a shorter window. The position of that window depends on the direction of the warmth change:

**Decreasing warmth → transition fires AFTER the keyframe**

```
Civil dawn (100%)
|--- ramp X min ---|--- hold at 8% ---|
                                   Sunrise (8%)
```

The ramp completes within X minutes of civil dawn. Warmth then holds flat at the destination value until the next keyframe.

**Increasing warmth → transition fires BEFORE the keyframe**

```
                   |--- hold at 0% ---|--- ramp Y min ---|
Solar noon (0%)                                        Sunset (64%)
```

Warmth holds flat at the source value and only starts ramping Y minutes before the destination keyframe, arriving at exactly the right value at the keyframe time.

**If the transition duration exceeds the available window**, it is silently clamped to the full window (same as setting 0).

**Solar noon is a special case.** It sits at the minimum of the day curve so its own transition value is never used — the ramp arriving at noon is controlled by the sunrise transition (decreasing, fires after sunrise), and the ramp leaving noon is controlled by the sunset transition (increasing, fires before sunset). The flat hold at noon's warmth between those two ramps lasts however long is left between them.

**Dark mode** stays on through any ramp that starts from a dark-mode keyframe. It turns off precisely at the first non-dark keyframe — not mid-ramp.

### Example schedule

| Keyframe | Warmth | Dark mode | Transition |
|---|---|---|---|
| Civil dawn | 100% | ON | e.g. 20 min after |
| Sunrise | 8% | OFF | e.g. 30 min after |
| Solar noon | 0% | OFF | — |
| Sunset | 64% | ON | e.g. 45 min before |
| Civil dusk | 100% | ON | e.g. 20 min before |

Result: dark mode stays on from civil dawn through the ramp to sunrise, turns off at sunrise, warmth drops to 0% within 30 minutes of sunrise and holds flat all day, then ramps up to 64% in the 45 minutes before sunset, and continues rising to 100% in the 20 minutes before civil dusk.

---

## Installation

Copy `main.lua` to `plugins/autowarmth.koplugin/main.lua` on your KOReader device. On Kindle this is typically `/mnt/us/extensions/autowarmth.koplugin/` (user plugin directory, takes priority over the built-in version).

To reapply changes after a KOReader update, run from the KOReader repo root:

```bash
patch -p1 < autowarmth-warmth-control.patch
```
