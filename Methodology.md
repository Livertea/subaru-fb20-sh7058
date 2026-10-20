# Methodology

How tables are found and verified. This is the reproducible core of the project — the part that transfers to other ROMs.

## 1. Table hunting

**Directory decode first.** The ROM's table directory is parsed for entries (address, dims, type). Then the directory is distrusted: dims validated against raw bytes, 0x7FF8 sentinel rejected, checkerboards flagged, collisions resolved (zero-collision invariant — no two tables may claim the same bytes).

**Dispatcher tracing.** Two table-lookup dispatchers found: main @0xBE964 (~479 call sites), secondary @0xBEA18 (~182 sites, type byte selects data type — type-0 = float32). Every call site is a table reference. The re-triage pass against fully-resolved call sites found 101 tables the directory hunt missed.

**The four indirection layers.** A "zero references" verdict must check all four before concluding dead: (1) direct literal ref to data, (2) ref to the dispatcher record, (3) pointer tables holding record addresses (e.g. VE via @0x6D89C), (4) code referencing the pointer tables. The project's own "VE is dead" verdict was wrong on layers 3–4. The 11% unreferenced figure is an upper bound.

## 2. Verification

**Emulator execution.** The SH-2A interpreter runs the real dispatcher entry points against candidate tables: exact u16 on-axis, correct interpolation off-axis. Taint tracking follows ROM values into live code.

**Evidence-carrying handoffs.** Every claim ships as Claim / Confidence / Evidence / Ruled-out / Falsification. Interpretation is separated from evidence structurally.

**Independent re-reads.** A verifier re-reading the same report adds nothing; re-reading ROM bytes directly catches errors. Any claim changing a shipped definition gets an independent re-read.

**Falsification tests.** Where possible, claims get a test that would break them (16-cell taint falsification on the knock→timing path; the E/F blend threshold at load 250).

## 3. Blind-portability test

The methodology was validated by running it blind: the FA20 BRZ ROM was mapped from the binary alone (38/38 axis addresses exact, 7/7 dims exact against the public def afterward), and the EP5I200F 2013 ROM was ported at 91% in ~48 minutes. See [Porting-Guide](Porting-Guide.md).

## 4. What the method can't do

- **Scalings** the dispatcher ignores (record scale/offset are metadata) — VE and warmup-trim scalings are computationally unresolvable; they need dyno/MAF data
- **Dead application paths** — emulation validates logic, not liveness (@0x42700)
- **Hardware-only logic** — the SH7058's knock peripheral does in silicon what the ROM doesn't show
- **The fuel PW path** — unidentified after 11+ dead ends, mapped to a trim path 10-19 (FUELMAIN: no load/MAF/RPM term), main computer still unidentified; the method's honest limit, narrowing

## 5. Anti-hallucination controls

Full treatment: [Avoiding-Hallucinations](Avoiding-Hallucinations.md). Short version: byte-grounding, zero-collision invariant, call-site evidence, mechanical reject audits, confidence tiers, owned misses, documented dead ends, adversarial screens, and permanent skepticism.
