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
| **Tables mapped** | **602** |
| **Current full revision** | 0.3 (488 tables) |
| **Letter files** (pending validation) | 0.3.1a (19), 0.3.1b (2), 0.3.1c (4), 0.3.1d (1), 0.3.1e (5), 0.3.1f (5), 0.3.1g (78) |
| **DTC switches mapped** | 236 P-codes |
| **Disassembly** | Ghidra environment working, key subsystems traced |
| **GitHub** | [Livertea/subaru-fb20-sh7058](https://github.com/Livertea/subaru-fb20-sh7058) |

### ⚠️ Validation Status

**This is exploratory work. None of these tables have been validated on real hardware.**

The tables have been validated against the ROM bytes (structure, axes, scaling), and many are confirmed as code-referenced by the firmware (not dead data). But no one has flashed a modified ROM or confirmed behavior on a running engine.

The plan:
1. **Get the cable** → pull the actual ROM from the car
2. **Validate** every table against real hardware via data logging
3. **Only then** consider tuning

**Do not flash anything based on these definitions yet.**

## Maintenance Note

**The original author will likely not be actively maintaining this project going forward.**

This wiki is a snapshot of exploratory research. If you find errors, want to expand the coverage, or want to take over maintenance — please do. That's why it's public. See [Contributing](Contributing.md) for how to help.

---

## How It Was Built

Almost entirely by **AI agents working in parallel**, directed by a human enthusiast. The human had the vision, asked the right questions, caught mistakes, and validated every definition file in RomRaider. The AI did the implementation: reading hex dumps, decoding table structures, cross-referencing definitions, and writing XML.

This is disclosed openly because it matters for verification. The project's standing rule: **trust nothing, verify everything against the raw ROM bytes.**

See [Methodology](Methodology.md) for the full workflow.

## Key Breakthroughs

- **DTC P-code mapping** — All 236 diagnostic trouble code switches mapped via the descriptor table, without disassembly. See [DTC-Mapping](DTC-Mapping.md).
- **Volumetric Efficiency resolved** — 84 VE tables identified (the FB20 is speed-density, not MAF-based like the WRX).
- **Table dispatcher found** — The firmware's central table lookup at `0xBE964` with 479 call sites. See [Ghidra-Guide](Ghidra-Guide.md).
- **X-tau transient fueling** — 12-table system identified, trigger condition found via disassembly.
- **Honda SH7058** — The methodology was applied to a Honda Civic Si ROM, producing the first open-source Honda SH7058 definition. See [Honda-Project](Honda-Project.md).
- **Gen 1 cross-reference** — Compared against SH7055 (2003–2010) definitions: 16 of 28 scaling formulas are character-identical across 8 years. Denso froze the physics, only changed data representation.
- **XmlToIdc-equivalent for Ghidra** — In progress: a script to import all 602 tables as Ghidra labels, making future disassembly dramatically faster.

## Quick Links

- [Definitions](Definitions.md) — Download the XML files
- [Table-Reference](Table-Reference.md) — What's been mapped, by category
- [Methodology](Methodology.md) — How to hunt tables yourself
- [Contributing](Contributing.md) — How to help validate
- [FAQ](FAQ.md) — Common questions

## Project History

| Date | Milestone |
|------|-----------|
| 2026-10-17 | Project started. First table addresses confirmed. |
| 2026-10-18 | DTC P-code mapping cracked (236 codes). |
| 2026-10-18 | Definition v0.1 through v0.2.6 built iteratively. |
| 2026-10-18 | v0.3 released (488 tables). |
| 2026-10-18 | Ghidra disassembly phase begins. Dispatcher, knock, fuel subsystems traced. |
| 2026-10-18 | First open-source Honda SH7058 definition built (v0.1). |
| 2026-10-18 | Letter files 0.3.1d–g released (89 tables). Total: 602. |
| 2026-10-18 | Gen 1 (SH7055) cross-reference complete. 16/28 scalings identical. |
| 2026-10-18 | GitHub repo published: Livertea/subaru-fb20-sh7058 (CC BY-NC-SA 4.0). |
| 2026-10-18 | XmlToIdc-equivalent Ghidra label script in development. |
| 2026-11-06 | *(Planned)* Cable arrives. ROM pull and hardware validation begins. |
