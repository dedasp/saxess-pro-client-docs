# Saxess Pro – Client Onboarding & Integration Guide

## Product Overview

**Saxess Pro** is a secure, biometric-first authentication platform designed for high-assurance identity verification and authorization.  
It combines **biometric smart cards**, a **secure iOS/Android application**, and **FAPI 2.0-compliant APIs** to deliver phishing-resistant, out-of-band authentication for enterprise and financial-grade use cases.

🔗 Product Page: https://s.technology/product/saxess-pro/

---

## Documentation Landscape

This repository contains the **official and supported documentation** for integrating with Saxess Pro.

| Document | Purpose |
|--------|--------|
| [SaxessPro.md](./SaxessPro.md)| High-level overview of the Saxess Pro platform and architecture |
| [API.md](./API.md) | **Official Client API Integration Guide** |
| [Official Node.js SDK](https://www.npmjs.com/package/@saxess-pro/client-auth) | Ready-to-use SDK for backend integration |


---

## Getting Started – Client Onboarding

### Initiating Onboarding

To begin onboarding, **contact the Saxess team**:

📧 **contact@s.technology**

The Saxess team will guide you through organisation creation, subscription setup, and credential provisioning.

---

## Saxess Pro Platform Components

Clients integrate with the following Saxess Pro components:

- **Saxess Pro Biometric Cards**
- **Saxess Pro iOS/Android Application**
- **Saxess Pro Dashboard (Web Console)**
- **Saxess Pro Authentication APIs**

---

## Organisation & Platform Setup

### Step 1: Organisation Creation

1. A **Platform Administrator** creates an **Organisation** within the Saxess Pro platform.
2. Subscription details and validity are configured.
3. An **Organisation Admin** is designated.

### Step 2: Organisation Admin Invitation

- The Organisation Admin receives an **invitation email** from the Saxess Pro Dashboard.
- The invitation is used to complete **one-time account setup**.

---

## Device & Biometric Setup

### Admin Setup

The Organisation Admin performs the following:

1. **Configures biometric credentials** on the Saxess Pro biometric card.
2. **Installs the Saxess Pro iOS/Android application** on a trusted mobile device.
3. Completes **one-time device binding** using the invitation email.

This establishes:
- Secure device ownership
- Biometric trust anchor
- Card-to-device binding

---

## Organisation Configuration

Once onboarded:

1. Organisation Admin logs into the **Saxess Pro Dashboard**.
2. Configures **Client Applications** that will use Saxess Pro authentication:
   - Web applications
   - Mobile applications
   - Backend / server applications
3. Application-specific credentials and policies are generated.

---

## Employee / User Onboarding

### Inviting Employees

Organisation Admins can:

1. Invite employees/users via email OR Invite employees/users programmatically via API integration.
2. Enable auto-approval for trusted clients or internal systems, allowing users (and their devices) to be approved instantly without manual intervention.
3. Assign **roles and access policies**.
4. Each employee completes:
   - One-time biometric card setup
   - iOS/Android app installation
   - Secure phone + card binding

### Access Control

- Employees are granted access to internal or external applications based on **roles**.
- Authorization policies are enforced centrally via Saxess Pro.

---

### Support & Contact

- For onboarding, integration support, or commercial inquiries:

📧 contact@s.technology

---



