# Node.js / TypeScript SDK

The official Node.js & TypeScript SDK for SnerdMQ. Written in 100% TypeScript with full type safety.

[![npm version](https://img.shields.io/npm/v/snerdmq-node)](https://www.npmjs.com/package/snerdmq-node)

## Installation

```bash
npm install snerdmq-node
```

The post-install script automatically downloads the correct Rust daemon binary for your OS (macOS, Linux, or Windows).

## Quickstart

```typescript
import { SnerdQueue } from 'snerdmq-node';

// 1. Initialize — spawns the Rust daemon in the background
const queue = new SnerdQueue();

// 2. Register a handler for your job type
queue.registerHandler('send_email', async (data) => {
    console.log(`Sending email to ${data.to}...`);
    // Your business logic here
});

// 3. Enqueue a job
queue.enqueue({
    id: `email-${Date.now()}`,
    type: 'send_email',
    data: { to: 'user@example.com', subject: 'Welcome!' },
    maxRetries: 3,
    retryAfter: 0.5,  // Retry after 30 minutes on failure
});

// 4. (Optional) Handle permanently failed tasks
queue.registerMaxRetryHandler('send_email', async (data) => {
    console.error(`Failed after all retries: ${JSON.stringify(data)}`);
});

// 5. Clean shutdown (Required)
process.on('SIGINT', () => {
    console.log("Shutting down SnerdMQ...");
    queue.shutdown();
    process.exit(0);
});
```

## API Reference

### `new SnerdQueue(options?)`

Creates a new queue instance and spawns the background daemon.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `storagePath` | `string` | `.snerdata` | Path to the queue storage directory (the task log lives at `<path>/tasks/tasks.log`) |

### `queue.registerHandler(type, handler)`

Registers an async handler function for a task type.

```typescript
queue.registerHandler('task_type', async (data: any) => {
    // Process the task
    // Throw an error to trigger retry
});
```

### `queue.enqueue(task)`

Enqueues a new background job.

```typescript
queue.enqueue({
    id: string,           // Unique task ID
    type: string,         // Task type (matches a registered handler)
    data: object,         // JSON-serializable payload
    maxRetries?: number,  // Max retry attempts (default: 0)
    retryAfter?: number,  // Backoff in hours (default: 0)
    rateLimitGroup?: string,  // Rate limit group name
    maxPerMinute?: number,    // Max executions per minute
    autoDedupe?: boolean,     // Drop duplicate payloads
    urgencyScore?: number,    // Priority (0.0–1.0)
    executeAt?: string | Date, // Future execution time
    cron?: string,            // Cron schedule
    webhookUrl?: string,      // HTTP webhook URL
    maxExecutionSeconds?: number, // Hard timeout
});
```

### `queue.registerMaxRetryHandler(type, handler)`

Registers a handler for tasks that have permanently failed (Dead Letter Queue).

### `queue.startDashboard(port?)`

Starts the built-in React dashboard on the specified port (default: 9090).

### `queue.yieldProgress(message)`

Streams a progress update from within a handler to the dashboard.

### `queue.shutdown()`

Gracefully kills the background daemon process.

## Dashboard

```typescript
const queue = new SnerdQueue();
queue.startDashboard(9090);
// Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```typescript
const queue = new SnerdQueue();

// Two job types sharing the same queue, daemon, and dashboard
queue.registerHandler('process_image', async (data) => {
    console.log(`Processing image: ${data.image_id}`);
});
queue.registerHandler('send_otp_email', async (data) => {
    console.log(`Sending OTP to: ${data.to}`);
});

queue.startDashboard(8080); // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — e.g. with Node's `cluster` module, every forked worker needs its own `storagePath`.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```typescript
const images = new SnerdQueue({ storagePath: '.snerdata-images' });
const emails = new SnerdQueue({ storagePath: '.snerdata-emails' });

images.startDashboard(8080);
emails.startDashboard(8081);
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```typescript
// Each server runs its own daemon on its own storage dir (local disk works fine)
const queue = new SnerdQueue({
    storagePath: '/var/data/snerd'
});
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.


## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```typescript
import { SnerdQueue } from 'snerdmq-node';

const queue = new SnerdQueue({ storagePath: './.snerdata' });

function initEmailWorkers() {
    queue.registerHandler('send_email', async (data) => {
        console.log(`Sending email to ${data.email}...`);
    });
    queue.registerMaxRetryHandler('send_email', async (payload) => {
        // Payload includes taskId, taskType, and data
        console.log(`Email to ${payload.data.email} failed permanently. Dead letter processing...`);
    });
}

function initImageWorkers() {
    queue.registerHandler('process_image', async (data) => {
        console.log(`Processing image ${data.imageId}...`);
    });
}

async function main() {
    initEmailWorkers();
    initImageWorkers();

    queue.startDashboard(8080);

    // Keep process alive and handle shutdown
    process.on('SIGINT', async () => {
        await queue.shutdown();
        process.exit(0);
    });
    process.on('SIGTERM', async () => {
        await queue.shutdown();
        process.exit(0);
    });
}

main();
```
