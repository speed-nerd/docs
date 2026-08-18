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

## Distributed Scaling

```php
$queue = new SnerdQueue(null, "/mnt/aws-efs-shared-drive/snerd_tasks.log");
```
