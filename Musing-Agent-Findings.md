# Musing Agent Findings (2026-10-18)

> **⚠️ UNVERIFIED RESEARCH — MAY BE 100% WRONG**
>
> Five AI agents were given autonomy to muse on the project and test theories. These are their findings, provided as-is. The human (Johnathon) has not verified any of this. If something is wrong, that's on the AI.

---

## Agent 1: Technical Achievement + 0xD0000 Theory

**Theory:** The 0xD0000 region dismissed as "not a table" might be meaningful.

**Verdict: CONFIRMED. We were wrong to dismiss it.**

### Findings

0xD0000 is not "repeating 128B f32 template blocks." It's a structured shared component:
- `0xD0000`: f32 x2 = `100000.0, 110000.0` (header/signature)
- `0xD0008`: u16 x12 = packed data (~1.02-1.05 as x/32768, looks like multipliers)
- `0xD0020`: f32 x9 = monotonic axis `(0, 3, 5, 10, 20, 50, 60, 70, 75)`
- `0xD0044`: u16 x10 = packed data
- `0xD0058`: f32 x9 = same axis repeated
- `0xD007C`: u16 x10 = packed data (slightly different values)

**17 directory entries reference 0xD0000.** It's a shared structure, not a standalone table.

The signature appears at 6 locations (0xCF0B8, 0xCF124, 0xCFFB8, 0xD0000, 0xD00B8, 0xD0100) forming 3 identical pairs. Only 0xD0000 is live (has directory refs). Others are orphaned copies.

### False Positives Identified

Two tables in our definitions are likely **false positives**:
- `0xD0044` "Unknown 2D" — actually packed u16 data within the 0xD0000 structure
- `0xD007C` "Unknown 2D" — same issue

**If removed: 755 → 753 tables.**

### Action Items
- [ ] Review 0xD0044 and 0xD007C, remove if confirmed as false positives
- [ ] Document 0xD0000 as shared structure (not a tunable table)
- [ ] Update NEGATIVE results list: change "0xD0000 (not a table)" to "0xD0000 (shared structure, not standalone table)"

---

## Agent 2: AI Labor + Empty Flash Theory

**Theory:** The empty flash region was intentionally left for future features.

**Verdict: NOT SUPPORTED. Likely a linker gap.**

### Findings

- Actual gap: `0xDFCE8` to `0xF8E00` = **~100KB**, not 128KB as previously estimated
- Boundaries are **ragged** (not aligned to flash erase blocks)
  - Start: 0xDFCE8 (4,888 bytes before round 0xE0000)
  - End: 0xF8E00 (3,584 bytes after round 0xF8000)
- If Denso intentionally reserved blocks, boundaries would align to 32KB/64KB/128KB. They don't.
- **2 code references** to the range, both are comparison constants (not pointers). Code never reads/writes there.
- The 0xF8000+ section has **38 code references** (directly addressed, not via dispatcher)

**Most likely explanation:** Linker places main code+calibration ending at 0xDFCE8, then places DBW/throttle section at fixed address 0xF8000 (hardcoded pointers). The ~100KB gap is unused linker space.

**However:** The practical opportunity (TephraMod-style code injection) is real regardless of intent.

### Action Items
- [ ] Update docs: change "128KB" to "~100KB (0xDFCE8–0xF8E00)"
- [ ] Update theory: "intentionally reserved" → "linker slack, but usable for code injection"

---

## Agent 3: Seeding the Ether + Base Timing Scaling

**Theory:** Base Timing x*0.05 scaling can be proven from disassembly.

**Verdict: The premise was stale. XML already uses x*0.1 (correct). Docs were wrong.**

### Findings

The 0.4 XML uses **`x*0.1`** for Base Timing, not `x*0.05`. The docs (REFLECTIONS_V2.md, FB20_POWER_POTENTIAL.md) said `x*0.05` but were stale.

**Physical plausibility:**

| Condition | Raw | x*0.1 | x*0.05 |
|-----------|-----|-------|--------|
| Idle | 79 | 7.9° ✓ | 3.95° (too retarded) |
| Cruise | 173 | 17.3° ✓ | 8.65° (too low) |
| WOT | 163 | 16.3° ✓ | 8.15° (absurd) |

**Disassembly:** Zero `0.1f` or `0.05f` constants in code. The ECU likely works in native 0.1°/LSB — no scaling operation exists because no scaling is needed. The "scaling" is physical interpretation, not computation.

**Confidence: High** that x*0.1 is correct.

### Action Items
- [x] Fixed: Updated REFLECTIONS_V2.md and FB20_POWER_POTENTIAL.md (x*0.05 → x*0.1)
- [ ] The "highest-risk scaling" is actually already correct in the XML

---

## Agent 4: Methodology + Tip-in Y-Axis

**Theory:** Tip-in enrichment Y axes can be determined from code.

**Verdict: PARTIALLY CONFIRMED for T1, UNCONFIRMED for T2.**

### Findings

**T1 (@0xD13D4):** Directory entry (0xB4C08) points to **0xCEB64** (shared ECT axis).
- Row peaks increase monotonically: [6.42, 6.42, 7.00, 7.70, 8.35, 8.47, 8.60, 8.73]ms
- Physics: colder → more enrichment needed
- **Likely ECT, hot→cold row ordering. Medium confidence.**

**T2 (@0xD1474):** Candidate axis at 0xD14F4 ([3, 6, 8, 10, 12, 14...]), but **low confidence**, not confirmed by directory.

**Concerning:** Neither table found in code reference searches. Possibly dead tables or indirect access.

**Bonus:** Discovered **second dispatcher @0xBEA18** (see separate page).

### Action Items
- [ ] Update T1 Y-axis: "TBD" → "ECT (likely, unconfirmed row mapping)"
- [ ] Keep T2 as TBD
- [ ] Flag both as "not found in code refs — may be dead or indirectly accessed"

---

## Agent 5: Human-AI Collaboration + Hidden Tables

**Theory:** There are tables using neither dispatcher nor directory.

**Verdict: NOT SUPPORTED for tables. But scalar blocks are unmapped.**

### Findings

Scanned 4,321 direct flash references, cross-referenced against 755 tables.

**No hidden table population found.** The throttle tables at 0xFxxxx remain the only confirmed "bypass" group.

**But:** Two dense scalar parameter clusters found (not tables):
- `0xC9948–0xCA22C` (2.3KB, 553 refs): Float constants
- `0xD8374–0xD8AA8` (1.8KB, 593 refs): Mixed scalars

These overlap with P1-P5 (unidentified). They're real calibration data but **not tables** — individual constants.

### Coverage Assessment

- **Tables:** ~95% credible. The 755 count is solid.
- **All calibration data (incl. scalars):** ~80-85%. Scalar parameter blocks not mapped.

### Recommendation

**Don't chase more tables.** The table hunt is done. Remaining work:
1. Link orphan axes to tables (metadata)
2. Identify scalar parameter blocks (different methodology)
3. Validate on hardware (cable Nov 6)

---

## Summary of Action Items

### Corrections Needed
- [ ] Remove 2 false positives (0xD0044, 0xD007C) → 755 becomes 753
- [ ] Update empty flash: 128KB → ~100KB, "intentional" → "linker slack"
- [x] Fix Base Timing docs: x*0.05 → x*0.1 (DONE)
- [ ] Update T1 tip-in Y-axis: TBD → ECT (likely)
- [ ] Document 0xD0000 as shared structure

### Documentation Needed
- [x] Second dispatcher page (DONE)
- [ ] Update Methodology.md for multiple dispatchers
- [ ] Update NEGATIVE results list (0xD0000, empty flash)

### Strategic
- [ ] Table hunt is DONE. Shift to: axis linking, scalar blocks, hardware validation

---

*All findings provided as-is. May be 100% wrong. Human verification required.*
