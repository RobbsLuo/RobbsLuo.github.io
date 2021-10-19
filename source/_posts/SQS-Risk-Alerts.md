---
title: 风险预警：用 AWS SQS 把提醒链路接起来
date: 2021-10-19 11:00:00
tags:
  - Career
  - AWS
  - SQS
categories:
  - Career
description: Collateral 系统的风险预警要触达客户，中间经过多个环节。用 AWS SQS 把提醒链路串起来后，解耦了生产端和消费端，重试和扩展都简单多了。
---

## 风险预警的链路问题

抵押品价值一旦跌破某个水位，系统得及时提醒客户补仓。这是 Collateral 系统的一块硬需求，风险预警。

听起来就是发个通知的事，但实际做起来没那么简单。

最早的做法是什么？计算模块算完之后，直接在代码里调通知服务发消息。计算和通知耦合在一起，就像你炒完菜直接端着锅去敲邻居的门，菜是炒好了，但你和邻居的关系被绑死了。

问题很明显：

- 通知服务挂了，计算也跟着失败。一个不影响计算结果的事情，把整个 Job 拖崩了。
- 没法扩展通知渠道。今天发短信，明天要加邮件，后天要加 App 推送。每加一个渠道就得改计算模块的代码。
- 失败重试很难做。通知没发成功怎么办？在计算代码里写重试逻辑？那重试几次、间隔多久、失败后怎么办，全是计算模块的负担。

直接调用的问题，就是把不相关的事情绑在一起，一荣俱损。

## 为什么选 SQS

我选 AWS SQS 来解这个问题，原因很直接：

它是托管的，不用自己搭消息队列、不用管高可用、不用运维。金融系统自己运维一个 Kafka 集群的成本不低，SQS 直接省了这块。

SQS 的概念也少，就队列、消息、消费、删除。没有 topic、partition、consumer group 那套复杂模型。对于"发个提醒"这种场景，够用了。

另外它天然支持重试。消息被消费后如果不删，过一会会重新回到队列里。重试逻辑是 SQS 内置的，不用自己写。

## 队列设计

我先按通知渠道拆了队列：

```yaml
# 队列设计（脱敏示意）
queues:
  - name: collateral-alert-sms
    visibility_timeout: 60       # 消费中 60s 内不可见
    message_retention: 1209600   # 消息最多保留 14 天
    redrive_policy:
      max_receive_count: 5       # 重试 5 次后进死信队列
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

每个渠道一个队列，好处是互不阻塞。短信服务慢了不会影响邮件的发送。如果以后要加新渠道，新建一个队列就行，生产端和消费端都不用改。

每个队列还配了**死信队列（DLQ）**。消息重试到上限还没成功，自动转到 DLQ 里，不会一直堵在主队列。运维同学盯 DLQ 就行，有问题人工介入处理。

## 生产端：怎么发消息

计算模块算出需要预警的结果后，往队列里丢消息。这一步要做的就是构造消息体、发送，仅此而已。

```python
# 生产端（脱敏示意）
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
        """根据预警类型发到对应队列"""
        queue_url = self.queue_urls[alert_type]

        # 消息体脱敏：只包含通知渲染所需的元数据
        message = {
            "alert_id": alert_payload["alert_id"],
            "client_ref": alert_payload["client_ref"],   # 客户标识，非真实信息
            "alert_level": alert_payload["alert_level"],
            "trigger_date": alert_payload["trigger_date"],
            "template_id": alert_payload["template_id"],
        }

        sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(message),
            # 加消息属性方便过滤和追踪
            MessageAttributes={
                "alert_level": {"DataType": "String",
                                 "StringValue": alert_payload["alert_level"]},
            },
        )
```

这里有个细节：消息体里不包含完整的业务数据，只有一个 `alert_id` 和渲染模板需要的元数据。具体的客户信息、抵押品明细，由消费端按需从数据库查。这样消息体小、传输快，也降低了敏感信息在队列里流转的风险。

## 消费端：怎么处理消息

消费端是一个独立的 Python 服务，长期轮询队列：

```python
# 消费端（脱敏示意）
class AlertConsumer:
    def __init__(self, queue_url, channel):
        self.queue_url = queue_url
        self.channel = channel  # "sms" / "email" / "push"

    def poll(self):
        while True:
            messages = sqs.receive_message(
                QueueUrl=self.queue_url,
                MaxNumberOfMessages=10,
                WaitTimeSeconds=20,    # 长轮询，减少空请求
            )

            for msg in messages.get("Messages", []):
                try:
                    self._process(msg)
                    # 处理成功才删除消息
                    sqs.delete_message(
                        QueueUrl=self.queue_url,
                        ReceiptHandle=msg["ReceiptHandle"],
                    )
                except Exception as e:
                    logger.error(f"处理失败: {e}, msg_id={msg['MessageId']}")
                    # 不删除，消息会在 visibility_timeout 后重新可见

    def _process(self, msg):
        body = json.loads(msg["Body"])
        # 从数据库补全业务数据（脱敏）
        detail = self._fetch_alert_detail(body["alert_id"])
        # 渲染并发送通知
        rendered = self.template.render(body["template_id"], detail)
        self.sender.send(self.channel, body["client_ref"], rendered)
```

关键设计：只有处理成功才删消息。如果 `_process` 抛异常，消息不会被删除，等 `visibility_timeout` 过后会重新回到队列，被另一个消费者拉取。这就是 SQS 内置的重试机制。

长轮询（`WaitTimeSeconds=20`）也值得一提。没有消息时，SQS 会 hold 住请求最多 20 秒，减少空轮询的请求量，省钱也省资源。

## 死信队列和告警

消息重试 5 次都失败，会进 DLQ。我单独写了一个 DLQ 监控脚本，一旦 DLQ 里有消息，立刻触发告警通知运维：

```python
# DLQ 监控（脱敏示意）
def check_dlq(dlq_url):
    result = sqs.get_queue_attributes(
        QueueUrl=dlq_url,
        AttributeNames=["ApproximateNumberOfMessages"],
    )
    count = int(result["Attributes"]["ApproximateNumberOfMessages"])
    if count > 0:
        # 触发告警：有消息进死信队列了
        alert_service.notify(
            level="critical",
            message=f"SQS DLQ {dlq_url} 有 {count} 条积压消息",
        )
```

这样，任何一条预警消息都不会悄无声息地丢掉。要么成功发出去，要么进 DLQ 被人工处理。

## 收益

上 SQS 之后最大的变化是：计算模块和通知模块彻底解耦了。

- 通知服务挂了，计算照样跑完，消息在队列里等着。
- 通知服务恢复后，积压的消息自动被消费，不用人工干预。
- 加新渠道只加新队列，计算模块一行代码都不用改。

消息队列真正的价值是解耦。生产端不用关心消费端活没活着，消费端不用关心生产端发了几条。各干各的，通过队列这个缓冲层协作。

回头看，风险预警这种链路天然适合用消息队列。生产端（计算）和消费端（通知）的可靠性要求不同、速度不同、扩展方式也不同。硬拧在一起只会互相拖累，拆开反而都舒服了。
