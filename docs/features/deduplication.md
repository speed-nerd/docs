# Deduplication

SnerdMQ can automatically detect and drop duplicate tasks before they enter the queue. When you enable deduplication, the daemon computes a cryptographic hash of the task type and payload — if an identical task is already pending execution, the new one is silently dropped.

## How It Works

1. You set `auto_dedupe: true` on a task
2. Before enqueueing, the daemon computes a SHA-256 hash of `task_type + task_data`
3. It checks the hash against all currently pending (not yet executed) tasks
4. If a match is found, the new task is **silently dropped** — no error, no exception
5. If no match, the task is enqueued normally

This is perfect for preventing duplicate work from idempotent operations — like sending the same email twice when a user double-clicks a button.

## Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `auto_dedupe` | bool | `false` | Enable payload-hash deduplication for this task |

!!! important
    Deduplication only checks against tasks **currently pending** in the queue. Once a task has been executed or deleted, it no longer participates in dedup checks.

## Code Examples

=== "Node.js"

    ```typescript
    // Both of these will result in only ONE task in the queue
    queue.enqueue({
        id: 'digest-1',
        type: 'send_digest',
        data: { user: 'john', period: 'daily' },
        autoDedupe: true,  // First one enqueues
    });

    queue.enqueue({
        id: 'digest-2',
        type: 'send_digest',
        data: { user: 'john', period: 'daily' },
        autoDedupe: true,  // Silently dropped — identical payload pending
    });
    ```

=== "Python"

    ```python
    await queue.enqueue(
        task_id='digest-1',
        task_type='send_digest',
        data={'user': 'john', 'period': 'daily'},
        auto_dedupe=True,  # First one enqueues
    )

    await queue.enqueue(
        task_id='digest-2',
        task_type='send_digest',
        data={'user': 'john', 'period': 'daily'},
        auto_dedupe=True,  # Silently dropped
    )
    ```

=== "Go"

    ```go
    autoDedupe := true
    queue.Enqueue("digest-1", "send_digest",
        map[string]interface{}{"user": "john", "period": "daily"},
        3, 0.0, "", 0,
        &autoDedupe, nil, nil, nil, nil, nil,
    )

    queue.Enqueue("digest-2", "send_digest",
        map[string]interface{}{"user": "john", "period": "daily"},
        3, 0.0, "", 0,
        &autoDedupe, nil, nil, nil, nil, nil,  // Silently dropped
    )
    ```

=== "Ruby"

    ```ruby
    queue.enqueue(
      task_id: "digest-1",
      task_type: "send_digest",
      data: { "user" => "john", "period" => "daily" },
      auto_dedupe: true,  # Enqueues
    )

    queue.enqueue(
      task_id: "digest-2",
      task_type: "send_digest",
      data: { "user" => "john", "period" => "daily" },
      auto_dedupe: true,  # Silently dropped
    )
    ```

=== "PHP"

    ```php
    $queue->enqueue("digest-1", "send_digest",
        ["user" => "john", "period" => "daily"],
        3, 0.0, null, null,
        true,  // auto_dedupe: enqueues
        null, null, null, null, null
    );

    $queue->enqueue("digest-2", "send_digest",
        ["user" => "john", "period" => "daily"],
        3, 0.0, null, null,
        true,  // auto_dedupe: silently dropped
        null, null, null, null, null
    );
    ```

=== "Java"

    ```java
    queue.enqueue("digest-1", "send_digest",
        "{\"user\":\"john\",\"period\":\"daily\"}",
        3, 0.0, null, null,
        true,   // autoDedupe: enqueues
        null, null, null, null, null
    );

    queue.enqueue("digest-2", "send_digest",
        "{\"user\":\"john\",\"period\":\"daily\"}",
        3, 0.0, null, null,
        true,   // autoDedupe: silently dropped
        null, null, null, null, null
    );
    ```

=== "C# / .NET"

    ```csharp
    await queue.Enqueue(
        taskId: "digest-1",
        taskType: "send_digest",
        jsonData: "{\"user\":\"john\",\"period\":\"daily\"}",
        autoDedupe: true  // Enqueues
    );

    await queue.Enqueue(
        taskId: "digest-2",
        taskType: "send_digest",
        jsonData: "{\"user\":\"john\",\"period\":\"daily\"}",
        autoDedupe: true  // Silently dropped
    );
    ```

## What Gets Hashed

The dedup hash covers:

- **`task_type`** — The task type string (e.g., `"send_email"`)
- **`task_data`** — The serialized JSON payload

The `task_id` is **not** included in the hash — two tasks with different IDs but identical type + data will still be considered duplicates.

## Use Cases

- **Double-click prevention** — User clicks "Send" twice, only one email goes out
- **Idempotent API calls** — Prevent duplicate LLM generation requests
- **Event dedup** — Multiple event sources fire the same notification, process it once
- **Webhook dedup** — Upstream systems retry webhooks, dedup ensures single processing
- **Batch dedup** — Bulk enqueue scripts skip tasks already in the queue
