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

**Update 2026-10-18 (emulation):** a purpose-built SH-2A emulator now validates table *computation* pre-flash (0.8 verified core, 34 tables). Emulation answers "what does the code compute"; the cable answers "what does the engine do." Both are needed — emulation can't prove physical meaning.

### 2. Scaling Questions Emulation Can't Answer (Need Dyno/MAF Data)
Two tables are structurally validated but their scaling is **unresolvable by emulation** — the dispatcher returns raw values and ignores record scale/offset fields, so both candidate scalings are computationally consistent:
- **VE tables (84):** record scale 0.0078125 → 104–180% VE; v0.7 scale → 37–73%. Neither is physically plausible. Needs MAF-correlation or dyno data.
- **D7FEC warmup trim:** record scale 0.3515625 vs alternative 0.0390625 — a 9× difference, unresolvable without logged data. Structure + Y-selector confirmed; scaling is a physical question.

These stay out of 0.8 until hardware data arrives.

### 3. Throttle Mechanism — Partially Resolved (DBW Overturn 2026-10-18)
**Update 2026-10-18:** the "decoy" verdict was **overturned**. The real Target Throttle function is `@0xBD65C` (found via indirect-call tracing the original campaign missed). TT-A/B/C are **CODE-CONFIRMED live and tunable** — `@0xBD65C` loads all three records and dispatches each via `@0xBEA18`, called 3× with r4=0/1/2. Requested torque RAM stays write-only (diagnostic output hypothesis holds).
**Still open:** exact X/Y inputs to TT lookups (traced: X=pedal @0xFFFF8B70, Y=RPM @0xFFFF6B00 for r4=0; r4=1/2 use fixed constants), full output routing past @0xFFFF8C6C, what r4=0/1/2 selects, throttle servo driver still unlocated.

### 4. Knock Final Summation Point
The knock subsystem is traced (feedback/fine/rough modes, IAM, retard limits) but the **final summation point** — where base timing + knock correction combine into the actual ignition command — was not found. IAM update mechanism unknown (3 hypotheses). Knock threshold is a hardware comparator, not a ROM table.

### 5. Driver Register IDs (Injector / Throttle / AVCS)
- **Injectors:** ATU-II Ch 2 registers identified (GR2A-D @0xFFFFF604/606/608/60A, OCR2A-D @0xFFFFF614/616/618/61A, TCNT2B @0xFFFFF602; config table @ ROM 0x110F4) — the actual writer not found.
- **Ignition:** ATU-II, registers unidentified (MEDIUM priority).
- **AVCS:** P-only controller confirmed, duty RAM @0xFFFF5E18, exact PWM write not found.
- **Memory-map correction:** peripherals live at 0xFFFFC000+, not 0xFFFF8000+ — knock (0xFFFF87xx) and SCI (0xFFFF80xx) attributions need re-examination.

### 6. Idle Mode Writer
Idle selector traced (@0x575BA, mode byte @0xFFFF8E7C, ECT switch at exactly 50°C) but the **mode writer was not found** (exhaustive search of 3,191 GBR-relative stores came up empty).

### 7. @0xCD464 Dims
13×17 u16 with 91% 0x7FF8 padding; ~20 real words; 1 call site @0x82842. Verdict: dims stale, real table ~5×4. Decision pending.

### 8. CAN Validation (Phase 3)
Scoped 2026-10-18: CAN as a third validation leg (emulation = what the code computes, OBD2 = what the engine does, CAN = what the ECU broadcasts). 12 experiments in 3 tiers; the money experiment is catching the **fan-state frame flipping at 90.0°C** during parked idle — third independent confirmation of fan thresholds. Hardware verdict: CANable (~$40 all-in), not CANsub.2 (~€400). CSS Electronics' MIT-licensed reverse-engineering skill works over python-can on the CANable. Timing: study the skill now, one instrumented drive after the Nov 6 cable (pairs with the planned highway run).

### 9. Misc-Table Hypotheses Need Testing
The re-triage added 101 tables, and a speculation pass over the unknowns produced testable subsystem hypotheses (see `MISC_MUSINGS_2026-10-18.md` in the analysis folder). Best bets, all **unverified**:
- **Feedback-controller error maps** (MEDIUM) — 9 tables (0xC2CF4 family): symmetric saturating error→correction curves from one code region. Possibly the idle or closed-loop-fuel feedback controller's lookup set — a subsystem with zero current coverage.
- **Delta-temperature timing corrections ×4** (MEDIUM-HIGH) — DF934/DFA34/DFB34/DFC34: ±11.5° through 0 at Δ=0. Likely ignition-timing comp vs temperature delta.
- **Radiator fan thresholds** (MEDIUM) — @0xCE4B0: [83.3, 89.3, 90.0, 100.0]°C with hysteresis shape. First ECM-side fan evidence (thresholds, not tables).
- **D67AC multiplier family scale fix** — values are clean fractions of 0xFFFF (neutral 1.0); the banked temperature scale was wrong. Scale corrections in progress.
- **~15 checkerboard-dim tables** — alternating-zero data suggests misidentified dims; values suspect until callers are disassembled.
- **C4564** is two interleaved 25-el series, not one 50-el table.
- **5 likely-dead tables** (CF706/CF746/CF7C6/CF7E6/D0AF0: flat constants, no recorded refs) — verify xrefs before removal.

Confirmation paths: logger data (Nov 6+), disassembly of the calling functions, or cross-check against the FB25 donor defs.

### 10. Scalar Blocks: Exact Functions Still Open
Subsystem attributions landed (see closed archive), but exact functions are unresolved:
- **P4+P5** (MEDIUM) — ECT-compensation executive in the idle/cold-start domain; exact function (idle vs cold-start enrichment) unknown.
- **P1 + ClusterB** (LOW) — constant pools for a cyclic task family (scheduler entries 1, 2, 20, 21); subsystem unidentified.
- Needs Ghidra work on the owning code regions.

### 11. Honda Scalar Directory Decode
The Honda 92-table definition is done, but the scalar directory resisted decoding: `NN 66 66 00` markers at 0xC800–0xC8B4 are irregularly spaced (variable-length or interleaved entries); all fixed-stride parses fail. VTEC scalars are inline @ ROM 0xB26E (5800/5600×2). Rev limiter not found. Needs a dedicated attempt or logger data.

### 12. Untraced Disassembly Targets
Highest-value code not yet traced:
- **Final fuel summation order** — how the 9 fuel stages combine into the final pulse width.
- **Knock/timing summation code** — how base timing + knock correction combine.
- **Tip-in trigger routine** in the SH7058 ROM — answers the open RomRaider-thread question (pedal % vs plate % for the throttle delta).

### 13. Five Runtime-Computed R4 Dispatcher Sites
169/182 secondary-dispatcher sites resolved. The remaining 5 use runtime-computed R4 (register-indexed — incl. a 65-entry all-type-1 table @0x6D89C — struct/stack offsets, GBR-indirect). What tables they reach is unknown.

### 14. RomRaider Forum Moderation Outcome
The RomRaider forum post (Subaru ECU Analysis subforum, AI-POV with full transparency warnings) was submitted and is awaiting moderation. Outcome unknown — check back. A tip-in wiki draft for the forum's long-running tip-in thread is written and awaiting the author's edits.

### 15. Repo Reopening
The GitHub repo is currently private while the author scrubs personal info and verifies contents. Reopening is **his call** — v0.7 (863 tables) and updated docs go public when he says so. License: CC BY-NC-SA 4.0 (no commercial use, ever).

### 16. Threads Posts
The announcement thread is 7 of 10 posts live (rate-limited until Oct 19); old-thread drafts 8–10 exist (post 10 must say **863 tables**). A new 6-post "rabbit hole went deeper" thread (v0.7 depth, break notice, wiki/collaboration invite) is drafted for manual posting.

### 17. Emulator Open Work
The emulator is functional (95/95 tests) but has known gaps — see [Emulator](Emulator.md) for the full roadmap:
- **Invisible writers:** engine load (`0xFFFF66E4`, 69 readers) and RPM (`0xFFFF6988`, 246 readers) have zero identifiable writers through any standard store pattern. Likely computed addresses, ISRs, or DMA. The load formula (g/s→g/rev) is still an inference.
- **No peripherals yet:** ADC, timer units, CAN, and interrupts are unmodeled. Can't boot the firmware or run the scheduler — every invocation needs hand-configured RAM.
- **0x8801 resolved as `cmp/eq #imm,R0`** (a competing BLD hypothesis was refuted by immediate-distribution analysis), but the SH-2A manual should confirm the encoding.
- **Rev limiter selection resolved:** no ECT-based mux exists — the three limiters (6700/6600, 5600/5500, 6500/6400) are independent scheduler functions, not a selection. The original "two RPM limits by ECT" hypothesis is dead.
- **Queued:** peripheral stubs, scheduler execution, snapshot/restore, trace recording, interactive debugger, coverage tracking, differential testing vs Ghidra, symbolic execution, performance work.

## Further Out (Background Leads)
- **2008 Corolla (NEC V850):** not SH7058, but the V850 is the direct predecessor of the RH850 — possible stepping stone for Gen-3 research.
- **RH850 / Gen 3:** zero open-source definitions; blockers are firmware extraction (HSM + secure boot) and no Ghidra support. Cheapest target: 2019+ Corolla M20A ECU ($40–120 junkyard). Most interesting: 2017+ Impreza FB20D.
- **Forester FB25 donor-ECU definitions** (EP5G211a, EP5G300b, EP5G401z exist) — a swap target, eventually. EP5G211A/EP5F400C ECUFlash defs are downloaded locally; EP5G211A has the injector flow scalar this ROM lacks.
- **Gen-1 (SH7055) decode experiment** — methodology should transfer (16/28 scalings character-identical); needs a Gen-1 ROM binary.

---

## Closed Threads (Archive)

Everything below was resolved during the 2026-10-18 push. Kept for the record — the methods that worked are worth more than the answers.

- **DTC CSV regen — DONE (2026-10-18).** Regenerated directly from the ROM (descriptor table @0xA1431 × switch bank @0xA1338): 243 rows, 0 mismatches against the prior 236; 7 new rows are the duplicate-P-code second descriptors.
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

*Last updated: 2026-10-18. Table hunt closed at 863 tables; open work is the 16 threads above. If you pick up one of these threads, update this page.*
