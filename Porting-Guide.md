# Porting Guide: How This Work Transfers to Other Engines and Cars

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

This page answers one question: *I have a different car — how much of the EP5GD00A work helps me?*

Entries are ordered from **closest relative** (nearly a copy-paste job) to **furthest** (only the method survives). For each: what transfers verbatim, what must be redone from scratch, where to find public definitions and ROMs, and an honest difficulty estimate.

## The Iron Law (read this once, it applies to every entry below)

> **Port the METHOD. Never the ADDRESSES.**

This was learned the hard way: the CVT reference definition's addresses were verified garbage on the MT ROM, and blind porting was abandoned on day one. Across Subaru, Honda, and Mazda, **zero** same-named tables shared addresses (the single exception in all our cross-checks: speed limiters at `0xF8E44` between EP5GD00A and the Forester FB25). What *does* transfer: table **names**, **scalings**, **axis identities**, **DTC descriptor layouts**, and above all the **workflow** — dispatcher-tracing, directory-decode, reject audits ([Methodology](Methodology.md)).

If you take one sentence from this page: find the lookup function first, cross-reference everything, validate against raw bytes, and never trust an address you didn't earn.

---

## 1. Other FB20 SH7058 — 2012–2016 Impreza / Crosstrek XV (EASIEST)

**Relation:** Same engine family, same chip, same Denso firmware generation. This is the closest possible relative — some of these ROMs may differ from EP5GD00A by only calibration values.

**What transfers verbatim:** Nearly everything structural. The directory format, both dispatchers' *mechanisms* (not addresses), the DTC descriptor layout (20-byte entries, P-code in bytes 3–4 BE), the scaling formulas, the axis pool layout. Expect a large fraction of table *names* to exist with identical dims and scalings.

**What must be redone:** Every address. Even between two FB20 ROMs, assume nothing ports until verified — the CVT/MT lesson applies *within* the family too.

**Known public defs:**
- **Merp/SubaruDefs, Alpha branch (free, downloadable):** FB20 ECUFlash defs `EP5F300H`, `EP5F301I`, `EP5I200F`, `EP5L001G`, plus `EP5G601B` (2012 EDM Impreza 2.0i CVT SH7058 — header-confirmed). Also `EP5F300S`/`T`/`r`, `EP5L000P`, `EP5N000Z` (XV 2.5i) were listed but not opened. Caveat: Alpha-branch community defs; the repo README itself warns of errors.
- **Local reference copies** (in this project's `reference_defs/`): `RR_EP5G211a.xml`, `RR_EP5G300b.xml`, `RR_EP5G401z.xml` (Forester/XV FB-family RomRaider defs). Note: `RR_EP5G300b.xml` (EDM Forester 2.0i FB20 CVT) could not be independently corroborated in public listings — treat its provenance with caution.

**Known ROM bins:** Effectively none free. The only public FB20 SH7058 ROM dump remains the 2022 RomRaider forum thread t=19254 dump (the one this project used). A three-agent hunt across GitHub, forums, and commercial sources found **zero free FB20 ROM bins on the open web**. Paid path: **avtochip.ru** lists a 2012 Subaru XV 2.0 Denso SH7058 entry (`PF_11CYH4N` / `LEP5F300`, 1024kb) — note `PF_11CYH4N` is the *same internal ROM ID* as EP5GD00A, making LEP5F300 the closest documented sibling ROM. It's a paid chiptuning file service (price on request). **XRT (Xtreme Racing Tuning)** e-tunes FB20/FB25 NA (e.g. 2013 Crosstrek FB20 5MT ROM `EP5IA00B`) by email — proof that private defs exist, nothing downloadable.

**Difficulty:** 1/5 *if you have the ROM*. Directory-decode first, diff the directory against EP5GD00A's, then dispatcher-trace only the entries that don't match. A weekend project with the method in hand.

---

## 2. FB25 — Forester 2.5 SH7058 (2012–2013)

**Relation:** Same chip, same Denso firmware generation, bigger displacement sibling. The most likely future project for this author (a Forester FB25 donor ECU is the long-term swap candidate).

**What transfers verbatim:** The method wholesale — directory format, both dispatcher mechanisms, DTC descriptor layout, most scaling formulas. Table names largely overlap (same Denso template).

**What must be redone:** Every address. Verified: **0 of 23 same-named tables share addresses** between EP5GD00A and the FB25 defs, with the lone exception of speed limiters at `0xF8E44`. Treat it as a fresh ROM that happens to speak the same language.

**The interesting divergence:** The FB25 definition **has an "Injector Flow Scaling" scalar** (`EP5G211A` ECUFlash def, verified at `0xcc7f0`; also present in `EP5F400C` at `0xcae80`). Every search method failed to find any equivalent on EP5GD00A. Either the FB20 hides it differently or Denso genuinely omitted it — an open question worth answering by comparing the two ROMs side by side once an FB25 bin surfaces.

**Known public defs:**
- **Merp/SubaruDefs, Alpha branch (free):** `EP5G211A` (2012 USDM Forester 2.5, **MT** — best lead: manual like the author's car, full tip-in suite), `EP5F400C` (2012 USDM Forester 2.5X MT), `EP5D201D`, `EP5D202N`, `EP5G401Z` (2013 EDM XV 2.5i CVT). Local copies of the `EP5G211A` and `EP5F400C` ECUFlash defs are in this project's `reference_defs/`.
- RomRaider: `2011EDMForester2.0XAT-EP5D004L_v2.xml` (Forester 2.0X AT).

**Known ROM bins:** None free anywhere public (same hunt as above). No FB25 `.bin` on GitHub or Internet Archive.

**Difficulty:** 2/5 with a ROM in hand. Same playbook as the FB20, plus a genuinely interesting research question (the missing injector scalar).

---

## 3. FA20 — BRZ / 86 / FR-S

**Relation:** Different engine (FA20 boxer, Toyota co-development), related Denso ECU. Same vendor habits, different calibration universe.

**What transfers verbatim:** The method, the 32BITBASE template familiarity, many scaling formulas, the DTC descriptor approach, the checksum-search technique. The *shape* of the work.

**What must be redone:** Everything structural. Different engine means different table set (port+direct injection on later FA20s, different AVCS, different torque model). Directory and dispatcher addresses will differ; verify the dispatcher mechanism still holds before assuming it.

**Known public defs (local reference copies in `reference_defs/`):**
- `ZA1JS10C_v7.xml`, `ZA1JS20G_v1.xml`, `ZA1JU01A_v22.xml` — from callistoprime/brz-defs, 2,400–2,700 tables each. The BRZ community is the best-documented Denso reverse-engineering scene; their defs are the richest structural reference available.

**Known ROM bins:** The BRZ scene shares ROMs more freely than the FB scene (check the RomRaider forums and brz-defs community spaces).

**Difficulty:** 3/5. You'd be re-running the full methodology, but on a platform where the community already did half the scouting for you.

---

## 4. Turbo EJ, Gen-1 SH7055 — 2006–2014 WRX / STI

**Relation:** Older Denso chip (SH7055 vs SH7058), same 32-bit SuperH lineage, same 32BITBASE definition template. Denso changed the silicon, not the format.

**What transfers verbatim:** A surprising amount. Verified against the reference defs: **float32BE axes in both generations, and 16 of 28 scaling formulas are character-identical** between the SH7055 WRX defs and EP5GD00A. Knock architecture is unchanged Gen-1→2. The dispatcher-tracing method should apply directly — the lookup-function-first approach is generation-agnostic.

**What must be redone:** Addresses (as always), plus genuine care around turbo-specific subsystems this project never touched: boost control, per-gear compensations, turbo-specific DTCs, the whole boost-error machinery. The tip-in thread this project contributed to is *all* turbo 16/32-bit ECUs — that community knowledge runs the other direction here and is worth mining.

**Known public defs (local reference copies in `reference_defs/`):**
- `2011_WRX_ecu_defs-REV8.xml`, `2012UsdmWrxMt-AE5K500L_v1.xml` — SH7055 Gen-1, from td-d/SubaruDefs Alpha.

**Known ROM bins:** The WRX scene is the best-supplied in Subaru-land; ROMs circulate in the usual community places (this project deliberately did not collect them — see the scouting notes in [Methodology](Methodology.md)).

**Planned experiment:** A clean Gen-1 decode-from-scratch to measure the method's hit rate against an existing public def. **Blocked: no Gen-1 ROM binary in hand.** (Note: the tip-in thread's "16-bit ECUs" — 2002–2005 WRX — are a *different architecture*, not Gen-1 for our purposes. Gen-1 here means 32-bit SH7055.)

**Difficulty:** 2/5 for the method transfer, 4/5 for turbo-subsystem interpretation. The experiment is the honest way to find out.

---

## 5. Honda SH7058 — same silicon, different vendor logic

**Relation:** Same physical chip (Denso SH7058), completely different manufacturer (Keihin/Honda). The silicon is identical; everything Denso-specific is absent.

**What transfers verbatim:** The *meta-method*: directory-decode-first, physical validation against raw bytes, reject audits, bank-finds-before-merging. The Ghidra SH-2 setup and Capstone notes transfer directly (same silicon).

**What must be redone:** The directory format is different: **`[u8 xdim][0x66][u8 ydim][0x66][u32 meta][u32 x_axis][u32 y_axis][u32 data]`** (20-byte entries), and Honda uses **uint16BE axes, not float32BE**. An early "no directory" conclusion here was flat wrong — the directory spanned ROM `0xA454–0xDB8C` all along, and directory-decode-first took the project from 4 tables to **92** (`RR_HONDA_RRB_A080_0.3.xml`, banked, not merged). The scalar directory format is still uncracked; 66 of 92 tables are unidentified; fuel/VTC scalings unverified. The meta field meaning is unknown.

**Known public defs:** Ours is an open one — we haven't found another public Honda SH7058 definition, but verify before repeating that. BitEdit has no Honda SH7058 module.

**Known ROM bins:** The project's test ROM (`37805-RRB-A080-M1.decrypted.bin`, 992KB partial) — provenance is the project's own.

**Difficulty:** 3/5 to replicate the 92, 5/5 to finish (scalar directory + 66 unknowns + zero hardware validation). The honest status: a solid start, not a finished definition.

---

## 6. Mazda SH7058 — MX-5 NC (same silicon, opposite architecture)

**Relation:** Same chip, same vendor (Denso), but a fundamentally different firmware architecture. This is the entry that proves the Iron Law.

**What transfers verbatim:** Almost nothing structural — and that's the finding. The scaling *formulas* may rhyme; the computation-flow documentation approach transfers.

**What must be redone:** Everything. The MX-5 NC (CAL `LFG7EG`) has **no dispatcher and no directory** — proven by layout analysis, not by failure to find them. Tables live at hardcoded addresses. The method here inverts: subsystem-by-subsystem analysis instead of lookup-function-first. Also opposite axis behavior: 504 unique axes with only 7 shared, vs. Subaru's aggressive axis pooling. LibreTuner's 875-table count is inflated; the real map count is ~300–400. The only surviving commonalities with Subaru: throttle tables near `0xF0000` and packed DTC data near `0xDD250`.

**Known public defs:** LibreTuner (community), no RomRaider def.

**Difficulty:** 4/5. You're doing archaeology without the map — which is exactly why the Mazda result matters: it defines the boundary of where this methodology works.

---

## 7. Saab Trionic and beyond — what the rest of the world taught us

**Relation:** Different vendor, different chip, different decade. Included because the cross-pollination ran both directions.

**What transferred *to* this project (steals, credited):**
- **Masked-pattern checksum searching** — from the Saab/etnoy scene's approach to finding checksum routines by instruction idiom rather than by address. This is what located EP5GD00A's checksum at ROM `0x152` and proved the ECU doesn't verify at boot.
- **Computation-flow documentation** — algorithms documented as dataflow (fuel/timing/DBW chains), not just table lists. Directly modeled on the Trionic community's reference style.
- **LaTeX reference skeleton** — modeled on the Saab t7ref document structure.

**What didn't transfer:** The Saab scene never needed dispatcher-tracing at all — Trionic ROMs ship with embedded symbol tables. Their 25 years of practice validated the *approach* (find the lookup, cross-reference everything) a third time, from the opposite direction: when the symbols are present, you don't need the hunt; when they're absent, the hunt is the only way.

**Beyond (not attempted):** Gen-3 Denso (RH850) — zero open-source definitions exist anywhere we looked. The method *should* transfer (find the lookup function first), but that's a hypothesis, not a result. Toyota/Lexus Denso ECUs of the SH7058 era are the obvious next frontier and completely unexplored by this project.

---

## Practical notes for porters

1. **Get the ROM first.** Every entry above is gated on a binary. Without it you're theorizing. (The FB20/FB25 bin shortage is the single biggest blocker in this entire space.)
2. **Collect reference defs before touching the ROM** ([Methodology](Methodology.md) Stage 0). Names and scalings transfer; use them as search patterns, never as addresses.
3. **Run the reject audit on yourself.** The 0.3.1r audit (48 of 60 rejects should have been added) is the cheapest lesson in this wiki: your aesthetic filters are wrong, the code references are right.
4. **Mind the license.** This wiki and the EP5GD00A definition are **CC BY-NC-SA 4.0 — no commercial use, ever**. That license travels with anything derived from this work. If you're building a commercial tuning product on top of these methods, that was never on offer.
5. **Verify on hardware.** Nothing here has been. The cable, the logger, the wideband — that's where a definition becomes real. Everything before that is educated guessing with receipts.

---

*Part of the [EP5GD00A project wiki](Home.md). Built primarily by AI agents; errors expected. Trust nothing, verify everything.*
