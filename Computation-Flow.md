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

## 1. DBW Chain (Step 1 confirmed, Step 2+ are decoys)

Torque-based throttle control. The ECU never maps pedal→plate directly; it computes a requested torque and converts that to a plate position. **MAJOR UPDATE 2026-10-18:** Step 1 is now code-confirmed, but **Step 2 (torque→throttle) has no code-referenced table lookup** — the Target Throttle and Torque Limit tables are decoys.

```
                     ┌──────────────────────────────────────────────┐
                     │  STEP 1: Driver torque request  🔬 CONFIRMED  │
                     │  Function @0x55E6E                           │
                     │  Requested Torque A/B  @0xFA08C / @0xFA774   │
                     │  20×18 u16BE, x*0.0078125                   │
                     │  X = pedal % [0xFFFF8BA0]  Y = RPM [0xFFFF8AC4]│
                     │  Output → RAM 0xFFFF8C80                    │
                     └──────────────────────┬───────────────────────┘
                                            │ raw torque value 0–~183 🧮
                                            ▼
                     ┌──────────────────────────────────────────────┐
                     │  STEP 2: Torque → throttle plate              │
                     │  ⚠️ NO CODE-REFERENCED TABLE LOOKUP EXISTS    │
                     │  Target Throttle A/B/C: ZERO code refs        │
                     │  Torque Limit A/B: ZERO code refs             │
                     │  14/21 DBW records (67%) unreferenced         │
                     │  Real mechanism: ❓ UNKNOWN (4 hypotheses)     │
                     └──────────────────────┬───────────────────────┘
                                            ▼
                                throttle servo (code untraced ❓)
```

**Details and evidence:**

- **Step 1** 🔬 DISASSEMBLED (@0x55E6E, ~170 insns): pedal % from RAM `0xFFFF8BA0` → fr12, RPM from `0xFFFF8AC4` → fr14. **A/B family selection** via mode byte (r6≠0 → A family @0xFA08C, r6=0 → B family @0xFA774). Each path looks up THREE tables (primary + duplicate + 2×2). Post-lookup: slew integrator on state `[0xFFFF8C84]` (`fmin(+0.04, 100.0)` ramp), then `fmin` clamp via @0xBEAC0. Output → RAM `0xFFFF8C80`. The function never touches torque-limit or target-throttle tables.
- **Step 2 — NEGATIVE FINDING (HIGH confidence):** Requested torque RAM `0xFFFF8C80` has **zero code consumers** (checked 32-bit literal, GBR-relative, and register-indirect addressing). **All Target Throttle tables (A @0xF9240, B @0xFAEF4, C @0xF9800) have zero code references.** **All Torque Limit tables (@0xFB698/@0xFB79C + 2 more) have zero code references.** The v0.7 "Target Throttle Plate Position" labels describe real, structurally-valid data (TT-A shows 0–86° plate angle) that the ECU does not use. **Throttle remap via these tables would be placebo tuning.**
- **Per-gear torque limits (NEW):** the shelved "Timing Comp Per Gear" family (14 tables @0xDB280–0xDC104) was re-examined. Function ~@0x557C0 selects 6 of 14 records (T1, T4, T7, T8, T13, T14) via conditional branches; values reach 195–210 (physically absurd as timing °, consistent with FB20 peak torque ~200 Nm). **Verdict: real, code-used per-gear torque limit tables**, mislabeled as timing. 6 tables for 5MT (5 gears + reverse/neutral). Branch conditions not yet traced to gear signal.
- **Negative finding (confirmed):** there is **no "Requested Torque Base"** table on this ECU. 🔬 SEARCHED-AND-ABSENT.
- **Negative finding (confirmed):** no throttle closing-rate / damping table exists. Rev hang is handled elsewhere (see Fuel chain §2.8). 🔬 SEARCHED-AND-ABSENT.
- **Limiters** 🧮 ROM-VERIFIED where noted: `Rev Limit (Fuel Cut) @0xCEA78 = 6700.0 RPM` (exact float). `Speed Limiting (Fuel Cut) B @0xCE6B0`, `Rev Limit B @0xD401C` also in the Limits category. (The @0xF8E44–0xF8E68 "Speed Limiting Throttle" tables are byte-identical with zero refs — likely dead weight, not progressive closure.)
- ❓ OPEN THREAD: the real torque→throttle mechanism. Hypotheses: (1) algorithmic conversion via inverse airflow model (no table), (2) undiscovered computed addressing, (3) torque copied to another RAM location, (4) throttle follows filtered pedal directly. The throttle servo driver itself was not found (sink hunt).

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

- `MAF Sensor Scaling @0xDCEA0` (48-el f32, Y = volts @0xDCDE0 0.78–4.45V, data 0.96–290 g/s) — 🔬 DISASSEMBLED as the live table. **Full path traced:** ADC @0xFFFF4042 (uint16 counts, sole reader is fn @0x4938) → ×(5/65536) → volts → 1D directory @0xB71D0 → main dispatcher @0xBE964 → g/s @0xFFFF40B4 (sole writer) → temp-comp fn @0x24072 (table @0xB1BA8, −40…110°C, 1.00 @ 50°C, 2.56× @ −40°C) → 2D temp×MAF fuel table @0xB1E58 (fn @0x28F7C) → **load @0xFFFF66E4** (60 code refs) → fuel/timing/AVCS. A 65-el impostor @0xD5EEC was ruled out (zero code refs).
- `MAP Sensor Scaling @0xDCCD8` — f32BE `[250.0, −150.0]` 🧮 (units TBD, likely kPa-ish offsets).
- **Barometric: none live** — Baro Comp A/B confirmed dead (zero refs). 🔬

### 2.2 Cylinder fill — VE (LIVE via pointer tables, scaling unresolved)

- 42 tables `@0xC4E70` (12×11) + 42 High-RPM `@0xC6418` (12×18), all byte-unique (MD5). 🔬
- **VE is LIVE — the "dead" verdict was wrong.** Code @0x28D5A references pointer table @0x6D89C (84 entries) → dispatcher records → VE tables. Literal xref scans miss pointer-table indirection; this was the systematic error. **Lesson: a table is only "dead" after checking all four layers** (table addr → record addr → pointer tables → code).
- **Selection:** index 0–41 derived from cam positions (GBR+0x142/0x144/0x146); 42 = intake×exhaust cam position combos (dual AVCS). Function does 5 STD + 5 HI lookups per invocation (purpose unresolved).
- ❓ OPEN THREADS: STD↔HI switchover condition (600–4000 RPM overlap unexplained); 5+5 lookup purpose; VE output consumer (NOT in fuel hub @0x43F64 path — VE's role in fueling is unclear).
- **Scaling UNRESOLVED:** v0.7 `x·0.00305175781` → 37–73% (low for NA); record `x·0.0078125` → 94–187% (impossible). Neither is physically right. Do not tune VE until resolved via hardware.

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
- ~~`Throttle Tip-in Enrichment A @0xD13D4` and `B @0xD1474`~~ — **DEMOTED: zero code references.** These are structurally valid (16×8, ECT × time) but the ECU does not call them. Do not tune. 🔬
- **Active tip-in** uses the Compensation family: `Tip-in Enrichment Compensation A @0xD2014` / `C @0xD217C` / `J @0xD1F60` (16×8, X=ECT @0xCEB64, Y=time-since-tip-in, scale 1/256, unit % enrichment), selected by RAM `[0xFFFF9609]` + GBR+87. Trigger fn @0x34430, compensation @0x344BE. Tip-in threshold lives in RAM `0xFFFF6988` (cable-only, not a table).
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

### 2.9 Final output — pulse width (summation order RESOLVED)

**Fuel hub @0x43F64** 🔬 DISASSEMBLED (255 insns) — the final fuel computation. Summation order now known explicitly (open thread #1 resolved):

```
1. base   = OL_Fueling(cruise ? @0xD487C : @0xD46E8, load, RPM)
2. corr   = Correction1x1(cruise ? @0xD496C : @0xD4960, load, RPM)
3. blend  = clamp(Table5(@0xD4120), 0.0, 1.0)     [via BE6A0]
4. mult   = Table6(@0xD40E8)
5. fuel   = float(corr) × blend
6. IF (mode): fuel += (1.0 − blend) × float(base)  [fmac]
7. fuel  *= mult
8. IF (mode): fuel *= float(@GBR+0)
9. fuel  -= float(@GBR+12)
10. fuel  = clamp(fuel, 0.0, float(word @0xD3FC0)) [via BE6A0]
11. out   = truncate(fuel) → RAM 0xFFFF83A6 (+ rate limiting)
```

- **Cruise/non-cruise selection:** r5=1 → cruise tables, r5=0 → non-cruise. Derives from byte @0xFFFF9460 (cruise-active flag).
- **Tip-in feeds fuel directly:** the hub reads tip-in RAM state (fr13/fr14) — tighter coupling than table structure suggests.
- **Injector latency** 🔬 DISASSEMBLED (@0x35A96): looks up record 0xB5048 by battery voltage [0xFFFF4128], scales raw u16 ×0.25 → float µs → RAM 0xFFFF78DC. **v0.7's "Injector Latency" (5-el, x*0.00025 ms) is VALIDATED** — the record is nominally 5×2 but both Y-rows are byte-identical (degenerate). Values: 3.41 ms @ 6.5V → 0.51 ms @ 16.5V (textbook dead-time).
- **Injector flow scalar: NOT FOUND** (🔬 searched-and-absent). Gen-1's `2707090/x` constant searched as float32BE/LE + int variants — zero hits in ROM. Flow is hardcoded in code or computed differently.
- **Output drivers:** Injectors = ATU-II Channel 2 (GR2A-D @0xFFFFF604–60A, OCR2A-D @0xFFFFF614–61A), config table @0x110F4. **Writer function NOT found.** Ignition = ATU-II (channel unidentified). **Memory-map correction:** peripherals are at 0xFFFFFxxx (ATU-II), NOT 0xFFFF8000+ (that's RAM). The 0xFFFF87xx "knock peripheral" attributions need re-examination.

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

- `Base Timing A @0xCC7A0` / `B @0xCCA88` (13×13 u16BE, x·0.1 °BTDC) — 🧮 ROM-VERIFIED: raw 37–386 → **3.7–38.6°** (idle 7.9°, cruise 17.3°, WOT 16.3° — physically sane). **⚠️ Base Timing A is PHANTOM** — byte-identical to KCA Max E rows 4–16 (see §3.2). Pending removal from v0.7.
- `Base Timing C @0xCD670` / `D @0xCD778` (7×13, x·0.1) — ❓ the X axis `[500, 575, 600, 630, 650, 690, 750]` is an **unidentified quantity** (not load/MAP/RPM; timing rises weakly with it).
- **Timing blend** 🔬 DISASSEMBLED (real entry @0x81E28, blend core @0x81F00): `result = w·E + (1−w)·F` where `w = clamp((input − T1)/(T2 − T1), 0, 1)` from RPM-indexed tables. **Mode selection:** blend vs single-table via RAM [0xFFFF6960] against 250.0/249.0 ROM thresholds (hysteresis band). Output → RAM 0xFFFFA238. **v0.7 correction:** code uses X axis [500..755] for Base Timing F/E (@0xCCFB4/@0xCCE78), Y = RPM [700..6400].
- `Base Timing Idle @0xD4640` (2D float, 16-el).
- ❓ OPEN THREAD: cruise vs non-cruise assignment of the A–F variants.

### 3.2 Knock control

**⚠️ MAJOR CORRECTION: the "Knock Correction Advance Max A–E" tables are FAKE.** All five (@0xCC450/0xCC2F6/0xCC638/0xCC6D8/0xCC738) have **zero code references, zero dispatcher records, and physically impossible values (30–80°)**. This ECU does NOT use the traditional Subaru KCA×IAM architecture. Do not tune these as knock tables. 🔬

**Downstream of the timing blend** 🔬 DISASSEMBLED:
- **@0x81B50** — threshold comparator: reads blended timing [0xFFFFA238], adds offset, sets/clears flag @0xFFFFA620.
- **@0x81F80** — safety gate + limiter: multi-condition check, then **clamps final timing to [−6.67°, +21.11°]** (ROM floats @0x82188/0x8218C). This is the last verified stage before ignition output.
- **Final summation point (base + knock → spark): NOT FOUND.** ❓

**Real knock system — three-mode architecture** 🔬 DISASSEMBLED (31 functions @0x04B000–0x04C700), 25 verified parameter tables:
- **Feedback** [1100–7000 RPM]: immediate retard on knock (5.0° coarse / 0.35° fine), recovery +0.10° steps.
- **Fine** [1500–3600 RPM] × load: learned per-cell, RAM table @0xFFFF8800/8818, limits +8.0°/−7.0° (extended −10.0°).
- **Rough / IAM** [1300–4400 RPM]: global multiplier, **initial 0.50**, step 0.25, RAM @0xFFFF30F0.
- ECT gates: 75°C high / −20°C low. Retard limit −7.0° std / −10.0° ext.
- **Knock sensor = hardware comparator** (signal @0xFFFF87E4, control @0xFFFF87BC) — threshold comparison in hardware, software reads digital result. No threshold table in ROM.
- ❓ OPEN: what IAM multiplies (no KCA table exists); the final summation code.

### 3.3 Dwell and AVCS

- Exactly **one** Ignition Dwell table `@0xDD4C4` (16×5, RPM × battery volts) — 🔬 proven by all 9 reference defs having exactly one; no dwell-vs-temp, coil-charge, or spark-duration tables.
- **AVCS** 🔬 DISASSEMBLED: 4 target tables @0xC1650–0xC1D28 (intake/exhaust × cruise/non-cruise), selected by two-stage mux @0x16002. Position error computed @0x163FC (target − actual). **Duty = P-only controller:** `duty = clamp((error/1.6) + 0.5, 0, 255)` via @0xBE6DC — no integral, no derivative. **Scaling RESOLVED:** v0.7's 360/65535 is correct (triple-sourced: our ROM physics + fa20guy's FA20 + 2005–06 Legacy SH7055 defs). The 12.6° max is a calibration choice; FSM's 55–68° is the phaser mechanical limit. Control loop operates in raw units; degree scaling is display-only. AVCS Warmup Correction @0xC1F18 = 0.11–1.0 multiplier (x/32768).

---

## 4. Auxiliary Control Loops

### 4.1 Idle speed target selector

🔬 DISASSEMBLED (@0x575BA): an 8-way mode switch. Mode byte (1–8) at RAM `0xFFFF8E7C` selects among 9 Idle Speed Target tables via pointer arrays (@0x57768/0x5778C) — **not direct record xrefs** (another pointer-indirection case). Mapping: Mode 2 → K (1800 RPM cold), Mode 3 → M (650 RPM warm). **ECT switch at ~50°C** (table @0xD9250; hardware log confirms: 1823 RPM @ 15°C, 657 RPM @ 90°C). The mode-byte writer (ECT→mode state machine) was not found.

### 4.2 Radiator fan control

🔬 DISASSEMBLED (@0x34760–0x347F6): six hysteresis latches driving fan request flags. **Thresholds @0xCE4B0 = [83.3, 89.3, 90.0, 100.0]°C** → low ON 90.0 / OFF 83.3, high ON 100.0 / OFF 89.3. Companion tables: @0xCE4C0 [−55, −50] (extreme-cold, purpose unknown), @0xCE4D0 [5, 10] (likely VSS cutout). The ECU implements fan control directly (not a separate module).

### 4.3 Rev limiter

🔬 DISASSEMBLED (@0x408C4): **fuel cut** (not spark), two-channel RPM watchdog with hysteresis. **Cut 6700 / resume 6600** (100 RPM hysteresis); second channel 4000/3800 under unidentified conditions (mode byte @0xFFFF6116 = 5/7). Table @0xCEA78 = [6700, 6600, 3800, 4000]. **10 limiter flags ORed @0x3F048 → master cut flag @0xFFFF80FC** (any active = cut). Countdown timer @0x3E27C present but disabled (reload = 0). Related: Rev Limit A @0xCEAD0 [5600/5500/4000/3500], Rev Limit B @0xD401C (80/75 = ECT gate), Speed Limit @0xCEA10/14 (200/190 kph).

---

## 5. Diagnostics Side: SSM (reference, not control flow)

The SSM dispatcher `@0x65380` 🔬 loads handler table base `0x6EFBC`, reads PID from `[GBR+0]`, `jsr @r2` to the handler. Control block `0xFFFF9464` (PID +0x0, param +0x14, flags +0x120). This is the *observation* path (logger), not the control path — documented here so it isn't confused with the chains above. Parameter inventory (our OpenSID equivalent) is an ❓ open thread.

---

## 6. Open Threads (the honest list)

| # | Question | Blocks | Status |
|---|----------|--------|--------|
| 1 | ~~Final fuel summation order~~ | — | 🟢 **RESOLVED** — fuel hub @0x43F64 traced 2026-10-18 (see §2.9) |
| 2 | Relation between pre-computed 8×8 transient tables and the real-time X-tau filter code (@0x36616) | Understanding tip-in fully | Open |
| 3 | Injector flow scalar — where is it, or how is flow computed? (Gen-1 constant not in ROM) | Injector rescaling | Open |
| 4 | DFCO tables — resume threshold, cut counters, enrichment singles | Decel tuning | Open |
| 5 | Actual knock/timing summation code (final combine before ignition) | Trusting the timing model | Open |
| 6 | Real torque→throttle mechanism (Target Throttle tables are decoys); throttle servo driver | Throttle behavior | Open — **updated**: Step 2 has no table lookup |
| 7 | VE STD↔HI switchover; 5+5 lookup purpose; VE output consumer | Fueling across conditions | Partial — cam-position index found, switchover open |
| 8 | Base-timing C/D X-axis quantity [500–750] | Interpreting C/D | Open |
| 9 | 0xFFFF87xx re-examination (RAM space, not peripherals per memory-map correction) | Knock sensor calibration | Open — **updated** |
| 10 | Cruise/non-cruise variant assignment across all chains | Applying any tune correctly | Partial — fuel hub selection traced |
| 11 | Per-gear torque limit branch conditions (gear signal source) | Using the 6 torque tables | **New** |
| 12 | Idle mode-byte writer (ECT→mode state machine @0xFFFF8E7C) | Idle behavior | **New** |
| 13 | AVCS exhaust error loop; actual cam-position sensor path | AVCS tuning | **New** |

---

## 7. What This Page Is Not

- **Not a tuning guide.** Saab's scene writes per-table "how to tune / what goes wrong" only because they can flash and test. We can't — not until the cable arrives and the checksum is found. That guidance comes later.
- **Not complete.** Roughly 58% of tables are high-confidence; the chains above touch perhaps 100 of them. The remaining ~660 tables (switches, scalars, auxiliary maps) feed these chains in ways we haven't traced.
- **Not verified.** Said three times on this page already; saying it again because it matters.

---

*Page created 2026-10-18. Status: second draft — Oct 18 tuning-path campaigns integrated (fuel hub, MAF path, timing blend, knock, DBW Step 1/2, VE liveness, idle/fan/limiter/AVCS). Sources: PROGRESS.md analysis log, CAMPAIGN_*.md, DISASM_*.md, v0.7 XML. Spot-verified against ROM bytes 2026-10-18 (see 🧮 tags).*
