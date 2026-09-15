# 08 · 除錯與安全

這一群裝的是整片 RZ/V2H 上「你平常摸不到、但關鍵時刻救命」的三套硬體：**晶片內建的除錯／追蹤子系統（CoreSight）**、**執行環境隔離與記憶體存取管控（TrustZone ＋ 9 顆 TZC-400）**、以及**安全模組（Trusted Secure IP 加密引擎 ＋ OTP／Device-ID／JTAG-disable）**。

這一群也最容易讓人誤以為板子「缺功能」。原因是：它們大多不是啟用中、綁好驅動程式、`ls /dev` 就看得到的東西——多數落在資源地圖那六格狀態裡的「**存在·Linux 未曝露**」與「**未搭載**」。把每一個的真實狀態講清楚，你才不會浪費時間去 Linux 裡翻找一個根本不會出現在 Linux 裡的節點。這一群的通則只有一句：**這三套幾乎都不是靠 Linux 使用者空間指令去用的**——CoreSight 靠外部探針、TrustZone 靠開機韌體、加密引擎在本板 Linux 上摸不到（本料號 Security＝N/A，真硬體手冊 Table 1.1-1〔p78〕一手核實、矽晶未搭載此模組）——所以「找不到」對這一群而言常常正是**正確狀態**，不是故障。

（另外先劃清一條界線：**日常除錯的第一工具不在這一群**——是 CN8 的 UART 序列主控台（FT234XD → SCIF0，115200 8N1），開機 log、kernel 當機現場、救援登入都靠它，完整教學見第 01 章 1.4 節。這一群的 CoreSight／JTAG 是更深一層、需要外部探針的晶片級除錯，多數讀者用不到。）

> **本群組的板上硬體是這顆料號**：R9A09G057H44GBG（板卡手冊 Page 11(板卡手冊) 元件表 U1，H44＝RZ/V2HP 版）。先記住這顆料號的一個關鍵取捨——**板上量到 Linux 沒有任何硬體加密介面**（無 `/dev/tee`、無 Renesas TRNG 的 hwrng 節點、CA55 指令集也無 ARMv8 Crypto Extension）。至於這顆料號到底有沒有搭載 Trusted Secure IP 硬體，**真硬體手冊 `r01uh1032` §1.1.2 Product Lineup Table 1.1-1〔p78〕已給答案**：這張 SKU 陣容表——正是 00／02 檔用來坐實本料號 ISP＝Available〔Mali-C55〕的同一張真 PDF 表——把 R9A09G057H44GBG 的 Security 欄逐字標為 **N/A**，可 pypdf 逐字核實，屬一手來源，故本料號**未搭載**。（另一份 datasheet `r01ds0429` 的同類陣容表雖為劣化轉檔，但真硬體手冊既已提供乾淨 SKU 依據，就不再是障礙。）這件取捨貫穿整個〈單元三〉，也是本群組最常被誤解的一點。

## 本群組單元清單

| 單元 | 一句話 | 板上狀態 |
|---|---|---|
| [CoreSight（晶片內建除錯與追蹤子系統）](#單元一coresight晶片內建除錯與追蹤子系統) | Arm 的除錯／追蹤硬體：不透過作業系統，直接從晶片外部看 CPU 執行狀態、設硬體斷點、記錄程式流程 | 存在·**Linux 無 debug/trace 節點**；只能靠外部探針經 JTAG／SWD |
| [TrustZone（含 9× TZC-400）](#單元二trustzone含-9-tzc-400-位址空間控制) | CA55 的安全／一般世界隔離 ＋ 9 顆匯流排存取管控器，把記憶體與周邊按「來自哪個世界」放行或擋下 | 硬體存在·**無安全 OS runtime**（無 `/dev/tee`、無 OP-TEE）；TZC 區域由開機 TF-A 設定 |
| [Security IP（加密引擎＋OTP／Device-ID／JTAG-disable）](#單元三security-ip加密引擎otpdevice-idjtag-disable) | 一顆選配的硬體安全模組（加解密／真亂數／金鑰不落地）＋ 矽片固定存在的一次性燒錄記憶體 | **混合**：板上 Linux 無加密引擎／TRNG／`/dev/tee` 介面、CA55 亦無 crypto 延伸（板上實證）；本料號 Security＝N/A、未搭載 Trusted Secure IP（真硬體手冊 Table 1.1-1〔p78〕一手核實，同坐實 ISP 的那張表）；OTP／Device-ID／JTAG-disable 矽片固定存在但無 Linux runtime |

**這一群共用的觀念先講一次**：這三套硬體都跟「安全邊界」與「開機／韌體層」有關，所以它們的曝露方式跟前面幾群（相機、音訊、序列埠那些「開機就掛好驅動程式」的周邊）根本不同。它們要嘛被開機韌體（TF-A／u-boot）握在手裡、要嘛靠 Linux 之外的外部工具（JTAG 探針）才碰得到、要嘛在這顆料號上壓根沒搭載。往下讀每一個單元時，「Linux 下怎麼看到它」那一小節多半會告訴你「你看不到，這是預期的」——重點是它接著會告訴你**該去哪裡看**。

- [單元一：CoreSight（晶片內建除錯與追蹤子系統）](#單元一coresight晶片內建除錯與追蹤子系統)
- [單元二：TrustZone（含 9× TZC-400 位址空間控制）](#單元二trustzone含-9-tzc-400-位址空間控制)
- [單元三：Security IP（加密引擎＋OTP／Device-ID／JTAG-disable）](#單元三security-ip加密引擎otpdevice-idjtag-disable)
- [本群組的證據邊界與未查證項](#本群組的證據邊界與未查證項)

---

## 單元一：CoreSight（晶片內建除錯與追蹤子系統）

### 這是什麼（機制）

這顆 SoC 內建一整套 Arm **CoreSight** 除錯子系統。它的作用一句話講完：**不透過作業系統，直接從晶片外部看 CPU 內部的執行狀態、設中斷點（breakpoint）、抓程式流程**。手冊原文寫得很直白：「This LSI has a debug interface for boundary scan function and debug support for CA55, CM33, and CR8.」——也就是這套除錯介面同時服務三種核心：四顆應用核 CA55、系統管理核 CM33、兩顆即時核 CR8（手冊 4.9.1 Overview）。

這套除錯介面在同一組實體接腳上多工共用**兩種通訊協定**：**JTAG**（IEEE 1149.1 邊界掃描標準）與 **SWD**（Serial Wire Debug，序列線除錯）。共用的接腳是 `TCK_SWCLK`／`TMS_SWDIO`／`TDI`／`TDO`／`TRSTN`（手冊 4.9.1.1 Features；4.9.1.3 Table 4.9-1）。這兩種協定的差別是接腳數與速度取捨（JTAG 腳多、SWD 只需兩線），但對讀者而言重點是：**它們走的都是這條晶片外部的除錯通道，跟 Linux 無關**。除錯通道的核心是 **DAP**（Debug Access Port，除錯存取埠），它可以「不經過 CPU、直接切換 Access Port 來控制內部 IP」（手冊 4.9.1.1：「Direct control of IP without going through CPU by switching Access Port (AP)」）——這正是外部探針能在系統跑到一半、甚至 CPU 卡死時仍能讀寫晶片內部的原因。

第二個能力是 **Trace（程式流程追蹤）**：它不是「停下來看一眼」，而是「連續側錄每顆核心跑過的程式計數器歷史」。CA55、CM33、CR8 三種核心各自有一個 **ETM**（Embedded Trace Macrocell，內嵌追蹤產生器）產生追蹤資料；這些資料經 **Trace funnel**（多路合併器）匯集，暫存進晶片內建的 **ETF**（Embedded Trace FIFO，追蹤緩衝區），或經 **ETR**（Embedded Trace Router）直接寫到系統匯流排／記憶體（手冊 4.9.1.2，Fig 4.9-1、Fig 4.9-2）。追蹤資料路徑大致長這樣：

```text
  CA55 ×4 ─┐
  CM33    ─┤ 各核心的 ETM ──► Trace Funnel ──► ETF（晶片內 FIFO）──► 外部探針讀出
  CR8 ×2  ─┘  （追蹤產生器）    （多路合併）      ETF0 32KB / ETF1 16KB        或
                                                 ETF2 8KB  / ETF3 4KB     ETR ──► 系統匯流排／記憶體
  GIC-600 ─── 只接 Cross Trigger（中斷控制器，無 trace 輸出）
```

第三個能力是**互相觸發**：**CTM**（Cross Trigger Matrix，交叉觸發矩陣）讓 CPU 核心與除錯元件之間可以互相連動事件——例如一顆核心命中中斷點時，連動另一顆核心一起暫停；它還能連動 SYC（系統計數器）、WDT（看門狗計時器）、GTM（通用計時器）（手冊 4.9.1.1：「Interlocking operation of CA55, CM33, CR8, system counter (SYC), watchdog timer (WDT), general timer (GTM), and debug component by cross trigger」）。另外還有一個獨立用途的 **Boundary Scan（邊界掃描）**：它測的不是 CPU，而是「這顆晶片與電路板上其他 IC 之間的接線有沒有接對」——是生產測試用的，透過 `BSCANP` 接腳切換進入（手冊 4.9.1.3，Table 4.9-4、4.9-5）。

最後一個關鍵機制，會直接影響你能不能用除錯：**除錯功能不是永遠開著的**。有一支開機接腳 **MD_BOOT3**，它的值必須在「開機釋放 power-on reset（PRST#）之前」就決定好——`0`＝一般操作（除錯功能模組整個處於 reset 狀態、無法在開機後使用），`1`＝除錯操作（除錯功能可開機使用）。手冊明講這個值「must be fixed before releasing the power-on reset (PRST#)」，**開機後無法動態切換**（手冊 4.9.1.3 Table 4.9-2、4.9-3）。換句話說，能不能用 JTAG/SWD，一部分在硬體開機一瞬間就已註定。

### Linux 下怎麼看到它

**簡短版：你在 Linux 裡看不到它，而且這是正常的。** doc07 §15 的板上狀態欄講得很明確：「PRESENT in silicon, but NO Linux debug/trace nodes on this board」——矽片裡有這套子系統，但這片板子的 Linux 端沒有掛它，`no /sys/bus/coresight devices, no /dev/cpu/*/etm`。這正是資源地圖那六格狀態裡「**存在·Linux 未曝露**」的典型代表。

要驗證「本板沒有 self-hosted CoreSight」，跑這一行（回傳「找不到」才是本板的預期狀態）：

```bash
ls /sys/bus/coresight/devices 2>/dev/null || echo 'no coresight nodes'
```

（出處：doc07 §15。）

那要怎麼「打開」它？如果你想在 Linux 側啟用 self-hosted trace（讓 Linux 自己驅動 CoreSight 側錄），需要**核心編譯選項 `CONFIG_CORESIGHT` ＋ device tree 裡對應的 graph 節點**，而本板出貨的 device tree 沒有設定這些（doc07 §15）。這是額外的系統設定工作，不是開機就有。

不過真正實務上用 CoreSight 的路徑，其實**完全繞過 Linux**：用**外部除錯探針**（JTAG 或 SWD）搭配業界工具，透過除錯用的 APB 匯流排直接存取晶片。典型做法是用 OpenOCD 這類開源工具連進去：

```bash
openocd -f interface/jlink.cfg -c 'transport select swd' -f target/renesas_rzv2h.cfg
# 另一個終端機接進 OpenOCD 的指令埠
telnet localhost 4444
# 進去後可下 halt（暫停 CPU）／reg（讀暫存器）／mdw <位址>（讀記憶體字）等
```

（出處：doc07 §15 使用範例。`<位址>` 是你要讀的記憶體位址佔位。）除了 OpenOCD，商用工具還有 Arm Development Studio、Lauterbach TRACE32。這些工具跑在你的 PC 上、透過探針硬體連到板子的除錯接腳，**跟板上的 Linux 核心是不是活著完全無關**——這也是外部探針除錯最大的價值：Linux 掛了、還沒開機、或你要看的根本是 Linux 管不到的 bare-metal 核心（CR8／CM33 韌體）時，它照樣能用。

### 關鍵能力與限制

- **追蹤緩衝容量共 60 KB，分四顆**：ETF0 32 KB（總入口）、ETF1 16 KB、ETF2 8 KB、ETF3 4 KB（手冊 4.9.1.2 Fig 4.9-2 圖上標籤，逐字讀取）。這 60 KB 就是「不外接 ETR、只靠晶片內 FIFO」時能側錄的追蹤深度上限。（附帶一提：本手冊 4.2〈運算單元〉在講 CM33 時提到「60 KB 的 CoreSight ETF 追蹤緩衝」，指的就是這四顆 ETF 的總和——這是整個 trace 子系統共用的緩衝，不是 CM33 專屬。）
- **Trace 對象是三種 CPU、不含中斷控制器**：CA55（4 核）、CM33、CR8（2 核）各有獨立 ETM trace 輸出；GIC-600（中斷控制器）只接 Cross Trigger，**沒有** trace 輸出（手冊 4.9.1.2 Fig 4.9-1）。
- **除錯位址空間 4 MB，base `0x1F000000`**（手冊 4.9 Debug Interface；本手冊 4.4 文件查閱表亦列此位址）。內部各模組的位址（ROM table `0x1F000000`、Timestamp gen `0x1F010000`、ETF0 `0x1F020000`、ETF1 `0x1F030000`、ETF2 `0x1F040000`、ETF3 `0x1F050000`、ETR `0x1F060000`、Trace Funnel0–3 `0x1F070000`–`0x1F0A0000`、CTI0 `0x1F0B0000`）——**這段位址明細轉錄自 doc07 §15**，doc07 標明其出處為手冊 4.9.2（暫存器明細頁，落在本手冊取材的 p1119–1123 功能概說範圍之外）；**本群組筆記未直接查證原始頁面，僅轉引 doc07 的既有整理**，引用時請留意這個邊界。
- **暫存器層級細節要另查 Arm 文件**：ETM／ETR／ETF 的暫存器操作明細要看 Arm 官方「CoreSight Trace Memory Controller Technical Reference Manual」，本 SoC 手冊只講整合方式（手冊 4.9.1.2 段末）。**這份 Arm 文件不在本手冊的可用來源清單內，尚未查證。**
- **JTAG 可被 OTP 永久關閉**：可透過 OTP 保險絲（write-once，燒錄後不可逆）永久關掉 JTAG——這條「不可逆」的限制很重要，細節在〈單元三：Security IP〉的 JTAG-disable。

### 什麼情況下你會用到它

**判斷準則（先判斷你的需求屬不屬於「作業系統管不到」的範疇）**：只要你的需求是「**不倚賴作業系統**、在系統運作到一半時檢視 CPU 暫存器／記憶體／中斷狀態、設硬體斷點、逐步執行、或連續記錄程式計數器歷史」，那就屬於 bring-up（硬體從通電到基本功能可用、能跑起來的過程）除錯或即時系統驗證的範疇。這類需求都得走 JTAG/SWD 外部探針：本板 Linux 端沒有掛 CoreSight，沒有任何 Linux 指令碰得到它。這一條**不分應用領域**：以移動載具的飛控韌體開發為例、或以工業機台的即時控制韌體開發為例，只要你在開發跑在 CR8／CM33 上的 bare-metal 韌體，遇到「核心卡死、Linux 也還沒起來、printf 都印不出來」的當下，外部探針常常是你唯一能問「它現在停在哪一行」的工具。

**反過來，什麼情況下你不需要它**：如果你只是想觀察一般 Linux 應用程式的行為（以替使用者空間程式做效能剖析為例），走 `ftrace`／`perf`／`gdbserver` 這類作業系統層工具就夠了，完全不必碰 CoreSight。CoreSight 的主場是「Linux 尚未開機、或要跨過 Linux 直接看 bare-metal 核心狀態」——目的落在使用者空間的事，別拿它殺雞用牛刀。

**如果你要的是 trace（連續側錄）而非斷點單步**：前提是要先在 Linux 核心設定裡打開 `CONFIG_CORESIGHT`、並補上對應的 device tree graph 節點——這是額外的系統設定工作，本板出廠設定並未內建。判斷準則是：你需要的是「事後回放整段執行軌跡」（trace）還是「當下停下來檢查」（斷點）？前者要付出上述設定成本，後者用外部探針即可。

> **尾註（出處）**：官方硬體手冊 `r01uh1032` §4.9 Debug Interface（p1119–1123，功能概說；暫存器明細自 4.9.2 起，未讀）。除錯位址明細轉引自 doc07 §15（未直接查證原始頁）。外部工具：OpenOCD、Arm Development Studio、Lauterbach TRACE32。暫存器操作細節另見 Arm「CoreSight Trace Memory Controller Technical Reference Manual」（不在本手冊可用來源清單，未查證）。

---

## 單元二：TrustZone（含 9× TZC-400 位址空間控制）

### 這是什麼（機制）

**TrustZone** 是 Arm CPU 的一種「執行環境隔離」機制：它把 CA55 的執行狀態分成 **Secure（安全）** 與 **Non-secure（一般）** 兩個世界。但光有 CPU 模式區隔還不夠——如果一般世界的程式還是能直接讀寫安全世界的記憶體，隔離就形同虛設。所以記憶體與周邊也要能依「這筆存取來自哪個世界」放行或擋下，這件事由 **TZC-400**（CoreLink TrustZone Address Space Controller，Arm 授權的位址空間控制器 IP）負責。手冊原文：「This LSI has nine address space controllers (TZCs) to realize memory access in a safe area. It performs security checks on transactions to memory or peripherals. Transactions must meet security requirements to access memory or peripherals.」（手冊 3.5.1 Overview）

這顆 SoC 一共裝了 **9 個** TZC-400 實例，各自掛在不同匯流排前面當「關卡」，過濾通往各塊記憶體／周邊的存取（手冊 3.5.1.1 Features、3.5.1.2 Fig 3.5-1 方塊圖、3.5.1.3 文字列點，逐一核對）：

| TZC-400 實例 | 守的對象 | filter unit 數與分工 |
|---|---|---|
| `TZC400_XSPI` | xSPI（外接 flash 介面） | 1 |
| `TZC400_SRAMM` | 內部 SRAM0、SRAM1 | 2（各 1） |
| `TZC400_SRAMA` | 內部 SRAM2 | 1 |
| `TZC400_AXI_RCPU` | RCPU Bus（即時處理相關的內部匯流排） | 1 |
| `TZC400_DDR00` | DDR0 記憶體 | 3（分別放行 Video0／Video1／DRP Bus） |
| `TZC400_DDR01` | DDR0 記憶體 | 2（放行 COM Bus／ACPU Bus） |
| `TZC400_DDR10` | DDR1 記憶體 | 3（同 DDR00：Video0／Video1／DRP） |
| `TZC400_DDR11` | DDR1 記憶體 | 2（同 DDR01：COM／ACPU） |
| `TZC400_PCIE` | PCIe | 2（對應 PCIE0／PCIE1 兩個 controller） |

每一顆 TZC 可以在自己管的位址空間裡切出「**最多 8 個安全區域 ＋ 1 個涵蓋剩餘位址的預設區域（default base region）**」，每個區域可分別用軟體透過 APB 介面設定存取權限（手冊 3.5.1.1：「The ability to define up to eight address regions in the area map.」「A default base region to cover all remaining portions of the address map.」「Software programmable security access permissions for each address region through an APB interface.」）。實際把關的是 **filter unit**：只有在「一筆 ACE-Lite 匯流排交易的安全狀態與身分，符合它所存取的記憶體區域的安全設定」時才放行資料傳輸；一顆 TZC 底下的所有 filter unit **共用同一組 region 設定暫存器**，以確保一致性（手冊 3.5.1.1、3.5.1.3：「All filter units operate from one set of shared region configuration registers. This ensures consistency across all filter units.」）。

它還有幾個值得記住的性質：支援對 Non-secure 存取做**身分過濾（identity-based filtering）**，也支援最多 **256 個 outstanding transaction**（同時在途的交易）（手冊 3.5.1.1 末兩點）。存取違規時可設定發中斷通知，每顆 TZC 各有一條中斷線 `TZCINT_*`（\* ＝ XSP、MSRM、ASRM、ACRCB、DDR00、DDR01、DDR10、DDR11、PCI）（手冊 3.5.1.1 Features ＋ Fig 3.5-1 各圖上的 Interrupt 輸出）。最後一點很關鍵：**TZC 本身沒有對外接腳**，它純粹是內部匯流排層級的存取管控機制（手冊 3.5.1.3 External Pins：「In the TZC, there are no external pins.」）——你在板子上找不到任何一支「TZC 腳位」，它完全活在晶片內部。

### Linux 下怎麼看到它

**跟 CoreSight 一樣，你在 Linux 裡幾乎看不到它，而這也是正常的。** doc07 §16 的板上狀態：「Hardware PRESENT (CA55 has TrustZone; TZC-400s instantiated), BUT NO secure-OS runtime on this board」——硬體都在（CA55 有 TrustZone、9 顆 TZC-400 也都實例化了），但**這片板子沒有跑在安全世界裡的作業系統**，所以沒有 OP-TEE、沒有 `/dev/tee`。

關鍵在於「**誰擁有 TZC 的設定權**」：TZC-400 的區域設定是「由開機時的 **TF-A**（BL31，跑在 EL3 最高權限層的開機韌體）一次設好的，不是 Linux 能碰的」（doc07 §16：「TZC-400 region programming is owned by TF-A (BL31/EL3) at boot, not by Linux. No userspace path.」）。也因為安全世界的 TEE 根本不存在，`libteec`／`tee-supplicant` 這一整套 TEE client framework 沒有任何對象可以溝通（doc07 §16）。

驗證指令（回傳「找不到」才是本板正常狀態，**不是異常**）：

```bash
ls /dev/tee* 2>/dev/null || echo 'no TEE device (no OP-TEE)'
dmesg | grep -iE 'optee|tee_'
```

（出處：doc07 §16。✅ 2026-07-17 板上重執行（transcript：live/ch04-cpu-periph.txt）：第一行落到 `no TEE device (no OP-TEE)`，TEE 確實不存在——出處 `07-hardware-unit-usage-guide.md:274`。）

> ⚠️ **注意（找不到 `/dev/tee` 是預期狀態，不是壞掉）**：
> - **情境**：你檢查 TEE 裝置，預期會看到 OP-TEE。
> - **症狀**：`ls /dev/tee*` 沒有結果，`dmesg` 裡也沒有任何 TEE 驅動程式綁定。
> - **原因**：本板沒有安全 OS runtime——TrustZone 硬體存在，但沒有安全韌體被載入到安全世界。
> - **預防／處理**：這個「找不到」是本板的**預期狀態**；TZC-400 的區域規劃由開機時的 TF-A（BL31／EL3）負責，Linux 沒有使用者空間路徑可存取它（出處 doc07 §268、§273）。要驗證安全韌體「確實存在」，只能間接看——見下一段的 MTD 分割區。

想確認安全開機韌體本身存不存在，只能從 xSPI flash 的 MTD 分割區間接看：

```bash
cat /proc/mtd
```

會看到 `mtd0='bl2'`、`mtd1='fip'`——這兩個分割區存放的正是 TF-A／secure boot 相關映像檔（`bl2` 是第二階段開機載入器、`fip` 是 Firmware Image Package，內含 BL31 等）（doc07 §16；本手冊 4.1（00 檔）與 04 記憶體與儲存群已逐字驗證這四個 MTD 分割區：`mtd0` bl2／`mtd1` fip／`mtd2` env／`mtd3` test-area，`mtd0`／`mtd1` 是開機韌體，只讀不寫）。

TZC 的暫存器位址（供未來若需 bare-metal 存取參考）：filter 透過 APB 以 `REGION_SETUP_LOW/HIGH_<n>`、`REGION_ATTRIBUTES_<n>`、`REGION_ID_ACCESS_<n>` 設定；部分實例位址如 `TZC400_XSPI` `0x10470000`、`TZC400_SRAMM` `0x10460000`——**這段轉錄自 doc07 §16 標明的手冊行號（15770 行），落在本手冊取材的 p352–356 概說範圍之外的暫存器明細頁，本群組筆記未直接查證，僅轉引**（本手冊 4.4 文件查閱表亦列出這兩個位址）。

### 關鍵能力與限制

- **9 個 TZC-400 實例**，對應 filter unit 數逐字列於手冊 3.5.1.3：xSPI＝1、SRAM0+SRAM1（`TZC400_SRAMM`）＝2、SRAM2（`TZC400_SRAMA`）＝1、RCPU bus＝1、DDR（2 個 TZC，各 3 個與 2 個 filter unit，×2 組即 DDR0／DDR1）、PCIe＝2。
- **每個 TZC 最多 8 個可程式化安全區域 ＋ 1 個 default region**（手冊 3.5.1.1）。
- **支援最多 256 個 outstanding transaction on normal path**（手冊 3.5.1.1）。
- **TZC-400 是 Arm 授權標準 IP**，region 設定暫存器位址等暫存器層級細節要查 Arm 官方 Technical Reference Manual，本 SoC 手冊只描述整合方式（手冊 3.5.1 Overview：「for details on the functions of TZC-400, see the relevant Technical Reference Manual」）。**這份 Arm 文件不在本手冊可用來源清單內，未查證。**
- **與記憶體控制器的關係**：本手冊 04 記憶體與儲存群提到 LPDDR4/4X 控制器帶「in-line ECC、TZC-400」——這裡的 TZC-400 就是上表 `TZC400_DDR00/01/10/11` 這四顆守 DDR 的實例。TrustZone 對 DDR 的存取管控，正是靠這幾顆 filter unit 實作的。
- **與 CM33 的 TrustZone-M 是兩件事**：本手冊 4.2 提到 CM33 帶「TrustZone-M 安全延伸」（Armv8-M 架構的安全延伸）——那是 CM33 這顆微控制器核心自己的執行模式隔離，與本單元講的「CA55 的 TrustZone ＋ TZC-400 匯流排管控」是不同層級的東西，名字相近但別混為一談。

### 什麼情況下你會用到它

**判斷準則（先判斷你的系統需不需要「兩個互看不到的世界」）**：TrustZone 這類「CPU 執行模式隔離 ＋ 匯流排存取管控」機制，設計目的是把系統切成「一般作業系統世界」與「受信任的安全世界」，兩邊互相看不到對方的記憶體。這對任何需要「金鑰保護、韌體簽章驗證、防止一般應用程式讀取敏感資料」的場景都有意義，**不限於特定應用領域**。

**但「有硬體」不等於「能用」**：要真正用到這個能力，你需要一個跑在 Secure World 裡的作業系統（常見是 OP-TEE）搭配對應驅動程式，而本板目前沒有這一層。判斷準則是：如果你的專案需要安全金鑰儲存、Secure Boot 驗證鏈、或 TEE 應用（以處理生物特徵資料、DRM 金鑰管理為例），你得**自行整合 OP-TEE 並準備 TF-A 的安全設定**，這不是開機就有的東西，要納入你的工程量估算。

**如果你的需求只是基本完整性檢查**：例如只想確認「開機時韌體有沒有被竄改」，TF-A 本身在開機流程中已經做了一部分工作（BL31 在 EL3 設定 TZC 區域）。判斷準則是：你要的是「執行期的安全應用」（需要 TEE runtime）還是「開機期的完整性把關」（TF-A 已涵蓋一部分）？後者可以先從 bootloader／TF-A 的開機記錄檢查起，不必動到 TrustZone runtime。

> **尾註（出處）**：官方硬體手冊 `r01uh1032` §3.5 TrustZone Address Space Controller (TZC)（p352–356，功能概說；暫存器明細自 3.5.2 起，未讀）。TZC 暫存器位址轉引自 doc07 §16（手冊 15770 行，未直接查證）。安全設定的實際擁有者：TF-A（BL31／EL3 開機韌體）。TZC-400 暫存器功能細節另見 Arm「TZC-400 Technical Reference Manual」（不在本手冊可用來源清單，未查證）。

---

## 單元三：Security IP（加密引擎＋OTP／Device-ID／JTAG-disable）

> ⚠️ **先講最重要的一句**：這顆 H44 料號在**板上量到 Linux 沒有任何硬體加密介面**（無 `/dev/tee`、無 Renesas TRNG hwrng、CA55 無 ARMv8 crypto 延伸）——這是板上實證。而「Linux 摸不到」本身確實不等於「矽晶沒有」（同 ISP 的教訓：這顆矽晶其實含 Mali-C55 ISP，只是 device tree 沒啟用）——但 Security 這一格不必靠板上反推：真硬體手冊 `r01uh1032` Table 1.1-1〔p78〕（就是坐實本料號 ISP＝Available 的同一張真 PDF SKU 表）把 R9A09G057H44GBG 的 Security 欄逐字標為 **N/A**，可 pypdf 核實，故本料號**未搭載** Trusted Secure IP。本單元的「這是什麼」照手冊完整講清楚它「若搭載會是什麼」，〈關鍵能力與限制〉再把板上實證與這條 SKU 判定講清楚。請把兩部分分開讀——機制是通用知識，本料號搭載與否已由一手 SKU 表坐實。

### 這是什麼（機制）

手冊 4.8 描述的 **Trusted Secure IP** 是一顆「**選配（option）**」硬體安全模組，由三個部分組成：存取管理電路（access management circuit）、加密引擎（encryption engine）、亂數產生器（random number generator）（手冊 4.8 前言：「This LSI incorporates a Trusted Secure IP module to provide security functions. The module consists of an access management circuit, encryption engine, and random number generator.」）。它的設計目的濃縮成三個關鍵字：**機密性**（confidentiality，防偷看）、**完整性**（integrity，防竄改）、**真實性**（authenticity，防冒充）——搭配對應的驅動程式即可達成（手冊 4.8：「In combination with the Trusted Secure IP driver, the Trusted Secure IP can prevent eavesdropping (confidentiality), falsification of information (integrity), and impersonation (authenticity).」）。

它最關鍵的安全特性是：**金鑰只活在 Trusted Secure IP 內部，永遠不會被讀出到外部匯流排**（手冊：「Key information to be used in encrypting and decrypting data is only stored within the Trusted Secure IP, and any external access can be shut out to obtain a system with strong security.」）。加解密過程中，金鑰與中間資料也都不會暴露到模組外（手冊 4.8.2.2）。這道「金鑰不落地」的設計，是硬體安全模組跟「用 CPU 跑軟體加密」最本質的差別——軟體加密的金鑰終究要載進一般記憶體，而這顆模組讓金鑰連晶片內部的匯流排都出不去。

它靠一台「**操作模式狀態機**」來守這道防線：Reset → Trusted Secure IP enabled mode → Self-diagnosis mode → Random number generator entropy estimation mode → Encryption engine active mode；任何一步只要偵測到「不正常存取（例如程式被竄改或跑飛）」，就會切到 **Irregular access detected state**，之後鎖死、不再輸出任何資料（手冊 4.8.2.1，Fig 4.8-2：「When irregular access to the Trusted Secure IP … is attempted, the access management circuit does not accept any subsequent access and stops the output of any data from the Trusted Secure IP.」）——這是硬體層面防止「軟體漏洞被拿來偷金鑰」的防線。

它怎麼做到「使用者金鑰永遠不以明文進入晶片外部」？靠一套**金鑰安裝流程（Key Installation）**：使用者先用 Key-2 把自己的金鑰 Key-1 加密成 eKey-1、送進晶片；晶片內部用自己保管的 Key-2 解回 Key-1，再把它轉換成「只有這顆晶片內部辨識得出的**金鑰產生資訊（key generation information）**」存進外部 flash；之後加解密都用這組資訊、而非明文金鑰（手冊 4.8.2.3，Fig 4.8-4、4.8-5）。更妙的是，這組金鑰產生資訊會結合這顆晶片獨有的 **Unique ID** 加上亂數產生——所以就算你把它整包複製到另一顆晶片，也用不了（手冊 4.8.1 Table 4.8-1「Unique ID」項：「Combining the unique ID with the key generation information prevents the illicit copying of data to another LSI.」）。

另一個**獨立但相關**的硬體區塊是 **OTP**（One-Time Programmable memory，一次性可燒錄記憶體）：實體上「**每個位元只能被寫入一次**」的非揮發性儲存區，用來放晶片個體 ID、開機相關設定、以及使用者自訂資料（手冊 4.10.1.1：「Writing to the non-volatile OTP memory macro (core) of the OTP unit proceeds in 32-bit units. However, the same bit can be written only once.」）。OTP 切出幾個功能區（手冊 4.10.2 Table 4.10-3，逐字讀取）：Chip Product ID（晶片個體識別碼，位址 `0F3h`–`0F6h`）、One-time read area enable setting（`12Ah`）、Boot device drive strength setting（開機裝置驅動強度設定，`12Ch`）、User Area 1（一次性讀取區，`160h`–`1DFh`）、User Area 2（一般使用者自訂區，`1E0h`–`3DFh`）。OTP 支援對指定區域**分別**設定寫保護（write-protect）與讀保護（read-protect）（手冊 4.10.1.1 Table 4.10-1）。

OTP 開機時會自動跑一段初始化序列：釋放系統重置後，先把 OTP 記憶體核心電源開啟，自動把 OTP 位址 `0000h`–`127Fh` 範圍內容讀進內部暫存暫存器，接著把 OTP 記憶體核心斷電（省電），完成後 OTP 單元才可被軟體存取（手冊 4.10.1.2，Fig 4.10-1）。之後若要再讀寫 OTP，軟體要自己先把 OTP 電源設定（OTPPWR）打開。

最後一個 OTP 承載的安全機制，把本單元跟〈單元一 CoreSight〉接了起來——**OTP-backed 的「JTAG-disable」保險絲**：把 JTAG-disable 這個設定燒錄進 OTP 之後，就是**實體層級不可逆地永久關閉 JTAG 除錯介面**（doc07 §49 轉錄；datasheet Figure 1.1-1 方塊圖將「JTAG Disable」列為晶片內建功能區塊之一，與「OTP 32-Kbits」並列）。它不需要 Trusted Secure IP 搭載也存在，因為它靠的是 OTP，而 OTP 是矽片固定的。

### Linux 下怎麼看到它

doc07 §49 把本單元的板上狀態判定為「**MIXED（混合）**」，因為它其實混了「未搭載」與「存在但無介面」兩種狀態，要拆開看：

- **加密引擎／TRNG／Secure-Boot 加速器：Linux 上摸不到（板上實證）；本料號未搭載（一手 SKU 表核實）。** 板上沒有 `/dev/tee`、沒有由 Renesas TRNG 提供的 hwrng 節點、也沒有廠商加密引擎的核心模組（doc07 §49 為線索，板上實證見本手冊 4.2）。矽晶層有沒有這顆 Trusted Secure IP，真硬體手冊 `r01uh1032` Table 1.1-1〔p78〕把本料號 Security 欄逐字標為 **N/A**（可 pypdf 核實，與 ISP＝Available 同一張真 PDF SKU 表）——故本料號**未搭載**此 IP（另一份 datasheet `r01ds0429` 的同類表雖劣化，本判定已改以真硬體手冊為據，見下一段）。**CA55 crypto extension：確定無**（CPU flags 無 `aes`／`sha2`，板上實證，本手冊 4.2）。
- **OTP／Device-Unique-ID／JTAG-disable：矽片上物理固定存在，但沒有 Linux 層的存取介面。** OTP 單元「沒有 userspace／MTD 層的曝露」——即使 OTP 硬體存在，Linux 應用層也摸不到它（doc07 §49）。

還有一件容易搞混、要分清楚的事：**核心加密 API（kernel crypto API）在這顆矽片上會退回純軟體 AES／SHA 運算**，這是因為 CA55 的指令集**沒有 ARMv8 Crypto Extension**——這是「CPU 指令集層級沒有硬體加速」，跟「Trusted Secure IP 這顆獨立模組是否搭載」是**兩件不同的事**，只是這兩者在本板恰好都缺席（doc07 §49；本手冊 4.2 已印證 CA55 的 CPU flags 裡沒有 `aes`／`sha2`）。

驗證指令與它們的意義（doc07 §49 使用範例）：

```bash
openssl speed -evp aes-256-gcm      # 會跑在 NEON/scalar 路徑，沒有 ARMv8 AES 硬體加速
head -c 32 /dev/urandom | xxd       # 讀到的熵源來自核心 PRNG，不是 Renesas 硬體 TRNG（TRNG 不存在於這顆矽片）
```

換句話說，本群組（除錯 ＋ 安全）三個單元在 Linux 使用者層**都沒有暴露對應介面**——CoreSight「no sysfs nodes」、TrustZone「no /dev/tee」、Security IP「OTP 無 userspace 曝露」——它們只能透過開機韌體或外部工具間接觸及（doc07 §49 一併引用了前兩者作佐證）。

> 💡 **OTP 內容在本板唯一「間接被用到」的地方**：本手冊 4.4 文件查閱表提到，溫度感測器（TSU，見 07 通訊與感測介面群）的校正 **trim 值來自 OTP**（0.0625 °C/code）。也就是說，OTP 雖然對 Linux 使用者空間完全不曝露，但它出廠燒錄的每顆晶片校正值，是被**核心的 thermal 驅動程式**在背後讀去換算溫度的。這是本板上 OTP 內容確實有在發揮作用的一個實例——只是它發生在核心驅動程式內部，不是你能用一行 Linux 指令讀出來的。

### 關鍵能力與限制

- **這是選配（option）模組；本板實際料號 R9A09G057H44GBG（板卡手冊 Page 11(板卡手冊) 元件表 U1）的 Security 欄標為 N/A——這條已由真硬體手冊坐實。** 真硬體手冊 `r01uh1032` §1.1.2 Product Lineup Table 1.1-1〔p78〕把 R9A09G057H44GBG 的 Security 欄逐字標為 **N/A**（此表可 pypdf 開啟、逐字核實，正是 00／02 檔用來坐實本料號 ISP＝Available〔Mali-C55〕的同一張表；同頁 CA55 表另註『Cryptographic extension supported (for security-supported products only)』，與本板 CPU flags 無 `aes`／`sha` 一致）——屬一手來源，故本料號**未搭載**此 IP。另一份 datasheet `r01ds0429` 的同類陣容表雖為劣化轉檔（`file` 判為 `data`、無法 pypdf 開啟、`_extracted` grep 不到陣容表，G2），但真硬體手冊既已提供乾淨 SKU 表，此判定不再倚賴它。**板上這一側**也一致：Linux 沒有任何硬體加密介面（無 `/dev/tee`、無 Renesas TRNG hwrng、CA55 無 crypto 延伸）——這是板上實證。（提醒：「Linux 摸不到」本身不足以證明矽晶沒有某顆 IP——ISP 就是矽晶含、device tree 未啟用的反例；但 Security 這一格不必靠板上反推，SKU 表已直接標 N/A。）

- **若有搭載，加密引擎規格（純屬手冊規格描述，非本板實測，逐字取自手冊 4.8.1 Table 4.8-1）**：
  - **AES**：符合 NIST FIPS PUB 197，金鑰長度 128／192／256 bits，區塊大小 128 bits；支援模式 ECB／CBC／CTR（NIST SP 800-38A）、CMAC（SP 800-38B）、CCM（SP 800-38C）、GCM（SP 800-38D）、XTS（SP 800-38E）、GCTR；AES-GCM 由 AES-GCTR ＋ GHASH 組合實現。
  - **RSA**：金鑰長度最高 4096 bits，區塊大小最高 4096 bits。
  - **HASH**：支援 SHA1、SHA224／SHA256、GHASH，區塊大小 512 bits。
  - **ECC**：相容 ECDSA 與 ECDH，資料區塊長度 256 bits。
  - **亂數產生器**：32-bit true random number generator，驅動程式可組合成 128-bit 或 256-bit 真亂數，作為加解密金鑰。
  - **中斷源**：10 個（手冊 4.8.3 Table 4.8-2：PROC_BUSY、ROMOK、LONG_PLG、WRRDY0、WRRDY1、WRRDY4、RDRDY0、RDRDY1、IWRRDY、IRDRDY）。
  - 支援 module-stop 低功耗設定。
  - 加解密單一區塊（32 bits × 4）處理時間：11、13 或 15 cycles（依演算法而異）（手冊 4.8.2.4 Fig 4.8-6）。

- **OTP 規格**（手冊 4.10.1.1 Table 4.10-1、4.10.1.3 Table 4.10-2，逐字讀取）：核心層級寫入單位 **32-bit**（同一位元僅能寫一次）；經由控制暫存器寫入時的操作寬度 **16-bit**；讀取寬度 **32-bit**。
- **OTP 容量**：「OTP 32-Kbits」（datasheet Figure 1.1-1 方塊圖標籤）。
- **OTP 基底位址**：`<OTP_base>` ＝ `0x1_0450_0000`（一般 AXI 視角）／CM33 non-secure 視角 `0x5045_0000`／CM33 secure 視角 `0x4045_0000`（手冊 4.10.2.1 Table 4.1-3，逐字對照三個 Note）。
- **使用 Trusted Secure IP 的加密功能，必須搭配 Renesas 提供的專屬驅動程式，且需另外向業務窗口索取**——手冊未內建此驅動程式的公開規格（手冊 4.8.4.1：「Use of the Trusted Secure IP requires the Trusted Secure IP driver provided by Renesas Electronics. Please contact our sales office for information regarding the Trusted Secure IP driver.」）。
- **軟體加密的實測吞吐（本板 CA55 純軟體路徑，供對照）**：因為 CA55 沒有 ARMv8 crypto 延伸，AES／SHA 只能軟體跑——單核 AES 約 35–56 MB/s、4 核約 137 MB/s（本手冊 4.2〈加密吞吐〉；出處 `05-compute-benchmark.md:79-80`）。這個吞吐足以應付資料簽章與一般小流量加密，但若有大量加密儲存／傳輸需求，要把這個上限納入設計。

### 什麼情況下你會用到它

**判斷準則一（要硬體加密／加速前，先查你這顆晶片的確切料號）**：若你的應用需要「硬體級金鑰保護 ＋ 加密演算法硬體加速」（以儲存裝置全碟加密金鑰不落地為例、或以需要高吞吐量 AES／RSA 運算為例），第一步不是去找驅動程式，而是**查清楚手上這顆 RZ/V2H 的確切料號是不是 Security＝Available**（真硬體手冊 `r01uh1032` Table 1.1-1〔p78〕即逐列出各料號的 Security 欄，可逐字核實：本板 H44 標 N/A，另有 H45／H46／H48 標 Available）。關鍵在於：同一個 RZ/V2H 家族內，不同封裝料號是否搭載 Security IP **不同**，這件事不能只看手冊正文判斷——要對照你實際拿到的晶片印字料號，去查 Product Lineup 表。

**判斷準則二（若板上量到沒有硬體加密介面，別繞遠路）**：如果你的板子跟本板一樣、在 Linux 上量到沒有硬體加密引擎與 TRNG 介面（本板即如此，且真硬體手冊 Table 1.1-1〔p78〕標本料號 Security＝N/A），實務上加密需求就只能靠 CPU 軟體運算（以 OpenSSL 純軟體路徑為例）或另外加裝外部安全晶片（如 TPM）。判斷準則是：你要花時間找 Renesas 的專屬加密驅動程式之前，先在板上確認硬體介面真的在——在本板上它不在 Linux 裡，找了也綁不上。

**判斷準則三（只要個體 ID，不必動用整個加密模組）**：OTP 裡的 Chip Product ID（晶片個體識別碼）**不需要 Trusted Secure IP 搭配也能存在**。如果你只是要一個「這顆晶片獨一無二的 ID」（以裝置授權、防複製的序號比對為例），理論上 OTP 是候選；但本板 doc07 已標明它沒有 Linux userspace／MTD 層存取介面。判斷準則是：實際要不要用，得先確認你是否願意撰寫 bare-metal／韌體層的存取程式（透過手冊 4.10.2.1 所述的 OTP control registers）——這不是 Linux 應用層一行指令就讀得到的。

**判斷準則四（JTAG-disable 是出貨前的最終防護，燒了不能回頭）**：這個 OTP 保險絲的判斷準則是「**產品出貨前的最終防護手段**」。開發階段一定要留著 JTAG／SWD 可用以便除錯（參見〈單元一：CoreSight〉）；只有在你確定不再需要實體除錯、且要防止有心人透過 JTAG 讀取記憶體內容或韌體時，才會考慮在量產前燒斷這個保險絲。務必記住：**OTP 寫入不可逆，燒斷後這片板子永遠無法再用 JTAG／SWD 除錯**——這是一條有去無回的決定，要在確定產品定案後才做。

> **尾註（出處）**：官方硬體手冊 `r01uh1032` §4.8 Trusted Secure IP（p1106–1118，功能概說與操作原理）＋ §4.10 OTP（p1137–1141，功能概說；OTP 完整暫存器清單自 4.10.2.1 起，Table 4.10-4 (2/2) 控制暫存器細節頁未讀）。搭載與否的判定：**真硬體手冊 `r01uh1032` §1.1.2 Product Lineup Table 1.1-1〔p78〕**（本料號 R9A09G057H44GBG Security 欄逐字標 N/A，可 pypdf 核實、屬一手來源，同表亦據以坐實 ISP＝Available〔Mali-C55〕）；另一份 datasheet `r01ds0429` 的同類表 Table 1.2-1 為劣化格式（G2），本判定不倚賴它。OTP 存在另有真硬體手冊 §4.10 為據（上引）。板上實證（無 `/dev/tee`／hwrng／crypto flags）：本手冊 4.2。軟體加密吞吐：`05-compute-benchmark.md`。加密功能需 Renesas Trusted Secure IP driver（需向業務窗口索取，規格未公開）。JTAG-disable 轉錄自 doc07 §49。

---

## 動手：溫度監看與熱安全餘裕（以晶片溫度感測 TSU 為例）

前面三個單元講的是「除錯」與「加密安全」。這一節補上「**熱**安全」——怎麼在 Linux 上讀晶片溫度、看清楚它的自動保護門檻（trip point）在哪、以及在高負載下溫度與門檻之間還剩多少餘裕。這件事不需要任何外部工具，一條 `cat` 就能讀，但要看懂讀到的數字代表什麼、哪個數字才是「快要出事」的紅線，得先把機制接起來。

> **感測器本身的機制在 07**：晶片溫度由 TSU（Temperature Sensor Unit，晶片內建溫度感測單元，2 組 TSU0／TSU1）量測，走 Linux thermal 框架、驅動程式 `rzv2h_thermal`、曝露為兩個 thermal zone——這部分的硬體機制（類比感測器＋專屬 ADC、量的是 die 溫度而非環境溫度）在 07〈通訊與感測介面群〉的 TSU 小節已完整講過（官方出處 `r01uh1032` §7.11，p3739）。本節只聚焦「怎麼把它當熱安全監看工具用」。

### 步驟一：讀兩個 thermal zone 的當下溫度

```bash
for z in /sys/class/thermal/thermal_zone*; do echo "== $z type=$(cat $z/type) temp=$(cat $z/temp)"; done
```

預期輸出（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-tsu.txt；當下 load average 1.37，非閒置也非重載）：

```text
== /sys/class/thermal/thermal_zone0 type=sensor-thermal0 temp=37000
== /sys/class/thermal/thermal_zone1 type=sensor-thermal1 temp=38000
```

判讀：`temp` 的單位是**毫度**（milli-degree Celsius），所以 `37000` = 37°C、`38000` = 38°C。兩個 zone 對應晶片上兩組 TSU（type 字串 `sensor-thermal0`／`sensor-thermal1`），量的是**封裝內部 die 溫度**，不是機箱或環境溫度。

### 步驟二：讀 trip point——晶片的自動保護紅線在哪

光讀當下溫度不夠，你得知道「到幾度系統會自己動作」。thermal 框架把這些門檻叫 **trip point**：

```bash
for z in /sys/class/thermal/thermal_zone*; do for t in $z/trip_point_*_type; do n=${t%_type}; echo "$z ${n##*/}: $(cat $n\_type) $(cat $n\_temp) hyst=$(cat $n\_hyst 2>/dev/null)"; done; done
```

預期輸出（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-tsu.txt）：

```text
/sys/class/thermal/thermal_zone0 trip_point_0: critical 120000 hyst=1000
/sys/class/thermal/thermal_zone1 trip_point_0: critical 120000 hyst=1000
```

判讀：每個 zone 只有**一個** trip point，型別 `critical`、溫度 `120000` 毫度 = **120°C**、遲滯（hyst）1°C。`critical` 是最高級別——溫度撞到它，核心會直接觸發緊急關機保護晶片。**要記住的重點：這裡沒有註冊任何 `passive`（被動降頻）或 active（主動散熱）trip，只有一條 120°C 的緊急關機線。** 換句話說，120°C 以下沒有任何「自動幫你降頻」的門檻在管你——溫度管理若要更細緻，得你自己讀值做策略（見下方陷阱框與 07 的節流做法）。

再確認一次「沒有綁任何散熱裝置」：

```bash
grep -H . /sys/class/thermal/cooling_device*/type 2>/dev/null; echo rc=$?
```

預期輸出（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-tsu.txt）：`rc=2`——`grep` 找不到任何 `cooling_device*` 檔案（rc=2 是「檔案不存在」），代表這片板子的 thermal 框架下**沒有註冊任何 cooling device**（風扇、cpufreq 降頻器都沒掛）。這跟上面「只有 critical trip、沒有 passive trip」是同一件事的兩面。

### 步驟三：把溫度放進 stress 負載脈絡看餘裕

單看 38°C 沒有意義，要看「壓下去會到哪、離 120°C 紅線還多遠」。以連續全核 CPU 壓力為例（用 `stress-ng` 把 4 顆 CA55 壓滿 5 分鐘，每 30 秒取一次溫度與頻率）：

```bash
# 先記基線
for z in /sys/class/thermal/thermal_zone*/temp; do echo "baseline $z $(cat $z)"; done
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 壓 5 分鐘，邊壓邊取樣
stress-ng --cpu 4 --timeout 300 --metrics-brief &   # 乾淨映像檔未預裝時：sudo apt install stress-ng
for i in $(seq 1 10); do sleep 30; echo "t=$((i*30))s temps=$(cat /sys/class/thermal/thermal_zone0/temp) $(cat /sys/class/thermal/thermal_zone1/temp) freq=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq)"; done
```

實測結果（✅ 2026-07-18 板上實測，transcript：live/win-03-stress.txt；量測條件：板上、`stress-ng --cpu 4`、5 分鐘）：

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

把這組數字讀懂：
- **基線 tz0 34°C／tz1 35°C**（該日、該機的 idle 值；跨 session 的 idle 實際橫跨約 34–36°C，引用單點要標日期／主機）。
- **壓滿 5 分鐘後末值 tz0 37°C／tz1 38°C，峰值 tz1 39°C**（出現在 t=180–270s）。也就是全核壓力只把 die 溫度推高約 3–4°C。
- **全程頻率固定 1700000 Hz（1.7 GHz）零降頻**——這正好對上步驟二的發現：沒有 passive trip、沒有 cooling device，所以系統不會（也沒有門檻要它）自動降頻；能穩在 1.7 GHz 是因為根本沒撞到任何會觸發節流的溫度。
- **離紅線的餘裕**：峰值 39°C 距離唯一的 critical trip 120°C 還有約 **81°C** 的餘裕。以這個負載型態（純 CPU）看，熱完全不是瓶頸。

### 驗證判準

- **讀得到兩個 zone、`temp` 是合理的室溫等級毫度值**（數萬，例如 `37000`）：代表 TSU 驅動程式正常作動。
- **trip point 讀到 `critical 120000`**：確認你看到的是緊急關機線，而不是把某個中間值誤當紅線。
- **壓力測試中頻率不掉、溫度遠低於 trip**：代表在該負載下熱有餘裕；反過來，若你看到頻率開始往下掉、或溫度逼近某個門檻，才是需要介入散熱／節流的訊號。

> ⚠️ **注意（別把 die 溫度當環境溫度、也別以為系統會自動幫你降頻）**：
> - **情境**：你想用 `thermal_zone*/temp` 當作機箱或環境溫度、或假設溫度過高時系統會自動降頻保護。
> - **症狀**：以為溫度偏高就是環境熱；或在長時間重載（以 NPU＋GPU＋CPU 同時滿載為例）下溫度悄悄爬升卻沒看到任何自動降頻。
> - **原因**：TSU 量的是晶片內部 die 溫度，與環境溫度可能有明顯落差（有主動散熱或氣流時差更大）；而本板 thermal 框架下只有一條 120°C 的 critical trip、沒有 passive 降頻 trip、也沒有 cooling device——120°C 以下沒有任何自動節流在保護你。
> - **預防／處理**：要量環境溫度得另接外部感測器（見 07）。要在撞到 120°C 緊急關機之前就控制溫度，得**自己**讀 `thermal_zone*/temp` 搭配自訂節流策略（以溫度過某個自訂上限就降低送進 NPU／GPU 的工作量為例）。這一段要納入你的系統設計，不能指望開機就有。

> **尾註（出處）**：TSU 感測器機制與精度規格見 07〈通訊與感測介面群〉TSU 小節（官方硬體手冊 `r01uh1032` §7.11 Temperature Sensor Unit，p3739）；校正 trim 值 0.0625 °C/code 由核心 thermal 驅動程式從 OTP 讀取換算（見本群組〈單元三〉的 OTP 說明）。板上讀值／trip／cooling device 為板上實測 live/ch4-w1-tsu.txt（2026-07-22）；溫度與 stress 負載關係為板上實測 live/win-03-stress.txt（2026-07-18）。

---

## 本群組的證據邊界與未查證項

為了讓你能判斷「哪些內容是直接讀官方手冊、哪些是轉引開發紀錄」，這裡把本群組的證據邊界誠實攤開——引用前請對照這一節，別把轉引內容當成已直接查證的結論。

- **實讀的 PDF 頁段**：手冊 `r01uh1032ej0130` 的 p352–356（TZC，5 頁）＋ p1106–1123（Trusted Secure IP ＋ Debug Interface，18 頁）＋ p1137–1141（OTP，5 頁）＝ 共 28 頁功能概說。
- **未讀的暫存器明細頁（依單元地圖規則本就排除）**：CoreSight 自 4.9.2 起、TZC 自 3.5.2 起、Trusted Secure IP 的暫存器細節、OTP 4.10.2.1 起的完整暫存器清單（Table 4.10-4 只讀到 (1/2) 頁，(2/2) 控制暫存器細節頁落在頁段外未讀）。凡本群組出現的暫存器**位址**明細（CoreSight 除錯位址表、TZC 實例位址、OTP 基底位址），除 OTP 基底位址直接取自手冊 4.10.2.1 Table 4.1-3 外，CoreSight 與 TZC 的位址表**皆轉錄自 doc07**（`07-hardware-unit-usage-guide.md` §15／§16）、未直接查證原始暫存器頁，文中均已就地標記。
- **未查證的外部文件**：Arm「CoreSight Trace Memory Controller Technical Reference Manual」、Arm「TZC-400 Technical Reference Manual」、TF-A（BL31／EL3 韌體）——doc07 引用了這些文件的部分資訊，但它們本體不在本手冊可用來源清單內，故凡轉引自 doc07 而非直讀 PDF 的內容，皆已標「未直接查證」。
- **開發紀錄交叉比對**：`06-hardware-resource-map.md`（doc06）全文讀畢，未單獨提及 CoreSight／TrustZone／Security IP 三者（其「運算與加速器」「介面匯流排」等節聚焦在有 device tree 節點、有驅動程式綁定的區塊），與 doc07 §15／§16／§49「本板無 Linux 介面」的結論一致、無衝突——這也是為什麼這三個單元在本章資源地圖的狀態標記裡全都不是「✅ 啟用」、而是「存在·Linux 未曝露」與「未搭載／混合」。
- **Security 料號判定（一手核實）**：本板料號經板卡手冊元件表坐實為 R9A09G057H44GBG（Page 11(板卡手冊)）；真硬體手冊 `r01uh1032` Table 1.1-1〔p78〕把它的 Security 欄逐字標為 **N/A**（可 pypdf 核實，與坐實 ISP＝Available 同一張表），故本料號**未搭載** Trusted Secure IP。板上這一側一致：Linux 無任何硬體加密介面（無 `/dev/tee`／hwrng、CA55 無 crypto 延伸，本手冊 4.2 實證）。
