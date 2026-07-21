# g4 · Memory and Storage

This group covers, in one pass, the five hardware units on the RZ/V2H that answer the question of "where does the data actually sit": the **internal SRAM** — the fastest thing on the chip, yet reserved in its entirety; the **Boot ROM**, the first thing the SoC runs after power-on; the **LPDDR4/4X controller**, which shoulders the system's main memory; the **xSPI/NOR flash**, which carries the small boot image; and the **SDHI/SD card**, which holds the OS and bulk data. Together they form one complete memory-and-storage path, running all the way from "on-chip, volatile, a landing spot for code" to "off-chip, non-volatile, for the filesystem."

Let's start with a diagram that puts these five units into one shared frame of reference, so you won't lose your bearings as you read each section below — the top two blocks are on-chip, the other three are off-chip; the left half is the path for "used for execution / as working memory," and the right half is the path for "used for long-term data storage":

```text
                     RZ/V2H Memory and Storage Path (This Board's Configuration)
  ┌─────────────────────────── On-Chip ────────────────────────────────┐
  │  Boot ROM 128 KB (read-only)          Internal SRAM 6 MB (12×512 KB, ECC) │
  │  First thing to run at boot;          Landing zone for firmware/CM33/CR8, │
  │  invisible to Linux after handoff     cross-core IPC                     │
  │                                       — Reserved, not in Linux's          │
  │                                         general memory pool               │
  └───────────────────────────────────────────────────────────────────────────┘
                                     │
  ┌─────────────────────────── External (off-chip) ───────────────────────────┐
  │  LPDDR4/4X controller ×2  →  16 GB LPDDR4X (volatile main memory)          │
  │      Linux general RAM (~15.5 GB available) + hardware-accelerator         │
  │      carveouts                                                             │
  │                                                                             │
  │  xSPI  →  NOR flash 64 MB (non-volatile, byte-addressable, XiP-capable)    │
  │      Small boot-image partitions: bl2 / fip / env / test-area              │
  │                                                                             │
  │  SDHI0  →  SD card 256 GB (non-volatile, block device, removable)          │
  │      rootfs and bulk data: /dev/mmcblk0                                    │
  └───────────────────────────────────────────────────────────────────────────┘
```

> The **16 GB / 64 MB / 256 GB marked in the diagram are the capacities this particular board actually has soldered on or plugged in** — they are not the ceiling of what each controller supports. How large a capacity each controller can actually handle on its own is covered separately in each section's "Key Capabilities & Limits." Whenever you see a capacity figure, first separate two different things: "how much this particular board happens to be fitted with" versus "how much this controller can support at maximum."

## Units in This Group

| Unit | One-liner | Board Status |
|---|---|---|
| [Internal SRAM 6 MB](#internal-sram-6-mb-12512-kb-ecc) | 12 blocks of 512 KB each, built-in on-chip RAM with ECC, serving as a landing area for firmware and heterogeneous cores and as a cross-core shared buffer | Enabled but **Reserved** (claimed by boot firmware/CM33-CR8 remoteproc/vring, not added to Linux's general memory pool) |
| [Boot ROM](#boot-rom) | 128 KB read-only ROM that stores the boot firmware — the first thing the SoC runs after power-on | Enabled (mandatory boot path); **not separately logged in the development notes (an honest gap)** |
| [LPDDR4/4X Memory Controller ×2](#lpddr44x-memory-controller-2) | Dual-channel external DRAM controller, supporting LPDDR4/4X-3200 and in-line ECC — the sole path to the system's main memory | Enabled (16 GB LPDDR4X-3200, initialized by BL2, ~15.5 GB available to Linux) |
| [xSPI (NOR Flash Interface)](#xspi-nor-flash-interface) | High-throughput, low-pin-count serial flash interface, carrying the boot-use NOR flash, memory-mappable for direct reads | Enabled (`xspi@11030000`; MTD `/dev/mtd0..3` = bl2/fip/env/test-area) |
| [SDHI/eMMC/SDIO Host ×3](#sdhiemmcsdio-host-3) | 3 channels of SD/MMC host interface, ch0 compatible with eMMC, carrying the boot media and high-capacity block storage | Partially Enabled (SDHI0 = rootfs on `/dev/mmcblk0`; the two eMMC controllers are DT-disabled) |

## In This File

- [Internal SRAM 6 MB (12×512 KB, ECC)](#internal-sram-6-mb-12512-kb-ecc)
- [Boot ROM](#boot-rom)
- [LPDDR4/4X Memory Controller ×2](#lpddr44x-memory-controller-2)
- [xSPI (NOR Flash Interface)](#xspi-nor-flash-interface)
- [SDHI/eMMC/SDIO Host ×3](#sdhiemmcsdio-host-3)

---

## Internal SRAM 6 MB (12×512 KB, ECC)

### What This Is (the Mechanism)

This LSI (large-scale integration — meaning the SoC chip itself) has **12 blocks of on-chip RAM, 512 KB each**, built in (numbered SRAM0 through SRAM11); the Manual states its role very directly: it serves as the working-memory area for the three core types CM33, CA55, and CR8 (the Manual, 3.2.1, verbatim: "This LSI has twelve 512-Kbyte areas of on-chip RAM for use as the CM33, CA55, and CR8 working areas."). The three core types here are the ones covered in 4.2 — the system-management core Cortex-M33, the application-processing core Cortex-A55, and the real-time core Cortex-R8 — and SRAM is a block of fast on-chip memory the three of them share.

Every SRAM block has built-in **ECC** (error-correcting code — storing a few extra check bits alongside the data, so bit flips can be detected, and even corrected, on readback). It can detect and correct 1-bit errors, and detect 2-bit errors; whenever a 1-bit or 2-bit error occurs, the faulting address is logged into an ECC error address register, so software can later look up exactly where the error happened (the Manual, 3.2.1).

The data bus width differs from one SRAM block to the next, and so does the number of ECC channels it's split into (a "channel" here being a slice of data width over which the ECC circuit computes independently): SRAM0/1/3 are 64-bit (split into 2 channels of 32-bit each); SRAM2 is 128-bit (split into 4 channels); SRAM4 through SRAM11 are 256-bit (split into 8 channels) (the Manual, 3.2.1.1 through 3.2.1.4). The wider the block, the more data it can move in one go, which favors cores with higher bandwidth needs (real-time computation on the CR8, for example).

The Manual's Table 3.2-1 (p265) lists the base address of every individual SRAMm block. Starting from `0x08000000`, each block adds `0x80000` (= 512 KB) going up: SRAM0 = `0_0800_0000h`, SRAM2 = `0_0810_0000h`, all the way to SRAM11 = `0_0858_0000h` (the Manual, Table 3.2-1; the address-increment pattern is also covered in this handbook's 4.4, the address cross-reference table in "Official Documentation Guide"). The same table also gives the mapped Code Area/Data Area addresses under the CM33's address space (for example, SRAM0's Code Area on the CM33 non-secure side = `1800_0000h`) — the same physical SRAM block has different addresses depending on which core, and which security state, you're viewing it from.

The register-level detail for ECC (control and address registers such as SRAMm_CTL_n and SRAMm_EADp_n) is in the Manual's 3.2.2 through 3.2.3; ECC initialization and usage notes start at 3.2.4. These are register-level specifics that this section won't expand on — it's enough for you to know they exist, and which section to go look them up in.

### How You See This Under Linux

The conclusion first: **on this board, SRAM has no general-purpose `/dev` node — "directly allocating a block of SRAM to use" from within Linux simply isn't something you can do** — because the entire block has already been claimed by boot-stage firmware and the heterogeneous cores.

Mechanism-wise, Linux has two standard ways of handling this kind of on-chip SRAM: one is through the `mmio-sram` SRAM driver (source in the kernel's `drivers/misc/sram.c`, with device tree compatible string `mmio-sram`), which exposes it as a "memory pool" for other drivers to borrow from; the other is to carve it up as a reserved-memory carveout (a reserved region — a slice cut out of the memory address space at boot time and marked as dedicated to a particular piece of hardware, off-limits to Linux's general allocator), handed off to remoteproc (the remote processor framework, which Linux uses to load firmware for coprocessors and manage their power state) to consume. Neither approach produces a device file that a userspace program can open (Source: `07-hardware-unit-usage-guide.md:222-240`).

On-board measurement (doc07 §14): this block of SRAM is claimed by boot firmware (bl2/fip, i.e., the second-stage bootloader and the firmware image), by CM33/CR8's remoteproc, and by the vring/IPC mailbox (the shared message ring and mailbox used for cross-core communication) — it is **not** folded into Linux's general memory pool. This handbook's resource map therefore marks its status as "Enabled but **Reserved**" — the hardware is alive and in use, it's just not open for your everyday Linux programs to repurpose (Source: `07-hardware-unit-usage-guide.md:222-240`; the status vocabulary is defined in file 00's "Learn to Read 'Status' First" — the "Reserved" entry there uses exactly this 6 MB of SRAM as its example).

You can see this "Reserved" status for yourself on the board:

```bash
# See where SRAM/reserved regions/cross-core blocks land in the physical address map
sudo cat /proc/iomem | grep -iE 'sram|reserved|cm33|vring'
# List every memory-reservation node (the live device tree)
ls /proc/device-tree/reserved-memory/
# Check remoteproc0's (CM33 on this board) current name and state
cat /sys/class/remoteproc/remoteproc0/{name,state}
```

The expected output of the last command is `cm33` and `offline` — CM33's remoteproc entry exists, but it's offline by default (no firmware built in), which matches the description of CM33 given in 4.1 and 4.2.

### Key Capabilities & Limits

- **Total capacity 6 MB**: 12 × 512 KB (the Manual, 3.2.1, verbatim "twelve 512-Kbyte areas"). This is the sum of all 12 blocks, not the usable size of a single block — and as noted above, on this board the full 6 MB is already Reserved and does not enter Linux's general memory pool.
- **ECC capabilities (the Manual, 3.2.1.5, enumerated verbatim)**:
  - Can detect and correct 1-bit errors, and detect 2-bit errors; but **an error of 3 bits or more cannot be handled correctly, and may lead to misdetection and mis-correction** (the Manual, verbatim: "An error involving 3 or more bits cannot be correctly handled and may result in erroneous detection and correction.") — this is the boundary of what ECC can do; don't assume ECC is a cure-all.
  - Error detection, error correction, and 1-bit correction can each be independently enabled or disabled.
  - Each channel has **8** ECC error address registers; once a channel's 8 registers are all full, it generates an overflow interrupt.
  - Whether to issue an interrupt request can be set independently for 1-bit and 2-bit errors.
- **Bus-width tiers**: SRAM0/1/3 = 64-bit, SRAM2 = 128-bit, SRAM4-11 = 256-bit (the Manual, 3.2.1.1-3.2.1.4).
- **Board status**: Enabled but **Reserved** (doc07 §14; doc06 §3) — not general-purpose RAM that Linux is free to allocate.

### When You'd Actually Use This

Mechanism first, then whether you'd actually use it. SRAM is **on-chip** memory — accessing it doesn't need to go through arbitration by the external DDR controller, so latency is low, and it's already available extremely early in boot, before DDR has even finished initializing. Precisely because of these two traits, boot firmware and the heterogeneous cores (CM33/CR8) commonly use SRAM as a landing area for three purposes: first, as the place where code and data live **during boot itself** (when DDR may not be up yet); second, as the shared buffer (vring, mailbox) for **cross-core IPC** (inter-processor communication); third, for latency-sensitive real-time data that can't afford the jitter of DDR arbitration.

From this, here's a decision rule you can apply:

- **You'll be using SRAM in this role** — if your application needs to load CM33/CR8 firmware, or needs low-latency shared-memory communication between Linux and the real-time cores, then under the hood you'll be using SRAM (though this is mostly arranged for you by the remoteproc framework and the firmware, not something you configure by hand). Take motor control as an example: if a hard real-time control loop running on the R8 needs to exchange state with a Linux application on the A55, that cross-core message ring will quite naturally land on SRAM, or on a vring reserved out of DRAM.
- **You won't be using SRAM directly** — ordinary Linux applications (image processing or AI inference data buffers, for example) won't, and can't, be allocated this SRAM: it isn't added to the kernel's page allocator, and its 6 MB capacity is far smaller than the dataset sizes typical of this kind of workload (model weights and frame buffers routinely run to hundreds of MB). Needs like this should go through DDR — see the LPDDR4/4X section below.

This board uses SRAM0/1 as the cold-boot landing area for CM33, with the remaining blocks used by remoteproc's vring — that's just **one instance** of SRAM's general role of "hosting heterogeneous-core boot and IPC," not the only way SRAM can be used — swap in a different firmware set, a different division of labor between cores, and the blocks it lands on will differ.

> **Endnote (Sources)**: The Hardware User's Manual r01uh1032ej0130 §3.2 SRAM (p265-268, functional overview; registers from 3.2.2 p269). On-board evidence: `07-hardware-unit-usage-guide.md:222-240` (doc07 §14), `06-hardware-resource-map.md` (doc06 §3).

---

## Boot ROM

### What This Is (the Mechanism)

This is a **Boot ROM**, and it has exactly one job: storing the boot firmware (the Manual, 3.3.1, verbatim: "This ROM is a boot ROM, which stores the boot F/W."). The contents of a ROM (read-only memory) are fixed at the factory and can't be rewritten at runtime.

Its capacity is **128 KB** (the Manual, 3.3.1, verbatim: "The ROM has a capacity of 128 KB.") — and that's also the only spec figure the Manual gives for this ROM.

Internally, this ROM hangs off the system bus through a 64-bit AXI3 interface, and it's read-only (the Manual, Figure 3.3-1: `System Bus → 64-bit AXI3 Master → 64-bit AXI3 Slave → 128-KB ROM`). AXI3 is an on-chip bus protocol defined by Arm; you don't need to remember the details — just know that it's a high-speed channel connecting blocks inside the SoC.

### How You See This Under Linux

**You won't see this Boot ROM from within Linux, and that's normal — it's not broken.** It only executes during the very earliest boot stage right after the SoC powers on; once it hands control over to the next-stage bootloader, it's completely invisible to Linux — no device node, no interface a user can query. The development notes doc06/doc07 don't have a section on this Boot ROM either.

Here's a boundary that needs to be honestly flagged: **the Boot ROM is one of the "honest gaps" in this handbook's resource inventory.** It's present on the datasheet's functional block diagram, and boot is guaranteed to pass through it, so it's judged as "Enabled"; but the development notes (doc06/doc07) never inventoried it on its own from start to finish, and this handbook won't fabricate any on-board detail to fill that gap. Everything said above in "What This Is" comes purely from a literal translation of the official hardware Manual's 3.3; whatever the Manual doesn't say (the actual flow or version of its internal firmware, for instance) simply isn't written here (the gap-flagging convention follows the inventory standard set in 4.1/the unit-map).

### Key Capabilities & Limits

- **Capacity 128 KB** (the Manual, 3.3.1, verbatim — the only spec the Manual gives).
- **Read-only, hangs off the system bus through a 64-bit AXI3 interface** (the Manual, Figure 3.3-1).
- **Board status**: Enabled (mandatory boot path), but **never separately inventoried in the development notes at all** — this is an honest gap; this section only translates the Manual's literal text, and does not fabricate any on-board measurement data.

### When You'd Actually Use This

Let's get one mechanism fact straight first, and you'll understand why "you'll almost never operate it yourself": the Boot ROM is the SoC's factory-fixed "first thing to run at boot." After power-on, the SoC is guaranteed to execute the internal Boot ROM first, and only then, according to the boot mode selected by the board's dip switches/straps (the physical DIP switches or pin bindings that select the boot mode), load the next-stage bootloader. The point of understanding that it exists isn't "how to use it" — it's **understanding where the whole boot chain starts.**

As for what options exist for "which boot mode it hands off to," that layer is listed verbatim in the datasheet's (a separate official document, r01ds0429, not part of this Manual, r01uh1032) Table 1.3-4, "Boot":

- **CA55 boot**: mode 0 = boot from eSD, mode 1 = boot from eMMC, mode 2 = boot from serial flash hanging off the xSPI bus address space, mode 3 = boot via SCIF download.
- **CM33 boot**: mode 2 = xSPI serial flash, mode 3 = SCIF download.

From this, here's a decision rule: **you'll only run into questions at the Boot ROM layer when you're touching the RZ/V2H's boot flow itself** — take "changing the boot medium" as an example (say you want to switch from booting off the SD card to booting off eMMC — that's changing the boot mode from mode 0 to mode 1), or take "debugging a boot failure" as an example (the board powers on and stalls at a very early stage, never even reaching the bootloader — that's when you go back and think about which mode the Boot ROM handed off to, and whether that medium actually has a correct image on it). Everyday application development never needs to touch it at all. Which boot mode this particular board actually uses can be pieced together by cross-checking the on-board status in the xSPI and SDHI sections below: the small boot image sits on xSPI's NOR flash (corresponding to mode 2), and rootfs sits on SDHI0's SD card (corresponding to the eSD family under mode 0).

> **Endnote (Sources)**: The Hardware User's Manual r01uh1032ej0130 §3.3 ROM (p276, functional overview). Source for the boot-mode list: datasheet r01ds0429 Table 1.3-4 (Boot). Board status is an honest gap (not separately listed in doc06/doc07); this section is translated solely from the Manual's literal text.

---

## LPDDR4/4X Memory Controller ×2

### What This Is (the Mechanism)

This is the **external bus controller** for the two SDRAM (external main-memory chip) types LPDDR4 and LPDDR4X, with 2 channels total, supporting LPDDR4-3200 and LPDDR4X-3200, an interface bus width of 32-bit, and in-line ECC (the Manual, 3.4.1, verbatim: "This unit is an external bus controller for LPDDR4 and LPDDR4X. This unit supports LPDDR4-3200 and LPDDR4X-3200. The interface bus width is 32 bits. It supports the in-line ECC feature."). LPDDR (low-power DDR) is a low-power DRAM spec commonly used in mobile and embedded devices; "in-line ECC" means the check bits are stored interleaved with the regular data, so error correction can be done without needing a separate ECC chip.

This unit is made up of two blocks: the DDR controller block (the memory controller, abbreviated MC) and the DDR PHY block (the physical layer, abbreviated PHY) (the Manual, 3.4.1). Here's a boundary that needs to be honestly flagged: **the Manual itself states plainly that "this manual is a simplified version," and that more PHY detail has to be looked up separately in the "User's Manual Additional Document"** (the Manual, 3.4.1, verbatim: "This manual is a simplified version. For more information, refer to the User's Manual Additional Document."). So when this section talks about the PHY, it only translates the part the Manual actually has written down, and won't pretend this Manual covers every PHY detail. Separately, the actual configuration (setup code) for this block is the responsibility of this LSI's Linux software package, not something the Manual itself teaches (the Manual, 3.4.1, verbatim: "For the setup of this block, refer to the Linux software package of this LSI, which contains the DDR setup codes.") — in other words, the answer to how DDR gets initialized lives in the software package and the boot firmware, not in the register manual.

In terms of internal structure (the Manual, Figure 3.4-1, p278): 5 AXI4 ports feed into the DDR Controller — port0/1/2 are 256-bit each, port3/4 are 128-bit each; internally, the DDR Controller contains an Address Shifter, an Arbiter, a Command Queue, a Write Data Holding Queue, DRAM Command Processing, plus sub-blocks for ECC/BIST/Low Power; from there it connects to the DDR PHY through the DFI interface, and finally out to the external DRAM's physical pins. The point of having multiple AXI4 ports is that different sources — CPU, accelerators, display, and so on — can each issue memory requests over their own port, with the Arbiter deciding the order.

A few of the MC capabilities listed in the Manual's Table 3.4-1 (p277) are worth remembering: a fully pipelined command/read/write data interface; advanced bank look-ahead (opening memory banks ahead of time to improve throughput); a programmable register interface for controlling memory parameters and protocol (including auto pre-charge); complete automatic initialization at boot; Weighted Round-Robin arbitration across port requests; ECC (single-bit/double-bit error reporting, single-bit correction, programmable removal of ECC storage); and built-in self-test (BIST). On the low-power side (same table): support for multiple low-power states such as power-down, self-refresh, and I/O retention, switchable either automatically or through a software interface.

### How You See This Under Linux

**DRAM has no userspace device node** — because DRAM itself is system memory, managed directly by the Linux kernel's memory management (the mm subsystem). There's no `/dev/ddr` for you to open; the way you "use DRAM" is simply by naturally using it whenever you allocate any regular memory — malloc, mmap, loading a program.

The controller's and PHY's register blocks (on-board measurement: DDR0's MEMC sits around `0x14C00000`, DDR PHY around `0x14E00000`, quoting doc07's citation of the Manual's Table 1.8-1) are held by the boot firmware/secure world (the high-privilege execution environment under Arm TrustZone), and are **not exposed to userspace**. In other words, DDR initialization and timing calibration are finished by BL2 (the second-stage bootloader) at boot time; at runtime, Linux is only a "user" of it, not the one who "configures" it.

What you *can* check is memory usage and specs, not controller registers:

```bash
free -h              # Check current total/used/available memory (human-readable units)
cat /proc/meminfo    # Check more detailed memory statistics (MemTotal, etc.)
```

On this board, the first line of `cat /proc/meminfo` reads `MemTotal: 15565904 kB` (about 14.8 GiB — this is the amount the kernel has folded into its allocator, i.e., the figure after subtracting the boot firmware's and the various hardware accelerators' reserved regions), with `MemAvailable` at about `14863188 kB` (Source: `00_inventory.txt:90-97`; checked on the board again 2026-07-17: `MemTotal` was still consistently `15565904 kB` (transcript: live/ch04-inventory.txt); `MemFree`/`MemAvailable` fluctuate with the current load, so only `MemTotal` should be trusted as the comparison baseline). Not all 16 GB of physical memory goes to Linux — a batch of carveouts is sliced off at boot time for accelerators, display, camera, and cross-core communication, adding up item by item to about 2.34 GB; the full reservation map is in 4.1's "Memory Reservation Map," and this section won't repeat it.

As for bandwidth measurement: doc07 §48 only logs an "expected peak of near 25.6 GB/s" as a theoretical value, **explicitly marked "not measured,"** and recommends actually measuring it with a tool like tinymembench. The actual CPU-side bandwidth and latency measured has a full set of benchmark numbers in this handbook's 4.2, "Memory Bandwidth & Latency" (for example, a 4-thread STREAM Triad measured 5585 MB/s, only about 22% of the controller's theoretical peak) — not repeated here; just a reminder to **not cite the 25.6 GB/s theoretical peak as if it were a measured number**.

### Key Capabilities & Limits

- **Supported specs**: LPDDR4 (JEDEC JESD209-4D), LPDDR4X (JEDEC JESD209-4-1A) (the Manual, 3.4.1.1, verbatim).
- **DRAM interface specs (the Manual, Table 3.4-1, verbatim)**: LPDDR4 = 3200 Mbps (1600 MHz); LPDDR4X = 3200 Mbps (1600 MHz); bus width = 32-bit (16-bit per channel); rank = 1 or 2; density = **up to 8 GB per channel (byte mode not supported)**. Adding the two channels together, the theoretical capacity ceiling at the controller-spec level is 16 GB.
- **A caveat tied to the part-number suffix (the Manual, 3.4.1 Note 1, verbatim)**: "This function is supported by the devices other than "#AC0" and "#BC0". "#AC0" and "#BC0" do not support it." — meaning certain package-suffix models (#AC0, #BC0) don't support a given function. The Manual doesn't spell out on this page which function that is, so you need to check the actual part-number suffix of the unit you have — **don't assume every model supports it**.
- **On-board measurement (doc07 §48, Source: `07-hardware-unit-usage-guide.md:797-809`)**: 16 GB of LPDDR4X-3200 is fitted, with about 15.5 GB available to Linux; 25.6 GB/s is the theoretical peak bandwidth (doc07's original text explicitly marks it "not measured"); both channels are initialized by the boot firmware BL2; DDR0/DDR1's primary regions are mapped at `0x40000000` and `0x140000000` respectively (8 GB each, quoting doc07's citation of the Manual's Table 1.8-1).
- **The capacity is this board's configuration, not a spec ceiling (a 7-2 reminder)**: 16 GB is the capacity this specific board actually has soldered on and initialized, which happens to max out the controller's theoretical ceiling of "8 GB per channel, 16 GB across two channels"; solder a different, smaller (or, within spec, different) capacity onto another board, and it only affects "how much data you can store" — it **does not affect the controller's own capability** — the rate, ECC, and arbitration mechanism all stay the same. The WS125 board manual's spec table (`ws125-rdk-board-manual.md` Table 1) lists "Memory LPDDR4 1600MHz (8GB) x2," which cross-confirms doc07's measured 16 GB (8 GB×2) — but that's only "how much this particular board is fitted with"; it can't be generalized into "the RZ/V2H can only take 16 GB."

### When You'd Actually Use This

Mechanism-wise, the LPDDR4/4X controller is this SoC's **only general-purpose path to main memory**: any program running on the CA55 (Linux), CM33, or CR8, whose data volume exceeds its core's TCM (tightly-coupled memory) or SRAM capacity, ultimately accesses memory through this path. In other words, you're "using it practically all the time" — you just don't normally notice.

What actually needs your judgment is "should this piece of data go on SRAM or DDR." Decision rule:

- When your data (model weights, an image frame buffer, a regular application's heap/stack) **exceeds SRAM's 6 MB or the core's own TCM capacity**, and **doesn't require the kind of deterministic, low-latency access "independent of DDR arbitration" that SRAM offers**, it falls onto the DDR path. This covers the overwhelming majority of ordinary workloads.
- Take AI inference as an example: a large model's weights and intermediate tensors, and image frame buffers, are typical workloads that eat into this path's bandwidth and capacity. (Incidentally, the 512 MB carveout for DRP-AI3 mentioned in 4.2 and the g1 group is also sliced out of this same DDR pool, not SRAM — noted here only for comparison; its spec details belong to g1 and aren't repeated in this section.)

One capability boundary worth remembering: DDR capacity is large (16 GB on this board), but **the bandwidth the CPU side can actually pull is far below the controller's theoretical peak** (about 22% as measured in 4.2) — this isn't a malfunction, it's a consequence of the limited memory-level parallelism of a small in-order core like the A55. High-bandwidth image/tensor movement, when it's actually needed, is achieved through the accelerators' (DRP-AI3/DRP) dedicated DMA paths, bypassing the CPU entirely — so when you're planning a system, "moving large amounts of data" is something you should hand to accelerator DMA, not ask the CPU to do.

> **Endnote (Sources)**: The Hardware User's Manual r01uh1032ej0130 §3.4 LPDDR4/4X Controller (DDR) (p277-280, functional overview; registers from 3.4.2 p281); the Manual states of its own accord that the PHY's detailed specs are a simplified version, requiring a separate check of the "User's Manual Additional Document." On-board evidence: `07-hardware-unit-usage-guide.md:797-809` (doc07 §48), `00_inventory.txt:90-97`, `06-hardware-resource-map.md` (doc06 §3); CPU-side bandwidth benchmarks are in this handbook's 4.2. Other official documents: datasheet r01ds0429 (spec cross-check), the WS125 board manual `ws125-rdk-board-manual.md` Table 1 (board's as-fitted capacity); the RZ-series DRAM compatibility list r01an7912 is listed by name only, its use still to be verified, and it wasn't read in full.

---

## xSPI (NOR Flash Interface)

### What This Is (the Mechanism)

xSPI (Expanded Serial Peripheral Interface) is an interface protocol designed for memory devices, aiming for high data throughput and a low signal-pin count, while keeping limited backward compatibility with legacy SPI devices; its electrical interface can theoretically deliver up to 200 MB per second of raw data throughput (the Manual, 7.2.1, verbatim: "The xSPI protocol specifies the interface for Memory Devices, which provides high data throughput, low signal count, and limited backward compatibility with legacy SPI devices. The electrical interface can deliver up to 200 MB per second raw data throughput."). What it typically connects to is NOR flash — a type of flash that's non-volatile (data survives power loss) and byte-addressable (you can read any given address directly, the way you would with memory).

In terms of internal blocks (the Manual, Figure 7.2-1, p2502): data comes in from the internal peripheral bus, passes through the FIFO, Bridge Control Channel 0/Channel 1, Command Control, and Link Control, and finally goes out through External signals (the external pins); a separate Register Control block handles configuration.

It supports several protocol modes (the Manual, Table 7.2-1, enumerated verbatim), differing in how many pins are used and whether it's SDR or DDR (single/double data rate — transferring data once or twice per clock edge):

- **1/4/8-pin, SDR or DDR**: `1S-1S-1S`, `4S-4D-4D`*1, `8D-8D-8D`*1 (Note 1: DDR access isn't supported when the XSPI0_DS signal isn't connected).
- **2/4-pin, SDR**: `1S-2S-2S`, `2S-2S-2S`, `1S-4S-4S`, `4S-4S-4S`.
- It also supports a configurable address length, a configurable initial access latency period, and **XiP** (execute-in-place — code doesn't need to be copied into RAM first; the CPU executes directly from flash).

On the feature side (same table), a few more things are worth knowing: Write Data Mask is supported; In-band Reset is supported (compliant with the JESD252 standard); memory-mapping (mapping flash contents onto an address range, so flash can be read the way regular memory is) reaches up to a 256 MB address space (128 MB per CS [chip select], or up to 256 MB on CS0 alone); a prefetch feature lowers burst-read latency; an outstanding buffer improves burst-write throughput; manual command supports configuring up to 4 sets of commands and status register polling; Input Strobe port timing shift is also supported. The whole unit has only 1 channel, and can act as a master initiating transactions against up to 2 slaves, with 2 interrupt sources.

On address mapping (the Manual, Table 7.2-3, p2503) — and these are **default values, not hardwired ones**: CS0's internal address is `0_2000_0000h` (seen from the CM33: non-secure = `7000_0000h` / secure = `6000_0000h`), and CS1 is `0_2800_0000h` (`7800_0000h`/`6800_0000h`); the table's footnote states plainly that these can be changed through registers such as `SYS_SPI_STAADDCS1` and `SYS_SPI_ENDADDCS0-1`.

### How You See This Under Linux

This board uses xSPI for boot-time NOR flash; the path on the Linux side is: the `xspi-if` driver → the MTD subsystem → `/dev/mtd0..mtd3` (with corresponding `/dev/mtdblock*` nodes too). MTD (Memory Technology Device) is Linux's abstraction layer for "erasable, block-organized" storage devices like flash, distinct from the block-device abstraction used for ordinary hard disks/SD cards.

The controller's register base address is `0x11030000` (device tree node name `xspi@11030000`); the memory-mapped read region is 256 MB, mapped at `0x10000000` (doc07, quoting the Manual's Table 1.8-1). This corresponds to the datasheet's boot mode 2 (serial flash on xSPI) — in other words, this is the flash this board's boot process draws from.

This board's NOR flash is sliced into four MTD partitions, each with a clear purpose (partition names verbatim, confirmed by re-running `cat /proc/mtd` on the board 2026-07-17, transcript: live/ch04-inventory.txt; Source: `07-hardware-unit-usage-guide.md:769-781`, `04-hardware-quickref.md:278`):

```text
dev:    size   erasesize  name
mtd0: 0001d200 00001000 "bl2"
mtd1: 001c2e00 00001000 "fip"
mtd2: 00020000 00001000 "env"
mtd3: 00e00000 00001000 "test-area"
```

`mtd0` = bl2 (the second-stage bootloader), `mtd1` = fip (the firmware image package), `mtd2` = env (U-Boot environment variables), `mtd3` = test-area. The tools for querying and flashing are mtd-utils (`flash_erase`, `flashcp`, `mtdinfo`):

```bash
cat /proc/mtd                          # List all MTD partitions
mtdinfo /dev/mtd2                      # View details of a given partition
sudo flash_erase /dev/mtd2 0 0         # Erase the entire env partition
sudo flashcp -v new-env.bin /dev/mtd2  # Write a new image into the env partition
```

> ⚠️ **Note (don't run flashing experiments on the boot-firmware partitions)**:
> - **Situation**: you want to use `flash_erase`/`flashcp` to run a write test against the xSPI NOR flash's MTD partitions.
> - **Symptom**: if you touch `mtd0`/`mtd1`, the board may fail to boot next time.
> - **Cause**: `mtd0` = bl2 and `mtd1` = fip are the firmware images used for booting — erase them and there's nothing left to boot from.
> - **Prevention/Handling**: for write testing, only touch `mtd2` (env) or `mtd3` (test-area), and **never touch `mtd0`/`mtd1`** (Source: `07-hardware-unit-usage-guide.md:769-781`).

### Key Capabilities & Limits

- **Electrical-interface theoretical throughput ceiling: 200 MB/s** (the Manual, 7.2.1, verbatim).
- **Address space: up to 256 MB** (128 MB per CS, or 256 MB on CS0 alone) (the Manual, Table 7.2-1).
- **This board's actual status (doc07 §46, Source: `07-hardware-unit-usage-guide.md:769-781`)**: `xspi@11030000` serves as the boot-time NOR flash, exposed as MTD partitions (mtd0=bl2/mtd1=fip/mtd2=env/mtd3=test-area), corresponding to datasheet boot mode 2.
- **The flash chip soldered onto this board is 64 MB — not a spec ceiling (a 7-2 reminder)**: the WS125 board manual's Table 1 states verbatim "QSPI Flash ROM 64MB," meaning the flash chip soldered onto this particular board is 64 MB, **far smaller than** the controller's spec ceiling of 256 MB. This is this board's specific configuration, not the controller's capacity ceiling — different boards may solder on flash of different capacities, and the actual usable space depends on which chip is soldered on. What you should be looking at when selecting parts is the protocol and addressing ceiling the controller supports (200 MB/s, 256 MB), not how large a flash chip some particular board happens to have soldered on.

### When You'd Actually Use This

Mechanism first: the NOR flash that xSPI connects to has three characteristics that determine its use — **byte-addressable** (any address can be read directly, the way memory is), **XiP-capable** (code can run without being copied into RAM first), and **capacity that's typically MB-scale, but with a stable boot path**. This is exactly the mechanism-level reason it's listed as the serial flash boot mode (boot mode 2): the very earliest boot stage needs a block of non-volatile memory that "can be reliably read, even executed directly, the instant power comes on" — and these characteristics of NOR flash line up precisely with that need.

From this, here's a decision rule to help you decide "should this batch of data go on NOR flash or somewhere else":

- **Well-suited to xSPI/NOR**: scenarios needing "boot-time use, small capacity, high reliability, code that can execute directly without being copied" — bootloaders, a small amount of configuration data (U-Boot environment variables, for example).
- **Not suited to xSPI/NOR**: scenarios needing GB-scale bulk data storage — take the OS root filesystem, data logging, or model files as examples — these should use a block device like SDHI/eMMC (see the next section), not xSPI/NOR.

This board reserves xSPI for small boot images like `bl2`/`fip`/`env` and puts rootfs on the SD card — this reflects exactly the fundamental difference between "NOR flash" and "block device," not some clever trick unique to this board. Any RZ/V2H application that follows the same division of labor — "small boot image vs. bulk data storage" — will arrive at a similar interface choice.

> **Endnote (Sources)**: The Hardware User's Manual r01uh1032ej0130 §7.2 Expanded SPI (xSPI) (p2501-2503, functional overview; registers from 7.2.2 p2504). On-board evidence: `07-hardware-unit-usage-guide.md:769-781` (doc07 §46), `04-hardware-quickref.md:278`, `06-hardware-resource-map.md` (doc06 §2); the verbatim MTD output was confirmed by re-running on the board 2026-07-17. Other official documents: the WS125 board manual `ws125-rdk-board-manual.md` Table 1 (the board's flash chip capacity); the mtd-utils toolset. Source for the boot-mode cross-reference: datasheet r01ds0429 Table 1.3-4.

---

## SDHI/eMMC/SDIO Host ×3

### What This Is (the Mechanism)

This is a 3-channel SD/MMC host interface (host interface — meaning a controller where the SoC side actively initiates transactions to read and write an externally connected SD card or eMMC). The three channels' capabilities aren't equal: **channel 0 supports SD and e-MMC**; **channel 1 and channel 2 support SD only** (the Manual, 6.2.1.1, verbatim: "Channel 0 supports SD and e-MMC. Channel 1 supports SD. Channel 2 supports SD.").

At the start of this chapter, the Manual has a CAUTION box, verbatim: "Development of the SD host-related products needs the conclusion of the following agreement. • "SD Host/Ancillary Product License Agreement (SD HALA)"" — this is Renesas's reminder about the licensing agreement attached to "developing SD host-related products." In translating it, it's enough to honestly flag that this licensing precondition exists; this section won't go into the process details.

On the Features side, the Manual's 6.2.1.1 lists a long string of items verbatim; picking out the ones that matter to you: an SD memory/IO card interface (1-bit/4-bit SD bus); support for SD, SDHC, and SDXC memory card access; compliance with SD specification version 3.01; support for Default, high-speed, UHS-I/SDR50, SDR104, and DDR50 transfer modes; an SD clock (SD_CLK) frequency of `SDHI_x_IMCLK frequency / 2ⁿ` (n = 0 to 9, x = 0 to 2 being the channel number); error checking via CRC7 (command/response) and CRC16 (data); 2 interrupt requests; support for card detection and write protection. On the MMC side: an MMC interface (1-/4-/8-bit MMC bus); support for e-MMC device access; support for Backward-compatible, high-speed, HS-DDR, and HS200 transfer modes; support for High-priority interrupt (HPI); and support for SDIO 3.0.

Internal blocks (the Manual, Figure 6.2-1, p1517): the AXI bus connects to an AXI master I/F and an AXI slave I/F respectively; the master I/F connects through a DMAC to the Host I/F, which then connects to the SD/MMC I/F and out to the physical SD/MMC bus; the Host I/F and SD/MMC I/F are each paired with their own RAM buffer; there's also a separate interrupt-request output. The register base addresses of the three controllers (the Manual, Table 6.2-2, verbatim): SD0 = `0_15C0_0000h` (CM33 non-secure `55C0_0000h` / secure `45C0_0000h`), SD1 = `0_15C1_0000h`, SD2 = `0_15C2_0000h`.

### How You See This Under Linux

This board uses SDHI channel 0 to carry rootfs (the root filesystem). The path on the Linux side is: the `renesas_sdhi`/`tmio_mmc` drivers → the block-device node `/dev/mmcblk0` (plus partitions `mmcblk0p1..`). If an SDIO card is inserted instead, it mounts over the same mmc/sdio bus as well.

This board's actual status (doc07 §47, Source: `07-hardware-unit-usage-guide.md:783-795`): `SDHI0 @0x15C00000` carries rootfs on `/dev/mmcblk0` (a 256 GB Samsung SD card); the other two eMMC controllers, `@0x15C10000` and `@0x15C20000`, are disabled in the device tree — because this particular board doesn't have physical eMMC populated (soldered on/plugged in). So this handbook's resource map marks this unit's overall status as "Partially Enabled": SDHI0 is alive, the two eMMC controllers are switched off (spot-checked on the board 2026-07-18: `mmc@15c10000`/`mmc@15c20000` were both `disabled`, transcript: live/ch04b-dt-status.txt).

The tools for querying and operating on it are util-linux (`lsblk`, `fdisk`), e2fsprogs, and mmc-utils:

```bash
lsblk                                # View the block-device and partition tree
sudo fdisk -l /dev/mmcblk0           # View this card's partition table
cat /sys/block/mmcblk0/device/name   # Read the product name out of the card's CID
mmc extcsd read /dev/mmcblk0         # Mainly for eMMC; on an SD card it mostly just shows card info
```

This board's boot card's identity (Source: `05-compute-benchmark.md:48`, `04b_emmc_id.txt:2-8`): type SD, manfid `0x1b`, oemid `"SM"`, manufactured 2025/10, ext4-formatted, about 206 GB usable. Its speed sits at the UHS-I SD card tier, not the 150-300 MB/s of eMMC 5.1. The full storage I/O benchmarks (sequential read/write, fio random read/write, etc.) are in this handbook's 4.2, "Storage I/O," and aren't repeated here.

> ⚠️ **Note (the script labels the boot card as eMMC; it's actually an SD card)**:
> - **Situation**: you want to confirm exactly what device the boot storage is.
> - **Symptom**: the inventory script's output file `04_storage.txt` has a header labeling it "eMMC," but the card's reported identity is actually an **SD card** (the script's label doesn't match the actual device identity) (Source: `05-compute-benchmark.md:191`, `04_storage.txt:1`).
> - **Cause**: this board ships with an external SD card as its boot medium; the script's label carried over a generic term that doesn't match the card actually inserted — you should trust the identity the card reports about itself (`/sys/block/mmcblk0/device/type`, etc.) instead.
> - **Prevention/Handling**: to determine what the boot storage actually is, don't trust the script's filenames — read the card's CID/type instead; to confirm its speed tier, look at which transfer mode it's running in (UHS-I/SDR104/DDR50), not its capacity.

### Key Capabilities & Limits

- **3 channels, with unequal capability division**: channel 0 = SD+eMMC, channel 1/channel 2 = SD only (the Manual, 6.2.1.1).
- **Compliant with SD specification version 3.01, supporting up to the SDXC tier**, with transfer modes covering UHS-I/SDR50/SDR104/DDR50 (SD side) and HS-DDR/HS200 (eMMC side) (the Manual, 6.2.1.1).
- **This board's status (doc07 §47)**: SDHI0 carries rootfs on `/dev/mmcblk0` (a 256 GB Samsung SD card); the two eMMC controllers are DT-disabled (this board has no physical eMMC populated).
- **The card capacity is this board's configuration, not a controller ceiling (a 7-2 reminder)**: 256 GB is the capacity of the SD card actually inserted in this particular board, not the SDHI controller's capacity ceiling — the controller itself supports up to the SDXC spec tier (a theoretical ceiling far above 256 GB). A larger- or smaller-capacity card only affects "how much data can be stored" — it **does not affect the transfer-speed ceiling** — the speed ceiling is set by the transfer mode (UHS-I/SDR104/DDR50, etc.), unrelated to card capacity.
- **The card recommended by the original board manufacturer (the WS125 board manual, §3.9, verbatim)**: "The RDK has micro SD card connectors (SD1). SD1 is connected to the SD0 interface of RZ/V2H and can be used as a boot device. The power supply VDD1833_SD0 of RZ/V2H is fixed to 1.8V for high-speed data transfer. Renesas recommends using the built-in SanDisk 64GB SD card for high-speed data transfer." — this is the card the original board ships with and recommends using. Swapping to a different batch, or a different card of your own choosing, is fine, as long as it supports the UHS-I spec to reach high-speed mode; the "64 GB" is only the factory shipping recommendation, and likewise isn't a spec ceiling.

### When You'd Actually Use This

Mechanism first: SDHI/eMMC follows the "**high-capacity block storage, operable through a standard filesystem**" path — data is organized in blocks, and once a filesystem is mounted on it, it can be mounted/copied/formatted just like an ordinary hard disk. This is fundamentally different in nature from the byte-addressable, directly-executable path that xSPI/NOR follows.

From this, here's a decision rule:

- **What falls on the SDHI path**: whenever an application needs GB-scale data (OS image files, log files, or datasets, for example) and needs standard filesystem operations (mount, copy, format), use SDHI/SD/eMMC.
- **What shouldn't use this path**: small images that need to be reliably read, or even executed in place, right at boot time (bootloader, environment variables) — those should go through xSPI/NOR (see the previous section).

How to choose between SD and eMMC is an **application-level trade-off**, not something the controller dictates. This board's choice of an SD card (SDHI0) as the boot+rootfs device, corresponding to datasheet Table 1.3-4's boot mode 0 (booting from eSD), is one concrete instance; switching to eMMC (boot mode 1) would be another instance of the same interface family. The trade-off between the two comes down to this: an SD card is removable, which is convenient for mass-production flashing and updates; eMMC is soldered directly onto the board, and is less prone to vibration causing poor contact or a card working loose. So — **for applications affected by vibration (a mobile vehicle, for example)**, eMMC is often preferred for its mechanical robustness; **for applications not affected by vibration (a fixed desktop-style system, for example)**, the removability and easy updating of an SD card may matter more. This is an application-level judgment call, and readers should decide it based on their own use case, rather than copying whichever one some particular board happened to use.

> ⚠️ **Note (don't stake high-reliability, power-loss-safe data logging directly on a consumer-grade SD card)**:
> - **Situation**: you want to use this same boot SD card as a high-reliability data-logging device as well (something like a flight black box, where the data needs to stay intact even after power is lost, as an example).
> - **Symptom**: under sustained high-frequency writes, a consumer-grade SD card's write endurance and power-loss (sudden power cut) reliability become a real risk.
> - **Cause**: what ships is a consumer-grade external SD card, and its write endurance and power-loss data integrity are a genuine weakness for a need like "high-reliability data logging" (Source: `05-compute-benchmark.md:196-198`).
> - **Prevention/Handling**: needs like this should switch to on-board eMMC or an industrial-grade pSLC card, and separate high-frequency logging from rootfs onto different storage devices so they don't drag each other down. As for ordinary workloads — take image-plus-sensor data logging as an example, where a 4 Mbps video stream is about 0.5 MB/s — this card's speed is more than enough (Source: `05-compute-benchmark.md:128-131`); whether to upgrade depends on your reliability requirements, not your speed requirements.

> **Endnote (Sources)**: The Hardware User's Manual r01uh1032ej0130 §6.2 SD/MMC Host Interface (SD) (p1516-1521, functional overview; registers from 6.2.2). On-board evidence: `07-hardware-unit-usage-guide.md:783-795` (doc07 §47), `05-compute-benchmark.md:48,191,196-198`, `04b_emmc_id.txt:2-8`, `04_storage.txt:1`; storage benchmarks are in this handbook's 4.2; the `mmc@15c10000`/`15c20000` disabled status was spot-checked on the board 2026-07-18. Other official documents: the WS125 board manual `ws125-rdk-board-manual.md` §3.9 (SD card connector and recommended card); boot-mode cross-reference: datasheet r01ds0429 Table 1.3-4; tools: util-linux/e2fsprogs/mmc-utils.
