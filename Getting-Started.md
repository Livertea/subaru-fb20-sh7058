# Getting Started

## What you need

1. **RomRaider** (free, open-source) — the editor that reads the definition XML
2. **A ROM dump** of your ECU — read it with EcuFlash
3. **The definition file** — `RR_EP5GD00A_0.8.4a.xml` (459 tables)
4. **A flashing cable** — see the warning below

## Check your CAL ID first

This definition works **only** on CAL ID **EP5GD00A**: 2012 Impreza 2.0i, FB20, **5-speed manual**. It will not work on the CVT version (different addresses — verified), other model years, or any other CAL ID. Read your CAL ID with RomRaider's ECU Info or any OBD-II scanner before touching anything.

## ⚠️ Cable warning — read this

The recommended budget cable is the **Washinglee OpenPort 2.0 clone** (~$80). Two rules, both non-negotiable:

1. **Use ONLY the driver bundled with the clone.** Installing official Tactrix drivers or software will **destroy the clone** (documented kill switch). Keep the flashing laptop on the bundled driver forever.
2. **Dedicated flashing station.** Use a spare laptop or a frozen VM snapshot for EcuFlash. Never let EcuFlash auto-update with the cable connected.

The author's cable arrives ~Nov 6, 2026. Until hardware validation is done, **do not flash a modified ROM**.

## Loading the definition

1. In RomRaider: `ECU Definitions` → `ECU Definition Manager` → add the XML
2. Open your ROM dump — tables populate by category
3. Letter files (if any) load *alongside* the base definition for per-table validation

## What you can actually tune today

Per the [Liveness-Matrix](Liveness-Matrix.md) — only **CLEAR**-tier systems:

- **Base Timing E/F** — primary ignition tables. Small changes (+2–4° midrange), knock logging mandatory
- **6700 RPM limiter** — raise cautiously, stay ≤7000 (valvetrain)
- **Target Throttle A/B/C** — throttle mapping
- **AVCS intake targets** — closed-loop; verify cam response with logging

Everything else is CAUTION (verify before relying), BLOCKED (no live target found — wait for the cable), or DO NOT TUNE (dead, fake, or protection).

## What you cannot tune yet

- **Fuel** — the live injector PW path is **trim-path-mapped, main computer unidentified 10-19** ([Fuel-System](Fuel-System.md)): the mapped path (@0x426FE) is a trim path, not the main fuel computer. Still not tunable until Mode $22 logging closes the remaining gaps.
- **Warmup enrichment** — no live warmup fuel multiplier found. Drive logs show the ECU pulling −22% STFT during warmup, so the enrichment exists — we just haven't located the table.
- **Rev hang** — mechanism FOUND 10-19 (REVHANG, emulator-verified): a steady ~12–15% pedal boost — the plate holds a constant 11.37% plate-over-pedal offset on lift, no timer or decay. Source is a **~7.8× multiplicative gain inside the pedal orchestrator @0xBB638** (`([r15+0] − 0.84) × 100.0 / 12.81`), **not** the ReqTq-B idle cells (they're 1:1 — the boost is upstream). **Current tuning target: ROM 0xBB968** (the ×100.0 gain pool float); patching to ×12.81 collapses [0xFFFF67A4] 18.46% → 2.37% at the lift point. (@0xBB980 Path 4 is a verified passthrough — REVHANG's own arbitrator attribution was refuted by T67A4.) See [Idle-and-DBW](Idle-and-DBW.md).
- **Exhaust AVCS** — open-loop driven (@0x17F88 duty map, no feedback loop in ROM). Tunable as a feedforward map; don't expect closed-loop correction.

## If something looks wrong

It might be. Check [Avoiding-Hallucinations](Avoiding-Hallucinations.md) for how errors get caught, and [Project-Log](Project-Log.md) for what's already been found and fixed. Report issues with the ROM bytes to back it up.
