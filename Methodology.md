# Methodology: How to Hunt Tables

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


This is the table-discovery workflow developed during the EP5GD00A project. It's written as a guide someone else could follow on a different Denso SH7058 ROM.

> **Status:** This methodology produced 507 tables. It has not been independently replicated on another ROM yet. The Honda project (see [Honda-Project](Honda-Project.md)) is the first test of transferability.

---

## Prerequisites

- A ROM dump (1MB binary from the target ECU)
- A hex editor or Python with `struct`
- Reference definitions from related ECUs (same manufacturer, same chip family)
- Patience

## Step 1: Decode the Table Directory

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

**Rule: Never trust the directory alone. Always validate against raw bytes (Step 2).**

## Step 2: Physical Validation

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

## Step 3: Determine Scaling

The raw `uint16` values need a formula to convert to physical units. Sources:

1. **Reference definitions** — Same-named tables in WRX/Forester/BRZ definitions often share scaling formulas, even when addresses differ completely.
2. **Physical reasoning** — If values are 800–7000 and the axis is RPM, it's probably RPM directly. If values are 0–255 and it's a percentage, try `x/2.55` or `x*0.392157`.
3. **Cross-validation** — The same physical quantity (e.g., ECT) should use the same scaling across tables.

**Common Denso scalings we confirmed:**
| Formula | Use |
|---------|-----|
| `x * 0.05` | Ignition timing (degrees) |
| `x / 32768` | Multipliers (0.0–2.0 range) |
| `x * 0.0078125` | Fueling (AFR numerator) |
| `(x - 760) * 0.01933677` | Manifold pressure (relative psi) |
| `x * 0.01933677` | Absolute pressure |
| `96560.6 / x` | Engine speed per km (gear calculation) |

> **Note:** The WRX definition labels `96560.6/x` as "Rev/Mile" but it actually computes rev/km. We corrected this.

## Step 4: DTC Descriptor Mapping

For ECUs with per-DTC enable/disable switches:

1. **Find the descriptor table** — Look for a region with fixed-size entries (20 bytes in our case, at `0xA1431`)
2. **Identify the P-code encoding** — In our ROM, bytes 3–4 of each entry are the P-code as big-endian `uint16` (e.g., `0x0420` = P0420)
3. **Map all entries** — Read every descriptor, extract P-codes, build the full table
4. **Validate** — Check against known codes (P0305/P0306 should be disabled on a 4-cylinder; P0420 should exist)

This method mapped all 236 DTC switches **without any disassembly**. See [DTC-Mapping](DTC-Mapping.md) for details.

## Step 5: Scaling Fingerprints

Once you have confirmed scalings from reference definitions, use them as **ROM-wide search patterns**:

1. Take a known table with a distinctive scaling
2. Search the ROM for data that produces sane values with that scaling
3. Candidates found this way are likely the same *type* of table

This is how we found tables the directory didn't list clearly.

## Step 6: Code-Reference Tracing

The most powerful method, developed late in the project:

1. **Set up Ghidra** with the SH-2 processor (see [Ghidra-Guide](Ghidra-Guide.md))
2. **Find the table dispatcher** — Denso uses a central routine for table lookups (ours is at `0xBE964`)
3. **Extract all call sites** — Every `jsr` to the dispatcher is a table access
4. **Resolve directory entries** — Each call site references a directory entry; resolve to the table address
5. **Cross-reference** — Any address the code uses that isn't in your definition is a missing table

This found 238 additional 2D tables we had missed through directory analysis alone.

### SH-2 Literal Pool Technique

If you can't find the dispatcher yet, use this simpler approach:

1. Scan the code region for `MOV.L @(disp,PC),Rn` instructions (opcode `0xD0`–`0xDF`)
2. Each one loads a 32-bit value from a literal pool
3. Resolve the literal pool entries to get the actual addresses
4. Filter for addresses in the calibration/table directory range
5. These are code-referenced tables

## Step 7: The Validation Loop

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

## The Iron Law

> **Methodology transfers, structures never do.**

Across Subaru, Honda, Nissan, and Mitsubishi — same chip, same vendor, sometimes same engine family — the *approach* works everywhere but no *specific* table, address, or data format ports directly. Every ROM is its own puzzle.
