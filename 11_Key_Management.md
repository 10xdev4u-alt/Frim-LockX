---
title: 'Phase 11: Key Management'
project: FIRM-LOCK
phase: 11
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Security
  - Cryptography
  - PKI
  - HSM
status: DRAFT
dependencies: "[[10_Secure_Update.md]]"
next: "[[12_Verifier_Backend.md]]"
---

# Phase 11: Key Management

> **Objective:** To define a comprehensive lifecycle management strategy for all cryptographic keys used in the FIRM-LOCK ecosystem, ensuring their security from generation to revocation. A flawed key management strategy can undermine even the strongest cryptographic algorithms.
> **Duration:** 2 Days
> **Deliverables:**
> - An inventory and taxonomy of all cryptographic keys.
> - A detailed lifecycle plan for device-level and vendor-level keys.
> - A proposed Public Key Infrastructure (PKI) and chain of trust.
> - "Break glass" procedures for key compromise and rotation.

---

## Table of Contents
1. [[#1. Key Inventory & Taxonomy]]
2. [[#2. Device Identity Key Lifecycle]]
3. [[#3. Vendor Key Management (Firmware & Manifest Signing)]]
4. [[#4. Public Key Infrastructure (PKI) & Chain of Trust]]
5. [[#5. Key Management Procedures in the Field]]

---

## 1. Key Inventory & Taxonomy

To manage our keys, we must first identify them. The FIRM-LOCK ecosystem relies on several distinct keys, each with a specific role and security requirement.

| Key Name                    | Type       | Scope          | Storage Location                               | Use Case                               |
|:----------------------------|:-----------|:---------------|:-----------------------------------------------|:---------------------------------------|
| **Device Private Key**      | Asymmetric (ECC P-256) | Unique per Device | **ATECC608A Secure Element (Slot 0)**        | Signing attestation evidence.          |
| **Device Public Key**       | Asymmetric (ECC P-256) | Unique per Device | Verifier Database, Device Certificate        | Verifying attestation evidence.        |
| **Firmware Signing Key**    | Asymmetric (ECC P-256) | Per Product Line  | **Offline HSM or Air-gapped Machine**        | Signing firmware binaries for MCUboot. |
| **Manifest Signing Key**    | Asymmetric (ECC P-256) | Per Product Line  | **Online HSM (CI/CD Integration)**           | Signing SUIT manifests for OTA updates.|
| **Root CA Key**             | Asymmetric (RSA-4096)  | Vendor-wide     | **Offline, Air-gapped Root HSM**             | The ultimate root of trust for the PKI.|

**Key Concepts:**
*   **Device-Level Keys:** Unique to each physical device. A compromise only affects that single device.
*   **Class-Level Keys:** Shared across a whole product line (e.g., all FIRM-LOCK v1 devices). A compromise is catastrophic, affecting all devices. These require the highest level of protection.

---

## 2. Device Identity Key Lifecycle

This lifecycle covers the unique key pair that gives each FIRM-LOCK device its identity.

### 2.1 Provisioning (Key Generation)
This is a critical one-time process during manufacturing.
1.  **Generation:** The `atcab_genkey()` command is sent to the ATECC608A. The chip generates a new, random ECC P-256 private key internally and stores it in a pre-configured slot (e.g., Slot 0). **The private key never leaves the chip.**
2.  **Public Key Extraction:** The corresponding public key is derived by the chip and returned to the manufacturing station.
3.  **Certificate Creation:** The manufacturing station creates a device certificate, embedding the public key, serial number, and other identifiers. This certificate is then signed by a dedicated "Provisioning CA" key (which chains up to the vendor's Root CA).
4.  **Registration:** The signed device certificate is uploaded to the Verifier's device database, establishing a trusted record of the device's identity.
5.  **Locking:** The ATECC608A's configuration zone and the key slot are permanently locked to prevent any further modification. The `PrivateKey` flag for the slot is set, ensuring the key can be used for signing but can never be read.

### 2.2 Storage & Usage
*   **Storage:** The private key is physically stored within the secure boundary of the ATECC608A. The chip itself has hardware countermeasures against physical probing, fault injection, and side-channel analysis.
*   **Usage:** When the firmware needs to sign attestation evidence, it sends the 32-byte hash to the ATECC608A via the `atcab_sign()` command. The chip performs the ECDSA signature operation internally and returns the 64-byte signature. The key remains protected.

### 2.3 Revocation
A device might need to be revoked if it is lost, stolen, or known to be compromised.
*   **Process:** Revocation is managed entirely on the Verifier side. There is no way to "delete" the private key from a deployed device.
*   **Implementation:** The Verifier maintains a Certificate Revocation List (CRL) or an online status check. When processing attestation evidence, the Verifier first checks if the device's identity certificate has been blacklisted. If so, it rejects the evidence, even if the signature is valid.

---

## 3. Vendor Key Management (Firmware & Manifest Signing)

These class-level keys are the "crown jewels" of the system. Their compromise would allow an attacker to sign malicious firmware that would be trusted by every device.

### 3.1 Generation
*   **Method:** All vendor-level keys **MUST** be generated on a system that is not connected to the internet.
    *   **Gold Standard:** A FIPS 140-2 Level 3 (or higher) Hardware Security Module (HSM).
    *   **Alternative:** A dedicated, permanently air-gapped laptop running a trusted OS (e.g., Tails Linux) to generate keys using OpenSSL.
*   **Ceremony:** Key generation should be a formal, documented "ceremony" involving multiple trusted individuals to prevent a single person from having unauthorized access.

### 3.2 Storage & Access Control
*   **Root CA Key:** This key should be stored on an offline HSM or encrypted on multiple USB drives stored in separate physical safes (Shamir's Secret Sharing). It should only be brought online to sign a new intermediate CA certificate, an extremely rare event.
*   **Firmware/Manifest Signing Private Keys:**
    *   **HSM Integration:** The best practice is to integrate the CI/CD pipeline with an online HSM (e.g., AWS CloudHSM, Azure Dedicated HSM). The CI/CD runner can request the HSM to sign a hash, but it never has direct access to the key itself.
    *   **Air-gapped Signing:** For smaller operations, the compiled binary can be transferred (via a read-only medium) to an air-gapped "signing machine" where the key is stored. The signed binary is then transferred back. This is manual but highly secure.
*   **NEVER:** Private keys must **NEVER** be stored in source control, on a developer's laptop, in a wiki, or in any shared server filesystem.

### 3.3 Key Rotation
If a key is suspected of compromise or has reached the end of its planned cryptographic lifetime, it must be rotated. This is a delicate, multi-stage process.

**Key Rotation Sequence:**
```
  Time ────────────────────────────────────────────────────────────>
  
  Phase A: Normal Operation
  - Devices trust Public Key A.
  - Firmware is signed with Private Key A.
  
  Phase B: Introduce New Key
  - Generate new keypair (Private/Public Key B).
  - Update bootloader to trust BOTH Public Key A AND Public Key B.
  - Deploy this new bootloader, signed with Private Key A.
  
  Phase C: Transition to New Key
  - Start signing all new firmware with Private Key B.
  - Devices running the updated bootloader will accept this new firmware.
  
  Phase D: Deprecate Old Key
  - Once all devices are updated, update the bootloader again to ONLY trust Public Key B.
  - Deploy this final bootloader, signed with Private Key B.
  - Public Key A is now fully deprecated.
```

---

## 4. Public Key Infrastructure (PKI) & Chain of Trust

A PKI formalizes the trust relationships between all parties.

```
                                  ┌──────────────────┐
                                  │ Root CA          │ (Offline)
                                  │ (FIRM-LOCK Corp.)│
                                  └────────┬─────────┘
                                           │ Signs
                        ┌──────────────────┴──────────────────┐
                        │                                     │
                        ▼                                     ▼
              ┌──────────────────┐                  ┌──────────────────┐
              │  Vendor CA       │ (Online)         │  Provisioning CA │ (Online)
              │(Signs Firmware)  │                  │(Signs Devices)   │
              └────────┬─────────┘                  └────────┬─────────┘
                       │ Signs                               │ Signs
                       ▼                                     ▼
             ┌──────────────────┐                  ┌──────────────────┐
             │ Firmware Signing │                  │ Device Identity  │
             │ Certificate      │                  │ Certificate      │
             │ (Embedded in     │                  │ (Sent to Verifier│
             │  Bootloader)     │                  │  Database)       │
             └──────────────────┘                  └──────────────────┘
```

*   **Trust Anchor:** The device's bootloader has the **Vendor CA's public key** baked into it as its ultimate trust anchor for firmware updates.
*   **Verification Path:** When a device receives a new firmware update, it can verify the signature all the way up the chain: Firmware Signature -> Firmware Signing Certificate -> Vendor CA.

---

## 5. Key Management Procedures in the Field

### 5.1 Device Re-Provisioning
If a device's Secure Element fails in the field, the entire device must be replaced or returned for secure re-provisioning.
*   **Procedure:**
    1.  The old device is securely decommissioned.
    2.  A new device is provisioned with a **new, unique identity key**.
    3.  The Verifier database is updated to associate the physical asset (e.g., "Pump Station 12") with the new device's identity and revokes the old one.
*   **Rationale:** It is insecure to "clone" an identity from an old SE to a new one. This would break the fundamental guarantee of a unique hardware-bound key.

### 5.2 "Break Glass" Procedure for Key Compromise
If a vendor-level signing key (e.g., the Firmware Signing Key) is compromised, immediate action is required.
1.  **Containment:** Immediately revoke the compromised key's certificate via the Vendor CA.
2.  **Generate New Key:** Perform an emergency key generation ceremony to create a new keypair.
3.  **Patch Bootloader:** Create a new bootloader that trusts the **new** public key and revokes the old one.
4.  **Emergency Rollout:** Sign this new bootloader with the **compromised** key (this is likely the only way old devices will accept it). This is the "one last signature". The goal is to deploy the new trust anchor as quickly as possible.
5.  **Transition:** Once the new bootloader is deployed, all subsequent firmware must be signed with the new, secure private key.
6.  **Investigation:** Conduct a full post-mortem to determine how the key was compromised and improve security procedures.

---

**Next Phase:** [[12_Verifier_Backend.md|Phase 12: Verifier Backend]]
