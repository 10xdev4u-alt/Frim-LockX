---
title: 'Phase 13: System Integration'
project: FIRM-LOCK
phase: 13
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Integration
  - E2ETesting
  - LoRaWAN
status: DRAFT
dependencies: "[[12_Verifier_Backend.md]]"
next: "[[14_Testing_and_Certification.md]]"
---

# Phase 13: System Integration

> **Objective:** To integrate all hardware, firmware, and backend components into a single cohesive system and to test the complete, end-to-end attestation and update flows. This phase moves from component-level validation to system-level verification.
> **Duration:** 3 Days
> **Deliverables:**
> - A fully integrated hardware and software prototype.
> - A functional communication link between the device and the Verifier backend via a LoRaWAN network.
> - Successful completion of end-to-end test plans for attestation and secure updates.
> - A troubleshooting guide for common integration issues.

---

## Table of Contents
1. [[#1. Integration Strategy & Data Flow]]
2. [[#2. Hardware-Firmware Integration]]
3. [[#3. Device-to-Backend Integration (LoRaWAN)]]
4. [[#4. End-to-End Attestation Test Plan]]
5. [[#5. End-to-End Secure Update Test Plan]]
6. [[#6. Common Integration Challenges & Troubleshooting]]

---

## 1. Integration Strategy & Data Flow

We will follow a phased integration strategy to isolate issues and build complexity incrementally.

**Phases:**
1.  **Hardware/Firmware (Hw/Fw) Integration:** Get the final, complete firmware (bootloader + application) running on the breadboard prototype.
2.  **Device/Network (Dev/Net) Integration:** Get the device to successfully join a LoRaWAN network and transmit basic data packets.
3.  **Network/Backend (Net/Be) Integration:** Configure the LoRaWAN network server to forward device data to our Verifier backend's API endpoints.
4.  **Full System Test:** Conduct the end-to-end tests using all integrated components.

### Full End-to-End Data Flow
```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────┐   ┌──────────┐
│ FIRM-LOCK│   │ LoRa     │   │ LoRaWAN  │   │ LoRaWAN Network  │   │ Verifier │
│ Device   │──▶│ Gateway  │──▶│ Network  │──▶│ Application      │──▶│ Backend  │
│          │   │          │   │ Server   │   │ Server (HTTP Int)│   │ (API)    │
└──────────┘   └──────────┘   └──────────┘   └──────────────────┘   └──────────┘
 (Uplink: Evidence)

┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────────┐   ┌──────────┐
│ FIRM-LOCK│   │ LoRa     │   │ LoRaWAN  │   │ LoRaWAN Network  │   │ Verifier │
│ Device   │◀──│ Gateway  │◀──│ Network  │◀──│ Application      │◀──│ Backend  │
│          │   │          │   │ Server   │   │ Server (Downlink)│   │ (API)    │
└──────────┘   └──────────┘   └──────────┘   └──────────────────┘   └──────────┘
 (Downlink: Challenge)
```

---

## 2. Hardware-Firmware Integration

This step involves moving from test firmware snippets to the complete application.

*   **Action 1: Build Complete Firmware.**
    *   Compile the MCUboot bootloader configured as per [[06_Bootloader_Design.md|Phase 06]].
    *   Compile the core application firmware from [[08_Firmware_Core.md|Phase 08]].
    *   Use `imgtool.py` to sign the application binary, creating the final `app-signed.bin`.

*   **Action 2: Flash the Prototype.**
    *   Connect the ST-LINK debugger to the Nucleo board.
    *   Erase the entire flash memory of the MCU.
    *   Flash the MCUboot binary to its starting address (e.g., `0x08060000`).
    *   Flash the signed application binary to the primary slot address (e.g., `0x08000000`).

*   **Action 3: Initial Boot Test.**
    *   Open a serial console connected to the Nucleo's Virtual COM Port.
    *   Reset the device.
    *   **Expected Output:**
        1.  Logs from MCUboot indicating it is starting.
        2.  Logs from MCUboot showing "Verifying signature..."
        3.  Logs from MCUboot showing "Booting image from primary slot."
        4.  Logs from the application firmware's `main()` function, indicating the application has started.
        5.  Logs from the FreeRTOS scheduler starting up.
    *   **Success Criteria:** The application runs. This proves the bootloader correctly verified and jumped to the application.

---

## 3. Device-to-Backend Integration (LoRaWAN)

This step establishes the communication link over the air. We will use **The Things Network (TTN)** as our LoRaWAN Network Server for this prototype, as it's free and well-documented.

*   **Action 1: Configure TTN Application.**
    *   Create a new application in the TTN console.
    *   Register a new device within the application. Since we are using a custom protocol, we will register it using ABP (Activation by Personalization) for simplicity, or use OTAA (Over-the-Air Activation) for a full test. This involves adding the `DevEUI`, `AppEUI`, and `AppKey` to both the TTN console and the device's firmware.

*   **Action 2: Implement LoRaWAN Stack.**
    *   Integrate the LoRaWAN stack into the firmware's Communication Task. This involves handling join requests and formatting data into LoRaWAN packets.
    *   The `lora_packet_t` defined in Phase 08 will be the payload of the LoRaWAN message.

*   **Action 3: Configure TTN HTTP Integration.**
    *   In the TTN console, navigate to `Application -> Integrations -> Webhooks`.
    *   Create a new webhook.
    *   Set the **Uplink message** URL to the public endpoint of our Verifier backend: `https://<verifier-host>/api/evidence`.
    *   This tells TTN to automatically `POST` the JSON data from any device uplink to our API.

*   **Action 4: Configure Downlink Path.**
    *   To send a `Challenge`, the Verifier needs to send a downlink message. The process is:
        1.  Verifier calls the TTN API: `POST /api/v3/as/applications/{app_id}/devices/{dev_id}/down/push`
        2.  The body of this API call contains the `Challenge` payload, correctly encoded.
        3.  TTN queues the downlink message and sends it to the device after the next uplink.

---

## 4. End-to-End Attestation Test Plan

This is the first full test of the entire system.

**Prerequisites:**
*   The Verifier backend is running and publicly accessible.
*   A device and an appraisal policy for its firmware are registered in the Verifier database.
*   The integrated prototype is powered on and has successfully joined the TTN network.

**Test Steps:**
1.  **Initiate:** Using the Verifier's web dashboard (or a cURL command), make a `POST` request to `/api/attestations/{device_id}`.
2.  **Observe (Verifier):** Check the Verifier logs. It should log "Generated nonce XXX for device YYY" and "Sending challenge via downlink."
3.  **Observe (TTN Console):** Look at the device's data feed. You should see a downlink message being queued and then sent.
4.  **Observe (Device):** Check the device's serial console output. It should log:
    *   "Received downlink packet."
    *   "Attestation challenge received with nonce XXX."
    *   "Building evidence..."
    *   "Evidence signed successfully."
    *   "Transmitting evidence uplink."
5.  **Observe (TTN Console):** An uplink message should appear from the device. The payload should be the raw bytes of the `AttestationEvidence` structure. The HTTP integration should show a successful forward to the Verifier API.
6.  **Observe (Verifier):** The Verifier logs should show:
    *   "Received evidence packet."
    *   The full `appraise_evidence` logic trace: "Nonce OK", "Signature OK", "PCRs OK", "Anti-rollback OK".
    *   "Device {device_id} is TRUSTED."
7.  **Final Verification:** In the Verifier web dashboard, refresh the device's page. Its status should now be green and show `TRUSTED`, with an updated "Last Seen" timestamp.

**Success Criteria:** The device status is updated to `TRUSTED` in the Verifier database.

---

## 5. End-to-End Secure Update Test Plan

This tests the full OTA update flow.

**Prerequisites:**
*   The device is currently running firmware v1.0 and is `TRUSTED`.
*   A new firmware, v1.1, has been compiled and signed.
*   The v1.1 binary and a corresponding SUIT manifest have been uploaded to the Update Server.

**Test Steps:**
1.  **Initiate:** An operator triggers an update for the device to v1.1 from the Verifier/Update dashboard.
2.  **Observe (Device):** The device receives an "Update Available" notification. It logs its intent to start the update process.
3.  **Observe (Device & Network):** The device begins requesting the firmware binary in fragments. Observe the series of request/response packets over LoRaWAN. This will be slow.
4.  **Observe (Device):** Once the download is complete, the device logs:
    *   "Download complete. Verifying image hash against manifest."
    *   "Hash OK. Requesting swap and rebooting."
5.  **Observe (Device Console):** The device reboots. The MCUboot logs should appear, indicating it has detected a swap request and is performing the swap. It then boots the new image.
6.  **Observe (Device Console):** The new firmware (v1.1) starts up. It should log its new version number. It performs its self-tests and logs "New image confirmed. Marking update as permanent."
7.  **Final Verification:** The device automatically performs a new attestation. Follow the End-to-End Attestation Test Plan again. This time, the Verifier should receive evidence containing the new firmware version (v1.1), and after appraising it against the new policy, mark the device as `TRUSTED`.

**Success Criteria:** The device successfully runs the new firmware and re-attests successfully with the new version number.

---

## 6. Common Integration Challenges & Troubleshooting

*   **Problem: Device doesn't join LoRaWAN network.**
    *   **Cause:** Incorrect `DevEUI`, `AppEUI`, or `AppKey` in firmware. Mismatch between device and network server radio frequency plans. Poor antenna connection.
    *   **Solution:** Double-check all keys and region settings. Ensure the antenna is securely connected.
*   **Problem: Uplink message appears in TTN but doesn't reach the Verifier API.**
    *   **Cause:** Incorrect webhook URL. Firewall blocking requests to the Verifier host. The Verifier API is down.
    *   **Solution:** Use a tool like `requestbin.com` as the webhook URL to inspect the data TTN is sending. Check Verifier server logs and firewall rules.
*   **Problem: Verifier receives data but reports a parsing error.**
    *   **Cause:** Mismatch in data structures between the C firmware and the Python backend. Most commonly caused by **endianness** or **struct padding**.
    *   **Solution:** Ensure both the firmware and backend are using the same byte order (little-endian). Use `__attribute__((packed))` on all C structs sent over the wire to prevent the compiler from adding padding. Log the raw byte string on the server to manually inspect its structure.

---

**Next Phase:** [[14_Testing_and_Certification.md|Phase 14: Testing & Certification]]
