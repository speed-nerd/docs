# Troubleshooting

Common issues and their solutions when running SnerdMQ in production.

## Daemon Won't Start

**Symptom:** Application throws an error about the daemon binary not being found.

**Causes & Solutions:**

1. **Binary not downloaded** — Run the installer:
    ```bash
    # Node.js: automatic on npm install
    # Python:
    snerdmq-install
    # Go:
    go run github.com/speed-nerd/snerdmq-go/cmd/snerdmq-install@latest
    # Ruby:
    snerdmq-install
    # PHP: automatic on composer require
    ```

2. **Wrong binary path** — Ensure the binary is in your `$PATH` or in the SDK's expected location:
    ```bash
    which snerdmq
    ls ./bin/snerdmq  # Go SDK places it here
    ```

3. **Permission denied** — Make the binary executable:
    ```bash
    chmod +x ./bin/snerdmq
    ```

## Queue File Locked

**Symptom:** `Error: EBUSY: resource busy or locked` or `flock` errors.

**Cause:** Another process is holding an exclusive lock on the queue file.

**Solutions:**

1. **Check for zombie processes:**
    ```bash
    ps aux | grep snerdmq
    kill -9 <pid>  # Kill stale daemon
    ```

2. **Stale lock file** — On some systems, a crash can leave a stale lock:
    ```bash
    # Remove the queue file (WARNING: loses pending tasks)
    rm .snerdata/tasks/tasks.log
    ```

3. **NFS/EFS lock issues** — If you point a single instance at a network drive for durable storage, ensure `flock` is supported:
    ```bash
    # AWS EFS supports flock natively
    # Some NFS implementations may not — check your provider
    ```
    Note: network volumes are for single-instance durability only. Two instances on the same storage fail fast by design (exclusive lock) — scale by sharding, not by sharing. If a second queue targets busy storage, its daemon refuses to start and `enqueue` raises `[Snerd] Engine terminated before ack for task '<id>'` (or `Cannot enqueue task: engine is not running`) instead of hanging.

## High Memory Usage

**Symptom:** Application memory grows over time.

**Cause:** SnerdMQ loads pending tasks into memory for fast access. Very large queues can consume significant RAM.

**Solutions:**

1. **Process tasks faster** — Add more workers or optimize handler performance
2. **Reduce queue depth** — Enqueue less, or increase processing throughput
3. **Check for stuck tasks** — Tasks that keep failing and retrying accumulate:
    ```bash
    curl http://localhost:9090/api/stats
    # Check total_failed and queue_depth
    ```

## Tasks Not Executing

**Symptom:** Tasks are enqueued but handlers never fire.

**Checklist:**

1. **Is the listener running?** — Ensure `startListening()` / `start_listening()` was called
2. **Handler type matches?** — The `task_type` in `enqueue()` must exactly match the registered handler name
3. **Is the daemon alive?** — Check for the child process:
    ```bash
    ps aux | grep snerdmq
    ```
4. **Check stderr output** — The daemon logs errors to stderr

## Rate Limiting Issues

**Symptom:** Tasks are stuck in "paused" state and never execute.

**Cause:** Rate limit threshold is too low for your workload.

**Solutions:**

1. **Increase `max_per_minute`:**
    ```typescript
    queue.enqueue({
        rateLimitGroup: 'api',
        maxPerMinute: 200,  // Increase from 50
    });
    ```

2. **Remove rate limiting** if not needed — Set `rateLimitGroup` to empty/null

## Cron Jobs Not Firing

**Symptom:** Cron tasks execute once but never repeat.

**Checklist:**

1. **Did the first execution succeed?** — Cron jobs only re-execute after **success**. If the handler fails, it enters retry mode instead.
2. **Is the cron expression valid?** — Test it:
    ```bash
    # Standard cron: "0 8 * * *" = daily at 8 AM
    # Shorthand: "2h" = every 2 hours
    ```
3. **Is the daemon still running?** — Cron scheduling requires the daemon to be alive

## Deduplication Not Working

**Symptom:** Duplicate tasks are still getting through.

**Cause:** Dedup only checks against **pending** tasks. If the first task already executed, the second one won't be detected as a duplicate.

**Solution:** Ensure `auto_dedupe: true` is set, and understand that dedup is a "same-tick" check, not a historical one.

## Dashboard Not Loading

**Symptom:** `http://localhost:9090` shows a blank page or connection refused.

**Solutions:**

1. **Did you call `startDashboard()`?** — The dashboard doesn't start automatically
2. **Port conflict** — Try a different port:
    ```typescript
    queue.startDashboard(9091);
    ```
3. **Static assets path** — The dashboard serves `static/` files relative to your working directory. Ensure you're running from the correct directory.

## Graceful Shutdown Issues

**Symptom:** Pending tasks are lost when the application exits.

**Solution:** Always shut down gracefully:

=== "Node.js"
    ```typescript
    process.on('SIGINT', () => {
        queue.shutdown();
        process.exit(0);
    });
    ```

=== "Python"
    ```python
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        queue.shutdown()
    ```

=== "Go"
    ```go
    // queue.Wait() blocks until SIGINT
    queue.Wait()
    ```

## Getting Help

If you're stuck:

1. **Check the [GitHub Issues](https://github.com/speed-nerd/snerdmq/issues)** for known problems
2. **Enable debug logging** — Set environment variable `SNERD_DEBUG=1` for verbose output
3. **Open an issue** — Include your OS, SDK version, daemon version, and a minimal reproduction
