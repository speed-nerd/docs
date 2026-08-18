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

    await queue.start_listening()

if __name__ == "__main__":
    asyncio.run(main())
```

## API Reference

### `SnerdQueue(storage_path=None)`

Creates a new queue instance and spawns the background daemon.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `storage_path` | `str` | `.snerdata/tasks/tasks.log` | Path to the queue storage file |

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

## Distributed Scaling

```python
queue = SnerdQueue(storage_path='/mnt/aws-efs-shared-drive/snerd_tasks.log')
```
