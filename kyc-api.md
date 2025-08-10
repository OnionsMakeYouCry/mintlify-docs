# KYC API v1

Base host: `https://api.verisecid.com`

- All endpoints are under `/api/v1/kyc/...`.
- Only requests targeting allowed API hosts are served (default allowlist includes `api.verisecid.com`, `localhost`, `127.0.0.1`). You can override via `API_ALLOWED_HOSTS` (comma-separated).
- CORS is enabled for these endpoints (methods vary per handler; see below).

## Authentication

- Public (workspace) endpoints: `Authorization: Bearer <public_api_key>`
  - The public key corresponds to a user record in Firestore `users.apiPublicKey` (e.g., `pb-xxxxx`).
  - If the key matches a user, the request is authenticated as that user.
- Direct verification endpoint: `Authorization: Bearer <secret_key>`
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

On session creation, if the user has no workspace, a default workspace is created and the user’s API keys are migrated onto it. The user’s `currentWorkspaceId` is set/updated.

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
  "metadata": { "customerId": "12345" }
}
```

Successful response (201):

```json
{
  "sessionId": "<session-id>",
  "verificationUrl": "https://<host>/verify/<session-id>",
  "workspaceId": "<workspace-id>",
  "userId": "<uid>",
  "status": "not_started",
  "createdAt": "2025-08-09T10:00:00.000Z"
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

- Auth required: `Authorization: Bearer <public_api_key>`
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
  "verificationUrl": "https://<host>/verify/<session-id>"
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
  -H "Authorization: Bearer pb-YourPublicKey"
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
  "verificationUrl": "https://<host>/verify/<sessionId>"
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
