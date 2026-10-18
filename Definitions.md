# Definitions

Download the RomRaider XML definition files for EP5GD00A.

> **⚠️ Not validated on hardware.** These definitions are structurally validated against ROM bytes but have not been tested on a running engine. Do not flash a modified ROM based on these files.

---

## Versioning

This project uses semantic versioning `X.X.X`:
- **First digit:** 0 = in development
- **Second digit:** Stage (1 = essentials, 2 = extended, 3 = major additions)
- **Third digit:** Revision within stage

**Letter files** (e.g., `0.3.1a`) contain **only new tables** — they're standalone files for validation before merging into the next full revision. Load them alongside the base version in RomRaider.

---

## Current Releases

### v0.3 — Full Definition (488 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.xml` |
| **Tables** | 488 |
| **Date** | 2026-10-18 |
| **Status** | Current full revision |

**What's new in 0.3** (from 0.2.6):
- 10× Tip-in Enrichment Compensation (ECT) — D2000 family
- 4× Idle Speed Targets K–N (family now A–N, 14 total)
- 20× Warmup Compensation (ECT)
- 5× Warmup Compensation Step tables
- 3× special/unknown tables
- **Fix:** Table A @0xCB974 corrected from "Timing Map" to "Unknown Multiplier" (validated NOT timing)

**Fixes included:** 4 Injector Correction tables corrected from broken `<data>` format to proper `storageaddress` attributes.

### v0.3.1a — Letter File (19 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1a.xml` |
| **Tables** | 19 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents** (all code-confirmed as used by firmware):
- 4× Mode Lookup 4×6 tables (integer index axes, scaling TBD)
- 1× Knock/Timing Table @0xCC8FA (8×10, scaling TBD)
- 14× Knock scalars (Feedback params, Advance Step params, Extended Retard)

### v0.3.1b — Letter File (2 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1b.xml` |
| **Tables** | 2 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents:**
- Tip-in Enrichment Compensation (ECT) J @0xD1F60
- Tip-in Enrichment Compensation (ECT) Warm B @0xD25B0

6 other candidates were **rejected** during validation (suspicious data patterns suggesting directory misdecode).

### v0.3.1c — Letter File (4 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1c.xml` |
| **Tables** | 4 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents:**
- Timing Maps G–J @0xD72C4+ (13×17 uint8, 3–18°)
- Key finding: directory implied uint16, but uint8 reading reveals clean timing data

---

## Older Versions

Previous full revisions are preserved for reference:
- **0.2.6** (446 tables) — Last 0.2.x revision
- **0.2.5** (417 tables)
- **0.2.4** (403 tables)
- Earlier versions in `~/workspace/your_files/`

Old files are never renamed or overwritten (project convention).

---

## How to Use

1. Download the XML file(s)
2. Place in your RomRaider `definitions` folder
3. Open your EP5GD00A ROM in RomRaider's ECU Editor
4. Tables appear organized by category

**Requirements:**
- RomRaider (free, open-source)
- A ROM dump from an EP5GD00A ECU
- `<internalidaddress>` must be `2004` (hex), `<internalidstring>` must be `EP5GD00A`

### Loading Letter Files

Letter files are standalone — load them **in addition to** the base version, not instead of it. In RomRaider, you can load multiple definition files for the same ROM.

---

## Changelog

See `RR_EP5GD00A_0.3_CHANGELOG.md` for the detailed 0.3 changelog.
