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
