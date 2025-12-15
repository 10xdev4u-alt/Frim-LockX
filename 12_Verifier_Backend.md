---
title: 'Phase 12: Verifier Backend'
project: FIRM-LOCK
phase: 12
date: 2025-12-15
tags:
  - FIRM-LOCK
  - Backend
  - API
  - Verifier
  - FastAPI
status: DRAFT
dependencies: "[[11_Key_Management.md]]"
next: "[[13_System_Integration.md]]"
---

# Phase 12: Verifier Backend

> **Objective:** To design the server-side application that manages devices, initiates attestation challenges, and appraises the evidence received from FIRM-LOCK devices to render a final trust verdict.
> **Duration:** 3 Days
> **Deliverables:**
> - A complete software architecture for the Verifier backend.
> - A defined technology stack and database schema.
> - A full RESTful API specification for device interaction and management.
> - Detailed logic for the core Attestation Engine.

---

## Table of Contents
1. [[#1. Verifier Architecture]]
2. [[#2. Database Schema]]
3. [[#3. Attestation Engine: Core Logic]]
4. [[#4. RESTful API Specification]]
5. [[#5. User Interface & Alerting]]

---

## 1. Verifier Architecture

The Verifier is a backend service that acts as the central authority for device integrity. It needs to be scalable, secure, and provide clear interfaces for both devices and human operators.

### 1.1 Architectural Block Diagram
```
  ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
  │ Human Operator │   │ LoRaWAN Network│   │   CI/CD        │
  │ (Web Browser)  │   │ Server / IoT   │   │   Pipeline     │
  └───────┬────────┘   └───────┬────────┘   └───────┬────────┘
          │ (HTTPS)            │ (MQTT/HTTP)        │ (HTTPS)
          ▼                    ▼                    ▼
┌───────────────────────────────────────────────────────────────┐
│ Verifier Backend Service                                      │
│                                                               │
│   ┌───────────────────────────────────────────────────────┐   │
│   │ RESTful API Gateway (FastAPI)                         │   │
│   │ (Authentication, Request Validation, Routing)         │   │
│   └───────────────────────┬───────────────────────────────┘   │
│                           │                                   │
│           ┌───────────────┴───────────────┐                   │
│           │                               │                   │
│           ▼                               ▼                   │
│   ┌────────────────┐            ┌──────────────────┐          │
│   │ Attestation    │◀───────────▶│ Session & Nonce  │          │
│   │ Engine         │            │ Store (Redis)    │          │
│   └────────────────┘            └──────────────────┘          │
│           │                                                   │
│           │ Appraises against...                              │
│           ▼                                                   │
│   ┌───────────────────────────────────────────────────────┐   │
│   │ Persistence Layer (PostgreSQL Database)             │   │
│   │ ┌────────────────┐  ┌────────────────┐  ┌───────────┐ │   │
│   │ │ Device         │  │ Appraisal      │  │ Audit Log │ │   │
│   │ │ Registry       │  │ Policy Store   │  │           │ │   │
│   │ └────────────────┘  └────────────────┘  └───────────┘ │   │
│   └───────────────────────────────────────────────────────┘   │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack
*   **Language/Framework:** **Python 3 with FastAPI**.
    *   **Rationale:** FastAPI provides extremely high performance (comparable to Node.js and Go) and automatically generates interactive API documentation (via OpenAPI/Swagger). Python's extensive libraries for cryptography and database access make it ideal for rapid development.
*   **Database:** **PostgreSQL**.
    *   **Rationale:** The relationships between devices, firmware versions, and appraisal policies are highly relational. PostgreSQL's transactional integrity and robust feature set are perfect for this model.
*   **Session/Nonce Store:** **Redis**.
    *   **Rationale:** Attestation sessions are short-lived. Storing the generated nonces in a fast in-memory database like Redis is far more efficient than writing them to the main SQL database. Redis's built-in key expiration feature is perfect for automatically cleaning up timed-out sessions.

---

## 2. Database Schema

The PostgreSQL database is structured into three primary tables.

```
Table: devices
- device_id (UUID, Primary Key)
- serial_number (TEXT, Unique)
- device_certificate (TEXT)
- registered_at (TIMESTAMP)
- last_seen (TIMESTAMP)
- current_status (VARCHAR: TRUSTED, UNTRUSTED, UNKNOWN)
- current_firmware_version (VARCHAR)

Table: appraisal_policies
- policy_id (SERIAL, Primary Key)
- firmware_version (VARCHAR, Unique)
- golden_pcr_0 (BYTEA)
- golden_pcr_1 (BYTEA)
- golden_pcr_2 (BYTEA)
- golden_pcr_3 (BYTEA)
- minimum_security_counter (INTEGER)
- created_at (TIMESTAMP)

Table: audit_log
- log_id (SERIAL, Primary Key)
- device_id (UUID, Foreign Key -> devices.device_id)
- timestamp (TIMESTAMP)
- event_type (VARCHAR: ATTEST_PASS, ATTEST_FAIL, UPDATE_START, etc.)
- details (JSONB)
```

**Relationships:**
*   Each entry in `devices` represents a single physical device provisioned in the factory.
*   Each entry in `appraisal_policies` represents a known-good firmware version with its "golden" measurements.
*   The `audit_log` provides a forensic history of every significant event for each device.

---

## 3. Attestation Engine: Core Logic

The Attestation Engine is the heart of the Verifier. It implements the Verifier-side state machine from [[09_Attestation_Protocol.md|Phase 09]].

### Pseudocode for `appraise_evidence` function
```python
def appraise_evidence(evidence: AttestationEvidence) -> (bool, str):
    # 1. Nonce Verification (Freshness)
    stored_nonce = redis.get(f"session:{evidence.nonce}")
    if not stored_nonce:
        return (False, "Nonce expired or invalid (replay attack?)")
    # Consume the nonce to prevent reuse
    redis.delete(f"session:{evidence.nonce}")

    # 2. Find the Device and its Certificate
    device = db.query(Device).filter(Device.certificate == evidence.device_identity).first()
    if not device:
        return (False, "Device identity not recognized")
    if device.status == "REVOKED":
        return (False, "Device has been revoked")

    # 3. Signature Verification (Authenticity)
    public_key = extract_public_key_from_cert(device.device_certificate)
    signed_content = evidence.get_signed_portion()
    is_signature_valid = crypto.ecdsa_verify(public_key, signed_content, evidence.signature)
    if not is_signature_valid:
        return (False, "Signature validation failed")

    # 4. Policy Appraisal (Integrity)
    policy = db.query(AppraisalPolicy).filter(Policy.version == evidence.firmware_version).first()
    if not policy:
        return (False, f"No appraisal policy found for firmware version {evidence.firmware_version}")

    # 4a. PCR Check
    for i in range(4):
        if evidence.pcrs[i] != policy.golden_pcrs[i]:
            return (False, f"PCR[{i}] mismatch")

    # 4b. Anti-Rollback Check
    if evidence.security_counter < policy.minimum_security_counter:
        return (False, "Anti-rollback check failed")

    # 5. All checks passed!
    # Update device status in the database
    device.status = "TRUSTED"
    device.last_seen = now()
    device.current_firmware_version = evidence.firmware_version
    db.commit()

    return (True, "Device is TRUSTED")
```

---

## 4. RESTful API Specification

The API is the primary interface for interacting with the Verifier.

### API Interaction Diagram
```
   Operator                 Device                   API
      │                      │                        │
      │ POST /attestations/{id}
      ├───────────────────────────────────────────────▶
      │                      │                        │
      │                      │   202 Accepted         │
      │◀──────────────────────────────────────────────┤
      │                      │                        │
      │                      │ Challenge(Nonce)       │
      │                      ◀────────────────────────┤ (via LoRa server)
      │                      │                        │
      │ Evidence(Nonce,...)  │                        │
      │─────────────────────▶│                        │
      │                      │ POST /evidence         │
      │                      ├────────────────────────▶
      │                      │                        │
      │                      │ 200 OK                 │
      │                      ◀────────────────────────┤
      │                      │                        │
      │ GET /devices/{id}    │                        │
      ├───────────────────────────────────────────────▶
      │                      │                        │
      │ { "status": "TRUSTED" }
      ◀───────────────────────────────────────────────┤
      │                      │                        │
```

### Endpoint Definitions

*   **Device Management**
    *   `POST /devices`: Register a new device. Requires a signed device certificate.
    *   `GET /devices`: List all registered devices.
    *   `GET /devices/{device_id}`: Get the detailed status of a single device.
    *   `DELETE /devices/{device_id}`: Revoke (blacklist) a device.

*   **Policy Management**
    *   `POST /policies`: Upload a new appraisal policy for a new firmware version.
    *   `GET /policies/{version}`: Retrieve the policy for a specific version.

*   **Attestation**
    *   `POST /attestations/{device_id}`
        *   **Action:** Initiates an attestation check for the specified device.
        *   **Process:** Generates a nonce, stores it in Redis, and sends a `Challenge` message to the device via the integrated network server (e.g., LoRaWAN Application Server).
        *   **Response:** `202 Accepted`
    *   `POST /evidence`
        *   **Action:** The endpoint where the network server forwards the `AttestationEvidence` packet from the device.
        *   **Process:** The API gateway receives the evidence and passes it to the `appraise_evidence` function in the Attestation Engine. The result is logged.
        *   **Response:** `200 OK`

---

## 5. User Interface & Alerting

While the API is the core, a human interface is needed for operators.

*   **Web Dashboard:** A simple web frontend (e.g., built with React or Vue) will be created to interact with the API. It will provide:
    *   A list of all devices with their current status (TRUSTED, UNTRUSTED) indicated by color-coded icons.
    *   A detailed view for each device, showing its full attestation history from the `audit_log`.
    *   Forms for registering new devices and uploading new appraisal policies.
*   **Alerting:** When the `appraise_evidence` function returns a `False` result, the Attestation Engine will trigger an alert.
    *   **Mechanism:** This can be configured to send a message to a webhook (e.g., for Slack or Microsoft Teams notifications), send an email to a security mailing list, or integrate with a SIEM system.
    *   **Content:** The alert will contain the device ID, timestamp, and the specific reason for failure (e.g., "PCR[1] mismatch", "Signature validation failed").

---

**Next Phase:** [[13_System_Integration.md|Phase 13: System Integration]]
