# Getting Started
>
> **Audience:** Developers integrating TrustLayer for the first time.
>
> **Prerequisites:** A terminal with `curl`, and basic knowledge of REST APIs and JSON.
>
> **Time to complete:** about 15 minutes.
TrustLayer helps you verify who your customers are (KYC) and stop fraud before it reaches your platform. This guide takes you from zero to your first completed identity verification in the sandbox.

**[Five steps to your first verification]**


(**INSERIR AQUI IMAGEM**../../images/illustrations/getting-started-steps.svg)

## What you can build with TrustLayer
| Product | What it does | API reference |
|---|---|---|
| **Identity Verification** | Checks an identity document and matches it to a live selfie. | [Identity Verification API](03-identity-verification-api.md) |
| **KYC workflows** | Chains checks (document, selfie, watchlists, risk) into one configurable flow. | [KYC Workflow](04-kyc-workflow.md) |
| **Risk Score** | Scores signups, logins and transactions from 0 to 100. | [Risk Score API](05-risk-score-api.md) |
| **Fraud Signals** | Explains *why* a score is high, with machine-readable signal codes. | [Fraud Signals](07-fraud-signals.md) |
## Key concepts
Learn these five terms before you start. The rest of the documentation uses them consistently.
| Term | Definition |
|---|---|
| **Applicant** | The person you want to verify. One applicant can have many verifications over time. |
| **Verification** | A single attempt to verify an applicant, run against one workflow. |
| **Workflow** | A sequence of checks configured by your compliance team in the Admin Console. Identified by a `workflow_id`. |
| **Risk assessment** | A score from 0 (lowest risk) to 100 (highest risk), with the fraud signals that produced it. |
| **Event** | A notification that TrustLayer sends to your webhook endpoint when something changes. |
## Environments
TrustLayer has two fully isolated environments. Credentials from one environment do not work in the other.
| Environment | Base URL | Credential prefix | Real checks? |
|---|---|---|---|
| Sandbox | `https://sandbox.api.trustlayer.io/v1` | `tl_test_` | No. Results are simulated. |
| Production | `https://api.trustlayer.io/v1` | `tl_live_` | Yes. Each check is billed. |
> [!TIP]
> Build and test your entire integration in the sandbox. It's free, and it lets you simulate any result, including fraud scenarios. See [Sandbox Environment](09-sandbox-environment.md).
## Quickstart
### Step 1. Create a sandbox account
1. Go to `https://app.trustlayer.io/signup` and create an account.
2. Confirm your email address.
3. In the Console, check that the environment switch in the lower-left corner shows **Sandbox**.
### Step 2. Generate API credentials
1. In the Console, go to **Developers › API keys**.
2. Click **Create credentials**.
3. Name the credentials (for example, `Local development`) and select the scopes `verifications:write` and `risk:write`.
4. Copy the **client ID** and **client secret**.
> [!WARNING]
> The Console shows the client secret only once. Store it in a secrets manager or in an environment variable. Never commit it to Git or ship it in a mobile or frontend app.
Export the credentials in your terminal:
```bash
export TRUSTLAYER_CLIENT_ID="tl_test_cid_4f9a2c71"
export TRUSTLAYER_CLIENT_SECRET="tl_test_sk_your_secret_here"
```
### Step 3. Request an access token
Exchange your credentials for a short-lived access token:
```bash
curl -X POST https://sandbox.api.trustlayer.io/v1/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$TRUSTLAYER_CLIENT_ID" \
  -d "client_secret=$TRUSTLAYER_CLIENT_SECRET" \
  -d "scope=verifications:write risk:write"
```
Response:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "verifications:write risk:write"
}
```
Save the token for the next steps:
```bash
export TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
```
To learn how tokens work and how to refresh them, see [Authentication](02-authentication.md).
### Step 4. Create a verification
Create an applicant and start a verification in a single request. The sandbox includes a ready-to-use workflow called `wf_sandbox_default`.
```bash
curl -X POST https://sandbox.api.trustlayer.io/v1/verifications \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 7f1c9a52-quickstart-001" \
  -d '{
    "workflow_id": "wf_sandbox_default",
    "applicant": {
      "first_name": "Mariana",
      "last_name": "Costa",
      "email": "mariana.costa@example.com",
      "external_id": "user_1029"
    },
    "redirect_url": "https://example.com/onboarding/done"
  }'
```
Response (`201 Created`):
```json
{
  "id": "ver_8kQ2mX4pL9",
  "object": "verification",
  "status": "pending_input",
  "workflow_id": "wf_sandbox_default",
  "applicant_id": "app_5Rt8wQ2nB7",
  "hosted_url": "https://verify.sandbox.trustlayer.io/s/ses_Zx81Kq",
  "hosted_url_expires_at": "2026-10-09T18:00:00Z",
  "created_at": "2026-10-08T18:00:00Z"
}
```
### Step 5. Complete the flow and get the result
1. Open the `hosted_url` in your browser.
2. Follow the screens. In the sandbox, you can upload any image; the result is simulated.
   ![End-user verification journey](../../images/illustrations/verification-journey.svg)
3. Retrieve the result:
   ```bash
   curl https://sandbox.api.trustlayer.io/v1/verifications/ver_8kQ2mX4pL9 \
     -H "Authorization: Bearer $TOKEN"
   ```
   ```json
   {
     "id": "ver_8kQ2mX4pL9",
     "object": "verification",
     "status": "approved",
     "decision": {
       "outcome": "approved",
       "decided_by": "automatic",
       "decided_at": "2026-10-08T18:03:12Z"
     },
     "risk": { "score": 12, "level": "low" }
   }
   ```
🎉 You've completed your first verification.
> [!NOTE]
> In production, don't poll for results. Subscribe to the `verification.completed` webhook event instead. See [Webhooks](06-webhooks.md).
## Next steps
- Secure your integration: [Authentication](02-authentication.md)
- Understand every status a verification can have: [KYC Workflow](04-kyc-workflow.md)
- Plan your production integration end to end: [Integration Guide](10-integration-guide.md)
