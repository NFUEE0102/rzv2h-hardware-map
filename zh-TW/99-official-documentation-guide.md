# 99 · 官方文件查閱指路（4.4）

> 本檔是「04 · 全板硬體資源地圖」資料夾的官方文件指路檔（對應章節號 **4.4**）。章首說明與 4.1 總覽在 [00-總覽與IP啟用地圖](00-overview-and-ip-enablement-map.md)，運算單元在 [01-運算單元](01-compute-units.md)，各周邊單元在 g2–g8 群組檔——那些檔案裡引用的官方手冊章節與頁碼，全部可以用本檔教的方法自己查到。

前面各檔你已經用到一大堆硬體事實——某個單元的暫存器起始位址、某條匯流排跑在哪個時脈、DRP-AI3 的 512 MB 保留區從哪裡開始。這些數字沒有一個是憑空捏造的，它們全都能在手冊倉庫隨附的官方文件資料夾裡查到原始出處。這一份檔案的目的，是把「那一疊文件」變成**你自己能查的地圖**：遇到一個問題，你要能立刻判斷該翻哪一份、翻到哪一章，而不是把 43 MB 的手冊從頭滑到尾。

會查文件，比記住任何一個數字都重要。因為這本手冊裡的每個實測值都有量測條件、都可能因為你手上的板子版本或映像檔而略有不同；但「官方文件怎麼查」是一輩子的技能——你以後要接的新周邊、要設的新暫存器、要對的新規格，答案都在同一疊文件裡。

## 本檔目錄

- [為什麼是「一疊」而不是「一本」](#為什麼是一疊而不是一本)
- [資料夾裡實際有什麼：9 份文件](#資料夾裡實際有什麼9-份文件)
- [硬體手冊太大，用 `_toc_full.txt` 當索引](#硬體手冊太大用-_toc_fulltxt-當索引)
- [查哪章：常見單元 → 章節／頁 一覽](#查哪章常見單元--章節頁-一覽)
- [只查規格數字：datasheet 的 Section 1 就夠](#只查規格數字datasheet-的-section-1-就夠)
- [那份 AWO／韌體部署文件（進階，先知道在哪）](#那份-awo韌體部署文件進階先知道在哪)
- [官方線上資源](#官方線上資源)
- [動手驗證：確認你會用這疊文件當地圖](#動手驗證確認你會用這疊文件當地圖)

## 為什麼是「一疊」而不是「一本」

新手最常見的誤會，是以為晶片原廠會給你「一本大全」，什麼都查得到。實際上 Renesas（以及幾乎所有 SoC 原廠）把資訊**依用途拆成好幾份**，每一份回答一種不同性質的問題。搞清楚這個分工，你就不會拿著 datasheet 去找暫存器 offset（找不到），也不會抱著 4800 頁的硬體手冊去查「這顆晶片到底有幾條 CAN」（太慢）。

這疊文件大致分成五種角色，各自回答一種問題：

```text
  你心裡的問題                        該找哪一種文件                回答的層次
 ───────────────────────────────────────────────────────────────────────────
 「這顆晶片有什麼？多少？」    →   Datasheet（規格總表）      「是什麼 / 多少」
 「這個單元的暫存器怎麼設？」  →   User's Manual: Hardware    「暫存器 / bit / 位址」
                                    （硬體手冊，暫存器級）
 「這個功能怎麼跑起來？」      →   Application Note /         「操作步驟 / 流程」
                                    Quick Start（應用/上手）
 「軟體介面（如 GStreamer）    →   User's Manual: Software    「element / API / 管線」
   的用法與參數？」                 （軟體手冊）
 「為什麼要這樣設計？」        →   White Paper（白皮書）      「架構 / 原理 / 取捨」
```

另外還有一個**層級**上的分工要先分清楚：上面這些都是講 **SoC（晶片）**的。而「這塊**板子**的連接器接哪裡、40-pin header 哪支腳是什麼、DIP 開關怎麼撥」是**載板（board）層**的問題，查的是 RDK 板卡手冊（見下面 WS125 那份）；晶片手冊裡沒有這些答案。

把這張表記成一句口訣就夠用了：**問「是什麼、多少」找 datasheet；問「暫存器怎麼設」找硬體手冊；問「怎麼跑起來」找 application note；問「軟體介面怎麼用」找軟體手冊；問「為什麼」找白皮書；問「板子怎麼接」找板卡手冊。**

> 💡 **提示**：Renesas 的文件編號前綴本身就會告訴你它是哪一種文件，不用打開就能猜個八九不離十。`R01DS` = Datasheet（規格書）、`R01UH`／`R16UH` = User's Manual: Hardware（硬體手冊；`R16UH` 為板卡層）、`R01US` = User's Manual: Software（軟體手冊）、`R01AN` / `R20AN` = Application Note（應用說明）、`R01QS` = Quick Start（快速上手）、`R01WP` = White Paper（白皮書）。看到檔名開頭的這幾個字母，你就先知道它回答哪一層的問題了（依 Renesas 文件編號慣例整理；各份的實際身分已按下表逐一核對檔案自報的標題）。

## 資料夾裡實際有什麼：9 份文件

官方文件放在手冊倉庫同層的 `reference-docs/`（自本檔所在資料夾起算為相對路徑 `../../reference-docs/`）。先把它列出來看看有哪些：

```bash
ls -la ../../reference-docs/
```

實際輸出（逐字，2026-07-18 於本資料夾實跑；日期時間欄是你複製倉庫的時間，會與此不同）：

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

九個 `.pdf` 檔名一字排開，用上面的編號前綴心法掃一遍，你已經能認出誰是誰；另外兩個非 PDF 的檔案是 `README.md`（資料夾自己的說明）與 `_toc_full.txt`（硬體手冊的完整目錄，下一節整節都在講它——上面這份輸出沒列到它是因為它以底線開頭排在最後，`ls -la` 完整輸出的最後一行是 `246904 … _toc_full.txt`）。

> **關於「份數／檔案大小」的準據**：本節的文件清單與位元組數，一律以上面對 `../../reference-docs/` 直接跑 `ls -la` 的輸出為準（9 份 PDF，datasheet `r01ds0429` 為 264,840 B 的文字轉檔）。後面幾個 ✅ 驗證步驟引用的 transcript `live/ch04b-docs-local.txt` 只用來佐證 `_toc_full.txt` 的 `grep` 內容（章頁碼、CAN-FD／I2C 行數等），**不作為文件清單或檔案大小的依據**——若你手上另有一份對「別份文件副本」跑出來的 `ls`（例如份數或 datasheet 大小對不上），以你自己剛剛對 `reference-docs/` 跑出的這份為準即可，兩者是不同副本、不是矛盾。

下面這張表把每一份的角色、大小（位元組數逐字取自上面 `ls -la`）、以及**目前被讀取到什麼程度**都列清楚——「讀取狀態」這一欄尤其要看，關係到你能不能信任這份手冊對它的描述。頁數與檔案自報標題以 `pdfinfo`（poppler-utils）於工作站實跑讀出，2026-07-18：

| 檔名（前綴類型） | 大小 | 什麼問題查它 | 讀取狀態／格式注意 |
|---|---|---|---|
| `r01ds0429ej0130-rzv2h.pdf`（R01DS · Datasheet，Rev.1.30，2025-09-05，144 頁） | 264,840 B（≈259 KB） | RZ/V2H 晶片**規格總表**：「這顆晶片有幾條 CAN／幾個通道／最高幾 MHz」「料號之間差什麼」——查 Section 1（Table 1.3-x 子表、Table 1.4 單元縮寫對照） | **已讀 Section 1 Overview**。⚠️ 格式注意：副檔名雖是 `.pdf`，內容實為**純文字轉檔**（開頭第一行即自報 `R01DS0429EJ0130 Rev.1.30 Page 1 of 144`），一般 PDF 閱讀器打不開——直接用文字編輯器或 `grep` 讀即可；要看**圖**（如方塊圖）得另取 Renesas 原版 PDF |
| `r01uh1032ej0130-rzv2h.pdf`（R01UH · 硬體手冊，Rev.1.30，4816 頁） | 43,861,979 B（≈43.9 MB） | 完整**硬體使用手冊（暫存器級）**：每個單元的暫存器 offset、bit 定義、bare-metal 位址、章節細節——「暫存器怎麼設」全在這裡 | **未逐頁讀取**；其**目錄**已抽成 `_toc_full.txt` 當索引（見下節）；`pdfinfo` 自報標題 `RZ/V2H Group User's Manual: Hardware (Non-Agreement)`、`Pages: 4816` |
| `r01an7723ej0400-…-awo-…-startup-guide.pdf`（R01AN · App Note，17 頁） | 1,105,559 B（≈1.1 MB） | RZ/V2H・RZ/V2N **AWO（CM33 喚醒／睡眠控制）範例程式啟動指南**：CM33 韌體部署、Yocto 建置流程 | **已透過對應 MD 完整讀取**（重點與坑見本檔末〈那份 AWO／韌體部署文件〉） |
| `r01qs0077ej0400-rzv2h-multi-os-pkg.pdf`（R01QS · Quick Start，32 頁） | 2,254,319 B（≈2.3 MB） | **Multi-OS Package 快速上手**：要讓 R8／M33 跑韌體、A55 與它們 RPMsg 通訊時，第一份要翻的文件（01 檔提到的文件編號 R01QS0077 就是它） | **內文未讀取**；`pdfinfo` 自報標題 `RZ/V2H Quick Start Guide for RZ Multi-OS Package`（標題已核對，內容描述依標題與編號） |
| `r01us0653ej0202-rzv2h_rzv2n_GStreamer_UME.pdf`（R01US · 軟體手冊，Rev.2.02，89 頁） | 2,918,713 B（≈2.9 MB） | **GStreamer UME（Linux Interface Specification GStreamer User's Manual）**：`v4l2src`／`vspmfilter`／`omxh264enc` 這些 element 的參數、buffer 管理與管線配方——第 2 章影像管線用到的 GStreamer 語法，官方依據就是這一類手冊 | **內文未逐頁讀取**；`pdfinfo` 自報標題 `RZ/V2H Group and RZ/V2N Group User's Manual: Software`（標題已核對；「element／buffer 管理」的角色描述沿自倉庫 `reference-docs/README.md` 既有條目） |
| `r01wp0022eu0100-rzv2h-drp-ai3.pdf`（R01WP · White Paper，10 頁） | 1,396,682 B（≈1.4 MB） | **DRP-AI3 架構白皮書**：8/80 TOPS 怎麼來、sparse pruning、10 TOPS/W 與 fanless 訴求——「為什麼這樣設計」查它 | 已有對應 MD（由 GPU／NPU 深入挖掘任務讀取） |
| `r01an7912ej0104-rz-family-dram-list.pdf`（R01AN · App Note，12 頁） | 434,057 B（≈434 KB） | 推測為 RZ 系列**相容 DRAM（LPDDR4/4X）型號清單**——自製載板選記憶體晶片時查 | **未讀取，僅列名**（用途依編號與檔名推測） |
| `r20an0842ea0401-rzv2h-evk-exampleprojects.pdf`（R20AN · App Note，12 頁） | 365,249 B（≈365 KB） | RZ/V2H **EVK 範例專案集**說明——找官方範例程式的地圖 | **內文未讀取**；`pdfinfo` 自報標題 `RZV2H-EVK Example Project Bundle`（標題已核對） |
| `REN_WS125V2HRDKREFZ_MAH_20260323.pdf`（R16UH · **板卡**硬體手冊，R16UH0052EU0100 Rev.1.00，2026-03-23，13 頁） | 1,684,127 B（≈1.7 MB） | **WS125-V2HRDKREFZ RDK 載板手冊**：連接器與腳位（40-pin header、相機連接器）、DIP 開關、SD 卡連接器與建議卡、板卡實裝的 flash——**板子層**的問題查它，晶片手冊沒有這些 | ⚠️ 格式注意：副檔名雖是 `.pdf`，實際是 **ZIP 壓縮檔**（內含 `1.jpeg`–`13.jpeg` 頁面影像＋`1.txt`–`13.txt` 頁面文字＋`manifest.json`），一般 PDF 閱讀器打不開——先解壓（`unzip`）再看頁面影像或文字。開發紀錄曾以其 MD 轉寫（`ws125-rdk-board-manual.md`）引用 Table 1（板卡 flash）與 3.9 節（SD 卡連接器） |

> ⚠️ **注意（兩個「假 PDF」——打不開不是檔案壞了）**：
> **情境**：你用 PDF 閱讀器開 `r01ds0429ej0130-rzv2h.pdf`（datasheet）或 `REN_WS125V2HRDKREFZ_MAH_20260323.pdf`（板卡手冊）。
> **症狀**：閱讀器報格式錯誤打不開；`pdfinfo` 也回 `May not be a PDF file`。
> **原因**：這兩個檔案只是**沿用了 `.pdf` 副檔名**——datasheet 那份實際內容是純文字轉檔（`file` 指令判為 `data`，開頭即為文件自報頭 `R01DS0429EJ0130 Rev.1.30 Page 1 of 144`）；板卡手冊那份實際是 ZIP（`file` 判為 `Zip archive data`，內含頁面影像與文字）。
> **預防／處理**：datasheet 直接用文字編輯器／`less`／`grep` 讀（本手冊引用它的內容都是文字，正好夠用）；板卡手冊先 `unzip` 再看。要原版排版與圖，去 Renesas 官網照文件編號（R01DS0429、R16UH0052）另行下載。

> ⚠️ **注意**：表格「讀取狀態」欄標「未讀取」「內文未讀取」的那幾份，它們的「什麼問題查它」是**依文件編號慣例＋檔案自報標題**寫的，沒有逐頁讀過內文。
> **情境**：你照著上表，直接在報告或設計文件裡寫「DRAM 相容清單見 `r01an7912`」。
> **症狀**：真打開來，內容可能與你以為的用途有出入（例如涵蓋範圍、適用型號不同）。
> **原因**：這幾份的描述來自「編號前綴＋檔名＋標題」的推斷，不是讀過內文的結論——推斷通常對，但不保證。
> **預防**：要**精確引用**這幾份的內容前，先自己把檔案打開翻一下確認；把「推測用途」當成「先找到大概哪一份」的線索，而不是可以直接轉述的事實。

## 硬體手冊太大，用 `_toc_full.txt` 當索引

會查文件的人，八成時間都花在那本 43.9 MB 的硬體手冊（`r01uh1032ej0130-rzv2h.pdf`）上——因為只有它才有暫存器級的細節。但它有 4816 頁，你不會想整本翻。好消息是資料夾裡已經把它的**完整目錄**抽成一個純文字檔 `_toc_full.txt`（246,904 位元組），你可以用 `grep` 在裡面秒查任何單元落在哪一章、哪一頁。

**先誠實說一件事：這個檔案自己沒有標題頁自報身分。** 打開 `_toc_full.txt` 開頭是 `Cover` / `Notice` / `How to Use This Manual` / `Table of Contents`，並沒有一行寫「我是 r01uh1032」。所以「它就是硬體手冊的目錄」這句話，是**推斷**出來的——但這個推斷有四條你自己也能驗證的硬證據：

1. **章節編號逐一對得上。** 這份目錄的章節編號，跟本章各檔引用的「hw_manual 章節」完全吻合：`4.6 Interrupt Controller` 就是 GIC 那章、`7.4 SCIF`、`9.2 Camera Data Receiver Unit (CRU)`⋯⋯每一個都對。若它是別份文件，編號不會這麼巧全部命中。
2. **頁數量級對得上。** 這份目錄一路編到約 4816 頁（結尾是 `APPENDIX A PACKAGE DIMENSIONS`、`REVISION HISTORY`），符合 43.9 MB 的完整硬體手冊；datasheet 全本才 144 頁（它的文字轉檔開頭自報 `Page 1 of 144`），不可能是它。
3. **版次對得上。** 目錄結尾 `REVISION HISTORY` 最新一版是 `1.30`，正好對應檔名 `r01uh1032**ej0130**` 的 `0130`（＝ Rev. 1.30）。
4. **`pdfinfo` 直接對得上。** 對 PDF 本體跑 `pdfinfo ../../reference-docs/r01uh1032ej0130-rzv2h.pdf`，自報標題 `RZ/V2H Group User's Manual: Hardware (Non-Agreement)`、`Pages: 4816`——頁數與目錄結尾（`4816 Back Cover`）一字不差（2026-07-18 工作站實跑）。

四條證據指向同一個結論：`_toc_full.txt` 就是硬體手冊 `r01uh1032ej0130` 的完整目錄。下面就放心把它當索引用。

先看它最頂層的骨架——整本手冊分成 10 個 SECTION。你可以自己抓出來：

```bash
grep -E "^[0-9]+	SECTION " ../../reference-docs/_toc_full.txt
```

（`grep` 樣式裡那個是 Tab 字元：檔案每一行的格式是「頁碼＋Tab＋章節標題」。）預期輸出（逐字，✅ 2026-07-18 重執行核對一致，transcript：live/ch04b-docs-local.txt；每行前面的數字是**該章在硬體手冊 PDF 裡的起始頁**）：

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

這 10 個 SECTION 就是你腦中的第一層分類。看標題大概就能猜到要往哪走：處理器（含 NPU/DRP）在 SECTION 2、記憶體（含 DDR/SRAM/TZC）在 SECTION 3、系統核心（PFC 接腳、CPG 時脈、中斷、DMAC）在 SECTION 4、計時器在 5、高速介面（SD/乙太/USB/PCIe）在 6、低速介面（xSPI/序列埠/I²C/CAN/ADC/溫度感測）在 7、音訊在 8、影像（相機/縮放/顯示/編解碼/GPU）在 9、電氣特性在 10。

下一步是往下鑽一層，找到你要的**單元**在第幾章、第幾頁。假設你在接乙太網路，想查 GBETH 的暫存器，直接 grep 單元名：

```bash
grep "Gigabit Ethernet" ../../reference-docs/_toc_full.txt
```

預期輸出（逐字，✅ 2026-07-18 重執行核對一致，transcript：live/ch04b-docs-local.txt）：

```text
1632	  6.3 Gigabit Ethernet Interface (GBETH)
```

一行就給你三個資訊：GBETH 在**第 6.3 章**、屬於 SECTION 6（高速介面）、從 PDF 的**第 1632 頁**開始。你就可以打開那本 43.9 MB 的 PDF，直接跳到第 1632 頁，不用滑 4816 頁。

> 💡 **提示**：`_toc_full.txt` 的每一行都是「頁碼 Tab 標題」，縮排深度代表章節層級——0 層縮排是 `SECTION`、2 個空格是「章」（如 `6.3`）、4 個空格以上是更細的小節與暫存器。想只看「章」這一層、把細碎的暫存器行過濾掉，可以卡縮排：`grep -E "^[0-9]+	  [0-9]" ../../reference-docs/_toc_full.txt` 只留下 2 空格縮排的章標題。

grep 單元名很方便，但有一個坑一定要先講，否則你第一次查熱門單元就會被淹沒：

> ⚠️ **注意**：直接 grep 單元名，常常會把「章標題」淹沒在一堆「暫存器名」裡。
> **情境**：你要找 I²C 匯流排那一章，直覺打 `grep "I2C" ../../reference-docs/_toc_full.txt`。
> **症狀**：吐出 **40 行**，滿滿都是 `7.7.2.2.1 I2C Bus Control Register 1 (RIICm_ICCR1)`、`7.3.7 Simple I2C Mode`⋯⋯這種暫存器與模式的細項，你要的「那一章」被埋在中間。
> **原因**：手冊裡每一個帶「I2C」字樣的暫存器、每一個 Simple-I2C 模式小節都會被 grep 命中；而且 RSCI（序列埠）也有 Simple-I2C 模式，所以連 7.3 那邊的行都混進來。
> **預防**：認準**章標題的長相**——它是「2 空格縮排 + 章號 + 帶括號的單元英文縮寫」，例如你真正要的是這一行：`2968	  7.7 I2C Bus Interface (RIIC)`。想一次撈到位，就把縮排與括號寫進樣式，例如 `grep -E "^[0-9]+	  7\.7 " ../../reference-docs/_toc_full.txt`，直接得到 `2968	  7.7 I2C Bus Interface (RIIC)` 一行乾淨結果。RZ/V2H 的 I²C 控制器在手冊裡叫 **RIIC**，記住這個縮寫，查起來更準。

## 查哪章：常見單元 → 章節／頁 一覽

把上面的 grep 技巧對每個單元跑一遍，整理成下面這張對照表，之後你要查任何一個板上單元的暫存器細節，照表跳頁即可。表中「章節（頁）」欄的頁碼**逐字取自 `_toc_full.txt`**（就是你 grep 會看到的那個數字；✅ 2026-07-18 以 `grep` 逐行重執行核對，含 `1.8 Address Map`（p167），各表章名與頁碼全部一致，transcript：live/ch04b-docs-local.txt）；「暫存器 base」欄則來自本手冊硬體盤點的探測結果（出處 `07-hardware-unit-usage-guide.md`，即 d06），供你對照 `/proc/iomem` 或寫 bare-metal 時使用。

**SECTION 2 PROCESSORS（處理器與 AI 加速器，起始頁 212）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| CPU（CA55／CR8／CM33 三核心組態） | 2.2 CPU（p213） | — |
| DRP-AI3 NPU（AI-MAC + DRP0） | 2.3 AI Accelerator (DRP-AI)（p251） | AI-MAC `0x16800000`、DRP0 `0x17000000` |
| DRP（可重組處理器 DRP1，與 DRP-AI 內的 DRP0 不同單元） | 2.4 DRP（p259） | — |

**SECTION 3 MEMORY（記憶體，起始頁 264）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| 6 MB 共享 SRAM（SRAM0–11） | 3.2 SRAM（p265） | `0x08000000` 起，每塊 +`0x80000`（512 KB） |
| LPDDR4/4X 記憶體控制器 | 3.4 LPDDR4/4X Controller (DDR)（p277） | MEMC `0x1E000000`（DDR0）／`0x1E010000`（DDR1）、PHY `0x1A000000`（DDR0）／`0x1C000000`（DDR1） |
| TrustZone 位址空間控制（TZC-400） | 3.5 TrustZone Address Space Controller (TZC)（p352） | TZC400_XSPI `0x10470000`、TZC400_SRAMM `0x10460000` 等 |

**SECTION 4 SYSTEM（系統核心：接腳、時脈、中斷、DMA，起始頁 360）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| 接腳多工／pin-mux（PFC） | 4.2 Pin Function Controller (PFC)（p361） | PFC `0x10410000`，86 ports |
| 時脈產生（CPG／PLL／除頻／閘控／reset） | 4.4 Clock Pulse Generator (CPG)（p620） | CPG `0x10420000` |
| 電源管理（PMU／power domain） | 4.5 Power Management Unit (PMU)（p790） | — |
| 中斷控制（GIC-600／ICU／ELC） | 4.6 Interrupt Controller（p800） | GIC-600 基底 `0x14900000`、ICU `0x10400000`（ELC 整合於此） |
| DMA 控制器（DMAC） | 4.7 DMA Controller (DMAC)（p994） | DMAC0 `0x11400000`、DMAC1 `0x14830000`、DMAC2 `0x14840000`（DMAC1／2 相距 64 KB） |
| 除錯介面（CoreSight／JTAG／SWD） | 4.9 Debug Interface（p1119） | CoreSight（CST）`0x1F000000`（16 MB） |

> 📌 **位址校正註記**：上表 GIC-600 與 DMAC0 兩格，早期轉引來源在此有轉錄落差——GIC 曾被記成 `0x14800000`（那在 §1.8 Address Map 其實是 SRAM2(REG) 區）、DMAC0 曾被記成 `0x11C00000`（那其實是同 SECTION 7 表裡 ADC 的位址）。來源文件此處轉錄有誤，已對官方手冊 §1.8 Address Map（p167）、§4.6.2.2（p961）、Table 4.7-4（p1001）核正：GIC-600 基底 `0x14900000`，DMAC0／1／2＝`0x11400000`／`0x14830000`／`0x14840000`。

**SECTION 5 TIMER（計時器群，起始頁 1161）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| 即時時鐘（RTC） | 5.3 Realtime Clock (RTC)（p1166） | `0x11C00800`（唯讀鏡射位址 `0x11C00C00`） |
| 看門狗（WDT） | 5.4 Watchdog Timer (WDT)（p1218） | WDT0(CM33) `0x11C00400`、WDT1(CA55) `0x14400000`、WDT2/3(CR8) `0x13000000`/`0x13000400` |
| 通用計時器（GTM，與 OSTM 為同一 IP） | 5.5 General Timer (GTM)（p1236） | GTM0 `0x11800000` … GTM7 `0x12C03000` |
| 比較匹配計時器（CMTW） | 5.6 Compare Match Timer W (CMTW)（p1254） | CMTW0 `0x11C01800` … CMTW7 `0x13001800` |
| 通用計時器／PWM（GPT） | 5.7 General-Purpose Timer (GPT)（p1283） | GPT0 `0x13010000`、GPT1 `0x13020000` |
| PWM 輸出閘控（POEG） | 5.8 Port Output Enable for GPT (POEG)（p1504） | POEG0A `0x13001C00` … POEG1D `0x13003800` |

**SECTION 6 HIGH-SPEED INTERFACE（高速介面，起始頁 1515）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| SD 卡／eMMC 主控（SDHI） | 6.2 SD/MMC Host Interface (SD)（p1516） | SD0/1/2 `0x15C00000`/`0x15C10000`/`0x15C20000` |
| 千兆乙太網路（GBETH，含 PTP） | 6.3 Gigabit Ethernet Interface (GBETH)（p1632） | GBETH0/1 `0x15C30000`/`0x15C40000`，參考時脈 125 MHz |
| USB 3.2 Host | 6.4 USB3.2 Gen2x1 Interface (USB3)（p1636） | USB30／31 Host `0x15850000`／`0x15860000` |
| USB 2.0 Host/Function | 6.5 USB2.0 Interface（p1753） | USB20／21 Host `0x15800000`／`0x15810000`（USB20 Function `0x15820000`） |
| PCIe Gen3（RC/EP 可選） | 6.6 PCI Express 3.0 Interface (PCIe)（p2025） | outbound window `0x30000000`(PCIE0)/`0x38000000`(PCIE1) |

**SECTION 7 LOW-SPEED INTERFACE（低速介面，起始頁 2500）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| 外接 flash 介面（xSPI） | 7.2 Expanded Serial Peripheral Interface (xSPI)（p2501） | `0x11030000`，flash 視窗 256 MB@`0x20000000` |
| Renesas 序列通訊（RSCI，10 ch） | 7.3 Serial Communications Interface (RSCI)（p2585） | RSCI0 `0x12800C00` … RSCI9 `0x12803000`（板上 serial console＝RSCI6@`0x12802400`） |
| 帶 FIFO 序列埠（SCIF） | 7.4 Serial Communications Interface with FIFO (SCIF)（p2780） | SCIF0 `0x11C01400` |
| Renesas SPI（RSPI，3 ch） | 7.5 Serial Peripheral Interface (RSPI)（p2826） | RSPI0/1/2 `0x12800000`/`0x12800400`/`0x12800800` |
| CRC 運算單元 | 7.6 CRC Operation Unit (CRC)（p2954） | — |
| I²C 匯流排（RIIC，9 ch） | 7.7 I2C Bus Interface (RIIC)（p2968） | RIIC0 `0x14400400` … RIIC7 `0x14402000`、RIIC8 `0x11C01000`（CM33 常開域） |
| I3C 匯流排 | 7.8 I3C Bus Interface (I3C)（p3061） | I3C0 `0x12400000` |
| CAN-FD（6 ch） | 7.9 CAN-FD Interface (CANFD)（p3363） | CANFD base `0x12440000`，單一 base 涵蓋 6 channels |
| 12-bit ADC | 7.10 12-Bit A/D Converter (ADC)（p3670） | ADC `0x11C00000` |
| 溫度感測器（TSU） | 7.11 Temperature Sensor Unit (TSU)（p3739） | TSU0 `0x11000000`、TSU1 `0x14002000`（兩基底位址經真硬體手冊 §1.8 位址圖 `<TSU0_base>`／`<TSU1_base>` 核實；trim 值來自 OTP〔晶片內一次性燒錄記憶體〕，0.0625 °C/code——trim 值屬二手轉錄，見 g7 待覆核） |

> 💡 **提示**：CAN-FD 若要逐一暫存器寫驅動程式，除了硬體手冊 7.9 章，Renesas 另有一份 FSP 版 CAN-FD 手冊 `r01us0478`，收的暫存器說明更完整（不在本資料夾，需另向 Renesas 取得）。接 DroneCAN/UAVCAN 控制器（以飛行控制器為例）時，通常先看 7.9 認位址、再翻 `r01us0478` 對逐個暫存器（出處 d06，`07-hardware-unit-usage-guide.md`）。

**SECTION 9 IMAGE（影像／顯示／編解碼／GPU，起始頁 3956）**

| 想查的單元／主題 | 硬體手冊章節（頁） | 暫存器 base（d06 探測） |
|---|---|---|
| MIPI 相機接收（CRU／CSI-2） | 9.2 Camera Data Receiver Unit (CRU)（p3957） | CRU0–3 `0x16000000`/`10000`/`20000`/`30000` |
| 影像縮放（ISU） | 9.3 Image Scaling Unit (ISU)（p4247） | `0x16450000` |
| LCD 控制器（LCDC／DU） | 9.4 LCD Controller (LCDC)（p4372） | DU `0x16460000`、FCPVD `0x16470000` |
| MIPI DSI 顯示輸出 | 9.5 MIPI DSI Interface (DSI)（p4548） | DSI_LINK `0x16430000`、DSI_DPHY `0x16440000` |
| 影片硬體編解碼（VCD，H.264/H.265） | 9.6 H.265/H.264 Multi Codec (VCD)（p4683） | VCD 子區塊位於 `0x16400000` 影像區域（VLC／FCPC／CE＝`0x16400000`／`0x16410000`／`0x16420000`） |
| GPU（Mali-G31／GE3D） | 9.7 3D Graphics Engine (GE3D)（p4685） | `0x14850000`（64 KB） |

> ⚠️ **注意**：VCD 那一格特別提醒——若你在別處看到「VCD 在 hw_manual 15.x」之類的章號，**以這張表的 9.6 為準**。
> **情境**：你在別處（舊筆記、討論串或某份摘要）看到「VCD 在 hw_manual 15.x」或「1.8/15.x」這種標註，照著去手冊找 VCD 編解碼器。
> **症狀**：翻到「15.x」找不到章——整本手冊只有 10 個 SECTION，根本沒有第 15 章。
> **原因**：那個「1.8」指的是 `1.8 Address Map`（VCD 的 `0x16400000` 位址區在那裡列出），是**位址區編號、不是章號**；而「15.x」這種編號對不上手冊的最終目錄——手冊裡 VCD 實際落在 9.6。
> **預防**：查章號一律回到 `_toc_full.txt` 這個權威目錄——它 grep 出來的 `4683	  9.6 H.265/H.264 Multi Codec (VCD)` 才是實際落點；位址區域另可在 `1.8 Address Map`（p167）交叉對照。

> 📌 **位址校正註記**：早期轉引把 VCD 位址記成 `0x14800000`，但 §1.8 Address Map 顯示 `0x14800000` 是 SRAM2(REG) 區；VCD 三個子區塊（VLC／FCPC／CE）實為 `0x16400000`／`0x16410000`／`0x16420000`，與 g2 章的 DT 節點 `vcp4@16400000` 一致。來源文件此處轉錄有誤，已對官方手冊 §1.8 核正。

## 只查規格數字：datasheet 的 Section 1 就夠

如果你要查的不是「暫存器細節」而是「規格數字」（例如某介面幾通道、最高幾 MHz、料號差異），那就不必動硬體手冊，直接翻 datasheet（`r01ds0429`——記得本資料夾這份是文字轉檔，`grep` 反而特別順手）的 Section 1 規格總表，它的子表分得很細（出處 `datasheet.md`，即 d18 讀取範圍）：

- Table 1.3-1 CPU、1.3-2 Accelerator Engines、1.3-3 SRAM／外部記憶體、1.3-4 Boot、1.3-5 System/DMAC/中斷/時脈、1.3-6 通訊/儲存/網路、1.3-7 計時器、1.3-8 Audio、1.3-9 ADC、1.3-10 內部感測器、1.3-11 Security、1.3-12 GPIO、1.3-13 電源、1.3-14 溫度、1.3-15 品質、1.3-16 封裝。
- 另外 **Table 1.4-1／1.4-2 List of Units** 是一張「單元縮寫全稱對照表」——查 `CRU` / `DRP` / `DRP-AI` / `GPV` / `TZC` 這些縮寫是什麼意思時看它。特別留意：`DRP` 指的是單元編號 DRP1，`DRP-AI` 則是 DRP0＋AI-MAC 組成，兩者共用「DRP」字首但**是不同單元**，別搞混（出處 `datasheet.md`，191–194、785–874）。

> ⚠️ **注意**：datasheet 的方塊圖（Figure 1.4-1 Block Diagram）不要照文字轉檔逐字讀。
> **情境**：你想從 datasheet 的整晶片方塊圖看清楚各匯流排怎麼連，直接讀本資料夾這份文字轉檔裡對應的段落。
> **症狀**：文字順序錯亂、同一個方塊的標籤跟數字被拆到不相鄰的行，甚至出現顛倒的亂碼字元（如 `suB UPCM`）。
> **原因**：原圖是多欄並排的方塊圖，PDF→文字的轉檔工具把版面拆散重排了——本資料夾的 datasheet 正是這種文字轉檔（見前面格式注意）。
> **預防**：要看方塊圖，去 Renesas 官網取原版 PDF 看**第 16 頁**的圖，別採信轉檔後的文字順序（出處 d18，datasheet.md 轉檔提醒）。

## 那份 AWO／韌體部署文件（進階，先知道在哪）

九份文件裡，`r01an7723`（AWO Example Program Startup Guide，R01AN7723EJ0400 Rev.4.00）是唯一被完整讀過的 application note。它教的是**進階韌體部署**——怎麼在 CM33 上跑 AWO（喚醒／睡眠控制）範例、怎麼從 Yocto build 出 CA55 與 CM33 的產物。這**不是本章的動手範圍**（本章講的是硬體資源地圖），所以這裡只留幾個「將來你真的要碰 R8/M33 韌體時」用得上的指路重點與已知的坑（出處 d18《一》，startup-guide.md）：

- **文件結構**：共 6 章——1 Specifications／2 Proven Environment／3 RZ/V2H Setup／4 RZ/V2N Setup／5 Invocation（Suspend-to-RAM 不支援）／6 Invocation（S2R 支援）。注意這份文件同時涵蓋 RZ/V2H 與 RZ/V2N 兩塊板，看的時候要認清你在哪一章。（「AWO」的英文全稱原文自始至終只用縮寫、沒有展開，所以本手冊也不替它杜撰全名。）
- **驗證過的工具版本**（照抄原文，供你對齊環境）：e2 studio **2025-12**；RZ/V2H AI SDK **v6.00**、RZ/V2N AI SDK **v6.30**；RZ Flexible Software Package（FSP）**v4.0.0**（出處 startup-guide.md:89–94）。

> ⚠️ **注意**：用 J-Link 除錯 RZ/V2H 的 AWO 範例，J-Link DLL 版本卡得很死。
> **情境**：你沿用手邊現成的 J-Link 軟體去除錯 RZ/V2H 的 CM33 冷開機 AWO 範例。
> **症狀**：可能無法正確運作。
> **原因**：原文明載「RZ/V2H AWO example ⋯ requires the use of J-Link DLL version 7.96e」——這個冷開機環境綁定特定版本。
> **預防**：把 J-Link 軟體更新到 **7.96e** 這個特定版本再除錯；RZ/V2N 的 AWO 範例則沒有這個限制，可用 J-Link DLL **8.60**（出處 startup-guide.md:71–74）。

> ⚠️ **注意**：這份文件在 `layer.conf` 裡的 bitbake 語法，RZ/V2H 段和 RZ/V2N 段**寫法不一樣**，照抄要看清楚是哪一段。
> **情境**：你參考這份文件改 Yocto 的 `layer.conf` 啟用 CM33 cold boot，順手把 RZ/V2H 段的寫法套到 RZ/V2N（或反過來）。
> **症狀**：語法不被 bitbake 接受，或設定沒生效。
> **原因**：RZ/V2H 章節用**底線**語法 `MACHINE_FEATURES_append = " RZV2H_CM33_BOOT"`；RZ/V2N 章節用**冒號**語法 `MACHINE_FEATURES:append = " RZV2N_CM33_BOOT"`——這是原文本身在兩個平行小節的用字差異（可能反映不同 bitbake 版本的語法過渡期），不是筆誤。
> **預防**：照抄時認清你在改哪一塊板的段落，用該段落原本的語法；另外原文明確警告，取消 `*_CM33_BOOT` 那行註解時，**不要**連帶取消它下面 `SRAM_REGION_ACCESS`／`CM33_FIRMWARE_LOAD`／`CA55_CPU_CLOCKUP` 三行的註解（原文只寫「Be sure NOT to uncomment」、沒說明技術原因，所以照做即可、別自行推想）（出處 startup-guide.md:113–122、244–255）。

還有一個細節會呼應到 01 檔「板上 A55 實跑 1.7 GHz、datasheet 額定 1.8 GHz」那個差異：這份 AWO 文件提到，若要把 CA55 操作頻率設成 **1.8 GHz**，需要在 `configurator.xml` 裡另外啟用「Clock up for CA55」選項（出處 startup-guide.md:393–395）。也就是說 1.8 GHz 不是預設值，要主動開。

## 官方線上資源

除了倉庫這疊文件，官方還有線上文件與程式碼倉庫，需要最新版或原始碼時往這裡找（出處 `04-hardware-quickref.md`，即 d03，224–226）：

- 官方 RDK 文件站：`https://renesas-rdk.github.io/rzv2h_rdk_documentation/latest/`
- RZ/V2H Linux BSP（原始碼）：`https://github.com/renesas-rz/rzv_linux-cip`
- DRP-AI TVM（把模型編譯到 DRP-AI3 的工具鏈）：`https://github.com/renesas-rz/rzv_drp-ai_tvm`

## 動手驗證：確認你會用這疊文件當地圖

做完這幾件事，代表你已經會用這疊文件當地圖了：

1. ✅ **列出九份文件**（2026-07-18 於本資料夾實跑，輸出見前面〈資料夾裡實際有什麼〉）。執行 `ls -1 ../../reference-docs/*.pdf`，數一數是不是 **9 個 `.pdf`**，並對每一個檔名開頭的前綴（`R01DS`/`R01UH`/`R01US`/`R01AN`/`R01QS`/`R01WP`/`R20AN`/`REN_…R16UH`）說得出它是哪一類文件。
2. 📼 **從問題反查到文件。** 給自己出四題，說得出該翻哪一份：
   - 「這顆晶片有幾條 CAN？」→ datasheet（`r01ds0429`）Section 1 規格總表。
   - 「GPT 計時器的暫存器 offset 是多少？」→ 硬體手冊（`r01uh1032`），用 `_toc_full.txt` 找章。
   - 「DRP-AI3 的 8 TOPS 是怎麼算出來的？」→ 白皮書（`r01wp0022`）。
   - 「40-pin header 的某支腳接到哪裡？」→ WS125 RDK **載板**手冊（`REN_WS125…`，記得先解壓）——這是板卡層問題，晶片手冊沒有。
3. ✅ **用 `_toc_full.txt` 跳到正確章頁**（2026-07-18 重執行，transcript：live/ch04b-docs-local.txt）。隨便挑一個板上單元（例如你正在接的 CAN），執行 `grep "CAN-FD" ../../reference-docs/_toc_full.txt`——實測會吐出 **9 行**（1 行章標題＋8 行帶「CAN-FD」字樣的暫存器小節），認**第一行、也是唯一 2 空格縮排的章行**：`3363	  7.9 CAN-FD Interface (CANFD)`——也就是 SECTION 7、第 7.9 章、PDF 第 3363 頁。這個頁碼就是你要在 43.9 MB 手冊裡跳過去的目標。
4. ✅ **避開 grep 淹沒坑**（2026-07-18 重執行：`grep -c "I2C"` 實測正是 40 行，transcript：live/ch04b-docs-local.txt）。對一個「熱門」單元（如 `I2C`）直接 grep，看它吐出幾十行暫存器；再用「認章標題長相」的方式（2 空格縮排＋帶括號單元名，如 `2968	  7.7 I2C Bus Interface (RIIC)`）把你要的那一章挑出來。做到這步，你就不會再被暫存器名的噪音干擾。
5. 📼 **交叉對帳（選做）。** 挑一個你在本章前面看過 base 位址的單元（例如 GBETH `0x15C30000`），到手冊對應章（6.3，p1632）確認暫存器地圖起點對得上——當「盤點筆記的 base」與「手冊的章」對得起來，你就真的把這疊文件走通了。
