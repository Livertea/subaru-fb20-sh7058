# Ghidra Disassembly Guide

## Setup

### What Was Built

A working SH7058 disassembly environment using Ghidra 11.3.2:

| Component | Details |
|-----------|---------|
| **Ghidra** | 11.3.2 |
| **Processor** | `SuperH:BE:32:SH-2` (big-endian) |
| **ROM** | 1MB loaded at base `0x00000000` |
| **RAM** | 64KB at `0xFFFF0000–0xFFFFFFFF` |
| **Entry point** | `0x00000C0C` (from reset vector) |

### Critical Gotcha

Use `-processor "SuperH:BE:32:SH-2"`, **not** `-language` (which is invalid for `analyzeHeadless`).

The SH7058 uses the SH-2E core (SH-2 + FPU). Ghidra's `sh-2.slaspec` does not decode FPU instructions; the community-standard workaround is to import as **SH-2A** (`sh-2a.slaspec` defines FPU), which handles the extra instructions gracefully since SH-2-compiled code never triggers SH-2A-only opcodes.

### Memory Map

```
0x00000000 – 0x000FFFFF   Flash (1MB) — code + calibration + table directory
0xFFFF0000 – 0xFFFFFFFF   RAM (64KB) — variables, learned values, table mirrors
0xFFFFFFC0 – 0xFFFFFFFF   Vector table
```

### Key Addresses

| Address | Description |
|---------|-------------|
| 0x00000000 | Reset vector → PC=0xC0C |
| 0x00000C0C | Entry point (verified clean disassembly) |
| 0x00002004 | CAL ID string "EP5GD00A" |
| 0x000A1431 | DTC descriptor table |
| 0x000B0000+ | Table directory region |
| 0xFFFFFFFC | Internal ID pointer → 0x2004 |

---

## Key Findings

### 1. The Table Dispatcher @ 0xBE964

**Denso does not use a single interpolation subroutine.** Instead, there is a **central dispatcher** at `0xBE964` with **479 call sites** across the ROM.

**Calling convention:**
- R0 = table directory entry address (`0xBxxxx`)
- Then `jsr` to dispatcher
- Internally does `jsr @r2` — the actual interpolation routine is passed as a function pointer
- This suggests **different interpolation routines per table type** (1D/2D/3D)

**Why this matters:** The 408 unique directory entries referenced by the dispatcher represent **every table the ECU actually uses**. Cross-referencing these against the definition found 238 additional 2D tables that directory analysis alone had missed.

**Helper:** Calls `0xBEDE0` (FPU-heavy, likely axis index computation).

### 2. Table Access Pattern

The firmware accesses tables three ways:
- **15,302 references** → RAM mirrors (`0xFFFFxxxx`) — code primarily uses RAM copies
- **7,154 references** → Table directory directly
- **5,505 references** → Flash calibration directly (`0x000Cxxxx`/`0x000Dxxxx`)

**Code never references flash table addresses directly** — it goes through RAM mirrors or the table directory. This is why the dispatcher cross-reference is so powerful.

### 3. X-tau Transient Fueling Trigger

**Found:** Subroutine at `0x365DC` returns 1/0 for transient detection.

- Conditional branch at `0x36592` guards transient table access
- Threshold table @0xD26C0 is 16×3 ECT-indexed (columns likely [MAP delta, TPS delta, filter])
- The full X-tau system (12 tables) is confirmed as actively used, not dead code

### 4. Knock Subsystem Map

**31 functions** identified in `0x04B000–0x04C700`. Data flow:

```
Knock Sensor
    → [0x04B16A] Feedback correction (immediate)
    → [0x04B544] Range/mode selection
        → [0x04BD38] Main knock control (Fine range)
            → [0x04BF1A] Fine correction applier
            → [0x04C112] Fine correction limits
        → [0x04C490] Rough correction (IAM range)
            → [IAM functions] Global multiplier update
    → Timing Output
```

**Key finding:** Fine correction learns to RAM at `0xFFFF8800` — the learned table lives in RAM, not flash. Knock Advance Max tables have **zero direct code refs** (accessed via directory/RAM only).

### 5. SSM Base Mapping

**12 function-pointer dispatch tables** found at `0x6E300–0x6EFFF` (largest: 151 entries).

Handler pattern matches DSchultz's documented SSM method: tiny functions that resolve a RAM address and return it.

**Identified:**
- `0xFFFF8800` — very likely the **live IAM variable** (HIGH confidence)
- `0xFFFF4024` — likely the **main sensor data struct** (MEDIUM-HIGH, 44 code refs)

The dispatcher itself was not found — likely accesses tables through a RAM-resident pointer initialized at boot.

### 6. Fuel Subsystem Map

**6 fueling functions** mapped:
1. X-tau transient fueling @0x3638E
2. Transient trigger @0x365DC
3. Primary OL Fueling @0x43F6C
4. Tip-in enrichment @0x344BE
5. Injector Latency (ref at 0x35AAE)
6. Warmup/Post-start (0x125B4, 0x12628)

**Table lookup convention confirmed:** r4=directory entry, call via `jsr @r2`, r13=RAM struct, r0=mode flag.

---

## Reusable Scripts

Scripts are in `~/workspace/tools/ghidra_scripts/`:
- `SSMBaseHunt.py` — Find SSM dispatch tables
- `KnockDisasm.py` — Knock subsystem analysis helper
- Plus 10 additional analysis scripts from the dispatcher investigation

## What's Next for Disassembly

1. **RAM mirror mapping** — Map the 15,302 RAM refs back to flash tables via boot-copy code
2. **Interpolation R2 values** — Identify which routine handles which table type
3. **SSM dispatcher** — Trace from CAN RX / boot code
4. **Knock sensor A/D path** — Not yet traced (different code region)
5. **0xD0000 investigation** — 20 code refs, unclear structure, highest-priority unknown
