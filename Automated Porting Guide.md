<!-- markdownlint-disable MD033 -->
# Automated Porting Guide: Competitor MCU → Nuvoton NuMicro

## Concept

When you already have **the competitor's source code** and **know the chip part number** (e.g., STM32F407, R7FA6M4), you can leverage **GitHub Copilot + NuTRM agent + NuCodeGen agent** inside VS Code to produce a porting guide to a Nuvoton NuMicro chip — with minimal manual effort.

| Extension | Role in Automation | Link |
|-----------|-------------------|------|
| **GitHub Copilot** | Analyzes competitor source code, extracts peripheral usage, generates API mapping | Built-in |
| **NuTRM** | Queries Nuvoton TRM to find matching peripherals, registers, pin assignments, and clock config | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-trm-chatbot) |
| **NuCodeGen** | Generates Nuvoton initialization code (clock, GPIO, peripherals) from a GUI configurator | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-nucodegen) |

### How It Differs from the Manual Approach

| Aspect | Manual Way | Automatic Way |
|--------|-----------|---------------|
| Code analysis | You read the competitor code yourself | Copilot reads and summarizes the code |
| Peripheral mapping | You search TRM yourself | Copilot + NuTRM batch-query all peripherals |
| Init code | You configure peripheral by hand | NuCodeGen generates from your config; Copilot helps fill gaps |
| API porting | You map HAL calls manually | Copilot generates the full mapping table |

---

## Prerequisites

1. **VS Code** 1.110+
2. **GitHub Copilot** installed and signed in
3. **NuTRM** extension installed
4. **NuCodeGen** extension installed
5. Competitor source code opened in VS Code workspace
6. Target Nuvoton chip decided (e.g., M487)

---

## Automated Workflow

### Phase 1 — Let Copilot Analyze the Competitor Code

Open the competitor project in VS Code and use **Copilot Chat** to extract all the information you need in one shot.

**Prompt (paste into Copilot Chat):**

```text
Analyze the source code in this workspace. This is a project for an STM32F407 MCU.
Extract and summarize:
1. All peripherals used (UART, SPI, I2C, Timer, PWM, ADC, DMA, GPIO, etc.)
2. For each peripheral: instance name, configuration (baud rate, clock speed, mode, etc.)
3. Pin assignments (which GPIO pins are mapped to which functions)
4. Clock configuration (system clock source, PLL, AHB/APB dividers)
5. Interrupt handlers (IRQ names and what they do)
6. DMA channels and their trigger sources

Output as a structured markdown table.
```

> **Result:** Copilot scans all `.c` / `.h` files and returns a comprehensive peripheral summary table — no manual code reading required.

#### Example Copilot Output

```markdown
| Peripheral | Instance | Config | Pins | IRQ |
|-----------|----------|--------|------|-----|
| UART | USART1 | 115200, 8N1 | PA9 (TX), PA10 (RX) | USART1_IRQHandler |
| SPI | SPI1 | Master, CPOL=0, CPHA=0, 10 MHz | PA5 (SCK), PA6 (MISO), PA7 (MOSI) | — |
| I2C | I2C1 | Master, 400 kHz | PB6 (SCL), PB7 (SDA) | I2C1_EV_IRQHandler |
| Timer | TIM2 | 1 ms periodic | — | TIM2_IRQHandler |
| ADC | ADC1 | Ch0, Ch1, 12-bit, single conv | PA0, PA1 | — |
| GPIO | GPIOA | PA5 output (LED) | PA5 | — |
| Clock | — | HSE 8 MHz → PLL → 168 MHz SYSCLK | — | — |
```

---

### Phase 2 — Query NuTRM for Nuvoton Peripheral Matching

With the competitor peripheral table from Phase 1, switch to the **NuTRM** agent and batch-query all peripherals against the target Nuvoton chip.

**Prompt (paste into NuTRM Chat):**

```text
/M480 I'm migrating from STM32F407 to M487. I need the following peripherals:
- UART: 115200 baud, 8N1 (was USART1 on STM32)
- SPI: Master mode, 10 MHz (was SPI1)
- I2C: Master, 400 kHz fast mode (was I2C1)
- Timer: 1 ms periodic interrupt (was TIM2)
- ADC: 2 channels, 12-bit (was ADC1 Ch0/Ch1)
- GPIO: 1 LED output

For each, tell me:
1. Which M487 peripheral instance to use
2. Available pins (multi-function pin selection)
3. Any feature differences or limitations compared to STM32F4
```

> **Result:** NuTRM returns the Nuvoton equivalents with **TRM citations** — direct links to the exact documentation sections.

#### Example Combined Mapping Table

After merging Copilot's analysis with NuTRM's answers:

| Function | STM32F407 | M487 (NuTRM) | Notes |
|----------|-----------|--------------|-------|
| Debug UART | USART1 (PA9/PA10) | UART0 (PB12 TX / PB13 RX) | Both support 115200 8N1 |
| SPI Flash | SPI1 Master (PA5–PA7) | SPI0 Master (PA0–PA3) | M487 SPI0 supports up to 50 MHz |
| I2C Sensor | I2C1 (PB6/PB7) | I2C0 (PA4 SDA / PA5 SCL) | 400 kHz fast mode supported |
| System Tick | TIM2, 1 ms | TIMER0, periodic mode | Use TIMER_Open() with 1000 Hz |
| Analog Input | ADC1 Ch0/Ch1 (PA0/PA1) | EADC Ch0/Ch1 (PB0/PB1) | M487 uses Enhanced ADC (EADC) |
| LED | GPIOA PA5 | GPIOC PC9 | Direct GPIO push-pull output |
| System Clock | HSE 8 MHz → PLL → 168 MHz | HXT 12 MHz → PLL → 192 MHz | Adjust timing constants |

---

### Phase 3 — Generate Initialization Code with NuCodeGen

1. Open **NuCodeGen** in VS Code (Command Palette → `NuCodeGen`).
2. Select chip: **M487JIDAE** (or your specific part number).
3. Configure peripherals based on the mapping table from Phase 2:

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

4. Click **Generate** → NuCodeGen produces:
   - `clk_conf.c` — clock tree configuration
   - `sys_init.c` — `SYS_Init()` with MFP (multi-function pin) setup
   - Peripheral-specific init functions

---

### Phase 4 — Auto-Generate the API Porting Layer with Copilot

Use Copilot to generate the HAL-to-BSP mapping code automatically.

**Prompt (paste into Copilot Chat with competitor code attached):**

```text
I'm porting this STM32F4 project to Nuvoton M487.
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

### Phase 5 — Verify and Fill Gaps with NuTRM

For any peripheral where Copilot's output needs register-level precision, query NuTRM:

```text
/M480 Show the SPI_CTL register bit fields for SPI0

/M480 What is the EADC module's sample-and-hold time configuration?

/M480 Show the multi-function pin register for UART0 TX on PB12
```

NuTRM returns the exact register layout with **clickable TRM citations**, so every detail is traceable to the official documentation.


---

## End-to-End Summary

```
┌─────────────────────────────┐
│   Competitor Source Code     │
│   + Chip Part Number         │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Phase 1: Copilot Analyzes  │  ← GitHub Copilot reads all .c/.h files
│  → Peripheral summary table │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Phase 2: NuTRM Matches     │  ← NuTRM queries Nuvoton TRM
│  → Nuvoton peripheral map   │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Phase 3: NuCodeGen Builds  │  ← NuCodeGen generates init code
│  → Clock, GPIO, periph init │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Phase 4: Copilot Ports     │  ← Copilot generates API mapping
│  → HAL wrapper + IRQ rename │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  Phase 5: NuTRM Verifies    │  ← NuTRM confirms register details
│  → Exact register citations │
└─────────────────────────────┘
└─────────────────────────────┘
```

---

## Key Prompts Cheat Sheet

| Phase | Tool | Prompt |
|-------|------|--------|
| 1 — Analyze | Copilot | `Analyze this STM32F407 project. Extract all peripherals, pins, clocks, IRQs, and DMA as a table.` |
| 2 — Match | NuTRM | `/M480 I need UART 115200, SPI Master 10 MHz, I2C 400 kHz, Timer 1 ms, ADC 2ch. Map each to M487.` |
| 3 — Generate | NuCodeGen | *(GUI)* Select M487 → configure peripherals → Generate |
| 4 — Port | Copilot | `Generate STM32 HAL → Nuvoton BSP mapping and a porting_layer.h wrapper.` |
| 5 — Verify | NuTRM | `/M480 Show {register_name} bit fields` |


---

## Tips for Other Competitor MCUs

This workflow is **not limited to STM32**. The same approach works for any competitor:

| Competitor | Phase 1 Prompt Adjustment |
|------------|--------------------------|
| **Renesas RA** | `Analyze this Renesas RA6M4 project. Extract all r_sce, r_uart, r_spi driver usage...` |
| **NXP LPC** | `Analyze this LPC5500 project. Extract all MCUXpresso SDK driver calls...` |
| **TI MSP432** | `Analyze this MSP432 project. Extract all DriverLib peripheral usage...` |
| **Microchip SAM** | `Analyze this SAMD51 project. Extract all ASF/Harmony driver calls...` |

Simply replace the chip name and HAL/SDK terminology in the prompts — the NuTRM + NuCodeGen steps remain the same.
