# FAQ

## Is this safe to flash?

**No.** Nothing here has been validated by flashing a modified ROM to a running car. Tables are emulator- and code-validated, which proves our *description* of the ROM is correct — not that changing a value does what you expect. **Do not flash a modified ROM based on this wiki.** Wait for hardware validation.

## Will this work on my car?

As shipped, only **EP5GD00A** (2012 Impreza 2.0i, FB20, 5MT) — check with RomRaider's ECU Info or any OBD-II scanner. **Addresses never port between ROMs.** But: the 2013 USDM Impreza 2.0i MT ROM (EP5I200F) was ported at **91% in ~48 minutes** with the calibration byte-identical — so other FB20 SH7058 ROMs (2012–2016 Impreza/Crosstrek) are port candidates, not dead ends. See [Porting-Guide](Porting-Guide.md). Not the CVT (untested), not WRX/Forester/BRZ.

## What's the difference between 0.8.4a and 0.8.4?

Letter files (0.8.4a) contain **only the new tables** for validation in RomRaider before merging. Numbered versions (0.8.4) are the full merged definition. If a letter file ever contains old tables, that's a bug — it's in the [Project-Log](Project-Log.md).

## What can I tune right now?

Only **CLEAR**-tier systems ([Liveness-Matrix](Liveness-Matrix.md)): Base Timing E/F, the 6700 limiter, Target Throttle A/B/C, AVCS intake targets. Everything else is CAUTION (verify first), BLOCKED (wait for the cable), or DO NOT TUNE.

## Can I fix rev hang with this?

Yes — the mechanism is a steady ~12–15% pedal boost (plate holds a constant 11.37% plate-over-pedal offset on lift, emulator-verified), sourced from a **~7.8× multiplicative gain inside the pedal orchestrator @0xBB638**. The **tuning target is ROM 0xBB968** (the ×100.0 gain pool float): patching it to ×12.81 collapses [0xFFFF67A4] 18.46% → 2.37% at the lift point (emulator-demonstrated, T67A4 10-19). Note the ReqTq-B idle cells are NOT the target — they're 1:1; the boost is upstream. See [Idle-and-DBW](Idle-and-DBW.md).

## Why is there no fuel tuning?

The live injector PW path is **trim-path-mapped, main computer unidentified 10-19** ([Fuel-System](Fuel-System.md)): FUELMAIN proved the mapped path (@0x426FE) has no load/MAF/RPM term — it's a trim path, not the main fuel computer, which is still unidentified. The live output driver @0x435E8 is real (emulator-confirmed), so the output stage is driven by the trim computation. Eleven-plus dead ends, all documented. The car fuels correctly — the path exists, static analysis just hasn't found the main computer. Mode $22 logging is the way in. Until then, no fuel table is tunable. This is honest, not an oversight.

## Are the knock tables safe to adjust?

The 26 parameters are code-read and safe to *adjust*, but **never disable knock**. Note the 5 fake "Knock Correction Advance" tables are quarantined — they're not knock tables (impossible values, zero references).

## What's "permanent skepticism"?

The project's standing rule: no CONFIRMED/RESOLVED/VALIDATED label is ever final. Every claim stays open to refutation. In one day (10-19) the sweep voided two prior claims (B3 kill-3, decel-adder death). That's the system working.

## Was this really built by AI?

Largely, yes — agents did the implementation under human direction (vision, mistake-catching, RomRaider validation). Disclosed because it affects how you should treat the output: with skepticism, regardless of who did it. **Trust nothing, verify everything.**

## Can I help?

Yes — RomRaider validation needs no cable: load the definition against an EP5GD00A dump, check tables display sane values, report anomalies with ROM bytes to back it up. See [Attributions](Attributions.md) for everyone whose work made this possible.

## License?

CC BY-NC-SA 4.0. Attribution, non-commercial, share-alike. **No commercial use, ever.**
