# RR_EP5GD00A 0.3 Changelog

**Previous:** 0.2.6 (446 tables) → **Current:** 0.3 (488 tables)
**Net change:** +42 tables, 1 fix
**Date:** 2026-10-18

This is a major version bump (0.2 → 0.3) reflecting the addition of two entirely new table families and crossing the 480-table threshold.

---

## FIXES (1)

### Table A @0xCB974 — Corrected Identity
- **Was:** "Timing Map (Unverified) @0xCB974" (Timing category, x/1000 scaling → degrees)
- **Now:** "Unknown Multiplier (Load×RPM) @0xCB974" (Misc category, x/32768 scaling → multiplier)
- **Reason:** Structural validation 2026-10-18 proved this is NOT ignition timing. Only 21 unique values out of 80 (heavily quantized, not smooth). Non-monotonic across load. At idle, values are 9–22° higher than Base Timing A — no compensation table adds 22° at idle. As x/32768, values are 0.165–0.784×, consistent with the thermal/warmup multiplier cluster it belongs to.
- **Note:** "NOT ignition timing — validated 2026-10-18. Thermal/warmup cluster member."

---

## NEW TABLES (42)

### D2000 Tip-in Enrichment Compensation (ECT) Family (10 tables)

Identified 2026-10-18 via cross-reference with callisto's FA20 definition (structural match to his "Tip-in Enrichment Compensation B/C (ECT)" tables) and Forester EP5G211a precedent. All use shared ECT X-axes, values decreasing with temperature (more compensation when cold). Small-int Y-axes select tip-in severity modes.

**Sub-family 1 — 16×5 (3 tables):**
- Tip-in Enrichment Compensation (ECT) A @0xD2014 (X @0xCEBA4, Y @0xD2000)
- Tip-in Enrichment Compensation (ECT) B @0xD20C8 (X @0xCEBA4, Y @0xD20B4)
- Tip-in Enrichment Compensation (ECT) C @0xD217C (X @0xCEBA4, Y @0xD2168)
- Scaling: x/65536 (values 0.90–0.975)

**Sub-family 2 — 16×3 (2 tables):**
- Tip-in Enrichment Compensation (ECT) D @0xD2228 (X @0xCEB64, Y @0xD221C)
- Tip-in Enrichment Compensation (ECT) E @0xD2294 (X @0xCEB64, Y @0xD2288)
- Scaling: x/65536

**Sub-family 3 — 16×2 (4 unique tables):**
- Tip-in Enrichment Compensation (ECT) F @0xD22FC
- Tip-in Enrichment Compensation (ECT) G @0xD231C
- Tip-in Enrichment Compensation (ECT) H @0xD233C
- Tip-in Enrichment Compensation (ECT) I @0xD235C
- All share X @0xCEB64, Y @0xD22F4 ([4,5])
- Scaling: x/65536
- Note: Paired duplicates exist at 0xD237C–0xD24F4; only 4 unique added.

**Sub-family 4 — 7×3 (1 representative):**
- Tip-in Enrichment Compensation (ECT) Warm @0xD255C (X @0xD2534 [50..110°C], Y @0xD2550 [3,4,5])
- Scaling: x/65536
- Note: 4 identical directory entries (0xD255C–0xD2658); adding one representative.

### 0xD8000+ Warmup Block (32 tables)

Discovered via region sweep 2026-10-18. This entire region is Denso's cold-start/warmup calibration block — previously almost entirely unmapped.

**Idle Speed Targets — 4 new (HIGH confidence):**
- Idle Speed Target K @0xD9270
- Idle Speed Target L @0xD9290
- Idle Speed Target M @0xD9310
- Idle Speed Target N @0xD9350
- All: 16-el, X @0xD8BA8 (ECT -40..110°C), scaling x*0.195312 → RPM
- Note: Named K-N to avoid collision with existing F-J (added in 0.2.6). These are additional siblings with different cold-start RPMs (1200/1300/1700/1800), suggesting different operating conditions (neutral/in-gear, A/C on/off, etc.). Total idle target family is now 14 tables (A-N).

**Warmup Compensation (ECT) — 20 tables (HIGH confidence structure):**
- Warmup Compensation (ECT) A through T
- Addresses: 0xD8ED6, 0xD8EF6, 0xD8F36, 0xD8F56, 0xD8F76, 0xD8FB6, 0xD8FD6, 0xD8FF6, 0xD9016, 0xD9036, 0xD9056, 0xD9076, 0xD9096, 0xD90B6, 0xD90D6, 0xD9116, 0xD9136, 0xD9156, 0xD9176, 0xD93D0
- All: 16-el, X @0xD8BA8 (ECT -40..110°C), scaling x*0.00152588 (values ~1.6–18.0% enrichment)
- Values decrease monotonically with ECT (12.5% at -40°C → 1.8% at operating temp)
- Note: Fuel vs timing identity needs logger validation. A/B pairing pattern observed (Denso map-family design).

**Warmup Compensation Step (ECT) — 5 tables (MEDIUM confidence):**
- Warmup Compensation Step (ECT) A @0xD9526
- Warmup Compensation Step (ECT) B @0xD9546
- Warmup Compensation Step (ECT) C @0xD9566
- Warmup Compensation Step (ECT) D @0xD9586
- Warmup Compensation Step (ECT) E @0xD95A6
- All: 16-el, X @0xD8BA8, scaling x*0.00152588
- Note: Step-function character, identity provisional.

**Special tables — 3 tables (MEDIUM confidence):**
- Unknown Switch Table @0xD8EC0 (11-el, X @0xD8E94 [0-100])
- Unknown Table @0xD91FC (9-el, X @0xD91D8)
- Unknown ECT Switch @0xD9250 (16-el, X @0xD9210 ECT)

---

## NOT ADDED (held for future work)

- Format A 4 tables @0xD8000 region (data format unclear, may be float32)
- Format B 16×2/2×2 tables (likely float32 data, need re-decode)
- All-zeros table @0xD940C (disabled/placeholder)
- Constants @0xD8E52, @0xD8E72, @0xD9390 (not tables)
- D2000 sub-family 3 duplicates (0xD237C–0xD24F4, 4 paired duplicates)
- D2000 sub-family 4 duplicates (0xD2578–0xD2658, 3 identical copies)

---

## TABLE COUNT RECONCILIATION

| Type | 0.2.6 | 0.3 | Change |
|------|-------|-----|--------|
| 3D   | 136   | 146 | +10 (D2000 family) |
| 2D   | 60    | 92  | +32 (warmup block) |
| 1D   | 14    | 14  | 0 |
| Switch | 236 | 236 | 0 |
| **Total** | **446** | **488** | **+42** |

---

## VALIDATION

- XML is well-formed (parsed successfully with Python ElementTree)
- Table A fix confirmed: renamed, recategorized to Misc, rescaled to x/32768
- All 42 new tables confirmed present by name search
- All axis addresses verified against raw ROM bytes (float32BE confirmed)
- D2000 sub-family 3 Y-axis: shared @0xD22F4 confirmed ([4.0, 5.0])
- Follows existing 0.2.6 XML format exactly
