---
name: Authenticate a person with a Lithuanian eID card
description: >-
  Run the ATK authentication ceremony end to end - create an audience-bound request from your
  backend, drive the card on the holder's device, and verify the returned identity yourself.
api: openapi/tvarka-atk-api-openapi-original.json
generated: '2026-08-31'
method: generated
source: https://atk.tvarka.pro/docs/quickstart/
operations:
  - createAuthRequest
  - submitAuthCertificate
  - completeAuth
  - getAuthRequest
  - cancelAuthRequest
  - getJwks
---

# Authenticate a person with a Lithuanian eID card

Base URL `https://atk.tvarka.pro/v1`. Two credentials, two places:

- **Backend only** — HTTP Basic `keyId:keySecret`, also gated by your server IP allow-list. Never
  send this to a browser, a phone or an agent-facing client.
- **Device** — the `clientToken` returned at creation. It is a short-lived ES256 JWT scoped to one
  request and cannot create requests or read tenant data.

## 1. Create the request (backend, Basic)

`createAuthRequest` — `POST /auth/requests`.

Body: `audience` (required — the exact origin approved for your provider), `method` (required —
`physical` or `nfc`; contract 1.4.0 removed `smart_id` and `mobile_id`, and remote-method ceremonies
now live in the Tvarka Sign API), optionally `message`, `externalId`, `webhookUrl`.

A 202 returns `requestId`, `verificationCode`, `status`, `expiresAt`, plus `clientToken`, `nonce`,
`challengeProfile` for `physical` and a `pairing` hint for `nfc`.

If `audience` does not match an approved origin you get **403 `origin_not_allowed`**. A malformed
`webhookUrl` is rejected at creation with **400 `invalid_webhook_url`**.

Pass only `requestId` and `clientToken` to the device. Show `verificationCode` to the person.

## 2. Submit the card certificate (device, clientToken)

`submitAuthCertificate` — `POST /auth/{requestId}/certificate`, body `{ "certificate": "<base64
DER>" }`.

Returns `dtbs` (the exact data to be signed), `dtbsHashAlgOid` (SHA-256 `2.16.840.1.101.3.4.2.1` or
SHA-384 `...2.2`), `operationToken`, `challenge` and the parsed `cert`. Sign `dtbs` exactly as
returned — do not recompute it.

Certificate rejections come back as **422**: `cert_invalid`, `untrusted_chain`, `cert_revoked`,
`cert_expired`, `not_qualified`, `cert_purpose_mismatch`.

## 3. Complete (device, clientToken)

`completeAuth` — `POST /auth/{requestId}/complete`, body `signature`, `signatureAlgorithm` (`ES256`
or `ES384`), `operationToken`, optionally `signatureEncoding` (`P1363` or `DER` — the server
auto-detects either).

Returns `status`, `method`, `sig`, `cert` and an optional `assertion` JWT.

`operationToken` is one-shot: replaying it returns **409 `operation_token_spent`**. Presenting a
different card mid-ceremony returns **409 `identity_swap`**.

## 4. Verify the result yourself

Do not take `status: "done"` as proof.

1. Re-derive the `atk-auth-v1` challenge from **your own** `audience` and the returned `nonce`.
2. Verify `sig` against `cert` for that challenge.
3. If you use the `assertion` JWT, verify it against the JWKS from `getJwks`
   (`GET /.well-known/atk-jwks.json`, at the host root, outside `/v1`) — key `atk-1`, `ES256`.
4. Check the certificate subject: `serialNumber` looks like `PNOLT-<personal code>`, `c` is `LT`.

## 5. Or wait for the result server-side

`getAuthRequest` — `GET /auth/{requestId}` (Basic or clientToken). Non-terminal statuses are
`pending`, `awaitingCard`, `awaitingCredentials`, `finalizing`. Terminal success is `done`; terminal
failure is `cancelled`, `timeout`, `deviceError`, `pinBlocked`, `cardRemoved` or `certInvalid`.

`cancelAuthRequest` — `POST /auth/{requestId}/cancel` — abandons a non-terminal request. Acting on a
terminal request returns **409 `request_terminal`**.

## Rules

- CAN and PIN stay on the holder's device; `awaitingCredentials` is SDK-reported and the API never
  sees either value.
- Errors are `{code, message, requestId, retryable}` — **not** RFC 9457. Retry only 429 (honour
  `Retry-After`, in seconds), 500, 502 and 503. Never retry a 4xx by re-running the ceremony.
- A completed authentication is a verified national identity. Do not run it speculatively or in a
  retry loop on behalf of a user who is not present at the card.
- Sandbox and production share this host. Only the credential differs, so double-check which
  `keyId` you are holding before calling anything billable.
