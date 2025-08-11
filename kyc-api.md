# KYC API v1

Base host: `https://api.verisecid.com`

- All endpoints are under `/api/v1/kyc/...`.
- Version-first URLs are also supported on API hosts: `/v1/...` will 308-redirect to `/api/v1/...`.
- Only requests targeting allowed API hosts are served (default allowlist includes `api.verisecid.com`, `localhost`, `127.0.0.1`). You can override via `API_ALLOWED_HOSTS` (comma-separated).
- CORS is enabled for these endpoints (methods vary per handler; see below).

## Authentication

- Public (workspace) endpoints: `Authorization: Bearer <public_api_key>`
  - The public key corresponds to a workspace record in Firestore `workspaces.apiPublicKey` (e.g., `pb-xxxxx`).
  - If the key matches a workspace, the request is authenticated as that workspace (and its `ownerUserId`).
- Direct verification endpoint and protected GET: `Authorization: Bearer <secret_key>`
  - The secret key can match either `users.apiSecretKey` or `workspaces.apiSecretKey`.

## Data model overview

Firestore collections used by the API:

- `users/{uid}`
  - `uid: string`
  - `email: string`
  - `displayName: string`
  - `apiPublicKey?: string` (e.g., `pb-...`)
  - `apiSecretKey?: string` (e.g., `sc-...`)
  - `currentWorkspaceId?: string`
- `workspaces/{workspaceId}`
  - `name: string`
  - `ownerUserId: string` (user.uid)
  - `apiPublicKey?: string`
  - `apiSecretKey?: string`
  - `createdAt: Timestamp`
  - `updatedAt?: Timestamp`
- `kyc_hosted_sessions/{sessionId}`
  - `workspaceId: string`
  - `userId: string` (owner user.uid)
  - `status: "not_started" | "processing" | "completed" | "failed"`
  - `createdAt: Timestamp`
  - `updatedAt?: Timestamp`
  - `redirectUrl?: string | null`
  - `metadata?: Record<string, unknown> | null`

Prerequisite: a workspace must already exist and contain API keys (`apiPublicKey`, `apiSecretKey`). No default workspace is created by the API.

---

## Create KYC hosted session

POST `/api/v1/kyc/sessions`

- Auth required: `Authorization: Bearer <public_api_key>`
- CORS: `POST, OPTIONS`
- Content-Type: `application/json`

Request body (optional):

```json
{
  "redirectUrl": "https://yourapp.com/kyc/return",
  "metadata": { "customerId": "12345" },
  "firstName": "Jane",
  "lastName": "Doe"
}
```

Successful response (201):

```json
{
  "sessionId": "<session-id>",
  "verificationUrl": "https://verisecid.com/verify/<session-id>",
  "workspaceId": "<workspace-id>",
  "userId": "<uid>",
  "status": "not_started",
  "createdAt": "2025-08-09T10:00:00.000Z",
  "applicantFirstName": "Jane",
  "applicantLastName": "Doe"
}
```

Error responses:

- 401 {"error":"missing_authorization"} when header is absent
- 401 {"error":"invalid_credentials"} when public key is invalid
- 500 {"error":"internal_error","message":"..."} on server error

Example:

```bash
curl -X POST "https://api.verisecid.com/api/v1/kyc/sessions" \
  -H "Authorization: Bearer pb-YourPublicKey" \
  -H "Content-Type: application/json" \
  -d '{
    "redirectUrl": "https://yourapp.com/kyc/return",
    "metadata": { "customerId": "12345" }
  }'
```

---

## Get a single session

GET `/api/v1/kyc/sessions/{sessionId}`

 - Auth required: `Authorization: Bearer <secret_key>`
- CORS: `GET, OPTIONS`

Authorization rules:

- The requesting user must be the session owner (`userId`) OR the owner of the associated `workspaceId`.

Successful response (200):

```json
{
  "sessionId": "<session-id>",
  "workspaceId": "<workspace-id>",
  "userId": "<uid>",
  "status": "not_started",
  "redirectUrl": null,
  "metadata": { "customerId": "12345" },
  "createdAt": "2025-08-09T10:00:00.000Z",
  "updatedAt": "2025-08-09T10:05:00.000Z",
  "verificationUrl": "https://verisecid.com/verify/<session-id>"
}
```

Error responses:

- 401 {"error":"missing_authorization"} when header is absent
- 401 {"error":"invalid_credentials"} when public key is invalid
- 403 {"error":"forbidden"} when the user does not own the session/workspace
- 404 {"error":"not_found"} when session does not exist
- 500 {"error":"internal_error","message":"..."} on server error

Example:

```bash
curl -X GET "https://api.verisecid.com/api/v1/kyc/sessions/SESSION_ID" \
  -H "Authorization: Bearer sc-YourSecretKey"
```

---

## Direct verification (non-session)

POST `/api/v1/kyc/verify`

- Auth required: `Authorization: Bearer <secret_key>` (user or workspace secret)
- CORS: `POST, OPTIONS`
- Content-Type: `application/json`

Request body:

```json
{
  "sessionId": "optional-custom-id",
  "expectedDocumentType": "passport", // or "id-card"
  "documentFrontBase64": "<base64 or null>",
  "documentBackBase64": "<base64 or null>",
  "selfieBase64s": ["<base64>", "<base64>", "<base64>"]
}
```

Successful response (200):

```json
{
  "sessionId": "generated-or-provided",
  "status": "completed" | "failed",
  "result": { /* KycVerificationResult */ },
  "verificationUrl": "https://verisecid.com/verify/<sessionId>"
}
```

Error responses:

- 401 {"error":"missing_authorization"} when header is absent
- 401 {"error":"invalid_credentials"} when secret key is invalid
- 415 {"error":"unsupported_media_type"} if not `application/json`
- 400 {"error":"invalid_request"} for malformed input
- 500 {"error":"internal_error","message":"..."} on server error

Example:

```bash
curl -X POST "https://api.verisecid.com/api/v1/kyc/verify" \
  -H "Authorization: Bearer sc-YourSecretKey" \
  -H "Content-Type: application/json" \
  -d '{
    "expectedDocumentType": "passport",
    "documentFrontBase64": "<base64>",
    "selfieBase64s": ["<base64>"]
  }'
```

---

## Host restrictions (Middleware)

API requests are served only when the Host header matches an allowlist. Default:

- `api.verisecid.com`, `localhost`, `127.0.0.1`

Override with environment variable:

```bash
API_ALLOWED_HOSTS=api.verisecid.com,localhost,127.0.0.1
```

When a host is not allowed, handlers return `404` with `{ "error": "invalid_host" }` to avoid leaking API surface on non-API domains.

---

## Environment variables

- `API_ALLOWED_HOSTS` (optional) — comma-separated hostnames allowed for `/api/*`.
- Firebase client (for app UI) requires standard `NEXT_PUBLIC_FIREBASE_*` variables.
- Server-side verification actions require AI provider keys (e.g., `OPENAI_API_KEY`), if used.

---

## Status lifecycle (sessions)

- Initial: `not_started` (set on creation)
- Typical progression: `not_started` → `processing` → `completed` (or `failed`)
- Session gating in the Verify UI prevents reuse when `status === "completed"`.

> Note: Session status transitions can be managed either via Firestore writes (Admin/client) or future API endpoints (e.g., `PATCH /api/v1/kyc/sessions/{id}`) depending on your integration needs.

---

## Verification UI

The `verificationUrl` points to the hosted flow at `/verify/{sessionId}`. The UI captures document and face images, then invokes a server action to produce a structured verdict. The API does not expose that verdict; your backend can store results to `kyc_hosted_sessions` or another collection as needed.

---

## Webhooks

VerisecID can notify your server about submission creation and final verification outcomes for hosted sessions.

### Configure webhooks

- In the Console → Workspace Settings → Notifications:
  - Enable webhooks
  - Set `webhookUrl` (HTTPS)
  - A signing `webhookSecret` is stored per workspace
  - Choose events to receive

Supported events:

- `submission.created` (emitted immediately after a submission is persisted)
- `verification.completed` (emitted when a session finishes with pass)
- `verification.failed` (emitted when a session finishes with fail)

KYC Status field (in addition to legacy status):

- Approved: AI pass and auto-approve enabled
- Submitted: AI pass and auto-approve disabled (awaiting manual decision)
- Declined: AI fail, or manual rejection

Delivery semantics:

- Events are filtered by your workspace `webhookEvents` allow-list
- If the selected final-status event is not allowed, `submission.created` is sent (when allowed)
- At-least-once delivery; receivers should de-duplicate using the `id` field
- No retry/backoff yet (coming soon). You can monitor deliveries in `kyc_webhook_deliveries` if you use Firestore directly

Security:

- Each request includes an HMAC signature header using your `webhookSecret`:

```
X-KYC-Signature: sha256=<hex-hmac-of-body>
```

Verify the signature by recomputing HMAC-SHA256 over the exact raw body JSON string and comparing constant-time to the header value.

### Event payload

POST requests send a JSON body with this structure:

```json
{
  "type": "submission.created | verification.completed | verification.failed",
  "id": "<eventType>:<submissionId>",
  "timestamp": "2025-08-10T21:50:43.235Z",
  "status": "Approved | Submitted | Declined",
  "data": {
    "session": {
      "id": "<sessionId>",
      "status": "Approved | Submitted | Declined",
      "kycStatus": "not_started | processing | completed | failed",
      "redirectUrl": "https://...",
      "metadata": { },
      "createdAt": { "_seconds": 0, "_nanoseconds": 0 },
      "updatedAt": { "_seconds": 0, "_nanoseconds": 0 },
      "images": {
        "documentFrontUrl": "https://...",
        "documentBackUrl": "https://...",
        "selfieUrls": ["https://...", "https://...", "https://..."]
      }
    },
    "submission": {
      "id": "<submissionId>",
      "sessionId": "<sessionId>",
      "workspaceId": "<workspaceId>",
      "pass": true,
      "expectedDocumentType": "passport | id-card",
      "detectedDocumentType": "passport | id-card | unknown",
      "document": { /* structured fields about the document */ },
      "face": { /* structured fields about the face match */ },
      "overall": { "pass": true, "reasons": [], "user_message": null },
      "kycStatus": "Approved | Submitted | Declined",
      "createdAt": { "_seconds": 0, "_nanoseconds": 0 }
    }
  }
}
```

Notes:

- The session object intentionally excludes `workspaceId` and `userId` for privacy. The `submission.workspaceId` is included as a non-sensitive identifier.
- Firestore timestamps are sent in `{ _seconds, _nanoseconds }` format. Convert to ISO as needed.

### Test your webhook

You can trigger a test event from the Console or via API:

POST `/api/v1/kyc/webhooks/test`

- Auth: none
- CORS: `POST, OPTIONS`
- Body:

```json
{ "workspaceId": "<workspaceId>" }
```

Response (200): `{ ok: true, status: 200 }` for successful delivery. Failures respond with `{ ok: false, ... }` but still HTTP 200.

### Programmatic notify (internal)

POST `/api/v1/kyc/submissions/notify`

- Auth: none (middleware-exempt for webhook fanout)
- CORS: `POST, OPTIONS`
- Body: `{ "sessionId": "<sessionId>" }`
- Behavior: loads session, latest submission, workspace webhook config, and attempts delivery. Returns 200 with `{ ok, delivered, status? }`.

