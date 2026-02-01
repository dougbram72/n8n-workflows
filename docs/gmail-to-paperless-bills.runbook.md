# Runbook – gmail-to-paperless-bills.v1

## Overview

This workflow ingests **Gmail bills/receipts** into **Paperless‑ngx** once per day. It is designed to be:

- **Idempotent** — no duplicate documents when retried.
- **Observable** — every run is logged with a `run_id` and counts.
- **Safe** — failed uploads are sent to a **review queue** instead of being silently dropped.

## Trigger

- **Type:** Cron (n8n Cron node)
- **Intended schedule:** Daily at 07:05 America/Chicago (set this in the Cron node parameters).
- **Concurrency:** Single run at a time is sufficient.

## Data flow

1. **Cron Trigger → Set Run ID**
   - `Set Run ID` generates a `run_id` like `run_2026-02-01T13:05:00.123Z`.
   - This `run_id` is injected into downstream items.

2. **Set Run ID → List Bill Emails (Gmail)**
   - Gmail node searches the last 7 days for **bill/receipt‑like** emails:
     - Query:  
       `in:inbox newer_than:7d (subject:(invoice OR bill OR statement OR "payment due" OR receipt) OR from:(baelectric.com OR smarthub.coop OR rainbowcommunications.com))`
   - Response includes full message payloads (`format: full`).

3. **List Bill Emails → Filter + Deduplicate (Function)**
   - Extracts:
     - `gmail_id` (Gmail `id`)
     - `messageId` (RFC‑822 `Message-ID` header)
     - `subject`, `from`, `internalDate`
     - `payload` (for later attachment handling)
   - **Idempotency key:** `messageId`
   - ***TODO:*** This node is currently a stub; it should be extended to:
     - Look up `messageId` in a **Processed IDs DataTable**.
     - Mark items with `isNew = false` if they already exist.
     - Only pass `isNew = true` items forward.

4. **Filter + Deduplicate → Prepare For Paperless (Function)**
   - For each `isNew` item:
     - Builds a `title` for Paperless, e.g.:  
       `YYYY-MM-DD – <from> – <subject>`
     - Keeps `run_id`, `gmail_id`, `messageId`, `subject`, `from`, `internalDate`.

5. **Prepare For Paperless → Upload To Paperless (HTTP Request)**
   - Endpoint:  
     `POST {{$env.PAPERLESS_BASE_URL}}/api/documents/post_document/`
   - Headers:
     - `Authorization: Token {{$env.PAPERLESS_API_TOKEN}}`
   - Body:
     - `multipart/form-data` with at least:
       - `title` – from `Prepare For Paperless`.
       - `document` – **binary** PDF (attachment or generated from HTML).
     - Optional fields (set in n8n UI):
       - `correspondent`
       - `document_type`
       - `tags` (e.g., `bills`, `electric`, `2026`).

6. **Upload To Paperless → Mark Processed (Function)**
   - On success:
     - Writes the `messageId` and `run_id` to the **Processed IDs store** (stubbed now; should become a DataTable or DB write).
     - Ensures future runs skip this `messageId`.

7. **Mark Processed → Log Run Summary (Function)**
   - Aggregates a simple summary:
     - `run_id`
     - `timestamp`
     - `totalUploaded`
   - ***TODO:*** Replace with a DataTable node writing to a **Run Log** table.

8. **Error Path → Queue Failed Docs (Function)**
   - If `Upload To Paperless` (or other critical step) fails:
     - Route the failing item into `Queue Failed Docs`.
     - Writes:
       - `run_id`, `gmail_id`, `messageId`, `subject`, `from`, `error`
     - ***TODO:*** Replace with a DataTable node writing into a **Review Queue** table.

## Idempotency

- **Key:** `messageId` (RFC‑822 `Message-ID` header), not Gmail `id`.
- **Store:** `Processed IDs` DataTable (or DB) with at least:
  - `messageId` (unique)
  - `first_run_id`
  - `first_processed_at`
- **Behavior:**
  - On every run:
    - Before uploading, check if `messageId` exists.
    - If yes → skip (already ingested).
    - If no → upload and insert into Processed IDs.

This ensures re‑runs (due to errors, outages) do not create duplicate documents in Paperless.

## Logging & observability

- **Run Log table** (DataTable/DB):
  - Columns:
    - `run_id` (primary key)
    - `started_at`
    - `finished_at`
    - `total_found`
    - `total_new`
    - `total_uploaded`
    - `total_failed`
  - The **Log Run Summary** node should be extended to:
    - Write one row per run.
    - Update `finished_at`, `total_*` once the workflow completes.

- Optionally, send a notification (email/Signal via Alfred) when:
  - `total_failed > 0`, or
  - `total_new > X` (unexpected surge).

## Review queue

- **Review Queue table** (DataTable/DB):
  - Columns:
    - `run_id`
    - `messageId`
    - `gmail_id`
    - `subject`
    - `from`
    - `error`
    - `status` (`pending`, `reprocessed`, `ignored`)
  - When `Upload To Paperless` fails:
    - Write one row per failed item.
  - A separate, manual or semi‑automated workflow can:
    - Pull from this queue.
    - Attempt re‑upload with human oversight.
    - Mark `status` appropriately.

## Configuration

### Environment variables

Set these on the n8n host:

- `PAPERLESS_BASE_URL`  
  e.g. `http://192.168.1.7:8000`
- `PAPERLESS_API_TOKEN`  
  API token from your Paperless‑ngx profile.

### n8n credentials

- **Gmail OAuth2** credential:
  - Must have permission to read relevant email.
  - Referenced in the **List Bill Emails** node.

### DataTables / DB

Create:

1. `Processed IDs` (for idempotency)
2. `Run Log` (for run summaries)
3. `Review Queue` (for failed docs)

Wire the stub Function nodes to actual DataTable nodes using n8n’s DataTable node (or your DB of choice).

## Failure modes & recovery

- **Gmail unavailable / quota exceeded:**
  - Workflow should fail early; log run with `total_failed` and consider a notification.
- **Paperless API unreachable:**
  - All uploads will fail; items go to Review Queue.
  - After Paperless is back up, re‑run a “replay failed docs” workflow against the Review Queue.
- **Incorrect Gmail query (too broad / too narrow):**
  - Adjust `q` in `List Bill Emails`.
  - Check Run Log for sudden drops/spikes in `total_new`.

## No-silent-failure gates

- If `List Bill Emails` finds more than a configured upper threshold (e.g., >200 new candidates), consider:
  - Stop processing and send a “suspicious volume” alert.
- If `Upload To Paperless` fails for a large percentage of new items:
  - Stop processing, write to Review Queue, send alert.
