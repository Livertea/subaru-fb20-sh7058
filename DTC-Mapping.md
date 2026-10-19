# DTC P-Code Mapping

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


## The Breakthrough

All **243 diagnostic trouble code (DTC) enable/disable switches** in the EP5GD00A ROM were mapped with their P-code labels — **without any disassembly**.

This was done by decoding the DTC descriptor table — a method we haven't seen documented elsewhere for Denso ECUs (if prior art exists, we'd like to know).

## How It Works

### The Descriptor Table

At ROM address `0xA1431`, there is a table of fixed-size **20-byte entries** — one per DTC.

| Byte(s) | Content |
|---------|---------|
| 0–2 | Unknown (descriptor flags) |
| **3–4** | **P-code as big-endian uint16** |
| 5–19 | Unknown (additional descriptor data) |

**The key insight:** Bytes 3–4 of each 20-byte entry encode the P-code directly. For example:
- Bytes `04 20` → P0420 (Catalyst System Efficiency Below Threshold)
- Bytes `04 55` → P0455 (Evaporative Emission System Leak Detected - Large Leak)

### The Switch Table

Each DTC also has a corresponding enable/disable switch (a single byte: `01` = enabled, `00` = disabled). The descriptor table order maps to the switch table order, giving us:

```
P-Code → Descriptor Entry → Switch Index → Switch Address → Enabled/Disabled
```

### Validation

The mapping was validated by checking known codes:
- **P0305/P0306** (Cylinder 5/6 Misfire) → **disabled** from factory ✅
  - Correct — this is a 4-cylinder engine, cylinders 5 and 6 don't exist
- **P0420** (Catalyst Efficiency) → Switch 57, **enabled** ✅
- **P0455** (EVAP Large Leak) → Switch 214, **enabled** ✅
  - This is the car's actual current diagnostic code

**Result:** 160 enabled / 83 disabled out of 243 total.

> **Mapping completeness (verified by independent ROM parse, 2026-10-18):** the descriptor table is **245 entries** (indices 0–244), not 309 — beyond index 244 the format changes completely (a different data structure, not more descriptors). All 245 switch bytes are clean 0x00/0x01. **243/243 real P-codes are in the definition**; the 2 excluded are indices 70–71, both P0000 placeholders (correctly excluded). 7 entries are genuine **second descriptors** for duplicate P-codes (P0340, P0452, P0453, P0451, P0441, P2419, P2420) with **independent** switch states — e.g. P0340: index 3 disabled / index 132 enabled. Correctly mapped as separate switches.

## Notable Codes

| P-Code | Switch | State | Description |
|--------|--------|-------|-------------|
| P0420 | 57 | Enabled | Catalyst System Efficiency Below Threshold (Bank 1) |
| P0455 | 214 | Enabled | Evaporative Emission System Leak Detected (Large Leak) |
| P0301–P0304 | various | Enabled | Cylinder 1–4 Misfire Detected |
| P0305–P0306 | various | Disabled | Cylinder 5–6 Misfire (N/A on 4-cyl) |
| P0171/P0172 | various | Enabled | System Too Lean / Too Rich (Bank 1) |

## The Full Mapping

The complete CSV with all 243 codes:

- **File:** `dtc_pcode_mapping.csv` — ✅ regenerated 2026-10-18: **243 rows** (245 descriptor entries minus indices 70–71, the P0000 placeholders). Regenerated directly from the ROM: descriptor table @0xA1431 (20-byte entries, P-code at bytes 3–4) cross-checked against the switch bank @0xA1338. All 236 previously-mapped rows verified byte-identical; the 7 new rows are the duplicate-P-code second descriptors (P0340 idx 3, P0452 idx 23, P0453 idx 24, P0451 idx 25, P0441 idx 179, P2419 idx 180, P2420 idx 181).
- **Columns:** P-Code, Switch Index, State (1=enabled/0=disabled), Descriptor Address

## In the Definition

All 243 DTC switches are included in the RomRaider definition as Switch-type tables, named with their P-code (e.g., `(P0420) DTC`). They appear under the DTC category in RomRaider.

## ⚠️ A Note on Disabling Codes

These switches control whether the ECU **reports** a diagnostic code. Disabling a code does not fix the underlying condition — it only suppresses the check-engine light.

**Emissions-related codes** (catalyst, EVAP, EGR, O2 sensors) should not be disabled to circumvent emissions equipment. The project's stance:

- ✅ Using the mapping to **understand** what the ECU monitors
- ✅ Calibrating monitors to match **installed, functioning** hardware
- ❌ Disabling monitors to hide removed or defeated emissions equipment

## Prior Art

The FA20 (BRZ/86) open-source definition by callisto includes ~454 P-code-labeled DTC switches, but his README explicitly warns: *"DO NOT ATTEMPT TO DISABLE/ENABLE CEL CODES USING THESE DEFINITIONS."*

Our contribution is the **method** — decoding the descriptor table to derive the mapping from first principles, rather than through disassembly or trial-and-error.
