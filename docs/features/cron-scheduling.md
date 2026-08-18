# Cron Scheduling

SnerdMQ supports cron expressions for recurring background jobs. Schedule tasks to run on fixed intervals — hourly, daily, weekly, or any custom schedule — without external cron daemons or separate scheduler libraries.

## How It Works

A cron job in SnerdMQ is a **repeatable task** that re-executes on a fixed schedule **after each successful run**:

```
Schedule tick → Execute → SUCCESS → Wait for next schedule tick → Execute → ...
```

If a cron job **fails**, it temporarily switches to retry mode using `retry_after` backoff. Once it recovers, it returns to its normal cron schedule.

## Supported Formats

### Standard Cron Expressions

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
* * * * *
```

**Examples:**

| Expression | Meaning |
|------------|---------|
| `0 * * * *` | Every hour, on the hour |
| `0 8 * * *` | Every day at 8:00 AM |
| `0 8 * * 1` | Every Monday at 8:00 AM |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 1 * *` | First day of every month at midnight |
| `0 9 * * 1-5` | Weekdays at 9:00 AM |

### Shorthand Notation

SnerdMQ also supports convenient shorthand intervals:

| Shorthand | Equivalent | Meaning |
|-----------|------------|---------|
| `10m` | `*/10 * * * *` | Every 10 minutes |
| `2h` | `0 */2 * * *` | Every 2 hours |
| `1d` | `0 0 * * *` | Every day at midnight |

## Code Examples

=== "Node.js"

    ```typescript
    // Run a daily digest at 8:00 AM
    queue.enqueue({
        id: 'daily-digest',
        type: 'send_digest',
        data: { recipients: 'all-users' },
        cron: '0 8 * * *',
    });

    // Health check every 5 minutes
    queue.enqueue({
        id: 'health-check',
        type: 'check_services',
        data: {},
        cron: '*/5 * * * *',
    });

    // Using shorthand
    queue.enqueue({
        id: 'cleanup',
        type: 'cleanup_temp_files',
        data: {},
        cron: '2h',  // Every 2 hours
    });
    ```

=== "Python"

    ```python
    # Run a daily digest at 8:00 AM
    await queue.enqueue(
        task_id='daily-digest',
        task_type='send_digest',
        data={'recipients': 'all-users'},
        cron='0 8 * * *',
    )

    # Health check every 5 minutes
    await queue.enqueue(
        task_id='health-check',
        task_type='check_services',
        data={},
        cron='*/5 * * * *',
    )
    ```

=== "Go"

    ```go
    cronExpr := "0 8 * * *"
    queue.Enqueue(
        "daily-digest", "send_digest",
        map[string]interface{}{"recipients": "all-users"},
        3, 0.0, "", 0,
        nil, nil, nil,
        &cronExpr,  // cron schedule
        nil, nil,
    )
    ```

=== "Ruby"

    ```ruby
    queue.enqueue(
      task_id: "daily-digest",
      task_type: "send_digest",
      data: { "recipients" => "all-users" },
      cron: "0 8 * * *",
    )
    ```

=== "PHP"

    ```php
    $queue->enqueue(
        "daily-digest", "send_digest",
        ["recipients" => "all-users"],
        3, 0.0, null, null,
        null, null, null,
        "0 8 * * *",  // cron
        null, null
    );
    ```

=== "Java"

    ```java
    queue.enqueue(
        "daily-digest", "send_digest",
        "{\"recipients\":\"all-users\"}",
        3, 0.0, null, null,
        null, null, null,
        "0 8 * * *",  // cron
        null, null
    );
    ```

=== "C# / .NET"

    ```csharp
    await queue.Enqueue(
        taskId: "daily-digest",
        taskType: "send_digest",
        jsonData: "{\"recipients\":\"all-users\"}",
        maxRetries: 3,
        retryAfterHours: 0.0,
        cron: "0 8 * * *"
    );
    ```

## Cron vs Retry: Key Differences

| | Cron Job | Retryable Job |
|---|---|---|
| **Triggers again when** | Success (on schedule) | Failure (after backoff) |
| **Purpose** | Recurring scheduled work | Recovery from transient failures |
| **Backoff** | N/A — fixed schedule | `retry_after` hours |
| **Combined behavior** | If a cron job fails, it uses `retry_after` to recover, then returns to its cron schedule |

## Persistence

Cron jobs are persisted to the queue's append-only log. If your application restarts, all scheduled cron jobs are preserved and will resume on their next scheduled tick.
