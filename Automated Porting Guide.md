<!-- markdownlint-disable MD033 -->
# Using NuTRM and NuCodeGen to Port Your STM32F103 Code to NuMicro M487

## Introduction

This document shows you **how to use two VS Code extensions — NuTRM and NuCodeGen — as AI-powered assistants** to speed up each step of that porting process.

Instead of manually searching datasheets, cross-referencing registers, and writing initialization code from scratch, you can let these tools do the heavy lifting while you focus on your application logic.

| Extension | How It Helps You | Link |
|-----------|-----------------|------|
| **NuTRM** | Ask questions about Nuvoton chip peripherals, registers, clocks, and pinouts — get instant answers with TRM citations | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-trm-chatbot) |
| **NuCodeGen** | Visually configure your target Nuvoton chip and generate ready-to-use initialization C code | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-nucodegen) |

### What You'll Need

- Your existing STM32F103 (or other competitor) source code
- The competitor chip part number (e.g., STM32F103)
- A target Nuvoton NuMicro chip (e.g., M487)

---

## Setup

1. Install **VS Code** 1.110 or higher.
2. Install and sign in to **GitHub Copilot**.
3. Install the **NuTRM** extension from the [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-trm-chatbot).
4. Install the **NuCodeGen** extension from the [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-nucodegen).
5. Open your STM32F103 project in VS Code.

---

## How to Use the Tools — Step by Step

Below, we show you exactly how NuTRM and NuCodeGen assist at each step.

### Step 1 — Understand Your STM32F103 Code (Copilot Assists)

**Porting Guide reference:** *Step 1 — Analyze the Competitor Project*

Open your STM32F103 project in VS Code and ask **Copilot Chat** to analyze it for you:

**Prompt:**

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

Copilot scans your `.c` / `.h` files and returns a summary like this:

#### Example Output

```markdown
| Peripheral | Instance | Config | Pins | IRQ |
|-----------|----------|--------|------|-----|
| UART | USART1 | 115200, 8N1 | PA9 (TX), PA10 (RX) | USART1_IRQHandler |
| SPI | SPI1 | Master, CPOL=0, CPHA=0, 10 MHz | PA5 (SCK), PA6 (MISO), PA7 (MOSI) | — |
| I2C | I2C1 | Master, 400 kHz | PB6 (SCL), PB7 (SDA) | I2C1_EV_IRQHandler |
| Timer | TIM2 | 1 ms periodic | — | TIM2_IRQHandler |
| ADC | ADC1 | Ch0, Ch1, 12-bit, single conv | PA0, PA1 | — |
| GPIO | GPIOA | PA5 output (LED) | PA5 | — |
| Clock | — | HSE 8 MHz → PLL → 72 MHz SYSCLK | — | — |
```

This gives you the complete list of peripherals to migrate — without reading every source file yourself.

---

### Step 2 — Find the Matching NuMicro Peripherals (NuTRM Assists)

**Porting Guide reference:** *Step 2 — Find the Matching Nuvoton Peripheral Features*

Now take the peripheral list from Step 1 and ask **NuTRM** to find the Nuvoton equivalents. Open the Copilot Chat Panel, select the **NuTRM** agent, and use the `/{chip}` slash command:

**Prompt:**

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

NuTRM searches the M487 Technical Reference Manual and returns answers with **clickable TRM citations** — you can click any link to jump directly to the official documentation.

#### Example Result — Peripheral Mapping Table

Combine Copilot's analysis (Step 1) with NuTRM's answers to build your mapping:

| Function | STM32F103 | M487 (from NuTRM) | Notes |
|----------|-----------|-------------------|-------|
| Debug UART | USART1 (PA9/PA10) | UART0 (PB12 TX / PB13 RX) | Both support 115200 8N1 |
| SPI Flash | SPI1 Master (PA5–PA7) | SPI0 Master (PA0–PA3) | M487 SPI0 supports up to 50 MHz |
| I2C Sensor | I2C1 (PB6/PB7) | I2C0 (PA4 SDA / PA5 SCL) | 400 kHz fast mode supported |
| System Tick | TIM2, 1 ms | TIMER0, periodic mode | Use TIMER_Open() with 1000 Hz |
| Analog Input | ADC1 Ch0/Ch1 (PA0/PA1) | EADC Ch0/Ch1 (PB0/PB1) | M487 uses Enhanced ADC (EADC) |
| LED | GPIOA PA5 | GPIOC PC9 | Direct GPIO push-pull output |
| System Clock | HSE 8 MHz → PLL → 72 MHz | HXT 12 MHz → PLL → 192 MHz | Adjust timing constants |

> **NuTRM advantage:** Every answer includes a direct link to the TRM section, so you don't need to search the PDF manually.

---

### Step 3 — Generate Initialization Code (NuCodeGen Assists)

**Porting Guide reference:** *Step 3 — Generate Initialization Code*

With the peripheral mapping from Step 2, use **NuCodeGen** to generate all the Nuvoton initialization code — no need to write `SYS_Init()`, clock setup, or MFP pin configuration by hand.

1. Open **NuCodeGen** in VS Code (Command Palette → `NuCodeGen`).
2. Select your target chip (e.g., **M487JIDAE**).
3. Configure each peripheral to match the mapping table from Step 2:

| NuCodeGen Setting | Value |
|-------------------|-------|
| Clock Source | HXT 12 MHz |
| PLL Output | 192 MHz |
| UART0 | 115200 baud, 8N1 |
| SPI0 | Master, CPOL=0 CPHA=0, 10 MHz |
| I2C0 | 400 kHz |
| TIMER0 | Periodic, 1 ms |
| EADC | Ch0, Ch1 enabled |
| PC.9 | GPIO Output |

4. Click **Generate** → NuCodeGen produces ready-to-use C files:
   - /clk clock tree configuration
   - /pin with MFP (multi-function pin) setup
   - /UART Peripheral-specific init functions

> **NuCodeGen advantage:** You get correct API-level init code without reading the BSP yourself. The generated code handles clock gating, pin MFP selection, and peripheral configuration in the right order.

---

### Step 4 — Map STM32 HAL Calls to Nuvoton BSP (Copilot Assists)

**Porting Guide reference:** *Step 4 — Port the Application Logic*

Now you need to replace STM32 HAL function calls with Nuvoton BSP equivalents. Ask **Copilot** to generate the mapping:

**Prompt:**

```text
I'm porting this STM32F103 project to Nuvoton M487.
Here is the peripheral mapping:
- USART1 → UART0
- SPI1 → SPI0
- I2C1 → I2C0
- TIM2 → TIMER0
- ADC1 → EADC
- GPIOA PA5 → GPIOC PC9

Generate:
1. A mapping table of STM32 HAL calls → Nuvoton BSP calls
2. A wrapper header file that #defines STM32 HAL names to Nuvoton equivalents
3. Updated interrupt handler function names
```

#### Example Copilot-Generated Wrapper

```c
/* porting_layer.h — STM32 HAL → Nuvoton BSP compatibility shim */

#ifndef PORTING_LAYER_H
#define PORTING_LAYER_H

#include "NuMicro.h"

/* UART */
#define PORTING_UART_PORT       UART0
#define PORTING_UART_BAUDRATE   115200

/* SPI */
#define PORTING_SPI_PORT        SPI0
#define PORTING_SPI_SPEED       10000000

/* I2C */
#define PORTING_I2C_PORT        I2C0

/* GPIO — LED */
#define LED_ON()    GPIO_SET_PIN(PC, BIT9)
#define LED_OFF()   GPIO_CLR_PIN(PC, BIT9)
#define LED_TOGGLE() GPIO_TOGGLE(PC, BIT9)

#endif /* PORTING_LAYER_H */
```

---

### Step 5 — Verify Register Details and Fill Gaps (NuTRM Assists)

**Porting Guide reference:** *Step 5 — Verify Register-Level Details*

During porting, you'll inevitably hit cases where you need to check exact register bit fields, timing constraints, or pin alternatives. This is where NuTRM shines — ask it directly instead of searching through a 1000+ page PDF:

```text
/M480 Show the SPI_CTL register bit fields for SPI0

/M480 What is the EADC module's sample-and-hold time configuration?

/M480 Show the multi-function pin register for UART0 TX on PB12

/M480 What is the interrupt vector number for UART0?

/M480 What are the clock divider options for TIMER0?
```

Every answer includes **clickable TRM citations** — you can verify the information by clicking the link to jump to the exact section in the official TRM document.

> **When to use NuTRM during porting:**
> - A peripheral doesn't behave as expected — check the register configuration
> - You need to pick an alternative pin — ask about MFP options
> - You're unsure about clock divider values — ask for the exact register fields
> - You need the interrupt vector name or number — NuTRM knows it


---

## Summary: Which Tool Helps Where

```
┌───────────────────────────────────────┐
│   Your STM32F103 Source Code           │
│   + Chip Part Number                  │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 1: Copilot analyzes your code   │  ← understands your STM32F103 project
│  → "what peripherals do I use?"       │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 2: NuTRM finds Nuvoton matches  │  ← searches M487 TRM for you
│  → "which M487 peripheral matches?"   │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 3: NuCodeGen generates init     │  ← produces ready-to-use C code
│  → clock, GPIO, peripheral setup      │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 4: Copilot maps HAL → BSP      │  ← rewrites API calls for you
│  → wrapper header + IRQ renaming      │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 5: NuTRM verifies details       │  ← confirms register-level info
│  → exact bit fields with TRM links    │
└───────────────────────────────────────┘
```

---

## Quick Prompt Reference

| Porting Step | Assistant | What to Ask |
|-------------|-----------|-------------|
| Analyze STM32 code | Copilot | `Analyze this STM32F103 project. List all peripherals, pins, clocks, IRQs, and DMA as a table.` |
| Find NuMicro equivalents | NuTRM | `/M480 I'm migrating from STM32F103. I need UART 115200, SPI Master 10 MHz, I2C 400 kHz, Timer 1 ms, ADC 2ch. Map each to M487.` |
| Generate init code | NuCodeGen | *(GUI)* Select M487 → configure peripherals → Generate |
| Map HAL to BSP | Copilot | `Map these STM32 HAL calls to Nuvoton BSP equivalents: HAL_UART_Transmit, HAL_SPI_TransmitReceive, ...` |
| Check registers | NuTRM | `/M480 Show the SPI_CTL register bit fields` |
| Check pin options | NuTRM | `/M480 Which pins support UART0 TX?` |
| Compare chips | NuTRM | `Compare UART features of M460 vs M480` |


---

## Not Just STM32 — Works for Any Competitor

The same approach works regardless of which competitor MCU you're migrating from. The only difference is the prompt wording in Step 1:

| Your Current MCU | Step 1 Prompt Adjustment |
|------------------|---------------------------|
| **Renesas RA** | `Analyze this Renesas RA6M4 project. Extract all r_sce, r_uart, r_spi driver usage...` |
| **NXP LPC** | `Analyze this LPC5500 project. Extract all MCUXpresso SDK driver calls...` |
| **TI MSP432** | `Analyze this MSP432 project. Extract all DriverLib peripheral usage...` |
| **Microchip SAM** | `Analyze this SAMD51 project. Extract all ASF/Harmony driver calls...` |

Steps 2–5 (NuTRM + NuCodeGen + Copilot) remain exactly the same — they work on the Nuvoton side, which doesn't change regardless of where you're migrating from.
