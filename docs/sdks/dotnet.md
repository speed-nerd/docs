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
