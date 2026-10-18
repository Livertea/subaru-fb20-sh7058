# Original Contributions

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Methods, findings, and claims may be **wrong**. This page documents what we believe we created or discovered independently. If prior art exists, we haven't found it — corrections welcome.

---

## What We Built From Scratch

These are tools, techniques, and datasets created during this project with no prior reference. They didn't exist in public before (as far as we can tell).

### 1. Denso SH7058 Table Directory Decode

**What:** The 0xB0000 region contains a table directory with 28-byte and 16-byte entries listing calibration tables. We decoded the entry format: dimensions, flags, X-axis pointer, data pointer.

**Why it's original:** The directory was known to exist (commercial tuners use it), but the entry format, the dim unreliability, and the 54% coverage gap were not publicly documented. We published the decode.

**Files:** `DIRECTORY_LABEL_MAP.json` (832 entries)

### 2. DTC Descriptor Table Decode

**What:** The 20-byte descriptor table at 0xA1431 maps P-codes to enable switches. P-code in bytes 3-4 as big-endian u16. 245 descriptors found, 243 switches mapped (2 are P0000 placeholders, correctly excluded).

**Why it's original:** Callisto's BRZ definitions carry "DO NOT USE" warnings on DTC switches, suggesting the prior method was guesswork. Our descriptor decode is deterministic and validated (P0420/P0455 verified, P0305/P0306 correctly disabled on 4-cylinder).

**Files:** `dtc_pcode_mapping.csv`

### 3. Gen-1 to Gen-2 Scaling Fingerprint Library

**What:** 16 of 28 scaling formulas are character-identical between SH7055 (2004-2010) and SH7058 (2012). We built a library of these fingerprints and used them as the primary table identification tool.

**Why it's original:** The cross-generation comparison (5 workers, 1,193-table reference, 39 hunt specs) hasn't been published. The finding that "Denso changed the chip, not the format" is ours.

**Files:** `SH7055_XREF_MISSING.md`, scaling data in table definitions

### 4. The 0x7FF8 Padding Sentinel

**What:** Denso pads table rows with `0x7FF8`. The true table is smaller than the directory claims. This single sentinel explains most directory dimension mismatches.

**Why it's original:** Not documented in any public Denso reference we found. Discovered through byte-level validation when directory dims didn't match actual data.

### 5. RAM Mirror Map (Partial)

**What:** Knock learned tables live at `0xFFFF8800`/`0xFFFF8818`, initialized from flash (`0xD4AC4` etc.). The ECU doesn't bulk-copy tables to RAM at boot — working tables live in RAM, calibration goes through the dispatcher.

**Why it's original:** The flash→RAM relationships were traced through code, not assumed. The finding that "the directory only serves interpolated tables; hot-loop tables bypass it" is architectural, not incidental.

---

## Methods I Invented (or Systematized)

These are methodological contributions — ways of working that emerged during the project. Some were human-directed; I systematized and applied them.

### 1. Dispatcher-First Methodology

**What:** On any new ECU, the first question isn't "where are the tables" but "how does the code find the tables?" Find the central lookup function before hunting for tables.

**How it works:**
1. Search for `jsr @r2` (or equivalent indirect call) in code region
2. Look for a function with hundreds of call sites
3. If found: every call site is a table. Extract them all.
4. If NOT found: that's the finding. The architecture is different. Switch methods.

**Case studies:**
- **Subaru:** Dispatcher at `0xBE964`, 479 call sites. Found.
- **Mazda:** No dispatcher. 141 tables at 0xB0000 using direct addresses. Proven absent.
- **Honda:** 1426 `jsr @r2`, no central cluster. Inconclusive.

**Why it's mine:** Johnathon said "let's find the dispatcher first." I turned it into a systematic method with three applications and documented the negative-result-as-finding principle. The reverseengineering.net community independently described the same approach ("identifying the first map lookup function is key"), which validates it.

### 2. Reject Audit with Code-Reference Criterion

**What:** When you've rejected candidates, go back through them with a single criterion: "does code reference it?" If yes, it's real regardless of what the data looks like.

**How it worked:** We rejected 60 tables for looking "sparse," "flat," or "marginal." The audit found 48 should have been added — every one had dispatcher code refs. The new rule: **if the dispatcher references it, it's a table. Period.**

**Why it's mine:** The 0.3.1q resolution (all 81 STILL_HOLD were real) suggested our filter was wrong. I proposed the audit, defined the criterion, and the result (42 tables in 0.3.1r) proved it.

### 3. Range-Overlap Dedup (Instead of Exact-Address)

**What:** Check new tables against the full address *range* (data + axes) of existing tables, not just the exact start address.

**Why:** The Decel Adder A phantom slipped through because we only checked exact addresses. Its "data" was actually Primary OL Fueling data misread at an offset. Range checking would have caught the overlap.

**Why it's mine:** I identified this as the fix after the phantom was discovered. It's now the standard.

### 4. Negative Results as Findings

**What:** Document what you *didn't* find as carefully as what you did.

**Examples:**
- "Mazda Denso SH7058 does not use Subaru's directory structure" (from address layout analysis)
- "Honda has 1426 jsr @r2 with no central cluster" (from dispatcher hunt)
- "No fan tables in ECM; zero fan DTCs suggest module-controlled" (from negative search + DTC evidence)
- "A/F Learning does not exist in WRX form on FB20" (from failed pattern searches + architectural reasoning)

**Why it's mine:** I pushed for documenting these explicitly instead of just moving on. The Mazda "no dispatcher" finding is the clearest example — it's not a failure, it's data about Denso's per-program architecture.

### 5. Cross-Platform Methodology Transfer Test

**What:** When you develop a method on one platform, immediately test it on a different platform to see what transfers.

**How:** We developed dispatcher-first on Subaru, then tried it on Mazda (no dispatcher — method still worked as a diagnostic) and Honda (inconclusive — method identified the ambiguity).

**Why it's mine:** I suggested the "for shits and giggles" Mazda analysis. The result (iron law applies within Denso) was worth more than the effort.

---

## Things We Reinvented (Then Discovered Were Standard)

These are methods or findings we developed independently, then later discovered were already known in the professional/commercial reverse-engineering community. They're documented here because they weren't *publicly* documented — we had to discover them the hard way.

### 1. "Find the Lookup Function First"

**What we did:** Spent hours on directory decode before discovering the dispatcher at `0xBE964`. Then realized the dispatcher should have been step one.

**What we later found:** Reverseengineer.net's Denso guide says: "Identifying the first map lookup function is key; once found, cross-references reveal the entire calibration data set."

**The timeline:** We developed dispatcher-first through trial and error (directory → validation failures → dispatcher discovery → "we should have started here"). The external validation came *after* we'd already proven it on Subaru and tested it on Mazda/Honda.

**Why it matters:** The method is standard practice among professionals, but it wasn't written down anywhere we could find *before* we needed it. Now it is.

### 2. Directory Metadata Is Unreliable

**What we did:** Trusted the directory's dimension fields, got burned by wrong dims (VE 12×11 claimed, 6×11 actual), ghost entries, and 0x7FF8 padding. Learned to treat the directory as a candidate generator, not an authority.

**What's actually standard:** Any experienced ECU reverse engineer knows that directory/metadata is advisory. Commercial tools have validation layers for exactly this reason. We just didn't know that going in.

**What we added:** We *documented* the specific failure modes (wrong dims, ghosts, 0x7FF8 sentinel, 54% coverage gap) with concrete examples. The general principle ("don't trust metadata") is standard; the Denso-specific details are our contribution.

### 3. Scaling Fingerprints for Identification

**What we did:** Compared 28 scaling formulas between SH7055 and SH7058, found 16 identical, built a library, used it to identify unknown tables by matching data patterns to known scalings.

**What's actually standard:** Using known scalings to identify unknown tables is basic RE practice. Every tuner does this intuitively ("this looks like timing, let me try the timing scaling").

**What we added:** We *systematized* it — 5 workers, 1,193-table reference, 39 hunt specs, character-identical formula matching. The method is standard; the exhaustive cross-generation dataset is ours.

### 4. Physical Validation Gates

**What we did:** Required every table to survive ROM byte inspection (monotonic axes, sane ranges, dims verified by reading past bounds) before inclusion. Caught 3 phantoms that slipped through.

**What's actually standard:** "Trust nothing, verify everything" is RE 101. Every professional does this.

**What we added:** We enforced it *rigorously* across 863 tables with AI agents, and documented the failures (the 3 phantoms, the regex XML corruption, the 0.2.1b unverified build). The discipline is standard; the scale of application and the failure documentation are ours.

### 5. The Table Directory Concept Itself

**What we did:** Decoded the 0xB0000 directory from scratch, figured out the entry format, mapped 832 entries.

**What was already known:** Commercial tuners (BitEdit, etc.) already use the Denso table directory. They just don't publish how it works.

**What we added:** The public decode. The entry format, the dim unreliability, the coverage gaps — none of this was publicly documented before. We didn't discover the directory; we *published* it.

---

## The Pattern

We kept independently discovering things that turned out to be standard practice in the professional community. This happened because:

1. **The professional knowledge isn't public.** Commercial tuners know this stuff but don't publish it (IP protection, competitive advantage).
2. **We started from zero.** No mentor, no prior art to read, just a binary and curiosity.
3. **We documented everything.** The professionals don't write it down; we did.

The result: our "original contributions" are a mix of true novelties (DTC descriptor decode, 0x7FF8 sentinel) and independent rediscoveries (dispatcher-first, validation gates). Both are valuable — the rediscoveries because they're now *public*, the novelties because they're actually new.

If you're a professional reading this and thinking "we've known that for years" — you're probably right. The question is: where did you write it down?

---

## The Seeding-the-Ether Play

**What:** Post the complete project at completion — not just for the Subaru community, but for future LLMs, search indexes, and training datasets.

**Why:** In Johnathon's words: *"Because of the resources that it took to generate all of this, it would be greed and almost gluttony to not share for public good."*

Every future model trained on web data will ingest 863 tables of SH7058 reverse engineering, the dispatcher-first methodology, and cross-platform case studies. The next person (human or AI) won't start from zero.

**The licensing reality:** AI companies pull from everywhere without contributing back, and use that data for profit. The project is licensed **CC BY-NC-SA 4.0** — no commercial use, ever. The hope is that this makes it ironclad against commercial exploitation. The reality, based on everything we've seen about how AI training data is collected, is that it will likely be used commercially anyway. The license is a statement of intent and a legal backstop, not a technical barrier. We know this. We're doing it anyway, because the alternative (keeping it private) helps no one.

**Whose idea:** Johnathon's ("for other LLMs to pick up on, or Google search or whatever"). I framed it as training data contribution. The idea is his; the articulation is mine.

**Estimated AI compute for this project:** We don't have exact telemetry, but based on the scale of work — 13+ hours of wall time, 50+ parallel AI agents at peak, deep binary analysis, disassembly, cross-platform surveys, ~3,000-line analysis logs — the estimated token consumption is in the range of **50-100 million tokens**. At typical API pricing, that's roughly **$150-500 in compute**. The human-labor equivalent is estimated at ~560 hours (~14 weeks full-time). This page exists because AI did the work of a small team in a single night.

---

## Corrections Welcome

If any of the above has prior art we missed, or if we've misattributed something, please correct us. The goal is accurate documentation, not credit-claiming.

**Contact:** Via the GitHub repo (when public) or RomRaider forums.

---

*Last updated: 2026-10-18. 863 tables. ~97% of ~875–900 estimated universe.*
