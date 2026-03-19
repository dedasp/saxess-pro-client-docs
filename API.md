# Saxess Pro — Client Integration API  
These APIs allow client applications to initiate **biometric-gated CIBA authentication** and verify server availability.  
All endpoints listed below are **official**, **stable**, and **supported**.

---

#### Table of Contents
- [General API Information](#general-api-information)
- [API](#api)
  - [/api/status](#apistatus)
    - Health-check endpoint
  - [/api/auth/par](#apiauthpar)
    - Pushed Authorization Request (PAR) endpoint.
    - Used to securely register the authorization request before initiating CIBA.
  - [/api/auth/backchannel-authentication](#apiauthbackchannel-authentication)
    - Initiates the CIBA authentication flow using the request_uri obtained from PAR.
  - [/api/auth/token — CIBA Polling](#apiauthtoken--ciba-polling)
    - Polling endpoint for CIBA token
  - [/api/auth/token — Refresh Token](#apiauthtoken--refresh-token)
    - Exchange refresh token for new access token
  - [/api/orgs/{org_id}/employees/auto-invite — Invite User/Employee](#apiorginvite)
    - Invite a user/employee programmatically with optional auto-approval.
    - Requests must be signed using RSA-SHA256 to ensure authenticity and integrity.
---

## General API Information
- **Base Endpoint:** `https://pro-be.s.technology`
- All responses are **JSON**
- Authentication uses OIDC CIBA with JWT client authentication.
- All sensitive parameters are sent via signed JWT request objects.
- User authentication is completed out-of-band via biometric approval.
- Client private and certificate keys must be securely stored (HSM/KMS recommended). Public keys and certificates are registered during client onboarding.
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

## /api/auth/par + /api/auth/backchannel-authentication

Initiates a **CIBA Backchannel Authentication** flow using **PAR (Pushed Authorization Request)**.

This flow involves 2 steps:

1. **PAR (`/auth/par`)** → Register signed request object  
2. **Backchannel (`/auth/backchannel-authentication`)** → Initiate authentication  

This endpoint expects:

- A **JWT client assertion** generated with the client's private key  
- A **REQUEST OBJECT** (also a JWT) containing:  
  - `client_id`  
  - `login_hint` (user email)  
  - `scope`  
  - `authorization_details` (RAR)  
- A **RAR object** specifying biometric or transaction requirements  
- All JWTs must be signed using **PS256** with short-lived expiry  

---

### **Step 1: PAR Request**

`POST {{domain}}/api/auth/par`

#### **Body (x-www-form-urlencoded or JSON)**
```json
{
  "client_id": CLIENT_ID,
  "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
  "request": "<REQUEST_OBJECT_JWT>",
  "client_assertion": "<CLIENT_ASSERTION_JWT>"
}

---

### **Response**

```json
{
  "request_uri": "urn:ietf:params:oauth:request_uri:xyz",
  "expires_in": 90
}
```

---


### **Step 2: Backchannel Authentication**

`POST {{domain}}/api/auth/backchannel-authentication`

#### **Body (x-www-form-urlencoded or JSON)**
```json
{
  "client_id": CLIENT_ID,
  "request_uri": "<REQUEST_URI_FROM_PAR>",
  "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
  "client_assertion": "<CLIENT_ASSERTION_JWT>"
}

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
| request_uri  | Reference to pushed authorization request             |
| auth_req_id  | Unique CIBA authentication request ID                 |
| expires_in   | Validity of the request (in seconds)                  |
| interval     | Polling frequency for token endpoint (in seconds)     |

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

    [
      {
        type: "transfer",
        purpose: "transaction_approval",
        actions: ["approve"],
        resource: CLIENT_ID,
        amount: {
          value: "5000",
          currency: "USDT"
        },
        destination: {
          address: "0xabcd1234ef567890",
          network: "ethereum"
        }
      }
    ]

    [
      {
        type: "payment_initiation",
        purpose: "merchant_checkout",
        resource: CLIENT_ID,
        creditor: {
          name: "Acme Online Store",
          account: "DE89370400440532013000"
        },
        instructedAmount: {
          currency: "EUR",
          amount: "249.99"
        },
        remittanceInformation: "Order #84512",
        executionDate: "2026-02-25"
      }
    ]

    [
      {
        type: "account_information",
        purpose: "financial_overview",
        resource: CLIENT_ID,
        actions: [
          "read_balances",
          "read_transactions"
        ],
        accounts: [
          {
            iban: "FR7630006000011234567890189"
          }
        ],
        time_period: {
          from: "2026-01-01",
          to: "2026-02-20"
        }
      }
    ]

    [
      {
        type: "crypto_wallet_sign",
        purpose: "smart_contract_execution",
        resource: CLIENT_ID,
        wallet: {
          address: "0xabcd1234ef567890",
          network: "ethereum"
        },
        contract: {
          address: "0xdef7890123456789",
          method: "swapExactTokensForETH"
        },
        max_gas_fee: {
          value: "0.02",
          currency: "ETH"
        }
      }
    ]

    [
      {
        type: "identity_verification",
        purpose: "loan_application",
        resource: CLIENT_ID,
        requested_claims: [
          "full_name",
          "date_of_birth",
          "national_id_number"
        ],
        verification_level: "liveness_plus_document",
        validity_period: 86400
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
- `iat`
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
- `iat`

These JWTs must follow **OIDC CIBA specifications** and must be signed using **PS256**.


```js
// ----- Build RAR (Rich Authorization Request) -----
 const authorizationDetails = [
      {
        type: "biometric_assertion",
        purpose: "secure_login",
        resource: CLIENT_ID,
        actions: ["authenticate"],
        device_binding: "nfc-card"
      }
    ];

    // Build JWT client assertion for auth
    const clientAssertion = jwt.sign(
      {
        iss: CLIENT_ID,
        sub: CLIENT_ID,
        aud: `${AUTH_SERVER_URL}/auth/par`,
        jti: crypto.randomBytes(16).toString("hex"),
        exp: Math.floor(Date.now() / 1000) + 60,
        iat: Math.floor(Date.now() / 1000),
      },
      privateKey,
      { algorithm: "PS256", keyid: key_id }
    );
    // Build REQUEST OBJECT (protects all CIBA request parameters)
    const requestObject = jwt.sign(
      {
        iss: CLIENT_ID,
        sub: CLIENT_ID,
        aud: `${AUTH_SERVER_URL}/auth/backchannel-authentication`,
        client_id: CLIENT_ID,
        login_hint: email,
        scope: "openid profile email",
        authorization_details: authorizationDetails,
        jti: crypto.randomBytes(16).toString("hex"),
        exp: Math.floor(Date.now() / 1000) + 60,
        iat: Math.floor(Date.now() / 1000)
      },
      privateKey,
      { algorithm: "PS256", keyid: key_id }
    );
    // Request CIBA authentication
    const resp = await axios.post(
      `${AUTH_SERVER_URL}/auth/par`,
      {
        client_id: CLIENT_ID,
        request: requestObject,
        client_assertion_type:
          "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        client_assertion: clientAssertion
      },
      {
        httpsAgent,
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          "Cache-Control": "no-store",
          "Pragma": "no-cache"
        }
      }
    );

    let { request_uri, expires_in } = resp.data;
    console.log("CIBA PAR initiated, request_uri:", request_uri);

    // Build JWT client assertion for auth
    const clientAssertion2 = jwt.sign(
      {
        iss: CLIENT_ID,
        sub: CLIENT_ID,
        aud: `${AUTH_SERVER_URL}/auth/backchannel-authentication`,
        jti: crypto.randomBytes(16).toString("hex"),
        exp: Math.floor(Date.now() / 1000) + 60,
        iat: Math.floor(Date.now() / 1000),
      },
      privateKey,
      { algorithm: "PS256", keyid: key_id }
    );

    // Request CIBA authentication
    const resp2 = await axios.post(
      `${AUTH_SERVER_URL}/auth/backchannel-authentication`,
      {
        client_id: CLIENT_ID,
        request_uri: request_uri,
        client_assertion_type:
          "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
        client_assertion: clientAssertion2
      },
      {
        httpsAgent,
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
          "Cache-Control": "no-store",
          "Pragma": "no-cache"
        }
      }
    );

    ({ auth_req_id, expires_in, interval } = resp2.data);
```


---

## /api/auth/token — CIBA Polling
Token polling endpoint for **OIDC CIBA**.  
The client calls this endpoint repeatedly (based on the `interval` value from `/api/auth/backchannel-authentication`) until the request is approved / rejected / expired.    

Clients must respect the interval value. Polling faster may result in slow_down or temporary blocking.

---

### **Request**
`POST {{domain}}/api/auth/token`

#### **Body (x-www-form-urlencoded)**
```json
  grant_type=urn:openid:params:grant-type:ciba
  auth_req_id=AUTH_REQ_ID
  client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
  client_assertion=SIGNED_JWT

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

#### **Rejected Response**  
Returned when biometric approval **rejected**.

```json
{
  "status": "request_rejected"
}
```

#### **Expired Response**  
Returned when biometric approval timeline has **expired**.

```json
{
  "status": "expired_token"
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
        aud: `${AUTH_SERVER_URL}/auth/token`,
        jti: Math.random().toString(36).substring(2),
        exp: Math.floor(Date.now() / 1000) + 60,
        iat: Math.floor(Date.now() / 1000),
      },
      privateKey,
      { algorithm: "PS256", keyid: key_id }
    );


 const params = new URLSearchParams();
    params.append("grant_type", "urn:openid:params:grant-type:ciba");
    params.append("auth_req_id", auth_req_id);
    params.append(
      "client_assertion_type",
      "urn:ietf:params:oauth:client-assertion-type:jwt-bearer"
    );
    params.append("client_assertion", clientAssertion);

    const resp = await axios.post(
      `${AUTH_SERVER_URL}/auth/token`,
      params,
      {
        httpsAgent,
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
        },
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
// Expired → { status: "expired_token" }
// Failed → { status: "error_message" }


// verifying Auth server Signature

  const JWKS = createRemoteJWKSet(
    new URL(`${AUTH_SERVER_URL}/auth/jwks.json`)
  );

  const { payload } = await jwtVerify(tokens.id_token, JWKS, {
      issuer: JWT_ISSUER,
      audience: CLIENT_ID,
    });

    console.log("ID Token payload:");
    console.dir(payload, { depth: null, colors: true });

    const sessionToken = jwt.sign(
      {
        email: payload.email,
        name: payload.sub,
        acr: payload.acr || "urn:mfa:biometric",
      },
      process.env.RP_SESSION_SECRET,
      { expiresIn: "1h" }
    );

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


```


---


## /api/auth/token — Refresh Token
Refresh Token grant endpoint for obtaining a new access token.  
Used when the existing access token has expired and the client needs to rotate tokens.

---

### **Request**
`POST {{domain}}/api/auth/token`

#### **Body (x-www-form-urlencoded)**
```json
  grant_type=urn:openid:params:grant-type:ciba
  refresh_token: "xxx",
  client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
  client_assertion=SIGNED_JWT

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



## /api/orgs/{org_id}/employees/auto-invite — Invite User/Employee

 - Invite a new employee/user to the organization


---

### **Request**
`POST {{domain}}/api/orgs/{org_id}/employees/auto-invite`

#### **Headers **
```json
Content-Type: application/json
X-Signature: BASE64_ENCODED_RSA_SHA256_SIGNATURE
X-Key-Id: CLIENT_KEY_ID
```

#### **Body (application/json)**
```json
{
  "email": "user@example.com",
  "full_name": "John Doe",
  "client_id": "sync-client", // Client identifier issued during onboarding
  "auto_approve": true, // If true, user is auto-approved after device registration; otherwise remains pending and need dashboard approval
  "timestamp": 1710000000 // Unix timestamp (used for replay protection)
}
```

#### **Request Signing **
The request body must be:
 - JSON stringified
 - Signed using RSA-SHA256 with the client’s private key
 - Signature sent in X-Signature header (Base64 encoded)
 - Server verifies signature using the client’s public key
 - timestamp is validated to prevent replay attacks (recommended ±2 min window)
 - X-Key-Id is used to identify which public key to use for verification

```json
const signature = crypto.sign(
  "RSA-SHA256",
  Buffer.from(JSON.stringify(body)),
  privateKey
).toString("base64");
```

---

### **Response**

#### **Successful Response**  
Returned when the invite is successfully created.
```json
{
  id: user_id,
  email: 'user_email',
  full_name: 'user_name',
  status: 'invited',
  org_id: org_id,
  org_name: org_name
}
```

#### **Errors**
```json
{
  "error": [
    "invalid_request",
    "missing_signature_headers",
    "invalid_timestamp",
    "invalid_client",
    "invalid_public_key",
    "invalid_signature"
  ]
}
```

#### ** Example Code
```js

 const timestamp = Math.floor(Date.now() / 1000);

  const body = {
    email: "himang305+3@gmail.com",
    full_name: "Himanshu Tests",
    client_id: "sync-client",
    auto_approve: true,
    timestamp: timestamp
  };

  const bodyString = JSON.stringify(body);
  const signature = crypto.sign(
    "RSA-SHA256",
    Buffer.from(bodyString),
    privateKey
  );
  const signatureBase64 = signature.toString("base64");

  axios.post(
    AUTH_SERVER_URL + "/orgs/2/employees/auto-invite",
    body,
    {
      headers: {
        "Content-Type": "application/json",
        "X-Signature": signatureBase64,
        "X-Key-Id": key_id
      }
    }
  ).then(res => {
    console.log(res.data);
  }).catch(err => {
    console.error(err.response?.data || err);
  });
  
```
