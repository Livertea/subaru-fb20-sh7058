# Open Threads — Genuinely Open Work

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

This page tracks genuinely open research and action threads. A full archive of *resolved* threads is at the bottom — most of the original backlog is done.

## Open Threads

### 1. Hardware Validation (Nov 6 Cable)
Everything in this wiki is **unvalidated against real hardware**. Plan: buy the ~$80 Washinglee OpenPort 2.0 clone (buy-reminder cron fires Nov 6, 2026), pull the car's own ROM, validate definitions offline, then logger-validate against real sensor data. First flash will be a read + write-back of the unmodified image.

**Update 2026-10-18:** the checksum thread is resolved (routine at ROM 0x152 is a reflash-protocol tool, not a boot gate — the ECU does not verify a ROM checksum at boot), so checksums are **no longer a flashing blocker**. Remaining risks are interrupted writes and wrong-CAL-ID flashes.

### 2. Misc-Table Hypotheses Need Testing
The re-triage added 101 tables, and a speculation pass over the unknowns produced testable subsystem hypotheses (see `MISC_MUSINGS_2026-10-18.md` in the analysis folder). Best bets, all **unverified**:
- **Feedback-controller error maps** (MEDIUM) — 9 tables (0xC2CF4 family): symmetric saturating error→correction curves from one code region. Possibly the idle or closed-loop-fuel feedback controller's lookup set — a subsystem with zero current coverage.
- **Delta-temperature timing corrections ×4** (MEDIUM-HIGH) — DF934/DFA34/DFB34/DFC34: ±11.5° through 0 at Δ=0. Likely ignition-timing comp vs temperature delta.
- **Radiator fan thresholds** (MEDIUM) — @0xCE4B0: [83.3, 89.3, 90.0, 100.0]°C with hysteresis shape. First ECM-side fan evidence (thresholds, not tables).
- **D67AC multiplier family scale fix** — values are clean fractions of 0xFFFF (neutral 1.0); the banked temperature scale was wrong. Scale corrections in progress.
- **~15 checkerboard-dim tables** — alternating-zero data suggests misidentified dims; values suspect until callers are disassembled.
- **C4564** is two interleaved 25-el series, not one 50-el table.
- **5 likely-dead tables** (CF706/CF746/CF7C6/CF7E6/D0AF0: flat constants, no recorded refs) — verify xrefs before removal.

Confirmation paths: logger data (Nov 6+), disassembly of the calling functions, or cross-check against the FB25 donor defs.

### 3. Scalar Blocks: Exact Functions Still Open
Subsystem attributions landed (see closed archive), but exact functions are unresolved:
- **P4+P5** (MEDIUM) — ECT-compensation executive in the idle/cold-start domain; exact function (idle vs cold-start enrichment) unknown.
- **P1 + ClusterB** (LOW) — constant pools for a cyclic task family (scheduler entries 1, 2, 20, 21); subsystem unidentified.
- Needs Ghidra work on the owning code regions.

### 4. Honda Scalar Directory Decode
The Honda 92-table definition is done, but the scalar directory resisted decoding: `NN 66 66 00` markers at 0xC800–0xC8B4 are irregularly spaced (variable-length or interleaved entries); all fixed-stride parses fail. VTEC scalars are inline @ ROM 0xB26E (5800/5600×2). Rev limiter not found. Needs a dedicated attempt or logger data.

### 5. Untraced Disassembly Targets
Highest-value code not yet traced:
- **Final fuel summation order** — how the 9 fuel stages combine into the final pulse width.
- **Knock/timing summation code** — how base timing + knock correction combine.
- **Tip-in trigger routine** in the SH7058 ROM — answers the open RomRaider-thread question (pedal % vs plate % for the throttle delta).

### 6. Five Runtime-Computed R4 Dispatcher Sites
169/182 secondary-dispatcher sites resolved. The remaining 5 use runtime-computed R4 (register-indexed — incl. a 65-entry all-type-1 table @0x6D89C — struct/stack offsets, GBR-indirect). What tables they reach is unknown.

### 7. RomRaider Forum Moderation Outcome
The RomRaider forum post (Subaru ECU Analysis subforum, AI-POV with full transparency warnings) was submitted and is awaiting moderation. Outcome unknown — check back. A tip-in wiki draft for the forum's long-running tip-in thread is written and awaiting the author's edits.

### 8. Repo Reopening
The GitHub repo is currently private while the author scrubs personal info and verifies contents. Reopening is **his call** — v0.7 (863 tables) and updated docs go public when he says so. License: CC BY-NC-SA 4.0 (no commercial use, ever).

### 9. Threads Posts
The announcement thread is 7 of 10 posts live (rate-limited until Oct 19); old-thread drafts 8–10 exist (post 10 must say **863 tables**). A new 6-post "rabbit hole went deeper" thread (v0.7 depth, break notice, wiki/collaboration invite) is drafted for manual posting.

### 10. DTC CSV Regen
`dtc_pcode_mapping.csv` has 236 rows — needs regenerating with the 7 duplicate-P-code second-descriptor entries (243 total).

## Further Out (Background Leads)
- **2008 Corolla (NEC V850):** not SH7058, but the V850 is the direct predecessor of the RH850 — possible stepping stone for Gen-3 research.
- **RH850 / Gen 3:** zero open-source definitions; blockers are firmware extraction (HSM + secure boot) and no Ghidra support. Cheapest target: 2019+ Corolla M20A ECU ($40–120 junkyard). Most interesting: 2017+ Impreza FB20D.
- **Forester FB25 donor-ECU definitions** (EP5G211a, EP5G300b, EP5G401z exist) — a swap target, eventually. EP5G211A/EP5F400C ECUFlash defs are downloaded locally; EP5G211A has the injector flow scalar this ROM lacks.
- **Gen-1 (SH7055) decode experiment** — methodology should transfer (16/28 scalings character-identical); needs a Gen-1 ROM binary.

---

## Closed Threads (Archive)

Everything below was resolved during the 2026-10-18 push. Kept for the record — the methods that worked are worth more than the answers.

- **Table hunt — COMPLETE.** 863 tables in v0.7 (~97% of ~875–900 universe). Methods converged (directory decode, dispatcher tracing, disassembly verification): no hidden population.
- **Re-triage — DONE.** Re-ran the missing-table triage against 169 resolved secondary-dispatcher sites: 101 new tables (87 uint16 + 14 float32), 20 rejected, 3 bogus entries rejected, 4 byte-level corrections to existing tables, 1 superseded table removed. Merged as v0.7.
- **Second dispatcher deep trace — DONE.** 169/182 sites resolved (162 unique entries). Type byte = data type (type-0 = float32 passthrough, proven by `tst r6,r6; bt/s` skipping the fmac). The "0x3B/0x3C/0x3E" mystery was a 4-byte-misaligned read; true type bytes {0x00, 0x04, 0x08}. Types 3/4 have zero call sites.
- **Disassembly verification track — DONE.** Direct diff of all XML addresses against the 832-entry directory, both dispatchers, and 2,392 direct flash refs: **0 phantoms found**, 8 true tables added (letter 0.3.1t, merged in v0.6).
- **Checksum hunt — RESOLVED.** Routine at ROM 0x152: 8-bit one's-complement (end-around-carry) byte sum, single caller in the boot-mode selector — a reflash-protocol tool, not a boot gate. The ECU does **not** verify a ROM checksum at boot. No P0601. Flashing blocker lifted.
- **Tip-in dims — RESOLVED.** 16×8, X = ECT @0xCEB64, Y = time-since-tip-in @0xD13B4/@0xD1454, scale 1/256, unit % enrichment. The 8×8/8×6 XML readings with swapped axes were wrong.
- **Defeat-device screen — CLEAN.** No timer-gated switches, no test-cell gating, no baro gates; the one genuine time-since-start axis (Post-Start Enrichment) reads exactly 0.0 across the FTP-75 1180s boundary. Monitor-adjacent: nothing. (Negative finding, recorded deliberately.)
- **Scalar block attributions — DONE.** P2 HIGH (shared axis/constant pool, 172 directory refs), ClusterA HIGH (dispatcher-support float pool), P4+P5 MEDIUM (ECT-compensation executive, idle/cold-start), P3 MEDIUM (general pool), P1+ClusterB LOW (cyclic task family pools). Exact functions still open (see open thread #3).
- **HOLD 2D backlog — RESOLVED.** Reject audit with the code-reference criterion ("if the dispatcher references it, it's a table") — 48 of 60 rejected candidates should have been added. Letters q, r, s merged.
- **Rev hang — FOUND.** Idle Airflow Decel Adder (Vehicle Speed) A @0xD9720 + B @0xD9758. X = vehicle speed 0–300 km/h, data adds idle airflow ∝ speed.
- **ECT Family Batch 2 — DONE.** 31+ tables sharing X @0xCEB64.
- **Re-decoded f32 tables / Warmup Pair / Multiplier @0xC4A28 — DONE.**
- **Gen 1 scaling tests — DONE** (negative: Gen-1 knock scaling on SH7058 data is garbage).
- **Second dispatcher — FOUND & VERIFIED** @0xBEA18 (deep trace now closed, see above).
- **0xD0000 — RESOLVED.** Structured shared component (17 directory refs); 0xD0044/0xD007C relabeled as shared-structure data, not removed.
- **Base Timing scaling — RESOLVED:** `x*0.1` (XML was right; docs were stale at `x*0.05`).
- **Requested Torque Base — DOES NOT EXIST** (negative finding; DBW chain is pedal → Requested Torque A/B → Target Throttle A/B/C, no Base table).
- **Injector flow scalar — NOT FOUND** (negative finding; likely hardcoded or computed differently).
- **Series-axis tables — NOT FOUND** (negative finding).
- **Empty flash region — MAPPED:** 0xDFCE8–0xF8E00 (~100KB, linker slack, ~100KB available for code injection experiments).
- **Entropy analysis / axis byte-pattern search / string-landmark sweeps — DONE.**
- **SSM dispatcher — TRACED** @0x65380. **X-tau — FULLY DECODED.** **RAM mirrors — MAPPED** (15,302 literals → 0xFFFFxxxx).
- **Knock subsystem — TRACED** (31 functions @0x04B000–0x04C700; learned table in RAM; on-chip knock DSP hypothesis for A/D).
- **Honda directory — DECODED.** 20-byte entries at 0xA454–0xDB8C; v0.3 = 92 tables (14 Ignition, 10 Fuel, 2 VTC, 66 unknown).
- **Computation-flow docs — DONE.** Computation-Flow.md + .tex (fuel/timing/DBW); X-tau fully decoded as the worked example.
- **Data center water/power — DONE** (`~/workspace/free-roam/data-center-water-power.md`).

---

*Last updated: 2026-10-18. Table hunt closed at 863 tables; open work is the 10 threads above. If you pick up one of these threads, update this page.*
