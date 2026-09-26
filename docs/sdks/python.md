# Python SDK

The official Python SDK for SnerdMQ. Built for modern `asyncio` applications (FastAPI, Sanic, etc.).

[![PyPI version](https://img.shields.io/pypi/v/snerdmq-python)](https://pypi.org/project/snerdmq-python/)

## Installation

**1. Install the package:**
```bash
pip install snerdmq-python
```

**2. Download the Rust engine:**
```bash
snerdmq-install
```

## Quickstart

```python
import asyncio
from snerdmq import SnerdQueue

async def send_email(data):
    print(f"Sending email to {data['to']}...")
    # Your business logic here

async def main():
    queue = SnerdQueue()
    queue.register_handler('send_email', send_email)

    await queue.enqueue(
        task_id='email-123',
        task_type='send_email',
        data={'to': 'user@example.com', 'subject': 'Welcome!'},
        max_retries=3,
        retry_after_hours=0.5,
    )

    # Handle permanently failed tasks
    async def handle_failed(data):
        print(f"Failed after all retries: {data}")
    queue.register_max_retry_handler('send_email', handle_failed)

    try:
        await queue.start_listening()
    except asyncio.CancelledError:
        pass
    finally:
        print("Shutting down SnerdMQ...")
        await queue.shutdown()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
```

## API Reference

### `SnerdQueue(storage_path=None)`

Creates a new queue instance and spawns the background daemon.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `storage_path` | `str` | `.snerdata` | Path to the queue storage directory (the task log lives at `<path>/tasks/tasks.log`) |

### `queue.register_handler(task_type, handler)`

Registers an async handler function for a task type.

```python
async def my_handler(data):
    # Process the task
    # Raise an exception to trigger retry
queue.register_handler('task_type', my_handler)
```

### `await queue.enqueue(...)`

Enqueues a new background job.

```python
await queue.enqueue(
    task_id='unique-id',
    task_type='task_type',
    data={'key': 'value'},
    max_retries=3,
    retry_after_hours=0.5,
    rate_limit_group='api_group',
    max_per_minute=50,
    auto_dedupe=True,
    urgency_score=0.9,
    execute_at='2026-12-31T23:59:00Z',
    cron='0 8 * * *',
    webhook_url='https://example.com/webhook',
    max_execution_seconds=300,
)
```

### `queue.register_max_retry_handler(task_type, handler)`

Registers a handler for permanently failed tasks (Dead Letter Queue).

### `queue.start_dashboard(port=9090)`

Starts the built-in React dashboard.

### `await queue.yield_progress_async(message)`

Streams a progress update from within an async handler.

### `queue.yield_progress(message)`

Sync variant for fire-and-forget progress updates.

## Dashboard

```python
queue = SnerdQueue()
queue.start_dashboard(9090)
# Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```python
queue = SnerdQueue()

# Two job types sharing the same queue, daemon, and dashboard
async def process_image(data):
    print(f"Processing image: {data['image_id']}")

async def send_otp_email(data):
    print(f"Sending OTP to: {data['to']}")

queue.register_handler('process_image', process_image)
queue.register_handler('send_otp_email', send_otp_email)

queue.start_dashboard(8080)  # one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — with Gunicorn/Uvicorn multi-worker setups, every worker needs its own `storage_path`.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```python
images = SnerdQueue(storage_path='.snerdata-images')
emails = SnerdQueue(storage_path='.snerdata-emails')

images.start_dashboard(8080)
emails.start_dashboard(8081)
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```python
# Each server runs its own daemon on its own storage dir (local disk works fine)
queue = SnerdQueue(storage_path='/var/data/snerd')
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.




## Advanced Orchestration (v0.3.0 Features)

SnerdMQ v0.3.0 introduced powerful new primitives for managing complex background jobs. Below are realistic, production-like scenarios showing how to utilize these features in Python:

```python
# 1. Sharded Queues
# Context: A developer needs to scale their deployment across 4 servers to handle massive load, but they don't want to use Redis.
# How to use: Tell the daemon how many shards to claim on boot. The queue handles the OS-level locking automatically.

queue = SnerdQueue(max_local_shards=4)
```

```python
# 2. Worker Pools
# Context: A system has both slow AI generation tasks and fast transactional emails. We want to prevent AI tasks from starving the email workers.

# Enqueue an AI task to a dedicated pool
await queue.enqueue(
    task_id='ai-gen-123',
    task_type='ai_generation',
    data={'prompt': 'A majestic horse'},
    pool='ai-pool'
)

# Enqueue an email task to a fast, urgent pool
await queue.enqueue(
    task_id='email-123',
    task_type='send_email',
    data={'to': 'user@example.com'},
    pool='urgent'
)
```

```python
# 3. Job Chaining (DAGs)
# Context: A video processing pipeline where a video must be transcoded, then uploaded to S3, and finally an email notification must be sent.

# Step 1: Transcode
await queue.enqueue(
    task_id='transcode-1',
    task_type='transcode_video',
    data={'file': 'raw.mp4'}
)

# Step 2: Upload (Waits for Step 1)
await queue.enqueue(
    task_id='upload-1',
    task_type='upload_s3',
    data={'file': 'processed.mp4'},
    trigger_after_ids=['transcode-1']
)

# Step 3: Notify (Waits for Step 2)
await queue.enqueue(
    task_id='notify-1',
    task_type='send_email',
    data={'status': 'done'},
    trigger_after_ids=['upload-1']
)
```

```python
# 4. Cron & Scheduled Jobs
# Context: A system needs to run a database cleanup script every night at midnight.

await queue.enqueue(
    task_id='db-cleanup',
    task_type='cleanup_job',
    data={'table': 'sessions'},
    cron='0 0 * * *'
)
```

```python
# 5. Hard Timeouts
# Context: A background worker is making an HTTP request to a flaky third-party API that might hang indefinitely. We forcefully kill it if it runs over 5 minutes.

await queue.enqueue(
    task_id='api-fetch-1',
    task_type='fetch_data',
    data={'endpoint': '/sync'},
    max_execution_seconds=300
)
```

```python
# 6. Webhook Callbacks
# Context: A developer is using AWS Lambda or Vercel Serverless functions and wants SnerdMQ to trigger the function via an HTTP POST request rather than running a local worker.

await queue.enqueue(
    task_id='serverless-job',
    task_type='resize_image',
    data={'img': 'cat.jpg'},
    webhook_url='https://api.example.com/webhook/snerdmq'
)
```

```python
# 7. The Dead Letter Queue (DLQ)
# Context: A task has failed its maximum number of retries (e.g., the SendGrid API is down for hours). The developer needs to catch this to alert the team on Slack.

async def alert_slack_func(data):
    print(f"Task permanently failed! Alerting Slack with data: {data}")

queue.register_max_retry_handler('send_email', alert_slack_func)
```

## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```python
import asyncio
from snerdmq import SnerdQueue

queue = SnerdQueue(storage_path="./.snerdata")

async def send_email(data):
    print(f"Sending email to {data['email']}...")

async def dlq_send_email(data):
    print(f"Email to {data['email']} failed permanently. Dead letter processing...")

async def process_image(data):
    print(f"Processing image {data['imageId']}...")

def init_workers():
    queue.register_handler('send_email', send_email)
    queue.register_max_retry_handler('send_email', dlq_send_email)
    queue.register_handler('process_image', process_image)

async def main():
    init_workers()
    queue.start_dashboard(8080)
    await queue.start_listening()

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        # Gracefully shut down on Ctrl+C
        queue.shutdown()
```
