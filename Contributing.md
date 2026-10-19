# Contributing

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---


## Maintenance Note

**The original author is taking a break from active work on this project**, but plans to return for hardware validation once the cable arrives (~Nov 6, 2026).

In the meantime: collaborators wanted. If you find errors, want to expand the coverage, or want to help validate — please do. That's why it's public.

The hope is simple: that something here is of use to someone, somewhere — maybe even ten years from now. A kid with a beat-up 2012 Impreza and no budget finds this repo, and the tables save them months. The methodology gets reused on ROMs nobody's touched yet. That's why it's public.

---

## How to Help

### 1. Validate Tables in RomRaider (No Cable Needed)

The easiest way to contribute:

1. Download the definition files (see [Definitions](Definitions.md))
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

Follow the [Methodology](Methodology.md) to find tables we missed. The dispatcher cross-reference found 238 additional 2D tables — many still need individual validation and labeling.

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

The Ghidra project is set up (see [Ghidra-Guide](Ghidra-Guide.md)). Open tasks:
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
6. **No firmware blobs.** Never commit ROM binaries or firmware images. Work from your own pulls.

## Where to Discuss

- **RomRaider Forums** — Subaru ECU Analysis subforum
- **GitHub Issues** — [Livertea/subaru-fb20-sh7058](https://github.com/Livertea/subaru-fb20-sh7058/issues)

## License

All definitions and documentation are licensed under **CC BY-NC-SA 4.0** (Creative Commons Attribution-NonCommercial-ShareAlike).

- ✅ Use, share, and adapt freely
- ❌ **No commercial use** — this work cannot be sold or built into paid products
- ✅ Derivatives must be shared under the same terms
- ✅ Credit the project

See the `LICENSE` file for the full legal text.
