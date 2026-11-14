# Evidence: sample workbook inspection

Static inspection of the supplied XLSX files with Python/openpyxl; this is not runtime database or Grafana evidence.

## Oasis Builders
- Workbook: `samples/oasis-builders-nov-2026.xlsx`
- `Invoices` sheet: 600 non-empty data rows, 32 columns, 8 distinct invoice numbers (`OBS-2611-001` through `OBS-2611-008`), 75 rows per invoice.
- This high row count matters because the validator performs a buyer participant lookup for every row (see `services/api/internal/validate/validate.go`, around line 166); the mock directory sleeps 200 ms per lookup (`services/asp-mock/main.go`, around line 335). Serial lookup cost could be about 120 seconds before other processing, if all rows require lookups.
- This is a code-and-workbook-derived estimate, not a measured import duration.

## Al Noor Fuel
- Workbook: `samples/al-noor-fuel-nov-2026.xlsx`
- 13 data rows, 6 invoices. Three invoices have a 1–2 fils total mismatch when the PINT mapper's rounded line VAT and raw quantity × price gross calculation are compared:
  - `ANF-2611-0042`: exclusive 14,307.08 + VAT 715.35 = 15,022.43; inclusive computed by mapper = 15,022.42 (−0.01).
  - `ANF-2611-0044`: exclusive 11,254.84 + VAT 562.75 = 11,817.59; inclusive = 11,817.57 (−0.02).
  - `ANF-2611-0046`: exclusive 5,343.80 + VAT 267.20 = 5,611.00; inclusive = 5,610.99 (−0.01).
- The other three invoices balance to two decimals.
- This reproduces the BR-CO-15 arithmetic discrepancy from the sample data and mapper logic.

## Bluepeak / Sahara Medical
- Workbook: `samples/sahara-medical-batch-1116.xlsx`
- 60 data rows, 40 distinct invoice numbers (`SMS-1116-001` onward).
- ASP mock rate limiter allows an initial burst of 8 documents per sender and refills at 2 documents/second (`services/asp-mock/main.go`, lines 32–38, 137–151).
- API submission worker pool is configured with 4 workers (`services/api/main.go`, submitter construction); `submit.go` handles 200, 422 and selected 5xx responses but has no 429 branch. A 429 therefore leaves the record in `Submitted` after it was set there, with no ASP message ID.
- This matches the reported symptoms (Submitted, blank ASP reference, some received, most not) and is a strong code-level explanation. Confirm actual per-invoice 429 responses from the runtime logs/DB before calling it conclusively proven for this specific batch.

## Ghaf Tree Café
- Workbook: `samples/ghaf-cafe-nov-2026.xlsx`
- 8 data rows, 4 distinct invoices (`GTC-118` through `GTC-121`).
- Both issue date and due date are entered as `13/11/2026` and `13/12/2026` in the workbook; parser accepts ISO `YYYY-MM-DD` or Excel numeric serial dates, not `DD/MM/YYYY` (`sheet.go`, around lines 167–184).
- The portal validator applies this parser to both date columns (`validate.go`, around lines 112–117). The reported error is consistent with the sample.

## API restart alert
- Supplied earlier live checks (reported in chat): API container `Restarts=2`, `OOMKilled=false`, `ExitCode=0`, current status running; no `error|fatal|panic|restart|shutdown|signal|killed` matches in the sampled API logs; API memory snapshot about 90.33 MiB / 128 MiB. A single current memory snapshot does not prove historical memory pressure.
- `curl http://localhost:8080/health` returned 404; this was the wrong route. The code registers `/healthz` (`httpapi.go`), so the 404 is not evidence that the API was unhealthy.
- Restart cause remains undetermined. Need Docker events, healthcheck history, timestamped logs and platform events around 09:30–09:50 exercise time. Do not label it OOM or a code crash without evidence.
