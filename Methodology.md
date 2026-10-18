# Methodology: How to Hunt Tables

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

## External Validation

**Independent confirmation:** The reverse-engineering community has independently described this approach. From [reverseengineer.net's Denso guide](https://reverseengineer.net/denso-continental-ecu-reverse-engineering/):

> "Denso uses a proprietary calibration data format that differs from Bosch's map structure. Axis definitions, scaling factors, and map lookup routines follow Denso-specific patterns. **Identifying the first map lookup function is key; once found, cross-references reveal the entire calibration data set.**"

That's our dispatcher, described independently. The "find the lookup function first" methodology is validated.

**On standards:** Denso's SH7058 firmware (2012) is **proprietary, pre-AUTOSAR**. Denso became an AUTOSAR Core Partner in January 2026, but Gen 2 (SH7058/SH7055) ECUs do not use AUTOSAR. There is no public standard for Denso's calibration format — the directory/dispatcher structure documented here was reverse-engineered, not specified.

---

This is the table-discovery workflow developed during the EP5GD00A project. It's written as a guide someone else could follow on a different Denso SH7058 ROM.

> **Status:** This methodology produced 863 tables (v0.7). Validated across three platforms: Subaru (two dispatchers, directory decoded), Mazda (dispatcher proven absent — itself a finding), Honda (directory decoded, 92 tables from 4).

---

## How This Project Was Actually Run

The technical steps below are only half the methodology. The other half is *how the work was organized* — because this was built by AI agents working in parallel, and the process that kept them honest is as reusable as the ROM techniques. This section documents that process the way it actually happened, stage by stage.

### The workflow rules

These were set early and never broken:

- **Versioning is semantic `X.X.X`**: first digit 0 = in development, second = stage, third = revision.
- **Numbers are merged definitions. Letters are proposals.** New tables were always banked first in a standalone letter file (`0.3.1a`, `0.3.1b`, … `0.3.1t`) for review — never merged silently into a numbered release. A new letter every iteration; old files never renamed, never overwritten.
- **Bank finds, ask before merging.** Agents produced findings as standalone files. Merging into the numbered definition was a separate, explicit decision.
- **Every claim carries its evidence.** Table entries got confidence labels (HIGH/MEDIUM/LOW); reports distinguished disassembled evidence from shape inference from open threads.

### Stage 0 — Scouting (what's already out there)

Before touching the ROM: collected reference definitions (WRX, Forester, BRZ), confirmed no public SH7058 ROM binary exists anywhere (all projects exclude ROMs for copyright), and found the one usable ROM — a 2022 RomRaider forum dump of the same CAL ID. First lesson, learned the hard way: **the CVT reference definition's addresses were garbage on the MT ROM. Blind porting was abandoned.** Names and scalings transfer; addresses never do.

### Stage 1 — Directory decode (the first map)

The ROM's table directory (~0xB0000–0xB8000, 832 entries) was decoded first. This produced the first usable definition (v0.1.9, 60 tables — the "first tunable file"). Lessons that became permanent rules:

- **The directory lies.** Dims, orientation, and data types in directory entries are unreliable — always validate against raw bytes.
- `0x7FF8` is a Denso padding sentinel, not data.
- **Range-overlap dedup is required** — exact-address dedup misses tables that share byte ranges.

### Stage 2 — Dispatcher tracing (the main hunt)

Found the main table-lookup dispatcher (@0xBE964, ~479 call sites), then — much later — a **second** dispatcher (@0xBEA18) that an earlier pass had dismissed. The rule that emerged: **if the dispatcher references it, it's a table. Period.** "Sparse," "flat," "marginal," and "non-monotonic" are not valid rejection reasons — this was proven when a reject audit (0.3.1r) showed 48 of 60 rejected candidates should have been added, and every single one had dispatcher references.

### Stage 3 — Reject audits (correcting our own filters)

Every rejection round was re-audited with a single criterion: does code reference it? This stage exists because the hunters kept outsmarting themselves — inventing aesthetic reasons to reject tables the firmware clearly used. The audit, not the hunter, had the final word.

### Stage 4 — Cross-platform (testing the method, not the tables)

Deliberately pointed the method at other ECUs to see what transferred:

- **Mazda MX-5 NC (Denso SH7058):** no dispatcher, no directory — proven by layout. The *absence* was the finding. Opposite axis-sharing behavior (504 unique axes, only 7 shared vs. Subaru's aggressive pooling).
- **Honda (Keihin SH7058):** an early "no directory" conclusion was **wrong** — the directory was there at 0xA454–0xDB8C all along, and directory-decode-first took it from 4 tables to 92. Honda uses uint16BE axes, not float32BE.
- **Saab Trionic survey:** the Saab scene never needed our methods (their ROMs ship with symbol tables), but their 25 years of practice validated the approach a third time — and gave us two steals: masked-pattern checksum searching and computation-flow documentation.

The iron law survived contact with all three: **methodology transfers, structures never do.**

### Stage 5 — Disassembly verification (proving the hunt complete)

Three independent methods were run to convergence: main-dispatcher call sites (407 table addresses), secondary-dispatcher sites (169 of 182 resolved), direct flash references, and the full directory (832 entries). Result: **zero phantoms, 8 genuinely missing tables**, merged as v0.6 (763 tables). The type-byte "mystery" (0x3B/0x3C/0x3E indexing past the jump table) turned out to be a 4-byte-misaligned read — the true type bytes are pre-scaled {0x00, 0x04, 0x08}, and the type byte selects the **data type**: type-0 tables are float32 (no fmac scaling). A later **re-triage** re-ran the missing-table hunt against the fully-resolved secondary-dispatcher sites and found **101 more tables** (87 uint16 + 14 float32), merged as v0.7 (863 tables). Estimated universe revised ~785 → ~875–900; coverage ~97%.

### Stage 6 — Musing agents (structured dissent)

Five agents were each given a philosophical angle and explicit autonomy to test a theory — including overturning "closed" labels. They did: the "not a table" verdict on 0xD0000 was wrong (it's a shared structure with 17 directory refs), and the second dispatcher exists (dismissed earlier). They also produced hard negative findings (Requested Torque Base doesn't exist; injector flow scalar doesn't exist) and resolved the base-timing scaling dispute (`x*0.1`, not `x*0.05`). **The process that assumes the last conclusion was wrong is the one that got it right.**

### Stage 7 — Documentation and seeding (this wiki)

Computation-flow docs (fuel/timing/DBW algorithms, not just tables), a LaTeX reference skeleton, the DTC mapping writeup, and the checksum hunt — which resolved the last flashing blocker: the ECU does **not** verify a ROM checksum at boot (the checksum routine is a reflash-protocol tool, not a boot gate).

### Scale note

Roughly 560 human-hours of equivalent labor, done in ~13 wall-clock hours via parallel agents, at an estimated 50–100M tokens of compute (~a bathtub of water, ~a day of household electricity — the physical ledger is in the free-roam notes). The parallelism only worked because of the workflow rules above: without letter-files-for-approval and the reject audits, fast agents just produce fast garbage.

---

## Prerequisites

- A ROM dump (1MB binary from the target ECU)
- A hex editor or Python with `struct`
- Reference definitions from related ECUs (same manufacturer, same chip family)
- Patience

## Step 1: Find the Dispatcher**s** (Do This First)

**This is the most important step.** Before directory decode, before anything else.

Denso ECUs may use **one or more** central table lookup routines (dispatchers). We found **two** on the EP5GD00A:

| Dispatcher | Address | Mechanism | Call sites |
|------------|---------|-----------|------------|
| Main | `0xBE964` | R2 set internally via helper | ~479 |
| Secondary | `0xBEA18` | Type-byte dispatch via jump table | 182 |

### Main Dispatcher (@0xBE964)

- 479 call sites, R0 = directory entry address, `jsr @r2` to type-specific interpolation
- R2 is set *inside* the dispatcher via helper @0xBEDE0, not by the caller
- 336 of 479 calls have pre-call input conditioning helpers

### Secondary Dispatcher (@0xBEA18)

- Uses **type byte** at [directory_entry+16] to index jump table at `0xBEA48`
- 5 types (0-4), each with its own handler; only types 0–2 have call sites in practice
- If type ≠ 0, applies linear scaling: `result = scale×raw + offset` (floats at [entry+20], [entry+24])
- Takes R4 = directory entry (main dispatcher takes R0)
- 169 of 182 call sites resolved; the "type bytes past the jump table" mystery was a 4-byte-misaligned read — true type bytes are pre-scaled {0x00, 0x04, 0x08}
- See [Second-Dispatcher](Second-Dispatcher.md) for full details

### How to Find Them

1. Search for `jsr @rX` (register-indirect calls) with high call-site counts
2. Search for type-byte dispatch pattern: `mov.b @[rX+16],rY` followed by jump table lookup
3. Search literal pools for candidate dispatcher addresses
4. **Don't assume one dispatcher covers all tables** — and don't assume there's only one dispatcher

**If you find them:** Every call site is a table. Extract them all.

**If you DON'T find them:** That's also a finding. The Mazda MX-5 NC (Denso SH7058, same chip) has no dispatcher — tables live at hardcoded addresses where Subaru has its directory. The absence of a dispatcher tells you the architecture is different. Switch to subsystem-by-subsystem analysis.

**Cross-platform results:**
- Subaru EP5GD00A: two dispatchers (`0xBE964`, `0xBEA18`) ✓
- Mazda LFG7EG: No dispatcher (proven by address layout analysis) ✓
- Honda RRB-A080: directory decoded first; dispatcher question superseded ✓

## Step 2: Decode the Table Directory

Denso ECUs contain a **table directory** — a region of the ROM (around `0xB0000` in our case) that lists calibration tables with their dimensions and addresses.

**What to look for:**
- Repeating fixed-size entries (we found 28-byte and 16-byte formats)
- Each entry contains: dimensions (X count, Y count), data address, axis addresses
- The directory is the fastest way to get candidate tables

**Critical warning:** **The directory lies.** We found:
- Wrong dimensions (directory says 10×16, real table is 10×8)
- Stale entries (pointing to old/deleted tables)
- Ghost entries (overlapping real tables with garbage metadata)
- Placeholder entries (all zeros, all 0xFF)

**Rule: Never trust the directory alone. Always validate against raw bytes (Step 3).**

## Step 3: Physical Validation

For every candidate table from the directory:

1. **Read the axis data** at the listed axis addresses
   - Axes are typically `float32BE` (big-endian 32-bit float)
   - Check: are the values monotonic? In a sane range? (RPM: 500–7000, ECT: -40–120, etc.)
2. **Read the table data** at the data address
   - Data is typically `uint16BE` (big-endian 16-bit unsigned)
   - Check: do values make physical sense with the suspected scaling?
3. **Verify dimensions** by reading past the listed bounds
   - If the directory says 16 rows but rows 8–15 are garbage, the real table is 8 rows
4. **Check for shared axes** — Denso reuses axis arrays across tables (saves ROM space)

**If any check fails, the table is either misdecoded or not what you think it is. Do not add it.**

## Step 4: Determine Scaling

The raw `uint16` values need a formula to convert to physical units. Sources:

1. **Reference definitions** — Same-named tables in WRX/Forester/BRZ definitions often share scaling formulas, even when addresses differ completely.
2. **Physical reasoning** — If values are 800–7000 and the axis is RPM, it's probably RPM directly. If values are 0–255 and it's a percentage, try `x/2.55` or `x*0.392157`.
3. **Cross-validation** — The same physical quantity (e.g., ECT) should use the same scaling across tables.

**Common Denso scalings we confirmed:**
| Formula | Use |
|---------|-----|
| `x * 0.1` | Ignition timing (degrees) |
| `x / 32768` | Multipliers (0.0–2.0 range) |
| `x * 0.0078125` | Fueling (AFR numerator) |
| `(x - 760) * 0.01933677` | Manifold pressure (relative psi) |
| `x * 0.01933677` | Absolute pressure |
| `96560.6 / x` | Engine speed per km (gear calculation) |

> **Note:** The WRX definition labels `96560.6/x` as "Rev/Mile" but it actually computes rev/km. We corrected this.

## Step 5: DTC Descriptor Mapping

For ECUs with per-DTC enable/disable switches:

1. **Find the descriptor table** — Look for a region with fixed-size entries (20 bytes in our case, at `0xA1431`)
2. **Identify the P-code encoding** — In our ROM, bytes 3–4 of each entry are the P-code as big-endian `uint16` (e.g., `0x0420` = P0420)
3. **Map all entries** — Read every descriptor, extract P-codes, build the full table
4. **Validate** — Check against known codes (P0305/P0306 should be disabled on a 4-cylinder; P0420 should exist)

This method mapped all 243 DTC switches **without any disassembly**. See [DTC-Mapping](DTC-Mapping.md) for details.

## Step 6: Scaling Fingerprints

Once you have confirmed scalings from reference definitions, use them as **ROM-wide search patterns**:

1. Take a known table with a distinctive scaling
2. Search the ROM for data that produces sane values with that scaling
3. Candidates found this way are likely the same *type* of table

This is how we found tables the directory didn't list clearly.

## Step 7: Code-Reference Tracing

The most powerful method, developed late in the project:

1. **Set up Ghidra** with the SH-2 processor (see [Ghidra-Guide](Ghidra-Guide.md))
2. **Find the table dispatcher** — Denso uses a central routine for table lookups (ours are at `0xBE964` / `0xBEA18`)
3. **Extract all call sites** — Every `jsr` to the dispatcher is a table access
4. **Resolve directory entries** — Each call site references a directory entry; resolve to the table address
5. **Cross-reference** — Any address the code uses that isn't in your definition is a missing table

This found hundreds of tables missed through directory analysis alone — and the re-triage against fully-resolved call sites is what closed the hunt.

### SH-2 Literal Pool Technique

If you can't find the dispatcher yet, use this simpler approach:

1. Scan the code region for `MOV.L @(disp,PC),Rn` instructions (opcode `0xD0`–`0xDF`)
2. Each one loads a 32-bit value from a literal pool
3. Resolve the literal pool entries to get the actual addresses
4. Filter for addresses in the calibration/table directory range
5. These are code-referenced tables

## Step 8: The Validation Loop

Every table goes through this cycle:

```
Directory/Discovery → Physical Validation → Scaling Determination
        ↓ (fails any check)
    Reject or Re-investigate
        ↓ (passes all checks)
    Add to definition with confidence label
        ↓
    Code-reference check (is the firmware actually using it?)
        ↓
    Reject audit (does the rejection hold up? re-check with code refs)
        ↓
    Hardware validation (when cable arrives)
```

**Confidence labels we use:**
- **HIGH** — Physically validated + code-referenced + cross-checked against references
- **MEDIUM** — Physically validated, structure clear, identity reasonable but not confirmed
- **LOW** — Structure plausible but identity uncertain
- **(Unverified)** — Added to definition but needs further validation

## Common Pitfalls

1. **Trusting the directory** — It lies. Always validate.
2. **Wrong data type** — We spent time confused by "unusual values" that turned out to be `uint8` data read as `uint16`. When data doesn't make sense, question the type before questioning the table.
3. **Ghost entries** — Stale directory entries that overlap real tables. If two entries point to overlapping addresses, one is wrong.
4. **Shared axes** — Don't assume each table has unique axes. Denso pools them aggressively.
5. **Assuming addresses port** — Zero same-named tables share addresses between EP5GD00A and the WRX. Names and scalings transfer; addresses never do.
6. **Building from unverified reports** — Version 0.2.1b was built from a worker report without physical validation. Two out of three tables were broken. Never skip validation.
7. **Rejecting on aesthetics** — "Sparse," "flat," or "weird-looking" data is not a rejection reason. Only a failed physical check or a missing code reference is.
8. **Assuming one dispatcher** — We missed the second dispatcher for most of the project because we stopped looking after finding the first.

## The Iron Law

> **Methodology transfers, structures never do.**

Across Subaru, Honda, and Mazda — same chip, same vendor, sometimes same engine family — the *approach* works everywhere but no *specific* table, address, or data format ports directly. Every ROM is its own puzzle.
