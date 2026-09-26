# Worker Pools

By default, embedded SnerdMQ backends (`snerd-go` and `snerd-rust`) process tasks using a single global worker pool. While effective for general use cases, high-throughput systems often require resource isolation to ensure low-priority background jobs do not starve high-priority user-facing tasks. 

Worker Pools allow you to partition your execution resources. 

## How Worker Pools Work

When initializing your queue, you can define multiple named pools, each with its own concurrency limit. Tasks can then be assigned to a specific pool upon creation.

1. **Isolation:** A long-running task in the `background-pool` cannot consume workers allocated to the `urgent-pool`.
2. **Dedicated Semaphores:** Each pool operates its own concurrency semaphore.
3. **The Default Pool:** Any task created without a specific pool assignment will automatically fall back to the mandatory `"default"` pool.

## Code Examples

### Embedded Go (`snerd-go`)

In `snerd-go`, you define your pools using `snerd.NewAnyQueueWithPools`, providing a map of pool names to concurrency limits.

```go
package main

import (
	"fmt"
	"time"

	"github.com/speed-nerd/snerd-go"
)

func main() {
	// 1. Define worker pools (the "default" pool is required)
	pools := map[string]int{
		"default": 100, // General tasks
		"urgent":  50,  // Reserved for high-priority tasks
	}

	// 2. Initialize the queue with pools
	queue := snerd.NewAnyQueueWithPools("my-isolated-queue", pools, 1*time.Second)

	// 3. Register a task handler
	queue.RegisterTaskHandler("SEND_EMAIL", func(task snerd.SnerdTask) error {
		fmt.Printf("Processing %s in pool: %v\n", task.TaskType, task.Pool)
		return nil
	})

	// 4. Create an urgent task and assign it to the "urgent" pool
	urgentTask, _ := snerd.CreateTask("SEND_EMAIL", map[string]interface{}{"to": "vip@example.com"}, 3, 0.0)
	
	urgentPool := "urgent"
	urgentTask.Pool = &urgentPool // Assign to specific pool

	queue.EnqueueSnerdTask(urgentTask)
    
	// 5. Create a standard task (will fallback to "default")
	standardTask, _ := snerd.CreateTask("SEND_EMAIL", map[string]interface{}{"to": "user@example.com"}, 3, 0.0)
	queue.EnqueueSnerdTask(standardTask)

	select {}
}
```

### Embedded Rust (`snerd-rust`)

In `snerd-rust`, use `SnerdQueue::new_with_pools` and set the `pool` field on `RetryableTask`.

```rust
use std::collections::HashMap;
use std::sync::Arc;
use snerd_rust::queue::SnerdQueue;
use snerd_rust::file_store::FileStore;
use snerd_rust::rate_limiter::RateLimiter;
use snerd_rust::task::RetryableTask;

#[tokio::main]
async fn main() {
    let file_store = FileStore::new(&std::path::PathBuf::from("/tmp/snerd_data")).unwrap();
    let rate_limiter = RateLimiter::new(&std::path::PathBuf::from("/tmp/snerd_data"));

    // 1. Define worker pools
    let mut pools = HashMap::new();
    pools.insert("default".to_string(), 100);
    pools.insert("urgent".to_string(), 50);

    // 2. Initialize the queue with pools
    let queue = SnerdQueue::new_with_pools("my-queue", file_store, rate_limiter, pools);

    // 3. Register handler
    queue.register_task_handler("SEND_EMAIL", |task| {
        Box::pin(async move {
            println!("Processing in pool: {:?}", task.pool);
            Ok(())
        })
    }).await;

    // 4. Enqueue to a specific pool
    let mut urgent_task = RetryableTask::new(
        "task-1".to_string(),
        "SEND_EMAIL".to_string(),
        "{}".to_string(),
        3,
        1.0,
        None,
        None,
        None,
        None,
        None,
        None,
        None,
        None,
        Some("urgent".to_string()) // pool
    );
    queue.enqueue(urgent_task).unwrap();
    
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
}
```

## The Polyglot Worker Pools Broker

Worker pools shine when used alongside SnerdMQ's webhook features. You can build a system where a Go Producer sends webhooks to a stateless Rust worker, and the Rust worker ingests the HTTP payload directly into an isolated `snerd-rust` pool. This ensures that a surge of webhooks won't overwhelm your Rust backend, providing fault-tolerant execution and retry isolation!

**Rust (Worker Pools Broker Receiver):**

```rust
#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]
struct SnerdWebhookPayload {
    task_id: String,
    task_type: String,
    #[serde(rename = "data")]
    parameters: String, 
}

// The receiver places the webhook into an isolated snerd-rust pool!
async fn handle_send_push(
    State(queue): State<Arc<SnerdQueue>>,
    Json(payload): Json<SnerdWebhookPayload>
) -> StatusCode {
    
    let mut task = RetryableTask::new(
        payload.task_id,
        payload.task_type,
        payload.parameters,
        3, 1.0, None, None, None, None, None, None, None, None,
        Some("push-pool".to_string()) // 1. Enqueue it in an isolated pool for reliable execution
    );
    
    // 2. Acknowledge the HTTP request immediately
    match queue.enqueue(task) {
        Ok(_) => StatusCode::OK,
        Err(_) => StatusCode::INTERNAL_SERVER_ERROR
    }
}
```
