# SnerdMQ — Background Jobs Without the Infrastructure

**SnerdMQ** is a polyglot background job queue that runs as an embedded sidecar daemon — not a standalone server. It persists jobs to an append-only log with OS-level file locking, giving you Redis-grade reliability with zero network hops, zero external dependencies, and sub-millisecond enqueue latency.

## Why SnerdMQ?

| Traditional Queues | SnerdMQ |
|---|---|
| Redis, RabbitMQ, or Kafka as a separate service | Embedded daemon spawned by your app |
| Network latency on every enqueue/dequeue | Local file I/O — sub-millisecond |
| Separate infrastructure to manage, monitor, and scale | Zero config — just install the SDK |
| OOM kills when the queue grows | Bounded memory, disk-backed persistence |
| Single-language clients | 7 official SDKs on the same engine and storage format |

## Key Features

- **Retries & Dead Letter Queue** — Automatic retry with configurable backoff. Permanently failed tasks land in the DLQ with a handler you control.
- **Smart Rate Limiting** — Group tasks by API or resource and SnerdMQ enforces velocity limits to prevent 429 errors.
- **Cron Scheduling** — Recurring jobs on any cron schedule. Combined with retry logic for self-healing scheduled tasks.
- **HTTP Webhooks** — Execute tasks via HTTP POST instead of local handlers. Perfect for serverless or cross-service orchestration.
- **Payload Deduplication** — Cryptographic hashing silently drops identical pending payloads.
- **Priority Queue** — Binary Max-Heap floats high-urgency tasks to the front, bypassing FIFO.
- **Live Dashboard** — Built-in React UI with real-time stats, job table, and progress streaming over WebSocket.
- **Progress Streaming** — Handlers can emit partial progress updates (ideal for LLM token streaming or multi-step ETL).

## Supported Languages

SnerdMQ ships official SDKs for:

=== "Node.js / TypeScript"
    ```bash
    npm install snerdmq-node
    ```
    [:octicons-arrow-right-24: Node.js SDK](sdks/node.md)

=== "Python"
    ```bash
    pip install snerdmq-python
    snerdmq-install
    ```
    [:octicons-arrow-right-24: Python SDK](sdks/python.md)

=== "Go"
    ```bash
    go get github.com/speed-nerd/snerdmq-go
    ```
    [:octicons-arrow-right-24: Go SDK](sdks/go.md)

=== "Ruby"
    ```bash
    gem install snerdmq
    ```
    [:octicons-arrow-right-24: Ruby SDK](sdks/ruby.md)

=== "PHP"
    ```bash
    composer require speed-nerd/snerdmq
    ```
    [:octicons-arrow-right-24: PHP SDK](sdks/php.md)

=== "Java / Kotlin"
    ```groovy
    implementation 'io.github.speed-nerd:snerdmq:1.0.3'
    ```
    [:octicons-arrow-right-24: Java SDK](sdks/java.md)

=== "C# / .NET"
    ```bash
    dotnet add package SnerdMQ
    ```
    [:octicons-arrow-right-24: .NET SDK](sdks/dotnet.md)

## Quick Example

=== "Node.js"
    ```typescript
    import { SnerdQueue } from 'snerdmq-node';

    const queue = new SnerdQueue();

    queue.registerHandler('send_email', async (data) => {
        console.log(`Sending email to ${data.to}...`);
    });

    queue.enqueue({
        id: `email-${Date.now()}`,
        type: 'send_email',
        data: { to: 'user@example.com', subject: 'Hello' },
        maxRetries: 3,
        retryAfter: 0.5,
    });
    ```

=== "Python"
    ```python
    from snerdmq import SnerdQueue

    queue = SnerdQueue()

    async def send_email(data):
        print(f"Sending email to {data['to']}...")

    queue.register_handler('send_email', send_email)

    await queue.enqueue(
        task_id='email-123',
        task_type='send_email',
        data={'to': 'user@example.com', 'subject': 'Hello'},
        max_retries=3,
        retry_after_hours=0.5,
    )
    ```

=== "Go"
    ```go
    import snerdmq "github.com/speed-nerd/snerdmq-go"

    queue, _ := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{})
    queue.RegisterHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
        fmt.Printf("Sending email to %v\n", data["to"])
        return nil
    })
    queue.Enqueue("email-123", "send_email", map[string]interface{}{"to": "user@example.com"}, 3, 0, "", 0, nil, nil, nil, nil, nil, nil)
    ```

## How It Works

Each SDK spawns the same Rust-compiled `snerdmq` daemon as a child process and communicates over JSON via stdin/stdout pipes. The daemon handles all queue orchestration — persistence, retries, rate limiting, scheduling, and deduplication — while your SDK handles task execution in your language.

```
┌─────────────────────────────────┐
│  Your Application (any language) │
│                                  │
│  ┌────────────────────────────┐ │
│  │   SnerdMQ SDK (thin client)│ │
│  │   JSON ↔ stdin/stdout       │ │
│  └──────────┬─────────────────┘ │
└─────────────┼───────────────────┘
              │
    ┌─────────▼──────────┐
    │  snerdmq daemon     │
    │  (Rust sidecar)     │
    │                     │
    │  • Append-only log  │
    │  • OS file locking  │
    │  • Priority heap    │
    │  • Rate limiter     │
    │  • Cron scheduler   │
    └─────────────────────┘
```

[:octicons-arrow-right-24: Read the full architecture](architecture.md)

## Scaling

SnerdMQ scales from a single laptop to a cluster:

- **Single machine** — The daemon writes to local disk. Sub-millisecond latency.
- **Multiple workers** — One queue per worker/server, each with its own storage: the daemon exclusively locks its storage directory, so scaling out means sharding. OS-level `flock` keeps writes safe.
- **Kubernetes / ECS** — Bundle the daemon binary in your app container. No sidecar containers needed.

[:octicons-arrow-right-24: Production deployment guide](production/deployment.md)
