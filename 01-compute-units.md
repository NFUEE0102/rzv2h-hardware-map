# 01 · Compute Units
> This file is the compute-units deep-dive file in the "04 · Whole-Board Hardware Resource Map" folder (corresponding to section number **4.2**). The chapter opener, learning objectives, the `<board IP>` placeholder, and the marking conventions, along with the 4.1 whole-board overview, are all in [00-overview-and-ip-enablement-map.md](00-overview-and-ip-enablement-map.md); where the text says "4.1" it refers to the 00 file, and "4.4" refers to the 99 file. Steps marked ✅ for on-board verification all come with a transcript filename, pointing to the on-board recordings under `live/` in the handbook folder (written as `../live/ch04*.txt` starting from this folder).

## Files in This Folder

| File | Contents |
|---|---|
| [00-overview-and-ip-enablement-map.md](00-overview-and-ip-enablement-map.md) | Chapter opener + 4.1 whole-board overview and enablement map + end-of-chapter recap / quick-reference / measured-data tables |
| **01-compute-units.md** (this file) | A55 / R8 / M33 / GPU / DRP-AI3 deep dive: specs, benchmark measurements, capability-ceiling derivation; DRP1's position is in the 00 file's grouped summary table, with the deep dive in Chapter 3 |
| [02-video-capture-codec-display.md](02-video-capture-codec-display.md) ~ [08-debug-and-security.md](08-debug-and-security.md) | Peripherals and interfaces, unit by unit: imaging (02), audio (03), memory and storage (04), system backbone (05), timing (06), communication and sensing (07), debug and security (08) |
| [09-official-documentation-guide.md](09-official-documentation-guide.md) | Which official document answers which question, and the `_toc_full.txt` quick-lookup trick |

---

## In This File

- [4.2 Compute Units Deep Dive](#42-compute-units-deep-dive)
  - [Why "a Bunch of Different Cores" Instead of One Fast CPU](#why-a-bunch-of-different-cores-instead-of-one-fast-cpu)
  - [Four Cores, Four Roles (Purpose and Specs of Each Core)](#four-cores-four-roles-purpose-and-specs-of-each-core)
  - [Hands-On: Confirm These Cores Are All Present and in the Right State](#hands-on-confirm-these-cores-are-all-present-and-in-the-right-state-see-markers-below-for-each-step)
  - [Benchmark Measurements (Every Entry Comes With Its Measurement Conditions)](#benchmark-measurements-every-entry-comes-with-its-measurement-conditions)
  - [Capability Ceiling Derivation: What Hz Can a Given Algorithm Hit](#capability-ceiling-derivation-what-hz-can-a-given-algorithm-hit)
  - [The Three Compute Engines, Each in One Sentence](#the-three-compute-engines-each-in-one-sentence)
  - [Hands-On Verification: Confirm Your Board Matches This Section's Data](#hands-on-verification-confirm-your-board-matches-this-sections-data-see-markers-below-for-each-step)

---

## 4.2 Compute Units Deep Dive

The previous section (4.1, see [00-overview-and-ip-enablement-map.md](00-overview-and-ip-enablement-map.md)) already took roll call across the SoC's hardware blocks — which are enabled, which are disabled, how memory is partitioned. This section does something different: it zooms in on "the cores that actually do the arithmetic" — A55, R8, M33, GPU, plus the compute side of the DRP-AI3 NPU — and looks at each one, one at a time, to see clearly **what it is, what it's good at, and how fast it can go**. When you're doing system planning, this is the section you'll flip back to most often: the answers to "where do I put the real-time control loop (flight control as an example), where do I put perception inference, where do I put image pre-processing" all live here.

(Every board address that appears in this section is written as `<board IP>`; see "What You'll Need" at the start of the 00 file for why.)

### Why "a Bunch of Different Cores" Instead of One Fast CPU

The typical desktop mindset is "one CPU, as fast as possible." Embedded SoCs take a different road — **heterogeneous computing**: the chip packs several compute units with different specialties, and each one only does what it's best at. The RZ/V2H is built this way. Understanding this is the same as understanding "why software for this class of real-time application needs to be split apart and placed on different cores."

This SoC has five kinds of units that do arithmetic (Source: 06-hardware-resource-map.md:28-40):

```text
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  RZ/V2H (R9A09G057H44) Heterogeneous Compute Units                                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                     │
│  ┌──────────────────────┐                                                                                           │
│  │  Cortex-A55 × 4      │   General compute / Linux apps / estimation / planning                                    │
│  │  @1.7 GHz            │   Runs Ubuntu, ROS, EKF, MPC, comms protocols                                             │
│  └──────────────────────┘   ← Your code's "home turf"                                                               │
│                                                                                                                     │
│  ┌──────────────────────┐                                                                                           │
│  │  Cortex-R8 × 2       │   Hard real-time / motor control / safety co-processor                                    │
│  │  @800 MHz            │   Ideal spot for the real-time inner loop                                                 │
│  └──────────────────────┘   ← Not yet exposed to Linux; firmware not loaded                                         │
│                                                                                                                     │
│  ┌──────────────────────┐                                                                                           │
│  │  Cortex-M33          │   System management / low-power always-on supervisor                                      │
│  │  @200 MHz            │   remoteproc0, offline by default                                                         │
│  └──────────────────────┘   ← Firmware not bundled                                                                  │
│                                                                                                                     │
│  ┌──────────────────────┐                                                                                           │
│  │  DRP-AI3 NPU         │   INT8 CNN inference engine (vision AI)                                                   │
│  │  /dev/drpai0         │   8 dense / 80 sparse TOPS                                                                │
│  └──────────────────────┘   ← The decisive edge for perception                                                      │
│                                                                                                                     │
│  ┌──────────────────────┐                                                                                           │
│  │  Mali-G31 GPU        │   Per-pixel parallel image/CV, zero-copy, HMI                                             │
│  │  @630 MHz /dev/mali0 │   OpenCL 3.0, 1 shader core                                                               │
│  └──────────────────────┘   ← Not the AI workhorse, not a general-purpose FP accelerator                            │
│                                                                                                                     │
│  (DRP1, a reconfigurable processor for non-AI CV acceleration, is also on board — see the 4.1 grouped summary table)│
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

This section will go on to dive into the compute side of the first four categories one at a time (A55, R8, M33, GPU; DRP-AI3 is the star of the perception chapter, so here we'll only cover "its throughput and capability boundaries as a compute engine," leaving the details to the DRP-AI chapter). By the end you'll have three things: **the purpose and specs of each core**, **benchmark measurements with their conditions attached to every entry**, and **a capability-ceiling derivation that translates measurements into "what Hz can a given algorithm hit."**

> 💡 **Tip**: As you read this section, keep one throughline in mind — "the A55 has general-purpose compute to spare; the real constraint isn't FLOPS (floating-point operations per second), it's (1) real-time jitter, (2) Python dispatch overhead, and (3) no hardware crypto." (Source: 09-compute-capability.md:17, 05-compute-benchmark.md:27) Every number that follows is there to help you quantify exactly where these three constraints bite.

### Four Cores, Four Roles (Purpose and Specs of Each Core)

#### Cortex-A55 × 4: Home Turf for Your Code

**What it is.** Four Arm Cortex-A55 cores (stepping r2p0), a single cluster, running at 1.7 GHz. This is the only group of cores that runs Linux directly — Ubuntu, your C/C++/Python programs, ROS, estimators, planners, communication protocols all live on these four cores by default. They are themselves the host for the Linux kernel, so they **don't** have a corresponding device node; you won't find something like `/dev/a55` — for core topology, check `/sys/devices/system/cpu/cpu0` through `cpu3` instead (Source: 07-hardware-unit-usage-guide.md:22).

**Key specs (measured on the board):**

| Item | Value | Source |
|---|---|---|
| Core count | 4 (single cluster, one thread per core) | 00_inventory.txt:29-42 |
| Clock | 1.7 GHz (selectable 212.5 / 425 / 850 / 1700 MHz) | 05-compute-benchmark.md:45, 00_inventory.txt:68-72 |
| BogoMIPS | 48.00 | 00_inventory.txt:75-80 |
| CPU part / variant / rev | 0xd05 / 0x2 / 0 | 00_inventory.txt:75-80 |
| Instruction set extensions | `fp asimd(NEON) fphp asimdhp(fp16) asimddp(INT8 dot) crc32 atomics lrcpc dcpop` | 05-compute-benchmark.md:46, 00_inventory.txt:43 |
| Crypto extensions | **No `aes`/`sha2`** (this point keeps coming back later — it matters) | 05-compute-benchmark.md:46 |

**Why "has dotprod and fp16, but no aes/sha" matters.** `asimddp` (INT8 dot product) lets the A55 do INT8 dot products relatively efficiently, which helps the CPU fallback path for quantized neural networks. (Quantization = compressing weights/activations from fp32 down to low-bit integers such as INT8, trading precision for a smaller model and faster NPU/integer math.) `fphp`/`asimdhp` (fp16) gives half-precision floating-point hardware support. But the instruction set **does not have** ARMv8 Crypto Extensions — meaning AES and SHA can only run as software implementations, with throughput pinned to the tens-of-MB/s range (the benchmark section later gives you exact numbers). This isn't a defect — it's a characteristic of this H44 part number as measured on the board: `aes`/`sha2` are simply absent from the CPU flags (see the table above; measured in §4.2 of this handbook). The datasheet separately notes that the crypto extension is "security variant products only," which is usable as a clue but is provisional, coming from a degraded-format source (Source: 07-hardware-unit-usage-guide.md:19, datasheet.md:180).

**An honest heads-up about cache naming (sources don't agree on the terminology).** Run `lscpu` and you'll see "L2 cache: 1 MiB (1 instance)" (Source: 00_inventory.txt:44); but the datasheet's phrasing is "L1 32KB I + 32KB D/core, **L2 = 0KB, L3 = 1MB** (ECC, up to 1.26 GHz)" (ECC = error-correcting code; Source: datasheet.md:150-157). Both sides are talking about the same 1 MB shared cache — they just **disagree on which level to call it**: the OS reports it as L2, the datasheet counts it as L3. All you need to know is "there's a 1 MB shared last-level cache here"; don't get hung up on whether it's called L2 or L3.

> ⚠️ **Warning**: The A55's **boot frequency is determined by the two `BOOTPLLCA[1:0]` signals on the on-board DIP switch `DSW1`**, and the board manual's table is the sole authority (ON = High, OFF = Low): Low:Low = 1.1 GHz, Low:High = 1.5 GHz@0.9V, High:Low = 1.6 GHz@0.9V, **High:High = 1.7 GHz@0.9V (factory default)** — all four settings are defined, and **there is no 1.8 GHz DIP setting** (Source: WS125 board manual §3.3 DSW1 table, Page 6; the complete bit-by-bit table is in Chapter 1). The datasheet's "max 1.8 GHz@0.9V" is the **silicon's rated ceiling**, not a boot setting selectable on DSW1 — the two are numbers at different levels, so don't conflate them. **Prevention**: don't copy "the exact frequency of a given setting" into your program from secondhand notes; to find out which frequency is actually in use, read `scaling_cur_freq` after boot using the command below. Every CPU measurement in this handbook was taken at the **default 1.7 GHz, governor locked to performance** — that part is unambiguous.

#### Cortex-R8 × 2: The Ideal Spot for Hard Real-Time Control (Flight Control as an Example; but Currently Invisible to Linux)

**What it is.** Two Arm Cortex-R8 cores (dual-core MPCore), 800 MHz, Armv7-R architecture, with NEON+FPU, 32KB of I/D-cache each (ECC), plus 128KB each of tightly-coupled memory (I-TCM / D-TCM, ECC), and MMU support (Source: 07-hardware-unit-usage-guide.md:33, datasheet.md:158-164). The R-series is Arm's "real-time processor" line — its value isn't in computing fast, it's in **computing on time**: predictable interrupt latency, and TCM that keeps critical code immune to cache misses. The datasheet lists its intended uses as hard real-time, motor control, and safety co-processor.

> ⚠️ **Warning**: This R8 **does not have dual-link lock-step** — the hardware manual `r01uh1032` §1.1.3 Functions, Table 1.1-2 CPU, Cortex-R8 (CR8) row states verbatim "No support for dual-link lock-step technology" (p78; the degraded datasheet `r01ds0429` L164 carries the same line and can serve as a corroborating clue). Lock-step is a safety mechanism that runs the same program on two cores and compares them cycle by cycle to catch hardware faults; without it, you cannot treat this pair of R8 cores as cores that have "passed functional-safety redundancy." When doing safety-critical design, this is a boundary you need to know up front.

**Current on-board status: present, but not exposed by the running Linux.** This is the point people misunderstand most easily, and it's exactly why the 4.1 grouped summary table marks R8 as 🟡. R8 is PRESENT on the silicon, and the remoteproc framework has already reserved the communication ring (vring) addresses it needs (0x42f00000–0x43500000) — but you won't find an R8 node under `/sys/class/remoteproc` in the **running Linux** (Source: 07-hardware-unit-usage-guide.md:35; ✅ Verified on the board; transcript: live/ch04b-runtime.txt: `ls /sys/class/remoteproc/` shows only `remoteproc0` (i.e., M33), and `/sys/class/uio` is empty too — until firmware is loaded, neither R8's remoteproc node nor its UIO device will appear). The reason: in the stock image, R8 is started by U-Boot during the boot stage, not by Linux once it's up and running. To bring it up and let the A55 talk to it over RPMsg (remote processor messaging — Linux's cross-core message channel), you need to load R8 firmware from the Renesas Multi-OS Package. Only once that firmware is loaded does the RPMsg path appear, via the UIO (userspace I/O — a lightweight driver framework that maps hardware registers directly into userspace) devices `42f00000.rsctbl` / `43000000.vring-ctl0` / `43200000.vring-shm0`, with `rpmsg_sample_client` as the example program on the CA55 side (Source: 07-hardware-unit-usage-guide.md:36).

**Why the hard real-time inner loop (flight control as an example) belongs here.** The "real-time" benchmark later will prove it: on Linux, the A55 (a PREEMPT, non-real-time kernel) has worst-case scheduling jitter that lands in the sub-millisecond range, which is too tight for hard-deadline loops at 300–500 Hz or above. R8 doesn't carry that layer of Linux scheduling uncertainty, which makes it exactly where loops like the attitude/RPM inner loop or motor commutation — the kind where "miss one beat and things go wrong" — belong (Source: 06-hardware-resource-map.md:122, 09-compute-capability.md:35). Reference figures for the R8 itself — CoreMark, a cascaded-PID control tick, interrupt entry latency and jitter, UART behavior, each compared against an STM32H745 Cortex-M7 under identical workloads — are in "Measured: Cortex-R8 vs STM32H745" immediately below (Source: 05-compute-benchmark.md §13).

#### Measured: Cortex-R8 vs STM32H745 (real-time comparison)

Where the A55 figures earlier in this section tell you what the Linux side can do, this subsection gives the reference figures for the core the hard real-time control path actually sits on: the **Cortex-R8** (800 MHz class), compared against the Cortex-M7 of an **STM32H745** (NUCLEO-H745ZI-Q, 400 MHz class) under identical workloads. Use it when deciding whether an inner loop belongs on this SoC's R8 or on a separate real-time MCU (Source: 05-compute-benchmark.md §13).

**Setup the reference values assume.** Both sides run the same C source, built with the same GCC 13.3.1 at `-O2`, each timed with its own core cycle counter (M7: DWT, R8: PMU) and converted with the measured clock. The R8 side is a FreeRTOS task on core1 (FSP 3.1.0) at 792.1 MHz (PMU calibrated against the GTM tick), with a 1 kHz tick and rpmsg live in the background and the A55 Linux side idle. The H745 side is bare metal with no interrupt interference, at 403.8 MHz (PLL set for 400 MHz, HSE from the ST-LINK MCO, about 0.9% fast), on its factory direct-SMPS supply at VOS1. In the UART figures the H745's RX path is real hardware driven from a PC, while on the R8 the RSCI5 pins are unwired on this board, so its RX path is **simulated** (details in the UART table below).

**Bottom line: the R8 computes faster, the H745 reacts faster.**

| Aspect | R8 vs. H745 | Better |
|---|---|---|
| General compute (CoreMark) | 2,676 vs. 1,615 iter/s | R8 ×1.66 (per MHz it is the M7 that wins: 3.38 vs. 4.00) |
| Conventional cascaded-PID flight-control tick (float) | 2.44 vs. 5.06 µs | R8 ×2.07 (nearly identical cycle counts; the gap is clock) |
| Double-precision floating point (dgemm) | 244.8 vs. 51.1 MFLOPS | R8 ×4.79 |
| Main-memory copy (memcpy 128 KB) | 117 vs. 518–654 MB/s | H745 ×4.4–5.6 |
| Tick interrupt entry latency | 497 vs. 74 ns | H745 ×6.7 |
| Tick period jitter | ±440 vs. ±30 ns | H745 ×14.9 |

Under 921600 baud full-duplex UART plus a 1 kHz control loop, both sides hold **zero overruns and zero bad frames** at under 10% load — either one is up to the job for a conventional flight controller.

**The two platforms side by side.**

| Item | RZ/V2H · Cortex-R8 core1 | STM32H745ZI · Cortex-M7 |
|---|---|---|
| Clock | 792.1 MHz (PMU calibrated against the GTM tick) | 403.8 MHz (PLL set for 400 MHz, HSE taken from the ST-LINK MCO, about 0.9% fast) |
| Power / voltage | — | Direct SMPS, VOS1 (as the board ships) |
| Floating-point unit | VFPv3-D16 (single + double precision) | FPv5-D16 (single + double precision) |
| Code location | System SRAM / DDR + I-cache | Flash (2 WS) + 16 KB I-cache |
| Data | SRAM / DDR, D-cache on | DTCM (stack/statics), AXI SRAM (heap), D-cache on |
| Execution environment | A FreeRTOS task on core1 (FSP 3.1.0), with a 1 kHz tick and rpmsg in the background | Bare metal, no interrupt interference |

**Compute and memory.**

| Parameter | Conditions | R8 | H745 | Unit | Better |
|---|---|---:|---:|---|---|
| CoreMark | 30,000 iterations, CRC 0x5275 passed | 2,676 | 1,615 | iter/s | R8 ×1.66 |
| CoreMark / MHz | Per-cycle efficiency | 3.38 | 4.00 | /MHz | H745 ×1.18 |
| dgemm 16×16 | Double precision | 244.8 | 51.1 | MFLOPS | R8 ×4.79 |
| sgemm 16×16 | Single precision | 288.8 | 107.9 | MFLOPS | R8 ×2.68 |
| sin() | Double precision, newlib | 151 | 409 | ns/call | R8 ×2.71 |
| sinf() | Single precision | 96 | 279 | ns/call | R8 ×2.89 |
| sqrt() | Double precision | 65 | 138 | ns/call | R8 ×2.11 |
| memcpy 8 KB | L1-resident | 1,495 | 1,504 | MB/s | Tie |
| memcpy 128 KB | Main memory (R8: DDR, H745: AXI SRAM) | 117 | 518–654 | MB/s | H745 ×4.4–5.6 |
| memset 128 KB | Main memory, write only | 1,089 | 946 | MB/s | R8 ×1.15 |

Two things to keep in mind when quoting this table. The H745's memcpy 128 KB figure depends on where the buffer lands in AXI SRAM and how it is aligned — 654 MB/s for one placement, 518 MB/s for another — hence the range. And the R8's CoreMark (2,676) and the A55 single-core CoreMark given earlier in this section belong to **different cores**; they cannot be placed side by side directly.

**Conventional cascaded-PID flight control.** The workload is a full PX4-style chain, all float, PX4 default gains, with every loop run on every tick — a worst case, since real PX4 runs the position/attitude loops slower than the rate loop:

> gyro/accelerometer biquad low-pass + notch → Mahony quaternion attitude → position P → velocity PID (anti-windup) → thrust vector + tilt limit → attitude setpoint quaternion built from the thrust direction and heading → quaternion attitude P → rate PID (D on measurement + filtering, integral clamp) → Quad-X mixer (airmode desaturation, yaw margin, thrust linearization)

| Parameter | R8 | H745 | Unit | Better |
|---|---:|---:|---|---|
| Tick time Min | 2.26 | 4.73 | µs | R8 ×2.10 |
| Tick time Typ (mean) | 2.44 | 5.06 | µs | R8 ×2.07 |
| Tick time Max | 7.96 | 6.38 | µs | H745 ×1.25 |
| Cycles per tick | 1,933 | 2,040 | cycles | Tie |
| CPU load at 1 kHz | 0.24 | 0.51 | % | — |

Figures are sampled once per 100,000 ticks. The R8's higher maximum comes from FreeRTOS interrupts cutting in; the H745 side is bare metal with no interrupts, so treat the maximum column as favoring the H745 by construction.

**UART + interrupt latency.**

**Conditions.** A timer interrupt at the highest priority drives the 1 kHz loop and runs the PID chain above inside the interrupt; every tick sends one 50 B MAVLink v2-format telemetry packet at 921600 baud (TX by interrupt and, separately, by DMA). RX streams in continuously at the 921600 byte rate — one interrupt per byte, each running the same CRC deframer. The UART/DMA interrupts sit at a lower priority and can be pre-empted by the tick; their timings have the pre-empted time subtracted out. The figures below are for the TX-interrupt + RX-saturated condition unless the row says otherwise.

| Item | R8 | H745 |
|---|---|---|
| Tick timer | GTM0 (the one GTM Linux has not claimed), GIC priority 0, not masked by FreeRTOS critical sections | TIM5, NVIC priority 0 |
| UART | RSCI5 (P72/P73); **these pins are unwired on this board, so RX is simulated**: a GPT6 overflow interrupt routed through INTR8SEL fires at 92.16 kHz, reads RDR once and runs the same deframer; TX really transmits, with nothing on the other end | USART3 → ST-LINK virtual COM port; **RX is driven for real from a PC, and the received telemetry is verified** |
| TX DMA | DMAC_B unit 0 (FSP `R_DMAC_B_Reconfigure`) + D-cache clean | DMA1 + DMAMUX (registers written directly) + D-cache clean |
| ISR style | Register level, hooked into the FSP vector table at run time; the tick ISR saves the VFP registers itself | Register level |

| Parameter | Conditions | R8 Min / Typ / Max | H745 Min / Typ / Max | Unit | Better |
|---|---|---:|---:|---|---|
| Tick interrupt entry latency | Includes one counter read (R8 99 ns, H745 40 ns) | 390 / 497 / 870 | 64 / 74 / 104 | ns | H745 ×6.7 |
| Tick period deviation | Relative to the mean | −444 / 0 / +435 | −30 / 0 / +29 | ns | H745 ×14.9 |
| PID controller | Run inside the interrupt | 2.27 / 2.41 / 2.93 | 4.93 / 5.17 / 5.41 | µs | R8 ×2.1 |
| Telemetry pack + start TX | TX interrupt mode | 2.25 / 2.27 / 3.10 | 1.75 / 1.76 / 1.99 | µs | H745 ×1.3 |
| Telemetry pack + start TX | DMA mode | 6.34 / 6.95 / 7.54 | 1.95 / 1.95 / 1.99 | µs | H745 ×3.6 |
| Whole tick interrupt | TX interrupt mode | 5.18 / 5.35 / 6.13 | 7.44 / 7.69 / 7.97 | µs | R8 ×1.4 |
| Whole tick interrupt | DMA mode | 9.85 / 10.09 / 10.69 | 7.93 / 8.01 / 8.16 | µs | H745 ×1.3 |
| UART interrupt body | TX interrupt mode, about 140,000 per second | — / 207 / 981 | — / 214 / 550 | ns | Tie on average |
| DMA completion interrupt body | Once per packet | — / 54 / 236 | — / 124 / 208 | ns | R8 ×2.3 |
| Interrupt-body CPU load | TX interrupt / DMA mode | 3.48 / 1.86 | 3.75 / 2.83 | % | See the reading notes below |

**Link integrity under these conditions**: on the R8, 55,292 RX packets (simulated path) with 0 errors and 30,000 telemetry packets out with 0 skipped; on the H745, 54,002 RX packets with 0 errors and ORE 0, and 30,000 telemetry packets arriving at the PC with 0 errors and no gaps in the sequence numbers.

**How to read these numbers:**

1. **Interrupt response is about 7× faster and about 15× steadier on the H745.** The M7's NVIC stacks in hardware in roughly 35 ns (about 14 cycles); an R8 interrupt has to go through the GIC, the FreeRTOS `IRQ_Handler` and the FSP dispatch table, which comes to about 400 ns once the counter read is subtracted. That is the structural reason behind the entry-latency and jitter rows, and it does not go away by tuning the application.
2. **What each interrupt actually does costs about the same on both sides** (the UART interrupt body is ~0.2 µs either way) — but the CPU-load row counts only the interrupt body. The R8 pays an extra ~0.4 µs or more of entry/exit overhead per interrupt, which at 140,000 interrupts per second works out to an **estimated ~6% more**; on the H745 it is about 0.8%. That figure is an estimate derived from the entry latency — exit overhead is not included — so budget interrupt-heavy designs on the R8 with margin.
3. **Starting a DMA is slower on the R8** (6.95 vs. 1.95 µs), mainly because going through the FSP driver reconfigures the channel for every packet, plus the TE/TIST handshake; writing the registers directly saves roughly 5 µs.
4. Under 921600 full duplex plus a 1 kHz control loop, both sides show **zero overruns and zero bad frames** — both meet the UART needs of a conventional flight controller (RC input, telemetry, GPS).

> ⚠️ **Warning**: **Scenario** — you drive UART TX from DMA on the R8 through the FSP driver, expecting DMA to cost less CPU time than interrupt-driven TX. **Symptom** — "pack telemetry + start TX" balloons from 2.27 µs to 6.95 µs, and the whole tick interrupt goes from 5.35 µs to 10.09 µs — worse than the H745 in the same mode (1.95 µs / 8.01 µs). **Cause** — `R_DMAC_B_Reconfigure` reconfigures the channel on every packet, and the TE/TIST handshake is paid every time. **Prevention/Handling** — set the channel up once and write the registers directly for each packet; that buys back roughly 5 µs per tick.

**What this means when you choose a core:**

- **Compute**: on the same float control code the R8's per-cycle efficiency is on par with the M7, and roughly 2× the clock buys roughly 2× the speed; for double precision the gap widens to 2.7–4.8×.
- **Real-time behavior**: the H745's interrupt latency and period jitter are both an order of magnitude smaller, and its memory subsystem (on-chip SRAM) moves data that spills out of L1 4–6× faster; the R8's main memory is the DDR it shares with the A55.
- **Choosing between them**: a conventional cascaded-PID flight controller is only a 0.2–0.5% load on either part, so compute is not the bottleneck. If what you care about is timing determinism and interrupt response, the H745 has the advantage; if you need double precision or heavier arithmetic, the R8 is clearly stronger.

**Boundaries on these reference values (read before quoting them):**

- The **R8's UART RX figures come from the simulated path** — the pins are unwired on this board — and its TX side has no receiver for verification. The H745's UART figures are end-to-end against a real PC.
- The H745 on its factory SMPS supply is limited to VOS1 (400 MHz); LDO + VOS0 would reach 480 MHz (about +19%), which requires board rework, so its figures here are 400 MHz-class figures.
- The R8 runs under FreeRTOS with the background tick and rpmsg interrupts live, the H745 bare metal with no interrupts — a slight disadvantage for the R8 in the **maximum** columns, negligible for the averages.
- The R8 values hold with the A55 Linux side idle; under a heavily loaded A55, DDR contention is an additional factor these figures do not cover.
- CoreMark here is a self-made port rather than an EEMBC-certified submission, though the parameters and the CRC both satisfy the performance-run rules.

#### Cortex-M33: The Low-Power Always-On System Manager

**What it is.** A single Arm Cortex-M33, 200 MHz, Armv8-M architecture, with FPU, DSP extensions, TrustZone-M security extensions, plus a 60KB CoreSight ETF program-flow trace buffer and JTAG/SWD (Source: 07-hardware-unit-usage-guide.md:47, datasheet.md:165-172). Its role is "low-power always-on supervisor" — keeping lightweight, long-running work like system management, power sequencing, and wake decisions alive even while the main cores sleep.

**Current on-board status: exposed via remoteproc, but offline by default with firmware missing.** Unlike R8, M33 **is visible** in Linux: `/sys/class/remoteproc/remoteproc0` is it, named `cm33`, default state `offline`, with the firmware field showing `rproc-cm33-fw` (✅ Verified on the board: all three values match verbatim; transcripts: live/ch04-cpu-periph.txt [name/state], live/ch04-followup.txt [firmware]) (the actual filename is `rzv2h_cm33_rpmsg_linux-rtos_example.elf`). The reserved memory it uses is rsctbl@0x42f00000, vring@0x43000000, vdev0buffer@0x43200000 (Source: 07-hardware-unit-usage-guide.md:49-50).

> ⚠️ **Warning**: **Scenario** — you follow the docs and try to start CM33 (remoteproc0) by running `echo start > /sys/class/remoteproc/remoteproc0/state`. **Symptom** — it fails to start, because the firmware file simply doesn't exist. **Cause** — the stock image doesn't ship with this `.elf` firmware built in. **Prevention/Handling** — you need to first download the Renesas Multi-OS Package (document number `R01QS0077`), place the firmware under `/lib/firmware/`, and modify the device tree (Source: 04-hardware-quickref.md:171,173). Until you've done this, remoteproc0 sitting at `offline` is the **expected state**, not something broken.

**The boundaries and version pinning Renesas spells out for when you go further (RDK official documentation v1.1.1, Chapter 3 "RZ/V Multi-OS").** The warning above only says "you need firmware and a device tree change"; Renesas separately pins down a few prerequisites that will otherwise block you, so check them off before you start:

| Item | Official value | Why it matters |
|---|---|---|
| Multi-OS Package | **v3.2** | The source of the firmware and the examples; the `R01QS0077` in the warning above is a document number |
| RZ/V **FSP** | **v3.1** | Used to build the example firmware; **do not install the RA family's FSP** (different product line, different version numbering scheme) |
| Segger **J-Link firmware** | **7.96e** (Renesas names this exact version) | Version compatibility in this kind of toolchain is usually pinned very tightly; installing the wrong version fails at the connection stage |
| JTAG debugging | Requires **DSW1's SW6 set to ON** | Corresponds to `MD_BOOT3` = Debug in the DIP table in Chapter 1 §1.2; it should be OFF during normal Linux operation |

**How far the default IPL goes (this is the easiest one to misjudge).** Renesas's own words: "On our default IPL, the following features are enabled by default: **Remoteproc support and CM33 and CR8 invocation from U-Boot.**" — that is, **remoteproc support, and invoking CM33/CR8 from U-Boot, are supported by default**. As for going further, Renesas separately writes: "If you want to use other features of multi-OS, such as **CM33 cold boot** or CA55 1.8 GHz support, feel free to contact us for support at renesas-rdk."

> ⚠️ **Warning (don't misread "CM33 cold boot requires contacting us" as "CM33 can't be used")**: that sentence is scoped to **cold boot** specifically (bringing CM33 up ahead of Linux) — it does **not** say CM33 is unusable across the board; the default IPL explicitly supports invoking it from U-Boot. And "starting CM33 via remoteproc from an already-running Linux" is a **third scenario** that the sentence doesn't cover at all, and one this handbook has not managed to verify on this board. **Keep the three paths separate when planning**: ① available by default = the remoteproc framework is there, invocation from U-Boot; ② requires contacting Renesas = cold boot; ③ unresolved = starting it from a running Linux. Note also that the "contact" in question is a support invitation on the `renesas-rdk` GitHub project, not an official Renesas support channel.

> 💡 **Tip (a Module Standby trap that's easy to get bitten by)**: Renesas points out that **peripherals that haven't been explicitly enabled go into Module Standby mode once Linux boots**. To keep Linux from switching off peripherals that CM33 needs, the Multi-OS Package patches `drivers/clk/renesas/r9a09g057-cpg.c` to change GTM's clock entries from `DEF_MOD` to **`DEF_MOD_CRITICAL`** (marked as a critical clock that may not be gated off). **This mechanism matters well beyond CM33 itself**: any time you plan to have a secondary core operate a peripheral independently, you have to ask whether the Linux side will gate its clock. Renesas also gives two ways to disable a peripheral on the Linux side: edit the source dts to change `status` from `"okay"` to `"disabled"`, or simply comment out the corresponding overlay line in `/boot/uEnv.txt` (for the overlay mechanism, see [00-overview-and-ip-enablement-map.md](00-overview-and-ip-enablement-map.md)).

#### Mali-G31 GPU: Parallel Offload and Zero-Copy, Not the AI Workhorse

**What it is.** An Arm Mali-G31 (Bifrost architecture, arch 7.0.9 r0p0), clocked at 630 MHz. On spec, it has **only 1 shader core / 1 OpenCL compute unit**, 8KB of L2 cache, and 32KB of local memory (type Global — the G31 has no dedicated local memory). It shares the same RAM as the rest of the system (unified memory, 14.84 GiB), so it can do **zero-copy** work together with the camera and encoder (Source: 08-gpu-deep-dive.md:23-28, 06_gpu.txt:9).

**What it can and can't do.** This is the most commonly misunderstood core on the board, so let's be clear about where it fits (Source: 08-gpu-deep-dive.md:12-14, 100-108):

- **It's a usable OpenCL 3.0 GPGPU compute device**, not just for display — kernels actually run on it, with support for fp16 (roughly 2× fp32 throughput), SPIR-V, and the most valuable one, `cl_arm_import_memory_dma_buf` zero-copy (Source: 08-gpu-deep-dive.md:37-41).
- **Good for**: per-pixel, massively parallel image/CV operations (color conversion, resize, warp, filtering, thresholding, optical flow, feature-point pre-processing); offloading image pre-processing while the A55 is busy with control/protocols; graphics/HMI/OSD overlay (GLES 3.2); running a handful of NN ops that DRP-AI3 doesn't support, in fp16, as a fallback.
- **Not good for**: acting as a general-purpose floating-point accelerator (you'll see later that its raw FLOPS are less than a single A55's); serving as the primary AI inference engine (that's DRP-AI3's job); large matrices/heavy computation (hard-limited by 1 CU + 32KB local mem).

> ⚠️ **Warning**: **Scenario** — you want to check whether Vulkan is usable. **Symptom/Current state** — libmali does indeed **contain Vulkan strings**, but the board has **no** `vulkaninfo`, and the ICD has never been confirmed. **Cause** — a string being present doesn't mean the capability is actually usable. **Prevention** — until you've actually verified it with `vulkaninfo` or by loading the Mali Vulkan ICD, **don't treat Vulkan as a confirmed capability on this board** (Source: 08-gpu-deep-dive.md:42). Likewise, the two sources disagree on the OpenCL version too: clinfo measures **3.0**, the datasheet lists **2.0** (Source: 08-gpu-deep-dive.md:37, datasheet.md:197-201) — go with whatever your board's `clinfo` actually prints.

#### DRP-AI3 NPU: Its Face as a Compute Engine

DRP-AI3 is the star of the perception chapter; here we'll only cover its specs and throughput "as a compute engine," so you have numbers to work with when deciding where compute should go.

**What it is.** The AI accelerator = DRP0 (96 processing elements, reconfigurable) + AI-MAC (4096 INT8 MACs [multiply-accumulate units], 3MB of local SRAM: 1MB weights + 2MB features, with support for sparse / N:M pruning). Peak **8 dense TOPS / 80 sparse TOPS** (TOPS = tera operations per second, 10¹² operations per second; this is the INT8 fixed-point peak, not floating-point FLOPS), power efficiency roughly **10 TOPS/W** (Source: 07-hardware-unit-usage-guide.md:61, 09-compute-capability.md:51, datasheet.md:195-196). Device node `/dev/drpai0` (drpai-rz driver 1.20 rel.3 V2H), paired with a **512MB DDR carveout @0x240000000** as its main memory (Source: 07-hardware-unit-usage-guide.md:63, 00_inventory.txt:135-139).

Why it's this board's "decisive edge for perception" — once you get to the inference benchmark later, you'll see it clearly in the numbers: it can squeeze YOLOX from 145 ms on the CPU down to 15 ms, while freeing up all four A55 cores completely to run other work — flight control and communication protocols, for example (Source: 05-compute-benchmark.md:147).

### Hands-On: Confirm These Cores Are All Present and in the Right State (see markers below for each step)

Before running any benchmark, spend two minutes confirming "these are indeed the cores I'm looking at, and the frequency is locked correctly." Every command in this section comes with expected output, transcribed verbatim from measured evidence.

**✅ Step 1 (verified on the board): Confirm the four A55 cores and their clock.**

```bash
lscpu | grep -E 'Model name|CPU max|CPU min|Core\(s\)'
```

Expected output (verbatim, ✅ Verified on the board; transcript: live/ch04-followup.txt; also see 00_inventory.txt:29-42):

```text
Model name:                           Cortex-A55
Core(s) per cluster:                  4
CPU max MHz:                          1700.0000
CPU min MHz:                          212.5000
```

**✅ Step 2 (verified on the board; transcript: live/ch04-cpu-periph.txt): Confirm the frequency is locked at 1.7 GHz and the governor is performance.** This step matters a lot — every measurement in this section was taken under this setting; if you reproduce it without locking performance, the numbers won't match because of dynamic frequency scaling.

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

Expected output (Source: 07-hardware-unit-usage-guide.md:178-179):

```text
1700000
performance
```

(`scaling_cur_freq` is in kHz; 1700000 kHz = 1.7 GHz.)

**✅ Step 3 (verified on the board; transcript: live/ch04-followup.txt): Confirm the instruction set extensions — dotprod/fp16 present, aes/sha absent.** This determines whether you can count on hardware crypto.

```bash
grep -o 'asimddp\|fphp\|aes\|sha2' /proc/cpuinfo | sort -u
```

Expected output (Source: 07-hardware-unit-usage-guide.md:27, 00_inventory.txt:43): only two lines, `asimddp` and `fphp`, will print — `aes` and `sha2` will **not** appear. If you want to see the full flags, look directly at the Features line in `/proc/cpuinfo`; as measured:

```text
fp asimd evtstrm crc32 atomics fphp asimdhp cpuid asimdrdm lrcpc dcpop asimddp
```

**✅ Step 4 (verified on the board; transcript: live/ch04-cpu-periph.txt): Confirm M33's remoteproc status (offline by default is normal).**

```bash
cat /sys/class/remoteproc/remoteproc0/name
cat /sys/class/remoteproc/remoteproc0/state
```

Expected output (Source: 07-hardware-unit-usage-guide.md:238):

```text
cm33
offline
```

Don't panic if you see `offline` — as mentioned before, this is the expected state when firmware hasn't been loaded.

**✅ Step 5 (verified on the board; transcript: live/ch04-inventory.txt): Confirm all the accelerator device nodes are present.**

```bash
ls -l /dev/drpai0 /dev/mali0 /dev/drp1 /dev/media0 /dev/video0 /dev/dri/card0
```

Expected: all six nodes present. Their measured major/minor device numbers are `/dev/dri/card0`(226,0), `/dev/drpai0`(511,0), `/dev/mali0`(10,125), `/dev/media0`(250,0), `/dev/video0`(81,3), `/dev/drp1`(234,1) (Source: 00_inventory.txt:119-127; ✅ Verified on the board — all six major/minor numbers match; transcript: live/ch04-inventory.txt). To further confirm driver versions, check the boot messages:

```bash
dmesg | grep -iE 'DRP-AI Driver|DRP Driver|Probed as mali'
```

Expected output (verbatim, Source: 00_inventory.txt:131,135,149):

```text
drp-rz 18000000.drp1: DRP Driver version : 1.00 rel.3 V2H
drpai-rz 17000000.drpai: DRP-AI Driver version : 1.20 rel.3 V2H
mali ... Probed as mali0
```

(After the system has been up a while, the dmesg ring buffer may have already pushed these lines out — see the warning box in 4.1; `journalctl -k -b` still recovers the line `mali 14850000.gpu: Probed as mali0` (✅ Verified on the board; transcript: live/ch04b-runtime.txt), though the DRP/DRP-AI version lines aren't guaranteed to still be there. Not finding them doesn't mean the driver isn't loaded — just check whether `/dev/drpai0`/`/dev/drp1` exist.)

### Benchmark Measurements (Every Entry Comes With Its Measurement Conditions)

> **Common measurement baseline (applies to all data below unless individually noted otherwise)**: on-board at `ubuntu@<board IP>`, kernel `6.10.14-arm64-renesas` (SMP **PREEMPT**, not PREEMPT_RT), Ubuntu 24.04.4 LTS aarch64, CPU governor locked to `performance` (1.7 GHz throughout, no frequency-scaling interference during measurement). Raw logs are in `../../assets/compute-benchmark-20260621/results/` and are reproducible (Source: 05-compute-benchmark.md:5-10).
> The benchmarks in this section are heavy-load and would interfere with live services, so they rest on the live recording (📼).

Every number this section gives you is clearly labeled with "what tool, under what conditions, where's the raw file." **Data is meant to be checked, not taken on faith** — that's why the sourcing is written in such fine detail; you can go back and verify every single entry.

#### Integer Compute: CoreMark, sysbench, 7-zip

CoreMark is the classic integer composite benchmark for embedded CPUs, and it best reflects single-core and multi-core performance for "ordinary code."

| Metric | Single Thread | 4 Threads | Multi-Core Scaling | Source |
|---|---|---|---|---|
| CoreMark | **6852** | **26511** | **3.87×** | 05-compute-benchmark.md:60 |
| sysbench cpu(prime 20000) | 332.3 ev/s | 1306.4 ev/s | 3.93× | 05-compute-benchmark.md:61 |
| 7-zip composite MIPS | 1414 | 4925 | — | 05-compute-benchmark.md:62 |

**What these numbers tell you.** Multi-core scaling of 3.87× (CoreMark) and 3.93× (sysbench) is **close to the theoretical ceiling of 4×**, which means the workload distribution and memory contention across the four cores are both handled well — split your work four ways across the four cores and you get nearly 4× the throughput, without getting choked by the bus (Source: 05-compute-benchmark.md:78). Single-core CoreMark of 6852 works out to roughly 4.03 CoreMark/MHz, a normal level for the A55.

**📼 Hands-on: reproducing CoreMark (including one gotcha you will definitely hit).** CoreMark is an open-source benchmark released by EEMBC, and it doesn't come pre-installed on the board — you need to **get the source first and cd into its directory** before the Makefile below is available. Clone it in your home directory, then `cd` in (this handbook's measurements were run from inside the CoreMark source directory itself — see the build path `.../src/coremark` at the top of the source log):

```bash
git clone https://github.com/eembc/coremark
cd coremark
```

Once you're inside the `coremark/` directory, build and run the single-thread performance run:

```bash
make XCFLAGS="-O2 -funroll-loops -DPERFORMANCE_RUN=1" load run1.log
```

(The `make` line is verbatim from 01c_coremark.txt:2.) This builds and runs the single-thread performance run, writing the result into `run1.log`. If you type this make line straight from your home directory without first cloning/`cd`-ing in, you'll get `No such file` or `No targets` — because there's no Makefile to be found. Expect to see the following in the log (verbatim, Source: 01c_coremark.txt:79-85):

```text
Total time (secs): 16.053000
Iterations/Sec   : 6852.301750
Iterations       : 110000
CoreMark 1.0 : 6852.301750 / GCC13.3.0 ... / Heap
```

The 4-thread version pushes `Iterations/Sec` up to `26510.815208` (Source: 01c_coremark.txt:87-92).

> ⚠️ **Warning**: **Scenario** — you want to just trust the `DERIVED` field the measurement script prints automatically (CoreMark/MHz, multi-core scaling ratio). **Symptom** — that line prints obviously-bogus placeholder numbers like `CoreMark/MHz(1T)=0.001 | scaling=1.00x`. **Cause** — the awk derivation expression in the script gets fed an empty string and misreads the division operator as the start of a regular expression, throwing a syntax error (every CPU log shows the same DERIVED-field bug). **Prevention/Handling** — **don't trust the DERIVED line**; the correct approach is to hand-calculate from the raw `Iterations/Sec` above it in the log: `6852.301750 ÷ 1700 ≈ 4.03 CoreMark/MHz`, `26510.815208 ÷ 6852.301750 ≈ 3.87×`. That's exactly how the 4.03 and 3.87× in this handbook's table were calculated (Source: 01c_coremark.txt:93, 05-compute-benchmark.md:60).

**stress-ng broken down by method (4 threads, bogo ops/s real)**, showing you the relative cost of different computation types (Source: 01b_cpu_fix.txt:11-23, 05-compute-benchmark.md:63-65):

| Method | bogo ops/s | Description |
|---|---|---|
| `int64` | 3271 | Fastest — integer |
| `float` | 2543 | Single-precision float |
| `double` | 1218 | Double-precision, roughly half the speed of single |
| `fft` | 968 | Transform with memory access |
| `matrixprod` | 65 | Heaviest — matrix multiply (high cache pressure) |

#### Floating Point & Linear Algebra: SGEMM, DGEMM, Small Matrices

Control and estimation algorithms (EKF, MPC, ADRC) are fundamentally matrix operations, so floating-point/linear-algebra throughput is the key to judging "how complex an estimator this board can run." Here GEMM (general matrix multiply) is measured with numpy + OpenBLAS, in units of GFLOPS (billion floating-point operations per second).

| Metric | Single Core | 4 Cores | % of Theoretical Peak | Source |
|---|---|---|---|---|
| SGEMM (fp32) | 7.9 GFLOPS | **28 GFLOPS** | ~51% (peak 54.4) | 05-compute-benchmark.md:88, 02_fp.txt:5-8 |
| DGEMM (fp64) | 3.1 GFLOPS | 10.8 GFLOPS | ~40% | 05-compute-benchmark.md:89, 02_fp.txt:9-11 |

**Two takeaways.** First, **fp64 costs roughly 2.5× fp32** (3.1 vs. 7.9 GFLOPS) — remember this when choosing estimator precision: if fp32 will do, don't reach for fp64 without thinking (Source: 09-compute-capability.md:17). Second, 40–51% of theoretical peak is already a good asymptotic result for large dense matrices; as you'll see shortly, the **small** matrices common in control and estimation don't reach this efficiency.

> 💡 **Tip**: For the 4-core SGEMM number, the main document says "28," the raw log (N=2048) says "28.04," and a rounding to "27.8" shows up elsewhere too (Source: 02_fp.txt:5-8, 05-compute-benchmark.md:88). These are just different roundings of the same measurement — the difference is within 1% and doesn't change any conclusion. This handbook consistently uses the main document's "28 GFLOPS (4 cores)" as the representative value for aggregate compute.

**The truth about small matrices — dispatch overhead is the bottleneck.** Measuring a 12×12 GEMM gives 59457 ops/s, which works out to 16.82 µs per call (d08 rounds this to ~16.9 µs, a rounding of the same measurement; Source: 02_fp.txt:25, 05-compute-benchmark.md:90). But of that 16.82 µs, **roughly 15.8 µs is Python/numpy dispatch overhead — only about 1.1 µs is actual computation** (Source: 09-compute-capability.md:15). This has a huge impact on how you should write your control law:

- If you use **native C/Eigen**, a single 12-state matrix operation is in the µs range — the A55 has plenty of headroom for the linear algebra in a 1 kHz control loop (Source: 05-compute-benchmark.md:92-94).
- If you use **Python/numpy**, you're stuck paying that fixed 15.8 µs dispatch tax, and this tax **doesn't care how small the matrix is** — the "capability ceiling" section later will quantify this as "a Python EKF is stuck in the low-thousands-of-Hz range no matter how small the state dimension is."

#### Memory Bandwidth & Latency: STREAM, tinymembench

| Metric | Value | Source |
|---|---|---|
| STREAM Triad (4 threads) | 5585 MB/s (Copy 4456 / Scale 5836 / Add 5632) | 05-compute-benchmark.md:102, 03_mem.txt:35-39 |
| tinymembench NEON LDP/STP copy | 3463 MB/s | 05-compute-benchmark.md:103, 03b_tinymembench.txt:34 |
| standard memcpy / memset | 3037 / 5761 MB/s | 05-compute-benchmark.md:104 |
| mbw MEMCPY / MCBLOCK | 3295 / 4429 MiB/s | 05-compute-benchmark.md:105, 03_mem.txt:4-21 |

**Memory specs and one important "why."** The board has 16 GB of LPDDR4 (1600 MHz = 3200 MT/s, 8 GB × 2, 2×32-bit, theoretical peak 25.6 GB/s; the spec summary table says LPDDR4 while the board block diagram labels it LPDDR4X-3200, and the SoC controller supports both — see the memory overview in Chapter 1), no swap; `meminfo` measures `MemTotal 15565904 kB`, `MemAvailable 14863188 kB` (Source: 00_inventory.txt:90-97). Note: **the bandwidth the CPU side can actually pull (STREAM 5.6 GB/s) is only about 22% of the memory controller's peak** (Source: 05-compute-benchmark.md:107-109).

This 22% isn't a defect — it's just the A55's nature: it's a small, in-order core with limited memory-level parallelism (MLP), so it can't keep enough outstanding memory requests in flight at once to fill up the bandwidth. The image/tensor moves that actually need high bandwidth are handled through DRP-AI3/DRP's dedicated DMA paths, **bypassing the CPU entirely** (Source: 05-compute-benchmark.md:109-111). So there's no need to be disappointed by this number: 16 GB of capacity is plenty for multiple models plus buffers plus ROS, and the work that actually eats bandwidth has dedicated hardware to carry it.

> ⚠️ **Warning**: There's an alignment problem between sources on the DRAM random-read latency number, so be careful when citing it. The main document says "~196 ns (64 MB working set) / dual-path ~226 ns," but in tinymembench's raw line-by-line output, 196.0 / 226.2 ns actually correspond to the **16 MB** row; the real **64 MB** row is 208.7 / 233.5 ns (Source: 05-compute-benchmark.md:106, 03b_tinymembench.txt:82,84). L2 latency (≤1 MB working set) is roughly 15–19 ns. **Symptom** — if you follow the main document and slap "196 ns" onto a "64 MB" label, you've actually mismatched the two. **Prevention** — when citing an exact latency, go back to the **working-set-size column** in tinymembench's raw output; for memory-bound estimates (the A* case later, for instance), it's enough to use "random latency on the order of ~196–210 ns" — no need to fuss over whether it's pinned to 16 MB or 64 MB.

#### Storage I/O: This Is a Consumer-Grade SD Card

| Metric | Value | Source |
|---|---|---|
| Sequential write (dd buffered+fdatasync, 1GiB) | 28.2 MB/s | 05-compute-benchmark.md:121, 04_storage.txt:6 |
| Sequential read (dd, drop_caches) | 61.9 MB/s | 05-compute-benchmark.md:122 |
| hdparm buffered read / cached read | 71.0 / 1141 MB/s | 05-compute-benchmark.md:123,126 |
| fio 1M sequential read/write | 76.4 / 32.4 MB/s | 05-compute-benchmark.md:124 |
| fio 4K random read/write | 4353 / 1491 IOPS (17/6 MB/s), latency 3.4/10.7 ms | 05-compute-benchmark.md:125 |

**The truth about boot storage.** The rootfs is mounted on `/dev/mmcblk0`, which is actually a **256 GB Samsung SD card** (type SD, manfid 0x1b, oemid "SM", manufactured 2025/10), formatted ext4, with roughly 206 GB usable (Source: 05-compute-benchmark.md:48, 04b_emmc_id.txt:2-8). Its speed is UHS-I SD card class, not the 150–300 MB/s of eMMC 5.1. For a workload like a drone, as an example — 4 Mbps video at roughly 0.5 MB/s, plus telemetry logging — this speed is more than enough (Source: 05-compute-benchmark.md:128-131).

> ⚠️ **Warning**: **Scenario** — you want to confirm what device the boot storage actually is, or you're considering using it as a high-reliability data-logging device (a flight black box, for example). **Symptom** — the header in `04_storage.txt` labels it "eMMC," but the card identifies itself as an **SD card** (the script's label doesn't match the measured device identity — it's actually SD) (Source: 05-compute-benchmark.md:191, 04_storage.txt:1). **Cause** — the board ships with an external SD card as its boot medium, and a consumer-grade SD card's **write endurance and power-loss reliability** are real risks for this kind of high-reliability data logging. **Prevention/Handling** — for high-reliability, power-loss-safe data logging needs like this, switch to on-board eMMC or an industrial-grade pSLC card, and separate high-frequency logs from the rootfs onto different storage devices (Source: 05-compute-benchmark.md:196-198).

> ⚠️ **Warning**: **Scenario** — you use fio with `O_DIRECT` to do 4K random read/write on this SD card. **Symptom** — this path is abnormally slow, only 18 / 5.6 MB/s, with submission latency as high as 55–180 ms. **Cause** — the source only states the phenomenon without explaining the root cause, and this handbook won't fabricate one. **Prevention/Handling** — measure with buffered + fdatasync instead, which better reflects actual filesystem throughput (the 28.2 MB/s write in the table above was measured this way) (Source: 05-compute-benchmark.md:128-129, 04_storage.txt:2-3).

#### Crypto Throughput: No Hardware Acceleration, Software Only

Earlier sections repeatedly mentioned that the A55 lacks aes/sha extensions — here's the exact cost (software implementation, Source: 05-compute-benchmark.md:71-76):

| Algorithm | Single Core | 4 Cores | Source |
|---|---|---|---|
| AES-256-GCM | 34.7 MB/s | 137.6 MB/s | 05-compute-benchmark.md:71 |
| AES-256-CBC | 56.5 MB/s | — | 05-compute-benchmark.md:72 |
| SHA-256 | 111 MB/s | 441 MB/s | 05-compute-benchmark.md:73 |
| SHA-512 | 174 MB/s | — | 05-compute-benchmark.md:74 |
| RSA-2048 sign/verify | 156 / 5809 ops/s | — | 05-compute-benchmark.md:75 |
| ECDSA P-256 sign/verify | 6284 / 2333 ops/s | — | 05-compute-benchmark.md:76 |

> ⚠️ **Warning** (engineering background, not a debugging incident): **Scenario** — you need to encrypt data in transit or at rest. **Symptom** — the A55 lacks ARMv8 Crypto Extensions, so AES/SHA can only run in software, with throughput pinned to the tens-of-MB/s range. **Cause** — `aes`/`sha2` are absent from the CPU flags. **Prevention/Handling** — encrypting a 4 Mbps video stream (needing roughly 0.5 MB/s) is comfortably within reach; but if you have a high-volume encrypted storage/transfer need, watch the bandwidth ceiling (single-core AES ~35–56 MB/s, 4-core ~137 MB/s) (Source: 05-compute-benchmark.md:79-80). One more thing to note for security design: on this H44 part you can't reach any hardware Security IP from Linux, and there's no `/dev/tee` either (TEE = Trusted Execution Environment, such as OP-TEE) — Security = N/A for this part number, i.e. Not Populated, as verified in the hardware manual's Table 1.1-1, p78 (see 08) — so **don't assume you can do high-volume encryption or hardware key isolation**; this throughput is enough for data signing (Source: 09-compute-capability.md:44).

#### DRP-AI3 NPU Inference: Squeezing 145 ms Down to 15 ms

**Measurement setup:** model YOLOX-nano@416 (DRP-AI TVM compiled, INT8 quantized), 351 samples, runtime settings `DRP0_max_freq_factor=2`, `AI-MAC_freq_factor=2`, camera image 1280×720 UYVY (Source: 05-compute-benchmark.md:137-138, 05_npu_drpai.txt:1,4).

| Stage | Time | Source |
|---|---|---|
| Pure inference (DRP-AI3, INT8) | mean 15.26 / p50 15.00 / p95 16.20 / max 27.10 ms | 05-compute-benchmark.md:142, 05_npu_drpai.txt:45 |
| Pre-processing | 6.86 ms (DRP-accelerated) | 05-compute-benchmark.md:143 |
| Post-processing | 1.62 ms (partly on A55) | 05-compute-benchmark.md:143 |
| **End-to-end** | **23.7 ms → 42.1 FPS** | 05-compute-benchmark.md:144 |
| CPU reference (onnxruntime fp32, 4 threads) | 145.6 ms (6.9 FPS) | 05-compute-benchmark.md:145 |
| CPU reference (1 thread) | 422 ms (2.4 FPS) | 05-compute-benchmark.md:145 |

**The speedup ratio and what it really means.** DRP-AI3 is **9.7×** faster than the CPU at 4 threads, and **28.1×** faster than 1 thread (both computed against the 15.0 ms pure-inference p50 as the divisor: 145.6 ÷ 15.0, 422 ÷ 15.0; Source: 05b_npu_cpu.txt, 05_npu_drpai.txt:45). But more important than "how many times faster" is this: it turns YOLOX inference from 145 ms that eats up all four A55 cores into 15 ms on a single dedicated NPU, **freeing the four A55 cores completely to run control and communication protocols (flight control as an example)** — this is exactly what makes "detect/track while flying" possible in the first place (Source: 05-compute-benchmark.md:147).

> 💡 **Tip**: The "inferences per second" figure for pure inference has two different values across the two sources — the main document says **66.7 inf/s** (= 1000 ÷ p50's 15.0 ms), while the raw log prints **65.5 inf/s** (= 1000 ÷ mean's 15.26 ms) (Source: 05-compute-benchmark.md:142, 05_npu_drpai.txt:50). Both are correct — it's just **a difference between using p50 or mean as the divisor**. When citing this, just be clear about which central-tendency convention you're using; don't lump the two numbers together and treat them as a contradiction.

**A counterintuitive but important conclusion: YOLOX-nano doesn't come close to saturating the NPU.** At 15.3 ms, YOLOX-nano's effective compute is only about 65 GFLOP/s — **less than 1%** of the 8 dense TOPS (= 8000 GOP/s; GOP/s = giga operations per second, billion operations per second) (Source: 05-compute-benchmark.md:148, 09-compute-capability.md:53). This proves that a very small model is bound by **fixed overhead/memory bandwidth**, not by compute. The practical implication: you have plenty of headroom left to run a bigger/more accurate model, a pruned model, or multiple models in parallel — the NPU's compute is nowhere near wrung out.

#### GPU (Mali-G31) Compute Throughput

Measured with `gpu_cl_bench.c` (OpenCL, using vec4 + 4 independent accumulators to expose instruction-level parallelism, ILP), verbatim output (Source: 08-gpu-deep-dive.md:51-53; raw log `../../assets/hardware-investigation-20260621/results/17_gpu_opencl.txt`):

```text
Device: Mali-G31 r0p0 | 1 CU @ 630 MHz
vadd correctness: PASS (C[12345]=9258.750 expect 9258.750)
fma_bench(vec4,ILP): 37239.56 ms, 167772 MFLOP -> 4.51 GFLOPS (fp32 peak)
```

**How to read this 4.51 GFLOPS.** This is already **about 89%** of this single-CU Bifrost core's theoretical peak at 630 MHz (4 lanes × 2 × 630M ≈ 5 GFLOPS) — essentially maxed out (Source: 08-gpu-deep-dive.md:63). fp16 is roughly 2× that (~9 GFLOPS). Here's where it lands in the ranking of compute units (Source: 08-gpu-deep-dive.md:58-61):

| Unit | fp32 GFLOPS | Relative to GPU |
|---|---|---|
| Mali-G31 (measured) | 4.51 | 1× |
| Single A55 (measured SGEMM) | 7.9 | 1.75× |
| 4×A55 (measured SGEMM) | 28 | 6.2× |
| DRP-AI3 (INT8) | 8 TOPS | Not comparable (different units) |

In other words — **this GPU's raw FLOPS are lower than a single A55's, roughly 1/6 of the 4-core CPU's**. Its value isn't in "being faster than the CPU," it's in **parallel offload** (moving per-pixel work off the CPU) and **zero-copy** (consuming the camera's/encoder's dma_buf directly) (Source: 08-gpu-deep-dive.md:65).

**Hands-on: enabling OpenCL** (install steps 📼 per the recording; verification commands ✅ Verified on the board; transcripts: live/ch04-followup.txt [clinfo], live/ch04-cpu-periph.txt [eglinfo]; Source: 08-gpu-deep-dive.md:73-75):

```bash
sudo apt install -y ocl-icd-opencl-dev opencl-headers clinfo
echo /usr/lib/aarch64-linux-gnu/libmali.so | sudo tee /etc/OpenCL/vendors/mali.icd
clinfo | grep -iE 'Platform|Device Name|OpenCL'
```

The first two lines register libmali.so as the OpenCL ICD vendor (installation is a state-changing operation, and it is already done on the board), and the third line verifies it took effect. Expect it to list `ARM Platform / Mali-G31 r0p0 / OpenCL 3.0` (✅ Verified on the board with `clinfo`; transcript: live/ch04-followup.txt: verbatim `Platform Name  ARM Platform`, `Device Name  Mali-G31 r0p0`, `Platform Version  OpenCL 3.0 v1.r54p1-…`; also see 08-gpu-deep-dive.md:75, 07-hardware-unit-usage-guide.md:97). To confirm the renderer, use `eglinfo | grep -i renderer`, expecting to print `OpenGL ES profile renderer: Mali-G31` (✅ Verified on the board; transcript: live/ch04-cpu-periph.txt).

> ⚠️ **Warning**: **Scenario** — in a headless environment, you try to use `glmark2-es2 --off-screen` to get a graphics score out of the GPU. **Symptom** — you get the error message `Could not initialize canvas` and it won't run. **Cause** — (EGL, mesa, DRM node, and KMS are all components of the Linux graphics display stack: EGL is the interface layer connecting a drawing API to the windowing system, mesa is the general-purpose open-source graphics driver, and DRM/KMS are the kernel subsystems that manage display and graphics devices.) The generic EGL falls through to mesa, which can't connect to the `mali_kbase` kernel module; the board's only DRM node is the display controller — there's no GPU render node. **Prevention/Handling** — you need a Wayland compositor or a KMS surface to get a graphics score; this is a limitation of the headless setup, not a hardware problem. Note: **the OpenCL compute path is not affected by this limitation** — the 4.51 GFLOPS earlier was obtained normally, headless, using OpenCL (Source: 05-compute-benchmark.md:156-157, 08-gpu-deep-dive.md:51).

#### Real-Time Scheduling Latency: cyclictest (the Single Most Critical Item for Real-Time Control)

For hard real-time control (flight control as an example), "how fast on average" matters far less than "**how late in the worst case**" — miss one hard deadline and the controlled plant can go unstable. This is exactly what cyclictest measures: the latency distribution of scheduled wakeups.

| Scenario | Min | Avg | **Worst** | Source |
|---|---|---|---|---|
| Idle (P80, interval 250 µs, all cores, 30 s) | 14 µs | 18–19 µs | **347–420 µs** (taking the full range across the raw log's four measurement runs, 347–420; the source document's summary notes 359–420) | 05-compute-benchmark.md:166, 07_realtime.txt:6-9 |
| Loaded (stress-ng cpu4+vm2, 30 s) | 13 µs | 21–29 µs | **593–926 µs** | 05-compute-benchmark.md:167, 07_realtime.txt:12-15 |

**How to read this table.** The kernel is `6.10.14-arm64-renesas`, SMP **PREEMPT but not PREEMPT_RT** (Source: 05-compute-benchmark.md:6). A sub-millisecond worst-case latency is exactly what you'd expect from a typical PREEMPT (non-RT) kernel. In plain terms:

- For **≤400 Hz (period ≥2.5 ms)** outer-loop control — plenty of margin. A worst-case jitter of 0.93 ms still leaves room against a 2.5 ms period.
- For a **1 kHz (period 1 ms)** hard real-time inner loop — a jitter of roughly 0.93 ms gets **tight**, eating up almost the entire period (Source: 05-compute-benchmark.md:169-170).

This is exactly why the earlier section said "the hard real-time inner loop (flight control as an example) belongs on R8" — it's not that the A55 can't crunch the numbers, it's that the Linux scheduling uncertainty at this layer is standing in the way.

> ⚠️ **Warning**: **Scenario** — in this systemd + `CONFIG_RT_GROUP_SCHED` (cgroup v2) environment, you try to use `chrt -f` to set a process to SCHED_FIFO real-time priority (even as root). **Symptom** — `chrt -f` reports `EPERM` (permission denied). **Cause** — cgroup v2's RT_GROUP_SCHED blocks RT bandwidth for sub-cgroups by default. **Prevention/Handling** — the data in the table above was only obtained by temporarily releasing RT bandwidth for the measurement: before running cyclictest, run `sudo sh -c 'echo -1 > /proc/sys/kernel/sched_rt_runtime_us'`, then restore it to the default after measuring with `sudo sh -c 'echo 950000 > /proc/sys/kernel/sched_rt_runtime_us'` (this is only a temporary setting for the duration of the measurement, not a permanent change); the long-term fix is to rebuild a PREEMPT_RT kernel and adjust the cgroup RT bandwidth settings (Source: 05-compute-benchmark.md:171-172, 07_realtime.txt:2-3). This gotcha matters a lot: before planning any SCHED_FIFO loop on the A55, you need to clear this block first, or priority simply won't be settable at all.

#### Thermals & Throttling: Fanless, Zero Throttling (Two Measurements, Different Conditions)

| Dataset | Idle | Peak Under Full Load | Throttling | Source |
|---|---|---|---|---|
| **d04** (with time-series CSV) | 35 / 36 °C (tz0/tz1) | 38 / 39 °C, all cores, 5 min | 1700 MHz throughout, 548/548 samples, zero throttling | 05-compute-benchmark.md:180-182, 08_thermal.txt:8-10 |
| **d03** (no measurement conditions or host recorded) | ~35 °C (room temp 26 °C) | ~50 °C after 5 min of stress-ng; ~42 °C under continuous YOLOX inference | No throttling, no reboot | 04-hardware-quickref.md:210-212 |

**The two measurements' peaks differ a lot, yet both say no throttling — this has to be laid out honestly side by side, not cherry-picked.** d04's peak under full load is 38–39 °C, while d03's peak after 5 minutes of stress-ng reaches roughly 50 °C (Source: 04-hardware-quickref.md:210-212, 05-compute-benchmark.md:180-182). The gap comes from different measurement conditions (load type, environment, whether time-series data was attached) — d04 has point-by-point CSV evidence, while d03 records no conditions or host. The shared conclusion is still reliable: **fanless, zero throttling, only a few degrees of temperature rise under full load**; the board has a metal heatsink and needs no active cooling for now — this also echoes the DRP-AI3 white paper's claim that "fanless alone reaches the class of fan-equipped competitors" (Source: 05-compute-benchmark.md:184-185, 04-hardware-quickref.md:214).

> ⚠️ **Warning**: **Scenario** — you see "only 38 °C under full load" and conclude thermals are a non-issue. **Symptom (expected)** — once mounted in an enclosure, the temperature will run noticeably higher than on a bare board. **Cause** — every thermal test so far has been done on a **bare board** (no enclosure); an enclosure will substantially restrict airflow. **Prevention** — before actually mounting it in a sealed, passively-cooled embedded environment (an airborne platform, for example), you must **re-measure** thermals in the enclosed state — don't treat the bare-board numbers as final (Source: 04-hardware-quickref.md:214).

### Capability Ceiling Derivation: What Hz Can a Given Algorithm Hit

Benchmarks give you "raw throughput," but what you really want to know is "**what Hz can my EKF/MPC/FFT actually hit**." This section translates the measurements into that answer.

**Methodology first, so you know how much to trust these Hz numbers** (Source: 09-compute-capability.md:5,17):

1. **The anchor is measured; the extrapolation is estimated.** Every number below is anchored to the measured values; anything marked **ESTIMATE** is an estimate extrapolated from measurements, with stated assumptions — it is **not directly measured**. When writing code, treat "measured" and "ESTIMATE" as separate categories.
2. **Small matrices take a 40% discount.** The measured 7.9/3.1 GFLOPS are asymptotic values for large dense matrices; the small matrices common in control and estimation (N<50) can typically only achieve 30–50% of that, so every O(N³) core estimate uniformly applies a **40% discount**.
3. **These Hz figures are "the compute ceiling of a single loop on one dedicated core,"** unless noted otherwise. The real-world achievable hard real-time rate still needs to further deduct scheduling/IO overhead, and apply the jitter ceiling from cyclictest earlier.

> **Common anchors (all measured on the board)**: fp64 DGEMM 3.1 (1 core)/10.8 (4 cores), fp32 SGEMM 7.9 (1 core)/28 (4 cores) GFLOPS, numpy 12×12 59k ops/s (16.9 µs, including 15.8 µs dispatch), cyclictest under load max 926 µs / avg 29 µs, CoreMark 6852/26511, 7-zip 1414/4925 MIPS, STREAM 5.6 GB/s, memcpy 3.5 GB/s, DRAM latency 196 ns (Source: 09-compute-capability.md:15).

#### EKF (Extended Kalman Filter): What Hz Can It Hit

EKF is the backbone of real-time state estimation (typical applications include attitude, position, and INS/GNSS fusion). The following are native C, single-core unless noted otherwise, and all ESTIMATE (Source: 09-compute-capability.md:23-27):

| EKF Size | Single-Core Ceiling (fp64 / fp32) | 4 Cores | Practical Recommendation |
|---|---|---|---|
| 6-state (quadrotor attitude/position) | >100 kHz (~7 µs/step) | — | Running at 200–1000 Hz uses <1% of one core |
| 15-state (INS/GNSS error-state) | ~90 kHz / 230 kHz | — | Comfortably 200–400 Hz, <1% of a core |
| 50-state | ~2.5 kHz / 6.3 kHz | ~8.6 kHz | Far exceeds the typical 100–500 Hz real-time control requirement |
| 100-state | ~310 Hz / 790 Hz | ~1.08 kHz | Still real-time capable at 100–300 Hz |

**The plain-language conclusion of this table**: at this scale (state dimensions mostly ranging from a handful to a few dozen), the A55 has **way more compute than an EKF needs** — you can run several small-to-medium EKFs simultaneously and still have plenty left over (Source: 09-compute-capability.md:17). What will actually bite you isn't computation — it's the Python trap below.

> ⚠️ **Warning**: **Scenario** — you write a high-frequency estimation or control loop in Python/numpy. **Symptom** — the frequency gets stuck in the low thousands of Hz or even lower, and shrinking the state dimension doesn't make it any faster. **Cause** — each numpy call is measured at roughly 16.9 µs (of which about 15.8 µs is pure dispatch overhead), and this is a **floor that's independent of matrix size**; a Python EKF making roughly 10 numpy calls per step tops out at best around 5900 Hz (20 calls → ~2950 Hz; 50 calls → ~1180 Hz; if written as an element-by-element Python loop → drops to the tens of Hz). **Prevention/Handling** — production estimation/control logic **must be written in native C/C++**; Python is fine for offline analysis and prototyping, not for high-frequency real-time loops (Source: 09-compute-capability.md:27,40).

#### MPC (Model Predictive Control): What Hz Can It Hit

Condensed MPC (interior-point, ~12 iterations), native C fp64 single-core, decision dimension nz = number of inputs × horizon, all ESTIMATE (Source: 09-compute-capability.md:29-30):

| Decision Dimension nz | Single-Core fp64 | Notes |
|---|---|---|
| 20 (e.g. n=6, H=10, m=2) | ~9.7 kHz | — |
| 40 | ~1.2 kHz | — |
| 80 (e.g. n=12, H=20, m=4) | ~150 Hz | fp32 ~2.5× faster, 4 cores ~3.5× faster |
| 120 (e.g. n=12, H=30, m=4) | ~45 Hz | Already on the slow side |

**Practical recommendation (state ≤12, horizon ≤20, inputs ≤4, warm-started)**: a single core can easily reach 100–200 Hz, leaving 3 cores free for other work; real solvers (OSQP/qpOASES) using warm-start are typically **3–10× faster** than the cold-start dense-IP bound in the table above, so the table above is a **conservative lower bound** (Source: 09-compute-capability.md:29-30).

> ⚠️ **Warning**: **Scenario** — you want to run a large MPC with a high update rate, long horizon, and many inputs. **Symptom** — the update rate drops below 50 Hz. **Cause** — once the dense condensed MPC's decision dimension m×H exceeds roughly 120 (e.g., state 12, horizon 30, 4 inputs), even the fp32 4-core cold-start bound falls below 50 Hz. **Prevention** — if you need >50 Hz, don't push the horizon past 30; an MPC with a long horizon/many inputs/high update rate simply isn't reachable without a sparse/structured solver, and it doesn't belong in a hard loop to begin with (Source: 09-compute-capability.md:30,42).

#### FFT: What Hz Can It Hit

FFT (real-time, fp32, roughly 5N·log₂N flops, native C, single-core @30% efficiency, all ESTIMATE, Source: 09-compute-capability.md:31):

| Points N | Transforms/sec | Points N | Transforms/sec |
|---|---|---|---|
| 1024 | ~46k/s | 65536 | ~450/s |
| 4096 | ~9.6k/s | 262144 | ~100/s |
| 16384 | ~2.1k/s | 1M | ~23/s |

Large FFTs shift from being compute-bound to being memory-transfer-bound (memcpy 3.5 GB/s), but within the range of the table above, compute still dominates. Practical judgment: small-to-medium FFTs (N≤16384) are fine for high-rate per-sample streaming; **large FFTs (N≥262144) are better suited to offline or chunked processing, not high-rate per-sample streaming DSP** (Source: 09-compute-capability.md:31,43).

#### Other Common Core Computations (for Planning Reference)

- **Dense LU decomposition** (flops≈⅔N³, the foundation of any solver): N=50 ~15k/38k Hz (fp64/fp32), N=100 ~1.9k/4.7k, N=200 ~230/590, N=500 ~15/38 (4-core fp64 ~52), N=1000 ~2 Hz (no longer real-time) (Source: 09-compute-capability.md:28).
- **Graph search / A***: single-core ~1.4–2.8 M nodes/s, 4-core ~5–10 M/s; but large grid graphs get bound by DRAM random latency (~196 ns/miss), dropping in practice to roughly 0.5–1 M nodes/s (Source: 09-compute-capability.md:32).
- **NLS / bundle adjustment / MHE** (Gauss-Newton, dense ~N³/3 per iteration): N=50 ~30k iter/s, N=100 ~3.7k, N=200 ~460, N=500 ~30 (fp64 single-core); an NLS with 200 variables and 10 iterations takes roughly 22 ms → ~45 Hz; sparse problems (typical SLAM/MHE) are much faster (Source: 09-compute-capability.md:33).

> ⚠️ **Warning**: **Scenario** — you do large dense fp64 linear algebra inside a real-time loop. **Symptom** — the solve can't keep up with the control period. **Cause** — dense LU at N=1000 runs at roughly 2 Hz (single-core)/6.5 Hz (4 cores), and fp64 single-core is ≤15 Hz for N≥500. **Prevention** — keep dense fp64 problem dimensions at N≤200 to maintain >100 Hz, or switch to a sparse structure (Source: 09-compute-capability.md:41).

#### How to Split the Budget Across the Four Cores

The measured 4-thread scaling of 3.87× means roughly 3.8 independent cores' worth of compute is available. Here's one practical example of how to allocate cores (using an airborne workload as an example, Source: 09-compute-capability.md:34):

```text
core0 : 200 Hz MPC (n12/H20, ~30% utilization) + safety monitoring
core1 : 2-3 EKFs @400 Hz (each <5%)
core2 : planner / A*
core3 : DRP-AI3 NPU feeding + OS / ROS
```

Even with this allocation, there's still plenty of headroom left for an application stack like the one described earlier — reconfirming that "the A55's limit isn't compute."

**Finally, let's nail down the ceiling for Linux hard real-time**: worst-case jitter under load is 926 µs. For jitter to stay under 10% of the period, the period has to be >9.26 ms (i.e. the period is **10×** the jitter — the more conservative end → about 108 Hz); relax the criterion to a period of only **3×** the jitter (less margin, the more aggressive end) and you get roughly 360 Hz. So the **reliable hard-loop ceiling on Linux/A55 falls in the range of roughly 100–360 Hz**: the low end (108 Hz) has the large margin and the high end (360 Hz) the small one, so it is *not* the case that 360 Hz is the conservative figure. Anything requiring a guaranteed hard deadline above 300–500 Hz (inner-loop attitude/RPM, motor commutation) **must run on the Cortex-R8** (Source: 09-compute-capability.md:35). This point, the earlier cyclictest data, and R8's role all confirm each other — it's the one sentence in this section most worth remembering.

#### DRP-AI3: What FPS Can Each Model Category Hit

Finally, back to the NPU's capability side. The following are all INT8, inference-only; except for YOLOX-nano, which is measured, everything else is **ESTIMATE** (extrapolated from the nano anchor by FLOPs ratio, Source: 09-compute-capability.md:63-69). **To work out real end-to-end numbers, remember to deduct roughly 8 ms for pre + post-processing.**

| Category | Representative Models and Estimated FPS |
|---|---|
| Object detection | YOLOX-nano@416 **measured 15.3 ms / 65 FPS (inference), 42 FPS (end-to-end)**; YOLOX-tiny 35–50, YOLOv5n/8n@640 28–45, SSD-MobileNetV2 55–80, YOLOX-s@640 14–22 |
| Image classification | ResNet-50@224 65–125, ResNet-18 90–160, MobileNetV2 80–160, EfficientNet-Lite 50–100 (most efficient in this class) |
| Semantic segmentation | DeepLabV3+MobileNet@512 14–28, UNet@256 22–40, UNet@512 8–25, SegFormer-B0 5–12 |
| Pose estimation | MoveNet-Lightning@192 60–100, Thunder@256 40–65, HRNet-W32 14–28, RTMPose 33–65 |
| Monocular depth | MiDaS-small@256 14–28, FastDepth@224 33–65 (outputs relative depth; metric use requires scale calibration) |
| Face/ReID | SCRFD@640 33–65, MobileFaceNet@112 100–200, ArcFace-R50@112 55–100, OSNet 50–100 (well-suited to target following, drones as an example) |
| Temporal/sequence | 1D-CNN/TCN 60–120, ST-GCN 28–65; RNN/LSTM/attention-based models fall back to the A55 |

**DRP-AI3's capability boundary (stated honestly)**: it's competent at INT8 CNNs for detection/classification/segmentation/pose/depth/face-ReID (roughly 7–60 FPS); it **cannot effectively run** LLMs, large ViT/transformers, networks that require fp32, or computation graphs with dynamic shapes (Source: 09-compute-capability.md:55).

> ⚠️ **Warning**: **Scenario** — you want to run multiple models simultaneously (detection + ReID + depth) or multiple streams, expecting total FPS to be the sum of each model's FPS. **Symptom** — overall FPS comes in far below what you expected. **Cause** — the board has **only one `/dev/drpai0` AI-MAC — a single, serialized NPU** — so multiple models/streams execute sequentially via time-slicing; total FPS = 1 ÷ (sum of each model's latency), with no true parallelism. **Prevention** — when planning a pipeline, budget total latency using a "sequential time budget," not a "parallelism assumption"; also note that R8/M33 cannot be used for AI offload under Linux (Source: 09-compute-capability.md:79).

> ⚠️ **Warning**: **Scenario** — you measure a fast pure-inference time and then use that inference time directly as the latency of the whole pipeline when scheduling. **Symptom** — the real end-to-end figure comes out noticeably longer than the inference time. **Cause** — pure inference itself has **no fixed floor**: measured on the board, 224² classifiers run mobilenetv2 in **1.34 ms** and resnet50 in **4.19 ms**, entirely on DRP-AI with zero CPU fallback (Source: `assets/hardware-investigation-20260621/results/19_drpai_models.txt`), so small models really can land far below 10 ms. What makes end-to-end longer than pure inference is the pre-processing and post-processing stages **outside** inference (measured for YOLOX-nano@416: pre-processing 6.86 ms [DRP-accelerated] + post-processing 1.62 ms [partly A55], Source: 05-compute-benchmark.md:143) — that's the cost of pipeline steps like image scaling and format conversion, not a fixed NPU startup latency added to every frame. **Prevention** — when scheduling, always budget end-to-end as "inference time + pre/post-processing" (measured end-to-end for YOLOX-nano@416 is 23.7 ms vs. pure inference 15.3 ms); don't treat a small model's pure-inference time as the overall latency, and don't assume there's a fixed startup floor (Source: 09-compute-capability.md:78).

### The Three Compute Engines, Each in One Sentence

Let's wrap up the whole section into a single "what goes where" comparison table (measured comparison, Source: 09-compute-capability.md:89-91):

| Engine | Measured Capability | Good For | Not Good For |
|---|---|---|---|
| **4×A55 CPU** | 28 GFLOPS fp32 / 10.8 fp64 / CoreMark 26511 | Estimation (EKF/MHE), planning (A*/MPC), comms protocols, CPU fallback | Hard real-time >300 Hz (0.93 ms jitter, must move to R8), large fp64, high-frequency Python loops |
| **DRP-AI3 NPU** | YOLOX-nano 15 ms (42 FPS end-to-end), 8/80 TOPS | INT8 CNN detection/classification/segmentation/pose/depth/ReID (~7–60 FPS) | LLM/large transformers, fp32 networks, dynamic shapes |
| **Mali-G31 GPU** | 4.51 GFLOPS fp32, OpenCL 3.0 | Parallel per-pixel image/CV kernels, camera zero-copy, HMI | General-purpose FP acceleration, primary AI inference |

One-sentence summary: as a single-board heterogeneous compute platform for "control (flight control as an example) + perception + communication," the RZ/V2H is fully up to the job, and DRP-AI3 is the decisive edge for vision inference; you just need to design around three engineering realities — **(1) boot storage is a consumer-grade SD card, (2) the A55 has no AES/SHA hardware crypto, and (3) the default kernel is not PREEMPT_RT and SCHED_FIFO is blocked by cgroup** (Source: 05-compute-benchmark.md:27). This section's benchmarks and warning boxes have given you exact numbers and workarounds for all three.

### Hands-On Verification: Confirm Your Board Matches This Section's Data (see markers below for each step)

Once you've finished this section, use the steps below to quickly confirm "this board's compute-unit state matches this section's baseline." The pass/fail criteria are written after each step.

**✅ Verification 1 (verified on the board; transcript: live/ch04-cpu-periph.txt): Are the clock and governor locked correctly?**

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

Pass/fail: should be `performance` and `1700000`. If not, your benchmark numbers won't match this section because of frequency scaling (Source: 07-hardware-unit-usage-guide.md:178-179).

**✅ Verification 2 (verified on the board, with no output; transcript: live/ch04-followup.txt): Are the crypto extensions really absent?**

```bash
grep -oE 'aes|sha2' /proc/cpuinfo | sort -u
```

Pass/fail: there should be **no output at all**. Any output means your board/kernel differs from this section's assumptions, and the crypto-throughput conclusions need to be re-evaluated (Source: 05-compute-benchmark.md:46).

**⏸ Verification 3 (a benchmark load that would interfere with live services, so it rests on the live recording; requires `sudo apt install sysbench` first, since a clean image doesn't have it pre-installed and you'll otherwise get command not found): Run a sysbench and see whether multi-core scaling is close to linear.**

```bash
sysbench cpu --cpu-max-prime=20000 --threads=1 --time=20 run | grep 'events per second'
sysbench cpu --cpu-max-prime=20000 --threads=4 --time=20 run | grep 'events per second'
```

Pass/fail: 4 threads ÷ 1 thread should be close to **3.9×** (this section measured 1306.4 ÷ 332.3 ≈ 3.93×). Noticeably below 3.5× indicates background load or a thermal/power anomaly (Source: 05-compute-benchmark.md:61, 01_cpu.txt:3-17).

**✅ Verification 4 (verified on the board, see note): Confirm thermals are normal and there's no throttling.**

```bash
cat /sys/class/thermal/thermal_zone0/temp
cat /sys/class/thermal/thermal_zone1/temp
```

Pass/fail: at idle, both zones should read around 35000–36000 (i.e., 35–36 °C) (units of m°C). After 5 minutes of full load, they should still sit well below the throttling threshold — this section measured 1700 MHz throughout, 548/548 samples, zero throttling (Source: 00_inventory.txt:156-157, 05-compute-benchmark.md:180-182). **Note: these are bare-board numbers; you must re-measure once it's in an enclosure.** (✅ Verified on the board; transcripts: live/ch04-cpu-periph.txt [temperature], live/ch04-reserved-mem.txt [load average]: both zones read `39000` (39 °C) with resident services running and a load average around 1.0, so the board was not actually idle; running a few degrees above the 35–36 °C idle baseline is reasonable under those conditions. To check the "35–36 °C idle" figure properly, measure with no load.)

**⏸ Verification 5 (requires SCHED_FIFO and a full-load scenario that would interfere with live services, so it rests on the live recording; optional, requires rt-tests).**

```bash
# -p80 requires SCHED_FIFO; temporarily release RT bandwidth first, or you'll get EPERM (a temporary setting for the duration of the measurement)
sudo sh -c 'echo -1 > /proc/sys/kernel/sched_rt_runtime_us'
sudo cyclictest -p80 -i250 -a -t -D30 -q
# restore to the default immediately after measuring
sudo sh -c 'echo 950000 > /proc/sys/kernel/sched_rt_runtime_us'
```

Pass/fail: worst-case latency at idle should be in the **sub-millisecond range** (this section measured idle max 347–420 µs, loaded max 593–926 µs). If worst-case latency is far beyond 1 ms, it indicates an unexpected source of high latency, and will affect your planning around "the A55 hard real-time ceiling is roughly 100–360 Hz" (Source: 05-compute-benchmark.md:166-167, 07_realtime.txt:6-15). Remember: if `chrt -f` reports EPERM, that's cgroup RT_GROUP_SCHED blocking you, not insufficient privileges (see the warning box earlier).

If everything passes, it means the board in your hands stands on the same baseline as this section's capability-ceiling derivation — the chapters that follow (perception, control, system integration) can then safely cite the Hz numbers here when planning your system's software architecture.

---
