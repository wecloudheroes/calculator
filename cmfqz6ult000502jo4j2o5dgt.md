---
title: "Implementing OAuth 2.0 with Azure AD (Frontend + Backend) — a detailed step-by-step blog"
datePublished: Fri Sep 19 2025 15:09:31 GMT+0000 (Coordinated Universal Time)
cuid: cmfqz6ult000502jo4j2o5dgt
slug: implementing-oauth-20-with-azure-ad-frontend-backend-a-detailed-step-by-step-blog
tags: azure, oauth, apis, jwt, msal

---

**Goal:** show an end-to-end implementation where a Single Page App (SPA) authenticates a user using Azure AD (OAuth 2.0 Authorization Code + PKCE), receives an **access token (JWT)**, calls a protected backend API, and the backend validates the token before returning data. I’ll also cover machine-to-machine (Client Credentials), how to *expose* scopes, token validation, and best practices.

> register two apps in Azure AD (Frontend SPA + Backend API), expose a scope on the backend, use **MSAL.js** in the frontend (auth code + PKCE) to get an access token (a JWT), and validate that JWT on the backend using Azure’s JWKS (public keys). [Microsoft Learn+2Microsoft Learn+2](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?utm_source=chatgpt.com)

---

# Prerequisites

* Azure subscription & permissions to create **App registrations** in Microsoft Entra ID (Azure AD). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?utm_source=chatgpt.com)
    
* Local dev: Node 18+, npm/yarn (for SPA + server examples).
    
* Libraries shown: `@azure/msal-browser` (frontend), `jose` (Node backend). (I show code for React/vanilla SPA and Node backend; Python/ .NET examples notes follow.)
    

---

# Overview (high level)

1. Register **Backend API** in Azure AD → **Expose an API** and add a custom scope (e.g. `api://<api-client-id>/access_as_user`). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-expose-web-apis?utm_source=chatgpt.com)
    
2. Register **Frontend SPA** in Azure AD → Redirect URI type **SPA** (or web) and request the API scope. Use Authorization Code Flow **with PKCE** (recommended for SPAs). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/msal-authentication-flows?utm_source=chatgpt.com)
    
3. Frontend uses **MSAL.js** to sign in and acquire an access token (JWT). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/msal/javascript/?utm_source=chatgpt.com)
    
4. Frontend sends `Authorization: Bearer <access_token>` to backend.
    
5. Backend validates JWT using Azure AD’s **JWKS** (from the OpenID Connect discovery endpoint), checks `aud`, `iss`, `exp`, and required `scp`/`roles`. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    

---

# Step 1 — Register apps in Azure AD (Portal steps)

**A. Create Backend API app**

1. Azure Portal → Microsoft Entra ID → **App registrations** → **New registration**. Enter name (e.g. `my-backend-api`). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?utm_source=chatgpt.com)
    
2. After creation → *Expose an API* → set the **Application ID URI** (default looks like `api://<client-id>` or a custom URI). Click **Add a scope** and create a scope like `access_as_user` with a display name / admin consent description. (This becomes `api://<client-id>/access_as_user`.) [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-expose-web-apis?utm_source=chatgpt.com)
    

**B. Create Frontend SPA app**

1. Azure Portal → App registrations → **New registration** → name `my-spa`.
    
2. Set Redirect URI → **Single-page application (SPA)** and add URL e.g. `http://localhost:3000`. Ensure type `spa` if available. (This enables the auth code flow for SPAs.) [Microsoft Learn+1](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?utm_source=chatgpt.com)
    
3. In the SPA app → *API permissions* → **Add a permission** → My APIs → select your Backend API → select the scope `access_as_user`. Then **Grant admin consent** (or user consent depending on your tenant policy).
    

> Note: For machine-to-machine (server → server) create a **confidential client** app (backend) and use **Client Credentials Flow** (see Step 7). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow?utm_source=chatgpt.com)

---

# Step 2 — Understand the endpoints (OIDC discovery & JWKS)

Azure publishes OpenID configuration and JWKS for your tenant:

* Discovery document (v2 example):  
    `https://login.microsoftonline.com/<TENANT_ID>/v2.0/.well-known/openid-configuration`  
    This includes `jwks_uri` which lists public signing keys used to verify JWT signatures. Use this to validate tokens and handle key rollover automatically. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    

---

# Step 3 — Frontend: MSAL.js (Auth code + PKCE) example

Install:

```plaintext
npm install @azure/msal-browser
```

Minimal `auth.js` (vanilla or integrate in React):

```plaintext
import { PublicClientApplication } from "@azure/msal-browser";

const msalConfig = {
  auth: {
    clientId: "<SPA_CLIENT_ID>",
    authority: "https://login.microsoftonline.com/<TENANT_ID>",
    redirectUri: "http://localhost:3000",
  },
  cache: {
    cacheLocation: "sessionStorage", // or "localStorage" (sessionStorage recommended)
  }
};

const msalInstance = new PublicClientApplication(msalConfig);

const loginRequest = {
  scopes: ["openid", "profile", "api://<API_CLIENT_ID>/access_as_user"]
};

// Sign-in
async function signIn() {
  const loginResp = await msalInstance.loginPopup(loginRequest);
  return loginResp;
}

// Acquire token to call API
async function getAccessToken() {
  const account = msalInstance.getAllAccounts()[0];
  try {
    const tokenResp = await msalInstance.acquireTokenSilent({
      scopes: ["api://<API_CLIENT_ID>/access_as_user"],
      account
    });
    return tokenResp.accessToken;
  } catch (err) {
    // fallback to interactive if silent fails
    const tokenResp = await msalInstance.acquireTokenPopup({
      scopes: ["api://<API_CLIENT_ID>/access_as_user"]
    });
    return tokenResp.accessToken;
  }
}
```

Then call backend:

```plaintext
const token = await getAccessToken();
fetch('http://localhost:4000/secure-data', {
  headers: { Authorization: `Bearer ${token}` }
});
```

**Why this?** MSAL implements the Authorization Code + PKCE flow for SPAs (recommended over implicit). PKCE prevents auth code injection and is required for SPA auth code flows. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/msal-authentication-flows?utm_source=chatgpt.com)

Reference: MSAL.js docs & samples. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/msal/javascript/?utm_source=chatgpt.com)

---

# Step 4 — Backend: validate the incoming JWT (Node.js example using `jose`)

Install:

```plaintext
npm install express jose
```

`server.js` (minimal, production code should add caching, logging, better error messages):

```plaintext
import express from "express";
import { createRemoteJWKSet, jwtVerify } from "jose";

const app = express();
const port = 4000;

const tenantId = "<TENANT_ID>";
const audience = "api://<API_CLIENT_ID>"; // or the client-id GUID
const issuer = `https://login.microsoftonline.com/${tenantId}/v2.0/`;
const jwksUri = `https://login.microsoftonline.com/${tenantId}/discovery/v2.0/keys`;

// create a JWKS remote key store (handles key rollover)
const JWKS = createRemoteJWKSet(new URL(jwksUri));

async function verifyAccessToken(token) {
  // will throw on invalid
  const { payload } = await jwtVerify(token, JWKS, {
    issuer,
    audience
  });
  // payload contains claims like sub, scp, roles, exp
  return payload;
}

app.get("/secure-data", async (req, res) => {
  try {
    const auth = req.headers.authorization || "";
    const token = auth.split(" ")[1];
    if (!token) return res.status(401).send({ error: "missing token" });

    const payload = await verifyAccessToken(token);

    // Example: ensure required scope present
    const scopes = (payload.scp || "").split(" ");
    if (!scopes.includes("access_as_user")) {
      return res.status(403).send({ error: "insufficient scopes" });
    }

    res.send({ message: "Hello from protected API", user: payload.sub });
  } catch (err) {
    console.error(err);
    res.status(401).send({ error: "invalid token" });
  }
});

app.listen(port, () => console.log(`API listening ${port}`));
```

**Notes**

* We use the `jwks_uri` so the library fetches the current public keys (handles rollover). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    
* Validate `aud` (audience) and `iss` (issuer), and `exp`. Also check `scp` (delegated scopes) or `roles` (app roles) depending on flow. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    

---

# Step 5 — Expose API scopes & grant consent (Azure Portal)

* In your Backend app registration → **Expose an API** → Add scope (e.g., `access_as_user`).
    
* In SPA app → API permissions → add the scope from your backend. Grant admin consent if needed. This wires up the permission so your frontend can request the scope and Azure AD will issue an access token targeted to your API. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-expose-web-apis?utm_source=chatgpt.com)
    

---

# Step 6 — Machine-to-machine: Client Credentials Flow (server → server)

If a service (daemon/app) needs to call the API (no user), use the **Client Credentials** flow:

1. Create an app registration for the caller (confidential client) and create a client secret or certificate.
    
2. Give that caller **Application** permission to your API (Expose an API → add application permission OR select `/.default` scopes).
    
3. Acquire token (example curl):
    

```plaintext
curl -X POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token \
  -d 'client_id=<CLIENT_ID>&client_secret=<CLIENT_SECRET>&grant_type=client_credentials&scope=api://<API_CLIENT_ID>/.default'
```

The returned access token is for the app (contains `roles` claim if app roles granted). Use this token to call the API. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow?utm_source=chatgpt.com)

---

# Step 7 — Troubleshooting & common pitfalls

* **Using implicit flow for SPAs**: legacy and less secure. Use auth code + PKCE. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/msal-authentication-flows?utm_source=chatgpt.com)
    
* **Wrong audience (**`aud`): If API validates audience incorrectly (Graph vs your API), token validation fails. Ensure `aud` matches your API App ID URI or client-id as configured. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    
* **Token expired**: Check `exp` claim; refresh tokens are not available to SPAs in some flows; MSAL handles refresh via silent renew. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/msal/javascript/?utm_source=chatgpt.com)
    
* **Key rollover**: Always use `jwks_uri` / automatic key fetching so your server accepts new Microsoft signing keys when rotated. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/signing-key-rollover?utm_source=chatgpt.com)
    

---

# Security best practices

* **SPAs**: use Authorization Code + **PKCE** (no client secrets in browser). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/msal-authentication-flows?utm_source=chatgpt.com)
    
* **Don't store long-lived secrets in browser**. Use sessionStorage or in-memory storage for tokens; MSAL offers caching controls. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/msal/javascript/?utm_source=chatgpt.com)
    
* **Validate all tokens on server** (signature, `iss`, `aud`, `exp`, scopes/roles). Treat tokens as bearer credentials. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    
* **Use HTTPS** everywhere.
    
* **Use short token lifetimes** and rotate client secrets / certificates.
    
* For service-to-service, use **Client Credentials** (confidential client) instead of API keys — prefer OAuth for least privilege and revocation support. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow?utm_source=chatgpt.com)
    

---

# Quick reference links (official)

* Register an app (quickstart) — Microsoft Entra ID (Azure AD). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?utm_source=chatgpt.com)
    
* OAuth 2.0 Auth Code Flow (authorization code with PKCE) — Microsoft identity platform. [Microsoft Learn+1](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow?utm_source=chatgpt.com)
    
* MSAL.js overview & docs. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/msal/javascript/?utm_source=chatgpt.com)
    
* Expose an API & configure scopes. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-expose-web-apis?utm_source=chatgpt.com)
    
* Access token validation guidance. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens?utm_source=chatgpt.com)
    
* Client Credentials flow (daemon apps). [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow?utm_source=chatgpt.com)