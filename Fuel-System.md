# Fuel System

## The honest headline

**The live fuel injector pulse-width computation path: the mapped path is a TRIM path, not the main computer (refined 10-19).** This was the most thoroughly burned-down system in the project — 11+ dead ends, each documented so nobody re-hunts them. What FUELMAIN proved: full disassembly of @0x426FE (all 6 formula variants) shows **no load term, no MAF term, no RPM term** — output is ~1.7 ms constant with realistic inputs. The ATU-II output driver @0x435E8 *is* live (emulator-confirmed peripheral writes), and it consumes @0x426FE's ticks — so the live output stage is driven by a trim-path computation. **The true load-responsive PW computer is still unidentified** — "unidentified" again describes the main path. The FUELVAR "[0xFFFF82D0]=8.0 ms seed" was fantasy: real @0x42A14 output is ~0.90 (WARMENTER: 0.8909→0.9017 over 60–200°F, moving 0.4% in the WRONG direction for enrichment). The drive-log warmup enrichment (+22% cold) is NOT in the mapped chain — ECT moves nothing with the required shape. The car demonstrably fuels correctly (STFT ±2% at cruise, WOT lambda 0.83–0.91, clean decel cut) — the path exists; static analysis just hasn't found the main computer. Fuel stays at CAUTION in the liveness matrix; **no fuel table is tunable.**

## What's dead (CERTAIN/HIGH)

- **@0x42700** "fuel PW computer" — zero references ROM-wide. (Emulator-validated its *logic* 9/9, which validated nothing about liveness — a standing lesson: EMU-validated ≠ live.)
- **@0x43F64** "fuel hub" — zero references
- **ATU-II injector driver** — config table real, all 6 loaders dead
- **HI7000 fuel parameters** — none exist; only a 25-entry descriptor
- **Injector Correction A–D, Injector Latency** — zero references (dead tables)
- **@0x34F48 "warmup"** — refuted (no ECT reference; shared-pool confusion)

## Live cells, unclear consumers

- **PW Base @0x42A14** — live, task-scheduled. Upstream inputs identified 10-19; PW taint proven to the ATU-II down-counter. Its output [0xFFFF82D0] **has a second reader** (WARMENTER 10-19): **@0x175BC** (AVCS/airflow domain, not fuel) — `[0xFFFF5E78] = guarddiv([82C0]/[82D0]) × 4.3687 × guarddiv(RPM/60.0) × 2.0e-6` → @0x158F6 AVCS limit/status bookkeeping. [82D0] is **consumed, not discarded** — the old "zero readers / computed and discarded" claim is refuted. (The earlier orphan list missed it because @0x175BC's address reaches via a literal pool, which a u32 address scan can't see.)
- **MAF temp-comp @0x0243D8** — live. Output [0xFFFF671C] has zero live readers.

## The boot disarmer (paradox resolved 10-19)

The fuel-multiplier seed [0xFFFF7868] boots as march-test residue (0x5AA5A55A) and is **never written** by the app — which predicted catastrophic fueling on boot, contradicting the running car. Resolved: **task #1 @0x270AC zeroes gate byte [0xFFFF6955]** before the armer decides, so the garbage copy never happens. The car runs because the poisoned path is disarmed at every boot. (The earlier "0xA5 residue" reading was a harness artifact from not running boot init.)

## Boot architecture: the monitor is bypassed (10-19)

The SCI reflash monitor **@0x924 has no app-exit path** — all 7 command handlers loop back or tail-jump to the transfer dispatcher (the only exit, cmd 0xA0, jumps to an uploaded kernel address — that's the reflash primitive, not the app handoff). On normal boot the monitor is **skipped entirely** (r14=0 at 0x682 → the monitor branch is not taken); the "handoff" is the boot code's vector jump via the **@0x40 trampoline** (`jmp @r14`). **[0xFFFF889A] is zeroed by @0x4D63C** (pc=0x4D6A4) during boot init, before @0x503DC's gate check — so the gate is open (0x00), Task 1 runs, and the disarmer fires. The "gate residue 0xA5" was a harness artifact from emulating @0x503DC without running the boot init first; in the real boot, @0x4D63C zeroes [889A] first. (Secondary writer @0x4E314 exists but is not the boot path.)

### The mode system (mapped 10-19, MODE889A)

byte[0xFFFF889A] is a full mode system, not a flag. **Exactly two writer instructions ROM-wide:** 0x4D6A4 (@0x4D63C, Task 1) and 0x4E332 (@0x4E314, repair-only) — and the 0x00/0xFF/0xA5 values all come from the SAME instruction; no separate "non-zero writers" exist. The mode is a **(mode,~mode) word pair** at [0xFFFF889A]: @0xBEAF0 builds it, @0xBEAD0 validates it, @0x4E314 repairs it from the **authoritative shadow byte[0xFFFF889C]** (a 0xFF→0x00 demotion was demonstrated when the shadow disagreed). **28 readers ROM-wide (889A-ARM 10-19 — the earlier "three consumers" count was wrong):** the three named dispatch sites are the important ones — the Task-1 decision @0x4D63C, the Task-0 dispatcher @0x4EB5E, and **Task-3's big sub-dispatcher @0x504FC** (not the OS scheduler — correcting an earlier label). A 3M-step emulator boot from reset takes the **0x00 path** with zero writes to 889A/889B/889C/9413/35D8 before stalling in the 0xC4E hardware wait loop (power/clock stabilization on peripheral 0xFFFFED18 bit15). **Four dispatch sites discriminate 0xFF vs 0xA5** (emulator-verified three-way dispatch) — MODE889A's "both skip" model is dead, though the 0xFF/0xA5 modes remain unreachable on a real boot. **CLOSED-negative 10-19 (MODETRIG):** nothing flips the mode. byte[0xFFFF9413] is **always 0**; the 0xFF/0xA5 modes are **unreachable**. @0x61556 (true entry — @0x6155E is mid-function) is **dead code**: it does have one static caller (@0x4D370 `jsr @r2` via pool 0x4D584 ← trampoline @0x0DD62 ← table entry @0x112AC, walked by @0xD5A0) — but the only r4=1 walker site demands r5=1 from 65521+r14, **impossible** for a 16-bit r14, and even if reached, a delay-slot instruction clobbers r6 to 1, failing the check's second condition. The mode system is pinned at 0x00 in sustained operation; there is no physical trigger to hunt.

### The cyclic ADC refresh chain (mapped 10-19, ADCREFRESH)

The writer of [0xFFFF4110]/[0xFFFF4114] is **@0x56A0** (conversion entry; the primary @0x5694 is boot-only), store at 0x5746. Chain: **0x3EA0 task table #9 → @0xDCA8 → @0x56A0 → [4110]/[4114]**, with `[4110+i] = 7.62939453125e-05 × K1[i] × ADC[i] + K2[i]` — @0xDCA8 fires **every 2 scheduler ticks (~0.954 Hz** under the corrected 524.288 ms tick; the old 4.7684 Hz was derived from the refuted 104.8576 ms tick). Upstream **(corrected 10-19, ADCPINS — the old "DMA" claim is refuted):** the [0xFFFF4064..] buffer is **CPU-filled from the ADC result registers** — `[0xFFFF4064+2n] ← ADDRn`, n=0..11 (A/D1 branch maps ADDR(12+n); A/D2 branch ADDR(24+n)). **No DMAC involvement in the ADC path at all** — the AN mapping is direct and unambiguous: **APP1 = AN9, APP2 = AN10** (SH7058 A/D0, ADDR9/ADDR10, manual-grounded). → @0xDB68 (table #6) → @0x41D6 copy loops → [0xFFFF4024] words (APP1/APP2 = words 9/10 @[4036]/[4038]); downstream: pedal task @0x249E4 → @0x24D18 ([4110]→[67BC], filters, plausibility) → @0xBB638 → [0xFFFF67A4]. ECT f32 [0xFFFF42D8] is written **only by @0xA144** via @0xDB68 — the same 0x3EA0 table #6. That's the ECT liveness proof: the ECT cell the warmup hunt reads is genuinely refreshed every cycle.

### The real fuel dispatcher is @0x503DC (mapped 10-19, FUEL50188)

**@0x50188 is a state gate, not the fuel scheduler driver** — 37 callees (36 jsr + 1 tail-call), mode-gated on [0xFFFF889A], and not one of them is a fuel PW function. The actual fuel dispatcher is **@0x503DC**: 26 functions, calling **@0x42A14** (trim writer) and **@0x42BF6** (per-cylinder scheduler); @0x42F28 is a trim-only crank scheduler (full disassembly). **Both dispatchers are 889A-gated and run at most once per boot (RUNENTRY 10-19)** — @0xDCE6/@0xDD1A/@0xDD46 are boot-phase init scripts in the 12-entry table @0x3EA0 (straight-line callers ending in a one-way `jmp @0x35A4`, never returning); 0xDD0E is a fall-through label inside @0xDCE6, not an entry. The sweep covered all three dispatchers (@0x50188: 37 fns, @0x504FC: 88 fns, @0x503DC: 26 fns) plus the 55-function task table with load/MAF taint and RPM-fixed perturbation: **load taint reaches no fuel PW cell**. No tick/PW cell tracks load in emulation. The only load-capable path into the @0x426FE trim computation is via the mults ← **[0xFFFF6644]** — whose writer was not found in 206 swept functions — **but that path is information-dead (T6650 10-19):** perturbing [0xFFFF6650] across −40…200°C moves NOTHING ([7874]≡1.0, [7868]≡2.3984, mults≡2.3984, ticks≡9) because table 0xB3BC0 is flat-1.0 in this calibration — taint without information. ([0xFFFF6650] itself is a second NTC thermistor temperature, f32 °C ×1.0, broadcast copy of [0xFFFF4120] — likely the FB20's other thermistor, IAT.) The main load-responsive fuel computer is still unidentified; this section names the dispatcher, not the computer.

## What the drive logs prove (the system works, we just can't see it)

- **Closed-loop fueling is sane:** STFT ±2% across cruise loads (30–80%). Whatever computes PW, its base calibration is good.
- **WOT enrichment is real:** lambda 0.83–0.91 at 90–100% load. Power enrichment exists and behaves.
- **Decel fuel cut is real:** lambda 1.38 (sensor lean limit), open-loop, on lift. The cut works; its trigger flag is unmapped.
- **Warmup enrichment is real:** STFT pulls −22% peak at ECT ~134°F, decaying to zero by 181°F. Something adds fuel when cold — the table is unidentified.

## What's OPEN (cable targets)

The Nov 6 Mode $22 logging list for fuel: [0xFFFF671C], injector driver registers, the warmup multiplier hunt, and the true load-responsive PW computer (FUELMAIN proved the mapped path is a trim path, not the main computer). [0xFFFF82D0]'s live readers are now known (@0x175BC, AVCS/airflow). Until then: **no fuel table is tunable.** Warmup −30%, WOT fueling, and rev-hang fueling changes are all blocked.

## For contributors

If you hunt the fuel path: the Speeduino fuel equation (`PW = REQ_FUEL·VE·MAP·E + accel + deadtime`) is the null hypothesis — Denso's deviations from it are the interesting bits. Check all four indirection layers before declaring anything dead (see [Avoiding-Hallucinations](Avoiding-Hallucinations.md) §12).
