---
title: Risk Alerts: Wiring the Pipeline with AWS SQS
date: 2021-10-19 11:00:00
tags:
  - Career
  - AWS
  - SQS
categories:
  - Career
lang: en
description: When collateral values breach a threshold, the system has to notify clients promptly. Using AWS SQS to decouple the computation side from the notification side made retries and channel scaling dramatically simpler.
---

## The Alert Pipeline Problem

When collateral value drops below a certain threshold, the system needs to promptly alert the client to top up. That's a core capability of the Collateral system, what we call risk alerts.

Sounds like just "send a notification," right? In practice, it's more complicated than that.

The earliest approach was: after the computation module finishes, it directly calls the notification service inline. Computation and notification are coupled together, like cooking a meal and then carrying the pot next door to bang on your neighbor's door. The food is done, but you've now welded yourself to your neighbor.

The problems were obvious:

- If the notification service goes down, the computation fails too. Something that has nothing to do with the calculation result drags the entire Job down.
- No way to scale notification channels. SMS today, email tomorrow, App push the day after. Every new channel means modifying the computation module's code.
- Retry is a nightmare. What if the notification doesn't go through? Write retry logic inside the computation code? Now the computation module has to worry about retry count, backoff intervals, and failure handling, none of which is its job.

> **Direct coupling means tying unrelated things together, and one failure takes down both.**

## Why SQS

I picked AWS SQS for this, and the reasons were straightforward.

First, it's managed. No standing up your own message queue, no worrying about high availability, no ops overhead. Self-hosting a Kafka cluster for a financial system isn't cheap, and SQS eliminates that entire line item.

Second, it's simple. SQS has very few concepts: queue, message, consume, delete. No topics, partitions, or consumer groups to reason about. For a "send an alert" use case, that's all you need.

Third, retry is built in. If a consumer doesn't delete a message after consuming it, the message becomes visible again in the queue after a timeout. Retry logic is native to SQS, with no need to roll your own.

## Queue Design

I split queues by notification channel:

```yaml
# Queue layout (sanitized)
queues:
  - name: collateral-alert-sms
    visibility_timeout: 60       # invisible to others for 60s while consumed
    message_retention: 1209600   # retain messages up to 14 days
    redrive_policy:
      max_receive_count: 5       # after 5 attempts, move to DLQ
      dead_letter_queue: collateral-alert-sms-dlq

  - name: collateral-alert-email
    visibility_timeout: 120
    message_retention: 1209600
    redrive_policy:
      max_receive_count: 5
      dead_letter_queue: collateral-alert-email-dlq

  - name: collateral-alert-push
    visibility_timeout: 30
    message_retention: 1209600
    redrive_policy:
      max_receive_count: 3
      dead_letter_queue: collateral-alert-push-dlq
```

One queue per channel means they don't block each other. If SMS sending is slow, email still flows. Adding a new channel later just means creating a new queue; neither the producer nor the consumer code needs to change.

Each queue has a **dead-letter queue (DLQ)** attached. If a message exhausts its retry attempts, it automatically lands in the DLQ instead of clogging the main queue forever. Ops watches the DLQ and intervenes manually when something lands there.

## Producer: How Messages Get Sent

After the computation module identifies results that need alerting, it drops a message into the appropriate queue. That's all the producer does: construct the message body and send it.

```python
# Producer (sanitized)
import boto3
import json

sqs = boto3.client("sqs", region_name="ap-southeast-1")

class AlertProducer:
    def __init__(self):
        self.queue_urls = {
            "sms": "https://sqs.ap-southeast-1.amazonaws.com/xxx/collateral-alert-sms",
            "email": "https://sqs.ap-southeast-1.amazonaws.com/xxx/collateral-alert-email",
            "push": "https://sqs.ap-southeast-1.amazonaws.com/xxx/collateral-alert-push",
        }

    def send_alert(self, alert_type, alert_payload):
        """Route the alert to the right queue by type"""
        queue_url = self.queue_urls[alert_type]

        # Sanitized message body: only metadata needed for rendering
        message = {
            "alert_id": alert_payload["alert_id"],
            "client_ref": alert_payload["client_ref"],   # client identifier, not real PII
            "alert_level": alert_payload["alert_level"],
            "trigger_date": alert_payload["trigger_date"],
            "template_id": alert_payload["template_id"],
        }

        sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(message),
            # Message attributes for filtering and tracing
            MessageAttributes={
                "alert_level": {"DataType": "String",
                                 "StringValue": alert_payload["alert_level"]},
            },
        )
```

One detail worth calling out: the message body doesn't contain full business data. It holds only an `alert_id` and the metadata needed for template rendering. The actual client details and collateral specifics are fetched from the database by the consumer on demand. This keeps messages small and avoids pushing sensitive data through the queue.

## Consumer: How Messages Get Processed

The consumer is a standalone Python service that long-polls the queue:

```python
# Consumer (sanitized)
class AlertConsumer:
    def __init__(self, queue_url, channel):
        self.queue_url = queue_url
        self.channel = channel  # "sms" / "email" / "push"

    def poll(self):
        while True:
            messages = sqs.receive_message(
                QueueUrl=self.queue_url,
                MaxNumberOfMessages=10,
                WaitTimeSeconds=20,    # long polling, cuts empty requests
            )

            for msg in messages.get("Messages", []):
                try:
                    self._process(msg)
                    # Only delete after successful processing
                    sqs.delete_message(
                        QueueUrl=self.queue_url,
                        ReceiptHandle=msg["ReceiptHandle"],
                    )
                except Exception as e:
                    logger.error(f"Processing failed: {e}, msg_id={msg['MessageId']}")
                    # Don't delete — message reappears after visibility_timeout

    def _process(self, msg):
        body = json.loads(msg["Body"])
        # Fetch business data from DB to complete the alert (sanitized)
        detail = self._fetch_alert_detail(body["alert_id"])
        # Render and send the notification
        rendered = self.template.render(body["template_id"], detail)
        self.sender.send(self.channel, body["client_ref"], rendered)
```

The key design decision is to only delete the message after successful processing. If `_process` raises an exception, the message stays in the queue. After `visibility_timeout` expires, it becomes visible again and gets picked up by another consumer. That's the built-in SQS retry mechanism; no custom retry code is required.

**Long polling** (`WaitTimeSeconds=20`) is also worth highlighting. When there are no messages, SQS holds the request open for up to 20 seconds instead of returning immediately. This cuts down on empty polling requests and saves money and resources.

## Dead-Letter Queue Monitoring

If a message fails all 5 retry attempts, it lands in the DLQ. I wrote a separate DLQ monitor that triggers an ops alert the moment anything shows up there:

```python
# DLQ monitoring (sanitized)
def check_dlq(dlq_url):
    result = sqs.get_queue_attributes(
        QueueUrl=dlq_url,
        AttributeNames=["ApproximateNumberOfMessages"],
    )
    count = int(result["Attributes"]["ApproximateNumberOfMessages"])
    if count > 0:
        # Fire alert: messages are piling up in the dead-letter queue
        alert_service.notify(
            level="critical",
            message=f"SQS DLQ {dlq_url} has {count} backed-up messages",
        )
```

This way, no alert message silently disappears. Either it gets delivered successfully, or it lands in the DLQ for manual handling.

## What We Gained

After wiring up SQS, the biggest change was clear: the computation module and the notification module were fully decoupled.

- If the notification service is down, computation still runs to completion. Messages just wait in the queue.
- When the notification service recovers, the backlog is consumed automatically, with no manual intervention needed.
- Adding a new channel means adding a new queue. The computation code doesn't change at all.

> **The core value of a message queue isn't speed, it's decoupling. The producer doesn't need to know whether the consumer is alive. The consumer doesn't need to know how many messages the producer sent. Each does its own thing, coordinated through the queue as a buffer.**

Risk alert pipelines turn out to be a natural fit for message queues. The reason is simple: the producer (computation) and the consumer (notification) have different reliability requirements and different scaling patterns. Forcing them together just drags both sides down. Split them apart, and everyone breathes easier.
