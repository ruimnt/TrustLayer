# Authentication
> **Audience:** Backend developers and security engineers.

> **Prerequisites:** API credentials from > **Developers › API keys** in the Console. See [Getting Started](01-getting-started.md#step-2-generate-api-credentials).
TrustLayer uses **OAuth 2.0 with the client credentials grant** for server-to-server requests. Your backend exchanges a client ID and client secret for a short-lived access token, then sends that token in the `Authorization` header of every API request.
For code that runs on a user's device (web or mobile SDK), TrustLayer uses a separate, single-use **SDK session token**. Your client secret never leaves your server.
## How authentication works

**[OAuth 2.0 client credentials flow]**

(**INSERIR IMAGEM AQUI**../../images/diagrams/auth-flow.svg)

1. Your backend sends its client ID and client secret to `POST /oauth/token`.
2. TrustLayer returns an access token valid for 3,600 seconds (1 hour).
3. Your backend caches the token.
4. Your backend calls the API with `Authorization: Bearer <access_token>`.
5. TrustLayer validates the token's signature, expiry and scope.
6. TrustLayer returns the response.
7. When the token expires, the API returns `401` with the code `token_expired`.
8. Your backend requests a new token and retries the request once.
## Request an access token
```http
POST /v1/oauth/token HTTP/1.1
Host: api.trustlayer.io
Content-Type: application/x-www-form-urlencoded
grant_type=client_credentials&client_id=tl_live_cid_4f9a2c71&client_secret=tl_live_sk_...&scope=verifications:write%20risk:write
```
### Body parameters
| Parameter | Type | Required | Description |
|---|---|---|---|
| `grant_type` | string | Yes | Always `client_credentials`. |
| `client_id` | string | Yes | Your client ID. Starts with `tl_test_cid_` or `tl_live_cid_`. |
| `client_secret` | string | Yes | Your client secret. Starts with `tl_test_sk_` or `tl_live_sk_`. |
| `scope` | string | No | Space-separated list of scopes. Defaults to all scopes granted to the credentials. |
> [!TIP]
> You can also send the client ID and secret with HTTP Basic authentication (`Authorization: Basic base64(client_id:client_secret)`) instead of in the body. Both methods are supported.
### Response
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6InRsLTIwMjYtMDkifQ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "verifications:write risk:write"
}
```
| Field | Type | Description |
|---|---|---|
| `access_token` | string | A signed JWT. Treat it as an opaque string; its format can change. |
| `token_type` | string | Always `Bearer`. |
| `expires_in` | integer | Lifetime of the token, in seconds. |
| `scope` | string | Scopes actually granted. Can be narrower than what you requested. |
## Call the API with the token
Send the token in the `Authorization` header:
```bash
curl https://api.trustlayer.io/v1/verifications?limit=10 \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6InRsLTIwMjYtMDkifQ..."
```
> [!WARNING]
> Always use HTTPS. TrustLayer rejects plain HTTP requests and requires TLS 1.2 or later.
## Scopes
Request only the scopes your service needs. This limits the impact if a token leaks.
| Scope | Allows |
|---|---|
| `verifications:read` | Read verifications, applicants and documents. |
| `verifications:write` | Create, update, submit and cancel verifications. Includes `verifications:read`. |
| `risk:read` | Read risk assessments. |
| `risk:write` | Create risk assessments. Includes `risk:read`. |
| `webhooks:manage` | Create, update and delete webhook endpoints. |
| `reports:read` | Download exported reports. |
## Cache and refresh tokens
Requesting a new token for every API call adds latency and can trigger rate limits on `/oauth/token` (60 requests per minute per client). Cache the token and refresh it shortly before it expires.
**Node.js**
```javascript
// tokenProvider.js — caches the token and refreshes it 60 seconds before expiry
let cached = { token: null, expiresAt: 0 };
export async function getAccessToken() {
  const now = Date.now();
  if (cached.token && now < cached.expiresAt - 60_000) {
    return cached.token;
  }
  const response = await fetch(`${process.env.TRUSTLAYER_BASE_URL}/oauth/token`, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "client_credentials",
      client_id: process.env.TRUSTLAYER_CLIENT_ID,
      client_secret: process.env.TRUSTLAYER_CLIENT_SECRET,
      scope: "verifications:write risk:write",
    }),
  });
  if (!response.ok) {
    throw new Error(`TrustLayer auth failed with status ${response.status}`);
  }
  const data = await response.json();
  cached = { token: data.access_token, expiresAt: now + data.expires_in * 1000 };
  return cached.token;
}
```
**Python**
```python
# token_provider.py — caches the token and refreshes it 60 seconds before expiry
import os
import time
import requests
_cache = {"token": None, "expires_at": 0.0}
def get_access_token() -> str:
    if _cache["token"] and time.time() < _cache["expires_at"] - 60:
        return _cache["token"]
    response = requests.post(
        f"{os.environ['TRUSTLAYER_BASE_URL']}/oauth/token",
        data={
            "grant_type": "client_credentials",
            "client_id": os.environ["TRUSTLAYER_CLIENT_ID"],
            "client_secret": os.environ["TRUSTLAYER_CLIENT_SECRET"],
            "scope": "verifications:write risk:write",
        },
        timeout=10,
    )
    response.raise_for_status()
    data = response.json()
    _cache.update(token=data["access_token"], expires_at=time.time() + data["expires_in"])
    return _cache["token"]
```
> [!NOTE]
> If you run several server instances, share the cached token through a store such as Redis. Otherwise, each instance requests its own token, which is allowed but less efficient.
## SDK session tokens (client-side)
The TrustLayer Web and Mobile SDKs run on your user's device, so they can't hold your client secret. Instead, your backend creates a **session token** scoped to a single verification and passes it to the SDK.
```bash
curl -X POST https://api.trustlayer.io/v1/verifications/ver_8kQ2mX4pL9/sdk_token \
  -H "Authorization: Bearer $TOKEN"
```
```json
{
  "sdk_token": "ses_Zx81KqP0aT3mV7",
  "verification_id": "ver_8kQ2mX4pL9",
  "expires_at": "2026-10-08T18:30:00Z"
}
```
Session tokens:
- Are valid for 30 minutes.
- Can only upload media and submit the verification they were created for.
- Can't read results. Your backend reads results with its own access token.
## Authentication errors
| HTTP status | Error code | Cause | What to do |
|---|---|---|---|
| `400` | `invalid_request` | A required parameter is missing in `/oauth/token`. | Check that you send `grant_type`, `client_id` and `client_secret`. |
| `401` | `invalid_client` | The client ID or secret is wrong, or the credentials are suspended. | Check the credentials in the Console. Confirm you're using the right environment. |
| `401` | `missing_token` | The `Authorization` header is missing or malformed. | Send `Authorization: Bearer <token>`. |
| `401` | `token_expired` | The access token has expired. | Request a new token and retry once. |
| `403` | `insufficient_scope` | The token doesn't include the scope this endpoint requires. | Request a token with the required scope. |
| `403` | `environment_mismatch` | A `tl_test_` token was sent to production, or the reverse. | Use the base URL that matches your credentials. |
For the full error format, see [Error Codes](08-error-codes.md).
## Security best practices
- [ ] Store the client secret in a secrets manager (for example, AWS Secrets Manager or HashiCorp Vault), never in source code.
- [ ] Use separate credentials for each service and environment, so you can revoke one without affecting the others.
- [ ] Rotate client secrets at least every 90 days. The Console lets two secrets be active at the same time during rotation.
- [ ] Restrict production credentials to your server IPs in **Developers › API keys › IP allowlist**.
- [ ] Never log access tokens or client secrets.
> [!CAUTION]
> If you suspect a client secret has leaked, suspend the credentials immediately in **Developers › API keys**. Suspension takes effect within 60 seconds and invalidates all tokens issued with those credentials.
## Related
- [Getting Started](01-getting-started.md)
- [Error Codes](08-error-codes.md)
- Admin Guide: [Managing users](../admin/03-managing-users.md) (who can create API credentials)
