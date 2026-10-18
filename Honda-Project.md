# Honda SH7058 Project

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


## Why Honda?

The Subaru methodology was developed on a Denso SH7058. Honda also used the SH7058 extensively (2006–2012) — but in ECUs built by **Keihin** (now Hitachi Astemo), not Denso.

The question: does the table-hunting methodology transfer across manufacturers on the same chip?

**Answer: Yes — but the data formats don't.**

This project produced an open-source Honda SH7058 calibration definition using the same methodology developed for Subaru — directory decode first, then dispatcher tracing, then physical validation.

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
| **Table directory** | Yes (0xB0000 region) | **Yes (0xA454–0xDB8C, 20-byte entries)** |
| **Load units** | g/rev | **mg/stroke** (likely) |

**The critical adjustment:** Honda uses `uint16BE` for axes, not `float32BE`. Same CPU, completely different calibration architecture. This validates the project's iron law: *methodology transfers, structures never do.*

### Directory Entry Format (20 bytes)

| Bytes | Content |
|-------|---------|
| 0 | X dim (u8) |
| 1 | 0x66 (marker) |
| 2 | Y dim (u8) |
| 3 | 0x66 (marker) |
| 4–7 | Meta (u32, meaning unknown) |
| 8–11 | X axis pointer (u32) |
| 12–15 | Y axis pointer (u32) |
| 16–19 | Data pointer (u32) |

Dims were verified against actual axis lengths. Some meta=0x03 groups have overlapping data packing (verified as distinct tables).

## v0.3 — Directory Decoded (92 tables)

| | |
|---|---|
| **File** | `RR_HONDA_RRB_A080_0.3.xml` |
| **Tables** | 92 |
| **Date** | 2026-10-18 |
| **Status** | Early — NOT for tuning |

### Breakdown

- **14 Ignition tables** — A/B HIGH confidence (match v0.1 data exactly); rest MEDIUM by shape
- **10 Fuel tables** (probable, MEDIUM — scaling unknown)
- **2 VTC Intake Cam Target** (HIGH structure, 12×12)
- **66 Unknown 3D** (LOW — directory-validated, unidentified)

89 tables have clean monotonic axes (5 use signed int16 axes); 3 degenerate flagged LOW. All tables carry per-table confidence comments in the XML.

### Corrections to earlier versions

- v0.1's "Ignition 20×20" was **wrong** — the block is 20×10; the next 200 values are Ignition B @0x16018.
- v0.2's VTC entries were misaligned — the real tables are 12×12 @0x197B8/@0x198D8 (v0.2 started 24 bytes into the data, missing row 0).

## v0.1 — Proof of Concept (2 tables)

| | |
|---|---|
| **File** | `RR_HONDA_RRB_A080_0.1.xml` |
| **Tables** | 2 |
| **Date** | 2026-10-18 |
| **Status** | Superseded by v0.3 — proof of concept only |

### Tables

1. **Ignition Timing (20×20)** @ 0x15E38/0x15E60/0x15E88
   - X: RPM [500–9000], Y: Load [500–9000] (likely mg/stroke)
   - Scaling UNVERIFIED

2. **VTEC Engagement RPM** @ 0xB26E
   - Values [5800, 5600, 5800, 5600] — engagement/disengagement with hysteresis

## Scalar Directory — Not Cracked (Scoped)

The scalar (non-table) directory resisted decoding. What's known:

- Region 0xC800–0xC8B4 contains `NN 66 66 00` / `NN 66 02 00` markers interleaved with `00 01 44 XX`-pattern values — but markers are **not regularly spaced** (12/16-byte gaps), so all fixed-stride parses fail. Likely variable-length entries or interleaved structures.
- Ground truth nearby: **VTEC scalars are inline** at ROM 0xB26E = 5800/5600/5800/5600 (u16).
- **Rev limiter: not found.**

The 92 3D tables are the substance; scalars are the long tail. Needs a dedicated attempt or logger data.

## What's Next for Honda

To reach a useful definition:
1. Identify the 66 unknown 3D tables (BitEdit's Honda map list is the ID guide)
2. Verify fuel and VTC scalings (currently unknown)
3. Decode the scalar directory (see above)
4. Find the rev limiter
5. Verify via logger data or Hondata documentation

## Significance

An early open-source Honda SH7058 definition built ground-up with the Subaru methodology — directory decode took it from 4 tables to 92 in one pass. The transferable lesson: the *method* (find the directory, validate against raw bytes, never trust dims) worked on Keihin's completely different data layout with only the axis-type assumption changed.

## Prior Honda Work

The only prior open-source Honda SH7058 effort found:
- **HondaReflashTool** (bouletmarc) — Open-source CAN/J2534 reflash tool, stagnant since 2022. Contains 416-parameter definitions that appear to be ported from Hondata (not ground-up reverse engineering).
- **jpancotti/rwd-xray** — RWD decryption tool, actively maintained.

We haven't found another ground-up Honda SH7058 table-hunting project — if one exists, we'd like to know about it (see [Contributing](Contributing.md)).
