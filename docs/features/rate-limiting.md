# Rate Limiting

SnerdMQ includes a built-in token-bucket rate limiter that prevents your tasks from overwhelming third-party APIs. When you're bursting hundreds of LLM generation jobs, SnerdMQ automatically pauses dispatch to prevent 429 "Too Many Requests" errors.

## How It Works

Rate limiting in SnerdMQ uses a **rolling 60-second window** per group:

1. You assign tasks to a `rate_limit_group` (e.g., `"openai_api"`)
2. You set a `max_per_minute` cap for that group
3. The daemon tracks execution velocity for each group
4. When the limit is hit, further tasks in that group are **paused** (not dropped)
5. As the window slides and capacity frees up, paused tasks resume dispatching

This is **backpressure**, not rejection — tasks wait in the queue and execute as soon as capacity allows.

## Configuration

| Parameter | Type | Description |
|-----------|------|-------------|
| `rate_limit_group` | string | Groups tasks for shared rate limiting (e.g., `"anthropic"`, `"sendgrid"`) |
| `max_per_minute` | int | Maximum task executions per 60-second rolling window for this group |

## Code Examples

=== "Node.js"

    ```typescript
    queue.enqueue({
        id: `llm-${Date.now()}`,
        type: 'generate_response',
        data: { prompt: 'Explain quantum physics' },
        rateLimitGroup: 'anthropic',
        maxPerMinute: 50,  // Max 50 Anthropic API calls per minute
    });
    ```

=== "Python"

    ```python
    await queue.enqueue(
        task_id='llm-123',
        task_type='generate_response',
        data={'prompt': 'Explain quantum physics'},
        rate_limit_group='anthropic',
        max_per_minute=50,
    )
    ```

=== "Go"

    ```go
    queue.Enqueue(
        "llm-123", "generate_response",
        map[string]interface{}{"prompt": "Explain quantum physics"},
        3, 0.0,
        "anthropic",  // rate_limit_group
        50,           // max_per_minute
        nil, nil, nil, nil, nil, nil,
    )
    ```

=== "Ruby"

    ```ruby
    queue.enqueue(
      task_id: "llm-123",
      task_type: "generate_response",
      data: { "prompt" => "Explain quantum physics" },
      rate_limit_group: "anthropic",
      max_per_minute: 50,
    )
    ```

=== "PHP"

    ```php
    $queue->enqueue(
        "llm-123", "generate_response",
        ["prompt" => "Explain quantum physics"],
        3, 0.0,
        "anthropic",  // rate_limit_group
        50            // max_per_minute
    );
    ```

=== "Java"

    ```java
    queue.enqueue(
        "llm-123", "generate_response",
        "{\"prompt\":\"Explain quantum physics\"}",
        3, 0.0,
        "anthropic",  // rateLimitGroup
        50,           // maxPerMinute
        null, null, null, null, null, null
    );
    ```

=== "C# / .NET"

    ```csharp
    await queue.Enqueue(
        taskId: "llm-123",
        taskType: "generate_response",
        jsonData: "{\"prompt\":\"Explain quantum physics\"}",
        maxRetries: 3,
        retryAfterHours: 0.0,
        rateLimitGroup: "anthropic",
        maxPerMinute: 50
    );
    ```

## Multiple Rate Limit Groups

You can define independent rate limit groups for different services:

```
Group: "openai_api"     → max 60/min
Group: "anthropic"      → max 50/min
Group: "sendgrid"       → max 100/min
Group: "db_writes"      → max 200/min
```

Each group has its own independent rolling window. Tasks without a `rate_limit_group` execute immediately with no throttling.

## Use Cases

- **LLM API calls** — Prevent 429 errors when bursting GPT-4 or Claude requests
- **Email sending** — Respect SendGrid/Mailgun rate limits
- **Database writes** — Throttle bulk inserts to avoid connection pool exhaustion
- **Webhook dispatch** — Stay within downstream webhook rate limits
- **Payment processing** — Respect Stripe/PayPal API quotas
