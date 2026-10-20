# Project Log

An honest changelog. Wins, dead ends, and mistakes — including ours. If something broke, it's here.

## 2026-10-17 — Day one

- Project started. First table addresses confirmed on the EP5GD00A ROM (source: anonymous t=19276 thread dump).
- CVT reference addresses verified garbage on the MT ROM. Blind porting abandoned immediately — the decision that shaped everything.

## 2026-10-18 — The build

- **DTC P-code map cracked:** 243 switches via the 20-byte descriptor table. No disassembly needed.
- **v0.3 → v0.7** built iteratively: 488 → 863 tables. Re-triage found 101 tables the first hunt missed (secondary dispatcher type-byte = data type).
- **Second dispatcher found:** @0xBEA18 (type-dispatched, incl. float32 tables).
- **Ghidra phase:** dispatchers, knock/fuel/SSM traces. The four-layer reference rule established (after the VE "dead" verdict was proven wrong by a pointer table).
- **Defeat-device screen:** clean. No timer gates, no test-cell detection.
- **Emulator built:** purpose-built SH-2A interpreter (Unicorn doesn't do SuperH). 95/95 tests.
- **0.8 verified core:** 34 tables, tier-tagged. Letter files 0.8.1a (18), 0.8.2b (72) validated.
- **Repo went public, then private** for personal-info scrub. Reopening after review.
- **FA20 blind experiment:** methodology measured against the public BRZ def — 38/38 axes exact. Porting proven feasible.
- **EP5I200F port:** 69/76 tables (91%) in ~48 min. 2012→2013 calibration byte-identical.

## 2026-10-19 — Tracing, validation, and mistakes

- **Tracing hours 1–3:** 64+ tracers. Timing chain staged end-to-end. Knock 3-mode mapped. Advance formula decoded. Crank chain closed. Scheduler mapped. B3 chain analyzed (kill-3 voided). Dwell defined.
- **0.8.4a sweep:** 47 new tables emulation-verified (26 knock params, 20 idle/DBW, B3). **459-table definition built.**
- **Decel-adder reversal:** the "dead / DO NOT TUNE" verdict refuted — both adders are live. The old zero-refs finding was a buggy mask artifact.
- **First drive logs analyzed:** fuel sane (±2%), WOT enrichment real, decel cut real, 50°C idle switch confirmed live, rev hang caught (plate held 17–20% on lift), cruise timing +7–10° over tables (knock learned-advance hypothesis — falsified 10-19, then refined: mechanism confirmed, cruise attribution IAM-constrained, KNOCKADV).
- **Simulator rebuilt** with 0.8.4a tables and published.

### The 0.8.4a versioning fuckup (10-19, owned)

The file was built as a **full 459-table merge under the letter name 0.8.4a** — violating the project's own rule (letter = new tables only). When this was pointed out, the fix attempt made it worse: an extraction script with a broken indent match **overwrote the file with a 528-byte empty XML**, destroying the only good copy. No backup existed. The user's downloaded copy is the only surviving good version.

Lessons, recorded so they stick:
1. Never run an unverified extraction script against the only copy of a file. Write to temp, validate, then move.
2. Definition builds get a backup before any post-processing. Always.
3. The versioning rule (letter = delta, number = full) is now written into the definition workflow, not just memory.

### Record-keeping debt (10-19, acknowledged)

The project's records grew organically across dozens of agent sessions: stale matrix rows, superseded reports left unmarked, the old wiki behind the research by a full day. This wiki rebuild is the cleanup — a clean base to iterate from, with the Project-Log as the place where the mess gets admitted instead of buried.

## What's next (updated 10-19 evening — the list moves fast)

- **Cable (~Nov 6):** Mode $22 logging (fuel path, idle air, 6640 tiebreak), full ROM SHA-256, KT2 seed/key test. (The G/H/I/J blend revival is dead per 84C0GATE — [84C0] is pinned at 0 by a one-way latch; logging [84C0]+[851C] would only confirm the pin. The TIER11 "scheduler-enable hole" is dead too — TIER11EN: no writer exists anywhere in ROM, IMI11A/B can never fire; the tick-driven scheduler model is refuted.)
- **Tracing continues:** true base-timing table (G/H/I/J, [0xFFFF879C], [0xFFFF87C4] all refuted as carriers — it's somewhere else), [0xFFFF87C4] positive writer, 78 N.m source (refined 10-19: enters via the [0xFFFF8B9C] cap → [0xFFFF8B78] → [0xFFFF8BCC] floor; @0xBD120's computed side refuted at the [8BCC] level; the [8B9C] cap's upstream is the remaining open item), warmup enrichment (still the white whale — WARMENTER proved it's NOT in the mapped fuel chain), the true load-responsive fuel PW computer (FUELMAIN proved the mapped path is a trim path). Closed 10-19: slot-byte writer, knock learned-advance (refuted), rev-hang target, fuel upstream, spark sum equation, boot disarmer, [0xFFFF84C0] writer (@0x46496 — revival killed by 84C0GATE), IGNOUT ignition chain (mapped end-to-end, spark-blind), TIER11 negative (tick-driven scheduler refuted), [0xFFFF6988] writer (boot one-shot, may never fire), CMI10B trigger @0xEA28, [0xFFFF8BCC] writer + 78 chain.
- **Hardware validation:** the line that matters. Nothing is truly validated until a modified flash changes engine behavior.
