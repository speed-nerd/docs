---
sidebar_position: 9
---

# Hard Timeouts & Cancellation

When writing background tasks, there is always a risk that a third-party dependency hangs indefinitely or a developer accidentally introduces an infinite `while` loop. Without intervention, these "zombie" tasks will consume a worker thread forever, eventually starving the entire worker pool.

SnerdMQ solves this via **Hard Timeouts & Cancellation**.

## Overview

You can add a `maxExecutionSeconds` parameter when enqueuing a task. 

If the task takes longer than this duration to execute:
1. The SnerdMQ daemon forcefully terminates the worker's execution.
2. The task is immediately marked as **failed**.
3. If the task has remaining retries (`maxRetries`), it will follow normal backoff procedures and try again later. Otherwise, it will be sent to the Dead Letter Queue.

## Why it Matters

Protecting your background worker capacity is critical for production readiness. By enforcing hard upper limits on task duration, you prevent your application from grinding to a halt due to misbehaving API endpoints or inefficient code.

## Language SDK Behavior

Because different languages have different concurrency models, the way the SDK enforces the hard timeout locally varies slightly, but the Rust daemon always acts as the ultimate enforcer at the IPC level.

### Python / Asyncio
The Python SDK wraps your asynchronous handler in an `asyncio.wait_for`. If the timeout is reached, your handler is cancelled via an `asyncio.exceptions.TimeoutError`.

```python
await queue.enqueue(
    task_id='risky-task',
    task_type='process_data',
    data={},
    max_execution_seconds=300 # Kill if running > 5 mins
)
```

### Go
In Go, the SDK passes a `context.Context` to your handler. When the timeout expires, the context is automatically cancelled (`ctx.Done()` is closed). It is your responsibility to respect the context in your Go code (e.g. `req.WithContext(ctx)`).

```go
maxExec := 300
queue.Enqueue(snerd.RetryableTask{
    TaskId: "risky-task",
    TaskType: "process_data",
    TaskData: map[string]interface{}{},
    MaxExecutionSeconds: &maxExec,
})
```

### Node.js
Because JavaScript runs in a single-threaded event loop, the Node.js SDK cannot forcefully preempt a synchronous infinite `while (true)` loop. However, for asynchronous operations (like HTTP requests or database calls), the SDK will throw a `TimeoutError` and immediately report the failure back to the daemon.

```javascript
await queue.enqueue(
    'risky-task', 'process_data', {}, 
    3, 0, null, null, 0, null, null, 
    300 // max_execution_seconds
);
```

## Best Practices

* **Always set a timeout for network calls:** If your background job makes HTTP requests, use `maxExecutionSeconds` as a global safeguard, even if your HTTP client has its own timeout logic.
* **Keep it generous:** Hard timeouts should be a worst-case safety net. Set them to 3-5x the expected duration of the task to avoid flaking under temporary system load.
