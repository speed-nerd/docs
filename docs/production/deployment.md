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

To share a queue across multiple servers, mount a **shared network volume** (AWS EFS, NFS, etc.):

```bash
# All 10 servers point to the same shared file
# OS-level file locking (flock) handles synchronization
./snerdmq /mnt/shared/snerd_tasks.log
```

### AWS ECS / Fargate

```yaml
# task-definition.json
{
  "containerDefinitions": [{
    "name": "app",
    "mountPoints": [{
      "sourceVolume": "efs-volume",
      "containerPath": "/mnt/shared"
    }],
    "environment": [{
      "name": "SNERD_STORAGE_PATH",
      "value": "/mnt/shared/snerd_tasks.log"
    }]
  }],
  "volumes": [{
    "name": "efs-volume",
    "efsVolumeConfiguration": {
      "fileSystemId": "fs-12345678"
    }
  }]
}
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          volumeMounts:
            - name: snerd-storage
              mountPath: /mnt/shared
          env:
            - name: SNERD_STORAGE_PATH
              value: /mnt/shared/snerd_tasks.log
      volumes:
        - name: snerd-storage
          persistentVolumeClaim:
            claimName: snerd-efs-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: snerd-efs-pvc
spec:
  accessModes:
    - ReadWriteMany  # Required for multi-node
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

## Storage Path Configuration

All SDKs accept a custom storage path:

=== "Node.js"
    ```typescript
    const queue = new SnerdQueue({
        storagePath: '/mnt/shared/snerd_tasks.log'
    });
    ```

=== "Python"
    ```python
    queue = SnerdQueue(storage_path='/mnt/shared/snerd_tasks.log')
    ```

=== "Go"
    ```go
    queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{
        StoragePath: "/mnt/shared/snerd_tasks.log",
    })
    ```

=== "Ruby"
    ```ruby
    queue = Snerdmq::SnerdQueue.new(storage_path: "/mnt/shared/snerd_tasks.log")
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

- [ ] Use a **shared volume** for multi-node deployments
- [ ] Set `SNERD_STORAGE_PATH` explicitly (don't rely on defaults)
- [ ] Ensure your container image includes the daemon binary
- [ ] Use `ReadWriteMany` access mode for PVCs in Kubernetes
- [ ] Monitor queue health via the dashboard or `/api/stats` endpoint
- [ ] Set up alerts for Dead Letter Queue events
- [ ] Configure graceful shutdown handlers in your application
