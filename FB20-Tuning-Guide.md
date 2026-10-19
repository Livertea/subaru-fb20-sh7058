# FB20 Tuning Guide (EP5GD00A, 2012 Impreza 2.0i 5MT)

> **⚠️ UNVERIFIED RESEARCH — MAY BE 100% WRONG**
>
> Everything on this page is **exploratory analysis, not validated tuning instruction**. The definition file (v0.7, 863 tables) was built by AI-assisted reverse engineering of a ROM dump. **Zero tables have been validated on a real car. Zero ROMs have been flashed.** The tuning cable (~$80 OpenPort 2.0 clone) is expected around **November 6, 2026**; until hardware confirms these tables do what the analysis claims, treat every suggestion below as a hypothesis with receipts, never as instructions. If you act on any of it, you own the consequences — including a bricked ECU or a blown engine.
>
> **AI transparency:** this entire project — the definition, the disassembly analysis, and this page — was produced with AI assistance and has not been independently verified by a human tuner. Verify everything.

**Status tags used on this page:** 🟢 high-confidence (disassembly + ROM bytes agree) · 🟡 medium (strong shape/reference evidence, code path untraced) · 🔴 unverified (structure plausible, identity unconfirmed — **do not tune from these**) · ❓ open thread.

---

## 1. The honest headline: what a tune actually buys you

The FB20 is a 148 hp / 145 lb-ft economy engine (10.5:1 compression, intake-only AVCS, port injection, 6400 rpm redline) in a ~2900 lb car. Set expectations before touching anything:

| Scenario | Realistic gain | Source |
|---|---|---|
| Tune only, stock engine, 93 octane | **+8–12 hp crank** (~6–9 whp) | 4roller.github.io FB20 guide; COBB's honest 2 hp OTS result on the FA20 suggests the low end |
| Tune + bolt-ons (intake, header, exhaust), 93 octane | **+18–28 hp crank** (~15–22 whp, ~12–18%) | Throttlehappy's FB25 dyno: 120 → 138 whp with intake + axleback + tune |
| Intake alone + tune | **~+4 whp — negligible** | Throttlehappy, measured |

**The header is the key mod.** The stock exhaust manifold has an integrated catalytic converter close to the head — it is the single biggest physical restriction. An intake alone is worth almost nothing without it.

The peak numbers are modest, and anyone promising 20+ hp from a tune alone on this engine is selling something. **The real wins are drivability:** throttle response (the stock DBW map is numb), torque-curve shape (filling dips), and rev-hang delete. If you want power, buy a WRX. If you want to understand exactly how your ECU works and make a daily driver more enjoyable, that's what this definition is for.

Limiting factors, so nobody is surprised: 10.5:1 compression (modest timing headroom vs. the FA20's 12.5:1), intake-only AVCS (no exhaust cam phasing on the 2012), economy cam profiles (no table changes the physical lobes), port injection only (no DI charge-cooling), and a factory calibration optimized for 87 octane and fuel economy — which is also where the headroom comes from.

---

## 2. Throttle & torque: the DBW chain (biggest "feel" gain)

Subaru DBW is torque-based: pedal % → requested torque → plate position. The ECU never maps pedal to plate directly. Remapping this chain is the single most transformative drivability change, and it adds **zero actual power** — it just stops the car feeling asleep.

**The tables** (all 🟢/🟡 — chain endpoints confirmed in disassembly, variant selection logic untraced):

- `Requested Torque A` @0xFA08C / `B` @0xFA774 — 20×18, X = pedal % (0–100), Y = RPM (1000–7010), scaling x·0.0078125. This is the driver's torque request. Duplicates exist at 0xFA40C/0xFAAF4 — best hypothesis is cruise/non-cruise variants (❓ which is which is unconfirmed).
- `Requested Torque Limit A` @0xFB698 / `B` @0xFB79C — 10×5 float32. Clamps on the request (exact placement in the chain unconfirmed).
- `Target Throttle Plate Position A` @0xF923C (50×12) / `B` @0xFAEF4 (50×12) / `C` @0xF9800 (50×19) — X = requested torque, Y = RPM, data = plate %. (A's address corrected in v0.7: was banked 4 bytes off.) Which of A/B/C is active when — cruise? failsafe? — is ❓ unconfirmed.

**What the analysis would change:** the stock request maps are conservative — small pedal inputs produce disproportionately small torque requests (fuel-economy tuning). Raising the low-pedal cells of Requested Torque A/B makes the car feel dramatically more responsive. This is the "butt dyno" mod: same engine, awake car.

**Caution:** the three Target Throttle variants' selection logic is untraced. Changing the wrong variant may do nothing — or change the failsafe map. Logger validation (once the cable arrives) comes before any change here. Negative findings that matter: there is **no "Requested Torque Base"** table (a genuine Subaru-vs-textbook architectural difference) and **no throttle closing-rate/damping table** — don't go looking for them.

---

## 3. Rev-hang delete

**The mechanism** (🟡 — value shapes ROM-verified, code path untraced): when the throttle closes while the car is moving, the ECU *adds* idle airflow proportional to vehicle speed, holding revs up between shifts. The tables:

- `Idle Airflow Decel Adder (Vehicle Speed) A` @0xD9720 / `B` @0xD9758 — 9-el, X = vehicle speed 0–300 km/h, data 0.0 → 5.0/5.5 g/s (x·0.001525879). 🔴 **Both are currently Unverified** — the value shapes are textbook (0 g/s at 0–25 km/h, ramping with speed) and the category/address neighborhood fit, but the code path is not traced. Treat as the leading hypothesis, not a confirmed kill switch.

**What the analysis would change:** zero the tables — **start by halving, not zeroing**, drive, log, then zero. Halving first is the cautious path because these adders interact with idle stability and decel fueling; an abrupt zero could produce stalling on clutch-in or harsh decel. Supporting cast (untraced, do not touch yet): `Decel Initial Idle Min Airflow A/B`, ramping adders, and the CL↔OL transition tables below.

**What we don't know:** DFCO (deceleration fuel cut-off) tables were searched for and **not found** — resume threshold, cut counters, enrichment singles are all unmapped. Don't guess at DFCO behavior from these tables.

---

## 4. Timing

**Base tables** (🟢 structure + values ROM-verified; scaling x·0.1 confirmed — old docs saying x·0.05 were wrong):

- `Base Timing A` @0xCC7A0 (13×13) / **`B` @0xCCA3A (13×17 — corrected in v0.7; the old 13×13 was a window into the real table)** / `C` @0xCD670 / `D` @0xCD778 (7×13) / `E` @0xCCED4 / `F` @0xCD010 / `Base Timing Idle` @0xD4640 (2D float).
- Stock values are sane: ~7.9° idle, ~17.3° cruise, ~16.3° WOT. The factory calibration is conservative for 87 octane; on 93 octane the analysis suggests 2–4° of mid-range headroom.
- ❓ `Base Timing C/D`'s X-axis [500–750] is an **unidentified quantity** (not load, MAP, or RPM). Do not tune C/D until that axis is identified. Cruise vs non-cruise assignment of the A–F variants is also unconfirmed.

**Knock control — the safety net** (🟢 disassembled: 31 knock functions @0x04B000–0x04C700):

- `Knock Correction Advance Max A–E` @0xCC450/0xCC2F6/0xCC638/0xCC6D8/0xCC738 (x·0.01) — the ceiling the ECU may advance toward: ~8° at light-load/high-RPM, collapsing to ~0.5° at WOT.
- Parameters: retard limit **−7.0°** @0xD4A6C, IAM initial **0.50** @0xD4AB8, learned table in RAM @0xFFFF8800/8818.
- The FB20 at 10.5:1 is **not heavily knock-limited** on 87 (unlike the 12.5:1 FA20), which is where the 93-octane headroom comes from.

**What the analysis would change:** advance mid-range timing on 93 octane until the knock system starts pulling feedback correction, then back off 2°. **Never advance timing without a knock-monitoring solution** — the OpenPort cable + RomRaider logger will do this once it arrives. Until then, timing changes are off the table. Exactly one `Ignition Dwell` table exists @0xDD4C4 (16×5, RPM × battery volts) — there is no dwell-vs-temp or coil-charge table; don't invent one.

---

## 5. Fueling

**Open-loop targets** (🟢): `Primary Open Loop Fueling Non-Cruise` @0xD46E8 (10×16) / `Cruise` @0xD487C (10×11), AFR = 14.7/(1 + x·0.0078125). Stock runs stoich (14.7) in light load, enriching with load/RPM. NA engines typically make best power around 12.5–13.0 AFR at WOT — if stock is richer than that under load, there's a few hp in leaning it out. Which map is active when (cruise selection logic) is ❓.

**Closed-loop targets** (🟡): `CL Fueling Target Compensation A–D (Load)` @0xD34A6/0xD36F6/**0xD3948**/0xD3B96 (11×21 each; C's address corrected +2 bytes in v0.7) and `CL Fueling Target Compensation (ECT)` @0xC3FB0. Which of A–D applies when (bank × cruise is the leading guess) is unconfirmed.

**CL↔OL transition** (🟡): `CL to OL Transition with Delay (Throttle)` @0xD5180 / `(Accelerator)` @0xD51D0 / `B` @0xD5220 — 16-el, RPM 500–5200, throttle thresholds ~16–33%, accelerator ~39–50%. Getting into open loop sooner (less delay) is a small but real transient-response gain. The "with delay" counter logic and hysteresis are ❓.

**Warmup/cranking** (🟢 values): `Warmup Enrichment Multiplier` @0xCF399 — peaks 1.76× at −20–0°C, tapering to 1.0 by 30°C. Cranking IPW tables @0xCF586/0xCF5E6/0xCF646 (22.9 ms cold → 10.8 ms). Leave warmup alone unless you enjoy hard starts.

**Warmup enrichment — the first-tune candidate** (🟢 hardware-measured, Oct 2026): a 23-minute warmup datalog (60.8°F → 194°F ECT, 6,796 rows) showed **STFT −8% to −23% through warmup, normalizing to +0.7% at full temp**. Base fueling is essentially perfect — the *warmup enrichment alone* is too aggressive for this engine (replacement engine, ~140k mi). The `Post-Start Enrichment` table @0xC07C4 (11×29, X = ECT −30..20°C, Y = time 0..1400 s, x/1000 scaling) is now **emu-validated** — the ECU follows it exactly, and the result is too rich. The "rich at idle" concern is warmup-specific, not a general fueling problem.

**What the analysis would change:** trim the cold region of Post-Start Enrichment **~30–40%** (ECT −30..0°C × t = 50..600 s), targeting STFT −5..−10% during warmup instead of −8..−23%. This is the planned first real tune: a single well-understood table, a clear before/after log comparison, and a change that's trivially reversible. Flash order (once the cable arrives): unmodified write-back first, then this, then rev hang, then DBW, timing last with knock logging.

**VE** (🟢): 42 tables @0xC4E70 + i·0x84 (6×11) + 42 high-RPM @0xC6418 + i·0xD8 (6×18), x·0.00305175781 → %. What selects among the 42 variants is ❓ — do not rescale VE for bolt-ons until the selector is understood, or you'll fix one condition and break eleven others.

**The injector problem** (🔬 searched-and-absent): **there is no injector flow scalar on this ECU.** It was searched for in every encoding and does not exist; flow is likely hardcoded in code. The FB25 sibling defs *do* have one ("Injector Flow Scaling" @0xcc7f0 on EP5G211A) — ours doesn't. **Consequence: injector rescaling has no single-scalar path.** Injector latency data exists near @0xD26A0, but a larger-injector swap would need per-table fueling rework, not a constant. Do not swap injectors expecting a one-cell fix.

**What the analysis would change, in order:** (1) CL↔OL transition delay — earliest, safest transient gain; (2) OL targets toward 12.5–13.0 at WOT on 93, validated with a wideband; (3) nothing else in fueling until the final summation order is traced — **how base AFR, warmup, tip-in, X-tau, and closed-loop trims combine (additive? multiplicative? in what order?) is the biggest fuel-chain open thread**, and changing maps without knowing the combination math is guessing.

---

## 6. Tip-in enrichment

**The tables** (🟡 — dims resolved by re-triage, code path untraced): `Throttle Tip-in Enrichment A` @0xD13D4 / `B` @0xD1474 — **16×8** (corrected in v0.7; the old 8×8/8×6 with swapped axes was wrong), X = ECT @0xCEB64, Y = time-since-tip-in, scale **1/256**, unit **% enrichment** (not ms — also corrected). Compensations: `Tip-in Enrichment Compensation (RPM)` @0xCFAC8, `(ECT) A–J` @0xD2014….

**Rules of thumb** (distilled from 15 years of community trial and error — starting points, not gospel):

- **Don't set the minimum dTPS threshold near 0%.** Throttle sensor noise is ±0.1% (worse on DBW); a ~0% threshold lets noise constantly trip the routine and wrecks closed-loop fueling. ~1–2% is the sane range.
- **The minimum pulse-width threshold must scale with injector size.** Fit injectors twice the size without scaling it and small throttle movements fall below the threshold and get nothing — the classic post-swap tip-in stumble.
- **Injector swap only:** scale the tip-in table by the inverse injector-size ratio, then scale the min-PW threshold the same way.
- **Airflow mods:** more airflow generally wants *more* tip-in, not less — several tuners cured stumbles by increasing tip-in back toward stock after the "correct" rescale.
- **Light-throttle lean spots (1–3% delta):** add to small-delta cells or lower the threshold toward ~1%. Never 0%.
- The A/B pair is believed to be AVLS-off vs. AVCS-on variants (unconfirmed in code).

---

## 7. AVCS

**Intake cam** (🟡): `AVCS Intake Cruise` @0xC1650 / `Non-Cruise` @0xC18F0 (14×18), `AVCS Warmup Correction` @0xC1F18 (16×25). Advancing/retarding the intake cam shifts the torque curve — more overlap up top, less down low. The factory calibration is a fuel-economy compromise; there's character to be found here once the cruise/non-cruise assignment is confirmed.

**Exhaust cam — do not touch.** The definition contains `AVCS Exhaust Cruise` @0xC1B74 / `Non-Cruise` @0xC1D28, but the 2012 FB20 is documented as **intake-only AVCS**. Those exhaust tables are either vestigial or misidentified. Until hardware or disassembly says otherwise, they are 🔴.

---

## 8. Limiters

- `Rev Limit (Fuel Cut)` @0xCEA78 = **6700.0 RPM** (exact float, 🟢). Raising it is possible; the power peak is at 6200, so this mostly avoids bouncing off the limiter, not making power. Valve-train limits above stock are unknown — small steps, with logs.
- Speed limiting: throttle tables @0xF8E44/0xF8E50/0xF8E5C/0xF8E68 show **progressive throttle closure** (increasing values, not hysteresis pairs — 🟡 shape inference). `Speed Limiting (Fuel Cut) B` @0xCE6B0 also exists. Raising the speed limiter is a legal-liability question as much as a tuning one; noted here for completeness, not endorsed.

---

## 9. What NOT to touch, and why

**The emissions line (non-negotiable).** Catalytic converters, O2 sensors, EVAP, and EGR stay — no deletes, no disable instructions, ever. This page contains no delete how-tos and never will. The agreed path for a P0420 on cheap aftermarket cats is: **(1)** a $10 O2 spacer first; **(2)** calibrating the rear-O2 monitor to installed, functioning hardware — **never** disabling the monitor. The DTC switch map (243 switches, P0420 = switch 57) is documented for *diagnosis*, not for turning monitors off.

**Unverified (🔴) tables.** Roughly 30% of the definition is low-confidence. "Unknown … (Unverified)" means the structure passed code-reference checks but the identity is a guess. **Do not tune from these.** That includes the Idle Airflow Decel Adders in §3 until their code path is traced — the rev-hang recipe above is a hypothesis, not a procedure.

**Things that don't exist — stop looking:**
- Injector flow scalar (searched-and-absent — see §5)
- Requested Torque Base (architectural negative finding)
- Throttle closing-rate/damping table (rev hang is airflow-based, §3)
- DFCO tables (searched-and-absent; decel behavior is unmapped)
- Per-cylinder timing comps, per-gear timing 4×3, fan tables in the ECM (all searched-and-absent)

**MAF Sensor Scaling** @0xDCEA0 — do not touch unless the intake changed, and then rescale it properly (g/s vs volts, wideband-verified). A wrong MAF curve corrupts *every* load-indexed table downstream. (A 65-el impostor @0xD5EEC was ruled out — it's a limit/diagnostic curve, not the sensor.)

**Anything whose combination math is untraced.** Final fuel summation order, knock/timing summation code, DBW variant selection, VE 42-variant selection, cruise/non-cruise assignment across all chains — changing inputs to an equation you haven't read is not tuning, it's lottery.

**The 0xD0000 shared component** (@0xD0044/@0xD007C and friends) — a structured blob with 17 directory references whose purpose is unidentified. Not a tuning target.

---

## 10. The validation roadmap (what happens before any of this is real)

1. **Cable arrives (~Nov 6)** → pull the car's actual ROM with EcuFlash; verify it matches the dump this definition was built from.
2. **Validate the definition offline** — every table's data against the fresh ROM before anything is written.
3. **Logger validation** — log knock feedback, IAM, AFR (wideband required for fueling), throttle vs. pedal, tip-in behavior. Confirm the tables do what the analysis claims.
4. **First flash = read + write-back of an unmodified image.** Prove the write path before changing a single cell.
5. **One change at a time**, smallest effective delta, with logs before and after. (Rev-hang halving before zeroing. Timing in 1° steps. Fueling against the wideband.)

Until step 3, this page is a map of a territory nobody has walked.

---

*Page created 2026-10-18. Sources: v0.7 definition (863 tables), Computation-Flow.md, FB20_POWER_POTENTIAL.md, TIPIN_WIKI_DRAFT_ROMRAIDER.md, re-triage report (TRIAGE_RERUN_2026-10-18.md). All table addresses cited from the v0.7 XML; v0.7 corrections applied (Base Timing B 13×17 @0xCCA3A, CL Fuel Comp C @0xD3948, Target Throttle A @0xF923C, tip-in 16×8 @1/256 % enrichment). License: CC BY-NC-SA 4.0 — no commercial use, ever.*
