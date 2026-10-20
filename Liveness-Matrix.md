# Liveness Matrix

## The framework

A table is only tunable if all three levels hold:

- **L1 — Table live:** code references the table (direct, dispatcher record, pointer table, or code referencing the pointer table — all four layers checked)
- **L2 — Computation live:** the using function actually runs (callers, task scheduling, or emulator proof)
- **L3 — Output consumed:** the result reaches an actuator or the next live stage

**Verdicts:** **CLEAR** (tune now) · **CAUTION** (live but verify before relying) · **BLOCKED** (no live target — needs the cable) · **DO NOT TUNE** (dead, fake, or protection).

> **Corrections to the 10-19 03:02 matrix** (applied here): the decel adders are **LIVE** (refutes the old DO NOT TUNE — proven via @0x5BF46 10-19); the dwell table is **definition-complete** (not dead); exhaust AVCS is **open-loop driven** (@0x17F88), not zero-reader dead — tunable as a feedforward map.

## The matrix (current)

| System | L1 | L2 | L3 | Verdict |
|---|---|---|---|---|
| Timing — Base E/F | ✓ | ✓ | ✓* | **CLEAR** |
| Timing — protection factors | ✓ | ✓ | ? | **CAUTION** |
| Timing — B3 high-RPM | ✓ | ✓ | ✗ (inert) | Computed-but-discarded |
| Timing — dwell | ✓ | ✓ | ✓ | Definition-complete (Pφ' = 10 MHz grounded 10-19; dwell computer @0x4C908 orphaned — no periodic caller) |
| Fuel — all paths | ✗/partial | ✗ | ✗ | **BLOCKED** |
| Knock — 26 params | ✓ | ✓ | ? | **CAUTION** |
| Knock — 5 fake KCA | — | — | — | **DO NOT TUNE** (fake) |
| Idle — airflow tables | ✓ | ✓ | ? | **CAUTION** |
| Idle — decel adders | ✓ | ✓ | ✓ | **CAUTION** (live; scaling unverified) |
| AVCS — intake | ✓ | ✓ | ✓ (hw) | **CLEAR** |
| AVCS — exhaust | ✓ | ✓ | ✓ (open-loop) | **CAUTION** (feedforward, no feedback) |
| Throttle — TT-A/B/C | ✓ | ✓ | ✓ | **CLEAR** |
| Limiters — 6700 | ✓ | ✓ | ✓ | **CLEAR** |
| Limiters — 4400 diag | ✓ | ✓ | ✓ | **DO NOT TOUCH** |
| Decel fuel cut | ? | ✓ | ✓ (log) | **BLOCKED** (trigger unmapped; cut confirmed by log) |
| DTC infrastructure | ✓ | ✓ | ✓ | Infrastructure |

\* Base E/F L3 is a qualified pass: the staged chain is continuous but the [0xFFFFA238]→[0xFFFF87C8] hop is unmapped. The terminal IGT hop is **documented (IGNOUT + DWELLFLOW 10-19):** @0x5B00 is a CAN scheduler (not ignition — decoder-bug claim overturned), the crank path doesn't drive spark, and the mapped chain @0x4C908 → @0x8714 → @0x8B2A → @0x8D94 → GR2x/OCR2x is dwell-driven and spark-blind — no spark-advance value reaches any compare register through any ROM-visible path. Dwell drives the compare registers directly: the **@0x8C48 tail writes `u16[OCR2x] = C + mirror/2`** (linear, slope 1/2, all 4 cylinders, emulator-validated — overturning the old "never dwell-derived" negative). **But the output stage is boot-only (DWELLTRIG 10-19):** no periodic runtime trigger exists; the one dispatch chain that could reach it can't select walker B (@0x4C908), the sole caller of the dwell setter — walker B is dead code, and the no-hidden-driver negative is now triple-sourced (bsr + jsr + RAM-table censuses, REGION9XXX/JSRCLEAN).

## Per-system notes

**Timing — Base E/F (CLEAR).** Records 0xB38B4/0xB38D0 → @0xBEA18 lookups → fmac blend → [0xFFFFA238]. **VERIFIED.** The drive-log timing gap (7–10° above tables at cruise) was hypothesized as +8.0° fine knock-learned advance — **CONSTRAINED 10-19 (KNOCKADV):** the +8° learning mechanism is CONFIRMED real, but the cruise attribution is IAM-constrained — the positive authority is **(1−IAM)×[0xFFFF87C4]** enforced every knock cycle by @0x4C022, so the gap can be [0xFFFF8800] only if IAM<1 at cruise. (The 20.18° "emulated cruise total" was a cold-RAM transient — steady state [8460]=−46.5° — and [8460..846C] have zero ROM-wide readers per SPARKOUT, so the emulated side is closed negative.) The gap's true source is still open (see [Drive-Log-Findings](Drive-Log-Findings.md)).

**Knock — 26 params (CAUTION).** All taint-verified 10-19. Three-mode: feedback (Range A 1100–7000, −5.0° max), fine-learned (Range B 1500–3600, limits +8.0/−7.0), IAM (Range C 1300–4400, init 0.50). Safe to adjust; **never disable knock.** The knock→timing path is fully mapped: [0xFFFF8798] consumer found and downstream-traced (emu-validated); [0xFFFF8808]→[0xFFFF879C] taint confirmed; true spark sum @0x4ADCE: [0xFFFF8798] = [0xFFFF879C] + [0xFFFF87C8] + [0xFFFF8800].

**Fuel (CAUTION — trim path mapped, main computer unidentified 10-19).** The most burned-down system in the project. Dead with certainty: @0x42700, @0x43F64, ATU-II injector driver, HI7000 fuel params, Injector Correction A–D, Injector Latency. Refined 10-19 (FUELMAIN): the mapped path (@0x426FE) is a **trim path** — no load/MAF/RPM term, ~1.7 ms constant output — not the main fuel computer, which is **still unidentified**. The live output driver @0x435E8 is real (emulator-confirmed), and [0xFFFF82D0] has a second reader (@0x175BC, AVCS/airflow). The car demonstrably fuels correctly (STFT ±2%, WOT lambda 0.83–0.91). Still not tunable; Mode $22 logging is the way forward for the remaining gaps.

**Idle — decel adders (CAUTION, newly live).** Both Idle Airflow Decel Adders (@0xD9720/@0xD9758) are looked up by live idle code (@0x5BF46) via the dispatcher with consumed outputs — proven 10-19, refuting the old "dead" verdict (the zero-refs finding was a buggy `mov.l` mask artifact). Scaling unverified (raw values). Prime suspects in the rev-hang mechanism.

**AVCS.** Intake is closed-loop P-only (`clamp(err/1.6+0.5, 0, 255)`) — tunable, verify with logging. Exhaust is **open-loop** (duty = @0x17F88 map, no feedback loop in ROM) — tunable as a feedforward map; don't expect closed-loop correction.

**Decel fuel cut (BLOCKED trigger, confirmed behavior).** Drive logs confirm the cut (lambda 1.38, open-loop, 323 rows). The triggering flag/thresholds remain unmapped — do not tune the cut itself.

## What unlocks the rest

The Nov 6 cable with Mode $22 logging: [0xFFFF82D0] (fuel), [0xFFFF8EE4] (idle air), [0xFFFF6640] (IAT-vs-ECT tiebreak), [0xFFFF8888] (advance slot byte), exhaust AVCS cells, plus a full ROM SHA-256 against the thread dump.
