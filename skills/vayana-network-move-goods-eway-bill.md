---
name: vayana-network-move-goods-eway-bill
description: Generate an Indian E-Way Bill for a consignment through Vayana Atlas, update Part B when the vehicle changes, extend validity in transit, and cancel or reject when the movement does not happen.
api: vayana-network-atlas
generated: '2026-09-02'
method: generated
source: openapi/vayana-network-atlas-openapi.json
operations:
- login
- eas_basic_v1_generate_ewaybill
- eas_basic_v1_generate_ewaybill_by_invoice
- eas_basic_v1_fetch_ewaybill
- eas_basic_v1_update_ewaybill_partb
- eas_basic_v1_update_ewb_transporter
- eas_basic_v1_extend_ewaybill_validity
- eas_basic_v1_cancel_ewaybill
- eas_enriched_v1_get_ewaybill_pdf
- eas_basic_v1_fetch_ewaybill_error_list
---

# Move goods under an E-Way Bill

An E-Way Bill authorises a consignment to travel. It is time-boxed and vehicle-bound, so the
interesting work is the updates in transit, not the initial create.

## Two ways in

- **From an existing e-invoice** — `POST /atlas/v1/irp/{irp}/eway-bill/by-einvoice`
  (`eas_basic_v1_generate_ewaybill_by_invoice`). Prefer this when an IRN already exists: the
  consignment details come from the registered invoice, so the two documents cannot disagree.
- **Standalone** — `POST /atlas/v1/eway/{ewb-provider}/eway-bill`
  (`eas_basic_v1_generate_ewaybill`), for movements with no e-invoice behind them.

## Steps

1. **Authenticate** with `login`, exactly as in the e-invoice skill. Reuse the token.
2. **Generate** by one of the two routes above. Keep the returned E-Way Bill number.
3. **Read it back** with `GET /atlas/v1/eway/{ewb-provider}/eway-bill/{ewb-number}`
   (`eas_basic_v1_fetch_ewaybill`) to confirm validity dates before the vehicle leaves.
4. **Render for the driver** with `GET /atlas/v1/eway/{ewb-provider}/eway-bill/{ewb-number}/pdf`
   (`eas_enriched_v1_get_ewaybill_pdf`).

## In transit

- **Vehicle changed** — `PUT /atlas/v1/eway/{ewb-provider}/eway-bill/part-b`
  (`eas_basic_v1_update_ewaybill_partb`). Part B carries the conveyance; it must match the vehicle
  actually carrying the goods.
- **Transporter changed** — `PATCH /atlas/v1/eway/{ewb-provider}/eway-bill/transporter`
  (`eas_basic_v1_update_ewb_transporter`).
- **Running out of time** — `PATCH /atlas/v1/eway/{ewb-provider}/eway-bill/extend-validity`
  (`eas_basic_v1_extend_ewaybill_validity`). Extension is only available around expiry; Vayana does
  not publish the eligible window, so treat expiry as a hard deadline you must plan around.
- **Multi-vehicle movement** — initiate the group, then add and change vehicles within it via the
  multi-vehicle operations.

## Reversing it

`PATCH /atlas/v1/eway/{ewb-provider}/eway-bill/cancel` (`eas_basic_v1_cancel_ewaybill`) cancels a
bill you raised. `reject` (EAS `POST .../ewayapi/reject`) is the other side: refusing a bill someone
else raised against your GSTIN.

**No cancellation window is published by Vayana.** The NIC portal enforces one. Confirm it there.

Note also that **Close E-Way Bill exists only in sandbox** — do not build a production flow that
depends on closing a bill.

## Failure handling

- The E-Way Bill error vocabulary is the NIC portal's own, and Vayana exposes the whole dictionary
  as a callable operation: `GET /atlas/v1/eway/{ewb-provider}/eway-bill-error`
  (`eas_basic_v1_fetch_ewaybill_error_list`). Call it once, cache it, and resolve numeric error
  codes against it rather than string-matching messages.
- A snapshot of all 287 codes is in `errors/vayana-network-problem-types.yml`, but the live
  operation is authoritative.
- **No idempotency key exists.** A retried generate can produce a second E-Way Bill for one
  consignment. Fetch by consigner document number before retrying.
