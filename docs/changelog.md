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

**Example (CLI Sharding):**
```bash
# Provision 10 shards on a shared EFS mount for your replicas to claim
snerdmq add-shards 10 /mnt/efs/snerd-queue
```

#### 2. Worker Pools (Resource Isolation)
You can now partition your queue execution resources to ensure high-throughput background jobs do not starve high-priority user-facing tasks.
- Create distinct concurrency limits for different pools (e.g., `default: 100`, `urgent: 50`, `ai_generation: 10`).
- **Shared Worker Pools:** When using Embedded Sharding, Snerd natively shares your worker pools across all claimed shards, ensuring your strict concurrency limits are enforced process-wide to prevent resource explosion.
- **Webhook Integration:** Use Worker Pools with Webhook tasks to build fault-tolerant Polyglot architectures (e.g., route Python tasks to a dedicated Rust worker pool).

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

#### 3. Job Chaining (DAGs)
You can now orchestrate complex workflows by defining dependencies between tasks using `trigger_after_ids`. 
- **Linear Pipelines:** Run tasks sequentially (Task A → Task B).
- **Fan-In:** Wait for multiple tasks to finish before triggering the final step (Task A, B, C → Task D).
- Tasks are automatically unblocked and executed the moment all of their parent dependencies succeed.

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

### 📦 Ecosystem Version Alignment
To support these features, the SnerdMQ ecosystem has been updated to the following versions:
- **Core Embedded Engines** (`snerd-rust`, `snerd-go`): `v0.3.0`
- **SnerdMQ IPC Daemon**: `v0.3.0`
- **Language SDKs**: Updated to support the new features. Depending on the SDK, these were bumped to `v0.4.0` (Node, Python, Ruby, PHP) or `v1.1.0` (Java, C#, Go SDKs).
