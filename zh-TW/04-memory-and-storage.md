# 04 · 記憶體與儲存

這一組要把 RZ/V2H 上「資料停在哪裡」的五個硬體單元一次講清楚：晶片裡跑得最快、卻整塊被保留起來的**內部 SRAM**；SoC 通電後執行的**開機第一棒 Boot ROM**；扛起系統主記憶體的**LPDDR4/4X 控制器**；承載開機小映像的**xSPI／NOR flash**；以及放作業系統與大量資料的**SDHI／SD 卡**。它們合起來構成一條從「晶片內、揮發性、給程式碼落地」到「晶片外、非揮發性、給檔案系統」的完整記憶體與儲存路徑。

先用一張圖把這五個單元擺進同一個座標，你後面讀每一節時就不會迷路——最上面兩塊在晶片內，其餘三塊在晶片外；左半邊是「拿來執行／當工作記憶體」的路徑，右半邊是「拿來長期存資料」的路徑：

```text
                       RZ/V2H 記憶體與儲存路徑（本板配置）
  ┌──────────────────────── 晶片內（on-chip）────────────────────────┐
  │  Boot ROM 128 KB（唯讀）        內部 SRAM 6 MB（12×512 KB，ECC）    │
  │  開機第一棒，交棒後              韌體／CM33／CR8 落地、跨核 IPC      │
  │  對 Linux 不可見                 —— 保留，未入 Linux 一般記憶體池    │
  └───────────────────────────────────────────────────────────────────┘
                                    │
  ┌──────────────────────── 晶片外（external）───────────────────────┐
  │  LPDDR4/4X 控制器 ×2  →  16 GB LPDDR4（揮發性主記憶體）             │
  │      Linux 一般 RAM（約 15.5 GB 可用）＋ 硬體加速器 carveout        │
  │                                                                    │
  │  xSPI  →  NOR flash 64 MB（非揮發、位元組定址、可 XiP 直接執行）    │
  │      開機小映像分割區：bl2 ／ fip ／ env ／ test-area              │
  │                                                                    │
  │  SDHI0  →  SD 卡 256 GB（非揮發、區塊裝置、可抽換）                 │
  │      rootfs 與大量資料：/dev/mmcblk0                               │
  └───────────────────────────────────────────────────────────────────┘
```

> 圖裡標的 **16 GB／64 MB／256 GB 是這片板子實際焊接或插上的容量**，不是各控制器規格的門檻——控制器本身能撐多大，逐節的〈關鍵能力與限制〉會分開講。看到容量時請先分清楚「這片板子剛好裝了多少」與「這顆控制器最多能接多少」是兩回事。

## 本組單元一覽

| 單元 | 一句話 | 板上狀態 |
|---|---|---|
| [內部 SRAM 6 MB](#內部-sram-6-mb12512-kbecc) | 12 塊各 512 KB、帶 ECC 的晶片內建 RAM，給韌體與異質核心當落地區與跨核共享緩衝 | 啟用但**保留**（開機韌體／CM33-CR8 remoteproc／vring 佔用，未入 Linux 一般記憶體池） |
| [Boot ROM](#boot-rom) | 128 KB 唯讀 ROM，存開機韌體，是 SoC 通電後的第一棒 | 啟用（開機必經）；**開發紀錄未單列（誠實缺口）** |
| [LPDDR4/4X 記憶體控制器 ×2](#lpddr44x-記憶體控制器-2) | 雙 channel 外部 DRAM 控制器，支援 LPDDR4/4X-3200、in-line ECC，是系統主記憶體的唯一路徑 | 啟用（16 GB LPDDR4／4X-3200，1600 MHz＝3200 MT/s，BL2 初始化，約 15.5 GB 可供 Linux） |
| [xSPI（NOR flash 介面）](#xspinor-flash-介面) | 高吞吐、少腳位的序列 flash 介面，承載開機用 NOR flash，可記憶體映射直接讀 | 啟用（`xspi@11030000`；MTD `/dev/mtd0..3`＝bl2／fip／env／test-area） |
| [SDHI／eMMC／SDIO Host ×3](#sdhiemmcsdio-host-3) | 3 組 SD／MMC 主機介面，ch0 相容 eMMC，承載開機媒體與大容量區塊儲存 | 部分啟用（SDHI0＝rootfs 於 `/dev/mmcblk0`；兩個 eMMC 控制器 DT 停用） |

## 章內目錄

- [內部 SRAM 6 MB（12×512 KB，ECC）](#內部-sram-6-mb12512-kbecc)
- [Boot ROM](#boot-rom)
- [LPDDR4/4X 記憶體控制器 ×2](#lpddr44x-記憶體控制器-2)
- [xSPI（NOR flash 介面）](#xspinor-flash-介面)
- [SDHI／eMMC／SDIO Host ×3](#sdhiemmcsdio-host-3)

---

## 內部 SRAM 6 MB（12×512 KB，ECC）

### 這是什麼

這顆 LSI（large-scale integration，大型積體電路——就是這片 SoC 晶片本身）內建了 **12 塊各 512 KB 的晶片內建 RAM**（編號 SRAM0 到 SRAM11），手冊把它的定位講得很直接：作為 CM33、CA55、CR8 三種核心的工作記憶體區（手冊 3.2.1 逐字："This LSI has twelve 512-Kbyte areas of on-chip RAM for use as the CM33, CA55, and CR8 working areas."）。這裡的三種核心，就是 4.2 講過的系統管理核 Cortex-M33、應用處理核 Cortex-A55、即時核 Cortex-R8——SRAM 是它們共用的一塊晶片內快記憶體。

每一塊 SRAM 都內建 **ECC**（error-correcting code，錯誤更正碼——在資料之外多存幾個校驗位元，讀回時可偵測甚至更正位元翻轉）。它能偵測並更正 1-bit 錯誤、偵測 2-bit 錯誤；一旦發生 1-bit 或 2-bit 錯誤，出錯的位址會被記進 ECC error address 暫存器，供軟體事後查是哪裡出的錯（手冊 3.2.1）。

各塊 SRAM 的資料匯流排寬度不一樣，切出來的 ECC channel（通道，ECC 電路各自獨立運算的一段資料寬度）數目也不同：SRAM0／1／3 是 64-bit（切成 2 個 32-bit channel）；SRAM2 是 128-bit（切 4 個 channel）；SRAM4 到 SRAM11 是 256-bit（切 8 個 channel）（手冊 3.2.1.1 到 3.2.1.4）。寬度越大的區塊，一次搬得動的資料越多，對頻寬需求高的核心（例如 CR8 即時運算）越有利。

手冊的 Table 3.2-1（p265）逐一列出每塊 SRAMm 的 base address（基底位址）。以 `0x08000000` 為起點，每塊往上加 `0x80000`（＝512 KB）：SRAM0 ＝ `0_0800_0000h`、SRAM2 ＝ `0_0810_0000h`、一路到 SRAM11 ＝ `0_0858_0000h`（手冊 Table 3.2-1；位址遞增規律亦見本手冊 4.4〈官方文件查閱指路〉的位址對照表）。同一張表還給了 CM33 位址空間下的 Code Area／Data Area 對映位址（例如 SRAM0 在 CM33 non-secure 側的 Code Area ＝ `1800_0000h`）——同一塊實體 SRAM，從不同核心、不同安全狀態看過去，位址不一樣。

ECC 的暫存器層細節（SRAMm_CTL_n、SRAMm_EADp_n 等控制與位址暫存器）在手冊 3.2.2 到 3.2.3；ECC 的初始化與使用注意事項在 3.2.4 起。這些是暫存器級明細，本節不展開，只讓你知道它存在、以及從哪一節去查。

### Linux 下怎麼看到它

先講結論：**在這片板子上，SRAM 沒有一般用途的 `/dev` 節點，你在 Linux 裡「直接配一塊 SRAM 來用」是行不通的**——因為它整塊被開機階段的韌體與異質核心佔走了。

機制上，Linux 對這種晶片內 SRAM 有兩種標準處理法：一是透過 `mmio-sram` 這個 sram 驅動程式（原始碼在核心的 `drivers/misc/sram.c`，device tree 的 compatible 字串是 `mmio-sram`），把它曝露成一個「記憶體池」給其他驅動程式去借用；二是把它劃成 reserved-memory carveout（保留區——開機時就從記憶體位址空間切出一塊、標記為某硬體專用，Linux 的一般配置器碰不到），交給 remoteproc（remote processor framework，Linux 用來替協處理器載入韌體、管理開關機的框架）消費。兩種都不會生出一個給使用者程式開啟的裝置檔案（出處 `07-hardware-unit-usage-guide.md:222-240`）。

本板實測（doc07 §14）：這塊 SRAM 被開機韌體（bl2／fip，也就是第二階段開機載入器與韌體映像）、CM33／CR8 的 remoteproc、以及 vring／IPC mailbox（跨核通訊用的共享訊息環與信箱）佔用，**沒有**併入 Linux 的一般記憶體池。因此本手冊在資源地圖裡把它的狀態標成「啟用但**保留**」——硬體是活的、也在用，只是不開放給你的一般 Linux 程式挪用（出處 `07-hardware-unit-usage-guide.md:222-240`；狀態詞彙定義見 00 檔〈先學會讀「狀態」〉，「保留」一格舉的正是這塊 6 MB SRAM）。

你可以自己在板上把這個「保留」狀態看出來：

```bash
# 看實體位址對照表裡的 SRAM／保留區／跨核區塊落在哪
sudo cat /proc/iomem | grep -iE 'sram|reserved|cm33|vring'
# 列出所有記憶體保留區節點（存活裝置樹）
ls /proc/device-tree/reserved-memory/
# 看 remoteproc0（本板為 CM33）目前的名稱與狀態
cat /sys/class/remoteproc/remoteproc0/{name,state}
```

最後一條的預期輸出，會是 `cm33` 與 `offline`——CM33 的 remoteproc 入口在、但預設離線（韌體未內建），這與 4.1、4.2 對 CM33 的描述一致。

### 關鍵能力與限制

- **總容量 6 MB**：12 × 512 KB（手冊 3.2.1 逐字 "twelve 512-Kbyte areas"）。這是全部 12 塊加總，不是單塊可用量；而且如上一段所述，這 6 MB 在本板已被保留、不進 Linux 一般記憶體池。
- **ECC 能力（手冊 3.2.1.5 逐字列舉）**：
  - 可偵測並更正 1-bit 錯誤、偵測 2-bit 錯誤；但**3-bit 以上的錯誤無法被正確處理，可能導致誤判與誤更正**（手冊逐字："An error involving 3 or more bits cannot be correctly handled and may result in erroneous detection and correction."）——這是 ECC 的能力邊界，不要以為 ECC 萬能。
  - 錯誤偵測、錯誤更正、1-bit 更正這三件事，可以分別致能或停用。
  - 每個 channel 有 **8 個** ECC error address 暫存器；當一個 channel 的這 8 個暫存器全存滿時，會產生 overflow（溢位）中斷。
  - 可分別針對 1-bit、2-bit 錯誤設定「要不要發中斷請求」。
- **匯流排寬度分級**：SRAM0/1/3 ＝ 64-bit、SRAM2 ＝ 128-bit、SRAM4–11 ＝ 256-bit（手冊 3.2.1.1–3.2.1.4）。
- **板上狀態**：啟用但**保留**（doc07 §14；doc06 §3）——不是 Linux 可自由配置的一般 RAM。

### 什麼情況下你會用到它

先講機制，才談你要不要用它。SRAM 是**晶片內**的記憶體，存取不需要經過外部 DDR 控制器的仲裁，因此延遲低、而且在 DDR 還沒初始化完成的開機極早期就已經可用。正因為這兩個特性，開機韌體與異質核心（CM33／CR8）常把 SRAM 當作三種用途的落地區：一是**開機當下**程式碼與資料的存放處（此時 DDR 可能還沒起來）；二是**跨核 IPC**（inter-processor communication，處理器間通訊）的共享緩衝（vring、mailbox）；三是對延遲敏感、不想承擔 DDR 仲裁抖動的即時資料。

由此可以給你一條判斷準則：

- **會用到 SRAM 這個角色**——如果你的應用需要載入 CM33／CR8 韌體，或需要在 Linux 與即時核心之間做低延遲的共享記憶體通訊，那麼底層就會用到 SRAM（雖然多半是由 remoteproc 框架與韌體替你安排，不是你手動配置）。以馬達控制為例，一個跑在 R8 上的硬即時控制迴圈若要與 A55 上的 Linux 應用交換狀態，那條跨核訊息環很自然會落在 SRAM 或保留在 DRAM 的 vring 上。
- **不會直接用到 SRAM**——一般 Linux 應用程式（例如影像處理、AI 推論的資料緩衝）不會、也配不到這塊 SRAM：它未加入核心的 page allocator，而且 6 MB 的容量遠小於這類工作負載常見的資料集大小（動輒數百 MB 的模型權重與影格緩衝）。這類需求該走 DDR，見下面 LPDDR4/4X 一節。

本板把 SRAM0／1 用作 CM33 的冷開機落地區、其餘區塊供 remoteproc 的 vring 使用，只是「SRAM 承接異質核心開機與 IPC」這個通用定位的**其中一個實例**，不是 SRAM 唯一的用法——換一套韌體、換一種核心分工，落地的區塊就不同。

> **尾註**：《硬體使用手冊》r01uh1032ej0130 §3.2 SRAM（p265–268，功能概說；暫存器自 3.2.2 p269）。板上證據：`07-hardware-unit-usage-guide.md:222-240`（doc07 §14）、`06-hardware-resource-map.md`（doc06 §3）。

---

## Boot ROM

### 這是什麼

這是一顆 **Boot ROM**，它的職責只有一件事：儲存開機韌體（手冊 3.3.1 逐字："This ROM is a boot ROM, which stores the boot F/W."）。ROM（read-only memory，唯讀記憶體）的內容在出廠時就固定寫死，執行期間不可改寫。

它的容量是 **128 KB**（手冊 3.3.1 逐字："The ROM has a capacity of 128 KB."）——這也是手冊對這顆 ROM 給出的唯一一項規格數字。

在晶片內部，這顆 ROM 透過 64-bit 的 AXI3 介面掛在 system bus（系統匯流排）上，是唯讀的（手冊 Figure 3.3-1：`System Bus → 64-bit AXI3 Master → 64-bit AXI3 Slave → 128-KB ROM`）。AXI3 是 Arm 定義的一種晶片內匯流排協定，你不需要記細節，只要知道它是 SoC 內部連接各區塊的高速通道即可。

### Linux 下怎麼看到它

**你在 Linux 裡看不到這顆 Boot ROM，這是正常的，不是壞掉。** 它只在 SoC 通電後的最初開機階段執行——把控制權交給下一階段的 bootloader（開機載入器）之後，對 Linux 完全不可見：沒有裝置節點、也沒有任何使用者可查的介面。開發紀錄 doc06／doc07 也都沒有這顆 Boot ROM 的段落。

這裡要對你誠實標一個邊界：**Boot ROM 是本手冊資源盤點裡的一個「誠實缺口」**。它在 datasheet 的功能區塊圖上有、開機也一定會經過它，所以判斷為「啟用」；但開發紀錄（doc06／doc07）從頭到尾沒有單獨盤點過它，本手冊不會因此杜撰任何板上細節。上面〈這是什麼〉講的一切，全部來自官方硬體手冊 3.3 的字面內容轉譯；凡是手冊沒寫的（例如它內部韌體的實際流程、版本），本節就不寫（缺口標記依 4.1／unit-map 的盤點口徑）。

### 關鍵能力與限制

- **容量 128 KB**（手冊 3.3.1 逐字，這是手冊給出的唯一規格）。
- **唯讀、64-bit AXI3 介面掛在 system bus 上**（手冊 Figure 3.3-1）。
- **板上狀態**：啟用（開機必經），但**開發紀錄完全未單獨盤點**——這是誠實缺口，本節只按手冊字面轉譯，不編造板上實測數據。

### 什麼情況下你會用到它

先講清楚一個機制事實，你就會明白為什麼「你幾乎永遠不會主動操作它」：Boot ROM 是 SoC 出廠固定的「開機第一棒」。通電後，SoC 一定先執行內部 Boot ROM，再依板上的 dip switch／strap（開機模式選擇的實體指撥開關或接腳綁定）選定的開機模式，去載入下一階段的 bootloader。理解它存在的意義，重點不在於「怎麼用它」，而在於**理解整條開機鏈的起點**。

至於「交棒給哪個開機模式」有哪些選項，這一層由 datasheet（另一份官方文件 r01ds0429，非本手冊 r01uh1032 內容）的 Table 1.3-4「Boot」逐字列出：

- **CA55 開機**：mode 0 ＝ 從 eSD 開機、mode 1 ＝ 從 eMMC 開機、mode 2 ＝ 從掛在 xSPI 匯流排空間上的 serial flash 開機、mode 3 ＝ 從 SCIF 下載開機。
- **CM33 開機**：mode 2 ＝ xSPI serial flash、mode 3 ＝ SCIF 下載。

由此給你判斷準則：**只有當你在動 RZ/V2H 的開機流程時，才會碰到 Boot ROM 這一層的問題**。一種情況是「更換開機媒介」：例如想從 SD 卡改成 eMMC 開機，就是把開機模式從 mode 0 換到 mode 1。另一種是「除錯開機失敗」：板子通電後停在很早的階段、連 bootloader 都沒進去，就要回頭想 Boot ROM 交棒給哪個模式、那個媒介上有沒有正確的映像。日常應用開發完全不需要碰它。這片板子實際用的是哪一種開機模式，對照下面 xSPI 與 SDHI 兩節的板上狀態就能拼出來：開機小映像放在 xSPI 的 NOR flash（對應 mode 2），rootfs 放在 SDHI0 的 SD 卡（對應 mode 0 的 eSD 家族）。

> **尾註**：《硬體使用手冊》r01uh1032ej0130 §3.3 ROM（p276，功能概說）。開機模式清單出處：datasheet r01ds0429 Table 1.3-4（Boot）。板上狀態為誠實缺口（doc06／doc07 未單列），僅依手冊字面轉譯。

---

## LPDDR4/4X 記憶體控制器 ×2

### 這是什麼

這是給 LPDDR4 與 LPDDR4X 這兩類 SDRAM（外部主記憶體晶片）用的**外部匯流排控制器**，共 2 個 channel，支援 LPDDR4-3200 與 LPDDR4X-3200，介面匯流排寬度 32-bit，並支援 in-line ECC（手冊 3.4.1 逐字："This unit is an external bus controller for LPDDR4 and LPDDR4X. This unit supports LPDDR4-3200 and LPDDR4X-3200. The interface bus width is 32 bits. It supports the in-line ECC feature."）。LPDDR（low-power DDR）是行動與嵌入式裝置常用的低功耗 DRAM 規格；「in-line ECC」指把校驗位元夾在一般資料裡一起存，不需要額外的 ECC 晶片就能做錯誤更正。

這個單元由兩塊組成：DDR controller block（記憶體控制器，簡稱 MC）與 DDR PHY block（實體層，簡稱 PHY）（手冊 3.4.1）。這裡要對你誠實標一個邊界：**手冊本身明講「這份手冊是簡化版」，PHY 的更多細節要另外看《User's Manual Additional Document》**（手冊 3.4.1 逐字："This manual is a simplified version. For more information, refer to the User's Manual Additional Document."）。所以本節講到 PHY 時只轉譯手冊有寫的部分，不會假裝這份手冊涵蓋 PHY 的全部細節。另外，這個區塊的實際設定（setup code）是由這顆 LSI 的 Linux 軟體套件負責，不是手冊本身教（手冊 3.4.1 逐字："For the setup of this block, refer to the Linux software package of this LSI, which contains the DDR setup codes."）——也就是說，DDR 怎麼被初始化，答案在軟體套件與開機韌體裡，不在暫存器手冊裡。

內部結構上（手冊 Figure 3.4-1，p278）：有 5 個 AXI4 埠接進 DDR Controller——port0／1／2 各 256-bit、port3／4 各 128-bit；DDR Controller 內部含 Address Shifter、Arbiter（仲裁器）、Command Queue、Write Data Holding Queue、DRAM Command Processing，以及 ECC／BIST／Low Power 等子區塊；再透過 DFI 介面接到 DDR PHY，最後接到外部 DRAM 的實體腳位。多個 AXI4 埠的意義是：CPU、加速器、顯示等不同來源可以各走各的埠向記憶體發請求，由 Arbiter 決定先後。

手冊 Table 3.4-1（p277）列出的 MC 能力值得記住幾條：完全管線化（fully pipelined）的指令／讀／寫資料介面、advanced bank look-ahead（提前開好記憶體 bank 以提高吞吐）、可程式化的 register 介面來控制記憶體參數與協定（含 auto pre-charge）、完整的開機自動初始化、Weighted Round-Robin（加權輪詢）仲裁各埠請求、ECC（single-bit／double-bit 錯誤回報、single-bit 更正、可程式化移除 ECC storage）、以及內建自我測試（BIST）。低耗電方面（同表）：支援 power-down、self-refresh、I/O retention 等多種低耗電狀態，可自動或由軟體介面切換。

### Linux 下怎麼看到它

**DRAM 沒有使用者裝置節點**——因為 DRAM 本身就是 Linux 核心記憶體管理（mm 子系統）直接管理的系統記憶體。你不會有一個 `/dev/ddr` 去開；你「用到 DRAM」的方式，就是配置任何一般記憶體（malloc、mmap、載入程式）時自然而然地用到它。

控制器與 PHY 的暫存器區塊（本板實測：DDR0 的 MEMC 約在 `0x14C00000` 一帶、DDR PHY 約在 `0x14E00000` 一帶，引自 doc07 對手冊 Table 1.8-1 的引用）由開機韌體／secure world（安全世界，Arm TrustZone 下的高權限執行環境）持有，**不曝露給 userspace**。也就是說，DDR 的初始化與時序調校由 BL2（第二階段開機載入器）在開機時做完，執行期的 Linux 只是「使用者」，不是「設定者」。

你能查的是記憶體的用量與規格，不是控制器暫存器：

```bash
free -h              # 看目前記憶體總量／已用／可用（人類可讀單位）
cat /proc/meminfo    # 看更細的記憶體統計（MemTotal 等）
```

本板 `cat /proc/meminfo` 的第一行是 `MemTotal: 15565904 kB`（約 14.8 GiB——這是核心納入配置器的量，已扣掉開機韌體與各硬體加速器保留區之後的數字），`MemAvailable` 約 `14863188 kB`（出處 `00_inventory.txt:90-97`；✅ 2026-07-17 板上重執行 `MemTotal` 一致仍為 `15565904 kB`（transcript：live/ch04-inventory.txt），`MemFree`／`MemAvailable` 隨當下負載浮動，比對基準時只認 `MemTotal`）。16 GB 實體記憶體並非全給 Linux——開機時就有一批 carveout 劃給加速器、顯示、相機與跨核通訊，逐項加總約 2.34 GB，完整的保留區地圖見 4.1〈記憶體保留區地圖〉，本節不重列。

至於頻寬量測：doc07 §48 只記「預期近 25.6 GB/s 峰值」為理論值、**明確標註「非實測」**，並建議可用 tinymembench 這類工具實際量。CPU 端實際量到的頻寬與延遲，本手冊 4.2〈記憶體頻寬與延遲〉有完整跑分（例如 STREAM Triad 4 緒實測 5585 MB/s，只達控制器理論峰值約 22%），這裡不重述；只提醒你**別把 25.6 GB/s 這個理論峰值當成實測數字引用**。

### 關鍵能力與限制

- **支援規格**：LPDDR4（JEDEC JESD209-4D）、LPDDR4X（JEDEC JESD209-4-1A）（手冊 3.4.1.1 逐字）。
- **DRAM 介面規格（手冊 Table 3.4-1 逐字）**：LPDDR4 ＝ 3200 Mbps（1600 MHz）；LPDDR4X ＝ 3200 Mbps（1600 MHz）；匯流排寬度 ＝ 32-bit（每 channel 16-bit）；rank ＝ 1 或 2；密度 ＝ **每 channel 最高 8 GB（byte mode 不支援）**。兩個 channel 相加，控制器規格層面的理論容量上限即 16 GB。
- **頻寬要把「兩顆控制器」一起算（別只用上一行的 32-bit）**：本 SoC 有 DDR0／DDR1 **兩顆**各 32-bit 的記憶體控制器（手冊暫存器一律命名 `DDR_MEMCm_*`，m＝0／1，即兩顆 MC 各一組），合計 64-bit 匯流排。所以理論峰值 ＝ 2 ×（3200 Mbps × 32-bit ÷ 8）＝ 2 × 12.8 ＝ **25.6 GB/s**；若只拿上一行的「32-bit」去套 3200 Mbps 只會算出 12.8 GB/s（正好一半，不是書上算錯）。真硬體手冊 `r01uh1032` §1.1 功能總覽圖 Figure 1.1-1（p77）逐字就寫作『32 bits × 2 (12.8 GB/s × 2)』——這正是全書一律用 25.6 GB/s（後面 4.2 的 STREAM 實測約 22% 也建在此峰值上）的來由。
- **料號後綴的一個但書（手冊 3.4.1 Note 1 逐字）**："This function is supported by the devices other than "#AC0" and "#BC0". "#AC0" and "#BC0" do not support it."——代表某些封裝後綴型號（#AC0、#BC0）不支援某項功能。手冊在此頁沒明講是哪項功能，所以你要核對自己拿到的實際料號後綴，**不可假設所有型號都支援**。
- **本板實測（doc07 §48，出處 `07-hardware-unit-usage-guide.md:797-809`）**：已裝載 16 GB LPDDR4／4X-3200（板卡規格總表寫 LPDDR4 1600 MHz、板卡方塊圖標 LPDDR4X-3200，控制器兩者皆支援），約 15.5 GB 可供 Linux 使用；25.6 GB/s 是理論峰值頻寬（doc07 原文明確標「非實測」）；由開機韌體 BL2 初始化雙 channel；DDR0／DDR1 的主要區域分別映射在 `0x40000000` 與 `0x140000000`（各 8 GB，引自 doc07 對手冊 Table 1.8-1 的引用）。
- **容量是本板配置，不是規格門檻（7-2 提醒）**：16 GB 是這片板子實際焊接並初始化的容量，分布在 DDR0／DDR1 兩顆控制器上（各 8 GB，見上文本板實測列）。這仍在規格內（每顆控制器每 channel 最高 8 GB），但**並未用到規格上限**：兩顆控制器合計最高可達 2 × 16 ＝ 32 GB，本板只配了一半。換一片板子焊更小或（在規格內）不同的容量，只會影響「能放多少資料」，**不影響控制器本身的能力**——速率、ECC、仲裁機制都不變。WS125 板卡手冊的規格表（`ws125-rdk-board-manual.md` Table 1）列 "Memory LPDDR4 1600MHz (8GB) x2"，與 doc07 實測的 16 GB（8 GB×2）互相印證，但這只是「這片板子配了多少」，不能推論成「RZ/V2H 只能配 16 GB」。

### 什麼情況下你會用到它

機制上，LPDDR4/4X 控制器是這顆 SoC **唯一的一般用途主記憶體路徑**：任何跑在 CA55（Linux）、CM33 或 CR8 上、且資料量超過各自 TCM（tightly-coupled memory，核心緊耦合記憶體）或 SRAM 容量的程式，最終都經由這條路徑存取記憶體。換句話說，你幾乎「無時無刻不在用它」，只是平常感覺不到。

真正需要你判斷的，是「某塊資料該放 SRAM 還是 DDR」。判斷準則：

- 當你的資料（模型權重、影像 frame buffer、一般應用程式的 heap／stack）**超出 SRAM 的 6 MB 或核心自身 TCM 的容量**，而且**不要求 SRAM 那種「與 DDR 仲裁無關」的確定性低延遲存取**時，就落在 DDR 這條路徑上。這涵蓋了絕大多數一般負載。
- 以 AI 推論為例：大型模型的權重與中介張量、影像的 frame buffer，都是典型會吃到這條路徑頻寬與容量的負載。（順帶一提，4.2 與 g1 群組提到的 DRP-AI3 那塊 512 MB carveout，也是從這個 DDR 池切出來的、而非 SRAM——這裡只作對照，其規格細節屬 g1，不在本節重述。）

一個要記住的能力邊界：DDR 容量很大（本板 16 GB），但**CPU 端能拉到的頻寬遠低於控制器理論峰值**（4.2 實測約 22%），這不是故障，是 A55 這種 in-order 小核心的記憶體層級平行度有限所致。真正需要高頻寬的影像／張量搬移，是靠加速器（DRP-AI3／DRP）的專屬 DMA 路徑達成、不經過 CPU——所以規劃系統時，「大量資料搬移」該想到的是加速器 DMA，而不是叫 CPU 去搬。

> **尾註**：《硬體使用手冊》r01uh1032ej0130 §3.4 LPDDR4/4X Controller (DDR)（p277–280，功能概說；暫存器自 3.4.2 p281）；PHY 詳細規格手冊自陳為簡化版，需另查《User's Manual Additional Document》。板上證據：`07-hardware-unit-usage-guide.md:797-809`（doc07 §48）、`00_inventory.txt:90-97`、`06-hardware-resource-map.md`（doc06 §3）；CPU 端頻寬跑分見本手冊 4.2。其他官方文件：datasheet r01ds0429（規格交叉核對）、WS125 板卡手冊 `ws125-rdk-board-manual.md` Table 1（板卡實裝容量）；RZ 系列 DRAM 相容清單 r01an7912 僅列名、用途待驗，未逐份讀取。

---

## xSPI（NOR flash 介面）

### 這是什麼

xSPI（Expanded Serial Peripheral Interface，擴充序列周邊介面）是一種為記憶體裝置設計的介面協定，訴求高資料吞吐、少訊號腳位，並與傳統 SPI 裝置維持有限度的回溯相容；它的電氣介面理論上可達每秒 200 MB 的原始資料吞吐（手冊 7.2.1 逐字："The xSPI protocol specifies the interface for Memory Devices, which provides high data throughput, low signal count, and limited backward compatibility with legacy SPI devices. The electrical interface can deliver up to 200 MB per second raw data throughput."）。它接的通常是 NOR flash——一種非揮發（斷電資料不失）、可位元組定址（byte-addressable，可以像記憶體一樣直接讀某一個位址）的 flash。

內部方塊上（手冊 Figure 7.2-1，p2502）：資料從內部周邊匯流排進來，經 FIFO、Bridge Control Channel 0／Channel 1、Command Control、Link Control，最後到 External signals（外部腳位）出去；另有一塊獨立的 Register Control 區塊管設定。

它支援多種協定模式（手冊 Table 7.2-1 逐字列舉），差別在用幾根腳位、以及是 SDR 還是 DDR（single／double data rate，每個時脈邊緣傳 1 次或 2 次資料）：

- **1／4／8-pin，SDR 或 DDR**：`1S-1S-1S`、`4S-4D-4D`＊1、`8D-8D-8D`＊1（註 1：不接 XSPI0_DS 訊號時不支援 DDR 存取）。
- **2／4-pin，SDR**：`1S-2S-2S`、`2S-2S-2S`、`1S-4S-4S`、`4S-4S-4S`。
- 另外可設定位址長度、可設定初始存取延遲週期，並支援 **XiP**（execute-in-place，就地執行——程式碼不必先複製到 RAM，CPU 直接從 flash 上執行）。

在功能面（同表），還有幾項值得知道：支援 Write Data Mask；支援 In-band Reset（符合 JESD252 標準）；記憶體映射（memory-mapping，把 flash 內容映射到一段位址、可像讀記憶體一樣讀 flash）最高可達 256 MB 位址空間（每個 CS〔chip select，晶片選擇線〕128 MB，或 CS0 單獨可用到 256 MB）；prefetch 功能降低 burst-read 延遲；outstanding buffer 提升 burst-write 吞吐；manual command 可設定最多 4 組指令並支援 status register polling；也支援 Input Strobe port timing shift。整個單元只有 1 個 channel，可作為 master 對最多 2 個 slave 發起交易，並有 2 個中斷來源。

位址映射方面（手冊 Table 7.2-3，p2503），這是**預設值、不是寫死的**：CS0 的內部位址是 `0_2000_0000h`（從 CM33 看 non-secure ＝ `7000_0000h`／secure ＝ `6000_0000h`）、CS1 是 `0_2800_0000h`（`7800_0000h`／`6800_0000h`）；表格附註明講這些可透過 `SYS_SPI_STAADDCS1`、`SYS_SPI_ENDADDCS0-1` 這些暫存器更改。

### Linux 下怎麼看到它

本板把 xSPI 拿來當開機用的 NOR flash，Linux 端的路徑是：`xspi-if` 驅動程式 → MTD 子系統 → `/dev/mtd0..mtd3`（另有對應的 `/dev/mtdblock*`）。MTD（Memory Technology Device）是 Linux 對 flash 這類「可抹除、以區塊為單位」儲存裝置的抽象層，不同於一般硬碟／SD 卡那種區塊裝置。

控制器的暫存器 base address 是 `0x11030000`（device tree 節點名為 `xspi@11030000`）；記憶體映射讀取區則是 256 MB，映射在 `0x10000000`（doc07 引自手冊 Table 1.8-1）。這對應 datasheet 的開機模式 2（serial flash on xSPI），也就是這片板子開機時的 flash 來源。

本板的 NOR flash 被切成四個 MTD 分割區，各有明確用途（分割區名逐字，✅ 2026-07-17 板上重執行 `cat /proc/mtd` 確認，transcript：live/ch04-inventory.txt；出處 `07-hardware-unit-usage-guide.md:769-781`、`04-hardware-quickref.md:278`）：

```text
dev:    size   erasesize  name
mtd0: 0001d200 00001000 "bl2"
mtd1: 001c2e00 00001000 "fip"
mtd2: 00020000 00001000 "env"
mtd3: 00e00000 00001000 "test-area"
```

`mtd0`＝bl2（第二階段開機載入器）、`mtd1`＝fip（韌體映像包）、`mtd2`＝env（U-Boot 環境變數）、`mtd3`＝test-area。查詢與燒錄工具是 mtd-utils（`flash_erase`、`flashcp`、`mtdinfo`）：

```bash
cat /proc/mtd                          # 列出所有 MTD 分割區
mtdinfo /dev/mtd2                      # 看某分割區的詳細資訊
sudo flash_erase /dev/mtd2 0 0         # 抹除整個 env 分割區
sudo flashcp -v new-env.bin /dev/mtd2  # 把新映像寫進 env 分割區
```

> ⚠️ **注意（別對開機韌體分割區做燒錄實驗）**：
> - **情境**：你想拿 `flash_erase`／`flashcp` 對 xSPI NOR flash 的 MTD 分割區做寫入測試。
> - **症狀**：若動到 `mtd0`／`mtd1`，板子下次可能開不了機。
> - **原因**：`mtd0`＝bl2、`mtd1`＝fip 是開機用的韌體映像，抹掉就沒東西可開機。
> - **預防／處理**：要做寫入測試，只碰 `mtd2`（env）或 `mtd3`（test-area），**絕不動 `mtd0`／`mtd1`**（出處 `07-hardware-unit-usage-guide.md:769-781`）。

### 關鍵能力與限制

- **電氣介面理論吞吐上限：200 MB/s**（手冊 7.2.1 逐字）。
- **定址空間：最高 256 MB**（每 CS 128 MB，或 CS0 單獨 256 MB）（手冊 Table 7.2-1）。
- **本板實際狀態（doc07 §46，出處 `07-hardware-unit-usage-guide.md:769-781`）**：`xspi@11030000` 作為開機用 NOR flash，以 MTD 分割區暴露（mtd0=bl2／mtd1=fip／mtd2=env／mtd3=test-area），對應 datasheet 開機模式 2。
- **本板焊的 flash 晶片容量是 64 MB，不是規格門檻（7-2 提醒）**：WS125 板卡手冊 Table 1 逐字寫 "QSPI Flash ROM 64MB"，也就是這片板子焊的是一顆 64 MB 的 flash，**遠小於**控制器規格上限 256 MB。這是本板的具體配置，不是控制器的容量門檻——不同板卡可能焊不同容量的 flash，實際可用空間看板上焊的是哪一顆。選型時該看的是控制器支援的協定與定址上限（200 MB/s、256 MB），而不是某片板子剛好焊了多大的 flash。

### 什麼情況下你會用到它

先講機制：xSPI 接的 NOR flash 有三個決定它用途的特性——**位元組定址**（可以像記憶體一樣讀任一位址）、**可 XiP 直接執行**（程式碼不必先複製到 RAM 就能跑）、以及**容量通常是 MB 級但開機路徑穩定**。這正是它被列為序列 flash 開機模式（開機模式 2）的機制性原因：開機最初階段需要一塊「一通電就能可靠地讀到、甚至直接執行」的非揮發記憶體，NOR flash 的這幾個特性剛好對上。

由此給你判斷準則，幫你決定「這批資料該放 NOR flash 還是別的地方」：

- **適合走 xSPI／NOR 的**：需要「開機用、小容量、高可靠度、可不經複製直接執行程式碼」的場景——bootloader、少量設定值（以 U-Boot 環境變數為例）。
- **不該走 xSPI／NOR 的**：需要 GB 級大量資料儲存的場景——以作業系統根目錄、資料記錄、模型檔案為例，這些該用 SDHI／eMMC 那類區塊裝置（見下一節），而不是 xSPI／NOR。

本板把 xSPI 留給 `bl2`／`fip`／`env` 這類小型開機映像、把 rootfs 放到 SD 卡，反映的正是「NOR flash」與「區塊裝置」的本質差異，不是本板獨有的巧思。任何 RZ/V2H 應用只要遵循相同的「開機小映像 vs. 大量資料儲存」分工原則，都會得到類似的介面選擇。

> **尾註**：《硬體使用手冊》r01uh1032ej0130 §7.2 Expanded SPI (xSPI)（p2501–2503，功能概說；暫存器自 7.2.2 p2504）。板上證據：`07-hardware-unit-usage-guide.md:769-781`（doc07 §46）、`04-hardware-quickref.md:278`、`06-hardware-resource-map.md`（doc06 §2）；MTD 逐字輸出 ✅ 2026-07-17 板上重執行確認。其他官方文件：WS125 板卡手冊 `ws125-rdk-board-manual.md` Table 1（板卡 flash 晶片容量）；工具 mtd-utils。開機模式對照出處 datasheet r01ds0429 Table 1.3-4。

---

## SDHI／eMMC／SDIO Host ×3

### 這是什麼

這是 3 個 channel 的 SD／MMC 主機介面（host interface——SoC 這一端主動發起交易、去讀寫外接的 SD 卡或 eMMC 的控制器）。三個 channel 的能力並不對等：**channel 0 支援 SD 與 e-MMC**；**channel 1、channel 2 只支援 SD**（手冊 6.2.1.1 逐字："Channel 0 supports SD and e-MMC. Channel 1 supports SD. Channel 2 supports SD."）。

手冊在這一章開頭有一個 CAUTION 框，逐字："Development of the SD host-related products needs the conclusion of the following agreement. • "SD Host/Ancillary Product License Agreement (SD HALA)""——這是 Renesas 對「開發 SD host 相關產品」附帶的授權協議提醒。轉譯時照實提醒你有此授權前提即可，本節不展開流程細節。

Features 方面，手冊 6.2.1.1 逐字列了一長串，挑對你有意義的講：SD memory／IO 卡介面（1-bit／4-bit SD bus）；支援 SD、SDHC、SDXC 記憶卡存取；符合 SD specification version 3.01；支援 Default、high-speed、UHS-I／SDR50、SDR104、DDR50 等傳輸模式；SD clock（SD_CLK）頻率 ＝ `SDHI_x_IMCLK 頻率 / 2ⁿ`（n ＝ 0 到 9，x ＝ 0 到 2 為通道編號）；錯誤檢查用 CRC7（command／response）與 CRC16（data）；有 2 個中斷請求；支援卡片偵測與寫入保護。在 MMC 側：MMC 介面（1-／4-／8-bit MMC bus）；支援 e-MMC 裝置存取；支援 Backward-compatible、high-speed、HS-DDR、HS200 等傳輸模式；支援 High-priority interrupt（HPI）；並支援 SDIO 3.0。

內部方塊（手冊 Figure 6.2-1，p1517）：AXI bus 分別接 AXI master I/F 與 AXI slave I/F，master I/F 經 DMAC 接到 Host I/F，再接到 SD/MMC I/F 出去到實體的 SD/MMC bus；Host I/F 與 SD/MMC I/F 各自搭配一塊 RAM 緩衝；另有獨立的中斷請求輸出。三組控制器的暫存器 base address（手冊 Table 6.2-2 逐字）：SD0 ＝ `0_15C0_0000h`（CM33 non-secure `55C0_0000h`／secure `45C0_0000h`）、SD1 ＝ `0_15C1_0000h`、SD2 ＝ `0_15C2_0000h`。

### Linux 下怎麼看到它

本板把 SDHI channel 0 拿來承載 rootfs（根檔案系統）。Linux 端的路徑是：`renesas_sdhi`／`tmio_mmc` 驅動程式 → 區塊裝置節點 `/dev/mmcblk0`（外加分割區 `mmcblk0p1..`）。如果插的是 SDIO 卡，也會走同一條 mmc／sdio bus 掛載。

本板實際狀態（doc07 §47，出處 `07-hardware-unit-usage-guide.md:783-795`）：`SDHI0 @0x15C00000` 承載 rootfs 於 `/dev/mmcblk0`（一張 256 GB 的 Samsung SD 卡）；另外兩個 eMMC 控制器 `@0x15C10000`、`@0x15C20000` 在 device tree 中被停用（disabled）——因為這片板子沒有 populate（焊上／插上）實體 eMMC。所以本手冊資源地圖把這個單元的整體狀態標成「部分啟用」：SDHI0 活著、兩個 eMMC 控制器關著（✅ 2026-07-18 板上抽驗 `mmc@15c10000`／`mmc@15c20000` 皆為 `disabled`，transcript：live/ch04b-dt-status.txt）。

查詢與操作工具是 util-linux（`lsblk`、`fdisk`）、e2fsprogs、mmc-utils：

```bash
lsblk                                # 看區塊裝置與分割區樹
sudo fdisk -l /dev/mmcblk0           # 看這張卡的分割表
cat /sys/block/mmcblk0/device/name   # 讀卡片 CID 裡的產品名
mmc extcsd read /dev/mmcblk0         # 主要用於 eMMC；用在 SD 卡上主要顯示卡片資訊
```

本板這張開機卡的身分（出處 `05-compute-benchmark.md:48`、`04b_emmc_id.txt:2-8`）：type SD、manfid `0x1b`、oemid `"SM"`、製造年月 2025／10，ext4 格式，可用約 206 GB。它的速度屬 UHS-I SD 卡等級，不是 eMMC 5.1 的 150–300 MB/s。完整的儲存 I/O 跑分（序列讀寫、fio 隨機讀寫等）在本手冊 4.2〈儲存 I/O〉，這裡不重列。

> ⚠️ **注意（腳本把開機卡標成 eMMC，實際查是 SD 卡）**：
> - **情境**：你想確認開機儲存到底是什麼裝置。
> - **症狀**：盤點腳本輸出 `04_storage.txt` 的檔頭把它標成「eMMC」，但卡片身分回報卻是 **SD 卡**（腳本標籤與實測裝置身分不一致）（出處 `05-compute-benchmark.md:191`、`04_storage.txt:1`）。
> - **原因**：本板出貨採用外接 SD 卡作為開機媒體，腳本的標籤沿用了通稱、與實際插的卡不符——以卡片自己回報的身分（`/sys/block/mmcblk0/device/type` 等）為準才對。
> - **預防／處理**：判斷開機儲存是什麼，別信腳本檔名，改讀卡片 CID／type；要確認速度等級，看它跑在哪個傳輸模式（UHS-I／SDR104／DDR50），而不是看容量。

### 關鍵能力與限制

- **3 channel，能力分工不對等**：channel 0 ＝ SD＋eMMC、channel 1／channel 2 ＝ SD only（手冊 6.2.1.1）。
- **符合 SD specification version 3.01，支援到 SDXC 等級**，傳輸模式含 UHS-I／SDR50／SDR104／DDR50（SD 側）與 HS-DDR／HS200（eMMC 側）（手冊 6.2.1.1）。
- **本板狀態（doc07 §47）**：SDHI0 承載 rootfs 於 `/dev/mmcblk0`（256 GB Samsung SD 卡）；兩個 eMMC 控制器 DT 停用（本板未 populate 實體 eMMC）。
- **卡片容量是本板配置，不是控制器門檻（7-2 提醒）**：256 GB 是這片板子實際插的 SD 卡容量，不是 SDHI 控制器的容量門檻——控制器本身支援到 SDXC 規範等級（理論上限遠高於 256 GB）。更大或更小容量的卡只影響「能存多少資料」，**不影響傳輸速度上限**——速度上限由 UHS-I／SDR104／DDR50 等傳輸模式決定，與卡容量無關。
- **原廠板卡的建議卡（WS125 板卡手冊 3.9 節逐字）**："The RDK has micro SD card connectors (SD1). SD1 is connected to the SD0 interface of RZ/V2H and can be used as a boot device. The power supply VDD1833_SD0 of RZ/V2H is fixed to 1.8V for high-speed data transfer. Renesas recommends using the built-in SanDisk 64GB SD card for high-speed data transfer."——這是原廠板卡出貨時搭載並建議使用的卡。不同批次或你自己換卡都可以，只要支援 UHS-I 規格即可達到高速模式；「64 GB」只是原廠出貨建議，同樣不是規格門檻。

### 什麼情況下你會用到它

先講機制：SDHI／eMMC 走的是「**大容量、可透過標準檔案系統操作的區塊儲存**」路徑——資料以區塊（block）為單位、掛上檔案系統後可以像一般硬碟那樣 mount／複製／格式化。這跟 xSPI／NOR 那種位元組定址、可直接執行的路徑本質不同。

由此給你判斷準則：

- **會落在 SDHI 這條路徑上的**：只要應用需要 GB 級資料（以作業系統映像檔、記錄檔、資料集為例）、且需要標準檔案系統操作（掛載、複製、格式化），就用 SDHI／SD／eMMC。
- **不該用這條路徑的**：小容量、需要開機時就可靠讀取甚至就地執行的映像（bootloader、環境變數），該走 xSPI／NOR（見上一節）。

在 SD 與 eMMC 之間怎麼選，是一個**應用面的取捨**，不是控制器規定必須用哪一種。本板選用 SD 卡（SDHI0）作為開機＋rootfs 裝置，對應 datasheet Table 1.3-4 的開機模式 0（booting from eSD），是一個具體實例；改成 eMMC（開機模式 1）則是同一介面家族的另一個實例。兩者的取捨在於：SD 卡可抽換，方便量產燒錄與更新；eMMC 直接焊在板上，較不怕震動導致接觸鬆脫或卡片脫落。因此——**以會受震動影響的應用（例如以移動載具為例）為例**，常因機械穩固性優先考慮 eMMC；**以不受震動影響的應用（例如以固定式桌上型系統為例）為例**，則可能更看重 SD 卡的可抽換與方便更新。這是應用面的判準，讀者要依自己的使用情境自行判斷，而不是照抄任何一片板子剛好用了哪一種。

> ⚠️ **注意（高可靠、斷電安全的資料紀錄，別直接押在消費級 SD 卡上）**：
> - **情境**：你想把開機用的這張 SD 卡，同時拿來當高可靠的資料紀錄裝置（以飛行黑盒子這類需要斷電後資料仍完整的紀錄為例）。
> - **症狀**：長期高頻寫入下，消費級 SD 卡的寫入耐久性與斷電（突然失去電源）可靠性會成為風險。
> - **原因**：出貨採用的是消費級外接 SD 卡，其寫入耐久性與斷電資料完整性，對「高可靠資料紀錄」這種需求是實實在在的弱點（出處 `05-compute-benchmark.md:196-198`）。
> - **預防／處理**：這類需求應改用板上 eMMC 或工業級 pSLC 卡，並把高頻 log 與 rootfs 分離到不同儲存裝置，避免互相拖累。至於一般負載——以影像加感測資料記錄為例，4 Mbps 影像串流約 0.5 MB/s——這張卡的速度綽綽有餘（出處 `05-compute-benchmark.md:128-131`），要不要升級要看你的可靠度需求，不是速度需求。

> **尾註**：《硬體使用手冊》r01uh1032ej0130 §6.2 SD/MMC Host Interface (SD)（p1516–1521，功能概說；暫存器自 6.2.2）。板上證據：`07-hardware-unit-usage-guide.md:783-795`（doc07 §47）、`05-compute-benchmark.md:48,191,196-198`、`04b_emmc_id.txt:2-8`、`04_storage.txt:1`；儲存跑分見本手冊 4.2；`mmc@15c10000`／`15c20000` 停用狀態 ✅ 2026-07-18 板上抽驗。其他官方文件：WS125 板卡手冊 `ws125-rdk-board-manual.md` 3.9 節（SD 卡連接器與建議卡）；開機模式對照 datasheet r01ds0429 Table 1.3-4；工具 util-linux／e2fsprogs／mmc-utils。
