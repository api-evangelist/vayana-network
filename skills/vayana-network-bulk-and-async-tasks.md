---
name: vayana-network-bulk-and-async-tasks
description: Run bulk e-invoice and E-Way Bill operations through Vayana Atlas and collect their results correctly using the long-running task pattern — status, result, paged result and zip download.
api: vayana-network-atlas
generated: '2026-09-02'
method: generated
source: openapi/vayana-network-atlas-openapi.json
operations:
- login
- eas_enriched_v1_generate_invoices
- eas_enriched_v1_generate_bulk_ewaybill_by_invoice
- eas_enriched_v1_gcs_generate
- eas_enriched_v1_task_status
- eas_enriched_v1_result_task
- eas_enriched_v1_download_task
---

# Run bulk work and collect the results

Every bulk operation in this API is asynchronous. It does not return your data — it returns a
`task-id`. Getting this pattern right is most of the work of integrating the enriched surface.

## The pattern

1. **Submit.** For example `POST /atlas/v1/irp/{irp}/einvoice/bulk`
   (`eas_enriched_v1_generate_invoices`) for bulk IRN generation,
   `POST /atlas/v1/irp/{irp}/eway-bill/bulk` (`eas_enriched_v1_generate_bulk_ewaybill_by_invoice`)
   for bulk E-Way Bills, or `POST /atlas/v1/gcs/bulk` (`eas_enriched_v1_gcs_generate`).
   The response is `{"status": "1", "data": {"task-id": "<uuid>"}}`.

2. **Poll status.** `GET /atlas/v1/eas-task/{task-id}` (`eas_enriched_v1_task_status`).
   Send `accept: application/json`. Wait for `data.task.status == "completed"`.
   **Nothing else works until it is completed** — result and download both fail before that.
   The status response also reports the maximum page number, which you need for step 3.

3. **Collect**, one of two ways:
   - `GET /atlas/v1/eas-task/{task-id}/result` (`eas_enriched_v1_result_task`) with
     `accept: application/json` for the JSON result.
   - `GET /atlas/v1/eas-task/{task-id}/download` (`eas_enriched_v1_download_task`) with
     `accept: application/zip` for the packaged files.

   For large results, page through the EAS route
   `/enriched/tasks/{flynn-version}/result/{task-id}/page/{page-number}`, from `0` to the maximum
   page the status call reported. There is no cursor and no next link.

## Rules that will bite you

- **`accept` is enforced per operation.** JSON on status/result/pages, zip on download. The wrong
  value is a documented failure, not a content-negotiation fallback.
- **Results expire after 7 days** from generation of the reference token. Collect and persist
  within that window or the work is lost and must be re-run — and re-running a bulk *generate*
  against a government portal is not free of consequence.
- **`404 err-resource-not-found`** means the `task-id` is unknown, or the `page-number` is out of
  range. The response `args` echoes the `task-id` you sent — check it against what you stored.
- **`400`** means the task is not in a state that supports the call you made; poll status first.
- **Batch caps are real:** verification calls take at most 50 IRNs and reject duplicates within a
  single call; PDF verification caps uploads at 5 GB.

## Retry policy

There is **no idempotency key on any operation**. If a bulk submission times out, do not simply
resubmit — you may register the whole batch twice. Poll for a `task-id` you already hold, or
reconcile with a fetch, before re-sending.
