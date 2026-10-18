# FAQ

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


## Is this safe to flash to my ECU?

**No.** These definitions have not been validated on real hardware. The tables are structurally validated against ROM bytes, but no one has confirmed correct behavior on a running engine.

**Do not flash a modified ROM based on these definitions.** An incorrect definition can cause engine damage, poor running, or a bricked ECU.

Wait for hardware validation results before considering any flashing.

## Will this work on my car?

**Only if your ECU has CAL ID EP5GD00A.** This is specifically for:
- 2012 Subaru Impreza 2.0i (FB20 engine)
- 5-speed manual transmission

It will **not** work on:
- CVT/automatic versions (different CAL ID, different addresses)
- Other model years (even if similar)
- WRX, Forester, Crosstrek, BRZ (different ROMs entirely)
- Any other CAL ID

**How to check your CAL ID:** Read it with RomRaider's ECU Info function or a generic OBD-II scanner that reports CAL ID.

**The iron law:** Addresses never port between ROMs. Even the CVT version of the same car has completely different table addresses.

## Can I help without a cable?

**Yes.** The most valuable contribution right now is RomRaider validation:
1. Load the definitions with an EP5GD00A ROM dump
2. Check that tables display sane values
3. Report anything that looks wrong

See [Contributing](Contributing.md) for details. No hardware required.

## What's the difference between 0.3 and 0.3.1a?

- **0.7** is a full definition (863 tables) — load this as your base
- **0.3.1a** through **0.3.1u** are letter files with only new tables — load these *in addition to* an older base if you want to validate tables individually

Letter files let you validate new tables separately before they're merged into the next full revision. Total across all files: **863 tables**.

## Why are some tables marked "(Unverified)"?

These tables are structurally confirmed (real data, clean axes, code-referenced) but their **identity or scaling** is uncertain. For example:
- We know it's a 13×17 table with timing-shaped data, but we're not sure *which* timing function it serves
- We know the axis is ECT-indexed, but we're not sure of the exact physical meaning

Unverified tables are included for visibility, not for tuning. Don't modify them until their function is confirmed.

## What's "rev hang" and can this fix it?

Rev hang is when the engine holds RPMs after you lift off the throttle to shift. It's caused by idle airflow tables that keep the throttle open during deceleration.

The likely culprits are mapped:
- **Idle Airflow Decel Adder A** @ 0xD4700
- **Idle Airflow Decel Adder B** @ 0xD487C

But: **not validated yet.** Don't change these until logger data confirms the relationship.

## Why 863 tables? My WRX definition has 2,000+.

The WRX definition you're thinking of likely includes:
- Many more DTC switches (we have 243; some defs have 400+)
- Per-gear variants of tables (our NA engine doesn't need these)
- Boost control tables (we're naturally aspirated — no turbo)
- Years of community additions

Our 863 tables (260 3D · 333 2D · 34 1D · 236 Switch) cover the tables the ECU actually references. The dispatcher cross-reference plus the re-triage against fully-resolved call sites found tables missed through directory analysis alone — most are now merged. The WRX simply has a more complex calibration because it's turbocharged.

## Was this really built by AI?

Mostly, yes. AI agents did the implementation — reading hex dumps, decoding structures, writing XML — working in parallel under human direction. The human provided vision, caught mistakes, asked the right questions, and validated every file in RomRaider.

This is disclosed because it affects how you should treat the output: with the same skepticism you'd apply to any reverse-engineering work, regardless of who (or what) did it. **Trust nothing, verify everything.**

## Can I use this for my Honda/Nissan/Mitsubishi?

Not directly. The **methodology** ([Methodology](Methodology.md)) transfers, but no specific tables, addresses, or scalings will work on a different ROM.

There is a nascent [Honda-Project](Honda-Project.md) applying the same approach to a Civic Si. It's v0.3 (92 tables) — early days, not for tuning.

## Who maintains this?

The original author is unlikely to actively maintain this going forward. The documentation is provided as-is. If you find errors or want to continue the work, you're welcome to — that's why it's public. See [Contributing](Contributing.md).
