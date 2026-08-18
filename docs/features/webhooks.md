# Webhooks

SnerdMQ can dispatch tasks via HTTP POST requests to any URL, enabling serverless execution patterns. Instead of running handlers locally, SnerdMQ sends the task payload to your endpoint — perfect for distributed architectures, serverless functions, and cross-service orchestration.

## How It Works

When you set a `webhook_url` on a task:

1. SnerdMQ **skips local handlers** entirely
2. Sends an HTTP POST to your URL with the task payload as the JSON body
3. Includes the `X-SnerdMQ-Event: Execute` header
4. If your endpoint returns a **non-200 status**, the task is retried
5. If the task permanently fails (max retries reached), SnerdMQ fires a final POST with `X-SnerdMQ-Event: MaxRetriesReached`

## Webhook Request Format

**Headers:**
```
Content-Type: application/json
X-SnerdMQ-Event: Execute
```

**Body:** The task's `data` payload as JSON.

**DLQ notification (max retries reached):**
```
Content-Type: application/json
X-SnerdMQ-Event: MaxRetriesReached
```

## Code Examples

=== "Node.js"

    ```typescript
    // Dispatch task to a remote serverless worker
    queue.enqueue({
        id: `transcode-${Date.now()}`,
        type: 'transcode_video',
        data: { file: 's3://bucket/video.mp4', format: 'h264' },
        maxRetries: 3,
        webhookUrl: 'https://workers.example.com/transcode',
    });
    ```

=== "Python"

    ```python
    await queue.enqueue(
        task_id='transcode-123',
        task_type='transcode_video',
        data={'file': 's3://bucket/video.mp4', 'format': 'h264'},
        max_retries=3,
        webhook_url='https://workers.example.com/transcode',
    )
    ```

=== "Go"

    ```go
    webhookUrl := "https://workers.example.com/transcode"
    queue.Enqueue(
        "transcode-123", "transcode_video",
        map[string]interface{}{"file": "s3://bucket/video.mp4"},
        3, 0.0, "", 0,
        nil, nil, nil, nil,
        &webhookUrl,  // webhook URL
        nil,
    )
    ```

=== "Ruby"

    ```ruby
    queue.enqueue(
      task_id: "transcode-123",
      task_type: "transcode_video",
      data: { "file" => "s3://bucket/video.mp4" },
      max_retries: 3,
      webhook_url: "https://workers.example.com/transcode",
    )
    ```

=== "PHP"

    ```php
    $queue->enqueue(
        "transcode-123", "transcode_video",
        ["file" => "s3://bucket/video.mp4"],
        3, 0.0, null, null,
        null, null, null, null,
        "https://workers.example.com/transcode",  // webhook_url
        null
    );
    ```

=== "Java"

    ```java
    queue.enqueue(
        "transcode-123", "transcode_video",
        "{\"file\":\"s3://bucket/video.mp4\"}",
        3, 0.0, null, null,
        null, null, null, null,
        "https://workers.example.com/transcode",  // webhookUrl
        null
    );
    ```

=== "C# / .NET"

    ```csharp
    await queue.Enqueue(
        taskId: "transcode-123",
        taskType: "transcode_video",
        jsonData: "{\"file\":\"s3://bucket/video.mp4\"}",
        maxRetries: 3,
        webhookUrl: "https://workers.example.com/transcode"
    );
    ```

## Handling Webhook Requests (Server Side)

Here's how to receive SnerdMQ webhook dispatches in your server:

=== "Node.js / Express"

    ```typescript
    app.post('/webhook/transcode', (req, res) => {
        const event = req.headers['x-snerdmq-event'];
        
        if (event === 'Execute') {
            // Normal task execution
            const { file, format } = req.body;
            transcodeVideo(file, format);
            res.status(200).json({ status: 'ok' });
        } else if (event === 'MaxRetriesReached') {
            // DLQ notification — task permanently failed
            console.error('Task permanently failed:', req.body);
            res.status(200).json({ status: 'acknowledged' });
        }
    });
    ```

=== "Python / FastAPI"

    ```python
    @app.post("/webhook/transcode")
    async def handle_webhook(request: Request):
        event = request.headers.get("X-SnerdMQ-Event")
        body = await request.json()
        
        if event == "Execute":
            transcode_video(body["file"], body["format"])
            return {"status": "ok"}
        elif event == "MaxRetriesReached":
            logger.error(f"Task permanently failed: {body}")
            return {"status": "acknowledged"}
    ```

## Use Cases

- **Serverless workers** — Dispatch tasks to AWS Lambda, Cloudflare Workers, or Vercel Functions
- **Cross-service orchestration** — Trigger work in a different microservice
- **CI/CD pipelines** — Fire webhooks to Jenkins, GitHub Actions, or CircleCI
- **Notification systems** — Push events to Slack, PagerDuty, or custom alerting
- **External APIs** — Trigger third-party workflows without local handler code

## Webhook + DLQ

When a webhook task permanently fails (reaches `max_retries`), SnerdMQ automatically sends a final HTTP POST to the **same `webhook_url`** with the `X-SnerdMQ-Event: MaxRetriesReached` header. This means you don't need SDK-side max retry handlers for webhook tasks — your endpoint handles both execution and failure notification.
