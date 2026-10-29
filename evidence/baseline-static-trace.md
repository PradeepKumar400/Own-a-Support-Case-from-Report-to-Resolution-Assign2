# Baseline static trace (not a runtime capture)

**Status: source inspection only; the supplied application was not executed in this authoring environment because Bun is not installed. Do not treat this file as an API capture.**

## Source facts
- `starter/src/worker.ts` called `pendingBatch(workspaceId, importId, batchSize, offset)` and incremented `offset` after each committed batch.
- `starter/src/repository.ts` selects only rows whose state is `pending`, ordered by ID, then applies `LIMIT ? OFFSET ?`.
- `commitBatch` changes each selected row from `pending` to `processed` in the same transaction as result insertion and a `batch_processed` event.
- Therefore the eligible pending set shrinks after every batch, while the offset continues increasing. The offset is applied to a shrinking set and skips eligible rows.
- With 12 rows and batch size 5, the code path predicts batches of 5, then 2 (offset 5 into the 7 remaining rows), then 0 (offset 7 into the 5 remaining rows). The worker then calls `finishImport`, which marks the import completed despite pending rows. This predicts 7 processed / 5 pending / 7 results, but this prediction must be confirmed by executing the app.

## Required runtime capture
Run the original code in a clean local lab database and save HTTP status + JSON from `/health`, `/imports/case-002`, `/records`, `/results`, `/events`, and the Maple control endpoints before patching. Keep the original evidence immutable. The static trace above is not a substitute for those captures.
