<!-- markdownlint-disable MD033 -->
# 使用 NuTRM 與 NuCodeGen 將 STM32F103 程式碼移植到 NuMicro M487

## 簡介

本文件說明**如何使用兩個 VS Code 擴充套件 — NuTRM 與 NuCodeGen — 作為 AI 輔助工具**，加速您的移植流程。

不再需要手動翻閱 datasheet、交叉比對暫存器、從頭撰寫初始化程式碼 — 讓這些工具替您處理繁瑣的工作，您只需專注於應用邏輯。

| 擴充套件 | 如何幫助您 | 連結 |
|---------|----------|------|
| **NuTRM** | 詢問 Nuvoton 晶片的周邊、暫存器、時脈與腳位配置 — 即時取得附有 TRM 引用的答案 | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-trm-chatbot) |
| **NuCodeGen** | 向 NuCodeGen 代理詢問，透過聊天提示產生目標 Nuvoton 晶片的初始化 C 程式碼 — 時脈、GPIO 與周邊設定 | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-nucodegen) |

### 您需要準備

- 您現有的 STM32F103（或其他競品）原始碼
- 競品晶片料號（例如 STM32F103）
- 目標 Nuvoton NuMicro 晶片（例如 M487）

---

## 環境設定

1. 安裝 **VS Code** 1.110 或更高版本。
2. 安裝並登入 **GitHub Copilot**。
3. 從 [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-trm-chatbot) 安裝 **NuTRM** 擴充套件。
4. 從 [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-nucodegen) 安裝 **NuCodeGen** 擴充套件。
5. 在 VS Code 中開啟您的 STM32F103 專案。

---

## 使用工具 — 逐步說明

以下說明 NuTRM 與 NuCodeGen 如何在每一步輔助您。

### 步驟 1 — 分析您的 STM32F103 程式碼（Copilot 輔助）

在 VS Code 中開啟您的 STM32F103 專案，並請 **Copilot Chat** 為您分析：

**提示詞：**

```text
Analyze the source code in this workspace. This is a project for an STM32F103 MCU.
Extract and summarize:
1. All peripherals used (UART, SPI, I2C, Timer, PWM, ADC, DMA, GPIO, etc.)
2. For each peripheral: instance name, configuration (baud rate, clock speed, mode, etc.)
3. Pin assignments (which GPIO pins are mapped to which functions)
4. Clock configuration (system clock source, PLL, AHB/APB dividers)
5. Interrupt handlers (IRQ names and what they do)
6. DMA channels and their trigger sources

Output as a structured markdown table.
```

Copilot 會掃描您的 `.c` / `.h` 檔案，回傳如下的摘要：

#### 範例輸出

```markdown
| 周邊 | 實例 | 設定 | 腳位 | IRQ |
|------|------|------|------|-----|
| UART | USART1 | 115200, 8N1 | PA9 (TX), PA10 (RX) | USART1_IRQHandler |
| SPI | SPI1 | Master, CPOL=0, CPHA=0, 10 MHz | PA5 (SCK), PA6 (MISO), PA7 (MOSI) | — |
| I2C | I2C1 | Master, 400 kHz | PB6 (SCL), PB7 (SDA) | I2C1_EV_IRQHandler |
| Timer | TIM2 | 1 ms 週期性 | — | TIM2_IRQHandler |
| ADC | ADC1 | Ch0, Ch1, 12-bit, 單次轉換 | PA0, PA1 | — |
| GPIO | GPIOA | PA5 輸出 (LED) | PA5 | — |
| 時脈 | — | HSE 8 MHz → PLL → 72 MHz SYSCLK | — | — |
```

如此即可取得完整的待移植周邊清單 — 不需逐一閱讀每個原始碼檔案。

---

### 步驟 2 — 查詢對應的 NuMicro 周邊（NuTRM 輔助）

將步驟 1 的周邊清單交給 **NuTRM** 查詢 Nuvoton 對應的周邊。開啟 Copilot Chat 面板，選擇 **NuTRM** 代理，並使用 `/{chip}` 斜線指令：

**提示詞：**

```text
/M480 I'm migrating from STM32F103 to M487. I need the following peripherals:
- UART: 115200 baud, 8N1 (was USART1 on STM32)
- SPI: Master mode, 10 MHz (was SPI1)
- I2C: Master, 400 kHz fast mode (was I2C1)
- Timer: 1 ms periodic interrupt (was TIM2)
- ADC: 2 channels, 12-bit (was ADC1 Ch0/Ch1)
- GPIO: 1 LED output

For each, tell me:
1. Which M487 peripheral instance to use
2. Available pins (multi-function pin selection)
3. Any feature differences or limitations compared to STM32F103
```

NuTRM 搜尋 M487 技術參考手冊（TRM），並回傳附有**可點擊 TRM 引用**的答案 — 點擊任一連結即可直接跳至官方文件。

#### 範例結果 — 周邊對應表

將 Copilot 的分析（步驟 1）與 NuTRM 的答案合併，建立對應表：

| 功能 | STM32F103 | M487（來自 NuTRM） | 備註 |
|------|-----------|-------------------|------|
| 除錯 UART | USART1 (PA9/PA10) | UART0 (PB12 TX / PB13 RX) | 兩者皆支援 115200 8N1 |
| SPI Flash | SPI1 Master (PA5–PA7) | SPI0 Master (PA0–PA3) | M487 SPI0 最高支援 50 MHz |
| I2C 感測器 | I2C1 (PB6/PB7) | I2C0 (PA4 SDA / PA5 SCL) | 支援 400 kHz 快速模式 |
| 系統計時 | TIM2, 1 ms | TIMER0, 週期模式 | 使用 TIMER_Open() 設定 1000 Hz |
| 類比輸入 | ADC1 Ch0/Ch1 (PA0/PA1) | EADC Ch0/Ch1 (PB0/PB1) | M487 使用增強型 ADC (EADC) |
| LED | GPIOA PA5 | GPIOC PC9 | 直接 GPIO 推挽輸出 |
| 系統時脈 | HSE 8 MHz → PLL → 72 MHz | HXT 12 MHz → PLL → 192 MHz | 需調整時序常數 |

> **NuTRM 優勢：** 每個答案都附有 TRM 章節的直接連結，不需手動搜尋 PDF。

---

### 步驟 3 — 產生初始化程式碼（NuCodeGen 輔助）

利用步驟 2 的周邊對應表，使用 **NuCodeGen** 聊天代理產生 Nuvoton **啟動初始化程式碼** — 在應用邏輯執行前的一次性設定：`SYS_Init()`、時脈配置與 MFP 腳位指定。

開啟 Copilot Chat 面板，選擇 **NuCodeGen** 代理，告訴它您的需求：

**提示詞：**

```text
I'm using M487JIDAE. Generate initialization code for:
- System clock: HXT 12 MHz → PLL → 192 MHz
- UART0: 115200 baud, 8N1
- SPI0: Master, CPOL=0, CPHA=0, 10 MHz
- I2C0: 400 kHz
- TIMER0: Periodic, 1 ms interrupt
- EADC: Ch0, Ch1 enabled
- PC.9: GPIO output (LED)
```

> **NuCodeGen 優勢：** 以自然語言描述需求即可取得正確的 API 層級初始化程式碼 — 不需自行研讀 BSP。產生的程式碼會以正確順序處理時脈閘控、MFP 腳位選擇與周邊配置。

---

### 步驟 4 — 將 STM32 HAL 呼叫替換為 Nuvoton BSP（Copilot + NuCodeGen + NuTRM 輔助）

步驟 3 已產生初始化程式碼，現在移植您的**應用邏輯** — 在執行階段傳送資料、讀取感測器和處理中斷的程式碼。

STM32 HAL 與 Nuvoton BSP 的 **API 完全不同** — STM32 使用結構控制代碼（`&huart1`），Nuvoton 使用周邊指標（`UART0`）。無法用簡單的 `#define` 巨集橋接。請使用 **Copilot** 改寫程式碼，並用 **NuCodeGen** 產生正確的 Nuvoton BSP 使用範例。

#### 4a. 向 NuCodeGen 取得執行階段 API 範例

步驟 3 產生的是 *初始化* 程式碼（啟動時呼叫一次）。現在您需要知道如何在執行階段 *使用* 各周邊 — 傳送位元組、讀取感測器、處理轉換。向 **NuCodeGen** 詢問：

**提示詞（NuCodeGen 代理）：**

```text
I'm using M487JIDAE. Show me how to:
- Send and receive data on UART0
- Do a SPI master transmit/receive on SPI0
- Write/read bytes as I2C master on I2C0
- Start and read EADC conversion on channel 0
- Set up a TIMER0 periodic interrupt at 1 ms
```

NuCodeGen 回傳使用 Nuvoton BSP API 的可直接使用的程式碼片段。請保持此輸出在聊天中可見 — Copilot 會在下一步中參考它。

#### 4b. 請 Copilot 改寫原始碼

在聊天上下文中仍保有步驟 4a 的 NuCodeGen BSP 範例的情況下，請 **Copilot** 替換您專案中的 STM32 HAL 呼叫：

**提示詞（Copilot）：**

```text
I'm porting this STM32F103 project to Nuvoton M487.
Use the Nuvoton BSP examples from the NuCodeGen output above as reference.
Here is the peripheral mapping:
- USART1 → UART0
- SPI1 → SPI0
- I2C1 → I2C0
- TIM2 → TIMER0
- ADC1 → EADC
- GPIOA PA5 → GPIOC PC9

For every STM32 HAL call in the source code, rewrite it using the Nuvoton BSP API.
Also update all interrupt handler function names to M487 equivalents.
```

#### 範例 — 改寫前後對照

| STM32 HAL（改寫前） | Nuvoton BSP（改寫後） |
|---|---|
| `HAL_UART_Transmit(&huart1, buf, len, 1000)` | `UART_Write(UART0, buf, len)` |
| `HAL_UART_Receive_IT(&huart1, buf, len)` | `UART_Read(UART0, buf, len)` |
| `HAL_SPI_TransmitReceive(&hspi1, tx, rx, len, 1000)` | `SPI_WRITE_TX(SPI0, tx[i])` / `SPI_READ_RX(SPI0)` 以迴圈處理 |
| `HAL_I2C_Master_Transmit(&hi2c1, addr, buf, len, 1000)` | `I2C_WriteByte(I2C0, addr, buf[i])` |
| `HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET)` | `GPIO_SET_PIN(PC, BIT9)` |
| `HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5)` | `GPIO_TOGGLE(PC, BIT9)` |
| `HAL_ADC_Start(&hadc1)` | `EADC_START_CONV(EADC, BIT0 \| BIT1)` |
| `HAL_ADC_GetValue(&hadc1)` | `EADC_GET_CONV_DATA(EADC, 0)` |

#### 中斷處理函式重新命名

STM32 與 Nuvoton 使用**不同的 IRQ 函式名稱**。向量表依據精確名稱呼叫這些函式 — 使用錯誤名稱代表您的處理函式將永遠不會被執行。

| STM32 IRQ Handler | Nuvoton M487 對應 |
|---|---|
| `USART1_IRQHandler()` | `UART0_IRQHandler()` |
| `SPI1_IRQHandler()` | `SPI0_IRQHandler()` |
| `I2C1_EV_IRQHandler()` | `I2C0_IRQHandler()` |
| `TIM2_IRQHandler()` | `TMR0_IRQHandler()` |
| `ADC1_2_IRQHandler()` | `EADC0_IRQHandler()` |

> **關鍵差異：** STM32 HAL 使用具有逾時參數的控制代碼式 API（例如 `HAL_UART_Transmit(&huart1, buf, len, 1000)`）。Nuvoton BSP 使用不含逾時的直接周邊指標 API（例如 `UART_Write(UART0, buf, len)`）。Copilot 可以為您改寫，但請檢查結果以確保逾時/錯誤處理邏輯在應用層中得到保留。

#### 4c. 檢查時序相關程式碼

**和時脈頻率相關的程式碼，從 72 MHz（STM32F103）搬到 192 MHz（M487）後可能會出錯**。

請 **Copilot** 幫您找出所有時序敏感的程式碼：

**提示詞：**

```text
Search this project for all timing-dependent code:
- Hard-coded delay loops (for-loop delays, busy-wait counters)
- HAL_Delay() calls (STM32 uses ms; Nuvoton CLK_SysTickDelay() uses µs)
- Timer reload/prescaler values calculated from a clock frequency constant
- Baud rate divisor calculations that reference a specific clock speed
- Timeout constants in communication protocols
- Any #define or variable containing clock frequency (e.g., 72000000, SYSCLK, HSE_VALUE)

For each, show the file, line, and what needs to change for M487 at 192 MHz HCLK.
```

**需要注意的項目：**

| 時序模式 | STM32F103 (72 MHz) | M487 (192 MHz) | 修正方式 |
|---|---|---|---|
| `HAL_Delay(100)` — 100 ms 延遲 | 使用 SysTick 1 kHz | `CLK_SysTickDelay(100000)` — 單位是 **µs** 而非 ms | 將 ms 轉換為 µs，或建立 1 ms SysTick ISR 搭配計數器 |
| `for(i=0; i<10000; i++)` 忙等 | 在 72 MHz 下約 X µs | 在 192 MHz 下約 X/2.67 µs（快 2.67 倍） | 改用 `CLK_SysTickDelay()` 或計時器延遲 |
| `#define SYSCLK 72000000` | 用於鮑率/計時器計算 | 錯誤 — M487 是 192 MHz | 改用 `SystemCoreClock` 或 `CLK_GetHCLKFreq()` |
| Timer prescaler: `PSC = 72 - 1` | 72 MHz / 72 = 1 MHz tick | M487 的計時器時脈可能不同 | 使用 `CLK_GetPCLK0Freq()` 或 `CLK_GetPCLK1Freq()` 推算 |
| UART 鮑率除頻器：手動計算 | 基於 APB2 = 72 MHz | UART 時脈來源可能不同 | 使用 `UART_Open(UART0, 115200)` — BSP 會自動計算除頻值 |

> 不要硬編碼時脈頻率。所有時序都應從 `SystemCoreClock`、`CLK_GetHCLKFreq()`、`CLK_GetPCLK0Freq()` 或 `CLK_GetPCLK1Freq()` 推導。Nuvoton BSP API（如 `UART_Open()`、`TIMER_Open()`、`SPI_Open()`）內部會自動處理時脈推導 — 請使用這些 API，而非手動計算暫存器值。

**NuTRM** 可以幫助您了解 M487 的時脈架構：

```text
/M480 What are the clock sources for UART0, SPI0, TIMER0, and EADC on M487?

/M480 Show the clock tree — which peripherals use PCLK0 vs PCLK1?

/M480 What is the relationship between HCLK, PCLK0, and PCLK1 on M487?
```

---

### 步驟 5 — 在硬體上驗證並除錯

Copilot 改寫 API 呼叫後，您需要驗證每個周邊是否正常運作。由於這是直接與真實硬體互動的裸機 MCU 程式碼，**傳統的單元測試並不實用** — BSP 呼叫直接操作您電腦上不存在的周邊暫存器。

#### 5a. 在 M487 開發板上逐一測試每個周邊

| 順序 | 周邊 | 驗證方式 |
|------|------|---------|
| 1 | **時脈** | 用示波器檢查腳位上的系統時脈輸出，或在除錯器中驗證 `SystemCoreClock` 值 |
| 2 | **GPIO (LED)** | LED 閃爍 — 最簡單的健全性檢查 |
| 3 | **UART** | 發送 "Hello" 字串，並在序列埠終端（如 PuTTY、Tera Term）中確認 |
| 4 | **Timer** | 在計時器 ISR 中切換 GPIO 腳位，用示波器或邏輯分析儀量測間隔 |
| 5 | **SPI** | 讀取外部 SPI Flash 的 JEDEC ID，並與 datasheet 值比對 |
| 6 | **I2C** | 讀取感測器的 WHO_AM_I 暫存器，並與 datasheet 值比對 |
| 7 | **ADC (EADC)** | 輸入已知電壓（如 1.65 V = 3.3 V 的一半），檢查轉換結果是否約為 2048（12-bit） |
| 8 | **時序** | 若原本有 `HAL_Delay(100)`（100 ms），用示波器確認移植後的延遲仍為 100 ms。檢查 Timer ISR 間隔是否符合預期。透過 UART 印出 `SystemCoreClock` 確認值為 192000000 |

> **提示：** 依照上述順序測試。如果時脈不正確，其他功能都不會正常運作。如果 UART 正常，您就能用 `printf` 來除錯其他所有周邊。

您可以請 Copilot 產生簡易測試程式：

**提示詞：**

```text
Generate a minimal test for each peripheral on M487:
- UART0: transmit "Hello M487\n" at 115200 baud
- GPIO PC.9: blink LED at 1 Hz
- TIMER0: toggle a GPIO pin every 1 ms in ISR
- SPI0: read SPI flash JEDEC ID (command 0x9F)
- I2C0: read one byte from slave address 0x68 register 0x75
- EADC: convert channel 0 and print result via UART

Each test should be a standalone function I can call from main().
```

#### 5b. 當某個周邊不正常時 — 詢問 NuTRM

當周邊行為不如預期時，使用 **NuTRM** 深入查詢暫存器層級的細節，而非翻閱 1000 頁以上的 PDF：

```text
/M480 Show the SPI_CTL register bit fields for SPI0

/M480 What is the EADC module's sample-and-hold time configuration?

/M480 Show the multi-function pin register for UART0 TX on PB12

/M480 What is the interrupt vector number for UART0?

/M480 What are the clock divider options for TIMER0?
```

每個答案都附有**可點擊的 TRM 引用** — 點擊連結即可跳至官方 TRM 文件的精確章節。

> **NuTRM 常見除錯場景：**
> - UART 輸出亂碼 → 檢查鮑率除頻暫存器和時脈來源
> - SPI 無回應 → 驗證 MFP 腳位指定和時脈極性/相位設定
> - Timer ISR 從不觸發 → 確認正確的 IRQ 向量名稱和 NVIC 啟用
> - I2C 無 ACK → 檢查上拉電阻配置和匯流排時脈頻率暫存器
> - ADC 結果始終為 0 → 驗證通道對應和觸發來源配置

---

## 總覽：各工具輔助對應

```
┌───────────────────────────────────────┐
│   您的 STM32F103 原始碼               │
│   + 晶片料號                          │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  步驟 1：Copilot 分析您的程式碼       │  ← 理解您的 STM32F103 專案
│  → 「我用了哪些周邊？」               │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  步驟 2：NuTRM 查詢 Nuvoton 對應     │  ← 為您搜尋 M487 TRM
│  → 「M487 哪個周邊對應？」            │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  步驟 3：NuCodeGen 代理產生初始化     │  ← 透過聊天提示詢問
│  → 時脈、GPIO、周邊設定               │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  步驟 4：NuCodeGen + Copilot 移植    │  ← NuCodeGen 展示 BSP 用法，
│  → 改寫 HAL 呼叫 + IRQ 重新命名      │    Copilot 改寫程式碼
│  → 檢查時序相關程式碼                 │    （延遲、預除頻、時脈）
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  步驟 5：硬體驗證 + NuTRM 除錯       │  ← 測試周邊
│  → 以 TRM 暫存器資訊修正問題          │
└───────────────────────────────────────┘
```

---

## 快速提示詞參考

| 移植步驟 | 輔助工具 | 提示詞 |
|---------|---------|--------|
| 分析 STM32 程式碼 | Copilot | `Analyze this STM32F103 project. List all peripherals, pins, clocks, IRQs, and DMA as a table.` |
| 查詢 NuMicro 對應 | NuTRM | `/M480 I'm migrating from STM32F103. I need UART 115200, SPI Master 10 MHz, I2C 400 kHz, Timer 1 ms, ADC 2ch. Map each to M487.` |
| 產生初始化程式碼 | NuCodeGen | `I'm using M487JIDAE. Generate init code for UART0 115200, SPI0 Master 10 MHz, I2C0 400 kHz, TIMER0 1 ms, EADC Ch0/Ch1, PC.9 GPIO output.` |
| 取得 BSP 使用範例 | NuCodeGen | `I'm using M487JIDAE. Show me how to send/receive on UART0, SPI0 master transfer, I2C0 read/write, EADC convert ch0, TIMER0 1ms ISR.` |
| 改寫 HAL 為 BSP | Copilot | `Rewrite all STM32 HAL calls in this project to Nuvoton BSP. USART1→UART0, SPI1→SPI0, I2C1→I2C0, TIM2→TIMER0, ADC1→EADC, PA5→PC9.` |
| 產生測試程式 | Copilot | `Generate a standalone test function for each peripheral: UART0 hello, LED blink, TIMER0 toggle, SPI0 JEDEC ID, I2C0 WHO_AM_I, EADC ch0 read.` |
| 查詢暫存器 | NuTRM | `/M480 Show the SPI_CTL register bit fields` |
| 查詢腳位選項 | NuTRM | `/M480 Which pins support UART0 TX?` |
| 尋找時序問題 | Copilot | `Search this project for hard-coded delays, clock frequency constants, and timer prescaler values that assume 72 MHz. List what needs to change for 192 MHz.` |
| 查詢時脈架構 | NuTRM | `/M480 What are the clock sources for UART0, SPI0, TIMER0, and EADC? Which use PCLK0 vs PCLK1?` |
| 比較晶片 | NuTRM | `Compare UART features of M460 vs M480` |

---

## 不只適用於 STM32 — 適用於任何競品

無論您從哪個競品 MCU 移植，相同的方法都適用。唯一的差別是步驟 1 的提示詞用語：

| 您目前的 MCU | 步驟 1 提示詞調整 |
|-------------|------------------|
| **Renesas RA** | `Analyze this Renesas RA6M4 project. Extract all r_sce, r_uart, r_spi driver usage...` |
| **NXP LPC** | `Analyze this LPC5500 project. Extract all MCUXpresso SDK driver calls...` |
| **TI MSP432** | `Analyze this MSP432 project. Extract all DriverLib peripheral usage...` |
| **Microchip SAM** | `Analyze this SAMD51 project. Extract all ASF/Harmony driver calls...` |

步驟 2–5（NuTRM + NuCodeGen + Copilot）完全相同 — 它們作用在 Nuvoton 端，無論您從哪個平台移植，都不需改變。
