# PHP SDK

The official PHP SDK for SnerdMQ. Works with Laravel, Symfony, or vanilla PHP. No Redis, no Beanstalkd, no RabbitMQ.

[![Packagist Version](https://img.shields.io/packagist/v/speed-nerd/snerdmq)](https://packagist.org/packages/speed-nerd/snerdmq)

## Installation

```bash
composer require speed-nerd/snerdmq
```

The post-install hook automatically downloads the correct Rust daemon binary for your OS.

## Quickstart

```php
<?php

require 'vendor/autoload.php';
use Snerdmq\SnerdQueue;

$queue = new SnerdQueue();

// Register a handler
$queue->registerHandler("send_email", function($data) {
    echo "Sending email to {$data['to']}...\n";
    // Throw an Exception to trigger retry
});

// Start listening
$queue->startListening();

// Enqueue a job
$queue->enqueue(
    "email-123",
    "send_email",
    ["to" => "user@example.com", "subject" => "Welcome!"],
    3,    // max retries
    0.5   // retry after hours
);

// Handle permanently failed tasks
$queue->registerMaxRetryHandler('send_email', function($data) {
    echo "Failed after all retries: " . json_encode($data) . "\n";
});

// Run the event loop (in a dedicated worker script)
$queue->listenLoop();
```

## API Reference

### `new SnerdQueue(storagePath = null)`

Creates a new queue instance and spawns the background daemon.

### `$queue->registerHandler($taskType, $callback)`

Registers a closure handler for a task type.

### `$queue->enqueue(...)`

Enqueues a new background job with positional arguments:

```php
$queue->enqueue(
    "task-id",          // string: unique task ID
    "task_type",        // string: task type
    ["key" => "value"], // array: JSON-serializable payload
    3,                  // int: max retries
    0.5,                // float: retry after hours
    "api_group",        // string: rate limit group
    50,                 // int: max per minute
    true,               // bool: auto dedupe
    0.9,                // float: urgency score
    "2026-12-31T23:59:00Z", // string: execute at
    "0 8 * * *",        // string: cron expression
    "https://...",      // string: webhook URL
    300                 // int: max execution seconds
);
```

### `$queue->registerMaxRetryHandler($taskType, $callback)`

Registers a closure handler for permanently failed tasks (Dead Letter Queue).

### `$queue->startDashboard($port)`

Starts the built-in React dashboard via PHP's built-in web server.

### `$queue->yieldProgress($message)`

Streams a progress update from within a handler.

## Hard Timeouts

The PHP SDK uses `pcntl_alarm` for local timeout enforcement:

- Requires the `pcntl` extension
- Not supported on Windows (daemon still enforces timeouts at IPC level)
- Avoid using other `pcntl_alarm` calls within handlers

## Dashboard

```php
$queue = new SnerdQueue();
$queue->startDashboard(9090);
// Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```php
$queue = new SnerdQueue();

// Two job types sharing the same queue, daemon, and dashboard
$queue->registerHandler("process_image", function($data) {
    echo "Processing image: {$data['image_id']}\n";
});

$queue->registerHandler("send_otp_email", function($data) {
    echo "Sending OTP to: {$data['to']}\n";
});

$queue->startDashboard(8080); // one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — multiple long-running PHP CLI workers each need their own `$storage_path`.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```php
$images = new SnerdQueue(null, ".snerdata-images");
$emails = new SnerdQueue(null, ".snerdata-emails");

$images->startDashboard(8080);
$emails->startDashboard(8081);
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```php
// Each server runs its own daemon on its own storage dir (local disk works fine)
$queue = new SnerdQueue(null, "/var/data/snerd");
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.




## Advanced Orchestration (v0.3.0 Features)

SnerdMQ v0.3.0 introduced powerful new primitives for managing complex background jobs. Below are realistic, production-like scenarios showing how to utilize these features in PHP:

```php
// 1. Sharded Queues
// Context: A developer needs to scale their deployment across 4 servers to handle massive load, but they don't want to use Redis.
// How to use: Tell the daemon how many shards to claim on boot. The queue handles the OS-level locking automatically.

$queue = new SnerdQueue(null, 4); // max_local_shards
```

```php
// 2. Worker Pools
// Context: A system has both slow AI generation tasks and fast transactional emails. We want to prevent AI tasks from starving the email workers.

// Enqueue an AI task to a dedicated pool
$queue->enqueue(
    'ai-gen-123', 'ai_generation', ['prompt' => 'A majestic horse'],
    3, 0, null, null, 0, null, null, null, null, 'ai-pool'
);

// Enqueue an email task to a fast, urgent pool
$queue->enqueue(
    'email-123', 'send_email', ['to' => 'user@example.com'],
    3, 0, null, null, 0, null, null, null, null, 'urgent'
);
```

```php
// 3. Job Chaining (DAGs)
// Context: A video processing pipeline where a video must be transcoded, then uploaded to S3, and finally an email notification must be sent.

// Step 1: Transcode
$queue->enqueue('transcode-1', 'transcode_video', ['file' => 'raw.mp4']);

// Step 2: Upload (Waits for Step 1)
$queue->enqueue(
    'upload-1', 'upload_s3', ['file' => 'processed.mp4'],
    3, 0, null, null, 0, null, null, null, ['transcode-1']
);

// Step 3: Notify (Waits for Step 2)
$queue->enqueue(
    'notify-1', 'send_email', ['status' => 'done'],
    3, 0, null, null, 0, null, null, null, ['upload-1']
);
```

```php
// 4. Cron & Scheduled Jobs
// Context: A system needs to run a database cleanup script every night at midnight.

$queue->enqueue(
    'db-cleanup', 'cleanup_job', ['table' => 'sessions'],
    3, 0, null, null, 0, '0 0 * * *'
);
```

```php
// 5. Hard Timeouts
// Context: A background worker is making an HTTP request to a flaky third-party API that might hang indefinitely. We forcefully kill it if it runs over 5 minutes.

$queue->enqueue(
    'api-fetch-1', 'fetch_data', ['endpoint' => '/sync'],
    3, 0, null, null, 0, null, null, 300
);
```

```php
// 6. Webhook Callbacks
// Context: A developer is using AWS Lambda or Vercel Serverless functions and wants SnerdMQ to trigger the function via an HTTP POST request rather than running a local worker.

$queue->enqueue(
    'serverless-job', 'resize_image', ['img' => 'cat.jpg'],
    3, 0, null, null, 0, null, 'https://api.example.com/webhook/snerdmq'
);
```

```php
// 7. The Dead Letter Queue (DLQ)
// Context: A task has failed its maximum number of retries (e.g., the SendGrid API is down for hours). The developer needs to catch this to alert the team on Slack.

$queue->registerMaxRetryHandler('send_email', function($data) {
    echo "Task permanently failed! Alerting Slack with data: " . json_encode($data) . "\n";
});
```

## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```php
<?php
require 'vendor/autoload.php';

use Snerd\SnerdQueue;

$queue = new SnerdQueue(['storage_path' => './.snerdata']);

$queue->registerHandler('send_email', function($data) {
    echo "Sending email to {$data['email']}...
";
});

$queue->registerMaxRetryHandler('send_email', function($data) {
    echo "Email to {$data['email']} failed permanently. Dead letter processing...
";
});

$queue->registerHandler('process_image', function($data) {
    echo "Processing image {$data['imageId']}...
";
});

$queue->startDashboard(8080);

// Graceful shutdown
if (function_exists('pcntl_signal')) {
    pcntl_async_signals(true);
    pcntl_signal(SIGINT, function() use ($queue) {
        $queue->shutdown();
        exit;
    });
    pcntl_signal(SIGTERM, function() use ($queue) {
        $queue->shutdown();
        exit;
    });
}

$queue->listenLoop();
```
