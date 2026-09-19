# Distributed Compute Grid — Message Protocol

This document is the single source of truth for how the Coordinator, Worker,
and Client processes talk to each other. Every message is a single line of
JSON sent over a TCP socket, terminated by `\n`.

All messages share a common envelope:

```json
{
  "type": "MESSAGE_TYPE_HERE",
  "senderId": "worker-1",
  "payload": { ... }
}
```

- `type` — one of the message types below (a String constant).
- `senderId` — who sent it. `"coordinator"`, or a worker/client's assigned ID.
- `payload` — an object whose shape depends on `type` (defined per message below).

---

## 1. Worker → Coordinator messages

### REGISTER
Sent once, right after a worker connects.
```json
{
  "type": "REGISTER",
  "senderId": "worker-1",
  "payload": {
    "maxConcurrentChunks": 2
  }
}
```

### TASK_REQUEST
Worker asks for a chunk of work when idle.
```json
{
  "type": "TASK_REQUEST",
  "senderId": "worker-1",
  "payload": {}
}
```

### HEARTBEAT
Sent every N seconds (agree on N — suggest 2s) while connected.
```json
{
  "type": "HEARTBEAT",
  "senderId": "worker-1",
  "payload": {
    "currentLoad": 1,
    "timestamp": 1732000000000
  }
}
```

### TASK_RESULT
Sent when a worker finishes computing a chunk.
```json
{
  "type": "TASK_RESULT",
  "senderId": "worker-1",
  "payload": {
    "taskId": "chunk-7",
    "success": true,
    "result": [[1,2,3],[4,5,6]],
    "errorMessage": null
  }
}
```

---

## 2. Coordinator → Worker messages

### TASK_ASSIGN
```json
{
  "type": "TASK_ASSIGN",
  "senderId": "coordinator",
  "payload": {
    "taskId": "chunk-7",
    "rowRange": [0, 49],
    "matrixA": [[1,2],[3,4]],
    "matrixB": [[5,6],[7,8]]
  }
}
```
Note: for matrix multiplication chunking, `matrixA` is only the rows this
worker needs; `matrixB` is sent in full (every worker needs all of it).

### NO_TASK_AVAILABLE
Sent instead of TASK_ASSIGN if the queue is currently empty.
```json
{
  "type": "NO_TASK_AVAILABLE",
  "senderId": "coordinator",
  "payload": {}
}
```

---

## 3. Client ↔ Coordinator messages

### JOB_SUBMIT
```json
{
  "type": "JOB_SUBMIT",
  "senderId": "client-1",
  "payload": {
    "jobType": "MATRIX_MULTIPLY",
    "matrixA": [[1,2],[3,4]],
    "matrixB": [[5,6],[7,8]]
  }
}
```

### JOB_RESULT
```json
{
  "type": "JOB_RESULT",
  "senderId": "coordinator",
  "payload": {
    "jobId": "job-42",
    "result": [[19,22],[43,50]],
    "elapsedMs": 842
  }
}
```

---

## 4. Chunking rule (agree on this explicitly)

For matrix multiplication with W workers and an N-row result matrix:
- Split rows of matrix A into W roughly-equal row ranges (last chunk absorbs
  any remainder rows).
- Each chunk's `taskId` should be stable and unique, e.g. `"chunk-<jobId>-<index>"`,
  so a reassigned chunk keeps the same ID after a worker failure.
- Coordinator reassembles the final matrix by placing each chunk's result
  rows back at `rowRange` in the output matrix — NOT by append order, since
  chunks can complete out of order.

## 5. Timeouts (agree on these numbers as a team)

| Constant | Suggested value | Meaning |
|---|---|---|
| `HEARTBEAT_INTERVAL_MS` | 2000 | How often a worker sends a heartbeat |
| `HEARTBEAT_TIMEOUT_MS` | 6000 | If no heartbeat received in this long, worker is presumed dead |
| `MAX_TASK_RETRIES` | 3 | How many times a chunk can be reassigned before the job fails |

## 6. Open questions to settle on the team call

- [ ] What happens if the queue is empty and a worker asks for a task — does
      it wait, poll again after a delay, or disconnect?
- [ ] Does the client wait synchronously for JOB_RESULT, or poll?
- [ ] Fixed port number for the coordinator (suggest `5000`)?
