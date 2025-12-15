---
title: 'Phase 08: Firmware Core Development'
project: FIRM-LOCK
phase: 08
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Firmware
  - RTOS
  - Driver
  - Attestation
status: DRAFT
dependencies: "[[07_Hardware_Prototype.md]]"
next: "[[09_Attestation_Protocol.md]]"
---

# Phase 08: Firmware Core Development

> **Objective:** To design and implement the core application firmware that runs on the hardware prototype. This includes setting up the development environment, architecting the software around a Real-Time Operating System (RTOS), and developing the drivers and agents necessary to perform remote attestation.
> **Duration:** 4 Days
> **Deliverables:**
> - A defined development toolchain and project structure.
> - An RTOS-based software architecture.
> - Drivers for interacting with the Secure Element.
> - A complete implementation of the Attestation Agent.
> - A validation plan including unit and integration tests.

---

## Table of Contents
1. [[#1. Development Environment Setup]]
2. [[#2. Core Firmware Architecture (RTOS-based)]]
3. [[#3. Secure Element Driver]]
4. [[#4. PCR Management]]
5. [[#5. Attestation Agent Implementation]]
6. [[#6. Communication Protocol (LoRa)]]
7. [[#7. Logging & Debug Strategy]]
8. [[#8. Validation Tests]]

---

## 1. Development Environment Setup

A standardized and efficient development environment is crucial for productivity and code quality.

*   **IDE:** **STM32CubeIDE**. This Eclipse-based IDE from ST provides seamless integration with code generation (CubeMX), compilation, and in-circuit debugging (via the ST-LINK on the Nucleo board).
*   **Compiler:** **GCC ARM Embedded**. The standard, open-source compiler for ARM Cortex-M processors.
*   **Debugger:** **OpenOCD / GDB**. Integrated directly into STM32CubeIDE for a one-click debugging experience.
*   **RTOS:** **FreeRTOS**. It is one of the most widely used real-time operating systems for microcontrollers, with excellent support within the STM32 ecosystem. It allows us to structure our application into independent, concurrent tasks, which is essential for managing complex operations like communication and cryptography without blocking the entire system.
*   **Project Structure:**
    ```
    FIRM-LOCK_Firmware/
    ├── Core/
    │   ├── Inc/                (Application header files)
    │   │   ├── main.h
    │   │   ├── freertos.h
    │   │   └── ...
    │   └── Src/                (Application source files)
    │       ├── main.c
    │       ├── attestation_agent.c
    │       ├── secure_element.c
    │       └── lora_comm.c
    ├── Drivers/
    │   ├── STM32U5xx_HAL_Driver/ (ST's Hardware Abstraction Layer)
    │   └── CMSIS/                (ARM Cortex Microcontroller Software Interface Standard)
    ├── Middlewares/
    │   └── Third_Party/
    │       └── FreeRTOS/
    └── Lib/
        ├── CryptoAuthLib/      (Microchip ATECC608A driver)
        └── RadioLib/           (LoRa driver)
    ```

---

## 2. Core Firmware Architecture (RTOS-based)

Using FreeRTOS, we can break our firmware into logical, prioritized tasks. This prevents a long-running operation (like LoRa communication) from interfering with a time-sensitive one.

### Task Breakdown
```
                                     ┌──────────────────┐
                                     │ FreeRTOS Kernel  │
                                     └────────┬─────────┘
             ┌────────────────────────────────┼────────────────────────────────┐
             │                                │                                │
             ▼ (High Prio)                    ▼ (Medium Prio)                  ▼ (Low Prio)
      ┌──────────────────┐           ┌──────────────────┐           ┌──────────────────┐
      │ Attestation Task │           │ Communication    │           │ Logging Task     │
      │ - Waits on queue │           │ Task             │           │ - Batches logs   │
      │ - Builds evidence│           │ - Polls LoRa radio│          │ - Writes to flash│
      │ - Signs w/ SE    │           │ - Routes packets │           │                  │
      └──────────────────┘           └──────────────────┘           └──────────────────┘
             ▲                                │
             │ (Writes to queue)              │
             └────────────────────────────────┘
```

*   **Attestation Task (High Priority):** This task remains dormant until a message is placed on its queue (e.g., an attestation request from the comm task). It then wakes up, performs the time-critical cryptographic operations, and sends the response. Its high priority ensures attestation requests are handled promptly.
*   **Communication Task (Medium Priority):** This task is responsible for all interactions with the LoRa radio. It periodically checks for incoming packets and routes them to the appropriate handler. It also manages the transmission of outgoing packets.
*   **Logging Task (Low Priority):** This task handles non-critical background activities, such as batching log messages and writing them to persistent flash storage, to avoid slowing down more important tasks.
*   **Idle Task / Watchdog:** FreeRTOS's idle task will be configured to "pet" the independent watchdog (IWDG) timer. If any higher-priority task gets stuck and starves the idle task, the watchdog will time out and reset the system, preventing a permanent freeze.

---

## 3. Secure Element Driver

This module abstracts all interactions with the ATECC608A, providing a clean API to the rest of the firmware. It is a wrapper around Microchip's CryptoAuthLib.

### Initialization
```c
#include "cryptoauthlib.h"

// Global configuration for the ATECC608A interface
ATCAIfaceCfg g_atecc608a_cfg = {
    .iface_type             = ATCA_I2C_IFACE,
    .devtype                = ATECC608A,
    .atcai2c.slave_address  = 0x60,
    .atcai2c.bus            = 1, // I2C bus 1
    .atcai2c.baud           = 400000, // 400 kHz
    .wake_delay             = 1500,
    .rx_retries             = 20
};

// Initializes the library and wakes up the chip.
ATCA_STATUS se_init(void) {
    ATCA_STATUS status = atcab_init(&g_atecc608a_cfg);
    if (status != ATCA_SUCCESS) {
        LOG_ERROR("CryptoAuthLib init failed: 0x%02X", status);
    }
    return status;
}
```

### Key Operations

*   **Generate Device Keypair (for provisioning only):**
    ```c
    // Generates a new ECC P-256 private key in the specified slot.
    // The private key is born on the chip and never leaves.
    ATCA_STATUS se_generate_device_key(uint8_t private_key_slot, uint8_t* out_public_key) {
        return atcab_genkey(private_key_slot, out_public_key);
    }
    ```
*   **Sign Data:**
    ```c
    // Signs a 32-byte digest with the private key stored in the specified slot.
    ATCA_STATUS se_sign_digest(uint8_t private_key_slot, const uint8_t* digest, uint8_t* out_signature) {
        // The ATECC608A signs the digest directly. The data must be hashed first.
        return atcab_sign(private_key_slot, digest, out_signature);
    }
    ```
*   **Verify Signature (External):**
    ```c
    // Verifies a signature against a digest using an externally provided public key.
    // Useful for verifying signatures from a server.
    ATCA_STATUS se_verify_external_signature(const uint8_t* digest, const uint8_t* signature, const uint8_t* public_key, bool* is_verified) {
        return atcab_verify_extern(digest, signature, public_key, is_verified);
    }
    ```

---

## 4. PCR Management

This module manages the Platform Configuration Registers (PCRs) as defined in the bootloader phase. The bootloader calculates the initial PCR values and passes them to the application, which stores and uses them.

*   **Storage:** For simplicity and speed, the PCRs will be stored in a global `g_pcr_bank` array in SRAM during runtime. They are recalculated from scratch on every boot, so they do not need to be stored in non-volatile memory.
*   **`pcr_extend` Function:** This function is identical to the one defined in [[06_Bootloader_Design.md|Phase 06]]. It provides a standard way to update a PCR value.
*   **`pcr_quote` Function:** This function provides a "quote" of the current PCR state, ready to be included in the attestation evidence.
    ```c
    // Copies the current values of all PCRs into an output buffer.
    // A "quote" is a snapshot of the PCR bank at a moment in time.
    void pcr_quote(uint8_t* out_pcr_snapshot, size_t snapshot_size) {
        if (snapshot_size < sizeof(g_pcr_bank)) {
            LOG_ERROR("Output buffer too small for PCR quote.");
            return;
        }
        memcpy(out_pcr_snapshot, g_pcr_bank, sizeof(g_pcr_bank));
    }
    ```

---

## 5. Attestation Agent Implementation

This is the core security logic of the application. It responds to challenges from the verifier.

### Data Structures
```c
// Evidence structure that will be sent to the verifier.
// This structure must be packed to ensure a consistent memory layout for hashing.
typedef struct __attribute__((packed)) {
    // --- Data to be signed ---
    uint32_t nonce;                 // The nonce received from the verifier.
    uint8_t  pcrs[4][32];           // Snapshot of all PCRs.
    uint8_t  device_pubkey[64];     // The device's public key, proving its identity.
    uint32_t firmware_version;      // From the image header.
    // --- End of signed data ---

    uint8_t  signature[64];         // ECDSA signature over all the fields above.
} attestation_evidence_t;
```

### Evidence Generation
```c
// Builds and signs the attestation evidence.
bool build_attestation_evidence(uint32_t nonce, attestation_evidence_t* evidence) {
    // 1. Fill in the evidence structure.
    evidence->nonce = nonce;
    evidence->firmware_version = APP_FIRMWARE_VERSION;

    // 2. Get a quote of the current PCR state.
    pcr_quote(evidence->pcrs, sizeof(evidence->pcrs));

    // 3. Read the device's public key from the Secure Element.
    atcab_read_pubkey(DEVICE_PRIVATE_KEY_SLOT, evidence->device_pubkey);

    // 4. Calculate the hash of the evidence structure (all fields except the signature).
    uint8_t digest[32];
    size_t signed_content_size = offsetof(attestation_evidence_t, signature);
    atcab_sha((uint16_t)signed_content_size, (const uint8_t*)evidence, digest);

    // 5. Offload the signing operation to the Secure Element.
    ATCA_STATUS status = se_sign_digest(DEVICE_PRIVATE_KEY_SLOT, digest, evidence->signature);

    if (status != ATCA_SUCCESS) {
        LOG_ERROR("Failed to sign evidence: 0x%02X", status);
        return false;
    }

    LOG_INFO("Attestation evidence generated successfully.");
    return true;
}
```

---

## 6. Communication Protocol (LoRa)

A simple packet structure is defined for LoRa communication.

*   **Packet Format:**
    ```c
    typedef struct __attribute__((packed)) {
        uint8_t  preamble;     // 0xFE
        uint8_t  msg_type;     // e.g., 0x01=Challenge, 0x02=Evidence
        uint16_t payload_len;
        uint8_t  payload[240]; // Max payload size for LoRa
        uint16_t crc16;
    } lora_packet_t;
    ```
*   **Sending Logic:**
    ```c
    void send_attestation_response(attestation_evidence_t* evidence) {
        lora_packet_t pkt;
        pkt.preamble = 0xFE;
        pkt.msg_type = MSG_TYPE_EVIDENCE;
        pkt.payload_len = sizeof(attestation_evidence_t);
        memcpy(pkt.payload, evidence, pkt.payload_len);

        // Calculate CRC over the packet (excluding the CRC field itself)
        pkt.crc16 = calculate_crc16((uint8_t*)&pkt, sizeof(pkt) - 2);

        // Transmit the packet via the LoRa radio driver.
        lora_transmit((uint8_t*)&pkt, sizeof(pkt));
    }
    ```

---

## 7. Logging & Debug Strategy

*   **Log Levels:** A standard tiered logging system (`LOG_ERROR`, `LOG_WARN`, `LOG_INFO`, `LOG_DEBUG`) will be used, controlled by a compile-time flag to strip out verbose logs from production builds.
*   **Persistent Logging:** For critical security events (e.g., attestation failure, failed signature check), a log entry will be written to a dedicated circular buffer in flash memory. This provides a forensic audit trail that survives reboots. The `Logging Task` will manage this process to avoid wear and performance issues.

---

## 8. Validation Tests

### Unit Tests (On-Target)
We will use a simple `assert`-based unit testing approach to test pure functions on the actual hardware.
*   **`test_pcr_extend()`:**
    1.  Initialize a PCR to a known value.
    2.  Extend it with known data.
    3.  Assert that the resulting PCR value matches a pre-calculated expected hash.
*   **`test_attestation_signature()`:**
    1.  Call `build_attestation_evidence()` with a known nonce.
    2.  Use `se_verify_external_signature()` to verify the generated signature against the generated evidence.
    3.  Assert that the verification passes.

### Integration Test Plan
1.  Set up a second LoRa device to act as a mock "verifier".
2.  The verifier sends an attestation challenge packet with a random nonce.
3.  **Expected:** The FIRM-LOCK device receives the challenge, the Comm Task queues a request for the Attestation Task, the Attestation Task builds and signs the evidence, and the device transmits a valid evidence packet back.
4.  The verifier receives the evidence and successfully validates the signature and the nonce.

---

**Next Phase:** [[09_Attestation_Protocol.md|Phase 09: Attestation Protocol]]
