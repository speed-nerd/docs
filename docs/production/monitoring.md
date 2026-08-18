# Monitoring

SnerdMQ provides multiple monitoring surfaces — from the built-in dashboard to programmatic APIs — so you can observe queue health in production.

## Built-in Dashboard

Every SDK ships with a React dashboard that provides real-time visibility:

```typescript
// Node.js
queue.startDashboard(9090);
```

```python
# Python
queue.start_dashboard(9090)
```

```go
// Go
queue.StartDashboard(9090)
```

Open **http://localhost:9090** to see:

- **Live stats**: total enqueued, processed, and failed jobs
- **Recent Jobs table**: per-task status (`queued`, `active`, `completed`, `failed`, `dead_letter`), retry counts, and feature badges
- **Real-time Progress Stream**: live output from `yieldProgress` calls

## JSON API

The dashboard exposes REST endpoints for programmatic access:

### `GET /api/stats`

Returns aggregate queue statistics.

```json
{
  "total_enqueued": 1542,
  "total_processed": 1498,
  "total_failed": 12,
  "total_active": 32,
  "total_dead_letter": 4,
  "queue_depth": 44
}
```

### `GET /api/tasks`

Returns the list of recent tasks with their status.

```json
{
  "tasks": [
    {
      "id": "email-123",
      "type": "send_email",
      "status": "completed",
      "retries": 0,
      "created_at": "2026-08-17T10:00:00Z",
      "features": ["rate_limit", "dedupe"]
    }
  ]
}
```

### `GET /api/progress`

Returns the latest progress events.

```json
{
  "progress": [
    {
      "task_id": "report-456",
      "message": "Step 7/10 complete",
      "timestamp": "2026-08-17T10:05:30Z"
    }
  ]
}
```

## Prometheus Metrics

You can expose queue stats to Prometheus by polling `/api/stats` with the Prometheus Node Exporter's `textfile` collector, or by writing a simple bridge:

```python
# Example: Prometheus bridge (Python)
import time
import requests
from prometheus_client import Gauge, start_http_server

queued = Gauge('snerdmq_queue_depth', 'Current queue depth')
processed = Gauge('snerdmq_total_processed', 'Total processed tasks')
failed = Gauge('snerdmq_total_failed', 'Total failed tasks')
dlq = Gauge('snerdmq_dead_letter_count', 'Dead letter queue count')

start_http_server(9100)

while True:
    stats = requests.get('http://localhost:9090/api/stats').json()
    queued.set(stats['queue_depth'])
    processed.set(stats['total_processed'])
    failed.set(stats['total_failed'])
    dlq.set(stats['total_dead_letter'])
    time.sleep(15)
```

## Health Checks

Add a health check endpoint to your application:

```typescript
app.get('/health', (req, res) => {
    try {
        const stats = queue.getStats();  // SDK-specific method
        res.json({
            status: 'healthy',
            queue_depth: stats.queue_depth,
            daemon_alive: true,
        });
    } catch (err) {
        res.status(503).json({
            status: 'unhealthy',
            error: err.message,
        });
    }
});
```

## Alerting Recommendations

Set up alerts for these conditions:

| Condition | Severity | Action |
|-----------|----------|--------|
| `queue_depth > 1000` | Warning | Scale up workers or investigate bottleneck |
| `total_dead_letter > 0` | Critical | Inspect DLQ entries, fix failing tasks |
| `total_failed / total_processed > 0.1` | Warning | High failure rate, check handler logs |
| `daemon process not running` | Critical | Restart application, check binary path |
| `queue file size > 1GB` | Warning | Investigate backlog, consider cleanup |

## Log Monitoring

SnerdMQ daemon logs to stderr. Capture it in your logging pipeline:

```bash
# Docker: logs are captured automatically
docker logs my-app-container

# Systemd: logs go to journalctl
journalctl -u my-app -f

# Kubernetes: logs go to stdout/stderr
kubectl logs -f deployment/my-app
```

## Dashboard in Production

!!! warning
    The built-in dashboard is intended for development and debugging. For production monitoring, use the `/api/stats` endpoint with your existing monitoring stack (Prometheus, Grafana, Datadog, etc.).

If you want to expose the dashboard in production:

- Put it behind authentication
- Restrict access to internal networks
- Or use the API endpoints directly with Grafana
