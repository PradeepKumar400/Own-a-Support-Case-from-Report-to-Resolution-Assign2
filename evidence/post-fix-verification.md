# Post-fix verification and case reconciliation capture

**Timestamp:** 2026-10-29T17:04:42Z (Local: 2026-10-29T22:34:42+05:30)  
**Environment:** Bun v1.4.4, Node.js v22.17.1, SQLite 3 (via bun:sqlite), Windows 10/11  
**Configuration:** `SUPPORT_LAB_PORT=4328`, `SUPPORT_LAB_BATCH_SIZE=5`, `SUPPORT_LAB_DB=.data/lab.sqlite`  
**Application state:** Patched code running against original lab database after executing `bun run resume-case`.

---

## 1. Resume command execution

Executed against the original lab database while the patched server was running:

```powershell
bun run resume-case
```

**Command output:**
```json
{"id":"case-002","workspaceId":"cedar-demo","state":"queued","pending":5}
```

The running worker picked up `case-002` from `state = 'queued'`, processed the remaining pending records [6, 7, 8, 9, 10] in a single bounded batch of 5, and marked the import `completed`.

---

## 2. Reconciled Cedar case (`cedar-demo` / `case-002`)

### Import Summary

```http
GET /imports/case-002 HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 102

{"id":"case-002","workspaceId":"cedar-demo","state":"completed","total":12,"processed":12,"pending":0}
```

- Total: `12`
- Processed: `12`
- Pending: `0`
- State: `completed`

### Source Records

```http
GET /imports/case-002/records HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 904

{
  "records": [
    {"id":1,"documentRef":"DOC-001","label":"Sample 001","state":"processed"},
    {"id":2,"documentRef":"DOC-002","label":"Sample 002","state":"processed"},
    {"id":3,"documentRef":"DOC-003","label":"Sample 003","state":"processed"},
    {"id":4,"documentRef":"DOC-004","label":"Sample 004","state":"processed"},
    {"id":5,"documentRef":"DOC-005","label":"Sample 005","state":"processed"},
    {"id":6,"documentRef":"DOC-006","label":"Sample 006","state":"processed"},
    {"id":7,"documentRef":"DOC-007","label":"Sample 007","state":"processed"},
    {"id":8,"documentRef":"DOC-008","label":"Sample 008","state":"processed"},
    {"id":9,"documentRef":"DOC-009","label":"Sample 009","state":"processed"},
    {"id":10,"documentRef":"DOC-010","label":"Sample 010","state":"processed"},
    {"id":11,"documentRef":"DOC-011","label":"Sample 011","state":"processed"},
    {"id":12,"documentRef":"DOC-012","label":"Sample 012","state":"processed"}
  ]
}
```

All 12 records are now `state = 'processed'`. Stable numeric IDs 1 through 12 are preserved.

### Persisted Results

```http
GET /imports/case-002/results HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 856

{
  "results": [
    {"recordId":1,"documentRef":"DOC-001","normalizedLabel":"SAMPLE 001"},
    {"recordId":2,"documentRef":"DOC-002","normalizedLabel":"SAMPLE 002"},
    {"recordId":3,"documentRef":"DOC-003","normalizedLabel":"SAMPLE 003"},
    {"recordId":4,"documentRef":"DOC-004","normalizedLabel":"SAMPLE 004"},
    {"recordId":5,"documentRef":"DOC-005","normalizedLabel":"SAMPLE 005"},
    {"recordId":6,"documentRef":"DOC-006","normalizedLabel":"SAMPLE 006"},
    {"recordId":7,"documentRef":"DOC-007","normalizedLabel":"SAMPLE 007"},
    {"recordId":8,"documentRef":"DOC-008","normalizedLabel":"SAMPLE 008"},
    {"recordId":9,"documentRef":"DOC-009","normalizedLabel":"SAMPLE 009"},
    {"recordId":10,"documentRef":"DOC-010","normalizedLabel":"SAMPLE 010"},
    {"recordId":11,"documentRef":"DOC-011","normalizedLabel":"SAMPLE 011"},
    {"recordId":12,"documentRef":"DOC-012","normalizedLabel":"SAMPLE 012"}
  ]
}
```

- Exactly 12 results (one per source record).
- Original results (IDs 1, 2, 3, 4, 5, 11, 12) retained their exact identities and normalized values.
- Missing results (IDs 6, 7, 8, 9, 10) have now been produced and persisted.
- Zero duplicate results.

### Events History

```http
GET /imports/case-002/events HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 589

{
  "events": [
    {"id":1,"type":"import_created","details":{"total":12,"state":"queued"}},
    {"id":3,"type":"processing_started","details":{}},
    {"id":4,"type":"batch_processed","details":{"recordIds":[1,2,3,4,5],"count":5}},
    {"id":5,"type":"batch_processed","details":{"recordIds":[11,12],"count":2}},
    {"id":6,"type":"processing_completed","details":{}},
    {"id":12,"type":"resume_requested","details":{"pending":5}},
    {"id":13,"type":"processing_started","details":{}},
    {"id":14,"type":"batch_processed","details":{"recordIds":[6,7,8,9,10],"count":5}},
    {"id":15,"type":"processing_completed","details":{}}
  ]
}
```

- Original event sequence (IDs 1, 3, 4, 5, 6) preserved.
- Resume request recorded as event 12 (`resume_requested`, `pending: 5`).
- Patched worker started processing (event 13).
- New batch committed (event 14, `recordIds: [6,7,8,9,10]`, `count: 5`).
- Processing completed (event 15).
- Every record ID [1..12] appears in exactly one `batch_processed` event.

---

## 3. Maple Paused Control Case (`maple-demo` / `control-002`)

### Control Summary

```http
GET /imports/control-002 HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: maple-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 100

{"id":"control-002","workspaceId":"maple-demo","state":"paused","total":3,"processed":0,"pending":3}
```

### Control Records

```http
GET /imports/control-002/records HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: maple-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 238

{
  "records": [
    {"id":13,"documentRef":"CTRL-001","label":"Control 001","state":"pending"},
    {"id":14,"documentRef":"CTRL-002","label":"Control 002","state":"pending"},
    {"id":15,"documentRef":"CTRL-003","label":"Control 003","state":"pending"}
  ]
}
```

### Control Results

```http
GET /imports/control-002/results HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: maple-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 14

{"results":[]}
```

### Control Events

```http
GET /imports/control-002/events HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: maple-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 84

{"events":[{"id":2,"type":"import_created","details":{"total":3,"state":"paused"}}]}
```

**Verification:** The paused Maple control case remains 100% untouched and pending throughout reconciliation.

