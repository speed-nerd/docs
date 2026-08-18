# Getting Started

This guide walks you through installing SnerdMQ and running your first background job. Pick your language below.

---

## Node.js / TypeScript

### Installation

```bash
npm install snerdmq-node
```

The post-install script automatically downloads the correct Rust daemon binary for your OS (macOS, Linux, or Windows).

### First Background Job

```typescript
import { SnerdQueue } from 'snerdmq-node';

// 1. Initialize — spawns the Rust daemon in the background
const queue = new SnerdQueue();

// 2. Register a handler for your job type
queue.registerHandler('send_email', async (data) => {
    console.log(`Sending email to ${data.to}...`);
    // Your business logic here
});

// 3. Enqueue a job
queue.enqueue({
    id: `email-${Date.now()}`,
    type: 'send_email',
    data: { to: 'user@example.com', subject: 'Welcome!' },
    maxRetries: 3,
    retryAfter: 0.5,  // Retry after 30 minutes on failure
});

// 4. (Optional) Handle permanent failures
queue.registerMaxRetryHandler('send_email', async (data) => {
    console.error(`Email permanently failed: ${JSON.stringify(data)}`);
});

// 5. (Optional) Start the live dashboard
queue.startDashboard(9090);
// Open http://localhost:9090 in your browser

// 6. Graceful shutdown
process.on('SIGINT', () => {
    queue.shutdown();
    process.exit(0);
});
```

---

## Python

### Installation

```bash
pip install snerdmq-python
snerdmq-install    # Downloads the Rust daemon binary
```

### First Background Job

```python
import asyncio
from snerdmq import SnerdQueue

async def send_email(data):
    print(f"Sending email to {data['to']}...")

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

    # Start listening for jobs (runs indefinitely)
    await queue.start_listening()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Go

### Installation

```bash
go get github.com/speed-nerd/snerdmq-go
go run github.com/speed-nerd/snerdmq-go/cmd/snerdmq-install@latest
```

### First Background Job

```go
package main

import (
    "context"
    "fmt"
    "time"

    snerdmq "github.com/speed-nerd/snerdmq-go"
)

func main() {
    queue, err := snerdmq.NewSnerdQueue(snerdmq.SnerdQueueConfig{})
    if err != nil {
        panic(err)
    }

    queue.RegisterHandler("send_email", func(ctx context.Context, data map[string]interface{}) error {
        fmt.Printf("Sending email to %v\n", data["to"])
        return nil
    })

    queue.StartListening()
    time.Sleep(500 * time.Millisecond)

    queue.Enqueue("email-123", "send_email",
        map[string]interface{}{"to": "user@example.com"},
        3, 0.5, "", 0, nil, nil, nil, nil, nil, nil)

    select {} // Block forever
}
```

---

## Ruby

### Installation

```bash
gem install snerdmq
snerdmq-install    # Downloads the Rust daemon binary
```

### First Background Job

```ruby
require 'snerdmq'

queue = SnerdQueue.new

queue.register_handler('send_email') do |data|
  puts "Sending email to #{data['to']}..."
end

queue.enqueue(
  task_id: 'email-123',
  task_type: 'send_email',
  task_data: { to: 'user@example.com', subject: 'Welcome!' }.to_json,
  max_retries: 3,
  retry_after_hours: 0.5
)

queue.start_listening
```

---

## PHP

### Installation

```bash
composer require speed-nerd/snerdmq
```

The post-install hook automatically downloads the Rust daemon binary.

### First Background Job

```php
<?php
require_once 'vendor/autoload.php';

use SnerdMQ\SnerdQueue;

$queue = new SnerdQueue();

$queue->registerHandler('send_email', function($data) {
    echo "Sending email to {$data['to']}...\n";
});

$queue->enqueue([
    'task_id' => 'email-123',
    'task_type' => 'send_email',
    'task_data' => json_encode(['to' => 'user@example.com']),
    'max_retries' => 3,
    'retry_after_hours' => 0.5,
]);

$queue->startListening();
```

---

## Java / Kotlin

### Installation

Add to your `build.gradle`:

```groovy
dependencies {
    implementation 'io.github.speed-nerd:snerdmq:1.0.3'
}
```

### First Background Job

```java
import snerdmq.SnerdQueue;

public class App {
    public static void main(String[] args) throws Exception {
        SnerdQueue queue = new SnerdQueue();

        queue.registerHandler("send_email", (data) -> {
            System.out.println("Sending email to " + data.get("to") + "...");
        });

        queue.enqueue("email-123", "send_email",
            "{\"to\": \"user@example.com\"}", 3, 0.5);

        queue.startListening();
    }
}
```

---

## C# / .NET

### Installation

```bash
dotnet add package SnerdMQ
```

### First Background Job

```csharp
using SnerdMQ;

var queue = new SnerdQueue();

queue.RegisterHandler("send_email", (data) => {
    Console.WriteLine($"Sending email to {data["to"]}...");
    return Task.CompletedTask;
});

queue.Enqueue("email-123", "send_email",
    "{\"to\": \"user@example.com\"}", 3, 0.5);

queue.StartListening();
```

---

## What's Next?

- [**Architecture**](architecture.md) — Understand how the daemon + SDK model works under the hood
- [**Features**](features/retries-and-dlq.md) — Explore retries, rate limiting, cron, webhooks, and more
- [**Production Deployment**](production/deployment.md) — Docker, Kubernetes, shared volumes, and monitoring
