# Sharded Queues

SnerdMQ scales by **sharding**. Instead of having multiple workers pull from one massive central database, a single logical SnerdMQ queue is divided into multiple independent **shards** on disk. 

Each shard is exclusively owned and executed by exactly one daemon process.

## Why Sharding?

Traditional queues (like Redis, RabbitMQ, or PostgreSQL tables) suffer from **lock contention**: when 100 workers try to dequeue jobs simultaneously from one central list, they constantly block each other.

SnerdMQ bypasses this bottleneck entirely. 
Because each daemon exclusively owns its shards, **enqueue and dequeue operations require zero network coordination and zero distributed locking.** The daemon simply writes to its local shard's append-only log, yielding sub-millisecond latencies regardless of cluster size.

## Storage Layout

When you create a queue, SnerdMQ provisions a `.snerdata` directory containing the cluster's "brain" (`membership.json`) and the individual shard subdirectories:

```text
.snerdata/
├── .membership.lock      # OS-level flock protecting membership changes
├── membership.json       # JSON registry of shards and their current owners
├── shard-0/
│   ├── .lock             # Exclusive lock held by the daemon owning this shard
│   └── tasks/
│       └── tasks.log     # The actual append-only job log for shard-0
├── shard-1/              
...
```

## How Routing Works

When your application calls `queue.enqueue(...)`, the local SDK sends the payload to its local daemon. 
The daemon computes an `FNV-1a` hash of the `task_id` and modulo-divides it against the number of shards the daemon currently owns. 

The daemon writes the job to the correct shard's `tasks.log` directly. **There is no cross-instance network forwarding.** You must design your system such that every worker can execute every type of job.

## The Membership Protocol

Daemons do not communicate with each other over the network. They coordinate entirely through `membership.json` and OS-level file locking.

1. **Claiming**: On boot, a daemon opens `membership.json` (protected by `.membership.lock`) and looks for unowned shards. It writes its ID (`hostname@pid`) into the registry and sets a `lease_expiry` 30 seconds into the future.
2. **Heartbeat**: Every 10 seconds, the daemon updates its lease in `membership.json`.
3. **Takeover**: If a daemon crashes (e.g., `SIGKILL` or OOM kill), it stops updating its lease. After the lease expires (plus a configurable clock skew margin), another daemon in the cluster will lazily detect the lapsed lease, update `membership.json` to claim the shard, and take over processing its backlog.

## Scaling Out (Adding Shards)

To increase throughput, you can expand a live queue's shard count using the CLI.

```bash
# Add 2 more shards to an existing queue
snerdmq add-shards 2 /path/to/.snerdata
```

This immediately creates `shard-X` directories and bumps the shard count in `membership.json`. Running daemons will immediately spot the free shards and claim them.

!!! warning "Shard counts cannot decrease"
    In v1, you can increase the shard count but you cannot decrease it. Live rebalancing is not supported.

## Standby Mode

A daemon will claim shards up to its configured `max_local_shards` (default: 1). If a daemon boots and discovers that all shards in the cluster are already claimed by healthy peers, the daemon enters **Standby Mode**.

In Standby Mode:
- The daemon idles and polls `membership.json` every 10 seconds looking for lapsed leases.
- Any attempt to enqueue a task on a standby daemon will immediately throw an error: `[Snerd] No shards owned by this instance`.

## Handling Shutdowns

When you send a `SIGINT` or `SIGTERM` to your application, the SDK gracefully shuts down the daemon:
1. The daemon immediately stops accepting new enqueues.
2. It waits for any in-flight jobs currently executing in your SDK to finish (up to `drain_timeout`, default 30s).
3. It formally releases its claims in `membership.json`, allowing other daemons to instantly take over without waiting for the lease to expire.

### At-Least-Once Delivery Guarantee

If a daemon crashes ungracefully (e.g. `SIGKILL` or power failure) while a job is executing, that job is never marked as completed in the shard log.
When another daemon takes over the crashed daemon's shard, **it will re-execute the job**.

!!! danger "Idempotency Requirement"
    SnerdMQ guarantees **at-least-once** delivery. You **must** design your job handlers to be idempotent. If a job is executed twice due to a crash-takeover event, it should not corrupt your database or charge a customer twice.

## Embedded Sharding (snerd-rust & snerd-go)

If you are using the raw embedded engines (`snerd-rust` or `snerd-go`) instead of the `snerdmq` IPC daemon, you can still leverage the exact same distributed coordination engine using `SnerdShardedQueue`.

Embedded Sharding allows you to spin up multiple instances of your Go/Rust application on the same machine and have them automatically negotiate and partition the tasks among themselves. It uses the exact same `membership.json` and 2-phase `flock` algorithm used by the daemon.

!!! tip "Shared Worker Pools"
    When you use `ShardedQueue` in the embedded SDKs, the engine guarantees that your **Worker Pool Concurrency Limits** are enforced process-wide. If you allocate a `default` pool of 100 workers and claim 10 shards, the engine will safely share those 100 workers dynamically across all 10 shards without exploding your thread/goroutine counts!
