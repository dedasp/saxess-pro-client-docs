# Official Documentation for the Saxess Pro Client APIs

## Prerequisites for Using the APIs
Before a client application can begin integrating or calling any Saxess Pro APIs, the following onboarding steps must be completed:

1. **Contact the Saxess team** at **contact@s.technology** to initiate onboarding.
2. **Request for an Organisation** within the Saxess Pro platform.
3. **Onboard the Organisation Admin** (primary administrator who will manage the platform).
4. **Create Client Platforms Apps** (web, mobile, backend) that will consume Saxess Pro APIs.
5. **Invite Employees/Users** and assign **appropriate roles** required for access and authorization.

>>Only after these steps are completed will API credentials and platform configurations be available for integration.

---

## API Documentation Overview

- Endpoints, parameters, payloads, and authentication flows described in the documents in this repository are considered **official** and **supported**.
- The use of any undocumented endpoints, parameters, or payloads is **not supported** — **use them at your own risk and with no guarantees**.

---

## Standards & Specifications Compliance

Saxess Pro authentication and authorization flows are built on open, industry-standard specifications to ensure interoperability, security, and enterprise readiness. The platform aligns with the following:

1. **Financial-grade API Security (FAPI 2.0)**  
   Saxess Pro follows FAPI 2.0 security principles for high-assurance, financial-grade API interactions.

2. **OIDC Client-Initiated Backchannel Authentication (CIBA) – RFC 9101**  
   Used for initiating secure, asynchronous, out-of-band user authentication.

3. **Rich Authorization Requests (RAR) – RFC 9396**  
   Used to express fine-grained, intent-based authorization requirements (e.g., biometric authentication, device binding).

4. **OAuth 2.0 JWT Client Authentication – RFC 7523**  
   Used for cryptographic client authentication via signed JWT assertions.

> These standards ensure that Saxess Pro integrates cleanly with modern identity, security, and compliance ecosystems.

---


## High-Level Authentication Flow (CIBA)

```text
Client Application
      |
      |  (1) Signed CIBA Request (JWT + RAR)
      v
Saxess Auth Server
      |
      |  (2) Trigger biometric approval
      v
User Device (App + NFC Card)
      |
      |  (3) Biometric approval / rejection
      v
Saxess Auth Server
      |
      |  (4) Token available via polling
      v
Client Application
```

Key characteristics:

1. No browser redirects

2. All sensitive parameters are protected via signed JWT request objects

3. User authentication is performed out-of-band using biometrics and device binding

---

### Documentation Index

Name | Description | Version
------------ | ------------ | ------------
[Saxess Pro Client Integration API](./SaxessPro-Client-API.md) | Unified API documentation for onboarding, authentication, device binding, notification signing, and verification flows | v1.0.0
