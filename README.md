# PIC 量測系統 — MEMS 探針 + 多級 MZI 晶片的 DC 掃描與暫態擷取平台

PIC 量測系統是一套針對 **SOI 220 nm 矽光子（PIC）晶片** 設計的量測平台，用於量測晶片上
**多顆可調 MZI 分光器** 與 **4 支 MEMS 光探針** 的 DC 相位掃描與暫態響應。平台以
**Arduino GIGA R1 WiFi（STM32H747）** 為控制核心，搭配 **DAC81416-08EVM** 驅動加熱器與探針，
**ADS9324EVM（改板）** 讀取 16 通道光功率與電壓訊號。

| 指標 | 規格 | 備註 |
|---|---|---|
| 待測晶片 | SOI 220 nm，1550 nm C-band | 多顆可調 MZI + 4 支 MEMS 探針 |
| MCU | Arduino GIGA R1 WiFi（STM32H747XI） | M7 480 MHz + M4 240 MHz，8 MB SDRAM |
| DAC | DAC81416-08EVM，16 ch，0–40 V | 加熱器直接驅動 + 探針被動分壓 |
| ADC | ADS9324EVM（改板），16 ch，16-bit SAR | 1 MSPS/ch，片上 oversampling |
| 穩態相位解析度 | ≈ 15–25 µrad（降頻 + 平均） | 片上 oversampling 至 4 kSPS |
| 暫態頻寬 | ≤ 325 kHz | 加熱器熱時間常數、探針機械響應 |
| 光偵測器 | InGaAs PD 模組（PDA10CS2） | 一柵一 PD，最多 4 探針 + 主輸出 + 參考 |
| Host PC | Python 掃描腳本、SAX 模型、擬合 | USB CDC，不在即時迴路內 |
| 電源 | 36 V 實驗室電源（限流）→ DAC VCC | LM2596S 降壓 5.3 V → ADS9324EVM |
| 狀態 | **v3 規劃值，零件待購、datasheet 待核** | 2026-10-08 |

## 系統架構

詳細系統規格與架構圖見 [`docs/pic-test-system-spec.html`](docs/pic-test-system-spec.html)。

```
┌──────────┐    LAN/SCPI    ┌──────────────┐
│ 可調雷射  │───────────────→│   Si 光子晶片  │←── 光纖輸入
│（C-band） │                │  SOI 220 nm   │
└──────────┘                │  多顆 MZI     │
                            │  4 支 MEMS 探針│
┌──────────┐    USB CDC     │  加熱器 θ₁/θ₂  │
│ Host PC  │←──────────────→│              │
│ Python   │                └──┬───┬───┬───┘
└──────────┘                   │   │   │
       │                 heater│probe│  │光柵/光纖出光
       │              0-40V   │≤7V  │  │
       │                ┌─────┴─────┘  │
       │     SPI-A      │              │
       ├───────────────→│ DAC81416     │
       │                │ -08EVM       │
┌──────┴───┐            │ 16 ch DAC    │
│ GIGA R1  │            └──────┬───────┘
│ WiFi     │                   │shunt/抽頭
│ STM32H747│     SPI-B    ┌────┴───────┐    ┌──────────┐
│ 8MB SDRAM│←────────────→│ ADS9324EVM │←───│ PD 模組  │
└──────────┘              │ 16 ch ADC  │    │ InGaAs   │
                          │（改板）     │    └──────────┘
```

### 責任分層

| 層級 | 負責 | 不負責 |
|---|---|---|
| Host PC | 掃描設定、資料紀錄、擬合、SAX 模型、凍結預測與留出驗證、雷射控制 | 任何需要即時時序的動作 |
| GIGA R1 韌體 | DAC 寫入、settle 等待、ADC 取樣（CONVST/BAS/DMA）、平均與標準差、暫態波形暫存、MAX\_CODE 與斜率限制、watchdog | 模型與決策 |
| 硬體（DAC EVM、被動轉接、電源） | POR/nRESET 歸零、探針分壓上限、電源限流、過溫關斷 | 最後一道防線，不依賴軟體 |

### 主要元件

| 元件 | 型號 | 用途 | Datasheet |
|---|---|---|---|
| MCU | Arduino GIGA R1 WiFi（STM32H747XI） | 系統控制、SPI 通訊、暫態波形暫存 | [`datasheets/mcu/`](datasheets/mcu/) |
| DAC | DAC81416-08EVM（16 ch，20-bit） | 加熱器驅動（0–40 V）、探針驅動（經分壓） | [`datasheets/dac/`](datasheets/dac/) |
| ADC | ADS9324EVM（16 ch，16-bit SAR，1 MSPS） | 光功率、shunt 電流、探針電壓讀取 | [`datasheets/adc/`](datasheets/adc/) |
| PD | Thorlabs PDA10CS2（InGaAs） | 1550 nm 光偵測，帶放大 | [`datasheets/pd/`](datasheets/pd/) |
| 降壓模組 | LM2596S | 36 V → 5.3 V 給 ADS9324EVM | — |
| 保護二極體 | BAT54A（SOT23） | ADC 輸入保護 | [`datasheets/adc/`](datasheets/adc/) |

## 目錄結構

| 路徑 | 內容 |
|---|---|
| `docs/` | 系統規格（HTML）、簡報、架構決策紀錄 |
| `docs/img/` | 系統圖片（PD 模組、OLS 等） |
| `docs/adr/` | 架構決策紀錄（ADR） |
| `datasheets/` | 元件 datasheet，依類別分資料夾：`adc/`、`dac/`、`pd/`、`mcu/` |
| `firmware/` | Arduino GIGA R1 韌體（待開發） |
| `sw/` | Host PC 端 Python 掃描腳本與資料處理（待開發） |
| `sw/scripts/` | 工具腳本 |
| `hw/` | 硬體設計檔案 |
| `hw/passive-board/` | 被動轉接板（探針分壓、shunt） |

## 驗收順序

1. EVM 改板後只上 J19 與 TP9，不接 MCU：量三軌電壓。
2. 接 MCU，讀 device ID。
3. 輸入短路，量雜訊直方圖，比對 datasheet。
4. DAC 餵已知電壓進 ADC，量線性度與增益。
5. 假負載取代晶片：驗證安全限制（MAX\_CODE、斜率限制、nRESET 歸零、watchdog）。
6. 接 TIA 與 PD，雷射開，跑單顆加熱器 I-V 與 P₂π（E0）。
7. 探針慢速升壓找解耦電壓與 pull-in 餘裕。
8. 暫態擷取：heater 步階、探針步階，SDRAM 緩衝與 USB 倒出。

## 待確認事項

- ADS9324 datasheet：power sequencing、1-lane 模式資料輸出腳位、oversampling 設定。
- GIGA R1 pinout：D3 timer 輸出、D2 中斷、D89–D91 SPI1、開機期間腳位狀態。
- ADS9324EVM 改板後實測三軌電壓與 IOVDD 電流。
- 原團隊 netlist：MZI 數量與連接、探針位置、輸出埠數。
- 加熱器電阻 R\_h（決定 shunt、電流與 DAC 耗散）。

## 版本歷程

| 版本 | 日期 | MCU | ADC | 重點變更 |
|---|---|---|---|---|
| v1 | — | PYNQ-Z2 | ΔΣ ADC | 初始規劃 |
| v2 | — | ESP32-S3 | ADS131M08（8 ch，24-bit ΔΣ） | 改用 MCU 方案 |
| **v3** | 2026-10-08 | **GIGA R1 WiFi** | **ADS9324**（16 ch，16-bit SAR） | 通道數 ↑、暫態頻寬 ↑、SDRAM 暫存 |
