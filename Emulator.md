# The SH-2A Emulator

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. The emulator is a tool for validating table *computation*, not a proof of physical correctness. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

## What It Is

A **purpose-built SH-2A CPU interpreter** written in Python, living in `ecu-tools/emu/`. It executes the EP5GD00A's actual firmware code — the real table-lookup routines, the real dispatcher functions — on a virtual SuperH CPU, so we can ask "what does the code compute?" instead of guessing from disassembly.

### Why purpose-built

The obvious tool for CPU emulation — [Unicorn](https://www.unicorn-engine.org/) — **does not support the SuperH architecture**. No existing emulator covered the SH-2A (the SH7058's core). The only path was writing one from scratch.

This was a deliberate engineering decision, not a shortcut. A purpose-built interpreter gives us:
- **Loud failures** — unimplemented opcodes raise traps instead of silently producing garbage
- **Instrumentation hooks** — watchpoints, taint tracking, and symbolic logging are first-class, not bolted on
- **Exact SH-2A semantics** — delay slots, the T flag, FPU behavior, all matched to the hardware manual

### Architecture

| File | Lines | Role |
|------|-------|------|
| `cpu.py` | ~880 | The interpreter — SH2A class, instruction decode/execute, FPU, taint tracking |
| `mem.py` | ~170 | Memory model — 1MB ROM (read-only), 64KB RAM (read-write), peripheral traps |
| `symbols.py` | ~226 | RAM symbol table — 28 known addresses with confidence levels and sources |
| `playback.py` | ~230 | Log playback — feeds real OBD2 CSV logs through the emulator as test vectors |
| `test_cpu.py` | — | 95 unit tests (per-opcode + regression) |
| `test_symbols.py` | — | 21 symbol table tests |
| `test_accept.py` | — | Dispatcher acceptance tests |

**Memory model:**
- **ROM:** 1MB at `0x00000000–0x000FFFFF`, read-only. Writes raise `EmulationTrap`.
- **RAM:** 64KB at `0xFFFF0000–0xFFFFFFFF`, read-write. (This was fixed 2026-10-18 — the original model only covered `0xFFFF8000+` and trapped on every real ECU variable.)
- **Stack:** r15 initialized to `0xFFFFFFFF` (grows down).
- **Peripherals:** NOT implemented. Any access outside ROM/RAM raises `EmulationTrap` — trap, don't stub.

---

## Capabilities

### Table-lookup validation (the primary use)

The emulator's main job: execute the ECU's real dispatcher code against candidate tables and compare outputs.

**Validated entry points:**

| Address | Inputs | Output | Use |
|---------|--------|--------|-----|
| `@0xBEA94` | r4=record, fr4=X, fr5=Y | r0=raw u16 | u16 2D/3D tables. **Fully validated.** Clamps on both ends. |
| `@0xBE964` | r4=12-byte dir, fr4=X | fr2=float | float32 1D tables. **Fully validated.** Clamps *below* axis min, **extrapolates above max**. |
| `@0xBEA18` | r4=record, fr4=X, fr5=Y | fr2=float | Generic type-dispatch. **Partial** — on-axis correct, off-axis wrong (r5/r6 pollution). Use type-specific wrappers. |

The convention discovery is worth telling: the spec initially targeted `@0xBEA18` expecting a float in `fr2`. Tracing the real injector-latency function (`@0x35A96`) showed the ECU actually calls **`@0xBEA94`** returning **raw u16 in r0** (via `ftrc`). `@0xBEA94` hardcodes the u16 lookup path, bypassing the type-dispatch table entirely. The generic dispatcher is the exception, not the rule.

### Write watchpoints

The emulator can watch a RAM address and report every instruction that writes to it. This is how the MAF writer was found — hiding in a **delay slot** (the store instruction sits *after* the `rts`), invisible to static analysis. See [Method Lessons](#method-lessons) below.

### Taint tracking (data-flow)

Mark a RAM address as tainted (e.g., "this is the MAF value"), run code, and see everywhere the tainted data flows. Taint propagates through data movement (`mov`, `fmov.s`, arithmetic, FPU ops) but not control flow — deliberately simple.

```python
cpu.taint_address(0xFFFF66A0, "MAF")  # temp-compensated MAF
# ... run fuel/timing functions ...
cpu.taint_report()  # every address/register holding MAF-derived data
```

Built to answer "who uses VE?" and "where does load go?" — the questions that xref hunting couldn't resolve.

### RAM symbol table

28 known RAM addresses mapped to symbolic names with confidence levels and sources. Enable `mem.symbolic_trace = True` and every RAM access logs as `LOAD_RAM` instead of `0xFFFF66E4`. Key entries:

| Symbol | Address | Confidence |
|--------|---------|------------|
| `LOAD` | 0xFFFF66E4 | HIGH |
| `RPM` | 0xFFFF6988 | HIGH |
| `MAF_TEMP_COMP` | 0xFFFF66A0 | HIGH |
| `DTC_STATUS` | 0xFFFF3720 | HIGH |
| `PEDAL` | 0xFFFF8B70 | HIGH |
| `IAM` | 0xFFFF30F0 | HIGH |

### Log playback

`playback.py` feeds real OBD2 CSV logs through the emulator as test vectors — set RAM to logged sensor values, run the ECU functions, compare emulator outputs against what the car actually did. First targets: idle target selection (ECT 50°C switch), timing lookups at logged RPM/load, fan threshold crossings.

This is the closest thing to differential testing available without a second ECU. When the emulator disagrees with the log, it's either an emulator bug or a knowledge gap — both worth finding.

---

## Test Suite: 95 Tests

The test suite is the emulator's immune system. Every opcode has unit tests with edge cases. The count grew from 77 to 95 as bugs were found and fixed — each fix added regression tests.

**What the tests cover:**
- Every implemented integer opcode (mov variants, ALU, shifts, branches with delay slots)
- Every FPU opcode (fmov.s variants, arithmetic, comparisons, conversions)
- Delay-slot semantics (the T-flag bug would have been caught by these if they'd existed first)
- Memory model (ROM write traps, RAM range, peripheral traps)
- Dispatcher acceptance (end-to-end table lookups with bit-exact expected values)

**Run them:** `python3 test_cpu.py` → `95 passed, 0 failed`

---

## Bugs Found and Fixed

Each bug has a story. They're worth telling because they show *how* the emulator gets better — always reactively, always because something didn't work.

### Bug 1: `0x4000` misdecoded as NOP (found during development)

It's actually `shll r0`. The shift-instruction bit masks were wrong across the entire family (shll/shlr/shll2/8/16). Fixed by correcting the masks. **Lesson:** don't trust your first reading of the opcode map; test every variant.

### Bug 2: Branch delay-slot T-flag (found during development)

`bt`/`bf` were deciding on the T flag *after* the delay slot executed. Real SH hardware decides *before* the delay slot runs — the delay slot can clobber T without affecting the branch. This broke the axis-search routine in the dispatcher until fixed. **Lesson:** the SH manual's timing diagrams aren't optional reading.

### Bug 3: `0x8801` — the opcode that wasn't (found 2026-10-18)

4,963 occurrences ROM-wide, completely unimplemented. Two agents investigated and reached **opposite conclusions**:

- **Agent A:** `cmp/eq #imm, R0` — the low-byte distribution (0x00–0x09 dominant, but 0x40/0x5A/0x8F/0xF6 present) fits 8-bit immediates. `0x88xx` as cmp/eq is canonical SH, in every manual since the SH-1.
- **Agent B:** `BLD` (bit load, T = bit N of R0) — DeepWiki documents SH-2A bit-manipulation in the `0x88xx` space; 94.6% of occurrences are followed by T-dependent branches.

**Resolution:** Agent A is almost certainly correct. The "followed by a branch" evidence doesn't discriminate — cmp/eq is *also* almost always followed by a branch (that's what it's for). The discriminator is the immediate distribution: values above 7 are invalid as BLD bit numbers but perfectly sensible as comparison constants. Agent B got a sensible result because in the cases tested, R0 happened to be 0 or 1 (where the two interpretations agree).

Implemented as `cmp/eq #imm, R0` with 5 new tests. **Lesson:** when two hypotheses both "work," find the case where they disagree and test *that*.

### Bug 4: `fmov.s @(R0,Rm),FRn` trapped on RAM (found 2026-10-18)

Was using `rdf32_rom` (ROM-only read); other FPU loads use `ldf32` (unified). One-line fix. **Lesson:** inconsistency between similar code paths is a bug waiting to happen.

### Bug 5: Missing `not`, `div1`, `cmp/eq` (found 2026-10-18)

Three unimplemented instructions. `div1` got the full SH-manual algorithm (Q=old T, M=MSB(Rm), shift-and-add/subtract with borrow/carry). 7 new tests. **Lesson:** the trace agents are an effective fuzzer — they find missing opcodes by trying to run real code.

### The refuted bug: `bf/s` 12-bit displacement (2026-10-18)

A trace agent reported that `bf/s` and `bt/s` use 12-bit displacements. This is **encoding-impossible** — `0x8Dxx` only has 8 bits for displacement in 16 bits total. All 121 large-displacement sites decode to valid targets under 8-bit. The emulator was already correct; the agent likely misidentified a `bra` (which genuinely has 12-bit displacement). Regression tests were added to lock in the correct behavior rather than "fixing" a non-bug. **Lesson:** verify the bug report before fixing it. The skepticism discipline applies to our own tools.

---

## Method Lessons

The emulator hasn't just validated tables — it's taught us how to do this work.

### Delay slots hide writers

The MAF writer (`fmov.s fr0,@r2` at `0x004958`) sits in the **delay slot** of the `rts` at `0x004956`. Static analysis missed it completely — the function *looked* like it returned without storing. Only the emulator's write watchpoints caught it.

**Rule:** delay-slot checking is now mandatory for any "zero writers" claim.

### Watchpoints beat static analysis for "does X happen"

"Does any code write to this address?" is a dynamic question. Static analysis answers it by pattern-matching instruction sequences — and misses delay slots, computed addresses, and ISR writes. The emulator answers it by *watching*. When the two disagree, the emulator wins.

### The emulator needs its own skepticism discipline

Every bug above was found reactively — something didn't work, we investigated, the emulator was wrong. That's backwards. A bug in the emulator gives wrong answers with the authority of "the real ROM code computed this" — the most dangerous kind of wrong in this project.

No "it works" label is ever final. Every opcode implementation is a hypothesis. The way you validate infrastructure is by trying to break it on purpose.

### The invisible-writer pattern

Two critical variables — engine load (`0xFFFF66E4`, 69 readers) and RPM (`0xFFFF6988`, 246 readers) — have **zero** identifiable writers through any standard store pattern. The writers must use computed addresses, ISRs, or DMA. This is now a recognized pattern, not an anomaly. If you can't find a writer, check non-standard mechanisms before concluding the variable is unused.

---

## Limitations

Honest accounting of what the emulator can't do:

**CPU-only, no peripherals.** The emulator models registers and memory. The ECU is a CPU *plus* ADC, timer units (ATU-II), CAN controller, and interrupt system. We can run a function in isolation; we can't boot the firmware and let it run. Every invocation requires hand-configured RAM — we're always testing out of context.

**Can't prove physical meaning.** The emulator answers "what does the code compute?" It cannot answer "what does the engine do?" Scaling disputes (VE, D7FEC) are physical questions that need dyno or MAF data. The emulator tells you the ECU will behave a certain way; only the engine tells you it *should*.

**Python speed.** Single-threaded, no JIT. Fine for thousands of instructions per table lookup. Not fine for running all scheduler tasks, fuzzing inputs, or sweeping ECT across a full range. Performance work is queued but not urgent yet.

**Validation is incomplete.** The unit tests verify what we thought to test. Differential testing against another SH implementation (Ghidra's SLEIGH emulator) is queued but not done. The strongest validation — booting the firmware — needs peripherals first.

---

## Roadmap

Queued improvements, in rough priority order (from `ecu-tools/EMU_ROADMAP.md`):

### Done (2026-10-18)
- ✅ Taint tracking (data-flow)
- ✅ RAM symbol table (28 symbols)
- ✅ Log playback (`playback.py`)
- ✅ RAM range fix (64KB at `0xFFFF0000+`)
- ✅ 0x8801, `not`, `div1`, `fmov.s` routing fixes

### Queued
- **Peripheral stubs** (ADC, timer, interrupts) — minimum viable to get the scheduler ticking. Highest value, hardest.
- **Scheduler execution** — run all tasks in order like the real ECU. Turns the emulator from function-tester to system-tester.
- **Snapshot/restore** — save full CPU+RAM state, sweep inputs programmatically. Prerequisite for fuzzing.
- **Trace recording** — full execution traces for offline analysis. Taint tells you *where* data flows; traces tell you *why*.
- **Interactive debugger** — REPL with step/breakpoint/inspect for 2am exploration.
- **Coverage tracking** — which parts of the ROM have we actually executed? Probably reveals unexplored subsystems.
- **Differential testing vs Ghidra** — run the same code in both emulators, compare. Cheap confidence boost.
- **Symbolic execution** (someday) — derive constraints instead of sweeping values. Hardest item; do last.
- **Performance** (PyPy/C extension/JIT) — when it hurts, not before.

---

## How to Use It

### Validating a table

```python
from emu.cpu import SH2A

cpu = SH2A()
cpu.load_rom("12_UDSM_Impreza_2.0i_Sport_MT_original.bin")

# Set up: r4 = dispatcher record, fr4/fr5 = X/Y inputs
cpu.r[4] = 0xB5048      # injector latency record
cpu.fr[4] = 11.5        # X = 11.5V
cpu.fr[5] = 0.0         # Y (unused for 1D)

cpu.run(0xBEA94)        # execute the real dispatcher
print(f"Raw u16 result: {cpu.r[0]}")  # compare against definition
```

### Tracing a function with watchpoints

```python
cpu.watch_write(0xFFFF66E4)  # watch the load variable
cpu.run(0x4938)              # run the MAF function
for pc, addr, val in cpu.watch_hits:
    sym = lookup(addr)       # symbolic name from symbols.py
    print(f"{pc:#x} wrote {val} to {sym.name if sym else addr:#x}")
```

### Taint tracking a dataflow

```python
cpu.taint_address(0xFFFF66A0, "MAF")
# ... run code ...
for addr, labels in cpu.taint_report():
    print(f"{addr:#x} tainted by {labels}")
```

### Pre-flash tune validation (`tune_diff.py`)

```bash
# What happens if I change injector latency at 11.5V?
python3 tune_diff.py --record 0xB5048 --type u16 \
  --set "0xD26A0[2]=5000" --sweep "6.5:16.5:21"

# Diff two ROMs through the real dispatcher code
python3 tune_diff.py --rom2 modified.bin --record 0xB5048
```

Validates a proposed tune change through the **real ROM dispatcher code** before flashing. The self-test: a +0.2500 ms change at X=11.5V shows max delta exactly +0.2500 ms, localized correctly, zero leakage outside, bit-exact on-axis.

### Log playback

```bash
# Feed a real OBD2 log through the emulator, compare against logged behavior
python3 emu/playback.py CSVLog_20261018_170710.csv
```

---

## Files

- `ecu-tools/emu/cpu.py` — the interpreter
- `ecu-tools/emu/mem.py` — memory model
- `ecu-tools/emu/symbols.py` — RAM symbol table
- `ecu-tools/emu/playback.py` — log playback
- `ecu-tools/emu/test_cpu.py` — 95 unit tests
- `ecu-tools/tune_diff.py` — pre-flash tune validation
- Analysis reports in the project archive

## See Also

- [Emulation-Validation](Emulation-Validation.md) — what the emulator validated (batches A–H)
- [Methodology](Methodology.md) — the broader RE workflow
- [Verified-Core-Definition](Verified-Core-Definition.md) — the 0.8 definition the emulator validated
- [Open-Threads](Open-Threads.md) — current open emulator work
