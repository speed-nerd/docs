# Deployment Guide

SnerdMQ runs as an embedded sidecar daemon — not a standalone server. This fundamentally changes how you deploy it compared to Redis or RabbitMQ.

## Single-Node Deployment

On a single server (VPS, EC2, Droplet), SnerdMQ writes to a local file. Zero configuration needed:

```bash
# Just install the SDK and run — the daemon spawns automatically
npm install snerdmq-node
node app.js
```

The queue file is created at `.snerdata/tasks/tasks.log` relative to your process working directory.

## Docker Deployment

Since SnerdMQ runs over stdio pipes (not TCP), you package the daemon binary **inside** your application container:

```dockerfile
FROM node:20-alpine

# Copy the SnerdMQ binary into your app container
COPY --from=ghcr.io/speed-nerd/snerdmq:latest /bin/snerdmq /usr/local/bin/snerdmq

# Your app's SDK spawns the daemon automatically
COPY package.json package-lock.json ./
RUN npm install
COPY . .

CMD ["node", "app.js"]
```

!!! note
    Do NOT deploy SnerdMQ as a standalone container. It's a child process, not a microservice.

### Docker Compose

```yaml
services:
  app:
    build: .
    volumes:
      - queue-data:/app/.snerdata
    deploy:
      replicas: 1  # Single replica uses local storage

volumes:
  queue-data:
```

## Distributed Deployment (Multi-Node)

SnerdMQ is designed to scale horizontally across multiple instances (e.g., Kubernetes pods or EC2 instances) while acting as a single, unified queue. This is achieved via **Sharding**.

Instead of giving each replica its own isolated storage directory, all replicas point to the **exact same shared storage directory** (e.g., an AWS EFS mount). SnerdMQ automatically divides the queue into shards and negotiates ownership between the active replicas using OS-level file locking and a central `membership.json` file.

=== "Node.js"
    ```typescript
    // All replicas use the exact same shared EFS mount
    const queue = new SnerdQueue({ storagePath: '/mnt/efs/snerd-queue' });
    ```

=== "Python"
    ```python
    # All replicas use the exact same shared EFS mount
    queue = SnerdQueue(storage_path='/mnt/efs/snerd-queue')
    ```

=== "Go"
    ```go
    // All replicas use the exact same shared EFS mount
    queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
        StoragePath: "/mnt/efs/snerd-queue",
    })
    ```

### How to Scale

1. **Create the shared storage**: Mount a POSIX-compliant distributed filesystem (like AWS EFS) to your containers.
2. **Provision Shards**: By default, a queue has 1 shard. Increase the shard count to match or exceed your desired peak replica count using the CLI:
   ```bash
   snerdmq add-shards 10 /mnt/efs/snerd-queue
   ```
3. **Deploy Replicas**: As you spin up new containers, they will inspect the `membership.json` file in the shared directory, spot the unowned shards, and automatically claim them.

If a replica crashes or scales down, its lease on the shard will expire, and another healthy replica will automatically take over the abandoned shard.

!!! warning "Idempotency and Filesystems"
    Cross-server sharding requires strict handler **idempotency** (due to crash takeovers) and a **POSIX-compliant distributed filesystem** (like EFS, not S3). 
    **Read the full [Cross-Server Deployment Guide](cross-server.md) before deploying.**

### Multi-Worker Processes (Single Machine)

If you are running a multi-process architecture on a *single* machine (e.g., Node `cluster`, Gunicorn workers, Puma), you can apply the exact same sharding pattern using your local disk:

1. Run `snerdmq add-shards 4 .snerdata`
2. Start your 4 Gunicorn workers pointing at `.snerdata`.
3. The 4 workers will automatically negotiate and claim 1 shard each, running 4 parallel queue executors safely on the same local directory.

## Storage Path Configuration

All SDKs accept a custom storage path — use it to isolate queues per workload (each path gets its own daemon, job log, and dashboard):

=== "Node.js"
    ```typescript
    const queue = new SnerdQueue({
        storagePath: '/var/data/snerd-emails'
    });
    ```

=== "Python"
    ```python
    queue = SnerdQueue(storage_path='/var/data/snerd-emails')
    ```

=== "Go"
    ```go
    queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
        StoragePath: "/var/data/snerd-emails",
    })
    ```

=== "Ruby"
    ```ruby
    queue = Snerdmq::SnerdQueue.new(storage_path: "/var/data/snerd-emails")
    ```

## Binary Installation

The SDKs handle binary downloads automatically, but you can also install manually:

**From Cargo (Rust):**
```bash
cargo install snerdmq
```

**From GitHub Releases:**
Download the pre-built binary for your platform from [github.com/speed-nerd/snerdmq/releases](https://github.com/speed-nerd/snerdmq/releases).

| Platform | Binary |
|----------|--------|
| macOS (Apple Silicon) | `snerdmq-aarch64-apple-darwin` |
| macOS (Intel) | `snerdmq-x86_64-apple-darwin` |
| Linux (x86_64) | `snerdmq-x86_64-unknown-linux-gnu` |
| Linux (aarch64) | `snerdmq-aarch64-unknown-linux-gnu` |
| Windows (x86_64) | `snerdmq-x86_64-pc-windows-msvc.exe` |

## Production Checklist

- [ ] One queue (daemon) per storage directory — register all job types on it
- [ ] For multi-replica services: each replica gets its own storage; scale by sharding
- [ ] Use shared volumes (EFS/NFS) only for single-instance durable state
- [ ] Set an explicit storage path in production (don't rely on defaults)
- [ ] Ensure your container image includes the daemon binary
- [ ] Monitor queue health via the dashboard or `/api/stats` endpoint
- [ ] Set up alerts for Dead Letter Queue events
- [ ] Configure graceful shutdown handlers in your application
