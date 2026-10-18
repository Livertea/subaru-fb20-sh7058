# Definitions

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


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

### v0.7 — Full Definition (863 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.7.xml` |
| **Tables** | 863 |
| **Date** | 2026-10-18 |
| **Status** | Current full revision |

**What's new in 0.7** (from 0.6):
- +101 tables from letter file 0.3.1u (87 uint16 + 14 float32) — found by re-running the missing-table triage against the 169 resolved secondary-dispatcher call sites
- **New structural discovery:** the secondary dispatcher's type byte selects the *data type* — type-0 = float32 tables with scaling passthrough (14 of the new tables)
- −1 table: 0.3.1t's "Unknown 2D @0xCDE30" was wrong (misread a float RPM axis as data), superseded by the 9×9 float table @0xCDE54
- 4 byte-level corrections: Base Timing B → 13×17 @0xCCA3A (old 13×13 was a window into it); CL Fueling Target Comp C 0xD3946 → 0xD3948; Target Throttle A 0xF9240 → 0xF923C (X axis 0xF9148 → 0xF9144); tip-in A/B resolved to 16×8, X=ECT, Y=time-since-tip-in, scale 1/256, % enrichment
- Breakdown: 260 3D · 333 2D · 34 1D · 236 Switch (functional switch count 243, incl. 7 DTC switches typed 2D)
- Coverage: ~97% of revised ~875–900 universe. Table hunt remains closed.

### v0.6 — Full Definition (763 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.6.xml` |
| **Tables** | 763 |
| **Date** | 2026-10-18 |
| **Status** | Superseded — current is v0.7 (863 tables) |

**What's new in 0.6** (from 0.5):
- +8 tables from letter file 0.3.1t (found by the disassembly verification track)
- Verification track complete: direct diff of all 763 XML addresses against the 832-entry directory, both dispatchers, and 2,392 direct flash refs — **0 phantoms**
- Breakdown: 159 3D · 327 2D · 34 1D · 243 Switch (functional switch count)

### v0.3 — Full Definition (488 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.xml` |
| **Tables** | 488 |
| **Date** | 2026-10-18 |
| **Status** | Superseded — current is v0.7 (863 tables) |

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

### v0.3.1d — Letter File (1 table)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1d.xml` |
| **Tables** | 1 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents:**
- Unknown ECT Compensation @0xC3E38 (16-el, smooth decrease 655→218)

### v0.3.1e — Letter File (5 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1e.xml` |
| **Tables** | 5 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents:** Tier 2 dispatcher tables, all code-confirmed:
- 0xC01B8 (16-el, binary step pattern)
- 0xC3FE8 (ECT comp, companion to 0xC3E38)
- 0xD98C8 (Warmup Comp ECT)
- 0xDE798 (ECT comp with plateaus)
- 0xCAC80 (9-el RPM table)

### v0.3.1f — Letter File (5 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1f.xml` |
| **Tables** | 5 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents:** From the 31-table ECT family @0xCEB64:
- 0xCECA4, 0xCF546, 0xCED34, 0xCF466, 0xCF4A6 (all 16-el, shared ECT axis)

### v0.3.1g — Letter File (78 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1g.xml` |
| **Tables** | 78 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Pending validation |

**Contents:** Bulk 2D triage from dispatcher cross-reference — 78 validated tables from the 238-table dispatcher discovery. All physically spot-verified against ROM bytes.

**Running total: 763 tables** (628 in v0.4 + 78 + 42 + 7 from letters q, r, s, merged in v0.5; +8 from letter 0.3.1t, merged in v0.6).

### v0.3.1u — Letter File (101 tables)

| | |
|---|---|
| **File** | `RR_EP5GD00A_0.3.1u.xml` |
| **Tables** | 101 (new only) |
| **Date** | 2026-10-18 |
| **Status** | Merged in v0.7 |

**Contents:** Re-triage of the 162 unique secondary-dispatcher directory entries (169 resolved call sites as code-ref source). 87 uint16 + 14 float32 (type-0) tables, all marked Unverified with call-site evidence. Uses real directory scale/offset, not the x/32768 placeholder. 20 candidates rejected (flat/zeros/alternating), 3 rejected as bogus/stale directory entries, 4 conflicts adjudicated as corrections to existing tables.

**Running total: 863 tables** (763 in v0.6 + 101 − 1 superseded, merged in v0.7).

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
