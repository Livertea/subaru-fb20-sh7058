# DTC P-Code Mapping

## The Breakthrough

All **236 diagnostic trouble code (DTC) enable/disable switches** in the EP5GD00A ROM were mapped with their P-code labels — **without any disassembly**.

This was done by decoding the DTC descriptor table, a method that appears to be novel (no prior published documentation of this technique for Denso ECUs was found).

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

**Result:** 160 enabled / 76 disabled out of 236 total.

## Notable Codes

| P-Code | Switch | State | Description |
|--------|--------|-------|-------------|
| P0420 | 57 | Enabled | Catalyst System Efficiency Below Threshold (Bank 1) |
| P0455 | 214 | Enabled | Evaporative Emission System Leak Detected (Large Leak) |
| P0301–P0304 | various | Enabled | Cylinder 1–4 Misfire Detected |
| P0305–P0306 | various | Disabled | Cylinder 5–6 Misfire (N/A on 4-cyl) |
| P0171/P0172 | various | Enabled | System Too Lean / Too Rich (Bank 1) |

## The Full Mapping

The complete CSV with all 236 codes is available:

- **File:** `dtc_pcode_mapping.csv`
- **Columns:** P-Code, Switch Index, State (1=enabled/0=disabled), Descriptor Address

## In the Definition

All 236 DTC switches are included in the RomRaider definition as Switch-type tables, named with their P-code (e.g., `(P0420) DTC`). They appear under the DTC category in RomRaider.

## ⚠️ A Note on Disabling Codes

These switches control whether the ECU **reports** a diagnostic code. Disabling a code does not fix the underlying condition — it only suppresses the check-engine light.

**Emissions-related codes** (catalyst, EVAP, EGR, O2 sensors) should not be disabled to circumvent emissions equipment. The project's stance:

- ✅ Using the mapping to **understand** what the ECU monitors
- ✅ Calibrating monitors to match **installed, functioning** hardware
- ❌ Disabling monitors to hide removed or defeated emissions equipment

## Prior Art

The FA20 (BRZ/86) open-source definition by callisto includes ~454 P-code-labeled DTC switches, but his README explicitly warns: *"DO NOT ATTEMPT TO DISABLE/ENABLE CEL CODES USING THESE DEFINITIONS."*

Our contribution is the **method** — decoding the descriptor table to derive the mapping from first principles, rather than through disassembly or trial-and-error.
