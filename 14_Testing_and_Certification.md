--- 
title: 'Phase 14: Testing & Certification'
project: FIRM-LOCK
phase: 14
date: 2025-12-15
tags:
  - 'FIRM-LOCK'
  - 'Testing'
  - 'Security'
  - 'Certification'
  - 'Pentesting'
status: DRAFT
dependencies: "[[13_System_integration.md]]"
next: "[[15_Troubleshooting_and_Failure_Modes.md]]"
---

# Phase 14: Testing & Certification

> **Objective:** To define and execute a comprehensive testing strategy that validates the security, reliability, and compliance of the entire FIRM-LOCK system. This phase moves beyond "does it work?" to "is it robust, is it secure, and can we prove it to a third party?"
> **Duration:** 4 Days
> **Deliverables:**
> - A formal testing strategy and philosophy.
> - A detailed scope and plan for third-party security penetration testing.
> - A test plan for environmental and reliability validation.
> - A clear path and checklist for achieving necessary regulatory and security certifications.

---

## Table of Contents
1. [[#1. Testing Strategy & Philosophy]]
2. [[#2. Security Penetration Testing Plan]]
3. [[#3. Environmental & Reliability Testing]]
4. [[#4. Compliance & Certification Plan]]

---

## 1. Testing Strategy & Philosophy

Our testing approach is based on the well-established "Testing Pyramid." This model ensures we have a robust and efficient testing process by focusing effort at the right levels.

### The Testing Pyramid
```
      ▲
     / \
    /E2E\    (End-to-End / Manual / Pentesting - Slow, Expensive)
   ┌─────┐
  /INTEGR\   (Integration Tests - e.g., Firmware <-> Backend)
 ├─────────┤
/   UNIT   \  (Unit Tests - Fast, Cheap, Numerous)
└────────────┘
```

*   **Unit Tests (Foundation):**
    *   **Scope:** Test individual functions and modules in isolation (e.g., the `pcr_extend` function, the `build_attestation_evidence` logic).
    *   **Owner:** Development Team.
    *   **Goal:** To catch bugs early and ensure each component works correctly before it's integrated. These tests should be fast and numerous, running on every commit.

*   **Integration Tests (Middle Layer):**
    *   **Scope:** Test the interaction between two or more components (e.g., does the Attestation Agent correctly use the Secure Element Driver? Does the MCUboot bootloader correctly jump to the application?).
    *   **Owner:** Development Team.
    *   **Goal:** To find issues at the interfaces between components. These are run less frequently than unit tests, perhaps on every pull request.

*   **End-to-End (E2E) Tests (Peak):**
    *   **Scope:** Test the entire system flow, as described in [[13_System_Integration.md|Phase 13]]. This involves the real hardware, the LoRaWAN network, and the live backend.
    *   **Owner:** QA Team / Integration Team.
    *   **Goal:** To validate that the system as a whole meets the user's requirements. These tests are slow and brittle, so they should be used judiciously to cover the most critical user journeys (e.g., a full attestation cycle).

**Automation First:** Wherever possible, tests will be automated and integrated into a CI/CD pipeline. This includes on-target tests that run on the actual hardware prototype.

---

## 2. Security Penetration Testing Plan

No system is secure until it has been challenged by an adversary. We will engage a third-party security consulting firm to perform a white-box penetration test.

*   **Objective:** To have independent experts attempt to break the security of the FIRM-LOCK system, identify vulnerabilities that we missed, and provide recommendations for hardening.
*   **Scope:** The test will cover all aspects of the system.
    *   **Hardware:** The physical device itself.
    *   **Firmware:** The bootloader and application binaries.
    *   **Protocol:** The LoRaWAN-based attestation protocol.
    *   **Backend:** The Verifier web API and database.

### 2.1 Hardware Attack Vectors
*   **Goal:** Attempt to compromise the device physically.
*   **Test Cases:**
    *   **JTAG/SWD Access:** Attempt to attach a debugger to a "production" device where debug ports should be fused off. **Success = No connection possible.**
    *   **Fault Injection ("Glitching"):** Use voltage or clock glitching on the MCU during the bootloader's signature verification process to try to cause it to skip the check and jump to an invalid image. **Success = Bootloader halts or detects the glitch.**
    *   **Side-Channel Analysis:** Monitor the power consumption of the ATECC608A during a signing operation to see if any information about the private key can be leaked. **Success = No discernible correlation between power trace and key bits.**
    *   **Flash Readout:** Attempt to use a programmer to read the protected flash memory containing the bootloader. **Success = Read operation fails or returns all zeros.**

### 2.2 Firmware Attack Vectors
*   **Goal:** Find vulnerabilities in the device's software.
*   **Test Cases:**
    *   **Reverse Engineering:** Provide the testers with the `app-signed.bin` file. They will attempt to reverse-engineer it using tools like Ghidra to find hardcoded secrets or logic flaws. **Success = No critical secrets are found (as they are in the SE).**
    *   **MCUboot Bypass:** Attempt to find a logical vulnerability in our MCUboot configuration that would allow a specially crafted image to bypass verification.
    *   **Memory Corruption:** Attempt to send malformed LoRaWAN packets to exploit potential buffer overflows or other memory bugs in the packet parsing code. **Success = Device correctly handles or discards malformed packets without crashing.**

### 2.3 Backend Attack Vectors
*   **Goal:** Compromise the Verifier backend.
*   **Test Cases:**
    *   **Standard Web Pentest:** Testers will perform a full OWASP Top 10 assessment of the API, checking for SQL Injection, broken access control etc.
    *   **API Fuzzing:** Send malformed `AttestationEvidence` to the `/api/evidence` endpoint to check for parsing errors or crashes in the Attestation Engine.
    *   **Race Conditions:** Attempt to use the same nonce in multiple simultaneous requests to see if the session management can be bypassed. **Success = Only the first request is processed; subsequent ones are rejected.**

---

## 3. Environmental & Reliability Testing

This testing ensures the device will survive in its intended operating environment.

*   **Temperature Cycling:**
    *   **Procedure:** Place several devices in a thermal chamber. Cycle the temperature between -20°C and +60°C while the devices are performing continuous attestation.
    *   **Success Criteria:** Devices operate correctly across the entire temperature range. LoRa frequency remains stable enough for communication.
*   **Power Cycling Stress Test:**
    *   **Procedure:** A test rig will be built to programmatically cut power to the device at random intervals during the firmware update process. This will be repeated hundreds of times.
    *   **Success Criteria:** The device never bricks. After each power loss, it either boots successfully into the new firmware (if the swap completed) or the old firmware (if it was interrupted).
*   **Long-Duration Soak Test:**
    *   **Procedure:** Run ten devices continuously for 30 days. The devices will perform an attestation every hour and a full firmware update once per day.
    *   **Success Criteria:** All ten devices are still fully operational after 30 days. Monitor for signs of memory leaks (heap size growing over time) or performance degradation.

---

## 4. Compliance & Certification Plan

Formal certification provides a stamp of approval that is essential for commercial and government customers.

### 4.1 Regulatory Compliance (Mandatory for Sale)
*   **FCC Part 15 (US) / CE RED (EU):**
    *   **Path:** We will engage a certified EMI/EMC testing lab. They will test the final, packaged device for radiated and conducted emissions.
    *   **Preparation:** The design already incorporates best practices (ground planes, filtering) to increase the likelihood of passing. The use of a pre-certified LoRa module simplifies this process.
*   **RoHS (Restriction of Hazardous Substances):**
    *   **Path:** This is primarily a supply chain management task.
    *   **Action:** Verify that every single component on the BOM from [[04_BOM.md|Phase 04]] is RoHS compliant by obtaining compliance certificates from the suppliers.

### 4.2 Security Certifications (Voluntary but Recommended)
Achieving one of these provides a significant competitive advantage and third-party validation of our security claims.
*   **Target: PSA Certified Level 1**
    *   **What it is:** A security assessment based on a questionnaire that covers best practices for IoT security. It's a foundational certification.
    *   **Path:** We will answer the questionnaire, referencing our design documents (Phases 02-11) as evidence that we meet the requirements for secure boot, secure update, crypto, etc.
*   **Aspirational Target: SESIP (Security Evaluation Standard for IoT Platforms) Level 3**
    *   **What it is:** A more rigorous, lab-based evaluation of the product's security functions. It involves a third-party lab performing vulnerability analysis and penetration testing against the defined security claims.
    *   **Path:** This would be a post-v1.0 goal. It requires a significant investment in time and resources but provides a high degree of assurance. Our design, with its hardware root of trust and measured boot, is architected to be capable of passing this level of evaluation.

---

**Next Phase:** [[15_Troubleshooting_and_Failure_Modes.md|Phase 15: Troubleshooting & Failure Modes]]
