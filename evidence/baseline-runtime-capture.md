# Baseline runtime capture (PSE-A2-002)

**Timestamp:** 2026-10-29T17:01:45Z (Local: 2026-10-29T22:31:45+05:30)  
**Environment:** Bun v1.4.4, Node.js v22.17.1, SQLite 3 (via bun:sqlite), Windows 10/11  
**Configuration:** `SUPPORT_LAB_PORT=4328`, `SUPPORT_LAB_BATCH_SIZE=5`, `SUPPORT_LAB_DB=.data/lab.sqlite`  
**Application state:** Unpatched code running against fresh seeded database.

---

## 1. Health check

```http
GET /health HTTP/1.1
Host: 127.0.0.1:4328

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 47

{"status":"ok","mode":"training","batchSize":5}
```

---

## 2. Seeded case (`cedar-demo` / `case-002`)

### Import Summary

```http
GET /imports/case-002 HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 101

{"id":"case-002","workspaceId":"cedar-demo","state":"completed","total":12,"processed":7,"pending":5}
```

### Source Records

```http
GET /imports/case-002/records HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 894

{
  "records": [
    {"id":1,"documentRef":"DOC-001","label":"Sample 001","state":"processed"},
    {"id":2,"documentRef":"DOC-002","label":"Sample 002","state":"processed"},
    {"id":3,"documentRef":"DOC-003","label":"Sample 003","state":"processed"},
    {"id":4,"documentRef":"DOC-004","label":"Sample 004","state":"processed"},
    {"id":5,"documentRef":"DOC-005","label":"Sample 005","state":"processed"},
    {"id":6,"documentRef":"DOC-006","label":"Sample 006","state":"pending"},
    {"id":7,"documentRef":"DOC-007","label":"Sample 007","state":"pending"},
    {"id":8,"documentRef":"DOC-008","label":"Sample 008","state":"pending"},
    {"id":9,"documentRef":"DOC-009","label":"Sample 009","state":"pending"},
    {"id":10,"documentRef":"DOC-010","label":"Sample 010","state":"pending"},
    {"id":11,"documentRef":"DOC-011","label":"Sample 011","state":"processed"},
    {"id":12,"documentRef":"DOC-012","label":"Sample 012","state":"processed"}
  ]
}
```

### Persisted Results

```http
GET /imports/case-002/results HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 505

{
  "results": [
    {"recordId":1,"documentRef":"DOC-001","normalizedLabel":"SAMPLE 001"},
    {"recordId":2,"documentRef":"DOC-002","normalizedLabel":"SAMPLE 002"},
    {"recordId":3,"documentRef":"DOC-003","normalizedLabel":"SAMPLE 003"},
    {"recordId":4,"documentRef":"DOC-004","normalizedLabel":"SAMPLE 004"},
    {"recordId":5,"documentRef":"DOC-005","normalizedLabel":"SAMPLE 005"},
    {"recordId":11,"documentRef":"DOC-011","normalizedLabel":"SAMPLE 011"},
    {"recordId":12,"documentRef":"DOC-012","normalizedLabel":"SAMPLE 012"}
  ]
}
```

### Events History

```http
GET /imports/case-002/events HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 343

{
  "events": [
    {"id":1,"type":"import_created","details":{"total":12,"state":"queued"}},
    {"id":3,"type":"processing_started","details":{}},
    {"id":4,"type":"batch_processed","details":{"recordIds":[1,2,3,4,5],"count":5}},
    {"id":5,"type":"batch_processed","details":{"recordIds":[11,12],"count":2}},
    {"id":6,"type":"processing_completed","details":{}}
  ]
}
```

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

---

## 4. Cross-workspace isolation checks

```http
GET /imports/control-002 HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 404 Not Found
Content-Type: application/json;charset=utf-8
Content-Length: 67

{"code":"not_found","message":"Import not found in this workspace"}
```

```http
GET /imports/case-002 HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: maple-demo

HTTP/1.1 404 Not Found
Content-Type: application/json;charset=utf-8
Content-Length: 67

{"code":"not_found","message":"Import not found in this workspace"}
```

```http
GET /imports/case-002 HTTP/1.1
Host: 127.0.0.1:4328

HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=utf-8
Content-Length: 88

{"code":"invalid_argument","message":"x-lab-workspace must be cedar-demo or maple-demo"}
```

---

## 5. Runtime reproduction with isolated CSV import

Submitted `starter/fixtures/case-002.csv` via `POST /imports`:

```http
POST /imports HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo
Content-Type: text/csv

[Body: 12 rows from fixtures/case-002.csv]

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
Content-Length: 100

{"id":"29a65986-6753-4e2f-8f67-5a4dd7e534f1","workspaceId":"cedar-demo","state":"queued","total":12}
```

After worker execution completed:

```http
GET /imports/29a65986-6753-4e2f-8f67-5a4dd7e534f1 HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 129

{"id":"29a65986-6753-4e2f-8f67-5a4dd7e534f1","workspaceId":"cedar-demo","state":"completed","total":12,"processed":7,"pending":5}
```

```http
GET /imports/29a65986-6753-4e2f-8f67-5a4dd7e534f1/results HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 510

{
  "results": [
    {"recordId":16,"documentRef":"DOC-001","normalizedLabel":"SAMPLE 001"},
    {"recordId":17,"documentRef":"DOC-002","normalizedLabel":"SAMPLE 002"},
    {"recordId":18,"documentRef":"DOC-003","normalizedLabel":"SAMPLE 003"},
    {"recordId":19,"documentRef":"DOC-004","normalizedLabel":"SAMPLE 004"},
    {"recordId":20,"documentRef":"DOC-005","normalizedLabel":"SAMPLE 005"},
    {"recordId":26,"documentRef":"DOC-011","normalizedLabel":"SAMPLE 011"},
    {"recordId":27,"documentRef":"DOC-012","normalizedLabel":"SAMPLE 012"}
  ]
}
```

```http
GET /imports/29a65986-6753-4e2f-8f67-5a4dd7e534f1/events HTTP/1.1
Host: 127.0.0.1:4328
x-lab-workspace: cedar-demo

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 350

{
  "events": [
    {"id":7,"type":"import_created","details":{"total":12,"state":"queued"}},
    {"id":8,"type":"processing_started","details":{}},
    {"id":9,"type":"batch_processed","details":{"recordIds":[16,17,18,19,20],"count":5}},
    {"id":10,"type":"batch_processed","details":{"recordIds":[26,27],"count":2}},
    {"id":11,"type":"processing_completed","details":{}}
  ]
}
```

**Verification:** The reproduction unequivocally confirms that the worker bug skips rows 6 through 10 (which stay `pending`), while processing only 7 rows and marking the job `completed`.

