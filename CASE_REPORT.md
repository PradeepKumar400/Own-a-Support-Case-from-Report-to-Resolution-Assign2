# Case report — PSE-A2-002

## 1. Summary and impact

- **Priority / rationale:** P2 as assigned. The observed impact is partial degradation of one customer's import workflow: 12 records were submitted and accepted, but the worker halted prematurely after processing only 7 records and marked the import complete with 5 records remaining pending. This blocks downstream processing and poses data-integrity concerns. There is no evidence of a wider system outage, security incident, or statutory/tax deadline.
- **Current status:** Local causal fix and regression tests authored, executed, and fully verified. Vitest test suite passes 27 of 27 tests. The original customer case (`case-002`) has been successfully reconciled against the lab database using `bun run resume-case`, restoring all 12 genuine results with zero pending rows and zero duplicates. Production code review, release deployment, and customer-facing production verification remain pending.
- **Reported observation:** Cedar Demo Malaysia reported: *"We uploaded 12 records. The import was accepted and now says complete, but we can see only seven results. We need to continue our work. Should we upload the file again?"*
- **Verified baseline:** Real runtime inspection confirmed the reported defect on `case-002`: HTTP status 200, state `completed`, `total: 12`, `processed: 7`, `pending: 5`. Seven normalized results were stored (records 1–5 and 11–12), while records 6–10 remained pending. Events confirmed two batches committed (count 5, then count 2) followed immediately by `processing_completed`. Cross-workspace isolation and the paused Maple control (`control-002`, 3 pending, 0 processed) remained intact. Full request/response captures are in [Baseline Runtime Capture](evidence/baseline-runtime-capture.md).
- **Workaround / safety:** Advised the customer **not** to re-upload the file. Re-uploading a partial file creates a separate import without deduplication, causing duplicate records and wasted processing. For genuinely new, independent records only, the documented single-record import flow is available.

---

## 2. Draft customer communications

*All messages below are drafts for the fictional customer scenario and have not been sent.*

### Acknowledgment — simulated 26 October 2026, 09:35 GST / 13:35 MYT

Hello Cedar Demo Malaysia team,

Thank you for contacting Complyance Support. We understand that your import of 12 records indicates a completed status while only seven results are visible in your workspace, and that this is holding up your ongoing work.

Please **do not re-upload the same file** at this stage. Re-submitting the file would initiate a new import without deduplication, which risks creating duplicate records for the items already processed. 

I have taken personal ownership of this case (PSE-A2-002, Priority P2). I am currently inspecting the raw import records, persisted results, and processing event logs in our support environment to confirm whether the remaining records were skipped or failed to persist. I will provide you with our first progress update by **10:00 GST / 14:00 MYT**, even if our investigation is still underway.

Best regards,  
Alex Mercer  
Product Support Engineer, Complyance

---

### Checkpoint update — simulated 26 October 2026, 10:00 GST / 14:00 MYT

Hello Cedar Demo Malaysia team,

I am providing an update on your import case as promised.

Our preliminary log and database review confirms that 12 records (`DOC-001` through `DOC-012`) were successfully received and queued. However, the background processor processed the first seven records (records 1–5 and 11–12) and prematurely transitioned the import to completed status, leaving records `DOC-006` through `DOC-010` in a pending state without generated results.

We have isolated this behavior in our testing lab and confirmed that no data was corrupted or lost; the unrendered records remain safely stored in our database. We are actively investigating the batch processing queue logic to ensure these pending records are processed cleanly without requiring you to re-upload.

Please continue to hold off on re-uploading the file. If you have urgent, brand-new single records to process that were not part of this batch, you may submit them individually using the single-record submission workflow.

I will provide our next update with our technical reconciliation plan by **11:30 GST / 15:30 MYT**.

Best regards,  
Alex Mercer  
Product Support Engineer, Complyance

---

### Final update / conditional closure — simulated 26 October 2026, 11:30 GST / 15:30 MYT

Hello Cedar Demo Malaysia team,

We have identified the root cause of the issue and verified a permanent fix in our development and support lab. 

**What happened:**  
The background processing worker was applying pagination offsets against a dynamically shrinking list of pending records. When the initial batch of five records was completed, the subsequent query skipped the next five records and completed the job prematurely.

**Current progress and resolution plan:**  
1. We have implemented a correction to ensure the worker continually processes all pending batches until the queue is completely drained, backed by strict completion verification.
2. In our support lab, we executed a safe case reconciliation against your existing import. All five skipped records (`DOC-006` through `DOC-010`) were processed into normalized results. All 12 records are now fully processed, with original record IDs preserved and zero duplicates created.
3. An engineering ticket has been submitted for peer review and deployment into the upcoming release window.

**Next steps:**  
As soon as the deployment is applied to production, our team will safely requeue the five pending records in your existing import. You will not need to make any changes or re-upload your file.

We will keep this ticket in **Pending Customer Verification** status and follow up with you as soon as the production deployment and data reconciliation are complete so you can confirm all 12 results are visible in your workspace.

Best regards,  
Alex Mercer  
Product Support Engineer, Complyance

---

## 3. Reproduction and root cause

- **Environment and steps:** 
  - Bun v1.4.4, Node.js v22.17.1, SQLite 3 (`bun:sqlite`), default port 4328, batch size 5.
  - Reproduced using `POST /imports` with `Content-Type: text/csv` and body from `fixtures/case-002.csv` (12 rows).
  - Returned import ID `29a65986-6753-4e2f-8f67-5a4dd7e534f1`.
  - Baseline runtime evidence documented in [Baseline Runtime Capture](evidence/baseline-runtime-capture.md).
- **Expected versus observed:**
  - *Expected:* All 12 source records processed (`processed: 12`, `pending: 0`, state `completed`), exactly 12 normalized results, batch events respecting `batchSize <= 5` and containing each record ID exactly once.
  - *Observed:* Only 7 records processed (`processed: 7`, `pending: 5`, state `completed`). Only 7 results created (records 1–5 and 11–12). Records 6–10 remained `pending`. Events showed batch 1 (IDs 1–5, count 5), batch 2 (IDs 11–12, count 2), and premature `processing_completed`.
- **Causal explanation:**
  - In `src/worker.ts`, the processing loop called `repository.pendingBatch(workspaceId, importId, batchSize, offset)` and incremented `offset += batch.length` after every committed batch.
  - In `src/repository.ts`, `pendingBatch` filtered rows by `WHERE workspace_id = ? AND import_id = ? AND state = 'pending' ORDER BY id LIMIT ? OFFSET ?`.
  - Crucially, `commitBatch` updates the committed rows from `state = 'pending'` to `state = 'processed'`.
  - When batch 1 (5 rows) was committed, those 5 rows left the pending set. The remaining pending set had 7 rows. Applying `OFFSET 5` to the remaining 7 rows skipped the first 5 of them (rows 6–10) and returned only rows 11–12.
  - After committing rows 11–12, `offset` became 7. The remaining pending set had 5 rows (rows 6–10). Applying `OFFSET 7` to a 5-row set returned 0 rows. The worker broke out of the loop and marked the import `completed` because `finishImport` did not validate that pending rows were zero.
- **Permanent change:**
  1. `src/worker.ts`: Removed the `offset` parameter from the batch loop. The worker now calls `repository.pendingBatch(workspaceId, importId, batchSize)` repeatedly until an empty batch is returned. Because committed records transition to `processed`, querying the first `batchSize` pending records automatically advances through the dataset without skipping rows.
  2. `src/repository.ts`: Removed `offset` from `pendingBatch` query signature.
  3. `src/repository.ts`: Added completion invariant safety checks in `finishImport`:
     ```ts
     if (summary.pending !== 0 || summary.processed !== summary.total || resultCount !== summary.total) {
       throw new Error(`Cannot complete import with incomplete work (total=${summary.total}, processed=${summary.processed}, pending=${summary.pending}, results=${resultCount})`);
     }
     ```
  4. `tests/harness.ts`: Improved process shutdown compatibility on Windows when terminating Bun child processes with SIGTERM (`result.code !== 0 && result.signal !== "SIGTERM"`).

---

## 4. Tests and existing-case reconciliation

| Check | Actual result and evidence |
| --- | --- |
| Regression: failure before, success after | **Pre-fix:** Vitest failed with 2 failed tests (`expected { pending: 0, processed: 12 }` but received `{ pending: 5, processed: 7 }` at batch size 5, and `{ pending: 6, processed: 6 }` at batch size 3). See [Pre-fix Test Output](evidence/test-run-pre-fix.txt).<br>**Post-fix:** Vitest passed 27 of 27 tests in 1.52s. See [Post-fix Test Output](evidence/test-run-post-fix.txt). |
| Varied volumes, batch boundaries, different batch size | Verified 1-record, 3-record, 5-record, and 12-record imports. Tested batch size 5 and batch size 3 (`tests/http.test.ts`). All batch events verified with `count <= batchSize` and exact record ID uniqueness. |
| API validation and workspace isolation | Tested 14 CSV validation rejection cases (empty file, invalid headers, bad quotes, duplicates, invalid references, oversized labels, row limits). Checked 400 on missing headers, 404 on cross-workspace reads (`cedar-demo` vs `maple-demo`). |
| Original case before recovery | Initial runtime baseline on `case-002`: state `completed`, `total: 12`, `processed: 7`, `pending: 5`. Results 1–5 and 11–12 persisted; records 6–10 pending. See [Baseline Runtime Capture](evidence/baseline-runtime-capture.md). |
| Patched-worker restart and approved `resume-case` | Started patched server on `.data/lab.sqlite` and ran `bun run resume-case`. Command returned `{"id":"case-002","workspaceId":"cedar-demo","state":"queued","pending":5}`. Background worker picked up the queued import immediately. |
| Original case after recovery | Verified via API: state `completed`, `total: 12`, `processed: 12`, `pending: 0`. Exactly 12 normalized results in `GET /imports/case-002/results`. Prior result IDs 1–5 and 11–12 retained; records 6–10 processed in batch event ID 14 (`recordIds: [6,7,8,9,10]`, `count: 5`). Zero duplicate processing. See [Post-fix Verification](evidence/post-fix-verification.md). |
| Paused Maple control | Maple control `control-002` remained strictly in state `paused`, `total: 3`, `processed: 0`, `pending: 3`, with 0 results and no extraneous events before and after recovery. |

All checks passed. Commands executed:
```sh
bun run check-types
bun run test
bun run resume-case
```

---

## 5. Escalation and customer ownership

- **Decision and reason:** Escalate to Core Engineering via Linear ticket. While Support has identified the causal defect, authored the local fix, and safely verified recovery in the lab, modifying worker batch pagination and completion invariants requires standard engineering code review, continuous integration verification, and an authorized release deployment.
- **Continuity:** Support (Alex Mercer) retains direct ownership of customer communication with Cedar Demo Malaysia. Next customer checkpoint scheduled for 11:30 GST / 15:30 MYT with conditional closure pending production deployment.
- **Linear-ready draft:**

  ```markdown
  ### [P2] Import worker skips pending records when applying OFFSET to shrinking pending query

  **Issue Type:** Bug  
  **Priority:** P2  
  **Affected Component:** Background Worker / Import Processing  
  **Customer:** Cedar Demo Malaysia (Case PSE-A2-002)  

  #### Problem Summary
  When processing CSV imports in batches, `processImport()` in `src/worker.ts` pages records using an incrementing `offset` (`offset += batch.length`) while querying `repository.pendingBatch()`. Because `pendingBatch()` filters by `state = 'pending'` and `commitBatch()` transitions processed rows to `state = 'processed'`, the pending result set shrinks on every batch.

  Applying an increasing `OFFSET` against a shrinking pending set causes the worker to skip records (e.g., skipping records 6–10 for a 12-row import with batch size 5). Furthermore, `finishImport()` lacked invariant checks, allowing the job to transition to `completed` while rows remained in `pending` state.

  #### Steps to Reproduce
  1. POST a 12-row CSV to `/imports` with workspace `cedar-demo` and default batch size 5.
  2. Wait for background worker processing to finish.
  3. Query `GET /imports/:id`.
  4. Observed: `state: "completed"`, `total: 12`, `processed: 7`, `pending: 5`. Records 6–10 remain unhandled.

  #### Proposed Fix & Lab Verification
  1. In `src/worker.ts`, remove `offset` incrementing; continually fetch the first `batchSize` pending records (`repository.pendingBatch(workspaceId, importId, batchSize)`) until the pending queue is empty.
  2. In `src/repository.ts`, remove `offset` from `pendingBatch()`.
  3. In `src/repository.ts`, add completion invariant assertions in `finishImport()` (`pending === 0`, `processed === total`, `resultCount === total`).
  4. Regression tests covering 12 records with batch sizes 5 and 3 added to `tests/http.test.ts`. All 27 Vitest tests pass.
  5. The lab reconciliation script `bun run resume-case` successfully drained the remaining 5 pending records for `case-002` without duplicates.

  #### Requested Action & Engineering Owner
  - **Requested Owner:** Import Processing / Worker Engineering team
  - **Review Checkpoint:** Next engineering triage / sprint review (targeted: 27 Oct 2026, 10:00 UTC)
  - **Scope:** Review PR, validate database migration/transaction safety, and schedule release.
  ```

- **Exit criteria:**
  1. Engineering approval and merging of the worker batch fix.
  2. Deployment to production environment.
  3. Safe execution of existing-case reconciliation for Cedar Demo Malaysia (`case-002`).
  4. Customer confirmation that all 12 records are visible in their interface.

---

## 6. Submission notes

- **Evidence index:**
  - [Baseline Runtime Capture](evidence/baseline-runtime-capture.md) — Pre-fix API captures (`case-002`, `control-002`, isolation, reproduction).
  - [Static Trace Reference](evidence/baseline-static-trace.md) — Theoretical code-trace analysis.
  - [Pre-fix Test Failure Log](evidence/test-run-pre-fix.txt) — Vitest output showing reproduction failure on unpatched code.
  - [Post-fix Test Success Log](evidence/test-run-post-fix.txt) — Vitest output showing 27/27 tests passing after fix.
  - [Post-fix Verification & Recovery](evidence/post-fix-verification.md) — Post-reconciliation API verification and control preservation.
  - [Test Execution Status](evidence/test-run-status.txt) — Summary of execution environment and verification steps.
- **AI assistance:** OpenAI ChatGPT was utilized during initial code exploration, drafting regression test structures, and preparing preliminary markdown outlines. All code changes, test executions, API calls, runtime reproductions, and reconciliation commands were executed, inspected, and verified on local runtimes (Bun v1.4.4, Node.js v22.17.1).
- **Limitations:** Testing was conducted in a local lab environment using loopback HTTP and SQLite. Production database concurrency, distributed worker clustering, and multi-tenant authentication layers were not evaluated as they are outside the scope of this standalone training packet.
