# C# / .NET SDK

The official .NET SDK for SnerdMQ. Leverages C#'s `async`/`Task` ThreadPool with native ASP.NET Core compatibility.

[![NuGet](https://img.shields.io/nuget/v/SnerdMQ)](https://www.nuget.org/packages/SnerdMQ)

## Installation

```bash
dotnet add package SnerdMQ
```

## Quickstart

```csharp
using SnerdMQ;

class Program
{
    static async Task Main(string[] args)
    {
        using var queue = new SnerdQueue();

        // Register an async handler
        queue.RegisterHandler("send_email", async (jsonData) =>
        {
            Console.WriteLine($"Sending email: {jsonData}");
            await Task.Delay(1000);
            // Throw to trigger retry
        });

        // Start listening on the ThreadPool
        queue.StartListening();

        // Enqueue a job
        await queue.Enqueue(
            taskId: "email-123",
            taskType: "send_email",
            jsonData: "{\"to\":\"user@example.com\"}",
            maxRetries: 3,
            retryAfterHours: 0.5,
            rateLimitGroup: "email_api",
            maxPerMinute: 100
        );

        // Handle permanently failed tasks
        queue.RegisterMaxRetryHandler("send_email", (data) =>
        {
            Console.WriteLine($"Failed after all retries: {data}");
        });

        await Task.Delay(-1);
    }
}
```

## API Reference

### `new SnerdQueue(storagePath = null)`

Creates a new queue instance and spawns the background daemon. Implements `IDisposable` for clean shutdown.

### `queue.RegisterHandler(taskType, Func<string, Task> handler)`

Registers an async handler for a task type.

### `await queue.Enqueue(...)`

Enqueues a new background job.

```csharp
await queue.Enqueue(
    taskId: "unique-id",
    taskType: "task_type",
    jsonData: "{\"key\":\"value\"}",
    maxRetries: 3,
    retryAfterHours: 0.5,
    rateLimitGroup: "api_group",
    maxPerMinute: 50,
    autoDedupe: true,
    urgencyScore: 0.9,
    executeAt: DateTime.Parse("2026-12-31T23:59:00Z"),
    cron: "0 8 * * *",
    webhookUrl: "https://example.com/webhook",
    maxExecutionSeconds: 300
);
```

### `queue.RegisterMaxRetryHandler(taskType, Action<string> handler)`

Registers a handler for permanently failed tasks (Dead Letter Queue).

### `queue.StartDashboard(port)`

Starts the built-in React dashboard over the embedded `HttpListener`.

### `queue.YieldProgress(message)`

Streams a progress update from within a handler.

## Hard Timeouts

The .NET SDK uses `Task.WhenAny` with `Task.Delay` for local timeout enforcement. The background Rust daemon also enforces timeouts at the IPC level.

## Dashboard

```csharp
using var queue = new SnerdQueue();
queue.StartDashboard(9090);
// Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```csharp
using var queue = new SnerdQueue();

// Two job types sharing the same queue, daemon, and dashboard
queue.RegisterHandler("process_image", (jsonData) =>
{
    Console.WriteLine($"Processing image: {jsonData}");
});

queue.RegisterHandler("send_otp_email", (jsonData) =>
{
    Console.WriteLine($"Sending OTP: {jsonData}");
});

queue.StartDashboard(8080); // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — multi-worker deployments need one storage directory per worker.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```csharp
using var images = new SnerdQueue(null, ".snerdata-images");
using var emails = new SnerdQueue(null, ".snerdata-emails");

images.StartDashboard(8080);
emails.StartDashboard(8081);
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```csharp
// Each server runs its own daemon on its own storage dir (local disk works fine)
using var queue = new SnerdQueue(null, "/var/data/snerd");
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.




## Advanced Orchestration (v0.3.0 Features)

SnerdMQ v0.3.0 introduced powerful new primitives for managing complex background jobs. Below are realistic, production-like scenarios showing how to utilize these features in C# / .NET:

```csharp
// 1. Sharded Queues
// Context: A developer needs to scale their deployment across 4 servers to handle massive load, but they don't want to use Redis.
// How to use: Tell the daemon how many shards to claim on boot. The queue handles the OS-level locking automatically.

using var queue = new SnerdQueue(maxLocalShards: 4);
```

```csharp
// 2. Worker Pools
// Context: A system has both slow AI generation tasks and fast transactional emails. We want to prevent AI tasks from starving the email workers.

// Enqueue an AI task to a dedicated pool
queue.Enqueue(
    taskId: "ai-gen-123", 
    taskType: "ai_generation", 
    data: new { prompt = "A majestic horse" },
    pool: "ai-pool"
);

// Enqueue an email task to a fast, urgent pool
queue.Enqueue(
    taskId: "email-123", 
    taskType: "send_email", 
    data: new { to = "user@example.com" },
    pool: "urgent"
);
```

```csharp
// 3. Job Chaining (DAGs)
// Context: A video processing pipeline where a video must be transcoded, then uploaded to S3, and finally an email notification must be sent.

// Step 1: Transcode
queue.Enqueue(taskId: "transcode-1", taskType: "transcode_video", data: new { file = "raw.mp4" });

// Step 2: Upload (Waits for Step 1)
queue.Enqueue(
    taskId: "upload-1", 
    taskType: "upload_s3", 
    data: new { file = "processed.mp4" },
    triggerAfterIds: new List<string> { "transcode-1" }
);

// Step 3: Notify (Waits for Step 2)
queue.Enqueue(
    taskId: "notify-1", 
    taskType: "send_email", 
    data: new { status = "done" },
    triggerAfterIds: new List<string> { "upload-1" }
);
```

```csharp
// 4. Cron & Scheduled Jobs
// Context: A system needs to run a database cleanup script every night at midnight.

queue.Enqueue(
    taskId: "db-cleanup", 
    taskType: "cleanup_job", 
    data: new { table = "sessions" },
    cron: "0 0 * * *"
);
```

```csharp
// 5. Hard Timeouts
// Context: A background worker is making an HTTP request to a flaky third-party API that might hang indefinitely. We forcefully kill it if it runs over 5 minutes.

queue.Enqueue(
    taskId: "api-fetch-1", 
    taskType: "fetch_data", 
    data: new { endpoint = "/sync" },
    maxExecutionSeconds: 300
);
```

```csharp
// 6. Webhook Callbacks
// Context: A developer is using AWS Lambda or Vercel Serverless functions and wants SnerdMQ to trigger the function via an HTTP POST request rather than running a local worker.

queue.Enqueue(
    taskId: "serverless-job", 
    taskType: "resize_image", 
    data: new { img = "cat.jpg" },
    webhookUrl: "https://api.example.com/webhook/snerdmq"
);
```

```csharp
// 7. The Dead Letter Queue (DLQ)
// Context: A task has failed its maximum number of retries (e.g., the SendGrid API is down for hours). The developer needs to catch this to alert the team on Slack.

queue.RegisterMaxRetryHandler("send_email", async (data) =>
{
    Console.WriteLine($"Task permanently failed! Alerting Slack with data: {data}");
});
```

## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```csharp
using System;
using System.Threading.Tasks;
using SnerdMQ;

class Program
{
    static async Task Main(string[] args)
    {
        var queue = new SnerdQueue(storagePath: "./.snerdata");

        // Email Workers
        queue.RegisterHandler("send_email", async (data) =>
        {
            var email = (string)data["email"];
            Console.WriteLine($"Sending email to {email}...");
        });

        queue.RegisterMaxRetryHandler("send_email", async (data) =>
        {
            var email = (string)data["email"];
            Console.WriteLine($"Email to {email} failed permanently. Dead letter processing...");
        });

        // Image Workers
        queue.RegisterHandler("process_image", async (data) =>
        {
            Console.WriteLine($"Processing image {(string)data["imageId"]}...");
        });

        queue.StartDashboard(8080);

        // Graceful shutdown
        Console.CancelKeyPress += (s, e) =>
        {
            e.Cancel = true;
            queue.Shutdown();
        };

        await queue.StartListening();
    }
}
```
