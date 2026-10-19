# Risk Score API
> **Audience:** Backend developers adding fraud checks to signups, logins and transactions; fraud and risk analysts who read the results.
> **Prerequisites:** An access token with the `risk:write` scope. See [Authentication](02-authentication.md).
The Risk Score API evaluates an event in your platform, such as a signup, login or payment, and returns a **risk score from 0 to 100**, a **recommendation** and the **fraud signals** that explain the score.
Identity verifications include a risk assessment automatically. Use this API directly when you want to score events that happen *after* onboarding, for example a large withdrawal from a new device.
## Endpoints at a glance
| Method | Endpoint | Description |
|---|---|---|
| `POST` | [`/risk/assessments`](#create-a-risk-assessment) | Score an event. |
| `GET` | [`/risk/assessments/{assessment_id}`](#retrieve-a-risk-assessment) | Retrieve an assessment. |
| `POST` | [`/risk/assessments/{assessment_id}/feedback`](#report-the-real-outcome-feedback) | Tell TrustLayer whether the event turned out to be fraud. |
## How to read a score
[Risk score scale]

(**INSERIR IMAGEM AQUI**../../images/diagrams/risk-score-scale.svg)

| Level | Score | `recommendation` | Suggested action |
|---|---|---|---|
| `low` | 0–29 | `allow` | Let the event proceed. |
| `medium` | 30–59 | `allow` | Let the event proceed and monitor the account. |
| `high` | 60–84 | `review` | Hold the event, or add friction such as two-factor authentication. |
| `critical` | 85–100 | `deny` | Block the event. |
> [!NOTE]
> Thresholds are defaults. Your compliance team can change them, and add rules that override the recommendation, in the Console. Always act on the `recommendation` field rather than hard-coding score ranges in your application.
## Create a risk assessment
```http
POST /v1/risk/assessments
```
The response is synchronous. The p95 latency is under 300 ms, so you can call it inline during a checkout or login.
**Body parameters**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `event_type` | enum | Yes | `signup`, `login`, `transaction`, `withdrawal`, `profile_change` or `password_reset`. |
| `applicant_id` | string | No | Links the assessment to a verified applicant. Strongly recommended after onboarding. |
| `external_id` | string | No | Your user ID. Used if you don't send `applicant_id`. |
| `context.ip_address` | string | Yes | IPv4 or IPv6 address of the end user (not your server). |
| `context.device_id` | string | No | Device fingerprint from the TrustLayer SDK. |
| `context.user_agent` | string | No | The user's browser or app user agent. |
| `identity.email` | string | No | Email used in the event. |
| `identity.phone` | string | No | Phone number in E.164 format. |
| `transaction.amount` | number | Conditional | Amount in major units (for example `5200.00`). Required when `event_type` is `transaction` or `withdrawal`. |
| `transaction.currency` | string | Conditional | ISO 4217 code, for example `BRL`. Required with `amount`. |
| `transaction.destination` | object | No | Destination account, for example `{ "type": "pix", "key_hash": "sha256:9f2c…" }`. |
| `rule_set_id` | string | No | Rule set to apply. Defaults to the account's default rule set. |
**Example request**
```bash
curl -X POST https://api.trustlayer.io/v1/risk/assessments \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 8e4b2f7a-withdrawal-99812" \
  -d '{
    "event_type": "withdrawal",
    "applicant_id": "app_9Kd2pL6xQ1",
    "context": {
      "ip_address": "185.220.101.7",
      "device_id": "dvc_new_c03f9a",
      "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
    },
    "transaction": {
      "amount": 5200.00,
      "currency": "BRL",
      "destination": { "type": "pix", "key_hash": "sha256:9f2c41e0b7" }
    }
  }'
```
**Response** — `201 Created`
```json
{
  "id": "ra_2Kp9vM4tR8",
  "object": "risk_assessment",
  "event_type": "withdrawal",
  "applicant_id": "app_9Kd2pL6xQ1",
  "score": 88,
  "level": "critical",
  "recommendation": "deny",
  "baseline": 20,
  "signals": [
    {
      "code": "IP_TOR",
      "category": "network",
      "severity": "high",
      "contribution": 35,
      "details": { "ip_address": "185.220.101.7", "exit_node": true }
    },
    {
      "code": "DEVICE_NEW",
      "category": "device",
      "severity": "low",
      "contribution": 12,
      "details": { "first_seen_at": "2026-10-08T18:41:02Z" }
    },
    {
      "code": "AMOUNT_UNUSUAL",
      "category": "behavior",
      "severity": "medium",
      "contribution": 21,
      "details": { "amount": 5200.0, "applicant_p95_amount": 640.0 }
    }
  ],
  "rules_matched": [
    {
      "rule_id": "rule_tor_block",
      "name": "Block Tor exit nodes",
      "action": "reject"
    }
  ],
  "model_version": "risk-2026.09",
  "created_at": "2026-10-08T18:41:05Z"
}
```
**Response fields**
| Field | Type | Description |
|---|---|---|
| `score` | integer | Final risk score, from 0 to 100. |
| `level` | enum | `low`, `medium`, `high` or `critical`. |
| `recommendation` | enum | `allow`, `review` or `deny`. Already includes the effect of matched rules. |
| `baseline` | integer | Starting score before signals are applied. |
| `signals` | array | Signals detected, sorted by `contribution` (largest first). See [Fraud Signals](07-fraud-signals.md). |
| `signals[].contribution` | integer | Points this signal added to (positive) or removed from (negative) the score. |
| `rules_matched` | array | Risk rules that matched, in priority order. |
| `model_version` | string | Version of the scoring model. Useful when comparing scores over time. |
> [!TIP]
> The score is `baseline` plus the sum of every `contribution`, limited to the range 0–100. In the example: 20 + 35 + 12 + 21 = 88. You can show this breakdown in your internal back office to help analysts. See the Admin Guide: [Understanding risk scores](../admin/05-understanding-risk-scores.md).
**Errors:** `400 validation_error`, `404 applicant_not_found`, `422 missing_transaction_amount`.
### Act on the recommendation
```javascript
const assessment = await trustlayer.post("/risk/assessments", payload);
switch (assessment.recommendation) {
  case "allow":
    return processWithdrawal(request);
  case "review":
    // Add friction instead of blocking outright
    return requireStepUpAuthentication(request, assessment.id);
  case "deny":
    await logBlockedAttempt(request, assessment.id);
    return { status: "blocked", message: "We couldn't complete this request." };
}
```
> [!WARNING]
> If the Risk Score API is unavailable (a timeout or `5xx` error), decide on a fallback in advance. Most teams **fail open** for low-value events (allow and score later) and **fail closed** for high-value events (hold for review). Set a client timeout of 2 seconds.
## Retrieve a risk assessment
```http
GET /v1/risk/assessments/{assessment_id}
```
| Path parameter | Type | Description |
|---|---|---|
| `assessment_id` | string | The assessment ID, for example `ra_2Kp9vM4tR8`. |
Returns the [risk assessment object](#create-a-risk-assessment). Assessments are kept for 5 years for audit purposes.
**Errors:** `404 resource_not_found`.
## Report the real outcome (feedback)
```http
POST /v1/risk/assessments/{assessment_id}/feedback
```
When you learn the real outcome of an event (for example, a chargeback 30 days later), report it. TrustLayer uses feedback to improve the model for your account.
| Parameter | Type | Required | Description |
|---|---|---|---|
| `outcome` | enum | Yes | `confirmed_fraud`, `confirmed_legitimate` or `chargeback`. |
| `fraud_type` | enum | No | `account_takeover`, `synthetic_identity`, `identity_theft`, `money_mule` or `other`. |
| `notes` | string | No | Free text, up to 500 characters. |
```bash
curl -X POST https://api.trustlayer.io/v1/risk/assessments/ra_2Kp9vM4tR8/feedback \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "outcome": "confirmed_fraud", "fraud_type": "account_takeover" }'
```
**Response** — `202 Accepted`
```json
{ "assessment_id": "ra_2Kp9vM4tR8", "feedback_status": "received" }
```
## Related
- [Fraud Signals](07-fraud-signals.md): every signal code and what it means
- [Webhooks](06-webhooks.md): `risk.assessment.completed` event
- Admin Guide: [Configuring risk rules](../admin/02-configuring-risk-rules.md)
