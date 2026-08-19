# snerd-go (Embedded Go Library)

`snerd-go` is a **native Go embedded library** — no Rust daemon, no child process, no binary downloads. It's the ideal choice for pure Go applications that want background jobs without bundling an external binary.

[![Go Reference](https://pkg.go.dev/badge/github.com/speed-nerd/snerd-go.svg)](https://pkg.go.dev/github.com/speed-nerd/snerd-go)

## When to Use This

| Use `snerd-go` | Use `snerdmq-go` (thin client) |
|---|---|
| Pure Go apps, no external binaries | Polyglot microservices sharing one queue file |
| Simple deployments, no binary to manage | When you need cross-language queue interop |
| Go-only services | Go + Node/Python/PHP services on the same queue |

## Installation

```bash
go get github.com/speed-nerd/snerd-go
```

That's it. No install scripts, no binary downloads.

## Quickstart

```go
package main

import (
    "context"
    "fmt"
    "time"

    snerd "github.com/speed-nerd/snerd-go"
)

func main() {
    // Create the queue (name, max concurrency, poll interval)
    // For local development this persists to ./.snerdata/tasks/tasks.log
    queue := snerd.NewAnyQueue("my-queue", 10, 2*time.Second)

    // Need a custom location instead? (durable network-drive storage, per-server
    // isolation, or keeping tests out of .snerdata)
    // queue := snerd.NewAnyQueueWithStorage("my-queue", 10, 2*time.Second, "/var/data/snerd/tasks.log")

    // Register a handler
    snerd.RegisterTaskHandler("send_email", func(ctx context.Context, data string) error {
        fmt.Printf("Sending email: %s\n", data)
        return nil  // Return error to trigger retry
    })

    // Register a DLQ handler
    snerd.RegisterMaxRetryHandler("send_email", func(ctx context.Context, data string) error {
        fmt.Printf("Permanently failed: %s\n", data)
        return nil
    })

    // Enqueue a task
    task, _ := snerd.NewSnerdTask(
        "email-123",
        "send_email",
        map[string]string{"to": "user@example.com"},
        3,    // max retries
        1.0,  // retry after hours
    )
    queue.EnqueueSnerdTask(task)

    select {} // Keep alive
}
```

## Advanced Configuration

```go
rateLimitGroup := "openai_api"
maxPerMinute := 50
autoDedupe := true
urgencyScore := 0.95
cronStr := "1h"

task, _ := snerd.NewSnerdTaskAdvanced(
    "task-123",
    "generate_report",
    map[string]string{"prompt": "A crab in space"},
    3,
    1.0,
    &rateLimitGroup,
    &maxPerMinute,
    &autoDedupe,
    &urgencyScore,
    nil,       // executeAt
    &cronStr,
    nil,       // webhookUrl
    nil,       // maxExecutionSeconds
)
queue.EnqueueSnerdTask(task)
```

## API Reference

### `NewAnyQueue(name, maxConcurrency, pollInterval)`

Creates a new queue with the given name, max worker goroutines, and polling interval. Panics if another queue instance already owns the default storage file.

### `NewAnyQueueWithStorage(name, maxConcurrency, pollInterval, storePath)`

Same as `NewAnyQueue`, but persists tasks to a custom file location instead of the default `.snerdata/tasks/tasks.log`:

```go
queue := snerd.NewAnyQueueWithStorage(
    "image-processing",
    10,
    2*time.Second,
    "/mnt/efs/image-jobs/tasks.log",
)
```

The rate limiter state (`rate_limits.json`) is stored alongside the task log, so two queues on different paths are fully independent.

!!! warning "One queue instance per storage file"
    Each queue takes an exclusive OS-level lock on its task log (e.g. `tasks.log.lock`) at creation. A second queue on the same file **panics** instead of racing it and double-executing tasks. Register all your task types on a single queue, or give each queue its own path.

### `RegisterTaskHandler(taskType, handler)`

Registers a handler for a task type. The handler signature is:
```go
func(ctx context.Context, data string) error
```

### `RegisterMaxRetryHandler(taskType, handler)`

Registers a Dead Letter Queue handler for permanently failed tasks.

### `NewSnerdTask(id, taskType, data, maxRetries, retryAfterHours)`

Creates a basic task.

### `NewSnerdTaskAdvanced(...)`

Creates a task with all advanced options (rate limit, dedup, priority, cron, webhook, timeout).

### `queue.EnqueueSnerdTask(task)`

Enqueues a task for background execution.

### `queue.StartDashboard(port)`

Starts the built-in React dashboard on the specified port.

## Dashboard

```go
queue := snerd.NewAnyQueue("my-queue", 10, 2*time.Second)
queue.StartDashboard(9090)
// Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Register every job type on a single queue and serve one shared dashboard:

```go
queue := snerd.NewAnyQueue("main", 10, 2*time.Second)

// Two job types sharing the same queue
snerd.RegisterTaskHandler("process_image", func(ctx context.Context, data string) error {
    fmt.Printf("Processing image: %s\n", data)
    return nil
})
snerd.RegisterTaskHandler("send_otp_email", func(ctx context.Context, data string) error {
    fmt.Printf("Sending OTP: %s\n", data)
    return nil
})

queue.StartDashboard(9090) // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

Need isolation between workloads? Give each queue its own storage file — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```go
images := snerd.NewAnyQueueWithStorage("images", 10, 2*time.Second, "./.snerdata-images/tasks.log")
emails := snerd.NewAnyQueueWithStorage("emails", 10, 500*time.Millisecond, "./.snerdata-emails/tasks.log")

images.StartDashboard(9090)
emails.StartDashboard(9091)
```

## Storage

By default, `snerd-go` writes to `.snerdata/tasks/tasks.log`.

Use `NewAnyQueueWithStorage` when you need:

- **Queue isolation** — separate log files per concern (`kyc-retry-queue.log` vs `image-processing.log`), which also gives each queue its own independent dashboard view
- **Durable network drives** — point a single instance at an EFS/NFS mount so its queue state survives container restarts
- **Test isolation** — write to a temp directory instead of clobbering `.snerdata`

### Distributed Scaling

A queue instance exclusively owns its storage file: it takes an OS-level lock (`<tasks.log>.lock`) at creation and holds it for its lifetime. A second instance pointed at the same file — in the same process or on another server — fails fast instead of racing it and double-executing tasks.

Scaling out therefore means **one queue per server**, each with its own storage. Your load balancer routes requests across servers, and every server processes the tasks it enqueued:

```go
// Each server runs its own queue on its own log file (local disk works fine)
queue := snerd.NewAnyQueueWithStorage(
    "worker-server-1",
    10,
    2*time.Second,
    "/var/data/snerd/tasks.log",
)
```

A shared network drive (AWS EFS or NFS) is still a good home for that log when a single instance needs durable storage. OS-level file locking keeps writes safe — no Redis required.
