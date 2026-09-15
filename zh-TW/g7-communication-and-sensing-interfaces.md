# g7 · 通訊與感測介面（序列／匯流排／網路／擴充／類比）

這是第 4 章〈全板硬體資源地圖〉的**群組深講參考檔**。第 4 章的總覽（00 檔 4.1）用分群總表把整顆 SoC 的功能區塊攤平，讓你一眼看到「這是什麼、板上什麼狀態、去哪深讀」；而這一份檔案，就是其中「通訊介面」與「感測／類比」兩群、每一個單元的**完整展開**。總表回答「有沒有、開沒開」，這裡回答「它到底怎麼運作、Linux 從哪裡看到它、它的能力邊界在哪、你在什麼情況下該用它」。

這一群管的是板子與「外面世界」之間的所有管道——把序列訊號、感測器匯流排、有線網路、高速擴充埠、類比輸入一次收齊，共 **14 個單元**：

- **序列類**：SCIF（開機就在的 console UART）、RSCI（多模式多通道序列）
- **匯流排類**：RSPI（SPI）、I²C（RIIC）、I3C、CAN-FD
- **運算輔助**：CRC（硬體校驗碼運算單元）
- **擴充／連接**：GPIO（PFC 腳位多工）、USB3.2、USB2.0、PCIe
- **網路**：GBETH ＋ PTP
- **感測／類比**：ADC（12-bit）、TSU（晶片溫度感測）

每個單元都用同一副骨架寫：**這是什麼**（機制，轉譯自官方硬體手冊）→ **Linux 下怎麼看到它**（裝置節點／sysfs／驅動程式，有板上實測就引用逐字證據）→ **關鍵能力與限制**（數值一律附出處頁碼）→ **什麼情況下你會用到它**（判斷準則，教你自己判斷適不適用，而不是照抄某條路徑）。每節結尾附官方出處，方便你回硬體手冊查暫存器明細。

> **佔位與範圍慣例（全檔適用）**：板子的網路位址由 DHCP 動態配發、會隨租約變動，本檔一律寫 `<板子IP>`，要連線前先在板上跑 `ip a` 以當下實際位址為準（緣由見第 4 章章首）。本群組**不涵蓋**無線鏈路／鏈路遙測相關內容——那不在本手冊的教學範圍。凡出現板上實測值，量測環境除另註外皆為：板上 Ubuntu 24.04.4 LTS（aarch64）、核心 `6.10.14-arm64-renesas`、CPU governor 鎖 `performance`；資源盤點類數字於 2026-07-17／18 板上複驗。

## 本群組單元清單

| 單元 | 一句話 | 板上狀態 |
|---|---|---|
| [SCIF](#scif單通道開機就在的-console-uart) | 單通道的非同步 UART（10 通道的 RSCI 也能設成非同步，見下列），開機韌體與核心訊息都走它 | 啟用（系統 console，`/dev/ttySC0`） |
| [RSCI](#rsci多通道多模式序列介面) | 10 通道、每通道可當 UART／同步／Simple-I²C／Simple-SPI／智慧卡／LIN | 部分啟用（僅 1 通道走 UART，其餘 9 停用） |
| [RSPI](#rspi標準-spi-主從控制器) | 標準 SPI 主／從，每通道 4 個從屬選擇腳 | 部分啟用（3 之 1，`/dev/spidev1.0`） |
| [I²C（RIIC）](#i²criic多裝置共線的低速匯流排) | 9 通道 I²C 主控，Fast-mode+ 到 1 Mbps | 啟用（4 條匯流排：i2c-3／4／8／9） |
| [I3C](#i3c向下相容-i²c-的新世代匯流排) | MIPI I3C，向下相容 I²C，多了 in-band 中斷與 hot-join | 停用／未接線（矽片存在，DT `disabled`） |
| [CAN-FD](#can-fdrs-canfd確定性抗雜訊的多節點匯流排) | 6 通道 CAN-FD，資料段最高 8 Mbps | 啟用但連結 DOWN（收發器由 GPIO 待命） |
| [CRC](#crc-運算單元硬體校驗碼未盤點缺口) | 硬體 CRC 運算單元，5 種多項式可選 | 未知（未盤點）——只引手冊規格 |
| [GPIO（PFC）](#gpiopfc腳位多工的切換總機) | 腳位功能多工器，通用 I/O ＋ 15 種特定功能切換 | 啟用（`/dev/gpiochip1`，96 lines） |
| [USB3.2](#usb32-gen2host超高速外接埠) | 2 個 USB3 host，最高 10 Gbps | 啟用（host-only，`xhci-renesas`） |
| [USB2.0](#usb20一般速度外接埠含-otg-矽片) | 1 個 OTG/DRD ＋ 1 個純 Host（矽片通道，非實體埠） | 啟用（ch0 矽片支援 OTG，本板設 Host） |
| [GBETH ＋ PTP](#gbeth--ptp有線乙太網路與硬體時間戳) | 2 通道 GbE，內建 IEEE 1588 硬體時間戳 | ch0＝`end0` UP；ch1 停用 |
| [PCIe](#pcie-30高速擴充匯流排) | Gen3，可切 Root Complex 或 Endpoint | 啟用為 Root Complex（無 endpoint） |
| [ADC](#adc12-bit-類比數位轉換器) | 12-bit 逐次逼近 ADC，最多 8 通道 | 啟用且作動（`iio:device0`） |
| [TSU](#tsu晶片溫度感測單元) | 2 組晶片內建溫度感測器 | 啟用且作動（2 個 thermal zone） |

---

## SCIF（單通道，開機就在的 console UART）

### 這是什麼

SCIF（Serial Communications Interface with FIFO）是這顆晶片上那顆**單通道**的非同步 UART 控制器（另有 10 通道的 RSCI 也能設成非同步 UART，見下一節）。UART（Universal Asynchronous Receiver/Transmitter，通用非同步收發器）是最古老、最基本的序列通訊方式：兩條線（TX 送、RX 收），雙方事先約好一個傳輸速率（鮑率，baud rate），一個位元一個位元地把資料串出去，沒有共用時脈線——「非同步」指的就是這件事。

SCIF 比最陽春的 UART 多了一組 **16 段深度的 TX／RX FIFO**（First-In-First-Out 緩衝佇列）。有了 FIFO，硬體可以先替你收好或存好一整批位元組再一次通知 CPU，連續高速通訊時 CPU 就不必每收發一個位元組就被打斷一次（r01uh1032 §7.4，p2780）。它有獨立的鮑率產生器可調速率，資料可選 LSB-first 或 MSB-first 送出。中斷來源共 6 種：傳輸結束（TEIF）、傳輸 FIFO 空（TXIF）、接收 FIFO 滿（RXIF）、接收資料就緒（DRIF）、接收錯誤（ERIF）、break 偵測或 overrun（BRIF）（§7.4.1，p2780）。

SCIF 之所以在這片板子上地位特殊，是因為它同時是晶片的**開機下載來源之一**（boot mode 3 = SCIF download）。也就是說，這顆 UART 與開機韌體共用同一塊硬體——這正是它天生就是「系統主控台（console）」的原因：從 u-boot 到 Linux 核心的每一行開機訊息，都是從這一路吐出來的。

### Linux 下怎麼看到它

在 Linux 裡，SCIF 是 `/dev/ttySC0`，驅動程式 `sh-sci`（device tree compatible 字串 `renesas,scif`）。它被設為系統 console，所以你用序列線接上這一路，就能看到開機全過程與登入提示。暫存器基底位址是 `0x11C0_1400`（板上狀態與位址出處：`07-hardware-unit-usage-guide.md` §25，該文引 r01uh1032 Table 7.4-3）。

板上實測可見這個節點確實存在（✅ 2026-07-17，transcript：live/ch04-followup.txt）：

```text
crw-rw---- 1 root dialout 204,  8 Jul 17 22:38 /dev/ttySC0
```

主／次裝置號 `(204, 8)`。它掛在 `dialout` 群組下，一般使用者要存取序列埠需加入該群組或用 `sudo`。

### 關鍵能力與限制

- **通道數：1 channel**——逐字「Channel: 1 channel」（p2780 Table 7.4-1）。全晶片只有這一路 SCIF，無法擴充。
- **FIFO 深度**：逐字「16-stage FIFO buffers for transmission and reception」（p2780）。
- **通訊模式**：僅非同步（Asynchronous communication），**沒有**同步模式（p2780）。要時脈同步的序列通訊得改用 RSCI 或 RSPI。
- **資料格式**：資料長度 7 或 8 bit；停止位 1 或 2 bit；同位可選偶／奇／無（p2780 Table 7.4-1）。

### 什麼情況下你會用到它

**機制上**，SCIF 與開機韌體共用硬體、開機就啟用、永遠在線——所以只要你需要一條「開機就存在、任何時候都可用」的除錯／console 序列埠，它天生就是答案。多數 SoC 開發板的主控台都走這一路，道理就在這裡。

**判斷準則**：

- 如果你要的只是**一條** console 級的除錯序列埠，SCIF 已經夠用，而且不必做任何設定就有。
- 如果你的應用需要**多路**序列通訊（例如同時接好幾個各自輸出序列資料的感測器，以工業現場的多顆序列儀表為例），SCIF 只有 1 個通道、無法擴充，這時該看下一節的 RSCI。
- 如果你只是想「借」console 埠來傳應用資料，要先想清楚它同時也是系統訊息的輸出口——核心的 log 會和你的資料在同一條線上互相干擾，除非你在核心命令列關掉 console 到這個埠。

> 官方出處：硬體手冊 `r01uh1032` §7.4 SCIF（p2780–2781，功能概說；暫存器明細自 §7.4.2 p2782 起）。板上狀態與位址：`07-hardware-unit-usage-guide.md` §25。

---

## RSCI（多通道、多模式序列介面）

### 這是什麼

RSCI（Renesas Serial Communications Interface，手冊內文簡稱 SCI）是一組**通用序列介面群**，提供 **10 個獨立通道**。它和 SCIF 最大的差別有兩點：通道多（10 對 1），而且每個通道都是「變形金剛」——可以被設定成以下**六種模式**之一（r01uh1032 §7.3.1，p2585）：

1. **非同步（UART/ACIA）**——就是一般 UART。
2. **8-bit 時脈同步**——比非同步多一條時脈線，收發雙方靠共用時脈對齊，抗抖動比純非同步好。
3. **Simple I²C**（僅 master）——用序列通道模擬 I²C 主控。
4. **Simple SPI**——用序列通道模擬 SPI。
5. **智慧卡介面**——相容 ISO/IEC 7816-3 的電氣訊號與傳輸協定。
6. **Simple LIN**——車用 LIN 匯流排。

每個通道都有 FIFO 緩衝器可做全雙工連續傳輸，傳輸速率由**各自獨立**的鮑率產生器決定（p2585）。這種「一個通道可以扮演多種人格」的設計，讓你在腳位吃緊時，可以用同一顆硬體區塊去頂替 I²C 或 SPI 的角色，而不必動用專屬的 RIIC／RSPI 通道。

### Linux 下怎麼看到它

矽片上有 10 個通道，但這片板子的 device tree **只把 1 個通道**設為 `okay`：RSCI6（暫存器位址 `0x1280_2400`），而且是以**非同步 UART 人格**運作，驅動程式 `rz-sci`。其餘 9 個通道在 device tree 裡皆為 `disabled`（出處：`07-hardware-unit-usage-guide.md` §26；`06-hardware-resource-map.md`）。

板上這個被啟用的通道，實測曝露為**兩個** `/dev/ttyS*` 節點（✅ 2026-07-17，transcript：live/ch04-followup.txt）：

```text
crw-rw---- 1 root dialout   4, 64 Jul 17 22:34 /dev/ttyS0
crw-rw---- 1 root dialout   4, 65 Jul 17 22:34 /dev/ttyS1
```

主號 `4`、次號 `64`／`65`（連號）。要留意：板上目前只用到它的 **UART 人格**，沒有啟用 Simple SPI 或 Simple I²C 人格——那些人格需要改 device tree、重新設定腳位多工才會出現。

### 關鍵能力與限制

- **通道數**：逐字「Number of channels: 10 channels」（p2585 Table 7.3-1）。板上只開 1 個。
- **Simple I²C 模式**：只能當 master，逐字「Transfer rate: Up to 400 kbps」（p2587）。
- **Simple SPI／同步模式**：傳輸／接收可選 1-stage 暫存器或 **32-stage FIFO**（p2586）。
- **非同步模式資料長度**：7、8 或 9 bit（p2586）——比 SCIF 多一個 9-bit 選項（某些多節點協定會用第 9 個位元當位址／資料旗標）。

### 什麼情況下你會用到它

**機制上**，RSCI 是「多路、多模式」的序列介面群，強項是彈性：一個硬體區塊可以是 UART、也可以是輕量的 SPI／I²C。

**判斷準則**：

- 只需要 **1 條** console 級 UART → 用 SCIF（開機就有，不必改 DT）。
- 需要 **2 條以上**獨立的序列連線（例如同時接數個各自輸出 UART 資料的模組，以多顆序列 GNSS／序列電池管理板為例）→ 這時 SCIF 不夠，要把對應的 RSCI 通道從 `disabled` 打開。
- 需要 **Simple SPI／Simple I²C／LIN** 這種輕量匯流排、卻又不想占用專屬的 RSPI／RIIC 通道 → 用 RSCI 的對應人格。
- 用它之前必須先改 device tree 把目標通道設 `okay`，並設定腳位多工（PFC）把該通道的訊號拉到實體腳位上——這是啟用任何額外 RSCI 通道的共同前置。

> 官方出處：硬體手冊 `r01uh1032` §7.3 Serial Communications Interface (RSCI)（p2585–2588，功能概說；暫存器明細自 §7.3.2 p2589 起）。板上狀態：`07-hardware-unit-usage-guide.md` §26、`06-hardware-resource-map.md`。

---

## RSPI（標準 SPI 主／從控制器）

### 這是什麼

RSPI（Renesas Serial Peripheral Interface）是完整規格的 SPI（Serial Peripheral Interface）主／從控制器。SPI 是一種**同步、全雙工**的匯流排：主端出一條時脈線（RSPCK），資料在 MOSI（主出從入）與 MISO（主入從出）兩條線上隨時脈同步移動，再加一條 SSL（Slave Select，從屬選擇）挑出當前要對話的從裝置。因為有時脈線且點對點，SPI 通常比 I²C 快得多。

RSPI 支援 4 線（MOSI／MISO／RSPCK／SSL）全雙工，或 3 線的時脈同步模式，內建 **32-bit × 16-stage** 的收發 FIFO，傳輸位元長度可選 4～32 bit（r01uh1032 §7.5.1，p2826）。它同時支援兩種協定慣例：Motorola SPI 模式與 TI SSP（Synchronous Serial Protocol）模式。特別實用的是每個通道有 **4 個從屬選擇腳（SSL0～SSL3）**，主模式下可各自控制 RSPCK 前後的延遲時間——這讓你能在同一條匯流排上串接多顆時序要求略有差異的 SPI 從裝置（§7.5.1.1，p2826–2827）。

### Linux 下怎麼看到它

板上啟用 1 條 RSPI——RSPI0（device tree 節點 `spi@12800000`），對外曝露為 `/dev/spidev1.0`，驅動程式 `renesas_spi_v2h`。RSPI1、RSPI2 的節點皆為 `disabled`（出處：`07-hardware-unit-usage-guide.md` §27；`06-hardware-resource-map.md` §2）。

板上實測（✅ 2026-07-17，transcript：live/ch04-followup.txt）：

```text
crw-rw---- 1 root spi     153,  0 Jul 17 22:34 /dev/spidev1.0
```

中斷也已註冊——`/proc/interrupts` 裡看得到 `12800000.spi:rx`／`:tx`／`:cend` 三條中斷線（沒接負載時計數為 0，是正常的）。`spidev` 是 Linux 讓使用者空間直接收發 SPI 訊框的通用介面，你用 `open("/dev/spidev1.0")` ＋ `ioctl(SPI_IOC_MESSAGE)` 就能收發。

### 關鍵能力與限制

- **通道數**：3 channels（p2826）。板上只開 RSPI0。
- **主模式位元率**：由 `RSPI_n_TCLK` 除頻 **2～4096** 產生（p2826）——實際上限取決於 TCLK 與除頻值。另有一份板上速查（`04-hardware-quickref.md`）把實用位元率概括為約 **50 Mbps**；手冊本身給的是上述除頻式定義，引用絕對速率時請以你的時脈設定實算為準。
- **錯誤偵測**：mode fault、underrun、overrun、parity 四種（p2826）。
- **SSL 腳位**：每通道 4 支（SSL0～SSL3），主模式下皆為輸出（p2827）。

### 什麼情況下你會用到它

**機制上**，SPI 是「快、但接線與定址彈性較差」的匯流排——沒有 I²C 那種靠位址在共線上定址的能力，改用實體 SSL 腳一對一挑裝置。

**判斷準則**：

- 對端裝置支援標準 SPI、而且你**速度優先於腳位數量**（例如高取樣率 ADC 模組、顯示控制晶片、外接快閃記憶體，以高速資料擷取前端為例）→ 選 RSPI。
- 本板已啟用的 RSPI0 有 4 個 SSL 腳，可掛 4 顆**同一條匯流排、共用同一組時脈**的 SPI 從裝置。
- 只有當你需要**多條各自獨立時脈**的 SPI 匯流排（例如兩顆裝置要求不同的 SPI 模式或速率、又不方便共用同一條匯流排）時，才需要改 device tree 開啟 RSPI1／RSPI2。判斷點是「這些從裝置能不能共用一組時脈與匯流排」——能，就一條 RSPI0 加多個 SSL 解決；不能，才多開通道。

> 官方出處：硬體手冊 `r01uh1032` §7.5 Serial Peripheral Interface (RSPI)（p2826–2829，功能概說；暫存器明細自 §7.5.2 p2830 起）。其他文件：FSP RSPI 手冊（PMOD 通道對映，本手冊尚未逐份開檔驗證）。板上狀態：`07-hardware-unit-usage-guide.md` §27、`06-hardware-resource-map.md` §2。

---

## I²C（RIIC）（多裝置共線的低速匯流排）

### 這是什麼

RIIC（Renesas I²C Bus Interface）是 9 通道的 I²C 主控制器，相容 NXP I²C 匯流排介面的功能子集。I²C（Inter-Integrated Circuit）是一種**兩線、多裝置共線**的匯流排：只用 SCL（時脈）與 SDA（資料）兩條線，匯流排上每個裝置有自己的位址，主端靠位址挑出對話對象。因為省線又能掛很多裝置，它是感測器與控制晶片最常用的介面。

RIIC 的 master／slave 皆可選，並支援**多主機（multi-master）仲裁**：當多個主機同時想發起傳輸，控制器會偵測匯流排上的實際電位是否與自己內部訊號一致——不一致就代表被別的主機蓋過去了，於是自動判定自己輸掉仲裁、退讓（r01uh1032 §7.7.1，p2968–2969）。start／restart／stop 條件由硬體自動產生與偵測，位址可選 7-bit 或 10-bit 格式（p2968）。SCL／SDA 兩線都有可程式化濾波視窗的數位雜訊濾波器（p2969）。

### Linux 下怎麼看到它

板上啟用 **4 條** I²C 匯流排，各有分工（驅動程式 `i2c-riic`；出處：`07-hardware-unit-usage-guide.md` §28、`06-hardware-resource-map.md` §2.1、`04-hardware-quickref.md`）：

| 匯流排 | 接了什麼 | 說明 |
|---|---|---|
| `i2c-3` | 顯示／HDMI 橋接晶片 ADV7535（@0x3d） | 系統顯示路徑 |
| `i2c-4` | 相機 TEVS（@0x48）＋ IMU LSM6DSO16IS（@0x6a） | 感測與相機 |
| `i2c-8` | 時脈產生器 VersaClock 3S（5L35023B-616NLGI8，U30；板卡手冊 Page 11(板卡手冊) BOM。@0x69 位址為開發文件所載、因不掃 i2c-8 未實測）＋ 供電 PMIC RAA215300 | **走 RIIC8，屬 always-on 電源域，開機必要，勿碰** |
| `i2c-9` | 未接裝置 | **空閒，留給你擴充** |

板上實測——`/dev/i2c-4` 節點 `(89, 4)` 存在（✅ 2026-07-17，transcript：live/ch04-cpu-periph.txt）；讀 i2c-4 上 IMU 的 WHO_AM_I 暫存器可驗證裝置活著：

```bash
i2cget -y 4 0x6a 0x0f    # 讀 LSM6DSO16IS 的 WHO_AM_I
```

逐字回 `0x22`（✅ 2026-07-17，transcript：live/ch04-cpu-periph.txt；`i2cget` 需先 `sudo apt install i2c-tools`，乾淨映像檔未預裝，否則會 command not found）。回 `0x22` 就代表 i2c-4 上的 IMU 認得出來。這個 `0x22` 是 2026-07-17 的參考讀值，不是保證的當下結果：i2c-4 上的相機（@0x48）與 IMU（@0x6a）模組可能正在送修、當下不在匯流排上（例如 2026-07-22 的 live/ch4-w1-i2c.txt 掃描即查無這兩個位址），此時你跑上面的指令會讀不到東西——那不是你做錯了，而是模組沒接上。

> ⚠️ **注意（i2c-8 上的裝置千萬別亂碰）**
> - **情境**：你想掃描或讀寫 `i2c-8` 上的裝置（例如 `i2cdetect -y 8` 之後手癢去寫）。
> - **症狀**：可能讓整塊板子斷電，或時脈跑掉、系統當掉。
> - **原因**：`i2c-8`（走 RIIC8，位於 always-on 電源域）掛的是供電 PMIC 與系統時脈產生器，這兩顆是供電與時脈的關鍵路徑。
> - **預防／處理**：不要動 `i2c-8` 上的任何裝置。要接自己的 I²C 感測器，用空著的 **`i2c-9`**。

### 關鍵能力與限制

- **通道數**：逐字「9 channels (RIIC0-7 in the PD_OTHERS domain, RIIC8 in the PD_AWO domain)」（p2968）。注意 RIIC8 獨自落在 always-on（AWO）電源域——這是為什麼它被拿來掛開機必要的時脈／供電晶片。
- **傳輸速率**：逐字「Fast-mode+ supported, up to 1 Mbps」（p2968）。
- **雜訊抑制**：SCL／SDA 皆有數位雜訊濾波器，濾波視窗可程式化（p2969）。

### 什麼情況下你會用到它

**機制上**，I²C 是低速、多裝置共線的匯流排——接線只要 2 條線加位址定址，適合連接對速度要求不高、但**裝置數量多**的感測器或控制晶片。

**判斷準則**：

- 裝置支援的最高速率若在 **1 Mbps（Fast-mode+）以內**，而且你重視**接線精簡**勝過吞吐量（以慣性測量單元、電源管理晶片、時脈產生器、環境感測器為例）→ 用 I²C／RIIC。
- 需要更高吞吐、且是點對點 → 改看 RSPI（SPI 沒有 1 Mbps 這道天花板）。
- 本板 `RIIC8`（＝`i2c-8`）位於 always-on 電源域，是專門保留給「開機階段就要存取」的裝置；使用者自己的感測器建議接一般電源域的匯流排（首選空著的 `i2c-9`），以免和開機流程互相牽動。這條準則對任何 RZ/V2H 使用者都成立：把「系統關鍵匯流排」與「使用者擴充匯流排」分開，避免手滑打到供電路徑。

### 動手：安全掃描 I²C 匯流排（含 i2c-8 禁區）

I²C 是共線匯流排，接上感測器後第一件事通常是「掃一下匯流排、確認裝置的位址有沒有回應」。掃描工具是 `i2cdetect`（`sudo apt install i2c-tools`，乾淨映像檔未預裝，缺了會 command not found）。這一節先講清楚「怎麼掃才安全」，再帶你把空匯流排與有裝置的匯流排讀懂。

**機制上有一個必須先知道的坑**：`i2cdetect` 預設的探測法會依位址範圍**混用「讀取位元組」與「寫入」**兩種試探——對某些位址它會發一個 write 動作去試裝置在不在。對一般感測器這頂多是無害的空寫，但對**電源管理／時脈這類「被寫到就會出事」的裝置，一個試探性寫入就足以讓板子斷電或時脈跑掉**。避開的方法是加 `-r`：強制只用「讀取位元組」探測，不對裝置寫入（`-r` 的語意出自 i2c-tools，非晶片手冊）。`-y` 則是略過互動確認。

**第一步：列出板上有哪些 I²C 匯流排。**

```bash
i2cdetect -l | sort -V
```

板上實測（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-i2c.txt）：

```text
i2c-3	i2c       	Renesas RIIC adapter            	I2C adapter
i2c-4	i2c       	Renesas RIIC adapter            	I2C adapter
i2c-8	i2c       	Renesas RIIC adapter            	I2C adapter
i2c-9	i2c       	Renesas RSCI I2C adapter        	I2C adapter
```

留意最後一列：`i2c-9` 的介面卡名稱是 **`Renesas RSCI I2C adapter`**——它不是 RIIC 通道，而是由 RSCI 的 Simple-I²C 人格提供的一條 I²C 匯流排（見本檔 RSCI 節），一樣可以當一般 I²C 匯流排掃描與使用。`i2c-3/4/8` 才是 RIIC。

**第二步：掃一條「確定安全」的使用者匯流排。** 空著留給擴充的 `i2c-9` 是最安全的練習對象：

```bash
i2cdetect -y -r 9
```

板上實測回一張全空的網格（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-i2c.txt）：

```text
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
...
70: -- -- -- -- -- -- -- --
```

**怎麼判讀這張網格**（三種符號）：

- `--`＝這個位址沒有任何裝置回應（沒 ACK）。整張全 `--` 就代表這是一條**空匯流排**——`i2c-9` 目前沒接東西，屬正常。
- 一個十六進位數字（如 `48`）＝有裝置在該位址回應了 ACK，數字就是它的 7-bit 位址。
- `UU`＝該位址有裝置、但已被某個核心驅動程式綁定占用（busy），`i2cdetect` 不去打擾它。看到 `UU` 是「有東西且已被系統接管」，不是壞掉。

**第三步：掃一條有分工的匯流排，學會看「該在卻不在」。** `i2c-4` 是相機／IMU 那條：

```bash
i2cdetect -y -r 4
```

板上實測（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-i2c.txt）：

```text
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: 10 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
...
60: -- -- -- -- 64 -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

`0x10` 與 `0x64` 兩個位址有裝置回應（這條匯流排上仍在線的其他裝置，本節不臆測其身分）；但本檔 I²C 分工表寫的相機 TEVS `@0x48` 與 IMU `@0x6a` 這兩個位址是 `--`——**因為這兩顆器材目前送修、實體不在板上**。這正是「該在卻不在」的教學價值：掃描結果對照你預期的位址表，缺了哪個就代表那顆裝置沒上線（沒插好、沒供電、或如本例被移走）。

> ⏸ **相機／IMU 讀取實驗暫緩**：本檔前面示範過用 `i2cget -y 4 0x6a 0x0f` 讀 IMU 的 WHO_AM_I 得 `0x22`（那是器材在板時的實測）。2026-07-22 這兩顆送修、`0x48/0x6a` 皆不在線上，故不重跑該讀值，相關讀取實驗待器材歸位再補。

**第四步（其實是「不做」）：i2c-8 絕對不掃。** 這一步的正確動作是**什麼都不對它做**：

```bash
# i2c-8（PMIC／時脈，always-on 電源域）——連掃描都不做
```

> ⚠️ **陷阱框（對 i2c-8 手癢就可能讓板子斷電）**
> - **情境**：你想「順手」對 `i2c-8` 跑 `i2cdetect`，看看上面掛了什麼。
> - **症狀**：板子可能瞬間斷電、時脈跑掉、系統當掉——而且是不可逆的當場失效。
> - **原因**：`i2cdetect` 預設探測會對部分位址發**寫入**試探；`i2c-8` 掛的是供電 PMIC 與系統時脈產生器，任何一個試探性寫入打到它的暫存器，就可能切掉自己的電或時脈。
> - **預防／處理**：**永不掃描 i2c-8**（連加了 `-r` 也不掃——這是本板的絕對禁區）。要接自己的感測器，用空著的 `i2c-9`；掃任何使用者匯流排都一律加 `-r` 用讀取探測。

**驗證判準**：`i2cdetect -y -r 9` 回全 `--` ＝ 空匯流排、工具正常運作；`i2cdetect -y -r 4` 在 `0x48/0x6a` 回 `--` ＝ 你的相機／IMU 目前不在線上（本例送修中）。整個過程你不會、也不該看到任何一條指令碰到 `i2c-8`。

> 官方出處：硬體手冊 `r01uh1032` §7.7 I2C Bus Interface (RIIC)（p2968–2972，功能概說；暫存器明細自 §7.7.2 p2973 起）。板上狀態：`07-hardware-unit-usage-guide.md` §28、`06-hardware-resource-map.md` §2.1。

---

## I3C（向下相容 I²C 的新世代匯流排）

### 這是什麼

I3C 是 MIPI 聯盟制定的新世代兩線匯流排，1 通道，同時相容 NXP I²C 子集與 **MIPI I3C-Basic v1.0** 協定（r01uh1032 §7.8.1.1，p3061）。它的設計目標是「用 I²C 的兩條線，換到接近 SPI 的效率，再加上 I²C 沒有的進階能力」。

除了向下相容傳統 I²C（Fast-mode 400 kbps／Fast-mode Plus 1 Mbps）之外，I3C 原生的 **SDR（Single Data Rate）模式**提供更結構化的傳輸：private message、broadcast message（common command code）、direct message（common command code）。更關鍵的是三項 I²C 沒有的能力（p3062）：

- **In-band interrupt**：從裝置可以直接在資料線上發中斷，**不必再拉一條額外的中斷腳**——省接線、省腳位。
- **Master-ship request**：次要主機可以請求接管匯流排。
- **Hot-join**：裝置可以在匯流排運作中動態加入。

### Linux 下怎麼看到它

矽片上有這顆硬體，但 device tree 節點是 `disabled`，板上也沒接任何 I3C 目標裝置。所以你**不會**看到 `/dev/i3c-*`，也**不會**有 `/sys/bus/i3c` 項目（出處：`07-hardware-unit-usage-guide.md` §29；`06-hardware-resource-map.md` §2.3）。要說明的是：Linux 核心本身**有** I3C 子系統與 Renesas I3C master 驅動程式，只是這片板子沒有把它綁定——這屬於第 4 章 00 檔〈先學會讀「狀態」〉那張六格狀態表裡的**停用**，不是「未搭載」，也不是驅動程式缺失。

### 關鍵能力與限制

- **通道數**：1 channel（p3061）。
- **相容協定速率**（I²C 相容表，p3061）：Standard-mode 0–100 kbps／Fast-mode 0–400 kbps／Fast-mode Plus 0–1 Mbps／High-speed mode 0–3.4 Mbps；I3C legacy I²C 訊息模式支援 Fm／Fm+（p3062）。
- **位址格式**：7-bit 位址（p3062）。
- **時脈延展（clock stalling）**能力（p3062）。

### 什麼情況下你會用到它

**機制上**，I3C 是「I²C 的接線、更多的能力」。但它的能力只有在**對端裝置也支援 I3C** 時才有意義。

**判斷準則**（很直接）：

- 你要接的裝置**只支援 I²C** → 直接用 RIIC，不必碰 I3C。
- 裝置本身支援 I3C，而且你需要它額外提供的能力——更高速率、in-band 中斷省一條中斷線、hot-join 動態加入 → 這時 I3C 才划算。這種情況常見於新一代感測器（例如某些新款 IMU 同時支援 I²C 與 I3C 兩種介面）。
- 本板出廠時這個介面未接線、也未在 device tree 啟用。要用之前得自行：改 DT 把節點打開、用 PFC 設定腳位多工、外接支援 I3C 的裝置——三者缺一不可。

> 官方出處：硬體手冊 `r01uh1032` §7.8 I3C Bus Interface (I3C)（p3061–3064，功能概說；暫存器明細自 §7.8.3 p3065 起）。板上狀態：`07-hardware-unit-usage-guide.md` §29、`06-hardware-resource-map.md` §2.3。

---

## CAN-FD（RS-CANFD）（確定性、抗雜訊的多節點匯流排）

### 這是什麼

RS-CANFD 是 6 通道的 CAN-FD 控制器，符合 **ISO 11898-1 (2015)** 標準，同一顆控制器可收發傳統 CAN 訊框與 CAN-FD 訊框（識別碼可用標準 11-bit 或延伸 29-bit）（r01uh1032 §7.9.1，p3363）。CAN（Controller Area Network）源自車用匯流排，強項是**多節點、抗雜訊、確定性**：所有節點掛在同一對差動線上，靠識別碼的優先權做非破壞性仲裁，即使匯流排上有雜訊或碰撞，也有硬體層級的仲裁與錯誤重傳機制。CAN-FD（Flexible Data-rate）在此之上把資料段的速率與長度都放寬，吞吐更高。

控制器內建可設定的訊息緩衝區（手冊列出「個別緩衝區」與「6 通道共用緩衝區」兩類，數量可依需求分配），接收端支援**最多 768 條過濾規則**，可把符合規則的訊息路由到指定緩衝區或 FIFO（p3363）。匯流排離線（bus-off）後的復原方式可選 ISO 11898 標準流程，或由程式介入強制恢復（p3364）。

### Linux 下怎麼看到它

CAN 在 Linux 裡是**網路裝置**（SocketCAN），不是 `/dev` 字元裝置。板上有 `can0`／`can1` 兩個 netdev（驅動程式 `rcar_canfd`），目前連結狀態皆為 **DOWN**（板上實測，`can0`／`can1` 都是 `DOWN <NOARP,ECHO>`）。

這兩個 netdev 對應的是硬體的哪兩個通道？從 `/proc/interrupts` 看得出來——註冊的中斷線是 `canfd.ch0_err`／`ch0_trx` 與 `canfd.ch3_err`／`ch3_trx`（外加共用的 `canfd.g_err`／`g_recc`），也就是說**啟用的是硬體 CAN 通道 0 與通道 3**，Linux 依序列舉成 `can0`／`can1`。這對應 WS125 RDK 載板的佈線：載板實際使用 **TCAN1046** CAN-FD 收發器晶片，透過兩組 3-pin 接頭 J2／J3 對外接出。這裡要分清兩組不同的腳位——**資料腳**是 **P80／P81＝CAN_CTX0/CRX0**（ch0 TX/RX）與 **P86／P87＝CAN_CTX3/CRX3**（ch3 TX/RX），由 PFC 多工成 CAN 功能（板上 `gpioinfo` 顯示 P80/P81/P86/P87 為 `kernel [used]`，transcript：live/ch4-w1-gpio.txt）；**收發器待命（standby）控制腳**則是另外兩支 GPIO——**PA2＝`can0_stb`、PA3＝`can3_stb`**（同一 transcript，兩者為 `output active-high [used]`，開機時已被核心 gpio-hog 占用）。別把資料腳誤當 standby 腳。其餘幾個 CAN 通道矽片上存在、但這片載板沒有全部佈線接出。

要把 CAN 帶起來，除了 `ip link` 之外，還有一個容易漏掉的前置步驟——**先解除收發器待命**：

> ⚠️ **注意（CAN 帶不起來，因為收發器在 standby）**
> - **情境**：你 `ip link set can0 up` 想把 CAN 帶起來接控制器節點。
> - **症狀**：介面帶起來後仍是 DOWN，怎麼也通訊不了。
> - **原因**：CAN 收發器由 GPIO（本板 `can0` 對應的 standby 腳）保持在 standby，實體層根本沒通電。這是**載板收發器外部電路的設計**，不是控制器本身的限制。
> - **預防／處理**：先用 GPIO 拉低對應的 standby 腳（`can0_stb`＝PA2／`can3_stb`＝PA3）解除待命（例如 `gpioset $(gpiofind PA2)=0`），再帶起介面。**注意 PA2／PA3 開機時已被核心 gpio-hog 占用**（`gpioinfo` 顯示 `[used]`，transcript：live/ch4-w1-gpio.txt）——直接 `gpioset` 會回 `Device or resource busy`，此時得先釋放該 hog，或改由擁有它的 device tree／驅動路徑控制，不能硬搶（出處：`07-hardware-unit-usage-guide.md` §30；`04-hardware-quickref.md`）。

### 關鍵能力與限制

- **通道數**：Six channels（p3363）。板上接出 2 個（硬體 ch0／ch3）。
- **傳統 CAN 速率**：Classical CAN mode 最高 1 Mbps（p3363）。
- **CAN-FD 速率**：逐字「Nominal bit rate: Max. 1 Mbps／Data bit rate: Max. 8 Mbps」（p3363）——仲裁段仍是 1 Mbps，加速的是資料段（最高 8 Mbps）。
- **錯誤狀態監控**：可讀取錯誤計數器，監控 stuff／form／ACK／CRC／bit／ACK-delimiter 等協定錯誤（p3364）。

### 什麼情況下你會用到它

**機制上**，CAN-FD 的價值在於**確定性 ＋ 抗雜訊 ＋ 多節點仲裁**——匯流排上多個節點爭用時有硬體仲裁，出錯有硬體重傳。這是它比 UART／SPI 適合「電氣環境惡劣、又要多節點可靠通訊」的原因。

**判斷準則**：

- 需要與其他控制器節點做確定性、抗雜訊的多節點匯流排通訊（以馬達驅動器、伺服節點、分散式感測器節點為例；移動載具上的 DroneCAN／UAVCAN 也是一種應用）→ CAN-FD 是常見選擇。
- 對端只支援傳統 CAN → 用 classical CAN 模式（最高 1 Mbps）維持相容。
- 對端也支援 CAN-FD、且你需要更高的資料吞吐 → 切到 FD 模式（資料段最高 8 Mbps）。判斷點就是「對端支不支援 FD」與「你需不需要那多出來的資料段頻寬」。
- 不論走哪種模式，使用前都必須先解除收發器待命（見上面注意框）——這是本板外部電路的固定前置。

### 動手：把 CAN-FD 介面帶起來（bitrate 機制與收發現況）

CAN 在 Linux 是 SocketCAN 網路裝置，帶起來的動作是 `ip link`，不是開 `/dev`。這一節帶你把 `can0` 正確帶到 UP，過程中你會撞到一個很典型的「設定契約」坑，也會看到本板收發能力的誠實邊界。

**第一步：開工前先看狀態。**

```bash
ip -br link | grep can
ip -details link show can0
```

板上實測（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-can-pre.txt）：

```text
can0             DOWN           <NOARP,ECHO>
can1             DOWN           <NOARP,ECHO>
```

`ip -details link show can0` 會吐出控制器的時序能力，其中幾個數字待會要用到（逐字節錄）：

```text
    can <FD> state STOPPED (berr-counter tx 0 rx 0) restart-ms 0
	  rcar_canfd: tseg1 2..128 tseg2 2..32 sjw 1..32 brp 1..1024 brp_inc 1
	  rcar_canfd: dtseg1 2..16 dtseg2 2..8 dsjw 1..8 dbrp 1..256 dbrp_inc 1
	  clock 80000000 ... parentdev 12440000.can
```

`can <FD>` 代表這是 FD 能力的控制器；`clock 80000000` 是 80 MHz 的 CAN 時脈（bitrate 換算的基準）；上排是仲裁段（nominal）時序範圍，下排 `dtseg/dbrp` 是資料段（data）時序範圍——**FD 之所以要兩組時序，就是因為仲裁段與資料段各自有一套 bit-timing**。

**第二步：確認收發工具。** 常用的 `candump`／`cansend` 來自 `can-utils`，但乾淨映像檔沒預裝：

```bash
which candump cansend      # 本板回空 → 未安裝
sudo apt install can-utils # 需要時再裝
```

若不想裝套件，Python 內建的 socketcan 就能收發——板上實測 `AF_CAN`／`CAN_RAW` 皆可用（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-can-pre2.txt）：

```bash
python3 -c "import socket; print(hasattr(socket,'AF_CAN'), hasattr(socket,'CAN_RAW'))"
# True True
```

**第三步：把 can0 帶起來（正確指令）。** 本板 DT 把控制器設在 **FD 模式**，所以帶起來時**必須同時給仲裁段 `bitrate`、資料段 `dbitrate`、以及 `fd on`**：

```bash
sudo ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on
```

板上實測成功帶起（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-can-fd-up.txt）：

```text
5: can0: <NOARP,UP,LOWER_UP,ECHO> mtu 72 qdisc pfifo_fast state UP mode DEFAULT group default qlen 10
    can <FD> state ERROR-ACTIVE (berr-counter tx 0 rx 0) restart-ms 0
	  bitrate 500000 sample-point 0.875
	  tq 25 prop-seg 34 phase-seg1 35 phase-seg2 10 sjw 5 brp 2
```

`state UP` 加 `ERROR-ACTIVE` 就是**正常帶起**的樣子——`ERROR-ACTIVE` 是 CAN 節點健康運作的預設錯誤狀態（錯誤計數為 0 時的正常態），**不是**出錯，別被字面嚇到。

**這幾個時序數字怎麼來的（bitrate 機制）**：你只給了目標 `bitrate 500000`，驅動程式自己從 80 MHz 時脈算出分頻與各段長度——

- 時間量子 `tq = brp ÷ clock = 2 ÷ 80 MHz = 25 ns`；
- 每個位元 = `sync(1) + prop-seg(34) + phase-seg1(35) + phase-seg2(10) = 80` 個 tq；
- 位元時間 = `80 × 25 ns = 2000 ns` → `1 ÷ 2000 ns = 500000 bit/s`，正好等於你設的 500 kbit/s；
- 取樣點 `sample-point = (1+34+35) ÷ 80 = 0.875`，即在位元的 87.5% 處取樣（CAN 慣用的偏後取樣，抗傳播延遲）。

> ⚠️ **陷阱框（只給 bitrate，介面帶不起來）**
> - **情境**：你照傳統 CAN 的習慣打 `sudo ip link set can0 up type can bitrate 500000`，沒給 `dbitrate`／`fd on`。
> - **症狀**：`RTNETLINK answers: Invalid argument`；`dmesg` 出現 `incorrect/missing data bit-timing` 與 `open_candev() failed: -EINVAL`，介面仍是 DOWN（板上實測，transcript：live/ch4-w1-can-diag.txt）。
> - **原因**：這是一個**跨處契約**——控制器在 DT 被設成 FD 模式（`dmesg` 可見 `global operational state (clk 1, fdmode 1)`），FD 模式**強制**要有資料段位元時序；你只給仲裁段 bitrate，資料段時序缺席 → `EINVAL`。
> - **預防／處理**：本板一律用 FD 形式帶起——`bitrate <仲裁> dbitrate <資料> fd on` 三者一起給。

**收發閉迴路的誠實邊界（⏸）**：教科書式的「內部 loopback 自收自發」——不接收發器、不接對端，讓控制器把送出的訊框回送給自己來驗證收發路徑——**在本板／本核心走不通**。`rcar_canfd` 驅動不支援 loopback 模式：

```bash
sudo ip link set can0 up type can bitrate 500000 dbitrate 2000000 fd on loopback on
# RTNETLINK answers: Operation not supported   （classical/FD、can0/can1 四種組合皆同）
```

（板上實測，transcript：live/ch4-w1-can-loopback.txt、ch4-w1-can-fd-loopback.txt。）因為 loopback 不可用，改用 Python socketcan 送框時，介面在 loopback 指令下始終沒 UP，送框一律得到 `OSError: [Errno 100] Network is down`，`ip -statistics` 的 RX/TX 全程為 0——**沒有任何一框真的送出或收到**。

⏸ 所以「cansend/candump 實際收發閉迴路」這一項**本手冊未能完成驗證**，原因有三：candump/cansend 未安裝、驅動不支援 loopback、且匯流排上沒有第二個 CAN 節點。**何時需要收發器**由此也就清楚了：若 loopback 可用，內部自測不需要收發器與對端；但本板既然只能走實體匯流排，要真的送收就得（1）先解除 TCAN1046 收發器的 standby（見上方注意框，GPIO 拉低 standby 腳），再（2）在匯流排上接上至少一個會回應的 CAN 節點。器材齊備後，收發流程如下（供照做）：

```bash
# 終端機 ：監聽 can0
candump can0
# 終端機 ：送一框（識別碼 0x123、資料 DEADBEEF）
cansend can0 123#DEADBEEF
```

**第四步：收工把介面帶回 DOWN。**

```bash
sudo ip link set can0 down
```

**驗證判準**：`ip -details link show can0` 出現 `state UP … ERROR-ACTIVE` 且 `bitrate 500000 sample-point 0.875` ＝ 介面帶起成功（控制器層級）；此時 `ip -statistics link show can0` 的 RX/TX 為 0 是**符合預期**的——沒有作動的收發器與對端節點，本來就不會有流量。真正的收發成功判準（RX/TX 計數增加、candump 收到框）需在收發器解待命且有對端後才驗，本板此環節⏸。

> 官方出處：硬體手冊 `r01uh1032` §7.9 CAN-FD Interface (CANFD)（p3363–3366，功能概說；暫存器明細自 §7.9.2 p3367 起）。其他文件：FSP CANFD 手冊 `r01us0478`、`can-utils`（軟體；本手冊尚未逐份開檔驗證這兩份）。載板佈線：WS125 RDK 手冊 §3.11。板上狀態：`07-hardware-unit-usage-guide.md` §30、`06-hardware-resource-map.md` §2.2。

---

## CRC 運算單元（硬體校驗碼，**未盤點缺口**）

### 這是什麼

CRC（Cyclic Redundancy Check，循環冗餘校驗）運算單元是一顆**硬體 CRC 計算器**，1 通道，可依你要的多項式產生對應長度的 CRC 碼——共 5 種可選：8-bit（CRC-8）、16-bit（CRC-16、CRC-CCITT）、32-bit（CRC-32、CRC-32C）（r01uh1032 §7.6.1，p2954）。CRC 是通訊與儲存裡最常見的完整性校驗法：把一段資料除以某個多項式，用餘數當校驗碼附在資料後面，收端重算比對就知道資料有沒有在傳輸中出錯。

硬體支援 8-bit 或 32-bit 兩種平行處理粒度輸入資料，並可切換輸出位元順序為 LSB-first 或 MSB-first，以配合不同通訊協定對 CRC 欄位的位元序慣例。另有 **snoop 功能**：可設定監控某個暫存器位址的讀寫動作，自動把經過該位址的資料餵進 CRC 計算，不必額外寫程式搬資料（p2954）。暫存器組非常精簡，只有 4 個：`CRCCR0`（控制）、`CRCDIR`（資料輸入）、`CRCDOR`（資料輸出）、`CRCSAR`（snoop 位址），基底位址 `0x1300_0800`（§7.6.2，p2956）。

### Linux 下怎麼看到它

**這個單元的板上狀態未經實測——這是一個誠實標記的缺口。** 硬體盤點文件 `06-hardware-resource-map.md` 與 `07-hardware-unit-usage-guide.md` 都**沒有**列出這個區塊，第 4 章 unit-map 的缺口清單把它標為「未知（未盤點）」。這裡**不揣測**板上是否有對應的 Linux 驅動程式或裝置節點，只忠實轉錄手冊規格。實際板上是否可從 Linux 存取，需要日後自行探測補查——例如檢查是否有對應的 mmio 節點（`0x1300_0800` 一帶）、或核心的 CRC 相關驅動程式是否綁定到這塊硬體。

### 關鍵能力與限制

- **通道數**：1 channel（p2954）。
- **可選多項式**（p2954 Table 7.6-1，多項式逐項照錄）：8-bit CRC-8（X⁸+X²+X+1）；16-bit CRC-16（X¹⁶+X¹⁵+X²+1）與 CRC-CCITT（X¹⁶+X¹²+X⁵+1）；32-bit CRC-32 與 CRC-32C。
- **限制**：逐字「The circuit does not have a function to divide data for calculation into CRC calculation units. Write data in 8-bit or 32-bit units.」（p2954 Note 1）——換句話說，把資料切成計算單位是**軟體的責任**，硬體本身不自動切割。

### 什麼情況下你會用到它

**機制上**，硬體 CRC 的價值是「把純軟體算 CRC 的 CPU 負擔卸載掉」。

**判斷準則**（附誠實邊界）：

- 若你的應用要在資料傳輸或儲存路徑上做完整性校驗（例如自訂序列協定裡的 CRC 欄位、韌體映像檔驗證），而且純軟體算 CRC 已經是可量測到的 CPU 負擔 → 硬體 CRC 單元原則上可以接手這部分工作。
- **但這是一條「若已盤點且有可用驅動路徑」的一般性判斷準則**。本板此單元目前缺乏板上實測依據，所以在你真的靠它之前，必須先自行探測（查是否有對應的 mmio 節點或核心驅動程式綁定），確認它真的能從 Linux 存取，再決定要不要用。在確認之前，把它當「規格上存在、板上待驗」看待。

> 官方出處：硬體手冊 `r01uh1032` §7.6 CRC Operation Unit (CRC)（p2954–2957，功能概說）。datasheet `r01ds0429`。板上狀態：**未盤點缺口**（`06`／`07` 皆未列，unit-map 標「未知」）。

---

## GPIO（PFC）（腳位多工的「切換總機」）

### 這是什麼

PFC（Pin Function Controller，腳位功能控制器）是外部接腳與晶片內部各功能單元之間的「切換總機」。這顆 SoC 的實體腳位遠少於內部功能訊號，所以每一支可多工的接腳都能在**通用 GPIO 模式**與最多 **15 種特定功能模式**（I²C、SPI、UART、CAN……）之間切換。切換由兩組暫存器組合決定：`PFC_PMC_mn`（模式選擇：Port 模式或 Control 模式）與 `PFC_PFC_mn`（功能選擇，Mode 1 至 15）（r01uh1032 §4.2.1.1，p362 Table 4.2-2）。

除了模式切換，PFC 還管理每支接腳的電氣特性：驅動強度、slew rate（訊號邊緣陡緩）、上／下拉電阻、輸入／輸出致能、N 通道開洩極、Schmitt 觸發、以及數位雜訊濾波（濾波器層級、級數、取樣間隔皆可各自設定）（§4.2.1，p361–364）。PFC 內還附掛一組獨立的 **Event Link Controller（PFC_ELC_GPIO）**，可讓 GPIO 上的事件直接連動到其他周邊，不需 CPU 介入（§4.2.1.6，p364）。

理解 PFC 的關鍵是這句話：**一支實體接腳，同一時間只能對應一種功能。** 這就是「腳位多工（pin-mux）」——某支腳若被切到 I²C 功能，就不能同時再拿來當通用 GPIO。第 4 章 4.1 節講的「很多單元矽片上有、卻在這片板子上停用」，根因常常就是腳位被別的功能佔走了。

### Linux 下怎麼看到它

在 Linux 裡，GPIO 是 `/dev/gpiochip1`，共 **96 條線**，驅動程式 `pinctrl-rzg2l`（它同時合併了 pinctrl〔腳位多工〕與 gpio 兩種功能）。存取走的是現代的 **gpiod 字元裝置 ABI**（`libgpiod` 工具：`gpioinfo`／`gpioget`／`gpioset`／`gpiofind`），**不是**舊式的 `/sys/class/gpio` sysfs 介面。

板上實測有一個容易混淆的地方——`gpiodetect` 會列出**兩個** gpiochip（✅ 2026-07-17，transcript：live/ch04-followup.txt）：

```text
gpiochip0 [gpio_dummy]      (32 lines)
gpiochip1 [10410000.pinctrl] (96 lines)
```

`gpiochip0` 是一個占位用的**假 expander**（`gpio_dummy`，32 lines），**不是**真正的 SoC 腳位；要操作實體腳位一律認 **`gpiochip1`**（96 lines，位址 `10410000.pinctrl`）。已知有數條線被系統 hog（占用）掉：CAN 收發器的 standby 腳（PA2／PA3）、相機／HDMI 的 reset 腳，另外開發時也有使用者把 PB2／PB3 拿去 `gpioset`。板上外露為標準樹莓派相容排列的 **40-pin 排針（J1，3.3V 準位）**——特殊功能腳位為 Pin3/5=I²C7、Pin8/10=UART5、Pin19/21/23/24/26=SPI6、Pin35/38/40=PCM（出處：WS125 RDK 手冊 §3.12、`04-hardware-quickref.md`）。大部分樹莓派 HAT 可相容，但用 **5V 邏輯**的 HAT 要先做電位轉換。

### 關鍵能力與限制

- **功能總覽**（逐字，p361 Table 4.2-1）：「GPIO control／Switching between functions multiplexed on the pins／Switching the drive strength of the pins／Switching the slew rate of the pins／Pull-up/down control of the pins／Input enable control／Output enable control／N-ch. open drain control／Schmitt control／OSC mode switching／Reset latch function／Digital noise filter control」。
- **模式層次**：埠模式（Port Mode）與控制模式（Control Mode）由 `PMC_mn` 位元切換；Port 模式下輸入／輸出方向再由 `PM_mn` 決定（p362）。
- **一支腳一種功能**：這是硬體限制，不是設定問題——腳位被切到專用功能就不能兼作 GPIO。

### 什麼情況下你會用到它

**機制上**，GPIO 是最基本的數位 I/O——任何「讀一個高低電位、輸出一個高低電位」的需求都走它。

**判斷準則**：

- 任何簡單數位輸入／輸出（讀開關／按鈕狀態、驅動 LED、致能／停用某個週邊晶片、bit-bang 低速自訂協定）→ 用 GPIO。
- **動手前先確認腳位有沒有被占用**：用 `gpioinfo` 看你想用的那條線是不是已經被其他功能（I²C／SPI／UART／CAN 收發器待命……）多工佔走。PFC 一次只能讓一支腳對應一種功能——腳位已被切到專用功能模式，就不能同時再當通用 GPIO 用；線空著才可放心設定成輸入或輸出。這條檢查對任何 RZ/V2H 板都適用，是避免「明明接了卻讀不到」的第一步。

> 官方出處：硬體手冊 `r01uh1032` §4.2 Pin Function Controller (PFC)（p361–365，功能概說；暫存器明細自 §4.2.2 p366 起）。其他文件：`libgpiod`（軟體）、WS125 RDK 手冊 §3.12（40-pin header 腳位）、`04-hardware-quickref.md`。板上狀態：`07-hardware-unit-usage-guide.md` §31、`06-hardware-resource-map.md` §2.2。

---

## USB3.2 Gen2（Host，超高速外接埠）

### 這是什麼

板上有 2 個 USB3 模組（ch0／ch1），每個模組由 **USB3HOST**（相容 eXtensible Host Controller Interface, xHCI）與一個測試子區塊（USB3TEST）組成；PHY（實體層收發器）走與 PCIe／SATA 共用架構的 **PIPE 介面**（r01uh1032 §6.4.1，p1636）。

要特別知道：這一章的 USB3 手冊是**簡化版**，逐字「This manual is a simplified version. For more information, refer to the User's Manual Additional Document.」（p1636）——完整的暫存器層級文件不在這一冊裡，要查暫存器細節得另外找 User's Manual Additional Document。

### Linux 下怎麼看到它

驅動程式 `xhci-renesas`。裝置掛上後出現在 `/dev/bus/usb/00X/00Y`（大容量儲存另有 `/dev/sdX`、序列裝置另有 `/dev/ttyACM*`／`/dev/ttyUSB*`）。USB30 Host 基底位址 `0x15850000`，USB31 Host `0x15860000`（出處：真硬體手冊 `r01uh1032` §1.8 Address Map，Table 1.8-1 Detailed Address Space，p170）。注意 `0x15840000` 是 **USB21 的 PHY** 區、不是 USB30 Host——這兩個位址只差一格，別把 PHY 位址誤當主控位址。

板上實測——`lsusb` 看得到 2 個 USB 3.0 root hub（ID `1d6b:0003`）與 2 個 USB 2.0 root hub（ID `1d6b:0002`，屬下一節的 USB2）（✅ 2026-07-18，transcript：live/ch04b-runtime.txt）：

```text
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
```

這一欄會隨插拔變動：2026-07-18 探測當下，其中一個 USB3 Type-A 埠上掛著一個 USB 無線網路卡（示範「有東西插著」的情形）；2026-06-21 的盤點則是這幾個 root hub 都沒掛裝置（＝2 個 Type-A 埠全空）。所以你自己板上 `lsusb` 的裝置列表，取決於當下插了什麼——但列出的 USB2.0／USB3.0 root hub 是控制器列舉，對應的是那 2 個實體 Type-A 埠，不是 4 個可插的埠。

### 關鍵能力與限制

- **支援速度**（逐字，p1636 Table 6.4-1／6.4-2，Host Controller 對五種速度皆支援）：「Super Speed Plus (10 Gbps), Super Speed (5 Gbps), high-speed (480 Mbps), full-speed (12 Mbps), and low-speed (1.5 Mbps) transfer」。

> 💡 **提示（別把 root hub 的 `20000` 讀成 20 Gbps 可用埠）**：在板上跑 `cat /sys/bus/usb/devices/usb2/speed`（或 `usb4`）會回 `20000`（Mbps）（✅ 2026-07-22，transcript：live/ch4-w1-usb.txt，兩個 USB3 root hub 皆是）。這是 xHCI root hub 對外**宣告**自己具備 SuperSpeedPlus 雙通道（x2）能力的數字，不代表這顆 SoC 的 PHY 跑得到 20 Gbps——本板 USB3 PHY 是 **Gen2x1（單通道，10 Gbps，§6.4 p1636）**，實際協商出來的 SuperSpeedPlus 連結上限就是 10 Gbps。所以看到 `20000` 別誤以為有 20 Gbps 可用。
- **傳輸型態**：支援全部四種——isochronous、interrupt、control、bulk，且支援 isochronous／interrupt 的 high-band transfer（p1636）。
- **通道數**：2（ch0／ch1）。
- **限制（host-only）**：本板矽片為 **host-only** 設計——板子只能當主機接周邊，**無法**反過來扮演 USB device 端接到別的主機。要 device／OTG 角色得看下一節的 USB2.0 ch0。

### 什麼情況下你會用到它

**機制上**，USB3 的價值是**高頻寬**（最高 10 Gbps）。

**判斷準則**：

- 需要大量資料高速傳輸的外接裝置（以外接 SSD、高解析度 UVC 攝影機、高速網路 dongle 為例）→ 用 USB3。
- 一般速度周邊（鍵盤滑鼠、UART 轉 USB、低速感測器）→ 不必占用 USB3，走 USB2 即可（見下節）。
- 若你的應用需要讓板子**反過來當 USB device**（例如對上位主機顯示為一個 USB 序列或儲存裝置）→ USB3 這一路做不到（host-only），要用 USB2.0 的 ch0 Function 模式。

### 動手：掛載 USB 儲存裝置

把隨身碟或外接 SSD 插上 USB 埠後，Linux 不會自動幫你「開啟」它——你得先找到它的裝置節點，再手動掛載到某個資料夾。這一節教這條流程。**機制上**，USB 大量儲存裝置會被 `usb-storage`／`uas` 驅動程式列舉成一顆 SCSI 磁碟，出現在 `/dev/sdX`（`sda`、`sdb`…），分割區則是 `/dev/sdX1`（此為 Linux 通用行為，非晶片手冊規範；本節掛載流程適用任一 USB 埠，不限 USB3）。

**先看沒插媒體時的基準狀態**（板上實測，✅ 2026-07-22 板上實測，transcript：live/ch4-w1-usb.txt）：

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

清單裡 `mtdblock0`–`mtdblock3` 是板上內建 flash（MTD，走 xSPI NOR 的開機／環境等小分割，容量都在 MB 級，不是 USB 儲存）；`mmcblk0` 才是系統本身的 eMMC／SD（根檔案系統掛在 `/`）。重點是**沒有任何 `/dev/sd*`**——代表當下沒有 USB 儲存裝置掛著。這就是你插碟前該看到的「乾淨」畫面；插碟後 `/dev/sda` 冒出來，就是它上線了。（`lsusb` 這時列的是 root hub 與當下插著的非儲存裝置，例如板上當時插著的一片 USB 無線網路卡；插了隨身碟才會多出對應的儲存裝置。）

**掛載流程**（⏸ 以下實體插碟步驟未實際執行——手邊無 USB 媒體；流程供插碟後照做）：

```bash
# 1) 插入 USB 隨身碟／SSD 到 Type-A 埠

# 2) 確認裝置節點冒出來（會多一顆 sdX 與分割區 sdX1）
lsblk
dmesg | tail          # 會看到類似 sd 0:0:0:0: [sda] ... 的列舉訊息

# 3) 查分割區的檔案系統型別
lsblk -f              # 或 sudo blkid /dev/sda1，看 FSTYPE（vfat/exfat/ext4…）

# 4) 建掛載點並掛載（把 sda1 換成你實際的分割區）
sudo mkdir -p /mnt/usb
sudo mount /dev/sda1 /mnt/usb
#   vfat/ext4 通常直接可掛；exfat 需先 sudo apt install exfat-fuse（或核心 exfat 支援）

# 5) 用完務必卸載再拔
sudo umount /mnt/usb
```

> ⚠️ **陷阱框（沒 umount 就拔，資料可能沒寫完）**
> - **情境**：複製檔案到隨身碟，`cp` 一回到提示字元就直接拔碟。
> - **症狀**：碟拿到別台機器，檔案打不開或內容不完整。
> - **原因**：Linux 為效能會把寫入放進快取，不保證在 `cp` 返回時已全部落盤；未 `umount`／`sync` 就拔，快取裡的資料就丟了。
> - **預防／處理**：一律 `sudo umount /mnt/usb` 成功後再拔碟；急著確認也可先 `sync`。

> ⚠️ **注意（掛載點要用空資料夾）**：掛載會**遮蔽**掛載點原有的內容——請用專門的空資料夾（如 `/mnt/usb`），別掛到有東西的目錄上。

**驗證判準**：插碟後 `lsblk` 多出 `sdX`／`sdX1`、`dmesg` 有 `[sda]` 列舉訊息 ＝ 裝置上線；掛載後 `df -h /mnt/usb` 看得到該碟容量、`ls /mnt/usb` 看得到檔案 ＝ 掛載成功。（⏸ 因手邊無媒體，實體掛載未執行；已驗證的是「未插碟時無 `/dev/sd*`」的基準狀態。）

> 官方出處：硬體手冊 `r01uh1032` §6.4 USB3.2 Gen2x1 Interface (USB3)（p1636–1640，功能概說；完整暫存器見 User's Manual Additional Document）。板上狀態：`07-hardware-unit-usage-guide.md` §42、`06-hardware-resource-map.md` §2.2。

---

## USB2.0（一般速度外接埠，含 OTG 矽片）

### 這是什麼

這顆晶片含 **1 通道 USB2.0 OTG/DRD**（Host／Function 雙角色）介面與 **1 通道純 Host** 介面（r01uh1032 §6.5.1，p1753）。DRD（Dual-Role Device）指同一個埠可以當主機、也可以當裝置，但這裡的切換是**靜態**的（開機／設定時決定），不是插拔時動態協商。

> ⚠️ **這裡講的是矽片通道，不是實體埠。** 本板實體對外只有 **2× USB3.2 Gen2 Type-A（CN2）＋ 1× micro-B（UART／function，非資料埠）**，沒有獨立的 USB 2.0 host 連接器（板卡手冊 §3.6／§3.7）。這些 USB2 通道是與 USB3 埠共構的 companion——`lsusb` 列出的 USB2.0 root hub 是 xHCI 控制器的列舉，不代表多出可插的實體埠。別把 root hub 數當成埠數。

- **ch0（OTG/DRD）**：支援 Host 模式（10 ch PIPE，含 default control PIPE）與 Function 模式，並具備 OTG（Rev. 2.0）、Battery Charging、DRD 功能。
- **ch1**：僅 Host 模式（p1754 Table 6.5-1／6.5-2）。

同樣是**簡化版手冊**，完整暫存器規格在額外文件中。

### Linux 下怎麼看到它

與 USB3 共用 `xhci-renesas`／EHCI-HS 驅動路徑，裝置一樣掛在 `/dev/bus/usb/*`。板上 ch0 雖然矽片支援 OTG，但目前**固定設定為 Host 模式**（出處：`07-hardware-unit-usage-guide.md` §43）。USB20 Host 基底 `0x15800000`，USB21 Host `0x15810000`，USB20 Function 子區塊 `0x15820000`（出處：真硬體手冊 `r01uh1032` §1.8 Address Map，Table 1.8-1，p170）。

板上 `lsusb` 對應的 2 個 USB 2.0 root hub（ID `1d6b:0002`）就是這兩路（✅ 2026-07-18，transcript：live/ch04b-runtime.txt）：

```text
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

### 關鍵能力與限制

- **速度**：High-Speed 480 Mbps／Full-Speed 12 Mbps／Low-Speed 1.5 Mbps（p1754）。
- **OTG 限制**：逐字「Session Request Protocol (SRP) and Host Negotiation Protocol (HNP) are not supported」（p1753 Note 1）——OTG 角色切換**不支援動態協商**，只能靜態設定。
- **Battery Charging**：逐字「For this LSI, IDP_SRC will flow to a maximum of 32 μA」（p1753 Note 3）。

### 什麼情況下你會用到它

**機制上**，USB2 是「夠用就好」的一般速度埠——把高速頻寬留給 USB3。

**判斷準則**：

- 一般速度的 USB 周邊（以鍵盤滑鼠、UART 轉 USB 轉接器、低速感測器模組為例）→ 走 USB2 即可，不占 USB3 資源。
- 需要讓本板**反過來**當 USB device（例如對上位主機顯示為 USB 序列或大量儲存裝置）→ 必須用 **ch0 的 Function 模式**。但這要改動目前固定的 Host 設定，而且因為不支援 SRP／HNP，角色切換得用**靜態設定**而非插拔自動協商——判斷點是「你能不能接受在設定階段就固定角色」。

> 官方出處：硬體手冊 `r01uh1032` §6.5 USB2.0 Interface（p1753–1757，功能概說；完整暫存器見額外文件）。板上狀態：`07-hardware-unit-usage-guide.md` §43、`06-hardware-resource-map.md` §2.2。

---

## GBETH ＋ PTP（有線乙太網路與硬體時間戳）

### 這是什麼

GBETH 是 2 通道的 Ethernet MAC（Media Access Control，媒體存取控制器），相容 Synopsys DWMAC/EQOS 架構，符合 IEEE 802.3-2008——支援 Ethernet MAC、RGMII、MII 三種介面，但**不支援 GMII**（r01uh1032 §6.3.1.1，p1632–1633）。每個通道各有 4 組獨立的 TX DMA 與 4 組 RX DMA，採原生 DMA，描述子可用 dual-buffer（ring）或 linked-list（chained）鏈結（p1632）。

它內建一項對「多節點時間同步」很重要的能力：**IEEE 1588-2008 v2 硬體時間戳**，參考時脈 125 MHz，並相容 IEEE 802.1AS-2011（gPTP）與 IEEE 802.1Qav／Qat（音視訊時間敏感網路，TSN）（p1632–1633）。PTP（Precision Time Protocol）讓網路上多個節點把時鐘對齊到同一個時基，硬體時間戳則是把「封包進出網路卡的那一刻」精準記錄下來，避免軟體排程抖動污染時間量測。另支援 31 組 MAC 位址過濾暫存器、256-bit hash filter，以及 IEEE 802.3az（Energy Efficient Ethernet，含 LPI 低功耗模式與 Wake-on-LAN）。

### Linux 下怎麼看到它

驅動程式 `stmmac`／`dwmac`，網路介面名稱是 **`end0`**（對應 GBETH0，狀態 UP，能力 1 Gbps）。硬體時間戳裝置是 `/dev/ptp0`。GBETH0 基底 `0x15C30000`，GBETH1 `0x15C40000`；GBETH1（ch1）的 device tree 節點是 `disabled`（板上實測：`ethernet@15C30000` 為 `okay`、`ethernet@15C40000` 為 `disabled`，✅ 2026-07-18，transcript：live/ch04b-dt-status.txt）。

一個新手常踩的命名坑，加上實測連線速率：

> ⚠️ **注意（乙太網路介面叫 `end0`，不是 `eth0`）**
> - **情境**：你沿用習慣打 `ifconfig eth0`，或設定檔寫 `eth0`。
> - **症狀**：找不到 `eth0`。
> - **原因**：本板的有線網路介面命名為 **`end0`**（不是 `eth0`）。
> - **預防／處理**：一律用 `end0`。它的 PHY 是 KSZ9131，能力 1 Gbps——**實際連線速率看對端交換器與線材**，用 `cat /sys/class/net/end0/speed` 可查（2026-07-18 板上實測回 `100`，即當時以 100 Mbps 連線；能力是 1 Gbps，連線速率是協商結果，兩者不是同一件事）。MAC 位址是隨機產生但持久的（格式 `xx:xx:xx:xx:xx:xx`，每塊板子各不相同——用 `ip link show end0` 查你自己板上的那組，別套用任何範例值）（出處：`04-hardware-quickref.md`）。

### 關鍵能力與限制

- **通道數**：2（板上只開 ch0）。
- **速率**：10／100／1000 Mbps，全雙工或半雙工（p1632）。
- **Jumbo frame**：可程式化訊框長度，最大到 16 KB − 1，逐字「Jumbo mode support in cut-through mode only (not implemented in store and forward due to TX and RX FIFO size)」（p1632）——jumbo 只在 cut-through 模式支援。
- **時戳基礎**：逐字「IEEE1588 time base information, with reference clock of 125 MHz」（p1633）。
- **介面**：僅 RGMII 與 MII，逐字「GMII is not supported」（p1633）。

### 什麼情況下你會用到它

**機制上**，`end0` 是板子對外的主要有線網路連線——韌體／映像檔傳輸、遠端管理、資料上傳都走這裡；PTP 則是這顆網路卡多出來的「精準時間」能力。

**判斷準則**：

- 一般網路傳輸 → `end0` 直接當標準網路卡用即可，不必碰 PTP。
- 需要**跨裝置的精準時間同步**（多節點資料要對齊到同一個時基，以多感測器同步取樣戳記、分散式量測系統為例）→ 才需要動用 `/dev/ptp0` 搭配 `linuxptp`（`ptp4l`／`phc2sys`）。判斷點是「你的資料需不需要跨節點對時到硬體時戳等級的精度」——不需要就別多花力氣。
- 需要**第二張**獨立網路介面 → 才去改 device tree 打開 ch1（`15C40000`），並確認板上實際有對應的 PHY 佈線（停用≠佈線存在）。

> 官方出處：硬體手冊 `r01uh1032` §6.3 Gigabit Ethernet Interface (GBETH)（p1632–1635，功能概說）。其他文件：`linuxptp`（軟體）、PHY KSZ9131 datasheet（第三方）。板上狀態：`07-hardware-unit-usage-guide.md` §44、`06-hardware-resource-map.md` §2.2。

---

## PCIe 3.0（高速擴充匯流排）

### 這是什麼

PCIe（PCI Express）是點對點的高速擴充匯流排。這顆 SoC 的 PCIe 核心是**雙角色設計**：同一顆核心可透過設定切換成 **Root Complex（RC，主動端，接外部裝置）** 或 **Endpoint（EP，被動端，被別的主機當成一張介面卡）**——因此內建 Type 0（EP 用）與 Type 1（RC 用）兩種 Configuration Register（r01uh1032 §6.6.1，p2025）。

共 2 個 unit，lane 組態可選 4-lane 單一通道，或 2-lane × 2 channel 的 multi-link 組態（channel 0／1 各自獨立的 PCIe core，共用同一顆晶片但邏輯上分離）（§6.6.1.1，p2027）。內建 DMA controller，8 個通道，descriptor 或 register 控制模式（p2026）。符合 PCI Express Base Specification 4.0，實際線速支援 Gen1（2.5 GT/s）／Gen2（5.0 GT/s）／Gen3（8.0 GT/s）（p2026）。

### Linux 下怎麼看到它

驅動程式 `rzv2h-pcie`（device tree compatible `pcie-rzv2h`，Renesas Root Complex 實作），走標準 PCI 子系統——可用 `lspci` 檢視，裝置出現在 `/sys/bus/pci/devices`。板上啟用為 **Root Complex**，記憶體視窗 `0x30000000`–`0x37FFFFFF`（對應手冊「PCIE0 area 128 Mbytes @ 0x30000000」），PCIE1 未使用（出處：`07-hardware-unit-usage-guide.md` §45；`06-hardware-resource-map.md` §2.2）。目前板上沒有接 endpoint，所以 `lspci` 通常只看到 root port 本身。

### 關鍵能力與限制

- **Unit 數**：2（p2025）。板上只用 PCIE0。
- **Lane 組態**：逐字「Lane implementation x4 / x2 × 2ch (when Multilink is selected)」（p2026）。
- **AXI 介面限制**：逐字「Little endian is only supported」，且非對齊（unaligned）傳輸在 master 介面不支援（p2025）。
- **Data payload**：最高 256 bytes；read request size 最高 512 bytes（p2026）。
- **Virtual channel**：僅支援 VC0，不支援額外 virtual channel（p2026）。
- **Outstanding transfer 數**：1 到 8（p2026）。

### 什麼情況下你會用到它

**機制上**，PCIe 給的是「最高頻寬的擴充路徑」，適合接需要大頻寬、且本身就是 PCIe endpoint 的高速週邊。

**判斷準則**：

- 需要外接高速週邊控制晶片（以 NVMe SSD 解決儲存瓶頸、高速網路卡、FPGA 加速卡為例），而且該裝置本身是 PCIe endpoint → 接上本板的 PCIe Root Complex，Linux 端就是一般標準 PCI 裝置探測流程。
- 反過來，若你的場景是要讓**這片板子本身**被另一台主機當成一張 PCIe 裝置卡（EP 模式）→ 矽片雖然支援 RC／EP 雙角色，但本板出廠**固定跑 RC**。要用 EP 需另外設定 `SYS_PCIE_MODE` 暫存器並確認韌體／device tree 支援該角色；這不是目前板上驗證過的路徑，屬「要自行開通並驗證」的方向。判斷點是「板子是接別人、還是被別人接」。

> 官方出處：硬體手冊 `r01uh1032` §6.6 PCI Express 3.0 Interface (PCIe)（p2025–2035，功能概說＋方塊圖；暫存器明細自 §6.6.4 p2037 起）。板上狀態：`07-hardware-unit-usage-guide.md` §45、`06-hardware-resource-map.md` §2.2。

---

## ADC（12-bit 類比數位轉換器）

### 這是什麼

ADC（Analog-to-Digital Converter）是 12-bit **逐次逼近（successive approximation）**類比數位轉換器，最多 8 個可選類比輸入通道（r01uh1032 §7.10.1，p3670）。它把外部的類比電壓（例如電位計分壓、電流分流電阻上的壓降）轉成數位讀值。

它有三種操作模式（p3670）：

- **Single scan**：選定的通道各轉換一次後停止。
- **Continuous scan**：持續依通道編號順序循環轉換。
- **Group scan**：把通道分成 2 組（A／B）或 3 組（A／B／C），各組可獨立設定觸發條件；當高優先權組別有轉換請求進來時，會**中斷正在進行的低優先權組別**掃描，優先權順序固定為 A > B > C。

轉換啟動條件三選一（p3671 Table 7.10-1）：軟體觸發、內部觸發（由 GPT 定時器＋ELC 事件連結觸發）、外部觸發（ADTRG 腳位）。用 GPT ＋ ELC 觸發，等於讓 ADC 在硬體層被計時器精準地定時啟動，不必 CPU 每次去戳。

### Linux 下怎麼看到它

ADC 走 Linux 的 **IIO（Industrial I/O）**子系統，不是字元裝置——你**不會** `cat /dev/adc`，而是讀 sysfs 屬性。裝置是 `/sys/bus/iio/devices/iio:device0`，驅動程式 `rzv2h-adc`（板上 `.../name` 逐字回 `rzv2h-adc`，✅ 2026-07-17，transcript：live/ch04-followup.txt）。

板上實測（✅ 2026-07-18，transcript：live/ch04b-runtime.txt）：8 個通道各有 `in_voltage0..7_raw` 與 `in_voltage0..7_get_value` 兩個屬性；讀 `in_voltage0_raw` 回 `2819`。**本板驅動程式沒有曝露 `in_voltage_scale` 屬性**，所以要換算成實際電壓得按 12-bit 解析度、0–1.8 V 參考電壓範圍自行計算：

```text
電壓 ≈ raw ÷ 4095 × 1.8 V
例：raw 2819 → 2819 ÷ 4095 × 1.8 ≈ 1.24 V
```

（出處：`07-hardware-unit-usage-guide.md` §37；unit-map。）

### 關鍵能力與限制

- **Unit／通道數**：1 unit，最多 8 channels（p3670）。
- **解析度**：可選 12-bit 或 8-bit（p3671）。
- **取樣率**（依 A/D 轉換時脈而定，逐字，p3671）：「2.5 Msps (when A/D conversion clock ADC_0_ADCLK is 50 MHz) / 2.0 Msps (40 MHz) / 1.0 Msps (20 MHz) / 0.5 Msps (10 MHz) / 0.25 Msps (5 MHz)」。
- **參考電壓範圍**：0–1.8 V（換算電壓的基準；本板未曝露 scale 屬性，需自算）。

### 什麼情況下你會用到它

**機制上**，ADC 是板子讀取「非數位介面」訊號的入口——凡是輸出類比電壓的感測器都靠它。

**判斷準則**：

- 讀取類比感測器輸出（以電位計、電流分流電阻取樣、電池電壓分壓、光敏電阻為例；移動載具上的氣壓計／電流計／舵機位置回授也是一種應用）→ 用 ADC。
- **操作模式的取捨**（不是規格門檻，是依你的量測需求選）：只需要單點量測或不常變動的訊號 → `single scan` 就夠；需要對同一批通道持續監控（以即時監看多路類比訊號為例）→ `continuous scan`；不同訊號之間有優先權差異（例如某個安全相關訊號要能打斷背景量測、優先被讀到）→ 才用 `group scan` 加優先權設定。判斷點是「你的訊號要不要即時、要不要分優先權」。

### 動手：從 IIO 讀 ADC 並換算電壓

ADC 走 IIO 子系統，讀值就是 `cat` sysfs 屬性。流程是四步：**列舉裝置 → 讀 raw → 換算成電壓 → 驗證讀值合不合理**。

**第一步：列舉 IIO 裝置、確認就是 ADC。**

```bash
ls -l /sys/bus/iio/devices/
cat /sys/bus/iio/devices/iio:device0/name
ls /sys/bus/iio/devices/iio:device0
```

板上實測（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-adc.txt）：`iio:device0` 連到 `.../11c00000.adc/`，`name` 逐字回 `rzv2h-adc`，屬性裡每個通道有 `in_voltageN_raw` 與 `in_voltageN_get_value` 一對，共 8 通道（0–7）。**注意清單裡沒有 `in_voltage_scale`**——本板驅動程式不曝露 scale 屬性，換算得自己來（下面第三步）。

**第二步：讀 raw 數位讀值。**

```bash
for f in /sys/bus/iio/devices/iio:device0/in_voltage*_raw; do
  echo "${f##*/}=$(cat $f)"
done
```

板上實測（✅ 2026-07-22 板上實測，transcript：live/ch4-w1-adc.txt）：

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

raw 是 12-bit 的數位碼，範圍 0–4095（0＝0 V、4095＝滿刻度）。

**第三步：換算成實際電壓。** 因為本板沒有 scale 屬性，按 12-bit 解析度、0–1.8 V 參考範圍自算：

```text
電壓 ≈ raw ÷ 4095 × 1.8 V
例：raw 2816 → 2816 ÷ 4095 × 1.8 ≈ 1.24 V
    raw 2318 → 2318 ÷ 4095 × 1.8 ≈ 1.02 V
```

**第四步：驗證讀值——而這一步會撞到一個關鍵陷阱。** 隔幾秒把同一組屬性再讀一次，數字會變：第一次讀 `in_voltage0_raw` 是 `2816`，稍後再讀變成 `1311`（換算 ≈ 0.58 V）——同一個通道、什麼都沒改，讀值卻從 1.24 V 跳到 0.58 V（板上實測，transcript：live/ch4-w1-adc.txt）。

> ⚠️ **陷阱框（懸空腳的 ADC 讀值會漂，不能當量測結果）**
> - **情境**：你直接 `cat in_voltageN_raw`，把數字換算成電壓就當成量到的訊號。
> - **症狀**：同一通道連讀兩次差很多（如 `2816` → `1311`），換算出的電壓跳來跳去，毫無重現性。
> - **原因**：這些 ADC 通道**目前沒接任何訊號源**，輸入處於懸空（floating）狀態；懸空腳抓到的是雜訊與漏電位，每次轉換的結果本來就會漂，不是有效量測。
> - **預防／處理**：量測前先把一個**已知、落在 0–1.8 V 範圍內**的電壓接到對應通道（例如自 1.8 V 電源做的分壓、或實驗室電源），再讀 raw、對照換算值是否接近預期。懸空腳的讀值一律不可當結果。

> ⚠️ **注意（別超過參考電壓範圍）**：ADC 參考範圍是 0–1.8 V，輸入電壓**不要超過 1.8 V**，超過可能損傷腳位。要量更高電壓先用分壓電阻降到範圍內。

⏸ **本節的驗證邊界**：已驗證的是「裝置列舉、raw 讀取、換算公式、以及懸空即漂的行為」；**未做**的是「接已知電壓、確認 raw 對得上預期電壓」的絕對準度驗證——本節實測沒有接參考訊號源。要完整驗證整條轉換鏈，需備一個已知電壓源，待器材齊備後補做。

**驗證判準**：`name` 回 `rzv2h-adc`、8 個通道各有 `in_voltageN_raw` ＝ 驅動與裝置就緒；接上已知電壓後，`raw ÷ 4095 × 1.8` 與實際輸入相符（差在 ADC 精度內）＝ 轉換鏈正確。若讀值持續漂動且無對應輸入，代表該腳懸空、讀值無效。

> 官方出處：硬體手冊 `r01uh1032` §7.10 12-Bit A/D Converter (ADC)（p3670–3674，功能概說；暫存器明細自 §7.10.2 p3675 起）。其他文件：`libiio`（軟體）。板上狀態：`07-hardware-unit-usage-guide.md` §37。

---

## TSU（晶片溫度感測單元）

### 這是什麼

TSU（Temperature Sensor Unit）是晶片內建的溫度感測單元，由一顆類比主感測器加上專屬 ADC 組成，把感測器輸出的類比訊號轉成對應溫度的數位碼（r01uh1032 §7.11.1，p3739）。轉換結果可經 APB 匯流排讀取，並可設定與上下限溫度值比較、超出範圍時觸發中斷（p3739）。

轉換啟動方式二選一：軟體透過暫存器設定觸發，或由 ELC（Event Link Controller）觸發；轉換模式固定為 single scan，並可設定取樣次數做平均以降低雜訊（p3739）。晶片上共有 **2 組獨立的 TSU（TSU0／TSU1）**，各自連到系統匯流排、CPG（時脈供應）與 ICU（中斷控制）（§7.11.2，p3740）。這是量**晶片本體（die）**溫度的感測器——注意，不是外部環境溫度。

### Linux 下怎麼看到它

TSU 走 Linux 的 **thermal 框架**，曝露為 `/sys/class/thermal/thermal_zone0` 與 `thermal_zone1`（兩組 TSU 對兩個 zone），驅動程式 `rzv2h_thermal`。板上實測讀值約 **~34–36°C**（跨 session 橫跨；此批 `07-hardware-unit-usage-guide.md` §38 記 34–35°C，另有 session 讀到 35／36，引用單點時標日期／主機；亦見 `05-compute-benchmark.md` 散熱相關量測）。讀溫度就是 `cat /sys/class/thermal/thermal_zone0/temp`（回的是毫度，例如 `35000` = 35°C），不走 `/dev`。

### 關鍵能力與限制

- **Unit 數**：2（TSU0／TSU1）（p3739；unit-map）。
- **精度規格**（解析度 0.0625°C／code、量測範圍 −40～125°C、精度 ±5°C、14.9 ksps）：**這組數值有來源層級的誠實標記**——本手冊實讀的頁段（p3739–3741）是功能概說／連接圖／腳位描述，**未涵蓋**列出這些數值的暫存器規格頁（§7.11.6）。此組數值出處為 `07-hardware-unit-usage-guide.md` §38（該文轉錄自 r01uh1032 Table 7.11-5／7.11-6），**非**本手冊直接核對 PDF 逐字覆核。引用這幾個精度數字時，請知悉它們是二手轉錄、待日後直接對 §7.11.6 覆核。

### 什麼情況下你會用到它

**機制上**，TSU 量的是**封裝內部**（die）溫度，用來做熱保護或概略掌握系統冷熱。

**判斷準則**：

- 需要因應高溫**自動降頻、暫停高功耗運算單元**（以長時間高負載的 NPU／GPU 工作為例）以避免熱關機 → 讀 `thermal_zone*` 的溫度值搭配自訂節流策略，是常見做法。
- **重要邊界**：TSU 讀的是晶片內部溫度，**不能**直接當環境溫度用——在有主動散熱或機殼氣流的系統裡，die 溫度與環境溫度可能有明顯落差。若你要的是機箱／環境溫度，得另外接外部溫度感測器（例如透過 ADC 或 I²C 感測器）。判斷點是「你要保護的是晶片、還是量周遭環境」——這兩件事用不同的感測器。

> 官方出處：硬體手冊 `r01uh1032` §7.11 Temperature Sensor Unit (TSU)（p3739–3745，功能概說；暫存器明細自 §7.11.6 p3746 起——精度數值頁，本手冊未直接覆核）。板上狀態與精度轉錄：`07-hardware-unit-usage-guide.md` §38、`05-compute-benchmark.md`（散熱）。

---

## 把這一群接起來：選介面的判斷準則

這 14 個單元裡有一半是「讓板子跟外部裝置說話」的介面。新手最常問的不是「這介面是什麼」，而是「同一個裝置，我到底該用哪一條」。把前面各節的判斷準則收攏成一張對照表——**先看對端裝置支援什麼協定，再看你的量測需求**：

| 你的需求 | 首選 | 為什麼（機制） |
|---|---|---|
| 開機就要有的除錯／console 序列埠 | **SCIF** | 與開機韌體共用硬體，永遠在線 |
| 多條獨立序列連線，或輕量 SPI／I²C／LIN 人格 | **RSCI** | 10 通道、每通道六種模式可選（板上需改 DT 開通） |
| 點對點高速、對端是標準 SPI | **RSPI** | 有時脈線、全雙工，速度優先於腳位彈性 |
| 多裝置共線、對端 ≤1 Mbps、重視省線 | **I²C（RIIC）** | 兩線加位址定址，掛多裝置最省 |
| 對端支援 I3C、要 in-band 中斷／hot-join | **I3C** | I²C 的接線換到更高能力（板上未接線） |
| 多節點、抗雜訊、確定性、對端是 CAN 節點 | **CAN-FD** | 硬體仲裁＋錯誤重傳（用前先解收發器待命） |
| 讀類比電壓感測器 | **ADC** | 唯一的類比輸入路徑 |
| 監控晶片溫度做熱保護 | **TSU** | 量 die 溫度（不是環境溫度） |
| 高頻寬外接裝置（SSD／UVC 相機） | **USB3.2** | 最高 10 Gbps（host-only） |
| 一般速度 USB 周邊 | **USB2.0** | 夠用就好，把頻寬留給 USB3 |
| 有線網路；需要跨節點精準對時 | **GBETH（＋PTP）** | 標準網路卡；`/dev/ptp0` 做硬體時戳 |
| 接高速 PCIe endpoint（NVMe／加速卡） | **PCIe** | 最高頻寬的擴充路徑（板上為 RC） |
| 簡單數位 I/O、控制某晶片致能腳 | **GPIO（PFC）** | 用前先 `gpioinfo` 確認腳位沒被多工佔走 |
| 資料完整性硬體校驗 | **CRC** | 卸載軟體算 CRC 的負擔（板上未盤點，用前先探測） |

這張表刻意用「需求 → 介面」而非「介面 → 用途」排列，因為做系統整合時你手上先有的是「一個要接的裝置」，再回推該用哪條匯流排。表裡任何一列的完整機制、能力邊界與板上狀態，都在上面對應的單元小節。凡標「需改 DT／未接線／未盤點／用前先確認」的，都是提醒你：**規格上存在，不等於這片板子現在拿來就能用**——這正是第 4 章 00 檔（4.1 總覽與六格狀態表）反覆強調的「啟用有層次」。

> **本群組資料保真聲明**：以上機制與能力數值逐字取自官方硬體手冊 `r01uh1032ej0130`（頁碼隨文標註）；板上狀態與裝置節點取自開發紀錄 `06-hardware-resource-map.md`（doc06）、`07-hardware-unit-usage-guide.md`（doc07）與 `04-hardware-quickref.md`，標 ✅ 者為 2026-07-17／18 板上複驗逐字輸出（transcript 收於 live/ch04*.txt，逐處隨文標註）。**兩處誠實邊界**：CRC 運算單元板上狀態未盤點（只引手冊規格）、TSU 精度數值為 doc07 二手轉錄（未直接覆核 §7.11.6 暫存器頁）——兩者都已在各自小節就地標明，不以推測填補。FSP RSPI／CANFD 手冊、WS125 RDK 載板手冊全份此次未逐份開檔，引用其內容處均已註明來源、未擴大轉述。
