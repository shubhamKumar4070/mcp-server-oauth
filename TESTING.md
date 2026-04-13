---

### File 2: `TESTING.md`

```markdown
# QA & Testing Guide: Dual-Auth MCP Server

This guide outlines the Postman test cases required to validate both branches of the MuleSoft Choice Router, ensuring the security gateway is functioning correctly before connecting live AI agents.

## 🛠️ Prerequisites
* Anypoint Studio running the Mule application locally (e.g., `http://localhost:8081`).
* Postman (or equivalent REST client).
* Your Okta Developer Tenant Domain, Client ID, and Client Secret.

---

## ✅ Test Case 1: The Enterprise Path (Bearer Token)
**Goal:** Verify that Branch A correctly introspects an active JWT.

### Step 1.1: Generate a Token from Okta
* **Method:** `POST`
* **URL:** `https://{your-okta-domain}/oauth2/default/v1/token`
* **Headers:** `Content-Type: application/x-www-form-urlencoded`
* **Body:**
  * `grant_type`: `client_credentials`
  * `client_id`: `[Your Client ID]`
  * `client_secret`: `[Your Client Secret]`
*(Copy the resulting `access_token` string).*

### Step 1.2: Authenticate with MuleSoft
* **Method:** `POST`
* **URL:** `http://localhost:8081/` (Adjust path if needed)
* **Headers:**
  * `Authorization`: `Bearer [PASTE_TOKEN_HERE]`
* **Expected Result:** HTTP 200/202 (Session Initiated). 
* **Expected Mule Logs:** 1. `Auth Strategy: Introspecting Bearer Token via Okta...`
  2. `SUCCESS: MCP Session Established. Token verified.`

---

## ✅ Test Case 2: The Developer Path (Direct Credentials)
**Goal:** Verify that Branch B intercepts custom headers and successfully negotiates a token with Okta on the client's behalf.

* **Method:** `POST`
* **URL:** `http://localhost:8081/`
* **Headers:** (Ensure the `Authorization` header is REMOVED)
  * `x-client-id`: `[Your Client ID]`
  * `x-client-secret`: `[Your Client Secret]`
* **Expected Result:** HTTP 200/202.
* **Expected Mule Logs:**
  1. `Auth Strategy: Validating Direct Credentials via Okta...`
  2. `SUCCESS: MCP Session Established. Direct Credentials verified.`

---

## 🚫 Test Case 3: Negative Security Testing
**Goal:** Prove the gateway rejects unauthorized traffic.

### Scenario 3.1: Expired or Invalid Token
* **Test:** Execute **Test Case 1.2**, but change a few characters in the Bearer token.
* **Expected Result:** HTTP 401 Unauthorized.
* **Expected Mule Logs:** `Bearer Token is inactive or expired.`

### Scenario 3.2: Invalid Client Credentials
* **Test:** Execute **Test Case 2**, but change a few characters in the `x-client-secret`.
* **Expected Result:** HTTP 401 Unauthorized.
* **Expected Mule Logs:** `Invalid Client ID or Secret provided by Dev Agent.`

### Scenario 3.3: Unauthenticated Access Attempt
* **Test:** Send a `POST` request to `http://localhost:8081/` with **NO** `Authorization`, `x-client-id`, or `x-client-secret` headers.
* **Expected Result:** HTTP 401 Unauthorized.
* **Expected Mule Logs:** `Request blocked. Missing Authorization or x-client-id headers.`