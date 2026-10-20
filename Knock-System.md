# Knock System

## Three-mode architecture (VERIFIED)

The ECU runs three knock-correction modes, all code-confirmed with 26 taint-verified parameter tables:

| Mode | Range | Behavior | Limits |
|---|---|---|---|
| **Feedback** | RPM 1100–7000 (Range A) | Immediate retard on knock event | −5.0° max, 0.35°/event |
| **Fine-learned** | RPM 1500–3600, load 0.45–2.10 (Range B) | Learned per-cell correction | **+8.0° / −7.0°** |
| **IAM** (Rough) | RPM 1300–4400 (Range C) | Global multiplier | Init 0.50, step 0.25 |

Feedback @0x4B16A/@0x4B476 · Fine-learned **@0x4C112** (the learner, +8.0 cap) → **@0x4C022** (every-cycle arbiter, [0xFFFF8800]←clamp) · @0x4BD38 = qualifier (status byte [0xFFFF87ED] only) · @0x4BF1A = conditional reset ([0xFFFF8800]←0.0) · IAM @0x4C490 → [0xFFFF30F0]. (Corrected 10-19, KNOCKADV — the wiki previously labeled the reset @0x4BF1A as the learner.)

## The 26 parameters (VERIFIED 10-19)

Every parameter taint-verified: the ROM value was observed flowing into live code through the emulator. Includes RPM/Load windows, retard values, step sizes, ECT thresholds, IAM init/step. Min Load A = **0.48** (corrects the v0.7 0.45 error — direct ROM read + emulation).

**5 fake "Knock Correction Advance" tables: quarantined.** Values 30–80° (physically impossible), zero code references. Not knock tables. Never tune them.

## Knock → timing plumbing

Real retard application: @0x47D74 → [0xFFFF85A4]. The [0xFFFF8800]→[0xFFFF8460] path carries genuine data flow (slope 1.0, 16-cell falsification test). The knock-sum mailbox [0xFFFF8798] consumer is **FOUND 10-19** — downstream-traced and emulator-validated; [0xFFFF8808]→[0xFFFF879C] taint confirmed; true spark sum @0x4ADCE: [0xFFFF8798] = [0xFFFF879C] + [0xFFFF87C8] + [0xFFFF8800]. The knock-timing blend weight [0xFFFF95EC] has exactly one writer: **@0x66FD6** (pc=0x67076). Knock-event descriptor bit1 setter found at **@0x4B956** — via a masked GBR-relative store class the project's tools previously didn't model (worth remembering for future writer hunts). The descriptor itself [0xFFFF87E4] is **software-managed** in on-chip RAM (29 confirmed writers) — the hardware-peripheral hypothesis is refuted.

## The learned-advance mechanism (CONFIRMED, cruise attribution CONSTRAINED 10-19)

Drive logs (10-19) showed cruise timing 7–10° above the E/F tables' maximum, matching the **+8.0° fine-learning limit** almost exactly. The mechanism is **CONFIRMED real** (KNOCKADV): the knock system demonstrably learns positive advance and adds it to final spark — emulator-observed [0xFFFF8800] writes of +8.0/+5.0/+4.0/+2.0, a code-confirmed +8.0° table cap, and a cruise differential test showing **+6.0° learned → +6.000° on all four per-cylinder publishes**.

What changed is the *cruise attribution*: it's now **IAM-constrained, not bare-falsified**. The positive authority is **(1−IAM)×[0xFFFF87C4]**, enforced every knock cycle by @0x4C022 — at IAM=1.0 a seeded +6.0° learned cell is destroyed → 0.000° in one cycle. So the 37–40° vs 31° cruise gap can be [0xFFFF8800] **only if IAM<1 at cruise**; at IAM=1 the gap must be IAM×[0xFFFF87C4] (the confounding adder). The +8.0°/−7.0° limits remain VERIFIED as ROM parameters. Actual learning path: **@0x4C112** (fine table, +8.0 cap) → **@0x4C022** ([8800]←clamp, every-cycle arbiter); @0x4BD38 is a qualifier (status byte [87ED] only), @0x4BF1A a conditional reset ([8800]←0.0).

**Rule: never disable knock.** The parameters are safe to adjust (code reads them), but the correction path must stay live.

## What's OPEN

- IAM's exact scaling into final timing (multiplier confirmed, magnitude MODEL)
- The [0xFFFF87C4] positive writer — [0xFFFF87C4] itself is **refuted** as a base-timing carrier (it's a knock-advance term ≤14.06°, 0.0 at cruise); who writes its positive side is still unknown. Note the KNOCKADV pairing: [87C4]'s authority in the positive path is **(1−IAM)** — at IAM=1 the learned term is destroyed every knock cycle by @0x4C022, so [8800] can only explain cruise timing if IAM<1 there.

## Closed 10-19 (was open at build)

- ~~Whether learned cells are consumed downstream~~ — **CLOSED:** [0xFFFF8798] knock-sum mailbox consumer found, downstream-traced, emulator-validated; [0xFFFF8808]→[0xFFFF879C] taint confirmed; true spark sum @0x4ADCE: [0xFFFF8798] = [0xFFFF879C] + [0xFFFF87C8] + [0xFFFF8800]
- ~~The [0xFFFF8798] mailbox consumer~~ — **CLOSED** (see above)
