# Changelog & Releases

This document tracks the major feature releases and version updates across the SnerdMQ ecosystem (including the Core Engines, IPC Daemon, and Language SDKs).

---

## 🚀 v0.3.0 (Engines) & SDK Updates
**Release Date:** September 2026

This release introduces massive scalability improvements, transforming SnerdMQ from a single-node queue into a highly concurrent, distributed background job architecture—without sacrificing its "zero infrastructure" philosophy.

### 🌟 New Features

#### 1. Sharded Queues (Distributed Scaling)
SnerdMQ can now scale horizontally across multiple instances while acting as a single, unified queue. 
- **Daemon Sharding:** Deploy multiple `snerdmq` daemons pointing to the same shared network directory (e.g., AWS EFS). They will automatically partition the workload using a robust `membership.json` lease and 2-phase OS flocking protocol.
- **Embedded Sharding (`snerd-rust` & `snerd-go`):** You can now natively shard workloads directly within the raw embedded engines across multiple processes on the same machine, safely dividing tasks without network hops.

**Example (SDK - Booting a Sharded Daemon):**
```python
# Boot a multi-tenant daemon that owns up to 4 shards locally
queue = SnerdQueue(max_local_shards=4)
```

**Example (Embedded Sharding in Rust):**
```rust
// Create a 10-shard queue safely embedded inside your Rust process
let sharded_queue = SnerdShardedQueue::new("main", Path::new(".snerdata"), 10).await;

// Enqueue jobs natively (tasks are instantly partitioned)
sharded_queue.enqueue(task).await.unwrap();
```

#### 2. Worker Pools (Resource Isolation)
You can now partition your queue execution resources to ensure high-throughput background jobs do not starve high-priority user-facing tasks.

**Example:**
```python
# Dedicate a task to the 'urgent' worker pool
await queue.enqueue(
    task_id='payment-123',
    task_type='process_payment',
    data={'amount': 100},
    pool='urgent'
)
```
*Run the daemon with `SNERD_POOLS="default:100,urgent:50"` to allocate workers.*

#### 3. Job Chaining & Workflows (DAGs)
You can now orchestrate complex workflows by defining dependencies between tasks using `trigger_after_ids`. Tasks are automatically unblocked and executed the moment all of their parent dependencies succeed.

**Example (Fan-In Chaining):**
```python
# Block this task from executing until both parent tasks successfully finish
await queue.enqueue(
    task_id='final-report',
    task_type='generate_pdf',
    data={'report_id': 99},
    trigger_after_ids=['data-fetch-1', 'data-fetch-2']
)
```

#### 4. 🕒 Cron & Scheduled Jobs (Delayed Execution)
Adding the ability to schedule jobs in the future.

**Example:**
```python
# Run every day at 08:00
await queue.enqueue(
    task_id='daily-digest',
    task_type='send_email',
    data={'template': 'daily'},
    cron='0 8 * * *'
)
```

#### 5. 🛑 Hard Timeouts & Cancellation
Protect developers from their own bad code taking down the entire worker pool.

**Example:**
```python
# Forcefully kill if running > 5 mins
await queue.enqueue(
    task_id='risky-task',
    task_type='process_data',
    data={},
    max_execution_seconds=300 
)
```

#### 6. 🪦 The Dead Letter Queue (DLQ) & Replay
Introduce a formal "Dead Letter" storage state. Grab permanently failed jobs and inject them back into the active queue.

**Example:**
```python
# Catch tasks that have permanently failed (Dead Letter Queue)
async def handle_failed_email(data):
    print(f"Email task failed after all retries! Data: {data}")

queue.register_max_retry_handler('send_email', handle_failed_email)
```
*(CLI replay command: `snerdmq replay --type="send_email"`)*

#### 7. 🌐 Webhook Callbacks
Instead of executing a local codebase, SnerdMQ can make a POST request to a provided URL when it's time to process the job. Unlocks integration with Serverless platforms.

**Example:**
```python
# Execute via HTTP instead of local handlers
await queue.enqueue(
    task_id='serverless-task',
    task_type='resize_image',
    data={'img': 'cat.jpg'},
    webhook_url='https://api.example.com/webhooks/snerdmq'
)
```

#### 8. 🔒 Dashboard Authentication & Controls
Add Basic Auth / JWT login to the Rust dashboard, and add action buttons to manually Pause, Cancel, or Retry jobs directly from the browser interface.

**Example:**
```python
# Secure the live dashboard with Basic Auth credentials
queue.start_dashboard(
    port=8080,
    auth_username="admin",
    auth_password="supersecretpassword"
)
```

### 📦 Ecosystem Version Alignment
To support these features, the SnerdMQ ecosystem has been updated to the following versions:
- **Core Embedded Engines** (`snerd-rust`, `snerd-go`): `v0.3.0`
- **SnerdMQ IPC Daemon**: `v0.3.0`
- **Language SDKs**: Updated to support the new features. Depending on the SDK, these were bumped to `v0.4.1` (Node, Python, Ruby, PHP) or `v1.1.1` (Java, C#, Go SDKs).
