---
id: authentication-errors
title: Authentication Error Reference
sidebar_label: Authentication Errors
description: Diagnose and resolve common OAuth 2.0 / OIDC authentication errors across K-Series environments.
keywords:
  - OAuth2
  - authentication
  - errors
  - troubleshooting
  - K-Series
---

# Authentication Error Reference

This guide covers the most common OAuth 2.0 and OpenID Connect errors encountered when integrating with the K-Series API. Each section includes the error message, root causes, and resolution steps.

---

## Environment Quick Reference

Before troubleshooting, confirm you are targeting the correct environment. Trial and production are **fully isolated** — clients, users, and tokens are not shared between them.

| Environment | Authorization Endpoint | Token Endpoint       | Client ID Prefix |
| ----------- | ---------------------- | -------------------- | ---------------- |
| Trial       | `auth.lsk-trial.app`   | `auth.lsk-trial.app` | `devp-v2-demo`   |
| Production  | `auth.lsk-prod.app`    | `auth.lsk-prod.app`  | `devp-v2-prod`   |

> [!TIP]
> You can identify your environment from your client ID. If it contains `devp-v2-demo`, you are on **trial**. If it contains `devp-v2` without the `demo` suffix, you are on **production**.

---

## Authorization Errors

These errors appear in the browser during the authorization step, before any token exchange occurs.

### Client Not Found (Wrong Environment)

**Error message:** `Client not found`

> [!WARNING]
> Mixing trial and production credentials is one of the most common causes of this error. Always verify that your authorization URL matches your client's environment.

![Client Not Found](https://632df67d.oauthimages.pages.dev/assets/clientNotFound.png)

<details>

<summary>Example: trial client ID sent to the production endpoint</summary>

The following request targets `lsk-prod.app` (production), but the `client_id` prefix `devp-v2-demo` indicates it belongs to the **trial** environment. Because these environments are isolated, the production server cannot find the client.

```
https://auth.lsk-prod.app/realms/k-series/protocol/openid-connect/auth
  ?response_type=code
  &client_id=devp-v2-demo-dasxtrial-b69ffe954c5cd33c57a79a94780eb2ab
  &scope=financial-api
  &redirect_uri=https%3A%2F%2Flocalhost%2F
```

</details>

**Resolution:**
Match your authorization URL to your client's environment. Check the client ID prefix against the Environment Quick Reference table above and use the corresponding endpoint.

---

### Invalid Redirect URI

**Error message:** `Invalid parameter: redirect_uri`

> [!CAUTION]
> The `redirect_uri` must be an **exact string match** against the value registered on the server — including protocol, path, and trailing slash. Even a single character difference will cause this error.

![Invalid Redirect URI](https://632df67d.oauthimages.pages.dev/assets/invalid_redirect.png)

<details>

<summary>Common causes</summary>

| Context              | Value                |
| -------------------- | -------------------- |
| Used in request      | `https://localhost`  |
| Registered on server | `https://localhost/` |

This request fails because the registered URI has a trailing `/` and the request does not.

Other common causes include:

- Trailing slash added or removed
- Typo or character encoding mismatch (e.g., `%2f` vs. `/`)
- Using `http://` instead of `https://`

</details>

**Resolution:**
Copy the **exact** redirect URI from the credential provisioning email or developer portal and use it character-for-character. If the URI needs to be updated, contact Technical Partner Support with your client ID.

---

### Invalid Scope

**Error message:** `Invalid scopes: ...` (returned as a query parameter in the redirect)

> [!IMPORTANT]
> Scope names must exactly match the values configured on the authorization server. Underscores vs. hyphens and incorrect suffixes are frequent mistakes.

<details>

<summary>Valid scopes and common mistakes</summary>

**Common mistakes:**

| Invalid         | Correct         |
| --------------- | --------------- |
| `financial_api` | `financial-api` |
| `items-api`     | `items`         |

**Valid scopes for K-Series clients:**

- `financial-api`
- `orders-api`
- `items`
- `offline_access`
- `staff-api`

**Example error URL:**

```
https://localhost/?error=invalid_scope
  &error_description=Invalid+scopes%3A+financial-api+items+offline_access+orders-api
  &iss=https%3A%2F%2Fauth.lsk-prod.app%2Frealms%2Fk-series
```

</details>

> [!TIP]
> Multiple scopes must be **space-delimited** in the raw value before URL encoding. For example: `scope=financial-api%20orders-api%20items`. Do not use commas or other separators.

**Resolution:**
Cross-reference your requested scopes against the valid scope list above. Ensure you use hyphens where required (not underscores) and that scope names are space-separated before URL encoding. Only request scopes your client has been provisioned for.

---

### Invalid Login Credentials

**Error message:** `Invalid username or password`

> [!WARNING]
> Trial and production environments maintain **separate user directories**. Production credentials will not work in a trial environment and vice versa.

![Invalid Login Credentials](https://632df67d.oauthimages.pages.dev/assets/invalidlogincredential.png)

<details>

<summary>Possible causes</summary>

- Username or password is incorrect
- Using production credentials in a trial environment (or vice versa)
- The account does not exist in the selected product/realm
- The account may be locked or disabled

</details>

**Resolution:**
Confirm you are logging in to the correct environment. Verify the username and password. If the account may be locked, contact Technical Partner Support to check the account status in the appropriate realm.

---

## Token Exchange Errors

These errors are returned as JSON responses when exchanging the authorization code for tokens via `POST` to the token endpoint.

### Invalid Code Grant

**HTTP Status:** `400 Bad Request`

```json
{
  "error": "invalid_grant",
  "error_description": "Code not valid"
}
```

> [!CAUTION]
> Authorization codes are **single-use** and expire in approximately **15 seconds**. Do not reuse or cache them.

<details>

<summary>Possible causes</summary>

- The authorization code has expired (approximately 15-second window)
- The code has already been used (codes are strictly one-time use)
- The code is malformed, truncated, or contains a transcription error

</details>

**Resolution:**
Restart the authorization flow to obtain a fresh code. Ensure your application exchanges the code for tokens **immediately** after receiving it — automate this step rather than copying manually. Never attempt to reuse a code.

---

### Invalid Client or Invalid Client Credentials

**HTTP Status:** `401 Unauthorized`

```json
{
  "error": "invalid_client",
  "error_description": "Invalid client or Invalid client credentials"
}
```

> [!IMPORTANT]
> The `Authorization` header must be present and correctly formatted as `Basic base64(client_id:client_secret)`.

<details>

<summary>Possible causes</summary>

- The `Authorization` header is missing entirely
- Credentials are not Base64-encoded
- The `client_id` or `client_secret` is incorrect
- Using the wrong client credentials for the environment (trial vs. production)

</details>

**Resolution:**
Generate the Base64 value and set the header correctly:

```bash
# Generate the Base64-encoded credentials
echo -n "client_id:client_secret" | base64

# Use the output in the Authorization header
# Authorization: Basic <base64_value>
```

> [!TIP]
> The `-n` flag in `echo` is critical. Without it, a newline character is appended to the input, producing an incorrect Base64 value that will be silently rejected by the server.

---

### Incorrect Redirect URI (Token Exchange)

**HTTP Status:** `400 Bad Request`

```json
{
  "error": "invalid_grant",
  "error_description": "Incorrect redirect_uri"
}
```

> [!WARNING]
> The `redirect_uri` in the token exchange request must be **identical** to the one used in the original authorization request — even though no redirect actually occurs at this step. This is required by the OAuth 2.0 specification as a security measure.

<details>

<summary>Possible causes</summary>

- The `redirect_uri` in the token request does not match the one from the authorization step
- A trailing slash was added or removed between steps
- URL encoding differences between the two requests

</details>

**Resolution:**
Use the **exact same** `redirect_uri` value in both your authorization request and your token exchange request. Store the value in a constant or configuration variable to prevent drift between the two calls. Compare the raw strings if needed — even URL encoding differences will cause a mismatch.

---

## Refresh Token Errors

### Invalid Refresh Token

**HTTP Status:** `400 Bad Request`

```json
{
  "error": "invalid_grant",
  "error_description": "Token is not active"
}
```

> [!CAUTION]
> Refresh tokens are long-lived but not permanent. An expired, reused, or malformed refresh token cannot be recovered — a full re-authentication is required.

<details>

<summary>Possible causes</summary>

- The refresh token has passed its expiry window
- The token was already used and the server enforces token rotation
- The refresh token is malformed or contains a transcription error
- The token belongs to a different client or environment

</details>

**Resolution:**
Re-authenticate the user through the full authorization flow to obtain a new token pair. Your application should handle this gracefully: when a refresh fails with `invalid_grant`, redirect the user back through the authorization flow automatically rather than surfacing the raw error.

> [!TIP]
> Ensure `offline_access` is included in your requested scopes to receive refresh tokens. Without this scope, the authorization server will only return an access token.

---

## Client Configuration Errors

### Unauthorized Client (Grant Type Not Allowed)

**HTTP Status:** `400 Bad Request`

```json
{
  "error": "unauthorized_client",
  "error_description": "Client not allowed for direct access grants"
}
```

> [!IMPORTANT]
> This error occurs when the client attempts to use a `grant_type` that has not been enabled for it on the authorization server.

<details>

<summary>Possible causes</summary>

- The client is not configured for the `authorization_code` grant type
- The client is attempting the `client_credentials` flow but is only approved for the authorization code flow
- The client is attempting to use `refresh_token` as a grant type but refresh is not enabled
- The client configuration was changed or provisioned incorrectly

</details>

**Resolution:**
Confirm which grant type(s) your client has been provisioned for. If you require a different grant type (e.g., `client_credentials`), contact Technical Partner Support to update your client configuration. The most common K-Series integration flow is `authorization_code`.

---

## Quick Diagnostic Reference

Use this table to quickly identify the most likely cause based on the error you are seeing.

| Error Message                                  | Where It Appears          | Most Likely Cause                            | Section                                                                     |
| ---------------------------------------------- | ------------------------- | -------------------------------------------- | --------------------------------------------------------------------------- |
| `Client not found`                             | Browser (auth page)       | Wrong environment (trial/prod mismatch)      | [Client Not Found](#client-not-found-wrong-environment)                     |
| `Invalid parameter: redirect_uri`              | Browser (auth page)       | Redirect URI does not match registered value | [Invalid Redirect URI](#invalid-redirect-uri)                               |
| `Invalid scopes`                               | Redirect URL query params | Scope name typo or wrong delimiter           | [Invalid Scope](#invalid-scope)                                             |
| `Invalid username or password`                 | Browser (login form)      | Wrong credentials or wrong environment       | [Invalid Login Credentials](#invalid-login-credentials)                     |
| `Code not valid`                               | Token endpoint JSON       | Expired or reused authorization code         | [Invalid Code Grant](#invalid-code-grant)                                   |
| `Invalid client or Invalid client credentials` | Token endpoint JSON       | Missing or malformed Authorization header    | [Invalid Client Credentials](#invalid-client-or-invalid-client-credentials) |
| `Incorrect redirect_uri`                       | Token endpoint JSON       | URI mismatch between auth and token requests | [Redirect URI (Token Exchange)](#incorrect-redirect-uri-token-exchange)     |
| `Token is not active`                          | Token endpoint JSON       | Expired or reused refresh token              | [Invalid Refresh Token](#invalid-refresh-token)                             |
| `Client not allowed for direct access grants`  | Token endpoint JSON       | Grant type not enabled for client            | [Unauthorized Client](#unauthorized-client-grant-type-not-allowed)          |
