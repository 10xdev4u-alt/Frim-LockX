---
title: 'Phase 10: Secure Update Mechanism'
project: FIRM-LOCK
phase: 10
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Firmware
  - OTA
  - SUIT
  - MCUboot
status: DRAFT
dependencies: "[[09_Attestation_Protocol.md]]"
next: "[[11_Key_Management.md]]"
---

# Phase 10: Secure Update Mechanism

> **Objective:** To design a secure, robust, and power-efficient mechanism for updating the firmware on a FIRM-LOCK device, ensuring that only authentic and authorized updates can be installed, even over unreliable, low-bandwidth networks.
> **Duration:** 3 Days
> **Deliverables:**
> - An update architecture leveraging MCUboot's swap mechanism.
> - A formal definition of the firmware update manifest, aligned with IETF SUIT.
> - A detailed end-to-end sequence for the entire update process.
> - Strategies for power-fail safety and bandwidth optimization.

---

## Table of Contents
1. [[#1. Update Goals & Threat Model]]
2. [[#2. Update Architecture (MCUboot-based)]]
3. [[#3. Firmware Update Manifest (IETF SUIT)]]
4. [[#4. End-to-End Update Flow]]
5. [[#5. Power-Fail Safety & Atomicity]]
6. [[#6. Bandwidth Optimization for Low-Power WANs]]

---

## 1. Update Goals & Threat Model

The ability to update firmware is critical for patching vulnerabilities and adding features. However, the update mechanism itself is a significant attack surface.

### Core Goals
*   **Authenticity & Integrity:** The device must cryptographically verify that the new firmware image originated from an authorized source (e.g., the project vendor) and was not modified in transit.
*   **Confidentiality:** (Optional) The firmware image may need to be encrypted to protect intellectual property.
*   **Anti-Rollback:** The device must reject attempts to install older, vulnerable versions of firmware over newer ones. This builds on the mechanism defined in [[06_Bootloader_Design.md|Phase 06]].
*   **Power-Fail Safety:** The update process must be atomic. A power failure at any point during the update must not "brick" the device. It should either complete the update or revert to the previous working version.

### Update-Specific Threats
*   **Malicious Update Injection (T03):** An attacker on the network attempts to send a malicious binary to the device.
*   **Interruption Attack:** An attacker cuts power or jams communication during the update process, attempting to leave the device in a corrupted, non-bootable state.
*   **Resource Exhaustion:** An attacker repeatedly sends large update files to a battery-powered device, draining its power.

---

## 2. Update Architecture (MCUboot-based)

We will leverage MCUboot's powerful and well-tested "swap/revert" update mechanism. This avoids the danger of directly overwriting the running firmware.

### Flash Layout for Updates
As defined in [[03_Architecture.md|Phase 03]], our flash memory is partitioned into dedicated slots:

```
Flash Memory:
┌───────────────────────────┐
│ Slot 0: Primary           │
│ (Active Firmware v1.0)    │
└───────────────────────────┘
┌───────────────────────────┐
│ Slot 1: Secondary         │
│ (Update Staging Area)     │
└───────────────────────────┘
┌───────────────────────────┐
│ Bootloader (MCUboot)      │
└───────────────────────────┘
```

### The Swap Mechanism
1.  **Download:** The new firmware update (v1.1) is downloaded and stored in the **Secondary Slot**. The device is still running the old firmware (v1.0) from the **Primary Slot**.
2.  **Verification:** The running application verifies the integrity and signature of the image in the Secondary Slot.
3.  **Swap Request:** If the image is valid, the application writes a "swap" request to a dedicated area of flash that MCUboot can read. It then triggers a system reboot.
4.  **The Swap:** On reboot, MCUboot detects the swap request. It performs the update by swapping the contents of the Primary and Secondary slots. This is done in a careful, power-fail-safe manner, typically by swapping small sectors one at a time and keeping a log of its progress.
5.  **First Boot:** MCUboot then attempts to boot the new firmware (v1.1) from the Primary Slot. The old firmware (v1.0) is now in the Secondary Slot, acting as a backup.
6.  **Confirmation:** The new v1.1 application runs its own self-tests. If everything is working correctly, it "confirms" the update by writing an `image_ok` flag. This tells MCUboot that the update was successful and the backup (v1.0) can be overwritten in the future.
7.  **Revert (If Needed):** If the new v1.1 application fails to boot or confirm itself, on the next reboot MCUboot will see that the swap was not confirmed. It will automatically **revert** the swap, copying v1.0 back into the Primary slot, ensuring the device returns to a known-good state.

This architecture directly mitigates interruption attacks.

---

## 3. Firmware Update Manifest (IETF SUIT)

A firmware binary alone is not enough. It needs metadata to describe what it is, where it goes, and how to verify it. We will use a manifest format aligned with the **IETF SUIT (Software Updates for IoT) standard**.

The manifest is a small, separate file that is downloaded and verified *before* the main firmware binary.

### SUIT Manifest Structure (Simplified)
```
┌──────────────────────────────────────────────────┐
│ SUIT Manifest (Signed)                           │
├──────────────────────────────────────────────────┤
│ 1. Metadata:                                     │
│    - Vendor ID: "FIRM-LOCK Project"              │
│    - Device Class ID: "FIRM-LOCK_HW_v1"          │
│                                                  │
│ 2. Versioning & Anti-Rollback:                   │
│    - Firmware Version: 1.1.0                     │
│    - Security Counter: 11                        │
│                                                  │
│ 3. Image Information:                             │
│    - Image Size: 114,230 bytes                   │
│    - Image Digest: SHA256(firmware.bin) -> [hash]│
│                                                  │
│ 4. Signature:                                    │
│    - ECDSA-P256 signature over the entire manifest │
│      (signed by the vendor's private key)        │
└──────────────────────────────────────────────────┘
```

**Benefits of this approach:**
*   **Efficiency:** The device can quickly download and verify the small manifest file. If the signature is invalid or the version is too old, it can reject the update *before* wasting time and power downloading the large firmware binary.
*   **Flexibility:** The SUIT standard is extensible and can describe updates for multiple components (e.g., bootloader, radio firmware) in a single manifest.

---

## 4. End-to-End Update Flow

This sequence diagram details the entire process, from the cloud to the device.

```
  Verifier         Update Server         Prover (Device)
     │                  │                      │
     │ 1. Attestation reveals device is on v1.0
     │                  │                      │
     │ 2. Trigger Update(DeviceID)
     ├─────────────────▶│                      │
     │                  │ 3. Generate Manifest for v1.1
     │                  │    (Sign with Vendor Key)
     │                  │                      │
     │                  │ 4. Notify Device:
     │                  │    "Update available at /fw/v1.1"
     │                  ├─────────────────────▶│
     │                  │                      │
     │                  │                      │ 5. Request Manifest
     │                  │◀─────────────────────┤
     │                  │                      │
     │                  │ 6. Send Manifest
     │                  ├─────────────────────▶│
     │                  │                      │
     │                  │                      │ 7. VERIFY MANIFEST:
     │                  │                      │    - Check Signature (Pass)
     │                  │                      │    - Check Version > Current (Pass)
     │                  │                      │
     │                  │                      │ 8. Request Firmware Binary
     │                  │                      │    (in fragmented blocks)
     │                  │◀─────────────────────┤
     │                  │                      │
     │                  │ 9. Send Firmware Chunks
     │                  ├─────────────────────▶│
     │                  │                      │ 10. Write chunks to Secondary Slot
     │                  │                      │
     │                  │                      │ 11. After last chunk, VERIFY IMAGE:
     │                  │                      │     - SHA256(image) == Digest in Manifest? (Pass)
     │                  │                      │
     │                  │                      │ 12. Request MCUboot Swap & Reboot
     │                  │                      │
     │                  │                      │ ... MCUboot performs swap ...
     │                  │                      │
     │                  │                      │ 13. New App (v1.1) boots
     │                  │                      │
     │                  │                      │ 14. Run self-tests, confirm update is OK
     │                  │                      │
     │                  │                      │ 15. Perform new attestation to confirm v1.1
     │                  │─────────────────────▶│
     │◀───────────────────────────────────────┤
     │                                        │
     │ 16. Verifier confirms device is now TRUSTED and on v1.1
     │                                        │
```

---

## 5. Power-Fail Safety & Atomicity

The MCUboot swap algorithm is designed to be **atomic**. This means it can be interrupted at any point by a power failure without corrupting the device.

**How it Works:**
*   MCUboot does not erase the old firmware until the new firmware is fully in place and verified.
*   It uses a "swap status" area in flash to keep track of its progress. It might copy one 4KB sector at a time, updating the status after each successful copy.
*   **If power fails mid-swap:** On the next boot, MCUboot will detect the incomplete swap from the status area. It knows exactly where it left off and can either complete the swap or revert it safely. The device never enters a state where there isn't at least one bootable image.
*   **The `image_ok` Flag:** The swap process isn't considered complete until the new application explicitly sets the `image_ok` flag. If the device reboots before this flag is set, MCUboot assumes the new application is faulty and automatically reverts to the previous version. This prevents a buggy update from permanently bricking the device.

---

## 6. Bandwidth Optimization for Low-Power WANs

Downloading a 128KB firmware image over LoRa is impractical due to low data rates and duty cycle limitations. Several strategies must be employed.

*   **Transport-Agnostic Design:** The core update logic should be separate from the transport layer. The device might receive the update over a faster channel (like BLE or USB from a technician's phone) if available.
*   **Fragmented Block Transfer:** The firmware binary must be broken down into small, numbered chunks (e.g., 256 bytes each). The device requests each chunk sequentially. This allows the download to be resumed if it's interrupted. A simple protocol is needed:
    *   Device to Server: `GET_CHUNK(ImageID, Chunk_#)`
    *   Server to Device: `CHUNK_RESPONSE(ImageID, Chunk_#_Data)`
*   **Delta (Differential) Updates:** This is the most efficient method. Instead of sending the full firmware binary, the update server computes a binary "diff" between the old version (v1.0) and the new version (v1.1). This patch file can be significantly smaller than the full image.
    *   **Process:** The device downloads the small patch file. It then reads the original firmware from its own Primary Slot, applies the patch in RAM, and writes the resulting new firmware to the Secondary Slot.
    *   **Trade-off:** This increases the complexity and CPU load on the device but drastically reduces airtime and power consumption. This is a key feature for battery-powered devices. We will use libraries like `bsdiff` as a starting point for this mechanism.

---

**Next Phase:** [[11_Key_Management.md|Phase 11: Key Management]]
