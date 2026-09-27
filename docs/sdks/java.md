# Java / Kotlin SDK

The official JVM SDK for SnerdMQ. Works with Java, Kotlin, and Scala. Thread-safe with `ExecutorService` and `ProcessBuilder`.

## Installation

=== "Gradle"

    ```groovy
    dependencies {
        implementation 'io.github.speed-nerd:snerdmq:1.0.3'
    }
    ```

=== "Maven"

    ```xml
    <dependency>
        <groupId>io.github.speed-nerd</groupId>
        <artifactId>snerdmq</artifactId>
        <version>1.0.3</version>
    </dependency>
    ```

## Quickstart

```java
import snerdmq.SnerdQueue;
import snerdmq.SnerdmqInstaller;

public class App {
    public static void main(String[] args) throws Exception {
        // 1. (Optional) Download the Rust daemon
        SnerdmqInstaller.ensureDownloaded();

        // 2. Initialize the queue
        SnerdQueue queue = new SnerdQueue();

        // 3. Register a handler
        queue.registerHandler("send_email", (jsonData) -> {
            System.out.println("Sending email: " + jsonData);
            // Throw RuntimeException to trigger retry
        });

        // 4. Start async listeners
        queue.startListening();

        // 5. Enqueue a job
        queue.enqueue(
            "email-123",
            "send_email",
            "{\"to\":\"user@example.com\"}",
            3,    // max retries
            0.5,  // retry after hours
            "email_api",  // rate limit group
            100,  // max per minute
            null, null, null, null, null, null
        );

        // 6. Handle permanently failed tasks
        queue.registerMaxRetryHandler("send_email", data -> {
            System.out.println("Failed after all retries: " + data);
        });

        Thread.sleep(Long.MAX_VALUE);
    }
}
```

## API Reference

### `new SnerdQueue(storagePath = null)`

Creates a new queue instance and spawns the background daemon.

### `queue.registerHandler(taskType, Consumer<String> handler)`

Registers a consumer handler for a task type. The handler receives the JSON data string.

### `queue.enqueue(...)`

Enqueues a new background job with positional arguments:

```java
queue.enqueue(
    "task-id",          // String: unique task ID
    "task_type",        // String: task type
    "{\"key\":\"val\"}", // String: JSON payload
    3,                  // int: max retries
    0.5,                // double: retry after hours
    "api_group",        // String: rate limit group
    50,                 // Integer: max per minute
    true,               // Boolean: auto dedupe
    0.9,                // Double: urgency score
    "2026-12-31T23:59:00Z", // String/Instant: execute at
    "0 8 * * *",        // String: cron expression
    "https://...",      // String: webhook URL
    300                 // Integer: max execution seconds
);
```

### `queue.registerMaxRetryHandler(taskType, Consumer<String> handler)`

Registers a consumer handler for permanently failed tasks (Dead Letter Queue).

### `queue.startDashboard(port)`

Starts the built-in React dashboard.

### `queue.yieldProgress(message)`

Streams a progress update from within a handler.

## Hard Timeouts

The Java SDK uses `CompletableFuture.orTimeout()` for local timeout enforcement. The background Rust daemon also enforces timeouts at the IPC level.

## Dashboard

```java
SnerdQueue queue = new SnerdQueue();
queue.startDashboard(9090);
// Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```java
SnerdQueue queue = new SnerdQueue();

// Two job types sharing the same queue, daemon, and dashboard
queue.registerHandler("process_image", (jsonData) -> {
    System.out.println("Processing image: " + jsonData);
});

queue.registerHandler("send_otp_email", (jsonData) -> {
    System.out.println("Sending OTP: " + jsonData);
});

queue.startDashboard(8080); // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — multiple JVM services on the same machine each need their own `storagePath`.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```java
SnerdQueue images = new SnerdQueue(null, ".snerdata-images");
SnerdQueue emails = new SnerdQueue(null, ".snerdata-emails");

images.startDashboard(8080);
emails.startDashboard(8081);
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```java
// Each server runs its own daemon on its own storage dir (local disk works fine)
SnerdQueue queue = new SnerdQueue(null, "/var/data/snerd");
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.




## Advanced Orchestration (v0.3.0 Features)

SnerdMQ v0.3.0 introduced powerful new primitives for managing complex background jobs. Below are realistic, production-like scenarios showing how to utilize these features in Java:

```java
// 1. Sharded Queues
// Context: A developer needs to scale their deployment across 4 servers to handle massive load, but they don't want to use Redis.
// How to use: Tell the daemon how many shards to claim on boot. The queue handles the OS-level locking automatically.

SnerdQueue queue = new SnerdQueue(null, 4); // maxLocalShards
```

```java
// 2. Worker Pools
// Context: A system has both slow AI generation tasks and fast transactional emails. We want to prevent AI tasks from starving the email workers.

// Enqueue an AI task to a dedicated pool
queue.enqueue(
    "ai-gen-123", "ai_generation", "{ \"prompt\": \"A majestic horse\" }",
    3, 0.0, null, null, null, null, null, null, null, null, "ai-pool", null
);

// Enqueue an email task to a fast, urgent pool
queue.enqueue(
    "email-123", "send_email", "{ \"to\": \"user@example.com\" }",
    3, 0.0, null, null, null, null, null, null, null, null, "urgent", null
);
```

```java
// 3. Job Chaining (DAGs)
// Context: A video processing pipeline where a video must be transcoded, then uploaded to S3, and finally an email notification must be sent.

// Step 1: Transcode
queue.enqueue("transcode-1", "transcode_video", "{ \"file\": \"raw.mp4\" }", 3, 0.0, null);

// Step 2: Upload (Waits for Step 1)
queue.enqueue(
    "upload-1", "upload_s3", "{ \"file\": \"processed.mp4\" }",
    3, 0.0, null, null, null, null, null, null, null, null, null, Arrays.asList("transcode-1")
);

// Step 3: Notify (Waits for Step 2)
queue.enqueue(
    "notify-1", "send_email", "{ \"status\": \"done\" }",
    3, 0.0, null, null, null, null, null, null, null, null, null, Arrays.asList("upload-1")
);
```

```java
// 4. Cron & Scheduled Jobs
// Context: A system needs to run a database cleanup script every night at midnight.

queue.enqueue(
    "db-cleanup", "cleanup_job", "{ \"table\": \"sessions\" }",
    3, 0.0, null, null, null, null, null, "0 0 * * *", null, null, null, null
);
```

```java
// 5. Hard Timeouts
// Context: A background worker is making an HTTP request to a flaky third-party API that might hang indefinitely. We forcefully kill it if it runs over 5 minutes.

queue.enqueue(
    "api-fetch-1", "fetch_data", "{ \"endpoint\": \"/sync\" }",
    3, 0.0, null, null, null, null, null, null, null, 300, null, null
);
```

```java
// 6. Webhook Callbacks
// Context: A developer is using AWS Lambda or Vercel Serverless functions and wants SnerdMQ to trigger the function via an HTTP POST request rather than running a local worker.

queue.enqueue(
    "serverless-job", "resize_image", "{ \"img\": \"cat.jpg\" }",
    3, 0.0, null, null, null, null, null, null, "https://api.example.com/webhook/snerdmq", null, null, null
);
```

```java
// 7. The Dead Letter Queue (DLQ)
// Context: A task has failed its maximum number of retries (e.g., the SendGrid API is down for hours). The developer needs to catch this to alert the team on Slack.

queue.registerMaxRetryHandler("send_email", (data) -> {
    System.out.println("Task permanently failed! Alerting Slack with data: " + data);
});
```

## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```java
import com.snerdmq.SnerdQueue;

public class App {
    public static void main(String[] args) throws Exception {
        SnerdQueue queue = new SnerdQueue("./.snerdata", null, 1);

        // Email Workers
        queue.registerHandler("send_email", data -> {
            System.out.println("Sending email to " + data.get("email") + "...");
        });

        queue.registerMaxRetryHandler("send_email", data -> {
            System.out.println("Email to " + data.get("email") + " failed permanently. Dead letter processing...");
        });

        // Image Workers
        queue.registerHandler("process_image", data -> {
            System.out.println("Processing image " + data.get("imageId") + "...");
        });

        queue.startDashboard(8080);

        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            queue.shutdown();
        }));

        queue.startListening();
    }
}
```
