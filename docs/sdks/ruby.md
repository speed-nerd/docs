# Ruby SDK

The official Ruby SDK for SnerdMQ. Ditch Sidekiq and Redis for lightweight, persistent background jobs.

[![Gem Version](https://badge.fury.io/rb/snerdmq.svg)](https://badge.fury.io/rb/snerdmq)

## Installation

**1. Install the gem:**
```bash
gem install snerdmq
# Or add to your Gemfile: gem 'snerdmq'
```

**2. Download the Rust engine:**
```bash
snerdmq-install
```

## Quickstart

```ruby
require 'snerdmq'

queue = Snerdmq::SnerdQueue.new

# Register a handler
queue.register_handler("send_email") do |data|
  puts "Sending email to #{data['to']}..."
  # Raise an exception to trigger retry
end

# Start listening (non-blocking threads)
queue.start_listening

# Enqueue a job
queue.enqueue(
  task_id: "email-123",
  task_type: "send_email",
  data: { "to" => "user@example.com", "subject" => "Welcome!" },
  max_retries: 3,
  retry_after_hours: 0.5,
)

# Handle permanently failed tasks
queue.register_max_retry_handler('send_email') do |data|
  puts "Failed after all retries: #{data.inspect}"
end

# Keep main thread alive
sleep
```

## API Reference

### `Snerdmq::SnerdQueue.new(storage_path: nil)`

Creates a new queue instance and spawns the background daemon.

### `queue.register_handler(task_type) { |data| ... }`

Registers a block handler for a task type.

### `queue.enqueue(...)`

Enqueues a new background job.

```ruby
queue.enqueue(
  task_id: "unique-id",
  task_type: "task_type",
  data: { "key" => "value" },
  max_retries: 3,
  retry_after_hours: 0.5,
  rate_limit_group: "api_group",
  max_per_minute: 50,
  auto_dedupe: true,
  urgency_score: 0.9,
  execute_at: "2026-12-31T23:59:00Z",
  cron: "0 8 * * *",
  webhook_url: "https://example.com/webhook",
  max_execution_seconds: 300,
)
```

### `queue.register_max_retry_handler(task_type) { |data| ... }`

Registers a block handler for permanently failed tasks (Dead Letter Queue).

### `queue.start_dashboard(port: 9090)`

Starts the built-in React dashboard.

### `queue.yield_progress(message)`

Streams a progress update from within a handler.

## Dashboard

```ruby
queue = Snerdmq::SnerdQueue.new
queue.start_dashboard(port: 9090)
# Open http://localhost:9090
```

## Queue Topology

**Recommended: one queue, all job types (singleton).** Each SDK client spawns its own Rust daemon and exclusively owns its storage directory (`.snerdata` by default). Register every job type on one client and serve a single shared dashboard:

```ruby
queue = Snerdmq::SnerdQueue.new

# Two job types sharing the same queue, daemon, and dashboard
queue.register_handler("process_image") do |data|
  puts "Processing image: #{data['image_id']}"
end

queue.register_handler("send_otp_email") do |data|
  puts "Sending OTP to: #{data['to']}"
end

queue.start_dashboard(port: 8080) # one dashboard shows every job type
```

All job types share the same job log, retry/DLQ pipeline, rate-limit state, and stats.

!!! warning "One queue per storage directory"
    The daemon takes an exclusive OS-level lock on its storage directory at startup. A second client on the same storage **fails fast** ("Another daemon is already running on storage ...") instead of double-executing jobs. This also applies across processes — with Puma/Unicorn clustered workers, every worker needs its own `storage_path`.

Need isolation between workloads? Give each queue its own storage directory — they become fully independent engines (own job log, rate limits, dashboard on its own port):

```ruby
images = Snerdmq::SnerdQueue.new(storage_path: ".snerdata-images")
emails = Snerdmq::SnerdQueue.new(storage_path: ".snerdata-emails")

images.start_dashboard(port: 8080)
emails.start_dashboard(port: 8081)
```

## Distributed Scaling

Scaling horizontally means **one queue per server**, each with its own storage — the load balancer routes requests, and every server processes the jobs it enqueued:

```ruby
# Each server runs its own daemon on its own storage dir (local disk works fine)
queue = Snerdmq::SnerdQueue.new(
  storage_path: "/var/data/snerd"
)
```

A shared network drive (AWS EFS or NFS) is still a good home for that storage when a single instance needs durable state across container restarts.
