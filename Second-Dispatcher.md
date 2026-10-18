# Second Dispatcher: @0xBEA18

> **⚠️ UNVERIFIED RESEARCH — MAY BE 100% WRONG**
>
> This page documents findings from AI-assisted disassembly. Nothing here has been validated on hardware. The disassembly interpretation may be incorrect. Trust nothing, verify everything.

---

## Summary

The EP5GD00A ROM contains **two** table lookup dispatchers, not one:

| Dispatcher | Address | Mechanism | Call sites |
|------------|---------|-----------|------------|
| Main | `0xBE964` | R2 set internally via helper @0xBEDE0 | ~479 (252 literal pools) |
| Secondary | `0xBEA18` | Type-byte dispatch via jump table | **182 jsr sites, 169 resolved** (162 unique directory entries) |

The secondary dispatcher was discovered 2026-10-18 during investigation of tip-in enrichment tables, which use it instead of the main dispatcher.

---

## How @0xBEA18 Works

Disassembly (SH-2, verified via Capstone):

```
0xBEA18: sts.l macl,@-r15     ; save registers
0xBEA1A: sts.l pr,@-r15
0xBEA1C: bsr 0xBEED0          ; input conditioning helper
0xBEA1E: sts.l mach,@-r15     ; (delay slot)
0xBEA20: mov #16,r0
0xBEA22: add r4,r0            ; r0 = dir_entry + 16
0xBEA24: mov.b @r0,r6         ; r6 = TYPE BYTE at [dir_entry+16]
0xBEA26: mova 0xBEA48,r0      ; r0 = jump table base
0xBEA28: mov.l @(r0,r6),r5    ; r5 = jump_table[type_byte]
0xBEA2A: mov.l @(12,r4),r1    ; r1 = [dir_entry+12] (data pointer?)
0xBEA2C: jsr @r5              ; CALL type-specific handler
0xBEA2E: mov.w @(0,r4),r0     ; (delay slot) dims?
0xBEA30: tst r6,r6            ; if type == 0, skip scaling
0xBEA32: bt/s 0xBEA3E
0xBEA34: fmov fr2,fr1         ; (delay slot)
0xBEA36: add #20,r4
0xBEA38: fmov @r4+,fr0        ; fr0 = float at [dir+20] (scale?)
0xBEA3A: fmov @r4,fr1         ; fr1 = float at [dir+24] (offset?)
0xBEA3C: fmac fr0,fr2,fr1     ; fr1 = scale*raw + offset
0xBEA3E: fmov fr1,fr0         ; result
...epilogue...
```

**Key differences from main dispatcher (@0xBE964):**
- Takes **R4** = directory entry (main takes R0)
- Uses **type byte** at [entry+16] to select handler (main uses R2 set internally)
- Applies **linear scaling** (scale×raw + offset) when type ≠ 0, using floats at [entry+20] and [entry+24]
- Type 0 = no scaling (raw passthrough)

---

## Type Handlers

Jump table at `0xBEA48`:

| Type (raw) | Type (pre-scaled) | Handler | Data type | Notes |
|------|-------------------|---------|-----------|-------|
| 0x00 | 0x00 | `0xBECF8` | **float32** | No post-scaling (`tst r6,r6; bt/s` skips the fmac). Directory scale reads 0.0 (unused). |
| 0x04 | 0x01 | `0xBED84` | uint16 | Linear scaling: `result = scale×raw + offset` |
| 0x08 | 0x02 | `0xBEDB0` | uint16 | Linear scaling: `result = scale×raw + offset` |
| — | 0x03 | `0xBED28` | (uint16) | **Zero call sites in the ROM** |
| — | 0x04 | `0xBED54` | (uint16) | **Zero call sites in the ROM** |

**The type byte selects the data type.** This was the key discovery of the 2026-10-18 re-triage: type 0 = float32 tables (proven by a 4×10 table @0xCE1C4 with 40 clean floats), types 1–2 = uint16 with `scale×raw + offset` via fmac. 14 of the 101 re-triage tables are float32 type-0.

**The "type byte mystery" is SOLVED.** The earlier report of type bytes like 0x3B/0x3C/0x3E (which would index past the 5-entry jump table) was a **4-byte-misaligned read** — 0x3B is the first byte of the scale float `3B800000` (= 1/256) at [entry+20], not the type byte. The SH `mov.l @(r0,r6)` index is unscaled, so the true type bytes are pre-scaled **{0x00, 0x04, 0x08}** — confirmed across all 169 resolved call sites, zero exceptions.

---

## Directory Entry Layout (28 bytes)

Decoded during the re-triage:

| Offset | Content |
|--------|---------|
| +0 | X dim (u16) |
| +2 | Y dim (u16) |
| +4 | X axis pointer (u32) |
| +8 | Y axis pointer (u32) |
| +12 | Data pointer (u32) |
| +16 | **Type byte** (pre-scaled: 0x00 / 0x04 / 0x08) |
| +20 | Scale (float32, unused when type = 0) |
| +24 | Offset (float32, unused when type = 0) |

## Which Tables Use Which Dispatcher?

**Main dispatcher (@0xBE964):** ~479 call sites. The majority of tables.

**Secondary dispatcher (@0xBEA18):** 182 jsr sites, 169 resolved (162 unique directory entries). Used by fuel/tip-in code and other subsystems. 5 sites use runtime-computed R4 (register-indexed, struct/stack offsets, GBR-indirect) and remain unresolved — e.g. a 65-entry all-type-1 table @0x6D89C.

**Call pattern for @0xBEA18:**
```
mov.l @(disp,PC),rN    ; rN = 0xBEA18 (from literal pool)
...
jsr @rN                ; call dispatcher
```

This differs from the main dispatcher, which is typically called via `jsr @r2` after a helper sets up R2.

---

## Implications for Methodology

The "find the dispatcher first" methodology needs updating:

**Old:** Find *the* dispatcher, follow its call sites.

**New:** Find *all* dispatchers. There may be more than one. Different subsystems may use different lookup mechanisms.

**How to find them:**
1. Search for `jsr @rX` (register-indirect calls) with high call-site counts
2. Search for type-byte dispatch pattern: `mov.b @[rX+16],rY` followed by jump table lookup
3. Search for literal pool entries containing candidate dispatcher addresses
4. Don't assume one dispatcher covers all tables

**No third dispatcher found** (searched for type-byte dispatch pattern, 0 candidates besides @0xBEA18).

---

## Open Questions

1. Why two dispatchers? Different data types? Different subsystems? Historical accident? (The type byte answers *how* they differ, not *why* both exist.)
2. Are there tables that use NEITHER dispatcher? (Throttle tables at 0xFxxxx use direct addressing, not either dispatcher.)
3. The 5 runtime-computed-R4 call sites (register-indexed, struct offsets, GBR-indirect) — what tables do they reach?

**Resolved since the first writeup:**
- ~~What do types 1-4 do differently from type 0?~~ → The type byte selects the data type: type 0 = float32 passthrough, types 1–2 = uint16 with linear scaling. Types 3–4 have zero call sites.
- ~~Do the high type-byte values (0x3B+) belong to @0xBEA18 or @0xBE964?~~ → Neither — they were a 4-byte-misaligned read of the scale float. True type bytes are {0x00, 0x04, 0x08}.

---

## Discovery Credit

Found by AI agent during tip-in enrichment investigation (2026-10-18). The agent was trying to determine tip-in Y axes from code, noticed the tip-in function calls @0xBEA18 instead of @0xBE964, and decoded the type-byte dispatch mechanism.

Verified by direct disassembly (Capstone, SH-2A + SHFPU + Big Endian).

---

*Last updated: 2026-10-18. May be 100% wrong.*
