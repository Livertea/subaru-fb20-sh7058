# Table Reference

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


Tables mapped in the EP5GD00A definition, organized by category. Counts are from v0.7 (full merged revision).

> **Status:** All tables are structurally validated against ROM bytes. Labels and identities range from HIGH to MEDIUM confidence. None are hardware-validated yet.

**Inventory breakdown:** 260 3D · 333 2D · 34 1D · 236 Switch = **863 tables**

*Counted directly from `RR_EP5GD00A_0.7.xml`: 236 tables typed Switch plus 7 DTC switches typed 2D = 243 functional switches.*

> **⚠️ Known issue:** Address 0xD487C has two tables claiming it ("Primary Open Loop Fueling Cruise" 10×16 and "Idle Airflow Decel Adder B" 11×10, same X axis). One is mislabeled. Also 0xD2798 ("Injector Correction C" and "Idle Speed Target G"). Found during automated inventory validation. Do not rely on these addresses until resolved.

---

## Fueling (~150 tables)

The largest category. The FB20 is speed-density (not MAF-based like the WRX turbo), so it has extensive volumetric efficiency tables.

### Primary Fueling
| Table | Address | Dims | Notes |
|-------|---------|------|-------|
| Primary Open Loop Fueling Non-Cruise | 0xD46E8 | 10×16 | Main fuel map, AFR scaling |
| Primary Open Loop Fueling Cruise | 0xD487C | 10×16 | Cruise variant |

### Volumetric Efficiency (84 tables)
- **VE A–AP** (42 tables) @ 0xC4E70 + i·0x84, 6×11
- **VE (High RPM) A–AP** (42 tables) @ 0xC6418 + i·0xD8, 6×18
- Scaling `x·0.00305175781` — matches FA20 to 9 significant digits
- This is **cylinder fill**, not "Engine Load Compensation" as initially thought

### Closed-Loop Compensation
- CL Fueling Target Compensation A–D (Load) @ 0xD34A6+
  - **Correction (v0.7):** Comp C was off by 2 bytes — data starts at **0xD3948**, not 0xD3946 (was reading the Y-axis tail as data)
- CL Fueling Target Compensation (ECT) @ 0xC3FB0 — 16-el, AFR offset

### Transient Fueling (X-tau System)
12-table system for transient fuel correction:
- T0–T11 transient tables @ 0xD2864–0xD3150
- Fuel Film Wall-Wetting X (in def)
- Fuel Film Tau @ 0xD329C (10×9)
- **Trigger:** Subroutine @0x365DC (found via disassembly)

### Tip-in Enrichment
- Tip-in Enrichment Compensation (RPM) @ 0xCFAC8
- Tip-in Enrichment Compensation (ECT) A–I, Warm @ 0xD2014+ (10 tables in 0.3)
- Tip-in Enrichment Compensation (ECT) J @ 0xD1F60, Warm B @ 0xD25B0 (0.3.1b)
- **Tip-in Enrichment A @0xD13D4 / B @0xD1474 — dims RESOLVED (v0.7):** 16×8, X = ECT @0xCEB64 (16-el), Y = time-since-tip-in @0xD13B4 (T1) / @0xD1454 (T2) (8-el, 0.25–1.125 s), scale 1/256, unit **% enrichment** (not ms). The old 8×8/8×6 readings with swapped axes were wrong; the corrected directory alignment is authoritative.

### Warmup / Cold Start
- ~28 Warmup Compensation (ECT) tables @ 0xD8ED6+ (0.3)
- 5 Warmup Compensation Step tables @ 0xD9526+
- Cranking Fuel tables (8)
- Post-Start Enrichment @ 0xC07C4

### Injectors
- Injector Latency @ 0xD26A0 (2D, 5-el)
- Injector Correction A–D @ 0xD26C0+ (16×3, ECT × Voltage)

### Fuel Multipliers
- Fuel Multiplier A (3D) @ 0xC4AC0 — 13×15
- Fuel Multiplier B (3D) @ 0xC4C10 — 13×22

---

## Timing (~30 tables)

### Base Timing (6 maps)
| Table | Address | Notes |
|-------|---------|-------|
| Base Timing A, C–F | 0xCCxxx range | Primary ignition maps |
| Base Timing B | 0xCCA3A | **13×17** (corrected in v0.7 — the old 13×13 was a window into this table; rows 0–2 have anomalous values, unexplained) |
| Timing Maps G–J | 0xD72C4+ | 13×17 uint8, lower values (3–18°) — possibly correction/limit tables |

- Scaling: `x * 0.1` → degrees
- 6 base maps vs WRX's 4

### Timing Compensation
- Timing Compensation (ECT) A–D
- Timing Compensation (IAT)
- Validated: stepped (not smooth) is normal for ECT compensation

---

## Knock Control (~40 tables)

### Correction Tables
- Knock Correction Advance Max A–E @ 0xCCxxx
- Fine Correction Learning RPM Rows @ 0xD4AD8 — 7-el [1300–4400] RPM
- Fine Correction Learning Load Columns @ 0xD4AF8 — 7-el [0.68–1.9] g/rev

### Parameters (Scalars)
- IAM Initial @ 0xD4AB8 = 0.50
- IAM Step @ 0xD4ABC = 0.25
- Fine Correction Advance/Retard values and limits
- Feedback Correction parameters (Cluster A)
- Advance Step parameters (Cluster B)

### Three-Mode Architecture (confirmed via disassembly)
1. **Feedback Correction** — Immediate, per-event. Range A [1100–7000] RPM.
2. **Fine Correction** — Learned, stored in RAM (0xFFFF8800). Range B [1500–3600] RPM.
3. **Rough Correction / IAM** — Global multiplier. Range C [1300–4400] RPM.

See [Ghidra-Guide](Ghidra-Guide.md) for the function map.

---

## Idle Control (~20 tables)

- **Idle Speed Targets A–N** (14 tables) — ECT-indexed, various conditions
- Idle Airflow Decel Adder A @ 0xD4700 (16×9) — **this is the rev hang table**
- Idle Airflow Decel Adder B @ 0xD487C
- Idle Error Correction Multipliers @ 0xDF60C+

---

## AVCS – Variable Valve Timing (~10 tables)

- AVCS Intake/Exhaust main tables
- AVCS Warmup Correction @ 0xC1F18
- Validated: 0.389–1.000 multiplier (cold → warm)

---

## Throttle / Drive-by-Wire (~10 tables)

- Target Throttle Plate Position A @ **0xF923C** (50×12 — corrected in v0.7, was off by 4 bytes; X axis @0xF9144), B @ 0xFAEF4 (50×19), C @ 0xF9800 (50×19)
- Requested Torque A @ 0xFA08C, B @ 0xFA774
- DBW chain: pedal → Requested Torque → Target Throttle

---

## Sensors (~15 tables)

- MAF Sensor Scaling @ 0xDCEA0
- MAP Sensor Scaling @ 0xDCCD8 — float32 [250.0, -150.0]
- Coolant Temp Sensor scaling
- Intake Air Temp Sensor scaling
- Fuel Temp Sensor scaling @ 0xDD198

---

## Limits (~15 tables)

- Rev Limit (Fuel Cut)
- Rev Limit Fuel Resume (MP) @ 0xC9848
- Speed Limiting (Fuel Cut) A/B
- Speed Limiting (Throttle) A–D @ 0xF8E44+
- MAF Sensor Limit (Voltage) @ 0xD5EEC

---

## DTC Switches (243 tables)

Every diagnostic trouble code has an enable/disable switch. All 243 mapped with P-code labels (236 typed Switch + 7 DTC switches typed 2D).

See [DTC-Mapping](DTC-Mapping.md) for the full story.

**Notable:**
- **P0420** = Switch 57 (catalyst efficiency)
- **P0455** = Switch 214 (EVAP large leak — the car's current code)

---

## Gear Determination (~5 tables)

- Gear Determination Thresholds @ 0xC4364 — 5-el, scaling `96560.6/x` → rev/km
- The ECU calculates gear from vehicle speed vs RPM ratio

---

## Thermal Management (~10 tables)

- Temperature multiplier cluster @ 0xCBABC–0xCBCD4
- Binary switch patterns (0.8× / 1.5×)
- Unknown Multiplier (Load×RPM) @ 0xCB974

---

## CL/OL Transition (~10 tables)

- CL to OL Transition (Base Pulse Width) @ 0xC4210
- CL Delay Maximum (Vehicle Speed) @ 0xC4380
- CL Delay Maximum Engine Speed (Neutral) @ 0xC43BC
- CL Delay (ECT) A–C

---

## Unknowns — Misc Table Hypotheses

318 tables sit in the `Misc` category — real directory entries with dispatcher call sites, but unidentified. In Oct 2026 every one was decoded from the ROM and assessed on value shape, axis identity, address neighborhood, and call-site evidence. Full analysis: `hidden_files/analysis/MISC_MUSINGS_2026-10-18.md`.

**This is speculation, not identification.** Nothing below was merged or renamed in the XML. Confidence tags: HIGH / MEDIUM / LOW. Treat all of it as "leads for logger/disassembly work," never as tuning guidance.

### Subsystem hypotheses (best leads first)

- **Feedback-controller error maps (MEDIUM).** C2CF4/C2DD0/C2EAC + C3388/C3408/C3488 + C3144/C3204/C32C4 — 9 tables, one code region (0x1CD00–0x1D500). Symmetric saturating error→correction curves (±100→±250, ±8→±0.16, 1.0-neutral deviation multipliers). Reads like a P/I/error lookup set for **idle RPM PI control or closed-loop O2 fuel feedback** — a subsystem with zero current coverage. Confirm: disassemble 0x1CD00–0x1D500; logger RPM error vs idle airflow correction.
- **Delta-T bipolar timing correction ×4 (MEDIUM-HIGH on shape).** DF934/DFA34/DFB34/DFC34 — byte-identical 9×10, ΔT=[-40..40]°C × RPM [600..5600], values ±11.5° passing exactly through 0 at Δ=0. Likely ignition-timing compensation vs temperature delta; 4 copies suggest 4 timing paths. Confirm: compare against WRX "Ignition Timing Comp (IAT)" shape.
- **Radiator fan thresholds (MEDIUM) — first ECM-side fan evidence.** CE4B0 = [83.3, 89.3, 90.0, 100.0]°C with hysteresis evaluation @0x34760: reads as a 2-speed fan set (low on 90 / off 83.3, high on 100 / off 89.3). This overturns the old "no fan tables in ECM" finding — these are *thresholds*, not tables; the ECM may only request while another module drives. Confirm: logger ECT vs fan state.
- **RPM×Load multiplier family, 5 variants (MEDIUM — with a scale caveat).** D67AC/D6864/D691C/D69D4/D6A8C (14×7 ×5) + D80BC. Neutral ≈1.0 with structured 0.4×–1.6× corrections, 5 sequential call sites. **The banked temperature scale is definitively wrong** (displays 11,520 "°C"; raw values are clean 0xFFFF fractions — true scale is fractional). Real family, wrong display; scale fix pending.
- **Sparse regional corrections (MEDIUM-LOW).** CD2AA/CD464 (± pair, 13×17) + CC9CA + CCC40/CCDE8 — mostly 0x7FF8 padding with small live islands in different RPM/load regions, one code region (0x82600–0x82A00). The padding *proves* selective operation. Candidates: idle/off-idle trims, knock-region corrections.
- Also: warmup-domain tables (DA9F0/DAA5C 16×3 ECT fuel-enrichment-shaped, C3FE8/DE798/CFC18 decay multipliers), thermal cluster (CBABC/CBB9C mode-gated 0.8/1.5 switch at ECT 40°C; CB974 load×RPM derate — EGR/purge duty candidate), Mode Lookup 4×6 A/B/D/E (duty-like values 13–250 keyed on mode indices 1–4 × 1–6; no C table found), X-tau-adjacent triplets (D2964/D2A24/D2AE4, call sites right after the X-tau filter), cold-start fuel candidates (CE0EC/CE1C4 float pair, bigger when colder), DB0A4 (purge-ramp-shaped if its Y axis is seconds — unconfirmed).

### Systematic caveats (affect many tables)

- **~15 tables show alternating-zero checkerboard data** (D1734, D1930, D1BA0/D1C14, D1CF0/D1D60/D1E44, D7D68 family, D8034, D0FEC, D1064/D1150, D1604, D1328) — probable **dimension misidentification**. Treat all values as suspect until callers are disassembled.
- **Fraction-of-0xFFFF values with wrong banked scales** (D67AC family, D1064/D1150, DF058/DF364) — XML currently displays nonsense; scales need correcting, not the data.
- **C4564 is two interleaved 25-el series**, not one 50-el table.
- **2×2 tables: real, not noise.** Only 8 exist in v0.7 (not 21); every one has a directory entry and ≥1 dispatcher call site.
- **Likely dead:** CF706, CF746, CF7C6, CF7E6 (flat 0.0032/0.0039 ×16), D0AF0 (flat 0.21875 ×16), D5B7C (flat 0.20 ×16) — constant values, no recorded call-site evidence. *Unconfirmed* (early letters didn't record refs) — verify xrefs before removing anything.

### Defeat-device screen — CLEAN

The Misc tables were explicitly screened for emissions-cheat patterns, given the invitation to think maliciously. Verdict: **nothing found.** No timer-gated switches (the 900–2200-max axes are all RPM axes); no narrow 20–30°C test-cell gating; no baro-gated switches; mode-selector tables show zero "test mode" outliers; zero Misc tables anywhere near the DTC descriptor region. The single genuine time-since-start axis in the definition (Post-Start Enrichment, 0–1400s) reads exactly 0.0 across the FTP-75 1180s boundary — textbook innocent. Combined with the near-zero cheat surface of a stoichiometric port-injected engine and enabled catalyst monitors, the honest read is: there is nothing here to find, and we looked where it would be.

---

## New in v0.7 — Hypotheses Under Investigation

The re-triage added 101 tables (87 uint16 + 14 float32 — the float tables use the secondary dispatcher's type-0 passthrough). A structured speculation pass over the unknown tables produced these subsystem hypotheses (see `MISC_MUSINGS_2026-10-18.md` in the analysis folder). All are **unverified guesses**, ranked by evidence:

- **Feedback-controller error maps** (MEDIUM) — 9 tables (0xC2CF4 family): symmetric saturating error→correction curves, all called from one code region (0x1CD00–0x1D500). Possibly the idle or closed-loop-fuel feedback controller's P/I lookup set — a subsystem with no current coverage.
- **Delta-temperature timing corrections** (MEDIUM-HIGH) — DF934/DFA34/DFB34/DFC34: byte-identical 9×10s, values ±11.5° passing exactly through 0 at Δ=0. Likely ignition-timing compensation vs temperature delta; 4 copies = 4 timing paths.
- **Radiator fan thresholds** (MEDIUM) — @0xCE4B0: [83.3, 89.3, 90.0, 100.0]°C with hysteresis shape. First ECM-side fan evidence (thresholds, not tables — which is why the table hunt missed them).
- **RPM×Load multiplier family** (MEDIUM) — D67AC/D6864/D691C/D69D4/D6A8C (14×7 ×5) + D80BC: values are clean **fractions of 0xFFFF** (0x3333=0.2, 0x6666=0.4, 0x9999=0.6, neutral 1.0) — the banked temperature scale was wrong and is being corrected. Structured 0.4×–1.6× cells, 5 sequential call sites.
- **Sparse regional corrections** (MEDIUM-LOW) — CD2AA/CD464 (± pair) + CC9CA + CCC40/CCDE8: each active in a different RPM/load island (0x7FF8 padding proves selective operation).

**Known data-quality issues in v0.7** (under review, not yet fixed): ~15 tables show alternating-zero checkerboard data (probable dimension misidentification — values suspect until callers are disassembled); C4564 is two interleaved 25-el series, not one 50-el table.

## Download

See [Definitions](Definitions.md) for the XML files containing all tables.
