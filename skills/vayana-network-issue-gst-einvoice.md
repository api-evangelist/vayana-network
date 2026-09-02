---
name: vayana-network-issue-gst-einvoice
description: Register a GST e-invoice with an Indian Invoice Registration Portal through Vayana Atlas, retrieve the signed invoice and QR code, and cancel it if the document was wrong.
api: vayana-network-atlas
generated: '2026-09-02'
method: generated
source: openapi/vayana-network-atlas-openapi.json
operations:
- login
- eas_basic_v1_gstn_search_gstin
- eas_basic_v1_generate_invoice
- eas_basic_v1_fetch_invoice_get
- eas_enriched_v1_download_invoice_pdf
- eas_basic_v1_cancel_invoice
- logout
---

# Issue a GST e-invoice

Registers an invoice with an IRP and gets back an IRN, an acknowledgement number and a
government-signed QR code. This is a **write against a government system**. Read the
reversibility note at the bottom before the first call.

## Before you start

- Base URL: `https://s.api.one.vayana.com` (sandbox). Vayana publishes **no production host** for
  Atlas in the contract — confirm yours with Vayana before going live. Do not guess it.
- You need: Vayana SSO credentials, your `organisation-id`, and the taxpayer's own IRP portal
  credentials (GSTIN, username, password). Vayana passes those through; it does not own them.
- `{irp}` must be `ni1` or `ni2`. The value `nic` is deprecated.

## Steps

1. **Authenticate.** `POST /theodore/apis/v1/authtokens` (`login`) with
   `{"handle": <email>, "password": <password>, "handleType": "email", "tokenDurationInMins": 360}`.
   Keep `data.token` and `data.expiry`. Send the token as `Authorization: Bearer <token>`.
   Mint **one** token and reuse it: a user may hold only 10 active sessions, and exceeding that
   returns `err-active-tokens-limit-exceeded` (401).

2. **Check the buyer is real and active.** `GET /atlas/v1/gstin/{gstin}`
   (`eas_basic_v1_gstn_search_gstin`). A cancelled or non-existent GSTIN is the most common cause of
   a rejected invoice, and it is far cheaper to catch here than after registration.

3. **Build the payload.** The request body is the government INV-01 document, Version `1.1` — not a
   Vayana abstraction. Populate `TranDtls`, `DocDtls`, `SellerDtls`, `BuyerDtls`, `ItemList` and
   `ValDtls`; add `DispDtls`, `ShipDtls`, `PayDtls`, `RefDtls`, `AddlDocDtls`, `ExpDtls` and
   `EwbDtls` where they apply. If you already generate INV-01 for the government portal, send it
   unchanged.

4. **Register it.** `POST /atlas/v1/irp/{irp}/einvoice` (`eas_basic_v1_generate_invoice`).
   Keep `Irn`, `AckNo`, `AckDt`, `SignedInvoice` and `SignedQRCode` — the signed values are the
   legal artifacts and cannot be regenerated later.

5. **Verify.** Read the body, not just the status line. On v3 payloads a `200` can still carry
   `status: "0"` with a populated `error` object. Treat `status == "1"` as the success condition.

6. **Fetch or render when needed.** `GET /atlas/v1/irp/{irp}/einvoice/{irn}`
   (`eas_basic_v1_fetch_invoice_get`) returns the registered document;
   `GET /atlas/v1/irp/{irp}/einvoice/{irn}/pdf` (`eas_enriched_v1_download_invoice_pdf`) returns a PDF.

7. **Release the session** when the batch is done: `POST /theodore/apis/v1/logout` (`logout`).

## Reversing it

`PATCH /atlas/v1/irp/{irp}/einvoice/cancel` (`eas_basic_v1_cancel_invoice`) voids the IRN.
Cancel reason is an enumerated code: `1` Duplicate, `2` Data entry mistake, `3` Order Cancelled,
`4` Others.

**Vayana does not publish a cancellation deadline.** The window is set by the IRP, not by Vayana,
and Vayana's documentation does not restate it. Do not assume one. Confirm the current limit with
the IRP before relying on being able to undo a registration.

## Failure handling

- **There is no idempotency key.** If a `POST` times out you cannot safely retry it: a second call
  may register a second invoice. Fetch by document details first to find out whether the original
  landed, then decide.
- `403` with `err-expired-token` — refresh via `PUT /theodore/apis/v1/authtokens` (`refresh_token`).
  Refresh only works until the hard session expiry (6x the token duration); after that, log in again.
- `500` with `err-unable-to-decrypt-message-with-key` or `err-aes-decryption-failed` — an encryption
  problem, not a data one. Check you are using the public key for the right environment and service.
- Full envelope and code list: `errors/vayana-network-problem-types.yml`.
