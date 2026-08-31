---
name: Collect qualified signatures from several people on one document
description: >-
  Use the Tvarka Sign API to raise a qualified signing ceremony for one or more signers, track it to
  a terminal state, nudge a slow signer, and retract it if plans change. This is the remote-method
  path (Smart-ID, Mobile-ID, LT ATK over NFC in the Tvarka Sign app) - the card-level ATK API does
  not do it.
api: openapi/tvarka-sign-api-openapi.yml
generated: '2026-08-31'
method: generated
source: https://sign-api.tvarka.pro/docs
operations:
  - createSigning
  - getSigning
  - listSignings
  - addSigner
  - remindSigner
  - cancelSigning
  - downloadSignedDocument
  - eraseSigning
  - simulateSigning
---

# Collect qualified signatures from several people on one document

**Nothing you call here produces a signature.** A qualified signature is a human act performed by
the signer with their own eID. Every operation below is about getting a document in front of the
right person and finding out what they did.

## Before you start

- Base URL `https://sign-api.tvarka.pro`, every path under `/v1/`.
- Auth is a single bearer key: `Authorization: Bearer <key>`. `tsk_test_` is the sandbox,
  `tsk_live_` is production. Request one at `info@tvarka.pro`; a sandbox tenant costs nothing.
- Build against the sandbox first. `simulateSigning`
  (`POST /v1/signings/{signingId}/simulate`) drives a signing to `completed`, `declined` or
  `expired` on demand so you can exercise every terminal path without waiting for a human. A live
  key gets `403` there.

## 1. Raise the ceremony — `createSigning`

`POST /v1/signings` with the document and the signers. Signers are **parallel by default**: whoever
opens their link first signs first, and each subsequent signer signs the output the previous one
produced. Set `signingOrder` to release them one at a time.

Worth setting on the way in, because none of it can be added later without another call:

- `externalId` — your own correlation key. It comes back on the signing and on every webhook, and
  it is the only thing that lets you tell a retry from a duplicate (see *Retries* below).
- `webhookUrl` — see step 3.
- `expiresInDays` is the hard deadline; `softDeadlineInDays` is the wanted-by date that triggers one
  automatic reminder round.

`402` means the workspace funding precondition is not met. The problem body may carry `recoveryUrl`
— that is a **browser page for a human operator**, not an endpoint to call.

## 2. Follow it — `getSigning` / `listSignings`

`GET /v1/signings/{signingId}` for one, `GET /v1/signings` for the tenant's list. Listing is cursor
paginated: pass the previous page's `nextCursor` as `startingAfter` and stop when `hasMore` is
false.

## 3. Prefer the webhook, but never depend on it

Deliveries carry `X-Tvarka-Signature: sha256=<HMAC-SHA256 of the raw body>` keyed with your webhook
secret — **verify the MAC before acting** — plus `X-Tvarka-Idempotency-Key` and `X-Tvarka-Event`.
Deduplicate by the idempotency key; delivery is at-least-once and retried with backoff.

The provider states delivery is a convenience and never the only way to learn an outcome: polling
always works, and a failed delivery never changes a signing's state. The payload carries no signer
identity data — no personal code, no certificate subject, no phone number.

Events: `signing.signer_signed`, `signing.signer_declined`, `signing.completed`,
`signing.declined`, `signing.cancelled`, `signing.expired`, `signing.failed`.

## 4. Nudge — `remindSigner`

`POST /v1/signings/{signingId}/signers/{signerId}/remind`. **One reminder per signer per hour** is
the published limit. Do not build a retry loop around it.

## 5. Collect — `downloadSignedDocument`

`GET /v1/signings/{signingId}/document` once the signing is `completed`. If you need the long-term
level, call `archiveSigning` first: PDF goes PAdES-B-T to B-LT and ASiC-E goes XAdES-T to XAdES-LT.
ADOC is refused by name, deliberately — ADOC-V1.0 specifies XAdES-T and its validators expect that
level. Calling archive twice is safe: already-archived output comes back with `upgraded: false`.

## Undoing things — know where reversal stops

- `cancelSigning` retracts every invitation that has **not** been used; the links die immediately
  and the identity data captured about those signers is erased. **Signatures already collected are
  untouched and the signed document stays downloadable.** That boundary is real: a qualified
  signature exists once it is made.
- `removeSigner` works only while that signer is still pending.
- `eraseSigning` cancels first, then purges. `eraseSignings` (`POST /v1/erasure`) does the same in
  bulk but only for **terminal** signings — in-flight ones are left alone.
- `deleteFile` drops stored bytes but does not unwind any signing that already referenced the token.

## Retries — the sharp edge

There is **no request idempotency header on this API**. A blind retry of `createSigning` raises a
second ceremony and bills a second time. Before retrying, list by your `externalId` and check
whether the first attempt landed.

## Errors

RFC 9457 `application/problem+json` throughout. Body validation **collects**: one
`validation-failed` response carries an `errors[]` array of JSON Pointer + detail, so fix all of
them in one pass. `409` is the one to expect most — it means the ceremony is not in a state that
permits the action (already terminal, signer already signed, already archived). `429` is declared on
every operation but carries no `Retry-After` and no `RateLimit-*` headers, so back off on your own
schedule.

## If you are an agent

There is a hosted MCP server at `https://sign-api.tvarka.pro/mcp` (registry `pro.tvarka/sign`) with
six tools: `list_documents`, `request_signatures`, `get_signing`, `list_signings`, `remind_signer`,
`cancel_signing`. It addresses documents already in the workspace vault, so no document bytes travel
through your context. `request_signatures` is billed and legally significant — put it behind human
approval. Everything else on this page (batches, roster edits, archive, download) is REST-only; see
`mcp/tvarka-sign-api-tool-crosswalk.yml`.
