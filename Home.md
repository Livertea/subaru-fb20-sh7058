# EP5GD00A: Open-Source ECU Definition for the 2012 Subaru Impreza FB20

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


## What Is This?

This is a ground-up reverse-engineering project to build a **RomRaider tuning definition** for the 2012 Subaru Impreza 2.0i Sport with manual transmission — a car for which no public tuning definition existed.

A "definition" is an XML file that tells RomRaider (the open-source ECU tuning tool) where every adjustable calibration table lives inside the ECU's firmware. Without it, the 1MB binary you read from the ECU is meaningless bytes. With it, you get hundreds of labeled tables — fuel maps, timing maps, rev limiters, idle targets — that you can view, understand, and modify.

## The Car

| | |
|---|---|
| **Vehicle** | 2012 Subaru Impreza Sport Premium (GP chassis, 5-door hatch) |
| **Engine** | FB20 — 2.0L naturally aspirated flat-4 |
| **Transmission** | 5-speed manual |
| **ECU** | Denso SH7058 (Renesas SuperH microcontroller) |
| **CAL ID** | EP5GD00A |
| **ROM size** | 1,048,576 bytes (1024 KB) |

## The Problem

No tuning definition existed for this ECU. The closest public reference was a definition for the CVT (automatic) version of the same car — but its table addresses turned out to be **completely wrong** for the manual transmission ROM. Every address had to be found from scratch.

## Current Status

| Metric | Value |
|--------|-------|
| **Tables mapped** | **863** (627 data tables + 236 DTC switches) |
| **Current full revision** | 0.7 (863 tables) |
| **Verified core** | **0.8** — 34 tables, each tier-tagged (see below) |
| **Letter files** (merged into v0.7) | 0.3.1a (19), 0.3.1b (2), 0.3.1c (4), 0.3.1d (1), 0.3.1e (5), 0.3.1f (5), 0.3.1g (78), 0.3.1h–t (165), 0.3.1u (101) |
| **Letter files** (pending approval) | 0.7.1a (9 unknown tables), 0.7.1b (2), 0.7.1c (8), 0.8.1a (18 sweep-validated) |
| **DTC switches mapped** | 243 P-codes (CSV regenerated 2026-10-18, incl. 7 duplicate-P-code second descriptors) |
| **Emulation** | Purpose-built SH-2A interpreter, **95/95 unit tests pass** — see [Emulator](Emulator.md) for the full story |
| **GitHub** | [Livertea/subaru-fb20-sh7058](https://github.com/Livertea/subaru-fb20-sh7058) (currently private — reopening after review) |

### The 0.8 Verified Core (2026-10-18)

v0.7 is a map of what's **in** the ROM. 0.8 is the start of a map of what the ECU **uses** — every table in it has been validated by executing the ECU's own table-lookup code in a purpose-built SuperH emulator:

- **EMU-VALIDATED** — table dims, axes, and scaling confirmed by running the real dispatcher code (e.g., injector latency: 5/5 voltages bit-exact)
- **CODE-CONFIRMED** — table referenced by disassembled code paths (rev limiter, fan thresholds, idle targets…)
- **HW-VALIDATED** — confirmed against datalogs from the car (fan thresholds, idle targets)

34 tables so far. 0.8 deliberately excludes ~590 unverified tables, 5 fake knock tables, phantom/decoy tables, and anything whose scaling can't be resolved computationally (VE, D7FEC warmup trim — those need dyno data, not more emulation).

See [Verified-Core-Definition](Verified-Core-Definition.md) and [Emulation-Validation](Emulation-Validation.md).

### ⚠️ Validation Status — the honest version

**No table has been validated on a running engine with a modified flash.** That's the line that matters.

What we *have* done, in increasing order of strength:
1. **ROM-structure validation** — every table verified against raw ROM bytes (dims, axes, scaling, no 0x7FF8 padding collisions).
2. **Code-reference validation** — tables confirmed reachable from disassembled code (the four-layer check: table → record → pointer table → code).
3. **Emulation validation** — the ECU's actual lookup routines executed against the tables; outputs match hand computation.
4. **Datalog validation** — a handful of tables (idle targets, fan thresholds) confirmed against OBD2 logs from the car.

What we have *not* done: flashed anything, or confirmed that changing a table changes engine behavior. Steps 1–3 validate that our *description* of the ROM is correct. Only hardware validation proves the *physical meaning*.

**Do not flash anything based on these definitions yet.**

## Maintenance Note

**The original author is taking a break from active work on this project**, but plans to return for hardware validation once the cable arrives (~Nov 6, 2026).

In the meantime: collaborators wanted. If you find errors, want to expand the coverage, or want to help validate — please do. That's why it's public. See [Collaborate](Collaborate.md) for what's needed most and [Contributing](Contributing.md) for how to help.

---

## How It Was Built

Almost entirely by **AI agents working in parallel**, directed by a human enthusiast. The human had the vision, asked the right questions, caught mistakes, and validated every definition file in RomRaider. The AI did the implementation: reading hex dumps, decoding table structures, cross-referencing definitions, and writing XML.

This is disclosed openly because it matters for verification. The project's standing rule: **trust nothing, verify everything against the raw ROM bytes.**

See [Methodology](Methodology.md) for the full workflow.

## Key Breakthroughs

- **DTC P-code mapping** — All 243 diagnostic trouble code switches mapped via the descriptor table, without disassembly. See [DTC-Mapping](DTC-Mapping.md).
- **Volumetric Efficiency resolved** — 84 VE tables identified (the FB20 is speed-density, not MAF-based like the WRX).
- **Table dispatcher found** — The firmware's central table lookup at `0xBE964` with ~479 call sites, plus a **second** dispatcher at `0xBEA18` (182 sites) whose type byte selects the data type — including float32 tables. See [Ghidra-Guide](Ghidra-Guide.md) and [Second-Dispatcher](Second-Dispatcher.md).
- **X-tau transient fueling** — 12-table system identified, trigger condition found via disassembly.
- **Honda SH7058** — The methodology was applied to a Honda Civic Si ROM, producing an open-source Honda SH7058 definition (92 tables). See [Honda-Project](Honda-Project.md).
- **Gen 1 cross-reference** — Compared against SH7055 (2003–2010) definitions: 16 of 28 scaling formulas are character-identical across 8 years. Denso froze the physics, only changed data representation.
- **Re-triage: 101 more tables** — Re-running the missing-table triage against the fully-resolved secondary-dispatcher call sites found 101 tables the first hunt missed (87 uint16 + 14 float32), merged as v0.7. The hunt used code references, not aesthetics: if the dispatcher references it, it's a table.
- **Tip-in enrichment resolved** — 16×8, X = ECT, Y = time-since-tip-in, scale 1/256, unit % enrichment (not ms). The directory dims were another instance of "the directory lies."
- **Checksum: no boot gate** — The checksum routine (ROM 0x152, 8-bit one's-complement sum) is a reflash-protocol tool, not a boot check. The ECU does not verify a ROM checksum at boot. This was the last flashing blocker; it's lifted.
- **Emulation-based validation** — A purpose-built SH-2A/SH-2A-FPU interpreter (95/95 tests) executes the ECU's real table-lookup code against candidate tables. Injector latency: 5/5 voltages bit-exact. The dispatcher ignores record scale/offset fields (proven, not assumed). Now with taint tracking, RAM symbol table (28 symbols), and log playback. See [Emulator](Emulator.md) and [Emulation-Validation](Emulation-Validation.md).
- **0.8 verified core** — 34 tables that survived emulation/code/datalog validation, tier-tagged by evidence strength. The other ~590 v0.7 tables are *unverified*, not *wrong* — but they're not in 0.8 until they earn it. See [Verified-Core-Definition](Verified-Core-Definition.md).
- **XmlToIdc-equivalent for Ghidra** — In progress: a script to import all 863 tables as Ghidra labels, making future disassembly dramatically faster.

## Quick Links

- [Definitions](Definitions.md) — Download the XML files
- [Table-Reference](Table-Reference.md) — What's been mapped, by category
- [Methodology](Methodology.md) — How to hunt tables yourself
- [FB20-Tuning-Guide](FB20-Tuning-Guide.md) — What tuning can actually do on this engine
- [Porting-Guide](Porting-Guide.md) — How this transfers to other cars
- [Collaborate](Collaborate.md) — What's needed, how to help
- [Contributing](Contributing.md) — How to help validate
- [FAQ](FAQ.md) — Common questions

## Project History

| Date | Milestone |
|------|-----------|
| 2026-10-17 | Project started. First table addresses confirmed. |
| 2026-10-18 | DTC P-code mapping cracked (236 codes). |
| 2026-10-18 | Definition v0.1 through v0.2.6 built iteratively. |
| 2026-10-18 | v0.3 released (488 tables). |
| 2026-10-18 | v0.4 released (628 tables). |
| 2026-10-18 | v0.5 released (755 tables). |
| 2026-10-18 | Ghidra disassembly phase begins. Dispatcher, knock, fuel subsystems traced. |
| 2026-10-18 | Honda SH7058 definition built (v0.1, 2 tables; later v0.3, 92 tables). |
| 2026-10-18 | Letter files 0.3.1a–s released. Total: 755. |
| 2026-10-18 | Letter file 0.3.1t released (8 tables, disassembly-verification finds). |
| 2026-10-18 | v0.6 released (763 tables). Disassembly verification track complete — 0 phantoms, 8 verification tables merged. Table hunt closed. |
| 2026-10-18 | Re-triage: 101 more tables found via secondary-dispatcher call sites (letter 0.3.1u). Type byte = data type discovered (type-0 = float32). |
| 2026-10-18 | Checksum hunt resolved: routine at ROM 0x152, no boot-time enforcement — flashing blocker lifted. |
| 2026-10-18 | v0.7 released (863 tables). 4 byte-level corrections applied (Base Timing B 13×17 @0xCCA3A, CL Fuel Comp C +2 bytes, Target Throttle A +4 bytes, tip-in A/B resolved 16×8). Table hunt remains closed (~97% of ~875–900). |
| 2026-10-18 | Defeat-device screen: clean (no timer-gated switches, no test-cell gating). Misc-table musings pass complete (feedback-controller, delta-T, fan-threshold hypotheses). |
| 2026-10-18 | Gen 1 (SH7055) cross-reference complete. 16/28 scalings identical. |
| 2026-10-18 | GitHub repo published: Livertea/subaru-fb20-sh7058 (CC BY-NC-SA 4.0). |
| 2026-10-18 | Repo pulled private for personal-info scrub; reopening after review. |
| 2026-10-18 | **Emulation project Phase 1 complete.** Purpose-built SH-2A/SH-2A-FPU interpreter (Unicorn doesn't support SuperH); 77/77 unit tests pass. Real dispatcher entry points identified (@0xBEA94 for u16, @0xBE964 for float32 1D); type 1/3 paths proven dead. |
| 2026-10-18 | **Emulator expanded.** 95/95 tests. Taint tracking, RAM symbol table (28 symbols), log playback (`playback.py`), RAM range fix (64KB at 0xFFFF0000+). Bugs fixed: 0x8801 (`cmp/eq #imm,R0`), `fmov.s` RAM routing, missing `not`/`div1`. One reported bug (bf/s 12-bit displacement) refuted as encoding-impossible. See [Emulator](Emulator.md). |
| 2026-10-18 | **0.8 verified-core definition released** (34 tables, tier-tagged: EMU-VALIDATED / CODE-CONFIRMED / HW-VALIDATED). Batch sweep added 18 more via 0.8.1a letter file. |
| 2026-10-18 | **Independent-rediscoveries list compiled** — 22 findings we made first and third parties later confirmed (strongest: AVCS scaling triple-sourced across three chip generations; fan thresholds with six corroborating sources). |
| 2026-10-18 | DTC CSV regenerated: 243/243 rows, 7 duplicate-P-code second descriptors included. |
| 2026-10-18 | Letter files 0.7.1a/b/c released (19 unknown tables, banked for approval). |
| 2026-10-18 | CAN validation scoped as Phase 3 (CSS Electronics skill assessed; CANable ~$40 verdict over CANsub.2's ~€400). |
| 2026-10-18 | XmlToIdc-equivalent Ghidra label script in development. |
| 2026-11-06 | *(Planned)* Cable arrives. ROM pull and hardware validation begins. |
