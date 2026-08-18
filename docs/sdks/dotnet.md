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

## Distributed Scaling

```csharp
using var queue = new SnerdQueue(null, "/mnt/aws-efs-shared-drive/snerd_tasks.log");
```
