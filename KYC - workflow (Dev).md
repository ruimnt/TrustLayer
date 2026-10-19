# KYC Workflow
> **Audience:** Developers who need to understand how a verification moves from creation to a final decision, and product managers designing the onboarding experience.
> **Prerequisites:** Familiarity with [verifications](03-identity-verification-api.md#the-verification-object).
A **workflow** is the sequence of checks a verification goes through, for example: document check → selfie and liveness → watchlist screening → risk score → decision. Your compliance team builds and publishes workflows in the Admin Console. Your code only needs to know the `workflow_id` and react to the status changes described on this page.
> [!NOTE]
> Developers don't configure checks or thresholds through the API. This keeps compliance decisions with the compliance team and lets them change rules without a code deploy. To learn how workflows are built, see the Admin Guide: [Creating a verification workflow](../admin/01-creating-a-verification-workflow.md).
## Workflow steps
Each workflow combines some of these steps, in the order your compliance team chooses.
| Step type | What it checks | Typical duration |
|---|---|---|
| `document_check` | The document is genuine, unexpired and not tampered with. Extracts name, date of birth and document number. | 3–8 s |
| `selfie_liveness` | A real person is present (not a photo, screen or mask), and their face matches the document photo. | 2–5 s |
| `watchlist_screening` | The applicant isn't on sanctions lists, PEP (politically exposed person) lists or adverse media lists. | 1–3 s |
| `database_lookup` | The document data matches government or credit bureau records (for example, CPF in Brazil). | 2–4 s |
| `address_proof` | A proof of address document is valid and recent. | 3–6 s |
| `risk_assessment` | Calculates a [risk score](05-risk-score-api.md) from all signals collected. | < 1 s |
## Verification lifecycle
Every verification has a `status`. The diagram shows every possible transition.
![Verification lifecycle](../../images/diagrams/kyc-workflow.svg)
| Status | Meaning | Final? | What your app should do |
|---|---|---|---|
| `created` | The verification exists, but the applicant hasn't started. | No | Send the applicant to the `hosted_url` or start the SDK. |
| `pending_input` | TrustLayer is waiting for the applicant to provide documents or a selfie. | No | Show a "continue verification" option if the user leaves. |
| `processing` | All input received. Checks are running. | No | Show a waiting screen. Wait for a webhook. |
| `manual_review` | The workflow needs a human decision. An analyst reviews it in the Console. | No | Tell the user the review can take a few hours. |
| `approved` | The applicant passed verification. | **Yes** | Unlock the account or feature. |
| `rejected` | The applicant failed verification. | **Yes** | Block the feature. Follow your policy for what to tell the user. |
| `expired` | The applicant didn't finish within 7 days. | **Yes** | Create a new verification if the user returns. |
| `cancelled` | You cancelled the verification through the API. | **Yes** | No action needed. |
> [!IMPORTANT]
> Base access decisions only on the final statuses `approved` and `rejected`. Never unlock a feature while a verification is in `processing` or `manual_review`.
### Resubmission
During manual review, an analyst can ask the applicant for new documents (for example, when a photo is blurry). The verification goes back to `pending_input`, a new `hosted_url` is generated, and TrustLayer sends the `verification.resubmission_requested` event. Send the new link to the applicant by email or in-app message.
## How the automatic decision works
When the checks finish, the workflow decides the outcome in this order:
1. **Hard failures.** If a critical check fails (for example, `document_authenticity` returns `failed` or a sanctions list match is confirmed), the verification is `rejected`.
2. **Risk rules.** The account's [risk rules](../admin/02-configuring-risk-rules.md) run in priority order. The first rule with a final action (`approve`, `review` or `reject`) wins.
3. **Score thresholds.** If no rule matched, the risk score decides, using the thresholds set in the workflow's decision step:
| Risk score | Default outcome |
|---|---|
| 0–29 | `approved` |
| 30–84 | `manual_review` |
| 85–100 | `rejected` |
## Decision reason codes
When a verification is rejected, `decision.reason_code` tells you why. Use it to decide what to show the user and whether they can try again.
| Reason code | Description | Can the applicant retry? |
|---|---|---|
| `document_expired` | The document is past its expiry date. | Yes, with a valid document. |
| `document_unreadable` | Image quality was too low after the maximum number of attempts. | Yes. |
| `document_unsupported` | The document type or country isn't accepted by the workflow. | Yes, with another document. |
| `document_tampered` | The document shows signs of editing or forgery. | No. |
| `face_mismatch` | The selfie doesn't match the document photo. | No. |
| `liveness_failed` | The selfie appears to be a photo, screen, mask or deepfake. | No. |
| `watchlist_match` | The applicant matches a sanctions or restricted list. | No. |
| `high_risk` | The risk score or a risk rule required rejection. | No. |
| `duplicate_identity` | The same document is already linked to another applicant in your account. | No. |
| `analyst_decision` | An analyst rejected the verification for another reason. | Depends on your policy. |
> [!WARNING]
> Don't tell applicants the exact reason for a fraud-related rejection (for example, `liveness_failed` or `high_risk`). Detailed messages help fraudsters adjust their attacks. Use a neutral message such as "We couldn't verify your identity."
## List available workflows
You can retrieve the published workflows in your account to validate a `workflow_id` or to show workflow names in your internal tools.
```http
GET /v1/workflows
```
```bash
curl https://api.trustlayer.io/v1/workflows \
  -H "Authorization: Bearer $TOKEN"
```
```json
{
  "object": "list",
  "data": [
    {
      "id": "wf_standard_kyc_br",
      "object": "workflow",
      "name": "Standard KYC — Brazil",
      "published_version": 4,
      "steps": ["document_check", "selfie_liveness", "watchlist_screening", "risk_assessment"],
      "updated_at": "2026-10-01T13:20:00Z"
    },
    {
      "id": "wf_enhanced_dd",
      "object": "workflow",
      "name": "Enhanced due diligence",
      "published_version": 2,
      "steps": ["document_check", "selfie_liveness", "database_lookup", "address_proof", "watchlist_screening", "risk_assessment"],
      "updated_at": "2026-09-12T09:02:00Z"
    }
  ],
  "has_more": false
}
```
## Workflow versioning
- Every time an admin publishes a workflow, TrustLayer creates a new, immutable version.
- A verification always runs on the version that was published **when the verification was created** (`workflow_version`). Publishing a new version doesn't affect verifications already in progress.
- The `workflow_id` never changes, so you don't need to update your code when the workflow changes.
> [!TIP]
> Ask your compliance team to notify developers when they add a step that changes the user experience (for example, adding `address_proof`). The `workflow_id` stays the same, but the applicant sees more screens.
## Related
- [Identity Verification API](03-identity-verification-api.md)
- [Webhooks](06-webhooks.md): event list and payloads
- Admin Guide: [Creating a verification workflow](../admin/01-creating-a-verification-workflow.md)
