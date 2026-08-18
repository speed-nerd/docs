# Priority Queue

SnerdMQ uses a Binary Max-Heap to dynamically reorder tasks by urgency. High-priority tasks float to the front of the queue, bypassing standard FIFO ordering with zero additional latency.

## How It Works

Every task has an `urgency_score` between `0.0` and `1.0`:

- **`0.0`** (default) — Standard FIFO ordering
- **`1.0`** — Highest priority, always executes first
- **`0.5`** — Mid-priority, behind urgent tasks but ahead of standard ones

The daemon maintains a **Binary Max-Heap** data structure that continuously re-sorts pending tasks. When a new high-urgency task arrives, it floats to the front of the execution line immediately — no waiting for lower-priority tasks to finish.

## Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `urgency_score` | float | `0.0` | Priority score (0.0–1.0). Higher values execute sooner |

## Code Examples

=== "Node.js"

    ```typescript
    // Standard task — executes in FIFO order
    queue.enqueue({
        id: 'report-1',
        type: 'generate_report',
        data: { format: 'pdf' },
    });

    // Urgent task — bypasses the queue
    queue.enqueue({
        id: 'report-urgent',
        type: 'generate_report',
        data: { format: 'pdf', client: 'vip' },
        urgencyScore: 0.95,  // Floats to the front
    });
    ```

=== "Python"

    ```python
    # Standard task
    await queue.enqueue(
        task_id='report-1',
        task_type='generate_report',
        data={'format': 'pdf'},
    )

    # Urgent task
    await queue.enqueue(
        task_id='report-urgent',
        task_type='generate_report',
        data={'format': 'pdf', 'client': 'vip'},
        urgency_score=0.95,
    )
    ```

=== "Go"

    ```go
    // Standard task
    queue.Enqueue("report-1", "generate_report",
        map[string]interface{}{"format": "pdf"},
        3, 0.0, "", 0,
        nil, nil, nil, nil, nil, nil,
    )

    // Urgent task
    urgency := 0.95
    queue.Enqueue("report-urgent", "generate_report",
        map[string]interface{}{"format": "pdf", "client": "vip"},
        3, 0.0, "", 0,
        nil, &urgency, nil, nil, nil, nil,
    )
    ```

=== "Ruby"

    ```ruby
    # Urgent task bypasses standard FIFO
    queue.enqueue(
      task_id: "report-urgent",
      task_type: "generate_report",
      data: { "format" => "pdf", "client" => "vip" },
      urgency_score: 0.95,
    )
    ```

=== "PHP"

    ```php
    $queue->enqueue(
        "report-urgent", "generate_report",
        ["format" => "pdf", "client" => "vip"],
        3, 0.0, null, null,
        null,
        0.99,  // urgency_score
        null, null, null, null
    );
    ```

=== "Java"

    ```java
    queue.enqueue(
        "report-urgent", "generate_report",
        "{\"format\":\"pdf\",\"client\":\"vip\"}",
        3, 0.0, null, null,
        null,
        0.99,  // urgencyScore
        null, null, null, null
    );
    ```

=== "C# / .NET"

    ```csharp
    await queue.Enqueue(
        taskId: "report-urgent",
        taskType: "generate_report",
        jsonData: "{\"format\":\"pdf\",\"client\":\"vip\"}",
        urgencyScore: 0.95
    );
    ```

## Priority Levels

A common pattern is to define priority tiers:

| Score | Tier | Use Case |
|-------|------|----------|
| `0.0` | Standard | Regular background jobs |
| `0.3` | Elevated | User-initiated actions |
| `0.7` | High | Time-sensitive processing |
| `0.9` | Critical | VIP users, SLA-bound tasks |
| `1.0` | Emergency | System-critical operations |

## How the Max-Heap Works

```
        [0.95]          ← Highest priority at root
       /      \
   [0.70]    [0.50]     ← Children always ≤ parent
   /    \      /
[0.30] [0.0] [0.0]      ← Standard FIFO tasks at bottom
```

When a new task is enqueued:
1. It's placed at the next available leaf position
2. It "bubbles up" by swapping with its parent until the heap property is restored
3. The task with the highest `urgency_score` is always at the root — ready for immediate dispatch

This gives **O(log n)** enqueue and **O(1)** dequeue for priority-sorted execution.

## Combining with Other Features

Priority works seamlessly with all other SnerdMQ features:

- **Priority + Rate Limiting** — Urgent tasks in a rate-limited group still get dispatched first when capacity opens
- **Priority + Dedup** — Duplicate check happens before priority sorting
- **Priority + Cron** — Cron tasks use their default priority (`0.0`) unless you set `urgency_score`
- **Priority + Retries** — Retried tasks retain their original urgency score
