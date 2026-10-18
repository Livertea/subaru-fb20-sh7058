# Computation Flow: Fuel, Timing, and DBW

> **⚠️ UNVERIFIED RESEARCH — MAY BE 100% WRONG**
>
> This page documents *order of operations* — what the ECU is believed to do with its tables, and in what sequence. Everything here is **exploratory and unverified**. Chain-stage order, table roles, and especially the combination formulas are hypotheses. **Zero hardware validation. Zero flashed ROMs.** Do not tune from this page.

---

## Why This Page Exists

Stolen from the Saab Trionic scene (see [Saab survey](../SAAB_SCENE_SURVEY.md) in analysis notes): before showing any fuel tables, they document the **12-step fuel calculation** — each step references the specific tables involved, with signal-flow diagrams showing how they combine into a final output.

We spent ~13 hours mapping **863 tables** (the "what"). This page is the first attempt at the **"how"** — the computation the ECU performs on those tables. Some stages are grounded in disassembly; others are inferred from table shapes and reference-definition cross-checks; the rest are open threads. Every claim carries an evidence tag.

### Evidence tags used on this page

| Tag | Meaning |
|-----|---------|
| 🔬 DISASSEMBLED | Verified in Capstone/Ghidra disassembly of this ROM |
| 🧮 ROM-VERIFIED | Raw bytes read from this ROM on 2026-10-18, values match the claim |
| 📖 REFERENCE-DERIVED | Matches Euro/CVT reference definitions; assumed same architecture, not proven |
| 🧪 SHAPE-INFERENCE | Table data physically behaves like the described phenomenon, but the code path is not traced |
| ❓ OPEN THREAD | Unknown. Listed so it isn't mistaken for solved. |

---

## 1. DBW Chain (most complete)

Torque-based throttle control. The ECU never maps pedal→plate directly; it computes a requested torque and converts that to a plate position. 🧪 SHAPE-INFERENCE on architecture, 🔬 chain endpoints confirmed.

```
                     ┌──────────────────────────────────────────────┐
                     │  STEP 1: Driver torque request                │
                     │  Requested Torque A/B  @0xFA08C / @0xFA774   │
                     │  20×18 u16BE, x*0.0078125                   │
                     │  X = pedal % (0–100)   Y = RPM (1000–7010)   │
                     └──────────────────────┬───────────────────────┘
                                            │ raw torque value 0–~183 🧮
                                            ▼
                     ┌──────────────────────────────────────────────┐
                     │  STEP 2: Torque limit clamp                   │
                     │  Requested Torque Limit A/B @0xFB698/@0xFB79C │
                     │  10×5 float32BE                              │
                     │  ❓ OPEN THREAD: exact placement in chain      │
                     └──────────────────────┬───────────────────────┘
                                            ▼
                     ┌──────────────────────────────────────────────┐
                     │  STEP 3: Torque → throttle plate              │
                     │  Target Throttle Plate Position A/B/C         │
                     │  @0xF9240 / @0xFAEF4 / @0xF9800              │
                     │  X = requested torque (5–187) Y = RPM        │
                     │  data = plate % (x*0.002270655357)           │
                     └──────────────────────┬───────────────────────┘
                                            ▼
                                throttle servo (code untraced ❓)
```

**Details and evidence:**

- **Step 1** 🧮 ROM-VERIFIED: `Requested Torque A @0xFA08C` row 0 raw = `[0, 9, 1636, 2991, 4339, 7412, 9316, 11259]` → ×0.0078125 = `[0.0, 0.07, 12.78, 23.37, 33.90, 57.91, 72.78, 87.96]`. X axis @0xF9FF4 = pedal %, Y axis @0xFA044 = RPM 1000–7010. Identical duplicates exist at 0xFA40C/0xFAAF4 — best hypothesis is **cruise / non-cruise variants** (❓ which is which, and the selection logic, is an open thread).
- **Step 2** 📖 REFERENCE-DERIVED: matches Euro "Requested Torque Limit" pattern. C/D variants were searched for and **not confirmed** (removed from definition). Whether the limit clamps the step-1 output before step 3, or clamps the final plate angle, is an ❓ open thread.
- **Step 3** 🧮 ROM-VERIFIED: A @0xF9240 (50×12) and B @0xFAEF4 (50×12), C @0xF9800 (50×19). The 50-u16 row stride holds 48 data columns + 2-u16 footer `[45089, X]` where 45089 (0xB021) = **102.38% max clamp** (🧪 shape-inference: a hard clamp value baked per row). Which of A/B/C is active when (cruise? failsafe? sport?) is an ❓ open thread.
- **Negative finding (confirmed):** there is **no "Requested Torque Base"** table on this ECU. The requested-torque A/B output (0–~183) feeds the target-throttle tables (accepting 5–400) directly — a **2-step chain**, unlike the 3-step HPAcademy textbook chain. Mazda *does* have a Base table, so this is a genuine Subaru architectural difference. 🔬 SEARCHED-AND-ABSENT.
- **Negative finding (confirmed):** no throttle closing-rate / damping table exists. Subaru DBW is torque-based; RX-8-style "Throttle Plate Request Decreasing Filter Factor" does not apply here. Rev hang is handled elsewhere (see Fuel chain §2.7). 🔬 SEARCHED-AND-ABSENT.
- **Limiters** 🧮 ROM-VERIFIED where noted: `Rev Limit (Fuel Cut) @0xCEA78 = 6700.0 RPM` (exact float). Speed-limiting throttle tables @0xF8E44/0xF8E50/0xF8E5C/0xF8E68 show **increasing** values = progressive throttle closure, not hysteresis pairs (🧪). `Speed Limiting (Fuel Cut) B @0xCE6B0`, `Rev Limit Fuel Resume (MP) @0xC9848`, `Rev Limit B @0xD401C` also in the Limits category.
- ❓ OPEN THREAD: the final throttle-plate servo control loop (PID? feedforward?) and how the Target Throttle output reaches the motor — code region not yet traced.

---

## 2. Fuel Chain (partially decoded)

The most complex chain. Stages are presented in Saab-style numbered order. **The overall order of operations is the hypothesis;** individual stage tables are the evidence.

```
MAF/MAP ──▶ VE (cylinder fill) ──▶ base fuel map (OL target AFR)
                                              │
            ┌─────────────────────────────────┘
            ▼
   closed-loop target comps ──▶ transient (tip-in / X-tau)
            │                             │
            ▼                             ▼
   warmup + cranking              CL↔OL transition logic
            │                             │
            └──────────────┬──────────────┘
                           ▼
               final injector pulse width
               (injector latency + flow — flow scalar ❓)
```

### 2.1 Air measurement

- `MAF Sensor Scaling @0xDCEA0` (48-el f32, Y = volts @0xDCDDC 0.0–4.26V, data 0.96–290 g/s) — 📖 reference-confirmed as the real one; a 65-el impostor @0xD5EEC was ruled out (it's a limit/diagnostic curve, identity TBD).
- `MAP Sensor Scaling @0xDCCD8` — f32BE `[250.0, −150.0]` 🧮 (units TBD, likely kPa-ish offsets).

### 2.2 Cylinder fill — VE (conflict resolved)

- 42 tables `@0xC4E70 + i·0x84` (6×11) + 42 High-RPM tables `@0xC6418 + i·0xD8` (6×18), scaling `x·0.00305175781` → % 🧮.
- **"Volumetric Efficiency (Cylinder Fill)" is correct;** the competing "Engine Load Compensation" label was retired after a decisive evidence round (🟢 resolved, see analysis notes).
- ❓ OPEN THREAD: what selects among the 42 variants (ECT? baro? per-gear? — the 0xDB280–0xDBFD0 6×5 "per-gear" family is a suspect).

### 2.3 Base fuel — open-loop target

- `Primary Open Loop Fueling Non-Cruise @0xD46E8` (10×16, X=load @0xD4680, Y=RPM @0xD46A8) and `Cruise @0xD487C` (10×11, X=RPM @0xD4850, Y=load @0xD4828). Scaling `AFR = 14.7/(1 + x·0.0078125)` 🧮.
- 🧮 ROM-VERIFIED: row 0 of Non-Cruise is all zeros → **14.7 stoich** in the light-load region; enrichment appears only at higher load/RPM. Physical sanity: correct.
- ❓ OPEN THREAD: cruise vs non-cruise selection logic.

### 2.4 Closed-loop targets

- `CL Fueling Target Compensation A–D (Load)` @0xD34A6 / 0xD36F6 / 0xD3948 / 0xD3B96 (11×21 each) and `CL Fueling Target Compensation (ECT)` @0xC3FB0 (2D 16-el) 🧮 exist. (C was corrected +2 bytes by the re-triage: 0xD3946 → 0xD3948 — it had been reading 2 bytes of Y-axis tail as data.)
- ❓ OPEN THREAD: which of A–D applies when (bank 1/2 × cruise/non-cruise is the leading guess); how these interact with the OL maps in the transition zone.

### 2.5 Transient enrichment — tip-in and X-tau

Two mechanisms coexist in this ROM, and **how they relate is the biggest fuel-chain open thread**:

**(a) Pre-computed transient response curves** 🧪 SHAPE-INFERENCE (HIGH confidence on the phenomenon):
- `Throttle Tip-in Enrichment A @0xD13D4` and `B @0xD1474` (**16×8**, X = **ECT** @0xCEB64, Y = time-since-tip-in @0xD13B4 / @0xD1454, scale 1/256, unit % enrichment — resolved by the re-triage; the directory's dims and the earlier 8×8 reading were wrong, another "directory lies" instance).
- `Tip-in Enrichment Compensation (RPM) @0xCFAC8`, `Tip-in Enrichment Compensation (ECT) A–J` @0xD2014… (16×5, x/65536).
- The **8×8 cluster @0xD2864–0xD3024**: X axis `[1..8]` = **engine cycles (intake events) since a fueling transient**. Evidence: Table 10 at 1000 RPM shows `[0.1187, 0.1187, 0.0500, 0.0250, 0.0125, …]` — consecutive ratios ~0.5, textbook first-order exponential decay = fuel-film evaporation signature. Tables 4/5 *rise* with X (film **accumulating** onto walls). Paired structure (0,1)/(2,3)/(4,5)/(8,9) likely = different transient types (tip-in vs tip-out? severity levels?) — ❓ trigger conditions unknown, needs logger or disassembly.
- `Fuel Film Wall-Wetting X @0xD319C` (10×9, x/32768, X=load @0xD3150, Y=RPM @0xD3178) — the "X" parameter of the X-tau model. 🧮 ROM-VERIFIED: 0.615 (idle, low load — poor atomization, high vacuum) → 0.050 (high load). Monotonic in both axes. Physically textbook.

**(b) Real-time X-tau filter in code** 🔬 DISASSEMBLED:
- Wall-wetting delta `@0x36B1C` (RAM history shift register `@0x363DE`), pair selection `@0x365DC` (6 ECT-indexed tables `0xCF946–0xCF9E6`), cycle counter `@0x36640` (stops at 625 cycles, u16 `@0xCE3D2`), filter `@0x36616` computing **`(A·B + C·D + E·F)·G`** — a weighted-sum filter over history terms.
- ❓ OPEN THREAD: is (b) the code that *consumes* the (a) tables, or are they two parallel implementations (code path vs. calibration fallback)? The 8×8 tables demonstrate the ECU uses **pre-computed transient response curves rather than real-time X-tau calculation** — at least in some modes.

### 2.6 Warmup and cranking

- `Warmup Enrichment Multiplier @0xCF399` (8-el, x/32768, ECT) — 🧮 ROM-VERIFIED: `[1.0, 1.01, 1.76, 1.76, 1.76, 1.51, 1.3, 1.0]` — peaks 1.76× at −20..0°C, tapers to 1.0 by 30°C.
- `Post-Start Decay Multiplier @0xCF0F0`, `Warmup Phase Weighting @0xCF21D` (🧪 medium).
- Cranking IPW: `1 @0xCF586`, `2 @0xCF5E6` (x·0.008 ms, Y=ECT @0xCEB64 — 🧮 verified 22.9 ms cold decaying to 10.8 ms), `3 @0xCF646`, `Warm Restart @0xCF766`.

### 2.7 CL↔OL transition

- `CL to OL Transition with Delay (Throttle) @0xD5180` / `(Accelerator) @0xD51D0` and `B @0xD5220` (16-el u8, RPM axis 500–5200) 🧮 ROM-VERIFIED: throttle thresholds `[16.3, 16.3, 25.0, 27.9, 31.4, 33.1 …]%`, accelerator `[39.1, 41.5, 44.4, 49.8 …]%`. Controls the fueling-mode switch on tip-in/lift — directly relevant to decel behavior.
- ❓ OPEN THREAD: the "with delay" counter logic and hysteresis between the two tables.

### 2.8 Decel, DFCO, and rev hang

- **Rev hang mechanism (resolved):** `Idle Airflow Decel Adder (Vehicle Speed) A @0xD9720` / `B @0xD9758` (9-el, x·0.001525879 g/s, X = vehicle speed 0–300 km/h) — 🧮 ROM-VERIFIED: `0.0, 0.0, 1.0, 1.75 …` g/s. When the throttle closes while moving, the ECU *adds* idle airflow proportional to vehicle speed — this is the rev hang. For delete: zero the tables (start by halving) 🧪 tuning hypothesis, unvalidated.
- `Decel Initial Idle Min Airflow A/B`, ramping adders, and the CL/OL transition tables above form the supporting cast (some from Euro cross-ref, 📖).
- ❓ OPEN THREAD: DFCO (deceleration fuel cut-off) — the 16-el RPM resume threshold and the cut-counter/enrich singles were **searched for and not found**. DFCO behavior on this ECU is unmapped.

### 2.9 Final output — pulse width

- Injector latency scalar exists near `@0xD26A0` 🧮.
- **Injector flow scalar: NOT FOUND** (🔬 searched-and-absent). The Gen-1 constants don't exist in any encoding in this ROM; flow is likely hardcoded in code or computed differently — ❓ open thread with tuning implications (can't rescale injectors by a single scalar).
- ❓ OPEN THREAD: the final summation order — how base AFR, warmup multiplier, tip-in compensation, X-tau correction, and CL trims combine (additive? multiplicative? in what order?) is not traced in code.

---

## 3. Timing Chain (partially decoded)

```
        ┌─ Base Timing A/B (13×13) ──────────────┐
        │  Base Timing C/D (7×13)                 │
        │  Base Timing E/F                        │──▶ base spark angle
        │  Base Timing Idle (2D)                  │
        └────────────────────────────────────────┘
                            │
        ┌─ Knock Correction Advance Max A–E ─────┐
        │  × IAM (starts 0.50)                   │──▶ dynamic advance
        │  + feedback correction (≤ −7.0°)       │
        │  + fine learning                       │
        └────────────────────────────────────────┘
                            │
                            ▼
                  final spark angle ──▶ dwell (@0xDD4C4, the only one)
```

### 3.1 Base timing

- `Base Timing A @0xCC7A0` / `B @0xCCA88` (13×13 u16BE, x·0.1 °BTDC) — 🧮 ROM-VERIFIED: raw 37–386 → **3.7–38.6°** (idle 7.9°, cruise 17.3°, WOT 16.3° — physically sane). The `x·0.05` scaling in old docs was wrong; the XML's `x·0.1` is correct. Zero `0.1f`/`0.05f` constants in code — the ECU likely uses native 0.1°/LSB.
- `Base Timing C @0xCD670` / `D @0xCD778` (7×13, x·0.1) — ❓ the X axis `[500, 575, 600, 630, 650, 690, 750]` is an **unidentified quantity** (not load/MAP/RPM; timing rises weakly with it). `E @0xCCED4`, `F @0xCD010` (axis fixes applied 2026-10-18). `Base Timing Idle @0xD4640` (2D float, 16-el).
- ❓ OPEN THREAD: cruise vs non-cruise assignment of the A–F variants; remaining 4 of 8 base tables are **not 13×13 anywhere in ROM** (proven by full-ROM smoothness sweep) — E/F may be two of them, or more tables hide in other dims.

### 3.2 Knock control

- `Knock Correction Advance Max A–E` @0xCC450/0xCC2F6/0xCC638/0xCC6D8/0xCC738 (x·0.01) — 🧮 ROM-VERIFIED A: raw 47–869 → **0.47–8.69°**. Shape is the KC-max signature: ~8° ceiling at light-load/high-RPM where knock-safe, collapsing to ~0.5° at WOT. (The Gen-1 scaling `(x·0.3515625)−20` gives 16–305° here — garbage, proven.)
- Parameter cluster `@0xD4A40–0xD4B50` 🧮 ROM-VERIFIED: RPM ranges @0xD4A5C `[1100, 1200, 6900, 7000]` / @0xD4A94 / @0xD4B30 (`[disable-below, enable-above, enable-below, disable-above]` hysteresis pairs — three ranges ≈ feedback / rough(IAM) / fine); load pairs @0xD4AA4 `[0.48, 0.50]` / @0xD4AAC `[2.10, 2.20]` (+ duplicates @0xD4B40/0xD4B48); retard limit **−7.0°** @0xD4A6C; IAM initial **0.50** @0xD4AB8; step values 0.03125/0.0625 @0xD4AC0/0xD4AD4.
- Knock code 🔬 DISASSEMBLED: **31 functions @0x04B000–0x04C700**. The learned knock table lives in RAM `@0xFFFF8800/8818`, initialized from flash `0xD4AC4–0xD4AD0` via function `0x4BB48` (🧮 init bytes at that address are consistent: 7.0f, 0.0, 0.0, 4.0f…). The 126 refs to `0xFFFF87xx` are **byte-wide `mov.b` to on-chip peripheral registers, not RAM** — leading hypothesis is a dedicated knock DSP unit (❓ open thread, needs hardware docs).
- **Combination formula** 📖 REFERENCE-DERIVED (Subaru-standard, from thinktuning cross-ref — **not traced in this ROM's disassembly**):
  `final advance = (KC advance max × IAM) + feedback correction + fine-learned correction`
  Treat as a working hypothesis, not a verified equation. ❓ The actual summation code is an open thread.
- ❓ OPEN THREAD: 5 knock + 4 base tables remain unlocated; which range (A/B/C) is feedback vs rough vs fine.

### 3.3 Dwell and AVCS

- Exactly **one** Ignition Dwell table `@0xDD4C4` (16×5, RPM × battery volts) — 🔬 proven by all 9 reference defs having exactly one; no dwell-vs-temp, coil-charge, or spark-duration tables.
- AVCS: 4 main cam-target maps @0xC1650–0xC1D28 (14×18/14×11) 🧮; how they feed cam phaser control — ❓ open thread.

---

## 4. Diagnostics Side: SSM (reference, not control flow)

The SSM dispatcher `@0x65380` 🔬 loads handler table base `0x6EFBC`, reads PID from `[GBR+0]`, `jsr @r2` to the handler. Control block `0xFFFF9464` (PID +0x0, param +0x14, flags +0x120). This is the *observation* path (logger), not the control path — documented here so it isn't confused with the chains above. Parameter inventory (our OpenSID equivalent) is an ❓ open thread.

---

## 5. Open Threads (the honest list)

| # | Question | Blocks |
|---|----------|--------|
| 1 | Final fuel summation order (base × warmup × tip-in × X-tau × CL trim — additive? multiplicative? in what order?) | Tuning fuel confidently |
| 2 | Relation between pre-computed 8×8 transient tables and the real-time X-tau filter code (@0x36616) | Understanding tip-in fully |
| 3 | Injector flow scalar — where is it, or how is flow computed? | Injector rescaling |
| 4 | DFCO tables — resume threshold, cut counters, enrichment singles | Decel tuning |
| 5 | Actual knock/timing summation code (is the Subaru-standard formula really what's in ROM?) | Trusting the timing model |
| 6 | DBW servo control loop and A/B/C variant selection | Throttle behavior |
| 7 | VE 42-variant selection logic | Fueling across conditions |
| 8 | Base-timing C/D X-axis quantity [500–750] | Interpreting C/D |
| 9 | Knock DSP unit hypothesis (0xFFFF87xx peripheral registers) | Knock sensor calibration |
| 10 | Cruise/non-cruise variant assignment across all chains | Applying any tune correctly |

---

## 6. What This Page Is Not

- **Not a tuning guide.** Saab's scene writes per-table "how to tune / what goes wrong" only because they can flash and test. We can't — not until the cable arrives and the checksum is found. That guidance comes later.
- **Not complete.** Roughly 58% of tables are high-confidence; the chains above touch perhaps 100 of them. The remaining ~660 tables (switches, scalars, auxiliary maps) feed these chains in ways we haven't traced.
- **Not verified.** Said three times on this page already; saying it again because it matters.

---

*Page created 2026-10-18. Status: first draft. Sources: PROGRESS.md analysis log, DISASSEMBLY_VERIFICATION.md, v0.6 XML (763 tables). Spot-verified against ROM bytes 2026-10-18 (see 🧮 tags).*
