---
name: Sign a document with a qualified electronic signature
description: >-
  Run the ATK signing ceremony - create a sign request for PAdES, ASiC-E, ADOC or raw CAdES, drive
  the card, then read both the status and the validation axis before downloading the container.
api: openapi/tvarka-atk-api-openapi-original.json
generated: '2026-08-31'
method: generated
source: https://atk.tvarka.pro/docs/
operations:
  - createSignRequest
  - submitSignCertificate
  - completeSign
  - getSignRequest
  - downloadSignedDocument
  - cancelSignRequest
---

# Sign a document with a qualified electronic signature

Same ceremony shape as authentication, with a document and a second result axis.

## 1. Create the sign request (backend, Basic)

`createSignRequest` — `POST /sign/requests`. The body is a `oneOf` over four shapes:

| `format` | What you get | `method` |
|---|---|---|
| `pades` | Signed PDF | `physical`, `nfc` |
| `asice` | XAdES-T ASiC-E container | `physical`, `nfc` |
| `adoc` | XAdES-T ADOC-V1.0 container | `physical`, `nfc` |
| `raw` | Detached CAdES from a bare hash, no container | `physical`, `nfc` |

Container formats require `document` (base64) **or** `documentRef`. A `documentRef` is
`{url, sha256, sizeBytes, authorization?}` and the host must be allow-listed — otherwise **403
`document_ref_host_not_allowed`**, or **502 `document_ref_fetch_failed`** if the fetch fails.
`raw` takes `hash` + `hashAlgOid` instead.

A 202 returns `requestId`, `verificationCode`, `status`, `expiresAt`, plus `clientToken` and a
`pairing` hint for card methods.

Documents over the tenant limit (15 MiB by default) return **413 `document_too_large`**. A format
your provider is not enabled for returns **403 `format_not_allowed`**.

## 2. Certificate, then signature (device, clientToken)

`submitSignCertificate` — `POST /sign/{requestId}/certificate`, body `{ "certificate": "<base64
DER>" }`. Returns `dtbs`, `dtbsHashAlgOid`, `operationToken`, `cert` and a `signing` block. Sign
`dtbs` verbatim.

`completeSign` — `POST /sign/{requestId}/complete`, body `signature`, `signatureAlgorithm`,
`operationToken` (+ optional `signatureEncoding`). Returns `status`, `method`, `validation`, `sig`,
`sigAlgOid`, `cert` and a one-off `downloadToken`.

If the container hash does not match what was signed you get **422 `document_hash_mismatch`** or
`document_size_mismatch`; a bad signature is **422 `signature_invalid`**.

## 3. Read BOTH axes

This is the mistake to avoid. A sign request has two independent axes:

- `status` — did the ceremony finish? (`done` on success.)
- `validation.status` — is the resulting signature qualified and valid? (`pending`, `passed`,
  `failed`, `notApplicable`, with a `level` such as `QUALIFIED_VALID`, `VALID_NOT_QUALIFIED`,
  `INDETERMINATE`.)

**Validation failure never appears on `status`.** A `status: "done"` signature can carry
`validation.status: "failed"`. Gate any legal or business decision on the validation axis, not on
`status` alone.

`getSignRequest` — `GET /sign/{requestId}` — returns both, and is also where you watch
`validation.status` move off `pending`.

## 4. Download the container

`downloadSignedDocument` — `GET /sign/{requestId}/document`, authenticated with Basic **or** the
one-off `downloadToken` as a query parameter. Before the artifact is assembled you get **404
`document_not_ready`**; if assembly failed, **500 `assembly_failed`**.

`cancelSignRequest` — `POST /sign/{requestId}/cancel` — for non-terminal requests only.

## Rules

- A completed signature is a **qualified electronic signature with legal effect under eIDAS**. It is
  not a reversible API call. Require explicit human intent for `createSignRequest` and
  `completeSign`; never retry a ceremony automatically.
- Billing point: the signed artifact is assembled and the signer signature is verified. Failed,
  cancelled and expired ceremonies are not charged.
- Retention defaults to 30 days. Download and store the container yourself; use
  `deleteSignRequest` to purge earlier.
