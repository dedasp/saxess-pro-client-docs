# Official Documentation for the Saxess Pro



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
[Saxess Pro Client Integration API](./API.md) | Unified API documentation for onboarding, authentication, device binding, notification signing, and verification flows | v1.0.0
