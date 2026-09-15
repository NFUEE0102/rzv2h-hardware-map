# 06 · 計時系統（計時器／PWM）

RZ/V2H 這片 SoC（system on chip，把處理器、記憶體控制器、各種周邊全塞進一顆晶片）裡，「會數時間」的硬體不只一種。官方硬體手冊把它們集中收在 SECTION 5 TIMER 底下，本群組就對應這一整節：系統時基、看門狗、通用計時器、比較匹配計時器、通用型計時器（也是唯一能輸出 PWM 波形的那顆）、PWM 輸出閘控，以及即時時鐘。

一開始你會覺得「怎麼有這麼多顆計時器」，但它們的分工其實很清楚，可以先用一句話抓住整群的骨架：

```text
RZ/V2H 計時系統：三種角色

┌── 系統時基（核心自用，應用層只能間接透過標準 API 使用）───────────┐
│  SYC          64-bit 系統計數器 → 供給 A55 的 generic timer          │
│               （Linux arch_timer 的硬體來源）與 GE3D 共用一份計數    │
│  GTM／OSTM    8 通道 32-bit，被核心徵用為 clockevent（排程 tick）    │
│  CMTW         8 通道 16/32-bit，被核心當輔助時脈來源                 │
│               ↑ 三者皆「沒有給應用挑通道操作」的介面                 │
├── 應用可控的定時／輸出（有對外接腳或標準 Linux 裝置）──────────────┤
│  GPT          16 通道 32-bit，唯一有對外 I/O 接腳、能產生 PWM 波形   │
│  POEG         GPT 輸出接腳的「故障安全閘」，本身不計數               │
│  （PWM 不是獨立周邊，是 GPT＋POEG 一起產生的）                       │
│  WDT          4 通道看門狗，各綁一顆核心；WDT1 曝露為 /dev/watchdog0 │
│  RTC          日曆時鐘，斷電仍走時；曝露為 /dev/rtc0                 │
└─────────────────────────────────────────────────────────────────────┘
```

這張圖也點出本群組最需要記住的兩件事：**(1)** 前三顆（SYC／GTM／CMTW）是系統層的時基供給，你的程式只能透過 Linux 標準時間 API 間接用它們，沒有「挑某一顆某一通道」的介面；**(2)** 真正「應用會想直接操作」的是後四顆——但其中最搶手的 PWM 輸出，在本板的 Linux 使用者空間偏偏沒有現成驅動程式（下文會逐一交代到底缺在哪、替代路徑是什麼）。

## 本群組單元清單

| 單元 | 一句話 | 板上狀態 |
|---|---|---|
| [SYC（系統計數器）](#1-syc系統計數器system-counter) | 64-bit 系統計數，供給 A55 generic timer 與 GE3D 共用的時基 | 啟用（供給 `arch_timer` 系統時基）；開發紀錄未單列（缺口，僅能反推） |
| [GTM／OSTM ×8ch](#2-gtmostm-8ch通用計時器一顆-8-通道-ip兩個名字) | 8 通道 32-bit 通用計時器，被核心徵用為系統 clockevent | 啟用且核心主動使用（`renesas_ostm` clockevent）；GTM 與 OSTM 是同一顆 IP，勿重複計數 |
| [CMTW ×8ch](#3-cmtw-8ch比較匹配計時器-w) | 8 通道 16/32-bit 比較匹配計時器，有 input capture／output compare 能力 | 啟用（`rz_cmtw` clockevent，未曝露為應用層裝置） |
| [GPT ×16ch](#4-gpt-16ch通用型計時器唯一能輸出-pwm-的計時器) | 16 通道 32-bit，唯一有對外接腳、能產生 PWM 波形的計時器 | 部分啟用（`gpt@13010000`＝GPT0 ch0–7 `okay`，其餘 15 節點 `disabled`）；**無 PWM chip 註冊** |
| [POEG／PWM 輸出](#5-poegpwm-輸出gpt-輸出接腳的故障安全閘) | GPT 輸出接腳的故障安全閘；PWM 不是獨立周邊，由 GPT＋POEG 產生 | `/sys/class/pwm` 為空（無 pwmchip、無 POEG 驅動程式）；建議交給即時核心 |
| [WDT ×4ch](#6-wdt-4ch看門狗計時器) | 4 通道看門狗，各綁一顆核心，逾時可重置晶片或發 NMI | 啟用（WDT1＝CA55，`/dev/watchdog0`，`rzv2h_wdt`）；WDT0/2/3 屬 CM33/CR8 韌體 |
| [RTC（RTCA-3）](#7-rtcrtca-3即時時鐘) | 日曆／二進位計數的即時時鐘，靠獨立石英與待機供電，斷電仍走時 | 啟用（`/dev/rtc0`，`rtca3`）；`hwclock -r` 可運作 |

> ⚠️ **注意（`<板子IP>` 佔位）**：本群組指令都在板子本機終端機執行，不牽涉板子網路位址；若你改用 SSH 遠端操作，板子的位址由 DHCP 動態配發、每次開機可能變，一律先 `ip a` 查當下位址，本手冊統一以 `<板子IP>` 佔位、不寫死。

---

## 1. SYC（系統計數器，System Counter）

### 這是什麼（機制）

SYC 是一顆「時基供給」單元——它自己不對應用層開放任何操作介面，角色是產生一份**共同、穩定的計數值**，供兩個下游共用：一是 Cortex-A55 內建的 generic timer（Arm 架構標準計時器，也就是 Linux 裡 `arch_timer` 的硬體來源；「generic timer」是 Arm 對每顆應用核心都內建的一組計時暫存器的統稱），二是 GE3D（本 SoC 上的 3D 繪圖引擎）。（r01uh1032 §5.2 概說）

它產生計數的方式，是借用 Arm CoreSight SoC-400（CoreSight 是 Arm 的除錯／追蹤基礎架構）裡的 timestamp generator 產生原始計數，SYC 再把這份計數轉成 **Gray code**（格雷碼，一種相鄰數值只差一個位元的編碼方式，用來在跨時脈域取樣時避免多位元同時翻轉造成的取樣錯誤）輸出。手冊的方塊圖（Figure 5.2-2）把這條路徑畫成 Time Stamp Generator → BIN2GLAY → Count output。（r01uh1032 §5.2.1.2）

計數的時脈來源是一顆 24 MHz 的 `SYC_0_CNT_CLK`。（r01uh1032 §5.2.1.1 Features）

它還有一個和除錯有關的特性叫「Halt on Debug」：當 CoreSight 透過 CTI（Cross Trigger Interface，跨觸發介面，讓除錯事件能在多個單元間互相通知）送出 HALTREQ 時，timestamp generator 會停止計數；送出 RESTARTREQ 時恢復。手冊特別註明，Cortex-A55 與 GE3D 共用的是**同一份計數**，所以兩者的停止／恢復是同步發生的（逐字：「The stop/restart control of the counter by the Halt on Debug function is also performed simultaneously for Cortex-A55 and GE3D」）。（r01uh1032 §5.2.3.2 NOTE）

把上面收攏成一句話：SYC 是系統層的時基來源，讓 A55 的 generic timer 與 GE3D 有一份共同、穩定的計數可用——它不是一顆「給應用挑功能來操作」的周邊。

### Linux 下怎麼看到它

老實說，你在使用者空間**幾乎看不到 SYC 本身**。這一點要講清楚，因為它是本群組唯一「開發紀錄未單獨盤點」的缺口單元：

- 開發紀錄（硬體單元使用指南，全文 49 個編號單元）裡**沒有單獨列出 SYC**——它是這張資源地圖裡誠實標記的缺口之一，不是既有結論。
- 唯一能查到的間接線索是中斷清單：`arch_timer` 在 `/proc/interrupts` 裡是活躍的（doc06 §3；本板 ✅ 2026-07-17 板上重執行 `cat /proc/interrupts` 時，`arch_timer` 確實在跳動，transcript：live/ch04-followup.txt）。`arch_timer` 活躍，代表 A55 的 generic timer（其計數源就是 SYC）確實在運作——但這是**反推**，不是對 SYC 本身的直接盤點。
- 沒有 `/dev` 節點，也沒有 sysfs 控制介面（sysfs 是核心把裝置狀態以檔案形式曝露在 `/sys` 底下的虛擬檔案系統）。你能做的驗證只有間接的一句：

```bash
cat /proc/interrupts | grep arch_timer
```

這確認的是 SYC 下游的 generic timer 中斷有沒有在動，**不是**直接讀 SYC 的 64-bit 計數值。使用者空間沒有任何手段可以直接讀那份計數，也不能存取 SYC 的 `PSELCTRL`／`PSELREAD` 暫存器。

> ⚠️ **注意（別把「反推」當「實測」引用）**：
> - **情境**：你想在文件或報告裡寫「本板 SYC 已驗證啟用、時脈 24 MHz」。
> - **症狀**：這句話沒有任何板上直接證據撐得起來。
> - **原因**：SYC 沒有 `/dev`／sysfs 節點，本板只能從 `arch_timer` 中斷活躍**反推**它在運作；24 MHz、64-bit 這些是**官方手冊規格**，不是板上量到的值。
> - **預防／處理**：引用時分清楚兩件事——「板上狀態」只能寫成「由 `arch_timer` 反推啟用」；規格數字一律標官方手冊出處，不要混成「實測」。

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| 計數位元寬度 | 64-bit Gray code 計數值（逐字：「64-bit gray code counter value generation」） | r01uh1032 §5.2.1.1 Features |
| 計數時脈 | 24 MHz（`SYC_0_CNT_CLK`） | r01uh1032 §5.2.1.1 |
| 位址空間 | 8 KB；base `<SYC0_base>`＝`0x1401_0000`（CM33 non-secure `0x5401_0000`、secure `0x4401_0C00`） | r01uh1032 §5.2.2 Table 5.2-1 |
| 存取分區 | 前半 4 KB（`PSELCTRL` 區）與後半 4 KB（`PSELREAD` 區），皆同時支援 secure／non-secure 存取 | r01uh1032 §5.2.2 Table 5.2-2、§5.2.3.1 |

限制面最重要的一句：它沒有使用者空間介面，能力表裡的位址與分區資訊，是要走裸機（bare-metal，指不經作業系統、直接對硬體暫存器讀寫）或除錯探針才用得到的——一般 Linux 應用開發碰不到，也不需要碰。

### 什麼情況下你會用到它

**機制**：只要你的程式呼叫標準 POSIX 時間 API——`clock_gettime(CLOCK_MONOTONIC)`（讀一個開機後單調遞增、不會被調時往回撥的時間）、`nanosleep`（睡眠指定時間）、或核心排程器的 tick——最終都間接依賴 A55 的 generic timer，而 generic timer 的計數源正是 SYC。換句話說，你**一直在用它**，只是隔了好幾層。

**判斷準則**：一般應用開發完全不需要、也無法直接程式化 SYC。不論是感測器輪詢、控制迴圈排程，或任何需要量測「經過多久」的場合，走 Linux 標準時間 API 就是在共用 SYC 這份時基。你唯一會實際「碰到」SYC 特性的情境，是做 CoreSight 除錯／追蹤、需要確保 A55 與 GE3D 在中斷點同步凍結計數時，才會用到 Halt on Debug——一般應用開發不會走到這一層。

**以工業檢測相機為例**：要為每張影像加上時戳、或某個即時控制迴圈要量測週期，只要走 Linux 標準時間 API，都是「機制」層面共用 SYC，而不是任何特定應用的選型結果——換成別的應用領域，結論一模一樣。

> 出處：官方硬體手冊 r01uh1032 **§5.2 System Counter (SYC)**（p1162–1163 概說；暫存器自 §5.2.2 p1164）。板上狀態反推自 doc06（`06-hardware-resource-map.md`）§3 的 `arch_timer` 中斷活躍；SYC 本身開發紀錄未單列（缺口）。

---

## 2. GTM／OSTM ×8ch（通用計時器，一顆 8 通道 IP、兩個名字）

### 這是什麼（機制）

先解決一個最容易數錯的問題：**GTM 和 OSTM 是同一顆硬體，不是兩顆。** 官方手冊 §5.5 的正式標題是「General Timer (GTM)」，但這一節裡的暫存器全部以 `OSTMn` 開頭命名（`OSTMnCMP`／`OSTMnCNT`／`OSTMnTE`／`OSTMnTS`／`OSTMnTT`／`OSTMnCTL`），因為這顆 IP 與 Renesas 既有的 OSTM（One-Shot/Interval Timer，單發／間隔計時器）暫存器相容。開發紀錄在兩個地方分別描述它（一處講 GTM、一處講 OSTM），但底層是**同一顆 8 通道硬體**。（r01uh1032 §5.5）

它共有 8 個通道 OSTM0–OSTM7，每通道各自獨立（逐字：「Number of channels 8」）。每通道有兩種運作模式（r01uh1032 §5.5.1、§5.5.2.2.2 Table 5.5-4）：

- **Interval timer mode（間隔計時模式）**：32-bit 的 down-counter（往下數的計數器），從 `OSTMnCMP` 設定值往下數到 0（初始值 `FFFF_FFFFh`），可重複觸發中斷或啟動 ELC。
- **Free-running comparison mode（自由運轉比較模式）**：32-bit 的 up-counter（往上數，初始值 `0000_0000h`），把計數值持續與 `OSTMnCMP` 比較，符合時觸發中斷／ELC。

每通道能產生中斷（`GTMn_GTMTINT`），或直接啟動 **ELC**（Event Link Controller，事件連結控制器——一種讓硬體事件不經過 CPU 就直接觸發另一個硬體動作的機制，可省下中斷處理的延遲）；但它**不能**直接觸發 DMAC（DMA Controller，直接記憶體存取控制器）。手冊逐列標明 8 個通道都是「Startup of Direct Memory Access Controller：Not possible」「Function Startup of Event Link Controller：Possible」。（r01uh1032 §5.5.1.1 Table 5.5-2）

最後一個本質特徵：它**沒有任何對外 I/O 接腳**，純粹是內部的計數／中斷／ELC 事件單元。這一點是它與下文 GPT（有對外接腳、能輸出波形）最根本的差異——GTM／OSTM 的計數結果只能留在晶片內部用，送不到外面的接腳上。

### Linux 下怎麼看到它

- **驅動程式**：`renesas_ostm`（核心原始碼 `drivers/clocksource/renesas-ostm.c`）。它被繫結為核心的 **clockevent／clocksource** 提供者——clocksource 是核心用來讀「現在時間」的計時來源，clockevent 是核心用來排定「在某個時間點觸發一個事件」（例如排程器的下一個 tick）的計時器。它**不是** `/dev` 字元裝置，也**沒有** ioctl 介面（ioctl 是使用者空間對裝置節點下控制指令的系統呼叫）。（doc07 §19，逐字：「bound as a clockevent/clocksource — NOT a /dev character device. It has no userspace ioctl interface」）
- **板上狀態**：至少一個通道被核心徵用為系統的 clockevent 裝置（驅動核心排程 tick），其餘通道由核心／韌體管理，不對應用層開放。（unit-map 06：「啟用且核心主動使用（`renesas_ostm` clockevent）」）
- **觀察方式**：

```bash
cat /proc/timer_list | grep -i ostm
```

這能確認它目前是作用中的 clockevent。（doc07 §19／§21）

- **應用層怎麼用**：只能透過標準 POSIX 計時器介面（`timerfd_create`、`nanosleep`、`clock_nanosleep`）**間接**使用。核心把這些請求排到它管理的計時器上，但**不保證一定落在 GTM／OSTM**——視核心排程器當下選用的 clockevent 而定。

> ⚠️ **注意（GTM 與 OSTM 別數成兩顆）**：
> - **情境**：你在板子的硬體清單裡同時看到「GTM ×8ch」與「OSTM（`renesas_ostm`）」兩條，直覺以為有共 16 個通道可用。
> - **症狀**：實際能用的計時通道，怎麼算都對不上你以為的數量。
> - **原因**：GTM 與 OSTM 是**同一顆 8 通道硬體**，只是用兩個名字各列一次；它們的暫存器彼此相容（`OSTMnCMP`／`OSTMnCNT`…）。
> - **預防／處理**：把它當**一組 8 通道**看待，不要重複計數；而且這 8 通道已被核心當系統時基占用，不開放應用層自由挑通道使用（出處 `07-hardware-unit-usage-guide.md:326,362-365`）。

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| 通道數 | 8（GTM0–GTM7／OSTM0–OSTM7；逐字「Number of channels 8」） | r01uh1032 §5.5.1.1 Table 5.5-1 |
| 計數器寬度 | 32-bit（`OSTMnCMP`／`OSTMnCNT` 皆 32-bit） | r01uh1032 §5.5.2.2.1／§5.5.2.2.2 |
| 兩模式初始值／方向 | interval timer mode＝down、初始值 `FFFF_FFFFh`；free-running comparison mode＝up、初始值 `0000_0000h` | r01uh1032 §5.5.2.2.2 Table 5.5-4 |
| 中斷路由能力 | 可啟動 ELC；**不能**啟動 DMAC | r01uh1032 §5.5.1.1 Table 5.5-2 |
| 對外接腳 | 無（純內部計數／中斷／ELC 單元） | r01uh1032 §5.5.1.1 |
| 8 通道暫存器基底 | GTM0＝`0x1180_0000`、GTM1＝`0x1180_1000`、GTM2＝`0x1400_0000`、GTM3＝`0x1400_1000`、GTM4＝`0x12C0_0000`、GTM5＝`0x12C0_1000`、GTM6＝`0x12C0_2000`、GTM7＝`0x12C0_3000` | r01uh1032 §5.5.2 Table 5.5-3 |

限制面關鍵一句：它沒有對外接腳、也沒有應用層可直接指定通道的介面——這決定了它「只能被核心當系統時基用」的定位。

### 什麼情況下你會用到它

**機制**：GTM／OSTM 是核心排程用的通用計時器，沒有對外接腳，不是給應用直接指定通道操作的周邊。

**判斷準則**：當你需要「精確、應用層可控的定時中斷或事件」時，先分辨你要的是哪一種——

- 若要的是「作業系統排程精度內的定時」（毫秒等級、可以接受核心排程延遲），用 `timerfd`／`nanosleep` 就夠了，核心會用 GTM／OSTM 之類的 clockevent 幫你排程；你不需要、也無法自行挑選特定 GTM 通道。
- 若要的是「硬體級、需要對外輸出波形或與其他訊號同步」的定時，那應該看下一節的 GPT（有 I/O 接腳與 PWM 能力）——因為 GTM／OSTM 完全沒有對外接腳，做不到這件事。

**以感測器輪詢排程為例**，或任何管線裡的節流計時器，只要精度落在毫秒等級、可以接受核心排程延遲，直接用 Linux 的 `timerfd`／`nanosleep` 即可——這些請求最終會被核心排程到 GTM／OSTM 之類的硬體 clockevent 上。換成別的應用，判斷準則相同：先問「我的定時要不要送到接腳上」，答案是「不用」就走標準 API。

> 出處：官方硬體手冊 r01uh1032 **§5.5 General Timer (GTM)**（p1236–1237 概說；暫存器自 §5.5.2 p1238）。開發紀錄 doc07（`07-hardware-unit-usage-guide.md`）§19（GTM）＋§21（OSTM，合併為同一顆）、doc06 §2。

---

## 3. CMTW ×8ch（比較匹配計時器 W）

### 這是什麼（機制）

CMTW（Compare Match Timer W）共 8 個通道，實際結構是 **4 通道 × 2 個 unit**（逐字：「eight channels (4 channels × 2 units)」）。每通道是一個可選 16-bit 或 32-bit 的 up-counter，當計數值與比較值符合時觸發中斷，然後計數器歸零回到 `0000_0000h`。（r01uh1032 §5.6.1 Table 5.6-1）

除了單純的 compare match（比對一致就中斷）之外，CMTW 還多了兩項 GTM／OSTM 沒有的能力（r01uh1032 §5.6.1）：

- **Input capture（輸入捕捉）**：外部訊號觸發時，把當下的計數值鎖存下來，每通道最多 2 個輸入接腳（`TICn0`／`TICn1`）。這讓它理論上能量測「外部脈波之間的時間間隔」。
- **Output compare（輸出比較）**：計數值符合比較值時，在對外接腳上輸出訊號，每通道最多 2 個輸出（`TOCn0`／`TOCn1`）。

但要注意：**只有 CMTW0–3 這 4 個通道有對外接腳**，CMTW4–7 完全沒有 I/O pin，只能做內部中斷／事件用途（逐字注記：「CMTW4 to CMTW7 do not have any I/O pins」）。（r01uh1032 §5.6.1 Table 5.6-3）

計數的除頻可選 4 種 prescaler（預除器，把輸入時脈先除以某個倍數再送進計數器，用來調整計數速度與可量測的時間範圍）：`PCLKL/8`、`/32`、`/128`、`/512`。（r01uh1032 §5.6.1 Table 5.6-1）另外它可產生 3 種事件連結（ELC）輸出，不經 CPU：compare match event、output compare 0 event、output compare 1 event。（r01uh1032 §5.6.1 Table 5.6-1）

### Linux 下怎麼看到它

- **驅動程式**：`rz_cmtw`（Renesas RZ CMTW clock-event／clocksource 驅動程式），繫結為核心計時器，**不是** `/dev` 節點，無 ioctl 介面。（doc07 §20，逐字：「bound as a kernel timer, NOT a /dev node」）
- **板上狀態**：核心已載入此驅動程式作為輔助時脈來源，但**未曝露為應用層裝置**。（unit-map 06：「啟用（`rz_cmtw` clockevent，未曝露為應用層裝置）」）
- **觀察方式**：

```bash
cat /proc/timer_list
ls /sys/devices/system/clocksource/
```

（doc07 §20）

- **能力上的斷點（很重要）**：應用層同樣只能透過核心的 POSIX 計時器介面間接使用，無法指定特定 CMTW 通道；而且**沒有任何驅動程式能讓應用層存取 CMTW0–3 的 input capture／output compare 接腳**——開發紀錄裡沒有對應的 sysfs 或字元裝置路徑。也就是說，CMTW 那項「量測外部脈波間隔」的能力，矽片上存在，但在本板的 Linux 應用層目前**不可達**。

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| 通道數 | 8（4ch × 2 units） | r01uh1032 §5.6.1 |
| 計數器寬度 | 16-bit 或 32-bit 可選 | r01uh1032 §5.6.1 Table 5.6-1 |
| Prescaler | `PCLKL/8`、`/32`、`/128`、`/512` 四選一 | r01uh1032 §5.6.1 Table 5.6-1 |
| I/O 接腳 | 僅 CMTW0–3 有 `TICn0`／`TICn1`（輸入捕捉）與 `TOCn0`／`TOCn1`（輸出比較）；CMTW4–7 無接腳 | r01uh1032 §5.6.1 Table 5.6-3 |
| 中斷來源 | compare match、input capture 0/1、output compare 0/1，每通道最多 5 種 | r01uh1032 §5.6.1 Table 5.6-1 |
| 暫存器基底 | CMTW0＝`0x11C0_1800`、CMTW1＝`0x11C0_1C00`、CMTW2＝`0x11C0_2000`、CMTW3＝`0x11C0_2400`、CMTW4＝`0x1300_0C00`、CMTW5＝`0x1300_1000`、CMTW6＝`0x1300_1400`、CMTW7＝`0x1300_1800` | r01uh1032 §5.6.2 Table 5.6-4 |

### 什麼情況下你會用到它

**機制**：CMTW 比 GTM／OSTM 多了 input capture／output compare 能力，理論上可以量測外部脈波寬度或產生比較輸出訊號——但這條能力在本板的 Linux 下沒有對應驅動程式可控制，目前僅被核心當一般計時器使用。

**判斷準則**：若你的應用需要「量測外部訊號的精確時間間隔」（**以旋轉編碼器的脈波間隔為例**，或量測外部觸發訊號到達的時間點），要先確認兩件事：

1. 該訊號實際接的是哪支接腳、是否落在 CMTW0–3 的 `TICn0`／`TICn1` 上——這要查電路圖與 PFC（Pin Function Controller，腳位功能控制器，決定一支實體接腳當下切換成哪一種功能）的 pinmux 設定才知道。
2. Linux 是否有對應驅動程式——**目前沒有**。

在沒有現成核心驅動程式的情況下，CMTW 的 input capture 能力屬於「矽片存在、Linux 應用層不可達」這一類。真要用，只有兩條路：自行開發核心驅動程式，或改走即時核心（Cortex-R8／M33）的韌體直接操作暫存器。

**以量測工業感測器的脈波週期為例**：走 Linux 應用層目前這條路走不通；只有當即時性需求高到值得投入開發成本時，才需要評估把這類量測工作移到即時核心韌體上、直接操作 CMTW。這是一個「能力面存在、決策要看投入產出」的典型例子，與是哪個專案無關。

> 出處：官方硬體手冊 r01uh1032 **§5.6 Compare Match Timer W (CMTW)**（p1254–1255 概說；暫存器自 §5.6.2 p1256）。開發紀錄 doc07 §20。接腳命名（`TICn0/1`、`TOCn0/1`，n＝0 to 3）另核對 datasheet `r01ds0429` Section 2 接腳功能表。

---

## 4. GPT ×16ch（通用型計時器，唯一能輸出 PWM 的計時器）

### 這是什麼（機制）

GPT（General-Purpose Timer）是一顆 32-bit 計時器，共 16 個通道，分成兩組：GPT0 涵蓋 ch0–7、GPT1 涵蓋 ch8–15（逐字：「The GPT is a 32-bit timer with 16 channels」）。它是本群組裡功能最豐富的一顆，也是**整片 SoC 上唯一能產生 PWM 波形、而且有對外 I/O 接腳的計時器**。（r01uh1032 §5.7.1）

先解釋 PWM：PWM（Pulse Width Modulation，脈寬調變）是一種用「一個週期裡高電位占多少比例（占空比 duty）」來表達類比大小的數位訊號——馬達轉速、伺服角度、LED 亮度都常用它來控制。GPT 每通道可以透過控制 up-counter、down-counter，或 up/down-counter（三角波計數，數上去再數下來）來產生 PWM 波形。

GPT 之所以能做「有品質」的功率控制波形，關鍵在幾個機制（r01uh1032 §5.7.1.1）：

- **每通道 4 支對外 I/O 接腳**：`GTIOCnA`、`GTIOCnB`（主輸出）與 `GTIOCnAN`、`GTIOCnBN`（前兩者的反相訊號）。（Table 5.7-1；§5.7.2 Table 5.7-3）
- **雙緩衝（double buffer）**：每通道有 2 組主要 output compare／input capture 暫存器（`GTCCRA`／`GTCCRB`），再加 4 組緩衝／比較暫存器（`GTCCRC`／`D`／`E`／`F`），可以在波峰或波谷的瞬間切換緩衝值，用來產生「不對稱」的 PWM 波形（逐字：「laterally asymmetric PWM waveforms」）。雙緩衝的意義是：你可以在計時器還在跑的時候，預先把下一週期要用的新占空比寫進緩衝暫存器，等到安全的切換點（波峰／波谷）才生效，避免波形中途出現毛刺。
- **Dead time（死區時間）產生**：上下橋臂切換時，硬體自動插入一小段「兩邊都關」的空檔，避免同時導通——這是橋式功率電路（例如三相馬達逆變器）的標準需求（逐字：「Generation of dead times in PWM operation」）。
- **與 ELC／外部觸發連動**：最多 8 個 ELC 事件可觸發 count start／stop／clear／up-count／down-count／input capture；也支援外部觸發接腳（`GTETRGA`–`D` 對應 GPT0、`GTETRGE`–`H` 對應 GPT1，經 POEG）觸發同樣動作，最多 4 個外部觸發。（Table 5.7-1；§5.7.2 Table 5.7-3）
- **可直接觸發 ADC 轉換**：透過 `GTADTRA`／`GTADTRB` 比較暫存器，在波形的特定相位啟動 A/D 轉換——這對「在 PWM 週期的固定點量測電流」這類馬達控制場合很有用。（§5.7.1.1）
- **13 種中斷來源**：`GTCCRA`–`F` 的比較／捕捉、計數器溢位／下溢、死區時間錯誤、A/B 訊號同高／同低、ADC 觸發比對等。（Table 5.7-2(2/2)）

### Linux 下怎麼看到它

這是本群組最需要誠實講清楚的一顆：**GPT 在本板的 Linux 使用者空間，沒有可用的操作路徑。**

- **板上狀態（開發紀錄）**：16 個 `gpt@…` device tree 節點裡，只有 `gpt@13010000`（GPT0，涵蓋 ch0–7）是 `okay`，其餘 15 個節點全部 `disabled`；而且**沒有 PWM chip 註冊**。（unit-map 06；doc07 §18，逐字：「No PWM chip is registered for it」；✅ 2026-07-18 板上逐一確認 16 個節點只有 `gpt@13010000` 為 `okay`，transcript：live/ch04b-dt-status.txt）
- **沒有標準 Linux PWM 路徑**：`/sys/class/pwm` 目錄存在但為空——沒有 `pwm-rzv2h`／`pwm-rzg2l` 驅動程式綁定到 `gpt@13010000`。雖然 DT 節點探測到了時脈／reset，但沒有曝露任何 `/dev` 或 sysfs 控制給應用層（doc07 §18，逐字：「Effectively 'no Linux driver' for application use; counting/PWM must be driven bare-metal via MMIO」；✅ 2026-07-17 板上實測 `/sys/class/pwm` 為空（transcript：live/ch04-reserved-mem.txt））。
- **確認方式**：

```bash
ls /sys/class/pwm        # 目錄存在但為空 → 確認沒有 pwmchip 註冊
ls -A /sys/class/pwm | wc -l    # 應印 0
```

- **唯一手段**：在 root 權限下用 `devmem`／`mmap` 直接讀寫暫存器，也就是裸機 MMIO（Memory-Mapped I/O，把硬體暫存器當成一段記憶體位址直接讀寫）。流程大致是：先寫入 key `0xA5` 到 `GTWP` 暫存器解除寫入保護、再用 `GTSTR` 啟動計數；要輸出 PWM 還得另外設定 `GTIOR`（腳位致能／準位）、`GTCR`（模式）、`GTPR`（週期）、`GTCCRA`（占空比），並透過 PFC 把接腳切換到 `GTIOCnA` 功能——核心不提供任何輔助函式。（doc07 §18／§24）

> ⚠️ **注意（想用標準 Linux PWM 卻找不到）**：
> - **情境**：你想用標準 Linux PWM sysfs（`/sys/class/pwm`）操作 GPT 來輸出伺服／舵機訊號。
> - **症狀**：`/sys/class/pwm` 底下沒有可用路徑，沒有 `pwm-rzv2h` 驅動程式綁定到 `gpt@13010000`。
> - **原因**：本板未設定 PWM chip 驅動程式——GPT 通道雖 probe 到時脈／reset，但不對使用者空間曝露 PWM 控制。
> - **預防／處理**：要在 A55／Linux 側操作 PWM，得走裸機 MMIO 直接操作 GPT 暫存器；若需要低抖動、與控制迴圈緊密同步的 PWM，更適合交給硬即時核心（見〈什麼情況下你會用到它〉）產生（出處 `07-hardware-unit-usage-guide.md:309`、`07-hardware-unit-usage-guide.md:418-421,434`）。

> ⚠️ **注意（40-pin Header 上標的 `PWM0`／`PWM1` 不等於現成可用）**：
> - **情境**：你看到板上 40-pin 樹莓派相容 header 的 Pin32／33 絲印標著 `PWM0`／`PWM1`（對應 `GPIO12`／`GPIO13`），以為插上去就能輸出 PWM。
> - **症狀**：PWM 無法直接使用。
> - **原因**：device tree 沒有替這兩支腳位開啟 PWM 功能。
> - **預防／處理**：要用得先改 device tree 開啟腳位功能（出處 `04-hardware-quickref.md:112`），並確認 pinmux 真的把該腳位導到 GPT 輸出。這和「無 PWM chip 註冊」是同一件事的兩面。
>
> ✅ **絲印那半的疑問已在 2026-08-06 解答（本注記原本標為待查）**：**是真的接到 GPT，不是沿用樹莓派命名慣例。**
>   查 `REN_WS125V2HRDKREFZ_MAH` p.10（header 每支腳的 port pin 網路名）＋ SoC 手冊 `r01uh1032`
>   **Table 1.2-3 List of Multiplexed Functional Pins**（PDF p.118–127）逐支比對：
>   - **Pin 32（`PWM0`）＝ `PA4` → `GTIOC6A`**
>   - **Pin 33（`PWM1`）＝ `PA7` → `GTIOC7B`**
>
>   並以 Renesas 官方無人機參考實作（[`renesas-rdk/rzv2h_drone_px4`](https://github.com/renesas-rdk/rzv2h_drone_px4) 的 `docs/HARDWARE.md`）
>   實機標註交叉驗證，四路全部吻合：pin 32＝GPT6A、pin 33＝GPT7B、pin 35＝GPT9A（`P96`）、pin 31＝GPT10B（`P53`）。
>
>   **順帶查清的更大一件事**：40-pin header 的 28 支訊號腳裡，**23 支可 mux 成 `GTIOC`**（不可的是 SPI6 那組
>   `P90`/`P91`/`P92`/`P93` 與 `PA0`）。因為同一個 `GTIOC` 輸出可能對到兩支腳、而一支腳同時只能是一種功能，
>   做二分圖最大匹配後：**全部拿來做 PWM 最多 20 路同時輸出**；若沿用參考實作的全部週邊（GPS／遙測／SBUS／
>   LiDAR／I²C），**仍有 13 路**。詳見
>   內部專案文件 `docs/04-uav-integration/05-renesas-rdk-px4-reference.md` §8.2①（該文件不隨本倉庫發布）。
>
>   ⚠️ 仍未閉合：**未取得 RDK 電路圖**（串聯電阻／準位轉換／板上既有負載可能擋掉特定腳位——
>   手冊 p.10 就看得到 I²C 那兩支掛了 2.2 kΩ 上拉 R180/R181），且有 7 支腳的「header 腳號 ↔ port pin」
>   對應尚未定出。**數量結論不受影響，但接線前必須補齊。**

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| 通道數 | 16（GPT0：ch0–7；GPT1：ch8–15） | r01uh1032 §5.7.1；§5.7.2.1 Table 5.7-4 Note |
| 計數器寬度 | 32-bit | r01uh1032 §5.7.1 |
| 每通道 I/O 接腳 | 4 個（`GTIOCnA`／`B`／`AN`／`BN`；逐字「Four input/output pins per channel」） | r01uh1032 §5.7.1.1 Table 5.7-1 |
| 時脈來源 | `clks_gpt` 及其 `/2 /4 /8 /16 /32 /64 /256 /1024` 分頻，或外部觸發 `GTETRGA`–`GTETRGH` | r01uh1032 §5.7.1.1 Table 5.7-2(1/2) |
| 特色機制 | 雙緩衝、不對稱 PWM、dead time 產生、ADC 轉換觸發、13 種中斷來源 | r01uh1032 §5.7.1.1 |
| 暫存器基底 | `<GPT0_base>`＝`0x1301_0000`（ch0–7）、`<GPT1_base>`＝`0x1302_0000`（ch8–15） | r01uh1032 §5.7.2 Table 5.7-4 |
| 板上可用範圍 | 僅 `gpt@13010000`（GPT0 ch0–7）DT `okay`，其餘 15 節點 `disabled`；**無 PWM chip 註冊** | unit-map 06；doc07 §18；✅ 2026-07-18 板上逐一確認（transcript：live/ch04b-dt-status.txt） |

> ⚠️ **注意（區分「矽片理論上限」與「本板 Linux 可用範圍」）**：手冊講的是矽片能力（16 通道、每通道 4 接腳），本板 device tree 目前只有 GPT0 ch0–7 這 8 個通道的節點是 `okay`、且無 PWM chip。引用 GPT 能力時務必分清這兩層，別讓讀者誤以為 16 通道／64 路輸出在本板 Linux 下都能直接用。

### 什麼情況下你會用到它

**機制**：GPT 是本 SoC 上唯一具備對外 I/O 接腳、能產生 PWM 波形的計時器——GTM／OSTM 完全沒有接腳；CMTW0–3 雖有接腳，但只有 input capture／output compare，沒有 PWM 波形產生能力。要輸出 PWM，硬體上只有 GPT 這一個候選。

**判斷準則**：若應用需要輸出 PWM（**以馬達控制、伺服驅動、LED 調光為例**，或任何需要占空比可調波形的場合），先回答「誰來驅動它」：

- Linux 使用者空間目前**沒有 pwmchip**。若堅持在 Linux 應用層操作，只能走 root 權限的裸機 MMIO——風險是沒有核心保護機制，容易誤寫到其他行程可能也在用的暫存器區。
- 若需要「即時、低延遲、與其他控制迴圈緊密同步」的 PWM 輸出（**以馬達逆變器換相為例**），比較穩妥的做法是把 GPT 的操作整個交給不受 Linux 排程抖動影響的核心（本 SoC 上未被 Linux 佔用的 Cortex-R8／M33 皆屬此類），而不是在 Linux 應用層裸機操作。

**以馬達控制為例**：需要精確占空比、低抖動的 PWM 時，GPT 的死區時間產生與雙緩衝機制正是硬體層面為這類應用設計的能力（機制）；但「由誰驅動」仍要看即時性需求（判斷準則）。**以簡單的 LED 調光為例**：若能接受 Linux 排程延遲，裸機 MMIO 也堪用，只是每次開機都要重新設定暫存器，沒有核心持久化機制。這兩個例子的差別不在「哪個專案」，而在「即時性要求」這條判斷準則——讀者換到自己的應用，照同一條線判斷即可。

> 出處：官方硬體手冊 r01uh1032 **§5.7 General-Purpose Timer (GPT)**（p1283–1287 概說；暫存器自 §5.7.2 p1288）。開發紀錄 doc07 §18；接腳命名（`GTIOCnA/B/AN/BN`、`GTETRGA-H`）另核對 datasheet `r01ds0429` Section 2；40-pin header 絲印見 `04-hardware-quickref.md`、WS125 RDK 載板手冊。

---

## 5. POEG／PWM 輸出（GPT 輸出接腳的「故障安全閘」）

### 這是什麼（機制）

先把最容易誤會的觀念講清楚：**PWM 在 RZ/V2H 上不是一個獨立的周邊。** 開發紀錄明確標註「PWM-output — NOT a standalone peripheral on RZ/V2H」——PWM 波形產生的邏輯完全在 GPT（§5.7），而 POEG（§5.8）只負責「輸出接腳要不要放行」這一層。兩者合起來才構成一條完整的 PWM 輸出鏈。（doc07 §24；unit-map 06）

POEG（Port Output Enable for GPT，GPT 埠輸出致能）本身不是計時器、不計數——它是 GPT 輸出接腳的一道**保護閘**：可以把 GPT 的輸出接腳切到停用（disable）狀態。它存在的理由是「故障安全」：萬一 PWM 因程式錯誤讓功率級進入危險狀態，POEG 能在硬體層面立刻切斷輸出，不必等 CPU 反應。（r01uh1032 §5.8.1）

有三種方式能觸發 POEG 停用輸出（r01uh1032 §5.8.1 Table 5.8-1）：

1. **輸入位準偵測**：`GTETRGn` 接腳偵測到上升緣或高準位（經極性／濾波選擇後）。
2. **GPT 自發的輸出停用請求**：當 `GTIOCmA` 與 `GTIOCmB` 同時被驅動到 active level（例如上下橋臂同時導通這種危險狀態）時，GPT 會發出請求，POEG 據此決定是否停用該對輸出接腳。
3. **軟體直接寫暫存器停用**。

它具備雜訊濾波：對 `GTETRGn` 輸入可選 `PCLKB/1`、`/8`、`/32`、`/128` 當濾波取樣時脈，連續取樣 3 次一致才視為有效訊號，避免被短暫雜訊誤觸發。（r01uh1032 §5.8.1 Table 5.8-1）

POEG 依 GPT0／GPT1 各分 4 組，共 8 組，每組各自獨立閘控：POEG0A–D 對應 GPT0（外部接腳 `GTETRGA`–`D`）、POEG1A–D（手冊接腳命名 `GTETRGE`–`H`）對應 GPT1。（r01uh1032 §5.8.2 Table 5.8-3；§5.8.1 Table 5.8-2）

關於「最多幾路 PWM」：開發紀錄推算「4 個 I/O 接腳 × 16 個 GPT 通道 ＝ 最多 64 路 PWM-capable 輸出，由 8 個 POEG 群組分組閘控」（doc07 §24，逐字：「up to 4 I/O pins x16 GPT channels = up to 64 PWM-capable outputs, gated by 8 POEG groups」）。這個 64 是依官方規格（每通道 4 接腳 × 16 通道）**推算所得的矽片理論上限**，與 §5.7.1.1「Four input/output pins per channel」及 16 通道規格一致——但**不是本板實測數字**，本板實際只有 GPT0 ch0–7 這 8 個通道的節點 `okay`（見上一節）。

### Linux 下怎麼看到它

和 GPT 一樣，本板 Linux 側沒有它的路徑：

- **沒有 Linux 驅動程式**：既沒有 `pwm-rzv2h`／`pwm-rzg2l` 的 pwmchip，也沒有 POEG 驅動程式，`/sys/class/pwm` 為空。（doc07 §24，逐字：「there is no pwm-rzv2h/pwm-rzg2l pwmchip and no POEG driver; /sys/class/pwm is absent」；✅ 2026-07-17 板上實測 `/sys/class/pwm` 為空（transcript：live/ch04-reserved-mem.txt））
- **要用 PWM 的完整流程**：直接程式化 GPT 暫存器並透過 PFC 把接腳切到 `GTIOCnA` 功能，再視需要設定對應 POEG 群組的 `POEGGn` 暫存器（doc07 §24 使用範例：以 `devmem` 直接寫 `GTWP`／`GTPR`／`GTCCRA`／`GTIOR`／`GTSTR`）。
- **建議路徑**：開發紀錄對「誰來驅動 GPT＋POEG 做即時 PWM 控制」的建議，是交給不受 Linux 排程影響的即時核心（本板上即 Cortex-R8，但該核心 Linux 未曝露，需先走 remoteproc／韌體載入路徑），而不是在 Linux 應用層裸機操作。這是通用的即時性判斷，與任何特定應用專案無關。（doc07 §24）

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| POEG 群組數 | 8（POEG0A/B/C/D 對應 GPT0；POEG1A/B/C/D，接腳命名 `GTETRGE`–`H`，對應 GPT1） | r01uh1032 §5.8.2 Table 5.8-3 |
| 雜訊濾波時脈 | `PCLKB/1`、`/8`、`/32`、`/128`（連續 3 次取樣一致才有效） | r01uh1032 §5.8.1 Table 5.8-1 |
| 每群組暫存器 | 一個 32-bit 控制暫存器 `POEG_POEGGn`，offset `0x0000` | r01uh1032 §5.8.2.1 |
| 暫存器基底 | POEG0A＝`0x1300_1C00`、0B＝`0x1300_2000`、0C＝`0x1300_2400`、0D＝`0x1300_2800`、POEG1A(E)＝`0x1300_2C00`、1B(F)＝`0x1300_3000`、1C(G)＝`0x1300_3400`、1D(H)＝`0x1300_3800` | r01uh1032 §5.8.2 Table 5.8-3 |
| 理論最大輸出 | 最多 64 路 PWM-capable 輸出（4 接腳 × 16 通道，官方規格推算，**非本板實測**） | doc07 §24 |
| 板上狀態 | `/sys/class/pwm` 為空（無 pwmchip、無 POEG 驅動程式） | unit-map 06；doc07 §24；✅ 2026-07-17 板上實測（transcript：live/ch04-reserved-mem.txt） |

### 什麼情況下你會用到它

**機制**：POEG 的存在理由是「安全」——GPT 輸出的 PWM 若因程式錯誤讓上下橋臂同時導通（`GTIOCmA` 與 `GTIOCmB` 同時 active），POEG 能自動偵測並切斷輸出。這是橋式驅動電路（H-bridge、三相逆變器）常見的硬體保護機制，關鍵在於它**不必等 CPU 中斷處理**，反應比純軟體判斷更快。

**判斷準則**：

- 若 PWM 輸出只是單純訊號產生（**以伺服馬達位置指令、LED 調光為例**），不涉及橋式功率級，POEG 的故障保護要不要啟用，取決於你是否已有其他保護手段。
- 若 PWM 驅動的是「會因短路／同時導通而損毀」的功率電路（**以 H-bridge 馬達驅動、DC/DC 轉換器為例**），POEG 的「輸出自動停用」就是值得評估的硬體保護層——它相對純軟體保護的本質優勢，就是不經 CPU、反應更快。

**以馬達控制為例**：POEG 提供的硬體級輸出停用能在故障當下立即生效，是故障安全設計上的價值所在；但本板 Linux 側目前完全沒有驅動程式可以設定它，實際要用就得走與 GPT 相同的裸機 MMIO 或即時核心路徑。讀者評估自己的應用時，判斷線是「我的 PWM 驅動的功率電路，會不會因誤動作而損毀」——會，就把 POEG 這層硬體保護納入考量。

> 出處：官方硬體手冊 r01uh1032 **§5.8 Port Output Enable for GPT (POEG)**（p1504–1505 概說；暫存器自 §5.8.3）＋ **§5.7 GPT**（PWM 波形產生在 GPT）。開發紀錄 doc07 §24。

---

## 6. WDT ×4ch（看門狗計時器）

### 這是什麼（機制）

WDT（Watchdog Timer，看門狗計時器）是一個 14-bit 的 down-counter，它的角色是「系統失控時的最後防線」。當系統跑飛了、無法再定期「刷新」計數器時，計數器一路往下數到 0（underflow，下溢），WDT 就會出手——可以重置整顆晶片，也可以改成產生一個 NMI（Non-Maskable Interrupt，不可遮蔽中斷，連「關中斷」都擋不掉的最高優先中斷）或 underflow 中斷。（r01uh1032 §5.4.1，逐字：「a 14-bit down counter that can be used to reset this LSI when the counter underflows because the system has run out of control and is unable to refresh the WDT. In addition, the WDT can be used to generate a non-maskable interrupt or an underflow interrupt」）

本 SoC 有 **4 個獨立的 WDT 實例，各自綁定不同核心**：WDT0 綁 CM33、WDT1 綁 CA55（全部 A55 核心）、WDT2 綁 CR8 core0、WDT3 綁 CR8 core1。每顆處理器核心都有自己專屬的看門狗，彼此獨立、互不影響。（r01uh1032 §5.4.2 Table 5.4-2）

計數的時脈來源是 **LOCO**（Low-speed On-Chip Oscillator，低速片上振盪器——一顆精度不高但不依賴外部零件的內建振盪器，正因為它獨立，才適合當「連主時脈都掛了也還在走」的看門狗時基），可選分頻 `1／16／32／64／128／256`。（r01uh1032 §5.4.1 Table 5.4-1）

「刷新」（俗稱「餵狗」）的方式是：在允許刷新的時間內，依序寫入 `00h` 再寫入 `FFh` 到 `WDTRR` 暫存器。（r01uh1032 §5.4.2.2.1，逐字：「The down-counter is refreshed by writing 00h and then writing FFh to WDTRR register (refresh operation) within the refresh-permitted period」）

WDT 還有一個進階的 **window 功能**：可以設定一段「允許刷新」與「禁止刷新」的時間窗口，過早或過晚刷新都會被視為 refresh error（刷新錯誤）而觸發中斷。這用來偵測「程式提早餵狗」這種異常——例如某段邏輯跳過了該做的工作卻提早刷新。（r01uh1032 §5.4.1 Table 5.4-1）逾時週期由 `CKS`（分頻）與 `TOPS`（Timeout Period Select）共同決定，`TOPS` 可選 1024／4096／8192／16384 個「分頻後時脈」週期；window 的起始位置（`RPSS`）可選 25%／50%／75%／100%（100% 代表不限制起點），結束位置（`RPES`）可選 75%／50%／25%/0%（0% 代表不限制終點）。（r01uh1032 §5.4.2.2.2）

### Linux 下怎麼看到它

- **只有 WDT1（綁 CA55 的那顆）在 Linux 下曝露**：裝置節點 `/dev/watchdog0`，驅動程式 `rzv2h_wdt`（核心原始碼 `drivers/watchdog/rzv2h_wdt.c`），走標準 Linux watchdog ioctl API（`WDIOC_*`）。（doc07 §22）
- **WDT0（CM33）與 WDT2／WDT3（CR8 core0/1）不歸 Linux 管**：它們屬於那些核心各自的韌體管轄，Linux 看不到、也控制不了。（unit-map 06：「WDT0/2/3 屬 CM33/CR8 韌體」）
- **工具與用法**：

```bash
wdctl                     # util-linux，查詢看門狗能力
# systemd 的 RuntimeWatchdogSec 可自動代為刷新；
# 也可自行開 fd 手動刷新：
exec 3>/dev/watchdog0     # 開啟即武裝（開始倒數）
printf '1' >&3            # 寫入任一位元組＝刷新（餵狗）
printf 'V' >&3; exec 3>&- # 寫 'V' 為「magic close」優雅解除武裝，再關 fd
# ⚠️ 本板 MAGICCLOSE=0，寫 V 無效、無法乾淨解除，見下方「動手」節 step 2／陷阱框
```

（doc07 §22 使用範例）

> ⚠️ **注意（刷新週期抓在兩個時間之間；window 太嚴會誤觸發）**：
> - **情境**：你替某個長時間運作的服務掛上 `/dev/watchdog0`，想讓它卡死時自動重開機。
> - **症狀**：刷新週期抓太短 → 正常執行都可能來不及餵狗、被誤重置；抓太長 → 系統真的卡死時要拖很久才恢復。開了 window 功能又提早刷新 → 觸發 refresh error。
> - **原因**：WDT 逾時是「上次刷新後多久沒再刷新」決定的；window 功能會連「刷太早」也判為錯誤。
> - **預防／處理**：刷新週期抓在「正常跑完一輪主迴圈所需時間」與「你能接受的最長無回應時間」之間；一般應用若不需要嚴格時序驗證，把 window 起點設 100%（不限制起點）即可，只保留「太晚沒餵」這一種觸發條件。

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| 通道數 | 4（逐字「Number of channels 4 channels」） | r01uh1032 §5.4.1 Table 5.4-1 |
| 計數器寬度 | 14-bit | r01uh1032 §5.4.1 |
| 分頻選項 | LOCO clock 的 `/1`、`/16`、`/32`、`/64`、`/128`、`/256` | r01uh1032 §5.4.1 Table 5.4-1 |
| 逾時週期選項 | 1024／4096／8192／16384 個分頻後時脈週期 | r01uh1032 §5.4.2.2.2 |
| Window 起始／結束 | 起始 25%/50%/75%/100%（不指定起點）；結束 75%/50%/25%/0%（不指定終點） | r01uh1032 §5.4.2.2.2 |
| 核心對應與基底 | `<WDT0_base>`＝`0x11C0_0400`（CM33）、`<WDT1_base>`＝`0x1440_0000`（CA55）、`<WDT2_base>`＝`0x1300_0000`（CR8 Core0）、`<WDT3_base>`＝`0x1300_0400`（CR8 Core1） | r01uh1032 §5.4.2 Table 5.4-2 |
| Linux 曝露 | 僅 WDT1＝`/dev/watchdog0`（`rzv2h_wdt`）；WDT0/2/3 屬各核心韌體 | doc07 §22；unit-map 06 |

### 動手：開檔餵狗、讀出 timeout、看它真的咬下去（bite 重置）

這一顆是本群組少數能在 Linux 下「整條路徑走到底」的單元——從武裝、餵狗、到讓它真的重置整片板子，都能實測。但也因為它會**真的把板子重開機**，每一步都要先想清楚後果。

**機制**：Linux watchdog 的約定是「**開啟 `/dev/watchdog0` 的那一刻就開始倒數**」（武裝）；之後只要往這個 fd 寫入任一位元組就算一次「刷新」（餵狗），把倒數歸位；停下看門狗的唯一乾淨方式，是寫入 magic close 字元 `V` 再關閉 fd。硬體端對應的就是第 6 節講的 `WDTRR` 刷新與 underflow（下溢）重置：餵不到、數到 0，WDT1 就經 **WDT → ICU → CPG 重置鏈**（見 05〈系統骨幹〉第 1 節 ICU）把晶片重開。

**第 1 步：確認裝置在，但先別碰 `/dev/watchdog0`。**（✅ 2026-07-22 板上實測，transcript：`live/ch4-w1-wdt-pre.txt`）

```bash
ls -l /dev/watchdog*
```

```text
crw------- 1 root root  10, 130 Jul 21 11:15 /dev/watchdog
crw------- 1 root root 243,   0 Jul 21 11:15 /dev/watchdog0
```

注意這裡**刻意不用 `cat /dev/watchdog0` 去「看一眼」**——光是開啟它就會武裝看門狗、開始倒數。要唯讀看能力，照理走 `/sys/class/watchdog/watchdog0/`；但本板實測那底下的 `timeout`／`identity`／`state` 等屬性檔**全部不存在**（逐字 `No such file or directory`）——這顆 kernel 沒把 watchdog 的 sysfs 屬性編進來，所以你**沒有一條完全不武裝就能讀到 timeout 的路**，只能靠下一步的 `wdctl`（它會開啟裝置、因而也會武裝）。

**第 2 步：用 `wdctl` 讀能力——但要知道這一步已經把狗放出來了。**（✅ 2026-07-22 板上實測，transcript：`live/ch4-w1-wdt-bite.txt`）

> ⚠️ **注意（下面這行 `wdctl` 一開啟裝置就武裝看門狗；本板無乾淨解除，約 60 秒後一定重開機）**：`wdctl` 為了讀能力必須開啟 `/dev/watchdog0`，而「開啟即武裝」是 watchdog API 的通則；本板 `MAGICCLOSE=0`、沒有中途乾淨解除的辦法，只要跑了這行、之後又沒有行程持續餵狗，約 60 秒後**整片板子**（所有核心／其他使用者與服務）一定硬體重開機——**遠端 SSH 連線會斷、共用板上會影響其他人**。要嘛做好重開機準備，要嘛跑完立刻用 `exec 3>/dev/watchdog0; printf '1' >&3` 持續餵狗撐住。

```bash
sudo wdctl /dev/watchdog0
```

```text
Device:        /dev/watchdog0
Identity:      Renesas RZ/V2H WDT Watchdog [version 0]
Timeout:       60 seconds
Pre-timeout:    0 seconds
FLAG           DESCRIPTION               STATUS BOOT-STATUS
KEEPALIVEPING  Keep alive ping reply          1           0
MAGICCLOSE     Supports magic close char      0           0
SETTIMEOUT     Set timeout (in seconds)       0           0
```

三個本板實測的硬事實，直接決定你能怎麼用它：
- **Timeout 固定 60 秒**，而且 `SETTIMEOUT` 旗標為 `0`——**本板不支援改逾時**，你只能接受 60 秒這個值。
- `MAGICCLOSE` 為 `0`——**本板不支援 magic close**。也就是說本節上方「Linux 下怎麼看到它」示範的「寫 `V` 優雅解除武裝」在這片板子上**不成立**（下方陷阱框詳述）。
- `KEEPALIVEPING` 為 `1`——寫入任一位元組可刷新，這條有效。餵狗就是開一個 fd、定期寫一個位元組：`exec 3>/dev/watchdog0; printf '1' >&3`（但一旦開了就得餵到底，見下）。

**第 3 步：讓它真的咬下去（bite 重置）——會重開機，看清楚再做。**（✅ 2026-07-22 板上實測，transcripts：`live/ch4-w1-wdt-verify.txt`、`ch4-w1-wdt-bite.txt`、`ch4-w1-wdt-verify2.txt`）

刻意武裝後不餵狗，就是開啟 fd、立刻關閉、且不送 magic close：

> ⚠️ **這是整機重置，不是只重啟你的行程**：下面這行會讓**整片板子**（所有核心、其他使用者與服務一起）在約 60 秒後硬體重開機；你這條 SSH／網路連線會中斷，共用板上會影響其他人。確認可承受、且沒有別人正在用這片板子，再做。

```bash
sudo sh -c '> /dev/watchdog0'    # 開啟即武裝、關閉不解除 → 60 秒後硬體重置
```

送出後 `dmesg` 立刻出現關鍵那一行：

```text
watchdog: watchdog0: watchdog did not stop!
```

這句話正是證據：fd 關了，但看門狗**沒有停**（因為不支援 magic close、關 fd 不解除），繼續倒數。約 60 秒後板子硬體重啟。重連後用開機時間對時間鏈驗證：

```bash
who -b ; awk '{print "uptime_sec="$1}' /proc/uptime
```

```text
         system boot  2026-07-22 15:12
uptime_sec=52.51
```

時間鏈完全吻合：**觸發 15:11:35 ＋ 60 秒逾時 → 約 15:12:35 咬下 → `who -b` 顯示 15:12 重新開機**。重開機後補查裝置身分，確認咬下去的正是綁 CA55 的 WDT1（transcript：`ch4-w1-wdt-verify2.txt`）：

```bash
cat /sys/class/watchdog/watchdog0/device/uevent
```

```text
DRIVER=rzv2h_wdt
OF_NAME=watchdog
OF_FULLNAME=/soc/watchdog@14400000
OF_COMPATIBLE_0=renesas,r9a09g057-wdt
```

`OF_FULLNAME=/soc/watchdog@14400000` 對上本節能力表的 `<WDT1_base>`＝`0x1440_0000`（CA55），驅動程式 `rzv2h_wdt`——確認 Linux 這條 `/dev/watchdog0` 就是 WDT1。

**驗證判準**：
- 咬下去成功的樣子：重連後 `/proc/uptime` 遠小於你的無回應等待時間（實測 52.51 秒 ≪ 觸發到重連的時距），且 `who -b` 的開機時刻 ≈ 觸發時刻 ＋ 60 秒。
- **開機 `dmesg` 不會有 reset-cause（重啟原因）那一行**——RZ/V2H BSP 不記錄重啟來源。所以「是不是看門狗咬的」要靠**重啟前**那句 `watchdog did not stop!` ＋ 時間鏈判定，不要指望開機 log 自己招認；別把「沒有 reset-cause 行」誤判成「沒咬到」。

> ⚠️ **注意（本板不支援 magic close：開了就別想乾淨收手）**
> - **情境**：你只是想 `wdctl` 或開一下 `/dev/watchdog0` 看看，或想寫 `V` 把它關掉。
> - **症狀**：`dmesg` 出現 `watchdog: watchdog0: watchdog did not stop!`，60 秒後板子無預警重開機。
> - **原因**：本板 `MAGICCLOSE=0`——寫 `V` 不會解除武裝；而「開啟即武裝」是 watchdog API 的通則，`wdctl` 為了讀能力也得開啟裝置，等於也武裝了它。一旦開過、又沒有行程持續餵狗，它一定咬。
> - **預防／處理**：把 `/dev/watchdog0` 當「開了就得負責餵到底」的資源——要嘛交給 systemd（在 `/etc/systemd/system.conf` 設 `RuntimeWatchdogSec=`，由 systemd 自動餵），要嘛你自己的行程開著它、用 `printf '1' >&3` 定期刷新；不要為了「看一眼」而開它。真的誤武裝了、又不想等重開機，只能趕快持續餵狗撐住，本板沒有乾淨的中途解除。

> ⚠️ **注意（timeout 抓在兩個時間之間；本板還不能改 60 秒）**
> - **情境**：你替長時間服務掛上 `/dev/watchdog0`，想調一個順手的逾時。
> - **症狀**：餵狗週期比 60 秒還長 → 正常執行也被誤重置；想用 `wdctl -s` 改逾時卻無效。
> - **原因**：本板 `SETTIMEOUT=0`，逾時鎖死 60 秒；且逾時是「上次刷新後多久沒再刷新」決定的。
> - **預防／處理**：把餵狗週期抓在「正常跑完一輪主迴圈的時間」與「60 秒」之間、且留足裕度（例如每 20–30 秒餵一次）；需要別的逾時值就得改走韌體或 device tree，不是 Linux 使用者空間能改的。

### 什麼情況下你會用到它

**機制**：看門狗的角色是「系統失控時的最後防線」——當主控迴圈卡死（deadlock、無窮迴圈、記憶體毀損導致行程停止回應），而且沒有其他機制能自動恢復系統時，WDT 逾時會強制重置晶片。

**判斷準則**：任何跑在無人值守、或需要長時間穩定運作場合的系統（**以工業控制器、遠端監控節點、任何自動化裝置為例**），都應該先問一句：「主迴圈卡死時，誰來讓系統恢復？」——

- 若答案是「沒有」，就該啟用對應核心的 WDT，並讓主迴圈定期刷新。
- 刷新週期要抓在「正常執行一輪主迴圈所需時間」與「你能接受的最長無回應時間」之間，太短容易誤觸發，太長會讓系統卡死太久才恢復。
- Window 功能用於偵測「刷新過早」這種異常；一般應用若不需要這麼嚴格的時序驗證，把 window 起點設 100%（不限制起點）即可。

**核心歸屬決定你走哪條路**：Linux 應用層（跑在 CA55 上）能直接透過 `/dev/watchdog0` 掛上 WDT1；但若即時控制迴圈跑在 R8／M33 上，該核心自己的 WDT（WDT2／WDT3 或 WDT0）要在**韌體裡自行刷新**，Linux 側管不到、也看不到它的狀態。這是「4 顆 WDT 各綁一顆核心」這個機制直接推出來的判斷準則：你的關鍵迴圈跑在哪顆核心，就用那顆核心的 WDT。

> 出處：官方硬體手冊 r01uh1032 **§5.4 Watchdog Timer (WDT)**（p1218–1219 概說；暫存器自 §5.4.2 p1220）。開發紀錄 doc07 §22。

---

## 7. RTC（RTCA-3，即時時鐘）

### 這是什麼（機制）

RTC（Realtime Clock，即時時鐘）本 SoC 的型號是 RTCA-3。它和前面所有計時器最本質的差別是：**它記的是「現在幾點」，不是「開機後過了多久」**——而且靠獨立的低頻石英振盪與待機供電，即使系統斷電重開機、或進入 suspend（休眠），它的計數也持續不斷。

它提供兩種計數模式，透過暫存器切換（r01uh1032 §5.3.1）：

- **Calendar count mode（日曆計數模式）**：以西元 2000–2099 共 100 年為範圍的日曆，用 BCD（Binary-Coded Decimal，二進位編碼十進位——每 4 個位元表示一個十進位數字，方便直接顯示年月日）呈現，自動處理閏年、12／24 小時制切換。
- **Binary count mode（二進位計數模式）**：不記年月日時分，只用 32-bit 二進位計數秒數，可用於非西曆的場合。（逐字：「it counts seconds, and retains the information as a serial value. This mode can be used for calendars other than the Gregorian calendar」）

時脈來源是一顆 32.768 kHz 的石英振盪（接在 `RTXIN`／`RTXOUT` 接腳上；32.768 kHz 是 RTC 的業界標準頻率，正好是 2 的 15 次方 Hz，除頻後容易得到 1 Hz），內部除頻產生 128 Hz 參考時脈。（r01uh1032 §5.3.1 Table 5.3-1；§5.3.2 Table 5.3-2）

它有 3 種中斷（r01uh1032 §5.3.1 Table 5.3-1(2/2)）：

- **Alarm（ALM，鬧鐘）**：calendar mode 下可選年／月／日／星期／時／分／秒任意欄位比對；binary mode 下可選 32-bit 計數器任一位元比對。
- **Periodic（PRD，週期）**：可選 2 秒、1 秒、1/2、1/4、1/8、1/16、1/32、1/64 或 1/128 秒為週期，共 9 檔。
- **Carry（CUP，進位）**：64 Hz 計數器進位到秒計數器時，或讀取 `R64CNT` 同時發生進位時觸發。

這三種事件都能直接輸出給 ELC（不經 CPU）。另外它有一組唯讀鏡射暫存器（base `<RTC_Read_Only_base>`），目的是避免「讀取當下正好發生進位」導致讀到不一致的值——你從鏡射區讀，就不會讀到一半被進位更新。（r01uh1032 §5.3.2 Table 5.3-3 Note 3）

### Linux 下怎麼看到它

- **裝置節點 `/dev/rtc0`**，驅動程式繫結名稱 `rtca3`（Renesas RTCA-3 binding），走標準 Linux RTC ioctl API（`RTC_RD_TIME`、`RTC_ALM_SET`、`RTC_WKALM_SET`、`RTC_AIE_ON`）。（doc07 §23）
- **板上狀態**：已驗證 `hwclock -r` 可正常運作。（unit-map 06；doc07 §23）
- **工具與用法**：

```bash
hwclock -r                        # util-linux，讀硬體時鐘
hwclock -s                        # 用 RTC 的時間設定系統時間（常放在開機腳本）
timedatectl                       # systemd 的時間管理
rtcwake -d rtc0 -m mem -s 60      # 設 RTC 鬧鐘，休眠到 RAM，60 秒後自動喚醒
```

（doc07 §23）

> ⚠️ **注意（板上有兩顆會記時間的晶片，`/dev/rtc0` 只對應其中一顆）**：
> - **情境**：你想確認 `/dev/rtc0` 到底是哪一顆硬體，或想找「另一顆 RTC」。
> - **症狀**：查資料時發現板上似乎有不只一處 RTC 功能，容易混淆。
> - **原因**：WS125 載板上另有一顆獨立的 PMIC（電源管理晶片，型號 `RAA215300A2GNP#HA2`，9-channel PMIC with RTC，見 `ws125-rdk-board-manual.md` BOM 表 U16）也內建 RTC 功能；但 `/dev/rtc0` 對應的是 **SoC 內建的 RTCA-3**（驅動程式繫結名稱 `rtca3` 已明確指出），兩者是不同晶片。
> - **預防／處理**：把 `/dev/rtc0` 認定為 SoC 的 RTCA-3。PMIC 內建的那顆 RTC 是否有獨立 Linux 驅動程式節點，開發紀錄未盤點，不可假設存在——要用先自行查證。（doc07 §23；BOM 見 WS125 RDK 載板手冊）

### 關鍵能力與限制

| 項目 | 數值／內容 | 出處 |
|---|---|---|
| 計數模式 | calendar（2000–2099，BCD，自動閏年）／binary（32-bit 秒數） | r01uh1032 §5.3.1 Table 5.3-1 |
| 時脈來源 | 32.768 kHz（`RTXIN` 外接石英），128 Hz 參考時脈 | r01uh1032 §5.3.1 |
| Alarm 比對粒度 | calendar mode 下年／月／日／星期／時／分／秒可個別選擇比對 | r01uh1032 §5.3.1 Table 5.3-1(2/2) |
| Periodic 週期 | 2 秒、1 秒、1/2、1/4、1/8、1/16、1/32、1/64、1/128 秒共 9 檔 | r01uh1032 §5.3.1 |
| 暫存器基底 | `<RTC_base>`＝`0x11C0_0800`（可讀寫）、`<RTC_Read_Only_base>`＝`0x11C0_0C00`（唯讀鏡射） | r01uh1032 §5.3.2 Table 5.3-3 |
| 實例數 | 單一實例（不像 WDT／GTM 有多通道） | unit-map 06；doc07 §23 |
| Linux 曝露 | `/dev/rtc0`（`rtca3`）；`hwclock -r` 可運作 | doc07 §23；unit-map 06 |

### 動手：讀 RTC、看它與系統時鐘／NTP 的關係，以及斷電後會發生什麼

**機制**：這片板子上其實有兩個時間在跑——**RTC**（`/dev/rtc0`，記「現在幾點」）與**系統時鐘**（核心維護，開機歸零後靠 monotonic clock 累加）。兩者的接點有兩處：開機時核心讀一次 RTC 當系統時間的初值（`hctosys`），開機後則由 NTP（`systemd-timesyncd`）持續校正系統時鐘。RTC 平時不參與系統計時，只在開機與 suspend／resume 邊界被讀寫。

**第 1 步：確認裝置身分與 hctosys。**（✅ 2026-07-22 板上實測，transcript：`live/ch4-w1-rtc.txt`）

```bash
cat /sys/class/rtc/rtc0/name
grep -H . /sys/class/rtc/rtc0/date /sys/class/rtc/rtc0/time /sys/class/rtc/rtc0/hctosys
```

```text
rtc-rtca3 11c00800.rtc
/sys/class/rtc/rtc0/date:2026-07-22
/sys/class/rtc/rtc0/time:07:06:45
/sys/class/rtc/rtc0/hctosys:1
```

`rtc-rtca3 11c00800.rtc` 對上能力表的 `<RTC_base>`＝`0x11C0_0800`，確認 `/dev/rtc0` 就是 SoC 內建的 RTCA-3（不是載板那顆 PMIC 內建 RTC，見本節上方注意框）。`hctosys:1` 是關鍵——代表**開機時核心用這顆 RTC 的時間設定了系統時鐘**（hardware clock → system）。也注意 sysfs 這裡的 `time` 是 **UTC**（07:06:45），不是本地時間。

**第 2 步：用 `hwclock`／`timedatectl` 讀，並看清 RTC 與 NTP 的分工。**（✅ 2026-07-22 板上實測，同一 transcript）

```bash
sudo hwclock -r          # /dev/rtc0 權限為 root 600，需 sudo
timedatectl
```

```text
2026-07-22 15:06:46.002293+08:00
```

```text
               Local time: Wed 2026-07-22 15:06:47 CST
           Universal time: Wed 2026-07-22 07:06:47 UTC
                 RTC time: Wed 2026-07-22 07:06:47
                Time zone: Asia/Taipei (CST, +0800)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no
```

三件事一起讀懂：
- **`RTC in local TZ: no`**——RTC 存的是 UTC。所以 `hwclock -r` 顯示的 `+08:00` 是工具幫你換算過的本地時間，硬體裡記的是 UTC。這是 Linux 建議的做法（RTC 存 UTC、時區在軟體層處理）。
- **`NTP service: active` 但 `System clock synchronized: no`**——這兩行不矛盾。NTP 服務有在跑，只是**此刻**尚未把系統標記為「已同步」。看時間源狀態就懂：

```bash
timedatectl timesync-status
```

```text
       Server: 103.186.118.217 (tw.pool.ntp.org)
Poll interval: 16h (min: 30min; max 1d)
      Stratum: 2
       Offset: +3.559ms
```

offset 只有 +3.559 ms、poll interval 已拉長到 16 小時——這是「上次成功同步後、進入低頻輪詢的正常過渡態」，不是壞掉。**別把 `synchronized: no` 誤讀成 NTP 失效。**

**第 3 步：斷電後 RTC 會怎樣——本板實情與誠實邊界。**

先分清兩種「重開機」，這是最容易誤判的地方：
- **暖重置（warm reset，電源沒斷）**：例如上一顆 WDT 咬下去重開機，RTC 域全程有電、時間不受影響。實測可證——WDT bite 重開機後（bite 事件見 `live/ch4-w1-wdt-bite.txt`），`date` 直接就是正確的 `Wed Jul 22 03:13:33 PM CST 2026`（該 `date` 讀值出自重開機後的 `live/ch4-w1-wdt-verify.txt`），沒有跳掉。
- **全斷電（拔掉電源）**：RTCA-3 要靠**獨立的待機供電**（電池／超級電容維持 `RTXIN` 石英與 RTC 域）才能在斷電期間繼續走時。這片板子在第 1 章初期設定時之所以**一定要設 NTP 自動校時**（第 1 章環境設定「設時區與 NTP 校時」明講：板上雖有 RTC，要準得靠 NTP），背後的實情就是：**不能把本板 RTC 當成「拔電也保時」的可信時源**。一旦待機供電不足，全斷電後 RTC 會失準甚至歸零，開機時戳就會是舊的或錯的。

> ⚠️ **注意（別把 RTC 當唯一可信時源）**
> - **情境**：你拔電搬動板子後重新通電，馬上看系統日誌時戳、或做需要正確時間的事（以憑證／TLS 有效期驗證為例、以多來源資料要對齊同一時間軸為例）。
> - **症狀**：開機初期時間是舊值或明顯偏差，日誌時戳對不上、憑證驗證可能因時間錯誤而失敗。
> - **原因**：全斷電期間本板 RTC 若無足夠待機供電就不走時；開機時 `hctosys` 只是忠實地把 RTC 當時（可能已失準）的值搬給系統時鐘，NTP 要連上網、跑完一輪才會把它拉回正確時間。
> - **預防／處理**：開機流程一律靠 NTP 校時（第 1 章已設）；NTP 對準後用 `sudo hwclock -w` 把正確系統時間**回寫**進 RTC，讓下次（暖）開機的初值是好的。真的需要「拔電也保時」，要自行確認載板 RTC 待機供電（電池／超級電容）確實有接且有效。

**誠實邊界**：本波板上實驗只做到「暖重置（WDT bite）後時間正確」這條的一手實測；**完整拔電、量測 RTC readback 是否歸零**的破壞性實驗未做——「本板全斷電後 RTC 具體歸零到什麼值」標**待補一手來源（全斷電 readback）**。上面的判斷準則錨在機制（待機供電是保時前提）與第 1 章「要準得靠 NTP」的設定實情，不逾越為「已量到歸零值」。

**驗證判準**：
- 第 1／2 步做對的樣子：`/sys/class/rtc/rtc0/name` 出現 `rtca3`、`hctosys` 為 `1`；`timedatectl` 的 `RTC time` 與 `Universal time` 接近（本板實測 RTC 與系統時鐘差 < 0.5 秒），且 `RTC in local TZ: no`。
- 要驗「NTP 是否真的把系統拉準」：看 `timedatectl timesync-status` 的 `Offset` 是否收在毫秒等級；持續同步後 `System clock synchronized:` 會轉為 `yes`（不同 session 狀態不同，2026-07-17 的 `live/ch01-env.txt` 即為 `yes`）。

### 什麼情況下你會用到它

**機制**：RTC 靠獨立的低頻石英振盪與待機供電維持運作，即使系統斷電重開機、或進入 suspend-to-RAM／disk，計數也能持續（前提是有電池或待機供電維持 `RTXIN` 振盪與 RTC 域供電）。這與 SYC／`arch_timer`（開機即歸零、跑在系統時脈上）本質不同：一個是「斷電也要記得現在幾點」，一個是「開機後量測經過多久」。

**判斷準則**：

- 需要「開機就知道現在的實際日期時間」（**以系統日誌要有正確時戳、憑證／TLS 要驗證有效期為例**）時，用 RTC 讀取初始時間（`hwclock -r`，或開機腳本 `hwclock -s`）。Linux 開機後系統時間的走時，仍由核心的 monotonic clock（依賴 SYC／`arch_timer`）負責；RTC 只在開機與 suspend／resume 邊界被讀寫一次，不是持續參與系統計時的元件。
- 需要「無網路連線時仍能定時喚醒」（**以低功耗週期性感測、定時任務為例**）時，RTC 的 Alarm 中斷搭配 `rtcwake` 是硬體層面在 CPU 完全休眠時仍能運作的計時喚醒手段——這是它相對 GTM／OSTM／CMTW 等其他計時器（深度 suspend 時通常隨核心域斷電）的本質差異。

**以資料記錄器為例**（需要正確時戳），或**以需要定時喚醒省電的感測節點為例**：只要涉及「斷電後仍要保留時間資訊」或「CPU 休眠時仍要能定時喚醒」，就會用到 RTC 而非其他計時器。這條判斷線對任何應用領域都成立——問的是「我需不需要跨斷電／跨休眠的時間」，而不是「我在做哪個專案」。

> 出處：官方硬體手冊 r01uh1032 **§5.3 Realtime Clock (RTC)**（p1166–1168 概說；暫存器自 §5.3.2 p1169）。開發紀錄 doc07 §23。板上另有 PMIC 內建 RTC（不同晶片）見 WS125 RDK 載板手冊 BOM（U16，`RAA215300A2GNP`）。

---

## 本群組速查

| 想做的事 | 用哪顆 | 板上怎麼碰到它 |
|---|---|---|
| 量測「經過多久」、`clock_gettime`／`nanosleep` | SYC（間接） | 標準 POSIX 時間 API；無直接介面 |
| 毫秒級定時中斷／事件（可接受排程延遲） | GTM／OSTM、CMTW（間接） | `timerfd`／`nanosleep`；核心自選 clockevent |
| 量測外部脈波間隔（input capture） | CMTW0–3（矽片有、Linux 不可達） | 目前無驅動程式；需自寫核心驅動程式或走即時核心 |
| 輸出 PWM（馬達／伺服／LED） | GPT（＋POEG 做安全閘） | 無 pwmchip；裸機 MMIO 或交給 R8／M33 |
| 功率電路故障時硬體切斷輸出 | POEG | 無驅動程式；隨 GPT 走裸機／即時核心 |
| 系統卡死自動重開機 | WDT（依核心選 WDT0–3） | CA55＝`/dev/watchdog0`；R8／M33 在自家韌體刷新 |
| 斷電保時、CPU 休眠時定時喚醒 | RTC | `/dev/rtc0`、`hwclock`、`rtcwake` |

> **一句話收束整群**：前三顆（SYC／GTM／CMTW）是系統時基，你只能透過標準 API 間接用；後四顆是應用會想直接碰的，但最搶手的 PWM（GPT＋POEG）在本板 Linux 沒有現成路徑——要嘛裸機 MMIO，要嘛交給即時核心。唯二在 Linux 有乾淨標準介面、拿來就能用的，是 WDT（`/dev/watchdog0`）與 RTC（`/dev/rtc0`）。

## 這一群的邊界與缺口（誠實標註）

- **SYC 是本群組唯一的盤點缺口**：它沒有 `/dev`／sysfs 節點，本板「啟用」的判斷是從 `arch_timer` 中斷活躍**反推**而來，不是直接量測。24 MHz、64-bit 等數字一律是官方手冊規格，引用時不可寫成「板上實測」。若後續要寫 SYC 的動手驗證小節，需先在板上補測（例如 `cat /proc/interrupts | grep arch_timer`）。
- **「64 路 PWM」是規格推算、不是本板實測**：那是 4 接腳 × 16 通道的矽片理論上限；本板實際只有 GPT0 ch0–7 這 8 個通道節點 `okay`、且無 PWM chip。寫作與引用時務必區分「矽片理論上限」與「本板 Linux 可用範圍」。
- **CMTW／GPT／POEG 的接腳能力在本板 Linux 應用層不可達**：input capture、output compare、PWM 輸出這幾項，矽片都有、但都沒有現成核心驅動程式曝露給應用層。要用只能自寫驅動程式，或走即時核心（Cortex-R8／M33）韌體——後者本板 Linux 未曝露，需先走 remoteproc／韌體載入路徑（屬別章範圍）。
- **40-pin header 的腳位對 GPT 的對應原則上已定案，但尚未逐線落實**：`PWM0`／`PWM1` 絲印確實接到 GPT（pin 32＝`PA4`→`GTIOC6A`，pin 33＝`PA7`→`GTIOC7B`；完整交叉核對與整個 header 的數量統計見 §4 的注意框）。仍未解決的是電氣面：手上沒有 RDK 電路圖，串聯電阻、準位轉換或板上既有負載可能排除特定腳位，且有 7 支 header 腳的「腳號 ↔ 埠腳」對應尚未確定。數量成立，但配線計畫要等這些補齊。
- **暫存器逐位元明細不在本檔範圍**：本群組只取官方手冊「功能概說」層級（Overview／Features／Block Diagram）。各單元的暫存器逐位元描述（`OSTMnCTL`、`GTIOR`、`GTCCRA-F`、`CMWIOR`、RTC 各暫存器…）與 Operation 章節的實際時序，若要寫暫存器層級的裸機控制教學，需另行取用手冊對應小節。
