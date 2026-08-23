# Architecture

SnerdMQ uses an **embedded sidecar** architecture. Instead of running a separate queue server (like Redis or RabbitMQ), each application instance spawns its own lightweight Rust daemon as a child process. Each daemon exclusively owns its storage directory — scale by giving each server its own queue and storage (sharding), never by pointing multiple instances at one shared file.

## Overview

```
┌──────────────────────────────────────────────────┐
│  Application Server (Node, Python, Go, etc.)      │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  SnerdMQ SDK (thin client)                   │ │
│  │                                              │ │
│  │  • Enqueue tasks       • Register handlers   │ │
│  │  • Stream progress     • Serve dashboard     │ │
│  │  • JSON-RPC over stdin/stdout                │ │
│  └──────────────────┬───────────────────────────┘ │
│                     │                              │
│  ┌──────────────────▼───────────────────────────┐ │
│  │  snerdmq daemon (Rust sidecar process)       │ │
│  │                                              │ │
│  │  ┌─────────────┐  ┌──────────────────────┐  │ │
│  │  │ Append-only │  │ Priority Queue       │  │ │
│  │  │ Log (disk)  │  │ (Binary Max-Heap)    │  │ │
│  │  └─────────────┘  └──────────────────────┘  │ │
│  │                                              │ │
│  │  ┌─────────────┐  ┌──────────────────────┐  │ │
│  │  │ Rate        │  │ Cron Scheduler       │  │ │
│  │  │ Limiter     │  │                      │  │ │
│  │  └─────────────┘  └──────────────────────┘  │ │
│  │                                              │ │
│  │  ┌─────────────────────────────────────────┐ │ │
│  │  │ OS File Locking (flock / fs3)           │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

## The Daemon

The `snerdmq` binary is a single, statically-compiled Rust executable (~5 MB) that handles all queue orchestration:

| Responsibility | Implementation |
|---|---|
| **Persistence** | Append-only JSON-lines log (`.snerdata/tasks/tasks.log`) |
| **Concurrency control** | OS-level file locking via `fs3` (`flock` on Linux/macOS, `LockFileEx` on Windows) |
| **Task ordering** | Binary Max-Heap keyed on `urgency_score` for priority dispatch |
| **Retry logic** | Configurable backoff with `retry_after_hours` |
| **Rate limiting** | Rolling-window velocity enforcement per `rate_limit_group` |
| **Scheduling** | Cron expression parsing for recurring jobs |
| **Deduplication** | xxHash-based payload fingerprinting |
| **Webhook dispatch** | HTTP POST with `X-SnerdMQ-Event` headers |
| **Hard timeouts** | `tokio::time::timeout` enforcement per task |
| **Log compaction** | Periodic rewrite to reclaim space from deleted/completed tasks |

The daemon is deliberately **not** a network service. It communicates exclusively over stdin/stdout pipes, which means:

- No port conflicts
- No firewall rules
- No authentication tokens
- No network latency

## The SDK Contract

Every SDK — regardless of language — follows the same contract:

1. **Spawn** the `snerdmq` binary as a child process
2. **Send** JSON messages to the daemon's stdin
3. **Read** JSON messages from the daemon's stdout
4. **Dispatch** incoming messages to registered handlers

### JSON-RPC Protocol

The protocol is newline-delimited JSON. Each line is a complete JSON object with an `action` field.

**SDK → Daemon (enqueue a task):**
```json
{
  "action": "enqueue",
  "task_id": "email-123",
  "task_type": "send_email",
  "task_data": "{\"to\": \"user@example.com\"}",
  "max_retries": 3,
  "retry_after_hours": 0.5,
  "auto_dedupe": true,
  "urgency_score": 0.0,
  "rate_limit_group": "email_api",
  "max_per_minute": 100
}
```

**Daemon → SDK (execute a task):**
```json
{
  "action": "execute",
  "task_id": "email-123",
  "task_type": "send_email",
  "task_data": "{\"to\": \"user@example.com\"}"
}
```

**Daemon → SDK (acknowledge enqueue):**
```json
{
  "action": "ack",
  "task_id": "email-123"
}
```

**SDK → Daemon (report result):**
```json
{
  "action": "result",
  "task_id": "email-123",
  "success": true
}
```

### Message Types

| Direction | Action | Purpose |
|---|---|---|
| SDK → Daemon | `enqueue` | Add a task to the queue |
| SDK → Daemon | `result` | Report task execution outcome |
| SDK → Daemon | `progress` | Send progress update for a running task |
| Daemon → SDK | `execute` | Dispatch a task for execution |
| Daemon → SDK | `ack` | Confirm task was enqueued |
| Daemon → SDK | `error` | Report an error (e.g., duplicate task) |
| Daemon → SDK | `progress` | Forward progress event to dashboard |
| Daemon → SDK | `max_retries_reached` | Dead Letter Queue event |

## The Append-Only Log

All task state is persisted to a single file: `.snerdata/tasks/tasks.log`

Each line is a JSON object representing the latest state of a task. When a task is updated (retry, completion, deletion), a new line is appended — the old line is never modified in place.

```jsonl
{"task_id":"email-123","task_type":"send_email","task_data":"{...}","retry_count":0,...}
{"task_id":"email-123","task_type":"send_email","task_data":"{...}","retry_count":1,"last_error":"timeout",...}
{"task_id":"email-123","task_type":"send_email","task_data":"{...}","deleted_at":"2026-08-18T10:30:00Z",...}
```

### Why Append-Only?

- **Crash safety** — No partial writes. Each line is atomically appended.
- **Audit trail** — Full history of every state transition.
- **Simplicity** — No database engine, no WAL, no checkpointing.
- **Compaction** — Periodically, the daemon rewrites the log keeping only the latest state of each task, reclaiming space.

### File Locking

All log access is protected by OS-level file locks:

- **Linux/macOS** — `flock(2)` system call
- **Windows** — `LockFileEx` API

Locks serve two purposes:

- **Write atomicity** — short-lived locks serialize appends and compaction, so the log is never corrupted.
- **Exclusive ownership** — at startup, every daemon/queue instance takes a persistent exclusive lock on its storage (`<storage>/.lock` for the daemon, `<tasks.log>.lock` for the embedded libraries). A second instance on the same storage refuses to start, guaranteeing exactly one executor per queue. This is why the scaling model is sharding — one queue per server, each with its own storage (see [Deployment](production/deployment.md)).

## Task Lifecycle

```
                    ┌──────────┐
                    │ Enqueued │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
              ┌─────│  Active  │─────┐
              │     └────┬─────┘     │
              │          │           │
         ┌────▼────┐ ┌──▼───┐ ┌────▼────────┐
         │ Failed  │ │ Done │ │ Timed Out   │
         └────┬────┘ └──────┘ └────┬────────┘
              │                     │
         ┌────▼──────────┐         │
         │ Retry?        │◄────────┘
         │ (if retries   │
         │  remaining)   │
         └────┬────┬─────┘
              │    │
          Yes │    │ No
              │    │
    ┌─────────▼┐ ┌─▼───────────┐
    │ Scheduled│ │ Dead Letter │
    │ (backoff)│ │ Queue (DLQ) │
    └──────────┘ └─────────────┘
```

1. **Enqueued** — Task is persisted to the log and awaiting dispatch
2. **Active** — Task has been dispatched to an SDK handler for execution
3. **Completed** — Handler returned success; task is soft-deleted from the queue
4. **Failed** — Handler threw an error; task is scheduled for retry or moved to DLQ
5. **Timed Out** — Execution exceeded `max_execution_seconds`; treated as a failure
6. **Retry** — Task's `retry_after_time` is set; it will be re-dispatched after the backoff
7. **Dead Letter Queue** — All retries exhausted; `max_retries_reached` event is fired

!!! note "Delivery semantics"
    SnerdMQ provides **at-least-once** delivery. The in-memory dispatch state (which task is currently executing) is not durable, so if the daemon is killed mid-execution, the task is re-dispatched after restart. Handlers must be **idempotent**.

## Embedded Libraries vs. Daemon SDKs

SnerdMQ offers two ways to use the queue engine:

| | Daemon SDKs (Node, Python, Go, Ruby, PHP, Java, .NET) | Embedded Libraries (snerd-rust, snerd-go) |
|---|---|---|
| **Architecture** | SDK spawns the Rust daemon as a child process | Engine runs in-process (no separate binary) |
| **Language** | Any language with SDK | Rust or Go only |
| **Performance** | IPC overhead (microseconds per message) | Zero IPC — direct function calls |
| **Polyglot** | All SDKs use the same storage format (each instance exclusively owns its own storage) | Same log file format — storage is portable across SDKs |
| **Binary required** | Yes (auto-downloaded on install) | No (compiled into your app) |
| **Use when** | You want language flexibility or polyglot services | You want maximum performance in Rust/Go |

Both approaches use the **same log file format** and **same OS file locking**, so an embedded Rust worker and a Node.js worker can share the exact same queue.
