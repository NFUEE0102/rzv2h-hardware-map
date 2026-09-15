# 06 · Timing System (Timers/PWM)

Inside the RZ/V2H SoC (system-on-chip, cramming the processor, memory controller, and every kind of peripheral into a single chip), there's more than one kind of "time-counting" hardware. The official hardware manual groups them all together under SECTION 5 TIMER, and this group maps to that entire section: the system time base, the watchdog, the general timer, the compare match timer, the general-purpose timer (also the only one that can output PWM waveforms), PWM output gating, and the realtime clock.

At first you might think "why are there so many timers," but the division of labor is actually pretty clear. Here's the skeleton of the whole group in one picture:

```text
RZ/V2H Timing System: Three Roles

┌── System time base (for the kernel's own use; the application layer can only use it indirectly, via standard APIs)────────┐
│  SYC      64-bit system counter → feeds the A55's generic timer                                                           │
│           (the hardware source of Linux's arch_timer), shares one count with GE3D                                         │
│  GTM/OSTM 8 channels, 32-bit; claimed by the kernel as a clockevent (scheduler tick)                                      │
│  CMTW     8 channels, 16/32-bit; used by the kernel as an auxiliary clock source                                          │
│           ↑ none of these three expose a "pick-your-own-channel" interface to applications                                │
├── Application-controllable timing/output (has external pins or a standard Linux device)───────────────────────────────────┤
│  GPT      16 channels, 32-bit; the only one with external I/O pins, able to produce PWM waveforms                         │
│  POEG     a "fail-safe gate" for GPT's output pins; doesn't count on its own                                              │
│           (PWM isn't a standalone peripheral — it's produced by GPT + POEG together)                                      │
│  WDT      4-channel watchdog, each bound to one core; WDT1 exposed as /dev/watchdog0                                      │
│  RTC      calendar clock, keeps running through power loss; exposed as /dev/rtc0                                          │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

This picture also flags the two things you most need to remember about this group: **(1)** the first three (SYC/GTM/CMTW) are system-level time-base supplies — your program can only use them indirectly through the standard Linux time APIs; there's no interface for "pick a specific channel on a specific one"; **(2)** the ones the application actually wants to touch directly are the last four — but the most sought-after of them, PWM output, happens to have no ready-made driver in this board's Linux userspace (the rest of this document goes unit by unit through exactly what's missing and what the alternative paths are).

## Units in This Group

| Unit | One-Line Summary | Board Status |
|---|---|---|
| [SYC (System Counter)](#1-syc-system-counter) | 64-bit system count, the time base feeding the A55's generic timer, shared with GE3D | Enabled (feeds the `arch_timer` system time base); not separately listed in the development notes (a gap — can only be inferred) |
| [GTM/OSTM ×8ch](#2-gtmostm-8ch-general-timer-one-8-channel-ip-two-names) | 8-channel, 32-bit general timer, claimed by the kernel as a system clockevent | Enabled and actively used by the kernel (`renesas_ostm` clockevent); GTM and OSTM are the same IP — don't double-count |
| [CMTW ×8ch](#3-cmtw-8ch-compare-match-timer-w) | 8-channel, 16/32-bit compare match timer with input capture/output compare capability | Enabled (`rz_cmtw` clockevent, not exposed as an application-layer device) |
| [GPT ×16ch](#4-gpt-16ch-general-purpose-timer-the-only-timer-that-can-output-pwm) | 16-channel, 32-bit; the only timer with external pins that can generate PWM waveforms | Partially Enabled (`gpt@13010000` = GPT0 ch0–7 is `okay`, the other 15 nodes are `disabled`); **no PWM chip registered** |
| [POEG/PWM Output](#5-poegpwm-output-gpt-output-pins-fail-safe-gate) | The fail-safe gate for GPT's output pins; PWM isn't a standalone peripheral — it's produced by GPT + POEG | `/sys/class/pwm` is empty (no pwmchip, no POEG driver); recommend handing this to a realtime core |
| [WDT ×4ch](#6-wdt-4ch-watchdog-timer) | 4-channel watchdog, each bound to one core; on timeout it can reset the chip or raise an NMI | Enabled (WDT1 = CA55, `/dev/watchdog0`, `rzv2h_wdt`); WDT0/2/3 belong to the CM33/CR8 firmware |
| [RTC (RTCA-3)](#7-rtc-rtca-3-realtime-clock) | Calendar/binary-count realtime clock, running off its own crystal and standby power, keeps time through power loss | Enabled (`/dev/rtc0`, `rtca3`); `hwclock -r` works |

> ⚠️ **Note (the `<board-IP>` placeholder)**: all commands in this group are run at the board's own local terminal and don't involve the board's network address; if you're instead operating remotely over SSH, the board's address is assigned dynamically by DHCP and can change on every boot — always run `ip a` first to check the current address. This handbook uniformly uses `<board-IP>` as a placeholder rather than hard-coding one.

---

## 1. SYC (System Counter)

### What This Is (the Mechanism)

SYC is a "time-base supply" unit: it doesn't itself expose any operating interface to the application layer. Its role is to generate a **shared, stable count value** for two downstream consumers. One is the Cortex-A55's built-in generic timer — the standard Arm architectural timer, and also the hardware source of `arch_timer` in Linux ("generic timer" is Arm's umbrella term for the set of timer registers built into every application core). The other is GE3D (the 3D graphics engine on this SoC). (r01uh1032 §5.2 overview)

It generates its count by borrowing the timestamp generator inside Arm CoreSight SoC-400 (CoreSight is Arm's debug/trace infrastructure) to produce a raw count. SYC then converts that count into **Gray code** for output — a binary encoding where adjacent values differ by only one bit, used to avoid sampling errors from multiple bits flipping at once when sampling across clock domains. The Manual's block diagram (Figure 5.2-2) draws this path as Time Stamp Generator → BIN2GLAY → Count output. (r01uh1032 §5.2.1.2)

The clock source for counting is a 24 MHz `SYC_0_CNT_CLK`. (r01uh1032 §5.2.1.1 Features)

It also has a debug-related feature called "Halt on Debug": when CoreSight sends a HALTREQ over the CTI (Cross Trigger Interface, which lets debug events notify each other across multiple units), the timestamp generator stops counting; when it sends RESTARTREQ, counting resumes. The Manual specifically notes that Cortex-A55 and GE3D share **the same count**, so the stop/restart of the two happens simultaneously (verbatim: "The stop/restart control of the counter by the Halt on Debug function is also performed simultaneously for Cortex-A55 and GE3D"). (r01uh1032 §5.2.3.2 NOTE)

Pulling all this together into one sentence: SYC is the system-level time-base source that gives the A55's generic timer and GE3D a shared, stable count to use — it's not a peripheral that's "there for applications to pick and operate."

### How You See This Under Linux

Honestly, you almost **never see SYC itself** from userspace. This needs to be said plainly, because it's the one unit in this group that the development notes never separately inventoried:

- The development notes (the hardware unit usage guide, 49 numbered units across the whole document) **don't list SYC separately** — it's one of the gaps this resource map honestly flags, not an existing conclusion.
- The only indirect clue available is the interrupt list: `arch_timer` is active in `/proc/interrupts` (doc06 §3; ✅ Verified on the board with `cat /proc/interrupts`: `arch_timer` is ticking; transcript: live/ch04-followup.txt). `arch_timer` being active means the A55's generic timer (whose count source is SYC) is indeed running — but this is an **inference**, not a direct inventory of SYC itself.
- There's no `/dev` node, and no sysfs control interface (sysfs is the virtual filesystem the kernel uses to expose device state as files under `/sys`). The only verification you can do is this one indirect line:

```bash
cat /proc/interrupts | grep arch_timer
```

What this confirms is whether the generic-timer interrupt downstream of SYC is active — **not** a direct read of SYC's 64-bit count value. There is no way for userspace to directly read that count, nor can it access SYC's `PSELCTRL`/`PSELREAD` registers.

> ⚠️ **Note (don't cite an "inference" as if it were a "measurement")**:
> - **Scenario**: you want to write, in a document or report, "this board's SYC has been verified enabled, at 24 MHz."
> - **Symptom**: that sentence has no direct on-board evidence to back it up.
> - **Cause**: SYC has no `/dev`/sysfs node; this board can only **infer** that it's running from `arch_timer`'s interrupt being active. The 24 MHz and 64-bit figures are **official Manual specs**, not values measured on the board.
> - **Prevention/Fix**: when citing this, keep the two things separate — "board status" can only be written as "enabled, inferred from `arch_timer`"; spec numbers should always be labeled with the official Manual as their source, never conflated with "measured."

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| Count width | 64-bit Gray code count value (verbatim: "64-bit gray code counter value generation") | r01uh1032 §5.2.1.1 Features |
| Count clock | 24 MHz (`SYC_0_CNT_CLK`) | r01uh1032 §5.2.1.1 |
| Address space | 8 KB; base `<SYC0_base>` = `0x1401_0000` (CM33 non-secure `0x5401_0000`, secure `0x4401_0C00`) | r01uh1032 §5.2.2 Table 5.2-1 |
| Access regions | First 4 KB (the `PSELCTRL` region) and second 4 KB (the `PSELREAD` region), both supporting secure and non-secure access | r01uh1032 §5.2.2 Table 5.2-2, §5.2.3.1 |

The most important line on the limits side: it has no userspace interface. The address and region info in the capability table above is only useful for bare-metal work (bare-metal meaning reading/writing hardware registers directly, without going through an OS) or debug probes — ordinary Linux application development never touches this, and doesn't need to.

### When You'd Actually Use This

**Mechanism**: as soon as your program calls a standard POSIX time API — `clock_gettime(CLOCK_MONOTONIC)` (reads a monotonically increasing time since boot that never gets stepped backward by clock adjustments), `nanosleep` (sleeps for a given duration), or the kernel scheduler's tick — it ultimately depends indirectly on the A55's generic timer, whose count source is precisely SYC. In other words, you're **always using it**, just through several layers of indirection.

**Decision rule**: ordinary application development never needs to, and can't, program SYC directly. Whether it's sensor polling, control-loop scheduling, or any situation where you need to measure "how much time has elapsed," going through the standard Linux time API means you're already sharing this SYC time base. The only situation where you'll actually "touch" SYC's own characteristics is CoreSight debugging/tracing, when you need A55 and GE3D to freeze their counts synchronously at a breakpoint — that's when Halt on Debug comes into play, and ordinary application development never gets down to that layer.

**Take an industrial inspection camera as an example**: whether you're timestamping every frame or a realtime control loop needs to measure its period, going through the standard Linux time API means you're sharing SYC at the mechanism level. This isn't a choice made for any particular application — swap in a different application domain and the conclusion is identical.

> Source: the official hardware manual r01uh1032 **§5.2 System Counter (SYC)** (p1162–1163 overview; registers from §5.2.2, p1164). Board status is inferred from doc06 (`06-hardware-resource-map.md`) §3's `arch_timer` interrupt activity; SYC itself was never separately listed in the development notes (a gap).

---

## 2. GTM/OSTM ×8ch (General Timer, One 8-Channel IP, Two Names)

### What This Is (the Mechanism)

Let's clear up the easiest thing to miscount first: **GTM and OSTM are the same hardware, not two separate ones.** The official Manual's §5.5 is formally titled "General Timer (GTM)," but every register in that section is named starting with `OSTMn` (`OSTMnCMP`/`OSTMnCNT`/`OSTMnTE`/`OSTMnTS`/`OSTMnTT`/`OSTMnCTL`), because this IP is register-compatible with Renesas's existing OSTM (One-Shot/Interval Timer). The development notes describe it in two separate places (one covering GTM, one covering OSTM), but underneath it's **the same 8-channel hardware**. (r01uh1032 §5.5)

It has 8 channels total, OSTM0–OSTM7, each fully independent (verbatim: "Number of channels 8"). Each channel has two operating modes (r01uh1032 §5.5.1, §5.5.2.2.2 Table 5.5-4):

- **Interval timer mode**: a 32-bit down-counter that counts down from the value set in `OSTMnCMP` to 0 (initial value `FFFF_FFFFh`), and can repeatedly trigger interrupts or start the ELC.
- **Free-running comparison mode**: a 32-bit up-counter (initial value `0000_0000h`) that continuously compares its count against `OSTMnCMP`, triggering an interrupt/ELC on a match.

Each channel can generate an interrupt (`GTMn_GTMTINT`), or directly start the **ELC** (Event Link Controller — a mechanism that lets a hardware event trigger another hardware action without going through the CPU, saving the latency of interrupt handling); but it **cannot** directly trigger the DMAC (DMA Controller). The Manual lists all 8 channels explicitly as "Startup of Direct Memory Access Controller: Not possible" and "Function Startup of Event Link Controller: Possible." (r01uh1032 §5.5.1.1 Table 5.5-2)

One last essential characteristic: it has **no external I/O pins at all** — it's purely an internal count/interrupt/ELC-event unit. This is the most fundamental difference between it and GPT further down (which has external pins and can output waveforms) — GTM/OSTM's count results can only be used inside the chip; they never reach an external pin.

### How You See This Under Linux

- **Driver**: `renesas_ostm` (kernel source `drivers/clocksource/renesas-ostm.c`). It's bound as the kernel's **clockevent/clocksource** provider — clocksource is the timing source the kernel uses to read "the current time," and clockevent is the timer the kernel uses to schedule "trigger an event at a certain point in time" (e.g., the scheduler's next tick). It is **not** a `/dev` character device, and has **no** ioctl interface (ioctl being the system call userspace uses to issue control commands to a device node). (doc07 §19, verbatim: "bound as a clockevent/clocksource — NOT a /dev character device. It has no userspace ioctl interface")
- **Board status**: at least one channel is claimed by the kernel as the system's clockevent device (driving the kernel's scheduler tick); the remaining channels are managed by the kernel/firmware and not exposed to the application layer. (unit-map 06: "Enabled and actively used by the kernel (`renesas_ostm` clockevent)")
- **How to observe it**:

```bash
cat /proc/timer_list | grep -i ostm
```

This confirms whether it's currently the active clockevent. (doc07 §19/§21)

- **How the application layer uses it**: only **indirectly**, through the standard POSIX timer interfaces (`timerfd_create`, `nanosleep`, `clock_nanosleep`). The kernel schedules these requests onto whatever timer it's managing, but there's **no guarantee** it lands on GTM/OSTM specifically — that depends on which clockevent the kernel scheduler currently has selected.

> ⚠️ **Note (don't count GTM and OSTM as two separate things)**:
> - **Scenario**: in the board's hardware inventory you see both "GTM ×8ch" and "OSTM (`renesas_ostm`)" listed, and your gut says that's 16 usable channels combined.
> - **Symptom**: however you count the actually-usable timer channels, the number never matches what you expected.
> - **Cause**: GTM and OSTM are **the same 8-channel hardware**, just listed once under each of two names; their registers are mutually compatible (`OSTMnCMP`/`OSTMnCNT`…).
> - **Prevention/Fix**: treat it as **one group of 8 channels**, don't double-count it — and note that these 8 channels are already claimed by the kernel as a system time base, not opened up for the application layer to freely pick a channel from (source `07-hardware-unit-usage-guide.md:326,362-365`).

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| Channel count | 8 (GTM0–GTM7/OSTM0–OSTM7; verbatim "Number of channels 8") | r01uh1032 §5.5.1.1 Table 5.5-1 |
| Counter width | 32-bit (`OSTMnCMP`/`OSTMnCNT` both 32-bit) | r01uh1032 §5.5.2.2.1/§5.5.2.2.2 |
| Initial value/direction, both modes | interval timer mode = down, initial value `FFFF_FFFFh`; free-running comparison mode = up, initial value `0000_0000h` | r01uh1032 §5.5.2.2.2 Table 5.5-4 |
| Interrupt routing capability | Can start the ELC; **cannot** start the DMAC | r01uh1032 §5.5.1.1 Table 5.5-2 |
| External pins | None (purely an internal count/interrupt/ELC unit) | r01uh1032 §5.5.1.1 |
| 8-channel register bases | GTM0 = `0x1180_0000`, GTM1 = `0x1180_1000`, GTM2 = `0x1400_0000`, GTM3 = `0x1400_1000`, GTM4 = `0x12C0_0000`, GTM5 = `0x12C0_1000`, GTM6 = `0x12C0_2000`, GTM7 = `0x12C0_3000` | r01uh1032 §5.5.2 Table 5.5-3 |

The key line on the limits side: it has no external pins, and no interface for the application layer to directly pick a channel — that's what pins down its role as "only usable by the kernel as a system time base."

### When You'd Actually Use This

**Mechanism**: GTM/OSTM is a general timer for kernel scheduling use, with no external pins — it's not a peripheral meant for applications to directly pick a channel and operate.

**Decision rule**: when you need "precise, application-controllable timed interrupts or events," first work out which kind you actually need —

- If what you want is "timing within OS scheduling precision" (millisecond-scale, tolerant of kernel scheduling delay), `timerfd`/`nanosleep` is enough — the kernel will schedule you onto a clockevent like GTM/OSTM under the hood; you don't need to, and can't, pick a specific GTM channel yourself.
- If what you want is "hardware-grade timing that needs to output a waveform externally or synchronize with other signals," that's the next section, GPT (which has I/O pins and PWM capability) — because GTM/OSTM has no external pins at all and simply can't do this.

**Take a sensor-polling schedule as an example**, or any throttling timer in a pipeline: as long as the precision needed is millisecond-scale and kernel scheduling delay is tolerable, just use Linux's `timerfd`/`nanosleep` directly — these requests ultimately get scheduled by the kernel onto a hardware clockevent like GTM/OSTM. Swap in a different application, and the decision rule is the same: first ask "does my timing need to reach an external pin," and if the answer is "no," use the standard API.

> Source: the official hardware manual r01uh1032 **§5.5 General Timer (GTM)** (p1236–1237 overview; registers from §5.5.2, p1238). Development notes doc07 (`07-hardware-unit-usage-guide.md`) §19 (GTM) + §21 (OSTM, merged as the same hardware), doc06 §2.

---

## 3. CMTW ×8ch (Compare Match Timer W)

### What This Is (the Mechanism)

CMTW (Compare Match Timer W) has 8 channels total, actually structured as **4 channels × 2 units** (verbatim: "eight channels (4 channels × 2 units)"). Each channel is an up-counter, selectable as 16-bit or 32-bit, that triggers an interrupt when the count matches the compare value, after which the counter resets to `0000_0000h`. (r01uh1032 §5.6.1 Table 5.6-1)

Beyond plain compare match (interrupt on a match), CMTW also has two capabilities that GTM/OSTM lack (r01uh1032 §5.6.1):

- **Input capture**: when an external signal fires, the current count value gets latched, with up to 2 input pins per channel (`TICn0`/`TICn1`). This means it can, in theory, measure "the time interval between external pulses."
- **Output compare**: when the count matches the compare value, it outputs a signal on an external pin, with up to 2 outputs per channel (`TOCn0`/`TOCn1`).

But note: **only CMTW0–3, 4 channels, have external pins** — CMTW4–7 have no I/O pins at all, and can only be used for internal interrupts/events (verbatim note: "CMTW4 to CMTW7 do not have any I/O pins"). (r01uh1032 §5.6.1 Table 5.6-3)

The counting prescaler is selectable from 4 options (a prescaler divides the input clock by some factor before feeding it to the counter, used to adjust counting speed and the measurable time range): `PCLKL/8`, `/32`, `/128`, `/512`. (r01uh1032 §5.6.1 Table 5.6-1) It can also generate 3 kinds of event-link (ELC) output without going through the CPU: compare match event, output compare 0 event, output compare 1 event. (r01uh1032 §5.6.1 Table 5.6-1)

### How You See This Under Linux

- **Driver**: `rz_cmtw` (the Renesas RZ CMTW clock-event/clocksource driver), bound as a kernel timer, **not** a `/dev` node, no ioctl interface. (doc07 §20, verbatim: "bound as a kernel timer, NOT a /dev node")
- **Board status**: the kernel has this driver loaded as an auxiliary clock source, but it is **not exposed as an application-layer device**. (unit-map 06: "Enabled (`rz_cmtw` clockevent, not exposed as an application-layer device)")
- **How to observe it**:

```bash
cat /proc/timer_list
ls /sys/devices/system/clocksource/
```

(doc07 §20)

- **A capability dead end (this matters)**: the application layer likewise can only use it indirectly through the kernel's POSIX timer interface, with no way to specify a particular CMTW channel; and **no driver exists that lets the application layer access CMTW0–3's input capture/output compare pins** — the development notes list no corresponding sysfs or character-device path for this. In other words, that "measure external pulse intervals" capability of CMTW exists in silicon (die), but is currently **unreachable** from this board's Linux application layer.

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| Channel count | 8 (4ch × 2 units) | r01uh1032 §5.6.1 |
| Counter width | Selectable 16-bit or 32-bit | r01uh1032 §5.6.1 Table 5.6-1 |
| Prescaler | One of `PCLKL/8`, `/32`, `/128`, `/512` | r01uh1032 §5.6.1 Table 5.6-1 |
| I/O pins | Only CMTW0–3 have `TICn0`/`TICn1` (input capture) and `TOCn0`/`TOCn1` (output compare); CMTW4–7 have no pins | r01uh1032 §5.6.1 Table 5.6-3 |
| Interrupt sources | Compare match, input capture 0/1, output compare 0/1 — up to 5 per channel | r01uh1032 §5.6.1 Table 5.6-1 |
| Register bases | CMTW0 = `0x11C0_1800`, CMTW1 = `0x11C0_1C00`, CMTW2 = `0x11C0_2000`, CMTW3 = `0x11C0_2400`, CMTW4 = `0x1300_0C00`, CMTW5 = `0x1300_1000`, CMTW6 = `0x1300_1400`, CMTW7 = `0x1300_1800` | r01uh1032 §5.6.2 Table 5.6-4 |

### When You'd Actually Use This

**Mechanism**: CMTW adds input capture/output compare capability beyond GTM/OSTM, which in theory lets it measure external pulse widths or generate a compare-triggered output signal — but this capability has no corresponding driver you can control under this board's Linux, and it's currently only used by the kernel as a plain timer.

**Decision rule**: if your application needs to "measure the precise time interval of an external signal" (**take a rotary encoder's pulse interval as an example**, or measuring when an external trigger signal arrives), you need to confirm two things first:

1. Which pin the signal is actually wired to, and whether it lands on CMTW0–3's `TICn0`/`TICn1` — you'll need to check the schematic and the pinmux (pin multiplexing — one physical pin shared by several functions) settings in the PFC (Pin Function Controller, which decides what function a physical pin is currently switched to).
2. Whether Linux has a corresponding driver — **currently it does not**.

Without an existing kernel driver, CMTW's input capture capability falls into the category of "exists in silicon, unreachable from the Linux application layer." To actually use it, there are only two paths: write your own kernel driver, or switch to a realtime core (Cortex-R8/M33) firmware that operates the registers directly.

**Take measuring the pulse period of an industrial sensor as an example**: going through the Linux application layer, this path simply doesn't work today; only once the realtime requirement is worth the development cost does it make sense to evaluate moving this kind of measurement work to realtime-core firmware that operates CMTW directly. This is a textbook example of "the capability exists, the decision comes down to cost versus benefit" — and it has nothing to do with which project you're working on.

> Source: the official hardware manual r01uh1032 **§5.6 Compare Match Timer W (CMTW)** (p1254–1255 overview; registers from §5.6.2, p1256). Development notes doc07 §20. Pin naming (`TICn0/1`, `TOCn0/1`, n = 0 to 3) also cross-checked against datasheet `r01ds0429` Section 2 pin function table.

---

## 4. GPT ×16ch (General-Purpose Timer, the Only Timer That Can Output PWM)

### What This Is (the Mechanism)

GPT (General-Purpose Timer) is a 32-bit timer with 16 channels total, split into two groups: GPT0 covers ch0–7, GPT1 covers ch8–15 (verbatim: "The GPT is a 32-bit timer with 16 channels"). It's the richest unit in this group in terms of capability, and it's also the **only timer on this entire SoC that can generate PWM waveforms and has external I/O pins to do it with**. (r01uh1032 §5.7.1)

First, a quick explainer on PWM: PWM (Pulse Width Modulation) is a digital signal that expresses an analog magnitude through "what proportion of each cycle is spent at high level" (the duty cycle) — motor speed, servo angle, and LED brightness are all commonly controlled this way. Each GPT channel can produce a PWM waveform by controlling an up-counter, a down-counter, or an up/down-counter (triangular counting — counting up then back down).

What lets GPT produce "high-quality" power-control waveforms comes down to a few mechanisms (r01uh1032 §5.7.1.1):

- **4 external I/O pins per channel**: `GTIOCnA`, `GTIOCnB` (the main outputs) and `GTIOCnAN`, `GTIOCnBN` (the inverted versions of the first two). (Table 5.7-1; §5.7.2 Table 5.7-3)
- **Double buffering**: each channel has 2 sets of primary output compare/input capture registers (`GTCCRA`/`GTCCRB`), plus 4 sets of buffer/compare registers (`GTCCRC`/`D`/`E`/`F`), letting you switch buffered values at the instant of a wave peak or trough to produce "laterally asymmetric" PWM waveforms (verbatim: "laterally asymmetric PWM waveforms"). What double buffering means in practice: while the timer is still running, you can pre-load the new duty cycle for the next cycle into a buffer register, and it only takes effect at a safe switch point (a peak or trough), avoiding glitches partway through the waveform.
- **Dead time generation**: when switching upper and lower legs of a bridge, the hardware automatically inserts a short "both off" gap to avoid simultaneous conduction — this is a standard requirement for bridge-type power circuits (e.g., three-phase motor inverters) (verbatim: "Generation of dead times in PWM operation").
- **Tied into ELC/external triggers**: up to 8 ELC events can trigger count start/stop/clear/up-count/down-count/input capture; it also supports external trigger pins (`GTETRGA`–`D` for GPT0, `GTETRGE`–`H` for GPT1, via POEG) triggering the same actions, up to 4 external triggers. (Table 5.7-1; §5.7.2 Table 5.7-3)
- **Can directly trigger ADC conversion**: via the `GTADTRA`/`GTADTRB` compare registers, it can kick off an A/D conversion at a specific phase of the waveform — useful for motor-control scenarios like "measure current at a fixed point in the PWM cycle." (§5.7.1.1)
- **13 interrupt sources**: compare/capture on `GTCCRA`–`F`, counter overflow/underflow, dead-time error, A/B signal both-high or both-low, ADC trigger comparison, and more. (Table 5.7-2(2/2))

### How You See This Under Linux

This is the one unit in this group that most needs to be stated plainly: **GPT has no usable operating path from this board's Linux userspace.**

- **Board status (development notes)**: of the 16 `gpt@…` device tree nodes, only `gpt@13010000` (GPT0, covering ch0–7) is `okay`; the other 15 nodes are all `disabled` — and **no PWM chip is registered**. (unit-map 06; doc07 §18, verbatim: "No PWM chip is registered for it"; ✅ Verified on the board — all 16 nodes confirmed, only `gpt@13010000` is `okay`; transcript: live/ch04b-dt-status.txt)
- **No standard Linux PWM path**: the `/sys/class/pwm` directory exists but is empty — there's no `pwm-rzv2h`/`pwm-rzg2l` driver bound to `gpt@13010000`. Even though the DT node has its clock/reset probed, nothing is exposed to the application layer as a `/dev` node or sysfs control (doc07 §18, verbatim: "Effectively 'no Linux driver' for application use; counting/PWM must be driven bare-metal via MMIO"; ✅ Verified on the board: `/sys/class/pwm` is empty; transcript: live/ch04-reserved-mem.txt).
- **How to confirm this**:

```bash
ls /sys/class/pwm        # directory exists but is empty -> confirms no pwmchip registered
ls -A /sys/class/pwm | wc -l    # should print 0
```

- **The only remaining option**: use `devmem`/`mmap` under root to read/write the registers directly — that is, bare-metal MMIO (Memory-Mapped I/O, treating a hardware register as a directly readable/writable chunk of memory address space). Roughly, the flow is: first write the key `0xA5` to the `GTWP` register to lift write protection, then start counting with `GTSTR`. To actually output PWM you also need to set `GTIOR` (pin enable/level), `GTCR` (mode), `GTPR` (period), and `GTCCRA` (duty cycle), and switch the pin to the `GTIOCnA` function via the PFC. The kernel provides no helper functions for any of this. (doc07 §18/§24)

> ⚠️ **Note (wanting standard Linux PWM and not finding it)**:
> - **Scenario**: you want to use standard Linux PWM sysfs (`/sys/class/pwm`) to drive GPT for a servo output.
> - **Symptom**: there's no usable path under `/sys/class/pwm` — no `pwm-rzv2h` driver bound to `gpt@13010000`.
> - **Cause**: this board has no PWM chip driver configured — GPT channels do probe their clock/reset, but PWM control is not exposed to userspace.
> - **Prevention/Fix**: to operate PWM from the A55/Linux side, you have to go through bare-metal MMIO directly on the GPT registers; if you need low-jitter PWM tightly synchronized with a control loop, it's a better fit to hand it to a hard realtime core (see "When You'd Actually Use This" below) (source `07-hardware-unit-usage-guide.md:309`, `07-hardware-unit-usage-guide.md:418-421,434`).

> ⚠️ **Note (the `PWM0`/`PWM1` silkscreen labels on the 40-pin header don't mean ready to use)**:
> - **Scenario**: you see the 40-pin Raspberry Pi-compatible header's Pin32/33 silkscreened `PWM0`/`PWM1` (corresponding to `GPIO12`/`GPIO13`) and assume you can just plug in and get PWM out.
> - **Symptom**: PWM can't be used directly.
> - **Cause**: the device tree doesn't turn on the PWM function for these two pins.
> - **Prevention/Fix**: to use it you'd first have to modify the device tree to enable the pin function (source `04-hardware-quickref.md:112`), and confirm the pinmux really does route that pin to a GPT output. This is the same issue as "no PWM chip registered," just seen from a different angle.
>
> ✅ **The silkscreen half of this question is settled**: those pins really do land on GPT — the labels are not just borrowed Raspberry Pi naming.
>   Cross-check `REN_WS125V2HRDKREFZ_MAH` p.10 (the port-pin net name of every header pin) against the SoC manual `r01uh1032`
>   **Table 1.2-3 List of Multiplexed Functional Pins** (PDF p.118–127), pin by pin:
>   - **Pin 32 (`PWM0`) = `PA4` → `GTIOC6A`**
>   - **Pin 33 (`PWM1`) = `PA7` → `GTIOC7B`**
>
>   Renesas's own drone reference design ([`renesas-rdk/rzv2h_drone_px4`](https://github.com/renesas-rdk/rzv2h_drone_px4), `docs/HARDWARE.md`) annotates the same header on real hardware, and all four of its channels agree: pin 32 = GPT6A, pin 33 = GPT7B, pin 35 = GPT9A (`P96`), pin 31 = GPT10B (`P53`).
>
>   **The larger thing the same cross-check settles**: of the 40-pin header's 28 signal pins, **23 can be muxed to a `GTIOC` function** (the ones that cannot are the SPI6 group `P90`/`P91`/`P92`/`P93`, plus `PA0`). Because one `GTIOC` output can be routed to either of two pins while a pin can only carry one function at a time, the usable count is a bipartite maximum matching (the largest set of output↔pin pairings in which no output and no pin is used twice): **giving the whole header over to PWM yields at most 20 simultaneous outputs**; keeping every peripheral the reference design uses (GPS/telemetry/SBUS/LiDAR/I²C) still leaves **13**. (The project's PX4 integration notes, §8.2①, carry the pin-by-pin matching.)
>
>   ⚠️ Still open: **the RDK schematic is not in hand** (series resistors, level shifting, or an existing on-board load may rule out particular pins — p.10 of the board manual already shows the two I²C pins carrying 2.2 kΩ pull-ups R180/R181), and for 7 pins the "header pin number ↔ port pin" mapping is not yet pinned down. **Neither affects the counts above, but both have to be closed before you wire anything up.**

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| Channel count | 16 (GPT0: ch0–7; GPT1: ch8–15) | r01uh1032 §5.7.1; §5.7.2.1 Table 5.7-4 Note |
| Counter width | 32-bit | r01uh1032 §5.7.1 |
| I/O pins per channel | 4 (`GTIOCnA`/`B`/`AN`/`BN`; verbatim "Four input/output pins per channel") | r01uh1032 §5.7.1.1 Table 5.7-1 |
| Clock source | `clks_gpt` and its `/2 /4 /8 /16 /32 /64 /256 /1024` dividers, or external trigger `GTETRGA`–`GTETRGH` | r01uh1032 §5.7.1.1 Table 5.7-2(1/2) |
| Notable mechanisms | Double buffering, asymmetric PWM, dead-time generation, ADC conversion trigger, 13 interrupt sources | r01uh1032 §5.7.1.1 |
| Register bases | `<GPT0_base>` = `0x1301_0000` (ch0–7), `<GPT1_base>` = `0x1302_0000` (ch8–15) | r01uh1032 §5.7.2 Table 5.7-4 |
| Range usable on this board | Only `gpt@13010000` (GPT0 ch0–7) has DT `okay`; the other 15 nodes are `disabled`; **no PWM chip registered** | unit-map 06; doc07 §18; ✅ Verified on the board (transcript: live/ch04b-dt-status.txt) |

> ⚠️ **Note (separate the "theoretical silicon ceiling" from "what's actually usable on this board's Linux")**: the Manual is describing silicon-level capability (16 channels, 4 pins per channel); this board's device tree currently only has an `okay` node for GPT0 ch0–7, those 8 channels, and no PWM chip. When citing GPT's capability, be sure to keep these two layers separate — don't let the reader come away thinking 16 channels/64 outputs are all directly usable under this board's Linux.

### When You'd Actually Use This

**Mechanism**: GPT is the only timer on this SoC with external I/O pins that can produce PWM waveforms — GTM/OSTM has no pins at all; CMTW0–3 do have pins, but only for input capture/output compare, with no PWM waveform generation capability. If you need to output PWM, GPT is hardware's only candidate.

**Decision rule**: if the application needs PWM output (**take motor control, servo drive, or LED dimming as examples**, or any scenario needing an adjustable-duty-cycle waveform), first answer "who's going to drive it":

- Linux userspace currently has **no pwmchip**. If you insist on operating it from the Linux application layer, the only path is bare-metal MMIO under root privileges — the risk being there's no kernel protection mechanism, so it's easy to mis-write into a register region some other process might also be using.
- If you need "realtime, low-latency PWM output tightly synchronized with other control loops" (**take motor inverter commutation as an example**), the more solid approach is to hand the entire GPT operation over to a core that isn't subject to Linux scheduling jitter. On this SoC, the Cortex-R8/M33 not claimed by Linux both qualify. That beats doing bare-metal operation from the Linux application layer.

**Take motor control as an example**: when you need a precise duty cycle with low jitter, GPT's dead-time generation and double buffering are exactly the hardware-level capabilities designed for this kind of application (the mechanism); but "who drives it" still depends on the realtime requirement (the decision rule). **Take simple LED dimming as another example**: if you can tolerate Linux scheduling delay, bare-metal MMIO is workable too — it's just that you'll have to reconfigure the registers on every boot, since there's no persistence mechanism in the kernel for this. The difference between these two examples isn't "which project" — it's the realtime requirement, which is the decision rule; readers can apply the same line of reasoning to their own application.

> Source: the official hardware manual r01uh1032 **§5.7 General-Purpose Timer (GPT)** (p1283–1287 overview; registers from §5.7.2, p1288). Development notes doc07 §18; pin naming (`GTIOCnA/B/AN/BN`, `GTETRGA-H`) also cross-checked against datasheet `r01ds0429` Section 2; 40-pin header silkscreen documented in `04-hardware-quickref.md` and the WS125 RDK carrier board manual.

---

## 5. POEG/PWM Output (GPT Output Pins' "Fail-Safe Gate")

### What This Is (the Mechanism)

Let's clear up the easiest concept to misunderstand first: **PWM is not a standalone peripheral on RZ/V2H.** The development notes explicitly state "PWM-output — NOT a standalone peripheral on RZ/V2H." The logic that generates the PWM waveform lives entirely in GPT (§5.7), while POEG (§5.8) is only responsible for the layer that decides "should the output pin be allowed through or not." Together, the two form one complete PWM output chain. (doc07 §24; unit-map 06)

POEG (Port Output Enable for GPT) is not itself a timer and doesn't count — it's a **protective gate** on GPT's output pins: it can switch a GPT output pin to a disabled state. The reason it exists is "fail-safe": if a software bug ever drives the power stage into a dangerous state via PWM, POEG can cut the output at the hardware level immediately, without waiting for the CPU to react. (r01uh1032 §5.8.1)

There are three ways to trigger POEG to disable its output (r01uh1032 §5.8.1 Table 5.8-1):

1. **Input level detection**: the `GTETRGn` pin detects a rising edge or high level (after polarity/filter selection).
2. **A GPT-initiated output-disable request**: when `GTIOCmA` and `GTIOCmB` are simultaneously driven to their active level (for example, both the upper and lower legs of a bridge conducting at once — a dangerous state), GPT issues a request, and POEG decides based on that whether to disable that pair of output pins.
3. **Software directly writing to a register to disable it.**

It has noise filtering built in: for the `GTETRGn` input, you can select `PCLKB/1`, `/8`, `/32`, or `/128` as the filter sampling clock, and the signal is only treated as valid once 3 consecutive samples agree — avoiding a false trigger from brief noise. (r01uh1032 §5.8.1 Table 5.8-1)

POEG is split into 4 groups each for GPT0/GPT1, 8 groups total, each gating independently: POEG0A–D correspond to GPT0 (external pins `GTETRGA`–`D`), POEG1A–D (Manual pin naming `GTETRGE`–`H`) correspond to GPT1. (r01uh1032 §5.8.2 Table 5.8-3; §5.8.1 Table 5.8-2)

On "how many PWM outputs at most": the development notes work out "4 I/O pins × 16 GPT channels = up to 64 PWM-capable outputs, gated by 8 POEG groups" (doc07 §24, verbatim: "up to 4 I/O pins x16 GPT channels = up to 64 PWM-capable outputs, gated by 8 POEG groups"). This figure of 64 is a **theoretical silicon ceiling worked out** from the official spec (4 pins per channel × 16 channels), consistent with §5.7.1.1's "Four input/output pins per channel" and the 16-channel spec. It is **not a number measured on this board**: in practice this board only has the 8 channels of GPT0 ch0–7 with `okay` nodes (see the previous section).

### How You See This Under Linux

Same as GPT — this board's Linux side has no path to it:

- **No Linux driver**: there's neither a `pwm-rzv2h`/`pwm-rzg2l` pwmchip nor a POEG driver; `/sys/class/pwm` is empty. (doc07 §24, verbatim: "there is no pwm-rzv2h/pwm-rzg2l pwmchip and no POEG driver; /sys/class/pwm is absent"; ✅ Verified on the board: `/sys/class/pwm` is empty; transcript: live/ch04-reserved-mem.txt)
- **The full flow to use PWM**: program the GPT registers directly and switch the pin to the `GTIOCnA` function via the PFC, then, if needed, set the corresponding POEG group's `POEGGn` register (doc07 §24's usage example: writing `GTWP`/`GTPR`/`GTCCRA`/`GTIOR`/`GTSTR` directly with `devmem`).
- **Recommended path**: the development notes' recommendation for "who should drive GPT + POEG for realtime PWM control" is to hand it to a realtime core not subject to Linux scheduling (on this board, that's Cortex-R8, though that core isn't exposed to Linux and would first need the remoteproc/firmware-loading path — remoteproc being the Linux framework for loading and starting firmware on a co-processor), rather than bare-metal operation from the Linux application layer. This is a general realtime-requirement judgment call, unrelated to any particular application project. (doc07 §24)

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| POEG group count | 8 (POEG0A/B/C/D correspond to GPT0; POEG1A/B/C/D, pin naming `GTETRGE`–`H`, correspond to GPT1) | r01uh1032 §5.8.2 Table 5.8-3 |
| Noise filter clock | `PCLKB/1`, `/8`, `/32`, `/128` (valid only once 3 consecutive samples agree) | r01uh1032 §5.8.1 Table 5.8-1 |
| Register per group | One 32-bit control register `POEG_POEGGn`, offset `0x0000` | r01uh1032 §5.8.2.1 |
| Register bases | POEG0A = `0x1300_1C00`, 0B = `0x1300_2000`, 0C = `0x1300_2400`, 0D = `0x1300_2800`, POEG1A(E) = `0x1300_2C00`, 1B(F) = `0x1300_3000`, 1C(G) = `0x1300_3400`, 1D(H) = `0x1300_3800` | r01uh1032 §5.8.2 Table 5.8-3 |
| Theoretical maximum outputs | Up to 64 PWM-capable outputs (4 pins × 16 channels, worked out from the official spec, **not measured on this board**) | doc07 §24 |
| Board status | `/sys/class/pwm` is empty (no pwmchip, no POEG driver) | unit-map 06; doc07 §24; ✅ Verified on the board (transcript: live/ch04-reserved-mem.txt) |

### When You'd Actually Use This

**Mechanism**: POEG's reason for existing is "safety" — if a software bug ever causes the PWM GPT outputs to drive the upper and lower bridge legs into simultaneous conduction (`GTIOCmA` and `GTIOCmB` both active at once), POEG can automatically detect this and cut the output. This is a common hardware protection mechanism for bridge-type drive circuits (H-bridges, three-phase inverters), and the key point is that it **doesn't have to wait on CPU interrupt handling** — it reacts faster than a purely software-based check.

**Decision rule**:

- If the PWM output is simple signal generation (**take a servo motor position command or LED dimming as examples**) that doesn't involve a bridge-type power stage, whether to enable POEG's fault protection depends on whether you already have other protection measures in place.
- If the PWM is driving a power circuit that "can be damaged by a short or simultaneous conduction" (**take an H-bridge motor drive or a DC/DC converter as examples**), POEG's "automatic output disable" is a hardware protection layer worth evaluating — its fundamental advantage over a purely software-based safeguard is that it doesn't go through the CPU, so it reacts faster.

**Take motor control as an example**: the value of the hardware-level output disable POEG provides is that it can take effect the instant a fault occurs — that's the whole point of fail-safe design. But this board's Linux side currently has no driver at all to configure it, so actually using it means going down the same bare-metal MMIO or realtime-core path as GPT. When readers are evaluating their own application, the deciding line is: "can the power circuit my PWM drives be damaged by a malfunction?" If yes, factor this hardware protection layer, POEG, into your design.

> Source: the official hardware manual r01uh1032 **§5.8 Port Output Enable for GPT (POEG)** (p1504–1505 overview; registers from §5.8.3) + **§5.7 GPT** (PWM waveform generation lives in GPT). Development notes doc07 §24.

---

## 6. WDT ×4ch (Watchdog Timer)

### What This Is (the Mechanism)

WDT (Watchdog Timer) is a 14-bit down-counter whose role is "the last line of defense when the system loses control." When the system has run away and can no longer periodically "refresh" the counter, the counter counts all the way down to 0 (underflow) and WDT steps in. It can reset the entire chip, or instead be configured to generate an NMI (Non-Maskable Interrupt — the highest-priority interrupt, one that can't even be blocked by "disabling interrupts") or an underflow interrupt. (r01uh1032 §5.4.1, verbatim: "a 14-bit down counter that can be used to reset this LSI when the counter underflows because the system has run out of control and is unable to refresh the WDT. In addition, the WDT can be used to generate a non-maskable interrupt or an underflow interrupt")

This SoC has **4 independent WDT instances, each bound to a different core**: WDT0 is bound to CM33, WDT1 to CA55 (all A55 cores), WDT2 to CR8 core0, WDT3 to CR8 core1. Every processor core has its own dedicated watchdog, each independent of the others. (r01uh1032 §5.4.2 Table 5.4-2)

The clock source for counting is **LOCO** (Low-speed On-Chip Oscillator), with selectable dividers of `1/16/32/64/128/256`. LOCO is a not-especially-precise but self-contained oscillator that doesn't depend on any external component. Precisely because it is independent, it suits being the time base for a watchdog that needs to "keep running even if the main clock itself has gone down." (r01uh1032 §5.4.1 Table 5.4-1)

The way you "refresh" it (colloquially, "feeding the dog") is: within the refresh-permitted window, write `00h` and then write `FFh` to the `WDTRR` register, in that order. (r01uh1032 §5.4.2.2.1, verbatim: "The down-counter is refreshed by writing 00h and then writing FFh to WDTRR register (refresh operation) within the refresh-permitted period")

WDT also has an advanced **window** feature: you can configure a time window of "refresh allowed" and "refresh forbidden" — refreshing too early or too late is both treated as a refresh error and triggers an interrupt. This is used to detect an anomaly like "the program fed the dog too early" — for example, some code path skipping work it was supposed to do but still refreshing on schedule. (r01uh1032 §5.4.1 Table 5.4-1) The timeout period is jointly determined by `CKS` (the divider) and `TOPS` (Timeout Period Select), where `TOPS` can select 1024/4096/8192/16384 "post-divider clock" cycles. The window's start position (`RPSS`) can be 25%/50%/75%/100% (100% meaning no restriction on the start), and its end position (`RPES`) can be 75%/50%/25%/0% (0% meaning no restriction on the end). (r01uh1032 §5.4.2.2.2)

### How You See This Under Linux

- **Only WDT1 (the one bound to CA55) is exposed under Linux**: device node `/dev/watchdog0`, driver `rzv2h_wdt` (kernel source `drivers/watchdog/rzv2h_wdt.c`), using the standard Linux watchdog ioctl API (`WDIOC_*`). (doc07 §22)
- **WDT0 (CM33) and WDT2/WDT3 (CR8 core0/1) aren't managed by Linux**: they belong to the respective firmware running on those cores; Linux can't see them or control them. (unit-map 06: "WDT0/2/3 belong to the CM33/CR8 firmware")
- **Tools and usage**:

```bash
wdctl                     # util-linux, query watchdog capabilities
# systemd's RuntimeWatchdogSec can refresh it automatically on your behalf;
# or open an fd yourself and refresh manually:
exec 3>/dev/watchdog0     # opening it arms it (starts the countdown)
printf '1' >&3            # writing any byte = refresh (feed the dog)
printf 'V' >&3; exec 3>&- # writing 'V' is the "magic close" — gracefully disarm, then close the fd
# ⚠️ On this board MAGICCLOSE=0: writing 'V' has no effect and there is no clean disarm — see
# the Hands-On section below, step 2 and its note box
```

(doc07 §22 usage example)

> ⚠️ **Note (pick a refresh period between the two extremes; too strict a window causes false triggers)**:
> - **Scenario**: you've hooked up `/dev/watchdog0` for some long-running service, wanting it to automatically reboot if the service ever hangs.
> - **Symptom**: refresh period set too short → even normal execution might not feed the dog in time and gets falsely reset; set too long → when the system genuinely hangs, it takes a long time to recover. Turn on the window feature and refresh too early → triggers a refresh error.
> - **Cause**: WDT's timeout is determined by "how long since the last refresh, with no refresh since"; the window feature also treats "refreshing too early" as an error.
> - **Prevention/Fix**: set the refresh period somewhere between "the time your main loop normally takes to run one full pass" and "the longest period of unresponsiveness you can tolerate"; for ordinary applications that don't need strict timing verification, just set the window start to 100% (no restriction on the start) and keep only the "refreshed too late" trigger condition.

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| Channel count | 4 (verbatim "Number of channels 4 channels") | r01uh1032 §5.4.1 Table 5.4-1 |
| Counter width | 14-bit | r01uh1032 §5.4.1 |
| Divider options | LOCO clock's `/1`, `/16`, `/32`, `/64`, `/128`, `/256` | r01uh1032 §5.4.1 Table 5.4-1 |
| Timeout period options | 1024/4096/8192/16384 post-divider clock cycles | r01uh1032 §5.4.2.2.2 |
| Window start/end | Start 25%/50%/75%/100% (no start restriction); end 75%/50%/25%/0% (no end restriction) | r01uh1032 §5.4.2.2.2 |
| Core mapping and bases | `<WDT0_base>` = `0x11C0_0400` (CM33), `<WDT1_base>` = `0x1440_0000` (CA55), `<WDT2_base>` = `0x1300_0000` (CR8 Core0), `<WDT3_base>` = `0x1300_0400` (CR8 Core1) | r01uh1032 §5.4.2 Table 5.4-2 |
| Linux exposure | Only WDT1 = `/dev/watchdog0` (`rzv2h_wdt`); WDT0/2/3 belong to their respective core's firmware | doc07 §22; unit-map 06 |

### Hands-On: Open It and Feed It, Read Out the Timeout, and Watch It Actually Bite (Reset)

This is one of the few units in this group where you can walk the **entire path** from Linux — arming it, feeding it, and letting it genuinely reset the whole board. And precisely because the last step **really does reboot the board**, think each step through before you run it.

**Mechanism**: the Linux watchdog convention is that **the moment you open `/dev/watchdog0`, the countdown starts** (arming); from then on, writing any byte to that fd counts as one "refresh" (feeding the dog) and reloads the countdown; the only clean way to stop it is to write the magic close character `V` and then close the fd. On the hardware side this is exactly the `WDTRR` refresh and the underflow reset described in section 6: miss a feed, the counter reaches 0, and WDT1 reboots the chip through the **WDT → ICU → CPG reset chain** (see [05-system-backbone-interrupts-clocks-power-dma-event-link.md](05-system-backbone-interrupts-clocks-power-dma-event-link.md) §1, ICU).

**Step 1: confirm the device is there — but don't touch `/dev/watchdog0` yet.** (✅ Verified on the board; transcript: `live/ch4-w1-wdt-pre.txt`)

```bash
ls -l /dev/watchdog*
```

```text
crw------- 1 root root  10, 130 Jul 21 11:15 /dev/watchdog
crw------- 1 root root 243,   0 Jul 21 11:15 /dev/watchdog0
```

Note that this **deliberately avoids `cat /dev/watchdog0` to "just take a look"** — merely opening it arms the watchdog and starts the countdown. The read-only way to inspect its capabilities would be `/sys/class/watchdog/watchdog0/`; on this board, however, the attribute files under there — `timeout`, `identity`, `state` and the rest — **do not exist at all** (each one gives a literal `No such file or directory`): this kernel was not built with the watchdog sysfs attributes. So **there is no path on this board that reads the timeout without arming the watchdog** — your only option is `wdctl` in the next step, which opens the device and therefore arms it too.

**Step 2: read the capabilities with `wdctl` — knowing that this step has already let the dog out.** (✅ Verified on the board; transcript: `live/ch4-w1-wdt-bite.txt`)

> ⚠️ **Note (the `wdctl` line below arms the watchdog the instant it opens the device; this board has no clean disarm, so it will reboot in about 60 seconds)**: to read the capabilities `wdctl` has to open `/dev/watchdog0`, and "opening arms it" is the general rule of the watchdog API; this board has `MAGICCLOSE=0` and no way to disarm cleanly partway through. Run that line and leave no process feeding the dog afterwards, and about 60 seconds later **the whole board** (every core, and everyone else's sessions and services) takes a hardware reboot. **A remote SSH session will drop, and on a shared board this affects other people.** Either be ready for the reboot, or immediately keep it fed with `exec 3>/dev/watchdog0; printf '1' >&3`.

```bash
sudo wdctl /dev/watchdog0
```

```text
Device:        /dev/watchdog0
Identity:      Renesas RZ/V2H WDT Watchdog [version 0]
Timeout:       60 seconds
Pre-timeout:    0 seconds
FLAG           DESCRIPTION               STATUS BOOT-STATUS
KEEPALIVEPING  Keep alive ping reply          1           0
MAGICCLOSE     Supports magic close char      0           0
SETTIMEOUT     Set timeout (in seconds)       0           0
```

Three hard facts about this board come straight out of that, and they decide how you can use it:

- **The timeout is fixed at 60 seconds**, and the `SETTIMEOUT` flag is `0` — **this board does not support changing the timeout**, so 60 seconds is the value you have to live with.
- `MAGICCLOSE` is `0` — **this board does not support magic close**. That means the "write `V` to gracefully disarm" line shown in "How You See This Under Linux" above **does not hold on this board** (the note box below spells it out).
- `KEEPALIVEPING` is `1` — writing any byte does refresh it, so that part works. Feeding the dog is just: open an fd and write a byte periodically — `exec 3>/dev/watchdog0; printf '1' >&3` (but once you've opened it you're committed to feeding it; see below).

**Step 3: let it actually bite (reset) — this reboots the board, so read before you run.** (✅ Verified on the board; transcripts: `live/ch4-w1-wdt-verify.txt`, `ch4-w1-wdt-bite.txt`, `ch4-w1-wdt-verify2.txt`)

Arming it on purpose and then not feeding it means: open the fd, close it immediately, and send no magic close.

> ⚠️ **This is a whole-machine reset, not just a restart of your process**: the line below makes **the entire board** (every core, every other user and service along with it) take a hardware reboot about 60 seconds later; your SSH/network session will drop, and on a shared board it affects other people. Make sure that is acceptable and that nobody else is using the board before you run it.

```bash
sudo sh -c '> /dev/watchdog0'    # opening arms it, closing doesn't disarm it -> hardware reset in 60 s
```

The moment you run it, the key line shows up in `dmesg`:

```text
watchdog: watchdog0: watchdog did not stop!
```

That line is the evidence: the fd was closed, but the watchdog **did not stop** (magic close isn't supported, and closing the fd doesn't disarm it) — it keeps counting down. About 60 seconds later the board resets in hardware. After reconnecting, verify against the boot time:

```bash
who -b ; awk '{print "uptime_sec="$1}' /proc/uptime
```

```text
         system boot  2026-07-22 15:12
uptime_sec=52.51
```

The time chain has to line up: **the moment you trigger it, plus the 60-second timeout, is when it bites — and that is the boot time `who -b` reports** (in the output above, triggered at 15:11:35 → bite around 15:12:35 → `who -b` showing 15:12). Once back up, confirm the device identity — that what bit really was WDT1, the instance bound to CA55 (transcript: `ch4-w1-wdt-verify2.txt`):

```bash
cat /sys/class/watchdog/watchdog0/device/uevent
```

```text
DRIVER=rzv2h_wdt
OF_NAME=watchdog
OF_FULLNAME=/soc/watchdog@14400000
OF_COMPATIBLE_0=renesas,r9a09g057-wdt
```

`OF_FULLNAME=/soc/watchdog@14400000` matches `<WDT1_base>` = `0x1440_0000` (CA55) in this section's capability table, with driver `rzv2h_wdt` — confirming that Linux's `/dev/watchdog0` is WDT1.

**Pass criteria**:
- What a successful bite looks like: after reconnecting, `/proc/uptime` is far smaller than the time you spent waiting for the board to come back (52.51 s above, against a much longer trigger-to-reconnect gap), and the `who -b` boot time ≈ trigger time + 60 s.
- **The boot `dmesg` carries no reset-cause line** — the RZ/V2H BSP doesn't record the source of a reset. So "was it the watchdog?" is decided by the `watchdog did not stop!` line **from before the reboot** plus the time chain; don't expect the boot log to confess, and don't read "no reset-cause line" as "it didn't bite."

> ⚠️ **Note (no magic close on this board: once opened, there's no clean way out)**
> - **Scenario**: you only wanted to run `wdctl`, or open `/dev/watchdog0` briefly to look at it, or write `V` to turn it off again.
> - **Symptom**: `dmesg` shows `watchdog: watchdog0: watchdog did not stop!`, and 60 seconds later the board reboots without warning.
> - **Cause**: this board has `MAGICCLOSE=0` — writing `V` does not disarm it; and "opening arms it" is the general rule of the watchdog API, so even `wdctl` arms it just by opening the device to read its capabilities. Once it has been opened and no process keeps feeding it, it will bite.
> - **Prevention/Fix**: treat `/dev/watchdog0` as a resource you are committed to feeding the moment you open it. Either hand it to systemd (set `RuntimeWatchdogSec=` in `/etc/systemd/system.conf` and let systemd feed it), or keep it open in your own process and refresh it periodically with `printf '1' >&3`. Don't open it just to "take a look." If you do arm it by accident and don't want to wait out a reboot, the only move is to start feeding it immediately and keep doing so — this board has no clean mid-flight disarm.

> ⚠️ **Note (pick the timeout between two bounds — and on this board 60 seconds isn't adjustable)**
> - **Scenario**: you hook `/dev/watchdog0` up to a long-running service and want to set a convenient timeout.
> - **Symptom**: a refresh period longer than 60 seconds → normal execution gets falsely reset; and `wdctl -s` has no effect when you try to change the timeout.
> - **Cause**: this board has `SETTIMEOUT=0`, so the timeout is locked at 60 seconds; and the timeout counts "how long since the last refresh, with no refresh since."
> - **Prevention/Fix**: put the refresh period between "the time your main loop normally takes for one full pass" and "60 seconds," with plenty of margin (feeding every 20–30 seconds, for instance); a different timeout value has to come from firmware or the device tree — it isn't something Linux userspace can change.

### When You'd Actually Use This

**Mechanism**: the watchdog's role is "the last line of defense when the system loses control" — when the main control loop hangs (deadlock, an infinite loop, or memory corruption causing the process to stop responding), and no other mechanism can recover the system automatically, WDT's timeout forces a chip reset.

**Decision rule**: any system running unattended, or in a context requiring long-term stable operation (**take an industrial controller, a remote monitoring node, or any automation device as examples**), should first ask: "if the main loop hangs, who brings the system back?" —

- If the answer is "no one," you should enable the WDT on the relevant core, and have the main loop refresh it periodically.
- The refresh period should be set somewhere between "the time your main loop normally takes to run one full pass" and "the longest period of unresponsiveness you can tolerate" — too short causes false triggers, too long lets the system stay hung for too long before recovering.
- The window feature is for detecting the anomaly of "refreshed too early"; ordinary applications that don't need this strict a timing check can just set the window start to 100% (no restriction on the start).

**Which core owns the loop decides which path you take**: the Linux application layer (running on CA55) can hook directly into WDT1 via `/dev/watchdog0`; but if your realtime control loop runs on R8/M33, that core's own WDT (WDT2/WDT3 or WDT0) has to be **refreshed from within its own firmware** — Linux can neither manage it nor see its state. This decision rule follows directly from the mechanism of "4 WDTs, each bound to one core": whichever core your critical loop runs on, use that core's own WDT.

> Source: the official hardware manual r01uh1032 **§5.4 Watchdog Timer (WDT)** (p1218–1219 overview; registers from §5.4.2, p1220). Development notes doc07 §22.

---

## 7. RTC (RTCA-3, Realtime Clock)

### What This Is (the Mechanism)

RTC (Realtime Clock) — this SoC's part is the RTCA-3. Its most fundamental difference from every timer covered so far is: **it keeps track of "what time it is right now," not "how long since boot"** — and thanks to its own independent low-frequency crystal oscillator and standby power supply, its count keeps running even through a full system power-off/reboot, or while in suspend.

It offers two counting modes, switched via a register (r01uh1032 §5.3.1):

- **Calendar count mode**: a calendar spanning the 100 years from 2000 to 2099 CE, represented in BCD (Binary-Coded Decimal — every 4 bits represents one decimal digit, making it convenient to display year/month/day directly), with automatic handling of leap years and 12/24-hour switching.
- **Binary count mode**: doesn't track year/month/day/hour/minute, just counts seconds as a 32-bit binary value, usable for non-Gregorian calendar contexts. (verbatim: "it counts seconds, and retains the information as a serial value. This mode can be used for calendars other than the Gregorian calendar")

The clock source is a 32.768 kHz crystal oscillator (attached to the `RTXIN`/`RTXOUT` pins; 32.768 kHz is the RTC industry-standard frequency, being exactly 2 to the 15th power Hz, which makes it easy to divide down to 1 Hz), internally divided down to produce a 128 Hz reference clock. (r01uh1032 §5.3.1 Table 5.3-1; §5.3.2 Table 5.3-2)

It has 3 kinds of interrupts (r01uh1032 §5.3.1 Table 5.3-1(2/2)):

- **Alarm (ALM)**: in calendar mode, you can select any combination of year/month/day/day-of-week/hour/minute/second fields to compare against; in binary mode, you can select any bit of the 32-bit counter to compare against.
- **Periodic (PRD)**: selectable period of 2 seconds, 1 second, 1/2, 1/4, 1/8, 1/16, 1/32, 1/64, or 1/128 second — 9 settings total.
- **Carry (CUP)**: triggers when the 64 Hz counter carries into the seconds counter, or when reading `R64CNT` coincides with a carry.

All three of these events can be output directly to the ELC (without going through the CPU). It also has a set of read-only mirror registers (base `<RTC_Read_Only_base>`), whose purpose is to avoid reading an inconsistent value when "a carry happens to occur at the exact moment you're reading" — reading from the mirror region means you'll never catch a value mid-update from a carry. (r01uh1032 §5.3.2 Table 5.3-3 Note 3)

### How You See This Under Linux

- **Device node `/dev/rtc0`**, driver binding name `rtca3` (the Renesas RTCA-3 binding), using the standard Linux RTC ioctl API (`RTC_RD_TIME`, `RTC_ALM_SET`, `RTC_WKALM_SET`, `RTC_AIE_ON`). (doc07 §23)
- **Board status**: `hwclock -r` works. (unit-map 06; doc07 §23)
- **Tools and usage**:

```bash
hwclock -r                        # util-linux, reads the hardware clock
hwclock -s                        # sets the system time from the RTC's time (often placed in a boot script)
timedatectl                       # systemd's time management
rtcwake -d rtc0 -m mem -s 60      # sets an RTC alarm, suspends to RAM, auto-wakes after 60 seconds
```

(doc07 §23)

> ⚠️ **Note (there are two time-keeping chips on this board; `/dev/rtc0` only corresponds to one of them)**:
> - **Scenario**: you want to confirm exactly which piece of hardware `/dev/rtc0` corresponds to, or you're trying to find "the other RTC."
> - **Symptom**: while researching, you find the board seems to have RTC functionality in more than one place, which is easy to confuse.
> - **Cause**: the WS125 carrier board separately has an independent PMIC (power management IC, part number `RAA215300A2GNP#HA2`, a 9-channel PMIC with RTC — see the BOM table U16 in `ws125-rdk-board-manual.md`) that also has RTC functionality built in; but `/dev/rtc0` corresponds to the **RTCA-3 built into the SoC** (the driver binding name `rtca3` makes this explicit) — the two are different chips.
> - **Prevention/Fix**: treat `/dev/rtc0` as the SoC's RTCA-3. Whether the PMIC's built-in RTC has its own independent Linux driver node is something the development notes never inventoried — don't assume it exists; verify it yourself before relying on it. (doc07 §23; BOM in the WS125 RDK carrier board manual)

### Key Capabilities & Limits

| Item | Value/Content | Source |
|---|---|---|
| Count modes | calendar (2000–2099, BCD, automatic leap years) / binary (32-bit seconds) | r01uh1032 §5.3.1 Table 5.3-1 |
| Clock source | 32.768 kHz (`RTXIN` external crystal), 128 Hz reference clock | r01uh1032 §5.3.1 |
| Alarm comparison granularity | In calendar mode, year/month/day/day-of-week/hour/minute/second can each be individually selected for comparison | r01uh1032 §5.3.1 Table 5.3-1(2/2) |
| Periodic period | 2 seconds, 1 second, 1/2, 1/4, 1/8, 1/16, 1/32, 1/64, 1/128 second — 9 settings total | r01uh1032 §5.3.1 |
| Register bases | `<RTC_base>` = `0x11C0_0800` (read/write), `<RTC_Read_Only_base>` = `0x11C0_0C00` (read-only mirror) | r01uh1032 §5.3.2 Table 5.3-3 |
| Instance count | A single instance (unlike WDT/GTM, no multiple channels) | unit-map 06; doc07 §23 |
| Linux exposure | `/dev/rtc0` (`rtca3`); `hwclock -r` works | doc07 §23; unit-map 06 |

### Hands-On: Read the RTC, See How It Relates to the System Clock and to NTP, and What Happens After a Power Loss

**Mechanism**: there are really two clocks running on this board — the **RTC** (`/dev/rtc0`, which remembers "what time it is now") and the **system clock** (maintained by the kernel: zeroed at boot, then advanced by the monotonic clock). They meet at two points: at boot the kernel reads the RTC once to seed the system time (`hctosys`), and after boot NTP (Network Time Protocol — here `systemd-timesyncd`) keeps correcting the system clock. The RTC takes no part in ordinary system timekeeping — it is only read or written at the boot and suspend/resume boundaries.

**Step 1: confirm the device identity and `hctosys`.** (✅ Verified on the board; transcript: `live/ch4-w1-rtc.txt`)

```bash
cat /sys/class/rtc/rtc0/name
grep -H . /sys/class/rtc/rtc0/date /sys/class/rtc/rtc0/time /sys/class/rtc/rtc0/hctosys
```

```text
rtc-rtca3 11c00800.rtc
/sys/class/rtc/rtc0/date:2026-07-22
/sys/class/rtc/rtc0/time:07:06:45
/sys/class/rtc/rtc0/hctosys:1
```

`rtc-rtca3 11c00800.rtc` matches `<RTC_base>` = `0x11C0_0800` in the capability table, confirming that `/dev/rtc0` is the SoC's built-in RTCA-3 (not the RTC inside the carrier board's PMIC — see the note box above in this section). `hctosys:1` is the key line: it means **the kernel used this RTC's time to set the system clock at boot** (hardware clock → system). Note also that sysfs `time` here is **UTC** (07:06:45), not local time.

**Step 2: read it with `hwclock`/`timedatectl`, and see the division of labor between RTC and NTP.** (✅ Verified on the board; same transcript)

```bash
sudo hwclock -r          # /dev/rtc0 is root-only, mode 600, so sudo is required
timedatectl
```

```text
2026-07-22 15:06:46.002293+08:00
```

```text
               Local time: Wed 2026-07-22 15:06:47 CST
           Universal time: Wed 2026-07-22 07:06:47 UTC
                 RTC time: Wed 2026-07-22 07:06:47
                Time zone: Asia/Taipei (CST, +0800)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no
```

Three things to read out of that together:

- **`RTC in local TZ: no`** — the RTC stores UTC. So the `+08:00` local time that `hwclock -r` prints is the tool converting for you; what the hardware holds is UTC. This is the recommended Linux arrangement (RTC in UTC, time zone handled in software).
- **`NTP service: active` while `System clock synchronized: no`** — these two lines don't contradict each other. The NTP service is running; it just hasn't marked the system as "synchronized" **at this moment**. Looking at the time-source status makes it obvious:

```bash
timedatectl timesync-status
```

```text
       Server: 103.186.118.217 (tw.pool.ntp.org)
Poll interval: 16h (min: 30min; max 1d)
      Stratum: 2
       Offset: +3.559ms
```

An offset of only +3.559 ms with the poll interval already stretched out to 16 hours is the normal quiet state after a successful sync has dropped into low-frequency polling — not a failure. **Don't read `synchronized: no` as "NTP is broken."**

**Step 3: what happens to the RTC after a power loss — what holds on this board, and where the evidence stops.**

First separate the two kinds of "reboot," which is where this is most often misjudged:

- **Warm reset (power never drops)**: the watchdog bite from the previous unit, for example. The RTC domain stays powered throughout and the time is unaffected. This is verifiable: after a WDT-bite reboot, `date` comes straight back with the correct `Wed Jul 22 03:13:33 PM CST 2026` and nothing has jumped. (✅ Verified on the board; transcripts: `live/ch4-w1-wdt-bite.txt` for the bite, `live/ch4-w1-wdt-verify.txt` for the post-reboot reading.)
- **Full power loss (supply removed)**: to keep counting while the board is dark, RTCA-3 needs **its own standby supply** (a battery or supercapacitor holding up the `RTXIN` crystal and the RTC domain). The reason the handbook's Chapter 1 setup insists on configuring NTP time sync ("Set the time zone and NTP time sync" states it plainly: the board has an RTC, but staying accurate depends on NTP) is exactly this: **this board's RTC cannot be treated as a time source you trust across a power cut** — once standby power is insufficient, the RTC drifts or zeroes while the board is off, and the boot timestamp is stale or wrong.

> ⚠️ **Note (don't make the RTC your only trusted time source)**
> - **Scenario**: you unplug the board to move it, power it back up, and immediately look at log timestamps — or do something that needs a correct time (**take certificate/TLS expiry validation as an example, or aligning data from several sources onto one timeline as another**).
> - **Symptom**: early in the boot the time is a stale value or clearly off; log timestamps don't line up, and certificate validation may fail because of the wrong time.
> - **Cause**: with no adequate standby supply this board's RTC doesn't count while the power is off; at boot, `hctosys` faithfully copies whatever the RTC currently holds (possibly wrong) into the system clock, and NTP only pulls it back once the network is up and a round has completed.
> - **Prevention/Fix**: always have the boot flow rely on NTP for time (already set up in Chapter 1); once NTP has settled, write the correct system time **back** into the RTC with `sudo hwclock -w` so the next (warm) boot starts from a good value. If you genuinely need "keeps time across a power cut," confirm for yourself that the carrier board's RTC standby supply (battery or supercapacitor) is actually fitted and working.

**Evidence boundary**: the first-hand evidence on this board covers only the warm-reset case — time still correct after a WDT bite. The destructive test, pulling the power completely and measuring whether the RTC readback has zeroed, is **not covered here**, so "what value this board's RTC comes back with after a full power loss" stands as **pending first-hand evidence (full power-cycle readback)**. The guidance above is anchored in the mechanism (standby power is the precondition for keeping time) and in the Chapter 1 setup fact that staying accurate depends on NTP; it does not overreach into claiming a measured zeroed value.

**Pass criteria**:
- Steps 1 and 2 done right look like: `/sys/class/rtc/rtc0/name` contains `rtca3` and `hctosys` is `1`; `timedatectl`'s `RTC time` is close to its `Universal time` (under 0.5 s apart on this board), and `RTC in local TZ: no`.
- To check whether NTP really has pulled the system into line, look at whether `timedatectl timesync-status` reports an `Offset` down at the millisecond level; with sync sustained, `System clock synchronized:` turns to `yes` (this varies by session — the Chapter 1 environment transcript `live/ch01-env.txt`, for instance, shows `yes`).

### When You'd Actually Use This

**Mechanism**: RTC relies on its own independent low-frequency crystal oscillator and standby power to keep running even through a full system power-off/reboot, or through suspend-to-RAM/disk (provided there's a battery or standby supply keeping the `RTXIN` oscillator and the RTC domain powered). This is fundamentally different from SYC/`arch_timer` (which zeroes at boot and runs off the system clock): one is "must remember what time it is even through a power loss," the other is "measure elapsed time since boot."

**Decision rule**:

- When you need to "know the actual current date and time as soon as you boot" (**take system logs needing correct timestamps, or certificates/TLS needing to validate an expiry date, as examples**), use the RTC to read the initial time (`hwclock -r`, or `hwclock -s` in a boot script). After boot, the passage of Linux system time is still handled by the kernel's monotonic clock (which depends on SYC/`arch_timer`); the RTC is only read/written once at the boot and suspend/resume boundaries — it's not a component continuously participating in system timekeeping.
- When you need to be "able to wake up on schedule even with no network connection" (**take low-power periodic sensing or scheduled tasks as examples**), the RTC's Alarm interrupt paired with `rtcwake` is a hardware-level scheduled-wake mechanism that still works while the CPU is fully asleep. That is its fundamental difference from other timers like GTM/OSTM/CMTW, which typically lose power along with their core domain during deep suspend.

**Take a data logger as an example** (needing correct timestamps), or **a sensor node needing scheduled wake-ups to save power as another**: whenever "retaining time information through a power loss" or "waking up on schedule while the CPU sleeps" is involved, you'll be using the RTC rather than any of the other timers. This decision line holds for any application domain — the question is "do I need time that survives power loss/survives sleep," not "which project am I working on."

> Source: the official hardware manual r01uh1032 **§5.3 Realtime Clock (RTC)** (p1166–1168 overview; registers from §5.3.2, p1169). Development notes doc07 §23. The board also has a PMIC with a built-in RTC (a different chip) — see the WS125 RDK carrier board manual's BOM (U16, `RAA215300A2GNP`).

---

## This Group's Quick Reference

| What you want to do | Which one to use | How you get at it on the board |
|---|---|---|
| Measure "how much time has elapsed," `clock_gettime`/`nanosleep` | SYC (indirectly) | Standard POSIX time API; no direct interface |
| Millisecond-scale timed interrupts/events (can tolerate scheduling delay) | GTM/OSTM, CMTW (indirectly) | `timerfd`/`nanosleep`; kernel picks the clockevent itself |
| Measure external pulse intervals (input capture) | CMTW0–3 (exists in silicon, unreachable from Linux) | Currently no driver; need to write a kernel driver yourself or use a realtime core |
| Output PWM (motor/servo/LED) | GPT (+ POEG as a safety gate) | No pwmchip; bare-metal MMIO or hand it to R8/M33 |
| Cut power-circuit output at the hardware level on a fault | POEG | No driver; follows GPT down the bare-metal/realtime-core path |
| Auto-reboot when the system hangs | WDT (pick WDT0–3 by core) | CA55 = `/dev/watchdog0`; R8/M33 refresh their own in their own firmware |
| Keep time through power loss, scheduled wake while the CPU sleeps | RTC | `/dev/rtc0`, `hwclock`, `rtcwake` |

> **The whole group in one sentence**: the first three (SYC/GTM/CMTW) are the system time base — you can only use them indirectly through the standard API. The last four are the ones the application actually wants to touch directly, but the most sought-after of them, PWM (GPT + POEG), has no ready-made path under this board's Linux — it's either bare-metal MMIO, or hand it to a realtime core. The only two that have a clean standard interface under Linux, ready to use as-is, are WDT (`/dev/watchdog0`) and RTC (`/dev/rtc0`).

## This Group's Boundaries and Gaps (Honestly Noted)

- **SYC is this group's one inventory gap**: it has no `/dev`/sysfs node; this board's "enabled" verdict is **inferred** from `arch_timer`'s interrupt being active, not directly measured. Figures like 24 MHz and 64-bit are always official Manual specs, never write them up as "measured on the board." If a future hands-on verification section is written for SYC, it needs board testing first (e.g. `cat /proc/interrupts | grep arch_timer`).
- **"64 PWM outputs" is a spec-derived figure, not something measured on this board**: that's the theoretical silicon ceiling of 4 pins × 16 channels; this board in practice only has the 8 channels of GPT0 ch0–7 with `okay` nodes, and no PWM chip. When writing about or citing this, always distinguish "theoretical silicon ceiling" from "what's actually usable on this board's Linux."
- **CMTW/GPT/POEG's pin capabilities are unreachable from this board's Linux application layer**: input capture, output compare, and PWM output all exist in silicon, but none of them have an existing kernel driver exposing them to the application layer. To use them you'd have to write your own driver, or go through the realtime core (Cortex-R8/M33) firmware — and the latter isn't exposed to Linux on this board, requiring the remoteproc/firmware-loading path first (which belongs to a different chapter's scope).
- **The 40-pin header's pin-to-GPT mapping is settled in principle, but not yet down to the wire**: the `PWM0`/`PWM1` silkscreen labels really do land on GPT (pin 32 = `PA4` → `GTIOC6A`, pin 33 = `PA7` → `GTIOC7B`; see §4's note box for the full cross-check and the header-wide counts). What is still open is the electrical side: the RDK schematic is not in hand, so series resistors, level shifting or an existing on-board load may rule out particular pins, and for 7 header pins the "pin number ↔ port pin" mapping is not yet pinned down. The counts hold; the wiring plan doesn't, until those are closed.
- **Register-level, bit-by-bit detail is out of scope for this document**: this group only covers the official Manual's "functional overview" level (Overview/Features/Block Diagram). The bit-by-bit register descriptions for each unit (`OSTMnCTL`, `GTIOR`, `GTCCRA-F`, `CMWIOR`, the various RTC registers…) and the actual timing in the Operation chapters — if you need to write register-level bare-metal control instructions, you'll need to pull the corresponding section of the Manual separately.
