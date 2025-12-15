---
title: 'Phase 05: Circuit Design (Schematic)'
project: FIRM-LOCK
phase: 05
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Hardware
  - Schematic
  - KiCad
status: DRAFT
dependencies: "[[04_BOM.md]]"
next: "[[06_Bootloader_Design.md]]"
---

# Phase 05: Circuit Design (Schematic)

> **Objective:** To create a complete and detailed electronic schematic for the FIRM-LOCK device. This document details the specific connections between all components, calculates necessary passive component values, and lays out the power distribution network, serving as the definitive blueprint for the physical hardware.
> **Duration:** 3 Days
> **Deliverables:**
> - A detailed power budget and thermal analysis.
> - A complete pin mapping for the MCU.
> - Section-by-section breakdown of the schematic design.
> - A pre-layout schematic review checklist and PCB guidelines.

---

## Table of Contents
1. [[#1. Design Requirements]]
2. [[#2. System Block & Power Tree Diagrams]]
3. [[#3. MCU Pin Mapping]]
4. [[#4. Schematic Sections]]
5. [[#5. Schematic Review Checklist]]
6. [[#6. PCB Layout Guidelines (Preview)]]

---

## 1. Design Requirements

### 1.1 Power Budget Calculation

A detailed power budget is critical for battery life estimation and component selection for the power supply. The budget is based on datasheet values for the components selected in [[04_BOM.md|Phase 04]].

| Operating Mode      | Component         | Voltage | Current (Typical) | Power (Typical) | Notes                                     |
|:--------------------|:------------------|:--------|:------------------|:----------------|:------------------------------------------|
| **Deep Sleep**      | **TOTAL**         | **3.3V**| **~9 µA**         | **~30 µW**      | **Target Battery Life > 1 year**          |
|                     | STM32U5 (Stop 2)  | 3.3V    | 8 µA              | 26.4 µW         | With RTC running, RAM retention           |
|                     | ATECC608A (Sleep) | 3.3V    | 0.1 µA            | 0.3 µW          |                                           |
|                     | LDO (Quiescent)   | 3.3V    | 0.9 µA            | 3.0 µW          | Quiescent current of LP5907               |
| **Active Idle**     | **TOTAL**         | **3.3V**| **~10 mA**        | **~33 mW**      | MCU active, peripherals idle              |
|                     | STM32U5 (Run @160MHz)| 3.3V  | 9.8 mA            | 32.3 mW         | Code executing from Flash                 |
|                     | Others            | 3.3V    | ~0.2 mA           | 0.7 mW          |                                           |
| **LoRa TX (Peak)**  | **TOTAL**         | **3.3V**| **~130 mA**       | **~429 mW**     | **Defines peak load for LDO**             |
|                     | RFM95W (+20 dBm TX)| 3.3V    | 120 mA            | 396 mW          | This peak is transient (<1 sec)           |
|                     | STM32U5 (Active)  | 3.3V    | 10 mA             | 33 mW           |                                           |
| **LoRa RX**         | **TOTAL**         | **3.3V**| **~25 mA**        | **~83 mW**      |                                           |
|                     | RFM95W (RX Mode)  | 3.3V    | 15 mA             | 49.5 mW         |                                           |
|                     | STM32U5 (Active)  | 3.3V    | 10 mA             | 33 mW           |                                           |

**Conclusion:** The power supply must handle a peak transient current of at least 150mA, while maintaining a very low quiescent current to maximize sleep-mode battery life. The selected LP5907 LDO, with its 250mA output capability and <1µA quiescent current, is a suitable choice.

### 1.2 Thermal Analysis

*   **Primary Heat Source:** The LoRa module (RFM95W) during transmission (~400 mW).
*   **Calculation:**
    *   Ambient Temperature (Target): -20°C to +60°C.
    *   Package Thermal Resistance (Module on PCB): ~40°C/W.
    *   Temperature Rise = 0.4W * 40°C/W = 16°C.
*   **Result:** At a 60°C ambient temperature, the LoRa module's case temperature could reach 60 + 16 = 76°C. This is well within its operating limit of 85°C. No heatsink is required.

---

## 2. System Block & Power Tree Diagrams

### Power Tree
This diagram shows how power is distributed from the input sources to the various components.
```
        ┌───────────┐                ┌───────────┐
        │ USB-C (5V)│                │ LiPo (3.7V)│
        └─────┬─────┘                └─────┬─────┘
              │                            │
              │ P-MOSFET Reverse Polarity  │
              │ Protection                 │
              └───────────┬────────────────┘
                          │
                          ▼
                    ┌───────────┐
                    │ LDO Input │
                    └─────┬─────┘
                          │
                          ▼
      ┌───────────────────────────────────┐
      │ LP5907 3.3V LDO Regulator         │
      │ (U4)                              │
      └─────────────────┬─────────────────┘
                        │ 3.3V Rail
    ┌───────────────────┼───────────────────┬───────────────────┐
    │                   │                   │                   │
    ▼                   ▼                   ▼                   ▼
┌───────────┐       ┌───────────┐       ┌───────────┐       ┌───────────┐
│ MCU       │       │ LoRa      │       │ Secure    │       │ Pull-ups, │
│ (U1)      │       │ Module(U3)│       │ Element(U2)       │ LEDs, etc.│
└───────────┘       └───────────┘       └───────────┘       └───────────┘
```

---

## 3. MCU Pin Mapping

This table defines the function of each critical MCU pin, serving as a reference for both schematic design and firmware development.

| Pin # | Pin Name | Function         | Peripheral       | Connection        | Notes                               |
|:------|:---------|:-----------------|:-----------------|:------------------|:------------------------------------|
| 55    | PB7      | I2C Data         | I2C1_SDA         | U2 (ATECC608A)    | 4.7kΩ pull-up to 3.3V               |
| 54    | PB6      | I2C Clock        | I2C1_SCL         | U2 (ATECC608A)    | 4.7kΩ pull-up to 3.3V               |
| 33    | PA5      | SPI Clock        | SPI1_SCK         | U3 (RFM95W)       |                                     |
| 34    | PA6      | SPI MISO         | SPI1_MISO        | U3 (RFM95W)       |                                     |
| 35    | PA7      | SPI MOSI         | SPI1_MOSI        | U3 (RFM95W)       |                                     |
| 32    | PA4      | SPI Chip Select  | GPIO_Output      | U3 (RFM95W) - NSS | Active low chip select              |
| 18    | PA0      | LoRa IRQ 0       | GPIO_Input_IRQ   | U3 (RFM95W) - DIO0| Interrupt on `TxDone` or `RxDone`     |
| 19    | PA1      | LoRa IRQ 1       | GPIO_Input_IRQ   | U3 (RFM95W) - DIO1| Interrupt on `RxTimeout`              |
| 20    | PA2      | Debug UART TX    | USART1_TX        | J1 (USB-C) -> VCP | Virtual COM Port via USB-C          |
| 21    | PA3      | Debug UART RX    | USART1_RX        | J1 (USB-C) -> VCP |                                     |
| 65    | PA13     | Debug Clock      | SWD_SWDCLK       | Debug Header      | Standard ARM SWD interface          |
| 62    | PA14     | Debug I/O        | SWD_SWDIO        | Debug Header      |                                     |
| 14    | PC13     | Status LED       | GPIO_Output      | LED1              | Active low output                   |
| 7     | PH3      | Boot0            | BOOT0            | GND               | Pulled to GND for boot from Flash   |

---

## 4. Schematic Sections

This section breaks down the schematic into logical blocks. (Note: These are descriptions, not a graphical schematic).

### 4.1 Power Supply
*   **Input:** A USB-C connector (J1) provides 5V for charging and operation. A standard 2-pin JST connector is provided for a 3.7V single-cell LiPo battery.
*   **Input Protection:** A P-channel MOSFET is used for reverse polarity protection on the battery input, which is more efficient than a simple diode.
*   **Regulation:** The LP5907 LDO (U4) is used to regulate the input voltage down to a clean 3.3V.
    *   **Decoupling:** A 10µF ceramic capacitor is placed at the LDO input, and a 10µF ceramic capacitor is placed at the output, as recommended by the datasheet for stability.
    *   **Bulk Capacitance:** A large 470µF electrolytic capacitor is placed physically close to the LoRa module's power input to handle the high transient current draw during transmission and prevent voltage sag.

### 4.2 MCU Core
*   **MCU:** The STM32U585AIIx (U1) is the heart of the system.
*   **Decoupling:** A set of 100nF ceramic capacitors are placed as close as possible to each VDD/VSS pair on the MCU to filter high-frequency noise. An additional 10µF capacitor is placed near the main power domain.
*   **Oscillator:** A 32 MHz crystal (Y1) is connected to the HSE (High-Speed External) pins. Two 12pF load capacitors are used, calculated based on the crystal's load capacitance and estimated PCB stray capacitance.
*   **Reset Circuit:** A simple RC circuit (10kΩ resistor, 100nF capacitor) is connected to the NRST pin to ensure a clean power-on reset.
*   **Boot Mode:** The BOOT0 pin is pulled directly to ground via a 10kΩ resistor, ensuring the MCU always boots from the main flash memory.

### 4.3 Secure Element Interface
*   **Connection:** The ATECC608A (U2) is connected to the MCU's I2C1 peripheral.
*   **Pull-ups:** 4.7kΩ pull-up resistors are placed on the SDA and SCL lines. The value is a standard choice for 400kHz I2C operation, providing a good balance between signal rise time and power consumption.
*   **ESD Protection:** A small TVS diode array (e.g., TPD2E001) is placed on the I2C lines near the debug header, in case these lines are exposed on a programming jig.
*   **Tamper Detection (Optional):** Two spare GPIO pins are routed to a loop of wire around the enclosure's perimeter. The firmware can detect if this loop is broken, indicating physical tampering.

### 4.4 LoRa Module
*   **Connection:** The RFM95W (U3) is connected to the MCU's SPI1 peripheral.
*   **DIO Pins:** The DIO0 and DIO1 pins are connected to interrupt-capable GPIOs on the MCU. This allows the MCU to sleep and be woken up by the radio upon packet reception or transmission completion, which is critical for low-power operation.
*   **Antenna:** The RF_OUT pin is routed via a 50Ω controlled-impedance trace to an edge-launch SMA connector (J2). A pi-network (C-L-C) is included in the layout but initially populated with 0-ohm resistors; this allows for antenna tuning if needed during RF testing.

### 4.5 Debug & Programming
*   **SWD:** A standard 10-pin ARM SWD header is provided for programming and debugging. This header will be depopulated on production units to mitigate T02.
*   **UART:** The MCU's USART1 is routed to the USB-C connector's data lines, which can be enumerated as a Virtual COM Port (VCP) for easy logging and debug output.
*   **LEDs:** A single status LED is connected to a GPIO pin for simple visual feedback (e.g., blinking error codes).

---

## 5. Schematic Review Checklist

This checklist will be used to formally review the schematic before proceeding to PCB layout.
- [ ] **Power:** Are all ICs properly decoupled with capacitors placed close to their power pins?
- [ ] **Power:** Is the bulk capacitance sufficient for the LoRa module's peak current?
- [ ] **MCU:** Are all unused pins configured correctly (e.g., tied to ground, set as analog inputs) to prevent floating inputs?
- [ ] **MCU:** Is the BOOT0 pin correctly configured for the desired boot mode?
- [ ] **Interfaces:** Are pull-up/pull-down resistors present on all open-drain or tri-state lines (I2C, NRST)?
- [ ] **Connectors:** Is the pinout of all connectors (USB, SWD) correct and standard?
- [ ] **Passives:** Have the tolerance and voltage rating of all passive components been checked?
- [ ] **Test Points:** Are test points placed on critical signals (e.g., 3.3V rail, I2C bus, key GPIOs) to facilitate debugging?
- [ ] **Silkscreen:** Are all components clearly labeled with their reference designators? Is pin 1 marked on all ICs?

---

## 6. PCB Layout Guidelines (Preview)

The schematic is only half the battle. Proper physical layout is critical, especially for RF performance.
*   **Layer Stackup:** A 4-layer stackup is strongly recommended:
    1.  **Top:** Signal + Components
    2.  **Layer 2:** Solid Ground Plane
    3.  **Layer 3:** Power Plane (3.3V)
    4.  **Bottom:** Signal
*   **RF Layout:**
    *   The RFM95W module and its antenna path must be placed as far away as possible from the MCU and other digital circuitry to prevent noise coupling.
    *   The trace from the module's RF_OUT pin to the SMA connector must be a 50Ω controlled-impedance microstrip.
    *   A "keepout" area with no components or ground plane should be placed directly under the antenna trace.
*   **Power Distribution:** Use wide traces or polygons for power routing. Avoid long, thin traces for power. Place decoupling capacitors as physically close to the IC power pins as possible.
*   **Crystal Oscillator:** The crystal and its load capacitors must be placed immediately next to the MCU's oscillator pins. The traces should be as short as possible.

---

**Next Phase:** [[06_Bootloader_Design.md|Phase 06: Bootloader Design]]
