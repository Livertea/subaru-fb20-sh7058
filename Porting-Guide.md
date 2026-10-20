# Porting Guide

*I have a different car — how much of the EP5GD00A work helps me?*

## The Iron Law

> **Port the METHOD. Never the ADDRESSES.**

The CVT reference definition's addresses were verified garbage on the MT ROM; blind porting died on day one. Across every cross-check (Subaru, Honda, Mazda): **zero** same-named tables shared addresses. What transfers: table **names**, **scalings**, **axis identities**, **DTC layouts**, and the **workflow** — dispatcher-tracing, directory-decode, reject audits ([Methodology](Methodology.md)).

## Case study: EP5I200F port (10-18) — first measured port

The 2013 USDM Impreza 2.0i MT ROM (EP5I200F, from the bludgod/RomRaider repo) was ported using v0.7 as a Rosetta Stone, scoped to 76 normally-tuned tables: **69/76 mapped (91%) in ~48 minutes** — 58 moved addresses, 11 byte-identical in place.

Findings for every future FB20 port:

- **2012→2013 calibration is essentially byte-identical** (correlation 1.0000). Porting within FB20 is address-mapping, not recalibration. The DBW/torque/limiter block (0xF8E44–0xFB79C) didn't move a byte.
- **Calibration moves in blocks.** The 24-scalar knock cluster moved intact (+0x66C). When one table in a cluster moves, check the same offset for neighbors.
- **The directory lies here too.** Stale entries exist — nothing taken on faith; every claim backed by data-correlation.
- **Duplicates are real — don't "dedupe."** Base Timing Idle has 5 byte-identical duplicates; they're separate slots the ECU reads independently.
- **The port caught a real v0.7 bug:** Base Timing F is 7×16, not 13×17 (fixed).
- **One genuine recalibration:** Ignition Dwell (correlation 0.94) — likely a coil revision; flag before tuning.
- **6 tables unresolved** (Base Timing C/D, Knock A/B/C/D) — need dispatcher tracing. **Base Timing A FOUND 10-19 (BASETIMING):** @0xCC436 (record @0xB3748), 13×17, **live-looked-up** via the 16-entry record list @0x9E8A8 consumed by @0x8242C via base+offset indexing — result → [0xFFFFA1F4], emulator-validated at cruise. The "zero code references" verdict is refuted (a layer the four-layer rule didn't cover). SIB1 @0xCC3C6 promoted from "lookup unconfirmed" to live-looked-up (→ [0xFFFFA1F8]). Note: the lettered B-family does NOT feed the spark sum [8798] — A-cand reads 13.43° at the cruise point and does not close the 37.3° gap.

Full report: `EP5I200F_PORT_REPORT_2026-10-18.md`, `address_map_ep5gd00a_to_ep5i200f.csv`.

## Other targets (difficulty)

- **Other FB20 SH7058** (2012–2016 Impreza/Crosstrek): 1/5 with the ROM. Directory-decode, diff against EP5GD00A, trace the rest. Weekend project.
- **FB25 SH7058** (Forester/XV 2.5): 2/5. Same chip and generation; expect structural similarity, all new addresses.
- **FA20 SH72531** (BRZ/86): 3/5. Different chip; the blind-mapping experiment measured the method at 100% table precision on this ROM — feasible, slower.
- **Honda SH7058**: 3/5. Separate spin-off project exists (92 tables) using the same methodology.
- **Anything else**: the method transfers; nothing else does.

## Known ROM sources

- **bludgod/RomRaider** (GitHub, free): 14 FB20/FB25 SH7058 ROMs (.hex), incl. EP5F300H (2013 EDM MT) and EP5I200F (2013 USDM MT). EP5GD00A itself: only the t=19276 thread dump.
- **Merp/SubaruDefs** (Alpha branch, free): FB20/FB25 ECUFlash defs (community quality — verify).
- **avtochip.ru** (paid): LEP5F300 lists the same internal ROM ID as EP5GD00A — closest documented sibling.
