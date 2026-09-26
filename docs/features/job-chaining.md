---
sidebar_position: 8
---

# Job Chaining (DAGs)

SnerdMQ allows you to model complex Directed Acyclic Graphs (DAGs) natively in the queue. By specifying a list of parent task IDs that must complete first, you can link background tasks into sophisticated, multi-step workflows.

Job chaining is particularly useful for AI orchestration, where you might need to sequentially fetch data, generate content, and compile results—scaling each step across different worker pools or languages.

## Overview

When you enqueue a task, you can pass an array of strings to the `trigger_after_ids` property. 

* The daemon will immediately durably save the task.
* However, it will **block** the task from dispatching.
* On every queue evaluation tick, the daemon checks if the specified parent tasks have completed.
* Once **all** parent tasks have completed, the child task is unlocked and allowed to execute according to its priority and schedule.

### Fail-Through Strategy

SnerdMQ employs a **Fail-Through** strategy for chained jobs:

1. If a parent task fails, it will be retried up to its `max_retries`.
2. While the parent is retrying, the child task remains blocked.
3. If the parent task reaches its maximum retries and fails permanently (sent to the Dead Letter Queue), it is marked as completed (with an error state).
4. SnerdMQ will then **propagate the block** by failing the child task immediately with the reason `blocked_by_failed_parent`. 

This ensures that downstream tasks do not execute if their prerequisites fail, preventing cascading data corruption or partial executions.

### Compaction Safety & The Orphaned Child

Because SnerdMQ relies on an append-only log that is periodically compacted, completed tasks are permanently erased from the system to save space. 

When the dispatch engine evaluates `trigger_after_ids`, it checks whether the parent task ID currently exists in the queue. **If the parent task ID is absent, SnerdMQ considers the parent task completed.**

This design guarantees that job chaining remains robust across daemon restarts and log compactions without O(N²) dependency checks. 

> [!WARNING]
> Because an absent task is considered "completed", if you specify a `trigger_after_ids` for a task that was *never enqueued* (e.g. a typo in the ID), the child task will immediately dispatch! Always ensure parent tasks are enqueued before or at the same time as their children.

## Examples

### Linear Pipelines (A → B → C)

In this example, we generate text, translate it, and then send an email.

```javascript
// 1. Parent Task
queue.enqueue({
    id: "generate_text_123",
    type: "LLM_GENERATE",
    data: { prompt: "Write a poem" }
});

// 2. Child Task (depends on 1)
queue.enqueue({
    id: "translate_text_123",
    type: "LLM_TRANSLATE",
    data: { target: "es" },
    trigger_after_ids: ["generate_text_123"]
});

// 3. Grandchild Task (depends on 2)
queue.enqueue({
    id: "send_email_123",
    type: "EMAIL",
    data: { to: "user@example.com" },
    trigger_after_ids: ["translate_text_123"]
});
```

### Fan-in Workflows (A & B → C)

Wait for multiple parallel tasks to finish before executing a final aggregation step.

```javascript
// Parallel Task A
queue.enqueue({
    id: "fetch_data_a",
    type: "API_FETCH",
    data: { source: "google" }
});

// Parallel Task B
queue.enqueue({
    id: "fetch_data_b",
    type: "API_FETCH",
    data: { source: "bing" }
});

// Aggregation Task (waits for both A and B)
queue.enqueue({
    id: "aggregate_results",
    type: "AGGREGATE",
    data: { },
    trigger_after_ids: ["fetch_data_a", "fetch_data_b"]
});
```

## Cross-Language Workflows

Because SnerdMQ is polyglot, the parent task could be executing in Python, while the child task is waiting to execute in Go. The daemon handles the orchestration entirely internally—your SDKs don't need to know about each other!

```python
# Parent in Python
queue.enqueue(
    task_id="analyze_image",
    task_type="CV_ANALYZE",
    data={"url": "image.jpg"}
)
```

```go
// Child in Go
queue.Enqueue(
    "send_notification",
    "PUSH_NOTIFY",
    map[string]interface{}{"msg": "Analysis complete!"},
    // ...
    []string{"analyze_image"} // trigger_after_ids
)
```
