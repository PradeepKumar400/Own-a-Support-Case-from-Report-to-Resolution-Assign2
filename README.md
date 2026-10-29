# PSE-A2-002 — Own a Support Case from Report to Resolution

This repository contains the supplied local TypeScript/Bun support lab, a bounded worker fix, regression tests, and a case report for the fictional Cedar Demo Malaysia import issue.

## Finding

The background worker was paginating through a dynamic result set that changes after each batch. `pendingBatch()` selects rows with `state = 'pending'`; `commitBatch()` changes the selected rows to `processed`. The worker was incrementing `OFFSET`, which caused the query to apply an increasing offset to a shrinking pending-only set and skip rows (skipping records 6–10 in a 12-row import with batch size 5). Additionally, `finishImport()` lacked invariant validation, allowing incomplete jobs with pending rows to be marked as `completed`.

## Fix

- **Batch selection logic:** In `src/worker.ts` and `src/repository.ts`, removed `OFFSET` pagination from `pendingBatch()`. The worker repeatedly queries the first bounded batch of pending records until the queue is drained.
- **Completion invariant check:** In `src/repository.ts`, `finishImport()` now enforces that `pending === 0`, `processed === total`, and the persisted result count matches `total` before marking an import `completed`.
- **Regression coverage:** In `tests/http.test.ts`, added end-to-end integration tests covering 12 records with default batch size 5 and isolated batch size 3, checking source/result identities, normalized values, unique IDs, and batch event bounds.
- **Cross-platform test compatibility:** In `tests/harness.ts`, updated child process termination to cleanly handle Windows signal termination.

## Setup and verification

Requirements: Bun >= 1.3 and Node.js >= 22. Run commands from `starter/`:

```sh
bun install --frozen-lockfile
bun run check-types
bun run test
```

### Lab execution & reconciliation

Start the lab server:

```sh
bun run start
```

Inspect health and baseline endpoints:

```powershell
curl.exe -sS -i http://127.0.0.1:4328/health
curl.exe -sS -i -H "x-lab-workspace: cedar-demo" http://127.0.0.1:4328/imports/case-002
curl.exe -sS -i -H "x-lab-workspace: cedar-demo" http://127.0.0.1:4328/imports/case-002/records
curl.exe -sS -i -H "x-lab-workspace: cedar-demo" http://127.0.0.1:4328/imports/case-002/results
curl.exe -sS -i -H "x-lab-workspace: cedar-demo" http://127.0.0.1:4328/imports/case-002/events
curl.exe -sS -i -H "x-lab-workspace: maple-demo" http://127.0.0.1:4328/imports/control-002
```

Reconcile the existing case using the approved lab script:

```sh
bun run resume-case
```

The running worker processes the remaining 5 pending records. Re-inspecting `case-002` confirms all 12 records processed, 0 pending, 12 genuine results, no duplicates, and the paused Maple control unchanged.

## Evidence and validation status

All evidence is captured and documented:
- `evidence/baseline-runtime-capture.md` — Real API responses capturing the pre-fix bug on `case-002` (7 processed / 5 pending) and reproduction via isolated CSV upload.
- `evidence/baseline-static-trace.md` — Detailed theoretical tracing of the offset pagination bug.
- `evidence/test-run-pre-fix.txt` — Vitest failure log demonstrating pre-fix regression failure on both 12-row tests.
- `evidence/test-run-post-fix.txt` — Vitest success log showing all 27 tests passing post-fix.
- `evidence/post-fix-verification.md` — Full API captures post-reconciliation demonstrating 12/12 processed rows and intact control.
- `evidence/test-run-status.txt` — Verification environment and test execution summary.
- `CASE_REPORT.md` — Complete case report with draft communications, technical root cause, escalation ticket draft, exit criteria, and limitations.

## Case status

Local fix implemented and fully verified via regression tests and lab database reconciliation. Production code review, staging deployment, and customer-facing production verification remain pending.

## AI assistance

OpenAI ChatGPT was utilized during initial code exploration, drafting regression test structures, and preparing preliminary markdown outlines. All code changes, test executions, API calls, runtime reproductions, and reconciliation commands were executed, inspected, and verified on local runtimes (Bun v1.4.4, Node.js v22.17.1).
