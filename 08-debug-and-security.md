# 08 · Debug & Security

This group covers three hardware subsystems on the RZ/V2H that you'd normally never touch — but that can save your life at the critical moment: the **chip-embedded debug/trace subsystem (CoreSight)**, **execution-environment isolation and memory-access control (TrustZone + 9 TZC-400 units)**, and the **security module (Trusted Secure IP encryption engine + OTP/Device-ID/JTAG-disable)**.

This group is also the one most likely to make people think the board is "missing a feature." The reason: most of it isn't the kind of thing that's enabled, has a driver bound, and shows up under `ls /dev` — most of it falls into the resource map's six-state taxonomy as either **Present · Not Exposed to Linux** or **Not Populated**. Once you know the real status of each one, you won't waste time digging through Linux for a node that was never going to show up there in the first place. This group has exactly one general rule: **almost none of these three systems are operated through Linux userspace commands**. CoreSight relies on an external probe, TrustZone relies on boot firmware, and the encryption engine is out of reach from this board's Linux (this part number's Security field = N/A, verified first-hand in the hardware manual Table 1.1-1 [p78]; the module is not populated in the silicon). So for this group, "can't find it" is often the **correct state**, not a malfunction.

(One boundary to draw first: **the first tool you reach for in everyday debugging is not in this group** — it's the UART serial console on CN8 (FT234XD → SCIF0, 115200 8N1), which carries the boot log, the scene of a kernel crash, and rescue logins; the full walkthrough is in Chapter 01 §1.4. The CoreSight/JTAG hardware in this group is a deeper, chip-level kind of debugging that needs an external probe, and most readers will never need it.)

> **The on-board hardware for this group is this part number**: R9A09G057H44GBG (carrier board manual, component list on Page 11, U1; H44 = the RZ/V2HP variant). Keep one key trade-off about this part number in mind from the start — **on the board, Linux has no hardware crypto interface of any kind** (no `/dev/tee`, no hwrng node backed by a Renesas TRNG [true random number generator], and no ARMv8 Crypto Extension in the CA55 instruction set). As to whether this part number carries the Trusted Secure IP hardware at all, **the hardware manual `r01uh1032` §1.1.2 Product Lineup Table 1.1-1 [p78] answers it**. That SKU (part-number) lineup table marks R9A09G057H44GBG's Security column verbatim as **N/A**, and it is the very same first-hand PDF table that 00/02 use to establish this part number's ISP = Available [Mali-C55]. It can be verified word for word with pypdf and is a first-hand source, so this part number is **not populated** with it. (The comparable lineup table in the datasheet `r01ds0429` is a degraded conversion, but with the hardware manual providing a clean SKU basis, that no longer blocks anything.) This trade-off runs through all of 〈Unit 3〉, and it's the single most commonly misunderstood thing about this group.

## Units in This Group

| Unit | One-liner | Board status |
|---|---|---|
| [CoreSight (Chip-Embedded Debug and Trace Subsystem)](#unit-1-coresight-chip-embedded-debug-and-trace-subsystem) | Arm's debug/trace hardware: view CPU execution state, set hardware breakpoints, and record program flow directly from outside the chip, with no dependency on the OS | Present · **no Linux debug/trace nodes**; only reachable via an external probe over JTAG/SWD |
| [TrustZone (incl. 9× TZC-400)](#unit-2-trustzone-incl-9-tzc-400-address-space-control) | CA55's secure/normal-world isolation + 9 bus access controllers that admit or block memory and peripheral access based on "which world it came from" | Hardware present · **no secure-OS runtime** (no `/dev/tee`, no OP-TEE); TZC regions are configured by TF-A at boot |
| [Security IP (Encryption Engine and OTP/Device-ID/JTAG-disable)](#unit-3-security-ip-encryption-engine-and-otpdevice-idjtag-disable) | An optional hardware security module (encryption/decryption, true random numbers, keys that never leave the chip) + the one-time-programmable memory that's physically fixed in the silicon regardless | **Mixed**: on the board, Linux has no encryption engine/TRNG/`/dev/tee` interface and the CA55 has no crypto extension (measured on the board); this part number's Security field = N/A, so the Trusted Secure IP is **not populated** (verified first-hand in the hardware manual Table 1.1-1 [p78], the same table that establishes ISP = Available); OTP/Device-ID/JTAG-disable are physically present in silicon but have no Linux runtime |

**One shared concept for this whole group, stated once up front**: all three of these hardware blocks are tied to "security boundaries" and the "boot/firmware layer," so the way they're exposed is fundamentally different from earlier groups (camera, audio, serial ports — peripherals that "have a driver bound the moment the board boots"). Each of them is either held entirely by boot firmware (TF-A [Trusted Firmware-A]/u-boot), only reachable through a tool outside Linux (a JTAG probe), or simply not populated on this part number at all. As you read through each unit below, the "How You See This Under Linux" section will mostly tell you "you can't see it, and that's expected" — the important part is that it then tells you **where to look instead**.

- [Unit 1: CoreSight (Chip-Embedded Debug and Trace Subsystem)](#unit-1-coresight-chip-embedded-debug-and-trace-subsystem)
- [Unit 2: TrustZone (incl. 9× TZC-400 Address Space Control)](#unit-2-trustzone-incl-9-tzc-400-address-space-control)
- [Unit 3: Security IP (Encryption Engine and OTP/Device-ID/JTAG-disable)](#unit-3-security-ip-encryption-engine-and-otpdevice-idjtag-disable)
- [Hands-On: Temperature Monitoring and Thermal Safety Margin (with the Chip's TSU as the Example)](#hands-on-temperature-monitoring-and-thermal-safety-margin-with-the-chips-tsu-as-the-example)
- [This Group's Evidence Boundary & Unverified Items](#this-groups-evidence-boundary--unverified-items)

---

## Unit 1: CoreSight (Chip-Embedded Debug and Trace Subsystem)

### What This Is (the Mechanism)

This SoC has a full Arm **CoreSight** debug subsystem built in. What it does can be summed up in one line: **view the CPU's internal execution state, set breakpoints, and capture program flow directly from outside the chip, with no dependency on the operating system**. The Manual's own wording is blunt about it: "This LSI has a debug interface for boundary scan function and debug support for CA55, CM33, and CR8." — meaning this debug interface serves all three core types at once: the four application cores CA55, the system management core CM33, and the two real-time cores CR8 (the Manual 4.9.1 Overview).

This debug interface multiplexes **two communication protocols** onto the same set of physical pins: **JTAG** (IEEE 1149.1 boundary-scan standard) and **SWD** (Serial Wire Debug). The shared pins are `TCK_SWCLK`/`TMS_SWDIO`/`TDI`/`TDO`/`TRSTN` (the Manual 4.9.1.1 Features; 4.9.1.3 Table 4.9-1). The difference between the two protocols is a pin-count-versus-speed trade-off (JTAG uses more pins, SWD needs only two wires), but what matters for our purposes is this: **both of them travel over this same external, off-chip debug channel — neither has anything to do with Linux**. The core of the debug channel is the **DAP** (Debug Access Port), which can perform "direct control of IP without going through the CPU by switching Access Port (AP)" (the Manual 4.9.1.1: "Direct control of IP without going through CPU by switching Access Port (AP)"). That is exactly why an external probe can still read and write inside the chip even while the system is mid-run, or even when the CPU is completely locked up.

The second capability is **Trace** (program-flow tracing): this isn't "pause and take a look" — it's "continuously record the program-counter history of everything each core has run." The three core types CA55, CM33, and CR8 each have their own **ETM** (Embedded Trace Macrocell) generating trace data. This data is merged through a **Trace Funnel** (a multiplexer), buffered inside the chip's built-in **ETF** (Embedded Trace FIFO), or written directly to the system bus/memory via the **ETR** (Embedded Trace Router) (the Manual 4.9.1.2, Fig 4.9-1, Fig 4.9-2). The trace data path roughly looks like this:

```text
  CA55 ×4 ─┐
  CM33    ─┤ Each core's ETM ──► Trace Funnel ──► ETF (on-chip FIFO) ──► read out by external probe
  CR8 ×2  ─┘  (trace generator)   (multiplexer)     ETF0 32KB / ETF1 16KB        or
                                                    ETF2 8KB  / ETF3 4KB     ETR ──► system bus/memory
  GIC-600 ─── only connects to Cross Trigger (interrupt controller, no trace output)
```

The third capability is **mutual triggering**: the **CTM** (Cross Trigger Matrix) lets CPU cores and debug components link up events with each other — for example, when one core hits a breakpoint, it can trigger another core to pause at the same time. It can also link to SYC (system counter), WDT (watchdog timer), and GTM (general timer) (the Manual 4.9.1.1: "Interlocking operation of CA55, CM33, CR8, system counter (SYC), watchdog timer (WDT), general timer (GTM), and debug component by cross trigger"). There's also a separate, independent-purpose **Boundary Scan**: what it tests isn't the CPU, but "whether the wiring between this chip and the other ICs on the circuit board is correctly connected" — it's for production testing, entered via switching the `BSCANP` pin (the Manual 4.9.1.3, Table 4.9-4, 4.9-5).

One last key mechanism, and it directly determines whether you'll be able to debug at all: **debug functionality isn't always on**. There's a boot pin, **MD_BOOT3**, whose value has to be decided "before the power-on reset (PRST#) is released" — `0` = normal operation (the entire debug function module sits in reset and can't be used after boot), `1` = debug operation (debug functionality is available after boot). The Manual states plainly that this value "must be fixed before releasing the power-on reset (PRST#)," and **it cannot be switched dynamically after boot** (the Manual 4.9.1.3 Table 4.9-2, 4.9-3). In other words, whether JTAG/SWD will be usable is partly already decided the instant the hardware boots.

### How You See This Under Linux

**Short version: you won't see it under Linux, and that's normal.** doc07 §15's board-status column puts it plainly: "PRESENT in silicon, but NO Linux debug/trace nodes on this board" — the subsystem is there in the silicon, but this board's Linux side has nothing bound to it: `no /sys/bus/coresight devices, no /dev/cpu/*/etm`. This is a textbook example of **Present · Not Exposed to Linux** from the resource map's six-state taxonomy.

To verify that "this board has no self-hosted CoreSight," run this line (getting "not found" back is this board's expected state):

```bash
ls /sys/bus/coresight/devices 2>/dev/null || echo 'no coresight nodes'
```

(Source: doc07 §15.)

So how would you "turn it on"? If you want to enable self-hosted trace on the Linux side (having Linux itself drive CoreSight capture), you need the kernel build option `CONFIG_CORESIGHT` **plus** the corresponding graph nodes in the device tree — and this board's shipped device tree doesn't configure either of those (doc07 §15). That's extra system-configuration work; it's not there out of the box.

In real-world practice, though, the path people actually use to work with CoreSight **completely bypasses Linux**: **an external debug probe** (JTAG or SWD) paired with industry tooling, accessing the chip directly over the debug APB bus. A typical approach is to connect in with an open-source tool like OpenOCD:

```bash
openocd -f interface/jlink.cfg -c 'transport select swd' -f target/renesas_rzv2h.cfg
# in another terminal, connect to OpenOCD's command port
telnet localhost 4444
# once connected you can issue halt (pause the CPU) / reg (read registers) / mdw <address> (read a memory word), etc.
```

(Source: doc07 §15 usage example. `<address>` is a placeholder for the memory address you want to read.) Besides OpenOCD, commercial tools include Arm Development Studio and Lauterbach TRACE32. These tools run on your PC and connect to the board's debug pins through probe hardware — **completely independent of whether the Linux kernel on the board is even alive**. This is exactly the biggest payoff of external-probe debugging: whether Linux has crashed, hasn't booted yet, or what you actually need to look at is a bare-metal core that Linux has no control over in the first place (CR8/CM33 firmware), the probe still works.

### Key Capabilities & Limits

- **Trace buffer capacity totals 60 KB across four units**: ETF0 32 KB (the main entry point), ETF1 16 KB, ETF2 8 KB, ETF3 4 KB (the Manual 4.9.1.2 Fig 4.9-2, labels on the diagram, read verbatim). This 60 KB is the ceiling on trace depth you can capture when "not using an external ETR, relying only on the on-chip FIFOs." (Side note: this handbook's 4.2, 〈Compute Units〉, mentions a "60 KB CoreSight ETF trace buffer" when discussing CM33 — that's the sum of these same four ETFs. It's a buffer shared by the whole trace subsystem, not something exclusive to CM33.)
- **Trace covers three CPU types, not the interrupt controller**: CA55 (4 cores), CM33, and CR8 (2 cores) each have an independent ETM trace output; GIC-600 (the interrupt controller) only connects to Cross Trigger and has **no** trace output (the Manual 4.9.1.2 Fig 4.9-1).
- **Debug address space is 4 MB, base `0x1F000000`** (the Manual 4.9 Debug Interface; this handbook's 4.4 document lookup table also lists this address). The internal modules sit at these addresses: ROM table `0x1F000000`, Timestamp gen `0x1F010000`, ETF0 `0x1F020000`, ETF1 `0x1F030000`, ETF2 `0x1F040000`, ETF3 `0x1F050000`, ETR `0x1F060000`, Trace Funnel0–3 `0x1F070000`–`0x1F0A0000`, CTI0 `0x1F0B0000`. **This address breakdown is transcribed from doc07 §15**, which attributes its own source to the Manual 4.9.2 — register-detail pages that fall outside the p1119–1123 functional-overview range this group read directly. **This group's notes have not directly verified the original pages and are only quoting doc07's existing write-up secondhand**, so keep this boundary in mind when citing it.
- **Register-level detail requires a separate Arm document**: the operational register detail for ETM/ETR/ETF is in Arm's official "CoreSight Trace Memory Controller Technical Reference Manual" — this SoC's Manual only covers how it's integrated (the Manual 4.9.1.2, end of section). **This Arm document is not in this handbook's list of available sources and has not been verified.**
- **JTAG can be permanently disabled via OTP**: an OTP fuse (write-once, irreversible once burned) can permanently turn off JTAG — this "irreversible" restriction matters, and the detail is under JTAG-disable in 〈Unit 3: Security IP〉.

### When You'd Actually Use This

**Decision rule (first work out whether your need falls into "the operating system can't reach this" territory)**: as long as what you need is to "inspect CPU registers/memory/interrupt state, set hardware breakpoints, single-step, or continuously record program-counter history, **without depending on the operating system**, while the system is mid-run," that falls under bring-up debugging (the process of getting hardware from power-on to basic working order) or real-time system verification. Needs like these all have to go through a JTAG/SWD external probe, because this board's Linux side has no CoreSight bound and no Linux command can touch it. This rule holds **regardless of application domain**: take flight-control firmware development for a mobile vehicle, or real-time control firmware development for industrial machinery, as examples. As long as you're developing bare-metal firmware that runs on CR8/CM33, the moment you hit "the core is stuck, Linux hasn't even come up yet, not even a printf will print," an external probe is often the only tool you have left to ask "where exactly is it stopped right now."

**Conversely, when you don't need it**: if all you want is to observe the behavior of an ordinary Linux application (say, profiling a userspace program's performance), OS-level tools like `ftrace`/`perf`/`gdbserver` are enough — no need to touch CoreSight at all. CoreSight's home turf is "Linux hasn't booted yet" or "bypassing Linux entirely to look at bare-metal core state" — for anything whose goal lives in userspace, don't bring out a sledgehammer to crack a nut.

**If what you want is trace (continuous capture) rather than breakpoint single-stepping**: the prerequisite is turning on `CONFIG_CORESIGHT` in the Linux kernel config and adding the corresponding device tree graph nodes — this is extra system-configuration work that this board's factory configuration doesn't include out of the box. The decision rule is: do you need to "replay the entire execution trace after the fact" (trace), or "stop and inspect right now" (breakpoint)? The former costs you the configuration work above; the latter just needs an external probe.

> **Endnote (Sources)**: the official hardware manual `r01uh1032` §4.9 Debug Interface (p1119–1123, functional overview; register details from 4.9.2 onward, not read). The debug address breakdown is quoted secondhand from doc07 §15 (original pages not directly verified). External tools: OpenOCD, Arm Development Studio, Lauterbach TRACE32. Register-operation details are also covered in Arm's "CoreSight Trace Memory Controller Technical Reference Manual" (not in this handbook's list of available sources, not verified).

---

## Unit 2: TrustZone (incl. 9× TZC-400 Address Space Control)

### What This Is (the Mechanism)

**TrustZone** is Arm's CPU mechanism for "execution-environment isolation": it splits CA55's execution state into two worlds, **Secure** and **Non-secure** (the "normal" world). But CPU-mode separation alone isn't enough — if code in the normal world can still directly read and write secure-world memory, the isolation is worthless in practice. So memory and peripherals also need to be able to admit or block a given access based on "which world this access came from," and that job belongs to **TZC-400** (CoreLink TrustZone Address Space Controller, an Arm-licensed address-space-controller IP). The Manual's own wording: "This LSI has nine address space controllers (TZCs) to realize memory access in a safe area. It performs security checks on transactions to memory or peripherals. Transactions must meet security requirements to access memory or peripherals." (the Manual 3.5.1 Overview)

This SoC has **9** TZC-400 instances installed in total, each sitting in front of a different bus as a "checkpoint" that filters access heading toward its block of memory/peripherals (the Manual 3.5.1.1 Features, 3.5.1.2 Fig 3.5-1 block diagram, 3.5.1.3 bulleted text, cross-checked one by one):

| TZC-400 Instance | What It Guards | Filter Unit Count & Assignment |
|---|---|---|
| `TZC400_XSPI` | xSPI (external flash interface) | 1 |
| `TZC400_SRAMM` | Internal SRAM0, SRAM1 | 2 (1 each) |
| `TZC400_SRAMA` | Internal SRAM2 | 1 |
| `TZC400_AXI_RCPU` | RCPU Bus (internal bus for real-time processing) | 1 |
| `TZC400_DDR00` | DDR0 memory | 3 (admits Video0/Video1/DRP Bus respectively) |
| `TZC400_DDR01` | DDR0 memory | 2 (admits COM Bus/ACPU Bus) |
| `TZC400_DDR10` | DDR1 memory | 3 (same as DDR00: Video0/Video1/DRP) |
| `TZC400_DDR11` | DDR1 memory | 2 (same as DDR01: COM/ACPU) |
| `TZC400_PCIE` | PCIe | 2 (corresponding to the two controllers PCIE0/PCIE1) |

Each TZC can carve up to "**8 secure regions** plus **1 default region** covering everything else (the default base region)" out of the address space it manages, and each region's access permissions can be set independently by software over the APB interface (the Manual 3.5.1.1: "The ability to define up to eight address regions in the area map." "A default base region to cover all remaining portions of the address map." "Software programmable security access permissions for each address region through an APB interface."). The component actually doing the gatekeeping is the **filter unit**: a data transfer is only admitted when "the security state and identity of an ACE-Lite bus transaction match the security setting of the memory region it's accessing." All the filter units under one TZC share a single set of region-configuration registers, to guarantee consistency (the Manual 3.5.1.1, 3.5.1.3: "All filter units operate from one set of shared region configuration registers. This ensures consistency across all filter units.").

A few more properties worth remembering: it supports **identity-based filtering** on Non-secure accesses, and supports up to **256 outstanding transactions** (in-flight at once) (the Manual 3.5.1.1, last two bullets). Access violations can be configured to raise an interrupt; each TZC has its own interrupt line `TZCINT_*` (`*` = XSP, MSRM, ASRM, ACRCB, DDR00, DDR01, DDR10, DDR11, PCI) (the Manual 3.5.1.1 Features + the Interrupt output shown on each diagram in Fig 3.5-1). One last, critical point: **the TZC itself has no external pins** — it's purely an internal, bus-level access-control mechanism (the Manual 3.5.1.3 External Pins: "In the TZC, there are no external pins."). You won't find a single "TZC pin" anywhere on the board; it lives entirely inside the chip.

### How You See This Under Linux

**Just like CoreSight, you'll see almost none of this under Linux, and that's normal too.** doc07 §16's board status: "Hardware PRESENT (CA55 has TrustZone; TZC-400s instantiated), BUT NO secure-OS runtime on this board" — the hardware is all there (CA55 has TrustZone, and all 9 TZC-400s are instantiated too), but **this board has no operating system running in the secure world**, so there's no OP-TEE, no `/dev/tee`.

The key question is "**who owns the right to configure the TZC**": TZC-400's region configuration is "set once by **TF-A** at boot (BL31, boot firmware running at the highest privilege level, EL3) — it's not something Linux can touch" (doc07 §16: "TZC-400 region programming is owned by TF-A (BL31/EL3) at boot, not by Linux. No userspace path."). And because the TEE (Trusted Execution Environment) in the secure world simply doesn't exist, the entire `libteec`/`tee-supplicant` TEE client framework has nothing to talk to (doc07 §16).

Verification commands (getting "not found" back is this board's normal state, **not** an anomaly):

```bash
ls /dev/tee* 2>/dev/null || echo 'no TEE device (no OP-TEE)'
dmesg | grep -iE 'optee|tee_'
```

(Source: doc07 §16. ✅ Verified on the board; transcript: live/ch04-cpu-periph.txt — the first line lands on `no TEE device (no OP-TEE)`, so TEE genuinely doesn't exist. Source: `07-hardware-unit-usage-guide.md:274`.)

> ⚠️ **Note (not finding `/dev/tee` is the expected state, not something broken)**:
> - **Scenario**: you check for a TEE device, expecting to see OP-TEE.
> - **Symptom**: `ls /dev/tee*` returns nothing, and `dmesg` shows no TEE driver bound either.
> - **Cause**: this board has no secure-OS runtime — the TrustZone hardware is present, but no secure firmware has been loaded into the secure world.
> - **Prevention/handling**: this "not found" is this board's **expected state**; TZC-400's region planning is owned by TF-A (BL31/EL3) at boot, and Linux has no userspace path to access it (Source: doc07 §268, §273). To verify that secure firmware "genuinely exists," you can only look indirectly — see the MTD partitions in the next paragraph.

To confirm whether the secure boot firmware itself exists, the only option is to look indirectly via the xSPI flash's MTD partitions:

```bash
cat /proc/mtd
```

You'll see `mtd0='bl2'` and `mtd1='fip'` — these two partitions hold exactly the TF-A/secure-boot-related images (`bl2` is the second-stage boot loader, `fip` is the Firmware Image Package, containing BL31 and others) (doc07 §16). This handbook's 4.1 (file 00) and the 04 Memory and Storage group have already verified these four MTD partitions verbatim: `mtd0` bl2 / `mtd1` fip / `mtd2` env / `mtd3` test-area — `mtd0`/`mtd1` are boot firmware, read-only, never written.

TZC register addresses (for future reference if bare-metal access is ever needed): filters are configured over APB using `REGION_SETUP_LOW/HIGH_<n>`, `REGION_ATTRIBUTES_<n>`, `REGION_ID_ACCESS_<n>`; addresses for some instances include `TZC400_XSPI` `0x10470000`, `TZC400_SRAMM` `0x10460000`. **This passage is transcribed from the Manual line number doc07 §16 cites (line 15770)**, a register-detail page that falls outside the p352–356 overview range this group read directly. This group's notes have not directly verified it and are only quoting it secondhand; this handbook's 4.4 document lookup table also lists these two addresses.

### Key Capabilities & Limits

- **9 TZC-400 instances**, with filter-unit counts transcribed verbatim from the Manual 3.5.1.3: xSPI = 1, SRAM0+SRAM1 (`TZC400_SRAMM`) = 2, SRAM2 (`TZC400_SRAMA`) = 1, RCPU bus = 1, DDR (2 TZCs, with 3 and 2 filter units respectively, × 2 sets meaning DDR0/DDR1), PCIe = 2.
- **Each TZC supports up to 8 programmable secure regions plus 1 default region** (the Manual 3.5.1.1).
- **Supports up to 256 outstanding transactions on the normal path** (the Manual 3.5.1.1).
- **TZC-400 is standard Arm-licensed IP**; register-level detail like the region-configuration register addresses requires Arm's official Technical Reference Manual — this SoC's Manual only describes how it's integrated (the Manual 3.5.1 Overview: "for details on the functions of TZC-400, see the relevant Technical Reference Manual"). **This Arm document is not in this handbook's list of available sources and has not been verified.**
- **Relationship to the memory controller**: this handbook's 04 Memory and Storage group mentions that the LPDDR4/4X controller carries "in-line ECC, TZC-400" — the TZC-400 there is exactly these four DDR-guarding instances in the table above, `TZC400_DDR00/01/10/11`. TrustZone's access control over DDR is implemented precisely through these filter units.
- **This is a separate thing from CM33's TrustZone-M**: this handbook's 4.2 mentions that CM33 carries a "TrustZone-M security extension" (the security extension of the Armv8-M architecture) — that's the microcontroller core CM33's own execution-mode isolation, a different layer from what this unit covers ("CA55's TrustZone + TZC-400 bus control"). The names are similar; don't conflate them.

### When You'd Actually Use This

**Decision rule (first work out whether your system needs "two worlds that can't see each other")**: mechanisms like TrustZone — "CPU execution-mode isolation plus bus access control" — are designed to split the system into a "normal OS world" and a "trusted secure world," where neither side can see the other's memory. This matters for any scenario that needs "key protection, firmware signature verification, or preventing ordinary applications from reading sensitive data," and it's **not limited to any specific application domain**.

**But "the hardware exists" doesn't mean "you can use it"**: to actually make use of this capability you need an operating system running in the Secure World (commonly OP-TEE) paired with matching drivers, and this board currently has none of that layer. The decision rule: if your project needs secure key storage, a Secure Boot verification chain, or a TEE application (say, handling biometric data or DRM key management), you'll have to **integrate OP-TEE yourself and prepare TF-A's security configuration** — this isn't something you get out of the box, so factor it into your engineering estimate.

**If all you need is a basic integrity check**: say, you just want to confirm "was the firmware tampered with at boot" — TF-A itself already does part of this work during the boot process (BL31 configures the TZC regions at EL3). The decision rule: do you need "a runtime security application" (which needs a TEE runtime), or "boot-time integrity gatekeeping" (which TF-A already partly covers)? For the latter, you can start by checking the bootloader/TF-A boot logs, without having to touch the TrustZone runtime at all.

> **Endnote (Sources)**: the official hardware manual `r01uh1032` §3.5 TrustZone Address Space Controller (TZC) (p352–356, functional overview; register details from 3.5.2 onward, not read). TZC register addresses are quoted secondhand from doc07 §16 (Manual line 15770, not directly verified). The actual owner of the security configuration: TF-A (BL31/EL3 boot firmware). TZC-400 register function details are also covered in Arm's "TZC-400 Technical Reference Manual" (not in this handbook's list of available sources, not verified).

---

## Unit 3: Security IP (Encryption Engine and OTP/Device-ID/JTAG-disable)

> ⚠️ **The single most important sentence, up front**: on this H44 part number, **the board measures out with no hardware crypto interface in Linux at all** (no `/dev/tee`, no Renesas TRNG hwrng, no ARMv8 crypto extension on the CA55) — that much is board-level evidence. And "Linux can't reach it" genuinely does not by itself mean "the silicon doesn't have it" (the ISP is the standing lesson here: this silicon does contain the Mali-C55 ISP, it's just not enabled in the device tree). But the Security field doesn't have to be inferred backwards from the board. The hardware manual `r01uh1032` Table 1.1-1 [p78] — the same first-hand PDF SKU table that establishes this part number's ISP = Available — marks R9A09G057H44GBG's Security column verbatim as **N/A**, verifiable with pypdf. So this part number is **not populated** with the Trusted Secure IP. This unit's "What This Is" section lays out, straight from the Manual, exactly what it would be if it were populated; 〈Key Capabilities & Limits〉 then sets out both the board-level evidence and this SKU determination. Read the two parts as separate things — the mechanism is general knowledge, and whether this part number carries it is already settled by a first-hand SKU table.

### What This Is (the Mechanism)

The **Trusted Secure IP** described in the Manual's 4.8 is an "**optional**" hardware security module made up of three parts: an access management circuit, an encryption engine, and a random number generator (the Manual 4.8 preamble: "This LSI incorporates a Trusted Secure IP module to provide security functions. The module consists of an access management circuit, encryption engine, and random number generator."). Its design purpose boils down to three keywords: **confidentiality** (prevents eavesdropping), **integrity** (prevents tampering), and **authenticity** (prevents impersonation) — achieved together with the matching driver (the Manual 4.8: "In combination with the Trusted Secure IP driver, the Trusted Secure IP can prevent eavesdropping (confidentiality), falsification of information (integrity), and impersonation (authenticity).").

Its most critical security property: **the key lives only inside the Trusted Secure IP and is never read out onto an external bus** (the Manual: "Key information to be used in encrypting and decrypting data is only stored within the Trusted Secure IP, and any external access can be shut out to obtain a system with strong security."). During encryption/decryption, neither the key nor the intermediate data is ever exposed outside the module either (the Manual 4.8.2.2). This "key never touches ground" design is the most fundamental difference between a hardware security module and "running software encryption on the CPU" — software encryption's key eventually has to be loaded into general memory, whereas this module keeps the key from ever leaving even the chip's internal bus.

It guards this line of defense with an "**operating-mode state machine**": Reset → Trusted Secure IP enabled mode → Self-diagnosis mode → Random number generator entropy estimation mode → Encryption engine active mode; if any step detects "irregular access (for example, the program has been tampered with or has run off the rails)," it switches to the **Irregular access detected state**, then locks down and stops outputting any data at all (the Manual 4.8.2.1, Fig 4.8-2: "When irregular access to the Trusted Secure IP … is attempted, the access management circuit does not accept any subsequent access and stops the output of any data from the Trusted Secure IP."). This is a hardware-level line of defense against "a software vulnerability being used to steal the key."

How does it manage to ensure "the user's key never leaves the chip in plaintext"? Through a **Key Installation** flow: the user first encrypts their own key Key-1 into eKey-1 using Key-2, and sends it into the chip. Internally, the chip uses its own retained copy of Key-2 to decrypt it back to Key-1, then converts it into "**key generation information** that only this specific chip can recognize internally" and stores that in external flash. From then on, encryption/decryption uses this piece of information, not the plaintext key (the Manual 4.8.2.3, Fig 4.8-4, 4.8-5). Even better, this key generation information is produced by combining this chip's own unique **Unique ID** with a random number — so even if you copy the whole thing to another chip, it won't work there (the Manual 4.8.1 Table 4.8-1, "Unique ID" entry: "Combining the unique ID with the key generation information prevents the illicit copying of data to another LSI.").

Another **independent-but-related** hardware block is **OTP** (One-Time Programmable memory): physically, a non-volatile storage area where "**each bit can only be written once**," used to hold the chip's individual ID, boot-related settings, and user-defined data (the Manual 4.10.1.1: "Writing to the non-volatile OTP memory macro (core) of the OTP unit proceeds in 32-bit units. However, the same bit can be written only once."). OTP is carved into several functional regions (the Manual 4.10.2 Table 4.10-3, read verbatim): Chip Product ID (the chip's individual identifier, address `0F3h`–`0F6h`), One-time read area enable setting (`12Ah`), Boot device drive strength setting (`12Ch`), User Area 1 (one-time-read region, `160h`–`1DFh`), User Area 2 (general user-defined region, `1E0h`–`3DFh`). OTP supports setting write-protect and read-protect **independently** on designated regions (the Manual 4.10.1.1 Table 4.10-1).

At boot, OTP automatically runs through an initialization sequence: after system reset is released, it first powers on the OTP memory core, automatically reads the contents of OTP addresses `0000h`–`127Fh` into internal staging registers, then powers the OTP memory core back down (to save power). Only after that completes can the OTP unit be accessed by software (the Manual 4.10.1.2, Fig 4.10-1). After that, if software wants to read or write OTP again, it has to turn on the OTP power setting (OTPPWR) itself first.

The last security mechanism carried by OTP is what ties this unit back to 〈Unit 1: CoreSight〉: the OTP-backed "**JTAG-disable**" fuse. Once the JTAG-disable setting is burned into OTP, it **permanently and irreversibly disables the JTAG debug interface at the physical level** (transcribed from doc07 §49; the datasheet's Figure 1.1-1 block diagram lists "JTAG Disable" as one of the chip's built-in functional blocks, alongside "OTP 32-Kbits"). It exists regardless of whether Trusted Secure IP is populated, because it rides on OTP, and OTP is fixed in the silicon.

### How You See This Under Linux

doc07 §49 classifies this unit's on-board status as "**MIXED**," because it's actually a mix of two different states — "not populated" and "present but with no interface" — and they need to be looked at separately:

- **Encryption engine/TRNG/Secure-Boot accelerator: out of reach from Linux (board-level evidence), and not populated on this part number (verified against the first-hand SKU table).** On the board there is no `/dev/tee`, no `hwrng` node backed by a Renesas TRNG, and no vendor encryption-engine core module (doc07 §49 as the lead; the board-level evidence is in §4.2 of this handbook). As to whether the Trusted Secure IP exists in the silicon, the hardware manual `r01uh1032` Table 1.1-1 [p78] marks this part number's Security column verbatim as **N/A** (verifiable with pypdf, the same first-hand PDF SKU table as ISP = Available). So this part number is **not populated** with that IP. (The comparable table in the datasheet `r01ds0429` is degraded, but this determination now rests on the hardware manual instead; see the next paragraph.) **CA55 crypto extension: definitively absent** (no `aes`/`sha2` in the CPU flags — board-level evidence, §4.2 of this handbook).
- **OTP/Device-Unique-ID/JTAG-disable: physically fixed in the silicon, but with no Linux-layer access interface.** The OTP unit "has no userspace/MTD-layer exposure" — even though the OTP hardware is there, the Linux application layer still can't reach it (doc07 §49).

One more thing that's easy to conflate and worth separating out clearly: **the kernel crypto API falls back to pure-software AES/SHA computation on this silicon**, and that's because CA55's instruction set has **no ARMv8 Crypto Extension**. That is "no hardware acceleration at the CPU instruction-set level," **a different matter** from "whether the standalone Trusted Secure IP module is populated." The two just happen to both be absent on this board (doc07 §49; this handbook's 4.2 has already confirmed that CA55's CPU flags contain no `aes`/`sha2`).

Verification commands and what they mean (doc07 §49 usage example):

```bash
openssl speed -evp aes-256-gcm      # runs on the NEON/scalar path — no ARMv8 AES hardware acceleration
head -c 32 /dev/urandom | xxd       # the entropy read here comes from the kernel PRNG, not a Renesas hardware TRNG (no TRNG exists on this silicon)
```

In other words, **none of this group's three units (debug + security) exposes a matching interface at the Linux userspace layer** — CoreSight ("no sysfs nodes"), TrustZone ("no `/dev/tee`"), Security IP ("no userspace exposure for OTP"). All three can only be reached indirectly, through boot firmware or external tools (doc07 §49 cites the first two as corroborating evidence as well).

> 💡 **The one place on this board where OTP content is "used indirectly"**: this handbook's 4.4 document lookup table notes that the temperature sensor's (TSU, see the 07 Communication and Sensing Interfaces group) **calibration trim value comes from OTP** (0.0625 °C/code). In other words, even though OTP is completely unexposed to Linux userspace, the per-chip calibration value burned in at the factory is still being read, behind the scenes, by **the kernel's thermal driver** to convert readings into a temperature. This is one concrete instance where OTP content on this board genuinely does something — it just happens inside a kernel driver, not somewhere you can read out with a single Linux command.

### Key Capabilities & Limits

- **This is an optional module; the part number actually on this board, R9A09G057H44GBG (carrier board manual, component list on Page 11, U1), has its Security field marked N/A — and that is now established by the hardware manual.** The hardware manual `r01uh1032` §1.1.2 Product Lineup Table 1.1-1 [p78] marks R9A09G057H44GBG's Security column verbatim as **N/A**. That table opens in pypdf and can be verified word for word, and it is the same table 00/02 use to establish this part number's ISP = Available [Mali-C55]. The CA55 table on the same page separately notes "Cryptographic extension supported (for security-supported products only)," consistent with this board's CPU flags carrying no `aes`/`sha`. That is a first-hand source, so this part number is **not populated** with that IP. The comparable lineup table in the datasheet `r01ds0429` is a degraded conversion (`file` reports it as `data`, pypdf can't open it, and the lineup table can't be grepped out of `_extracted` — G2), but with the hardware manual supplying a clean SKU table, this determination no longer depends on it. **The board side** agrees: Linux has no hardware crypto interface of any kind (no `/dev/tee`, no Renesas TRNG hwrng, no crypto extension on the CA55) — that is board-level evidence. (A reminder: "Linux can't reach it" is not on its own enough to prove a given IP is absent from the silicon — the ISP is the counter-example, present in silicon but not enabled in the device tree. But the Security field doesn't need to be inferred backwards from the board, because the SKU table marks it N/A directly.)

- **If it were populated, the encryption engine's specs would be as follows** (purely a spec description from the Manual, not something measured on this board — transcribed verbatim from the Manual 4.8.1 Table 4.8-1):
  - **AES**: compliant with NIST FIPS PUB 197, key lengths 128/192/256 bits, block size 128 bits; supported modes ECB/CBC/CTR (NIST SP 800-38A), CMAC (SP 800-38B), CCM (SP 800-38C), GCM (SP 800-38D), XTS (SP 800-38E), GCTR; AES-GCM is implemented as a combination of AES-GCTR + GHASH.
  - **RSA**: key length up to 4096 bits, block size up to 4096 bits.
  - **HASH**: supports SHA1, SHA224/SHA256, GHASH, block size 512 bits.
  - **ECC**: compatible with ECDSA and ECDH, data block length 256 bits.
  - **Random number generator**: a 32-bit true random number generator; the driver can combine outputs into a 128-bit or 256-bit true random number for use as an encryption/decryption key.
  - **Interrupt sources**: 10 total (the Manual 4.8.3 Table 4.8-2: PROC_BUSY, ROMOK, LONG_PLG, WRRDY0, WRRDY1, WRRDY4, RDRDY0, RDRDY1, IWRRDY, IRDRDY).
  - Supports module-stop low-power configuration.
  - Processing time for a single encryption/decryption block (32 bits × 4): 11, 13, or 15 cycles (depending on the algorithm) (the Manual 4.8.2.4 Fig 4.8-6).

- **OTP specs** (the Manual 4.10.1.1 Table 4.10-1, 4.10.1.3 Table 4.10-2, read verbatim): core-level write unit **32-bit** (the same bit can only be written once); operation width when writing via control registers **16-bit**; read width **32-bit**.
- **OTP capacity**: "OTP 32-Kbits" (datasheet Figure 1.1-1 block-diagram label).
- **OTP base address**: `<OTP_base>` = `0x1_0450_0000` (general AXI view) / CM33 non-secure view `0x5045_0000` / CM33 secure view `0x4045_0000` (the Manual 4.10.2.1 Table 4.1-3, cross-checked verbatim against all three Notes).
- **Using Trusted Secure IP's encryption functionality requires Renesas's dedicated driver, which has to be requested separately from your sales contact** — the Manual doesn't include a public spec for this driver (the Manual 4.8.4.1: "Use of the Trusted Secure IP requires the Trusted Secure IP driver provided by Renesas Electronics. Please contact our sales office for information regarding the Trusted Secure IP driver.").
- **Measured software-encryption throughput** (this board's CA55 pure-software path, for reference): because CA55 has no ARMv8 crypto extension, AES/SHA can only run in software — roughly 35–56 MB/s single-core, ~137 MB/s on 4 cores (this handbook's 4.2, 〈Crypto Throughput〉; source: `05-compute-benchmark.md:79-80`). This throughput is enough for data signing and ordinary low-volume encryption, but if you have heavy encrypted-storage or encrypted-transfer requirements, factor this ceiling into your design.

### When You'd Actually Use This

**Decision rule 1 (before you go after hardware encryption/acceleration, check your exact chip part number first)**: your application may need "hardware-level key protection plus hardware-accelerated encryption algorithms" — say, storage full-disk encryption where the key never touches ground, or high-throughput AES/RSA computation. If so, your first step isn't to go looking for a driver: it's to **find out whether the exact RZ/V2H part number you have is Security = Available**. The hardware manual `r01uh1032` Table 1.1-1 [p78] lists the Security column row by row for each part number and can be verified word for word: this board's H44 is marked N/A, while H45/H46/H48 are marked Available. The key point: within the same RZ/V2H family, whether Security IP is carried **differs** across package part numbers, and you can't determine this from the Manual's main text alone — you have to cross-reference the part number actually printed on your chip against the Product Lineup table.

**Decision rule 2 (if the board measures out with no hardware crypto interface, don't go the long way round)**: your board may be like this one and measure out with no hardware encryption engine or TRNG interface under Linux (this board does, and the hardware manual Table 1.1-1 [p78] marks its part number Security = N/A). In practice, encryption needs can then only be met through CPU software computation (say, OpenSSL's pure-software path) or by adding a separate external security chip such as a TPM (Trusted Platform Module). The decision rule: before you spend time hunting down Renesas's dedicated encryption driver, confirm on the board that the hardware interface is actually there — on this board it isn't present in Linux, so looking for the driver won't get you anything to bind it to.

**Decision rule 3 (if all you need is a unique ID, you don't need to bring in the whole encryption module)**: the **Chip Product ID** in OTP (the chip's individual identifier) exists regardless of whether Trusted Secure IP is populated. If all you want is "an ID that's unique to this specific chip" (say, for device licensing or anti-copy serial-number matching), OTP is, in principle, a candidate — but doc07 has already noted that this board has no Linux userspace/MTD-layer access interface for it. The decision rule: whether to actually use it depends first on whether you're willing to write bare-metal/firmware-layer access code (through the OTP control registers described in the Manual 4.10.2.1) — this isn't something a single Linux-application-layer command can read out.

**Decision rule 4 (JTAG-disable is a final, pre-shipment safeguard — once burned, there's no going back)**: the decision rule for this OTP fuse is "**a final safeguard measure before the product ships**." During development you should always keep JTAG/SWD available for debugging (see 〈Unit 1: CoreSight〉). Only once you're certain you no longer need physical debugging, and want to stop a determined attacker from reading memory contents or firmware over JTAG, would you consider burning this fuse before mass production. Keep firmly in mind: **OTP writes are irreversible — once burned, this board can never use JTAG/SWD for debugging again.** This is a one-way decision, to be made only once the product design is truly final.

> **Endnote (Sources)**: the official hardware manual `r01uh1032` §4.8 Trusted Secure IP (p1106–1118, functional overview and operating principles) + §4.10 OTP (p1137–1141, functional overview; the full OTP register list from 4.10.2.1 onward, with the Table 4.10-4 (2/2) control-register detail page not read). Whether it's populated: **the hardware manual `r01uh1032` §1.1.2 Product Lineup Table 1.1-1 [p78]** (this part number, R9A09G057H44GBG, has its Security column marked verbatim N/A; verifiable with pypdf and a first-hand source, and the same table is what establishes ISP = Available [Mali-C55]); the comparable table in the datasheet `r01ds0429`, Table 1.2-1, is in degraded format (G2), and this determination does not depend on it. OTP's existence rests separately on the hardware manual §4.10 (cited above). Board-level evidence (no `/dev/tee`/hwrng/crypto flags): §4.2 of this handbook. Software encryption throughput: `05-compute-benchmark.md`. The encryption functionality requires the Renesas Trusted Secure IP driver (must be requested from a sales contact, spec not public). JTAG-disable is transcribed from doc07 §49.

---

## Hands-On: Temperature Monitoring and Thermal Safety Margin (with the Chip's TSU as the Example)

The three units above are about "debugging" and "cryptographic security." This section adds **thermal** safety — how to read the chip temperature from Linux, see where its automatic protection threshold (trip point) sits, and how much margin is left between temperature and threshold under load. None of this needs an external tool; a single `cat` reads it. But to understand what the number means, and which number is the "about to go wrong" red line, the mechanism has to be joined up first.

> **The sensor's own mechanism is covered in 07**: chip temperature is measured by the TSU (Temperature Sensor Unit, the chip's built-in temperature sensing unit — two of them, TSU0/TSU1), goes through the Linux thermal framework, is driven by `rzv2h_thermal`, and is exposed as two thermal zones. The hardware mechanism (an analog sensor plus its own ADC, measuring die temperature rather than ambient) is covered in full in the TSU section of [07-communication-and-sensing-interfaces.md](07-communication-and-sensing-interfaces.md) (official source `r01uh1032` §7.11, p3739). This section focuses only on using it as a thermal-safety monitoring tool.

### Step 1: Read the Current Temperature of Both Thermal Zones

```bash
for z in /sys/class/thermal/thermal_zone*; do echo "== $z type=$(cat $z/type) temp=$(cat $z/temp)"; done
```

Expected output (✅ Verified on the board; transcript: `live/ch4-w1-tsu.txt`, taken at a load average of 1.37 — neither idle nor heavily loaded):

```text
== /sys/class/thermal/thermal_zone0 type=sensor-thermal0 temp=37000
== /sys/class/thermal/thermal_zone1 type=sensor-thermal1 temp=38000
```

How to read it: `temp` is in **millidegrees** Celsius, so `37000` = 37°C and `38000` = 38°C. The two zones correspond to the chip's two TSUs (type strings `sensor-thermal0`/`sensor-thermal1`), and what they measure is the **die temperature inside the package** — not the enclosure or the ambient temperature.

### Step 2: Read the Trip Points — Where the Chip's Automatic Red Line Is

Reading the current temperature isn't enough; you need to know at what temperature the system acts on its own. The thermal framework calls those thresholds **trip points**:

```bash
for z in /sys/class/thermal/thermal_zone*; do for t in $z/trip_point_*_type; do n=${t%_type}; echo "$z ${n##*/}: $(cat $n\_type) $(cat $n\_temp) hyst=$(cat $n\_hyst 2>/dev/null)"; done; done
```

Expected output (✅ Verified on the board; transcript: `live/ch4-w1-tsu.txt`):

```text
/sys/class/thermal/thermal_zone0 trip_point_0: critical 120000 hyst=1000
/sys/class/thermal/thermal_zone1 trip_point_0: critical 120000 hyst=1000
```

How to read it: each zone has exactly **one** trip point, of type `critical`, at `120000` millidegrees = **120°C**, with 1°C of hysteresis. `critical` is the highest level — hit it and the kernel triggers an emergency shutdown to protect the chip. **The point to remember: no `passive` (throttle-down) or active (cooling-device) trip is registered here, only that single 120°C emergency-shutdown line.** In other words, below 120°C there is no threshold at all that will "automatically clock you down" — any finer temperature management is something you have to build yourself out of the readings (see the gotcha box below, and the throttling approach in 07).

Confirm once more that no cooling device is bound:

```bash
grep -H . /sys/class/thermal/cooling_device*/type 2>/dev/null; echo rc=$?
```

Expected output (✅ Verified on the board; transcript: `live/ch4-w1-tsu.txt`): `rc=2` — `grep` finds no `cooling_device*` file at all (rc=2 means "no such file"), meaning **no cooling device is registered** under this board's thermal framework (no fan, no cpufreq throttler). That is the other side of the same coin as "only a critical trip, no passive trip" above.

### Step 3: Put the Temperature in the Context of a Stress Load to See the Margin

38°C on its own means nothing; what matters is "where does it get to under load, and how far is that from the 120°C red line?" Take a sustained all-core CPU load as the example (`stress-ng` saturating all 4 CA55 cores for 5 minutes, sampling temperature and frequency every 30 seconds):

```bash
# record the baseline first
for z in /sys/class/thermal/thermal_zone*/temp; do echo "baseline $z $(cat $z)"; done
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# load for 5 minutes, sampling as it goes
stress-ng --cpu 4 --timeout 300 --metrics-brief &   # if not on the clean image: sudo apt install stress-ng
for i in $(seq 1 10); do sleep 30; echo "t=$((i*30))s temps=$(cat /sys/class/thermal/thermal_zone0/temp) $(cat /sys/class/thermal/thermal_zone1/temp) freq=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq)"; done
```

Results (✅ Verified on the board; transcript: `live/win-03-stress.txt`; conditions: on the board, `stress-ng --cpu 4`, 5 minutes):

```text
baseline /sys/class/thermal/thermal_zone0/temp 34000
baseline /sys/class/thermal/thermal_zone1/temp 35000
1700000
t=30s temps=36000 38000 freq=1700000
t=60s temps=37000 38000 freq=1700000
t=90s temps=37000 38000 freq=1700000
t=120s temps=37000 38000 freq=1700000
t=150s temps=37000 38000 freq=1700000
t=180s temps=37000 39000 freq=1700000
t=210s temps=37000 39000 freq=1700000
t=240s temps=37000 39000 freq=1700000
t=270s temps=37000 39000 freq=1700000
t=300s temps=37000 38000 freq=1700000
```

Reading those numbers:
- **Baseline tz0 34°C / tz1 35°C** (the idle value for this particular run; idle readings across runs actually span roughly 34–36°C, so quote a single figure only together with the conditions it was taken under).
- **After 5 minutes at full load, tz0 ends at 37°C and tz1 at 38°C, with a peak of 39°C on tz1** (at t=180–270s). An all-core load, in other words, pushes the die temperature up by only about 3–4°C.
- **The frequency stays pinned at 1700000 Hz (1.7 GHz) throughout, with no throttling** — which lines up exactly with what step 2 found: there is no passive trip and no cooling device, so the system will not (and has no threshold telling it to) clock down automatically. It holds 1.7 GHz because nothing comes near a temperature that would trigger throttling.
- **Margin to the red line**: the 39°C peak leaves roughly **81°C** of margin to the single critical trip at 120°C. For this load profile (pure CPU), heat is nowhere near the bottleneck.

### Pass Criteria

- **Both zones readable, with `temp` a plausible room-temperature-scale millidegree value** (tens of thousands, e.g. `37000`): the TSU driver is working.
- **Trip point reads `critical 120000`**: confirms that what you are looking at is the emergency-shutdown line, not some intermediate value mistaken for the red line.
- **Frequency doesn't drop during the stress load and temperature stays far below the trip**: there's thermal margin at that load. Conversely, seeing the frequency start to fall, or temperature closing on a threshold, is the signal that cooling or throttling needs your attention.

> ⚠️ **Note (don't read die temperature as ambient, and don't assume the system will clock down for you)**:
> - **Scenario**: you use `thermal_zone*/temp` as an enclosure or ambient temperature, or assume the system will automatically throttle to protect itself when the temperature gets high.
> - **Symptom**: you take a highish reading as "the room is hot"; or under a long heavy load (**take NPU + GPU + CPU all saturated at once as an example**) the temperature creeps up without any automatic throttling ever appearing.
> - **Cause**: the TSU measures die temperature inside the chip, which can differ substantially from ambient (more so with active cooling or airflow); and this board's thermal framework has only the single 120°C critical trip — no passive throttling trip, and no cooling device — so below 120°C nothing is throttling on your behalf.
> - **Prevention/fix**: to measure ambient temperature you need an external sensor (see 07); to keep the temperature in hand well before the 120°C emergency shutdown, you have to read `thermal_zone*/temp` **yourself** and pair it with your own throttling policy (**for example, cutting the workload fed to the NPU/GPU once temperature passes a limit you choose**). That belongs in your system design — don't expect it to be there at boot.

> **Endnote (Sources)**: the TSU sensor mechanism and accuracy specs are in the TSU section of [07-communication-and-sensing-interfaces.md](07-communication-and-sensing-interfaces.md) (official hardware manual `r01uh1032` §7.11 Temperature Sensor Unit, p3739); the 0.0625 °C/code calibration trim value is read out of OTP and converted by the kernel thermal driver (see the OTP discussion in 〈Unit 3〉 of this group). On-board readings/trips/cooling devices are board measurements, transcript `live/ch4-w1-tsu.txt`; the temperature-versus-stress-load relationship is a board measurement, transcript `live/win-03-stress.txt`.

---

## This Group's Evidence Boundary & Unverified Items

So you can tell "which content comes from directly reading the official manual and which is quoted secondhand from development records," here's an honest, fully laid-out account of this group's evidence boundary — check this section before citing anything, and don't mistake secondhand content for a directly verified conclusion.

- **PDF page ranges actually read**: the Manual `r01uh1032ej0130`'s p352–356 (TZC, 5 pages) + p1106–1123 (Trusted Secure IP + Debug Interface, 18 pages) + p1137–1141 (OTP, 5 pages) = 28 pages of functional overview in total.
- **Register-detail pages not read** (excluded from the outset by the unit-map rules): CoreSight from 4.9.2 onward, TZC from 3.5.2 onward, the Trusted Secure IP register details, and the full OTP register list from 4.10.2.1 onward (Table 4.10-4 was only read through page (1/2); the (2/2) control-register detail page falls outside the page range read and was not read). For every register **address** breakdown that appears in this group (the CoreSight debug address table, the TZC instance addresses, the OTP base address) — except for the OTP base address, which was taken directly from the Manual 4.10.2.1 Table 4.1-3 — the CoreSight and TZC address tables are **all transcribed from doc07** (`07-hardware-unit-usage-guide.md` §15/§16) and have not been directly verified against the original register pages. Each is flagged in place in the text.
- **Unverified external documents**: Arm's "CoreSight Trace Memory Controller Technical Reference Manual," Arm's "TZC-400 Technical Reference Manual," TF-A (BL31/EL3 firmware) — doc07 cites some information from these documents, but the documents themselves are not in this handbook's list of available sources, so any content quoted secondhand from doc07 rather than read directly from the PDF has been marked "not directly verified."
- **Cross-checked against development records**: `06-hardware-resource-map.md` (doc06) was read in full and does not separately mention CoreSight/TrustZone/Security IP (its "Compute & Accelerators," "Interface Buses," and similar sections focus on blocks that have device tree nodes and bound drivers) — this is consistent with, and doesn't conflict with, doc07 §15/§16/§49's conclusion of "no Linux interface on this board." This is also exactly why, in this chapter's resource-map status markings, none of these three units is "✅ Enabled" — instead they're "Present · Not Exposed to Linux" and "Not Populated/Mixed."
- **Security part-number determination (verified first-hand)**: this board's part number is established as R9A09G057H44GBG by the carrier board manual's component list (Page 11); the hardware manual `r01uh1032` Table 1.1-1 [p78] marks its Security column verbatim as **N/A** (verifiable with pypdf, the same table that establishes ISP = Available), so this part number is **not populated** with the Trusted Secure IP. The board side agrees: Linux has no hardware crypto interface of any kind (no `/dev/tee`/hwrng, no crypto extension on the CA55 — board-level evidence in §4.2 of this handbook).
