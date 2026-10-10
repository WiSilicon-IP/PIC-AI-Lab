# PIC 量測系統開發規則

## 基本原則

**一律以繁體中文回覆。**

**安全優先，硬體決定上限，軟體不得繞過。**

量測系統涉及高壓（最高 40 V）驅動矽光子晶片上的加熱器與 MEMS 探針，探針有 pull-in 風險。所有安全邊界由硬體（被動分壓、電源限流、DAC POR 歸零）決定，韌體只做額外的軟體限制。

## 1. 安全設計

- 探針電壓上限由 VCC 與被動分壓比在硬體上決定，韌體的 `MAX_CODE` 與斜率限制是第二道防線。
- DAC nRESET 外部下拉：MCU 重置、燒錄或斷電時輸出歸零。
- 電源限流限制加熱器總功率。
- 任何碼值寫入前，必須通過 `MAX_CODE` 與斜率限制檢查。
- Watchdog 逾時必須使所有輸出歸零。

## 2. 韌體架構（GIGA R1）

- **雙核分工**：M4 跑 DAC 掃描序列與安全限制，M7 收 ADC 與 DMA。兩核透過共享記憶體溝通。
- **SPI 分離**：SPI-A（主排針 D10–D13）專用 DAC，SPI-B（D89–D91）專用 ADC，不共用匯流排。
- **CONVST 由硬體 timer 產生**，不依賴軟體延遲，確保取樣時序精確。
- **暫態波形存 SDRAM**，再經 USB CDC 倒出；不做即時串流。
- **雷射控制**：可調雷射由 GIGA UART（Serial1/2/3）以 SCPI 文字命令控制波長與功率，不經 Host PC。若雷射為 RS-232 介面需加 MAX3232 電壓轉換。
- **被動轉接板介面**：GIGA GPIO 預留給板上 MOS 開關（未來擴充）；板上分壓抽頭另接回 GIGA 內建 ADC，做校正與偵測，不佔 ADS9324 通道。
- Arduino core 建在 Mbed OS 上，SPI DMA 與 timer 觸發需直接使用 STM32 HAL。

## 3. Host PC 軟體（Python）

- Host PC 不在即時迴路內，只負責掃描設定、資料紀錄、擬合與模型。雷射控制已移至 GIGA 韌體。
- 通訊協定：USB CDC，文字命令 / 二進位資料。
- 資料格式：每筆量測包含 DAC 碼值、ADC 讀值、時間戳。
- 擬合自變數用實測功率（V² / R_h），不用電壓。

## 4. ADC 使用規範（ADS9324）

- 輸入範圍 ±2.5 V，TIA 增益配合讓條紋吃滿刻度。
- 穩態掃描：類比頻寬 25 kHz，片上 oversampling 降到 4–32 kSPS。
- 暫態擷取：類比頻寬 325 kHz，連續存 SDRAM。
- 1-lane SPI 模式，SCLK 從 10–20 MHz 開始，看波形再加速。
- 上電順序：EVM 先上電，GIGA 後接；所有 ADC GPIO 在確認上電前保持 Hi-Z。

## 5. DAC 使用規範（DAC81416）

- VCC 預設 40 V（電源最大 60 V），E0 量得 R_h 與 V₂π 後調整實際電壓。
- 加熱器通道 0–40 V 直接輸出。
- 探針通道經被動分壓到 0–7 V，`set_probe` 接受連續碼值（不是兩態開關）。
- J18 開路、外部 SPI。
- nLDAC 同步更新所有通道。

## 6. 電源設計

- 實驗室電源最大 60 V，預設 40 V 給 DAC VCC，限流支路保護。LM2596S 從限流之前取電。
- 被動轉接板預留 VCC 參考輸入：探針分壓參考可取 VDD–DAC 兩端，不限定 DAC–GND。
- 限流動作時只有加熱器掉電，MCU 與 ADC 不受影響。
- GIGA 由 USB 供電，不從高壓取電。
- LM2596S-ADJ 輸入上限 40 V：若電源超過 40 V 需改用 LM2596HV 或獨立降壓模組。

## 7. 檔案與命名慣例

- 文件以 kebab-case 命名（`pic-test-system-spec.html`）。
- Datasheet 依元件類別放在 `datasheets/` 下。
- 韌體放 `firmware/`，Python 腳本放 `sw/`。
- 硬體設計放 `hw/`。
- 圖片放 `docs/img/`。

## 8. 量測協定

- 每個量測點：設定 DAC → settle 等待 → CONVST 觸發 → 讀 M 筆 ADC → 算平均與標準差。
- 掃描序列由 Host PC 下達，GIGA 執行。
- 所有量測資料包含載台溫度（熱敏電阻 → MCU 內建 ADC）。
