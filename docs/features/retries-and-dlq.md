# Retries & Dead Letter Queue

SnerdMQ provides automatic retry with configurable backoff for failed tasks. When a task permanently fails (exhausts all retries), it lands in the Dead Letter Queue (DLQ) where you can inspect, alert, or reprocess it.

!!! note "Delivery semantics"
    SnerdMQ provides **at-least-once** delivery. In rare cases — e.g. if the daemon is killed while a task is executing — a task may be executed again after restart. Make your handlers idempotent.

## How Retries Work

When a task handler throws an error (exception, rejected promise, or non-zero exit), SnerdMQ:

1. **Catches the failure** at the IPC boundary
2. **Increments the retry counter** on the task
3. **Reschedules the task** with a `retry_after` backoff (in hours)
4. **Re-dispatches** the task when the backoff period elapses

If the handler succeeds on retry, the task is marked complete and removed from the queue. If it fails again, the cycle repeats until `max_retries` is exhausted.

## Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_retries` | int | `0` | Maximum number of retry attempts before moving to DLQ |
| `retry_after` (hours) | float | `0.0` | Hours to wait before retrying a failed task |

## Code Examples

=== "Node.js"

    ```typescript
    // Enqueue with retry config
    queue.enqueue({
        id: `email-${Date.now()}`,
        type: 'send_email',
        data: { to: 'user@example.com' },
        maxRetries: 3,
        retryAfter: 0.5,  // Wait 30 minutes before retrying
    });

    // Handle permanently failed tasks
    queue.registerMaxRetryHandler('send_email', async (data) => {
        console.error(`Failed after all retries: ${JSON.stringify(data)}`);
        // Alert your team, update database, send Slack message...
    });
    ```

=== "Python"

    ```python
    await queue.enqueue(
        task_id='email-123',
        task_type='send_email',
        data={'to': 'user@example.com'},
        max_retries=3,
        retry_after_hours=0.5,  # 30 minutes
    )

    async def handle_failed_email(data):
        print(f"Failed after all retries: {data}")

    queue.register_max_retry_handler('send_email', handle_failed_email)
    ```

=== "Go"

    ```go
    queue.Enqueue(
        "email-123", "send_email",
        map[string]interface{}{"to": "user@example.com"},
        3,    // max retries
        0.5,  // retry after hours
        nil, nil, nil, nil, nil, nil, nil,
    )

    queue.RegisterMaxRetryHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
        fmt.Printf("Failed after all retries: %v\n", data)
        return nil
    })
    ```

=== "Ruby"

    ```ruby
    queue.enqueue(
      task_id: "email-123",
      task_type: "send_email",
      data: { "to" => "user@example.com" },
      max_retries: 3,
      retry_after_hours: 0.5,
    )

    queue.register_max_retry_handler('send_email') do |data|
      puts "Failed after all retries: #{data.inspect}"
    end
    ```

=== "PHP"

    ```php
    $queue->enqueue(
        "email-123", "send_email",
        ["to" => "user@example.com"],
        3,    // max retries
        0.5   // retry after hours
    );

    $queue->registerMaxRetryHandler('send_email', function($data) {
        echo "Failed after all retries: " . json_encode($data) . "\n";
    });
    ```

=== "Java"

    ```java
    queue.enqueue(
        "email-123", "send_email",
        "{\"to\":\"user@example.com\"}",
        3,    // max retries
        0.5,  // retry after hours
        null, null, null, null, null, null, null, null
    );

    queue.registerMaxRetryHandler("send_email", data -> {
        System.out.println("Failed after all retries: " + data);
    });
    ```

=== "C# / .NET"

    ```csharp
    await queue.Enqueue(
        taskId: "email-123",
        taskType: "send_email",
        jsonData: "{\"to\":\"user@example.com\"}",
        maxRetries: 3,
        retryAfterHours: 0.5
    );

    queue.RegisterMaxRetryHandler("send_email", (data) => {
        Console.WriteLine($"Failed after all retries: {data}");
    });
    ```

## Dead Letter Queue (DLQ)

When a task exhausts its `max_retries`, SnerdMQ moves it to the Dead Letter Queue — a persistent store of permanently failed tasks. The DLQ:

- **Persists to disk** — DLQ entries survive process restarts
- **Triggers your handler** — `registerMaxRetryHandler` fires immediately when a task enters the DLQ
- **Stores the full payload** — you get the original task data so you can diagnose or reprocess

## Cron + Retry Interaction

If a cron job fails, it temporarily switches to retry mode using `retry_after` backoff. Once it succeeds again, it goes back to its normal cron schedule:

```
Cron tick → Execute → FAIL → retry_after backoff → Retry → FAIL → retry_after backoff → Retry → SUCCESS → Back to cron schedule
```

This ensures scheduled jobs self-heal from transient failures without losing their schedule.
