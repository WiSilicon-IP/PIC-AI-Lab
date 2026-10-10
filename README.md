# PIC-AI-Lab — PIC 智慧實驗平台：從 AI 自動量測、自動校正到光子運算

PIC-AI-Lab 是一套以 **imec SiPhotonics 0.13 µm 製程（IMEC-SiPh）矽光子（PIC）晶片** 為對象的智慧實驗平台，目標分三個階段：
讓 AI 自動量測晶片、自動校正晶片，最後用晶片上的 **MMI / MZI 網路做運算**。
待測晶片含 **多顆可調 MZI 分光器** 與 **4 支 MEMS 光探針**；硬體以
**Arduino GIGA R1 WiFi（STM32H747）** 為控制核心，搭配 **DAC81416-08EVM** 驅動加熱器與探針，
**ADS9324EVM（改板）** 讀取 16 通道光功率與電壓訊號。

## 三階段路線圖

| 階段 | 目標 | 關鍵能力 | 狀態 |
|---|---|---|---|
| **1. AI 自動量測** | 無人值守完成 DC 相位掃描與暫態擷取 | 掃描序列自動規劃、安全限制（MAX\_CODE、斜率、watchdog）、資料自動紀錄與擬合 | **進行中**（本文件所述量測系統） |
| **2. AI 自動校正** | 自動找到每顆 MZI 的 V₂π、R\_h 與探針工作點，並追蹤漂移 | 閉迴路控制（量測 → 模型 → 下一步電壓）、SAX 模型與留出驗證、溫度與雷射漂移補償 | 規劃中 |
| **3. AI 光子運算** | 以校正好的 MMI / MZI 網路執行矩陣運算 | 權重 → 相位 → 電壓映射、線上微調、運算結果回讀與誤差分析 | 規劃中 |

階段一的量測系統是後兩個階段的基礎：校正與運算都建立在同一套 DAC / ADC / 韌體 / Python 骨架上，
只是把 Host PC 端的「掃描腳本」換成「校正迴路」與「運算排程」。以下為階段一的系統規格。

## 階段一：量測系統

| 指標 | 規格 | 備註 |
|---|---|---|
| 待測晶片 | imec SiPhotonics 0.13 µm（IMEC-SiPh），1550 nm C-band | 多顆可調 MZI + 4 支 MEMS 探針 |
| MCU | Arduino GIGA R1 WiFi（STM32H747XI） | M7 480 MHz + M4 240 MHz，8 MB SDRAM |
| DAC | DAC81416-08EVM，16 ch，0–40 V | 加熱器直接驅動 + 探針被動分壓 |
| ADC | ADS9324EVM（改板），16 ch，16-bit SAR | 1 MSPS/ch，片上 oversampling |
| 穩態相位解析度 | ≈ 15–25 µrad（降頻 + 平均） | 片上 oversampling 至 4 kSPS |
| 暫態頻寬 | ≤ 325 kHz | 加熱器熱時間常數、探針機械響應 |
| 光偵測器 | InGaAs PD 模組（PDA10CS2） | 一柵一 PD，最多 4 探針 + 主輸出 + 參考 |
| Host PC | Python 掃描腳本、SAX 模型、擬合 | USB CDC，不在即時迴路內（雷射由 GIGA UART 控制） |
| 電源 | 60 V max 實驗室電源（限流）→ DAC VCC 預設 40 V | LM2596S 降壓 5.3 V → ADS9324EVM |
| 狀態 | **v3 規劃值，零件待購、datasheet 待核** | 2026-10-10 |

### 系統架構

![PIC-AI-Lab 階段一量測系統架構：Host PC、GIGA R1、DAC81416-08EVM、ADS9324EVM、PD 模組、可調雷射與 Si 光子晶片之間的連線，含光路、驅動電壓、類比量測、數位控制與電源五類訊號](docs/img/system-arch.svg?v=20261010d)

系統骨架：MCU 不直接驅動高壓，所有探針上限由 VCC 與被動分壓比在硬體上決定，韌體改不掉。
MCU 重置或斷電時 DAC 輸出歸零（nRESET 外部下拉）。完整規格見
[`docs/pic-test-system-spec.html`](docs/pic-test-system-spec.html)。

#### 責任分層

| 層級 | 負責 | 不負責 |
|---|---|---|
| Host PC | 掃描設定、資料紀錄、擬合、SAX 模型、凍結預測與留出驗證 | 任何需要即時時序的動作（含雷射控制） |
| GIGA R1 韌體 | DAC 寫入、settle 等待、ADC 取樣（CONVST / BAS / DMA）、平均與標準差、暫態波形暫存於 SDRAM、MAX\_CODE 與斜率限制、watchdog、上電自檢、**雷射控制（UART SCPI）**、被動板 GPIO（預留 MOS 開關）與分壓回授（內建 ADC） | 模型與決策 |
| 硬體（DAC EVM、被動轉接、電源） | POR / nRESET 歸零、探針分壓上限（參考可取 VDD–DAC）、電源限流、過溫關斷 | 最後一道防線，不依賴軟體 |

#### 主要元件

| 元件 | 型號 | 用途 | Datasheet |
|---|---|---|---|
| MCU | Arduino GIGA R1 WiFi（STM32H747XI） | 系統控制、SPI 通訊、暫態波形暫存 | [`datasheets/mcu/`](datasheets/mcu/) |
| DAC | DAC81416-08EVM（16 ch，20-bit） | 加熱器驅動（0–40 V）、探針驅動（經分壓） | [`datasheets/dac/`](datasheets/dac/) |
| ADC | ADS9324EVM（16 ch，16-bit SAR，1 MSPS） | 光功率、shunt 電流、探針電壓讀取 | [`datasheets/adc/`](datasheets/adc/) |
| PD | Thorlabs PDA10CS2（InGaAs） | 1550 nm 光偵測，帶放大 | [`datasheets/pd/`](datasheets/pd/) |
| 降壓模組 | LM2596S | ≤ 60 V → 5.3 V 給 ADS9324EVM | — |
| 保護二極體 | BAT54A（SOT23） | ADC 輸入保護 | [`datasheets/adc/`](datasheets/adc/) |

#### 通道配置

**ADC（16 ch）**

| 通道 | 第一版：單顆 MZI | 完整晶片（4 探針，多顆 MZI） |
|---|---|---|
| CH1–CH4 | 探針 ①② PD、主輸出 PD、輸入參考 PD | 探針 ①–④ 光柵 PD |
| CH5–CH8 | heater ①② shunt、探針 ①② 抽頭電壓 | 主輸出 PD ×N |
| CH9 | 備用 | 輸入參考 PD |
| CH10–CH13 | 備用 | 探針 ①–④ 抽頭電壓 |
| CH14–CH16 | 備用 | heater shunt ×3 |

**DAC（16 ch）**

| 通道 | 用途 | 範圍 |
|---|---|---|
| CH0–CH7 | 加熱器（最多 4 顆 MZI × 2 臂） | 0–40 V，直接接晶片，低端 shunt |
| CH8–CH11 | MEMS 探針 ①–④ | 0–40 V 經分壓 → 0–7 V；MAX\_CODE + 斜率限制 |
| CH12–CH15 | 備用 | dither / 第二晶片 |

#### 相位解析度估算

MZI 輸出 I ≈ (I₀/2)(1 + V cos φ)，正交點 dI/dφ ≈ I₀V/2。條紋擺幅佔半滿刻度時：

| | ADS9324（原生 1 MSPS） | ADS9324 降頻到 4 kSPS | ADS131M08（4 kSPS，v2） |
|---|---|---|---|
| SNR / 動態範圍 | 88 dB | 約 98–103 dB | 102 dB |
| rms 雜訊 | 4×10⁻⁵ | 約 1×10⁻⁵ | 8×10⁻⁶ |
| 單點相位解析度 | ≈ 80 µrad | ≈ 15–25 µrad | ≈ 16 µrad |

雷射功率漂移與熱漂移通常是 mrad 等級（比上面大兩個數量級），SNR 不是選型的決定因素；通道數與頻寬才是。

### 電源樹

![電源樹：60 V max 電源（預設 40 V）分成限流支路給 DAC VCC，與 LM2596S 降壓支路給 ADS9324EVM；GIGA 由 USB 供電](docs/img/power-tree.svg?v=20261010)

| 軌 | 來源 | 電壓 | 負載 |
|---|---|---|---|
| DAC VCC | 實驗室電源 → 限流 | 預設 40 V（電源最大 60 V） | 加熱器、探針分壓、DAC 輸出級耗散 |
| EVM LDO\_IN | LM2596S 模組 | 5.3 V（J19） | ADS9324 + 板上 LDO |
| AVDD 5 V / 1.8 V | EVM U5 / U6 | 5.0 / 1.8 V | ADC 類比 |
| IOVDD | EVM U4（TPS7A2033） | 3.3 V | ADC 數位 I/O |
| GIGA | PC USB-C | 5 V | MCU、SDRAM |

### ADS9324EVM 改板重點

EVM（DC392）為 FMC + FPGA 設計，接 3.3 V MCU 需五項改板：

1. **電源改外接** — JP1 跳到 [2-3] EXT\_PWR；J19 端子台接 5.2–5.5 V。
2. **IOVDD 改 3.3 V** — U4 貼 TPS7A2033PDBVR、R76 貼 0 Ω、R75 拆掉。
3. **斷開位準轉換器** — 拆 R86–R91、R93–R95（U11/U12 B 側 0 Ω）。
4. **補控制腳** — REFSEL 與 RESETn 由 MCU GPIO 驅動。
5. **接線** — J20/J21 走 1-lane SPI，排線 ≤ 10 cm，SCLK 串 22–33 Ω。

| EVM 腳 | 訊號 | 方向 | GIGA |
|---|---|---|---|
| J21-4 | SCLK | ← MCU | D91 |
| J21-5 | SDI | ← MCU | D90 |
| J21-3 | SDOUT | → MCU | D89 |
| J21-6 | CSn | ← MCU | D8 |
| J20-1 | CONVST | ← MCU | D3（timer） |
| J20-2 | BAS（DRDY） | → MCU | D2（中斷） |
| J21-7 | RESETn | ← MCU | D4 |
| J21-8 | REFSEL | ← MCU | D9 |
| J19 | 5.3 V | 電源 | LM2596S |

### 驗收順序

1. **EVM 電源驗證** — 改板後只上 J19，不接 MCU：量 TP8（5 V）、TP7（1.8 V）、TP9（3.3 V）。
2. **SPI 通訊** — 接 MCU，照初始化順序讀 device ID。
3. **雜訊基底** — 輸入短路（J1–J16 跳帽），量雜訊直方圖，比對 datasheet。
4. **線性度** — DAC 餵已知電壓進 ADC，量線性度與增益；驗證 SPI-A 與 SPI-B 無衝突。
5. **安全限制** — 假負載取代晶片：MAX\_CODE、斜率限制、nRESET 歸零、watchdog 逾時歸零。
6. **E0 量測** — 接 TIA 與 PD，雷射開，跑單顆加熱器 I-V 與 P₂π。
7. **探針特性** — 慢速升壓找解耦電壓與 pull-in 餘裕，定案分壓比。
8. **暫態擷取** — heater 步階、探針步階，SDRAM 緩衝與 USB 倒出不掉資料。

## 目錄結構

| 路徑 | 內容 |
|---|---|
| `docs/` | 系統規格（HTML）、簡報、heater driver proposal |
| `docs/img/` | 架構圖（SVG）、元件圖片 |
| `docs/adr/` | 架構決策紀錄（ADR） |
| `datasheets/` | 元件 datasheet，依類別分資料夾：`adc/`、`dac/`、`pd/`、`mcu/` |
| `firmware/` | Arduino GIGA R1 韌體（待開發） |
| `sw/` | Host PC 端 Python（待開發） |
| `sw/measure/` | 階段一：掃描腳本與資料處理 |
| `sw/calib/` | 階段二：閉迴路校正、SAX 模型、留出驗證 |
| `sw/compute/` | 階段三：權重 → 相位 → 電壓映射、運算排程、誤差回讀 |
| `sw/scripts/` | 工具腳本 |
| `data/calib/` | 校正表（JSON，依晶片 ID 與日期分資料夾，含版本與驗證誤差） |
| `hw/` | 硬體設計檔案 |
| `hw/passive-board/` | 被動轉接板（探針分壓、shunt） |

## 待確認事項

- ADS9324 datasheet：power sequencing、1-lane 模式資料輸出腳位（SDOUT 或 D0）、oversampling 設定與 24-bit 輸出格式。
- GIGA R1 官方 pinout：D3 timer 輸出、D2 中斷、D89–D91 SPI1、開機與燒錄期間各腳位狀態。
- ADS9324EVM 改板後實測三軌電壓與 IOVDD 電流；LM2596S 模組實品的輸入電容耐壓與 IC 來源。
- 原團隊 netlist：MZI 數量與連接、探針位置、輸出埠數（決定 PD 數與 ADC 通道）。
- 探針取光比對電壓的曲線是否單調、是否有遲滯。
- 加熱器電阻 R\_h（決定 shunt、電流與 DAC 耗散）。
- BP-ADS9324 供貨恢復時是否改回 BoosterPack（省掉改板）。

## 版本歷程

| 版本 | 日期 | MCU | ADC | 重點變更 |
|---|---|---|---|---|
| v1 | — | PYNQ-Z2 | ΔΣ ADC | 初始規劃 |
| v2 | — | ESP32-S3 | ADS131M08（8 ch，24-bit ΔΣ） | 改用 MCU 方案 |
| v3 | 2026-10-08 | GIGA R1 WiFi | ADS9324（16 ch，16-bit SAR） | 通道數 ↑、暫態頻寬 ↑、SDRAM 暫存 |
| **v3.1** | 2026-10-10 | **GIGA R1 WiFi** | **ADS9324** | 更名 PIC-AI-Lab、三階段路線圖、電源 60 V / VCC 預設 40 V、雷射改 GIGA UART SCPI、被動板 GPIO 與分壓回授 |
