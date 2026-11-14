# Submission

- **Name:** Ragul Krishnan
- **Date submitted:** 16 November 2026 (exercise date; enter actual submission date if different)
- **Repository link:** https://github.com/Ragul-Krishnan-G/pse-support-lab
- **Total time spent (real time):** Fill in your actual elapsed time before submission.

> Customer replies below are proposed drafts on the exercise clock, not messages actually sent. Runtime findings are distinguished from static code/sample analysis. Evidence files are in `evidence/`.

- **Items I fully investigated, and why I chose them:**
  - Incident 2 (API restarts): highest potential customer impact and possible P1.
  - Incident 4 (Bluepeak/Sahara): 40-invoice integration batch with possible missing ASP acknowledgements and duplicate risk.
  - Incident 3 (Al Noor): financial/VAT compliance impact; sample data allows arithmetic reproduction.
  - Incident 1 (Oasis): month-end urgency, large workbook, and duplicate-on-retry risk.
  - Incident 5 is triaged with a clear sample/code explanation and a customer workaround; further live import verification is not included.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 09:06
- **Reply (draft):**

> Hi Rashid,
>
> I understand the November upload has remained in Processing, you have retried it, and some invoice numbers are now reported as already existing. Please do not upload or retry the workbook again for now; that could create further duplicate attempts. I will reconcile the import batches and invoice statuses, including any ASP message references, and identify which documents were accepted before advising a safe retry. I will update you by 12:00 UAE time, or sooner if I confirm the affected invoice list.

### Priority and justification

- **Priority:** P2.
- **Why:** A large batch is delayed and retries are producing duplicate-number validation errors. Month-end close increases business impact. There is not enough evidence that all invoice submission to the ASP is unavailable, so P1 is not justified on the current facts.
- **SLA deadlines I am working to:** Received 09:05; first meaningful response due by 10:05 (P2, 1 hour); next update proposed by 12:00 (within the 2–4 hour cadence); resolution target 17:05 (8 hours, subject to business-hours clock).

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:05–09:15 | Read customer report and inspected supplied workbook structure. | 600 non-empty rows grouped into 8 invoice numbers, 75 rows per invoice. This is materially larger than the small sample batches. |
| 09:15–09:25 | Traced import and validation flow in `services/api/internal/importer/importer.go`, `sheet/sheet.go` and `validate/validate.go`. | Import processing is synchronous within a 10-minute context. Each invoice is grouped and stored before validation; valid invoices are queued for background submission. |
| 09:25–09:35 | Checked participant lookup implementation and sample workbook. | Validator performs participant lookup for each row; mock lookup sleeps 200 ms. A simple upper-bound estimate for 600 lookups is about 120 seconds, excluding other work. This is a code-derived estimate, not a measured duration. |
| 09:35–09:45 | Checked duplicate validation logic. | `AE-BR-09` rejects an invoice number already present for the same seller, excluding only the current import ID. Re-uploading a previously processed batch can therefore be rejected as duplicate even if the first attempt partially or fully progressed. |
| 09:45–09:55 | Considered whether the workbook itself is malformed. | Workbook has the expected `Invoices` sheet and invoice-number column. No workbook-format failure was established from static inspection. |

### Root cause

- **What happened:** The workbook is unusually large (600 rows). The code performs participant lookups per row, with 200 ms simulated lookup latency; this can make import processing noticeably slow. The customer's subsequent re-uploads are separate imports. The duplicate check sees invoice numbers already persisted by an earlier attempt and returns `AE-BR-09`. This explains the duplicate messages, but the exact status/outcome of each original attempt must be confirmed from the live database and ASP records.
- **Where** (component, `file:line`):
  - `services/api/internal/importer/importer.go:22,38–40` — 10-minute processing context.
  - `services/api/internal/validate/validate.go:69–80` — duplicate check against prior imports.
  - `services/api/internal/validate/validate.go:166` — participant lookup per row.
  - `services/asp-mock/main.go:40,335` — 200 ms mock participant lookup delay.
- **Why it happens:** The import pipeline validates rows sequentially and performs a remote participant lookup for each row. Duplicate detection is seller + invoice number based and intentionally excludes only the current import; it is not an idempotent replay mechanism.
- **Evidence**:
  - `evidence/sample-workbook-analysis.md` — workbook row/invoice counts and code-derived lookup estimate.
  - Code references above. No live import IDs, database query output or ASP reconciliation output were included in the supplied ZIP, so those must not be represented as checked.
- **Confidence and open questions:** High confidence in the static code paths and workbook counts; medium confidence that lookup latency materially contributed to the observed delay. Need the original and retry import IDs, invoice status history, ASP document list/message IDs and timestamped API logs to establish exactly what happened in the customer's attempts.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution/update draft; L1 can safely explain the hold and reconcile before any retry. Escalate to L2 if database reconciliation or data repair is needed.

> Hi Rashid,
>
> The workbook contains 600 detail rows across 8 invoice numbers. Our import flow performs participant checks while processing those rows, so this file can take longer than a small upload. The duplicate-number message is generated when the same seller and invoice number already exists in an earlier import; it does not by itself prove whether the earlier document reached the ASP. Please keep the file on hold. We are matching each invoice against its import history and ASP reference before recommending any retry. I will send the reconciled list and next action by 12:00 UAE time.

### Workaround offered

Do not re-upload the full file. Reconcile the 8 invoice numbers against all import attempts, invoice event history and ASP `GET /v1/documents?sender=<seller Peppol ID>`. Only retry invoices confirmed not received by the ASP and eligible for retry; obtain L2 approval if records need repair.

### Prevention ideas

- Make import/retry idempotency explicit and show existing invoice statuses before allowing another full upload.
- Cache or batch participant lookups rather than making a remote lookup for every spreadsheet row.
- Provide progress and estimated processing time for large workbooks.
- Add a reconciliation/export view showing invoice number, import ID, current status and ASP message ID.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 09:43
- **Sent to:** Priya Nair (CSM) and internal support/on-call queue.
- **Reply (draft):**

> Hi Priya,
>
> I have taken ownership of the API restart alert and the reports of intermittent 502s and invoices remaining in Validated since approximately 09:30. I am checking API health, container restart history, logs, submission queue and invoice states, and will involve Engineering immediately if the multi-customer submission impact is confirmed. I will provide the next update by 10:15 UAE time. I will keep Marina Bay's Wednesday go-live readiness separate from the incident recovery decision until service stability is verified.

### Priority and justification

- **Priority:** P1 provisional, pending confirmation that one or more customers cannot get invoices to the ASP.
- **Why:** The alert reports repeated API restarts, multiple customers report 502s, and invoices are stuck at Validated. If submissions are unavailable for one or more customers, the SLA defines this as P1. The 09:42 alert is sufficient to start urgent triage and page Engineering while confirming impact.
- **SLA deadlines I am working to:** Alert received 09:42; first response target 09:57–10:12 (15–30 minutes); next update by 10:15 and then every 30–60 minutes; resolution target 13:42 (4 hours). P1 clock is 24×7.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:42–09:50 | Reviewed Grafana alert and CSM escalation. | Alert states API restarted 3 times in 15 minutes; several customers report intermittent 502s and Validated invoices since about 09:30. |
| 09:50–10:00 | Checked container status and Docker inspect output supplied from the lab. | API container was running; restart count was 2; `OOMKilled=false`; exit code 0. |
| 10:00–10:05 | Reviewed sampled API logs and filtered for error/fatal/panic/shutdown/signal terms. | Three startup messages appeared in the sampled window; the filter returned no matching lines. This sample does not explain why the process restarted. |
| 10:05–10:10 | Checked current resource snapshot. | API memory was about 90.33 MiB of 128 MiB (70.57%) at snapshot time. This is not evidence of historical OOM. |
| 10:10 | Tested `/health` and checked API routes in source. | `/health` returned 404 because the implemented health endpoint is `/healthz`; the 404 was a wrong-path test, not proof of an outage. |
| Pending | Collect Docker events, healthcheck history, timestamped logs, Prometheus metrics and DB status counts for exercise window 09:30–09:50. | Not available in the uploaded project archive; cause remains unknown. |

### Root cause

- **What happened:** The API container restarted twice according to the supplied `docker inspect` output, while the exercise alert reports three restarts in its 15-minute window. The sampled logs and container metadata do not identify the trigger.
- **Where** (component, `file:line`): API container in `docker-compose.yml` (healthcheck and 128 MiB memory limit); API health handler at `services/api/internal/httpapi/httpapi.go` (`GET /healthz`). No specific faulty source line has been established.
- **Why it happens:** **Unknown from current evidence.** OOM is not supported by the inspected container's `OOMKilled=false`; a current memory snapshot cannot rule out earlier pressure. Exit code 0 and absent error logs also do not prove a cause.
- **Evidence:**
  - `evidence/api-restart-observations.md` — runtime observations supplied during the lab.
  - `docker-compose.yml` — API memory cap and healthcheck configuration.
  - `services/api/internal/httpapi/httpapi.go` — actual health route.
  - Required next evidence: `docker events` for the API container, `docker inspect` health state, `docker compose logs --timestamps api`, Prometheus API/5xx/queue metrics and SQL counts of invoices by status.
- **Confidence and open questions:** High confidence that the container restarted and was running at inspection time. Low confidence on cause. Need exact restart timestamps, event reason (manual stop, healthcheck, process exit, daemon action), historical memory and health status, and whether the 502s/Validated backlog affected all or only some customers.

### Customer resolution reply OR escalation note

- **Which one and why:** L3 Engineering escalation draft. P1 is provisional and a multi-customer production impact is reported; escalation should not wait for a complete RCA.

> ### Escalation: API restarts with intermittent 502s and invoices not progressing from Validated
>
> **Priority:** P1 provisional — multiple customers report 502s and invoices stalled at Validated; confirm whether no invoices are reaching the ASP.
> **Customer / seller:** Multiple customers; Marina Bay Hospitality is an upcoming go-live stakeholder. Seller IDs/import IDs not yet reconciled.
> **Reported by / at:** Grafana alert at 09:42; Priya Nair forwarded customer impact at 09:50, Monday 16 November 2026 (UTC+4).
> **Affected:** Multiple customers; invoices reportedly stuck since approximately 09:30. Exact import IDs, invoice IDs and counts pending.
>
> **Symptoms:** Intermittent 502 Bad Gateway during portal uploads; some invoices remain Validated rather than moving to Submitted/Accepted.
>
> **Evidence:**
> - Container inspect: restart count 2, running, `OOMKilled=false`, exit code 0.
> - Sampled API logs: three startup messages; no matching error/fatal/panic/shutdown/signal lines in the sampled filter.
> - API memory snapshot: 90.33 MiB / 128 MiB at inspection time; not historical evidence.
> - `/health` returned 404; source confirms health endpoint is `/healthz`.
> - Runtime observations: `evidence/api-restart-observations.md`.
>
> **Steps to reproduce:** Not yet reproduced; correlate a customer upload/502 timestamp with API container events and access logs.
>
> **Tried / ruled out:** Current container is running; inspected container metadata does not indicate OOM kill. Wrong `/health` endpoint is not a valid health test. No root cause ruled in.
>
> **Suspected cause:** Unknown. Need Engineering to correlate restart events, healthcheck failures, resource pressure and process termination.
>
> **Workaround:** Do not repeatedly retry uploads while status is uncertain. Preserve import IDs and reconcile invoice states before retrying.
>
> **Timeline:** 09:42 alert; 09:50 CSM impact report; triage observations collected thereafter.
>
> **Ask:** Please investigate API restart events and 502 window, confirm whether the submission path is available, and advise safe recovery for Validated invoices. Initial assessment requested by 10:30 UAE time.
> **Next customer update due:** 10:15 UAE time.

**Customer update draft:**

> Hi Priya,
>
> We have confirmed the API container restarted and is currently running. The available evidence does not yet identify why it restarted, so I am not attributing this to memory or a code failure without proof. Engineering is being asked to correlate the restart events with the 502 window and the invoices still in Validated. Please ask affected customers not to repeatedly re-upload the same files while we reconcile their import and invoice statuses. I will update you by 10:15 UAE time. Marina Bay go-live readiness will be confirmed only after we validate stable uploads and ASP submission end to end.

### Workaround offered

Avoid repeated uploads. Capture affected import IDs and invoice numbers; verify each invoice status and ASP message reference before retrying. If portal uploads are unavailable, preserve source workbooks and wait for API recovery rather than creating duplicate imports.

### Prevention ideas

- Alert on restart count, healthcheck failures, API 5xx rate and age/count of Validated invoices.
- Retain structured termination/restart events and correlate with container resource metrics.
- Add a documented recovery/reconciliation runbook for Validated and Failed invoices.
- Complete a Marina Bay go-live readiness test: upload, validation, ASP acceptance, message reference visibility and retry safety.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 10:16
- **Reply (draft):**

> Hi Fatima,
>
> Thank you for providing the workbook and the BR-CO-15 message. I understand the figures need to be explainable for your VAT audit. I will compare the invoice line amounts, calculated VAT and invoice-level totals for each rejected invoice and identify the exact rounding difference. Please do not manually alter accepted invoices or re-upload the rejected documents until we confirm the correction. I will provide an invoice-by-invoice explanation by 12:00 UAE time.

### Priority and justification

- **Priority:** P2.
- **Why:** A subset of six invoices is affected, with three failing the BR-CO-15 total check; the other invoices in the same file pass. This is not a total service outage, but the VAT/audit impact makes it important.
- **SLA deadlines I am working to:** Received 10:15; first response due by 11:15; next update proposed by 12:00 (within 2–4 hours); resolution target 18:15, with the P2 business-hours clock pausing at 18:00 UAE time if unresolved.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 10:15–10:25 | Inspected attached workbook and grouped rows by invoice number. | 13 data rows form 6 invoices. |
| 10:25–10:40 | Reproduced PINT total calculation from `services/api/internal/pint/pint.go`. | Mapper rounds each line VAT to 2 decimals, sums rounded VAT, but calculates inclusive total from raw quantity × unit price × (1 + rate). |
| 10:40–10:50 | Compared totals for each invoice to cents. | Three invoices have a BR-CO-15 mismatch: `ANF-2611-0042` by AED 0.01, `ANF-2611-0044` by AED 0.02, and `ANF-2611-0046` by AED 0.01. Three other invoices balance. |
| 10:50–11:00 | Compared source values with customer's description. | The mismatch arises from rounding differences between line net amounts, per-line VAT and gross total calculation; it is not evidence that dispatch quantities/prices are wrong. |

### Root cause

- **What happened:** Three invoices fail the BR-CO-15 arithmetic check by 0.01–0.02 AED. The mapper computes VAT by rounding each line's `net amount × rate`, but computes inclusive total using the unrounded `quantity × unit price × (1 + rate)`. When quantity × unit price has fractional fils, the two rounding paths can differ.
- **Where** (component, `file:line`): `services/api/internal/pint/pint.go:155–184` (line VAT and `gross`/`TaxInclusiveAmount` calculation); ASP rule in `services/asp-mock/main.go` around lines 190–230 (BR-CO-15 check).
- **Why it happens:** Mixed rounding bases are used for the same invoice totals. The final inclusive amount should reconcile to the exclusive amount plus the rounded VAT amount.
- **Evidence:**
  - `evidence/sample-workbook-analysis.md` contains the per-invoice amounts and differences.
  - Reproduced calculations:
    - `ANF-2611-0042`: 14,307.08 + 715.35 = 15,022.43; mapped inclusive = 15,022.42.
    - `ANF-2611-0044`: 11,254.84 + 562.75 = 11,817.59; mapped inclusive = 11,817.57.
    - `ANF-2611-0046`: 5,343.80 + 267.20 = 5,611.00; mapped inclusive = 5,610.99.
  - The other three invoice totals balance to 2 decimal places.
- **Confidence and open questions:** High confidence that the supplied workbook and mapper logic reproduce the mismatch. Confirm the exact portal/ASP response for each invoice from live import records before sending a final audit statement. Finance should approve any correction to source values; support should not silently adjust tax figures.

### Customer resolution reply OR escalation note

- **Which one and why:** L3 Engineering escalation draft because the discrepancy is generated by the product's payload-mapping calculation, not simply a user date/format issue.

> ### Escalation: PINT mapper creates BR-CO-15 total mismatch for Al Noor invoices
>
> **Priority:** P2 — three of six invoices in the supplied batch fail a VAT total consistency rule; customer is under external VAT audit review.
> **Customer / seller:** Al Noor Fuel & Lubricants LLC; seller ID 4 in seed data; Peppol ID `0235:1005362841`.
> **Reported by / at:** Fatima Khoury, email, 10:15 UAE time, 16 November 2026.
> **Affected:** Sample invoices `ANF-2611-0042`, `ANF-2611-0044`, `ANF-2611-0046`; three of six.
>
> **Symptoms:** ASP reports BR-CO-15: inclusive amount must equal exclusive amount plus VAT.
>
> **Evidence:** Sample workbook calculation in `evidence/sample-workbook-analysis.md`. Mapper rounds line VAT and separately computes gross from quantity × unit price. Differences are AED 0.01, 0.02 and 0.01 respectively. Source: `services/api/internal/pint/pint.go:155–184`.
>
> **Steps to reproduce:** Import supplied `samples/al-noor-fuel-nov-2026.xlsx`; compare generated PINT totals for the three invoice numbers above.
>
> **Tried / ruled out:** Workbook has 6 invoices; three reconcile, three have small rounding differences. No evidence supports changing dispatch quantities or prices.
>
> **Suspected cause:** High confidence: inconsistent rounding basis in `pint.Map` / `computeTotals`.
>
> **Workaround:** No manual tax adjustment recommended. Hold rejected invoices pending finance approval and engineering guidance.
>
> **Ask:** Engineering to make invoice inclusive total reconcile to rounded exclusive + rounded VAT, add regression tests for fractional unit prices, and advise whether the three invoices can be safely reprocessed. Initial guidance requested by 14:00 UAE time.
> **Next customer update due:** 12:15 UAE time.

### Workaround offered

Do not alter quantities or prices to force acceptance. Provide the customer with the three invoice numbers and calculated differences; hold only the affected invoices pending Finance/Engineering-approved correction. Preserve accepted invoices unchanged.

### Prevention ideas

- Use one consistent money-rounding strategy for line net, tax subtotal, VAT total and inclusive total.
- Add regression tests with fractional unit prices and multi-line invoices.
- Validate `inclusive = exclusive + VAT` before sending to the ASP and show a precise portal error if the source data itself does not reconcile.
- Include a calculation breakdown in invoice detail for auditability.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 11:31
- **Reply (draft):**

> Hi Daniel,
>
> Thanks for holding the resend. An HTTP 201 from our import endpoint confirms the batch was accepted for processing; it does not by itself confirm that every invoice was accepted by the ASP. I will reconcile all 40 invoice numbers against their submission responses and ASP message references. Please do not resend the batch until that reconciliation is complete, to avoid duplicate delivery. I will update you by 14:00 UAE time with the accepted, rejected and unresolved invoice lists.

### Priority and justification

- **Priority:** P2.
- **Why:** 40 invoices were submitted to the import endpoint, but the buyer received only some and many portal ASP references are blank. This is a major subset delivery degradation and carries payment-run impact. No evidence yet proves that no customers can submit at all, so P1 is not currently justified.
- **SLA deadlines I am working to:** Received 11:30; first response due by 12:30; next update proposed by 14:00; resolution target 19:30, with the P2 clock pausing at 18:00 UAE time.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 11:30–11:40 | Inspected customer report and supplied batch workbook. | 60 rows grouped into 40 distinct invoices. |
| 11:40–11:50 | Traced API submission worker and response handling. | Four workers are started. Invoice is changed to Submitted before the ASP request. |
| 11:50–12:00 | Checked ASP mock fair-use policy and response handling. | ASP allows an initial burst of 8 documents per sender and refills at 2/sec. It returns HTTP 429 with Retry-After when exceeded. |
| 12:00–12:10 | Compared ASP response handling in submitter. | Code handles 200, 422 and selected 5xx responses, but has no 429 branch. A 429 falls through without changing the invoice from Submitted or storing a message ID. This matches the blank ASP reference symptom. |
| Pending | Reconcile all 40 invoice IDs/statuses against ASP `GET /v1/documents?sender=0235:1006453719`, database `asp_response` and logs. | Runtime batch-specific response evidence not present in the ZIP; do not resend until complete. |

### Root cause

- **What happened:** The sample contains 40 invoices for one sender. The ASP mock enforces a per-sender rate limit (burst 8, refill 2/sec) and returns HTTP 429 when exceeded. The API submitter sets each invoice to Submitted before sending, but does not handle 429. Therefore a rate-limited invoice can remain Submitted with no ASP message ID/reference, even though the ASP did not accept it. This is a strong code-level explanation for the reported symptom; the actual 429 count for this specific batch must be confirmed from runtime records.
- **Where** (component, `file:line`):
  - `services/asp-mock/main.go:32–38,137–151` — rate limiter and 429 response.
  - `services/api/internal/submit/submit.go:110–159` — sets Submitted and switches only on 200, 422 and selected 5xx; no 429 handling.
  - `services/api/main.go` — submitter configured with 4 workers.
- **Why it happens:** The client does not respect `Retry-After` or transition a 429 response into a retryable state. The status is set to Submitted before response processing, and the unhandled response leaves it there.
- **Evidence:**
  - `evidence/sample-workbook-analysis.md` — 40 invoice count and ASP rate limit/code-path analysis.
  - Code references above.
  - Required live confirmation: query the 40 invoice rows for `status`, `asp_message_id`, `asp_response`; inspect API logs for `status_code=429`; query ASP document list for sender `0235:1006453719`.
- **Confidence and open questions:** High confidence in the code defect and sample rate-limit exposure. Medium confidence that 429 caused the reported production-lab batch symptom until actual batch logs/DB/ASP records are checked. Need to determine exactly which invoice numbers received an ASP message ID and which were never received.

### Customer resolution reply OR escalation note

- **Which one and why:** L3 Engineering escalation draft because the code does not handle a documented ASP rate-limit response and can leave an invoice in a misleading Submitted state.

> ### Escalation: ASP HTTP 429 is not handled; invoices remain Submitted without ASP references
>
> **Priority:** P2 — 40-invoice batch for a payment-run customer; subset delivery is missing.
> **Customer / seller:** Sahara Medical Supplies FZE, integrated by Bluepeak ERP; seller ID 5; sender Peppol ID `0235:1006453719`.
> **Reported by / at:** Daniel Okafor, email, 11:30 UAE time, 16 November 2026.
> **Affected:** 40 invoices, `SMS-1116-001` onward; exact subset pending reconciliation.
>
> **Symptoms:** Import endpoint returned HTTP 201; portal shows Submitted; many ASP references blank; buyer received only a few invoices.
>
> **Evidence:** Sample has 40 invoices. ASP mock rate limiter allows burst 8 then 2/sec (`services/asp-mock/main.go:32–38,137–151`). Submitter sets status Submitted before POST and has no HTTP 429 case (`services/api/internal/submit/submit.go:110–159`). `Retry-After` is emitted by ASP but not consumed by submitter.
>
> **Steps to reproduce:** Submit a batch exceeding the per-sender burst/rate threshold and inspect invoice status/message ID for 429 responses.
>
> **Tried / ruled out:** HTTP 201 only confirms import endpoint acceptance, not ASP acceptance. Customer connector log success does not establish per-invoice ASP receipt.
>
> **Suspected cause:** High confidence in missing 429 handling; batch-specific attribution awaits runtime confirmation.
>
> **Workaround:** Hold resend. Reconcile ASP document list and invoice records first; retry only invoices proven not received, after Engineering confirms safe handling.
>
> **Ask:** Engineering to implement bounded retry honoring `Retry-After`, record the 429 response, and transition the invoice to a retryable Failed state or equivalent. Add regression test for 40-document burst. Please confirm a safe one-time recovery plan by 14:00 UAE time.
> **Next customer update due:** 14:00 UAE time.

### Workaround offered

Do not resend all 40. Reconcile each invoice against the ASP document list and stored response. Retry only invoices confirmed absent from the ASP, after Engineering approves the recovery approach; preserve accepted message IDs to prevent duplicates.

### Prevention ideas

- Handle HTTP 429 explicitly and retry after the supplied `Retry-After`, with bounded exponential backoff and jitter.
- Store every ASP response, including 429, and use a retryable status rather than leaving it Submitted.
- Add per-sender rate-aware queueing so concurrency does not exceed ASP limits.
- Add an alert for Submitted invoices with blank ASP reference older than a threshold.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Proposed send time (exercise clock):** 12:11
- **Reply (draft):**

> Hi Hessa,
>
> I understand the date you entered means 13 November 2026. The workbook template currently expects dates in `YYYY-MM-DD` format, so please enter the issue date as `2026-11-13` and the due date as `2026-12-13`. I will confirm the corrected file passes validation before you send the invoices again. I will update you by 16:00 UAE time.

### Priority and justification

- **Priority:** P3.
- **Why:** One customer is affected by a date-format validation rule, and a clear data-entry workaround exists. The issue affects all four invoices in this customer's workbook but is not a multi-customer outage.
- **SLA deadlines I am working to:** Received 12:10; first response due by 16:10 (4 hours); daily updates if unresolved; resolution target within 3 business days.

### Investigation log

Workbook/code inspection confirms the supplied workbook uses `13/11/2026` and `13/12/2026`; the parser accepts ISO dates or Excel serial dates. The validator applies this check to both issue and due dates. This is a source-level diagnosis; no corrected re-upload was run during this review.

### Root cause

- **What happened:** The workbook uses day/month/year text dates. The importer does not parse this textual format, so the date validation rule rejects the invoices.
- **Where** (component, `file:line`): `services/api/internal/sheet/sheet.go:167–184` (`ParseDate`); `services/api/internal/validate/validate.go:112–117` (date validation).
- **Why it happens:** Only ISO `YYYY-MM-DD` strings and numeric Excel date serials are accepted. `DD/MM/YYYY` is not an accepted string format.
- **Evidence:** `evidence/sample-workbook-analysis.md` — 8 rows, 4 invoices, issue and due dates in DD/MM/YYYY.
- **Confidence and open questions:** High confidence from the workbook and code. Need a corrected sample upload to verify all four invoices pass.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution draft; product is working according to its current documented format rule, so explain the format and offer help rather than escalating as a defect.

> Hi Hessa,
>
> The dates themselves are valid; the format is the issue. Please change `13/11/2026` to `2026-11-13` and `13/12/2026` to `2026-12-13` in the workbook, keeping the same day and month. Save the file and upload the corrected copy. Please do not change invoice amounts or other fields. If you prefer, send us the corrected workbook through the approved support channel and we can help check the format before you submit it.

### Workaround offered

Convert both date columns to `YYYY-MM-DD` (`2026-11-13` and `2026-12-13`) and re-upload the corrected workbook once. Verify the invoice statuses before any further retry.

### Prevention ideas

- Add an Excel date-format example and validation note beside the date columns in the template.
- Make the error message include a concrete before/after example.
- Consider safely accepting common unambiguous date formats, with explicit locale handling, if product requirements allow.

---

## Overall

### Prioritisation

I triaged in timestamp order but elevated the API alert immediately because it could be P1 and affect multiple customers. My working priority order was: Incident 2 (provisional P1, multi-customer impact), Incident 1 (P2, month-end and duplicate risk), Incident 3 (P2, VAT/audit impact), Incident 4 (P2, 40 invoices and payment run), Incident 5 (P3, one customer with a clear workaround). Incident 4 was elevated for engineering attention after code inspection showed an unhandled 429 response. Priorities should be adjusted if live reconciliation shows a wider outage or confirms a safe workaround.

### What I would do differently

I would capture the import ID, invoice IDs and exact timestamps at the start of each investigation, then preserve a single evidence trail of API logs, ASP responses, database status history and Grafana metrics. I would avoid retrying any batch until the previous attempt is reconciled. I would also validate the exact health endpoint (`/healthz`) rather than testing an assumed `/health` path.

### What I learned / what I would need to learn

I traced the upload flow from Excel parsing through PINT mapping, portal validation, database status transitions and asynchronous ASP submission. I learned that HTTP 201 from the import endpoint is not the same as ASP acceptance; the ASP message ID is the meaningful receipt reference. I also learned to distinguish portal validation rejections from ASP rejections, and to treat 429/Retry-After as a separate retry path. I would next study Prometheus query design, Loki correlation by import/invoice ID, PostgreSQL status-history queries, Docker healthcheck/restart events, and Peppol/PINT AE rounding requirements.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| 14 Nov 2026 | Enter your actual start–end time | Environment setup, Docker/API/Grafana checks, source and sample workbook inspection, evidence preparation and submission draft. |

### AI usage disclosure

I used ChatGPT to help structure triage, explain the support workflow, inspect the supplied source files and sample workbook calculations, and draft customer acknowledgements/escalation notes. I checked code-related conclusions against the repository source, checked arithmetic against the supplied XLSX values, and kept runtime observations separate from static analysis. I did not treat unverified hypotheses as confirmed root causes. Final live database, Grafana and ASP reconciliation still needs to be performed in the lab before representing those checks as completed.
