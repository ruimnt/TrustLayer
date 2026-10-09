# Identity Verification API
> **Audience:** Backend developers who need the complete reference for verifications and applicants.
> **Prerequisites:** An access token with the `verifications:write` scope. See [Authentication](02-authentication.md).
The Identity Verification API lets you create applicants, run them through a KYC workflow, upload their identity documents and selfies, and read the result.
**Base URLs**
```text
Production: https://api.trustlayer.io/v1
Sandbox:    https://sandbox.api.trustlayer.io/v1
```
## Endpoints at a glance
| Method | Endpoint | Description |
|---|---|---|
| `POST` | [`/applicants`](#create-an-applicant) | Create an applicant. |
| `GET` | [`/applicants/{applicant_id}`](#retrieve-an-applicant) | Retrieve an applicant. |
| `POST` | [`/verifications`](#create-a-verification) | Create a verification. |
| `GET` | [`/verifications/{verification_id}`](#retrieve-a-verification) | Retrieve a verification and its result. |
| `GET` | [`/verifications`](#list-verifications) | List verifications, with filters. |
| `POST` | [`/verifications/{verification_id}/documents`](#upload-a-document) | Upload an identity document image. |
| `POST` | [`/verifications/{verification_id}/selfie`](#upload-a-selfie) | Upload a selfie or liveness video. |
| `POST` | [`/verifications/{verification_id}/submit`](#submit-a-verification) | Submit the verification for processing. |
| `POST` | [`/verifications/{verification_id}/cancel`](#cancel-a-verification) | Cancel a verification. |
## API conventions
These conventions apply to every TrustLayer endpoint.
| Topic | Convention |
|---|---|
| Format | Requests and responses use JSON (`Content-Type: application/json`), except file uploads, which use `multipart/form-data`. |
| Field names | `snake_case`. |
| Dates and times | ISO 8601 in UTC, for example `2026-10-08T18:00:00Z`. Dates of birth use `YYYY-MM-DD`. |
| Countries | ISO 3166-1 alpha-3 codes, for example `BRA`, `PRT`, `USA`. |
| IDs | Prefixed strings: `app_` (applicant), `ver_` (verification), `doc_` (document), `ra_` (risk assessment), `evt_` (event), `wf_` (workflow). |
| Request ID | Every response includes an `X-Request-Id` header. Include it when you contact support. |
### Idempotency
Network errors can make you unsure whether a request succeeded. To retry `POST` requests safely, send a unique `Idempotency-Key` header (for example, a UUID v4). If TrustLayer receives the same key within 24 hours, it returns the original response instead of creating a duplicate.
```http
Idempotency-Key: 3f6c1a2e-8d4b-4f7e-9a1c-2b5e7d9f0a13
```
> [!WARNING]
> Reusing an idempotency key with a *different* request body returns `409 idempotency_key_reused`. Generate a new key for every new operation.
### Pagination
List endpoints use cursor-based pagination.
| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | `20` | Number of items to return. Maximum `100`. |
| `starting_after` | string | — | Return items after this ID. Use the last ID from the previous page. |
The response contains `data` (the items) and `has_more` (whether another page exists).
### Rate limits
| Environment | Limit |
|---|---|
| Sandbox | 25 requests per second per account |
| Production | 100 requests per second per account (can be increased on request) |
Every response includes `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers. When you exceed the limit, the API returns `429 rate_limit_exceeded` with a `Retry-After` header.
---
## The verification object
```json
{
  "id": "ver_3Hn7aQ1zV2",
  "object": "verification",
  "status": "manual_review",
  "workflow_id": "wf_standard_kyc_br",
  "workflow_version": 4,
  "applicant_id": "app_9Kd2pL6xQ1",
  "external_id": "user_4471",
  "hosted_url": null,
  "checks": [
    { "type": "document_authenticity", "status": "passed" },
    { "type": "data_extraction", "status": "passed" },
    { "type": "face_match", "status": "passed", "similarity": 0.94 },
    { "type": "liveness", "status": "passed" },
    { "type": "sanctions_screening", "status": "passed" },
    { "type": "pep_screening", "status": "attention", "matches": 1 }
  ],
  "extracted_data": {
    "first_name": "JOAO",
    "last_name": "PEREIRA",
    "date_of_birth": "1991-03-14",
    "document_type": "drivers_license",
    "document_number": "0*****8812",
    "issuing_country": "BRA",
    "expiry_date": "2031-05-20"
  },
  "risk": {
    "assessment_id": "ra_7Vn2bX9cQ4",
    "score": 72,
    "level": "high",
    "signals": ["WATCHLIST_MATCH", "IP_VPN", "DEVICE_NEW", "EMAIL_AGE_OK"]
  },
  "decision": null,
  "created_at": "2026-10-08T16:58:40Z",
  "submitted_at": "2026-10-08T17:02:05Z",
  "updated_at": "2026-10-08T17:02:19Z"
}
```
| Field | Type | Description |
|---|---|---|
| `id` | string | Unique identifier, prefixed with `ver_`. |
| `status` | enum | Current status: `created`, `pending_input`, `processing`, `manual_review`, `approved`, `rejected`, `expired` or `cancelled`. See [KYC Workflow](04-kyc-workflow.md). |
| `workflow_id` | string | The workflow used for this verification. |
| `workflow_version` | integer | The published version of the workflow at creation time. |
| `applicant_id` | string | The applicant being verified. |
| `external_id` | string \| null | Your own identifier for the user, if provided. |
| `hosted_url` | string \| null | URL of the hosted verification flow. `null` after the applicant submits. |
| `checks` | array | Result of each check in the workflow. `status` is `passed`, `failed`, `attention` or `skipped`. |
| `extracted_data` | object \| null | Data read from the document. Sensitive numbers are masked. |
| `risk` | object \| null | Summary of the risk assessment. See [Risk Score API](05-risk-score-api.md). |
| `decision` | object \| null | Final outcome. `null` until the verification reaches `approved` or `rejected`. |
| `created_at`, `submitted_at`, `updated_at` | string | Timestamps in ISO 8601 (UTC). |
### The decision object
```json
{
  "outcome": "rejected",
  "decided_by": "analyst",
  "analyst_id": "usr_Fr3n4nd4",
  "reason_code": "document_tampered",
  "reason_text": "Photo area shows signs of digital editing.",
  "decided_at": "2026-10-08T17:20:44Z"
}
```
| Field | Type | Description |
|---|---|---|
| `outcome` | enum | `approved` or `rejected`. |
| `decided_by` | enum | `automatic` (decided by workflow rules) or `analyst` (decided in manual review). |
| `analyst_id` | string \| null | Console user who made the decision. |
| `reason_code` | string \| null | Machine-readable reason. See [reason codes](04-kyc-workflow.md#decision-reason-codes). |
| `reason_text` | string \| null | Human-readable reason. Don't show it to applicants; it can contain internal notes. |
| `decided_at` | string | When the decision was made. |
---
## Create an applicant
```http
POST /v1/applicants
```
Creates a person to verify. You can skip this call and send an `applicant` object inline when you [create a verification](#create-a-verification).
**Body parameters**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `first_name` | string | Yes | Given name, as shown on the document. |
| `last_name` | string | Yes | Family name. |
| `email` | string | No | Email address. Improves risk scoring. |
| `phone` | string | No | Phone number in E.164 format, for example `+5511987654321`. |
| `date_of_birth` | string | No | Date of birth (`YYYY-MM-DD`). |
| `country` | string | No | Country of residence (ISO 3166-1 alpha-3). |
| `tax_id` | object | No | Tax identifier, for example `{ "type": "cpf", "value": "12345678909" }`. |
| `external_id` | string | No | Your internal user ID. Must be unique per account. |
| `metadata` | object | No | Up to 20 key-value pairs of your own data. Values must be strings. |
**Example request**
```bash
curl -X POST https://api.trustlayer.io/v1/applicants \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 0b8f2d6e-1c4a-4e8b-9f3d-7a2c5e1b9d40" \
  -d '{
    "first_name": "João",
    "last_name": "Pereira",
    "email": "joao.pereira@example.com",
    "phone": "+5511987654321",
    "country": "BRA",
    "tax_id": { "type": "cpf", "value": "12345678909" },
    "external_id": "user_4471",
    "metadata": { "plan": "premium" }
  }'
```
**Response** — `201 Created`
```json
{
  "id": "app_9Kd2pL6xQ1",
  "object": "applicant",
  "first_name": "João",
  "last_name": "Pereira",
  "email": "joao.pereira@example.com",
  "phone": "+5511987654321",
  "country": "BRA",
  "tax_id": { "type": "cpf", "value": "*******8909" },
  "external_id": "user_4471",
  "metadata": { "plan": "premium" },
  "created_at": "2026-10-08T16:58:39Z"
}
```
**Errors:** `400 validation_error`, `409 duplicate_external_id`. See [Error Codes](08-error-codes.md).
## Retrieve an applicant
```http
GET /v1/applicants/{applicant_id}
```
| Path parameter | Type | Description |
|---|---|---|
| `applicant_id` | string | The applicant ID, for example `app_9Kd2pL6xQ1`. |
Returns the [applicant object](#create-an-applicant), plus `verification_ids`: the list of verifications created for the applicant, newest first.
**Errors:** `404 resource_not_found`.
---
## Create a verification
```http
POST /v1/verifications
```
Starts a verification for an applicant using a published workflow.
**Body parameters**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `workflow_id` | string | Yes | ID of a published workflow. Find it in the Console under **Workflows**. |
| `applicant_id` | string | Conditional | ID of an existing applicant. Required if you don't send `applicant`. |
| `applicant` | object | Conditional | Inline applicant data, with the same fields as [Create an applicant](#create-an-applicant). Required if you don't send `applicant_id`. |
| `capture_method` | enum | No | `hosted` (default): TrustLayer hosts the capture screens. `sdk`: you embed the Web or Mobile SDK. `api`: you upload media directly. |
| `redirect_url` | string | No | Where the hosted flow sends the applicant at the end. Required when `capture_method` is `hosted`. |
| `locale` | string | No | Language of the hosted flow: `pt-BR`, `en-US` or `es-ES`. Defaults to the browser language. |
| `context` | object | No | Device and network data used for risk scoring: `ip_address`, `user_agent`, `device_id`. |
**Example request**
```bash
curl -X POST https://api.trustlayer.io/v1/verifications \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 5d2a9c1f-7b3e-4a8d-b6f0-1e9c3a7d5b82" \
  -d '{
    "workflow_id": "wf_standard_kyc_br",
    "applicant_id": "app_9Kd2pL6xQ1",
    "capture_method": "hosted",
    "redirect_url": "https://acmebank.com.br/onboarding/done",
    "locale": "pt-BR",
    "context": {
      "ip_address": "203.0.113.24",
      "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X)",
      "device_id": "dvc_b72e11"
    }
  }'
```
**Response** — `201 Created`
```json
{
  "id": "ver_3Hn7aQ1zV2",
  "object": "verification",
  "status": "pending_input",
  "workflow_id": "wf_standard_kyc_br",
  "workflow_version": 4,
  "applicant_id": "app_9Kd2pL6xQ1",
  "capture_method": "hosted",
  "hosted_url": "https://verify.trustlayer.io/s/ses_Qm71Lp",
  "hosted_url_expires_at": "2026-10-15T16:58:40Z",
  "checks": [],
  "risk": null,
  "decision": null,
  "created_at": "2026-10-08T16:58:40Z"
}
```
> [!TIP]
> Always send the `context` object. IP address and device data feed several [fraud signals](07-fraud-signals.md), such as `IP_TOR` and `DEVICE_VELOCITY`. Without them, the risk score is less accurate.
**Errors:** `400 validation_error`, `404 workflow_not_found`, `409 workflow_not_published`, `409 verification_in_progress`.
## Retrieve a verification
```http
GET /v1/verifications/{verification_id}
```
Returns the [verification object](#the-verification-object).
| Query parameter | Type | Description |
|---|---|---|
| `expand[]` | string | Include related objects in full. Supported values: `applicant`, `risk.assessment`, `documents`. |
```bash
curl "https://api.trustlayer.io/v1/verifications/ver_3Hn7aQ1zV2?expand[]=risk.assessment" \
  -H "Authorization: Bearer $TOKEN"
```
**Errors:** `404 resource_not_found`.
## List verifications
```http
GET /v1/verifications
```
| Query parameter | Type | Description |
|---|---|---|
| `status` | enum | Filter by status. Repeat to filter by several, for example `status=approved&status=rejected`. |
| `applicant_id` | string | Only verifications for this applicant. |
| `workflow_id` | string | Only verifications that used this workflow. |
| `created_gte`, `created_lt` | string | Creation date range (ISO 8601). |
| `limit`, `starting_after` | — | See [Pagination](#pagination). |
```bash
curl "https://api.trustlayer.io/v1/verifications?status=manual_review&limit=2" \
  -H "Authorization: Bearer $TOKEN"
```
```json
{
  "object": "list",
  "data": [
    { "id": "ver_3Hn7aQ1zV2", "object": "verification", "status": "manual_review", "...": "..." },
    { "id": "ver_6Yb2kP0qR4", "object": "verification", "status": "manual_review", "...": "..." }
  ],
  "has_more": true
}
```
---
## Upload a document
```http
POST /v1/verifications/{verification_id}/documents
Content-Type: multipart/form-data
```
Use this endpoint only when `capture_method` is `api`. With the hosted flow or SDK, TrustLayer captures media for you.
**Form fields**
| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | Yes | JPEG, PNG or PDF. Maximum 10 MB. Minimum 1000 × 700 px for images. |
| `type` | enum | Yes | `national_id`, `drivers_license`, `passport` or `residence_permit`. |
| `side` | enum | Conditional | `front` or `back`. Required for two-sided documents. |
| `issuing_country` | string | Yes | ISO 3166-1 alpha-3, for example `BRA`. |
```bash
curl -X POST https://api.trustlayer.io/v1/verifications/ver_3Hn7aQ1zV2/documents \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@./cnh-front.jpg" \
  -F "type=drivers_license" \
  -F "side=front" \
  -F "issuing_country=BRA"
```
**Response** — `201 Created`
```json
{
  "id": "doc_4Gh8sK2mW6",
  "object": "document",
  "verification_id": "ver_3Hn7aQ1zV2",
  "type": "drivers_license",
  "side": "front",
  "issuing_country": "BRA",
  "quality": { "status": "accepted", "issues": [] },
  "created_at": "2026-10-08T17:00:11Z"
}
```
If the image is unusable, `quality.status` is `rejected` and `issues` explains why, for example `["glare", "cropped_corner"]`. Ask the user to take a new photo.
**Errors:** `400 validation_error`, `409 invalid_status` (the verification is no longer accepting media), `413 file_too_large`, `415 unsupported_media_type`.
## Upload a selfie
```http
POST /v1/verifications/{verification_id}/selfie
Content-Type: multipart/form-data
```
| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | Yes | JPEG or PNG for a still selfie; MP4 (up to 10 seconds) for a liveness video. |
| `liveness_mode` | enum | No | `passive` (single image, default) or `active` (video with head movement). Must match the workflow configuration. |
Returns a `201 Created` with a `selfie` object containing `id` and `quality`.
## Submit a verification
```http
POST /v1/verifications/{verification_id}/submit
```
Tells TrustLayer that all media is uploaded. The status changes from `pending_input` to `processing`. This call has no body.
**Response** — `202 Accepted` with the verification object.
> [!NOTE]
> Processing is asynchronous and usually takes 5–30 seconds. Listen for the `verification.completed` or `verification.manual_review` [webhook event](06-webhooks.md) instead of polling.
**Errors:** `409 invalid_status`, `422 missing_required_media` (for example, the back side of a national ID is missing).
## Cancel a verification
```http
POST /v1/verifications/{verification_id}/cancel
```
Cancels a verification in `created` or `pending_input` status. The hosted URL stops working immediately. You can't cancel a verification that is already processing or finished.
**Response** — `200 OK` with the verification object and `"status": "cancelled"`.
**Errors:** `409 invalid_status`.
## Related
- [KYC Workflow](04-kyc-workflow.md): statuses and transitions
- [Webhooks](06-webhooks.md): get notified when a verification finishes
- [Error Codes](08-error-codes.md)
- [OpenAPI specification](../../api/openapi.yaml)
