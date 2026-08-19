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

The daemon takes an **exclusive lock on its storage directory** at startup, so multiple replicas cannot process the same queue concurrently — a second daemon on the same storage refuses to start. This is deliberate: two processors on the same job log would race and double-execute jobs.

Scaling out therefore means **one queue per replica**, each with its own storage. Your load balancer routes requests across replicas, and every replica processes the jobs it enqueued:

=== "Node.js"
    ```typescript
    // Each replica runs its own daemon on its own storage (local disk works fine)
    const queue = new SnerdQueue({ storagePath: '/var/data/snerd' });
    ```

=== "Python"
    ```python
    # Each replica runs its own daemon on its own storage (local disk works fine)
    queue = SnerdQueue(storage_path='/var/data/snerd')
    ```

=== "Go"
    ```go
    // Each replica runs its own daemon on its own storage (local disk works fine)
    queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
        StoragePath: "/var/data/snerd",
    })
    ```

!!! warning "Multi-worker processes"
    The same lock applies *inside* a machine: with Gunicorn/Uvicorn workers, Node `cluster` forks, or Puma clustered workers, every worker is a separate process that spawns its own daemon. Give each worker its own `storagePath`, or run a single dedicated worker process for background jobs.

### Durable storage for a single instance

A shared network volume (AWS EFS, NFS) is still useful when **one** instance needs its queue state to survive restarts — e.g. a container that gets rescheduled but must keep its pending jobs:

```yaml
# Kubernetes: single-replica deployment with a durable volume
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-worker
spec:
  replicas: 1  # One daemon owns the storage; scale by sharding, not replicas
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          volumeMounts:
            - name: snerd-storage
              mountPath: /var/data/snerd
      volumes:
        - name: snerd-storage
          persistentVolumeClaim:
            claimName: snerd-pvc
```

For multi-replica services, prefer per-replica local storage (emptyDir or the container filesystem) — each replica's queue is independent by design.

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
