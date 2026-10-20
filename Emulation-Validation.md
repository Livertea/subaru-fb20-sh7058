# Emulation and Validation

## The emulator

A purpose-built **SH-2A/SH-2A-FPU interpreter** (~146/146 unit tests green) written because Unicorn doesn't support SuperH. It executes the ECU's actual machine code — including the table-dispatcher entry points (@0xBEA94 for u16, @0xBE964 for float32) — against candidate tables, with taint tracking, a RAM symbol table, and drive-log playback.

**What emulation proves:** that a table's structure (dims, axes, interpolation) matches what the code expects, and that values flow from ROM through code to RAM. **What it doesn't prove:** that the code path runs on a live engine (EMU-validated ≠ live — the @0x42700 fuel lesson), or that a scaling is physically correct (the dispatcher ignores record scale/offset fields — they're metadata, not computation).

## The validation ladder

1. **Decoded** — bytes make sense (dims, axes, no 0x7FF8 collisions)
2. **Code-supported** — disassembly shows code reading it (all four indirection layers)
3. **Emulator-validated** — the real lookup code returns the ROM value on-axis and interpolates correctly off-axis
4. **Log-confirmed** — the car agrees (drive-log cross-checks, 10-19)
5. **Hardware-validated** — a modified flash changes engine behavior (pending the cable)

Steps 1–4 are done for the 0.8.4a tables. Step 5 is the line that matters, and nothing has crossed it.

## Tripwires (earned the hard way)

The disassembler lies often enough that raw-opcode verification is mandatory:

- **shdis.py fop=0x6/0x7:** misdecoded direction (load vs store) — flips reader/writer conclusions
- **shdis.py fop=0xA:** decoded as load, is actually a store — hid the [0xFFFF66A0] writers in plain sight
- **shdis.py fop=1/2 swap:** prints fmul for fsub and vice versa — invalidated the "IAM added" stage (it's a multiplier)
- **PC-relative masks:** `((pc+4)&~3)+disp*4` for `mov.l`, `((pc+4)&~1)+disp*2` for `mov.w` — wrong mask lands 2 bytes off with a clean-looking wrong decode
- **Delay slots:** every branch executes the next instruction first — two reports were invalidated by missing them (the DTC "bug" that wasn't)
- **Subroutine returns in `rts` delay slots** — disassembly stopping at `rts` misses the return value
- **"Zero X" negatives** need an independent second scan — one buggy mask hid 26 `ldc vbr` sites and nearly flipped the interrupt model
- **Point-value store censuses are blind to range fills** — the third [0xFFFF6988] writer hid in a boot RAM-fill loop (`mov.l r7,@r4` sweeping 12,259 addresses); a census resolving each store to one effective address never fires on `ea == target`. Any "zero writers" verdict must also sweep fill-loop patterns (`mov.l rX,@rY` + `add #4,rY` + backward branch) — RAMPTR6988 10-19
- **Masked GBR-relative stores** are a whole store class the tools didn't model — the knock bit1 setter @0x4B956 hid in one
- **Emulator seeding is byte-order sensitive** — a whole "contradiction" was a seeding bug

Full list lives in the project's AGENTS.md. Every tripwire has a report it burned.

## Permanent skepticism

The standing rule, in the author's words: *"we always need to be skeptics… we don't deserve to take any breaks on validation."* No CONFIRMED/RESOLVED/VALIDATED label is ever final. The 0.8.4a sweep voided prior claims twice in one day (B3 kill-3, the decel adders) — that's the system working, not failing.

## Drive-log cross-checks (10-19)

The newest validation leg: OBD-II logs against emulator/simulator predictions.
- Fuel model sane (STFT ±2%) — **log-confirmed**
- 50°C idle switch — **log-confirmed**
- WOT enrichment, decel cut — **log-confirmed**
- Cruise timing gap — the +8.0° knock-learned-advance mechanism is **CONFIRMED real** (10-19, KNOCKADV), but the cruise attribution is **IAM-constrained**: the positive authority is (1−IAM)×[0xFFFF87C4], enforced every knock cycle by @0x4C022 — at IAM=1 the learned term is destroyed in one cycle, so the gap can be [0xFFFF8800] only if IAM<1 at cruise. Gap source still open.
- Rev hang plate-hold — **log-confirmed**, mechanism under trace

See [Drive-Log-Findings](Drive-Log-Findings.md).
