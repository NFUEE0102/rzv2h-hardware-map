# Changelog

All notable changes to this English translation are recorded here. Each entry
lists what changed relative to the internal Chinese handbook chapter
"全板硬體資源地圖" (Full-Board Hardware Resource Map) that this repo mirrors.

Each item is given in English, with the corresponding Chinese line underneath.

---

## 2026-09-15

Resynced against the Chinese source as of 2026-09-12. The original translation
was made from a state of the chapter that predated the handbook's v2 rewrite,
so this sync closes that gap: it carries both the v2 corrections and everything
added since.

### Corrected

- **Updated: SoC part number is R9A09G057H44GBG (the RZ/V2HP variant), not
  R9A09G057H42.** Corrected throughout `README.md`,
  `00-overview-and-ip-enablement-map.md`, `01-compute-units.md`,
  `g2-video-capture-codec-display.md` and `g8-debug-and-security.md`.
  中文：本板 SoC 料號更正為 R9A09G057H44GBG（H44＝RZ/V2HP 版），原譯文誤植為
  R9A09G057H42。

- **Updated: the Mali-C55 ISP is present in this silicon but not enabled in the
  device tree** — it is no longer described as Not Populated. Verified against
  the hardware manual `r01uh1032` §1.1.2 Product Lineup, Table 1.1-1, p78,
  which marks this part number's ISP column `Available (Mali-C55)`, with the
  Remark "The ISP is only present in the RZ/V2HP products." (p823). Imaging on
  this board still goes through CRU pure-DMA because the device tree has no ISP
  node.
  中文：Mali-C55 ISP 為「矽晶含、但 device tree 未啟用」，不再標為「未搭載」；
  依真硬體手冊 `r01uh1032` Table 1.1-1〔p78〕ISP 欄 `Available (Mali-C55)` 核實，
  影像仍走 CRU 純 DMA。

- **Updated: hardware Security IP is Not Populated on this part number, now
  sourced from the hardware manual instead of the datasheet.** The
  determination now cites `r01uh1032` Table 1.1-1, p78 (Security column = N/A,
  verbatim-checkable in a clean PDF) rather than the degraded datasheet
  conversion `r01ds0429` Table 1.2-1.
  中文：硬體 Security IP 未搭載之判定改以真硬體手冊 `r01uh1032` Table 1.1-1〔p78〕
  Security 欄 N/A 為一手來源，不再倚賴劣化轉檔的 datasheet 陣容表。

- **Updated: USB port composition.** The board has 2× USB3.2 Gen2 Type-A (CN2,
  USB2-compatible) plus 1× micro-B that is a UART serial/function port, not a
  data port; there is no separate USB 2.0 host connector. The "4 ports" reading
  came from `lsusb` showing two xHCI controllers each with a USB2/USB3
  companion.
  中文：USB 實體埠更正為 2× USB3.2 Gen2 Type-A（CN2）＋1× micro-B（UART／function，
  非資料埠），無獨立 USB 2.0 host 連接器；`lsusb` 的四個 root hub 是 2 個 xHCI
  控制器各帶 USB2／USB3 companion。

- **Updated: DRP-AI3 speedup is 9.7×/28.1×, not 9.5×/27.7×**, both computed
  against the 15.0 ms pure-inference p50 (145.6 ÷ 15.0 and 422 ÷ 15.0).
  中文：DRP-AI3 加速比更正為 9.7×／28.1×（皆以純推論 p50 15.0 ms 為除數）。

- **Updated: pure inference has no fixed startup floor.** The previous claim of
  a fixed 8–10 ms NPU startup/DMA floor is removed; measured on-board,
  mobilenetv2 runs at 1.34 ms and resnet50 at 4.19 ms. What makes end-to-end
  longer than inference is the pre/post-processing around it (6.86 + 1.62 ms
  for YOLOX-nano@416).
  中文：刪去「每張圖有 8–10 ms 固定 NPU 啟動下限」的說法；板上實測 mobilenetv2
  1.34 ms、resnet50 4.19 ms，端到端較長是前處理＋後處理（6.86＋1.62 ms）所致。

- **Updated: A55 boot frequency settings.** `BOOTPLLCA[1:0]` on DSW1 has four
  defined settings — 1.1 / 1.5 / 1.6 / 1.7 GHz (1.7 GHz is the factory
  default); there is no 1.8 GHz DIP setting. The datasheet's "max 1.8 GHz" is
  the silicon's rated ceiling, a number at a different level.
  中文：A55 啟動頻率四檔（1.1／1.5／1.6／1.7 GHz，出廠預設 1.7 GHz）以板卡手冊
  DSW1 表為準，不存在 1.8 GHz 的 DIP 檔位；datasheet 的 1.8 GHz 是矽晶額定上限。

- **Updated: memory is 16 GB LPDDR4 (1600 MHz = 3200 MT/s, 8 GB × 2)**; the
  spec summary says LPDDR4 while the board block diagram labels it LPDDR4X-3200
  and the controller supports both.
  中文：記憶體規格補明 16 GB LPDDR4（1600 MHz＝3200 MT/s，8 GB×2）；規格總表寫
  LPDDR4、板卡方塊圖標 LPDDR4X-3200，控制器兩者皆支援。

- **Updated: the 100–360 Hz hard-loop ceiling is a range, not a single
  conservative figure.** 108 Hz corresponds to a period 10× the jitter (larger
  margin) and 360 Hz to a period 3× the jitter (smaller margin).
  中文：100–360 Hz 硬迴圈上限說明為區間——108 Hz 為週期 10 倍抖動（餘裕大）、
  360 Hz 為 3 倍（餘裕小），非「360 Hz 較保守」。

- **Updated: reserved memory regions are not uniformly untouchable.** Most
  accelerator/display/camera carveouts carry `reusable` and can be borrowed via
  CMA until the hardware needs them (hence `CmaTotal` ≈ 1.7 GB); only the
  cross-core `vdev0*` regions carry `no-map`.
  中文：記憶體保留區區分兩類——多數帶 `reusable`、硬體用前可經 CMA 暫借
  （`CmaTotal` 約 1.7 GB）；只有跨核通訊的 `vdev0*` 帶 `no-map` 才真正碰不得。

- **Updated: i2c-8's clock generator is a VersaClock 3S, part number
  5L35023B-616NLGI8 (U30)**, per the board manual BOM; the @0x69 address is as
  recorded in the investigation documents and unverified, since i2c-8 is never
  scanned.
  中文：i2c-8 上的時脈產生器更正為 VersaClock 3S（料號 5L35023B-616NLGI8，U30）；
  @0x69 位址為開發文件所載、因不掃 i2c-8 而未實測核對。

- **Updated: the R8's lack of dual-link lock-step now cites a first-hand
  source** — `r01uh1032` §1.1.3 Functions, Table 1.1-2, Cortex-R8 (CR8) row,
  p78, verbatim.
  中文：R8 無 dual-link lock-step 改引一手來源——`r01uh1032` Table 1.1-2 CR8 列
  〔p78〕逐字記載。

- **Updated: TSU base addresses verified**, and the OTP trim value flagged as a
  secondhand transcription pending verification.
  中文：TSU 兩基底位址經真硬體手冊 §1.8 位址圖核實；OTP trim 值標註為二手轉錄、待覆核。

- **Updated: the `PWM0`/`PWM1` silkscreen on the 40-pin header really is wired
  to GPT**, resolving a previously open question — it is not just borrowed
  Raspberry Pi naming. Pin 32 = `PA4` → `GTIOC6A`, pin 33 = `PA7` → `GTIOC7B`,
  pin 35 = `P96` → `GTIOC9A`, pin 31 = `P53` → `GTIOC10B`. Of the header's 28
  signal pins, 23 can be muxed to a `GTIOC`: **20 simultaneous PWM channels** if
  the header is used for PWM alone, and **13** if the full reference peripheral
  set is kept — comfortably covering the 10 channels this was checked against.
  中文：40-pin header 的 `PWM0`／`PWM1` 絲印確認真的接到 GPT（pin 32＝`PA4`→
  `GTIOC6A`、pin 33＝`PA7`→`GTIOC7B`、pin 35＝`P96`→`GTIOC9A`、pin 31＝`P53`→
  `GTIOC10B`）；header 全用於 PWM 最多 20 路，沿用參考實作週邊後仍有 13 路。

- **Updated: CN6 attribution corrected** in the camera/interface descriptions,
  per the early development records.
  中文：依早期開發實錄修正 CN6 的歸因。

- **Updated: several counts and measurements now carry explicit
  evidence-strength caveats** — the whole-tree device-tree node count (136 in
  the investigation documents, not reproduced on the board), the "52 functional
  blocks" figure (from a degraded datasheet conversion, provisional), the PLL
  frequency summary table (inventory values, not each checked against a
  first-hand manual), and the idle chip temperature (~34–36 °C).
  中文：多處數字補上證據強度註記——整樹節點數（開發文件載 136、板上未複現）、
  52 個功能區塊（劣化轉檔、暫定）、PLL 頻率摘要表（盤點值、未逐一一手核實）、
  待機晶片溫度（~34–36 °C，跨 session）。

### Added

- **Device tree overlay mechanism.** A new section in
  `00-overview-and-ip-enablement-map.md` documents the six `/boot/uEnv.txt`
  overlay switches, the rule that only overlay lines may be edited, why adding
  `enable_overlay_X=1` alone does nothing without a matching `apply_ov_<name>`
  listed in `mmc_ovfdt`, and the incompatibility between the official 2-lane
  `tevs-cam0` overlay and this handbook's 4-lane route.
  中文：新增 device tree overlay 機制一節——`/boot/uEnv.txt` 六組開關、只准動
  overlay 行的規定、新增 overlay 必須同時補 `apply_ov_<name>` 並列入 `mmc_ovfdt`，
  以及官方 2-lane `tevs-cam0` overlay 與本手冊 4-lane 路線互斥。

- **OpenCVA is a shipped library, not a blank slate.** OpenCVA is now marked
  "official library not installed" rather than "no userspace driver," with the
  install procedure, the 18 auto-accelerated OpenCV functions, the official
  `wrapAffine`/`wrapPerspective` misspelling, and the `libshim` interaction to
  check first.
  中文：OpenCVA 改標為「官方函式庫未安裝」而非「無使用者空間驅動程式」，並補上
  安裝步驟、18 個自動加速函式、官方頁面 `wrapAffine`／`wrapPerspective` 拼寫錯誤，
  以及與 `libshim` 的相容性提醒。

- **Multi-OS boundaries and pinned versions.** `01-compute-units.md` now
  records Multi-OS Package v3.2, RZ/V FSP v3.1, J-Link firmware 7.96e, the
  DSW1 SW6 requirement for JTAG, what the default IPL does and does not
  support, and the Module Standby clock-gating trap (`DEF_MOD` →
  `DEF_MOD_CRITICAL`).
  中文：`01-compute-units.md` 補上 Multi-OS 的鎖版條件（Package v3.2、RZ/V FSP
  v3.1、J-Link 韌體 7.96e、JTAG 需 DSW1 SW6＝ON）、預設 IPL 支援範圍，以及
  Module Standby 時脈被關的坑（`DEF_MOD`→`DEF_MOD_CRITICAL`）。

- **Known issue: Ethernet.** Added per the official RDK documentation v1.1.1.
  中文：依官方 RDK 文件 v1.1.1 補上乙太網路的已知問題。

- **U-Boot interactive notes**, from hands-on testing at the U-Boot prompt.
  中文：補上 U-Boot 互動實測的操作說明。

- **PMU s2idle warning.** On this board, s2idle suspend has no working wakeup
  source — the RTC alarm can be set but does not wake the system — so a
  headless board must not be suspended. `g5` now carries this as a hard rule.
  中文：新增 PMU s2idle 警告——本板 s2idle 休眠後無可用喚醒源（RTC 鬧鐘設得起
  但醒不來），headless 板一律不要 suspend。

- **vspmfilter cost** documented in the video pipeline description.
  中文：補上 vspmfilter 的代價說明。

- **HDMI audio: indirect official evidence.** The official documentation states
  that enabling the audio codec overlay disables micro-HDMI audio, which
  implies HDMI audio output exists by design and is mutually exclusive with
  that overlay. The chapter's conclusion is adjusted accordingly, while still
  not claiming the board has been heard to output sound.
  中文：HDMI 音訊補上官方間接佐證——啟用 audio codec overlay 會停用 micro-HDMI
  音訊輸出，反證官方設計上存在 HDMI 音訊且與該 overlay 互斥；結論調整為設計上存在、
  本板實測待確認。

- **Video Codec Library restrictions** recorded as a lead on the long-standing
  "is the VCD hardware encoder usable right now" discrepancy: the library is
  only available on the default image, Ubuntu Desktop may be incompatible, and
  the on-board GStreamer plugins come from apt rather than a Renesas build. No
  conclusion is drawn from this.
  中文：補上 Video Codec Library 的三條使用限制，作為 VCD 硬體編碼器爭議的線索
  （只在預設映像檔可用、Ubuntu Desktop 可能不相容、GStreamer 外掛來自 apt）；
  仍不對該差異下結論。

- **Serial console pointer.** `g8-debug-and-security.md` now states up front
  that the first-line debugging tool is the CN8 UART serial console
  (FT234XD → SCIF0, 115200 8N1), not the CoreSight/JTAG hardware in that group.
  中文：`g8` 開頭補上一句——日常除錯的第一工具是 CN8 的 UART 序列主控台
  （FT234XD → SCIF0，115200 8N1），不是本群組的 CoreSight／JTAG。

- **Traditional Chinese originals.** The ten current Chinese chapter files are
  now included under [`zh-TW/`](zh-TW/README.md) with ASCII filenames matching
  their English counterparts, so any passage can be checked against its source.
  中文：新增 [`zh-TW/`](zh-TW/README.md)，收錄十個中文原始章節檔（檔名改用與英文版
  對應的 ASCII 名稱），可逐段對照原文。
