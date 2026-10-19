# How We Try Not to Hallucinate

This definition was built almost entirely by AI agents, supervised by one human with a hex editor and a healthy distrust of everything the agents produced. That makes hallucination — confident, plausible, wrong output — the single biggest risk to this project. A wrong table in a tuning definition isn't a typo; it's a value someone might flash into their ECU.

This page documents the controls we use to keep that from happening. They're not perfect. They're the reason the definition carries confidence labels instead of pretending to be certain.

## The core rule

**Every claim bottoms out in bytes or code, or it's labeled as speculation.**

A table exists because its bytes are at an address and the bytes decode sanely — not because an agent thought it should be there. A table does something because disassembly shows code reading it — not because its shape resembles a table that does that on another ECU. Anything that can't clear one of those two bars gets a low confidence label and a description that says so.

## The controls

### 1. Byte-grounding

Every table entry was decoded from actual ROM bytes and checked: axes monotonic, data ranges physically plausible, dimensions consistent with the bytes claimed. When the bytes disagreed with an earlier reading, the bytes won. Example: tip-in enrichment was first read as 8×8 in milliseconds, then re-triaged to 16×8 in percent enrichment at 1/256 scaling — because the raw data only made sense that way. Two earlier conclusions were superseded, on the record.

### 2. The zero-collision invariant

After decoding, every table's claimed byte range is checked against every other table's. No two distinct tables may claim the same data bytes. Overlaps that show up are resolved into one of three buckets — shared axis pools, aliased/stale directory entries, or the shared 0xD0000 component — and eliminated. If the invariant can't be satisfied, something is wrong with the decode, not with the invariant.

### 3. Call-site evidence

A table that no code reads is a rumor. Both table dispatchers were traced through disassembly (~479 call sites on the main dispatcher, ~182 on the secondary), and tables are expected to show up in that evidence. 145 of the 318 Misc tables have no call-site evidence — they're in the definition, but flagged low-confidence, because "decodes cleanly" is not the same as "the ECU uses it."

### 4. Disassembly cross-checks

Ghidra work doesn't just find tables — it falsifies them. The re-triage pass rejected 3 bogus/stale directory entries and 1 bad table outright, and corrected 4 byte-level mistakes (wrong addresses, wrong dims). Closure is gated on metrics: unresolved call sites get counted, not hand-waved.

### 5. Mechanical reject audits

Some decodes look right and aren't. Defenses, all learned the hard way:

- **0x7FF8 sentinel** — Denso's padding value. Tables full of it are empty flash, not data.
- **Stale/aliased directory entries** — the directory contains entries that point at garbage or at other tables' data. They get rejected, not decoded.
- **Checkerboard tables** — 16 tables decode into alternating patterns that scream "wrong dimensions." They're flagged *in their descriptions* as suspect and left alone until disassembly confirms the real layout. We don't guess dims.
- **Directory dims are not trusted** — validated against raw bytes every time. The CVT reference def's dims were verified garbage on this ROM, which is why blind porting was abandoned.

### 6. Confidence tiers, not binary claims

The definition ships at roughly 58% high / 12% medium / 30% low confidence. Low-confidence tables aren't hidden — they're labeled, with the reason, in the description field. A tuner can see exactly which tables are load-bearing and which are "decodes cleanly, use at your own risk."

### 7. Owning misses, in public

When something's wrong, it gets fixed and noted — not quietly rewritten. v0.7's changelog records the real fixes: five tables with invalid conversion expressions that made RomRaider throw warnings (and could have saved wrong values), ten tables with wrong scaling, one table re-banked from two colliding 2Ds into a single 3D. The ROM-dump thread citation was corrected from t=19254 to t=19276 when verification showed the old number was an unrelated thread. The pattern is: find it, fix it, say so.

### 8. Negative results are findings

Dead ends are documented so nobody re-hunts them or fills the gap with a guess: no per-cylinder timing tables, no fan tables in the ECM (though fan *thresholds* were later found — also documented), no injector flow scalar on this ROM, no third dispatcher. "We looked, it's not there" is a result.

### 9. Adversarial screens

At least once, the project was asked to "think malicious" — to hunt for defeat devices, timer-gated behavior, test-cell detection. The screen came back clean, and the methodology is on record. If you're going to trust a definition, you want to know someone tried to break that trust deliberately.

### 10. The verification ladder

Nothing in this project is called "verified" until it clears the ladder:

1. **Decoded** — bytes make sense.
2. **Disassembly-supported** — code reads it.
3. **Hardware-validated** — the car confirms it.

Steps 1 and 2 are done for most tables. Step 3 is waiting on the OpenPort cable. Until then, every claim is provisional, and the docs say so.

### 11. Tone and attribution discipline

No "first ever" claims. Exploratory language throughout. Every borrowed thing is credited in [Attributions](Attributions.md); unverified items are marked unverified rather than given fake citations. The AI-built nature of the work is stated openly, not buried.

### 12. Literal cross-references are not enough (the pointer-table lesson)

A table showing zero literal code references may still be live. The ECU reaches tables through **four indirection layers**: (1) direct literal reference to the data address, (2) reference to the table's dispatcher record, (3) pointer tables holding dispatcher record addresses, (4) code referencing those pointer tables. The project's own "all 84 VE tables are dead" verdict was wrong for exactly this reason — literal scans found nothing, but a pointer table at `0x6D89C` feeds every one of them through dispatcher records. **No "dead" verdict is valid until all four layers are checked.** The 11% unreferenced-records figure is an *upper bound*, not a count.

### 13. Permanent skepticism — no label is final

The project's standing rule, in the author's words: *"we always need to be skeptics, even in the long term running projects there are errors."* No CONFIRMED, RESOLVED, or VALIDATED label is ever final — every claim stays open to refutation regardless of how long the project has run. When emulation later proved a "validated" wrapper address was actually data bytes, the finding was corrected on the record, not defended. Confidence tiers (§6) are snapshots, not verdicts.

### 14. The dispatcher ignores record metadata

Emulation proved the lookup code never applies the scale/offset fields carried in dispatcher records — they're descriptive metadata, not computation. Two consequences: (1) a table's *structure* (dims, axes, interpolation) can be validated by emulation while its *scaling* remains a separate physical question (VE and the warmup-trim table both have computationally-unresolvable scalings); (2) record fields must be read as claims to check, not instructions to follow.

### 15. Independent rediscovery as validation

When the project's own methods independently re-derive something a third party later confirms — fan thresholds matching the FSM's thermostat window, base-timing scaling inside the factory idle spec, AVCS scaling identical across three Denso chip generations — that validates the *methodology*, not just the finding. The project maintains a standing list of these convergences; a table that is both emulation-validated *and* independently rediscovered carries two orthogonal evidence lines — the strongest confidence tier in the framework.

## Rules for contributors

If you're adding tables to this definition:

1. **Byte evidence first.** New tables go in a standalone file for review before merging — no exceptions.
2. **Label confidence honestly.** "Decodes cleanly, no call-site evidence" is a complete and acceptable description.
3. **Flag, don't fix, checkerboards.** Suspect dims get a warning in the description, not a guess.
4. **Collisions are blockers.** If your table's bytes overlap an existing table's, one of you is wrong. Resolve it before merging.
5. **Record dead ends.** If you hunted for something and it's not there, write it down.

---

*Part of the EP5GD00A open definition project. Licensed CC BY-NC-SA 4.0 — no commercial use. This is unverified research: nothing here is confirmed until hardware validation. Verify everything.*
