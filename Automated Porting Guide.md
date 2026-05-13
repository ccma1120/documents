<!-- markdownlint-disable MD033 -->
# Using NuTRM and NuCodeGen to Port Your STM32F103 Code to NuMicro M487

## Introduction

This document shows you **how to use two VS Code extensions — NuTRM and NuCodeGen — as AI-powered assistants** to speed up each step of that porting process.

Instead of manually searching datasheets, cross-referencing registers, and writing initialization code from scratch, you can let these tools do the heavy lifting while you focus on your application logic.

| Extension | How It Helps You | Link |
|-----------|-----------------|------|
| **NuTRM** | Ask questions about Nuvoton chip peripherals, registers, clocks, and pinouts — get instant answers with TRM citations | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-trm-chatbot) |
| **NuCodeGen** | Ask the NuCodeGen agent to generate initialization C code for your target Nuvoton chip — clock, GPIO, and peripheral setup via chat prompts | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Nuvoton.nuvoton-nucodegen) |

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

With the peripheral mapping from Step 2, use the **NuCodeGen** chatbot agent to generate the Nuvoton **startup initialization code** — the one-time setup that runs before your application logic: `SYS_Init()`, clock configuration, and MFP pin assignment.

Open the Copilot Chat Panel, select the **NuCodeGen** agent, and tell it what you need:

**Prompt:**

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


> **NuCodeGen advantage:** You get correct API-level init code by describing what you need in natural language — no need to read the BSP yourself. The generated code handles clock gating, pin MFP selection, and peripheral configuration in the right order.

---

### Step 4 — Replace STM32 HAL Calls with Nuvoton BSP (Copilot + NuCodeGen Assist)

With init code generated in Step 3, now port your **application logic** — the runtime code that sends data, reads sensors, and handles interrupts.

STM32 HAL and Nuvoton BSP have **completely different APIs** — STM32 uses handle structs (`&huart1`), while Nuvoton uses peripheral pointers (`UART0`). You cannot bridge them with simple `#define` macros. Instead, use **Copilot** to rewrite the code and **NuCodeGen** to generate correct Nuvoton BSP usage examples.

#### 4a. Ask NuCodeGen for Runtime API Examples

Step 3 generated the *init* code (called once at startup). Now you need to know how to *use* each peripheral at runtime — send bytes, read sensors, handle conversions. Ask **NuCodeGen**:

**Prompt (NuCodeGen agent):**

```text
I'm using M487JIDAE. Show me how to:
- Send and receive data on UART0
- Do a SPI master transmit/receive on SPI0
- Write/read bytes as I2C master on I2C0
- Start and read EADC conversion on channel 0
- Set up a TIMER0 periodic interrupt at 1 ms
```

NuCodeGen returns ready-to-use code snippets using the Nuvoton BSP API. Keep this output visible in the chat — Copilot will use it as reference in the next step.

#### 4b. Ask Copilot to Rewrite the Source Code

With the NuCodeGen BSP examples from Step 4a still in your chat context, ask **Copilot** to replace the STM32 HAL calls in your project:

**Prompt (Copilot):**

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

#### Example — Before vs After

| STM32 HAL (before) | Nuvoton BSP (after) |
|---|---|
| `HAL_UART_Transmit(&huart1, buf, len, 1000)` | `UART_Write(UART0, buf, len)` |
| `HAL_UART_Receive_IT(&huart1, buf, len)` | `UART_Read(UART0, buf, len)` |
| `HAL_SPI_TransmitReceive(&hspi1, tx, rx, len, 1000)` | `SPI_WRITE_TX(SPI0, tx[i])` / `SPI_READ_RX(SPI0)` in a loop |
| `HAL_I2C_Master_Transmit(&hi2c1, addr, buf, len, 1000)` | `I2C_WriteByte(I2C0, addr, buf[i])` |
| `HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET)` | `GPIO_SET_PIN(PC, BIT9)` |
| `HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5)` | `GPIO_TOGGLE(PC, BIT9)` |
| `HAL_ADC_Start(&hadc1)` | `EADC_START_CONV(EADC, BIT0 \| BIT1)` |
| `HAL_ADC_GetValue(&hadc1)` | `EADC_GET_CONV_DATA(EADC, 0)` |

#### Interrupt Handler Renaming

STM32 and Nuvoton use **different IRQ function names**. The vector table calls these by exact name — using the wrong name means your handler silently never runs.

| STM32 IRQ Handler | Nuvoton M487 Equivalent |
|---|---|
| `USART1_IRQHandler()` | `UART0_IRQHandler()` |
| `SPI1_IRQHandler()` | `SPI0_IRQHandler()` |
| `I2C1_EV_IRQHandler()` | `I2C0_IRQHandler()` |
| `TIM2_IRQHandler()` | `TMR0_IRQHandler()` |
| `ADC1_2_IRQHandler()` | `EADC0_IRQHandler()` |

> **Key difference:** STM32 HAL uses handle-based APIs with timeout parameters (e.g., `HAL_UART_Transmit(&huart1, buf, len, 1000)`). Nuvoton BSP uses direct peripheral-pointer APIs without timeouts (e.g., `UART_Write(UART0, buf, len)`). Copilot can rewrite these for you, but review the result to ensure timeout/error-handling logic is preserved in your application layer.

---

### Step 5 — Verify on Hardware and Debug with NuTRM

After Copilot rewrites the API calls, you need to verify each peripheral works correctly. Since this is bare-metal MCU code that interacts with real hardware, **traditional unit tests aren't practical** — the BSP calls talk directly to peripheral registers that don't exist on your PC.

#### 5a. Test One Peripheral at a Time on the M487 Board

| Order | Peripheral | How to Verify |
|-------|-----------|---------------|
| 1 | **Clock** | Check system clock output on a pin with an oscilloscope, or verify `SystemCoreClock` value in the debugger |
| 2 | **GPIO (LED)** | LED toggles visually — simplest sanity check |
| 3 | **UART** | Send a "Hello" string and check it in a serial terminal (e.g., PuTTY, Tera Term) |
| 4 | **Timer** | Toggle a GPIO pin in the timer ISR and measure the interval with an oscilloscope or logic analyzer |
| 5 | **SPI** | Read the JEDEC ID from the external SPI flash and compare with the datasheet value |
| 6 | **I2C** | Read the WHO_AM_I register from the sensor and compare with the datasheet value |
| 7 | **ADC (EADC)** | Apply a known voltage (e.g., 1.65 V = half of 3.3 V) and check the conversion result is ~2048 (12-bit) |

> **Tip:** Test in the order above. If the clock is wrong, nothing else will work. If UART works, you can use `printf` to debug everything else.

You can ask Copilot to generate simple test routines:

**Prompt:**

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

#### 5b. When Something Doesn't Work — Ask NuTRM

When a peripheral doesn't behave as expected, use **NuTRM** to drill into register-level details instead of searching through a 1000+ page PDF:

```text
/M480 Show the SPI_CTL register bit fields for SPI0

/M480 What is the EADC module's sample-and-hold time configuration?

/M480 Show the multi-function pin register for UART0 TX on PB12

/M480 What is the interrupt vector number for UART0?

/M480 What are the clock divider options for TIMER0?
```

Every answer includes **clickable TRM citations** — click the link to jump to the exact section in the official TRM document.

> **Common debugging scenarios where NuTRM helps:**
> - UART outputs garbage → check baud rate divider register and clock source
> - SPI no response → verify MFP pin assignment and clock polarity/phase settings
> - Timer ISR never fires → confirm correct IRQ vector name and NVIC enable
> - I2C no ACK → check pull-up configuration and bus clock frequency register
> - ADC result always 0 → verify channel mapping and trigger source configuration


---

## Summary: Which Tool Helps Where

```
┌───────────────────────────────────────┐
│   Your STM32F103 Source Code          │
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
│  Step 3: NuCodeGen agent generates    │  ← ask it via chat prompts
│  → clock, GPIO, peripheral setup      │
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 4: NuCodeGen + Copilot port    │  ← NuCodeGen shows BSP usage,
│  → rewrite HAL calls + IRQ renaming   │    Copilot rewrites the code
└──────────┬────────────────────────────┘
           │
           ▼
┌───────────────────────────────────────┐
│  Step 5: Verify on HW + NuTRM debug  │  ← test peripherals, ask NuTRM
│  → fix issues with TRM register info  │
└───────────────────────────────────────┘
```

---

## Quick Prompt Reference

| Porting Step | Assistant | What to Ask |
|-------------|-----------|-------------|
| Analyze STM32 code | Copilot | `Analyze this STM32F103 project. List all peripherals, pins, clocks, IRQs, and DMA as a table.` |
| Find NuMicro equivalents | NuTRM | `/M480 I'm migrating from STM32F103. I need UART 115200, SPI Master 10 MHz, I2C 400 kHz, Timer 1 ms, ADC 2ch. Map each to M487.` |
| Generate init code | NuCodeGen | `I'm using M487JIDAE. Generate init code for UART0 115200, SPI0 Master 10 MHz, I2C0 400 kHz, TIMER0 1 ms, EADC Ch0/Ch1, PC.9 GPIO output.` |
| Get BSP usage examples | NuCodeGen | `I'm using M487JIDAE. Show me how to send/receive on UART0, SPI0 master transfer, I2C0 read/write, EADC convert ch0, TIMER0 1ms ISR.` |
| Rewrite HAL to BSP | Copilot | `Rewrite all STM32 HAL calls in this project to Nuvoton BSP. USART1→UART0, SPI1→SPI0, I2C1→I2C0, TIM2→TIMER0, ADC1→EADC, PA5→PC9.` |
| Generate test routines | Copilot | `Generate a standalone test function for each peripheral: UART0 hello, LED blink, TIMER0 toggle, SPI0 JEDEC ID, I2C0 WHO_AM_I, EADC ch0 read.` |
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
