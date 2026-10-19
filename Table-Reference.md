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

> **⚠️ Known issue:** Address 0xD2798 has two tables claiming it ("Injector Correction C" and "Idle Speed Target G"). Found during automated inventory validation. Do not rely on this address until resolved. (The old 0xD487C collision note is resolved — Idle Airflow Decel Adder B is confirmed at 0xD9758, not 0xD487C.)

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
  - **⚠️ v0.7 scaling is wrong:** Comp C's AFR formula `(x*0.000224304213)-7.35` produces impossible values (0.00 AFR at x=32768). The data (20972–32670) reads as a **correction multiplier** via x/32768 → 0.64–1.00. Same issue likely affects the ECT variant. Do not tune with the AFR scaling.
  - Only Comp C has a dispatcher record — A/B/D have zero refs (may be dead or pointer-table-accessed; unresolved)
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
- **Post-Start Enrichment @0xC07C4** (11×29, X=ECT −30..20°C, Y=time 0..1400s, scale ÷1000) — EMU-VALIDATED (bilinear exact)
  - **Hardware finding:** the car over-fuels on warmup — datalog shows STFT −8..−23% during warmup, settling to +0.7% hot. The enrichment table is the prime suspect for the first real tune (trim cold region −30–40%, target STFT −5..−10%).
- Post-Start Decay Multiplier @0xCF0F0 — unresolved (non-standard struct, axis pointers don't match v0.7)

### Injectors
- Injector Latency @ 0xD26A0 (2D, 5-el) — EMU-VALIDATED (5/5 voltages bit-exact through the real dispatcher). **Note:** the record is 5×2, not 5×1 — Y=[−600, 0], both rows byte-identical.
- Injector Correction A–D @ 0xD26C0+ (16×3, ECT × Voltage)

### Fuel Multipliers
- **⚠️ Fuel Multiplier A (3D) @0xC4AC0 — v0.7 dims are wrong.** v0.7 says 13×22; the dispatcher record @0xB1E74 says **13×15** (X=load 150–750, Y=RPM 600–4000). v0.7's Y axis has 7 phantom rows (4400–6600) the code never reads. Do not tune the phantom rows.
- **⚠️ Fuel Multiplier B @0xC4C10 — suspected phantom.** Its data address sits *inside* Fuel Multiplier A's data region (row 12, col 12). Do not tune. Needs code-reference investigation.

---

## Timing (~30 tables)

### Base Timing (6 maps)
| Table | Address | Notes |
|-------|---------|-------|
| Base Timing A | 0xCC7A0 | **🚫 PHANTOM** — byte-identical to the fake KCA E rows; not a real table. Flagged for removal from v0.7. |
| Base Timing C–F | 0xCCxxx range | Primary ignition maps |
| Base Timing B | 0xCCA3A | **13×17** (corrected in v0.7 — the old 13×13 was a window into this table; rows 0–2 have anomalous values, unexplained) |
| Timing Maps G–J | 0xD72C4+ | 13×17 uint8, lower values (3–18°) — possibly correction/limit tables |

- Scaling: `x * 0.1` → degrees (idle point + FSM 14–16° window corroborate; highest-risk unverified item in v0.7)
- Base Timing E/F — EMU-VALIDATED (Oct 2026 sweep)

### Timing Compensation
- Timing Compensation (ECT) A–D
- Timing Compensation (IAT)
- Validated: stepped (not smooth) is normal for ECT compensation

---

## Knock Control (~25 parameter tables + ranges)

> **🚫 The "Knock Correction Advance Max A–E" tables in v0.7 are FAKE.** Five tables with zero code references, zero dispatcher records, and impossible values (30–80° of advance). This ECU does **not** use the KCA×IAM architecture. Do not tune them — they don't exist as far as the firmware is concerned. Flagged for removal from v0.7.

### Real Architecture: Three-Mode System (confirmed via disassembly)

The actual knock system uses **25 parameter tables** (direct-read constants, no dispatcher records — the values are used as-is, no interpolation):

1. **Feedback Correction** — Immediate, per-event. RPM range [1100–7000].
   - FB Retard Value @0xD4A70 = 5.0° (coarse), FB Retard Value B @0xD4A78 = 0.35° (fine)
   - FB Advance @0xD4A8C = 0.10, FB Neg Advance @0xD4A88 = −0.10, FB Step Size @0xD4A90 = 0.0312
   - ECT gates: High @0xD4A80 = 75°C, Low @0xD4A84 = −20°C
2. **Fine Correction** — Learned, stored in RAM. RPM range [1500–3600].
   - Fine Correction Learning RPM Rows @0xD4AD8 — 7-el [1300–4400] RPM
   - Fine Correction Learning Load Columns @0xD4AF8 — 7-el [0.68–1.9] g/rev
   - Knock Advance Step @0xD4AC0 = 0.0312, Step 2 @0xD4AD4 = 0.0625
   - Knock Retard Limit @0xD4A6C = −7.0° (standard), Extended Retard A/B @0xD4B24/0xD4B28 = −5.0°/−10.0°
3. **Rough Correction / IAM** — Global multiplier. RPM range [1300–4400].
   - IAM Initial @0xD4AB8 = 0.50, IAM Step @0xD4ABC = 0.25

**Mode hysteresis (new finding, Oct 2026):** Min/Max Load tables are 2-element [threshold, threshold+hysteresis] pairs — Min Load A @0xD4AA4 = [0.48, 0.50], Max Load A @0xD4AAC = [2.10, 2.20]. The mode activates at the first value and deactivates past the second. (Note: v0.7's campaign notes said 0.45 for Min Load A — the ROM reads 0.48, which is authoritative.)

**Knock threshold is a hardware comparator**, not a ROM table — the knock signal and control registers live in peripheral space. There is no tunable knock-sensitivity table.

All 25 tables ROM-validated (Oct 2026). See [Ghidra-Guide](Ghidra-Guide.md) for the function map.

---

## Idle Control (~20 tables)

- **Idle Speed Targets A–N** (14 tables) — ECT-indexed, various conditions. All 14 EMU-VALIDATED via the 1D dispatcher (Oct 2026 sweep).
- **Idle Airflow Decel Adder A @0xD9720 + B @0xD9758 — these are the rev-hang tables.** 9-element 1D curves (Vehicle Speed axis), EMU-VALIDATED. The adder exists to smooth the clutch-in transition on decel — reducing it deletes rev hang, but zeroing it outright can cause decel stalling or harsh engine braking. Reduce, don't zero, on the first pass.
- Idle Error Correction Multipliers A–C @0xDF60C/@0xDF6F8/@0xDF7E4 (9×9) — EMU-VALIDATED. Note: v0.7's axes for DF60C were wrong (listed DF6B0/DF6D4); correct axes are DF5C4/DF5E8 per the dispatcher record.
- Idle Speed Stability A @0xDA52C + B @0xDA6F8 (16×11) — EMU-VALIDATED. (Duplicate axis blocks in ROM: v0.7 lists shared axes for B, the record uses different-but-byte-identical addresses. Functionally equivalent.)
- Base Timing Idle @0xD4640 — EMU-VALIDATED.

---

## AVCS – Variable Valve Timing (~10 tables)

- AVCS Intake/Exhaust main tables
- AVCS Warmup Correction @ 0xC1F18
- Validated: 0.389–1.000 multiplier (cold → warm)

---

## Throttle / Drive-by-Wire (~10 tables)

> **🚫 Most DBW tables in v0.7 are decoys.** 14 of 21 DBW-region dispatcher records (67%) have zero code references, and the requested-torque RAM variable has zero consumers — the throttle servo is not driven by any tunable table in v0.7. The real throttle mechanism is unknown. Do not expect a throttle remap from these tables.

- Target Throttle Plate Position A @ **0xF923C** (50×12 — corrected in v0.7, was off by 4 bytes; X axis @0xF9144), B @ 0xFAEF4 (50×19), C @ 0xF9800 (50×19) — all unreferenced, treat as decoys
- Requested Torque A @0xFA08C (20×18) — EMU-VALIDATED
- **⚠️ Requested Torque B @0xFA774 — v0.7 axes are wrong.** v0.7 copied FA08C's axes (FA374/FA3C4); the dispatcher record @0xB6F90 points to **FA6DC/FA72C**. Emulation proves the ECU uses the record's axes (X=7.0/Y=2400 → 3403 with record axes vs 456 with v0.7's).
- Requested Torque Limit A @0xFB698 + B @0xFB79C — structurally valid and emulatable, but purpose-suspect (no consumers found). Validation confirms the definition matches the code, not that the ECU uses them.
- DBW chain: pedal → Requested Torque → Target Throttle (Step 2 has no code-referenced table lookup — the link is missing)

---

## Sensors (~15 tables)

- MAF Sensor Scaling @ 0xDCEA0
- MAP Sensor Scaling @ 0xDCCD8 — float32 [250.0, -150.0]
- Coolant Temp Sensor scaling
- Intake Air Temp Sensor scaling
- Fuel Temp Sensor scaling @ 0xDD198

---

## Limits (~15 tables)

- Rev Limit (Fuel Cut) @0xCEA78 = [6700, 6600, 3800, 4000] — CODE-CONFIRMED (3 code refs incl. the limiter routine; mode-selected via RAM 0xFFFF6116)
- **⚠️ Rev Limit Fuel Resume (MP) @0xC9848 — v0.7 decode is wrong.** Listed as uint16 1D with a psi scaling; the actual data is **float32** [760.0, 3500.0, 100.0, 6700.0], zero code refs. Mislabeled, purpose unknown.
- Speed Limiting (Fuel Cut) B @0xCE6B0 = [200, 190, 0, 70] — CODE-CONFIRMED
- Speed Limiting (Throttle) @0xCEB5C = [200, 210, −40, −30] — CODE-CONFIRMED
- Speed Limiting (Throttle) A–D @0xF8E44+ — four byte-identical tables, zero refs. Dead weight, candidates for removal.
- MAF Sensor Limit (Voltage) @0xD5EEC — dead (invalid record, zero refs)

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

## Known v0.7 Errors

Bugs found by the Oct 2026 emulation validation (running tables through the real ROM dispatcher code in the SH-2A emulator). All verified against ROM bytes — these are definition errors, not ROM errors. v0.8 excludes or corrects every one.

| # | Table | Address | Error | Correct |
|---|-------|---------|-------|---------|
| 1 | Fuel Multiplier A | 0xC4AC0 | v0.7 says 13×22 — 7 phantom rows | **13×15** per record @0xB1E74 (Y=RPM 600–4000) |
| 2 | Requested Torque B | 0xFA774 | v0.7 axes copied from FA08C | Axes **FA6DC/FA72C** per record @0xB6F90 |
| 3 | CL Fueling Target Comp C | 0xD3948 | AFR scaling gives impossible 0.00–1.62 AFR | **Correction multiplier** x/32768 → 0.64–1.00 |
| 4 | Rev Limit Fuel Resume | 0xC9848 | Labeled uint16/psi | Actually **float32** data, purpose unknown |
| 5 | Idle Error Corr Mult A | 0xDF60C | v0.7 axes DF6B0/DF6D4 | Axes **DF5C4/DF5E8** per record |
| 6 | Fuel Multiplier B | 0xC4C10 | Listed as real 5×13 table | **Suspected phantom** — data sits inside Fuel Mult A's region |

**Fake/decoy tables in v0.7** (flagged, not yet removed):
- 5× "Knock Correction Advance Max" A–E — zero code refs, impossible values (30–80°). This ECU has no KCA architecture.
- Base Timing A @0xCC7A0 — byte-identical to fake KCA E rows (phantom).
- Target Throttle A/B/C + Torque Limit A/B — 67% of DBW-region records unreferenced, no consumers. Decoys.
- Main Tip-in A/B @0xD13D4/@0xD1474 — zero code refs; the active tip-in uses the Compensation tables.
- MAF Limit @0xD5EEC, Baro Comp A/B — dead.

## Download

See [Definitions](Definitions.md) for the XML files containing all tables.
