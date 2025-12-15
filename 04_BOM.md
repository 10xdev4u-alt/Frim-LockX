---
title: 'Phase 04: Component Selection & Bill of Materials (BOM)'
project: FIRM-LOCK
phase: 04
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Hardware
  - BOM
  - SupplyChain
status: DRAFT
dependencies: "[[03_Architecture.md]]"
next: "[[05_Circuit_Design.md]]"
---

# Phase 04: Component Selection & Bill of Materials (BOM)

> **Objective:** To select, justify, and document every component required to build the FIRM-LOCK device. This phase translates the system architecture into a practical, costed, and manufacturable Bill of Materials, while also analyzing supply chain risks and certification requirements.
> **Duration:** 2 Days
> **Deliverables:**
> - A detailed master BOM with part numbers, suppliers, and cost breakdowns.
> - In-depth analysis and justification for each major component selection.
> - BOM variants for different use cases (prototype, production, high-assurance).
> - A thorough supply chain risk assessment.

---

## Table of Contents
1. [[#1. Master Bill of Materials (Production, 1k qty)]]
2. [[#2. Component Deep-Dive]]
3. [[#3. BOM Variants]]
4. [[#4. Supply Chain Risk Analysis]]
5. [[#5. Certification Requirements]]

---

## 1. Master Bill of Materials (Production, 1k qty)

This master BOM represents the target for a production run of 1,000 units. Prices are estimates based on 2025 market conditions and volume discounts.

| Ref | Category      | Part Number         | Qty | Unit $ | Total $  | Supplier | Description                               |
|:----|:--------------|:--------------------|:----|:-------|:---------|:---------|:------------------------------------------|
| U1  | MCU           | STM32U585AIIx       | 1   | $8.50  | $8,500   | Mouser   | ARM Cortex-M33 w/ TrustZone, 2MB Flash    |
| U2  | Secure Elem.  | ATECC608A-MAHDA-S   | 1   | $0.58  | $580     | Digi-Key | Secure hardware key storage, I2C          |
| U3  | LoRa Module   | RFM95W-915S2        | 1   | $4.20  | $4,200   | SparkFun | HopeRF LoRa Transceiver, 915 MHz          |
| U4  | LDO Regulator | LP5907MFX-3.3/NOPB  | 1   | $0.45  | $450     | TI       | 3.3V Low-Noise, Low-IQ LDO                |
| Y1  | Crystal       | ABLS-32.000MHZ-B4   | 1   | $0.30  | $300     | Abracon  | 32 MHz MCU Crystal                        |
| C1-C10| Capacitors    | (Various)           | 10  | $0.05  | $500     | Multiple | Decoupling and timing caps (MLCC)         |
| R1-R5 | Resistors     | (Various)           | 5   | $0.02  | $100     | Multiple | Pull-ups, current limiting (Thick Film)   |
| J1  | USB Connector | USB-C-31-M-12       | 1   | $0.65  | $650     | GCT      | USB Type-C for power and debug UART       |
| J2  | SMA Connector | CON-SMA-EDGE-S      | 1   | $0.90  | $900     | Linx     | Edge-launch SMA for antenna               |
| -   | PCB           | (Custom 4-Layer)    | 1   | $1.50  | $1,500   | JLCPCB   | 4-layer FR-4 with controlled impedance    |
| **-** | **TOTAL**     |                     | **-** | **$17.65**| **$17,680**|          | **Total Per-Unit Cost (excl. assembly)**  |

### BOM Cost Breakdown (1k Units)

```
Cost Contribution per Component:
────────────────────────────────────────
MCU (STM32U5)      : ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇ (48.2%)
LoRa (RFM95W)      : ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇ (23.8%)
PCB                : ▇▇▇▇▇▇ (8.5%)
SMA Connector      : ▇▇▇▇ (5.1%)
USB-C Connector    : ▇▇▇ (3.7%)
Secure Elem (608A) : ▇▇ (3.3%)
Other (Passives etc): ▇▇▇▇ (7.4%)
```

---

## 2. Component Deep-Dive

This section expands on the justification for each critical component choice, as introduced in [[03_Architecture.md|Phase 03]].

### 2.1 Microcontroller (MCU)

*   **Part:** `STM32U585AIIx`
*   **Justification:** As established in the architecture phase, the STM32U5's hardware security features are the cornerstone of our design. **ARM TrustZone** provides hardware isolation for our TCB. The **AES and SHA hardware accelerators** offload cryptographic operations, reducing boot time and power consumption. The generous **2MB of flash** provides ample space for the bootloader, primary/staging slots, and a golden image, fulfilling requirement SR-10.
*   **Key Specs:**
    *   Core: ARM Cortex-M33 @ 160 MHz with TrustZone
    *   Flash: 2 MB (with ECC)
    *   RAM: 786 KB
    *   Security: TrustZone, True RNG, Tamper Detection, 256-bit AES, SHA-256
*   **Datasheet:** [STMicroelectronics STM32U585xx](https://www.st.com/en/microcontrollers-microprocessors/stm32u585ai.html)
*   **Alternatives & Decision Matrix:** See Phase 03. The STM32U5's security feature set is the decisive factor, justifying its higher cost.

### 2.2 Secure Element

*   **Part:** `ATECC608A-MAHDA-S`
*   **Justification:** The ATECC608A is the most cost-effective way to satisfy requirement SR-02: "Private keys... SHALL be non-exportable". It acts as a dedicated hardware vault. Its internal **monotonic counters** are essential for implementing the anti-rollback mechanism (T03). Its small footprint and I2C interface make it easy to integrate.
*   **Key Specs:**
    *   Crypto: ECDSA P-256, ECDH, SHA-256, AES-128
    *   Storage: 16 secure key slots
    *   Interface: I2C (up to 1 MHz)
    *   Security: Internal hardware protection against DPA/SPA, fault injection.
*   **Datasheet:** [Microchip ATECC608A](https://www.microchip.com/en-us/product/ATECC608A)
*   **Integration:**
    ```
    ATECC608A         STM32U5
    ──────────────────────────
    VCC     ────────  3.3V
    GND     ────────  GND
    SDA     ────────  PB7 (I2C1_SDA) with 4.7kΩ pull-up
    SCL     ────────  PB6 (I2C1_SCL) with 4.7kΩ pull-up
    ```

### 2.3 LoRa Transceiver

*   **Part:** `RFM95W-915S2`
*   **Justification:** LoRa provides the long-range, low-power communication needed for our target environments (industrial IoT, tactical edge). The RFM95W is a widely available, well-documented, and low-cost module based on Semtech's SX1276 chipset. Its maturity ensures a stable driver base (RadioLib).
*   **Key Specs:**
    *   Frequency: 915 MHz (US ISM Band)
    *   TX Power: Up to +20 dBm
    *   Sensitivity: Down to -148 dBm
    *   Interface: SPI
*   **Datasheet:** [HopeRF RFM95W](https://www.hoperf.com/modules/lora/RFM95.html)
*   **Alternatives Considered:**
    *   **Direct SX1276 Chip:** Lower cost but requires complex RF layout and impedance matching. Using a module de-risks the hardware design.
    *   **LoRaWAN-enabled Modules (e.g., Murata):** Higher cost and complexity. We are implementing a custom protocol, so a full LoRaWAN stack is unnecessary.

---

## 3. BOM Variants

The master BOM is not a one-size-fits-all solution. Different stages of development and different end markets require different components.

### 3.1 Prototype BOM
For initial development and breadboarding. Focus is on ease of use, not cost or size.
*   **MCU:** STM32U575I-EV Evaluation Board (~$80)
*   **Secure Element:** Adafruit ATECC608A Breakout Board (~$8)
*   **LoRa Module:** Adafruit RFM95W Breakout Board (~$20)
*   **Total Cost:** ~$110 per prototype setup.

### 3.2 Cost-Optimized BOM
For high-volume, less critical applications where cost is the primary driver.
*   **MCU:** ESP32-C3. Sacrifices TrustZone but retains Secure Boot and basic crypto features. (Saves ~$6.00)
*   **LoRa Module:** Direct integration of SX1276 chip. (Saves ~$1.50, but adds NRE for RF design)
*   **Target Cost:** < $10 per unit.

### 3.3 High-Assurance BOM
For defense or critical infrastructure where security guarantees are paramount.
*   **Secure Element:** NXP SE050. Common Criteria EAL 6+ certified. (Adds ~$1.50)
*   **Power Supply:** Add supervisor IC for brown-out detection and precise resets. (Adds ~$0.50)
*   **PCB:** 6-layer PCB with dedicated ground planes for better signal integrity. (Adds ~$1.00)
*   **Target:** Maximum security, cost is a secondary concern.

---

## 4. Supply Chain Risk Analysis

A brilliant design is useless if you can't build it.

### Component Selection Decision Tree
```
Is the component critical for security?
  │
  ├─ YES: Prioritize trusted vendors and clear datasheets.
  │      │
  │      └─ Is it a single-source component?
  │           │
  │           ├─ YES: (e.g., STM32U5) -> High Risk. Maintain buffer stock. Design PCB to potentially accept a pin-compatible alternative if one exists.
  │           │
  │           └─ NO: (e.g., Resistors) -> Low Risk. Specify generic parameters (e.g., 4.7kΩ 0402 1%) to allow multiple suppliers.
  │
  └─ NO: Prioritize availability, cost, and lead time.
```

### Risk Assessment

| Component | Risk Type           | Likelihood | Impact | Mitigation                                                              |
|:----------|:--------------------|:-----------|:-------|:------------------------------------------------------------------------|
| **STM32U585** | Single Source, Allocation | Medium     | **High**   | Maintain a strategic relationship with ST and distributors. Hold buffer stock. |
| **ATECC608A** | Counterfeiting      | Medium     | **Critical**| Procure ONLY from authorized distributors (Digi-Key, Mouser, Arrow). Perform authenticity checks on first articles. |
| **RFM95W**    | Lead Time Fluctuation | High       | Medium | Module is multi-sourced (HopeRF, SparkFun, Adafruit). Can substitute with other SX1276-based modules with minor driver changes. |
| **MLCCs**     | Allocation          | High       | Medium | Use common values (100nF, 1µF) that are widely available from multiple vendors (Murata, TDK, Yageo). |

---

## 5. Certification Requirements

The final product must comply with various regulations to be sold legally. The design must account for these from the beginning.

*   **FCC Part 15 (US) / CE RED (EU):** Mandatory for any device with an intentional radiator (our LoRa module). Using a pre-certified module like the RFM95W simplifies this process immensely, as we only need to certify the final product's integration, not the radio itself.
*   **RoHS/REACH:** Directives restricting hazardous substances. Most modern components are compliant, but this must be verified for every part in the BOM.
*   **ITAR (International Traffic in Arms Regulations):** If this device is marketed for specific defense applications, it may fall under ITAR. This would heavily restrict sales to non-US entities and require strict compliance. This is a business decision that impacts design and marketing.
*   **PSA Certified / SESIP:** While not mandatory, achieving a security certification like PSA Certified Level 2 would provide a strong, independent validation of our security claims and serve as a powerful marketing tool. The architecture is designed with this goal in mind.

---

**Next Phase:** [[05_Circuit_Design.md|Phase 05: Circuit Design (Schematic)]]
