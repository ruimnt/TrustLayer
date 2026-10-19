# Webhooks
> **Audience:** Backend developers who need to react to verification results and risk events in real time.
> **Prerequisites:** A public HTTPS endpoint in your backend, and an access token with the `webhooks:manage` scope.
Verifications are asynchronous: checks take seconds, and manual reviews can take hours. Instead of polling the API, register a **webhook endpoint**. TrustLayer sends an HTTP `POST` request to that endpoint every time something important happens.
## How delivery works

**[Webhook delivery and retry schedule]**

(**INSERIR IMAGEM AQUI**../../images/diagrams/webhook-delivery.svg)

1. An event occurs, for example a verification is approved.
2. TrustLayer signs the payload with your endpoint's **signing secret**.
3. TrustLayer sends a `POST` request to your URL.
4. Your endpoint verifies the signature, stores the event, and returns any `2xx` status within 10 seconds.
5. If your endpoint times out or returns another status, TrustLayer retries up to 7 more times over 24 hours.
## Event types
| Event | Sent when |
|---|---|
| `verification.created` | A verification is created. |
| `verification.manual_review` | A verification enters `manual_review`. |
| `verification.resubmission_requested` | An analyst asks the applicant for new documents. |
| `verification.completed` | A verification reaches `approved` or `rejected`. |
| `verification.expired` | A verification expires after 7 days without input. |
| `risk.assessment.completed` | A risk assessment is created with `recommendation` other than `allow`. |
| `applicant.watchlist_hit` | Ongoing monitoring finds a new sanctions or PEP match for an approved applicant. |
| `review.decision_made` | An analyst makes a decision in the Console. Includes the analyst ID. |
> [!TIP]
> Most integrations only need `verification.completed`, `verification.manual_review` and `applicant.watchlist_hit`. Subscribe only to the events you handle, to reduce traffic.
## Register an endpoint
You can register endpoints in the Console (**Developers › Webhooks**) or through the API.
```http
POST /v1/webhooks
```
| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | string | Yes | HTTPS URL of your endpoint. |
| `events` | array | Yes | Event types to receive. Use `["*"]` for all events. |
| `description` | string | No | A note for your team, for example `Production onboarding service`. |
```bash
curl -X POST https://api.trustlayer.io/v1/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://api.acmebank.com.br/webhooks/trustlayer",
    "events": ["verification.completed", "verification.manual_review", "applicant.watchlist_hit"],
    "description": "Onboarding service (production)"
  }'
```
**Response** — `201 Created`
```json
{
  "id": "whk_6Fq1nT8zP3",
  "object": "webhook_endpoint",
  "url": "https://api.acmebank.com.br/webhooks/trustlayer",
  "events": ["verification.completed", "verification.manual_review", "applicant.watchlist_hit"],
  "status": "enabled",
  "signing_secret": "whsec_K8s2Xv9qL4mN7pR1",
  "created_at": "2026-10-08T19:00:00Z"
}
```
> [!WARNING]
> The `signing_secret` is returned only once, at creation. Store it securely. If you lose it, roll the secret in the Console.
Other webhook endpoints:
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/webhooks` | List endpoints. |
| `GET` | `/webhooks/{webhook_id}` | Retrieve an endpoint (without the secret). |
| `PATCH` | `/webhooks/{webhook_id}` | Update `url`, `events` or `status` (`enabled` / `disabled`). |
| `DELETE` | `/webhooks/{webhook_id}` | Delete an endpoint. |
| `POST` | `/webhooks/{webhook_id}/roll_secret` | Generate a new signing secret. The old one stays valid for 24 hours. |
## Event payload
Every event has the same envelope. The `data.object` field contains the full resource at the time of the event.
```http
POST /webhooks/trustlayer HTTP/1.1
Host: api.acmebank.com.br
Content-Type: application/json
User-Agent: TrustLayer-Webhooks/1.0
TrustLayer-Event-Id: evt_1Bz7kR3qW9
TrustLayer-Signature: t=1791480045,v1=5f1b3c9e2a7d4f8b0c6e1a9d3b7f2c5e8a0d4b6f9c1e3a7d5b2f8c0e6a4d9b1f
```
```json
{
  "id": "evt_1Bz7kR3qW9",
  "object": "event",
  "type": "verification.completed",
  "api_version": "2026-06-01",
  "created_at": "2026-10-08T17:20:45Z",
  "livemode": true,
  "data": {
    "object": {
      "id": "ver_3Hn7aQ1zV2",
      "object": "verification",
      "status": "approved",
      "applicant_id": "app_9Kd2pL6xQ1",
      "external_id": "user_4471",
      "risk": { "score": 72, "level": "high" },
      "decision": {
        "outcome": "approved",
        "decided_by": "analyst",
        "reason_code": null,
        "decided_at": "2026-10-08T17:20:44Z"
      }
    }
  }
}
```
| Field | Description |
|---|---|
| `id` | Unique event ID. Use it to detect duplicates. |
| `type` | The [event type](#event-types). |
| `api_version` | API version used to render `data.object`. |
| `livemode` | `true` in production, `false` in the sandbox. |
| `data.object` | The resource that changed. |
## Verify the signature
Anyone can send a request to a public URL. Verify the `TrustLayer-Signature` header on every request to confirm that it came from TrustLayer and wasn't changed.
The header has two parts:
- `t`: Unix timestamp of when TrustLayer sent the request.
- `v1`: HMAC-SHA256 of the string `{t}.{raw_request_body}`, using your signing secret as the key, encoded in hexadecimal.
To verify:
1. Extract `t` and `v1` from the header.
2. Build the signed payload: the timestamp, a period (`.`), and the **raw** request body.
3. Calculate the HMAC-SHA256 of the signed payload with your signing secret.
4. Compare your result to `v1` using a constant-time comparison.
5. Reject the request if `t` is more than 5 minutes old, to prevent replay attacks.
> [!IMPORTANT]
> Use the raw request body exactly as received. If your framework parses the JSON and you serialize it again, whitespace and key order can change and the signature won't match.
**Node.js (Express)**
```javascript
import crypto from "node:crypto";
import express from "express";
const app = express();
const SIGNING_SECRET = process.env.TRUSTLAYER_WEBHOOK_SECRET;
const TOLERANCE_SECONDS = 300;
// Use express.raw so the body is not parsed before verification
app.post("/webhooks/trustlayer", express.raw({ type: "application/json" }), async (req, res) => {
  const header = req.get("TrustLayer-Signature") ?? "";
  const parts = Object.fromEntries(header.split(",").map((p) => p.split("=")));
  const timestamp = Number(parts.t);
  if (!timestamp || Math.abs(Date.now() / 1000 - timestamp) > TOLERANCE_SECONDS) {
    return res.status(400).send("Invalid timestamp");
  }
  const expected = crypto
    .createHmac("sha256", SIGNING_SECRET)
    .update(`${timestamp}.${req.body}`)
    .digest("hex");
  const valid =
    parts.v1 &&
    parts.v1.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(parts.v1), Buffer.from(expected));
  if (!valid) {
    return res.status(400).send("Invalid signature");
  }
  const event = JSON.parse(req.body);
  await saveEventForProcessing(event); // store first, process asynchronously
  res.sendStatus(200);
});
```
**Python (Flask)**
```python
import hashlib
import hmac
import os
import time
from flask import Flask, abort, request
app = Flask(__name__)
SIGNING_SECRET = os.environ["TRUSTLAYER_WEBHOOK_SECRET"].encode()
TOLERANCE_SECONDS = 300
@app.post("/webhooks/trustlayer")
def trustlayer_webhook():
    header = request.headers.get("TrustLayer-Signature", "")
    parts = dict(item.split("=", 1) for item in header.split(",") if "=" in item)
    try:
        timestamp = int(parts["t"])
    except (KeyError, ValueError):
        abort(400, "Invalid signature header")
    if abs(time.time() - timestamp) > TOLERANCE_SECONDS:
        abort(400, "Timestamp outside tolerance")
    raw_body = request.get_data()  # raw bytes, not parsed JSON
    signed_payload = f"{timestamp}.".encode() + raw_body
    expected = hmac.new(SIGNING_SECRET, signed_payload, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, parts.get("v1", "")):
        abort(400, "Invalid signature")
    save_event_for_processing(request.get_json())  # store first, process asynchronously
    return "", 200
```
## Best practices
- **Respond fast.** Return `2xx` as soon as you've verified and stored the event. Do slow work (sending emails, updating other systems) in a background job.
- **Handle duplicates.** TrustLayer delivers each event *at least once*, so you can receive the same event twice. Store `event.id` and ignore events you've already processed.
- **Don't rely on order.** Events can arrive out of order. Check `data.object.status` or fetch the latest state with `GET /verifications/{id}` before acting.
- **Allowlist TrustLayer IPs** if your firewall requires it. The current list is at `https://api.trustlayer.io/v1/meta/webhook-ips`.
## Retries and failures
| Attempt | Time after the first attempt |
|---|---|
| 1 | Immediately |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |
| 6 | 6 hours |
| 7 | 12 hours |
| 8 | 24 hours |
After 8 failed attempts, the event is marked as `failed` and admins receive an email. You can replay failed events from **Developers › Webhooks › Event log**.
> [!CAUTION]
> If an endpoint fails every delivery for 3 consecutive days, TrustLayer disables it automatically. Re-enable it in the Console after you fix the problem.
## Test your endpoint
- **In the sandbox**, trigger any event on demand. See [Sandbox Environment](09-sandbox-environment.md#trigger-webhook-events).
- **Locally**, expose your local server with a tunnel tool such as ngrok, and register the tunnel URL as a sandbox endpoint.
## Related
- [KYC Workflow](04-kyc-workflow.md): when each verification event fires
- [Integration Guide](10-integration-guide.md): how webhooks fit into a complete integration
