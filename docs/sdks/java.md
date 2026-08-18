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

## Distributed Scaling

```java
SnerdQueue queue = new SnerdQueue(null, "/mnt/aws-efs-shared-drive/snerd_tasks.log");
```
