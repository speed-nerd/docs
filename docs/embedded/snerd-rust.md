# snerd-rust (Embedded Rust Library)

`snerd-rust` is a **native Rust embedded library** for background jobs — no daemon process, no IPC, no external dependencies beyond your crate. It's built on tokio for maximum async performance.

[![Crates.io](https://img.shields.io/crates/v/snerd-rust.svg)](https://crates.io/crates/snerd-rust)
[![docs.rs](https://docs.rs/snerd-rust/badge.svg)](https://docs.rs/snerd-rust)

!!! important
    If you're writing a Rust application, **do NOT use the `snerdmq` daemon**. Instead, use `snerd-rust` directly — it gives you native closures, full async support, and zero IPC overhead.

## When to Use This

| Use `snerd-rust` | Use `snerdmq` daemon |
|---|---|
| Pure Rust applications | Node.js, Python, Go, Ruby, PHP, Java, C# apps |
| Maximum performance, zero IPC overhead | Polyglot architectures (each service owns its own queue) |
| No external binary to bundle | When you need the sidecar architecture |

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
snerd-rust = "0.2.4"
tokio = { version = "1", features = ["full"] }
```

## Quickstart

```rust
use snerd_rust::file_store::FileStore;
use snerd_rust::queue::SnerdQueue;
use snerd_rust::rate_limiter::RateLimiter;
use snerd_rust::task::RetryableTask;
use std::time::Duration;

#[tokio::main]
async fn main() {
    // 1. Initialize the persistence store
    let file_store = FileStore::new(".snerdata/tasks/tasks.log").unwrap();

    // 2. Create the queue
    let queue = SnerdQueue::new(
        "my-queue",
        file_store,
        RateLimiter::new(&std::path::PathBuf::from(".snerdata")),
    );

    // 3. Register a handler
    queue.register_task_handler("send_email", |data| {
        println!("Sending email: {}", data);
        Ok(())  // Return Err to trigger retry
    }).await;

    // 4. Register a DLQ handler
    queue.register_max_retry_handler("send_email", |data| {
        println!("Permanently failed: {}", data);
        Ok(())
    }).await;

    // 5. Start the background processor
    queue.start_processor(Duration::from_secs(2)).await;

    // 6. Enqueue a task
    let task = RetryableTask::new(
        "email-123".to_string(),
        "send_email".to_string(),
        r#"{"to": "user@example.com"}"#.to_string(),
        3,    // max retries
        1.0,  // retry after hours
        None, None, None, None,
        None, None, None, None,
    );
    queue.enqueue(task).unwrap();

    tokio::time::sleep(Duration::from_secs(60)).await;
}
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Register every job type on a single queue and serve one shared dashboard:

```rust
let file_store = FileStore::new(".snerdata/tasks/tasks.log").unwrap();
let queue = SnerdQueue::new(
    "main",
    file_store,
    RateLimiter::new(&std::path::PathBuf::from(".snerdata")),
);

// Two job types sharing the same queue
queue.register_task_handler("process_image", |data| {
    println!("Processing image: {}", data);
    Ok(())
}).await;

queue.register_task_handler("send_otp_email", |data| {
    println!("Sending OTP: {}", data);
    Ok(())
}).await;

queue.start_dashboard(9090); // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

Need isolation between workloads? Give each queue its own storage file — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```rust
let images = SnerdQueue::new(
    "images",
    FileStore::new(".snerdata-images/tasks.log").unwrap(),
    RateLimiter::new(&std::path::PathBuf::from(".snerdata-images")),
);
let emails = SnerdQueue::new(
    "emails",
    FileStore::new(".snerdata-emails/tasks.log").unwrap(),
    RateLimiter::new(&std::path::PathBuf::from(".snerdata-emails")),
);

images.start_dashboard(9090);
emails.start_dashboard(9091);
```

## API Reference

### `FileStore::new(path)`

Creates a new persistence store backed by the given file path.

### `SnerdQueue::new(name, file_store, rate_limiter)`

Creates a new queue instance.

!!! warning "One queue instance per storage file"
    `SnerdQueue::new` takes an exclusive OS-level lock on the storage file (e.g. `tasks.log.lock`) and holds it for the queue's lifetime. A second queue on the same file **panics** instead of racing it and double-executing tasks. Register all your task types on a single queue, or create a `FileStore` with a different path for each queue.

### `queue.register_task_handler(task_type, handler)`

Registers an async handler closure for a task type. The handler receives the task's JSON data as a `String` and returns `Result<(), String>`.

### `queue.register_max_retry_handler(task_type, handler)`

Registers a Dead Letter Queue handler for permanently failed tasks.

### `queue.start_processor(poll_interval)`

Boots the background processing loop.

### `queue.enqueue(task)`

Enqueues a `RetryableTask` for background execution.

### `RetryableTask::new(...)`

Creates a new task with all configuration options:

```rust
RetryableTask::new(
    id,                    // String: unique ID
    task_type,             // String: handler name
    data,                  // String: JSON payload
    max_retries,           // u32
    retry_after_hours,     // f64
    rate_limit_group,      // Option<String>
    max_per_minute,        // Option<u32>
    auto_dedupe,           // Option<bool>
    urgency_score,         // Option<f64>
    execute_at,            // Option<String> (RFC3339)
    cron,                  // Option<String>
    webhook_url,           // Option<String>
    max_execution_seconds, // Option<u64>
)
```

### `queue.start_dashboard(port)`

Starts the built-in React dashboard.

## Dashboard

```rust
queue.start_dashboard(9090);
// Open http://localhost:9090
```

## Relationship with Other Languages

`snerd-rust` writes to the same `.snerdata/tasks/tasks.log` format as every other SnerdMQ SDK, so storage is portable across the ecosystem. However, each storage file is exclusively owned by its single running instance — a second instance (Rust or otherwise) on the same path refuses to start. Polyglot setups give each service its own storage; cross-language execution happens via the task's `webhook_url` field, which the engine dispatches over HTTP with retries.
