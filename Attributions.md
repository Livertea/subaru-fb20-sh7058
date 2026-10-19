# Attributions

## Why this page exists

This project took an absurd amount of resources to build — thousands of hours of agent computation, deep research passes, verification sweeps, and one very stubborn human. Every link that led to a windfall is listed here, in an attempt to credit those whose shoulders we stand on. Because of the resources that it took to generate all of this, it would be greed and almost gluttony to not share — and sharing means the credit goes out too.

A note on absence: if you contributed something this project used and you're not on this list, it is not ingratitude. Memory is imperfect, especially a machine's. Drop by [Collaborate](Collaborate.md) and tell us, and you'll be added.

---

## The ROM — without this, there is no project

**The anonymous uploader of the EP5GD00A ROM dump — [RomRaider forum thread t=19276](https://www.romraider.com/forum/viewtopic.php?t=19276)** ("Defintion request: 2012 USDM Subaru Imprerza Sport 2.0i MT", 2022, attachment "12 UDSM Impreza 2.0i Sport MT original.zip")

This entry goes first, above all others. A three-agent hunt across GitHub, forums, and commercial sources found **zero** other free FB20 ROM bins for this CAL ID anywhere on the open web. This single thread — one person deciding to upload their ROM dump — provided the only public binary of EP5GD00A on Earth, and the entire 863-table definition was built from it. Thank you, whoever you are — one zip file turned a wishlist into a working definition.

*(Corrected Oct 18, 2026: an earlier draft of this page cited t=19254 — that's an unrelated Outback thread. The real dump is t=19276.)*

---

## The Methods — how we figured out how to do this

**[DSchultz](https://www.romraider.com/forum/viewtopic.php?f=40&t=6303)** — "How To: Get started with IDA and disassembly" (RomRaider, 2010), plus the [cnblogs mirror](https://www.cnblogs.com/shangdawei/p/4552160.html) by shangdawei. The foundational SH7058 reverse-engineering guide: 16/32-bit table layout formats, type-byte bit definitions, the "have a quest" philosophy, RAM base addresses, literal-pool patterns. The project's Ghidra setup is a direct port of his approach — and our methodology page is documented as its descendant. Thanks for writing the guide that started the whole generation.

**[DSchultz — XmlToIdc](https://www.romraider.com/forum/viewtopic.php?f=40&t=6234)** (documented by **NSFW**: *"use XmlToIdc to tell IDA the names for the tables and extended parameters"*). The XML→IDC label-importer concept — table names and extended parameters pushed into the disassembler as labels — was reconstructed from NSFW's docs and is being built as our Ghidra label-importer. Every disassembly task we ran would have been faster with it. Thanks for the tool idea we're still catching up to.

**[NSFW](https://www.romraider.com/forum/viewtopic.php?f=40&t=6295)** — 1D-constants-via-FPU-load technique ("I knew those values from logging Fuel Pump Duty Cycle"), the SSM-base-from-ECU-ID method, the SH7055/SH7058/SH72531/SH72543R register listing, and — with **XSWREX** — the per-table header docs that defined Gen-1 metadata. His logging-informed disassembly technique is queued for identifying scalar blocks P1–P5 once hardware data arrives. Thanks for the tricks that only come from doing the work.

**[reverseengineer.net — Denso & Continental ECU Reverse Engineering Guide](https://reverseengineer.net/denso-continental-ecu-reverse-engineering/)** (also their [Bosch MED17/EDC17 guide](https://reverseengineer.net/bosch-med17-edc17-ecu-reverse-engineering/)): *"Identifying the first map lookup function is key; once found, cross-references reveal the entire calibration data set"* — the dispatcher-first methodology, described independently, validating our core method after the fact. They also handed us the Denso generation table (Gen 1 SH7055 / Gen 2 SH7058 / Gen 3 RH850) and the "Denso prefers sum-based calculations" hint that pointed the checksum hunt at a simple sum instead of a CRC. Thanks for publishing what a commercial firm didn't have to.

**[EECTuning.org — Ford EEC lookup-function thread](http://eectuning.org/forums/viewtopic.php?t=17697)**: "Find the Table and Function routines... Once you found the lookups then find the Calibration pointers." The dispatcher-first principle converged independently on a completely different ECU architecture (Ford EEC-IV/V) — the third validation of the method, alongside DSchultz and reverseengineer.net. Thanks for proving the method travels across chips.

**[The Saab Trionic community](https://github.com/etnoy/t7ref)** — etnoy's T7 Technical Reference, the [TxSuite tooling](https://txsuite.org/), Dilemma's "Analyzing Trionic 7" bible, Tomili and General Failure's checksum solvers, Mackan, and [Just4pLeisure's Trionic code](https://os.mbed.com/users/Just4pLeisure/code/Just4Trionic/). Two direct steals: (1) the Saab scene's **masked-pattern checksum search** found our checksum routine at ROM `0x152` and lifted the last flashing blocker; (2) their **computation-flow documentation** is the model for this wiki's Computation-Flow page and the LaTeX reference skeleton. The Trionic symbol-table contrast also proved dispatcher-first was the *correct* approach for an ECU without symbol tables. Thanks for the bible and the burglary material — in the best tradition, we gave credit and carried it forward.

**[ztan — V00C00G FA20 Ghidra learning log](https://www.ft86club.com/forums/showthread.php?p=3616796)** (FT86CLUB). Documented learning Ghidra and SuperH from scratch to diff K00G → S10C → V00C → U01A ROMs — the ROM-diffing-across-versions technique we haven't been able to try (it needs a second FB20 ROM), but it's our highest-value blocked method. Also the model for this project's "learn in public" documentation style. Thanks for showing the work in public, ztan.

**[David Shirley ("Tephra") — EvoEcu wiki](https://evoecu.logic.net/wiki/TephraMod)** ([wiki main page](https://evoecu.logic.net/wiki/Main_Page)), plus [Merlin's 165-page EcuFlash Evo 7-8-9 Tuning Guide (PDF)](https://evoecu.logic.net/mirror/merlin/Merlins%20EcuFlash%20EVO%207-8-9%20TUNING%20GUIDE-V1.6.pdf) and the [ELM documentation (PDF)](https://evoecu.logic.net/mirror/livemap/ELM%20Documentation.pdf). Proof that one dedicated reverser can take a Denso SH-2 ECU from black box to better-than-standalone — our stated aspiration. Two concrete transfers: the **code-injection pipeline** (free flash space → branch-patched code → checksum fix) validated our plan for the ~100KB empty flash region (`0xDFCE8–0xF8E00`), and Merlin's tuning guide is the template for the FB20 tuning guidance we'll write after hardware validation. (The old evolutionm.net ECUFlash sub-forum path is dead — the wiki and mirrors live on.) Thanks for showing that one person is enough.

**[Craig Smith — "The Car Hacker's Handbook", Ch. 7](https://publicism.info/engineering/penetration/7.html)**. The data-visualization technique — render the binary as a grayscale image; calibration tables show as structured patterns — adopted as our recommended technique for finding table clusters the directory decode may have missed. Thanks for the 30-minute method.

**[MHD / dmacpro91 — BMW-XDFs](https://github.com/dmacpro91/BMW-XDFs)**. MHD publishing a *partial* XDF that the community extends by fork validated our own "numbered release + letter files" model — and their honest warning that published XDFs carry "only a fraction of tables defined" is the framing we borrow for our own coverage claims. Thanks for the publishing model.

**[LegacyNsfw — EcuHacks](https://github.com/LegacyNsfw/EcuHacks)**. Rev-matching on downshift built as a ROM patch on stock Denso Subaru ECUs — with new tables, a cruise-switch UI, PID throttle control, and a porting guide. Same NSFW who wrote the RomRaider IDA guide credited above. This is **the template for future custom-feature work on this ECU**: every custom feature follows the same shape (hook + free space + new tables + trigger + repurposed actuator paths), and our disassembly is incidentally mapping exactly those hook points. Thanks for showing what a stock Denso ECU can be made to do.

**[The Speeduino project](https://speeduino.com/wiki/)** (wiki). Speeduino's fuel equation (`PW = REQ_FUEL·VE·MAP·E + accel + deadtime`) is the **null hypothesis** we check the @0x43F64 fuel-hub trace against — Denso's deviations from it are the interesting bits. An open-source ECU the project studied as the "what would a clean-sheet implementation look like" reference. Thanks for the equation everything gets measured against.

**[SECU-3](https://secu-3.org/)**. Their documented state machine (EM_START/IDLE/WORK) is an open-source parallel to our mode-byte discovery — convergent evidence that mode-switched control is universal in this ECU generation. Thanks for the independent confirmation that the pattern is real.

**[rusEFI](https://rusefi.com/)**. Open-source ECU hardware and firmware; another studied architecture in the open-ECU survey. Thanks for keeping the open hardware path alive.

**[The MegaSquirt manuals](https://www.megamanual.com/)**. The long-running DIY ECU documentation corpus — studied as part of the open-ECU survey for table formats and tuning workflows. Thanks for two decades of written-down knowledge.

**[openremap](https://docs.openremap.com/)** (MIT; Pinelo92, now at v-arapidis/openremap-core). Bin identification, confidence tiers (HIGH/MED/LOW/SUSPICIOUS/UNKNOWN), and tunes as portable, auditable "recipes." **Studied from source, not copied** (MIT license is compatible, but the recipe concept was re-derived for our table-level format rather than lifted). Our recipe-format design (`RECIPE_FORMAT_DESIGN_2026-10-18.md`) adapts their confidence-tier idea to table-level recipes with a CAL ID + CVN + sha256 identity gate. Thanks for the recipe concept.

**[Milburn & Timmers — ECU emulation paper]**. Built a custom ECU emulator in roughly a week and recovered UDS keys with it — the **existence proof** for our emulator plan, and the precedent that a purpose-built interpreter beats waiting for tool support (Unicorn doesn't do SuperH; we built our own SH-2A interpreter the same way). Their "validate the emulator against hardware" warning is a standing workbench principle. Thanks for proving it could be done fast.

---

## The Knowledge — facts that saved us from ourselves

**[eqtrian — "Next Problem to be Cracked: Tip-In (Accelerator) Enrichment"](https://www.romraider.com/forum/viewtopic.php?f=15&t=1735)** (RomRaider, 2007–ongoing) and the 100+ posts of contributors across it. The long-running community knowledge base on throttle tip-in enrichment across all turbo 16/32-bit Subaru ECUs — and the reason our tip-in table is resolved the way it is (16×8, X=ECT @0xCEB64, Y=time-since-tip-in, scale 1/256, % enrichment). We've contributed a wiki draft answering the thread's open pedal-vs-plate question, awaiting review — knowledge should flow both ways. Thanks to eqtrian, Elevenpoint7five, and everyone who posted across nineteen years of tip-in.

**[Merp / TD-D / SubaruDefs](https://github.com/TD-D/SubaruDefs)** (Td_d, Merp, NSFW, DSchultz, ztan, callisto — decades of community accumulation). Gave us the EP5G211A/EP5F400C FB25 ECUFlash defs whose Injector Flow Scaling scalar became our most important *negative* finding — every search method failed to find an EP5GD00A equivalent. Also gave us `RR_EP5G601B.xml` (EDM FB20 CVT def), which proved a windfall by being *wrong*: its addresses were verified garbage on the MT ROM, forcing the day-one rule — blind porting is dead; names and scalings transfer, addresses never do. Sixteen of 28 scaling formulas are character-identical SH7055→SH7058. Thanks for the defs and the anti-reference.

**[callisto — brz-defs](https://github.com/callistoprime/brz-defs)** ("fa20guy"). The 2,812-table FA20 definition served as the cross-generation comparison reference for the entire SH7055→SH7058 scaling fingerprint library — and as the ground-truth scorer for our FA20 blind-mapping experiment, which measured the methodology against it. Its DTC switches carry "DO NOT USE" warnings — the prior art's guesswork that proved our 20-byte DTC descriptor decode was genuinely novel and deterministic (243 switches, no guessing). Thanks for the reference, callisto — and for flagging your own uncertainty honestly.

**[Scoobypedia — RomRaider and Reflashing FAQ](https://scoobypedia.co.uk/index.php/Knowledge/RomRaiderAndReflashingFAQ)**. Its "invalid conversion expressions" warning caught a **real XML bug** in our own definition files before release — a community FAQ saving a broken definition. Thanks for the catch, Scoobypedia.

**[HP Academy](https://www.hpacademy.com/blog/what-is-romraider/)** (and their [WinOLS map-identification course](https://www.hpacademy.com/courses/winols-mastery-map-identification-and-editing/)). Their 3-step DBW chain (Torque Request → Base → Throttle) is what the Requested Torque Base hunt was built on — and its *absence* on both our ROM and Mazda's validated the negative finding. Their statement that RomRaider's open-source coverage ends at the mid-2010s generation defined the frontier we're working on; their FA20 knock-limited tuning reports grounded our FB20 power analysis. Thanks for the framework.

**[Xtreme Racing Tuning](https://xtremeracingtuning.com)**. E-tunes FB20/FB25 NA by email (e.g. the 2013 Crosstrek FB20 5MT ROM `EP5IA00B`) — the existence proof that FB20 NA tuning is real and that private SH7058 defs exist. Cited in the Porting Guide as the "nothing downloadable" contrast to this project's open approach. Thanks for proving the engine responds.

**[avtochip.ru](https://avtochip.ru/)**. A *paid* Russian chiptuning file service — marking that honestly, since this project only recommends free tools, but the credit is real. Their catalog listed LEP5F300, the closest documented sibling ROM to EP5GD00A (`PF_11CYH4N` — the same internal ROM ID as ours) — the only known paid path to a second FB20 SH7058 binary, which would unlock ROM-diffing (ztan's technique) as a table-finding method. Thanks for cataloging what nobody else had.

**[LEMON Manuals](https://lemon-manuals.org.ua/)**. The complete 2012 Impreza Sport Premium MT factory service manual (HTML, ~44,800 pages) — the ground-truth reference for exact diagnostic procedures, component locations, torque specs, and wiring diagrams behind the project's car questions, like the clutch cruise-cancel switch layout. Thanks for keeping the factory documentation alive.

**[Renesas SH7058 Hardware Manual (REJ09B0177)](http://www.datasheet.hk/download.php?id=1525686&pdfid=7F72CB129B3FA574FD3F861962439607&file=0264%5Csh-2e_447691.pdf)**. The knock-code addresses at `0xFFFF87xx` turned out to be hardware peripheral registers, not RAM — the SH7058 has a dedicated knock signal processing unit. That explained why knock tables were unfindable by code-xref (the interesting logic is in silicon, not the ROM) and is our standing reference for the remaining `0xFFFF8Cxx–0xFFFF8Fxx` peripheral traffic around scalar blocks P1/ClusterB. Thanks to Renesas for documenting the silicon.

**[4roller — Subaru Impreza Tuner](https://4roller.github.io/subaru-impreza-tuner/engine.html)** ("Naturally Aspirated CVT Dreams — GP7 FB20 TR580"). "An aggressive e-tuned [FB20] can expect up to 8–12hp" plus the MAF rule of thumb (1 g/s = 1.25 BHP) — the FB20-specific numbers behind our realistic 156–160hp crank estimate for a 93-octane tune. Thanks for the honest numbers, 4roller.

**[BitEdit / BitSoftware](https://bitsoftware.com/bitedit/catalog/101)** ([modules PDF](https://chiptuningmarket.com/wp-content/uploads/2022/07/BitEdit-modules.pdf)). BitEdit's Honda Keihin module map list (~20 table types: fuel/spark by VTEC, VTC, knock, limiters) became the hunt checklist for identifying our Honda project's 66 unknown tables; their "Denso Gen3" catalog confirmed the RH850 = 4MB identification in our Gen-3 survey. A commercial product, credited for a free and honest piece of its documentation. Thanks for the checklist.

**[The Louis Rossmann community](https://www.reddit.com/r/LouisRossmann/)** — the [Consumer Rights Wiki](https://consumerrights.wiki/), the [Repair Wiki](https://repair.wiki/), and Kevin O'Reilly's [FULU Foundation](https://fulu.org/). The venue research for our "seeding the ether" sharing strategy — and its key caution: Rossmann publicly walked back support for unrestricted tractor tuning, which is why this project leads with diagnostics, repair, and understanding, plus an explicit emissions line, rather than performance tuning. Thanks for the cautionary tale along with the venues.

**[EcuTek's documentation wiki](https://ecutek.atlassian.net/wiki/)**. Their B58 Tuning Guide's hierarchy (user levels, "start here" root folders, Beginner = Stage 1 tables) is the cited model for this wiki's documentation structure. A commercial tuner, credited for a good docs idea. Thanks for the information architecture.

---

## The Tools — the open stack this was built with

**[The RomRaider project](https://romraider.com/index.php/RomRaider/EditorXMLReferenceGuide)** and the [Editor](https://romraider.com/index.php/RomRaider/EditorXMLReferenceGuide) / [Logger](https://romraider.com/index.php/RomRaider/LoggerXMLReferenceGuide) XML reference guides. The entire artifact of this project is a RomRaider XML definition — the open format (32BITBASE template: name, category, type, level, scaling, axes, zero addresses) is what made a community-built, tool-consumable definition possible at all. Our 863 tables exist *as* RomRaider XML. To the volunteers who kept RomRaider alive from the early 2000s through the mid-2010s generation: thanks for the format everything else stands on.

**[Ghidra — the NSA's open-source reverse-engineering platform](https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.3.2_build/ghidra_11.3.2_PUBLIC_20250415.zip)** (v11.3.2, Apr 2025). Its SuperH support made the entire disassembly phase possible — and our documented gotcha (Ghidra's `sh-2.slaspec` doesn't decode FPU instructions; use SH-2A / `SH2A|SHFPU|BE` instead, which handles the SH7058's SH-2E FPU code) unlocked both dispatchers, the knock subsystem trace, the SSM dispatcher, X-tau, and the checksum hunt. Thanks to the NSA for the gift nobody expected.

**[Capstone — the Ultimate Disassembler](https://www.capstone-engine.org/)**. When the Ghidra project was locked by sibling agents mid-run, Capstone with the critical `CS_MODE_SH2A | CS_MODE_SHFPU | CS_MODE_BIG_E` flag combination independently resolved 169 of 182 secondary-dispatcher call sites and decoded the type-byte-as-data-type discovery (type-0 = float32 passthrough) — the re-triage's 101-table windfall ran on Capstone. Thanks for the lightweight tool that showed up when the heavy one couldn't.

**[EcuFlash / Tactrix](https://www.tactrix.com/index.php?page=ecuflash&FontSize=font-small)**. The flashing stack the project will use — Tactrix OpenPort 2.0 protocol knowledge underpins the hardware-validation plan, and EcuFlash's history (v1.26, 2006: "added a needed checksum module for DBW Subarus") corroborated that the ECU-side checksum routine is driven by the flashing tool's kernel, not a boot gate. Thanks for keeping the flashing path open-source.

**[fenugrec — npkern](https://github.com/fenugrec/npkern)**. Its `cmd_parser.c` documents `cks_add8` — *"one's complement checksum; if adding causes a carry, add 1 to sum"* — a byte-identical match to the checksum routine at ROM `0x152`, independently confirming Denso's checksum algorithm and giving us HIGH confidence in the finding that the ECU has no boot-time checksum gate. (Repo describes itself as Nissan/Infiniti — the SH7058 relevance is in the shared checksum idiom.) Thanks, fenugrec.

**[firefighter-19 — tuneforge](https://github.com/firefighter-19/tuneforge)**. Open-source SH7058 flash-kernel work with SecurityAccess seed/key documentation, corroborating the checksum and flashing analysis alongside npkern. Thanks for the open-source kernel work.

**[james-portman — subaru-ecu-flashing](https://github.com/james-portman/subaru-ecu-flashing)**. Documents the SH7058 CAN reflash flow (services `0x34`/`0x36`/`0x31`, kernel upload, per-packet sum checksums) — independent corroboration that the ROM `0x152` checksum routine is reflash-protocol tooling, not boot enforcement. Thanks for the protocol docs.

**[LibreTuner and the Mazda MX-5 NC community](https://github.com/LibreTuner/LibreTuner/tree/master/ui/resources/definitions/mx5_nc)** — [speepsio's romdrop](https://github.com/speepsio/romdrop) (~100 EcuFlash XMLs), [cdufresne81's nc-flash](https://github.com/cdufresne81/nc-flash) (AI-native tool with MCP server), [testbenchdev's ncec-rom-tool](https://github.com/testbenchdev/ncec-rom-tool) (open-source ROM tool for the Denso SH7058 Mazda NC ECU). **Mazda HAS the Torque Request Base table — Subaru DOESN'T.** That single comparison independently validated our negative finding that the FB20's DBW chain is pedal → Requested Torque A/B → Target Throttle. The NC analysis also proved the iron law *within* Denso: same SH7058 chip, but no directory, no dispatcher, opposite axis-sharing — "the absence of a dispatcher is itself the finding." And nc-flash's MCP-server-for-AI pattern is the model for what this project could become. Thanks to speepsio, cdufresne81, testbenchdev, and the LibreTuner crew.

**[Monochrome — chromedecu.org](http://chromedecu.org)**. Fully disassembled the 1999 3000GT VR4 ROM and added functions (Chrome ROM v2.0) — proof Tephra wasn't a one-off, and that the disassemble-understand-extend methodology is reproducible by a dedicated individual. Thanks for the second data point, Monochrome.

**[bouletmarc — HondaCalibFiles](https://github.com/bouletmarc/HondaCalibFiles)**. 5,357 publicly posted Honda ECM firmware files — source of our Honda test ROM `37805-RRB-A080-M1.rwd.gz` — plus the RWD decryption cipher (`(((i ^ k2) + k1) - k0) & 0xFF`, keys C/H/X) from HondaReflashTool's `Class_RWD.cs` that unlocked the Honda SH7058 spin-off (4 → 92 tables). His 416-parameter definitions (ported from Hondata) are the prior open-source Honda SH7058 baseline. Thanks, bouletmarc.

**[jpancotti — rwd-xray](https://github.com/jpancotti/rwd-xray)**. The actively maintained Honda RWD decryption tool suite; its `table_search.py` (find data tables by repeated byte patterns) was adopted as the concept for an automated table-pattern scanner we list as an untried medium-high-value technique. Thanks for the pattern-search idea, jpancotti.

**[CSS Electronics — AI CAN Bus Reverse Engineering skill](https://github.com/CSS-Electronics/can-bus-reverse-engineering-skills)** (MIT). A Claude Code skill that automates CAN signal reverse-engineering: bus survey, bit-correlation against reference signals (OBD2/GPS/dashboard video OCR), scale/offset fitting, DBC output. Featured in their video ["AI CAN Bus Reverse Engineering"](https://www.youtube.com/watch?v=jQDqWuL5-nQ) (Jul 2026). **MIT licensed — the methodology is directly reusable.** Queued as the reference for our CAN validation track: correlating CAN frames against OBD2 PIDs as ground truth is the same epistemic pattern as our independent-rediscoveries list. Thanks for publishing the skill, not just the video.

**[Ange Yaghi (AngeTheGreat) — engine-sim](https://github.com/ange-yaghi/engine-sim)** (MIT, 9.6k stars). A real-time thermodynamic engine simulation built for audio and feel — "NOT a scientific tool" per its README. Used as the reference for our ECU simulator's sound engine: his "Improving My Engine Sound Simulator With Real Data" methodology (pulse-train synthesis → saturating waveshaper → resonant lowpass as exhaust pipe → procedural IR convolver) replaced our first oscillator-based attempt. He solves the *engine* half; we solve the *ECU* half. Saved as the Tier 2/3 engine-model reference. Thanks for the real-data methodology, Ange.

---

## Harder to link, not forgotten

These entries get credited with the same gratitude — the link just couldn't be confirmed as of Oct 18, 2026, and we won't invent URLs.

**Subaru TSB 11-156-15 (content, not link).** The bulletin documents the naming quirk that EP5GD00A is listed as the *FED CVT* post-reflash CAL ID while our *MT* ROM carries it — the source behind our standing note (wrong-file flash or Subaru naming quirk), a trap any future EP5GD00A researcher would fall into without it. The TSB number itself is unverified — the only indexed "11-156-15" is an unrelated 1999 bulletin — so no link is published here until the correct bulletin is located. Credit to Subaru of America for the bulletin content; the citation gets fixed when the number does.

**Matt Mcleod ("Throttlehappy" / ThrottleHappy Tuning)**. Published 2.5i Legacy dyno results — intake + tune +4whp ("negligible"), intake + axleback + tune 120→138whp — the empirical anchor for our power-potential analysis, killing the "tune alone makes big power on an NA Subaru" myth with data. The exact LegacyGT thread URL couldn't be located as of Oct 18, 2026; it gets filled in when found. Thanks for the dyno sheets, Matt.

**The old evolutionm.net ECUFlash sub-forum path** (`evolutionm.net/forums/ecuflash/` — HTTP 404 as of Oct 18, 2026). The site root and `/forums/` are live; that specific path is gone. The Evo community attribution stands via the live [EvoEcu wiki](https://evoecu.logic.net/wiki/Main_Page) above. Dead links don't erase debts.

---

## People

Everyone named above in one place — because tools are made by people: eqtrian · DSchultz · NSFW · XSWREX · Elevenpoint7five · Merp · Td_d · ztan · callisto · David Shirley ("Tephra") · merlin@three.com.au · fenugrec · firefighter-19 · james-portman · Monochrome · etnoy · Dilemma · Tomili · General Failure · Mackan (Markus Södergren) · Just4pLeisure · Craig Smith · Matt Mcleod ("Throttlehappy") · speepsio · cdufresne81 · testbenchdev · bouletmarc · jpancotti · dmacpro91 · shangdawei · Louis Rossmann · Kevin O'Reilly · Pinelo92 · Milburn · Timmers · Ange Yaghi.

---

## A note on what follows

**This research is unverified.** Everything on this wiki — including the findings built on the sources above — was produced by AI-assisted analysis and **zero hardware validation as of writing**. Do not flash anything based on it until the hardware validation phase is complete. Treat every address, scaling, and claim as a hypothesis with evidence, not a fact with a warranty.

**AI transparency.** This entire project was built with AI assistance. The links above are the human-made ground it stands on — every method stolen, every fact borrowed, every tool used is theirs. The mistakes are ours.

**License.** This project is released under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) — attribution, non-commercial, share-alike. No commercial use, ever. That's what "not gluttony" means.
