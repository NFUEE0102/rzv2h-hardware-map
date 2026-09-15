# 01 · 運算單元（4.2 運算單元深入）

> 本檔是「04 · 全板硬體資源地圖」資料夾的運算單元深入檔（對應章節號 **4.2**）。章首說明、學習目標、`<板子IP>` 佔位與標記慣例、以及 4.1 全板總覽都在 [00-總覽與IP啟用地圖](00-overview-and-ip-enablement-map.md)；內文出現「4.1」指 00 檔、「4.4」指 99 檔。標 ✅ 的板上複驗步驟均附 transcript 檔名，指向 handbook 資料夾下 `live/` 的板上實錄（自本資料夾起算為 `../live/ch04*.txt`）。

## 資料夾檔案清單

| 檔案 | 內容 |
|---|---|
| [00-總覽與IP啟用地圖](00-overview-and-ip-enablement-map.md) | 章首＋4.1 全板總覽與啟用地圖＋章末回顧／速查表／實測數據表 |
| **01-運算單元.md**（本檔） | A55／R8／M33／GPU／DRP-AI3 深入：規格、benchmark 實測、能力上限推導；DRP1 的定位見 00 檔分群總表、深講見第 3 章 |
| [g2-影像擷取・編解碼與顯示](g2-video-capture-codec-display.md)～[g8-除錯與安全](g8-debug-and-security.md) | 周邊與介面逐單元深講：影像（g2）、音訊（g3）、記憶體與儲存（g4）、系統骨幹（g5）、計時（g6）、通訊與感測（g7）、除錯與安全（g8） |
| [99-官方文件查閱指路](99-official-documentation-guide.md) | 官方文件哪份答什麼問題、`_toc_full.txt` 秒查法 |

---

## 本檔章內目錄

- [4.2 運算單元深入](#42-運算單元深入)
  - [為什麼是「一堆不一樣的核心」，而不是一顆很快的 CPU](#為什麼是一堆不一樣的核心而不是一顆很快的-cpu)
  - [四顆核心，四種分工（各核用途與規格）](#四顆核心四種分工各核用途與規格)
  - [動手：先確認這幾顆核心都在、狀態正確](#動手先確認這幾顆核心都在狀態正確各步標記見下)
  - [Benchmark 實測數據（每一項都附量測條件）](#benchmark-實測數據每一項都附量測條件)
  - [能力上限推導：某演算法能跑到幾 Hz](#能力上限推導某演算法能跑到幾-hz)
  - [三大運算引擎，一句話各歸其位](#三大運算引擎一句話各歸其位)
  - [動手驗證：確認你手上這塊板子跟本節數據一致](#動手驗證確認你手上這塊板子跟本節數據一致各步標記見下)

---

## 4.2 運算單元深入

上一節（4.1，見 [00-總覽與IP啟用地圖](00-overview-and-ip-enablement-map.md)）你已經把整塊 SoC 的硬體區塊「點過名」，知道哪些啟用、哪些停用、記憶體怎麼切。這一節要做的事不一樣：把鏡頭拉近到「會算數的那幾顆核心」——A55、R8、M33、GPU，加上 DRP-AI3 NPU 的運算面——一顆一顆看清楚它們**各自是什麼、擅長什麼、能撐到多快**。做系統規劃時，這是你最常回頭翻的一節——因為「把即時控制迴圈放哪（以飛行控制為例）、把感知推論放哪、把影像前處理放哪」這種決定，答案全在這裡。

（本節出現的所有板子位址一律寫成 `<板子IP>`，緣由見 00 檔章首〈你需要準備什麼〉。）

### 為什麼是「一堆不一樣的核心」，而不是一顆很快的 CPU

一般桌機的思路是「一顆 CPU 越快越好」。嵌入式 SoC 走的是另一條路——**異質運算（heterogeneous computing）**：晶片裡放好幾種專長不同的運算單元，每種只做它最擅長的事。RZ/V2H 就是這種設計。理解它，等於理解「為什麼這類即時應用的軟體要拆開來擺在不同核心上」。

這塊 SoC 上會算數的單元有五種（出處 06-hardware-resource-map.md:28-40）：

```text
┌──────────────────────────────────────────────────────────────────┐
│  RZ/V2H (R9A09G057H44) 異質運算單元                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌────────────────────┐   一般運算 / Linux 應用 / 估測 / 規劃      │
│  │  Cortex-A55 × 4     │   跑 Ubuntu、ROS、EKF、MPC、通訊協定       │
│  │  @1.7 GHz           │   ← 你的程式「主場」                        │
│  └────────────────────┘                                            │
│                                                                    │
│  ┌────────────────────┐   硬即時 / 馬達控制 / 安全共處理器          │
│  │  Cortex-R8 × 2      │   即時內迴圈的理想落點                      │
│  │  @800 MHz           │   ← 目前 Linux 未曝露，待載入韌體          │
│  └────────────────────┘                                            │
│                                                                    │
│  ┌────────────────────┐   系統管理 / 低功耗常開監督者              │
│  │  Cortex-M33         │   remoteproc0，預設 offline                │
│  │  @200 MHz           │   ← 韌體未內建                             │
│  └────────────────────┘                                            │
│                                                                    │
│  ┌────────────────────┐   INT8 CNN 推論引擎（視覺 AI）             │
│  │  DRP-AI3 NPU        │   8 dense / 80 sparse TOPS                 │
│  │  /dev/drpai0        │   ← 感知的決定性優勢                       │
│  └────────────────────┘                                            │
│                                                                    │
│  ┌────────────────────┐   逐像素並行影像/CV、零拷貝、HMI           │
│  │  Mali-G31 GPU       │   OpenCL 3.0，1 shader core               │
│  │  @630 MHz /dev/mali0│   ← 不是 AI 主力，也不是通用浮點加速器     │
│  └────────────────────┘                                            │
│                                                                    │
│  （另有 DRP1 可重組處理器做非 AI 的 CV 加速，見 4.1 分群總表）      │
└──────────────────────────────────────────────────────────────────┘
```

這一節接下來會逐一深入前四類的運算面（A55、R8、M33、GPU；DRP-AI3 因為它是感知章的主角，這裡只講「它作為一個運算引擎的吞吐與能力邊界」，細節留給 DRP-AI 章）。看完你會拿到三樣東西：**每顆核的用途與規格**、**逐項附條件的 benchmark 實測數據**、以及**把實測翻譯成「某演算法能跑到幾 Hz」的能力上限推導**。

> 💡 **提示**：讀這一節時，心裡先記一條主線——「A55 是通用算力大幅過剩，真正的限制不是 FLOPS（每秒浮點運算次數），而是 (1) 即時抖動、(2) Python 派送開銷、(3) 沒有硬體加密。」（出處 09-compute-capability.md:17、05-compute-benchmark.md:27）後面所有數字都是在幫你量化這三條限制到底卡在哪裡。

### 四顆核心，四種分工（各核用途與規格）

#### Cortex-A55 × 4：你的程式主場

**它是什麼。** 四顆 Arm Cortex-A55（步進 r2p0），單一叢集（cluster），跑在 1.7 GHz。這是唯一直接跑 Linux 的核心群——Ubuntu、你的 C/C++/Python 程式、ROS、估測器、規劃器、通訊協定，預設全都在這四顆上面。它們本身就是 Linux kernel 的宿主（host），所以**沒有**對應的裝置節點（device node）；你看不到 `/dev/a55` 這種東西，核心拓撲要去 `/sys/devices/system/cpu/cpu0` 到 `cpu3` 查（出處 07-hardware-unit-usage-guide.md:22）。

**關鍵規格（實測，2026-06-21）：**

| 項目 | 值 | 出處 |
|---|---|---|
| 核心數 | 4（單叢集，每核一執行緒） | 00_inventory.txt:29-42 |
| 時脈 | 1.7 GHz（可選 212.5 / 425 / 850 / 1700 MHz） | 05-compute-benchmark.md:45、00_inventory.txt:68-72 |
| BogoMIPS | 48.00 | 00_inventory.txt:75-80 |
| CPU part / variant / rev | 0xd05 / 0x2 / 0 | 00_inventory.txt:75-80 |
| 指令集延伸 | `fp asimd(NEON) fphp asimdhp(fp16) asimddp(INT8 dot) crc32 atomics lrcpc dcpop` | 05-compute-benchmark.md:46、00_inventory.txt:43 |
| 加密延伸 | **無 `aes`／`sha2`**（這條後面會反覆出現，很重要） | 05-compute-benchmark.md:46 |

**為什麼「有 dotprod、fp16，卻沒有 aes/sha」很重要。** `asimddp`（INT8 dot product）讓 A55 能相對有效率地做 INT8 內積，這對量化神經網路（量化＝把權重／啟動值從 fp32 壓成 INT8 這類低位元整數，換取更小的模型與更快的 NPU／整數運算）的 CPU 後援路徑有幫助；`fphp`/`asimdhp`（fp16）讓半精度浮點有硬體支援。但指令集裡**沒有** ARMv8 Crypto Extensions——這代表 AES 與 SHA 只能用軟體實作跑，速度會被綁在數十 MB/s 等級（後面 benchmark 會給你確切數字）。這不是故障，是這顆 H44 型號在板上實測到的特性：CPU flags 裡就是沒有 `aes`／`sha2`（見上表，本手冊 4.2 實證）。datasheet 另註 crypto extension「僅供 security 版產品」，可作線索但屬劣化格式暫定（出處 07-hardware-unit-usage-guide.md:19、datasheet.md:180）。

**快取命名的一個誠實提醒（不同來源口徑不一）。** 你查 `lscpu` 會看到「L2 cache: 1 MiB (1 instance)」（出處 00_inventory.txt:44）；但 datasheet 的表述是「L1 32KB I + 32KB D/core、**L2 = 0KB、L3 = 1MB**（ECC，最高 1.26 GHz）」（ECC＝記憶體錯誤更正碼；出處 datasheet.md:150-157）。兩邊講的是同一塊 1 MB 共享快取，只是**命名層級不同**：作業系統把它報成 L2，datasheet 把它算成 L3。你只要知道「這顆有一塊 1 MB 的共享末級快取」即可，不必糾結它叫 L2 還是 L3。

> ⚠️ **注意**：A55 的**啟動頻率由板上 DIP 開關 `DSW1` 的 `BOOTPLLCA[1:0]` 兩位訊號決定**，板卡手冊的對照表是唯一權威（ON＝High、OFF＝Low）：Low:Low＝1.1 GHz、Low:High＝1.5 GHz@0.9V、High:Low＝1.6 GHz@0.9V、**High:High＝1.7 GHz@0.9V（出廠預設）**——四檔全部有定義，**不存在 1.8 GHz 的 DIP 檔位**（出處：WS125 板卡手冊 §3.3 DSW1 表，Page 6；完整逐位對照見第 01 章）。datasheet 上的「最高 1.8 GHz@0.9V」是**矽晶額定上限**，不是 DSW1 可選的啟動檔位——兩者是不同層的數字，別混為一談。**預防**：不要把「某檔位的確切頻率」從二手筆記抄進程式；開機後用下面的指令實際讀 `scaling_cur_freq` 確認。本手冊全篇的 CPU 實測都在**預設的 1.7 GHz、governor 鎖 performance** 下量得，這一點沒有歧義。

#### Cortex-R8 × 2：硬即時控制的理想落點（以飛行控制為例；但目前 Linux 看不到）

**它是什麼。** 兩顆 Arm Cortex-R8（雙核 MPCore），800 MHz，Armv7-R 架構，帶 NEON+FPU，I/D-cache 各 32KB（ECC），還有各 128KB 的緊耦合記憶體（I-TCM / D-TCM，ECC），支援 MMU（出處 07-hardware-unit-usage-guide.md:33、datasheet.md:158-164）。R 系列是 Arm 的「即時處理器」線——它的價值不在算得多快，而在**算得多準時**：可預測的中斷延遲、TCM 讓關鍵程式碼不受快取未命中干擾。datasheet 標明的設計用途就是硬即時、馬達控制、安全共處理器。

> ⚠️ **注意**：這顆 R8 **沒有 dual-link lock-step**——真硬體手冊 `r01uh1032` §1.1.3 Functions、Table 1.1-2 CPU 的 Cortex-R8（CR8）列逐字寫『No support for dual-link lock-step technology』（p78；劣化 datasheet `r01ds0429` L164 同載可為線索）。lock-step 是把兩顆核跑同一份程式、逐週期比對來抓硬體錯誤的安全機制；沒有它，代表你不能把這對 R8 當成「已通過功能安全冗餘」的核心來用。做安全關鍵設計時，這是要先知道的邊界。

**板上現況：存在，但執行中的 Linux 沒有曝露它。** 這是最容易誤會的一點，也正是 4.1 分群總表把 R8 標成 🟡 的完整背景。R8 在矽片上是 PRESENT（存在）的，remoteproc 框架也把它要用的通訊環（vring）位址保留好了（0x42f00000–0x43500000），但你在**執行中的 Linux** 裡找不到 `/sys/class/remoteproc` 底下的 R8 節點（出處 07-hardware-unit-usage-guide.md:35；✅ 2026-07-18 板上重執行（transcript：live/ch04b-runtime.txt）：`ls /sys/class/remoteproc/` 只有 `remoteproc0`（即 M33），`/sys/class/uio` 也是空的——韌體未載入前，R8 的 remoteproc 節點與 UIO 裝置都不會出現）。原因是：出貨映像檔（stock image）裡，R8 是由 u-boot 在開機階段啟動的，不是由跑起來之後的 Linux 啟動的。要讓它動起來、並讓 A55 與它用 RPMsg（remote processor messaging，Linux 的跨核訊息通道）通訊，需要載入 R8 韌體（來自 Renesas Multi-OS Package），韌體載入後 RPMsg 路徑才會經 UIO（userspace I/O，把硬體暫存器直接映射給使用者空間的簡易驅動程式框架）裝置 `42f00000.rsctbl` / `43000000.vring-ctl0` / `43200000.vring-shm0` 出現，CA55 端的範例程式是 `rpmsg_sample_client`（出處 07-hardware-unit-usage-guide.md:36）。

**為什麼硬即時內迴圈（以飛行控制為例）該放這裡。** 後面「即時性」benchmark 會證明：Linux 上的 A55（PREEMPT 非即時核心）最壞排程抖動落在次毫秒級，這對 300–500 Hz 以上的硬期限迴圈太緊。R8 沒有 Linux 那層排程不確定性，正是姿態/轉速內環、馬達換相這類「錯過一拍就出事」的迴圈該去的地方（出處 06-hardware-resource-map.md:122、09-compute-capability.md:35）。R8 本身的參考數字——CoreMark、串級 PID 控制 tick、中斷進入延遲與抖動、UART 表現，並在同樣工作負載下與 STM32H745 的 Cortex-M7 對比——見下方〈實測：Cortex-R8 對 STM32H745〉（出處 05-compute-benchmark.md §13）。

#### 實測：Cortex-R8 對 STM32H745（即時性對比）

本節前面的數字告訴你 Linux 那側能做到什麼；這一小節給的是硬即時控制路徑真正所在那顆核的參考數字：**Cortex-R8**（800 MHz 級）在同樣的工作負載下，對上 **STM32H745**（NUCLEO-H745ZI-Q，400 MHz 級）的 Cortex-M7。當你要決定某條內迴圈該放在這顆 SoC 的 R8、還是另外配一顆即時 MCU 時，翻這張表（出處 05-compute-benchmark.md §13）。

**這些參考值成立的條件。** 兩邊跑同一份 C 原始碼，同一版 GCC 13.3.1、`-O2`，各以核心自己的週期計數器計時（M7：DWT，R8：PMU），再以實測時脈換算。R8 側是 core1 上的 FreeRTOS 任務（FSP 3.1.0），792.1 MHz（PMU 對 GTM tick 校正），背景有 1 kHz tick 與 rpmsg，A55 Linux 端閒置；H745 側是無中斷干擾的裸機，403.8 MHz（PLL 設 400 MHz，HSE 取自 ST-LINK MCO，約快 0.9%），用出廠的直接 SMPS 供電、VOS1。UART 那組數字裡，H745 的 RX 是由 PC 真實灌進來的硬體路徑，R8 這邊 RSCI5 腳位在本板上未接線，因此 RX 路徑為**模擬**（細節見下方 UART 表）。

**一句話：R8 算得快，H745 反應快。**

| 面向 | R8 對 H745 | 較佳 |
|---|---|---|
| 一般運算（CoreMark） | 2,676 對 1,615 iter/s | R8 ×1.66（每 MHz 反而是 M7 強：3.38 對 4.00） |
| 常規串級 PID 飛控 tick（float） | 2.44 對 5.06 µs | R8 ×2.07（每週期幾乎相同，差距來自時脈） |
| 雙精度浮點（dgemm） | 244.8 對 51.1 MFLOPS | R8 ×4.79 |
| 主記憶體搬移（memcpy 128 KB） | 117 對 518–654 MB/s | H745 ×4.4–5.6 |
| tick 中斷進入延遲 | 497 對 74 ns | H745 ×6.7 |
| tick 週期抖動 | ±440 對 ±30 ns | H745 ×14.9 |

在 921600 baud 全雙工 UART 加 1 kHz 控制迴路下，兩者都維持**零溢位、零錯包**，負載都在 10% 以下，皆可勝任常規飛控。

**兩個平台並排。**

| 項目 | RZ/V2H · Cortex-R8 core1 | STM32H745ZI · Cortex-M7 |
|---|---|---|
| 時脈 | 792.1 MHz（PMU 對 GTM tick 校正） | 403.8 MHz（PLL 設 400 MHz，HSE 取自 ST-LINK MCO，約快 0.9%） |
| 電源 / 電壓 | — | 直接 SMPS、VOS1（板子出廠接法） |
| 浮點單元 | VFPv3-D16（單＋雙精度） | FPv5-D16（單＋雙精度） |
| 程式碼位置 | 系統 SRAM / DDR + I-cache | Flash（2 WS）+ 16 KB I-cache |
| 資料 | SRAM / DDR，開 D-cache | DTCM（堆疊／靜態）、AXI SRAM（heap），開 D-cache |
| 執行環境 | core1 FreeRTOS 任務（FSP 3.1.0），背景有 1 kHz tick 與 rpmsg | 裸機、無中斷干擾 |

**運算與記憶體。**

| 參數 | 條件 | R8 | H745 | 單位 | 較佳 |
|---|---|---:|---:|---|---|
| CoreMark | 30,000 次，CRC 0x5275 通過 | 2,676 | 1,615 | iter/s | R8 ×1.66 |
| CoreMark / MHz | 每週期效率 | 3.38 | 4.00 | /MHz | H745 ×1.18 |
| dgemm 16×16 | 雙精度 | 244.8 | 51.1 | MFLOPS | R8 ×4.79 |
| sgemm 16×16 | 單精度 | 288.8 | 107.9 | MFLOPS | R8 ×2.68 |
| sin() | 雙精度，newlib | 151 | 409 | ns/次 | R8 ×2.71 |
| sinf() | 單精度 | 96 | 279 | ns/次 | R8 ×2.89 |
| sqrt() | 雙精度 | 65 | 138 | ns/次 | R8 ×2.11 |
| memcpy 8 KB | L1 常駐 | 1,495 | 1,504 | MB/s | 相同 |
| memcpy 128 KB | 主記憶體（R8 DDR，H745 AXI SRAM） | 117 | 518–654 | MB/s | H745 ×4.4–5.6 |
| memset 128 KB | 主記憶體，只寫 | 1,089 | 946 | MB/s | R8 ×1.15 |

引用這張表時要記得兩件事。H745 的 memcpy 128 KB 會隨緩衝區落在 AXI SRAM 的位置與對齊而變——一種擺法 654 MB/s、另一種 518 MB/s——所以列成範圍。另外，R8 的 CoreMark（2,676）與本節前面那個 A55 單核數字**屬於不同核心**，不可直接並列。

**常規串級 PID 飛控。** 工作負載是仿 PX4 的完整鏈，全 float、PX4 預設增益，每個迴路每 tick 都跑——這是最壞情況，因為實際 PX4 的位置／姿態迴路比角速率迴路慢：

> 陀螺／加速度計 biquad 低通 + 陷波 → Mahony 四元數姿態 → 位置 P → 速度 PID（抗積分飽和）→ 推力向量 + 傾角限制 → 由推力方向與航向組姿態設定四元數 → 四元數姿態 P → 角速率 PID（量測端 D + 濾波、積分鉗位）→ Quad-X 混控（airmode 去飽和、偏航餘裕、推力線性化）

| 參數 | R8 | H745 | 單位 | 較佳 |
|---|---:|---:|---|---|
| tick 時間 Min | 2.26 | 4.73 | µs | R8 ×2.10 |
| tick 時間 Typ（平均） | 2.44 | 5.06 | µs | R8 ×2.07 |
| tick 時間 Max | 7.96 | 6.38 | µs | H745 ×1.25 |
| 每 tick 週期數 | 1,933 | 2,040 | cycles | 相同 |
| 1 kHz 下的 CPU 負載 | 0.24 | 0.51 | % | — |

數值每 100,000 tick 取樣一次。R8 的最大值較高是 FreeRTOS 中斷插隊所致；H745 側是無中斷裸機，所以「最大值」這一欄本來就對 H745 有利，看的時候要把這點算進去。

**UART + 中斷延遲。**

**條件。** 計時器中斷以最高優先權驅動 1 kHz 迴路，並在中斷內跑上面那條 PID 鏈；每 tick 以 921600 baud 送一包 50 B 的 MAVLink v2 格式遙測（TX 分成中斷與 DMA 兩種）。RX 則以 921600 的位元組速率持續進來，每個位元組一次中斷，各跑同一個 CRC 解框器。UART/DMA 中斷優先權較低，可被 tick 搶佔，其耗時已扣除被搶佔的時間。下表除另註明者外，都是 TX 中斷 + RX 灌滿這個條件下的值。

| 項目 | R8 | H745 |
|---|---|---|
| tick 計時器 | GTM0（Linux 唯一沒綁的 GTM），GIC 優先權 0，不受 FreeRTOS 臨界區遮罩 | TIM5，NVIC 優先權 0 |
| UART | RSCI5（P72/P73）；**這組腳位在本板上未接線，因此 RX 為模擬**：GPT6 溢位中斷經 INTR8SEL 以 92.16 kHz 觸發，讀一次 RDR 再跑同一解框器；TX 真實送出但無接收端 | USART3 → ST-LINK 虛擬 COM 埠；**RX 由 PC 真實灌入，並驗證收到的遙測** |
| TX DMA | DMAC_B unit 0（FSP `R_DMAC_B_Reconfigure`）+ D-cache clean | DMA1 + DMAMUX（暫存器直寫）+ D-cache clean |
| ISR 寫法 | 暫存器層級，執行期掛進 FSP 向量表；tick ISR 自行保存 VFP 暫存器 | 暫存器層級 |

| 參數 | 條件 | R8 Min / Typ / Max | H745 Min / Typ / Max | 單位 | 較佳 |
|---|---|---:|---:|---|---|
| tick 中斷進入延遲 | 含讀計數器一次（R8 99 ns，H745 40 ns） | 390 / 497 / 870 | 64 / 74 / 104 | ns | H745 ×6.7 |
| tick 週期偏差 | 相對平均 | −444 / 0 / +435 | −30 / 0 / +29 | ns | H745 ×14.9 |
| PID 控制器 | 在中斷內執行 | 2.27 / 2.41 / 2.93 | 4.93 / 5.17 / 5.41 | µs | R8 ×2.1 |
| 遙測打包 + 啟動 TX | TX 中斷模式 | 2.25 / 2.27 / 3.10 | 1.75 / 1.76 / 1.99 | µs | H745 ×1.3 |
| 遙測打包 + 啟動 TX | DMA 模式 | 6.34 / 6.95 / 7.54 | 1.95 / 1.95 / 1.99 | µs | H745 ×3.6 |
| 整個 tick 中斷 | TX 中斷模式 | 5.18 / 5.35 / 6.13 | 7.44 / 7.69 / 7.97 | µs | R8 ×1.4 |
| 整個 tick 中斷 | DMA 模式 | 9.85 / 10.09 / 10.69 | 7.93 / 8.01 / 8.16 | µs | H745 ×1.3 |
| UART 中斷本體 | TX 中斷模式，約 14 萬次/秒 | — / 207 / 981 | — / 214 / 550 | ns | 平均相同 |
| DMA 完成中斷本體 | 每包一次 | — / 54 / 236 | — / 124 / 208 | ns | R8 ×2.3 |
| 中斷本體 CPU 負載 | TX 中斷 / DMA 模式 | 3.48 / 1.86 | 3.75 / 2.83 | % | 見下方解讀 |

**這個條件下的鏈路完整性**：R8 側 RX（模擬路徑）55,292 包 0 錯、送出遙測 30,000 包 0 跳過；H745 側 RX 54,002 包 0 錯、ORE 0，PC 收到遙測 30,000 包 0 錯、序號無缺。

**這些數字怎麼讀：**

1. **中斷反應 H745 快約 7 倍、穩約 15 倍。** M7 的 NVIC 硬體入棧約 35 ns（約 14 週期）；R8 的中斷要經 GIC、FreeRTOS `IRQ_Handler` 與 FSP 分派表，扣掉讀計數器後約 400 ns。這是進入延遲與抖動兩列背後的結構性原因，調應用程式是調不掉的。
2. **每個中斷做的事兩邊差不多快**（UART 中斷本體都約 0.2 µs），但 CPU 負載那一列只算中斷本體。R8 每次中斷多出約 0.4 µs 以上的進出開銷，在每秒 14 萬次中斷下**估計再多約 6%**；H745 約 0.8%。這個 6% 是由進入延遲推算的估計值、未含退出開銷，所以在 R8 上做中斷密集的設計時要多留餘裕。
3. **R8 的 DMA 啟動較慢**（6.95 對 1.95 µs），主因是經 FSP 驅動每包重設通道，加上 TE/TIST 交握；改為直接寫暫存器可省下約 5 µs。
4. 兩邊在 921600 全雙工 + 1 kHz 控制下都是**零溢位、零錯包**，皆可勝任常規飛控的 UART 需求（RC 輸入、遙測、GPS）。

> ⚠️ **注意**：**情境**——你在 R8 上經 FSP 驅動用 DMA 送 UART TX，期待 DMA 比中斷式 TX 更省 CPU。**症狀**——「遙測打包 + 啟動 TX」從 2.27 µs 膨脹到 6.95 µs，整個 tick 中斷從 5.35 µs 變 10.09 µs，反而輸給同模式的 H745（1.95 µs／8.01 µs）。**原因**——`R_DMAC_B_Reconfigure` 每包都重設通道，TE/TIST 交握也每包都付一次。**預防／處理**——通道只設定一次，每包改用直接寫暫存器，每個 tick 可省回約 5 µs。

**選核時這代表什麼：**

- **算力**：R8 在同樣的 float 控制碼上每週期效率與 M7 相當，靠約 2 倍時脈快約 2 倍；雙精度運算差距擴大到 2.7–4.8 倍。
- **即時性**：H745 的中斷延遲與週期抖動都小一個量級，且記憶體子系統（晶片內 SRAM）讓超出 L1 的資料搬移快 4–6 倍；R8 的主記憶體是與 A55 共用的 DDR。
- **選型建議**：常規串級 PID 飛控對兩者都只是 0.2–0.5% 的負載，算力不是瓶頸；若重視時序確定性與中斷反應，H745 較有利；若要跑雙精度或較重的運算，R8 明顯較強。

**這些參考值的邊界（引用前先看）：**

- **R8 的 UART RX 數字來自模擬路徑**——腳位在本板上未接線——TX 端也沒有接收端可驗證。H745 的 UART 數字則是對真實 PC 的端到端數值。
- H745 在出廠 SMPS 供電下只能到 VOS1（400 MHz）；改 LDO + VOS0 可到 480 MHz（約 +19%），需改板，所以這裡列的是 400 MHz 級的數字。
- R8 在 FreeRTOS 下執行，背景 tick 與 rpmsg 中斷仍在；H745 為無中斷裸機——比較「最大值」時對 R8 略不利，對平均值影響可忽略。
- R8 的數值成立於 A55 Linux 端閒置的前提下；A55 重負載時的 DDR 爭用，是這些數字沒有涵蓋的另一個變因。
- CoreMark 為自行移植（非 EEMBC 認證提交），但參數與 CRC 皆符合 performance run 規則。

#### Cortex-M33：低功耗常開的系統管理員

**它是什麼。** 一顆 Arm Cortex-M33，200 MHz，Armv8-M 架構，帶 FPU、DSP 延伸、TrustZone-M 安全延伸，還有 60KB 的 CoreSight ETF 程式流程追蹤緩衝與 JTAG/SWD（出處 07-hardware-unit-usage-guide.md:47、datasheet.md:165-172）。它的角色是「低功耗常開監督者」——在主核睡眠時也能維持系統管理、電源序列、喚醒判斷這類輕量長守的工作。

**板上現況：透過 remoteproc 曝露，但預設離線、韌體缺。** 跟 R8 不同，M33 在 Linux 裡**看得到**：`/sys/class/remoteproc/remoteproc0` 就是它，名稱 `cm33`，預設狀態 `offline`，韌體項標成 `rproc-cm33-fw`（✅ 2026-07-17 板上重執行：三個值逐字一致；transcript：live/ch04-cpu-periph.txt〔name／state〕、live/ch04-followup.txt〔firmware〕）（實際檔名 `rzv2h_cm33_rpmsg_linux-rtos_example.elf`）。它用的保留記憶體是 rsctbl@0x42f00000、vring@0x43000000、vdev0buffer@0x43200000（出處 07-hardware-unit-usage-guide.md:49-50）。

> ⚠️ **注意**：**情境**——你想照著文件把 CM33（remoteproc0）啟動起來，執行 `echo start > /sys/class/remoteproc/remoteproc0/state`。**症狀**——啟動失敗，因為韌體檔案根本不存在。**原因**——出貨映像檔沒有內建這個 `.elf` 韌體。**預防／處理**——需要先下載 Renesas Multi-OS Package（文件編號 `R01QS0077`），把韌體放到 `/lib/firmware/` 底下，並修改裝置樹（device tree）（出處 04-hardware-quickref.md:171,173）。在你完成這些之前，remoteproc0 停在 `offline` 是**預期狀態**，不是壞掉。

**要往下走時，官方文件講明的邊界與鎖版條件（RDK 官方文件 v1.1.1，第 3 章〈RZ/V Multi-OS〉）。** 上面那則注意只講到「要韌體、要改裝置樹」，官方另外把幾條會卡住你的前提寫死了，動手前先對一遍：

| 項目 | 官方值 | 為什麼要注意 |
|---|---|---|
| Multi-OS Package | **v3.2** | 韌體與範例的來源；上面那則注意寫的 `R01QS0077` 是文件編號 |
| RZ/V **FSP** | **v3.1** | 建置範例韌體用；**別裝 RA 家族的 FSP**（不同產品線、版本號體系不同） |
| Segger **J-Link 韌體** | **7.96e**（官方明指這個版本） | 這類工具鏈的版本相容性通常卡得很死，裝錯版本會在連線階段就失敗 |
| JTAG 除錯 | 需把 **DSW1 的 SW6 撥到 ON** | 對應第 1 章 1.2 節 DIP 表的 `MD_BOOT3`＝Debug；平時跑 Linux 應為 OFF |

**預設 IPL 支援到哪裡（這條最容易誤判）。** 官方原話：「On our default IPL, the following features are enabled by default: **Remoteproc support and CM33 and CR8 invocation from U-Boot.**」——也就是說**預設就支援 remoteproc、以及從 U-Boot 呼叫 CM33／CR8**。至於更進一步的功能，官方另寫：「If you want to use other features of multi-OS, such as **CM33 cold boot** or CA55 1.8 GHz support, feel free to contact us for support at renesas-rdk.」

> ⚠️ **注意（別把「CM33 cold boot 要另外聯絡」誤讀成「CM33 不能用」）**：官方那句話限定的是 **cold boot**（讓 CM33 先於 Linux 開機）這條特定路徑，**不是**說 CM33 一律不可用——預設 IPL 明確支援從 U-Boot 呼叫它。而「從已經在跑的 Linux 用 remoteproc 啟動 CM33」是**第三種情境**，官方那句話並未涵蓋，本手冊也未在本板驗證成功。**規劃時請把這三條路分開看**：① 預設可用＝remoteproc 框架在、從 U-Boot 呼叫；② 需另外聯絡＝cold boot；③ 未有定論＝從執行中的 Linux 啟動。另外那個「聯絡」是 GitHub 上 `renesas-rdk` 專案的支援邀請，不是正式的 Renesas 支援窗口。

> 💡 **提示（一個容易被 Module Standby 咬到的坑）**：官方指出，**沒有被明確啟用的周邊，在 Linux 開機後會進入 Module Standby 模式**；為了避免 CM33 要用的周邊被 Linux 關掉，Multi-OS Package 會 patch `drivers/clk/renesas/r9a09g057-cpg.c`，把 GTM 的時脈項從 `DEF_MOD` 改成 **`DEF_MOD_CRITICAL`**（標記為關鍵時脈、不許被關）。**這個機制的意義超出 CM33 本身**：只要你打算讓副核獨立操作某個周邊，都要想到「Linux 這一側會不會把它的時脈關掉」。官方也給了兩種在 Linux 側停用周邊的做法：修改來源 dts 把 `status` 從 `"okay"` 改成 `"disabled"`，或直接把 `/boot/uEnv.txt` 裡對應的 overlay 那行註解掉（overlay 機制見 [00-總覽與IP啟用地圖](00-overview-and-ip-enablement-map.md)）。

#### Mali-G31 GPU：並行卸載與零拷貝，不是 AI 主力

**它是什麼。** 一顆 Arm Mali-G31（Bifrost 架構，arch 7.0.9 r0p0），時脈 630 MHz。規格上**只有 1 個 shader core / 1 個 OpenCL compute unit**，L2 快取 8KB，本地記憶體 32KB（型別為 Global，G31 沒有專屬 local memory）。它跟系統共用同一塊 RAM（統一記憶體 14.84 GiB），所以能跟相機、編碼器做**零拷貝**協作（出處 08-gpu-deep-dive.md:23-28、06_gpu.txt:9）。

**它能做什麼、不能做什麼。** 這是最常被誤會的一顆核心，所以講清楚定位（出處 08-gpu-deep-dive.md:12-14、100-108）：

- **它是可用的 OpenCL 3.0 GPGPU 計算裝置**，不只是拿來顯示——kernel 實跑通過，支援 fp16（約 2× fp32 吞吐）、SPIR-V、以及最有價值的 `cl_arm_import_memory_dma_buf` 零拷貝（出處 08-gpu-deep-dive.md:37-41）。
- **適合**：逐像素、可大規模並行的影像/CV 運算（色彩轉換、resize、warp、濾波、二值化、光流、特徵點前處理）；在 A55 忙於控制/協定時把影像前處理卸下；圖形/HMI/OSD 疊加（GLES 3.2）；少數 DRP-AI3 不支援的 NN 運算元用 fp16 補跑。
- **不適合**：當通用浮點加速器（後面會看到它 raw FLOPS 比一顆 A55 還少）；當 AI 推論主力（那是 DRP-AI3 的活）；大型矩陣/重計算（1 個 CU + 32KB local mem 的硬限制）。

> ⚠️ **注意**：**情境**——你想確認 Vulkan 能不能用。**症狀／現況**——libmali 裡確實**內含 Vulkan 字串**，但板上**沒有** `vulkaninfo`、也沒有確認過 ICD。**原因**——只有字串存在不等於能力可用。**預防**——在你實際用 `vulkaninfo` 或載入 Mali Vulkan ICD 驗證通過之前，**不要把 Vulkan 當成本板已確認的能力**（出處 08-gpu-deep-dive.md:42）。同理，OpenCL 版本兩份來源也不一致：clinfo 實測是 **3.0**，datasheet 標的是 **2.0**（出處 08-gpu-deep-dive.md:37、datasheet.md:197-201）——以你板上 `clinfo` 實際印出的為準。

#### DRP-AI3 NPU：作為運算引擎的一面

DRP-AI3 是感知章的主角，這裡只講它「作為一個運算引擎」的規格與吞吐面向，讓你在做「算力該擺哪」的決策時有數字可用。

**它是什麼。** AI 加速器 = DRP0（96 個 processing element，可重組）+ AI-MAC（4096 個 INT8 MAC〔乘積累加單元〕，3MB 本地 SRAM：1MB 權重 + 2MB 特徵，支援 sparse / N:M 剪枝）。峰值 **8 dense TOPS / 80 sparse TOPS**（TOPS＝tera operations per second，每秒兆〔10¹²〕次運算；此處是 INT8 定點的峰值，不是浮點 FLOPS），能效約 **10 TOPS/W**（出處 07-hardware-unit-usage-guide.md:61、09-compute-capability.md:51、datasheet.md:195-196）。裝置節點 `/dev/drpai0`（drpai-rz 驅動程式 1.20 rel.3 V2H），配一塊 **512MB DDR carveout @0x240000000** 當主記憶體（出處 07-hardware-unit-usage-guide.md:63、00_inventory.txt:135-139）。

它為什麼是這塊板子的「感知決定性優勢」，等到後面的推論 benchmark 你就會用數字看清楚：它能把 YOLOX 從 CPU 的 145 ms 壓到 15 ms，同時把四顆 A55 完全空出來跑其他運算——例如飛行控制與通訊協定（出處 05-compute-benchmark.md:147）。

### 動手：先確認這幾顆核心都在、狀態正確（各步標記見下）

在跑任何 benchmark 前，先花兩分鐘把「我面對的確實是這幾顆核心、頻率鎖對了」確認一遍。這一段每條指令都附預期輸出，輸出逐字來自實測證據。

**✅ 步驟 1（2026-07-17 板上重執行）：確認四顆 A55 與時脈。**

```bash
lscpu | grep -E 'Model name|CPU max|CPU min|Core\(s\)'
```

預期輸出（逐字，✅ 2026-07-17 板上重執行，transcript：live/ch04-followup.txt；另見 00_inventory.txt:29-42）：

```text
Model name:                           Cortex-A55
Core(s) per cluster:                  4
CPU max MHz:                          1700.0000
CPU min MHz:                          212.5000
```

**✅ 步驟 2（2026-07-17 板上重執行，transcript：live/ch04-cpu-periph.txt）：確認頻率鎖在 1.7 GHz、governor 是 performance。** 這一步很關鍵——所有實測數據都在這個設定下量得，你複現時若沒鎖 performance，數字會因為動態調頻而對不上。

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

預期輸出（出處 07-hardware-unit-usage-guide.md:178-179）：

```text
1700000
performance
```

（`scaling_cur_freq` 單位是 kHz，1700000 kHz = 1.7 GHz。）

**✅ 步驟 3（2026-07-17 板上重執行，transcript：live/ch04-followup.txt）：確認指令集延伸——有 dotprod/fp16，沒有 aes/sha。** 這決定了你能不能指望硬體加密。

```bash
grep -o 'asimddp\|fphp\|aes\|sha2' /proc/cpuinfo | sort -u
```

預期輸出（出處 07-hardware-unit-usage-guide.md:27、00_inventory.txt:43）：只會印出 `asimddp` 與 `fphp` 兩行，**不會**出現 `aes` 或 `sha2`。若你要看完整 flags，直接看 `/proc/cpuinfo` 的 Features 行，實測為：

```text
fp asimd evtstrm crc32 atomics fphp asimdhp cpuid asimdrdm lrcpc dcpop asimddp
```

**✅ 步驟 4（2026-07-17 板上重執行，transcript：live/ch04-cpu-periph.txt）：確認 M33 的 remoteproc 狀態（預設離線是正常的）。**

```bash
cat /sys/class/remoteproc/remoteproc0/name
cat /sys/class/remoteproc/remoteproc0/state
```

預期輸出（出處 07-hardware-unit-usage-guide.md:238）：

```text
cm33
offline
```

看到 `offline` 不要慌——如前所述，這是韌體未載入時的預期狀態。

**✅ 步驟 5（2026-07-17 板上重執行，transcript：live/ch04-inventory.txt）：確認加速器裝置節點都在。**

```bash
ls -l /dev/drpai0 /dev/mali0 /dev/drp1 /dev/media0 /dev/video0 /dev/dri/card0
```

預期：六個節點都存在。它們的主/次裝置號實測為 `/dev/dri/card0`(226,0)、`/dev/drpai0`(511,0)、`/dev/mali0`(10,125)、`/dev/media0`(250,0)、`/dev/video0`(81,3)、`/dev/drp1`(234,1)（出處 00_inventory.txt:119-127；✅ 2026-07-17 板上重執行，六個主／次號全部一致，transcript：live/ch04-inventory.txt）。想進一步確認驅動程式版本，看開機訊息：

```bash
dmesg | grep -iE 'DRP-AI Driver|DRP Driver|Probed as mali'
```

預期輸出（逐字，出處 00_inventory.txt:131,135,149）：

```text
drp-rz 18000000.drp1: DRP Driver version : 1.00 rel.3 V2H
drpai-rz 17000000.drpai: DRP-AI Driver version : 1.20 rel.3 V2H
mali ... Probed as mali0
```

（開機一陣子後，dmesg 環形緩衝區可能已把這些行擠掉——見 4.1 的注意框；`journalctl -k -b` 實測撈得回 `mali 14850000.gpu: Probed as mali0` 這行（✅ 2026-07-18，transcript：live/ch04b-runtime.txt），DRP／DRP-AI 版本行則不一定還在。查不到不代表驅動程式沒載，看 `/dev/drpai0`／`/dev/drp1` 在不在即可。）

### Benchmark 實測數據（每一項都附量測條件）

> **共同量測基準（除非個別另註，以下所有數據都適用）**：2026-06-21，於板上 `ubuntu@<板子IP>`，kernel `6.10.14-arm64-renesas`（SMP **PREEMPT**，非 PREEMPT_RT），Ubuntu 24.04.4 LTS aarch64，CPU governor 鎖 `performance`（全程 1.7 GHz，量測無調頻干擾）。紀錄檔在 `../../assets/compute-benchmark-20260621/results/`，可複現（出處 05-compute-benchmark.md:5-10）。
> 本段各 benchmark 屬重負載，2026-07 複查時板上有現役服務執行中，**未重跑**，維持 2026-06-21 實錄（📼）。

這一段給你的每個數字，都會標清楚「用什麼工具、什麼條件量的、原始檔在哪」。**數據是拿來查證的，不是拿來信仰的**——所以出處寫得很細，你可以逐條回頭核對。

#### 整數運算：CoreMark、sysbench、7-zip

CoreMark 是嵌入式 CPU 的經典整數綜合基準，最能反映「一般程式碼」的單核與多核表現。

| 指標 | 單執行緒 | 4 執行緒 | 多核擴展 | 出處 |
|---|---|---|---|---|
| CoreMark | **6852** | **26511** | **3.87×** | 05-compute-benchmark.md:60 |
| sysbench cpu(prime 20000) | 332.3 ev/s | 1306.4 ev/s | 3.93× | 05-compute-benchmark.md:61 |
| 7-zip 綜合 MIPS | 1414 | 4925 | — | 05-compute-benchmark.md:62 |

**這些數字告訴你什麼。** 3.87×（CoreMark）與 3.93×（sysbench）的多核擴展**接近理論上限 4×**，代表四顆核心的工作分配與記憶體競爭都處理得好——你把工作切成四份丟給四顆核，幾乎能拿到四倍吞吐，不會被匯流排卡死（出處 05-compute-benchmark.md:78）。單核 CoreMark 6852 換算約 4.03 CoreMark/MHz，屬 A55 的正常水準。

**📼 動手複現 CoreMark（含一個一定會踩到的坑）。** CoreMark 是 EEMBC 釋出的開源基準，板上不會預先附帶——你要**先取得原始碼、進到它的目錄**，才有下面那個 Makefile 可用。在你的家目錄先 clone 再 `cd` 進去（本手冊的量測就是在 CoreMark 原始碼目錄裡跑的，見紀錄檔開頭的建置路徑 `.../src/coremark`）：

```bash
git clone https://github.com/eembc/coremark
cd coremark
```

進到 `coremark/` 目錄後，再建置並執行單執行緒 performance run：

```bash
make XCFLAGS="-O2 -funroll-loops -DPERFORMANCE_RUN=1" load run1.log
```

（`make` 這行逐字出處 01c_coremark.txt:2）這會建置並跑單執行緒 performance run，結果寫進 `run1.log`。若你直接在家目錄打這行 make、卻沒有先 clone／`cd`，會得到 `No such file` 或 `No targets`——因為手上根本沒有那個 Makefile。預期在 log 裡看到（逐字，出處 01c_coremark.txt:79-85）：

```text
Total time (secs): 16.053000
Iterations/Sec   : 6852.301750
Iterations       : 110000
CoreMark 1.0 : 6852.301750 / GCC13.3.0 ... / Heap
```

4 執行緒版把 `Iterations/Sec` 拉到 `26510.815208`（出處 01c_coremark.txt:87-92）。

> ⚠️ **注意**：**情境**——你想直接採信量測腳本自動印出的 `DERIVED` 欄位（CoreMark/MHz、多核擴展倍率）。**症狀**——那一行印出來的是 `CoreMark/MHz(1T)=0.001 | scaling=1.00x` 這種明顯不合理的佔位數字。**原因**——腳本裡那段 awk 衍生運算式吃到空字串，把除法運算子誤判成正規表示式起始符號而出現語法錯誤（三輪 CPU log 的 DERIVED 欄位都中了同一個問題）。**預防／處理**——**不要採信 DERIVED 那一行**；正確的計算方式是拿 log 上方的原始 `Iterations/Sec` 手算：`6852.301750 ÷ 1700 ≈ 4.03 CoreMark/MHz`、`26510.815208 ÷ 6852.301750 ≈ 3.87×`。本手冊表格裡的 4.03 與 3.87× 就是這樣算出來的（出處 01c_coremark.txt:93、05-compute-benchmark.md:60）。

**stress-ng 各方法的細分（4 執行緒，bogo ops/s real）**，讓你看到不同運算型態的相對成本（出處 01b_cpu_fix.txt:11-23、05-compute-benchmark.md:63-65）：

| 方法 | bogo ops/s | 說明 |
|---|---|---|
| `int64` | 3271 | 整數最快 |
| `float` | 2543 | 單精度浮點 |
| `double` | 1218 | 雙精度約為單精度的一半速 |
| `fft` | 968 | 帶記憶體存取的變換 |
| `matrixprod` | 65 | 矩陣乘最重（快取壓力大） |

#### 浮點與線性代數：SGEMM、DGEMM、小矩陣

控制與估測演算法（EKF、MPC、ADRC）本質上是矩陣運算，所以浮點/線代吞吐是判斷「這板子能跑多複雜的估測器」的關鍵。這裡用 numpy + OpenBLAS 量測 GEMM（一般矩陣乘），單位 GFLOPS（每秒十億次浮點運算）。

| 指標 | 單核 | 4 核 | 對理論峰值 | 出處 |
|---|---|---|---|---|
| SGEMM（fp32） | 7.9 GFLOPS | **28 GFLOPS** | 約 51%（峰值 54.4） | 05-compute-benchmark.md:88、02_fp.txt:5-8 |
| DGEMM（fp64） | 3.1 GFLOPS | 10.8 GFLOPS | 約 40% | 05-compute-benchmark.md:89、02_fp.txt:9-11 |

**兩個要點。** 其一，**fp64 大約是 fp32 的 2.5 倍成本**（3.1 對 7.9 GFLOPS）——這在選擇估測器精度時要記得：能用 fp32 就別無腦上 fp64（出處 09-compute-capability.md:17）。其二，能拿到理論峰值的 40–51% 已經是大型稠密矩陣的漸近好成績；等下你會看到，控制與估測常見的**小**矩陣拿不到這個效率。

> 💡 **提示**：4 核 SGEMM 這個數字，主文件寫「28」、原始 log（N=2048）是「28.04」，四捨五入到「27.8」也在別處出現過（出處 02_fp.txt:5-8、05-compute-benchmark.md:88）。它們是同一筆量測的不同取整，差異在 1% 內、不改變任何結論。本手冊統一用主文件的「28 GFLOPS（4 核）」作為聚合算力的代表值。

**小矩陣的真相——派送開銷才是瓶頸。** 量測 12×12 的 GEMM，得到 59457 ops/s，換算每次 16.82 µs（d08 概記為 ~16.9 µs，同一筆量測的取整；出處 02_fp.txt:25、05-compute-benchmark.md:90）。但這 16.82 µs 裡，**約 15.8 µs 是 Python/numpy 的派送（dispatch）開銷，真正在算的只有約 1.1 µs**（出處 09-compute-capability.md:15）。這件事對你怎麼寫控制律影響巨大：

- 如果用**原生 C/Eigen**，單一個 12 態的矩陣運算就是 µs 等級，A55 跑 1 kHz 控制迴圈的線代量綽綽有餘（出處 05-compute-benchmark.md:92-94）。
- 如果用 **Python/numpy**，你會被那 15.8 µs 的固定派送稅金綁死，而且這稅金**跟矩陣多小無關**——後面「能力上限」會把這條量化成「Python EKF 無論狀態維度多小都卡在低千 Hz」。

#### 記憶體頻寬與延遲：STREAM、tinymembench

| 指標 | 值 | 出處 |
|---|---|---|
| STREAM Triad（4 緒） | 5585 MB/s（Copy 4456 / Scale 5836 / Add 5632） | 05-compute-benchmark.md:102、03_mem.txt:35-39 |
| tinymembench NEON LDP/STP copy | 3463 MB/s | 05-compute-benchmark.md:103、03b_tinymembench.txt:34 |
| standard memcpy / memset | 3037 / 5761 MB/s | 05-compute-benchmark.md:104 |
| mbw MEMCPY / MCBLOCK | 3295 / 4429 MiB/s | 05-compute-benchmark.md:105、03_mem.txt:4-21 |

**記憶體規格與一個重要的「為什麼」。** 板上是 16 GB LPDDR4（1600 MHz＝3200 MT/s，8 GB×2，2×32-bit，理論峰值 25.6 GB/s；規格總表寫 LPDDR4，板卡方塊圖標 LPDDR4X-3200，SoC 控制器兩者皆支援，見第 01 章記憶體全景），無 swap；`meminfo` 實測 `MemTotal 15565904 kB`、`MemAvailable 14863188 kB`（出處 00_inventory.txt:90-97）。注意：**CPU 端能拉到的頻寬（STREAM 5.6 GB/s）只有記憶體控制器峰值的約 22%**（出處 05-compute-benchmark.md:107-109）。

這個 22% 不是故障，是 A55 的本性——它是 in-order 的小核心，記憶體層級平行度（memory-level parallelism, MLP）有限，沒辦法同時掛很多筆未完成的記憶體請求把頻寬填滿。真正需要高頻寬的影像/張量搬移，是靠 DRP-AI3 / DRP 的專屬 DMA 路徑達成的，**不經過 CPU**（出處 05-compute-benchmark.md:109-111）。所以你不必為這個數字失望：16 GB 容量對多模型 + 緩衝 + ROS 綽綽有餘，而真正吃頻寬的活有專用硬體去扛。

> ⚠️ **注意**：DRAM 隨機讀延遲這個數字，來源之間有個對齊問題，引用時要小心。主文件寫「約 196 ns（64 MB 工作集）／雙路約 226 ns」，但 tinymembench 的原始逐行輸出裡，196.0 / 226.2 ns 其實對應的是 **16 MB** 那一列；真正的 **64 MB** 列是 208.7 / 233.5 ns（出處 05-compute-benchmark.md:106、03b_tinymembench.txt:82,84）。L2（≤1 MB 工作集）的延遲約 15–19 ns。**症狀**——你若照主文件把「196 ns」貼上「64 MB」標籤，其實張冠李戴。**預防**——要引用確切延遲時，回頭對 tinymembench 原始輸出的**工作集大小欄位**；記憶體綁死的估算（後面 A* 那類）用「約 196–210 ns 等級的隨機延遲」這個量級就夠，不必糾結是掛在 16 MB 還是 64 MB。

#### 儲存 I/O：這是一張消費級 SD 卡

| 指標 | 值 | 出處 |
|---|---|---|
| 序列寫（dd buffered+fdatasync, 1GiB） | 28.2 MB/s | 05-compute-benchmark.md:121、04_storage.txt:6 |
| 序列讀（dd, drop_caches） | 61.9 MB/s | 05-compute-benchmark.md:122 |
| hdparm 緩衝讀 / 快取讀 | 71.0 / 1141 MB/s | 05-compute-benchmark.md:123,126 |
| fio 1M 序列讀/寫 | 76.4 / 32.4 MB/s | 05-compute-benchmark.md:124 |
| fio 4K 隨機讀/寫 | 4353 / 1491 IOPS（17/6 MB/s），延遲 3.4/10.7 ms | 05-compute-benchmark.md:125 |

**開機儲存的真相。** rootfs 掛在 `/dev/mmcblk0`，這其實是一張 **256 GB 的 Samsung SD 卡**（type SD、manfid 0x1b、oemid "SM"、製造 2025/10），ext4 格式，可用約 206 GB（出處 05-compute-benchmark.md:48、04b_emmc_id.txt:2-8）。速度屬 UHS-I SD 卡等級，不是 eMMC 5.1 的 150–300 MB/s。以無人機這類負載為例——4 Mbps 視訊約 0.5 MB/s、加上遙測記錄——這速度綽綽有餘（出處 05-compute-benchmark.md:128-131）。

> ⚠️ **注意**：**情境**——你想確認開機儲存是什麼裝置，或想把它拿來當高可靠的資料紀錄裝置（以飛行黑盒子為例）。**症狀**——`04_storage.txt` 的檔頭把它標成「eMMC」，但卡片身分回報是 **SD 卡**（腳本標籤與實測裝置身分不符，實為 SD）（出處 05-compute-benchmark.md:191、04_storage.txt:1）。**原因**——出貨採用外接 SD 卡作為開機媒體，而消費級 SD 卡的**寫入耐久性與斷電可靠性**對這類高可靠資料紀錄是實實在在的風險。**預防／處理**——這類高可靠、斷電安全的資料紀錄需求，應改用板上 eMMC 或工業級 pSLC 卡，並把高頻 log 與 rootfs 分離到不同儲存裝置（出處 05-compute-benchmark.md:196-198）。

> ⚠️ **注意**：**情境**——你用 fio 加 `O_DIRECT` 對這張 SD 卡做 4K 隨機讀寫。**症狀**——這條路徑異常緩慢，只有 18 / 5.6 MB/s，提交延遲高達 55–180 ms。**原因**——來源只陳述了現象、沒說明根因，本手冊不代為杜撰原因。**預防／處理**——改用 buffered + fdatasync 量測，較能反映實際檔案系統吞吐（上表的 28.2 MB/s 寫就是這樣量的）（出處 05-compute-benchmark.md:128-129、04_storage.txt:2-3）。

#### 加密吞吐：沒有硬體加速，只能軟體跑

前面一再提到 A55 沒有 aes/sha 延伸，這裡給你確切代價（軟體實作，出處 05-compute-benchmark.md:71-76）：

| 演算法 | 單核 | 4 核 | 出處 |
|---|---|---|---|
| AES-256-GCM | 34.7 MB/s | 137.6 MB/s | 05-compute-benchmark.md:71 |
| AES-256-CBC | 56.5 MB/s | — | 05-compute-benchmark.md:72 |
| SHA-256 | 111 MB/s | 441 MB/s | 05-compute-benchmark.md:73 |
| SHA-512 | 174 MB/s | — | 05-compute-benchmark.md:74 |
| RSA-2048 sign/verify | 156 / 5809 ops/s | — | 05-compute-benchmark.md:75 |
| ECDSA P-256 sign/verify | 6284 / 2333 ops/s | — | 05-compute-benchmark.md:76 |

> ⚠️ **注意**（工程背景，非除錯事件）：**情境**——你需要加密資料傳輸或儲存。**症狀**——A55 缺 ARMv8 Crypto Extensions，AES/SHA 只能走軟體，吞吐被壓在數十 MB/s。**原因**——CPU flags 裡沒有 `aes`/`sha2`。**預防／處理**——4 Mbps 視訊串流的加密（約需 0.5 MB/s）綽綽有餘；但若有大量加密儲存/傳輸需求，要留意頻寬上限（單核 AES 約 35–56 MB/s、4 核約 137 MB/s）（出處 05-compute-benchmark.md:79-80）。做安全設計時另記一條：這顆 H44 在 Linux 上摸不到硬體 Security IP、也沒有 `/dev/tee`（TEE＝可信執行環境，如 OP-TEE）（本料號 Security＝N/A、未搭載——真硬體手冊 Table 1.1-1〔p78〕核實，見 g8），所以**不要假設能做大流量加密或硬體金鑰隔離**；這吞吐足夠做資料簽章（出處 09-compute-capability.md:44）。

#### DRP-AI3 NPU 推論：把 145 ms 壓到 15 ms

**量測設定：** 模型 YOLOX-nano@416（DRP-AI TVM 編譯、INT8 量化），351 次取樣，執行設定 `DRP0_max_freq_factor=2`、`AI-MAC_freq_factor=2`，相機影像 1280×720 UYVY（出處 05-compute-benchmark.md:137-138、05_npu_drpai.txt:1,4）。

| 階段 | 時間 | 出處 |
|---|---|---|
| 純推論（DRP-AI3, INT8） | mean 15.26 / p50 15.00 / p95 16.20 / max 27.10 ms | 05-compute-benchmark.md:142、05_npu_drpai.txt:45 |
| 前處理 | 6.86 ms（DRP 加速） | 05-compute-benchmark.md:143 |
| 後處理 | 1.62 ms（部分跑 A55） | 05-compute-benchmark.md:143 |
| **端到端** | **23.7 ms → 42.1 FPS** | 05-compute-benchmark.md:144 |
| CPU 對照（onnxruntime fp32, 4 緒） | 145.6 ms（6.9 FPS） | 05-compute-benchmark.md:145 |
| CPU 對照（1 緒） | 422 ms（2.4 FPS） | 05-compute-benchmark.md:145 |

**加速比與它的真正意義。** DRP-AI3 對 CPU 4 緒快 **9.7×**、對 1 緒快 **28.1×**（兩者皆以純推論 p50 15.0 ms 為除數：145.6÷15.0、422÷15.0；出處 05b_npu_cpu.txt、05_npu_drpai.txt:45）。但比「快幾倍」更重要的是：它把 YOLOX 推論從吃掉四顆 A55 的 145 ms，變成一顆專用 NPU 的 15 ms，**四顆 A55 因此完全空出來跑控制與通訊協定（以飛行控制為例）**——這才是「邊飛邊偵測/追蹤」能成立的關鍵（出處 05-compute-benchmark.md:147）。

> 💡 **提示**：純推論的「每秒推論次數」在兩份來源有兩個值——主文件寫 **66.7 inf/s**（= 1000 ÷ p50 的 15.0 ms），原始 log 印的是 **65.5 inf/s**（= 1000 ÷ mean 的 15.26 ms）（出處 05-compute-benchmark.md:142、05_npu_drpai.txt:50）。兩者都對，只是**除數用了 p50 還是 mean 的差別**。引用時講清楚你用的是哪個中位數口徑即可，別把兩個數字混在一起當成矛盾。

**一個反直覺但重要的結論：YOLOX-nano 根本沒吃滿 NPU。** 在 15.3 ms 下，YOLOX-nano 的有效算力只有約 65 GFLOP/s——8 dense TOPS（= 8000 GOP/s；GOP/s＝giga operations per second，每秒十億次運算）**用不到 1%**（出處 05-compute-benchmark.md:148、09-compute-capability.md:53）。這證明極小模型是被**固定開銷/記憶體頻寬綁死**，而不是被運算綁死。實務意義：你還有大把餘裕上更大/更準的模型、或做剪枝後的模型、或多個模型併行——NPU 的算力遠沒被榨乾。

#### GPU（Mali-G31）計算吞吐

用 `gpu_cl_bench.c`（OpenCL，用 vec4 + 4 個獨立累加器暴露指令級平行度 ILP）實測，逐字輸出（出處 08-gpu-deep-dive.md:51-53；紀錄檔 `../../assets/hardware-investigation-20260621/results/17_gpu_opencl.txt`）：

```text
Device: Mali-G31 r0p0 | 1 CU @ 630 MHz
vadd correctness: PASS (C[12345]=9258.750 expect 9258.750)
fma_bench(vec4,ILP): 37239.56 ms, 167772 MFLOP -> 4.51 GFLOPS (fp32 peak)
```

**這 4.51 GFLOPS 怎麼讀。** 它已經是這顆單 CU Bifrost 核心在 630 MHz 下理論峰值（4 lane × 2 × 630M ≈ 5 GFLOPS）的**約 89%**——幾乎滿載了（出處 08-gpu-deep-dive.md:63）。fp16 大約 2×（約 9 GFLOPS）。把它放進運算單元的排名裡（出處 08-gpu-deep-dive.md:58-61）：

| 單元 | fp32 GFLOPS | 相對 GPU |
|---|---|---|
| Mali-G31（實測） | 4.51 | 1× |
| 單顆 A55（實測 SGEMM） | 7.9 | 1.75× |
| 4×A55（實測 SGEMM） | 28 | 6.2× |
| DRP-AI3（INT8） | 8 TOPS | 不可比（單位不同） |

也就是說——**這顆 GPU 的 raw FLOPS 比一顆 A55 還少、約是四核 CPU 的 1/6**。它的價值不在「比 CPU 快」，而在**並行卸載**（把逐像素的活從 CPU 挪開）與**零拷貝**（直接吃相機/編碼器的 dma_buf）（出處 08-gpu-deep-dive.md:65）。

**動手啟用 OpenCL**（安裝步驟 📼 依實錄；查驗指令 ✅ 2026-07-17 板上重執行，transcript：live/ch04-followup.txt〔clinfo〕、live/ch04-cpu-periph.txt〔eglinfo〕；出處 08-gpu-deep-dive.md:73-75）：

```bash
sudo apt install -y ocl-icd-opencl-dev opencl-headers clinfo
echo /usr/lib/aarch64-linux-gnu/libmali.so | sudo tee /etc/OpenCL/vendors/mali.icd
clinfo | grep -iE 'Platform|Device Name|OpenCL'
```

前兩行把 libmali.so 註冊成 OpenCL ICD 供應商（安裝屬改動性操作，複查未重執行；板上已裝好），第三行驗證是否生效。預期會列出 `ARM Platform / Mali-G31 r0p0 / OpenCL 3.0`（✅ 2026-07-17 板上重執行 `clinfo`，transcript：live/ch04-followup.txt：逐字看到 `Platform Name  ARM Platform`、`Device Name  Mali-G31 r0p0`、`Platform Version  OpenCL 3.0 v1.r54p1-…`；另見 08-gpu-deep-dive.md:75、07-hardware-unit-usage-guide.md:97）。確認渲染器則用 `eglinfo | grep -i renderer`，預期印出 `OpenGL ES profile renderer: Mali-G31`（✅ 2026-07-17 板上重執行，transcript：live/ch04-cpu-periph.txt）。

> ⚠️ **注意**：**情境**——你在無頭（headless）環境下想用 `glmark2-es2 --off-screen` 幫 GPU 跑個圖形分數。**症狀**——出現錯誤訊息 `Could not initialize canvas`，跑不起來。**原因**——（EGL、mesa、DRM node、KMS 這幾個名詞都是 Linux 圖形顯示堆疊的元件：EGL 是繪圖 API 對接視窗系統的介面層、mesa 是通用的開源圖形驅動程式、DRM／KMS 是核心裡管理顯示與繪圖裝置的子系統。）通用 EGL 落到 mesa、接不上 `mali_kbase` 核心模組；板上唯一的 DRM node 是顯示控制器，沒有 GPU render node。**預防／處理**——需要 Wayland compositor 或 KMS surface 才能跑圖形分數，這是無頭封裝的限制、不是硬體問題。要注意的是：**OpenCL compute 路徑不受這個限制**，前面那 4.51 GFLOPS 就是在無頭下用 OpenCL 正常跑出來的（出處 05-compute-benchmark.md:156-157、08-gpu-deep-dive.md:51）。

#### 即時排程延遲：cyclictest（即時控制最關鍵的一項）

對硬即時控制（以飛行控制為例）來說，「平均多快」遠不如「**最壞情況下多晚**」重要——錯過一個硬期限，受控對象就可能失穩。cyclictest 量的就是這個：排程喚醒的延遲分布。

| 情境 | 最小 | 平均 | **最差** | 出處 |
|---|---|---|---|---|
| Idle（P80, interval 250 µs, 全核, 30 s） | 14 µs | 18–19 µs | **347–420 µs**（取原始 log 四次量測全距 347–420；來源文件摘要記 359–420） | 05-compute-benchmark.md:166、07_realtime.txt:6-9 |
| 滿載（stress-ng cpu4+vm2, 30 s） | 13 µs | 21–29 µs | **593–926 µs** | 05-compute-benchmark.md:167、07_realtime.txt:12-15 |

**怎麼讀這張表。** 核心是 `6.10.14-arm64-renesas`，SMP **PREEMPT 但非 PREEMPT_RT**（出處 05-compute-benchmark.md:6）。次毫秒級的最差延遲，正是一般 PREEMPT（非 RT）核心的預期表現。翻成白話：

- 對 **≤400 Hz（週期 ≥2.5 ms）** 的外迴圈控制——充裕。最差 0.93 ms 的抖動相對 2.5 ms 的週期還有餘裕。
- 對 **1 kHz（週期 1 ms）** 的硬即時內迴圈——約 0.93 ms 的抖動就**偏緊**了，幾乎吃掉整個週期（出處 05-compute-benchmark.md:169-170）。

這正是為什麼前面說「硬即時內迴圈（以飛行控制為例）該放 R8」——不是 A55 算不動，是 Linux 這層的排程不確定性擋在那裡。

> ⚠️ **注意**：**情境**——你想在這個 systemd + `CONFIG_RT_GROUP_SCHED`（cgroup v2）的環境下，用 `chrt -f` 把行程設成 SCHED_FIFO 即時優先權（即使用 root）。**症狀**——`chrt -f` 回報 `EPERM`（權限不足）。**原因**——cgroup v2 的 RT_GROUP_SCHED 預設就擋下 sub-cgroup 的 RT 頻寬。**預防／處理**——量測時是靠暫時放行 RT 頻寬才量到上表數據：跑 cyclictest 前先 `sudo sh -c 'echo -1 > /proc/sys/kernel/sched_rt_runtime_us'`，量完再 `sudo sh -c 'echo 950000 > /proc/sys/kernel/sched_rt_runtime_us'` 還原成預設值（這只是量測期間的臨時設定，不是永久改動）；長期方案要重建 PREEMPT_RT 核心並調整 cgroup RT 頻寬設定（出處 05-compute-benchmark.md:171-172、07_realtime.txt:2-3）。這條坑很關鍵：你在 A55 上規劃任何 SCHED_FIFO 迴圈前，得先解決這個封鎖，否則優先權根本設不上去。

#### 散熱與降頻：無風扇、零降頻（兩份量測，條件不同）

| 來源 | 待機 | 滿載峰值 | 降頻 | 出處 |
|---|---|---|---|---|
| **d04**（2026-06-21，有時序 CSV） | 35 / 36 °C（tz0/tz1） | 5 分鐘全核 38 / 39 °C | 全程 1700 MHz，548/548 取樣零降頻 | 05-compute-benchmark.md:180-182、08_thermal.txt:8-10 |
| **d03**（未附量測日期/主機） | 約 35 °C（室溫 26 °C） | stress-ng 5 分鐘後約 50 °C；YOLOX 連續推論約 42 °C | 無降頻、無重啟 | 04-hardware-quickref.md:210-212 |

**兩份量測的峰值差很多，都說沒降頻——這要誠實並列，不能只挑一個講。** d04 的滿載峰值是 38–39 °C，d03 的 stress-ng 5 分鐘峰值卻到約 50 °C（出處 04-hardware-quickref.md:210-212、05-compute-benchmark.md:180-182）。差距來自量測條件不同（負載型態、環境、是否附時序），d04 有逐點 CSV 佐證、d03 未附日期/主機。共同結論是可靠的：**無風扇、零降頻，滿載只升溫幾度**，板上有金屬散熱片，目前無需主動散熱；這也呼應 DRP-AI3 白皮書「fanless 即可達競品含風扇等級」的訴求（出處 05-compute-benchmark.md:184-185、04-hardware-quickref.md:214）。

> ⚠️ **注意**：**情境**——你看到「滿載才 38 °C」就放心把散熱當非問題。**症狀（預期）**——裝進機殼後溫度會比裸板高不少。**原因**——目前所有散熱測試都在**裸板**（未裝機殼）狀態，機殼封裝會大幅限制氣流。**預防**——實際裝進密閉、無主動散熱的嵌入式環境（以機載為例）前一定要在封裝狀態下**重新量測**散熱，別拿裸板數字當定案（出處 04-hardware-quickref.md:214）。

### 能力上限推導：某演算法能跑到幾 Hz

benchmark 給的是「原始吞吐」，但你真正想知道的是「**我的 EKF/MPC/FFT 到底能跑幾 Hz**」。這一段就是把實測翻譯成這個答案。

**先講方法論，你才知道這些 Hz 數字能信到什麼程度**（出處 09-compute-capability.md:5,17）：

1. **錨點是實測，外推是估計。** 下面每個數字都錨定在 2026-06-21 的實測值上；凡標 **ESTIMATE** 的，是由實測外推的估計、附了假設，**不是直接量到的**。寫程式時請把「實測」和「ESTIMATE」分開對待。
2. **小矩陣打 40% 折扣。** 實測的 7.9/3.1 GFLOPS 是大型稠密矩陣的漸近值；控制與估測常見的小矩陣（N<50）大概只能實現其 30–50%，所以所有 O(N³) 核心的估算一律打 **40% 折扣**。
3. **這些 Hz 是「單一迴圈在一顆專用核心上的運算天花板」**，除非另註。實際可達的硬即時速率還要再扣排程/IO 負擔、並套用前面 cyclictest 的抖動上限。

> **共同錨點（皆 2026-06-21 實測）**：fp64 DGEMM 3.1（1核）/10.8（4核）、fp32 SGEMM 7.9（1核）/28（4核）GFLOPS、numpy 12×12 59k ops/s（16.9 µs，含 15.8 µs 派送）、cyclictest 負載 max 926 µs / avg 29 µs、CoreMark 6852/26511、7-zip 1414/4925 MIPS、STREAM 5.6 GB/s、memcpy 3.5 GB/s、DRAM 延遲 196 ns（出處 09-compute-capability.md:15）。

#### EKF（擴展卡爾曼濾波）能跑幾 Hz

EKF 是即時狀態估測的骨幹（以姿態、位置、INS/GNSS 融合為典型應用）。以下 native C、單核除非另註，皆 ESTIMATE（出處 09-compute-capability.md:23-27）：

| EKF 規模 | 單核天花板（fp64 / fp32） | 4 核 | 實務建議 |
|---|---|---|---|
| 6-state（四旋翼姿態/位置） | >100 kHz（約 7 µs/step） | — | 跑 200–1000 Hz 只用一核 <1% |
| 15-state（INS/GNSS error-state） | 約 90 kHz / 230 kHz | — | 輕鬆 200–400 Hz，<1% 核心 |
| 50-state | 約 2.5 kHz / 6.3 kHz | 約 8.6 kHz | 遠超常見即時控制 100–500 Hz 需求 |
| 100-state | 約 310 Hz / 790 Hz | 約 1.08 kHz | 仍可 100–300 Hz 即時 |

**這張表的白話結論**：對這類尺度（狀態維度多在數個到數十個之間），A55 跑 EKF 的**運算力大幅過剩**——你可以同時跑好幾個中小型 EKF 還剩一堆餘裕（出處 09-compute-capability.md:17）。真正會咬你的不是運算，是下面這條 Python 陷阱。

> ⚠️ **注意**：**情境**——你用 Python/numpy 寫高頻的估測或控制迴圈。**症狀**——頻率卡在低千 Hz 甚至更低，而且你把狀態維度改小也沒變快。**原因**——實測每次 numpy 呼叫約 16.9 µs（其中約 15.8 µs 是純派送開銷），這是**與矩陣尺寸無關的下限**；一個約 10 次 numpy 呼叫/step 的 Python EKF，最好情況也只到約 5900 Hz（20 次呼叫→約 2950 Hz；50 次→約 1180 Hz；若寫成逐元素的 Python 迴圈→掉到數十 Hz）。**預防／處理**——正式量產的估測/控制邏輯**必須用原生 C/C++**；Python 適合離線分析與原型，不適合高頻即時迴圈（出處 09-compute-capability.md:27,40）。

#### MPC（模型預測控制）能跑幾 Hz

精簡 MPC（interior-point，約 12 次迭代），native C fp64 單核，決策維度 nz = 輸入數 × horizon，皆 ESTIMATE（出處 09-compute-capability.md:29-30）：

| 決策維度 nz | 單核 fp64 | 備註 |
|---|---|---|
| 20（如 n=6, H=10, m=2） | 約 9.7 kHz | — |
| 40 | 約 1.2 kHz | — |
| 80（如 n=12, H=20, m=4） | 約 150 Hz | fp32 快約 2.5×、4 核快約 3.5× |
| 120（如 n=12, H=30, m=4） | 約 45 Hz | 已偏慢 |

**實務建議（state ≤12、horizon ≤20、inputs ≤4，熱啟動）**：單核可輕鬆達 100–200 Hz，還能留 3 顆核給別的活；真實求解器（OSQP/qpOASES）用熱啟動通常比上表的冷啟動稠密 IP 界線再快 **3–10×**，所以上表是**保守下限**（出處 09-compute-capability.md:29-30）。

> ⚠️ **注意**：**情境**——你想跑高更新率、長 horizon、多輸入的大型 MPC。**症狀**——更新率掉到 50 Hz 以下。**原因**——稠密精簡 MPC 的決策維度 m×H 超過約 120（例如 state 12、horizon 30、4 輸入）時，即使 fp32 4 核冷啟動的界線也低於 50 Hz。**預防**——需要 >50 Hz 就別把 horizon 推超過 30；長 horizon/多輸入/高更新率的 MPC 不用稀疏/結構化求解器就達不到，而且它本來就不適合擺進硬迴圈（出處 09-compute-capability.md:30,42）。

#### FFT 能跑幾 Hz

FFT（即時，fp32，約 5N·log₂N flops，native C，單核 @30% 效率，皆 ESTIMATE，出處 09-compute-capability.md:31）：

| 點數 N | 每秒可算次數 | 點數 N | 每秒可算次數 |
|---|---|---|---|
| 1024 | 約 46k/s | 65536 | 約 450/s |
| 4096 | 約 9.6k/s | 262144 | 約 100/s |
| 16384 | 約 2.1k/s | 1M | 約 23/s |

大型 FFT 會從運算綁死轉為記憶體傳輸綁死（memcpy 3.5 GB/s），但上表範圍內仍以運算為主。實務判斷：小到中型 FFT（N≤16384）逐樣本高速率串流沒問題；**大型 FFT（N≥262144）適合離線或分塊處理，不適合逐樣本高速率串流 DSP**（出處 09-compute-capability.md:31,43）。

#### 其他常見核心運算（供規劃參考）

- **稠密 LU 解**（flops≈⅔N³，任何求解器的底層）：N=50 約 15k/38k Hz（fp64/fp32），N=100 約 1.9k/4.7k，N=200 約 230/590，N=500 約 15/38（4 核 fp64 約 52），N=1000 約 2 Hz（已非即時）（出處 09-compute-capability.md:28）。
- **圖搜尋 / A***：單核約 1.4–2.8 M 節點/s，4 核約 5–10 M/s；但大型格狀圖會被 DRAM 隨機延遲（約 196 ns/miss）綁死，實際掉到約 0.5–1 M 節點/s（出處 09-compute-capability.md:32）。
- **NLS / bundle-adjustment / MHE**（Gauss-Newton，稠密約 N³/3 每次迭代）：N=50 約 30k iter/s、N=100 約 3.7k、N=200 約 460、N=500 約 30（fp64 單核）；一個 200 變數、10 次迭代的 NLS 約需 22 ms → 約 45 Hz；稀疏問題（典型 SLAM/MHE）快得多（出處 09-compute-capability.md:33）。

> ⚠️ **注意**：**情境**——你在即時迴圈裡做大型稠密 fp64 線性代數。**症狀**——求解跟不上控制週期。**原因**——N=1000 稠密 LU 約 2 Hz（單核）/6.5 Hz（4 核），N≥500 的 fp64 單核 ≤15 Hz。**預防**——把稠密 fp64 問題維度控制在 N≤200 以維持 >100 Hz，或改用稀疏結構（出處 09-compute-capability.md:41）。

#### 四顆核心的預算怎麼分

實測 4 執行緒擴展 3.87×，等於約 3.8 顆獨立核心可用。一個實際可行的核心分配範例（以機載工作負載為例，出處 09-compute-capability.md:34）：

```text
core0 : 200 Hz MPC (n12/H20, 約 30% 使用率) + 安全監控
core1 : 2–3 個 EKF @400 Hz (各 <5%)
core2 : 規劃器 / A*
core3 : DRP-AI3 NPU 餵送 + OS / ROS
```

這樣配下來對前述這類應用堆疊仍有充裕餘裕——再次印證「A55 的限制不是算力」。

**最後把 Linux 硬即時的天花板釘死**：有負載下最壞抖動 926 µs。要抖動小於週期的 10%，週期得 >9.26 ms（＝週期為抖動的 **10 倍**，較保守的一端 → 約 108 Hz）；若把準則放寬到週期只需為抖動的 **3 倍**（餘裕較小、較激進的一端），約 360 Hz。所以 **Linux/A55 上可信賴的硬迴圈上限落在約 100–360 Hz 這個區間**——低頻端（108 Hz）餘裕大、高頻端（360 Hz）餘裕小，不是「360 Hz 比較保守」；任何需要保證 >300–500 Hz 硬期限的（內環姿態/轉速、馬達換相）**必須跑在 Cortex-R8 上**（出處 09-compute-capability.md:35）。這條和前面 cyclictest、和 R8 的定位，三處互相印證，是本節最該記住的一句話。

#### DRP-AI3 各模型類別能跑幾 FPS

最後回到 NPU 的能力面。以下 INT8、僅推論（inference-only），除 YOLOX-nano 為實測外皆 **ESTIMATE**（由 nano 錨點依 FLOPs 比例外推，出處 09-compute-capability.md:63-69）。**要算真實端到端，記得再扣約 8 ms 的前+後處理**。

| 類別 | 代表模型與估計 FPS |
|---|---|
| 物件偵測 | YOLOX-nano@416 **實測 15.3 ms / 65 FPS（推論）、42 FPS（端到端）**；YOLOX-tiny 35–50、YOLOv5n/8n@640 28–45、SSD-MobileNetV2 55–80、YOLOX-s@640 14–22 |
| 影像分類 | ResNet-50@224 65–125、ResNet-18 90–160、MobileNetV2 80–160、EfficientNet-Lite 50–100（此類效率最佳） |
| 語意分割 | DeepLabV3+MobileNet@512 14–28、UNet@256 22–40、UNet@512 8–25、SegFormer-B0 5–12 |
| 姿態估計 | MoveNet-Lightning@192 60–100、Thunder@256 40–65、HRNet-W32 14–28、RTMPose 33–65 |
| 單眼深度 | MiDaS-small@256 14–28、FastDepth@224 33–65（輸出相對深度，度量用途需尺度校正） |
| 人臉/ReID | SCRFD@640 33–65、MobileFaceNet@112 100–200、ArcFace-R50@112 55–100、OSNet 50–100（很適合目標跟隨，以無人機為例） |
| 時序/序列 | 1D-CNN/TCN 60–120、ST-GCN 28–65；RNN/LSTM/注意力型會退回 A55 |

**DRP-AI3 的能力邊界（誠實標註）**：它稱職於偵測/分類/分割/姿態/深度/人臉-ReID 的 INT8 CNN（約 7–60 FPS）；**無法有效執行** LLM、大型 ViT/transformer、需要 fp32 的網路、動態 shape 的計算圖（出處 09-compute-capability.md:55）。

> ⚠️ **注意**：**情境**——你想同時跑多個模型（偵測 + ReID + 深度）或多路串流，並期待總 FPS 是各模型相加。**症狀**——整體 FPS 遠低於期待值。**原因**——板上**只有一顆 `/dev/drpai0` AI-MAC，是單一序列化的 NPU**，多模型/多串流是時間分片依序執行，總 FPS = 1 ÷（各模型延遲總和），沒有真正的並行。**預防**——規劃管線時用「序列時間預算」抓總延遲，別用「並行假設」；另外 R8/M33 在 Linux 中不可用於 AI 卸載（出處 09-compute-capability.md:79）。

> ⚠️ **注意**：**情境**——你量到純推論很快，就直接拿推論時間當整條管線的延遲去排時程。**症狀**——實際端到端明顯比推論時間長。**原因**——純推論本身**沒有固定下限**：224² 分類器在板上實測 mobilenetv2 **1.34 ms**、resnet50 **4.19 ms**，全程跑在 DRP-AI、零 CPU fallback（出處 `assets/hardware-investigation-20260621/results/19_drpai_models.txt`），小模型確實能遠低於 10 ms。讓端到端比純推論長的，是推論**之外**的前處理＋後處理階段（YOLOX-nano@416 實測 前處理 6.86 ms〔DRP 加速〕＋後處理 1.62 ms〔部分 A55〕，出處 05-compute-benchmark.md:143）——這是影像縮放／格式轉換等管線步驟的成本，不是每張圖固定加一個 NPU 啟動延遲。**預防**——排時程時端到端一定要抓「推論時間＋前後處理」（YOLOX-nano@416 實測端到端 23.7 ms vs 純推論 15.3 ms），別把小模型的純推論時間當成整體延遲，也別假設有個固定的啟動地板（出處 09-compute-capability.md:78）。

### 三大運算引擎，一句話各歸其位

把整節收束成一張「該把什麼放哪」的對照表（實測對照，出處 09-compute-capability.md:89-91）：

| 引擎 | 實測能力 | 適合 | 不適合 |
|---|---|---|---|
| **4×A55 CPU** | 28 GFLOPS fp32 / 10.8 fp64 / CoreMark 26511 | 估測（EKF/MHE）、規劃（A*/MPC）、通訊協定、CPU 後援 | 硬即時 >300 Hz（抖動 0.93 ms，須改 R8）、大型 fp64、Python 高頻迴圈 |
| **DRP-AI3 NPU** | YOLOX-nano 15 ms（端到端 42 FPS）、8/80 TOPS | INT8 CNN 偵測/分類/分割/姿態/深度/ReID（約 7–60 FPS） | LLM/大型 transformer、fp32 網路、動態 shape |
| **Mali-G31 GPU** | 4.51 GFLOPS fp32、OpenCL 3.0 | 並行逐像素影像/CV kernel、相機零拷貝、HMI | 通用浮點加速、AI 主推論 |

一句話總結：RZ/V2H 作為「控制（以飛行控制為例）＋感知＋通訊」的單板異質運算平台完全勝任，DRP-AI3 是視覺推論的決定性優勢；你只需要在設計時處理三個工程現實——**(1) 開機儲存是消費級 SD 卡、(2) A55 沒有 AES/SHA 硬體加密、(3) 預設核心非 PREEMPT_RT 且 SCHED_FIFO 被 cgroup 擋住**（出處 05-compute-benchmark.md:27）。這三條，本節的 benchmark 和陷阱框都給了你確切數字與繞法。

### 動手驗證：確認你手上這塊板子跟本節數據一致（各步標記見下）

做完這一節，用下面幾步快速確認「我這塊板子的運算單元狀態和本節基準一致」。判斷標準寫在每步後面。

**✅ 驗證 1（2026-07-17 板上重執行，transcript：live/ch04-cpu-periph.txt）：時脈與 governor 鎖對了嗎？**

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

判斷：應為 `performance` 與 `1700000`。若不是，你的 benchmark 數字會因調頻而對不上本節（出處 07-hardware-unit-usage-guide.md:178-179）。

**✅ 驗證 2（2026-07-17 板上重執行，確認無任何輸出，transcript：live/ch04-followup.txt）：加密延伸確實缺席嗎？**

```bash
grep -oE 'aes|sha2' /proc/cpuinfo | sort -u
```

判斷：應**無任何輸出**。有輸出代表你的板子/核心和本節假設不同，加密吞吐的結論要重新評估（出處 05-compute-benchmark.md:46）。

**⏸ 驗證 3（benchmark 負載，現役服務佔用，未重執行——依實錄；需先 `sudo apt install sysbench`，乾淨映像檔未預裝、否則 command not found）：跑一個 sysbench，看多核擴展是否近線性。**

```bash
sysbench cpu --cpu-max-prime=20000 --threads=1 --time=20 run | grep 'events per second'
sysbench cpu --cpu-max-prime=20000 --threads=4 --time=20 run | grep 'events per second'
```

判斷：4 緒 ÷ 1 緒應接近 **3.9×**（本節實測 1306.4 ÷ 332.3 ≈ 3.93×）。明顯低於 3.5× 代表有背景負載或散熱/供電異常（出處 05-compute-benchmark.md:61、01_cpu.txt:3-17）。

**✅ 驗證 4（2026-07-17 板上重執行，見附註）：確認散熱正常、無降頻。**

```bash
cat /sys/class/thermal/thermal_zone0/temp
cat /sys/class/thermal/thermal_zone1/temp
```

判斷：待機時兩區應在 35000–36000（即 35–36 °C）附近（單位 m°C）。滿載 5 分鐘後應仍遠低於降頻門檻——本節實測全程 1700 MHz、548/548 取樣零降頻（出處 00_inventory.txt:156-157、05-compute-benchmark.md:180-182）。**注意：這是裸板數字，裝機殼後必須重測。**（✅ 2026-07-17 板上重執行附註，transcript：live/ch04-cpu-periph.txt〔溫度〕、live/ch04-reserved-mem.txt〔load average〕：當時兩區皆讀到 `39000`（39 °C）——板上有常駐服務在跑、load average 約 1.0，並非待機狀態，比 35–36 °C 的待機基準高幾度屬合理；要對「待機 35–36 °C」這條，得在無負載時量。）

**⏸ 驗證 5（需 SCHED_FIFO 與滿載情境，現役服務佔用，未重執行——依實錄；選做，需 rt-tests）。**

```bash
# -p80 要 SCHED_FIFO；先暫時放行 RT 頻寬，否則會 EPERM（量測期間的臨時設定）
sudo sh -c 'echo -1 > /proc/sys/kernel/sched_rt_runtime_us'
sudo cyclictest -p80 -i250 -a -t -D30 -q
# 量完立刻還原成預設值
sudo sh -c 'echo 950000 > /proc/sys/kernel/sched_rt_runtime_us'
```

判斷：idle 下最差延遲應在**次毫秒級**（本節實測 idle max 347–420 µs、負載 max 593–926 µs）。若最差延遲遠超 1 ms，代表有未預期的高延遲來源，會影響你對「A55 硬即時上限約 100–360 Hz」這條的規劃（出處 05-compute-benchmark.md:166-167、07_realtime.txt:6-15）。記得 `chrt -f` 若回報 EPERM，是 cgroup RT_GROUP_SCHED 擋的，不是你權限不夠（見前面的陷阱框）。

全部通過，代表你手上的板子與本節的能力上限推導站在同一個基準上——接下來的章節（感知、控制、系統整合）就能安心引用這裡的 Hz 數字來規劃你的系統軟體架構。

---
