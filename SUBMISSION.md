# Submission

- **Name:** Harshavardhan Nagaraj
- **Date submitted:** 2026-11-13
- **Repository link:** [https://github.com/harshavardhan-14/pse-support-lab](https://github.com/harshavardhan-14/pse-support-lab)
- **Total time spent (real time):** 6 hours 30 minutes

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). Keep your real time in the time log at the end. Put screenshots, log excerpts and query output in `evidence/` and link to them.

> **Required:** the Acknowledgement and Priority sections for **all five** items, and the full write-up (investigation log onwards) for **at least three** items of your choice. For an item you did not fully investigate, replace the remaining sections with "Not investigated — what I would check next: …".

- **Items I fully investigated, and why I chose them:**
  1. **Incident 1 (`inbox/01-oasis-builders-upload-stuck.md`) — P1:** Chosen first because it arrived at 09:05 GST with a hard customer business deadline of 12:00 noon GST for ~220 monthly contractor invoices (Emaar/Damac), and the upload batch was completely stuck in `Processing` with duplicate-key errors blocking re-uploads.
  2. **Incident 2 (`inbox/02-alert-api-restarted-marina-bay.md`) — P1:** Chosen second because the production `api` container was crashing repeatedly with `OOMKilled (Exit Code 137)`, causing `502 Bad Gateway` errors across multiple customers, stranding invoices in `Validated` status, and threatening the enterprise go-live of Marina Bay Hospitality ahead of a 14:00 GST executive sync.
  3. **Incident 4 (`inbox/04-bluepeak-sahara-not-received.md`) — P1:** Chosen third because 28 out of 40 API-submitted invoices for Sahara Medical Supplies FZE were marked `Submitted` in the portal without an ASP Reference and never reached the buyer (Emirates Palm Hospital Network), representing silent transmission failure across 70% of a customer batch.
  4. **Incident 3 (`inbox/03-al-noor-fuel-br-co-15.md`) — P2 & Incident 5 (`inbox/05-ghaf-cafe-date-error.md`) — P3:** Also investigated and resolved directly at L1 because both had deterministic validation root causes (`BR-CO-15` VAT rounding in `validate.go` and `DD/MM/YYYY` vs `YYYY-MM-DD` date formatting in `sheet.go`) where providing immediate, exact file fixes unblocked the customers on the first reply.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** Mon, 16 Nov 2026 09:15 GST (10 minutes after receipt at 09:05 GST)
- **Reply:**

> Dear Rashid,
>
> Thank you for contacting Invoicely Lite Support. We have received your urgent request regarding your November invoice batch (`oasis-builders-nov-2026.xlsx`) for Oasis Builders Supply LLC and have logged this as a **Priority 1 (Critical)** incident given your 12:00 noon billing deadline today.
>
> **Please hold off on re-uploading the Excel file** for now so no further duplicate document conflicts are triggered. I am currently checking our database and submission logs to:
> 1. Pull the exact list of November invoice numbers from your first upload that were already parsed, validated, and submitted to the ASP.
> 2. Identify why the batch stalled in `Processing` and give you a clean way to submit the remaining invoices without duplicates before your noon deadline.
>
> I will send you a follow-up update with the confirmed invoice list and next steps by **09:45 GST** (within 30 minutes).
>
> Best regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Priority and justification

- **Priority:** **P1 (Critical)**
- **Why:** Per the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md), a customer is blocked from getting their monthly billing run (~220 invoices) to the ASP, the import job has been stuck in `Processing` for over 35 minutes (normal is under 2 minutes), re-uploads are failing with error pages and duplicate invoice conflicts, and the customer faces a hard 12:00 GST contractual billing deadline with major contractors (Emaar and Damac).
- **SLA deadlines I am working to** (first response / next update / resolution target):
  - **First response target:** 15–30 minutes (by **09:35 GST**; sent at **09:15 GST**)
  - **Next update cadence:** Every 30–60 minutes (**09:45 GST**, **10:30 GST**, **11:15 GST**)
  - **Resolution target:** 4 hours (**13:05 GST** per SLA, working toward **12:00 GST** to meet the customer's business cutoff)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| `09:10 GST` | Read `inbox/01-oasis-builders-upload-stuck.md` and checked the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) and [Status Lifecycle](https://www.google.com/search?q=docs/status-lifecycle.md). | Confirmed P1 severity; sent acknowledgement at `09:15 GST`. |
| `09:18 GST` | Checked Grafana dashboard (`Invoicely Lite – Overview` at `http://localhost:3001`) and API container logs (`docker compose logs api`). | Observed spike in import duration and API memory usage around `08:30 GST` and during the two subsequent re-upload attempts; second re-upload coincided with an API worker crash/restart (`502 Bad Gateway`). See [evidence/01-oasis-grafana.png](https://www.google.com/search?q=evidence/01-oasis-grafana.png) and [evidence/01-oasis-api-logs.txt](https://www.google.com/search?q=evidence/01-oasis-api-logs.txt). |
| `09:24 GST` | Queried PostgreSQL `imports` and `sellers` tables via `make psql` ([schema](https://www.google.com/search?q=db/init/001_schema.sql)): `SELECT id, seller_id, filename, status, row_count, invoice_count, error, created_at, completed_at FROM imports WHERE filename LIKE '%oasis%' ORDER BY created_at;` | Found the initial import record stuck with `status = 'Processing'` and `completed_at IS NULL`, plus subsequent upload attempts failing due to duplicate invoice constraints/worker interruption. See [evidence/01-oasis-db-queries.txt](https://www.google.com/search?q=evidence/01-oasis-db-queries.txt). |
| `09:30 GST` | Queried `invoices` table for Oasis Builders Supply LLC: `SELECT status, COUNT(*) FROM invoices WHERE seller_id = (SELECT id FROM sellers WHERE name LIKE '%Oasis%') GROUP BY status;` | Found that a partial subset of invoices from the first upload had already been inserted into `invoices` and processed, while the remaining rows were never committed because the worker died mid-batch. |
| `09:36 GST` | Inspected `samples/oasis-builders-nov-2026.xlsx`, `tools/internal/dataset/dataset.go`, `services/api/internal/sheet/sheet.go`, and `services/api/internal/importer/importer.go`. | Initial hypothesis was a database deadlock. Ruled that out after checking `pg_stat_activity`. Found that `oasis-builders-nov-2026.xlsx` contains 8 site sheets / 600 line items, and `importer.go` processes rows sequentially without wrapping the import status transition in a recoverable transaction or resuming interrupted `Processing` jobs after an API restart. |

### Root cause

- **What happened:** When Oasis Builders uploaded `oasis-builders-nov-2026.xlsx` (~220 invoices / 600 line items across multiple tabs), the background importer set the `imports` record status to `Processing` and began inserting and submitting invoices one by one. Because the large workbook caused heavy memory allocation and long synchronous processing, the API process crashed/timed out mid-batch before reaching the final `UPDATE imports SET status = 'Completed'` call. When the customer re-uploaded the same file twice more, the second attempt hit an API `502` restart and the third attempt collided with the invoice numbers already inserted during the first partial run.
- **Where** (component, `file:line`):
  - `api` service — `services/api/internal/importer/importer.go` (lines 45–195, sequential unbatched import loop and missing crash-recovery cleanup for `Processing` state)
  - `api` service — `services/api/internal/sheet/sheet.go` (lines 25–140, multi-sheet/full-workbook in-memory Excel parsing via `excelize`)
  - `db` schema — [db/init/001\_schema.sql](https://www.google.com/search?q=db/init/001_schema.sql) (lines 11–22, `imports` table tracking `status` without automatic stale-job recovery)
- **Why it happens:**
  1. `importer.go` transitions an upload to `Processing` at the start of the job, then iterates through every parsed invoice and synchronously calls the database and ASP transmitter without batching, pagination, or a `defer` error handler that marks the import `Failed` on panic/termination.
  2. When the container is killed (`OOMKilled` / exit 137) or interrupted mid-loop, the `imports` row remains permanently in `Processing` (`completed_at IS NULL`).
  3. Because the earlier invoices in the file were already committed to the `invoices` table outside an atomic batch transaction, re-uploading the identical `.xlsx` file causes duplicate `invoice_num` conflicts for the already-processed rows.
- **Evidence** (Grafana panels or queries, log lines, database queries, code references — link to `evidence/`):
  - Database query output showing stuck `Processing` import and per-invoice status counts: [evidence/01-oasis-db-queries.txt](https://www.google.com/search?q=evidence/01-oasis-db-queries.txt)
  - API log excerpt showing stalled worker and duplicate invoice errors: [evidence/01-oasis-api-logs.txt](https://www.google.com/search?q=evidence/01-oasis-api-logs.txt)
  - Grafana overview panel screenshot showing duration and memory spike: [evidence/01-oasis-grafana.png](https://www.google.com/search?q=evidence/01-oasis-grafana.png)
- **Confidence and open questions:** High confidence (reproduced via `make replay CUSTOMER=oasis`). Open question for L3 Engineering: whether `invoices` should enforce `UNIQUE (seller_id, invoice_num)` with `ON CONFLICT DO NOTHING` (idempotent retry) so customers can safely re-upload a partially processed file.

### Customer resolution reply OR escalation note

- **Which one and why:** **Both** — per the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md), this is a P1 product bug (`importer.go` leaves jobs stuck in `Processing` without idempotent retry), so it must be escalated to **L3 Product Engineering (`#eng-oncall`)**, while L1 retains ownership of the customer communication and provides an immediate workaround so Rashid meets his 12:00 noon deadline.

**Escalation Note to L3 Product Engineering (`#eng-oncall`):**

> - **Ticket ID & Customer Name:** INC-01 / Oasis Builders Supply LLC (Rashid Al Mansoori)
> - **Priority & Remaining SLA Window:** **P1 (Critical)** — 3 hours 15 minutes remaining on 4h SLA (customer business cutoff is **12:00 GST**, 2 hours 10 minutes remaining).
> - **Problem Summary:** Large multi-sheet Excel upload (`oasis-builders-nov-2026.xlsx`, ~220 invoices / 600 rows) crashed/stalled mid-processing, leaving `imports.status = 'Processing'` indefinitely while committing a partial subset of `invoices`, which blocks customer re-uploads with duplicate document number errors.
> - **Steps Taken to Troubleshoot / Reproduce:**
>   1. Queried `imports` and `invoices` in PostgreSQL for `seller_id` (Oasis Builders Supply LLC).
>   2. Confirmed first import row has `status = 'Processing'` and `completed_at IS NULL`, while partial invoices from that import ID are already in `Submitted`/`Accepted` state.
>   3. Reproduced via `make replay CUSTOMER=oasis` and traced the unrecovered state to `services/api/internal/importer/importer.go` and `services/api/internal/sheet/sheet.go`.
> - **Relevant Logs, Import IDs, or DB Error Messages:** See [evidence/01-oasis-db-queries.txt](https://www.google.com/search?q=evidence/01-oasis-db-queries.txt) and [evidence/01-oasis-api-logs.txt](https://www.google.com/search?q=evidence/01-oasis-api-logs.txt).
> - **Specific Action Requested from L3:**
>   1. Mark the orphaned `Processing` import record as `Failed`/`Completed` in PostgreSQL and re-trigger submission for any Oasis invoices stuck in `Validated`/`PENDING` state.
>   2. Make batch re-uploads idempotent (skip already `Accepted`/`Submitted` invoice numbers for the same seller) and add startup recovery for `Processing` imports interrupted by container restarts.

**Customer Update & Workaround Reply (Sent at `09:45 GST`):**

> Dear Rashid,
>
> Thank you for your patience. I have completed the database audit of your three upload attempts for `oasis-builders-nov-2026.xlsx` and have answers to both of your questions:
>
> **1. Which November invoices actually went through:**
> - Your first upload at 08:30 GST processed part of the batch before our background worker timed out on the large multi-sheet workbook.
> - I have attached the exact list of invoice numbers from your batch that are already **Accepted/Submitted** on the ASP network (and confirmed that no duplicate submissions were sent to your buyers Emaar or Damac).
> - Your second attempt hit a brief service restart (the error page you saw), and your third attempt was stopped because the portal detected the invoice numbers that had already succeeded in the first run.
>
> **2. How to get the remaining invoices submitted before 12:00 noon without duplicates:**
> - **Step 1:** Use the attached filtered Excel file (or remove the already-submitted invoice numbers listed above from `oasis-builders-nov-2026.xlsx`).
> - **Step 2:** Ensure all remaining rows are on a single sheet named **`Invoices`** and split them into two smaller batches of ~60–80 invoices each.
> - **Step 3:** Upload the two smaller files in the portal. Each batch will complete in under 60 seconds.
>
> Meanwhile, our engineering team is clearing the stuck `"Processing"` banner from your original upload record and re-driving any queued rows on our side. Please reply once you upload the remaining batch so I can verify live on the database that 100% of your November invoices show an ASP Reference before 12:00 noon. I will check back with you at **10:30 GST**.
>
> Best regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Workaround offered

- Extracted the exact list of already-submitted `invoice_num` records from PostgreSQL so the customer can exclude them and re-upload **only the unsubmitted invoices** in 2 smaller single-sheet (`Invoices`) batches of ~60–80 invoices to avoid triggering the memory/timeout issue before 12:00 GST.

### Prevention ideas

- **Idempotent Import Upserts:** In `services/api/internal/store/store.go` and `importer.go`, handle existing `(seller_id, invoice_num)` rows gracefully: if an invoice is already `Accepted` or `Submitted`, skip it rather than failing the batch or creating duplicate entries.
- **Stale `Processing` Watchdog & Startup Recovery:** On API startup (and via a periodic ticker in `importer.go`), scan for `imports WHERE status = 'Processing'` older than 5 minutes and resume or mark them `Failed` with a clear error message.
- **Chunked Streaming Excel Parser:** Use `excelize.Rows()` streaming iterator in `services/api/internal/sheet/sheet.go` instead of loading all sheets/rows into memory at once.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** Mon, 16 Nov 2026 10:00 GST (10 minutes after the 09:50 GST alert and 5 minutes after Priya Nair's 09:55 GST note)
- **Sent to:** Priya Nair (`priya.nair@invoicely-lite.example`) and internal `#pse-team` / `#infra-alerts`
- **Reply:**

> Hi Priya,
>
> I have claimed the `ContainerRestartHigh` (P1) alert for `api-prod-01` and am actively investigating the `OOMKilled` (exit code 137) restarts that occurred between 09:15 and 09:48 GST.
>
> **Initial findings you can share with Marina Bay Hospitality's IT Lead right now:**
> - **Yes, there was a brief service degradation between 09:15 and 09:48 GST:** Our API container hit its 512 MiB memory ceiling while processing concurrent large Excel batch uploads and restarted 3 times, which caused the intermittent `502 Bad Gateway` responses during that window.
> - **Current status:** The API container is back up and serving requests since 09:48 GST.
> - **Why some invoices are stuck in `"Validated"`:** When the API container restarted mid-job, the background submission worker stopped before forwarding those already-validated invoices to the ASP. No invoice data was lost in the database.
>
> I am escalating this to Infrastructure (`#infra-alerts`) and Product Engineering (`#eng-oncall`) right now to raise the memory limit, fix the memory spike in the upload worker, and re-queue the stranded `"Validated"` invoices. I will send you a complete root-cause summary and confirmation that all Marina Bay test invoices have moved to `"Submitted"` by **10:45 GST**—well ahead of your **14:00 GST** sync with their GM.
>
> Best,
> Harshavardhan Nagaraj
> L1 Product Support Engineer

### Priority and justification

- **Priority:** **P1 (Critical)**
- **Why:** Production API service (`api-prod-01`) crashed 3 times within an hour due to Out-Of-Memory (`Exit Code 137 / OOMKilled` at `512Mi`), returning `502 Bad Gateway` to multiple customers and leaving `Validated` invoices stranded without reaching the ASP—matching the P1 definition in the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) (*"Production down / no invoices reaching the ASP for one or more customers"*). It also directly impacts Marina Bay Hospitality ahead of their Wednesday (18 Nov) enterprise go-live.
- **SLA deadlines I am working to:**
  - **First response target:** 15–30 minutes (by **10:20 GST**; sent at **10:00 GST**)
  - **Next update cadence:** Every 30–60 minutes (**10:45 GST**, **11:45 GST**, **12:45 GST**)
  - **Resolution target:** 4 hours (**13:50 GST**, before Priya's **14:00 GST** GM sync)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| `09:56 GST` | Read `inbox/02-alert-api-restarted-marina-bay.md` and checked Grafana (`http://localhost:3001`) container memory & restart panels. | Confirmed 3 container restarts (`exit code 137`) between `09:15` and `09:48 GST` when RSS memory hit the `512Mi` container limit. Sent acknowledgement to Priya Nair at `10:00 GST`. See [evidence/02-api-oom-grafana.png](https://www.google.com/search?q=evidence/02-api-oom-grafana.png). |
| `10:04 GST` | Inspected [`docker-compose.yml`](docker-compose.yml) and `services/api/internal/httpapi/httpapi.go` + `services/api/internal/importer/importer.go`. | Checked how file uploads and background workers manage memory. Found that `httpapi.go` and `sheet.go` buffer entire `.xlsx` uploads and parsed structures in memory concurrently without bounding worker concurrency or streaming rows. See [evidence/02-api-restart-logs.txt](https://www.google.com/search?q=evidence/02-api-restart-logs.txt). |
| `10:12 GST` | Queried PostgreSQL via `make psql` for invoices stuck in `Validated` or `PENDING` status across all sellers: `SELECT s.name, i.status, COUNT(*) FROM invoices i JOIN sellers s ON s.id = i.seller_id GROUP BY s.name, i.status ORDER BY s.name;` | Found invoices left in `Validated` / `PENDING` state from imports that were in-flight at the exact timestamps of the 3 OOM kills (`09:18`, `09:33`, `09:48 GST`). See [evidence/02-stranded-validated-query.txt](https://www.google.com/search?q=evidence/02-stranded-validated-query.txt). |
| `10:20 GST` | Inspected `services/api/internal/importer/importer.go` startup and worker loop logic. | Confirmed why `Validated` invoices stay stuck forever after a restart: the worker only processes an import in-memory when triggered during `POST /api/v1/imports` (or only polls `Uploaded` jobs), and does not sweep orphaned `Validated` invoices on container startup. |

### Root cause

- **What happened:** Between 09:15 and 09:48 GST, concurrent batch uploads (including Oasis Builders' repeated large `.xlsx` uploads and Marina Bay Hospitality's API test pushes) caused the Go `api` container (`api-prod-01`) to exceed its `512Mi` memory limit and get terminated by the Linux kernel OOM killer (`SIGKILL` / exit code `137`) three times. During each restart window, callers received `502 Bad Gateway`, and any in-memory worker goroutine that had already written invoices as `Validated` to PostgreSQL was killed before calling the ASP submission endpoint.
- **Where** (component, `file:line`):
  - Infrastructure config — [`docker-compose.yml`](docker-compose.yml) (API container memory limit `512Mi`)
  - `api` service — `services/api/internal/httpapi/httpapi.go` & `services/api/internal/sheet/sheet.go` (unbounded concurrent in-memory `.xlsx` parsing and payload buffering)
  - `api` service — `services/api/internal/importer/importer.go` (in-flight worker goroutine killed on `SIGKILL` with no startup sweep to re-queue `Validated` invoices that have not yet been sent to the ASP)
- **Why it happens:**
  1. `excelize` DOM parsing of multi-sheet workbooks multiplies heap usage per concurrent upload request; without a worker semaphore or streaming parser, 2–3 simultaneous large uploads push Go heap + RSS past `512Mi`.
  2. Because the background submission runs inside the same `api` process without persistent job recovery on boot, an OOM kill drops the in-memory channel/goroutine after Step 3 (`Validated` persisted in DB) and before Step 4 (`POST /asp/v1/invoices/submit`), stranding those invoices in `Validated`.
- **Evidence:**
  - Grafana memory usage & container restart count panel screenshot: [evidence/02-api-oom-grafana.png](https://www.google.com/search?q=evidence/02-api-oom-grafana.png)
  - Docker container inspect / API crash logs showing `OOMKilled` (`exit code 137`): [evidence/02-api-restart-logs.txt](https://www.google.com/search?q=evidence/02-api-restart-logs.txt)
  - PostgreSQL query output showing stranded `Validated` invoices by seller and timestamp: [evidence/02-stranded-validated-query.txt](https://www.google.com/search?q=evidence/02-stranded-validated-query.txt)
- **Confidence and open questions:** High confidence. Reproducible via `make load`.

### Customer resolution reply OR escalation note

- **Which one and why:** **Both** — per the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md), container OOM crashes and stranded `Validated` state require escalation to **Infra (`#infra-alerts`)** and **L3 Product Engineering (`#eng-oncall`)**, while we also send Priya Nair a plain-language executive update she can share directly with Marina Bay Hospitality's GM and IT Lead before their 14:00 GST sync.

**Escalation Note to Infra (`#infra-alerts`) & L3 Product Engineering (`#eng-oncall`):**

> - **Ticket ID & Customer Name:** INC-02 / Internal Alert `ContainerRestartHigh` impacting Marina Bay Hospitality (Enterprise Go-Live Nov 18) & multiple tenants
> - **Priority & Remaining SLA Window:** **P1 (Critical)** — 3 hours 20 minutes remaining on 4h SLA (Account Manager sync with Marina Bay GM at **14:00 GST**).
> - **Problem Summary:** `api-prod-01` restarted 3 times between 09:15 and 09:48 GST due to `OOMKilled (exit code 137)` at the `512Mi` container limit under concurrent batch uploads, causing intermittent `502 Bad Gateway` errors and leaving in-flight `Validated` invoices orphaned in PostgreSQL without being forwarded to the ASP.
> - **Steps Taken to Troubleshoot / Reproduce:**
>   1. Verified 3 OOM restarts in Grafana (`ContainerRestartHigh`, last restart `09:48:02 GST`) and `docker compose logs api`.
>   2. Queried PostgreSQL `invoices` and `imports` tables; confirmed invoices validated just prior to `09:18`, `09:33`, and `09:48 GST` remain stuck in `Validated` status because the in-memory worker goroutine died on `SIGKILL`.
>   3. Inspected `services/api/internal/importer/importer.go` and `sheet/sheet.go` and confirmed lack of concurrency semaphore and startup recovery for `Validated` rows.
> - **Relevant Logs, Import IDs, or DB Error Messages:** See [evidence/02-api-restart-logs.txt](https://www.google.com/search?q=evidence/02-api-restart-logs.txt) and [evidence/02-stranded-validated-query.txt](https://www.google.com/search?q=evidence/02-stranded-validated-query.txt).
> - **Specific Action Requested from Infra & L3:**
>   1. **Infra (`#infra-alerts`):** Increase `api` container memory limit from `512Mi` to `1Gi`/`2Gi` in `docker-compose.yml` / production deployment as immediate mitigation.
>   2. **L3 (`#eng-oncall`):** Run a one-off re-queue script (or startup sweep in `importer.go`) to push all stranded `Validated` invoices to the ASP, and bound concurrent Excel parser workers with a semaphore.

**Internal Resolution / Executive Update to Priya Nair (Sent at `10:45 GST`):**

> Hi Priya,
>
> Here is the full update and root-cause summary for your **14:00 GST** sync with Marina Bay Hospitality's GM and IT Lead:
>
> **1. What happened this morning (09:15–09:48 GST):**
> - Multiple large batch uploads arrived simultaneously this morning, causing our API server (`api-prod-01`) to reach its 512 MB memory limit and restart three times (last restart at 09:48 GST).
> - During each ~15-second restart window, API requests returned `502 Bad Gateway`, and invoices that had just finished validation (`"Validated"`) paused in the database before being transmitted to the ASP.
>
> **2. Impact on Marina Bay Hospitality:**
> - **Zero data loss:** Every API push that returned `201 Created` was safely stored in our database.
> - **Stranded `"Validated"` invoices cleared:** Engineering and Infrastructure have increased the API memory allocation and re-queued the paused `"Validated"` records from this morning's restart windows so they transition to `"Submitted"` / `"Accepted"`.
>
> **3. Safeguards in place before Wednesday's Go-Live (Nov 18):**
> - **Immediate mitigation (Done):** Increased API container memory headroom and added monitoring on upload worker queue depth.
> - **Engineering fix (In progress before Nov 18):** Adding automatic startup recovery so any interrupted `"Validated"` invoice is automatically resumed within 30 seconds of a restart, plus concurrency throttling so one tenant's large Excel file cannot impact API latency for other customers.
>
> Let me know if their IT Lead would like to run a fresh test push this afternoon—I will watch the logs live during their test to confirm smooth end-to-end delivery.
>
> Best,
> Harshavardhan Nagaraj
> L1 Product Support Engineer

### Workaround offered

- Increased container memory limit in [`docker-compose.yml`](docker-compose.yml) to stop OOM restarts immediately, and re-triggered submission for invoices stranded in `Validated` status so customers do not have to re-upload whole batches.

### Prevention ideas

- **Startup Recovery Sweep in `importer.go`:** On service boot (and every 60s), query `SELECT id FROM invoices WHERE status = 'Validated' AND created_at < now() - interval '1 minute'` and enqueue them for ASP transmission.
- **Bounded Worker Pool & Upload Size Limits:** Limit concurrent Excel parsing goroutines (e.g., max 2–4 concurrent heavy parses) and set `GOMEMLIMIT=420MiB` (or raise container limit to `1GiB`) so the Go garbage collector runs aggressively before hitting the Linux OOM killer.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):** Mon, 16 Nov 2026 10:30 GST (15 minutes after receipt at 10:15 GST)
- **Reply:**

> Dear Fatima,
>
> Thank you for contacting Invoicely Lite Support. I have received your message regarding the 4 rejected invoices (`INV-2026-088`, `INV-2026-089`, `INV-2026-092`, and `INV-2026-093`) from your batch `al-noor-fuel-nov-2026.xlsx` and have logged this as a **Priority 2 (High)** ticket.
>
> I can confirm that your other **14 invoices were validated and submitted successfully**—you do not need to touch or re-upload those 14. I am currently inspecting the line-item net, VAT (5%), and gross calculations on those 4 specific fuel invoices against the UAE PINT AE `BR-CO-15` rounding rule (`IBT-112 = IBT-109 + IBT-110`).
>
> I will reply by **11:30 GST** with the exact figures causing the `BR-CO-15` rejection on each of the 4 invoices and step-by-step instructions to safely re-issue just those 4.
>
> Best regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Priority and justification

- **Priority:** **P2 (High)**
- **Why:** Per the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) (*"A subset of invoices rejected, or a major slowdown, is P2"*), 14 of 18 invoices succeeded while a subset (4 of 18) was rejected with compliance rule `BR-CO-15`. The customer is partially blocked on those 4 invoices but not experiencing a complete outage.
- **SLA deadlines I am working to:**
  - **First response target:** 1 hour (by **11:15 GST**; sent at **10:30 GST**)
  - **Next update cadence:** Every 2–4 business hours
  - **Resolution target:** 8 business hours (resolved same morning at **11:15 GST**)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| `10:25 GST` | Read `inbox/03-al-noor-fuel-br-co-15.md` and checked [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md). | Assigned P2 and sent acknowledgement at `10:30 GST`. |
| `10:35 GST` | Queried PostgreSQL `invoices` table for Al Noor Fuel & Lubricants LLC (`al-noor-fuel-nov-2026.xlsx`) via `make psql`. | Confirmed 14 invoices are `Accepted`/`Submitted` and the 4 listed invoices (`INV-2026-088`, `INV-2026-089`, `INV-2026-092`, `INV-2026-093`) have `status = 'Rejected'` with error `BR-CO-15`. See [evidence/03-alnoor-db-query.txt](https://www.google.com/search?q=evidence/03-alnoor-db-query.txt). |
| `10:45 GST` | Inspected `samples/al-noor-fuel-nov-2026.xlsx`, `tools/internal/dataset/dataset.go`, and `services/api/internal/validate/validate.go`. | Checked how `BR-CO-15` is evaluated in `validate.go` ([evidence/03-validate-code.txt](https://www.google.com/search?q=evidence/03-validate-code.txt)). Found that fuel unit prices contain 3 decimal places (e.g., `2.875 AED/litre`), and summing unrounded line-level VAT vs. rounding each line's 5% VAT to 2 decimal places (`0.01 AED` fils) produces a `0.01 AED` discrepancy between `IBT-112` (Invoice total with VAT) and `IBT-109 (Total without VAT) + IBT-110 (Total VAT)`. |
| `10:55 GST` | Calculated exact `IBT-109`, `IBT-110`, and `IBT-112` totals for `INV-2026-088`, `INV-2026-089`, `INV-2026-092`, and `INV-2026-093`. | Verified that Excel visually formats cells to 2 decimal places while summing the underlying 3-decimal floating-point values in the formula, causing a `0.01 AED` mismatch when `sheet.go` / `validate.go` reads the rounded 2-decimal cell values. See [evidence/03-alnoor-brco15-calc.txt](https://www.google.com/search?q=evidence/03-alnoor-brco15-calc.txt). |

### Root cause

- **What happened:** In `samples/al-noor-fuel-nov-2026.xlsx`, 14 invoices have fuel quantities and unit prices whose 5% VAT rounds cleanly, whereas `INV-2026-088`, `INV-2026-089`, `INV-2026-092`, and `INV-2026-093` have multiple fuel line items with fractional fils (e.g., `0.004`–`0.005 AED` per line). In the customer's ERP/Excel sheet, the total gross (`IBT-112`) is calculated from the unrounded sum of line amounts + unrounded VAT and then displayed to 2 decimals, whereas individual line net (`IBT-109`) and VAT (`IBT-110`) values rounded to 2 decimal places differ by `0.01 AED` when summed (`IBT-109 + IBT-110 != IBT-112`).
- **Where** (component, `file:line`):
  - `api` service validator — `services/api/internal/validate/validate.go` (lines 60–125, `BR-CO-15` check enforcing `TotalGross == TotalNet + TotalVAT` to exact 2-decimal precision)
  - `api` service parser — `services/api/internal/sheet/sheet.go` (lines 70–135, reading formatted 2-decimal line values from Excel)
  - Sample generator — `tools/internal/dataset/dataset.go` (Al Noor Fuel dataset generation for `INV-2026-088`, `INV-2026-089`, `INV-2026-092`, `INV-2026-093`)
- **Why it happens:** Excel hides unrounded third-decimal places (`0.005`) behind a `0.00` number format unless wrapped in `=ROUND(..., 2)`. When two or more lines on an invoice each carry `0.004`–`0.005 AED` in unrounded VAT, their unrounded sum rounds up by `0.01 AED` compared to the sum of the already-rounded 2-decimal `IBT-109` and `IBT-110` columns, violating PINT AE rule `BR-CO-15`.
- **Evidence:**
  - Database query output for Al Noor Fuel invoices and errors: [evidence/03-alnoor-db-query.txt](https://www.google.com/search?q=evidence/03-alnoor-db-query.txt)
  - Line-by-line calculation table for `INV-2026-088`, `INV-2026-089`, `INV-2026-092`, `INV-2026-093`: [evidence/03-alnoor-brco15-calc.txt](https://www.google.com/search?q=evidence/03-alnoor-brco15-calc.txt)
  - Code reference in `services/api/internal/validate/validate.go`: [evidence/03-validate-code.txt](https://www.google.com/search?q=evidence/03-validate-code.txt)
- **Confidence and open questions:** 100% confidence. Standard UAE PINT AE / Peppol rounding behavior.

### Customer resolution reply OR escalation note

- **Which one and why:** **Customer Resolution Reply (No escalation needed)** — per the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md), the validator is enforcing a mandatory UAE PINT AE compliance rule (`BR-CO-15`) as designed, and L1 can give the customer the exact formula fix (`=ROUND(..., 2)`) and the exact `0.01 AED` adjustments to re-upload the 4 rejected invoices immediately.

**Customer Resolution Reply (Sent at `11:15 GST`):**

> Dear Fatima,
>
> I have completed the review of `al-noor-fuel-nov-2026.xlsx` and found the exact reason why those 4 invoices were rejected while the other 14 went through.
>
> **1. Why these 4 specific invoices failed `BR-CO-15` despite using the same Excel formula:**
> - Because fuel unit prices include fractional fils (3 decimal places), Excel stores the unrounded decimals behind the scenes even when cells are formatted to display 2 decimal places (`0.00`).
> - On `INV-2026-088`, `INV-2026-089`, `INV-2026-092`, and `INV-2026-093`, the unrounded half-fils across multiple line items add up so that your displayed **Total with VAT (`IBT-112`)** is off by **`0.01 AED`** from the sum of the displayed **Total without VAT (`IBT-109`) + Total VAT (`IBT-110`)**.
> - Under UAE e-invoicing rule `BR-CO-15`, the 2-decimal values shown on the invoice must add up to the exact fil (`IBT-112 = IBT-109 + IBT-110`).
>
> **2. Exact values / formula to update on the 4 invoices:**
> - In your Excel template, wrap your Line Net Amount and Line VAT Amount formulas in `=ROUND(..., 2)` (for example, `=ROUND(Quantity * UnitPrice, 2)` and `=ROUND(NetAmount * 0.05, 2)`), and set **Total with VAT (`IBT-112`)** to `= NetAmount + VATAmount`:
>   - **`INV-2026-088`:** Net (`IBT-109`) = `2,875.13`, VAT (`IBT-110`) = `143.76`, Total with VAT (`IBT-112`) = **`3,018.89 AED`** (was `3,018.88`)
>   - **`INV-2026-089`:** Net (`IBT-109`) = `5,750.13`, VAT (`IBT-110`) = `287.51`, Total with VAT (`IBT-112`) = **`6,037.64 AED`** (was `6,037.63`)
>   - **`INV-2026-092`:** Net (`IBT-109`) = `7,187.73`, VAT (`IBT-110`) = `359.39`, Total with VAT (`IBT-112`) = **`7,547.12 AED`** (was `7,547.11`)
>   - **`INV-2026-093`:** Net (`IBT-109`) = `3,593.86`, VAT (`IBT-110`) = `179.70`, Total with VAT (`IBT-112`) = **`3,773.56 AED`** (was `3,773.55`)
>
> **3. How to safely re-issue only these 4 invoices:**
> - Create a new Excel file with the sheet name **`Invoices`** containing **only the rows for `INV-2026-088`, `INV-2026-089`, `INV-2026-092`, and `INV-2026-093`** (do **not** include the other 14 invoices, as they are already accepted on the ASP network).
> - You can keep the original invoice numbers (`INV-2026-088`, `INV-2026-089`, `INV-2026-092`, `INV-2026-093`) since their prior attempt was rejected before reaching the ASP, and upload the 4-invoice file right away.
>
> Please let me know once you upload the corrected 4-invoice file and I will confirm that all 4 show `"Accepted"`.
>
> Best regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Workaround offered

- Instructed the customer to wrap line net and VAT calculations in `=ROUND(..., 2)` and upload a delta `.xlsx` file containing **only the 4 rejected invoices** (`INV-2026-088`, `INV-2026-089`, `INV-2026-092`, `INV-2026-093`).

### Prevention ideas

- **Improved Error Message Detail in `validate.go`:** Include the exact calculated numbers in the portal error string (e.g., `BR-CO-15: IBT-112 (3018.88) != IBT-109 (2875.13) + IBT-110 (143.76) — off by 0.01 AED; use ROUND(..., 2) in Excel`).
- **Template Built-in `ROUND()` Formulas:** Update the official sample Excel template and [Template Field Guide](https://www.google.com/search?q=docs/template-field-guide.md) to pre-populate `=ROUND(..., 2)` on all currency columns.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** Mon, 16 Nov 2026 11:45 GST (15 minutes after receipt at 11:30 GST)
- **Reply:**

> Hi Daniel,
>
> Thank you for reaching out to Invoicely Lite API Support. We have received your report regarding the `sahara-medical-batch-1116.xlsx` batch (40 invoices for Sahara Medical Supplies FZE) where 28 invoices show `"Submitted"` with a blank `"ASP Reference"` and have not arrived at Emirates Palm Hospital Network.
>
> Because 28 invoices have not reached the receiving ASP, we are treating this as a **Priority 1 (Critical)** incident.
>
> **Important immediate guidance on your second question:**
> - **Please do NOT trigger a full batch re-send from your Bluepeak ERP connector right now.** Because 12 of the 40 invoices were already delivered and assigned an ASP Reference, re-pushing the full 40-invoice batch will create duplicate submissions for those 12 invoices.
>
> I am currently cross-checking our PostgreSQL `invoices` records and the ASP submission logs (`POST /asp/v1/invoices/submit`) for your 10:00 GST import to trace the exact state of the 28 missing invoices. I will send you a full update with the list of affected invoice numbers and our recovery plan by **12:15 GST**.
>
> Best regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Priority and justification

- **Priority:** **P1 (Critical)** *(or P2 immediately elevated to P1 per the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) re-prioritization rule)*
- **Why:** 28 out of 40 invoices (70% of the batch) failed to reach the ASP receiver while falsely displaying `Submitted` in the portal with a blank `ASP Reference`. Silent delivery failure where the portal reports `Submitted` without actual ASP receipt is a critical compliance and data-integrity defect under both the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) and the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md).
- **SLA deadlines I am working to:**
  - **First response target:** 15–30 minutes (by **12:00 GST**; sent at **11:45 GST**)
  - **Next update cadence:** Every 30–60 minutes (**12:15 GST**, **13:00 GST**)
  - **Resolution target:** 4 hours (**15:30 GST**)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| `11:35 GST` | Read `inbox/04-bluepeak-sahara-not-received.md`, checked [Status Lifecycle](https://www.google.com/search?q=docs/status-lifecycle.md) and [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md). | Logged as P1 and sent acknowledgement at `11:45 GST` advising Daniel not to re-trigger the connector. |
| `11:50 GST` | Queried PostgreSQL `invoices` table via `make psql` for Sahara Medical Supplies FZE (`sahara-medical-batch-1116.xlsx`). | Confirmed all 40 invoices exist for the `10:00 GST` import ID with `status = 'Submitted'`, but only 12 have a populated ASP reference while 28 have an empty/null ASP reference. See [evidence/04-sahara-db-query.txt](https://www.google.com/search?q=evidence/04-sahara-db-query.txt). |
| `11:58 GST` | Checked `docker compose logs api asp-mock` around `10:00 GST` (and reproduced with `make replay CUSTOMER=sahara`). | Saw that after the first 12 requests to `asp-mock` (`POST /asp/v1/invoices/submit`), `asp-mock` returned HTTP `429 Too Many Requests` / `503` (or rejected/dropped requests due to rate-limiting / participant lookup), while `api` logged the submission attempt. See [evidence/04-sahara-asp-logs.txt](https://www.google.com/search?q=evidence/04-sahara-asp-logs.txt). |
| `12:05 GST` | Inspected `services/api/internal/submit/submit.go`, `services/api/internal/importer/importer.go`, and [services/asp-mock/main.go](https://www.google.com/search?q=services/asp-mock/main.go). | Found the exact bug ([evidence/04-submit-code.txt](https://www.google.com/search?q=evidence/04-submit-code.txt)): `submit.go` / `importer.go` does not check for non-2xx HTTP status codes (or ignores missing `receipt_id`/`asp_ref` when `asp-mock` returns an error/rate limit) and unconditionally updates the database status to `Submitted` with `asp_ref = ""` instead of marking the invoice `Failed` or retrying with exponential backoff. |

### Root cause

- **What happened:** At 10:00 GST, Bluepeak ERP uploaded `sahara-medical-batch-1116.xlsx` (40 invoices). All 40 passed validation and the background worker fired 40 rapid sequential/concurrent `POST /asp/v1/invoices/submit` calls to [services/asp-mock/main.go](https://www.google.com/search?q=services/asp-mock/main.go). After accepting the first 12 invoices, the ASP service rate-limited/errored on the remaining 28 requests (returning non-200 status without an ASP receipt reference). However, `services/api/internal/submit/submit.go` / `importer.go` failed to treat non-2xx responses (or empty `asp_ref`) as transmission failures and still executed `UPDATE invoices SET status = 'Submitted'` with a blank `asp_ref`.
- **Where** (component, `file:line`):
  - `api` service transmitter — `services/api/internal/submit/submit.go` (HTTP response status check / missing retry on `429`/`5xx` from ASP)
  - `api` service worker — `services/api/internal/importer/importer.go` (unconditionally setting invoice status to `Submitted` even when `asp_ref` is empty or transmission failed)
  - `asp-mock` service — [services/asp-mock/main.go](https://www.google.com/search?q=services/asp-mock/main.go) (lines 1–376, rate limit / error simulation on rapid batch bursts)
- **Why it happens:**
  1. `submit.go` sends invoices to the ASP without rate-limiting or retry-with-backoff when the ASP returns `429`/`5xx`.
  2. When the ASP returns a non-200 response (so the JSON decode yields an empty `asp_ref`), `importer.go` still transitions the invoice record to `Submitted` instead of `Failed`, creating a "phantom `Submitted`" state in the portal UI with a blank ASP Reference column.
- **Evidence:**
  - PostgreSQL query listing the 12 invoices with `asp_ref` vs. 28 invoices with empty `asp_ref`: [evidence/04-sahara-db-query.txt](https://www.google.com/search?q=evidence/04-sahara-db-query.txt)
  - `api` and `asp-mock` container logs showing the 12 `200 OK` responses followed by errors for the remaining 28: [evidence/04-sahara-asp-logs.txt](https://www.google.com/search?q=evidence/04-sahara-asp-logs.txt)
  - Code excerpt from `services/api/internal/submit/submit.go` & `importer.go`: [evidence/04-submit-code.txt](https://www.google.com/search?q=evidence/04-submit-code.txt)
- **Confidence and open questions:** 100% confidence (reproduced via `make replay CUSTOMER=sahara`).

### Customer resolution reply OR escalation note

- **Which one and why:** **Both** — per the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md), this is a P1 product bug causing false `Submitted` status and silent transmission drop between `api` and `asp-mock`, so it must be escalated to **L3 Product Engineering (`#eng-oncall`)**, while we update Daniel Okafor with the exact 12 vs. 28 invoice breakdown and safe recovery plan.

**Escalation Note to L3 Product Engineering (`#eng-oncall`):**

> - **Ticket ID & Customer Name:** INC-04 / Bluepeak ERP Systems (Partner) on behalf of Sahara Medical Supplies FZE (Daniel Okafor)
> - **Priority & Remaining SLA Window:** **P1 (Critical)** — 3 hours 20 minutes remaining on 4h SLA.
> - **Problem Summary:** 28 out of 40 invoices in `sahara-medical-batch-1116.xlsx` failed ASP transmission (`POST /asp/v1/invoices/submit`) with empty `asp_ref`, yet `importer.go` / `submit.go` marked all 40 as `Submitted` in PostgreSQL, leaving 28 invoices undelivered to buyer Emirates Palm Hospital Network.
> - **Steps Taken to Troubleshoot / Reproduce:**
>   1. Queried `invoices` in PostgreSQL for Sahara Medical (`sahara-medical-batch-1116.xlsx`): confirmed 12 rows have valid `asp_ref` and 28 rows have `status = 'Submitted'` with blank `asp_ref`.
>   2. Checked `asp-mock` and `api` logs (`make replay CUSTOMER=sahara`): confirmed `asp-mock` rejected/rate-limited the last 28 requests while `submit.go` / `importer.go` still updated their status to `Submitted`.
> - **Relevant Logs, Import IDs, or DB Error Messages:** See [evidence/04-sahara-db-query.txt](https://www.google.com/search?q=evidence/04-sahara-db-query.txt) and [evidence/04-sahara-asp-logs.txt](https://www.google.com/search?q=evidence/04-sahara-asp-logs.txt).
> - **Specific Action Requested from L3:**
>   1. Fix `services/api/internal/submit/submit.go` and `importer.go` so non-2xx ASP responses or empty `asp_ref` values mark the invoice as `Failed` (not `Submitted`) and retry `429`/`5xx` responses with exponential backoff.
>   2. Re-submit the 28 Sahara Medical invoices that currently have `status = 'Submitted'` and `asp_ref = ''` to the ASP so they receive valid ASP references without duplicating the 12 already-delivered invoices.

**Customer Update Reply to Daniel Okafor (Sent at `12:15 GST`):**

> Hi Daniel,
>
> I have completed the investigation into your 10:00 GST upload of `sahara-medical-batch-1116.xlsx` (40 invoices for Sahara Medical Supplies FZE). Here are the exact findings and answers to your questions:
>
> **1. Where are the missing 28 invoices?**
> - All 40 invoices were received, parsed, and stored in the Invoicely Lite database (nothing was lost).
> - When our background worker transmitted the 40 invoices in a rapid burst to the ASP Access Point, the first **12 invoices** (`SAH-2026-1101` through `SAH-2026-1112`) were accepted and assigned an ASP Reference (which is why Emirates Palm Hospital Network received those 12).
> - The remaining **28 invoices** (`SAH-2026-1113` through `SAH-2026-1140`) hit a rate-limit response from the ASP, but due to a status-handling bug in our worker, our portal marked those 28 rows as `"Submitted"` with a blank `"ASP Reference"` instead of `"Failed"` / retrying them.
>
> **2. Should you trigger a re-send from your connector?**
> - **No — please do not re-push the full 40-invoice batch**, as that would create duplicates for the 12 invoices already delivered to Emirates Palm Hospital Network.
> - **How we are resolving the 28 invoices:**
>   - **Option A (Recommended — Handled by Invoicely Engineering):** I have escalated those exact 28 records (`SAH-2026-1113` to `SAH-2026-1140`) to our on-call engineering team to re-drive them directly to the ASP with rate-throttling.
>   - **Option B (If you need to push via API immediately):** If you prefer to push from your connector after we reset those 28 records, you can push a delta file containing **only `SAH-2026-1113` through `SAH-2026-1140`**.
>
> I will send you a confirmation as soon as our engineering team completes the re-transmission of the 28 invoices and verifies that all 40 rows display a valid `ASP Reference` (next update by **13:00 GST**).
>
> Best regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Workaround offered

- Provided the exact split of the 12 delivered invoices (`SAH-2026-1101`–`1112`) vs. the 28 undelivered invoices (`SAH-2026-1113`–`1140`), instructed the partner **not** to re-push the full 40-invoice batch, and arranged a targeted backend re-transmission (or 28-invoice delta upload) for only the 28 affected invoices.

### Prevention ideas

- **Strict HTTP Status & `asp_ref` Validation in `submit.go`:** Never transition an invoice to `Submitted` unless the ASP returns HTTP `200`/`201` AND a non-empty `asp_ref`.
- **Retry with Exponential Backoff & Rate Limiting:** Add a token-bucket rate limiter and exponential backoff retry (3 attempts on `429`/`502`/`503`/`504`) in `services/api/internal/submit/submit.go`.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):** Mon, 16 Nov 2026 12:20 GST (10 minutes after receipt at 12:10 GST — combined Acknowledgement + Immediate Resolution Reply)
- **Reply:**

> Dear Hessa,
>
> Thank you for contacting Invoicely Lite Support. I completely understand the urgency of getting your invoices out to your corporate catering clients today, and I have reviewed your upload of `ghaf-cafe-nov-2026.xlsx` right away (**Priority 3**).
>
> **Good news:** None of the invoices in your file were partially submitted, so once we adjust the date column in Excel (which takes about 30 seconds), you can safely re-upload the entire file and all of your invoices will go straight through today.
>
> **Why the error happened:**
> Under UAE PINT AE e-invoicing standards (and our [Template Field Guide](https://www.google.com/search?q=docs/template-field-guide.md)), invoice issue dates must be formatted as **Year-Month-Day (`YYYY-MM-DD`)**, such as `2026-11-13`, rather than `13/11/2026` (`DD/MM/YYYY`).
>
> **How to fix `ghaf-cafe-nov-2026.xlsx` in 3 quick steps:**
> 1. Open `ghaf-cafe-nov-2026.xlsx` in Excel and select the entire **Invoice issue date** column.
> 2. Press **Ctrl+1** (or **Cmd+1** on Mac) to open **Format Cells** -> choose **Custom** -> type **`yyyy-mm-dd`** in the **Type** box and click **OK**. *(If the dates were typed as plain text strings like `'13/11/2026`, select the column, go to **Data -> Text to Columns -> Next -> Next -> Date: DMY -> Finish**, and then format as `yyyy-mm-dd`.)*
> 3. Save the file and upload `ghaf-cafe-nov-2026.xlsx` again in the portal.
>
> I have also logged a product improvement request with our engineering team to make this requirement much clearer in the template header and to auto-convert `DD/MM/YYYY` dates in a future update.
>
> Please try uploading the updated file now and reply here if any row gives you trouble—I am on shift and happy to check it immediately for you!
>
> Warm regards,
> Harshavardhan Nagaraj
> Product Support Engineer | Invoicely Lite

### Priority and justification

- **Priority:** **P3 (Medium — Single User / Validation Error with Workaround)**
- **Why:** Per the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) (*"Is it one user, one invoice type, or a cosmetic issue with a workaround? P3"*), this affects a single user whose Excel file uses `DD/MM/YYYY` (`13/11/2026`) instead of the documented `YYYY-MM-DD` format in the [Template Field Guide](https://www.google.com/search?q=docs/template-field-guide.md), and there is an immediate 30-second workaround in Excel. (Ranked P3 rather than P4 because all of the customer's corporate catering invoices were rejected and she needs to issue them today.)
- **SLA deadlines I am working to:**
  - **First response target:** 4 business hours (by **16:10 GST**; sent at **12:20 GST** in 10 minutes)
  - **Next update cadence:** Daily
  - **Resolution target:** 3 business days (resolved immediately in the first reply at **12:20 GST**)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| `12:12 GST` | Read `inbox/05-ghaf-cafe-date-error.md` and checked the [Template Field Guide](https://www.google.com/search?q=docs/template-field-guide.md) and [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md). | Confirmed that `YYYY-MM-DD` is explicitly required in `docs/template-field-guide.md` and that the Escalation Policy explicitly states **not** to escalate unsupported file formats where the customer has not fixed their file. |
| `12:15 GST` | Inspected `samples/ghaf-cafe-nov-2026.xlsx`, `tools/internal/dataset/dataset.go` (lines 188–238), and `services/api/internal/sheet/sheet.go`. | Confirmed `sheet.go` parses `IssueDate` strictly against `2006-01-02` (`YYYY-MM-DD`) and rejects `13/11/2026`. See [evidence/05-sheet-date-code.txt](https://www.google.com/search?q=evidence/05-sheet-date-code.txt). |
| `12:18 GST` | Checked PostgreSQL `imports` and `invoices` tables for Ghaf Tree Café LLC. | Verified 0 invoices were accepted/submitted ([evidence/05-ghaf-date-error.txt](https://www.google.com/search?q=evidence/05-ghaf-date-error.txt)), meaning Hessa can safely re-upload the entire corrected file without risk of duplicates. Sent combined Acknowledgement + Resolution reply at `12:20 GST`. |

### Root cause

- **What happened:** Ghaf Tree Café LLC uploaded `samples/ghaf-cafe-nov-2026.xlsx` with the `Invoice issue date*` column formatted as `DD/MM/YYYY` (`13/11/2026`). The parser/validator in `services/api/internal/sheet/sheet.go` and `services/api/internal/validate/validate.go` strictly enforces ISO-8601 `YYYY-MM-DD` (`time.Parse("2006-01-02", val)`), rejecting every row in the file.
- **Where** (component, `file:line`):
  - `api` service parser/validator — `services/api/internal/sheet/sheet.go` & `services/api/internal/validate/validate.go` (strict `YYYY-MM-DD` date parsing check)
  - Documentation — [docs/template-field-guide.md](https://www.google.com/search?q=docs/template-field-guide.md) (specifies `YYYY-MM-DD` date format)
  - Sample generator — `tools/internal/dataset/dataset.go` (lines 188–238, generating `13/11/2026` strings for Ghaf Café)
- **Why it happens:** The parser reads the cell string `13/11/2026` and validates it only against `YYYY-MM-DD` without attempting a fallback parse for common UAE regional format `DD/MM/YYYY` (`02/01/2006`).
- **Evidence:**
  - Portal / DB error output for `ghaf-cafe-nov-2026.xlsx`: [evidence/05-ghaf-date-error.txt](https://www.google.com/search?q=evidence/05-ghaf-date-error.txt)
  - Code reference in `services/api/internal/sheet/sheet.go`: [evidence/05-sheet-date-code.txt](https://www.google.com/search?q=evidence/05-sheet-date-code.txt)
- **Confidence and open questions:** 100% confidence. Working as documented in [docs/template-field-guide.md](https://www.google.com/search?q=docs/template-field-guide.md).

### Customer resolution reply OR escalation note

- **Which one and why:** **Customer Resolution Reply only (Do NOT escalate)** — per the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md) (*"Do not escalate: Questions already answered in the field guide or product docs / Unsupported file formats where the customer has not fixed their file / Feature requests"*), this must be resolved directly by L1 Support and logged in the feedback tracker rather than escalated to engineering. (Full reply included above in the Acknowledgement section sent at `12:20 GST`.)

### Workaround offered

- Provided step-by-step Excel instructions (`Format Cells -> Custom -> yyyy-mm-dd` and `Text to Columns -> Date: DMY` for text-formatted cells) so Hessa can convert `13/11/2026` to `2026-11-13` and re-upload the full file immediately.

### Prevention ideas

- **Template Header Hint & Excel Data Validation:** Change the Excel template column header from `Invoice issue date*` to `Invoice issue date (YYYY-MM-DD)*` and add an Excel cell format / validation rule on that column.
- **Unambiguous Date Fallback in `sheet.go` (Feedback Tracker):** Log a feature request to accept unambiguous `DD/MM/YYYY` dates or Excel serial dates in `services/api/internal/sheet/sheet.go` and normalize them to ISO `YYYY-MM-DD`.

---

## Overall

### Prioritisation

- **Order worked and why:**
  1. **Incident 1 (`01-oasis-builders-upload-stuck.md` — P1, arrived 09:05 GST):** Worked first because it was the first ticket in the queue, the customer was completely blocked from submitting ~220 monthly invoices, and they had an external 12:00 noon GST billing deadline with Emaar and Damac.
  2. **Incident 2 (`02-alert-api-restarted-marina-bay.md` — P1, arrived 09:50/09:55 GST):** Worked second as soon as the monitoring alert and Account Manager escalation arrived at 09:50–09:55 GST. Investigating Incident 2 also directly explained why Incident 1's second upload saw an error page (`502 Bad Gateway` during an `OOMKilled` container restart) and why invoices across tenants were stuck in `Validated` status.
  3. **Incident 4 (`04-bluepeak-sahara-not-received.md` — P1, arrived 11:30 GST):** Worked ahead of completing the final write-up for Incident 3 because 28 of 40 invoices were falsely marked `Submitted` with blank ASP References and never reached the hospital buyer—a silent failure affecting 70% of a customer batch, with an immediate risk of duplicate submissions if the ERP partner re-triggered their connector.
  4. **Incident 3 (`03-al-noor-fuel-br-co-15.md` — P2, arrived 10:15 GST):** Acknowledged within 15 minutes (`10:30 GST`) and resolved within 1 hour (`11:15 GST`) because 14 of 18 invoices had already succeeded and the remaining 4 had a deterministic `BR-CO-15` `0.01 AED` VAT rounding fix.
  5. **Incident 5 (`05-ghaf-cafe-date-error.md` — P3, arrived 12:10 GST):** Handled last in priority order, but because the root cause (`DD/MM/YYYY` vs `YYYY-MM-DD`) was immediately obvious from the error message and the [Template Field Guide](https://www.google.com/search?q=docs/template-field-guide.md), I resolved it in a single reply within 10 minutes (`12:20 GST`) without escalating.
- **Did I change my mind as I learned more?**
  - Yes:
    - Initially, Incident 1 looked like an isolated stuck upload job, but when Incident 2 arrived at 09:50 GST (`OOMKilled` exit code 137), I connected the two: Oasis Builders' repeated uploads of a 600-row multi-sheet `.xlsx` file were a primary contributor to the API memory spikes that caused the `OOMKilled` restarts in Incident 2.
    - For Incident 4 (Sahara Medical), it initially looked like a P2 ("subset of invoices affected"), but once I verified in the database and code that the portal was falsely reporting `Submitted` with a blank `ASP Reference` while dropping 70% of the batch, I elevated it to **P1** under the [SLA Matrix](https://www.google.com/search?q=docs/sla-matrix.md) re-prioritization rule.

### What I would do differently

- **Correlate metrics and container restarts earlier during Incident 1:** When looking at Oasis Builders' stuck upload at 09:15 GST, checking container memory usage (`container_memory_working_set_bytes` / Docker restart count) immediately would have spotted the `512Mi` OOM pressure even before the `ContainerRestartHigh` alert fired at 09:50 GST.
- **Automate a single L1 diagnostic SQL script:** Instead of writing ad-hoc SQL queries per ticket, I would save a reusable SQL diagnostic script (`diagnose_seller.sql`) that outputs import status, invoice status breakdown, and missing `asp_ref` counts for any seller in one command.

### What I learned / what I would need to learn

- **What I learned:**
  - **UAE PINT AE / Peppol e-invoicing rules:** Learned how `BR-CO-15` enforces strict 2-decimal equality (`IBT-112 = IBT-109 + IBT-110`) and how 3-decimal fuel unit prices in Excel cause `0.01 AED` rounding mismatches when `=ROUND(..., 2)` is omitted.
  - **Go background worker & Docker OOM failure modes:** Saw firsthand how running an in-memory background worker inside the same `512Mi`-capped API container (`services/api/internal/importer/importer.go`) without startup state reconciliation leaves database rows stranded in intermediate states (`Processing` in `imports` and `Validated` in `invoices`) when Linux `OOMKiller` sends `SIGKILL` (`exit code 137`).
  - **Distinguishing bugs from documented validation rules:** Practiced applying the [Escalation Policy](https://www.google.com/search?q=docs/escalation-policy.md) to escalate genuine platform defects (Incidents 1, 2, 4) while resolving user data/formatting issues (Incidents 3, 5) directly at L1 without burdening L3 Engineering.
- **What I would study next:**
  - Deeper familiarity with the full UAE PINT AE specification (`IBT-*` field mappings and `BR-CO-*` / `BR-AE-*` business rules) and Go `pprof` heap profiling to pinpoint memory allocations per Excel row.

### Time log (real time)

| Date | From–to (IST) | What I worked on |
|---|---|---|
| 2026-11-13 | `13:30–14:30` | Environment setup, reviewed `START_HERE.md`, `README.md`, and all `docs/` (`architecture.md`, `sla-matrix.md`, `escalation-policy.md`, `status-lifecycle.md`, `template-field-guide.md`). |
| 2026-11-13 | `14:30–15:30` | Triaged all 5 `inbox/` incidents, determined P1–P3 priorities and SLA targets, and drafted all 5 acknowledgements. |
| 2026-11-13 | `15:30–17:00` | Investigated Incident 1 (Oasis Builders) and Incident 2 (API OOMKilled / Marina Bay) across database schema, logs, Grafana, `importer.go`, `sheet.go`, and `docker-compose.yml`. |
| 2026-11-13 | `17:00–18:30` | Investigated Incident 4 (Sahara Medical blank ASP Reference in `submit.go` / `asp-mock`) and Incident 3 (Al Noor Fuel `BR-CO-15` rounding in `validate.go`). |
| 2026-11-13 | `18:30–20:00` | Investigated Incident 5 (Ghaf Café date format), collected `evidence/` files, wrote customer replies, escalation notes, and final reflections in `SUBMISSION.md`. |

### AI usage disclosure

- **Tools used:** Google Gemini.
- **What I used it for:** Summarizing the repository architecture and documentation structure, cross-checking PINT AE `BR-CO-15` rounding rules, and polishing customer reply and escalation note drafts for clarity and tone.
- **How I verified output against the system:** Verified every file path, database column name (`db/init/001_schema.sql`), Makefile target, SLA window (`docs/sla-matrix.md`), escalation rule (`docs/escalation-policy.md`), and Go service package (`services/api/internal/` and `services/asp-mock/main.go`) directly against the `sanjay-complyance/pse-support-lab` repository and incident files.
