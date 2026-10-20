# Drive-Log Findings (10-19)

OBD-II logs from the author's car vs. emulator/simulator predictions. Three logs: a 23-min cold-start idle (ECT 61→194°F), a 9-min warm drive, and a 20-min mixed drive. PIDs: RPM, timing advance, load, relative+absolute throttle, STFT, ECT, wideband lambda, fuel-system status. (LTFT's PID was misconfigured — 0.00 throughout; the 0x07 variant is needed next time.)

## Confirmed

**Fuel model sane.** STFT within ±2% across cruise loads (30–80%). The MAF and injector characterization holds against the real car.

**WOT enrichment real.** Lambda 0.83–0.91 at 90–100% load. Power enrichment exists and behaves.

**Decel fuel cut real.** Lambda pegs at 1.38 (sensor lean limit), fuel system open-loop, 323 rows on lift. The cut works; its trigger flag is still unmapped.

**50°C idle switch.** Caught live: 1528 → 1402 → 1246 RPM at ECT 120.2°F. The emulator predicted it; the car confirmed it.

**Warmup enrichment characterized.** Cold-start log: STFT pulls fuel the entire warmup, peaking at **−22%** (ECT ~134°F), decaying to zero by 181°F. The programmed enrichment is richer than closed-loop wants — direct support for the −30–40% warmup-reduction tuning goal. The live warmup table itself remains unidentified.

**Cold-start spark retard.** 8° at ECT 100°F vs 21° at 120°F — catalyst-heating retard, textbook.

**Warm idle baseline.** 2.26 g/s MAF at 658 RPM. The reference number.

## The rev-hang capture

In-gear lift at 2537 RPM: pedal 5–9%, **plate held at 17–20%**, RPM pinned ~2540 for 5+ seconds, closed-loop fueling (not cut). First direct evidence of the mechanism — the ECU commands the plate open on lift. Ten static leads were eliminated before this; the log succeeded where disassembly couldn't.

## The timing gap (mechanism confirmed, cruise attribution constrained 10-19)

Cruise timing logs 37–40°; the E/F tables max out near 30° at those points. The 7–10° gap matched the **+8.0° fine knock-learning limit** almost exactly — and the mechanism is now **CONFIRMED real** (KNOCKADV): the knock system demonstrably learns positive advance (emulator-observed [0xFFFF8800] writes of +8.0/+5.0/+4.0/+2.0, code-confirmed +8.0° cap, cruise differential +6.0° learned → +6.000° on all four per-cylinder publishes). What T8800 ruled out is the *unconditional* cruise attribution — and that's now constrained, not dead: the positive authority is **(1−IAM)×[0xFFFF87C4]**, enforced every knock cycle by @0x4C022. At IAM=1.0 a seeded +6.0° learned cell is destroyed → 0.000° in one cycle. So the gap can be [0xFFFF8800] **only if IAM<1 at cruise**; at IAM=1 it must be IAM×[0xFFFF87C4]. The 20.18° "emulated cruise total" behind the old 33–37° expectation was a cold-RAM transient (T8470: cycles 2–3 of the [8470] ramp; steady state [8460]=**−46.5°**) — and SPARKOUT confirmed [0xFFFF8460..0xFFFF846C] have **zero ROM-wide readers**: the arbitration block's per-cylinder values die unpublished, and coil drive is dwell-driven only. The emulated side is closed (negative); what remains is the logged side — cruise really logs 37–40°, and its live source is back open. What the constraint tells us: the cruise timing budget does not come through the knock-learned path **unless IAM<1 at cruise** — and the simulator does not need a learned-advance stage at IAM=1.

## IAT/ECT split quantified

At full warm: ECT 194°F, IAT 109°F — an 85°F split. The [0xFFFF6640] index behind the idle tables has to be one of these; the tiebreak is still open, but the divergence is now measured.

## Logging notes for next time

- Use the 0x07 LTFT PID (the labeled one is dead)
- Include MAF and IAT on drive logs (they were dropped this round)
- A genuine cold-start drive (not just idle) would capture warmup under load
