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
