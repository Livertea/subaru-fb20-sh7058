# Collaborate

> **⚠️ UNVERIFIED RESEARCH — READ THIS FIRST**
>
> Everything here is **exploratory and unverified**. Table counts, addresses, scalings, labels, and identities may be **wrong**. No table has been validated on real hardware. No ROM has been flashed. Do not tune, flash, or make decisions based on this data until it is independently verified.
>
> This project was built primarily by AI agents. Errors are expected. **Trust nothing, verify everything.**

---

## What this is

A from-scratch RomRaider definition for the **2012 Subaru Impreza 2.0i (FB20, 5MT)** ECU — CAL ID **EP5GD00A**, Denso SH7058. No public definition existed, so one got built: **863 tables in v0.7**, roughly ~97% of the estimated table universe, mapped by decoding the ROM's table directory, tracing both of its table-lookup dispatchers through disassembly, and diffing everything against the disassembly to prove no phantom tables.

The short version of how it went: fell down the rabbit hole, stayed down there for about two weeks, came back up with 863 tables and a wiki.

The motivation, in the author's own words: *"because of the resources that it took to generate all of this, it would be greed and almost gluttony to not share for public good."* So it's all going public. That's where you come in.

## Current state (Oct 18, 2026)

- **v0.7 definition: 863 tables** — 3D/2D/1D maps plus 243 DTC switches, all with evidence annotations
- **Disassembly-verified:** two dispatchers traced (@0xBE964 and @0xBEA18), checksum routine found and proven *not* to gate flashing, knock subsystem traced, X-tau transient-fuel filter fully decoded
- **Zero hardware validation.** Nobody has plugged into a car yet. The OpenPort cable lands ~Nov 6, 2026, and everything before that date is educated guessing with receipts
- **Repo currently private** while personal info gets scrubbed; reopens when the author says so

## What's actually needed

Roughly in order of value:

1. **Hardware validation.** Someone with a Tactrix/OpenPort 2.0 (genuine or clone) and a 2012–2013 Impreza/Crosstrek FB20: pull your ROM, check the CAL ID, open the definition in RomRaider, and tell us what breaks. Then logger-validate: do the tables' values move the way the labels say they should? This is the single highest-value contribution possible — it turns speculation into knowledge.
2. **Disassembly help.** The untraced targets are listed in [Open-Threads](Open-Threads.md): final fuel summation order, knock/timing summation code, the tip-in trigger routine. If you read SuperH or just want to learn, these are well-scoped puzzles with known addresses.
3. **Logger PID validation of the misc-table hypotheses.** [Table-Reference](Table-Reference.md) has a new batch of speculations about the 318 unknown tables (feedback-controller error maps, fan thresholds, delta-T timing corrections…). Each one lists what logger data would confirm or kill it. Got a logger and a free afternoon? Pick one.
4. **Honda SH7058 scalar-directory decode.** The sibling project (Honda Keihin SH7058, 92 tables so far) has its scalar directory still uncracked. Different ROM, same methods.
5. **Eyeballs.** Read the wiki, find the wrong thing, say so loudly. Every correction improves the definition.

## How the workflow works

A few conventions that keep the project sane — please follow them:

- **Versioning is semantic-ish `X.X.X`:** `0` = in development, second digit = stage, third = revision. Old files are **never renamed** — history stays readable.
- **Numbers are merged, letters are proposals.** `0.7` is the current definition. `0.3.1u` was a proposal file: 101 new tables banked for review before merging. New findings go in a **new letter file**, never merged silently.
- **Bank finds, ask before merging.** If you find tables, bank them in a letter file with evidence. Merging into a numbered release is a deliberate, reviewed step.
- **Every claim carries evidence.** Directory entry, call-site address, disassembly trace, ROM bytes — speculation is welcome but must be *labeled* speculation with a confidence level. See the Misc Musings section of [Table-Reference](Table-Reference.md) for the format.
- **Negative results are findings.** "Searched for X, it's not there" is worth writing down. The wiki has a track record of them.

## License

**CC BY-NC-SA 4.0 — no commercial use, ever.** Share it, remix it, build on it, but don't sell it or anything derived from it, and share alike. This is a statement of intent as much as a legal backstop: the work exists for the public good, not for someone's product.

**No firmware blobs, ever.** Definitions, documentation, methods, and analysis are public. ROM binaries are not distributed through this project — everyone works from their own pull or from publicly available sources. This is both a legal line (DMCA §1201(f) interoperability) and a practical one.

## What "done" looks like

1. Hardware-validated definition (logger-confirmed tables, corrected labels)
2. Finished wiki (methods documented well enough that someone else could repeat the whole thing on another ROM)
3. Everything public, in the ether, findable by the next person — human or otherwise — who falls down this particular rabbit hole

## Get in touch

Interested? Reach out — Threads **@brazenbuilds** DMs are open. Say what you'd like to work on (or just what caught your eye), and you'll get pointed at the right thread. No credentials required; curiosity and a willingness to be wrong in public are the whole job description.
