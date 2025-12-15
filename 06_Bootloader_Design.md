---
title: 'Phase 06: Bootloader Design'
project: FIRM-LOCK
phase: 06
date: 2025-12-15
tags:
  - 'FIRM-LOCK'
  - 'Firmware'
  - 'Bootloader'
  - 'MCUboot'
  - 'SecureBoot'
status: DRAFT
dependencies: "[[05_Circuit_Design.md]]"
next: "[[07_Hardware_Prototype.md]]"
---

# Phase 06: Bootloader Design

> **Objective:** To design a robust, secure bootloader that establishes the root of trust for the device's software stack. This document details the architecture, configuration, and security mechanisms of the bootloader, including signature verification, anti-rollback protection, and measured boot.
> **Duration:** 3 Days
> **Deliverables:**
> - A complete bootloader architecture based on MCUboot.
> - Detailed specifications for the firmware image format and signing process.
> - Pseudocode and implementation details for all core security functions.
> - A plan for fail-safe recovery and a validation test suite.

---

## Table of Contents
1. [[#1. Bootloader Architecture: The Chain of Trust]]
2. [[#2. MCUboot: Rationale & Configuration]]
3. [[#3. Firmware Image Format]]
4. [[#4. Signature Verification Algorithm]]
5. [[#5. Anti-Rollback Implementation]]
6. [[#6. Measured Boot (PCR Extension)]]
7. [[#7. Recovery & Fail-Safe Mechanism]]
8. [[#8. Build & Signing Process]]
9. [[#9. Validation Test Plan]]

---

## 1. Bootloader Architecture: The Chain of Trust

The boot process is not a single step but a sequence of stages, each one cryptographically verifying the next before passing control. This creates an unbroken "chain of trust" from an immutable hardware root to the final application.

```
┌──────────────────────────────────────────────────────────────┐
│  STAGE 0: MCU ROM Bootloader (Immutable)                    │
│  • Executed on power-on from factory-programmed ROM.        │
│  • Its only job is to load the Stage 1 bootloader from a    │
│    fixed address in flash.                                  │
│  • It is implicitly trusted as it's part of the silicon.    │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  STAGE 1: FIRM-LOCK Bootloader (MCUboot)                    │
│  • This is the core of our secure boot mechanism.           │
│  • Its integrity is protected by MCU flash protection       │
│    (e.g., STM32U5's WRP - Write Protection).                │
│  • It initializes the Secure Element.                       │
│  • It performs the signature verification and anti-rollback │
│    check on the application firmware.                       │
│  • It performs the "measured boot" process.                 │
└─────────────────────────────┬────────────────────────────────┘
                              │ (If verification passes)
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  APPLICATION FIRMWARE                                        │
│  • The main logic of the device.                            │
│  • Contains the attestation agent, communication stacks,    │
│    and business logic.                                      │
│  • It trusts that it has been loaded by a legitimate        │
│    bootloader.                                              │
└──────────────────────────────────────────────────────────────┘
```

This architecture directly addresses the threats of bootloader spoofing and tampering (T01) by anchoring trust in immutable hardware features.

---

## 2. MCUboot: Rationale & Configuration

Instead of writing a bootloader from scratch, we will use **MCUboot**, a widely adopted, open-source secure bootloader.

*   **Rationale:**
    *   **Battle-Tested:** MCUboot is used by major RTOS projects like Zephyr, NuttX, and Mynewt, meaning it has been vetted by a large community.
    *   **Portable:** It has existing ports for our target MCU family (STM32).
    *   **Feature-Rich:** It provides all the core features we need out-of-the-box: image signing (ECDSA, RSA), rollback protection, and multiple image slots for fail-safe updates.
    *   **Focus:** Using MCUboot allows us to focus on the *integration* and *policy* of our secure boot, rather than reinventing the low-level flash and crypto primitives.

*   **Key Configuration (`mcuboot_config.h`):**
    ```c
    // --- Signature Configuration ---
    // Use ECDSA P-256 for signatures, which is hardware-accelerated on many MCUs.
    #define MCUBOOT_SIGN_EC256

    // Use our custom Secure Element integration for crypto operations.
    #define MCUBOOT_USE_ENC_KEY_SE
    #define MCUBOOT_USE_SIGN_KEY_SE

    // --- Boot Strategy ---
    // Always validate the primary slot on every boot. Do not trust cached results.
    #define MCUBOOT_VALIDATE_PRIMARY_SLOT

    // --- Security Features ---
    // Enable the measured boot feature to record PCRs.
    #define MCUBOOT_MEASURED_BOOT

    // Enable hardware-based anti-rollback protection using the SE's monotonic counter.
    #define MCUBOOT_HW_ROLLBACK_PROT

    // --- Flash Configuration ---
    // Use the platform-specific flash driver for STM32U5.
    #define MCUBOOT_USE_FLASH_AREA_GET_SECTORS
    ```

---

## 3. Firmware Image Format

MCUboot wraps the raw application binary with a header and appends a trailer containing metadata and the signature. This structured format is what allows the bootloader to verify the image.

```
┌──────────────────────────────────────┐ ─┐
│  MCUboot Header (32 bytes)           │  │
│  - Magic: 0x96f3b83d                 │  │
│  - Load address, Header size         │  │
│  - Image size                        │  │
│  - Flags, Version number             │  │
├──────────────────────────────────────┤  │
│                                      │  │
│  Application Code & Data             │  │ Protected by
│  (.text, .rodata, .data)             │  │ Hash & Signature
│  ...                                 │  │
│                                      │  │
│                                      │  │
├──────────────────────────────────────┤ ─┘
│  TLV Area (Type-Length-Value)        │
│  - TLV Magic: 0x6907                 │
│  - TLV Size                          │
├──────────────────────────────────────┤
│  Tag: Image Hash (SHA-256)           │
│  Length: 32                          │
│  Value: [32-byte hash]               │
├──────────────────────────────────────┤
│  Tag: Security Counter               │
│  Length: 4                           │
│  Value: [Anti-rollback version]      │
├──────────────────────────────────────┤
│  Tag: ECDSA-P256 Signature           │
│  Length: 64                          │
│  Value: [64-byte signature]          │
└──────────────────────────────────────┘
```

The signature is calculated over the header and the application code. The hash is stored in the TLV area as a final integrity check.

---

## 4. Signature Verification Algorithm

The core of the secure boot process is the signature validation. Here is a pseudocode representation of the logic within MCUboot.

```python
def verify_firmware(firmware_slot_address):
    # 1. Read the image header from the flash address.
    header = flash.read(firmware_slot_address, sizeof(Header))
    if header.magic != MCUBOOT_MAGIC:
        return FAIL("Invalid magic number")

    # 2. Calculate the hash of the protected part of the image.
    # This includes the header itself and the application code.
    protected_len = header.header_size + header.image_size
    calculated_hash = sha256(flash.stream(firmware_slot_address, protected_len))

    # 3. Read the TLV (Tag-Length-Value) area.
    tlv_address = firmware_slot_address + protected_len
    tlv = parse_tlv_area(tlv_address)

    # 4. Compare hashes.
    hash_from_tlv = tlv.get(TAG_SHA256)
    if calculated_hash != hash_from_tlv:
        return FAIL("Hash mismatch, image is corrupt")

    # 5. Get the signature from the TLV.
    signature = tlv.get(TAG_SIGNATURE_ECDSA256)

    # 6. Get the public key for verification.
    # In our design, this is read from the Secure Element or baked into the bootloader.
    public_key = secure_element.read_signer_public_key()

    # 7. Perform the cryptographic verification.
    # This operation is offloaded to the Secure Element for security and performance.
    is_valid = secure_element.ecdsa_verify(
        public_key,
        calculated_hash,
        signature
    )

    if not is_valid:
        return FAIL("Signature is invalid")

    return PASS
```

---

## 5. Anti-Rollback Implementation

To mitigate downgrade attacks (T03), we use a monotonic counter inside the ATECC608A Secure Element. This is a hardware counter that can only be incremented.

*   **During Signing:** The firmware version number is embedded in the image's TLV area.
*   **During Boot:** The bootloader performs this check *after* signature verification.

**Pseudocode:**
```c
// This check is performed after a valid signature is confirmed.
ATCA_STATUS check_anti_rollback() {
    // 1. Read the security counter from the firmware image's TLV block.
    uint32_t image_security_counter = image_header.security_counter;

    // 2. Read the monotonic counter value from the Secure Element.
    // This counter was incremented after the last successful boot.
    uint32_t device_security_counter = 0;
    atcab_read_zone(ATCA_ZONE_DATA, COUNTER_SLOT, 0, 0, &device_security_counter, 4);

    // 3. Compare the versions.
    if (image_security_counter < device_security_counter) {
        // The image is valid, but older than the current running version.
        // This is a rollback attack.
        LOG_ERROR("Rollback detected! Image version: %d, Device version: %d",
                  image_security_counter, device_security_counter);
        return ATCA_GEN_FAIL; // Indicate failure
    }

    // 4. If the image version is new or the same, the check passes.
    // The counter will be incremented only after the new image
    // successfully boots and marks itself as "permanent".
    return ATCA_SUCCESS;
}
```

---

## 6. Measured Boot (PCR Extension)

Measured boot creates a cryptographic audit trail of the boot process. Instead of just trusting the code, we *measure* it. This is the foundation for our remote attestation capability.

**PCR Extend Operation:**
The core operation is `PCR_new = SHA256(PCR_old || SHA256(data_to_measure))`. This is a one-way function; you cannot remove a measurement from a PCR.

```
   ┌──────────────────┐                     ┌──────────────────┐
   │ Current PCR Value│                     │ Hash of New Data │
   │ (32 bytes)       │                     │ (32 bytes)       │
   └────────┬─────────┘                     └────────┬─────────┘
            │                                        │
            ▼                                        ▼
   ┌──────────────────────────────────────────────────────────┐
   │ Concatenated Buffer (64 bytes)                           │
   └──────────────────────────────────────────────────────────┘
                                │
                                ▼
                          ┌───────────┐
                          │ SHA-256   │
                          └───────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │ New PCR Value    │
                       │ (32 bytes)       │
                       └──────────────────┘
```

**Implementation in Bootloader:**
```c
// Global PCR state (in RAM)
uint8_t g_pcr_bank[4][32] = {0};

// Extends a PCR with the hash of the provided data.
void pcr_extend(uint8_t pcr_index, const uint8_t *data, size_t data_len) {
    uint8_t data_hash[32];
    uint8_t combined_buffer[64];

    // 1. Hash the data to be measured.
    atcab_sha(data_len, data, data_hash);

    // 2. Concatenate the old PCR value and the new hash.
    memcpy(combined_buffer, g_pcr_bank[pcr_index], 32);
    memcpy(combined_buffer + 32, data_hash, 32);

    // 3. Hash the combined buffer to get the new PCR value.
    atcab_sha(64, combined_buffer, g_pcr_bank[pcr_index]);
}

// --- Usage during boot ---
void perform_measured_boot() {
    // PCR[0]: Measure the bootloader itself.
    pcr_extend(0, BOOTLOADER_FLASH_START, BOOTLOADER_SIZE);

    // PCR[1]: Measure the application code.
    pcr_extend(1, app_header.image_address, app_header.image_size);

    // PCR[2]: Measure the configuration data.
    pcr_extend(2, CONFIG_FLASH_START, CONFIG_SIZE);

    // PCR[3]: Measure the device's unique public key from the SE.
    uint8_t pubkey[64];
    secure_element.read_public_key(pubkey);
    pcr_extend(3, pubkey, sizeof(pubkey));
}
```

---

## 7. Recovery & Fail-Safe Mechanism

A failed update or a buggy application should not result in a permanently bricked device (mitigating T07). MCUboot supports this with a "golden image" or "revert" mechanism.

*   **Boot Failure Detection:** The bootloader uses a few bytes in RAM (or a non-volatile register) as a boot failure counter.
*   **Application Responsibility:** A successful application boot must "pet" or reset this counter. If the application crashes before it can do this, the counter will remain.
*   **Recovery Logic (in MCUboot):**
    ```c
    // On boot, before jumping to app:
    boot_failure_count = read_boot_counter();
    boot_failure_count++;
    write_boot_counter(boot_failure_count);

    if (boot_failure_count >= MAX_BOOT_FAILURES) {
        LOG_CRITICAL("Max boot failures reached. Reverting to golden image.");
        // Mark the current primary slot as invalid.
        // MCUboot will automatically try to boot from another valid slot,
        // which could be the "golden" image.
        mark_image_as_invalid(PRIMARY_SLOT);
        reset_boot_counter();
        HAL_NVIC_SystemReset();
    }

    // ... jump to app ...
    ```

---

## 8. Build & Signing Process

Security must be usable. The process for a developer to sign and flash firmware needs to be simple and scriptable.

**Developer Workflow:**
```bash
# 1. Build the application firmware as usual.
$ make

# 2. Use MCUboot's 'imgtool.py' to sign the binary.
# This wraps the binary with the header and appends the TLV block.
$ imgtool sign \
    --key keys/device-signer-key.pem \
    --header-size 0x200 \
    --align 8 \
    --version 1.1.0 \
    --security-counter 10 \
    build/app.bin \
    build/app-signed.bin

# 3. (Optional) Verify the signature locally.
$ imgtool verify --key keys/device-signer-pubkey.pem build/app-signed.bin

# 4. Flash the signed image to the device's primary slot.
$ openocd -f stm32u5.cfg -c "program build/app-signed.bin 0x08000000 verify reset exit"
```

---

## 9. Validation Test Plan

To verify the bootloader's security, the following tests **SHALL** be performed:
- [ ] **Test 1 (Unsigned Firmware):** Attempt to flash a raw, unsigned binary. **Expected:** Bootloader refuses to boot.
- [ ] **Test 2 (Incorrect Signature):** Sign the firmware with a different, unauthorized key. **Expected:** Bootloader refuses to boot.
- [ ] **Test 3 (Corrupted Image):** Flip a single bit in a correctly signed image and flash it. **Expected:** Bootloader hash check fails; refuses to boot.
- [ ] **Test 4 (Rollback Attack):** Successfully boot version `N+1`. Then, attempt to flash the valid, signed version `N`. **Expected:** Bootloader anti-rollback check fails; refuses to boot.
- [ ] **Test 5 (Measured Boot):** Boot the same valid firmware twice. **Expected:** The final PCR values are identical both times.
- [ ] **Test 6 (Recovery):** Flash a deliberately broken application that crashes on startup. **Expected:** After 3 reboots, the device automatically rolls back to the golden image and boots successfully.

---

**Next Phase:** [[07_Hardware_Prototype.md|Phase 07: Hardware Prototype (Breadboard)]]
