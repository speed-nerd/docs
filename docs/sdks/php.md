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
