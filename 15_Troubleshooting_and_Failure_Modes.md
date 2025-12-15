---
title: 'Phase 15: Troubleshooting, Failure Modes & Field Fixes'
project: FIRM-LOCK
phase: 15
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Troubleshooting
  - FailureModes
  - FieldSupport
status: DRAFT
dependencies: "[[14_Testing_and_Certification.md]]"
next: "[[16_Final_Packaging_and_Pitch.md]]"
---

# Phase 15: Troubleshooting, Failure Modes & Field Fixes

> **Objective:** To document every conceivable way this system can fail and provide clear, actionable steps to diagnose, fix, and prevent those failures. This is a living document intended for field engineers, support teams, and future developers.
> **Audience:** Field engineers, support teams, future developers
> **Format:** Living document (updated as new issues are discovered)

---

## Table of Contents
1. [[#1. Diagnostic Framework]]
2. [[#2. Hardware Failure Modes]]
3. [[#3. Firmware/Bootloader Issues]]
4. [[#4. Attestation Protocol Failures]]
5. [[#5. Secure Update Failures]]
6. [[#6. Key Management Issues]]
7. [[#7. Environmental/Physical Failures]]
8. [[#8. Supply Chain & Counterfeit Detection]]
9. [[#9. Field Repair Procedures]]
10. [[#10. Lessons Learned Log]]

---

## 1. Diagnostic Framework

### 1.1 Systematic Debug Approach

When a device fails, a systematic approach is critical. Randomly trying fixes is inefficient. We use a 5-layer debug stack, starting from the physical layer and working up.

**The 5-Layer Debug Stack:**
```
┌─────────────────────────────────────────┐
│ LAYER 5: System Behavior               │ ← User-visible symptoms
│ (e.g., "attestation fails")            │
├─────────────────────────────────────────┤
│ LAYER 4: Protocol/Logic                │ ← Message formats, state machines
│ (e.g., "verifier rejects signature")   │
├─────────────────────────────────────────┤
│ LAYER 3: Firmware/Software             │ ← Code bugs, race conditions
│ (e.g., "PCR not extended")             │
├─────────────────────────────────────────┤
│ LAYER 2: Hardware Interface            │ ← I2C/SPI/UART issues
│ (e.g., "SE not responding")            │
├─────────────────────────────────────────┤
│ LAYER 1: Physical/Electrical           │ ← Power, solder, ESD damage
│ (e.g., "3.3V rail is 2.8V")            │
└─────────────────────────────────────────┘

Debug Strategy: Always start at Layer 1 and work your way up!
```

### 1.2 Essential Debug Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| **Multimeter** | Voltage, continuity | Layer 1: Power issues, shorts |
| **Oscilloscope** | Signal integrity, timing | Layer 2: I2C/SPI glitches, clock issues |
| **Logic Analyzer** | Protocol decode | Layer 2/4: Communication errors, verifying data |
| **JTAG/SWD Debugger** | Step-through, breakpoints | Layer 3: Firmware crashes, memory inspection |
| **Serial Console** | Logs, interactive debug | Layer 3/5: 80% of all issues start here |

### 1.3 LED Blink Codes (When Serial is Dead)

If the device is unresponsive over serial, it will communicate critical errors via blink codes on its status LED.

```c
// Emergency diagnostic blink patterns (blocking code)
void error_blink(error_code_t err) {
    while(1) {
        for (int i = 0; i < err; i++) {
            LED_ON();  delay_ms(200);
            LED_OFF(); delay_ms(200);
        }
        delay_ms(2000);  // Long pause between codes
    }
}

// Error codes:
// 1 blink:  Hardware init failed (SE or LoRa)
// 2 blinks: Bootloader signature check failed
// 3 blinks: Attestation error
// 4 blinks: Update process failed
// 5 blinks: Critical security violation (PCR mismatch)
// 6 blinks: Watchdog reset loop
// Solid ON: Stack overflow / memory corruption / hard fault
```

---

## 2. Hardware Failure Modes

### ISSUE #1: Secure Element Not Detected on Boot

**Symptom:**
```
[ERR] SE init failed: ATCA_COMM_FAIL
I2C scan shows no device at 0x60
```

**Root Causes (Ranked by Probability):**

#### A. Missing I2C Pull-Up Resistors (60% of cases)
**Diagnosis:** With the device powered on, use a multimeter to measure the voltage on the SDA and SCL lines. They should be pulled high to 3.3V. If they are at 0V or floating, the pull-ups are missing or have failed.
**Fix:** Add 4.7kΩ resistors from SDA to 3.3V and SCL to 3.3V.
**Prevention:** Ensure pull-up resistors are included in the PCB design as per the I2C specification.

---

#### B. Wrong I2C Address (25%)
**Diagnosis:** The ATECC608A default address is `0x60`, but it can be configured differently. Run an I2C scanner utility in firmware to check all possible addresses.
**Fix:** Update the firmware configuration (`g_atecc608a_cfg.atcai2c.slave_address`) to match the discovered address.
**Prevention:** Document the configured I2C address for each batch of devices during manufacturing.

---

#### C. Counterfeit/Damaged IC (10%)
**Diagnosis:** A genuine ATECC608A will respond to a "wake" sequence with a specific 4-byte pattern (`0x04 0x11 0x33 0x43`). If the wake command times out or returns incorrect bytes, the chip is likely damaged or counterfeit.
**Fix:** Replace the IC.
**Prevention:** Procure components ONLY from authorized distributors as specified in the BOM.

---

### ISSUE #2: LoRa Module TX Fails / No Range

**Symptom:** Firmware reports "LoRa TX OK" but a nearby receiver gets nothing.

**Root Causes:**

#### A. No Antenna / Wrong Impedance (70%)
**CRITICAL:** **NEVER transmit without an antenna** or a 50Ω dummy load connected. This can permanently damage the radio's Power Amplifier (PA) due to impedance mismatch.
**Diagnosis:** Measure the current draw during TX. A +20dBm transmission should draw ~120mA. If the current is significantly lower (<50mA), the PA is not delivering power and is likely damaged.
**Fix:** If the PA is damaged, the entire LoRa module must be replaced.
**Prevention:** Implement an antenna detection mechanism in firmware if the hardware allows, or have a strict pre-flight checklist for operators.

---

#### B. Wrong Frequency Configuration (20%)
**Diagnosis:** The transmitter and receiver must be on the exact same frequency (e.g., 915.0 MHz in the US, 868.0 MHz in the EU). Verify the configured frequency in the firmware on both ends. Use an SDR to confirm the actual transmission frequency.
**Fix:** Correct the frequency configuration in the firmware to match local regulations and the receiver's setting.

---

### ISSUE #3: Random Resets During Operation (Brownouts)

**Symptom:** The device boots, runs for a few seconds, then suddenly resets. The reset reason register indicates a "Brownout Reset". This often happens during LoRa transmission.

**Root Cause:** The high current spike (~120mA) of the LoRa TX causes the system voltage to sag below the MCU's brownout detection threshold (e.g., 2.8V).
**Diagnosis:** Use an oscilloscope to monitor the 3.3V rail during a LoRa transmission. If you see a significant voltage drop, you have found the problem.
**Fixes (Ranked):**
1.  **Add Bulk Capacitance:** Place a large capacitor (e.g., 470µF electrolytic) as close as possible to the LoRa module's VCC pin to supply current during the transient spike.
2.  **Lower TX Power:** Reduce the LoRa output power in firmware. This reduces current draw at the cost of range.
3.  **Use a Better Power Supply:** Replace the LDO with a buck converter that can handle higher peak loads more efficiently.

---

## 3. Firmware/Bootloader Issues

### ISSUE #4: Bootloader Refuses Valid Signed Firmware

**Symptom:**
```
[BOOT] Verifying app signature...
[BOOT] App signature verification: FAIL
[BOOT] Entering recovery mode.
```
You are certain the firmware was signed with the correct key.

**Root Cause: Public Key Mismatch (80%)**
The private key used to sign the firmware does not correspond to the public key embedded in the bootloader. This often happens if keys are regenerated and the bootloader isn't updated.
**Diagnosis:**
1.  In the bootloader, print a hash of the embedded public key it expects.
2.  On the development machine, generate a hash of the public key corresponding to the private key you used for signing.
3.  Compare the two hashes. If they differ, you have found the mismatch.
**Fix:** Re-flash the bootloader with the correct public key, or re-sign the firmware with the key that matches the bootloader.
**Prevention:** Store the hash of the official public signing key in version control.

---

### ISSUE #5: PCR Values Inconsistent Between Builds

**Symptom:** Attestation fails because a PCR value changed, even though the source code did not.
**Root Cause: Non-Deterministic Build.** The compiler is embedding build-specific information (like timestamps or file paths) into the binary, causing the hash to change on every compile.
**Diagnosis:**
1.  `make clean && make`. Copy the output binary to `app1.bin`.
2.  `make clean && make` again. Copy the output to `app2.bin`.
3.  `sha256sum app1.bin app2.bin`. If the hashes differ, the build is non-deterministic.
**Fix:** Add compiler and linker flags to enforce determinism.
```makefile
# Example Makefile flags for deterministic builds
CFLAGS += -Wdate-time  # Warn if __DATE__ or __TIME__ used
CFLAGS += -ffile-prefix-map=$(PWD)=.  # Strip absolute paths
LDFLAGS += --sort-section=name  # Ensure linker section order is consistent
```

---

## 4. Attestation Protocol Failures

### ISSUE #6: Verifier Times Out Waiting for Evidence

**Symptom:** The Verifier sends a `Challenge` but never receives `Evidence`.
**Root Cause A: LoRa Packet Loss (50%).** LoRa is not a guaranteed delivery protocol. The evidence packet was simply lost in transit.
**Fix:** Implement a simple retry mechanism. If the device does not receive a `Verdict` packet within a timeout after sending `Evidence`, it should re-transmit the same evidence up to 2 more times.

**Root Cause B: Device Is Sleeping (30%).** The device is in a deep sleep mode and is not listening for the challenge.
**Fix:** The device must use a "Wake-on-LoRa" mode, where the MCU is sleeping but the radio is in a low-power RX mode, configured to wake the MCU via an interrupt when a packet preamble is detected.

---

## 5. Secure Update Failures

### ISSUE #7: Update Fails with "Rollback Detected"

**Symptom:**
```
[UPDATE] Received firmware v1.2.0 (counter=5)
[UPDATE] Current device counter: 6
[UPDATE] ROLLBACK ATTACK DETECTED. Update rejected.
```
**Legitimate Cause:** A developer or operator is trying to flash an older (but still validly signed) firmware version for testing purposes.
**Fix (Debug Builds Only):**
```c
#ifdef ALLOW_ROLLBACK_DEBUG
  #warning "Rollback protection DISABLED - DEBUG ONLY"
  if (image_counter < device_counter) {
      log_warn("Rollback detected but allowed in debug mode.");
      // Bypass the rejection
  }
#endif
```
**Prevention:** The anti-rollback mechanism is a critical security feature. It should never be disabled in production builds.

---

### ISSUE #8: Update Bricks Device (Boot Loop)

**Symptom:** After a successful update, the device continuously reboots. MCUboot validates the new image, jumps to it, but the application crashes immediately, triggering a watchdog reset.
**Root Cause:** The new application has a critical bug that causes a crash during initialization, before it has a chance to "confirm" itself as good.
**Fix: Automatic Golden Image Rollback.** As designed in [[06_Bootloader_Design.md|Phase 06]], MCUboot will detect that the new image has failed to boot 3 consecutive times. It will then automatically mark the new image as invalid and revert to the previous working version, un-bricking the device.
**Field Recovery:** The device will now be back on the old firmware and can report the failed update attempt to the Verifier. The faulty firmware version can be blacklisted.

---

## 6. Key Management Issues

### ISSUE #9: Lost Private Firmware Signing Key

**Symptom:** The HSM or air-gapped laptop containing the master firmware signing key is destroyed or lost. No new firmware can be signed, and thus no new updates can be deployed.
**Prevention (Key Escrow):** The private key must never exist in only one place.
*   **Strategy:** Use Shamir's Secret Sharing to split the private key into 5 "shards".
*   Distribute these shards to 5 trusted executives.
*   Require at least 3 shards to be brought together to reconstruct the key.
*   This prevents a single point of failure or a single rogue actor from compromising the system.
**Recovery:** If the primary key is lost, use the escrowed shards to reconstruct it. Immediately initiate the key rotation procedure described in [[11_Key_Management.md|Phase 11]] to migrate the entire fleet to a new key.

---

## 7. Environmental/Physical Failures

### ISSUE #10: Device Works in Lab, Fails in Cold Field Environment

**Symptom:** Device passes all tests at 25°C but fails randomly at -15°C.
**Root Cause A: Crystal Oscillator Frequency Drift.** The MCU's crystal oscillator frequency changes slightly with temperature. At extreme cold, this drift can be enough to prevent the LoRa radio from locking onto a signal.
**Fix:** Use a Temperature-Compensated Crystal Oscillator (TCXO) instead of a standard crystal. Alternatively, implement a software-based temperature compensation routine that adjusts the radio frequency based on readings from an onboard temperature sensor.

**Root Cause B: Battery Voltage Sag.** The capacity and voltage of LiPo batteries drop significantly in the cold. A battery that provides 3.7V at room temperature might only provide 3.2V at -15°C, which is too low to handle a LoRa TX spike.
**Fix:** Use batteries rated for low-temperature operation. Implement firmware logic that enters a "low-power" mode, disabling non-critical functions when the battery voltage drops below a certain threshold.

---

## 8. Supply Chain & Counterfeit Detection

### ISSUE #11: Counterfeit Secure Element

**Symptom:** The device boots and the SE appears to initialize, but cryptographic operations (like signing) produce incorrect results or fail intermittently.
**Detection:** During provisioning, the firmware should perform a "test vector" signature.
1.  Sign a known, constant piece of data with the device's private key.
2.  Compare the resulting signature to a pre-computed "golden" signature generated by a known-genuine ATECC608A with the same key.
3.  If the signatures do not match, the chip is counterfeit.
**Fix:** Reject the device at the manufacturing stage.
**Prevention:** Only buy from authorized distributors.

---

## 9. Field Repair Procedures

### Procedure A: Reflash Bootloader (Bricked Device)

**Warning:** This is a last resort and requires physical access. It may erase the device's security history.
**Steps:**
1.  Connect an SWD debugger (e.g., ST-LINK) to the device's debug header.
2.  Use OpenOCD to connect to the MCU. If Readout Protection (RDP) is enabled, you may need to issue a mass erase command to disable it, which will wipe the entire flash.
    ```bash
    # Example: Mass erase to disable RDP Level 1
    openocd -f stm32u5.cfg -c "init; reset halt; stm32u5 unlock 0; exit"
    ```
3.  Flash the known-good bootloader binary to its correct address.
4.  Flash a valid, signed application binary to the primary slot.
5.  Reset the device. It should now boot normally. The device will need to be re-provisioned with the Verifier.

---

## 10. Lessons Learned Log

### Incident #1: Mass Attestation Failure After Daylight Saving Time

*   **Date:** 2025-03-10
*   **Impact:** 500 devices in the field failed their hourly attestation check simultaneously.
*   **Root Cause:** The `AttestationEvidence` included a Unix timestamp. The devices were using local time, while the Verifier was using UTC. When DST changed, the one-hour mismatch caused the Verifier to reject the evidence as "stale".
*   **Fix:** Firmware was updated to use UTC time exclusively for all protocol-level timestamps. The Verifier's freshness check was temporarily relaxed to allow devices to update.
*   **Lesson:** **Protocols must ALWAYS use UTC.** Never use local time for distributed systems.

---

**Next Phase:** [[16_Final_Packaging_and_Pitch.md|Phase 16: Final Packaging & Pitch]]
