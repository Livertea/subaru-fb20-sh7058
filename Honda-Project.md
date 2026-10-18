# Honda SH7058 Project

## Why Honda?

The Subaru methodology was developed on a Denso SH7058. Honda also used the SH7058 extensively (2006–2012) — but in ECUs built by **Keihin** (now Hitachi Astemo), not Denso.

The question: does the table-hunting methodology transfer across manufacturers on the same chip?

**Answer: Yes — but the data formats don't.**

This project produced what appears to be the **first open-source Honda SH7058 calibration definition ever created**.

---

## The Target

| | |
|---|---|
| **Vehicle** | 2006–2011 Honda Civic Si |
| **Engine** | K20Z3 2.0L i-VTEC |
| **ECU** | Keihin SH7058, 1MB |
| **Cal ID** | 37805-RRB-A080 |

## RWD Decryption

Honda distributes firmware as encrypted `.rwd` update files (not raw ROM dumps). The decryption was the first hurdle.

**Source file:** `37805-RRB-A080-M1.rwd.gz` from [bouletmarc/HondaCalibFiles](https://github.com/bouletmarc/HondaCalibFiles) (5,357 ECM firmware files, publicly posted by the HondaReflashTool author)

**Cipher** (from HondaReflashTool `Class_RWD.cs`):
```
(((i ^ k2) + k1) - k0) & 0xFF
```
**Keys:** k0=`0x43` ('C'), k1=`0x48` ('H'), k2=`0x58` ('X')

**Validation:** Decrypted firmware contains `37805-RRB-A080` ASCII at ROM `0x93EA0`. ✅

**Note:** The decrypted ROM starts at `0x8000` (bootloader missing) — RWD files are partial ROMs, but calibration data should be complete.

## Key Discovery: Different Data Formats

| Aspect | Subaru Denso SH7058 | Honda Keihin SH7058 |
|--------|---------------------|---------------------|
| **Axes** | float32BE | **uint16BE** |
| **Data** | uint16BE | uint16BE |
| **Endianness** | Big | Big |
| **Table layout** | [X][Y][data] contiguous | [X][Y][data] contiguous |
| **Table directory** | Yes (0xB0000 region) | **Not found yet** |
| **Load units** | g/rev | **mg/stroke** (likely) |

**The critical adjustment:** Honda uses `uint16BE` for axes, not `float32BE`. Same CPU, completely different calibration architecture. This validates the project's iron law: *methodology transfers, structures never do.*

## v0.1 — Proof of Concept (2 tables)

| | |
|---|---|
| **File** | `RR_HONDA_RRB_A080_0.1.xml` |
| **Tables** | 2 |
| **Date** | 2026-10-18 |
| **Status** | Proof of concept — NOT for tuning |

### Tables

1. **Ignition Timing (20×20)** @ 0x15E38/0x15E60/0x15E88
   - X: RPM [500–9000], Y: Load [500–9000] (likely mg/stroke)
   - Scaling UNVERIFIED

2. **VTEC Engagement RPM** @ 0xB26E
   - Values [5800, 5600, 5800, 5600] — engagement/disengagement with hysteresis

## What's Next for Honda

To reach a useful definition:
1. Hunt fuel tables (RPM × MAP, AFR targets)
2. Find VTC maps and VTEC High/Low variants
3. Verify scalings via Hondata documentation or logger data
4. Map the full table inventory (~200+ expected)

## Significance

Before this project: **zero** open-source Honda SH7058 definitions existed in any language. The only options were commercial tools ($449–$795).

This v0.1 proves an open Honda definition is achievable using the same methodology developed for Subaru.

## Prior Honda Work

The only prior open-source Honda SH7058 effort found:
- **HondaReflashTool** (bouletmarc) — Open-source CAN/J2534 reflash tool, stagnant since 2022. Contains 416-parameter definitions that appear to be ported from Hondata (not ground-up reverse engineering).
- **jpancotti/rwd-xray** — RWD decryption tool, actively maintained.

No ground-up Honda SH7058 table-hunting project existed before this one.
