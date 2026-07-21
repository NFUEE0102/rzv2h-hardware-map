# 99 · Official Documentation Guide (4.4)

> This file is the official-documentation guide for the "04 · Whole-Board Hardware Resource Map" folder (corresponding to chapter number **4.4**). The chapter's opening explanation and the 4.1 overview are in [00-overview-and-ip-enablement-map.md](00-overview-and-ip-enablement-map.md), the compute units are in [01-compute-units.md](01-compute-units.md), and the individual peripheral units are in the g2–g8 group files — the official Manual's chapter numbers and page numbers cited in those files can all be looked up yourself using the method this file teaches.

In the earlier files you've already used a huge pile of hardware facts — the register base address of some unit, which clock some bus runs on, where DRP-AI3's 512 MB reserved region starts. Not one of these numbers was made up out of thin air; every one of them can be traced back to its original source in the official-documentation folder that ships alongside the handbook repo. The purpose of this file is to turn "that stack of documents" into **a map you can look things up in yourself**: when a question comes up, you should be able to immediately tell which document to flip to and which chapter to flip to, instead of scrolling through a 43 MB manual from front to back.

Knowing how to look up documentation matters more than memorizing any single number. Every measured value in this handbook has measurement conditions attached, and every one of them can vary slightly depending on the board revision or image you're running; but "how to look up the official documentation" is a skill for life — whatever new peripheral you hook up next, whatever new register you configure, whatever new spec you need to check, the answer is in the same stack of documents.

## Table of Contents

- [Why It's "a Stack," Not "a Single Book"](#why-its-a-stack-not-a-single-book)
- [What's Actually in the Folder: 9 Documents](#whats-actually-in-the-folder-9-documents)
- [The Manual Is Too Big, So Use `_toc_full.txt` as the Index](#the-manual-is-too-big-so-use-_toc_fulltxt-as-the-index)
- [Which Chapter to Check: Common Units → Chapter/Page List](#which-chapter-to-check-common-units--chapterpage-list)
- [Only Need Spec Numbers: The Datasheet's Section 1 Is Enough](#only-need-spec-numbers-the-datasheets-section-1-is-enough)
- [That AWO/Firmware Deployment Document (Advanced, Know Where It Is For Now)](#that-awofirmware-deployment-document-advanced-know-where-it-is-for-now)
- [Official Online Resources](#official-online-resources)
- [Hands-On Verification: Confirm You Can Use This Stack of Documents as a Map](#hands-on-verification-confirm-you-can-use-this-stack-of-documents-as-a-map)

## Why It's "a Stack," Not "a Single Book"

The most common misconception beginners have is assuming the chip vendor will hand you "one giant compendium" that has everything in it. In reality, Renesas (like almost every SoC vendor) **splits information up by purpose into several separate documents**, each one answering a different kind of question. Once you understand this division of labor, you won't go hunting through the datasheet for a register offset (you won't find it), and you won't lug around the 4800-page Manual just to check "how many CAN channels does this chip actually have" (too slow).

This stack of documents breaks down into roughly five roles, each answering one kind of question:

```text
  The question in your head                     Which document to check            What level it answers
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────
 "What does this chip have? How many?"    →   Datasheet (spec summary table)     "what / how many"
 "How do I set this unit's registers?"    →   User's Manual: Hardware            "register / bit / address"
                                               (hardware manual, register-level)
 "How do I get this feature running?"     →   Application Note /                 "steps / procedure"
                                               Quick Start (application/bring-up)
 "Usage & params for a software           →   User's Manual: Software            "element / API / pipeline"
  interface (e.g. GStreamer)?"                (software manual)
 "Why was it designed this way?"          →   White Paper                        "architecture / rationale / trade-offs"
```

There's also a **level**-based division you need to get straight first: everything above is about the **SoC (the chip)**; whereas "where does this **board's** connector go, what's on which pin of the 40-pin header, how do the DIP switches get set" is a **carrier-board (board) level** question, and you look that up in the RDK board manual (see the WS125 document below) — the chip manuals have no answers for these.

It's enough to remember this table as a single rule of thumb: **ask "what is it, how many" → check the datasheet; ask "how do I set the registers" → check the Manual; ask "how do I get it running" → check the application note; ask "how do I use the software interface" → check the software manual; ask "why" → check the white paper; ask "how does the board wire up" → check the board manual.**

> 💡 **Tip**: Renesas's document-number prefix tells you what kind of document it is all by itself — you can guess correctly nine times out of ten without even opening it. `R01DS` = Datasheet, `R01UH`/`R16UH` = User's Manual: Hardware (hardware manual; `R16UH` is the board level), `R01US` = User's Manual: Software (software manual), `R01AN` / `R20AN` = Application Note, `R01QS` = Quick Start, `R01WP` = White Paper. Once you see these letters at the start of a filename, you already know which level of question it answers (compiled from Renesas's document-numbering convention; each document's actual identity has been individually verified against its self-reported title in the table below).

## What's Actually in the Folder: 9 Documents

The official documents live in `reference-docs/`, at the same level as the handbook repo (as a relative path from this file's folder, that's `../../reference-docs/`). Let's list it first and see what's in there:

```bash
ls -la ../../reference-docs/
```

Actual output (verbatim, run in this folder on 2026-07-18; the date/time column reflects when you cloned the repo, so yours will differ):

```text
total 53284
drwxrwxr-x 2 user user     4096 Jul 18 16:21 .
drwxrwxr-x 7 user user     4096 Jul 18 16:21 ..
-rw-rw-r-- 1 user user  1105559 Jul 18 16:21 r01an7723ej0400-rzv2h-rzv2n-awo-example-program-startup-guide.pdf
-rw-rw-r-- 1 user user   434057 Jul 18 16:21 r01an7912ej0104-rz-family-dram-list.pdf
-rw-rw-r-- 1 user user   264840 Jul 18 16:21 r01ds0429ej0130-rzv2h.pdf
-rw-rw-r-- 1 user user  2254319 Jul 18 16:21 r01qs0077ej0400-rzv2h-multi-os-pkg.pdf
-rw-rw-r-- 1 user user 43861979 Jul 18 16:21 r01uh1032ej0130-rzv2h.pdf
-rw-rw-r-- 1 user user  2918713 Jul 18 16:21 r01us0653ej0202-rzv2h_rzv2n_GStreamer_UME.pdf
-rw-rw-r-- 1 user user  1396682 Jul 18 16:21 r01wp0022eu0100-rzv2h-drp-ai3.pdf
-rw-rw-r-- 1 user user   365249 Jul 18 16:21 r20an0842ea0401-rzv2h-evk-exampleprojects.pdf
-rw-rw-r-- 1 user user     2500 Jul 18 16:21 README.md
-rw-rw-r-- 1 user user  1684127 Jul 18 16:21 REN_WS125V2HRDKREFZ_MAH_20260323.pdf
```

Line up the nine `.pdf` filenames and run them through the prefix trick above, and you can already tell who's who; the other two non-PDF files are `README.md` (the folder's own description) and `_toc_full.txt` (the Manual's complete table of contents — the entire next section is about it. It's not listed in the output above because it starts with an underscore and sorts last; the last line of the full `ls -la` output is `246904 … _toc_full.txt`).

> **What "document count / file size" is based on**: The document list and byte counts in this section are all based on the `ls -la` output shown above, run directly against `../../reference-docs/` (9 PDFs; the datasheet `r01ds0429` is a 264,840 B text conversion). The transcript `live/ch04b-docs-local.txt` cited in the ✅ verification steps further down is used only to back up the `grep` results against `_toc_full.txt` (chapter page numbers, the CAN-FD/I2C line counts, etc.) — it is **not** the basis for the document list or file sizes. If you have your own `ls` output run against a different copy of these documents (e.g., the count or the datasheet size doesn't match), just go with the one you ran yourself against `reference-docs/` — the two are different copies, not a contradiction.

The table below lays out each document's role, size (byte counts taken verbatim from the `ls -la` above), and **how thoroughly it's actually been read so far** — the "read status" column especially deserves a look, since it determines how much you can trust this handbook's description of that document. Page counts and each file's self-reported title were read out by running `pdfinfo` (poppler-utils) on the workstation, 2026-07-18:

| Filename (prefix type) | Size | What question to check it for | Read status / format notes |
|---|---|---|---|
| `r01ds0429ej0130-rzv2h.pdf` (R01DS · Datasheet, Rev.1.30, 2025-09-05, 144 pages) | 264,840 B (≈259 KB) | RZ/V2H chip **spec summary table**: "how many CAN channels / how many channels / what's the max MHz for this chip", "what's different between part numbers" — check Section 1 (Table 1.3-x subtables, Table 1.4 unit-abbreviation cross-reference) | **Section 1 Overview has been read**. ⚠️ Format note: despite the `.pdf` extension, the content is actually a **plain-text conversion** (the very first line self-reports `R01DS0429EJ0130 Rev.1.30 Page 1 of 144`) — a normal PDF reader can't open it; just read it with a text editor or `grep`. To see **figures** (like the block diagram) you'll need to get the original Renesas PDF separately |
| `r01uh1032ej0130-rzv2h.pdf` (R01UH · Hardware Manual, Rev.1.30, 4816 pages) | 43,861,979 B (≈43.9 MB) | The complete **Hardware User's Manual (register level)**: every unit's register offsets, bit definitions, bare-metal addresses, chapter-level detail — everything about "how do I set the registers" is in here | **Not read page by page**; its **table of contents** has been extracted into `_toc_full.txt` as an index (see the next section); `pdfinfo` self-reports the title `RZ/V2H Group User's Manual: Hardware (Non-Agreement)`, `Pages: 4816` |
| `r01an7723ej0400-…-awo-…-startup-guide.pdf` (R01AN · App Note, 17 pages) | 1,105,559 B (≈1.1 MB) | RZ/V2H・RZ/V2N **AWO (CM33 wake/sleep control) example-program startup guide**: CM33 firmware deployment, the Yocto build flow | **Fully read via the corresponding MD** (key points and gotchas are in "That AWO/Firmware Deployment Document" at the end of this file) |
| `r01qs0077ej0400-rzv2h-multi-os-pkg.pdf` (R01QS · Quick Start, 32 pages) | 2,254,319 B (≈2.3 MB) | **Multi-OS Package Quick Start**: the first document to flip through when you want R8/M33 to run firmware and have A55 talk to them over RPMsg (this is the document number R01QS0077 mentioned in file 01) | **Body text not read**; `pdfinfo` self-reports the title `RZ/V2H Quick Start Guide for RZ Multi-OS Package` (title verified; the content description is based on the title and document number) |
| `r01us0653ej0202-rzv2h_rzv2n_GStreamer_UME.pdf` (R01US · Software Manual, Rev.2.02, 89 pages) | 2,918,713 B (≈2.9 MB) | **GStreamer UME (Linux Interface Specification GStreamer User's Manual)**: parameters, buffer management, and pipeline recipes for elements like `v4l2src`/`vspmfilter`/`omxh264enc` — this is the kind of manual that's the official basis for the GStreamer syntax used in the Chapter 2 video pipeline | **Body text not read page by page**; `pdfinfo` self-reports the title `RZ/V2H Group and RZ/V2N Group User's Manual: Software` (title verified; the "element/buffer management" role description carries over from the existing entry in the repo's `reference-docs/README.md`) |
| `r01wp0022eu0100-rzv2h-drp-ai3.pdf` (R01WP · White Paper, 10 pages) | 1,396,682 B (≈1.4 MB) | **DRP-AI3 architecture white paper**: where the 8/80 TOPS figures come from, sparse pruning, the 10 TOPS/W and fanless goals — check this for "why was it designed this way" | Has a corresponding MD already (read as part of the GPU/NPU deep-dive task) |
| `r01an7912ej0104-rz-family-dram-list.pdf` (R01AN · App Note, 12 pages) | 434,057 B (≈434 KB) | Presumed to be the RZ family's **compatible DRAM (LPDDR4/4X) part-number list** — check this when picking memory chips for a custom carrier board | **Not read, listed by name only** (purpose inferred from the document number and filename) |
| `r20an0842ea0401-rzv2h-evk-exampleprojects.pdf` (R20AN · App Note, 12 pages) | 365,249 B (≈365 KB) | Describes the RZ/V2H **EVK example project bundle** — the map for finding official sample programs | **Body text not read**; `pdfinfo` self-reports the title `RZV2H-EVK Example Project Bundle` (title verified) |
| `REN_WS125V2HRDKREFZ_MAH_20260323.pdf` (R16UH · **board**-level hardware manual, R16UH0052EU0100 Rev.1.00, 2026-03-23, 13 pages) | 1,684,127 B (≈1.7 MB) | **WS125-V2HRDKREFZ RDK carrier-board manual**: connectors and pinout (40-pin header, camera connector), DIP switches, SD-card connector and recommended cards, the flash chip populated on the board — check this for **board-level** questions; the chip manuals don't have any of this | ⚠️ Format note: despite the `.pdf` extension, this is actually a **ZIP archive** (contains page images `1.jpeg`–`13.jpeg` + page text `1.txt`–`13.txt` + `manifest.json`), a normal PDF reader can't open it — extract it first (`unzip`) and then view the page images or text. Development notes have previously transcribed it into an MD (`ws125-rdk-board-manual.md`) citing Table 1 (on-board flash) and Section 3.9 (SD-card connector) |

> ⚠️ **Note (two "fake PDFs" — if it won't open, the file isn't broken)**:
> **Situation**: You open `r01ds0429ej0130-rzv2h.pdf` (the datasheet) or `REN_WS125V2HRDKREFZ_MAH_20260323.pdf` (the board manual) in a PDF reader.
> **Symptom**: The reader reports a format error and won't open it; `pdfinfo` also comes back with `May not be a PDF file`.
> **Cause**: These two files simply **kept the `.pdf` extension** — the datasheet's actual content is a plain-text conversion (the `file` command classifies it as `data`, and it starts with the document's self-reported header `R01DS0429EJ0130 Rev.1.30 Page 1 of 144`); the board manual is actually a ZIP (`file` classifies it as `Zip archive data`, containing page images and text).
> **Prevention/workaround**: Read the datasheet directly with a text editor / `less` / `grep` (everything this handbook cites from it is text, and that's plenty); `unzip` the board manual first and then view it. For the original layout and figures, download them separately from the Renesas website by document number (R01DS0429, R16UH0052).

> ⚠️ **Note**: For the documents marked "not read" or "body text not read" in the table's "read status" column, their "what question to check it for" was written **based on document-numbering convention plus the file's self-reported title** — the body text has not been read page by page.
> **Situation**: Following the table above, you write directly in a report or design document, "see `r01an7912` for the DRAM compatibility list."
> **Symptom**: When you actually open it, the content may not match what you assumed it was for (e.g., different coverage, different applicable part numbers).
> **Cause**: The descriptions for these documents come from inferring "number prefix + filename + title" — not from a conclusion reached by reading the body text. The inference is usually right, but it's not guaranteed.
> **Prevention**: Before **citing these documents precisely**, open the file yourself and page through it to confirm; treat "inferred purpose" as a lead for "roughly which document to check first," not as a fact you can restate directly.

## The Manual Is Too Big, So Use `_toc_full.txt` as the Index

Someone who knows how to look things up spends 80% of their time on that 43.9 MB Manual (`r01uh1032ej0130-rzv2h.pdf`) — because it's the only document with register-level detail. But it's 4816 pages, and you're not going to want to page through the whole thing. The good news is the folder already has its **complete table of contents** extracted into a plain-text file, `_toc_full.txt` (246,904 bytes), and you can `grep` through it to find which chapter and page any unit falls on in seconds.

**Let's be honest about one thing first: this file has no title page identifying itself.** Open `_toc_full.txt` and the beginning is `Cover` / `Notice` / `How to Use This Manual` / `Table of Contents` — there's no line that says "I am r01uh1032." So the statement "this is the Manual's table of contents" is an **inference** — but it's an inference backed by four pieces of hard evidence you can verify yourself:

1. **The chapter numbers match, one by one.** This table of contents' chapter numbers line up exactly with the "hw_manual chapter" citations used across this chapter's files: `4.6 Interrupt Controller` is indeed the GIC chapter, `7.4 SCIF`, `9.2 Camera Data Receiver Unit (CRU)`... every single one checks out. If this were some other document, the numbers wouldn't coincidentally hit every single time.
2. **The page-count order of magnitude matches.** This table of contents runs all the way up to around page 4816 (ending with `APPENDIX A PACKAGE DIMENSIONS`, `REVISION HISTORY`), consistent with the full 43.9 MB Manual; the datasheet is only 144 pages cover to cover (its text conversion self-reports `Page 1 of 144` at the start), so it couldn't be that.
3. **The revision number matches.** The latest version in the `REVISION HISTORY` at the end of the table of contents is `1.30`, which lines up exactly with the `0130` in the filename `r01uh1032**ej0130**` (= Rev. 1.30).
4. **`pdfinfo` matches directly.** Running `pdfinfo ../../reference-docs/r01uh1032ej0130-rzv2h.pdf` against the PDF itself self-reports the title `RZ/V2H Group User's Manual: Hardware (Non-Agreement)`, `Pages: 4816` — the page count matches the end of the table of contents (`4816 Back Cover`) exactly (run on the workstation 2026-07-18).

All four pieces of evidence point to the same conclusion: `_toc_full.txt` is the complete table of contents for the Manual, `r01uh1032ej0130`. From here on, use it as an index with confidence.

First, look at its top-level skeleton — the whole Manual is divided into 10 SECTIONs. You can pull this out yourself:

```bash
grep -E "^[0-9]+	SECTION " ../../reference-docs/_toc_full.txt
```

(That's a Tab character in the `grep` pattern: each line in the file is formatted as "page number + Tab + chapter title.") Expected output (verbatim, ✅ re-run and cross-checked consistent on 2026-07-18, transcript: live/ch04b-docs-local.txt; the number at the start of each line is **that chapter's starting page in the Manual PDF**):

```text
76	SECTION 1 OVERVIEW
212	SECTION 2 PROCESSORS
264	SECTION 3 MEMORY
360	SECTION 4 SYSTEM
1161	SECTION 5 TIMER
1515	SECTION 6 HIGH-SPEED INTERFACE
2500	SECTION 7 LOW-SPEED INTERFACE
3772	SECTION 8 AUDIO
3956	SECTION 9 IMAGE
4690	SECTION 10 ELECTRICAL CHARACTERISTICS
```

These 10 SECTIONs are the first-level classification to keep in your head. You can generally guess where to go just from the title: processors (including the NPU/DRP) are in SECTION 2, memory (including DDR/SRAM/TZC) is in SECTION 3, the system core (PFC pins, CPG clocks, interrupts, DMAC) is in SECTION 4, timers are in 5, high-speed interfaces (SD/Ethernet/USB/PCIe) are in 6, low-speed interfaces (xSPI/serial ports/I²C/CAN/ADC/temperature sensing) are in 7, audio is in 8, imaging (camera/scaling/display/codec/GPU) is in 9, and electrical characteristics are in 10.

The next step is to drill down one level and find which chapter and page your target **unit** is on. Say you're wiring up Ethernet and want to check GBETH's registers — just grep the unit name directly:

```bash
grep "Gigabit Ethernet" ../../reference-docs/_toc_full.txt
```

Expected output (verbatim, ✅ re-run and cross-checked consistent on 2026-07-18, transcript: live/ch04b-docs-local.txt):

```text
1632	  6.3 Gigabit Ethernet Interface (GBETH)
```

One line gives you three pieces of information: GBETH is in **Chapter 6.3**, belongs to SECTION 6 (high-speed interfaces), and starts on **page 1632** of the PDF. Now you can open that 43.9 MB PDF and jump straight to page 1632, instead of scrolling through 4816 pages.

> 💡 **Tip**: Every line in `_toc_full.txt` is "page number, Tab, title," and the indentation depth represents the chapter hierarchy — 0 levels of indent is a `SECTION`, 2 spaces is a "chapter" (like `6.3`), and 4 or more spaces is a finer-grained sub-section or register. If you only want to see the "chapter" level and filter out the fiddly register lines, you can pin the indentation: `grep -E "^[0-9]+	  [0-9]" ../../reference-docs/_toc_full.txt` keeps only the chapter titles with 2-space indentation.

Grepping the unit name directly is convenient, but there's a pitfall you need to hear about first, or the first time you look up a popular unit you'll get buried:

> ⚠️ **Note**: Grepping a unit name directly often buries the "chapter title" under a pile of "register names."
> **Situation**: You want to find the I²C bus chapter, so instinctively you type `grep "I2C" ../../reference-docs/_toc_full.txt`.
> **Symptom**: It spits out **40 lines**, full of register- and mode-level entries like `7.7.2.2.1 I2C Bus Control Register 1 (RIICm_ICCR1)`, `7.3.7 Simple I2C Mode`... and the "chapter" you actually want is buried in the middle.
> **Cause**: Every register with "I2C" in its name and every Simple-I2C-mode sub-section in the Manual gets matched by grep; and RSCI (the serial port) also has a Simple-I2C mode, so even lines from 7.3 get mixed in.
> **Prevention**: Learn to recognize what a **chapter title looks like** — it's "2-space indent + chapter number + a unit's English abbreviation in parentheses," for example the line you actually want: `2968	  7.7 I2C Bus Interface (RIIC)`. To nail it in one shot, bake the indentation and parentheses into the pattern, e.g. `grep -E "^[0-9]+	  7\.7 " ../../reference-docs/_toc_full.txt`, which gets you the single clean result `2968	  7.7 I2C Bus Interface (RIIC)` directly. RZ/V2H's I²C controller is called **RIIC** in the Manual — remember this abbreviation, it makes your searches more precise.

## Which Chapter to Check: Common Units → Chapter/Page List

Run the grep trick above once for every unit and organize the results into the reference table below — from here on, whenever you need to look up the register detail for any on-board unit, just jump to the page per the table. The page numbers in the table's "Chapter (page)" column are **taken verbatim from `_toc_full.txt`** (the same number you'd see from grep; ✅ re-verified line by line with `grep` on 2026-07-18, including `1.8 Address Map` (p167) — every chapter name and page number in the tables is consistent, transcript: live/ch04b-docs-local.txt); the "Register base" column comes from the results of this handbook's hardware inventory probing (Source: `07-hardware-unit-usage-guide.md`, i.e. d06), for you to cross-check against `/proc/iomem` or use when writing bare-metal code.

**SECTION 2 PROCESSORS (processors and AI accelerator, starting page 212)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| CPU (CA55/CR8/CM33 three-core configuration) | 2.2 CPU (p213) | — |
| DRP-AI3 NPU (AI-MAC + DRP0) | 2.3 AI Accelerator (DRP-AI) (p251) | AI-MAC `0x16800000`, DRP0 `0x17000000` |
| DRP (reconfigurable processor DRP1, a different unit from DRP0 inside DRP-AI) | 2.4 DRP (p259) | — |

**SECTION 3 MEMORY (memory, starting page 264)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| 6 MB shared SRAM (SRAM0–11) | 3.2 SRAM (p265) | starts at `0x08000000`, each block +`0x80000` (512 KB) |
| LPDDR4/4X memory controller | 3.4 LPDDR4/4X Controller (DDR) (p277) | MEMC `0x1E000000` (DDR0)/`0x1E010000` (DDR1), PHY `0x1A000000` (DDR0)/`0x1C000000` (DDR1) |
| TrustZone address space control (TZC-400) | 3.5 TrustZone Address Space Controller (TZC) (p352) | TZC400_XSPI `0x10470000`, TZC400_SRAMM `0x10460000`, etc. |

**SECTION 4 SYSTEM (system core: pins, clocks, interrupts, DMA, starting page 360)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| Pin multiplexing/pin-mux (PFC) | 4.2 Pin Function Controller (PFC) (p361) | PFC `0x10410000`, 86 ports |
| Clock generation (CPG/PLL/dividers/gating/reset) | 4.4 Clock Pulse Generator (CPG) (p620) | CPG `0x10420000` |
| Power management (PMU/power domains) | 4.5 Power Management Unit (PMU) (p790) | — |
| Interrupt control (GIC-600/ICU/ELC) | 4.6 Interrupt Controller (p800) | GIC-600 base `0x14900000`, ICU `0x10400000` (ELC is integrated here) |
| DMA controller (DMAC) | 4.7 DMA Controller (DMAC) (p994) | DMAC0 `0x11400000`, DMAC1 `0x14830000`, DMAC2 `0x14840000` (DMAC1/2 are 64 KB apart) |
| Debug interface (CoreSight/JTAG/SWD) | 4.9 Debug Interface (p1119) | CoreSight (CST) `0x1F000000` (16 MB) |

> 📌 **Address correction note**: For the GIC-600 and DMAC0 cells in the table above, the earlier secondhand source had a transcription discrepancy here — GIC was once recorded as `0x14800000` (which, per §1.8 Address Map, is actually the SRAM2(REG) region), and DMAC0 was once recorded as `0x11C00000` (which is actually the ADC's address, in the same SECTION 7 table). The source document's transcription was wrong here; it has been corrected against the official manual's §1.8 Address Map (p167), §4.6.2.2 (p961), and Table 4.7-4 (p1001): GIC-600 base `0x14900000`, DMAC0/1/2 = `0x11400000`/`0x14830000`/`0x14840000`.

**SECTION 5 TIMER (timer group, starting page 1161)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| Real-time clock (RTC) | 5.3 Realtime Clock (RTC) (p1166) | `0x11C00800` (read-only mirror address `0x11C00C00`) |
| Watchdog (WDT) | 5.4 Watchdog Timer (WDT) (p1218) | WDT0(CM33) `0x11C00400`, WDT1(CA55) `0x14400000`, WDT2/3(CR8) `0x13000000`/`0x13000400` |
| General-purpose timer (GTM, the same IP as OSTM) | 5.5 General Timer (GTM) (p1236) | GTM0 `0x11800000` … GTM7 `0x12C03000` |
| Compare match timer (CMTW) | 5.6 Compare Match Timer W (CMTW) (p1254) | CMTW0 `0x11C01800` … CMTW7 `0x13001800` |
| General-purpose timer/PWM (GPT) | 5.7 General-Purpose Timer (GPT) (p1283) | GPT0 `0x13010000`, GPT1 `0x13020000` |
| PWM output gating (POEG) | 5.8 Port Output Enable for GPT (POEG) (p1504) | POEG0A `0x13001C00` … POEG1D `0x13003800` |

**SECTION 6 HIGH-SPEED INTERFACE (high-speed interfaces, starting page 1515)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| SD card/eMMC host controller (SDHI) | 6.2 SD/MMC Host Interface (SD) (p1516) | SD0/1/2 `0x15C00000`/`0x15C10000`/`0x15C20000` |
| Gigabit Ethernet (GBETH, including PTP) | 6.3 Gigabit Ethernet Interface (GBETH) (p1632) | GBETH0/1 `0x15C30000`/`0x15C40000`, reference clock 125 MHz |
| USB 3.2 Host | 6.4 USB3.2 Gen2x1 Interface (USB3) (p1636) | USB30/31 Host `0x15850000`/`0x15860000` |
| USB 2.0 Host/Function | 6.5 USB2.0 Interface (p1753) | USB20/21 Host `0x15800000`/`0x15810000` (USB20 Function `0x15820000`) |
| PCIe Gen3 (RC/EP selectable) | 6.6 PCI Express 3.0 Interface (PCIe) (p2025) | outbound window `0x30000000`(PCIE0)/`0x38000000`(PCIE1) |

**SECTION 7 LOW-SPEED INTERFACE (low-speed interfaces, starting page 2500)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| External flash interface (xSPI) | 7.2 Expanded Serial Peripheral Interface (xSPI) (p2501) | `0x11030000`, flash window 256 MB@`0x20000000` |
| Renesas serial communications (RSCI, 10 ch) | 7.3 Serial Communications Interface (RSCI) (p2585) | RSCI0 `0x12800C00` … RSCI9 `0x12803000` (the on-board serial console = RSCI6@`0x12802400`) |
| Serial port with FIFO (SCIF) | 7.4 Serial Communications Interface with FIFO (SCIF) (p2780) | SCIF0 `0x11C01400` |
| Renesas SPI (RSPI, 3 ch) | 7.5 Serial Peripheral Interface (RSPI) (p2826) | RSPI0/1/2 `0x12800000`/`0x12800400`/`0x12800800` |
| CRC operation unit | 7.6 CRC Operation Unit (CRC) (p2954) | — |
| I²C bus (RIIC, 9 ch) | 7.7 I2C Bus Interface (RIIC) (p2968) | RIIC0 `0x14400400` … RIIC7 `0x14402000`, RIIC8 `0x11C01000` (CM33 always-on domain) |
| I3C bus | 7.8 I3C Bus Interface (I3C) (p3061) | I3C0 `0x12400000` |
| CAN-FD (6 ch) | 7.9 CAN-FD Interface (CANFD) (p3363) | CANFD base `0x12440000`, a single base covers 6 channels |
| 12-bit ADC | 7.10 12-Bit A/D Converter (ADC) (p3670) | ADC `0x11C00000` |
| Temperature sensor (TSU) | 7.11 Temperature Sensor Unit (TSU) (p3739) | TSU0 `0x11000000`, TSU1 `0x14002000` (trim values come from OTP [one-time-programmable memory on the chip], 0.0625 °C/code) |

> 💡 **Tip**: If you're writing a driver for CAN-FD register by register, besides Manual Chapter 7.9, Renesas also has a separate FSP-edition CAN-FD manual, `r01us0478`, with more complete register documentation (not in this folder — you'll need to get it from Renesas separately). When hooking up a DroneCAN/UAVCAN controller (a flight controller, for example), the usual approach is to check 7.9 first to get the addresses, then flip to `r01us0478` to work through the registers one by one (Source: d06, `07-hardware-unit-usage-guide.md`).

**SECTION 9 IMAGE (video/display/codec/GPU, starting page 3956)**

| Unit/topic you want to check | Manual chapter (page) | Register base (d06 probe) |
|---|---|---|
| MIPI camera receive (CRU/CSI-2) | 9.2 Camera Data Receiver Unit (CRU) (p3957) | CRU0–3 `0x16000000`/`10000`/`20000`/`30000` |
| Image scaling (ISU) | 9.3 Image Scaling Unit (ISU) (p4247) | `0x16450000` |
| LCD controller (LCDC/DU) | 9.4 LCD Controller (LCDC) (p4372) | DU `0x16460000`, FCPVD `0x16470000` |
| MIPI DSI display output | 9.5 MIPI DSI Interface (DSI) (p4548) | DSI_LINK `0x16430000`, DSI_DPHY `0x16440000` |
| Video hardware codec (VCD, H.264/H.265) | 9.6 H.265/H.264 Multi Codec (VCD) (p4683) | VCD's sub-blocks sit in the `0x16400000` video region (VLC/FCPC/CE = `0x16400000`/`0x16410000`/`0x16420000`) |
| GPU (Mali-G31/GE3D) | 9.7 3D Graphics Engine (GE3D) (p4685) | `0x14850000` (64 KB) |

> ⚠️ **Note**: A special heads-up about the VCD cell — if you see a chapter number like "VCD is in hw_manual 15.x" somewhere else, **go with 9.6 in this table**.
> **Situation**: You see a note somewhere else (old notes, a discussion thread, or some summary) saying "VCD is in hw_manual 15.x" or "1.8/15.x," and you go looking for the VCD codec in the Manual based on that.
> **Symptom**: You flip to "15.x" and can't find the chapter — the whole manual only has 10 SECTIONs; there's no Chapter 15 at all.
> **Cause**: That "1.8" refers to `1.8 Address Map` (where VCD's `0x16400000` address region is listed) — it's an **address-region number, not a chapter number**; and a numbering like "15.x" doesn't match the Manual's actual final table of contents — VCD actually lands in 9.6 in the Manual.
> **Prevention**: Always go back to `_toc_full.txt`, the authoritative table of contents, to look up chapter numbers — what it greps out, `4683	  9.6 H.265/H.264 Multi Codec (VCD)`, is the real location; the address region can additionally be cross-checked in `1.8 Address Map` (p167).

> 📌 **Address correction note**: An earlier secondhand source recorded VCD's address as `0x14800000`, but §1.8 Address Map shows that `0x14800000` is the SRAM2(REG) region; VCD's three sub-blocks (VLC/FCPC/CE) are actually `0x16400000`/`0x16410000`/`0x16420000`, consistent with the DT node `vcp4@16400000` in the g2 chapter. The source document's transcription was wrong here; it has been corrected against §1.8 of the official Manual.

## Only Need Spec Numbers: The Datasheet's Section 1 Is Enough

If what you need to check isn't "register detail" but rather "spec numbers" (e.g., how many channels some interface has, its max MHz, differences between part numbers), you don't need to touch the Manual at all — just flip straight to Section 1's spec summary table in the datasheet (`r01ds0429` — remember, this folder's copy is a text conversion, which actually makes `grep` especially convenient). Its subtables break things down in fine detail (Source: `datasheet.md`, i.e. the d18 reading scope):

- Table 1.3-1 CPU, 1.3-2 Accelerator Engines, 1.3-3 SRAM/external memory, 1.3-4 Boot, 1.3-5 System/DMAC/interrupts/clocks, 1.3-6 communications/storage/networking, 1.3-7 timers, 1.3-8 Audio, 1.3-9 ADC, 1.3-10 internal sensors, 1.3-11 Security, 1.3-12 GPIO, 1.3-13 power, 1.3-14 temperature, 1.3-15 quality, 1.3-16 packaging.
- There's also **Table 1.4-1/1.4-2 List of Units**, an "abbreviation-to-full-name cross-reference table" for units — check it when you need to know what abbreviations like `CRU` / `DRP` / `DRP-AI` / `GPV` / `TZC` stand for. Pay special attention: `DRP` refers to unit DRP1, while `DRP-AI` is made up of DRP0 + AI-MAC — the two share the "DRP" prefix but are **different units**, don't mix them up (Source: `datasheet.md`, 191–194, 785–874).

> ⚠️ **Note**: Don't read the datasheet's block diagram (Figure 1.4-1 Block Diagram) literally out of the text conversion.
> **Situation**: You want to see clearly how the various buses connect from the datasheet's whole-chip block diagram, so you read the corresponding section directly out of this folder's text conversion.
> **Symptom**: The text order is scrambled, the label and the number for the same block get split across non-adjacent lines, and you even get reversed, garbled characters (like `suB UPCM`).
> **Cause**: The original figure is a multi-column block diagram, and the PDF-to-text conversion tool tore the layout apart and reflowed it — this folder's datasheet is exactly this kind of text conversion (see the format note earlier).
> **Prevention**: To see the block diagram, get the original PDF from the Renesas website and look at the figure on **page 16** — don't trust the text order after conversion (Source: d18, datasheet.md conversion note).

## That AWO/Firmware Deployment Document (Advanced, Know Where It Is For Now)

Of the nine documents, `r01an7723` (AWO Example Program Startup Guide, R01AN7723EJ0400 Rev.4.00) is the only application note that's been read in full. What it teaches is **advanced firmware deployment** — how to run the AWO (wake/sleep control) example on CM33, and how to build CA55 and CM33 artifacts out of Yocto. This is **not within this chapter's hands-on scope** (this chapter covers the hardware resource map), so what's kept here is just a few pointers and known gotchas that'll be useful "for whenever you actually get into R8/M33 firmware" (Source: d18《一》, startup-guide.md):

- **Document structure**: 6 chapters in total — 1 Specifications / 2 Proven Environment / 3 RZ/V2H Setup / 4 RZ/V2N Setup / 5 Invocation (Suspend-to-RAM not supported) / 6 Invocation (S2R supported). Note that this document covers both the RZ/V2H and RZ/V2N boards at once, so make sure you know which chapter you're in as you read. (The original never spells out what "AWO" stands for — it uses the abbreviation throughout without ever expanding it, so this handbook won't make up a full name for it either.)
- **Verified tool versions** (copied straight from the original, so you can match your environment): e2 studio **2025-12**; RZ/V2H AI SDK **v6.00**, RZ/V2N AI SDK **v6.30**; RZ Flexible Software Package (FSP) **v4.0.0** (Source: startup-guide.md:89–94).

> ⚠️ **Note**: Debugging RZ/V2H's AWO example with J-Link, the J-Link DLL version is pinned tight.
> **Situation**: You use whatever J-Link software you already have on hand to debug RZ/V2H's CM33 cold-boot AWO example.
> **Symptom**: It may not work correctly.
> **Cause**: The original explicitly states "RZ/V2H AWO example ⋯ requires the use of J-Link DLL version 7.96e" — this cold-boot environment is tied to a specific version.
> **Prevention**: Update your J-Link software to this specific version, **7.96e**, before debugging; RZ/V2N's AWO example has no such restriction and can use J-Link DLL **8.60** (Source: startup-guide.md:71–74).

> ⚠️ **Note**: The bitbake syntax this document uses in `layer.conf` **differs between the RZ/V2H section and the RZ/V2N section** — look carefully at which section you're in before copying it.
> **Situation**: You follow this document to edit Yocto's `layer.conf` to enable CM33 cold boot, and casually apply the RZ/V2H section's syntax to RZ/V2N (or the other way around).
> **Symptom**: bitbake rejects the syntax, or the setting doesn't take effect.
> **Cause**: The RZ/V2H section uses **underscore** syntax, `MACHINE_FEATURES_append = " RZV2H_CM33_BOOT"`; the RZ/V2N section uses **colon** syntax, `MACHINE_FEATURES:append = " RZV2N_CM33_BOOT"` — this is a wording difference between the original's two parallel subsections (possibly reflecting a syntax transition period across different bitbake versions), not a typo.
> **Prevention**: When copying, be clear about which board's section you're editing, and use that section's own syntax; also, the original explicitly warns that when uncommenting the `*_CM33_BOOT` line, do **not** also uncomment the three lines below it, `SRAM_REGION_ACCESS`/`CM33_FIRMWARE_LOAD`/`CA55_CPU_CLOCKUP` (the original only says "Be sure NOT to uncomment" without explaining the technical reason, so just follow it — don't try to guess why) (Source: startup-guide.md:113–122, 244–255).

There's one more detail that echoes back to the discrepancy in file 01, "the on-board A55 actually runs at 1.7 GHz while the datasheet rates it at 1.8 GHz": this AWO document mentions that to set the CA55 operating frequency to **1.8 GHz**, you need to separately enable the "Clock up for CA55" option in `configurator.xml` (Source: startup-guide.md:393–395). In other words, 1.8 GHz is not the default — you have to turn it on deliberately.

## Official Online Resources

Besides this stack of documents in the repo, there are also official online docs and code repositories — check these when you need the latest version or the source code (Source: `04-hardware-quickref.md`, i.e. d03, 224–226):

- Official RDK documentation site: `https://renesas-rdk.github.io/rzv2h_rdk_documentation/latest/`
- RZ/V2H Linux BSP (source code): `https://github.com/renesas-rz/rzv_linux-cip`
- DRP-AI TVM (the toolchain for compiling models onto DRP-AI3): `https://github.com/renesas-rz/rzv_drp-ai_tvm`

## Hands-On Verification: Confirm You Can Use This Stack of Documents as a Map

Once you've done these few things, it means you can now use this stack of documents as a map:

1. ✅ **List the nine documents** (run in this folder on 2026-07-18, output shown earlier under "What's Actually in the Folder"). Run `ls -1 ../../reference-docs/*.pdf`, count whether it's **9 `.pdf` files**, and for each filename's leading prefix (`R01DS`/`R01UH`/`R01US`/`R01AN`/`R01QS`/`R01WP`/`R20AN`/`REN_…R16UH`), be able to say what kind of document it is.
2. 📼 **Work backward from a question to the document.** Quiz yourself with four questions and see if you can say which document to check:
   - "How many CAN channels does this chip have?" → the datasheet (`r01ds0429`), Section 1 spec summary table.
   - "What's the register offset for the GPT timer?" → the Manual (`r01uh1032`), find the chapter using `_toc_full.txt`.
   - "How is DRP-AI3's 8 TOPS figure calculated?" → the white paper (`r01wp0022`).
   - "Where does a given pin on the 40-pin header connect to?" → the WS125 RDK **carrier-board** manual (`REN_WS125…`, remember to extract it first) — this is a board-level question, the chip manuals don't have it.
3. ✅ **Use `_toc_full.txt` to jump to the correct chapter and page** (re-run 2026-07-18, transcript: live/ch04b-docs-local.txt). Pick any on-board unit at random (say, the CAN you're currently wiring up), run `grep "CAN-FD" ../../reference-docs/_toc_full.txt` — in practice this spits out **9 lines** (1 chapter-title line + 8 register sub-section lines containing "CAN-FD"), and recognize **the first line, and the only line with 2-space indentation**: `3363	  7.9 CAN-FD Interface (CANFD)` — that is, SECTION 7, Chapter 7.9, PDF page 3363. That page number is the target you jump to inside the 43.9 MB Manual.
4. ✅ **Avoid the grep-flooding pitfall** (re-run 2026-07-18: `grep -c "I2C"` actually returns exactly 40 lines in practice, transcript: live/ch04b-docs-local.txt). Grep a "popular" unit (like `I2C`) directly and watch it spit out dozens of register lines; then use the "recognize what a chapter title looks like" method (2-space indent + parenthesized unit name, e.g. `2968	  7.7 I2C Bus Interface (RIIC)`) to pick out the chapter you actually want. Once you can do this, you won't get thrown off by register-name noise anymore.
5. 📼 **Cross-check the numbers (optional).** Pick a unit whose base address you saw earlier in this chapter (e.g., GBETH `0x15C30000`), go to the manual's corresponding chapter (6.3, p1632), and confirm that the start of the register map matches — once "the base address from the inventory notes" and "the Manual's chapter" line up, you've genuinely learned to navigate this stack of documents.
