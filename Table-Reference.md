# Table Reference

Tables mapped in the EP5GD00A definition, organized by category. Counts are from v0.3 plus letter files (0.3.1a/b/c).

> **Status:** All tables are structurally validated against ROM bytes. Labels and identities range from HIGH to MEDIUM confidence. None are hardware-validated yet.

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
| Base Timing A–F | 0xCCxxx range | Primary ignition maps |
| Timing Maps G–J | 0xD72C4+ | 13×17 uint8, lower values (3–18°) — possibly correction/limit tables |

- Scaling: `x * 0.05` → degrees
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

See [[Ghidra-Guide]] for the function map.

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

- Target Throttle Plate Position A @ 0xF9240 (50×12), B @ 0xFAEF4 (50×19), C @ 0xF9800 (50×19)
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

## DTC Switches (236 tables)

Every diagnostic trouble code has an enable/disable switch. All 236 mapped with P-code labels.

See [[DTC-Mapping]] for the full story.

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

## Download

See [[Definitions]] for the XML files containing all tables.
