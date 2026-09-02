---
name: vayana-network-verify-counterparty
description: Verify an Indian business counterparty before trading with it using Vayana Atlas — PAN to GSTIN, GSTIN status, MCA company record, Udyam registration, bank account and a consolidated business profile report.
api: vayana-network-atlas
generated: '2026-09-02'
method: generated
source: openapi/vayana-network-atlas-openapi.json
operations:
- login
- rubix_post_kyc_v2_pan
- rubix_post_kyc_pan_detailed
- rubix_post_kyc_v1_panGstLink
- rubix_get_kyc_v1_panGstLink
- eas_basic_v1_gstn_search_gstin
- rubix_post_kyc_v2_mca
- rubix_get_kyc_v1_mcaInstant
- rubix_post_kyc_v1_udyam
- rubix_get_kyc_v1_udyam
- rubix_post_kyc_bankAccount
- rubix_post_kyc_v1_indian_search
- rubix_post_kyc_v1_indian_instantReport
- rubix_post_kyc_v1_indian_report_pdf
---

# Verify a business counterparty

Every operation here is **read-only**. Nothing in this skill can be got wrong in a way that costs
money — which is exactly why it is the right place to start with this API.

## The one thing to understand first: sync vs async

The Verification Suite has two shapes, and confusing them is the main integration mistake:

- **Instant** — one `GET`, answer in the response. PAN, GSTIN, MCA, bank account, vehicle, TAN,
  driving licence, voter ID, FSSAI, shop & establishment.
- **Two-step async** — a `-async` path that returns a `referenceId`, then a second call to
  `.../result/{referenceId}` to collect it. Aadhaar consent, EPF, passport, Udyam, Udyog and
  PAN-GST link work this way, because a human or an upstream registry has to respond.

Async operations are named `1. Place ... request` and `2. Get ... data` in the contract. Do not
poll the placement endpoint; call the result endpoint.

## A normal counterparty check

1. **Authenticate** with `login`. Reuse one token across the whole batch.
2. **Start from PAN.** `GET /atlas/v1/pan/{pan}` (`rubix_post_kyc_v2_pan`) confirms the tax identity
   exists; `GET /atlas/v1/pan/{pan}/details` (`rubix_post_kyc_pan_detailed`) returns the fuller record.
3. **Fan out to GSTINs.** `GET /atlas/v1/panGstLink-async/{pan}` (`rubix_post_kyc_v1_panGstLink`)
   returns a `referenceId`; collect with
   `GET /atlas/v1/panGstLink-async/result/{referenceId}` (`rubix_get_kyc_v1_panGstLink`).
   One PAN can hold many GSTINs — one per state — and this is the step that turns a single identity
   into the full registration footprint.
4. **Check each registration is live.** `GET /atlas/v1/gstin/{gstin}`
   (`eas_basic_v1_gstn_search_gstin`) per GSTIN. A cancelled registration is the finding that matters.
5. **Confirm the corporate entity.** `GET /atlas/v1/mca/{cin}` (`rubix_post_kyc_v2_mca`) for the
   filed company record, or `GET /atlas/v1/mcaInstant` (`rubix_get_kyc_v1_mcaInstant`) for the fast path.
6. **Check MSME status** if it affects payment terms: `GET /atlas/v1/udyam-async/{udyamNumber}`
   then `.../result/{referenceId}`.
7. **Validate settlement details before paying anyone.**
   `GET /atlas/v1/bankAccount/{ifsc}/{accountNumber}` (`rubix_post_kyc_bankAccount`).
8. **Or take the consolidated view.** `GET /atlas/v1/counterparty/search/{pageNo}/{pageSize}`
   (`rubix_post_kyc_v1_indian_search`) finds the counterparty,
   `GET /atlas/v1/bpr/report/{counterpartyId}` (`rubix_post_kyc_v1_indian_instantReport`) returns an
   instant business profile report, and `GET /atlas/v1/report/pdf/{reportId}`
   (`rubix_post_kyc_v1_indian_report_pdf`) renders it.

## Handle with care

These operations return personal data — Aadhaar, passport, driving licence, voter ID, EPF and bank
account details. Aadhaar has a **separate consent-based flow**
(`/atlas/v1/aadhaarConsent-async/...`, which requires an OTP at the result step) precisely because
consent is required. Use the consent flow for Aadhaar; do not log or persist any of these responses
beyond what your own retention policy allows.

## Failure handling

- Read `status` in the body, not just the HTTP code.
- Async results are retained for **7 days** from the reference token; collect them within that.
- `403 err-invalid-org` means `X-FLYNN-N-ORG-ID` is wrong — take it from
  `data.associatedOrgs[].organisation.id` in the login response.
