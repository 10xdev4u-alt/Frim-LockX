---
title: 'Phase 02: Threat Modeling & Attack Surface'
project: FIRM-LOCK
phase: 02
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Security
  - ThreatModeling
  - STRIDE
status: DRAFT
dependencies: "[[01_Research.md]]"
next: "[[03_Architecture.md]]"
---

# Phase 02: Threat Modeling & Attack Surface

> **Objective:** To systematically identify, analyze, and prioritize potential threats to the FIRM-LOCK system. This phase defines the security landscape we must build our defenses within, translating abstract risks into concrete security requirements.
> **Duration:** 2 Days
> **Deliverables:**
> - A comprehensive STRIDE analysis of all system components.
> - Detailed attack scenarios with adversary profiles and impact assessments.
> - A Data Flow Diagram (DFD) illustrating trust boundaries.
> - A mitigation strategy matrix to guide architectural decisions.
> - A formal list of security requirements.

---

## Table of Contents
1. [[#1. System Overview & Trust Boundaries]]
2. [[#2. STRIDE Threat Model]]
3. [[#3. Detailed Attack Scenarios]]
4. [[#4. Attack Tree: Firmware Compromise]]
5. [[#5. Mitigation Strategy Matrix]]
6. [[#6. Formal Security Requirements]]

---

## 1. System Overview & Trust Boundaries

Before analyzing threats, we must define the system's components and the boundaries between trusted and untrusted zones. The Trusted Computing Base (TCB) is the set of all hardware, firmware, and software components that are critical to the system's security. If any component inside the TCB is compromised, the entire system's security fails.

### Data Flow Diagram (DFD) with Trust Boundaries

This diagram shows the primary data flows and delineates the TCB. Anything outside the dotted line is considered untrusted.

```
                                     Untrusted Network (e.g., LoRaWAN)
                                                 │
                                                 ▼
      ┌──────────────────────────────────────────────────────────────────────────┐
      │ Untrusted External World                                                 │
      │                                                                          │
      │  ┌──────────────┐  Firmware Update  ┌───────────────┐ Attestation Request │
      │  │ Update Server│──────────────────▶│   Verifier    │◀────────────────────│ Field Operator
      │  └──────────────┘                   └───────┬───────┘                     │
      │                                             │                             │
      └─────────────────────────────────────────────┼─────────────────────────────┘
                                                    │ Challenge/Response
                                                    │
  ============================================== TCB Boundary ==============================================
                                                    │
      ┌─────────────────────────────────────────────┼─────────────────────────────┐
      │ FIRM-LOCK Device                            │                             │
      │                                             │                             │
      │   ┌──────────────────┐        Evidence      │                             │
      │   │ Attestation Agent├──────────────────────┘                             │
      │   │ (in Application) │                                                    │
      │   └────────┬─────────┘                                                    │
      │            │ Quote PCRs                                                   │
      │            ▼                                                              │
      │   ┌──────────────────┐                                                    │
      │   │  Bootloader      │◀───────────┐(Reads)                                │
      │   │  (MCUboot)       │            │                                       │
      │   └────────┬─────────┘            │                                       │
      │            │ Measures App         │                                       │
      │            ▼                      │                                       │
      │   ┌──────────────────┐            │                                       │
      │   │ Application FW   │────────────┘                                       │
      │   └──────────────────┘                                                    │
      │                                                                           │
      │----------------- Hardware Trust Boundary (CPU Domain) --------------------│
      │                                                                           │
      │   ┌──────────────────┐   Sign/Verify,   ┌──────────────────┐              │
      │   │   CPU Crypto     │◀────────────────▶│  Secure Element  │              │
      │   │   (TrustZone)    │     (I2C)        │   (ATECC608A)    │              │
      │   └──────────────────┘                  └──────────────────┘              │
      │                                                                           │
      └───────────────────────────────────────────────────────────────────────────┘
```

**Trust Zones:**
*   **Untrusted:** The external network, the verifier (which must treat device data as untrusted until verified), the update server, and any human operators.
*   **Trusted Computing Base (TCB):** The device's secure element, the bootloader, the attestation agent, and the hardware-protected mechanisms (e.g., ARM TrustZone). The application firmware is measured by the TCB but is not inherently part of it.

---

## 2. STRIDE Threat Model

We apply the STRIDE methodology to each critical component of the FIRM-LOCK device.

### Component: Secure Bootloader (MCUboot)
*   **Spoofing:** An attacker flashes a malicious bootloader that appears legitimate but bypasses security checks.
*   **Tampering:** An attacker modifies the bootloader on the flash chip to disable signature verification or PCR measurements.
*   **Repudiation:** A failed boot attempt (due to a bad signature) leaves no auditable trace, allowing an attacker to probe the system without detection.
*   **Information Disclosure:** A vulnerability in the bootloader's logging mechanism leaks memory addresses or cryptographic material (e.g., the public key).
*   **Denial of Service (DoS):** An attacker corrupts the bootloader, "bricking" the device and preventing it from loading any firmware.
*   **Elevation of Privilege:** A bug in the signature verification logic allows a specially crafted, invalid signature to be accepted, causing the bootloader to jump to malicious code with full device privileges.

### Component: Application Firmware
*   **Spoofing:** A malicious application masquerades as a legitimate one. This is what the bootloader is designed to prevent.
*   **Tampering:** An attacker modifies the application firmware after it has been measured by the bootloader (e.g., via a memory corruption vulnerability). This is a post-boot threat.
*   **Repudiation:** The application performs a sensitive action (e.g., changing a configuration setting) but fails to log it, making it impossible to audit.
*   **Information Disclosure:** The application leaks sensitive data (e.g., sensor readings, cryptographic nonces) over the network or a debug interface.
*   **Denial of Service (DoS):** A bug in the application (e.g., null pointer dereference) causes it to crash, leading to a watchdog reset and a potential boot loop.
*   **Elevation of Privilege:** A vulnerability in the application allows an attacker to gain control over the hardware or access functions reserved for the TCB.

### Component: Secure Element (ATECC608A)
*   **Spoofing:** An attacker replaces the genuine SE with a counterfeit chip that provides fake signatures, or an emulator that attempts to mimic its behavior.
*   **Tampering:** An attacker uses physical methods (e.g., fault injection, microprobing) to try to extract the private keys stored within the SE.
*   **Repudiation:** The device signs a piece of evidence, but an attacker claims the signature is invalid due to a faulty SE, attempting to undermine trust in the system.
*   **Information Disclosure:** A side-channel attack (e.g., power analysis) leaks information about the cryptographic keys during a signing operation.
*   **Denial of Service (DoS):** An attacker sends malformed I2C commands to the SE, causing it to lock up until the next power cycle. Repeatedly using the `Lock` command could permanently disable key slots.
*   **Elevation of Privilege:** Not directly applicable, as the SE is a hardware peripheral. The threat is an external actor gaining access to its privileged functions (e.g., key generation, signing).

### Component: Attestation Protocol
*   **Spoofing:** An attacker records a valid attestation response and replays it later to a verifier (Replay Attack). This would make a compromised device appear healthy.
*   **Tampering:** An attacker intercepts a challenge from the verifier and modifies it before it reaches the device, or modifies the evidence from the device before it reaches the verifier.
*   **Repudiation:** A device provides valid evidence of compromise, but the verifier or network infrastructure claims it was never received.
*   **Information Disclosure:** The attestation evidence contains unnecessary information that reveals details about the device's software or location.
*   **Denial of Service (DoS):** An attacker floods a device with attestation challenges, forcing it to spend all its CPU time and battery performing cryptographic operations.
*   **Elevation of Privilege:** An attacker finds a flaw in the verifier's policy logic, allowing evidence that should indicate a compromised state to be accepted as valid.

---

## 3. Detailed Attack Scenarios

| Scenario # | 1: Supply Chain Firmware Backdoor                                                                                             |
|------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Adversary**| Nation-State Actor or well-funded criminal organization.                                                                      |
| **Vector**   | Compromise a contract manufacturer or a developer's build environment.                                                        |
| **Prereqs**  | Significant resources, insider access, or ability to execute a sophisticated phishing/malware campaign.                       |
| **Impact**   | **Catastrophic.** A backdoor is inserted into the signed, "golden" firmware image before it's even programmed. The device is compromised from day one. |
| **Likelihood**| **Low**, but **Very High** impact.                                                                                             |

| Scenario # | 2: Physical Debug Port Access ("Evil Maid")                                                                                   |
|------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Adversary**| Insider, field technician, or moderately skilled attacker with physical access.                                               |
| **Vector**   | Connect a JTAG/SWD debugger to exposed pins on the PCB.                                                                       |
| **Prereqs**  | Physical access to the device, a $50 debugger, and knowledge of the MCU's architecture.                                       |
| **Impact**   | **High.** Attacker can dump the entire firmware, read memory, and bypass the secure boot process entirely if debug fuses are not blown. |
| **Likelihood**| **Medium** for field-deployed devices.                                                                                        |

| Scenario # | 3: Malicious OTA Update Injection                                                                                             |
|------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Adversary**| Network-based attacker.                                                                                                       |
| **Vector**   | Intercept or spoof the communication channel (e.g., LoRaWAN) to send a malicious firmware update to the device.               |
| **Prereqs**  | Ability to transmit on the correct frequency and knowledge of the update protocol.                                            |
| **Impact**   | **Low (if secure boot is working).** The device's bootloader should reject the update because it lacks a valid signature. The real impact is a temporary DoS. |
| **Likelihood**| **Medium.**                                                                                                                   |

| Scenario # | 4: Downgrade Attack                                                                                                           |
|------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Adversary**| Network-based attacker who has obtained a copy of an old, vulnerable (but validly signed) firmware version.                 |
| **Vector**   | Send the old firmware as a "new" update.                                                                                      |
| **Prereqs**  | Access to an old firmware binary.                                                                                             |
| **Impact**   | **High.** The device is rolled back to a state with a known, exploitable vulnerability.                                       |
| **Likelihood**| **Medium.**                                                                                                                   |

| Scenario # | 5: Side-Channel Attack on Secure Element                                                                                      |
|------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Adversary**| Sophisticated, well-resourced attacker with physical access.                                                                  |
| **Vector**   | Measure the device's power consumption or electromagnetic emissions during cryptographic operations.                          |
| **Prereqs**  | Oscilloscope, EM probes, specialized analysis software, and significant expertise.                                            |
| **Impact**   | **Critical.** If successful, the attacker could derive the device's private key, allowing them to clone the device or sign malicious messages. |
| **Likelihood**| **Low**, but a known risk for high-assurance systems. The ATECC608A has internal countermeasures against this.                |

---

## 4. Attack Tree: Firmware Compromise

This tree illustrates the different paths an attacker could take to achieve the goal of running unauthorized code on the device.

```
GOAL: Execute Unauthorized Code on Device
     │
     ├─ OR ─ Bypass Secure Boot
     │      │
     │      ├─ AND ─ Find flaw in bootloader signature verification logic
     │      │       └─ Craft malicious firmware that exploits the flaw
     │      │
     │      └─ AND ─ Gain physical access
     │              └─ Use JTAG/SWD to overwrite bootloader/app
     │                 └─ (Requires debug interface to be enabled)
     │
     ├─ OR ─ Exploit a Vulnerability in the Application
     │      │
     │      ├─ AND ─ Find a memory corruption bug (e.g., buffer overflow)
     │      │       └─ Send malicious input over network/USB to trigger bug
     │      │
     │      └─ AND ─ Find a logic bug that allows arbitrary code execution
     │
     └─ OR ─ Compromise the Update Process
            │
            ├─ AND ─ Steal the code-signing private key
            │       └─ Sign malicious firmware and send as a legitimate update
            │
            └─ AND ─ Find an old, vulnerable firmware version
                    └─ Perform a downgrade attack
                       └─ (Requires no anti-rollback mechanism)
```

---

## 5. Mitigation Strategy Matrix

| Threat ID | Threat Description                      | Severity | Mitigation Strategy                                                              | Residual Risk | Verification Method                               |
|-----------|-----------------------------------------|----------|----------------------------------------------------------------------------------|---------------|---------------------------------------------------|
| T01       | Malicious bootloader (Spoofing)         | **Critical** | Use MCU's one-time-programmable (OTP) memory or fuse bits to lock the initial bootloader (Stage 1). | Low           | Attempt to re-flash bootloader; verify it fails.  |
| T02       | Physical debug access (EoP)             | **Critical** | Permanently disable debug interfaces (JTAG/SWD) in production units via fuse bits (e.g., RDP Level 2 on STM32). | Low           | Attempt to connect a debugger to a production unit. |
| T03       | Downgrade attack                        | **High**     | Implement anti-rollback counter in the secure element. The bootloader must reject any firmware with a version number lower than the current one. | Low           | Attempt to flash an older, validly signed firmware. |
| T04       | Attestation replay attack (Spoofing)    | **High**     | Include a unique, unpredictable nonce (provided by the verifier) in every attestation challenge. The device must sign this nonce as part of the evidence. | Low           | Send the same challenge twice; verify the second response is processed correctly but may be flagged. Send an old response to a new challenge; verify it is rejected. |
| T05       | Key extraction from SE (Info. Disc.)    | **High**     | Use a certified secure element (like ATECC608A) with internal countermeasures against physical and side-channel attacks. Private keys never leave the chip. | Medium        | This is difficult to verify without a security lab. We rely on the chip vendor's claims and certifications. |
| T06       | DoS via attestation requests            | **Medium**   | Implement rate-limiting on the device for how often it will perform a full attestation. | Low           | Flood the device with requests and verify it remains responsive. |
| T07       | Boot loop due to bad application (DoS)  | **Medium**   | Implement a boot-failure counter. If the app fails to boot N times, the bootloader automatically rolls back to a "golden" factory image. | Low           | Flash a deliberately broken application; verify the device recovers after N reboots. |

---

## 6. Formal Security Requirements

Based on the threat model, the following formal requirements are established. These will guide the architecture and design in subsequent phases.

### SHALL (Mandatory) Requirements
*   **SR-01:** The system **SHALL** use a hardware-based root of trust for storing cryptographic identity keys.
*   **SR-02:** Private keys used for device identity **SHALL** be non-exportable from the hardware root of trust.
*   **SR-03:** The bootloader **SHALL** cryptographically verify the signature of all application firmware before execution.
*   **SR-04:** The system **SHALL** refuse to execute any firmware that does not have a valid signature.
*   **SR-05:** The system **SHALL** implement an anti-rollback mechanism to prevent the loading of older, vulnerable firmware versions.
*   **SR-06:** The production firmware **SHALL** permanently disable all hardware debug interfaces (e.g., JTAG, SWD).
*   **SR-07:** The attestation evidence **SHALL** include a cryptographic measurement of all boot-stage firmware and application code (Measured Boot).
*   **SR-08:** The attestation protocol **SHALL** use a server-provided nonce to protect against replay attacks.

### SHOULD (Recommended) Requirements
*   **SR-09:** The system **SHOULD** implement a watchdog timer to recover from application-level crashes.
*   **SR-10:** The bootloader **SHOULD** support a fail-safe recovery mechanism (e.g., rollback to a golden image) if the main application fails to boot multiple times.
*   **SR-11:** All external communication **SHOULD** be encrypted.
*   **SR-12:** The device **SHOULD** minimize the information exposed in log messages in production builds.

---

**Next Phase:** [[03_Architecture.md|Phase 03: System Architecture (HLD)]]
