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
    queue := snerd.NewAnyQueue("my-queue", 10, 2*time.Second)

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

Creates a new queue with the given name, max worker goroutines, and polling interval.

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

## Storage

By default, `snerd-go` writes to `.snerdata/tasks/tasks.log`. You can share this file with `snerdmq-node`, `snerdmq-python`, and other SDKs for cross-language queue sharing via OS-level file locking.
