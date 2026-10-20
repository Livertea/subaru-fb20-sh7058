# Table Reference

459 tables in 0.8.4a. Every table carries a liveness verdict from the [Liveness-Matrix](Liveness-Matrix.md) — check the verdict before touching anything.

## By category

| Category | Tables | Liveness | Notes |
|---|---|---|---|
| Timing | 14 | CLEAR / CAUTION | Base E/F are the primary ignition target; B3 high-RPM verified but channel-inert |
| Knock Control | 29 | CAUTION | 26 params taint-verified 10-19; 5 fake KCA quarantined (DO NOT TUNE) |
| Idle | 18 | CAUTION | Includes the decel adders — **proven live 10-19** (reverses the old "dead" verdict) |
| Torque | 6 | CLEAR | Requested Torque A/B, tip-in |
| Fuel | — | **CAUTION** | Live PW path mapped to a trim path 10-19, main computer unidentified; no fuel table tunable yet |
| AVCS | 4 | CLEAR (intake) / CAUTION (exhaust) | Intake closed-loop tunable; exhaust is open-loop driven (@0x17F88) — tunable as feedforward map |
| Limiters | 2 | CLEAR / DO NOT TOUCH | 6700 tunable; 4400 diagnostic is protection |
| DTC | 243 | Infrastructure | P-code switches, inhibitor table |
| Misc | remainder | Mixed | See per-table descriptions for confidence |

## Reading a table entry

Each table in the XML carries:
- **Address, dims, axes** — decoded from ROM bytes, zero-collision checked
- **Scaling** — e.g. `x*0.1` degrees. Scalings marked **VERIFIED** were confirmed by emulator or log; unmarked scalings are best-effort
- **Description** — includes the evidence tier and any caveats. A description that says "unverified" means exactly that

## Recently verified (0.8.4a, 10-19)

- **26/26 knock params** — every one taint-verified through the emulator (tainted ROM value observed reaching live code). Includes Min Load A = 0.48 (corrects the v0.7 0.45 error).
- **20/20 idle/DBW** — structure re-read, dispatcher emulation, 4-layer liveness. Includes the v0.7 axis corrections (Requested Torque B axes, ECM-A axes).
- **B3 High-RPM Timing** — 8×7, 7.4°–34.4°, live-looked-up but **computed-but-discarded** (B3 protection channel is inert downstream).
- **Dwell** — description updated with the corrected clock tree (TCR2A/B=0x04, one-shot armed); **Pφ' = 10 MHz grounded 10-19 (TICKRATE, two independent init paths)** — the old 2× absolute-ms crystal uncertainty is closed. (The dwell computer @0x4C908 is orphaned — one unread ROM reference, no periodic caller; see [Timing-System](Timing-System.md).)

## Known-bad entries (quarantined, not deleted)

- **5 fake "Knock Correction Advance" tables** — values 30–80° (physically impossible), zero code references. Not knock tables. Left in the XML quarantined so nobody re-discovers them.
- **Timing Maps G–J — REAL tables, label corrected 10-19.** The old "0xFFFF sentinel garbage" verdict was wrong: they have a confirmed lookup chain (taint 1.0 through the arbitration block @0x45AA4) and cleared scaling **(x·0.3515625)−20** — which refutes the 0.8.3 XML's x·0.1 on these tables. But they are **NOT "Base Timing"**: [0xFFFF84BC] is G-independent ([0xFFFF84C0]=0.0 multiplies the G term away; [0xFFFF84BC] is a 13° trim). Name them as timing-map-to-arbitration inputs. Revival caveat **REFUTED 10-19 (84C0GATE):** the byte[0xFFFF851C]==0 increment path is reachable only in the boot window before Task 1's first pass (a ≤0.0156×N-pass blip — the byte[851C]=1 store is @0x463E4, called only from Task 1's dispatcher @0x4D63C); then **[0xFFFF84C0] is pinned at 0 by a one-way latch, no clearer ROM-wide**. G/H/I/J cannot revive in sustained operation. Mode $22 logging of [84C0]+[851C] would only confirm the pin.
- **New u8 1D RPM timing table (not in any shipped XML)** — record/dir 0xB5618, X @0xD4F0C, data @0xD5230, scaling ×0.3515625−20. The true base-timing input to the arbitration block. Add to the definition.

## Versioning

Revisions ship as **letter files** (new tables only, e.g. 0.8.4a) for validation, then merge into **numbered full versions** (e.g. 0.8.4). A letter file never carries old tables — if it does, that's a bug, and it's in the [Project-Log](Project-Log.md).
