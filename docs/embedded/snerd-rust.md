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


## Advanced Orchestration (v0.3.0 Features)

SnerdMQ v0.3.0 introduced powerful new primitives for managing complex background jobs natively in the core engine. Below are examples of how to utilize these features when embedding SnerdMQ in Rust:

### 🍕 Sharded Queues (Scaling Out)

SnerdMQ natively supports distributed execution across multiple servers while acting as a single logical queue. Just mount a shared storage drive (like AWS EFS) and boot multiple daemons. They will automatically lock and negotiate ownership of shards. Just tell the queue how many shards to claim on boot.

```rust
// Boot a multi-tenant daemon that owns up to 4 shards locally
let queue = Arc::new(SnerdQueue::new("my-queue", file_store, rate_limiter));
```

### 🏊 Worker Pools

```rust
// Route an AI task to a dedicated pool
let mut task = RetryableTask::new("ai-1", "ai_generation", r#"{"prompt":"horse"}"#.to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None);
task.pool = Some("ai-pool".to_string());
queue.enqueue(task)?;

// Route an email task to a fast, urgent pool
let mut task2 = RetryableTask::new("email-1", "send_email", r#"{"to":"user@a.com"}"#.to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None);
task2.pool = Some("urgent".to_string());
queue.enqueue(task2)?;
```

### 🔗 Job Chaining (DAGs)

```rust
// Step 1: Transcode
queue.enqueue(RetryableTask::new("transcode-1", "transcode_video", r#"{"file":"raw.mp4"}"#.to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None))?;

// Step 2: Upload (Waits for Step 1)
let mut next_task = RetryableTask::new("upload-1", "upload_s3", r#"{"file":"out.mp4"}"#.to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None);
next_task.trigger_after_ids = Some(vec!["transcode-1".to_string()]);
queue.enqueue(next_task)?;
```

### 🕒 Cron & Scheduled Jobs

```rust
// Run every day at 08:00
let mut cron_task = RetryableTask::new("digest", "email", "{}".to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None);
cron_task.cron_expression = Some("0 8 * * *".to_string());
queue.enqueue(cron_task)?;
```

### 🛑 Hard Timeouts

```rust
// Forcefully kill if running > 5 mins
let mut risky_task = RetryableTask::new("risky", "fetch", "{}".to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None);
risky_task.max_execution_seconds = Some(300);
queue.enqueue(risky_task)?;
```

### 🌐 Webhook Callbacks

```rust
// Execute via HTTP instead of local handlers
let mut hook_task = RetryableTask::new("serverless", "resize", "{}".to_string(), 3, 0.0, None, None, None, None, None, None, None, None, None, None);
hook_task.webhook_url = Some("https://api.example.com/webhook".to_string());
queue.enqueue(hook_task)?;
```

*Built with ❤️ for John Wick tier engineering.*

