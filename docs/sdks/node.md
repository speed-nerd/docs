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

// 5. Clean shutdown
process.on('SIGINT', () => {
    queue.shutdown();
    process.exit(0);
});
```

## API Reference

### `new SnerdQueue(options?)`

Creates a new queue instance and spawns the background daemon.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `storagePath` | `string` | `.snerdata/tasks/tasks.log` | Path to the queue storage file |

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

## Distributed Scaling

```typescript
const queue = new SnerdQueue({
    storagePath: '/mnt/aws-efs-shared-drive/snerd_tasks.log'
});
```
