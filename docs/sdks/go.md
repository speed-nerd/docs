# Go SDK

The official Go SDK for SnerdMQ. Provides both a thin-client SDK (`snerdmq-go`) and an embedded library (`snerd-go`).

[![Go Reference](https://pkg.go.dev/badge/github.com/speed-nerd/snerdmq-go.svg)](https://pkg.go.dev/github.com/speed-nerd/snerdmq-go)

## Installation

**1. Install the module:**
```bash
go get github.com/speed-nerd/snerdmq-go
```

**2. Download the Rust engine:**
```bash
go run github.com/speed-nerd/snerdmq-go/cmd/snerdmq-install@latest
```

## Quickstart

```go
package main

import (
    "context"
    "fmt"
    "github.com/speed-nerd/snerdmq-go"
)

func main() {
    queue, err := snerdmq.NewSnerdQueue()
    if err != nil {
        panic(err)
    }

    // Register a handler
    queue.RegisterHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
        fmt.Printf("Sending email to %s...\n", data["to"])
        return nil  // Return error to trigger retry
    })

    // Start listening (non-blocking)
    queue.StartListening()

    // Enqueue a job
    queue.Enqueue(
        "email-123", "send_email",
        map[string]interface{}{"to": "user@example.com"},
        3,    // max retries
        0.5,  // retry after hours
        "", 0, // rate limit group, max per minute
        nil, nil, nil, nil, nil, nil,
    )

    // Handle permanently failed tasks
    queue.RegisterMaxRetryHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
        fmt.Printf("Failed after all retries: %v\n", data)
        return nil
    })

    queue.Wait()
}
```

## API Reference

### `NewSnerdQueue(config?)`

Creates a new queue instance.

```go
queue, _ := snerdmq.NewSnerdQueue()
// Or with custom config:
queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
    StoragePath: "/custom/path/tasks.log",
})
```

### `queue.RegisterHandler(taskType, handler)`

Registers a handler function for a task type.

### `queue.Enqueue(...)`

Enqueues a new background job with positional arguments:

```go
queue.Enqueue(
    taskId,             // string
    taskType,           // string
    data,               // map[string]interface{}
    maxRetries,         // int
    retryAfterHours,    // float64
    rateLimitGroup,     // string
    maxPerMinute,       // int
    autoDedupe,         // *bool
    urgencyScore,       // *float64
    executeAt,          // interface{} (string or time.Time)
    cron,               // *string
    webhookUrl,         // *string
    maxExecutionSeconds, // *int
)
```

### `queue.RegisterMaxRetryHandler(taskType, handler)`

Registers a handler for permanently failed tasks (Dead Letter Queue).

### `queue.StartDashboard(port)`

Starts the built-in React dashboard.

### `queue.YieldProgress(ctx, message)`

Streams a progress update from within a handler.

### `queue.Wait()`

Blocks the main thread (keeps the program running).

## Embedded Library: `snerd-go`

For pure Go applications that don't need the Rust daemon, use the embedded library:

```bash
go get github.com/speed-nerd/snerd-go
```

This provides native Go queue orchestration without spawning a child process.

## Distributed Scaling

```go
queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
    StoragePath: "/mnt/aws-efs-shared-drive/snerd_tasks.log",
})
```
