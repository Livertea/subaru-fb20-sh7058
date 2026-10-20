# EP5GD00A — Open-Source ECU Definition for the 2012 Subaru Impreza FB20

> **RESEARCH STATUS:** This is active reverse-engineering work, not a finished product. Tables are tagged by evidence tier — **VERIFIED** (emulator/code), **MODEL** (assumption), **OPEN** (unknown). Nothing here has been validated by flashing a modified ROM to a running car. **Do not flash based on this wiki.**
>
> This project was built largely by AI agents under human direction. The standing rule: *"we don't deserve to take any breaks on validation."* Every claim stays open to refutation. Trust nothing, verify everything.

---

## What this is

The first public RomRaider tuning definition for the **2012 Subaru Impreza 2.0i Sport 5MT** (FB20, Denso SH7058, CAL ID **EP5GD00A**). No public definition existed for this ECU — the closest reference (a CVT definition) had addresses that were verified garbage on the manual-transmission ROM, so every table was found from scratch.

A definition is the XML file that tells RomRaider where each calibration table lives in the ECU's 1MB firmware. Without it, a ROM dump is meaningless bytes. With it: timing maps, fuel trims, limiters, idle targets — viewable and editable.

## The car

| | |
|---|---|
| Vehicle | 2012 Subaru Impreza Sport Premium (GP, 5-door hatch) |
| Engine | FB20 — 2.0L NA flat-4 |
| Transmission | 5-speed manual |
| ECU | Denso SH7058 (Renesas SuperH) |
| CAL ID | EP5GD00A |
| CVN | 36214324 (pulled from the actual car via OBD-II Mode $09) |
| ROM | 1,048,576 bytes |

## Current status (10-19-2026)

| | |
|---|---|
| Definition | **0.8.4a — 459 tables** (412 carried + 47 newly verified) |
| Verification | Emulator-validated (purpose-built SH-2A interpreter), code-traced, drive-log cross-checked |
| Liveness framework | Every system tiered: **CLEAR** / **CAUTION** / **BLOCKED** / **DO NOT TUNE** ([Liveness-Matrix](Liveness-Matrix.md)) |
| Simulator | Interactive ECU simulator ([live build](https://muse.ai/s/ecu-simulator-wr6xcgnxmptxsa)) — staged timing chain, 3-mode knock, limiters, AVCS, tip-in |
| Hardware validation | Pending ~Nov 6 (Washinglee OpenPort 2.0 clone + Mode $22 logging) |
| Repo | Private during review; reopening after (CC BY-NC-SA 4.0) |

### What the emulator proved (10-19)

- **Timing chain, staged end-to-end:** Base Timing E/F fmac-blend → protection factors → knock correction → IAM. Advance formula fully decoded (Q16.16 crank degrees).
- **Knock, 3-mode:** feedback / fine-learned / IAM, all 26 parameter tables taint-verified through the emulator.
- **Rev hang, first direct evidence:** drive log caught the DBW plate held at 17–20% on lift with RPM pinned 5+ seconds. The decel adders (long thought dead) are **live**.
- **Fuel PW path: trim path mapped, main computer unidentified 10-19.** The most thoroughly burned-down system in the project — 11+ dead ends. FUELMAIN proved the mapped path (@0x426FE) is a trim path (no load/MAF/RPM term, ~1.7 ms constant); the live output driver @0x435E8 is real, but the load-responsive PW computer is still unidentified. The car fuels correctly, so the path exists; static analysis just hasn't found the main computer. This is honest, not a gap in effort.

## How to use this wiki

- **Want to tune?** Start with [Getting-Started](Getting-Started.md), then [Table-Reference](Table-Reference.md). Respect the liveness tiers — CLEAR means tunable, everything else means wait.
- **Want to understand the ECU?** [Timing-System](Timing-System.md), [Knock-System](Knock-System.md), [Fuel-System](Fuel-System.md), [Idle-and-DBW](Idle-and-DBW.md).
- **Want to verify our work?** [Emulation-Validation](Emulation-Validation.md) and [Methodology](Methodology.md). The [Project-Log](Project-Log.md) records what broke along the way, including our own mistakes.
- **Porting to another ROM?** [Porting-Guide](Porting-Guide.md) — 91% measured on the 2013 MT ROM.

## Evidence tiers

Every claim in this wiki carries one of three tags:

- **VERIFIED** — confirmed by emulator execution, disassembly, or drive-log data
- **MODEL** — a stated assumption bridging a gap; falsifiable, flagged as such
- **OPEN** — unknown; no claim made

If a page doesn't tag a claim, treat it as MODEL at best.

## License

CC BY-NC-SA 4.0 — attribution, non-commercial, share-alike. **No commercial use, ever.** The ROM dump that made this possible was one anonymous upload ([RomRaider t=19276](https://www.romraider.com/forum/viewtopic.php?t=19276)); sharing the work back is the point. See [Attributions](Attributions.md).
