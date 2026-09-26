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




## Advanced Orchestration (v0.3.0 Features)

SnerdMQ v0.3.0 introduced powerful new primitives for managing complex background jobs. Below are realistic, production-like scenarios showing how to utilize these features in Ruby:

```ruby
# 1. Sharded Queues
# Context: A developer needs to scale their deployment across 4 servers to handle massive load, but they don't want to use Redis.
# How to use: Tell the daemon how many shards to claim on boot. The queue handles the OS-level locking automatically.

queue = SnerdQueue.new(max_local_shards: 4)
```

```ruby
# 2. Worker Pools
# Context: A system has both slow AI generation tasks and fast transactional emails. We want to prevent AI tasks from starving the email workers.

# Enqueue an AI task to a dedicated pool
queue.enqueue(
  task_id: 'ai-gen-123',
  task_type: 'ai_generation',
  data: { prompt: 'A majestic horse' },
  pool: 'ai-pool'
)

# Enqueue an email task to a fast, urgent pool
queue.enqueue(
  task_id: 'email-123',
  task_type: 'send_email',
  data: { to: 'user@example.com' },
  pool: 'urgent'
)
```

```ruby
# 3. Job Chaining (DAGs)
# Context: A video processing pipeline where a video must be transcoded, then uploaded to S3, and finally an email notification must be sent.

# Step 1: Transcode
queue.enqueue(task_id: 'transcode-1', task_type: 'transcode_video', data: { file: 'raw.mp4' })

# Step 2: Upload (Waits for Step 1)
queue.enqueue(
  task_id: 'upload-1',
  task_type: 'upload_s3',
  data: { file: 'processed.mp4' },
  trigger_after_ids: ['transcode-1']
)

# Step 3: Notify (Waits for Step 2)
queue.enqueue(
  task_id: 'notify-1',
  task_type: 'send_email',
  data: { status: 'done' },
  trigger_after_ids: ['upload-1']
)
```

```ruby
# 4. Cron & Scheduled Jobs
# Context: A system needs to run a database cleanup script every night at midnight.

queue.enqueue(
  task_id: 'db-cleanup',
  task_type: 'cleanup_job',
  data: { table: 'sessions' },
  cron: '0 0 * * *'
)
```

```ruby
# 5. Hard Timeouts
# Context: A background worker is making an HTTP request to a flaky third-party API that might hang indefinitely. We forcefully kill it if it runs over 5 minutes.

queue.enqueue(
  task_id: 'api-fetch-1',
  task_type: 'fetch_data',
  data: { endpoint: '/sync' },
  max_execution_seconds: 300
)
```

```ruby
# 6. Webhook Callbacks
# Context: A developer is using AWS Lambda or Vercel Serverless functions and wants SnerdMQ to trigger the function via an HTTP POST request rather than running a local worker.

queue.enqueue(
  task_id: 'serverless-job',
  task_type: 'resize_image',
  data: { img: 'cat.jpg' },
  webhook_url: 'https://api.example.com/webhook/snerdmq'
)
```

```ruby
# 7. The Dead Letter Queue (DLQ)
# Context: A task has failed its maximum number of retries (e.g., the SendGrid API is down for hours). The developer needs to catch this to alert the team on Slack.

queue.register_max_retry_handler('send_email') do |data|
  puts "Task permanently failed! Alerting Slack with data: #{data}"
end
```

## Architecture Best Practices

When building production applications with SnerdMQ, it is recommended to initialize the queue as a Singleton, isolate your domain workers into separate files/functions, use Dead Letter Queues (DLQ) for failed tasks via `RegisterMaxRetryHandler`, and ensure manual graceful shutdown. The embedded Dashboard UI can also be easily served from the same instance.

```ruby
require 'snerdmq'

queue = SnerdQueue.new(storage_path: "./.snerdata")

def init_email_workers(queue)
  queue.register_handler('send_email') do |data|
    puts "Sending email to #{data['email']}..."
  end

  queue.register_max_retry_handler('send_email') do |data|
    puts "Email to #{data['email']} failed permanently. Dead letter processing..."
  end
end

def init_image_workers(queue)
  queue.register_handler('process_image') do |data|
    puts "Processing image #{data['imageId']}..."
  end
end

init_email_workers(queue)
init_image_workers(queue)

queue.start_dashboard(8080)

# Trap signals for graceful shutdown
trap('INT') { queue.shutdown; exit }
trap('TERM') { queue.shutdown; exit }

queue.start_listening
```
