# g7 · Communication & Sensing Interfaces (Serial / Bus / Network / Expansion / Analog)

This is the **group deep-dive reference file** for Chapter 4, "Full-Board Hardware Resource Map." Chapter 4's overview (file 00, section 4.1) uses a grouped summary table to flatten the whole SoC's functional blocks out so you can see at a glance "what this is, what state it's in on the board, where to go for the deep dive." This file is the **full expansion** of every unit in two of those groups — "communication interfaces" and "sensing / analog." The summary table answers "is it there, is it turned on"; this file answers "how does it actually work, where does Linux expose it, where are the limits of its capability, and when would you actually use it."

This group covers every channel between the board and the "outside world" — serial signaling, sensor buses, wired networking, high-speed expansion ports, and analog input, all gathered in one place. **14 units** in total:

- **Serial**: SCIF (the console UART present from boot), RSCI (multi-mode, multi-channel serial)
- **Bus**: RSPI (SPI), I²C (RIIC), I3C, CAN-FD
- **Compute assist**: CRC (hardware checksum calculation unit)
- **Expansion / connectivity**: GPIO (PFC pin multiplexing), USB3.2, USB2.0, PCIe
- **Network**: GBETH + PTP
- **Sensing / analog**: ADC (12-bit), TSU (on-chip temperature sensing)

Every unit is written on the same skeleton: **What This Is** (the Mechanism, translated from the official hardware manual) → **How You See This Under Linux** (device node / sysfs / driver, with verbatim evidence cited wherever on-board measurement is available) → **Key Capabilities & Limits** (every value comes with a source page number) → **When You'd Actually Use This** (a decision rule that teaches you to judge fit for yourself, rather than copying some specific path). Each section closes with the official source, so you can go straight back to the Manual to look up register details.

> **Placeholder & scope conventions (apply to the whole file)**: the board's network address is assigned dynamically by DHCP and changes with the lease, so this file always writes `<board-IP>` — before connecting, run `ip a` on the board to get the actual current address (see the start of Chapter 4 for why). This group **does not cover** wireless links or link telemetry — that's outside this handbook's teaching scope. Wherever an on-board measured value appears, the measurement environment is, unless noted otherwise: board running Ubuntu 24.04.4 LTS (aarch64), kernel `6.10.14-arm64-renesas`, CPU governor locked to `performance`; resource-inventory figures are verified on the board.

## Units in This Group

| Unit | One-liner | Board status |
|---|---|---|
| [SCIF](#scif-single-channel-the-console-uart-present-from-boot) | The single-channel async UART (the 10-channel RSCI can also be configured as async — see the row below); both boot firmware and kernel messages ride on it | Enabled (system console, `/dev/ttySC0`) |
| [RSCI](#rsci-multi-channel-multi-mode-serial-interface) | 10 channels; each channel can be UART / synchronous / Simple-I²C / Simple-SPI / smart card / LIN | Partially Enabled (only 1 channel running UART, the other 9 disabled) |
| [RSPI](#rspi-standard-spi-masterslave-controller) | Standard SPI master/slave, 4 slave-select pins per channel | Partially Enabled (1 of 3, `/dev/spidev1.0`) |
| [I²C (RIIC)](#i²c-riic-a-low-speed-bus-shared-by-many-devices) | 9-channel I²C master, Fast-mode+ up to 1 Mbps | Enabled (4 buses: i2c-3/4/8/9) |
| [I3C](#i3c-a-new-generation-bus-backward-compatible-with-i²c) | MIPI I3C, backward-compatible with I²C, adds in-band interrupt and hot-join | Disabled / not wired (present in silicon, DT `disabled`) |
| [CAN-FD](#can-fd-rs-canfd-deterministic-noise-resistant-multi-node-bus) | 6-channel CAN-FD, data phase up to 8 Mbps | Enabled but link DOWN (transceiver held in standby by GPIO) |
| [CRC](#crc-operation-unit-hardware-checksum-code-uninventoried-gap) | Hardware CRC calculation unit, 5 selectable polynomials | Unknown (not inventoried) — specs cited from the Manual only |
| [GPIO (PFC)](#gpio-pfc-the-switchboard-of-pin-multiplexing) | Pin function multiplexer, general I/O + 15 special-function switches | Enabled (`/dev/gpiochip1`, 96 lines) |
| [USB3.2](#usb32-gen2-host-ultra-high-speed-external-port) | 2 USB3 hosts, up to 10 Gbps | Enabled (host-only, `xhci-renesas`) |
| [USB2.0](#usb20-general-speed-external-port-includes-otg-silicon) | 1 OTG/DRD + 1 host-only (silicon channels, not physical ports) | Enabled (ch0 silicon supports OTG, this board configured as Host) |
| [GBETH + PTP](#gbeth--ptp-wired-ethernet-and-hardware-timestamping) | 2-channel GbE, built-in IEEE 1588 hardware timestamping | ch0 = `end0` UP; ch1 disabled |
| [PCIe](#pcie-30-high-speed-expansion-bus) | Gen3, switchable between Root Complex and Endpoint | Enabled as Root Complex (no endpoint attached) |
| [ADC](#adc-12-bit-analog-to-digital-converter) | 12-bit successive-approximation ADC, up to 8 channels | Enabled and active (`iio:device0`) |
| [TSU](#tsu-on-chip-temperature-sensing-unit) | 2 sets of on-chip temperature sensors | Enabled and active (2 thermal zones) |

---

## SCIF (Single Channel, the Console UART Present From Boot)

### What This Is (the Mechanism)

SCIF (Serial Communications Interface with FIFO) is the chip's **single-channel** asynchronous UART controller (the 10-channel RSCI covered in the next section can also be configured as an asynchronous UART). UART (Universal Asynchronous Receiver/Transmitter) is the oldest and most basic form of serial communication: two wires (TX to send, RX to receive), and both sides agree in advance on a transfer rate (the baud rate). Data then streams out one bit at a time with no shared clock line — that's exactly what "asynchronous" means here.

SCIF adds one thing on top of a bare-bones UART: a pair of **16-stage TX/RX FIFOs** (First-In-First-Out buffer queues). With a FIFO, the hardware can receive or stage a whole batch of bytes before notifying the CPU once, so during sustained high-speed communication the CPU doesn't have to be interrupted for every single byte sent or received (r01uh1032 §7.4, p2780). It has its own independent baud rate generator, and data can be sent LSB-first or MSB-first. There are 6 interrupt sources in total: transmit end (TEIF), transmit FIFO empty (TXIF), receive FIFO full (RXIF), receive data ready (DRIF), receive error (ERIF), and break detection or overrun (BRIF) (§7.4.1, p2780).

SCIF holds a special place on this board because it doubles as one of the chip's **boot download sources** (boot mode 3 = SCIF download). In other words, this UART shares its hardware with the boot firmware — which is exactly why it's the natural-born "system console": every line of boot output, from u-boot through to the Linux kernel, comes out through this one path.

### How You See This Under Linux

In Linux, SCIF is `/dev/ttySC0`, driven by `sh-sci` (device tree compatible string `renesas,scif`). It's configured as the system console, so wiring up a serial cable to this one path shows you the entire boot sequence and the login prompt. The register base address is `0x11C0_1400` (board status and address sourced from `07-hardware-unit-usage-guide.md` §25, which cites r01uh1032 Table 7.4-3).

On-board measurement confirms this node exists (✅ Verified on the board; transcript: live/ch04-followup.txt):

```text
crw-rw---- 1 root dialout 204,  8 Jul 17 22:38 /dev/ttySC0
```

Major/minor device numbers `(204, 8)`. It's under the `dialout` group, so a regular user needs to join that group or use `sudo` to access the serial port.

### Key Capabilities & Limits

- **Channel count: 1 channel** — verbatim, "Channel: 1 channel" (p2780 Table 7.4-1). This is the only SCIF on the whole chip; it can't be expanded.
- **FIFO depth**: verbatim, "16-stage FIFO buffers for transmission and reception" (p2780).
- **Communication mode**: asynchronous only (Asynchronous communication) — **no** synchronous mode (p2780). If you need clock-synchronized serial communication, use RSCI or RSPI instead.
- **Data format**: data length of 7 or 8 bits; 1 or 2 stop bits; parity selectable as even/odd/none (p2780 Table 7.4-1).

### When You'd Actually Use This

**Mechanically**, SCIF shares hardware with the boot firmware, is enabled from boot, and is always available — so whenever you need a debug/console serial port that "exists from boot and is usable at any time," it's the natural answer. Most SoC dev boards route their main console through exactly this kind of channel, for exactly this reason.

**Decision rule**:

- If all you need is **one** console-grade debug serial port, SCIF is already enough, and it needs no setup at all.
- If your application needs **multiple** serial links at once (say, several sensors that each output their own serial data stream — think a bank of serial industrial gauges), SCIF only has 1 channel and can't be expanded; look at RSCI in the next section instead.
- If you're just trying to "borrow" the console port to carry your own application data, remember it's also the system message outlet — kernel logs will collide with your data on the same line, unless you turn off console output to that port on the kernel command line.

> Official source: the Manual `r01uh1032` §7.4 SCIF (p2780–2781, functional overview; register details from §7.4.2 p2782 onward). Board status and address: `07-hardware-unit-usage-guide.md` §25.

---

## RSCI (Multi-Channel, Multi-Mode Serial Interface)

### What This Is (the Mechanism)

RSCI (Renesas Serial Communications Interface, abbreviated SCI in the Manual's own text) is a **general-purpose serial interface group** that provides **10 independent channels**. It differs from SCIF in two big ways: more channels (10 versus 1), and each channel is a "shapeshifter" — it can be configured into any one of **six modes** (r01uh1032 §7.3.1, p2585):

1. **Asynchronous (UART/ACIA)** — a regular UART.
2. **8-bit clock-synchronous** — one extra clock line beyond asynchronous, so both sides align on a shared clock, giving better jitter resistance than pure asynchronous.
3. **Simple I²C** (master only) — uses the serial channel to emulate an I²C master.
4. **Simple SPI** — uses the serial channel to emulate SPI.
5. **Smart card interface** — compatible with ISO/IEC 7816-3 electrical signaling and transfer protocol.
6. **Simple LIN** — the automotive LIN bus.

Each channel has its own FIFO buffer for full-duplex continuous transfer, and its transfer rate is set by an **independent** baud rate generator per channel (p2585). This "one channel, many personalities" design lets you use the same hardware block to stand in for I²C or SPI when pins are tight, instead of reaching for a dedicated RIIC or RSPI channel.

### How You See This Under Linux

The silicon has 10 channels, but this board's device tree only sets **1 channel** to `okay`: RSCI6 (register address `0x1280_2400`), running in its **asynchronous UART personality**, driven by `rz-sci`. The other 9 channels are all `disabled` in the device tree (source: `07-hardware-unit-usage-guide.md` §26; `06-hardware-resource-map.md`).

On the board, this one enabled channel measures out as **two** `/dev/ttyS*` nodes (✅ Verified on the board; transcript: live/ch04-followup.txt):

```text
crw-rw---- 1 root dialout   4, 64 Jul 17 22:34 /dev/ttyS0
crw-rw---- 1 root dialout   4, 65 Jul 17 22:34 /dev/ttyS1
```

Major number `4`, minor numbers `64`/`65` (consecutive). Worth noting: the board currently only uses its **UART personality** — the Simple SPI and Simple I²C personalities aren't enabled. Those personalities only appear after you edit the device tree and reconfigure pin multiplexing.

### Key Capabilities & Limits

- **Channel count**: verbatim, "Number of channels: 10 channels" (p2585 Table 7.3-1). Only 1 is enabled on the board.
- **Simple I²C mode**: master-only, verbatim "Transfer rate: Up to 400 kbps" (p2587).
- **Simple SPI / synchronous mode**: transmit/receive can select either a 1-stage register or a **32-stage FIFO** (p2586).
- **Asynchronous mode data length**: 7, 8, or 9 bits (p2586) — one more option than SCIF's, a 9-bit setting (some multi-node protocols use that 9th bit as an address/data flag).

### When You'd Actually Use This

**Mechanically**, RSCI is a "multi-channel, multi-mode" serial interface group — its strength is flexibility: one hardware block can be a UART, or it can be a lightweight SPI or I²C.

**Decision rule**:

- Need only **1** console-grade UART → use SCIF (there from boot, no DT changes needed).
- Need **2 or more** independent serial links (say, connecting several modules that each output their own UART data — think multiple serial GNSS receivers or serial battery-management boards) → SCIF isn't enough here; flip the corresponding RSCI channel from `disabled` to enabled.
- Need a lightweight **Simple SPI / Simple I²C / LIN** bus without tying up a dedicated RSPI/RIIC channel → use the matching RSCI personality.
- Before using it, you must edit the device tree to set the target channel to `okay`, and configure pin multiplexing (PFC) to route that channel's signals to physical pins — this is the shared prerequisite for enabling any additional RSCI channel.

> Official source: the Manual `r01uh1032` §7.3 Serial Communications Interface (RSCI) (p2585–2588, functional overview; register details from §7.3.2 p2589 onward). Board status: `07-hardware-unit-usage-guide.md` §26, `06-hardware-resource-map.md`.

---

## RSPI (Standard SPI Master/Slave Controller)

### What This Is (the Mechanism)

RSPI (Renesas Serial Peripheral Interface) is a full-spec SPI (Serial Peripheral Interface) master/slave controller. SPI is a **synchronous, full-duplex** bus: the master drives a clock line (RSPCK), data moves in sync with the clock on two lines — MOSI (master-out-slave-in) and MISO (master-in-slave-out) — and an additional SSL line (Slave Select) picks out which slave device is currently being addressed. Because it has a clock line and is point-to-point, SPI is usually much faster than I²C.

RSPI supports full-duplex 4-wire operation (MOSI/MISO/RSPCK/SSL) or a 3-wire clock-synchronous mode, with a built-in **32-bit × 16-stage** transmit/receive FIFO, and a selectable transfer bit length from 4 to 32 bits (r01uh1032 §7.5.1, p2826). It supports two protocol conventions at once: Motorola SPI mode and TI SSP (Synchronous Serial Protocol) mode. Particularly useful: each channel has **4 slave-select pins (SSL0–SSL3)**, and in master mode each can have its own configurable delay before/after RSPCK — which lets you chain several SPI slave devices with slightly different timing requirements onto the same bus (§7.5.1.1, p2826–2827).

### How You See This Under Linux

The board has 1 RSPI enabled — RSPI0 (device tree node `spi@12800000`) — exposed externally as `/dev/spidev1.0`, driven by `renesas_spi_v2h`. The RSPI1 and RSPI2 nodes are both `disabled` (source: `07-hardware-unit-usage-guide.md` §27; `06-hardware-resource-map.md` §2).

On-board measurement (✅ Verified on the board; transcript: live/ch04-followup.txt):

```text
crw-rw---- 1 root spi     153,  0 Jul 17 22:34 /dev/spidev1.0
```

Interrupts are registered too — `/proc/interrupts` shows three interrupt lines: `12800000.spi:rx`/`:tx`/`:cend` (counts sit at 0 with no load attached, which is normal). `spidev` is Linux's general-purpose interface for sending and receiving SPI frames directly from userspace — `open("/dev/spidev1.0")` plus `ioctl(SPI_IOC_MESSAGE)` is all you need to transfer.

### Key Capabilities & Limits

- **Channel count**: 3 channels (p2826). Only RSPI0 is enabled on the board.
- **Master-mode bit rate**: produced by dividing `RSPI_n_TCLK` by a factor of **2 to 4096** (p2826) — the actual upper bound depends on TCLK and the chosen divider. A separate on-board quick-reference (`04-hardware-quickref.md`) summarizes the practical bit rate as roughly **50 Mbps**; the Manual itself only gives the division-ratio-based definition above, so when citing an absolute rate, calculate it against your actual clock configuration.
- **Error detection**: four types — mode fault, underrun, overrun, parity (p2826).
- **SSL pins**: 4 per channel (SSL0–SSL3), all outputs in master mode (p2827).

### When You'd Actually Use This

**Mechanically**, SPI is "fast, but less flexible on wiring and addressing" — it has none of I²C's ability to address multiple devices on a shared bus by address; instead it picks a device one-to-one via a physical SSL pin.

**Decision rule**:

- The target device supports standard SPI, and you **care more about speed than pin count** (think high-sample-rate ADC modules, display controller chips, external flash memory — a high-speed data-acquisition front end is a good example) → choose RSPI.
- The board's enabled RSPI0 has 4 SSL pins, enough to hang 4 SPI slave devices **on the same bus, sharing the same clock**.
- Only when you need **multiple SPI buses with independently different clocks** (say, two devices that require different SPI modes or rates and can't conveniently share one bus) do you need to edit the device tree to enable RSPI1/RSPI2. The deciding factor is "can these slave devices share one clock and bus" — if yes, one RSPI0 with multiple SSLs handles it; if no, only then open up more channels.

> Official source: the Manual `r01uh1032` §7.5 Serial Peripheral Interface (RSPI) (p2826–2829, functional overview; register details from §7.5.2 p2830 onward). Other documents: the FSP RSPI manual (PMOD channel mapping; this handbook has not yet independently verified it file-by-file). Board status: `07-hardware-unit-usage-guide.md` §27, `06-hardware-resource-map.md` §2.

---

## I²C (RIIC) (A Low-Speed Bus Shared by Many Devices)

### What This Is (the Mechanism)

RIIC (Renesas I²C Bus Interface) is a 9-channel I²C master controller, compatible with a functional subset of the NXP I²C bus interface. I²C (Inter-Integrated Circuit) is a **two-wire, multi-device shared** bus: just SCL (clock) and SDA (data), with every device on the bus having its own address, and the master picking who it's talking to by address. Because it uses so few wires while still supporting many devices, it's the most common interface for sensors and control chips.

RIIC can be configured as either master or slave, and supports **multi-master arbitration**: when multiple masters try to start a transfer at the same time, the controller checks whether the actual voltage level on the bus matches its own internally driven signal. A mismatch means another master has overridden it, so it automatically concedes that it has lost arbitration and backs off (r01uh1032 §7.7.1, p2968–2969). Start/restart/stop conditions are generated and detected automatically by hardware, and addresses can be 7-bit or 10-bit format (p2968). Both SCL and SDA have digital noise filters with a programmable filtering window (p2969).

### How You See This Under Linux

The board has **4** I²C buses enabled, each with its own job (driver `i2c-riic`; source: `07-hardware-unit-usage-guide.md` §28, `06-hardware-resource-map.md` §2.1, `04-hardware-quickref.md`):

| Bus | What's on it | Notes |
|---|---|---|
| `i2c-3` | Display/HDMI bridge chip ADV7535 (@0x3d) | System display path |
| `i2c-4` | Camera TEVS (@0x48) + IMU LSM6DSO16IS (@0x6a) | Sensing and camera |
| `i2c-8` | Clock generator VersaClock 3S (5L35023B-616NLGI8, U30; carrier board manual, BOM on Page 11. The address @0x69 is taken from the development notes rather than measured, because i2c-8 is never scanned) + power-supply PMIC RAA215300 | **On RIIC8, in the always-on power domain, required for boot — do not touch** |
| `i2c-9` | No devices attached | **Free — reserved for your own expansion** |

On-board measurement — the `/dev/i2c-4` node `(89, 4)` exists (✅ Verified on the board; transcript: live/ch04-cpu-periph.txt); reading the WHO_AM_I register of the IMU on i2c-4 confirms the device is alive:

```bash
i2cget -y 4 0x6a 0x0f    # read LSM6DSO16IS's WHO_AM_I register
```

Returns verbatim `0x22` (✅ Verified on the board; transcript: live/ch04-cpu-periph.txt; `i2cget` requires `sudo apt install i2c-tools` first — it's not preinstalled on the clean image, otherwise you'll get command not found). Getting `0x22` back means the IMU on i2c-4 is recognized. Treat that `0x22` as a reference reading rather than a guaranteed result on your own board: the camera (@0x48) and the IMU (@0x6a) on i2c-4 are removable modules. When they aren't fitted, neither address responds (a bus scan then shows both as absent — see the hands-on section below), and the command above reads nothing back. That isn't you doing something wrong; the module simply isn't on the bus.

> ⚠️ **Caution (never touch the devices on i2c-8)**
> - **Scenario**: you want to scan or read/write devices on `i2c-8` (say, after running `i2cdetect -y 8` you get the itch to write to something).
> - **Symptom**: possibly powering off the whole board, or the clock going haywire and the system crashing.
> - **Cause**: `i2c-8` (on RIIC8, in the always-on power domain) has the power-supply PMIC and the system clock generator hanging off it — both are on the critical path for power and clocking.
> - **Prevention/fix**: don't touch anything on `i2c-8`. To attach your own I²C sensor, use the free **`i2c-9`** instead.

### Key Capabilities & Limits

- **Channel count**: verbatim, "9 channels (RIIC0-7 in the PD_OTHERS domain, RIIC8 in the PD_AWO domain)" (p2968). Note that RIIC8 sits alone in the always-on (AWO) power domain — that's exactly why it's used for the clock/power chips that are required at boot.
- **Transfer rate**: verbatim, "Fast-mode+ supported, up to 1 Mbps" (p2968).
- **Noise suppression**: both SCL and SDA have digital noise filters, with a programmable filter window (p2969).

### When You'd Actually Use This

**Mechanically**, I²C is a low-speed, multi-device shared bus — wiring is just 2 lines plus address-based addressing, well suited to connecting sensors or control chips where speed isn't critical but **device count** is high.

**Decision rule**:

- If the device's top speed fits within **1 Mbps (Fast-mode+)**, and you value **wiring simplicity** over throughput (think inertial measurement units, power management chips, clock generators, environmental sensors) → use I²C/RIIC.
- Need higher throughput and it's point-to-point → look at RSPI instead (SPI has no 1 Mbps ceiling).
- This board's `RIIC8` (= `i2c-8`) sits in the always-on power domain, reserved specifically for devices that must be accessible during the boot stage; your own sensors should go on a general power-domain bus instead (the free `i2c-9` is the first choice), so they don't interact with the boot sequence. This rule holds for any RZ/V2H user: keep "system-critical buses" separate from "user expansion buses," so a slip of the hand doesn't hit the power path.

### Hands-On: Scanning an I²C Bus Safely (Including the i2c-8 No-Go Zone)

I²C is a shared bus, so the first thing you usually do after wiring up a sensor is "scan the bus and see whether the device's address answers." The scanning tool is `i2cdetect` (`sudo apt install i2c-tools`; not preinstalled on the clean image, and without it you get command not found). This section first makes clear **how to scan safely**, then walks you through reading both an empty bus and a populated one.

**There's one mechanism-level trap you have to know first**: `i2cdetect`'s default probing method **mixes "read byte" and "write" probes** depending on the address range — for some addresses it issues a write to test whether a device is there. On an ordinary sensor that's at worst a harmless empty write, but on **devices where being written to causes real damage — power management and clocking — a single probe write is enough to cut the board's power or derail its clock**. The way to avoid that is `-r`: it forces read-byte probing only, never writing to the device (the meaning of `-r` comes from i2c-tools, not the chip manual). `-y` skips the interactive confirmation.

**Step 1: list the I²C buses on the board.**

```bash
i2cdetect -l | sort -V
```

On the board (✅ Verified on the board; transcript: `live/ch4-w1-i2c.txt`):

```text
i2c-3	i2c       	Renesas RIIC adapter            	I2C adapter
i2c-4	i2c       	Renesas RIIC adapter            	I2C adapter
i2c-8	i2c       	Renesas RIIC adapter            	I2C adapter
i2c-9	i2c       	Renesas RSCI I2C adapter        	I2C adapter
```

Note the last row: `i2c-9`'s adapter name is **`Renesas RSCI I2C adapter`** — it is not a RIIC channel but a bus provided by RSCI's Simple-I²C personality (see the RSCI section of this document), and it can be scanned and used like any other I²C bus. Only `i2c-3/4/8` are RIIC.

**Step 2: scan a bus that is known to be safe.** The free expansion bus `i2c-9` is the safest thing to practice on:

```bash
i2cdetect -y -r 9
```

On the board this comes back as a completely empty grid (✅ Verified on the board; transcript: `live/ch4-w1-i2c.txt`):

```text
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
...
70: -- -- -- -- -- -- -- --
```

**How to read that grid** (three symbols):

- `--` = no device answered at that address (no ACK). An all-`--` grid means this is an **empty bus** — nothing is currently attached to `i2c-9`, which is normal.
- A hexadecimal number (e.g. `48`) = a device ACKed at that address, and the number is its 7-bit address.
- `UU` = there is a device at that address, but a kernel driver has already claimed it (busy), so `i2cdetect` leaves it alone. `UU` means "present and already taken over by the system," not broken.

**Step 3: scan a bus that has a job, and learn to spot "should be there, isn't."** `i2c-4` is the camera/IMU bus:

```bash
i2cdetect -y -r 4
```

On the board (✅ Verified on the board; transcript: `live/ch4-w1-i2c.txt`):

```text
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: 10 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
...
60: -- -- -- -- 64 -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

Two addresses, `0x10` and `0x64`, have devices answering (other devices present on this bus; this section doesn't speculate about their identity) — but the camera TEVS `@0x48` and the IMU `@0x6a` listed in this document's I²C assignment table both come back `--`, **because neither of those two modules is physically fitted on the board scanned here**. That is exactly the teaching value of "should be there, isn't": compare the scan against the address table you expect, and whatever is missing is a device that isn't online (not seated properly, not powered, or — as here — removed from the board).

> ⏸ **The camera/IMU read-back depends on the modules being fitted**: earlier in this section, `i2cget -y 4 0x6a 0x0f` reads the IMU's WHO_AM_I back as `0x22` — that reading comes from a board with the modules fitted. Whenever `0x48`/`0x6a` are absent from the scan, that read has nothing to answer it, and it can only be run once the modules are back on the bus.

**Step 4 (which is really "don't"): never scan i2c-8.** The correct action for this step is to do **nothing at all** to it:

```bash
# i2c-8 (PMIC/clocking, always-on power domain) — not even a scan
```

> ⚠️ **Gotcha (an itchy finger on i2c-8 can cut the board's power)**
> - **Scenario**: you figure you'll "just" run `i2cdetect` on `i2c-8` to see what's hanging off it.
> - **Symptom**: the board may lose power instantly, the clock may go haywire, the system may hang — and it fails on the spot, irreversibly.
> - **Cause**: `i2cdetect`'s default probing issues **write** probes at some addresses; `i2c-8` carries the power-supply PMIC and the system clock generator, and any probe write landing on one of their registers can switch off your own power or clock.
> - **Prevention/fix**: **never scan i2c-8** — not even with `-r` added; on this board it is an absolute no-go zone. To attach your own sensor, use the free `i2c-9`; and scan any user bus with `-r` so probing stays read-only.

**Pass criteria**: `i2cdetect -y -r 9` returning all `--` = an empty bus and a working tool; `i2cdetect -y -r 4` returning `--` at `0x48`/`0x6a` = your camera/IMU is not currently on the bus. At no point in this procedure should any command touch `i2c-8`.

> Official source: the Manual `r01uh1032` §7.7 I2C Bus Interface (RIIC) (p2968–2972, functional overview; register details from §7.7.2 p2973 onward). Board status: `07-hardware-unit-usage-guide.md` §28, `06-hardware-resource-map.md` §2.1.

---

## I3C (A New-Generation Bus Backward-Compatible With I²C)

### What This Is (the Mechanism)

I3C is the new-generation two-wire bus defined by the MIPI Alliance — 1 channel, compatible with both the NXP I²C subset and the **MIPI I3C-Basic v1.0** protocol at the same time (r01uh1032 §7.8.1.1, p3061). It's designed to "keep I²C's two wires, but get close to SPI's efficiency, plus capabilities I²C never had."

Beyond backward compatibility with legacy I²C (Fast-mode 400 kbps / Fast-mode Plus 1 Mbps), I3C's native **SDR (Single Data Rate) mode** offers more structured transfers: private message, broadcast message (common command code), and direct message (common command code). More importantly, there are three capabilities I²C simply doesn't have (p3062):

- **In-band interrupt**: a slave device can raise an interrupt directly on the data line, **with no need for a separate interrupt pin** — saving both wiring and pins.
- **Master-ship request**: a secondary master can request to take over the bus.
- **Hot-join**: a device can join the bus dynamically while it's running.

### How You See This Under Linux

The silicon has this hardware, but the device tree node is `disabled`, and the board doesn't have any I3C target device wired up. So you'll **not** see `/dev/i3c-*`, and there'll be **no** `/sys/bus/i3c` entries either (source: `07-hardware-unit-usage-guide.md` §29; `06-hardware-resource-map.md` §2.3). Worth spelling out: the Linux kernel itself **does** have an I3C subsystem and a Renesas I3C master driver — this board just doesn't bind it. This falls under **Disabled** in the six-state table from Chapter 4's file 00 (*Learn to Read 'State' First*), not "Not Populated," and not a missing driver either.

### Key Capabilities & Limits

- **Channel count**: 1 channel (p3061).
- **Compatible protocol rates** (I²C-compatibility table, p3061): Standard-mode 0–100 kbps / Fast-mode 0–400 kbps / Fast-mode Plus 0–1 Mbps / High-speed mode 0–3.4 Mbps; I3C's legacy-I²C message mode supports Fm/Fm+ (p3062).
- **Address format**: 7-bit addressing (p3062).
- **Clock stalling** capability (p3062).

### When You'd Actually Use This

**Mechanically**, I3C is "I²C's wiring, with more capability." But that capability only matters if **the device on the other end also supports I3C**.

**Decision rule** (fairly direct):

- The device you're connecting **only supports I²C** → just use RIIC, no need to touch I3C.
- The device supports I3C, and you need what it adds — higher speed, an in-band interrupt that saves a pin, hot-join for dynamic device addition → then I3C pays off. This shows up most often in newer-generation sensors (for example, some newer IMUs support both I²C and I3C interfaces).
- As shipped, this interface is neither wired up nor enabled in the device tree on this board. To use it, you'd have to do all three yourself: flip the DT node on, set up pin multiplexing with PFC, and wire in an I3C-capable device — all three are required, none optional.

> Official source: the Manual `r01uh1032` §7.8 I3C Bus Interface (I3C) (p3061–3064, functional overview; register details from §7.8.3 p3065 onward). Board status: `07-hardware-unit-usage-guide.md` §29, `06-hardware-resource-map.md` §2.3.

---

## CAN-FD (RS-CANFD) (Deterministic, Noise-Resistant Multi-Node Bus)

### What This Is (the Mechanism)

RS-CANFD is a 6-channel CAN-FD controller conforming to **ISO 11898-1 (2015)**; the same controller can send and receive both classic CAN frames and CAN-FD frames (identifiers can be standard 11-bit or extended 29-bit) (r01uh1032 §7.9.1, p3363). CAN (Controller Area Network) originated as an automotive bus, and its strength is being **multi-node, noise-resistant, and deterministic**: every node hangs off the same differential pair, non-destructive arbitration is done by identifier priority, and even with noise or collisions on the bus, there's hardware-level arbitration and error retransmission. CAN-FD (Flexible Data-rate) builds on this by relaxing both the rate and length of the data phase, for higher throughput.

The controller has configurable message buffers built in (the Manual lists two categories — "individual buffers" and "buffers shared across 6 channels" — and the count can be allocated as needed); the receive side supports **up to 768 filter rules**, routing messages that match a rule to a designated buffer or FIFO (p3363). Recovery after bus-off (the error state a CAN node is forced into after repeated transmit errors) can follow the standard ISO 11898 procedure, or be forced by software intervention (p3364).

### How You See This Under Linux

CAN is a **network device** in Linux (SocketCAN), not a `/dev` character device. The board has two netdevs, `can0`/`can1` (driver `rcar_canfd`), both currently link state **DOWN** (on the board both show `DOWN <NOARP,ECHO>`).

Which hardware channels do these two netdevs correspond to? You can tell from `/proc/interrupts` — the registered interrupt lines are `canfd.ch0_err`/`ch0_trx` and `canfd.ch3_err`/`ch3_trx` (plus the shared `canfd.g_err`/`g_recc`), meaning **hardware CAN channels 0 and 3 are the ones enabled**, and Linux enumerates them in order as `can0`/`can1`. This matches the WS125 RDK carrier board's wiring: the carrier board actually uses a **TCAN1046** CAN-FD transceiver chip, brought out externally through two 3-pin headers J2/J3. Two different sets of pins have to be kept apart here — the **data pins** are **P80/P81 = CAN_CTX0/CRX0** (ch0 TX/RX) and **P86/P87 = CAN_CTX3/CRX3** (ch3 TX/RX), muxed to their CAN function by the PFC (`gpioinfo` on the board shows P80/P81/P86/P87 as `kernel [used]`; transcript: `live/ch4-w1-gpio.txt`). The transceiver's **standby control pins** are two separate GPIOs — **PA2 = `can0_stb`, PA3 = `can3_stb`** (same transcript, both `output active-high [used]`, hogged by the kernel at boot). Don't mistake the data pins for the standby pins. (WS125 RDK manual §3.11.) The remaining CAN channels exist in silicon, but this carrier board hasn't wired all of them out.

Beyond `ip link`, there's one easily missed prerequisite step to bringing CAN up — **releasing the transceiver from standby first**:

> ⚠️ **Caution (CAN won't come up, because the transceiver is in standby)**
> - **Scenario**: you run `ip link set can0 up`, expecting to bring CAN up and talk to a controller node.
> - **Symptom**: the interface comes up but still shows DOWN, and you can't communicate no matter what.
> - **Cause**: the CAN transceiver is held in standby by a GPIO (the standby pin corresponding to this board's `can0`), so the physical layer never actually powers on. This is **a design choice in the carrier board's external transceiver circuit**, not a limitation of the controller itself.
> - **Prevention/fix**: first pull the corresponding standby pin (`can0_stb` = PA2 / `can3_stb` = PA3) low via GPIO to release standby (e.g. `gpioset $(gpiofind PA2)=0`), then bring the interface up. **Note that PA2/PA3 are already hogged by the kernel at boot** (a *hog* is a device tree declaration that claims a GPIO line at boot; `gpioinfo` shows them as `[used]`; transcript: `live/ch4-w1-gpio.txt`) — calling `gpioset` directly comes back with `Device or resource busy`. You have to release that hog first, or drive the pin through whichever device tree/driver path owns it. It can't be taken by force. (Source: `07-hardware-unit-usage-guide.md` §30; `04-hardware-quickref.md`.)

### Key Capabilities & Limits

- **Channel count**: Six channels (p3363). 2 are brought out on the board (hardware ch0/ch3).
- **Classic CAN rate**: Classical CAN mode, up to 1 Mbps (p3363).
- **CAN-FD rate**: verbatim, "Nominal bit rate: Max. 1 Mbps / Data bit rate: Max. 8 Mbps" (p3363) — the arbitration phase is still 1 Mbps; what gets accelerated is the data phase (up to 8 Mbps).
- **Error-state monitoring**: readable error counters, monitoring protocol errors such as stuff/form/ACK/CRC/bit/ACK-delimiter errors (p3364).

### When You'd Actually Use This

**Mechanically**, CAN-FD's value is **determinism + noise resistance + multi-node arbitration** — when multiple nodes contend for the bus, arbitration is done in hardware, and errors get retransmitted in hardware. That's why it suits "electrically hostile environments that still need reliable multi-node communication" better than UART or SPI.

**Decision rule**:

- Need deterministic, noise-resistant multi-node bus communication with other controller nodes (think motor drivers, servo nodes, distributed sensor nodes — DroneCAN/UAVCAN on a mobile vehicle is one application) → CAN-FD is a common choice.
- The other end only supports classic CAN → use classical CAN mode (up to 1 Mbps) to stay compatible.
- The other end also supports CAN-FD, and you need higher data throughput → switch to FD mode (data phase up to 8 Mbps). The deciding factors are "does the other end support FD" and "do you actually need that extra data-phase bandwidth."
- Regardless of which mode you use, you must release the transceiver from standby before use (see the caution box above) — that's a fixed prerequisite of this board's external circuitry.

### Hands-On: Bringing a CAN-FD Interface Up (the Bitrate Mechanism, and How Far Send/Receive Goes)

On Linux, CAN is a SocketCAN network device — you bring it up with `ip link`, not by opening a `/dev` node. This section brings `can0` up properly; along the way you'll hit a very typical "configuration contract" trap, and see the honest boundary of what this board can send and receive.

**Step 1: look at the state before you start.**

```bash
ip -br link | grep can
ip -details link show can0
```

On the board (✅ Verified on the board; transcript: `live/ch4-w1-can-pre.txt`):

```text
can0             DOWN           <NOARP,ECHO>
can1             DOWN           <NOARP,ECHO>
```

`ip -details link show can0` prints the controller's timing capabilities, and a few of those numbers matter in a moment (excerpted verbatim):

```text
    can <FD> state STOPPED (berr-counter tx 0 rx 0) restart-ms 0
	  rcar_canfd: tseg1 2..128 tseg2 2..32 sjw 1..32 brp 1..1024 brp_inc 1
	  rcar_canfd: dtseg1 2..16 dtseg2 2..8 dsjw 1..8 dbrp 1..256 dbrp_inc 1
	  clock 80000000 ... parentdev 12440000.can
```

`can <FD>` means this is an FD-capable controller; `clock 80000000` is the 80 MHz CAN clock (the basis for converting a bitrate); the upper row is the arbitration-phase (nominal) timing range and the lower `dtseg/dbrp` row is the data-phase timing range — **FD needs two sets of timing precisely because the arbitration phase and the data phase each have their own bit timing.**

**Step 2: check your send/receive tools.** The usual `candump`/`cansend` come from `can-utils`, which isn't preinstalled on the clean image:

```bash
which candump cansend      # empty on this board -> not installed
sudo apt install can-utils # install when you need them
```

If you'd rather not install a package, Python's built-in socketcan support can send and receive — `AF_CAN`/`CAN_RAW` are both available on the board (✅ Verified on the board; transcript: `live/ch4-w1-can-pre2.txt`):

```bash
python3 -c "import socket; print(hasattr(socket,'AF_CAN'), hasattr(socket,'CAN_RAW'))"
# True True
```

**Step 3: bring can0 up (the correct command).** The device tree puts the controller in **FD mode** on this board, so bringing it up **requires the arbitration `bitrate`, the data-phase `dbitrate`, and `fd on` all together**:

```bash
sudo ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on
```

This brings it up on the board (✅ Verified on the board; transcript: `live/ch4-w1-can-fd-up.txt`):

```text
5: can0: <NOARP,UP,LOWER_UP,ECHO> mtu 72 qdisc pfifo_fast state UP mode DEFAULT group default qlen 10
    can <FD> state ERROR-ACTIVE (berr-counter tx 0 rx 0) restart-ms 0
	  bitrate 500000 sample-point 0.875
	  tq 25 prop-seg 34 phase-seg1 35 phase-seg2 10 sjw 5 brp 2
```

`state UP` together with `ERROR-ACTIVE` is what a **correctly brought-up** interface looks like — `ERROR-ACTIVE` is the default, healthy error state of a working CAN node (the normal state when the error counters are 0). It is **not** a fault; don't let the word frighten you.

**Where those timing numbers come from (the bitrate mechanism)**: all you supplied was the target `bitrate 500000`; the driver works out the prescaler and segment lengths itself from the 80 MHz clock —

- the time quantum `tq = brp ÷ clock = 2 ÷ 80 MHz = 25 ns`;
- each bit = `sync(1) + prop-seg(34) + phase-seg1(35) + phase-seg2(10) = 80` tq;
- bit time = `80 × 25 ns = 2000 ns` → `1 ÷ 2000 ns = 500000 bit/s`, exactly the 500 kbit/s you asked for;
- the `sample-point = (1+34+35) ÷ 80 = 0.875`, i.e. sampling at 87.5% of the bit (CAN's customary late sample point, for tolerance of propagation delay).

> ⚠️ **Gotcha (give it only a bitrate and the interface won't come up)**
> - **Scenario**: out of classical-CAN habit you type `sudo ip link set can0 up type can bitrate 500000`, with no `dbitrate`/`fd on`.
> - **Symptom**: `RTNETLINK answers: Invalid argument`; `dmesg` shows `incorrect/missing data bit-timing` and `open_candev() failed: -EINVAL`, and the interface stays DOWN (verified on the board; transcript: `live/ch4-w1-can-diag.txt`).
> - **Cause**: this is a **contract across two places** — the controller is set to FD mode in the device tree (`dmesg` shows `global operational state (clk 1, fdmode 1)`), and FD mode **requires** data-phase bit timing; supply only the arbitration bitrate and the data-phase timing is missing → `EINVAL`.
> - **Prevention/fix**: on this board always bring it up in FD form — `bitrate <arbitration> dbitrate <data> fd on`, all three together.

**The honest boundary on closing the send/receive loop (⏸)**: the textbook move — **internal loopback**, where the controller returns its own transmitted frames so you can verify the send/receive path with no transceiver and no peer — **is not available on this board's kernel.** The `rcar_canfd` driver doesn't support loopback mode:

```bash
sudo ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on loopback on
# RTNETLINK answers: Operation not supported   (same for all four combinations: classical/FD x can0/can1)
```

(Verified on the board; transcripts: `live/ch4-w1-can-loopback.txt`, `ch4-w1-can-fd-loopback.txt`.) Because loopback is unavailable, sending frames with Python socketcan gets nowhere: with the loopback option the interface never reaches UP, every send returns `OSError: [Errno 100] Network is down`, and `ip -statistics` shows RX/TX at 0 throughout — **not one frame is actually sent or received.**

⏸ So "a real send/receive loop with cansend/candump" can't be closed on a bare board, for three reasons: candump/cansend aren't installed, the driver doesn't support loopback, and there is no second CAN node on the bus. That also settles **when you need a transceiver**: if loopback were available, self-testing would need neither transceiver nor peer. But since this board can only go through the physical bus, actually sending and receiving means (1) releasing the TCAN1046 transceiver's standby first (see the caution box above — pull the standby pin low via GPIO), and then (2) putting at least one CAN node that will answer on the bus. With that in place, the send/receive flow is:

```bash
# Terminal A: listen on can0
candump can0
# Terminal B: send one frame (identifier 0x123, data DEADBEEF)
cansend can0 123#DEADBEEF
```

**Step 4: take the interface back DOWN when you're done.**

```bash
sudo ip link set can0 down
```

**Pass criteria**: `ip -details link show can0` showing `state UP … ERROR-ACTIVE` with `bitrate 500000 sample-point 0.875` = the interface is up (at the controller level); RX/TX of 0 in `ip -statistics link show can0` at that point is **expected** — with no active transceiver and no peer node there is no traffic to count. The real send/receive pass criterion (RX/TX counters increasing, candump receiving a frame) can only be checked once the transceiver is out of standby and a peer is attached; on a bare board that step stays ⏸.

> Official source: the Manual `r01uh1032` §7.9 CAN-FD Interface (CANFD) (p3363–3366, functional overview; register details from §7.9.2 p3367 onward). Other documents: FSP CANFD manual `r01us0478`, `can-utils` (software; this handbook has not yet independently verified either file-by-file). Carrier board wiring: WS125 RDK manual §3.11. Board status: `07-hardware-unit-usage-guide.md` §30, `06-hardware-resource-map.md` §2.2.

---

## CRC Operation Unit (Hardware Checksum Code, **Uninventoried Gap**)

### What This Is (the Mechanism)

The CRC (Cyclic Redundancy Check) operation unit is a **hardware CRC calculator**, 1 channel, that can produce a CRC code of the corresponding length for whichever polynomial you choose — 5 options in total: 8-bit (CRC-8), 16-bit (CRC-16, CRC-CCITT), 32-bit (CRC-32, CRC-32C) (r01uh1032 §7.6.1, p2954). CRC is the most common integrity-check method in communication and storage: divide a block of data by some polynomial, attach the remainder to the data as a check code, and the receiving side recomputes it and compares — a mismatch means the data got corrupted in transit.

The hardware supports two input granularities for parallel processing — 8-bit or 32-bit — and can switch the output bit order to LSB-first or MSB-first, to match different communication protocols' bit-order conventions for the CRC field. There's also a **snoop function**: you can set it to monitor read/write activity at a given register address and automatically feed the data passing through that address into the CRC calculation, with no need to write extra code to shuttle the data around (p2954). The register set is very compact — just 4 registers: `CRCCR0` (control), `CRCDIR` (data in), `CRCDOR` (data out), `CRCSAR` (snoop address), base address `0x1300_0800` (§7.6.2, p2956).

### How You See This Under Linux

**This unit's on-board status has not been measured — this is an honestly flagged gap.** Neither the hardware inventory document `06-hardware-resource-map.md` nor `07-hardware-unit-usage-guide.md` lists this block; Chapter 4's unit-map gap list marks it as "Unknown (not inventoried)." This section makes **no guesses** about whether there's a corresponding Linux driver or device node on the board — it faithfully transcribes only the Manual's specifications. Whether it's actually accessible from Linux on this board is something that needs to be probed and filled in later — for example, by checking whether there's a corresponding mmio node (around `0x1300_0800`), or whether any of the kernel's CRC-related drivers bind to this hardware block.

### Key Capabilities & Limits

- **Channel count**: 1 channel (p2954).
- **Selectable polynomials** (p2954 Table 7.6-1, polynomials transcribed item by item): 8-bit CRC-8 (X⁸+X²+X+1); 16-bit CRC-16 (X¹⁶+X¹⁵+X²+1) and CRC-CCITT (X¹⁶+X¹²+X⁵+1); 32-bit CRC-32 and CRC-32C.
- **Limitation**: verbatim, "The circuit does not have a function to divide data for calculation into CRC calculation units. Write data in 8-bit or 32-bit units." (p2954 Note 1) — in other words, splitting data into calculation units is **software's job**; the hardware doesn't do that splitting automatically.

### When You'd Actually Use This

**Mechanically**, the value of hardware CRC is "offloading the CPU cost of computing CRC in pure software."

**Decision rule** (with an honest boundary attached):

- If your application needs integrity checking somewhere in a data transfer or storage path (say, a CRC field in a custom serial protocol, or firmware image verification), and pure-software CRC is already a measurable CPU burden → the hardware CRC unit can, in principle, take that work over.
- **But this is a general decision rule that assumes "already inventoried, with a known-usable driver path."** This unit currently lacks any on-board measurement to back it up, so before you actually rely on it, you need to probe it yourself first (check whether there's a corresponding mmio node or kernel driver binding), confirm it's genuinely accessible from Linux, and only then decide whether to use it. Until confirmed, treat it as "exists on paper, unverified on the board."

> Official source: the Manual `r01uh1032` §7.6 CRC Operation Unit (CRC) (p2954–2957, functional overview). Datasheet `r01ds0429`. Board status: **an uninventoried gap** (neither `06` nor `07` lists it; the unit-map marks it "Unknown").

---

## GPIO (PFC) (the "Switchboard" of Pin Multiplexing)

### What This Is (the Mechanism)

PFC (Pin Function Controller) is the "switchboard" between external pins and the chip's internal functional units. This SoC has far fewer physical pins than internal functional signals, so nearly every multiplexable pin can switch between **general-purpose GPIO mode** and up to **15 special-function modes** (I²C, SPI, UART, CAN, and so on). The switch is decided by two register groups together: `PFC_PMC_mn` (mode select: Port mode or Control mode) and `PFC_PFC_mn` (function select, Mode 1 through 15) (r01uh1032 §4.2.1.1, p362 Table 4.2-2).

Besides mode switching, the PFC also manages each pin's electrical characteristics: drive strength, slew rate (how sharp or gradual the signal edge is), pull-up/pull-down resistors, input/output enable, N-channel open drain, Schmitt trigger, and digital noise filtering (filter level, stage count, and sampling interval are all independently configurable) (§4.2.1, p361–364). The PFC also has an independent **Event Link Controller (PFC_ELC_GPIO)** attached, which lets events on a GPIO pin drive other peripherals directly, with no CPU involvement needed (§4.2.1.6, p364).

The key to understanding the PFC is this: **a given physical pin can only correspond to one function at a time.** That's what "pin multiplexing" (pin-mux) means — if a pin is switched to the I²C function, it can't simultaneously serve as a general-purpose GPIO. The root cause behind what Chapter 4, section 4.1 describes as "a lot of units exist in silicon but are disabled on this board" is often exactly this: the pin got claimed by some other function.

### How You See This Under Linux

In Linux, GPIO is `/dev/gpiochip1`, with **96 lines** total, driven by `pinctrl-rzg2l` (which merges both pinctrl [pin multiplexing] and gpio functionality into one driver). Access goes through the modern **gpiod character-device ABI** (`libgpiod` tools: `gpioinfo`/`gpioget`/`gpioset`/`gpiofind`), **not** the legacy `/sys/class/gpio` sysfs interface.

There's an easy point of confusion in the on-board measurement — `gpiodetect` lists **two** gpiochips (✅ Verified on the board; transcript: live/ch04-followup.txt):

```text
gpiochip0 [gpio_dummy]      (32 lines)
gpiochip1 [10410000.pinctrl] (96 lines)
```

`gpiochip0` is a placeholder **fake expander** (`gpio_dummy`, 32 lines), **not** real SoC pins; for real physical pins, always go through **`gpiochip1`** (96 lines, address `10410000.pinctrl`). A number of lines are already known to be hogged by the system: the CAN transceiver's standby pins (PA2/PA3) and the camera/HDMI reset pins. PB2/PB3 can additionally be held by a userspace program that has claimed them with `gpioset`. The board exposes a standard Raspberry Pi–compatible **40-pin header (J1, 3.3V levels)** — the special-function pins are Pin3/5 = I²C7, Pin8/10 = UART5, Pin19/21/23/24/26 = SPI6, Pin35/38/40 = PCM (source: WS125 RDK manual §3.12, `04-hardware-quickref.md`). Most Raspberry Pi HATs are compatible, but HATs using **5V logic** need level shifting first.

### Key Capabilities & Limits

- **Function overview** (verbatim, p361 Table 4.2-1): "GPIO control / Switching between functions multiplexed on the pins / Switching the drive strength of the pins / Switching the slew rate of the pins / Pull-up/down control of the pins / Input enable control / Output enable control / N-ch. open drain control / Schmitt control / OSC mode switching / Reset latch function / Digital noise filter control".
- **Mode hierarchy**: Port Mode versus Control Mode is switched by the `PMC_mn` bit; within Port mode, input/output direction is then set by `PM_mn` (p362).
- **One pin, one function**: this is a hardware limitation, not a configuration choice — once a pin is switched to a dedicated function, it can't double as GPIO at the same time.

### When You'd Actually Use This

**Mechanically**, GPIO is the most basic form of digital I/O — any need to "read a high/low level" or "drive a high/low level" goes through it.

**Decision rule**:

- Any simple digital input/output (reading a switch/button state, driving an LED, enabling/disabling some peripheral chip, bit-banging a low-speed custom protocol) → use GPIO.
- **Check whether the pin is already claimed before you touch it**: use `gpioinfo` to see whether the line you want is already multiplexed away by another function (I²C/SPI/UART/CAN transceiver standby, etc.). The PFC only ever lets one pin serve one function at a time — once a pin is switched into a dedicated function mode, it can't simultaneously act as general-purpose GPIO; only a free line is safe to configure as an input or output. This check applies to any RZ/V2H board, and it's the first step in avoiding the "I wired it up but I'm not reading anything" trap.

> Official source: the Manual `r01uh1032` §4.2 Pin Function Controller (PFC) (p361–365, functional overview; register details from §4.2.2 p366 onward). Other documents: `libgpiod` (software), WS125 RDK manual §3.12 (40-pin header pinout), `04-hardware-quickref.md`. Board status: `07-hardware-unit-usage-guide.md` §31, `06-hardware-resource-map.md` §2.2.

---

## USB3.2 Gen2 (Host, Ultra-High-Speed External Port)

### What This Is (the Mechanism)

The board has 2 USB3 modules (ch0/ch1), each made up of a **USB3HOST** (compatible with the eXtensible Host Controller Interface, xHCI) plus a test sub-block (USB3TEST); the PHY (physical-layer transceiver) runs over the **PIPE interface**, an architecture shared with PCIe/SATA (r01uh1032 §6.4.1, p1636).

Worth knowing: this chapter's USB3 section of the Manual is a **simplified version** — verbatim, "This manual is a simplified version. For more information, refer to the User's Manual Additional Document." (p1636) — the full register-level documentation isn't in this volume; register details need to be looked up separately in the User's Manual Additional Document.

### How You See This Under Linux

Driver `xhci-renesas`. Once a device is plugged in, it shows up at `/dev/bus/usb/00X/00Y` (mass storage additionally gets `/dev/sdX`, serial devices additionally get `/dev/ttyACM*`/`/dev/ttyUSB*`). The USB30 Host base address is `0x15850000`, USB31 Host is `0x15860000` (source: the Manual `r01uh1032` §1.8 Address Map, Table 1.8-1 Detailed Address Space, p170). Note that `0x15840000` is the **USB21 PHY** region, not the USB30 Host — those two addresses are one slot apart, so don't mistake the PHY address for the controller's.

On-board measurement — `lsusb` shows 2 USB 3.0 root hubs (ID `1d6b:0003`) and 2 USB 2.0 root hubs (ID `1d6b:0002`, covered in the next section on USB2) (✅ Verified on the board; transcript: live/ch04b-runtime.txt):

```text
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
```

This listing changes with whatever's plugged in: with a USB wireless dongle in one of the USB3 Type-A ports, that dongle appears alongside the hubs (the "something's plugged in" case). With nothing attached, none of these root hubs carries a device (i.e. both Type-A ports free). So your own board's `lsusb` device list depends on whatever happens to be plugged in at the time — but the USB2.0/USB3.0 root hubs it lists are controller enumerations corresponding to those **2 physical Type-A ports**, not to 4 pluggable ports.

### Key Capabilities & Limits

- **Supported speeds** (verbatim, p1636 Table 6.4-1/6.4-2, the Host Controller supports all five speeds): "Super Speed Plus (10 Gbps), Super Speed (5 Gbps), high-speed (480 Mbps), full-speed (12 Mbps), and low-speed (1.5 Mbps) transfer".

> 💡 **Tip (don't read the root hub's `20000` as "20 Gbps available")**: running `cat /sys/bus/usb/devices/usb2/speed` (or `usb4`) on the board returns `20000` (Mbps) (✅ Verified on the board; transcript: `live/ch4-w1-usb.txt` — both USB3 root hubs report it). That is the xHCI root hub **advertising** its SuperSpeedPlus dual-lane (x2) capability; it does not mean this SoC's PHY can reach 20 Gbps — this board's USB3 PHY is **Gen2x1 (single lane, 10 Gbps, §6.4 p1636)**, so the SuperSpeedPlus link it actually negotiates tops out at 10 Gbps. Seeing `20000` is not 20 Gbps of usable bandwidth.

- **Transfer types**: all four supported — isochronous, interrupt, control, bulk — plus high-band transfer support for isochronous/interrupt (p1636).
- **Channel count**: 2 (ch0/ch1).
- **Limitation (host-only)**: this board's silicon is a **host-only** design — the board can only act as a host connecting to peripherals, and **cannot** flip around to act as a USB device attached to some other host. For a device/OTG role, see the USB2.0 ch0 section next.

### When You'd Actually Use This

**Mechanically**, USB3's value is **high bandwidth** (up to 10 Gbps).

**Decision rule**:

- External devices that need high-speed bulk data transfer (think external SSDs, high-resolution UVC cameras, high-speed network dongles) → use USB3.
- Ordinary-speed peripherals (keyboard/mouse, UART-to-USB adapters, low-speed sensor modules) → no need to tie up USB3, USB2 is fine (see next section).
- If your application needs the board to **flip around and act as a USB device** (say, presenting itself to an upstream host as a USB serial or storage device) → USB3 can't do this (host-only); you need USB2.0's ch0 in Function mode.

### Hands-On: Mounting a USB Storage Device

Plug a flash drive or external SSD into a USB port and Linux won't "open" it for you — you have to find its device node and mount it onto a directory yourself. This section covers that flow. **Mechanically**, a USB mass-storage device is enumerated by the `usb-storage`/`uas` driver as a SCSI disk and appears as `/dev/sdX` (`sda`, `sdb`, …), with partitions as `/dev/sdX1` (general Linux behavior, not something specified by the chip manual; this mount procedure applies to any USB port, not just USB3).

**First, the baseline with no medium attached** (✅ Verified on the board; transcript: `live/ch4-w1-usb.txt`):

```bash
lsblk -o NAME,SIZE,TYPE,TRAN,MOUNTPOINT,MODEL
ls /dev/sd*
```

```text
NAME          SIZE TYPE TRAN MOUNTPOINT MODEL
mtdblock0   116.5K disk
mtdblock1     1.8M disk
mtdblock2     128K disk
mtdblock3      14M disk
mmcblk0     238.8G disk mmc
├─mmcblk0p1   200M part mmc
└─mmcblk0p2 238.6G part mmc  /
ls: cannot access '/dev/sd*': No such file or directory
```

In that listing, `mtdblock0`–`mtdblock3` are the board's built-in flash (MTD, Linux's Memory Technology Device layer — the small boot/environment partitions on the xSPI NOR, all in the MB range, not USB storage); `mmcblk0` is the system's own eMMC/SD (the root filesystem is mounted at `/`). The point is that there is **no `/dev/sd*` at all** — meaning no USB storage device is attached. That's the "clean" picture you should see before plugging a drive in; once `/dev/sda` appears, it's online. (`lsusb` at this point lists the root hubs plus whatever non-storage devices are attached — a USB wireless dongle, for instance; a flash drive only adds a storage device once plugged in.)

**The mount procedure** (⏸ the physical steps below are a procedure to follow with your own medium; what is verified above is the no-medium baseline):

```bash
# 1) Plug the USB flash drive/SSD into a Type-A port

# 2) Confirm the device nodes appear (an extra sdX plus its partition sdX1)
lsblk
dmesg | tail          # you'll see enumeration messages like sd 0:0:0:0: [sda] ...

# 3) Check the partition's filesystem type
lsblk -f              # or sudo blkid /dev/sda1 — look at FSTYPE (vfat/exfat/ext4…)

# 4) Create a mount point and mount it (replace sda1 with your actual partition)
sudo mkdir -p /mnt/usb
sudo mount /dev/sda1 /mnt/usb
#   vfat/ext4 normally mount directly; exfat needs sudo apt install exfat-fuse first
#   (or kernel exfat support)

# 5) Always unmount before unplugging
sudo umount /mnt/usb
```

> ⚠️ **Gotcha (unplug without unmounting and the write may not have completed)**
> - **Scenario**: you copy files to the flash drive and pull it out the moment `cp` returns to the prompt.
> - **Symptom**: on another machine the files won't open, or their contents are incomplete.
> - **Cause**: for performance, Linux puts writes through a cache and gives no guarantee that everything has reached the medium by the time `cp` returns; unplug without `umount`/`sync` and whatever is still in the cache is lost.
> - **Prevention/fix**: always wait for `sudo umount /mnt/usb` to succeed before unplugging; if you're in a hurry to confirm, `sync` first.

> ⚠️ **Note (mount onto an empty directory)**: mounting **hides** whatever the mount point already contained — use a dedicated empty directory (such as `/mnt/usb`), never a directory that has things in it.

**Pass criteria**: after plugging the drive in, `lsblk` showing an extra `sdX`/`sdX1` and `dmesg` carrying the `[sda]` enumeration messages = the device is online; after mounting, `df -h /mnt/usb` showing the drive's capacity and `ls /mnt/usb` showing its files = the mount succeeded. (⏸ What is verified here is the baseline — no `/dev/sd*` with no medium attached; the mount itself is for you to run with your own drive.)

> Official source: the Manual `r01uh1032` §6.4 USB3.2 Gen2x1 Interface (USB3) (p1636–1640, functional overview; full register details in the User's Manual Additional Document). Board status: `07-hardware-unit-usage-guide.md` §42, `06-hardware-resource-map.md` §2.2.

---

## USB2.0 (General-Speed External Port, Includes OTG Silicon)

### What This Is (the Mechanism)

This chip includes **1 channel of USB2.0 OTG/DRD** (OTG = On-The-Go; dual Host/Function role) plus **1 host-only channel** (r01uh1032 §6.5.1, p1753). DRD (Dual-Role Device) means the same port can act as either host or device, but the switch here is **static** (decided at boot/configuration time), not negotiated dynamically on plug-in.

> ⚠️ **What this section describes are silicon channels, not physical ports.** Externally this board has only **2× USB3.2 Gen2 Type-A (CN2) + 1× micro-B (UART/function, not a data port)** — there is no separate USB 2.0 host connector (carrier board manual §3.6/§3.7). These USB2 channels are companions built alongside the USB3 ports: the USB2.0 root hubs that `lsusb` lists are xHCI controller enumerations and don't add pluggable physical ports. Don't read the root hub count as a port count.

- **ch0 (OTG/DRD)**: supports Host mode (10 ch PIPE, including the default control PIPE) and Function mode, with OTG (Rev. 2.0), Battery Charging, and DRD capability.
- **ch1**: Host mode only (p1754 Table 6.5-1/6.5-2).

Also a **simplified-version manual**, same as USB3 — full register specs are in the additional document.

### How You See This Under Linux

Shares the `xhci-renesas`/EHCI-HS driver path with USB3, and devices show up the same way, under `/dev/bus/usb/*`. On this board, ch0's silicon supports OTG, but it's currently **fixed in Host mode** (source: `07-hardware-unit-usage-guide.md` §43). USB20 Host base address is `0x15800000`, USB21 Host is `0x15810000`, and the USB20 Function sub-block is `0x15820000` (source: the Manual `r01uh1032` §1.8 Address Map, Table 1.8-1, p170).

The 2 USB 2.0 root hubs (ID `1d6b:0002`) that `lsusb` shows on the board are exactly these two paths (✅ Verified on the board; transcript: live/ch04b-runtime.txt):

```text
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

### Key Capabilities & Limits

- **Speeds**: High-Speed 480 Mbps / Full-Speed 12 Mbps / Low-Speed 1.5 Mbps (p1754).
- **OTG limitation**: verbatim, "Session Request Protocol (SRP) and Host Negotiation Protocol (HNP) are not supported" (p1753 Note 1) — OTG role switching **does not support dynamic negotiation**; it can only be set statically.
- **Battery Charging**: verbatim, "For this LSI, IDP_SRC will flow to a maximum of 32 μA" (p1753 Note 3).

### When You'd Actually Use This

**Mechanically**, USB2 is a "good enough" general-speed port — it leaves the high-speed bandwidth for USB3.

**Decision rule**:

- Ordinary-speed USB peripherals (think keyboard/mouse, UART-to-USB adapters, low-speed sensor modules) → USB2 is enough, no need to spend USB3 resources.
- Need this board to **flip around** and act as a USB device (say, presenting to an upstream host as a USB serial or mass-storage device) → you must use **ch0's Function mode**. But that means changing the currently fixed Host configuration, and because SRP/HNP aren't supported, the role switch has to be **statically configured** rather than negotiated automatically on plug-in — the deciding factor is "can you live with fixing the role at configuration time."

> Official source: the Manual `r01uh1032` §6.5 USB2.0 Interface (p1753–1757, functional overview; full register details in the additional document). Board status: `07-hardware-unit-usage-guide.md` §43, `06-hardware-resource-map.md` §2.2.

---

## GBETH + PTP (Wired Ethernet and Hardware Timestamping)

### What This Is (the Mechanism)

GBETH is a 2-channel Ethernet MAC (Media Access Control), compatible with the Synopsys DWMAC/EQOS architecture, conforming to IEEE 802.3-2008 — it supports three interfaces, Ethernet MAC, RGMII, and MII, but **does not support GMII** (r01uh1032 §6.3.1.1, p1632–1633). Each channel has 4 independent TX DMA and 4 RX DMA engines of its own, using native DMA, with descriptors that can be either dual-buffer (ring) or linked-list (chained) (p1632).

It has a capability that matters a lot for "multi-node time synchronization": built-in **IEEE 1588-2008 v2 hardware timestamping**, with a 125 MHz reference clock, compatible with both IEEE 802.1AS-2011 (gPTP) and IEEE 802.1Qav/Qat (audio/video time-sensitive networking, TSN) (p1632–1633). PTP (Precision Time Protocol) aligns multiple nodes on a network to the same time base, and hardware timestamping records the exact instant a packet crosses in or out of the network card, so software scheduling jitter doesn't pollute the time measurement. It also supports 31 MAC address filter registers, a 256-bit hash filter, and IEEE 802.3az (Energy Efficient Ethernet, including LPI low-power mode and Wake-on-LAN).

### How You See This Under Linux

Driver `stmmac`/`dwmac`, network interface name **`end0`** (corresponding to GBETH0, state UP, capability 1 Gbps). The hardware timestamp device is `/dev/ptp0`. GBETH0 base address is `0x15C30000`, GBETH1 is `0x15C40000`; GBETH1 (ch1)'s device tree node is `disabled` (✅ Verified on the board; transcript: live/ch04b-dt-status.txt — `ethernet@15C30000` is `okay`, `ethernet@15C40000` is `disabled`).

A naming trap beginners commonly hit, plus a measured link-speed reading:

> ⚠️ **Caution (the Ethernet interface is called `end0`, not `eth0`)**
> - **Scenario**: out of habit you type `ifconfig eth0`, or your config file says `eth0`.
> - **Symptom**: `eth0` can't be found.
> - **Cause**: this board's wired network interface is named **`end0`** (not `eth0`).
> - **Prevention/fix**: always use `end0`. Its PHY is a KSZ9131, capable of 1 Gbps — **the actual link speed depends on the switch and cable on the other end**, checkable via `cat /sys/class/net/end0/speed` (on the board this returns `100`, i.e. the link was up at 100 Mbps; capability is 1 Gbps, but link speed is a negotiated outcome — the two are not the same thing). The MAC address is randomly generated but persistent (format `xx:xx:xx:xx:xx:xx`, different on every board — check your own board's with `ip link show end0`, don't copy any example value) (source: `04-hardware-quickref.md`).

### Key Capabilities & Limits

- **Channel count**: 2 (only ch0 is enabled on the board).
- **Rate**: 10/100/1000 Mbps, full or half duplex (p1632).
- **Jumbo frames**: programmable frame length, up to 16 KB − 1, verbatim "Jumbo mode support in cut-through mode only (not implemented in store and forward due to TX and RX FIFO size)" (p1632) — jumbo frames are only supported in cut-through mode.
- **Timestamp base**: verbatim, "IEEE1588 time base information, with reference clock of 125 MHz" (p1633).
- **Interfaces**: RGMII and MII only, verbatim "GMII is not supported" (p1633).

### When You'd Actually Use This

**Mechanically**, `end0` is the board's primary wired network connection — firmware/image transfer, remote management, data upload, all of it goes through here; PTP is the extra "precise time" capability this network card happens to bring along.

**Decision rule**:

- Ordinary network traffic → just use `end0` as a standard network card, no need to touch PTP.
- Need **precise time synchronization across devices** (multiple nodes' data needs to align to the same time base — think synchronized multi-sensor sample timestamping, or distributed measurement systems) → that's when you'd reach for `/dev/ptp0` together with `linuxptp` (`ptp4l`/`phc2sys`). The deciding factor is "does your data need cross-node time alignment down to hardware-timestamp precision" — if not, don't bother.
- Need a **second** independent network interface → then go edit the device tree to enable ch1 (`15C40000`), and confirm the board actually has the matching PHY wiring in place (disabled ≠ wiring exists).

> Official source: the Manual `r01uh1032` §6.3 Gigabit Ethernet Interface (GBETH) (p1632–1635, functional overview). Other documents: `linuxptp` (software), PHY KSZ9131 datasheet (third-party). Board status: `07-hardware-unit-usage-guide.md` §44, `06-hardware-resource-map.md` §2.2.

---

## PCIe 3.0 (High-Speed Expansion Bus)

### What This Is (the Mechanism)

PCIe (PCI Express) is a point-to-point, high-speed expansion bus. This SoC's PCIe core is a **dual-role design**: the same core can be configured as either **Root Complex (RC, the active end, connecting to external devices)** or **Endpoint (EP, the passive end, appearing to some other host as an add-in card)**. That is why it has both Type 0 (for EP) and Type 1 (for RC) Configuration Registers built in (r01uh1032 §6.6.1, p2025).

There are 2 units total, and the lane configuration can be either a single 4-lane channel, or a multi-link configuration of 2-lane × 2 channels (channel 0/1 each with its own independent PCIe core, sharing the same chip but logically separate) (§6.6.1.1, p2027). It has a built-in DMA controller with 8 channels, in either descriptor or register control mode (p2026). It conforms to the PCI Express Base Specification 4.0, with actual line-rate support for Gen1 (2.5 GT/s) / Gen2 (5.0 GT/s) / Gen3 (8.0 GT/s) (p2026).

### How You See This Under Linux

Driver `rzv2h-pcie` (device tree compatible `pcie-rzv2h`, a Renesas Root Complex implementation), running through the standard PCI subsystem — you can inspect it with `lspci`, and devices show up under `/sys/bus/pci/devices`. The board has it enabled as **Root Complex**, with a memory window of `0x30000000`–`0x37FFFFFF` (matching the Manual's "PCIE0 area 128 Mbytes @ 0x30000000"); PCIE1 is unused (source: `07-hardware-unit-usage-guide.md` §45; `06-hardware-resource-map.md` §2.2). There's currently no endpoint attached on the board, so `lspci` usually only shows the root port itself.

### Key Capabilities & Limits

- **Unit count**: 2 (p2025). Only PCIE0 is used on the board.
- **Lane configuration**: verbatim, "Lane implementation x4 / x2 × 2ch (when Multilink is selected)" (p2026).
- **AXI interface limitation**: verbatim, "Little endian is only supported", and unaligned transfers are not supported on the master interface (p2025).
- **Data payload**: up to 256 bytes; read request size up to 512 bytes (p2026).
- **Virtual channel**: only VC0 is supported, no additional virtual channels (p2026).
- **Outstanding transfer count**: 1 to 8 (p2026).

### When You'd Actually Use This

**Mechanically**, PCIe offers "the highest-bandwidth expansion path," well suited to peripherals that need a lot of bandwidth and are themselves already a PCIe endpoint.

**Decision rule**:

- Need an external high-speed peripheral control chip (think an NVMe SSD to fix a storage bottleneck, a high-speed network card, an FPGA accelerator card), and that device is itself a PCIe endpoint → attach it to this board's PCIe Root Complex, and on the Linux side it's just the standard PCI device-probing flow.
- Conversely, if your scenario needs **this board itself** to be seen by another host as a PCIe add-in card (EP mode) → the silicon supports both RC/EP roles, but this board ships **fixed running as RC**. Using EP would require separately configuring the `SYS_PCIE_MODE` register and confirming firmware/device-tree support for that role — a path that has not been verified on the board, and one that falls under "needs to be opened up and verified yourself." The deciding factor is "is the board connecting to something else, or being connected to by something else."

> Official source: the Manual `r01uh1032` §6.6 PCI Express 3.0 Interface (PCIe) (p2025–2035, functional overview + block diagram; register details from §6.6.4 p2037 onward). Board status: `07-hardware-unit-usage-guide.md` §45, `06-hardware-resource-map.md` §2.2.

---

## ADC (12-Bit Analog-to-Digital Converter)

### What This Is (the Mechanism)

The ADC (Analog-to-Digital Converter) is a 12-bit **successive-approximation** analog-to-digital converter, with up to 8 selectable analog input channels (r01uh1032 §7.10.1, p3670). It converts an external analog voltage (say, a potentiometer's divided voltage, or the voltage drop across a current-sense shunt resistor) into a digital reading.

It has three operating modes (p3670):

- **Single scan**: each selected channel converts once, then stops.
- **Continuous scan**: keeps cycling through channels in order, continuously.
- **Group scan**: channels are split into 2 groups (A/B) or 3 groups (A/B/C), each with independently configurable trigger conditions; when a higher-priority group gets a conversion request, it **interrupts** a lower-priority group's scan that's already in progress. Priority order is fixed as A > B > C.

Conversion start is one of three trigger types (p3671 Table 7.10-1): software trigger, internal trigger (via a GPT timer plus an ELC event link), or external trigger (the ADTRG pin). Using GPT + ELC to trigger means the ADC gets started precisely on schedule by a hardware timer, with no need for the CPU to poke it every time.

### How You See This Under Linux

The ADC runs through Linux's **IIO (Industrial I/O)** subsystem, not as a character device — you **won't** `cat /dev/adc`; instead you read sysfs attributes. The device is `/sys/bus/iio/devices/iio:device0`, driven by `rzv2h-adc` (`.../name` returns verbatim `rzv2h-adc`; ✅ Verified on the board; transcript: live/ch04-followup.txt).

On-board measurement (✅ Verified on the board; transcript: live/ch04b-runtime.txt): each of the 8 channels has two attributes, `in_voltage0..7_raw` and `in_voltage0..7_get_value`; reading `in_voltage0_raw` returns `2819`. **This board's driver doesn't expose an `in_voltage_scale` attribute**, so converting to an actual voltage means calculating it yourself from the 12-bit resolution and the 0–1.8 V reference range:

```text
Voltage ≈ raw ÷ 4095 × 1.8 V
Example: raw 2819 → 2819 ÷ 4095 × 1.8 ≈ 1.24 V
```

(Source: `07-hardware-unit-usage-guide.md` §37; unit-map.)

### Key Capabilities & Limits

- **Unit/channel count**: 1 unit, up to 8 channels (p3670).
- **Resolution**: selectable 12-bit or 8-bit (p3671).
- **Sample rate** (depends on the A/D conversion clock, verbatim, p3671): "2.5 Msps (when A/D conversion clock ADC_0_ADCLK is 50 MHz) / 2.0 Msps (40 MHz) / 1.0 Msps (20 MHz) / 0.5 Msps (10 MHz) / 0.25 Msps (5 MHz)".
- **Reference voltage range**: 0–1.8 V (the basis for the voltage conversion; this board doesn't expose a scale attribute, so you need to calculate it yourself).

### When You'd Actually Use This

**Mechanically**, the ADC is the board's entry point for reading "non-digital-interface" signals — any sensor that outputs an analog voltage goes through it.

**Decision rule**:

- Reading an analog sensor's output (think a potentiometer, a current-sense shunt resistor reading, a battery voltage divider, a photoresistor — a barometer, current sensor, or servo position feedback on a mobile vehicle is one application) → use the ADC.
- **The choice of operating mode** (this isn't a spec threshold — it's chosen based on your measurement needs): only need a single-point reading, or a signal that doesn't change often → `single scan` is enough; need to continuously monitor the same batch of channels (think real-time monitoring of several analog signals) → `continuous scan`; some signals have priority over others (say, a safety-relevant signal that needs to interrupt background measurement and get read first) → then use `group scan` with priority settings. The deciding factor is "does your signal need to be real-time, and does it need priority levels."

### Hands-On: Reading the ADC Through IIO and Converting to Volts

The ADC goes through the IIO subsystem, so reading a value is just `cat` on a sysfs attribute. The flow is four steps: **enumerate the device → read raw → convert to a voltage → check whether the reading is plausible.**

**Step 1: enumerate the IIO devices and confirm this is the ADC.**

```bash
ls -l /sys/bus/iio/devices/
cat /sys/bus/iio/devices/iio:device0/name
ls /sys/bus/iio/devices/iio:device0
```

On the board (✅ Verified on the board; transcript: `live/ch4-w1-adc.txt`): `iio:device0` links to `.../11c00000.adc/`, `name` returns verbatim `rzv2h-adc`, and the attributes give each channel a pair of `in_voltageN_raw` and `in_voltageN_get_value`, 8 channels in total (0–7). **Note that there is no `in_voltage_scale` in that list** — this board's driver doesn't expose a scale attribute, so the conversion is yours to do (step 3 below).

**Step 2: read the raw digital values.**

```bash
for f in /sys/bus/iio/devices/iio:device0/in_voltage*_raw; do
  echo "${f##*/}=$(cat $f)"
done
```

On the board (✅ Verified on the board; transcript: `live/ch4-w1-adc.txt`):

```text
in_voltage0_raw=2816
in_voltage1_raw=2318
in_voltage2_raw=1914
in_voltage3_raw=1734
in_voltage4_raw=1648
in_voltage5_raw=1607
in_voltage6_raw=1613
in_voltage7_raw=1601
```

`raw` is a 12-bit digital code in the range 0–4095 (0 = 0 V, 4095 = full scale).

**Step 3: convert to an actual voltage.** Since this board has no scale attribute, work it out yourself from the 12-bit resolution and the 0–1.8 V reference range:

```text
voltage ≈ raw ÷ 4095 × 1.8 V
e.g.: raw 2816 → 2816 ÷ 4095 × 1.8 ≈ 1.24 V
      raw 2318 → 2318 ÷ 4095 × 1.8 ≈ 1.02 V
```

**Step 4: sanity-check the reading — and this is where you hit the key trap.** Read the same attributes again a few seconds later and the numbers move: `in_voltage0_raw` reads `2816` one moment and `1311` (≈ 0.58 V) shortly after — same channel, nothing changed, and the reading jumped from 1.24 V to 0.58 V (verified on the board; transcript: `live/ch4-w1-adc.txt`).

> ⚠️ **Gotcha (an ADC reading from a floating pin drifts and is not a measurement)**
> - **Scenario**: you `cat in_voltageN_raw`, convert the number to a voltage, and treat it as the signal you measured.
> - **Symptom**: two consecutive reads of the same channel differ wildly (`2816` → `1311`), and the converted voltage jumps around with no reproducibility.
> - **Cause**: those ADC channels have **no signal source connected** — the inputs are floating, and a floating pin picks up noise and leakage potentials, so every conversion legitimately lands somewhere different. It isn't a valid measurement.
> - **Prevention/fix**: before measuring, put a **known voltage within the 0–1.8 V range** on the channel (a divider off the 1.8 V rail, or a lab supply), then read raw and check that the converted value is close to what you expect. A reading from a floating pin is never a result.

> ⚠️ **Note (don't exceed the reference range)**: the ADC's reference range is 0–1.8 V. Keep the input **below 1.8 V**; going above it can damage the pin. To measure a higher voltage, divide it down into range first.

⏸ **Where the verification stops**: what is established here is device enumeration, raw reads, the conversion formula, and the drift behavior of a floating pin. What is **not** covered is absolute accuracy — applying a known voltage and confirming that raw matches the expected value — which needs a reference source on the input. Verifying the whole conversion chain requires one.

**Pass criteria**: `name` returning `rzv2h-adc` and 8 channels each having `in_voltageN_raw` = driver and device ready; with a known voltage applied, `raw ÷ 4095 × 1.8` matching the actual input (within the ADC's accuracy) = the conversion chain is correct. A reading that keeps drifting with no input applied means that pin is floating and the reading is not valid.

> Official source: the Manual `r01uh1032` §7.10 12-Bit A/D Converter (ADC) (p3670–3674, functional overview; register details from §7.10.2 p3675 onward). Other documents: `libiio` (software). Board status: `07-hardware-unit-usage-guide.md` §37.

---

## TSU (On-Chip Temperature Sensing Unit)

### What This Is (the Mechanism)

The TSU (Temperature Sensor Unit) is an on-chip temperature sensing unit, made up of an analog primary sensor plus a dedicated ADC, converting the sensor's analog output into a digital code corresponding to temperature (r01uh1032 §7.11.1, p3739). The conversion result can be read over the APB bus, and can be configured to compare against upper/lower temperature limits and trigger an interrupt when out of range (p3739).

Conversion can be started one of two ways: software triggers it via a register setting, or ELC (Event Link Controller) triggers it. The conversion mode is fixed as single scan, and you can configure the sample count for averaging to reduce noise (p3739). There are **2 independent TSUs (TSU0/TSU1)** on the chip, each connected separately to the system bus, the CPG (clock supply), and the ICU (interrupt control) (§7.11.2, p3740). This sensor measures the temperature of the **chip itself (the die)** — note, not the ambient environment.

### How You See This Under Linux

The TSU runs through Linux's **thermal framework**, exposed as `/sys/class/thermal/thermal_zone0` and `thermal_zone1` (the two TSUs map to the two zones), driven by `rzv2h_thermal`. On-board readings sit at roughly **34–36°C** (the spread covers several measurement runs: `07-hardware-unit-usage-guide.md` §38 records 34–35°C, while other runs read 35/36 — quote a single figure only together with the conditions it was taken under; see also the thermal measurements in `05-compute-benchmark.md`). Reading the temperature is just `cat /sys/class/thermal/thermal_zone0/temp` (returns millidegrees, e.g. `35000` = 35°C); it doesn't go through `/dev`.

### Key Capabilities & Limits

- **Unit count**: 2 (TSU0/TSU1) (p3739; unit-map).
- **Accuracy specs** (resolution 0.0625°C/code, measurement range −40～125°C, accuracy ±5°C, 14.9 ksps): **this set of values carries a source-level honesty flag** — the page range this handbook actually read directly (p3739–3741) is the functional overview, connection diagram, and pin description; it **does not cover** the register-spec page that lists these values (§7.11.6). This set of numbers is sourced from `07-hardware-unit-usage-guide.md` §38 (which transcribes them from r01uh1032 Table 7.11-5/7.11-6), and is **not** something this handbook has directly checked word-for-word against the PDF. When citing these accuracy figures, know that they're a secondhand transcription, still pending a direct check against §7.11.6.

### When You'd Actually Use This

**Mechanically**, the TSU measures **in-package (die)** temperature, used for thermal protection or getting a rough sense of how hot or cold the system is running.

**Decision rule**:

- Need to **automatically throttle or pause high-power compute units** in response to heat (think long-duration, high-load NPU/GPU workloads) to avoid a thermal shutdown → reading `thermal_zone*` temperature values combined with a custom throttling policy is a common approach.
- **Important boundary**: the TSU reads the chip's internal temperature, which **cannot** be used directly as ambient temperature — in a system with active cooling or chassis airflow, die temperature and ambient temperature can differ noticeably. If what you want is enclosure/ambient temperature, you need a separate external temperature sensor (say, through the ADC or an I²C sensor). The deciding factor is "are you protecting the chip, or measuring the surrounding environment" — these two jobs call for different sensors.

> Official source: the Manual `r01uh1032` §7.11 Temperature Sensor Unit (TSU) (p3739–3745, functional overview; register details from §7.11.6 p3746 onward — the accuracy-values page, not directly checked by this handbook). Board status and accuracy transcription: `07-hardware-unit-usage-guide.md` §38, `05-compute-benchmark.md` (thermal).

---

## Tying This Group Together: Decision Rules for Choosing an Interface

Half of these 14 units are interfaces for "getting the board to talk to the outside world." The question beginners ask most often isn't "what is this interface," it's "for this particular device, which one should I actually use." Here are the decision rules from each section above, collapsed into one lookup table — **look first at what protocol the other-end device supports, then at your measurement needs**:

| Your need | First choice | Why (the mechanism) |
|---|---|---|
| A debug/console serial port that has to be there from boot | **SCIF** | Shares hardware with boot firmware, always available |
| Multiple independent serial links, or a lightweight SPI/I²C/LIN personality | **RSCI** | 10 channels, six selectable modes per channel (needs a DT edit to enable on the board) |
| Point-to-point high speed, other end is standard SPI | **RSPI** | Has a clock line, full-duplex, speed over pin flexibility |
| Multiple devices on a shared bus, other end ≤1 Mbps, wiring simplicity matters | **I²C (RIIC)** | Two wires plus address-based addressing, most pin-efficient for many devices |
| Other end supports I3C, need in-band interrupt/hot-join | **I3C** | I²C's wiring, upgraded capability (not wired up on this board) |
| Multi-node, noise-resistant, deterministic, other end is a CAN node | **CAN-FD** | Hardware arbitration + error retransmission (release the transceiver from standby before use) |
| Reading an analog voltage sensor | **ADC** | The only analog input path |
| Monitoring chip temperature for thermal protection | **TSU** | Measures die temperature (not ambient) |
| High-bandwidth external device (SSD/UVC camera) | **USB3.2** | Up to 10 Gbps (host-only) |
| Ordinary-speed USB peripheral | **USB2.0** | Good enough, leaves the bandwidth for USB3 |
| Wired networking; need precise cross-node time sync | **GBETH (+ PTP)** | Standard network card; `/dev/ptp0` for hardware timestamping |
| Connecting a high-speed PCIe endpoint (NVMe/accelerator card) | **PCIe** | Highest-bandwidth expansion path (RC on this board) |
| Simple digital I/O, driving a chip's enable pin | **GPIO (PFC)** | Run `gpioinfo` first to confirm the pin isn't already multiplexed away |
| Hardware data-integrity checksum | **CRC** | Offloads the cost of computing CRC in software (not inventoried on this board — probe before use) |

This table is deliberately laid out as "need → interface" rather than "interface → use case," because during system integration, what you have in hand first is "a device I need to connect," and you work backward from there to figure out which bus to use. The full mechanism, capability boundaries, and board status behind any row in this table are in that row's corresponding unit section above. Anything flagged "needs a DT edit / not wired up / not inventoried / confirm before use" is a reminder: **existing on paper doesn't mean this particular board can use it right now** — which is exactly the point Chapter 4's file 00 (the 4.1 overview and the six-state table) keeps hammering: "enabled" has layers.

> **This group's data fidelity statement**: the mechanisms and capability figures above are quoted verbatim from the official hardware manual `r01uh1032ej0130` (page numbers cited inline); board status and device nodes are drawn from the development records `06-hardware-resource-map.md` (doc06), `07-hardware-unit-usage-guide.md` (doc07), and `04-hardware-quickref.md`; entries marked ✅ are verbatim output verified on the board (transcripts under live/ch04*.txt, cited inline at each point). **Two honest boundaries**: the CRC operation unit's on-board status has not been inventoried (only the Manual's specs are cited); the TSU accuracy figures are a secondhand transcription from doc07 (not directly checked against the §7.11.6 register page) — both are flagged in place in their own sections, with no gap filled by guesswork. The FSP RSPI/CANFD manuals and the WS125 RDK carrier board manual have not been opened and verified file-by-file; wherever their content is cited, the source is noted and the citation is not extended beyond what's given.
