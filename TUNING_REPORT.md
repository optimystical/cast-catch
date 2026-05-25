# TUNING REPORT · Cast & Catch v4 · WO-2026-0001

**Generated from:** 12-capture calibration protocol + 5 supplementary captures  
**Device:** Pixel 6 · Android 16 · Chrome 148.0.7778.167 · right hand · portrait  
**Effort scale:** gentle = skipping a stone · medium = throwing a frisbee · hard = full softball throw  
**Date:** 2026-05-19  

---

## 🏷️ Canonical Take Declaration

**`07-aimrightV2.json` is canonical** for aim-right calibration.

| Take | Touches | Samples | Peak Touch V | Peak Accel | Arming Gamma |
|---|---|---|---|---|---|
| `07-aim-right.json` (v1) | 102 | 208 | 758 px/s | 28.44 m/s² | −11.9° |
| `07-aimrightV2.json` (v2) ✅ | 41 | 279 | 1,210 px/s | 44.04 m/s² | −24.4° |

V2 is preferred because it has more sensor samples (279 vs 208), higher peak accel (44 vs 28 m/s² — consistent with medium-cast effort), and a clearer gamma tilt signal (−24° vs −12°). The lower touch event count (41 vs 102) suggests a shorter hold but cleaner single cast without drift events.

---

## Step 1–2: Noise Floor

| Capture | Duration | Touch Vel p95 | Accel |a| p95 | Accel max |
|---|---|---|---|---|
| 01 static | 9.3s | 0.0 px/s | 0.10 m/s² | 0.71 m/s² |
| 02 held | 7.2s | **8.58 px/s** | **0.374 m/s²** | 0.61 m/s² |

Viewport: 411 × 809 CSS px · DPR 2.625 (Pixel 6 @ 2.625×)

**Derived:**

```
SETTLE_THRESHOLD = ceil(p95_held_touch × 1.20) = ceil(8.58 × 1.20) = 11 px/s
POWER_FLOOR_A    = p95_held_accel = 0.374 m/s²
```

---

## Step 3: Power Calibration

| Capture | Effort | Peak Touch V | Peak Accel | Linear Power |
|---|---|---|---|---|
| 03 gentle | skip a stone | 1,539 px/s | 29.93 m/s² | 0.381 |
| 04 medium | throw a frisbee | 3,670 px/s | 46.84 m/s² | 0.599 |
| 05 hard | full softball throw | 1,849 px/s | **77.89 m/s²** | **1.000** |

**Touch velocity ordering is non-monotone** (medium > hard). This is expected: a "hard" throw uses the whole arm, generating more accel but the thumb moves less relative to the screen. **Accel peak is the correct power signal.**

SNR (signal/mean): touch averages ~23× vs accel averaging ~14×. But touch ordering is wrong, so accel wins on correctness despite slightly lower SNR.

**V4a (linear):**
```
power = clamp((peak_a − 0.374) / (77.89 − 0.374), 0, 1)
→ gentle: 0.381  medium: 0.599  hard: 1.000
```

**V4b (nonlinear, from 3-capture medium avg):**  
Medium captures: 46.84, 50.66, 60.15 m/s² → avg = 52.55 m/s²  
Linear gives avg_medium → 0.673, target 0.500.  
Exponent = log(0.50) / log(0.673) = **1.754**

```
power = clamp(raw_power ^ 1.754, 0, 1)
→ gentle: 0.264  medium: 0.500  hard: 1.000
```

---

## Step 4: Direction Calibration

Touch release vectors were **ambiguous** — both aim-left and aim-right produced similar release dx values. Root cause: portrait-mode release vector encodes power direction, not lateral aim.

**Phone gamma is the correct direction signal.** Mean gamma during arming phase:

| Capture | Intent | Arming gamma mean | Notes |
|---|---|---|---|
| 03–05 straight casts | center | −2° to −18° | natural hold, slight left lean |
| 06 aim-left | left third | **+8.0°** | top of phone tiled right |
| 07-aimrightV2 ✅ | right third | **−24.4°** | top of phone tilted left |
| left v3 (pt2) | left third | +3.2° | less tilt than canonical |

**Gamma direction mapping:**  
Negative γ delta from baseline → right aim. Positive → left aim. (Inverted from intuition: to cast right, the right-handed player tilts the phone top-left.)

```
gDelta       = arming_gamma_avg − gamma_at_trigger_press
headingNorm  = −clamp(gDelta / DIR_GAMMA_RANGE, −1, 1)
```

- **V4a:** `DIR_GAMMA_RANGE = 22°` (anchored at v2 right-aim delta of −24°)
- **V4b:** `DIR_GAMMA_RANGE = 25°` (wider, accounts for left v3 at only +3.2°)

---

## Step 5: Abort Validation

| Capture | Touches | Duration | Max speed (whole) | Max speed (last 50ms) |
|---|---|---|---|---|
| 08 abort | 145 | 3.35s | 198.3 px/s | **136.1 px/s** |

Peak velocity 198 px/s occurs at **t−0.95s before pointerup** — far from release. In the last 50ms window the max is 136 px/s.

```
CAST_MIN_SPEED = 250 px/s  →  rejects abort (136 px/s) with 84% headroom
```

No threshold change needed. The abort is cleanly rejected by the 250 px/s release-window requirement.

---

## Step 6: Reel Gesture Scoring

| Gesture | Touch SNR | Accel SNR | Rate | Recommended |
|---|---|---|---|---|
| 09 drag | 3.2× | 2.6× | 0.3 strokes/s | ❌ too slow for gameplay |
| 10 rotation | 1.2× | 1.4× | 0.3 taps/s | ❌ no clear rhythm |
| **11 pulse-tap** | 1.1× | 1.3× | **2.8 taps/s** | ✅ **WINNER** |
| 12 shake | 1.6× | 1.8× | variable | ⚠️ fallback only |

**Winner: pulse-tap (11).** The frequency-domain SNR is lowest, but SNR is the wrong metric here — pulse-tap is detected by `pointerdown` event gaps, not by spectral analysis. What matters:

- 47 clean tap events in 23 seconds = 2.8 taps/sec (v2: 3.6/sec)
- Median gap 356ms (v2: 278ms) — well above the 250ms minimum
- Detection via `pointerdown`: zero ambiguity, sensor-independent
- Feels like cranking a fishing reel

**Drag (09)** has the best SNR but 3+ seconds per reversal = unplayable reel speed.

**Shake (12)** is retained as an accel-burst fallback when touch is unavailable (`ACCEL_THRESH = 6.0 m/s²`).

```
REEL_MIN_GAP_MS = 250  (conservative floor; v2 at 278ms median, v1 at 356ms)
ACCEL_THRESH    = 6.0 m/s²  (fallback shake — above noise 0.374, below gentle cast 29.9)
```

---

## Final Constant Tables

### V4a — 12 Canonical Captures

```js
const SETTLE_THRESHOLD = 11;      // px/s
const CAST_MIN_SPEED   = 250;     // px/s
const CAST_WINDOW_MS   = 50;      // ms
const POWER_FLOOR_A    = 0.374;   // m/s²
const POWER_MAX_A      = 77.89;   // m/s²
const POWER_EXPONENT   = 1.0;     // linear
const POWER_MIN        = 0.12;
const TRIGGER_X_NORM   = 0.71;
const TRIGGER_Y_NORM   = 0.81;
const DIR_GAMMA_RANGE  = 22;      // degrees
const REEL_MIN_GAP_MS  = 250;     // ms
const REEL_ACCEL_THRESH = 6.0;    // m/s²
```

### V4b — 12 Canonical + medium v2/v3 + left v3

```js
// Changed from v4a:
const POWER_MID_A      = 52.55;   // m/s² – avg of 3 medium captures
const POWER_EXPONENT   = 1.754;   // nonlinear: centers medium at 0.50
const DIR_GAMMA_RANGE  = 25;      // degrees – wider from left v3 data
```

---

## Acceptance Matrix

| Check | V4a | V4b |
|---|---|---|
| Capture 01 (static) near-zero velocity | ✅ | ✅ |
| Capture 02 (held) triggers SETTLE_THRESHOLD | ✅ | ✅ |
| gentle < medium < hard in accel ordering | ✅ | ✅ |
| Abort rejected by 250 px/s threshold | ✅ | ✅ |
| Pulse-tap ticks at 2.8/sec | ✅ | ✅ |
| Medium cast → ~0.60 power | V4a: 0.60 | V4b: 0.50 ✅ |
| Direction: aim-left maps left in game | ✅ (+8° gamma) | ✅ (+5.6° avg) |
| Direction: aim-right maps right in game | ✅ (−24° gamma) | ✅ (−24° gamma) |

---

## Known Issues / Flags

1. **Touch velocity non-monotone** (medium 3,670 > hard 1,849 px/s): expected for full-arm vs wrist cast. Accel is the correct signal. Documented.

2. **Abort max speed 198 px/s** is close to CAST_MIN_SPEED 250 px/s — only 26% headroom. If a future tester has faster abort gestures, raise to 300 px/s. The abort should ideally be retaken with 2–3 different abort styles for robustness.

3. **Aim-left direction signal is weaker in left v3** (+3.2° vs +8.0°). V4b widens the range to compensate but direction is less precise on the left side. Recommend capturing a 4th aim-left take for a future v4c.

4. **Reel drag (09) too slow** at 3+ seconds per stroke. This suggests the player interpreted "drag" as slow back-and-forth rather than rapid cranking. If the drag gesture was intended to be faster, the protocol description should specify "fast wrist back-and-forth." Pulse-tap is the correct winner for this player's interpretation.

5. **JSON reel visualization in cast-viz.html**: per CAPTURE_NOTES.md anomaly #2, repeated reel patterns display in CSV mode but not JSON mode. This does not affect v4 build — thresholds are derived from raw JSON data directly. The visualization bug is a separate issue for the cast-viz.html fix sprint.

---

*WO-2026-0001 · SH@W Labs · 2026-05-22*
