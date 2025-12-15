---
title: 'Phase 07: Hardware Prototype (Breadboard)'
project: FIRM-LOCK
phase: 07
date: 2025-12-15
tags:
  - 'FIRM-LOCK'
  - 'Hardware'
  - 'Prototype'
  - 'Bringup'
status: DRAFT
dependencies: "[[06_Bootloader_Design.md]]"
next: "[[08_Firmware_Core.md]]"
---

# Phase 07: Hardware Prototype (Breadboard)

> **Objective:** To assemble and validate the core hardware design on a breadboard. This phase aims to de-risk the final PCB by proving that the chosen components can communicate, meet power targets, and support the initial firmware development.
> **Duration:** 2 Days
> **Deliverables:**
> - A fully assembled and wired breadboard prototype.
> - A completed bring-up procedure, validating each core component.
> - Initial power consumption measurements.
> - A documented set of common issues and their resolutions.

---

## Table of Contents
1. [[#1. Prototype Goals]]
2. [[#2. Bill of Materials (Prototype)]]
3. [[#3. Assembly and Wiring Guide]]
4. [[#4. Systematic Bring-Up Procedure]]
5. [[#5. Common Issues & Debugging]]
6. [[#6. Measurements & Validation]]
7. [[#7. Photo Documentation Checklist]]

---

## 1. Prototype Goals

The primary goal of this phase is to move from theory to practice. Before committing to the time and expense of a custom PCB, we must validate our core assumptions on a flexible, debug-friendly platform.
*   **Validate Circuit Design:** Confirm that the schematic from [[05_Circuit_Design.md|Phase 05]] is viable and that the components interact as expected.
*   **Test Communication Buses:** Verify that the MCU can communicate reliably with the Secure Element (I2C) and the LoRa module (SPI).
*   **Measure Power Consumption:** Take initial measurements of the prototype's power draw in different states (sleep, idle, TX) to validate the power budget.
*   **Enable Firmware Development:** Provide a stable hardware platform for developing and debugging the bootloader and core firmware described in [[06_Bootloader_Design.md|Phase 06]] and [[08_Firmware_Core.md|Phase 08]].

---

## 2. Bill of Materials (Prototype)

This BOM uses development boards and breakout modules for ease of assembly.

| Item                      | Part/Model                | Qty | Purpose                                       |
|:--------------------------|:--------------------------|:----|:----------------------------------------------|
| **MCU Dev Board**         | ST NUCLEO-U575ZI-Q        | 1   | The core MCU on an easy-to-use board.         |
| **Secure Element**        | Adafruit ATECC608A Breakout| 1   | Provides easy access to the SE pins.          |
| **LoRa Module**           | Adafruit RFM95W Breakout  | 1   | LoRa module with 3.3V regulator and level shifting. |
| **Breadboards**           | Full-Size Breadboard      | 2   | For assembling the circuit.                   |
| **Jumper Wires**          | M-M and M-F Assortment    | 1   | To connect the components.                    |
| **USB-C Cable**           | -                         | 1   | To power and program the Nucleo board.        |
| **LoRa Antenna**          | 915 MHz SMA Antenna       | 1   | **CRITICAL:** For operating the LoRa radio.   |
| **Passive Components**    | 4.7kΩ Resistors           | 2   | For I2C pull-ups.                             |
| **Test Equipment**        | Digital Multimeter        | 1   | For checking voltages and continuity.         |
| **Test Equipment**        | Logic Analyzer (optional) | 1   | For debugging I2C and SPI communication.      |

---

## 3. Assembly and Wiring Guide

Follow these instructions carefully. Incorrect wiring can damage components.

### Breadboard Layout Diagram
```
────────────────────────────────────────────────────────────────────────────────
| Power Rails (+5V, +3.3V, GND)                                                |
────────────────────────────────────────────────────────────────────────────────
|                                                                              |
|  ┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐  |
|  │ NUCLEO-U575ZI-Q  │       │ ATECC608A B/O    │       │ RFM95W B/O       │  |
|  │ (Spans both      │       │                  │       │                  │  |
|  │ breadboards)     │       │                  │       │                  │  |
|  └──────────────────┘       └──────────────────┘       └──────────────────┘  |
|                                                                              |
|                                                                              |
────────────────────────────────────────────────────────────────────────────────
```

### Step 1: Power Distribution
1.  Place the Nucleo board spanning the two breadboards.
2.  Connect the `+5V_STLINK` pin from the Nucleo board to the red power rail of one breadboard. This will be our 5V rail.
3.  Connect the `3.3V` pin from the Nucleo board to the red power rail of the *other* breadboard. This is our main 3.3V rail.
4.  Connect a `GND` pin from the Nucleo board to the blue power rail on both breadboards. These are our ground rails.
5.  **Verification:** Power the Nucleo board via USB. Use a multimeter to confirm the 5V and 3.3V rails are correct.

### Step 2: Secure Element (ATECC608A) Wiring
1.  Place the ATECC608A breakout board on the breadboard.
2.  Connect `VIN` to the **3.3V** rail.
3.  Connect `GND` to the **GND** rail.
4.  Connect `SDA` to Nucleo pin **PB7**.
5.  Connect `SCL` to Nucleo pin **PB6**.
6.  Place a 4.7kΩ resistor from the `SDA` line to the **3.3V** rail.
7.  Place a 4.7kΩ resistor from the `SCL` line to the **3.3V** rail.

### Step 3: LoRa Module (RFM95W) Wiring
1.  **CRITICAL:** Screw the 915 MHz antenna onto the SMA connector of the RFM95W breakout board. **NEVER power the module without an antenna attached.**
2.  Place the breakout board on the breadboard.
3.  Connect `Vin` to the **3.3V** rail.
4.  Connect `GND` to the **GND** rail.
5.  Connect `SCK` to Nucleo pin **PA5**.
6.  Connect `MISO` to Nucleo pin **PA6**.
7.  Connect `MOSI` to Nucleo pin **PA7**.
8.  Connect `CS` (Chip Select) to Nucleo pin **PA4**.
9.  Connect `G0` (DIO0) to Nucleo pin **PA0**.

### Wiring Checklist Table
| From Pin      | To Pin          | Wire Color | Status | Notes                       |
|:--------------|:----------------|:-----------|:-------|:----------------------------|
| Nucleo 3.3V   | 3.3V Rail       | Red        | [ ]    | Main power rail             |
| Nucleo GND    | GND Rail        | Black      | [ ]    | Ground rail                 |
| ATECC608A VIN | 3.3V Rail       | Red        | [ ]    |                             |
| ATECC608A GND | GND Rail        | Black      | [ ]    |                             |
| ATECC608A SDA | Nucleo PB7      | Blue       | [ ]    | I2C Data + 4.7kΩ pull-up    |
| ATECC608A SCL | Nucleo PB6      | Green      | [ ]    | I2C Clock + 4.7kΩ pull-up   |
| RFM95W Vin    | 3.3V Rail       | Red        | [ ]    |                             |
| RFM95W GND    | GND Rail        | Black      | [ ]    |                             |
| RFM95W SCK    | Nucleo PA5      | Yellow     | [ ]    | SPI Clock                   |
| RFM95W MISO   | Nucleo PA6      | Orange     | [ ]    | SPI Data Out                |
| RFM95W MOSI   | Nucleo PA7      | Brown      | [ ]    | SPI Data In                 |
| RFM95W CS     | Nucleo PA4      | Purple     | [ ]    | SPI Chip Select             |
| RFM95W G0     | Nucleo PA0      | White      | [ ]    | SPI Interrupt               |

---

## 4. Systematic Bring-Up Procedure

Test each component one by one. Do not proceed to the next step until the current one passes.

### Step 1: "Blinky" Test (MCU Sanity Check)
*   **Goal:** Confirm the MCU is alive and can be programmed.
*   **Action:** Load a simple "blinky" program using STM32CubeIDE that toggles the user LED on the Nucleo board.
*   **Expected:** The green user LED blinks once per second. This confirms the toolchain and debugger are working.

### Step 2: I2C Scan (Bus Verification)
*   **Goal:** Verify the I2C bus is wired correctly and the Secure Element is responding.
*   **Action:** Flash a program that scans the I2C bus and prints the addresses of any found devices to the serial console.
    ```c
    // Example code for I2C scan
    printf("Scanning I2C bus...\n");
    for (uint8_t addr = 1; addr < 128; addr++) {
        if (HAL_I2C_IsDeviceReady(&hi2c1, (uint16_t)(addr << 1), 3, 100) == HAL_OK) {
            printf("I2C device found at address 0x%02X\n", addr);
        }
    }
    ```
*   **Expected:** The serial console prints `I2C device found at address 0x60`. This is the default address for the ATECC608A.

### Step 3: Secure Element Test
*   **Goal:** Initialize the ATECC608A and read its unique serial number.
*   **Action:** Use the Microchip CryptoAuthLib to initialize the device and read its serial number.
    ```c
    #include "cryptoauthlib.h"
    // ... (after successful atcab_init) ...
    uint8_t serial_num[ATCA_SERIAL_NUM_SIZE];
    status = atcab_read_serial_number(serial_num);
    if (status == ATCA_SUCCESS) {
        printf("ATECC608A Serial: ");
        for (int i = 0; i < ATCA_SERIAL_NUM_SIZE; i++) {
            printf("%02X", serial_num[i]);
        }
        printf("\n");
    }
    ```
*   **Expected:** A unique 9-byte serial number is printed to the console.

### Step 4: LoRa Module Test
*   **Goal:** Initialize the RFM95W module and read its version register.
*   **Action:** Use a radio library (like RadioLib) to initialize the SPI bus and communicate with the module.
    ```c
    #include <RadioLib.h>
    // RFM95 radio = new Module(NSS, DIO0, RESET, DIO1);
    int state = radio.begin();
    if (state == RADIOLIB_ERR_NONE) {
        printf("LoRa init successful!\n");
    } else {
        printf("LoRa init failed, code %d\n", state);
    }
    ```
*   **Expected:** The serial console prints `LoRa init successful!`.

---

## 5. Common Issues & Debugging

### Troubleshooting Decision Tree
```
Problem: Nothing works!
  │
  └─ Is the power LED on the Nucleo board lit?
       │
       ├─ NO: Check USB cable and connection.
       │
       └─ YES: Run the "Blinky" test.
            │
            ├─ FAIL: Check STM32CubeIDE setup, driver installation, and debug configuration.
            │
            └─ PASS: Problem is with a peripheral.
                 │
                 └─ Does the I2C scan find the ATECC608A?
                      │
                      ├─ NO:
                      │  ├─ 1. Check I2C wiring (SDA/SCL swapped?).
                      │  ├─ 2. Verify 3.3V and GND to the breakout.
                      │  └─ 3. **Are the 4.7kΩ pull-up resistors installed correctly? (Most common issue)**
                      │
                      └─ YES: Problem is with the LoRa module.
                           │
                           ├─ 1. Check all SPI wiring (SCK, MISO, MOSI, CS).
                           ├─ 2. Verify 3.3V and GND to the breakout.
                           └─ 3. **Is the antenna securely attached?**
```

---

## 6. Measurements & Validation

Once the prototype is functional, perform these measurements to validate the design from Phase 05.

*   **Tool:** A precise multimeter (e.g., µCurrent Gold) or a power analyzer.
*   **Method:** Place the multimeter in series with the 3.3V power rail supplying the breakout boards.

| Measurement         | Test Code State                               | Expected (from P05) | Actual | Pass/Fail |
|:--------------------|:----------------------------------------------|:--------------------|:-------|:----------|
| **Deep Sleep Current**| MCU in `STOP 2` mode, peripherals off.        | ~9 µA               |        |           |
| **Idle Current**      | MCU running a `while(1)` loop, peripherals on.| ~10 mA              |        |           |
| **LoRa TX Current**   | During a LoRa packet transmission at +20dBm.  | ~130 mA (peak)      |        |           |

*   **Logic Analyzer Validation:**
    *   Place probes on the I2C bus (SDA, SCL). Trigger on the ATECC608A's address (0x60) and verify that clean I2C transactions are visible.
    *   Place probes on the SPI bus (SCK, MOSI, MISO, CS). Trigger on the CS line going low and verify that SPI commands are being sent to the LoRa module.

---

## 7. Photo Documentation Checklist

- [x] Overall shot of the completed breadboard setup.
- [x] Close-up shot of the MCU-to-Secure-Element wiring.
- [x] Close-up shot of the MCU-to-LoRa-module wiring.
- [x] Screenshot of the serial console showing the successful I2C scan and SE serial number.
- [x] (Optional) Photo of the logic analyzer screen showing a clean I2C or SPI transaction.

---

**Next Phase:** [[08_Firmware_Core.md|Phase 08: Firmware Core Development]]
