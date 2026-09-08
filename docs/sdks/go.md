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

    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    <-sigs
    queue.Shutdown()
}
```

## API Reference

### `NewSnerdQueue(config?)`

Creates a new queue instance.

```go
queue, _ := snerdmq.NewSnerdQueue()
// Or with a custom storage directory:
queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
    StoragePath: "/custom/path",
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

### `queue.Shutdown()`

Gracefully shuts down the queue and background daemon.

## Embedded Library: `snerd-go`

For pure Go applications that don't need the Rust daemon, use the embedded library:

```bash
go get github.com/speed-nerd/snerd-go
```

This provides native Go queue orchestration without spawning a child process.

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```go
queue, _ := snerdmq.NewSnerdQueue()

// Two job types sharing the same queue, daemon, and dashboard
queue.RegisterHandler("process_image", func(ctx context.Context, data map[string]interface{}) error {
    fmt.Printf("Processing image: %v\n", data["image_id"])
    return nil
})
queue.RegisterHandler("send_otp_email", func(ctx context.Context, data map[string]interface{}) error {
    fmt.Printf("Sending OTP to: %v\n", data["to"])
    return nil
})

queue.StartDashboard(8080) // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — multi-worker deployments need one storage directory per worker.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```go
images, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{StoragePath: ".snerdata-images"})
emails, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{StoragePath: ".snerdata-emails"})

images.StartDashboard(8080)
emails.StartDashboard(8081)
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```go
// Each server runs its own daemon on its own storage dir (local disk works fine)
queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
    StoragePath: "/var/data/snerd",
})
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.


## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	snerdmq "github.com/speed-nerd/snerdmq-go"
)

var queue *snerdmq.SnerdQueue

func initEmailWorkers() {
	queue.RegisterHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
		log.Printf("Sending email to %s...", data["email"])
		return nil
	})
	queue.RegisterMaxRetryHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
		log.Printf("Email to %s failed permanently. Dead letter processing...", data["email"])
		return nil
	})
}

func initImageWorkers() {
	queue.RegisterHandler("process_image", func(ctx context.Context, data map[string]interface{}) error {
		log.Printf("Processing image %s...", data["imageId"])
		return nil
	})
}

func main() {
	queue = snerdmq.NewSnerdQueue(snerdmq.SnerdQueueOptions{ StoragePath: "./.snerdata" })
	
	initEmailWorkers()
	initImageWorkers()

	queue.StartDashboard(8080)
	
	if err := queue.StartListening(); err != nil {
		log.Fatalf("Failed to start daemon: %v", err)
	}

	// Wait for termination signal
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
	<-sigChan

	// Manually shut down the queue safely
	queue.Shutdown()
}
```
