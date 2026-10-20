# Idle and DBW

## Idle control (VERIFIED)

**Feedforward, not PID.** The live idle controller is @0x585E0 (dispatcher entry 18). The "idle PID" @0x16340 is dead. No integrator, no Target-RPM input — the ECU computes idle airflow from tables and hopes.

**The 50°C switch — validated live 10-19.** Below 50°C ECT the idle target is the fast-idle map (~1600+ RPM); at 50°C (122°F) it steps down toward the 650 RPM warm target. Drive log caught the step live: 1528 → 1402 → 1246 RPM at ECT 120.2°F. The emulator predicted this; the car confirmed it.

**23 airflow tables** are IAT-indexed (not ECT — verified against the contested [0xFFFF6640]; the IAT-vs-ECT tiebreak itself remains open).

## The decel-adder reversal (10-19)

Both **Idle Airflow Decel Adders** (@0xD9720/@0xD9758) were long classified dead/DO NOT TUNE on a "zero references" finding. That finding was wrong — a buggy `mov.l` mask hid the references. Re-tracing proved both adders are **looked up by live idle code (@0x5BF46) via the dispatcher with consumed outputs**. They are live tables. Scaling is unverified (raw values) — CAUTION, not CLEAR.

## Rev hang — first direct evidence (10-19)

Ten static-analysis leads were eliminated with zero positive evidence. Then the drive log caught it:

> In-gear lift at 2537 RPM — pedal at 5–9%, but the **throttle plate held at 17–20% open**. RPM pinned at ~2540 for **5+ seconds**, still in closed-loop fueling (not fuel cut). The ECU deliberately holds the plate open on lift.

**What this means:** rev hang is a commanded behavior, not a mechanical lag. The plate position (absolute throttle) diverges from the pedal (relative throttle) by ~10 percentage points for seconds after lift.

**Mechanism FOUND 10-19 (REVHANG, emulator-verified):** rev hang is a steady **~12–15% pedal boost**, not a transient. On lift the plate holds a constant **11.37% plate-over-pedal offset** (23 rows: pedal constant 4.31%, plate constant 15.69%; the offset is identical across pedal 3.92–9.02%). No timer, no dashpot, no decay. The boost flows 1:1 through ReqTq-B → [0xFFFF8ADC] → [0xFFFF8AF0] → [0xFFFF8B9C] → @0xBCFC8 → [0xFFFF8B78] → @0xBD120 → [0xFFFF8BCC] → @0xBCEFC → [0xFFFF8B74] → @0xBCE7C → [0xFFFF8B70] → @0xBD65C → [0xFFFF8C50] — three log points reproduce EXACTLY in the emulator. **ReqTq-B @0xFA774 is NOT the delete target** (it's 1:1; the boost is upstream).

**Source of the boost (T67A4 10-19):** a **~7.8× multiplicative pedal-shaping gain inside the pedal orchestrator @0xBB638**: `[r15+4] = (([r15+0] − 0.84) × 100.0) / 12.81` (0xBB7FA–0xBB80E). The downstream 1D table @0xB2980 is 1:1 for all inputs > 1.5 (deadband only) — it contributes no gain. @0xBB980 Path 4 is a **verified passthrough** ([67A4]=4.31 → [67D4]=4.31) — REVHANG's own "arbitrator outputs the boost" attribution is REFUTED by T67A4; the boost is computed upstream in @0xBB638, not in the arbitrator.

**Current tuning target: ROM 0xBB968** — the ×100.0 gain pool float. Patching it to ×12.81 collapses [0xFFFF67A4] from 18.46% → 2.37% at the lift point (emulator-demonstrated). The PEDALCELL-era disable test result ([67D4] = sole knob for the 78 N.m idle floor) is untouched — what moved is WHERE to kill the boost: it's the @0xBB638 gain, not the ReqTq-B idle cells. [0xFFFF8BCC] itself is RAM, recomputed every pass (the "rev-hang-delete ROM cell" flag is withdrawn 10-19, T8BCC); the 78 enters via the [0xFFFF8B9C] cap → [0xFFFF8B78] → [0xFFFF8BCC] floor — see the [8BCC] section below.

**Pedal orchestrator refinements 10-19 (PEDALORCH):** all four pedal input writers found — [0xFFFF67A4]←@0xBB8A2, [0xFFFF67A8]←@0xBB8B4, [0xFFFF67E0]←@0xBBC66, [0xFFFF67F8]←@0x253E6 — all in the cyclic task tree, write-log confirmed. Only one of the three @0xBB980 calls is live (@0x24A68, r4=0); the other two (r4=1/2) sit in dead code @0x24A8A with zero ROM-wide callers — likely factory self-test or a superseded revision (r4=1/2 use fixed ROM-constant test vectors). At steady state [67A4]=[67D8]=[67F8]=[67D4] and [67E0]=1.0 — [0xFFFF67A4] carries the idle ~7%; the fmin is redundant at equilibrium.

**T67A4 10-19 — the gain inside the orchestrator:** [0xFFFF67A4] itself carries a **~7.8× multiplicative gain** from the pedal orchestrator @0xBB638: `[r15+4] = (([r15+0] − 0.84) × 100.0) / 12.81` (0xBB7FA–0xBB80E), then through the 1D table @0xB2980 (1:1 for all inputs > 1.5 — deadband only). Patching the ×100.0 pool float at **ROM 0xBB968** to ×12.81 collapses [67A4] 18.46% → 2.37% at the lift point (emulator-demonstrated). This is the rev-hang boost source — see the rev-hang section.

## The [0xFFFF8BCC] equation and the 78 chain (10-19, T8BCC)

[0xFFFF8BCC] is a **slew-rate-limited torque-demand follower** — not a source, not a ROM cell, and **not** the rev-hang-delete cell (that flag is withdrawn: [8BCC] is RAM, recomputed every pass). Sole live writer @0xBD120:0xBD330 → @0xBDD00 (mode 0 only):

`[8BCC] = fmax(fmin(prev − TL_down, TL_cap), [0xFFFF8B78])`

- prev < 110 (750 RPM): TL_down=0.05, TL_cap=10.0 (slow decay regime)
- prev ≥ 110: TL_down=100.0, TL_cap=100.0 (fast decay regime)

TL tables live at 0xB701C/0xB7030/0xB7044/0xB7058 via @0xBEA18 (TL2+TL3 = the 110 threshold). The 78 enters through the **[0xFFFF8B9C] cap in the pedal path → [0xFFFF8B78] → [0xFFFF8BCC] floor** — Hypothesis B (@0xBD120's computed side *creating* the 78) is refuted at the [8BCC] level. Readers: @0xBCFC8 (K target for the rate limiter), @0xBD120 (self), @0xBCEFC ([8B74] passthrough into the DBW chain), @0x558BE (plausibility: ([8BCC]−[8B9C]) > 5.0 clears mode byte [0xFFFF8BCA]), plus telemetry/init/dead ones. The delete target is now the **ROM 0xBB968 gain constant** (T67A4 10-19) — the ReqTq-B @0xFA774 idle cells are 1:1 and not the target.

**Cyclic ADC refresh (ADCREFRESH 10-19):** the pedal cells are fed by a cyclic ADC path. Writer of [0xFFFF4110]/[0xFFFF4114] is **@0x56A0** (conversion entry; primary @0x5694 is boot-only), store at 0x5746, chain **0x3EA0 task table #9 → @0xDCA8 → @0x56A0 → [4110]/[4114]**: `[4110+i] = 7.62939453125e-05 × K1[i] × ADC[i] + K2[i]` — @0xDCA8 fires every 2 scheduler ticks (~0.954 Hz under the corrected 524.288 ms tick). Upstream **(corrected 10-19, ADCPINS — the old "DMA" claim is refuted):** the [0xFFFF4064..] buffer is **CPU-filled from the ADC result registers** — `[0xFFFF4064+2n] ← ADDRn`, n=0..11 (A/D1 branch maps ADDR(12+n); A/D2 branch ADDR(24+n)). **No DMAC involvement in the ADC path at all** — the AN mapping is direct: **APP1 = AN9, APP2 = AN10** (SH7058 A/D0, ADDR9/ADDR10, manual-grounded). → @0xDB68 (table #6) → @0x41D6 copy loops → [0xFFFF4024] words (APP1/APP2 = words 9/10 @[4036]/[4038]); downstream: pedal task @0x249E4 → @0x24D18 ([4110]→[67BC], filters, plausibility) → @0xBB638 → [0xFFFF67A4].

**Sanity flag (TICKREF 10-19):** at the corrected 524.288 ms tick, @0xDCA8 refreshes the ADC→pedal-RAM staging at **~0.95 Hz — physically implausible for a drive-by-wire pedal input** (human pedal content needs ≥50–100 Hz sampling; even a slow background refresh would be ~10 Hz). Either the 524.288 ms tick doesn't actually fire, or the live pedal path isn't this task. **Mode $22 logging of [0xFFFF4110] on the Nov-6 cable is now MORE important, not less.**

## DBW chain (VERIFIED)

Pedal @0xBD7CA → [0xFFFF8C7C] → TT blend @0xBD65C: `plate = clamp((1−w)·TT-A + w·TT-C, 0, TT-B)`. TT-A @0xF923C, TT-C @0xF9800, TT-B @0xFAEF0 — all CLEAR, tunable. **No rate limiter in the plate chain** (fourth confirmation) — the plate *can* move fast; on lift, it's *commanded* to stay open. (Scope note: there IS a rate limiter @0xBC598 in the separate driver-demand *torque* chain, input [0xFFFF8ADC] — not [0xFFFF89DC] as previously misread. Different chain, different behavior.)

## Tip-in

Tip-in B @0xD20C8 (16×5, X=ECT, scale 1/65536). The tip-in A/B/C confusion from v0.7 is resolved: B is the live table.

## What's OPEN

- [0xFFFF6640] IAT-vs-ECT tiebreak (log shows 85°F split at full warm — quantified, not settled)
- Decel-adder scaling and units
- The [0xFFFF8B9C] cap's upstream source (the 78 enters via the pedal-path cap → [0xFFFF8B78] → [0xFFFF8BCC] floor; [8BCC] itself is RAM, sole live writer @0xBD120:0xBD330 → @0xBDD00, mode 0 only — the "rev-hang-delete ROM cell" flag on [8BCC] is withdrawn 10-19, T8BCC)
- The 78 N.m source — **refined 10-19 (T8BCC):** @0xBCFC8 can't produce it (emulator-proven); @0xBD120's computed side can't create it either (refuted at the [8BCC] level). It enters via the **[0xFFFF8B9C] cap**; the cap's own upstream is the remaining open item
