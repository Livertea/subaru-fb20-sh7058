# Contributing

## Maintenance Note

**The original author (Johnathon) will likely not be actively maintaining this project going forward.**

This documentation is provided as-is — a snapshot of exploratory research conducted in October 2026. If you find errors, want to expand the table coverage, or want to take over maintenance: **please do. That's why it's public.**

This isn't abandonment — it's an invitation. The methodology is documented, the definitions are open, and the ROM is out there. If this work is useful to you, you're welcome to run with it.

---

## How to Help

### 1. Validate Tables in RomRaider (No Cable Needed)

The easiest way to contribute:

1. Download the definition files (see [[Definitions]])
2. Load them in RomRaider with an EP5GD00A ROM
3. Check tables for:
   - **Sane values** — Do fueling tables show realistic AFRs? Does timing look plausible?
   - **Clean axes** — Are RPM/load/ECT axes monotonic and in expected ranges?
   - **Rendering errors** — Does RomRaider display the table correctly, or show garbage?

**Report issues** with: table name, address, what you expected vs what you saw, and a screenshot if possible.

### 2. Logger Validation (Cable Required)

When you have an OpenPort 2.0 (or compatible) cable:

1. Log the relevant parameters while driving
2. Compare logged values against what the definition predicts
3. Key validations needed:
   - **X-tau trigger conditions** — Log during tip-in/tip-out, look for transient enrichment
   - **Knock learned values** — Confirm `0xFFFF8800` tracks IAM
   - **Gear determination** — Compare calculated gear vs actual
   - **Fan thresholds** — Log ECT vs fan state (candidate @0xCE4B0 unconfirmed)

### 3. Table Hunting

Follow the [[Methodology]] to find tables we missed. The dispatcher cross-reference found 238 additional 2D tables — many still need individual validation and labeling.

**Workflow for new tables:**
1. Find and physically validate the table
2. Create a **letter file** (e.g., `0.3.1d`) with ONLY the new tables
3. Submit for review before merging into the next full revision
4. Never modify existing released files (project convention)

### 4. Documentation

- Fix errors you find in this wiki
- Expand table descriptions with your findings
- Add logger validation results
- Translate methodology insights to other Denso ROMs

### 5. Disassembly

The Ghidra project is set up (see [[Ghidra-Guide]]). Open tasks:
- RAM mirror mapping (15,302 refs to resolve)
- 0xD0000 investigation (20 code refs, unclear structure)
- SSM dispatcher tracing
- Knock sensor A/D input path

---

## Project Conventions

These aren't optional — they keep the project coherent:

1. **Old files are never renamed or overwritten.** New revisions get new filenames.
2. **Letter files** contain only new tables, for validation before merging.
3. **Every table** gets a confidence label: HIGH, MEDIUM, LOW, or (Unverified).
4. **Physical validation is mandatory.** No table enters a definition without ROM byte verification.
5. **Emissions equipment stays.** No contributions that defeat catalytic converters, EVAP, EGR, or O2 sensors.

## Where to Discuss

- **RomRaider Forums** — Subaru ECU Analysis subforum
- **GitHub Issues** — For this wiki/repo (when published)

## License

All definitions and documentation are provided for open community use. If you build on this work, credit the project and keep your contributions open as well.
