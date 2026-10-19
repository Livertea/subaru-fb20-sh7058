# The Verified Core (v0.8)

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

## The Concept

v0.7 is a map of what's *in the ROM* — 863 tables decoded from bytes and cross-referenced against disassembly. But "in the ROM" and "correctly described" aren't the same thing. The batch sweeps kept finding real bugs in v0.7: phantom rows, copied axes, wrong scalings (see [Emulation-Validation](Emulation-Validation.md)).

The **verified core** (v0.8) is a separate definition containing **only tables with evidence behind them**. Each table carries a tier tag saying exactly what kind of evidence. If a table isn't verified, it's not in v0.8 — even if it's in v0.7.

This is designed to grow: Phase 2 sweep results merge in as they're validated.

**Current: 34 tables** (16 initial + 18 from the first batch sweep).

---

## Verification Tiers

| Tier | Meaning | Evidence |
|------|---------|----------|
| **EMU-VALIDATED** | Executed through the emulated SH-2A dispatcher; dims, axes, and scaling match the ROM's computed output within 1% (in practice: exact) | [Emulation-Validation](Emulation-Validation.md) |
| **CODE-CONFIRMED** | Address appears in disassembled code as a referenced lookup (traced call chain, not just a dispatcher record) | Disassembly campaign reports |
| **HW-VALIDATED** | Values confirmed against real datalogs from the vehicle | Warmup/drive logs |
| **ROM-VALIDATED** | Direct ROM reads match expected values + structure checks (for tables with no dispatcher record — direct-read constants, not interpolated maps) | Batch validation reports |

Tiers are orthogonal evidence lines. A table that is both EMU-VALIDATED and independently rediscovered (see below) has the strongest confidence in the framework.

**Deliberately excluded from 0.8:** the 611 unverified v0.7 tables, the 5 fake KCA tables, the Base Timing A phantom, dead tables (MAF Limit, Baro Comp), DBW decoys, VE tables (scaling unresolved), and knock tables not yet catalogued.

---

## What's In It

**EMU-VALIDATED (initial 4):** Injector Latency `@0xD26A0`, MAF Scaling `@0xDCEA0`, Warmup Enrichment `@0xC04C0`, Unknown 3D `@0xC2F10`.

**CODE-CONFIRMED (12):** Rev Limit A/B, Speed Limit Enable/Disable, Fan thresholds `@0xCE4B0` + 2 companions, Tip-in Comp A/C/J, Idle Targets K/M.

**HW-VALIDATED (4, overlapping):** Fan thresholds, idle targets — confirmed against the warmup log.

**Batch sweep (+18 EMU-VALIDATED):** Unknown 16×3 `@0xD335C` (corrected scaling `(x*1.5259e-05)-0.5` confirmed via real code: 0.120 @ −40°C → 0.000 @ 40°C), Unknown 1×1 Cruise/Non-Cruise `@0xD496C`/`@0xD4960`, 7 unknown u8 tables from the 0.7.1c letter, Base Timing E/F, OL Fueling Cruise/Non-Cruise, AVCS Intake/Exhaust Cruise/Non-Cruise.

**Not promoted (scaling unresolvable by emulation):** D7FEC (structure confirmed, scaling is a physical question) and all 84 VE tables (structure 84/84 validated, scaling needs MAF/dyno data).

The full per-table evidence is in `RR_EP5GD00A_0.8_VERIFICATION.md`, shipped alongside the XML.

---

## Letter-File Versioning

The project versions definitions as semantic `X.X.X` (0 = in development). Standalone iteration files get a **new letter** each iteration — old files are never renamed or overwritten.

- **0.8** — the verified-core definition (34 tables).
- **0.8.1a** — letter file with the 18 sweep-validated tables extracted standalone, for review before merging.

New tables always go in a letter file for approval before merging into a numbered release. See [Methodology](Methodology.md) for the workflow rules.

---

## Independent Rediscoveries: Cross-Validation Against the Outside World

Emulation validates *computation*. A separate, orthogonal check: **did our independent reverse-engineering rediscover things that third-party sources confirm?** When a methodology rediscovers ground truth it never saw, that validates the methodology itself.

22 such cases are documented (strict rule: *we* found it first via disassembly/ROM analysis/logs → a *third party* confirmed it later via FSM, forums, published definitions, or Gen 1 cross-reference). Examples:

- **Fan thresholds** (90.0/83.3, 100.0/89.3°C) — found by disassembling the fan routine; confirmed by FSM thermostat specs, a WRX tech anecdote (100°C exactly), and the warmup log stabilizing at precisely 90°C.
- **AVCS scaling** (360/65535) — resolved from the ROM; confirmed byte-identical in published FA20 definitions and 2005–06 Legacy defs — three Denso chip generations.
- **Base timing scaling** (×0.1) — confirmed from ROM; the FSM's 14–16° idle window contains our table's 14.9° value and the log's 14.2°.
- **Injector latency scaling** (×0.00025) — found in our ROM; character-identical in Gen 1 SH7055 definitions, ~8 years apart. Denso froze the math.
- **Idle targets** (650 warm / 1800 cold, 50°C switch) — confirmed by FSM spec and the warmup log within 1%.

When Phase 2 emu-validates a table, cross-check it against the rediscoveries list. **A table that is both EMU-VALIDATED and independently rediscovered has two orthogonal evidence lines — the strongest confidence tier in the framework.**

---

## Files

- `RR_EP5GD00A_0.8.xml` — the verified-core definition
- `RR_EP5GD00A_0.8_VERIFICATION.md` — per-table evidence
- `RR_EP5GD00A_0.8.1a.xml` + `_README.md` — letter file (18 sweep tables, standalone)
