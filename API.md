# Saxess Pro — Client Integration API  
These APIs allow client applications to initiate **biometric-gated CIBA authentication** and verify server availability.  
All endpoints listed below are **official**, **stable**, and **supported**.

---

#### Table of Contents
- [General API Information](#general-api-information)
- [API](#api)
  - [/api/status](#apistatus)
    - Health-check endpoint
  - [/api/auth/authorize](#apiauthauthorize)
    - Initiates CIBA authentication
  - [/api/auth/token — CIBA Polling](#apiauthtoken--ciba-polling)
    - Polling endpoint for CIBA token
  - [/api/auth/token — Refresh Token](#apiauthtoken--refresh-token)
    - Exchange refresh token for new access token

---

## General API Information
- **Base Endpoint:** `https://pro-be.s.technology`
- All responses are **JSON**
- All requests must use:  
  `Content-Type: application/json`
- Authentication uses OIDC CIBA with JWT client authentication.
- All sensitive parameters are sent via signed JWT request objects.
- User authentication is completed out-of-band via biometric approval.
- Client private keys must be securely stored (HSM/KMS recommended). Public keys are registered during client onboarding.
---

# API

---

## /api/status
Health-check endpoint to confirm the API server is running.

### Request
`GET {{domain}}/api/status`

### Response
```json
{
  "message": "Saxess Pro API server is running",
  "status": "ok"
}
```

---

## /api/auth/authorize

Initiates a **CIBA Backchannel Authentication** flow.

This endpoint expects:

- A **JWT client assertion** generated with the client's private key  
- A **REQUEST OBJECT** (also a JWT) containing:  
  - `client_id`  
  - `login_hint` (user email)  
  - `scope`  
  - `authorization_details` (RAR)  
- A **RAR object** specifying biometric authentication requirements 
- Signs the CIBA authorization JWT using RS256 with required claims and a short-lived expiry.
  - openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.key
 

---

### **Request**

`POST {{domain}}/api/auth/authorize`

#### **Body (raw JSON)**
```json
{
  "client_id": CLIENT_ID,
  "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
  "request": "<REQUEST_OBJECT_JWT>",
  "client_assertion": "<CLIENT_ASSERTION_JWT>"
}
```

---

### **Response**

```json
{
  "auth_req_id": "AUTH_REQ_ID_HERE",
  "expires_in": 180,
  "interval": 10
}
```

---

## Parameter Explanation

| Field        | Description                                           |
|--------------|-------------------------------------------------------|
| auth_req_id  | Unique CIBA authentication request ID                |
| expires_in   | Validity of the request (in seconds)                 |
| interval     | Polling frequency for token endpoint (in seconds)    |

---

## RAR Structure (JSON Reference)

```json
[
  {
    "type": "biometric_auth",
    "purpose": "secure_login",
    "resource": "CLIENT_ID",
    "actions": ["authenticate"],
    "device_binding": "nfc-card"
  }
]
```

---

## Required JWT Assertions (Summary)

### **client_assertion (JWT)**
Must be signed with the client’s private key.

Must include the following claims:
- `iss`
- `sub`
- `aud`
- `exp`
- `jti`

---

### **request object (JWT)**
Encapsulates all CIBA request parameters.

Must include:
- `client_id`
- `login_hint`
- `authorization_details`
- `scope`
- `aud`
- `exp`

These JWTs must follow **OIDC CIBA specifications** and must be signed using **RS256**.


```js
// ----- Build RAR (Rich Authorization Request) -----
const authorizationDetails = [
  {
    type: "biometric_auth",
    purpose: "secure_login",
    resource: CLIENT_ID,
    actions: ["authenticate"],
    device_binding: "nfc-card"
  }
];

// ----- Build Client Assertion (JWT) -----
const clientAssertion = jwt.sign(
  {
    iss: CLIENT_ID,
    sub: CLIENT_ID,
    aud: `${AUTH_SERVER_URL}/token`,
    jti: crypto.randomBytes(16).toString("hex"),
    exp: Math.floor(Date.now() / 1000) + 60
  },
  privateKey,
  { algorithm: "RS256" }
);

// ----- Build Request Object (JWT) -----
const requestObject = jwt.sign(
  {
    iss: CLIENT_ID,
    aud: `${AUTH_SERVER_URL}/auth/authorize`,
    client_id: CLIENT_ID,
    login_hint: email,
    scope: "openid profile email",
    authorization_details: authorizationDetails,
    jti: crypto.randomBytes(16).toString("hex"),
    nbf: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 60
  },
  privateKey,
  { algorithm: "RS256" }
);

// ----- Send CIBA Authorization Request -----
const resp = await axios.post(
  `${AUTH_SERVER_URL}/auth/authorize`,
  {
    client_id: CLIENT_ID,
    request: requestObject,
    client_assertion_type:
      "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    client_assertion: clientAssertion
  },
  {
    headers: {
      "Content-Type": "application/json",
      "Cache-Control": "no-store",
      "Pragma": "no-cache"
    }
  }
);

const { auth_req_id, expires_in, interval } = resp.data;
```


---

## /api/auth/token — CIBA Polling
Token polling endpoint for **OIDC CIBA**.  
The client calls this endpoint repeatedly (based on the `interval` value from `/api/auth/authorize`) until the request is approved / rejected / expired.    

Clients must respect the interval value. Polling faster may result in slow_down or temporary blocking.

---

### **Request**
`POST {{domain}}/api/auth/token`

#### **Body (raw JSON)**
```json
{
  "grant_type": "urn:openid:params:grant-type:ciba",
  "auth_code": auth_req_id, // uuid format
  "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
  "client_assertion": "signed_req"
}
```

---

### **Response**

#### **Successful Response**  
Returned when the user has approved biometric authentication.

```json
{
  "access_token": "ACCESS_TOKEN_VALUE", // PASETO v2 public
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "ID_TOKEN_VALUE", // PASETO v2 public
  "refresh_token": "REFRESH_TOKEN_VALUE"
}
```

#### **Pending Response**  
Returned when biometric approval has **not yet been completed**.

```json
{
  "status": "authorization_pending"
}
```

#### **Expired Response**  
Returned when biometric approval timeline has **expired**.

```json
{
  "status": "invalid_authorization_code"
}
```

---

### **Example (Node.js)**

```javascript

// ----- Build Client Assertion for Token Request -----

const clientAssertion = jwt.sign(
  {
    iss: CLIENT_ID,
    sub: CLIENT_ID,
    aud: `${AUTH_SERVER_URL}/token`,
    jti: Math.random().toString(36).substring(2),
    exp: Math.floor(Date.now() / 1000) + 60
  },
  privateKey,
  { algorithm: "RS256" }
);


// ----- Send Token Request (CIBA Polling) -----

const resp = await axios.post(
  `${AUTH_SERVER_URL}/auth/token`,
  {
    grant_type: "urn:openid:params:grant-type:ciba",
    auth_code: auth_req_id,  // from /auth/authorize response
    client_assertion_type: "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    client_assertion: clientAssertion
  },
  {
    headers: { "Content-Type": "application/json" }
  }
);

// ----- Handle Response -----
console.log("Token response:", resp.data);
const tokens = resp.data;
    if (!tokens.id_token) {
        return res.json({ status: resp.data.status });
    }

// Success → tokens returned
// Pending → { status: "authorization_pending" }
// Expired → { status: "invalid_authorization_code" }
// Failed → { status: "error_message" }


// verifying Auth server Paseto Signature
 const payload = await verifyPaseto(tokens.id_token);
    if (!payload) {
      return res.json({ status: "invalid_token" });
    }

// generate jwt for internal client front - back authentication
    const sessionToken = jwt.sign(
      {
        email: payload.email,
        name: payload.sub,
        acr: payload.acr || "urn:mfa:biometric",
      },
      process.env.RP_SESSION_SECRET, // client secret
      { expiresIn: "1h" }
    );

    authRequests.set(auth_req_id, { status: "authenticated", tokens });

    return res.json({
      status: "authenticated",
      user: {
        email: payload.email,
        name: payload.sub,
        acr: payload.acr || "urn:mfa:biometric"
      },
      session_token: sessionToken,
      expires_in: "1h",
      refresh_token: tokens.refresh_token
    });




let pasetoPublicKey = null;

// Verifies the PASETO’s signature, issuer, audience, and expiration to ensure the token is authentic, untampered, and intended for this client.

async function verifyPaseto(token) {
  try {

    if (!pasetoPublicKey) {
        const jwks = await fetch(`${AUTH_SERVER_URL}/auth/jwks.json`)
          .then(res => res.json());

        pasetoPublicKey = loadPasetoPublicKeyFromJwks(jwks);
      }

    const payload = await V2.verify(token, pasetoPublicKey, {
      issuer: "https://saxess.identity",
      audience: CLIENT_NAME,
    });

    if (payload.exp < Date.now() / 1000) {
      throw new Error("Token expired");
    }

    console.log(" PASETO verified:", payload);
    return payload;

  } catch (err) {
    console.error(" Invalid token:", err.message);
    return null;
  }
}

function loadPasetoPublicKeyFromJwks(jwks, kid = "saxess-key-1") {
  const key = jwks.keys.find(k => k.kid === kid);
  if (!key) throw new Error("JWKS key not found");

  if (key.kty !== "OKP" || key.crv !== "Ed25519") {
    throw new Error("Invalid key type in JWKS");
  }

  return Buffer.from(key.x, "base64url");
}

```


---


## /api/auth/token — Refresh Token
Refresh Token grant endpoint for obtaining a new access token.  
Used when the existing access token has expired and the client needs to rotate tokens.

---

### **Request**
`POST {{domain}}/api/auth/token`

#### **Body (raw JSON)**
```json
{
  "grant_type": "refresh_token",
  "refresh_token": "xxx",
  "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
  "client_assertion": "signed_jwt"
}
```

---

### **Response**

#### **Successful Response**  
Returned when the refresh token is valid.

```json
{
  "access_token": "ACCESS_TOKEN_VALUE",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "NEW_REFRESH_TOKEN_VALUE"
}
```

#### **Invalid / Expired Refresh Token**
```json
{
  "error": "invalid_refresh_token"
}
```

---
