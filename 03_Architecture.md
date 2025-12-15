---
title: 'Phase 03: System Architecture (High-Level Design)'
project: FIRM-LOCK
phase: 03
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Architecture
  - HLD
  - Design
status: DRAFT
dependencies: "[[02_Threat_Modeling.md]]"
next: "[[04_BOM.md]]"
---

# Phase 03: System Architecture (High-Level Design)

> **Objective:** To define the high-level hardware, software, protocol, and data architecture for the FIRM-LOCK system. This document serves as the master blueprint, translating the requirements and threat mitigations from previous phases into a cohesive technical design.
> **Duration:** 3 Days
> **Deliverables:**
> - A detailed description of the core three-layer defense model.
> - Justified selection of key hardware components (MCU, SE).
> - A complete software architecture, including boot sequence and memory map.
> - Definitions for the attestation protocol, data structures, and deployment topologies.
> - Clear interface specifications for all major components.

---

## Table of Contents
1. [[#1. Architecture Overview: The Three-Layer Defense]]
2. [[#2. Hardware Architecture]]
3. [[#3. Software Architecture]]
4. [[#4. Protocol Architecture]]
5. [[#5. Data Architecture]]
6. [[#6. Deployment Architecture]]
7. [[#7. Interface Definitions]]

---

## 1. Architecture Overview: The Three-Layer Defense

The FIRM-LOCK architecture is founded on the principle of defense-in-depth. Security is not a single feature but a series of overlapping layers, ensuring that if one layer fails, another is there to contain the threat. Our model consists of three such layers, building upon each other to create a verifiable chain of trust from silicon to the cloud.

```
  ▲ Trust
  │
┌──────────────────────────────────────────────────────────────────┐
│ LAYER 3: Remote Services (Verification & Updates)                │
│ • Remote Attestation: Verifier checks device "fingerprint".      │
│ • Secure OTA Updates: Only signed, authorized firmware can be    │
│   installed.                                                     │
│ • Policy Engine: Decides if a device is healthy, quarantined, or│
│   needs an update.                                               │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ (Provides evidence, receives updates)
┌─────────────────────────────────▼────────────────────────────────┐
│ LAYER 2: Verifiable Boot & Execution (On-Device Firmware)        │
│ • Measured Boot: Bootloader measures every piece of code before  │
│   executing it, extending measurements into PCRs.                │
│ • Secure Bootloader (MCUboot): Cryptographically verifies all    │
│   firmware signatures before execution.                          │
│ • Anti-Rollback: Prevents downgrade attacks.                     │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ (Anchors trust, stores keys)
┌─────────────────────────────────▼────────────────────────────────┐
│ LAYER 1: Hardware Root of Trust (Immutable Silicon)              │
│ • Secure Element (ATECC608A): Stores device private key securely.│
│   The key never leaves the chip.                                 │
│ • MCU Security Features (STM32U5): Fused bootloader, disabled    │
│   debug ports (JTAG), and memory protection (TrustZone).         │
└──────────────────────────────────────────────────────────────────┘
```

This layered approach directly mitigates threats identified in [[02_Threat_Modeling.md|Phase 02]]. Layer 1 counters physical attacks (T02, T05), Layer 2 counters firmware tampering and downgrade attacks (T01, T03), and Layer 3 counters network-level replay and spoofing attacks (T04).

---

## 2. Hardware Architecture

The selection of hardware is the first and most critical architectural decision, as it defines the capabilities and constraints for all subsequent design.

### 2.1 MCU Selection Rationale

The main controller must provide a balance of performance, low-power operation, and, most importantly, hardware security features.

| Criterion         | **STM32U5 (Chosen)**                               | NRF52840                                       | ESP32-C3                                       |
|-------------------|----------------------------------------------------|------------------------------------------------|------------------------------------------------|
| **Security**      | **5/5:** ARM TrustZone-M, HW Crypto, Tamper Detect, OTP Fuses | **3/5:** ARM CryptoCell, but no TrustZone      | **3/5:** RISC-V Secure Boot, but less mature ecosystem |
| **Performance**   | **5/5:** Cortex-M33 @ 160 MHz                      | **3/5:** Cortex-M4 @ 64 MHz                    | **4/5:** RISC-V @ 160 MHz                      |
| **Power**         | **4/5:** Excellent low-power modes (<10 µA sleep)  | **5/5:** Best-in-class for BLE applications    | **3/5:** Higher sleep current                  |
| **Ecosystem**     | **5/5:** Mature STM32Cube ecosystem, extensive docs | **4/5:** Strong Nordic SDK, Zephyr support     | **4/5:** Strong Espressif IDF, community support |
| **Cost (1k qty)** | **3/5:** ~$8.50                                    | **4/5:** ~$6.00                                | **5/5:** ~$2.50                                |
| **TOTAL**         | **22/25**                                          | **19/25**                                      | **19/25**                                      |

**Decision:** The **STM32U5** is selected. While more expensive, its native support for ARM TrustZone provides a hardware-enforced isolation boundary between the "secure world" (our TCB) and the "normal world" (the main application). This is a powerful mitigation against vulnerabilities in the application firmware compromising the entire device, a key requirement from our threat model.

### 2.2 Secure Element Selection Rationale

The Secure Element (SE) acts as a vault for our most critical secrets.

| Criterion         | **ATECC608A (Chosen)**                             | SE050 (NXP)                                    | Software Emulation (No SE)                     |
|-------------------|----------------------------------------------------|------------------------------------------------|------------------------------------------------|
| **Security**      | **4/5:** Excellent key protection, internal countermeasures. | **5/5:** Common Criteria EAL 6+ certified.     | **1/5:** Keys stored in flash, extractable via debug. |
| **Cost (1k qty)** | **5/5:** ~$0.60                                    | **3/5:** ~$2.00                                | **5/5:** $0                                    |
| **Ease of Use**   | **4/5:** Good library support (CryptoAuthLib).     | **3/5:** More complex, full Java Card OS.      | **2/5:** Requires implementing all crypto securely. |
| **Power**         | **5/5:** Very low power consumption.               | **4/5:** Low power, but higher than ATECC.     | **N/A**                                        |

**Decision:** The **ATECC608A** is selected for its excellent balance of cost and security. It provides robust hardware protection against key extraction at a price point suitable for mass IoT deployment. While the SE050 offers higher certification, its cost and complexity are overkill for our initial target market. Software emulation is rejected outright as it fails to meet security requirement SR-02.

### 2.3 System Block Diagram

```
                    ┌──────────────────┐
   ┌───────────────▶│   Power Supply   │◀──── USB-C / Battery
   │                │  (3.3V LDO)      │
   │                └────────┬─────────┘
   │                         │ 3.3V
   │        ┌────────────────┼──────────────────────────────────┐
   │        │                │                                  │
   │   ┌────▼─────┐    ┌────▼─────┐      I2C       ┌────▼─────┐ │
   │   │  MCU     │◀───┼──────────┼───────────────▶│ Secure   │ │
   │   │ STM32U5  │    │          │                │ Element  │ │
   │   │          │    │          │                │ ATECC608A│ │
   │   └────┬─────┘    │          │                └──────────┘ │
   │        │          │          │                             │
   │        │          │      SPI │                             │
   │        │          │          │                             │
   │   ┌────▼─────┐    │          │                ┌────▼─────┐ │
   │   │ Debug    │◀───┼──────────┼───────────────▶│  LoRa    │ │
   └───│ (SWD/UART)│    │          │                │  RFM95   │ │
       └──────────┘    │          │                └────┬─────┘ │
                       │          │                     │       │
                       │          │                     │       │
                       └──────────┴─────────────────────┼───────┘
                                                        │
                                                   ┌────▼─────┐
                                                   │ Antenna  │
                                                   │ (SMA)    │
                                                   └──────────┘
```

---

## 3. Software Architecture

### 3.1 Boot Sequence State Machine

The boot process is the most critical phase for establishing trust. Any failure in this chain invalidates all subsequent security guarantees.

```
[POWER ON]
    │
    ▼
[MCU ROM Bootloader] --------------------▶ (Immutable, validates Stage 1 if fused)
    │
    ▼
[Stage 1: MCUboot]
    │
    ├─ 1. Initialize HW (Clocks, SE)
    │
    ├─ 2. Measure Self → Extend PCR[0]
    │
    ├─ 3. Verify App Signature
    │      │
    │      ├─ FAIL ─▶ [Check Boot Counter] ─▶ (>=3 failures?) ─▶ [ROLLBACK TO GOLDEN]
    │      │                                        │ (no)
    │      │                                        ▼
    │      └─ FAIL ───────────────────────────▶ [INCREMENT FAIL COUNT & REBOOT]
    │
    └─ PASS
         │
         ▼
[Stage 2: App Loader]
    │
    ├─ 4. Measure App Code → Extend PCR[1]
    │
    ├─ 5. Measure App Config → Extend PCR[2]
    │
    ├─ 6. Reset Boot Fail Counter
    │
    └─ 7. Jump to Application
         │
         ▼
[Application Execution]
    │
    ├─ Initialize Peripherals & RTOS
    │
    ├─ Run Attestation Agent Task
    │
    └─ Run Main Business Logic
```

### 3.2 Firmware Components
*   **Bootloader (MCUboot):** Responsible for signature verification, anti-rollback checks, and measured boot. It is the root of trust for the software stack.
*   **Application Firmware:** The main device logic. It runs in the "normal world" (if using TrustZone) and is considered untrusted by the bootloader until it is measured and verified.
*   **Attestation Agent:** A task within the application firmware responsible for collecting PCRs and other evidence, and interacting with the Secure Element to sign it.
*   **Update Client:** A task that handles receiving, validating, and staging new firmware updates, interacting with MCUboot to perform the final swap.

### 3.3 Memory Layout

A clearly defined memory map is essential for security and functionality. It prevents components from overwriting each other and allows the bootloader to know exactly what to measure.

```
STM32U5 2MB Flash Layout:
┌──────────────────┬──────────────────┬─────────────────────────────────────────┐
│ Start Address    │ End Address      │ Description                             │
├──────────────────┼──────────────────┼─────────────────────────────────────────┤
│ 0x0800 0000      │ 0x0801 FFFF      │ Slot 0: Primary App (128 KB)            │
│                  │                  │ Bootloader jumps here.                  │
├──────────────────┼──────────────────┼─────────────────────────────────────────┤
│ 0x0802 0000      │ 0x0803 FFFF      │ Slot 1: Staging/Update (128 KB)         │
│                  │                  │ New firmware is written here.           │
├──────────────────┼──────────────────┼─────────────────────────────────────────┤
│ 0x0804 0000      │ 0x0805 FFFF      │ Golden Image (128 KB)                   │
│                  │                  │ Factory recovery firmware.              │
├──────────────────┼──────────────────┼─────────────────────────────────────────┤
│ 0x0806 0000      │ 0x0807 FFFF      │ Bootloader (MCUboot) (128 KB)           │
│                  │                  │ Protected by flash security.            │
├──────────────────┼──────────────────┼─────────────────────────────────────────┤
│ 0x0808 0000      │ ...              │ (Reserved / Unused)                     │
├──────────────────┼──────────────────┼─────────────────────────────────────────┤
│ 0x081F F000      │ 0x081F FFFF      │ Config & Attestation Logs (4 KB)        │
└──────────────────┴──────────────────┴─────────────────────────────────────────┘
```

---

## 4. Protocol Architecture

### Attestation Flow (Challenge-Response)

This sequence ensures that evidence is fresh and bound to a specific request from a specific verifier, mitigating replay attacks (T04).

```
  Verifier                          Device (FIRM-LOCK)
     │                                │
     │ 1. Generate Nonce (e.g. 32 random bytes)
     │                                │
     │──── Challenge (Nonce) ────────▶│
     │                                │
     │                                │ 2. Receive Nonce
     │                                │
     │                                │ 3. Collect Evidence:
     │                                │    - PCR[0]: Bootloader hash
     │                                │    - PCR[1]: App hash
     │                                │    - PCR[2]: Config hash
     │                                │    - Device Certificate/Pubkey
     │                                │
     │                                │ 4. Concatenate: (Nonce || PCRs || Cert)
     │                                │
     │                                │ 5. Hash the concatenated data -> Digest
     │                                │
     │                                │ 6. Sign Digest with Device Private Key (in SE)
     │                                │
     │◀─── Evidence (PCRs, Cert, Signature) ──│
     │                                │
     │ 7. Verify Evidence:
     │    a. Re-calculate Digest from received data + original Nonce
     │    b. Verify Signature using Device Public Key
     │    c. Compare received PCRs to "golden" values
     │                                │
     │ 8. Make Policy Decision
     │                                │
     │──── Decision (PASS/FAIL) ─────▶│
     │                                │
```

---

## 5. Data Architecture

### 5.1 Measured Data (PCRs)

Platform Configuration Registers (PCRs) are the cryptographic logbook of the boot process. They are not set, but "extended". The operation is `PCR_new = SHA256(PCR_old || SHA256(data))`. This makes them tamper-evident.

*   **PCR[0]: Bootloader Measurement.** Records the hash of the MCUboot binary. Changes if the bootloader is modified.
*   **PCR[1]: Application Code Measurement.** Records the hash of the application's executable section. Changes on every new build.
*   **PCR[2]: Configuration Data Measurement.** Records the hash of a non-volatile configuration block. Changes if settings are updated.
*   **PCR[3]: Device Identity Measurement.** Records the hash of the device's public key or certificate, read from the Secure Element. Uniquely identifies the hardware.

### 5.2 Audit Logs

The device should maintain a small, circular buffer in flash to log critical security events. This is invaluable for forensics.
*   **Event Type:** (e.g., `BOOT_OK`, `SIG_FAIL`, `UPDATE_START`, `ATTEST_FAIL`)
*   **Timestamp:** (If available)
*   **Associated Data:** (e.g., failed signature's hash, source IP of request)

---

## 6. Deployment Architecture

FIRM-LOCK is designed to be flexible across different network topologies.

*   **Standalone (USB/BLE):** A technician connects directly to the device with a laptop or phone, which acts as the verifier. Use case: High-security, air-gapped environments.
*   **Star Network (LoRaWAN/Cellular):** Devices report to a central gateway, which forwards attestation evidence to a backend verifier. This is the most common IoT topology.
*   **Mesh Network (Future):** Devices could potentially attest to each other, building a localized web of trust without needing a central verifier. This is more complex and reserved for future work.

---

## 7. Interface Definitions

### 7.1 Hardware Interfaces
*   **MCU ↔ Secure Element:**
    *   **Bus:** I2C1
    *   **Pins:** `PB7` (SDA), `PB6` (SCL)
    *   **Protocol:** Standard I2C with 4.7kΩ pull-ups. ATECC608A wake-up sequence followed by CryptoAuthLib commands.
*   **MCU ↔ LoRa Module:**
    *   **Bus:** SPI1
    *   **Pins:** `PA5` (SCK), `PA6` (MISO), `PA7` (MOSI), `PA4` (NSS/CS)
    *   **Protocol:** Standard SPI protocol for reading/writing radio registers.

### 7.2 Software Interfaces (APIs)
*   **Bootloader → Application:**
    *   The bootloader passes a `boot_context` struct to the application upon successful boot.
    *   `struct boot_context { uint8_t pcr_bank[4][32]; uint32_t boot_reason; };`
*   **Application → Attestation Agent:**
    *   `int generate_attestation_evidence(const uint8_t* nonce, size_t nonce_len, uint8_t* out_evidence, size_t* out_len);`
    *   A non-blocking function that triggers the evidence generation process.

---

**Next Phase:** [[04_BOM.md|Phase 04: Component Selection (BOM)]]
