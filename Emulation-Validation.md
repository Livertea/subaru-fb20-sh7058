# Emulation-Based Validation

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

## Why Emulate the ECU?

The tables in v0.7 were decoded from ROM bytes and cross-referenced against disassembly. That's strong evidence — but it still leaves a gap: *does the firmware compute what we think it computes from these tables?*

Emulation closes that gap. Instead of reading a table's bytes and guessing the lookup math, we **execute the ECU's actual table-lookup code** on a virtual SH-2A CPU and compare its output against what the definition predicts. If the real code returns the same value our definition says it should, the definition is validated at the computation layer.

This is Phase 2 of the project (see [Methodology](Methodology.md)). The deliverable is the [Verified-Core-Definition](Verified-Core-Definition.md).

**What emulation can prove:** that the firmware computes the value the definition predicts — dims, axes, data type, and interpolation behavior.

**What emulation cannot prove:** that the scaling (raw → physical units) is correct, or that the table is actually *used* by the engine. Those need hardware validation (see below: computation vs. physical).

---

## Why a Purpose-Built Interpreter

The obvious tool for CPU emulation — [Unicorn](https://www.unicorn-engine.org/) — **does not support the SuperH architecture**. No existing emulator covered the SH-2A (the SH7058's core), so the only path was a purpose-built interpreter.

The interpreter lives in `ecu-tools/emu/`:

- **`cpu.py`** (~650 lines): full SH-2A interpreter — integer ops (mov, arithmetic, shifts, branches with delay slots, jsr/rts), FPU (fmov.s, fadd/fsub/fmul/fdiv, fcmp, float/ftrc, fmac, fsqrt), memory (1MB ROM read-only, 64KB RAM), and loud traps (`UnimplementedOpcode`, `EmulationTrap`) on anything it can't execute.
- **`test_cpu.py`**: 77 unit tests covering every implemented opcode with edge cases. **77/77 pass.**
- **`test_accept.py` / `test_accept_extended.py`**: dispatcher acceptance tests.

Two real CPU bugs were found and fixed during development:

1. **`0x4000` misdecoded as NOP** — it's actually `shll r0`. The shift-instruction masks were wrong across the whole family (shll/shlr/shll2/8/16). Fixed by correcting the bit masks.
2. **Branch delay-slot T-flag bug** — `bt`/`bf` were deciding on the T flag *after* the delay slot executed. Real SH hardware decides *before* the delay slot runs (the delay slot can clobber T without affecting the branch). This broke the axis-search routine until fixed.

A CPU audit across 6 sampled ROM regions (dispatchers, callers, handlers) found zero unimplemented-opcode traps — the interpreter covers every instruction family the table/dispatcher code uses.

---

## Phase 1: Acceptance

The goal: prove the interpreter can execute a real ECU table lookup end-to-end, with bit-exact results.

### The convention discovery

The spec initially targeted `@0xBEA18` (the type-dispatching secondary dispatcher), expecting a float result in `fr2`. That was wrong on both counts. Tracing the real injector-latency function (`@0x35A96`) showed the ECU actually calls **`@0xBEA94`** with this convention:

| Register | Meaning |
|----------|---------|
| `r4` | Dispatcher record pointer (28-byte) |
| `fr4` | X input (float) |
| `fr5` | Y input (float) |
| `r0` | **Raw u16 result** (via `ftrc`) |

`@0xBEA94` hardcodes the u16 lookup path (`@0xBEA94` → `@0xBEED0` axis searches → `@0xBEDB0` → `@0xBECA4` data load + interpolation), bypassing the type-dispatch table entirely. The generic `@0xBEA18` is the exception, not the rule.

### Acceptance results

- **Injector latency** (`@0xD26A0`, record `0xB5048`): **5/5 voltages exact u16 match** (13640, 6400, 3776, 2880, 2040).
- **Y-interpolation**: verified on a true 5×5 3D table (record `0xB1928`) — center, bilinear midpoint, and corner all exact.

### Closeout findings

Three entry points validated:

| Entry point | Use | Convention | Status |
|-------------|-----|-----------|--------|
| `@0xBEA94` | u16 2D/3D tables | r4=record, fr4=X, fr5=Y → r0=raw u16 | **Fully validated.** Clamps on both ends. |
| `@0xBE964` | float32 1D tables | r4=12-byte directory, fr4=X → fr2=float | **Fully validated.** Clamps *below* axis min, but **extrapolates above max** (does not clamp high). Phase 2 oracles must replicate this. |
| `@0xBEA18` | Generic type-dispatch | r4=record, fr4=X, fr5=Y → fr2=float | **Partial.** On-axis correct (3/3); off-axis wrong due to r5/r6 register pollution in the dispatch sequence. Use the type-specific wrappers instead. |

**Type-specific wrapper family:** the ECU hardcodes lookup paths rather than using generic dispatch. `@0xBEA94`→`@0xBEDB0` (u16/type 2) is the validated one. Type 1 and type 3 paths are **dead** — handlers exist in the dispatch table `@0xBEA48` but there are zero valid type-1/type-3 records in the ROM. Type 4 (177 records) works through the generic `@0xBEA18` on-axis. A later sweep found that the record's **type byte is advisory only** — wrappers hardcode their handlers and ignore it (`@0xBEA94` works on type-0, type-4-with-u16, and type-8 records alike).

---

## The `tune_diff.py` Tool

Built on the interpreter: validates a proposed tune change through the **real ROM dispatcher code** before flashing.

```bash
python3 tune_diff.py --record 0xB5048 --type u16 \
  --set "0xD26A0[2]=5000" --sweep "6.5:16.5:21"
```

Supports diffing two ROMs (`--rom2 modified.bin`), real-unit patch values (`--cooked`), and 3D sweeps. Scaling is auto-detected from the v0.7 XML by data address.

**Self-test passes:** a +0.2500 ms latency change at X=11.5V shows max delta exactly +0.2500 ms, localized to 9.0V<X<14.0V, zero leakage outside, bit-exact on-axis.

One finding from building it: the injector latency record `@0xB5048` is **5×2, not 5×1** — Y=[−600, 0], both rows byte-identical. Patching a single flat index is invisible to the dispatcher (it reads row 1 at Y=0.0).

---

## Phase 2: Batch Sweeps

With the entry points validated, tables were swept in batches through the emulated dispatchers. Each table: determine entry point from record type, sweep inputs, compare against the v0.7 definition (dims, axes, scaling).

**Batches completed (2026-10-18):**

| Batch | Scope | Result |
|-------|-------|--------|
| Initial sweep | 22 candidate tables | 18 EMU-VALIDATED; D7FEC and VE scaling unresolvable (see below) |
| A — Knock | 26 knock parameter tables | No dispatcher records — they're **direct-read constants**, not interpolated maps. 20 ROM-validated, 5 newly documented, 1 data mismatch corrected (Min Load A: ROM says 0.48, report said 0.45). Discovered mode-entry/exit **hysteresis pairs** ([0.48, 0.50] etc.) the disassembly report missed. |
| B — Fuel | 14 fuel-path tables | 4 EMU-VALIDATED. **Two v0.7 bugs:** Fuel Multiplier A has 7 phantom rows (13×22 → record says 13×15); CL Fueling Target's AFR scaling produces impossible values (0.00 AFR) — it's a correction multiplier (0.64–1.00), not AFR. Fuel Multiplier B is likely a phantom (its data sits *inside* Fuel Mult A's region). |
| C — Idle/Limiter/DBW | 31 tables | 23 EMU-VALIDATED, 3 code-confirmed. **v0.7 bug:** Requested Torque B copied FA08C's axes — the record points to different, correct ones (emulation proves it). |
| D — Types + VE | Type 1/3/4 paths + 84 VE tables | Types 1/3 dead (zero records). **84/84 VE tables structure-validated** (42 std 12×11 + 42 hi 12×18, all dims/axes/data types match). Scaling unresolvable. |

**Total: 34 tables in the verified core** (see [Verified-Core-Definition](Verified-Core-Definition.md)).

### Key methodological findings

1. **The dispatcher ignores record scale/offset fields.** Proven by D7FEC and VE: the record carries scale values (e.g. 0.3515625, 0.0078125) that the dispatcher code never applies. They're descriptive metadata, not code. Scaling disputes are physical questions, not computational ones.
2. **Type 4 ≠ u8.** VE tables are type 4 with u16 data. Don't assume type→width mapping.
3. **Emulation only applies to dispatcher-served tables.** Scalar parameters and small threshold arrays (knock system) bypass the dispatcher entirely — for these, ROM data validation + code-reference confirmation is the right method, not emulation. Future batches pre-screen for records first.

---

## What Emulation Can and Can't Prove

**The emulator answers: "what does the code compute?"** A separate question — "what does the engine do?" — needs the cable (datalogs) and eventually a dyno. Together they separate computation errors from physical misunderstandings.

Two cases proved the boundary:

- **D7FEC scaling**: the dispatcher returns raw u8 values (6, 23, 28 confirmed). The record's scale (0.3515625) is not applied. Whether the table means ×0.0390625 or ×0.3515625 is a *physical* question — only a log or dyno can answer it. Marked SCALE_UNRESOLVED, not promoted.
- **VE scaling**: record scale 0.0078125 → 104–180% VE; v0.7 scale → 37–73%. Neither is plausible for an NA FB20. Structure validated, scaling needs MAF/dyno cross-validation. Not promoted.

**Definition of EMU-VALIDATED:** the table's address, dimensions, axes, and scaling match the emulated dispatcher's output within 1% across sampled inputs (in practice: exact match in all tests to date). It does **not** prove physical correctness — that's what hardware validation is for.

---

## Files

- `ecu-tools/emu/cpu.py` — the interpreter
- `ecu-tools/emu/test_cpu.py` — 77 unit tests
- `ecu-tools/tune_diff.py` — pre-flash tune validation tool
- Analysis reports: `EMU_PHASE1_2026-10-18.md`, `BATCH_SWEEP_2026-10-18.md`, `KNOCK_VALIDATION_2026-10-18.md`, `FUEL_VALIDATION_2026-10-18.md`, `IDLE_LIMITER_DBW_VALIDATION_2026-10-18.md`, `TYPE134_VE_VALIDATION_2026-10-18.md` (in the project archive)
