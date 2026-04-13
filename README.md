# Secure MuleSoft MCP Server: Dual-Auth Architecture

This repository contains a production-grade MuleSoft Model Context Protocol (MCP) server integration. It exposes enterprise APIs to AI Agents (like MuleSoft Vibe and Salesforce Agentforce) over the modern **Streamable HTTP** transport while enforcing strict **Machine-to-Machine (M2M)** security via Okta.

## 🏗️ Architectural Overview

This system utilizes a **Dual-Auth API Gateway Pattern** built natively within MuleSoft. Because different AI clients have different capabilities and lifecycle constraints, the Mule app acts as a smart router to validate identities using two distinct OAuth 2.0 Client Credentials strategies.

### The Two Security Branches
1. **Branch A: The Token Path (Enterprise Production)**
   * **Target Client:** Salesforce Agentforce / Enterprise SaaS Tenants.
   * **Flow:** The external client authenticates with Okta independently, retrieves a temporary Access Token (JWT), and passes it to MuleSoft as a Bearer token. MuleSoft calls Okta's `/introspect` endpoint to verify the token is active.
   * **Advantage:** Highly secure; secrets never cross the Mule network.

2. **Branch B: The Direct Credential Path (Local Development)**
   * **Target Client:** MuleSoft Dev Agent (Vibe) in VS Code.
   * **Flow:** Because local VS Code settings (`a4d_mcp_settings.json`) are static and cannot auto-refresh expired tokens, Vibe passes its Okta Client ID and Secret directly via custom HTTP headers. MuleSoft intercepts these headers, calls Okta's `/token` endpoint on behalf of Vibe, and establishes the session if Okta grants the token.
   * **Advantage:** Eliminates the friction of 60-minute token expirations during active developer coding sessions.

---

## ⚙️ Phase 1: Okta IdP Configuration

Before running the Mule app, you must configure Okta to allow headless (M2M) applications.

1. **Create the App:** In Okta, create an **API Services** Application. Save the `Client ID` and `Client Secret`.
2. **Define Scopes:** Go to **Security > API > Authorization Servers**. Under the `default` server, create a scope named `mcp.invoke` and mark it as a default scope.
3. **Allow Client Credentials:** In the `default` server's **Access Policies** tab, add a rule where:
   * **Grant Type:** `Client Credentials`
   * **User is:** `Any user assigned the app` (Required bypass for M2M).

---

## 🐴 Phase 2: MuleSoft Configuration

The core security logic resides in the `<mcp:on-new-session-listener>` flow. 

### Properties (`config-local.yaml`)
```yaml
mule:
  okta:
    domain: "dev-xxxxxx.okta.com"
    client_id: "your_mulesoft_client_id"
    client_secret: "your_mulesoft_client_secret"
    
security:
  allow_direct_credentials: "true" # WARNING: Set to 'false' in Production!
🔒 Production Security Warning
Branch B (Direct Credentials) is an anti-pattern for production environments. Passing raw secrets on every request exhausts Okta API rate limits and increases exposure risk.
Always ensure security.allow_direct_credentials: "false" in your config-prod.yaml. External customers must utilize Branch A (Tokens) via Dynamic Client Registration (DCR).

💻 Phase 3: Client Setup (MuleSoft Vibe / VS Code)
To connect your local Anypoint Code Builder Vibe agent to this running Mule server without dealing with token timeouts, update your local MCP settings.

File: a4d_mcp_settings.json