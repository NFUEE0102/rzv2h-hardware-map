# RZ/V2H Hardware Resource Map

> Maintained by the **Power Conversion Technology Research Center**, Department of Electrical Engineering, National Formosa University — <https://nfuee0102.com> ｜ 國立虎尾科技大學 電機工程系 電能轉換技術研究中心

> Companion article: [RZ/V2H hardware resource map — what exists, its state, and how to verify it](https://nfuee0102.com/en/notes/2026-09-rzv2h-hardware-map/) ｜ [中文版](https://nfuee0102.com/notes/2026-09-rzv2h-hardware-map/)

Languages: English (this page) · [繁體中文](zh-TW/README.md)

A from-scratch, verify-everything map of what hardware actually exists on a
Renesas RZ/V2H RDK board (SoC part number R9A09G057H44GBG, the RZ/V2HP
variant), what state each piece is in, and how to check that state yourself
instead of taking anyone's word for it — including this document's.

## Why this exists

This is prerequisite reading, not a tutorial. Before you build anything on
this board — a camera pipeline, a real-time control loop, a custom driver —
you need an honest inventory of what's actually turned on, what's present
but not wired up to Linux, and what simply isn't populated on this exact
part number. Getting that inventory wrong wastes real time: chasing a
`/dev` node that will never appear because the hardware it belongs to isn't
on this die, or assuming something is broken when "not found" is actually
the correct, expected answer.

This repo is meant to be read *before* a hands-on course or workshop on this
board, so the actual teaching time can go to the interesting parts instead
of re-deriving "what hardware do we even have."

## The core idea: six ways "I can't find it" can be true

The single most useful concept in this map, used consistently across every
file:

| Status | Meaning | How to think about it |
|---|---|---|
| **Enabled** | Device tree marked `okay`, Linux driver bound, a usable `/dev` or sysfs node exists | Ready to use |
| **Partially Enabled** | Multiple instances of the same hardware class exist; only some are `okay` in the device tree | The enabled ones work; the rest need a DT change |
| **Present · Not Exposed to Linux** | The silicon has it, but running Linux has no driver node for it — only reachable via firmware (U-Boot/CM33/CR8) or an external probe | Not broken — it's just not Linux's to manage |
| **Reserved** | The hardware is enabled, but the whole block is claimed by boot firmware / remoteproc / a memory carveout, and never added to Linux's general-purpose pool | Exists, but not available for reuse |
| **Disabled** | Device tree explicitly marks it `disabled` | Needs a DT change *and* the hardware hooked up to come alive |
| **Not Populated** | This exact silicon (R9A09G057**H44**) simply doesn't have the module — a sibling part number does | Stop looking here; it's not on this chip |

Every "I looked for X and didn't find it" in this repo resolves to exactly
one of these six, and each file tells you which one and why.

## Layout

| File | Covers |
|---|---|
| [00-overview-and-ip-enablement-map.md](00-overview-and-ip-enablement-map.md) | Whole-board overview: the enablement map, how to read "enabled/disabled" counts without being misled by them, the six-state taxonomy in full, grouped summary tables, reserved memory, the clock tree |
| [01-compute-units.md](01-compute-units.md) | The CPU/GPU/accelerator complex: 4× Cortex-A55, Cortex-M33, 2× Cortex-R8, Mali-G31 GPU, DRP-AI3 |
| [02-video-capture-codec-display.md](02-video-capture-codec-display.md) | Camera capture (CRU/CSI-2), video codec, display output |
| [03-audio-subsystem.md](03-audio-subsystem.md) | Audio I/O and DSP |
| [04-memory-and-storage.md](04-memory-and-storage.md) | LPDDR4/4X, xSPI flash, eMMC, MTD layout |
| [05-system-backbone-interrupts-clocks-power-dma-event-link.md](05-system-backbone-interrupts-clocks-power-dma-event-link.md) | Interrupt controller, clock/power generation, DMA, event link |
| [06-timing-system-timers-pwm.md](06-timing-system-timers-pwm.md) | Timers (GTM/OSTM) and PWM generation |
| [07-communication-and-sensing-interfaces.md](07-communication-and-sensing-interfaces.md) | Serial (UART/I2C/SPI), buses, networking (GbE/PCIe), expansion, analog |
| [08-debug-and-security.md](08-debug-and-security.md) | CoreSight debug/trace, TrustZone + TZC-400, Trusted Secure IP, OTP |
| [09-official-documentation-guide.md](09-official-documentation-guide.md) | Where to find the answer in Renesas's own documents when this map isn't enough |

Read `00` first — it defines the vocabulary and conventions every other file
relies on. After that, the `02`–`08` files are independent; read whichever
subsystem you're about to touch.

## What's *not* in this repo

Every file cites two internal investigation documents (referred to inline
as "doc06" and "doc07") and a set of raw on-board verification transcripts
from the original project this was extracted from. Those aren't included
here — the citations are kept as provenance ("this number came from a live
measurement, not a guess"), not as links you're expected to follow. Where a
claim was verified directly against Renesas's official hardware manual or
datasheet, the citation says so explicitly (manual section number, page
range). Where it was carried over from one of those internal documents
without independently checking the original page, that's flagged too —
this map draws a hard, honest line between "I read this myself" and
"someone else's notes say this," and preserves that line in translation.

The official Renesas documents themselves (`r01uh1032` hardware manual,
`r01ds0429` datasheet) are not included — they're Renesas's copyrighted
material. `09-official-documentation-guide.md` tells you what to look for
and where.

## Source and sync status

The content here mirrors the internal handbook chapter "全板硬體資源地圖"
(Full-Board Hardware Resource Map) as of **2026-09-12**, and the English text
has been updated to match that state. The Traditional Chinese originals of
all ten chapter files are kept alongside the translation in
[`zh-TW/`](zh-TW/README.md), so you can check any passage against the source
it was translated from. [CHANGELOG.md](CHANGELOG.md) records what changed in
each sync.

## Provenance

Translated from the original Chinese-language hardware resource map chapter
of a larger onboard-development handbook for an RZ/V2H-based project. The
underlying research was done by direct, hands-on verification against the
official hardware manual and datasheet plus live testing on the physical
board — not summarized secondhand from other write-ups. This translation
preserves that content as faithfully as possible: numbers, register
addresses, and citations are unchanged from the original.

## License

Creative Commons Attribution 4.0 International (CC BY 4.0) — see
[LICENSE](LICENSE). You can share and adapt this freely, including
commercially, as long as you credit the source.

Copyright (c) 2026 Jian-You Chen
