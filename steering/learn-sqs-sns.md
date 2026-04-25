---
inclusion: manual
---

# Learn SQS & SNS with MiniStack

## SQS — Simple Queue Service

SQS is a **message queue**. One service puts a message in, another service reads it later. They don't need to be running at the same time — the queue holds the message until someone picks it up.

**When to use it:** decoupling services, handling traffic spikes, background jobs, retry logic.

### Core Concepts

- **Queue** — a buffer that holds messages
- **Producer** — sends messages to the queue
- **Consumer** — reads and processes messages, then deletes them
- **Visibility timeout** — when a consumer reads a message, it becomes invisible to others for N seconds. If not deleted in time, it reappears (for retry).
- **Dead-letter queue (DLQ)** — messages that fail too many times go here for inspection

### Hands-On: Basic Queue

```python
import boto3, json

sqs = boto3.client("sqs",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a queue
response = sqs.create_queue(QueueName="orders")
queue_url = response["QueueUrl"]

# Send a message
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps({"orderId": "ord-001", "amount": 49.99}),
)

# Receive messages (up to 10 at a time)
response = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=1,  # long polling — waits up to 1s for messages
)

for msg in response.get("Messages", []):
    body = json.loads(msg["Body"])
    print(f"Processing order: {body['orderId']}")

    # IMPORTANT: delete after processing, or it reappears after visibility timeout
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=msg["ReceiptHandle"],
    )
```

### Hands-On: Dead-Letter Queue

```python
# Create DLQ first
dlq = sqs.create_queue(QueueName="orders-dlq")
dlq_url = dlq["QueueUrl"]
dlq_attrs = sqs.get_queue_attributes(
    QueueUrl=dlq_url,
    AttributeNames=["QueueArn"],
)
dlq_arn = dlq_attrs["Attributes"]["QueueArn"]

# Create main queue with DLQ — after 3 failed attempts, message goes to DLQ
main_queue = sqs.create_queue(
    QueueName="orders-with-dlq",
    Attributes={
        "RedrivePolicy": json.dumps({
            "deadLetterTargetArn": dlq_arn,
            "maxReceiveCount": "3",
        }),
    },
)
queue_url = main_queue["QueueUrl"]

# Send a message
sqs.send_message(QueueUrl=queue_url, MessageBody="process me")

# Simulate 3 failed attempts (receive but don't delete)
for attempt in range(3):
    msgs = sqs.receive_message(QueueUrl=queue_url, VisibilityTimeout=0)
    print(f"Attempt {attempt + 1}: got {len(msgs.get('Messages', []))} messages")

# After maxReceiveCount, message moves to DLQ
import time; time.sleep(1)
dlq_msgs = sqs.receive_message(QueueUrl=dlq_url)
print(f"DLQ has {len(dlq_msgs.get('Messages', []))} messages")
```

### Hands-On: FIFO Queue

FIFO queues guarantee order and exactly-once delivery — useful for financial transactions.

```python
fifo_queue = sqs.create_queue(
    QueueName="payments.fifo",  # must end in .fifo
    Attributes={
        "FifoQueue": "true",
        "ContentBasedDeduplication": "true",
    },
)
fifo_url = fifo_queue["QueueUrl"]

# Send ordered messages
for i in range(3):
    sqs.send_message(
        QueueUrl=fifo_url,
        MessageBody=json.dumps({"step": i, "action": "charge"}),
        MessageGroupId="user-123",  # messages in same group are ordered
    )

# Receive in order
while True:
    msgs = sqs.receive_message(QueueUrl=fifo_url, MaxNumberOfMessages=1)
    if not msgs.get("Messages"):
        break
    msg = msgs["Messages"][0]
    print(json.loads(msg["Body"]))  # step 0, then 1, then 2
    sqs.delete_message(QueueUrl=fifo_url, ReceiptHandle=msg["ReceiptHandle"])
```

---

## SNS — Simple Notification Service

SNS is a **pub/sub** system. One publisher sends a message to a **topic**, and all subscribers receive it simultaneously. Unlike SQS (one consumer), SNS fans out to many.

**When to use it:** notifications, broadcasting events to multiple systems, triggering multiple Lambdas from one event.

### Core Concepts

- **Topic** — a named channel. Publishers send to it, subscribers receive from it.
- **Subscription** — a connection between a topic and an endpoint (SQS queue, Lambda, email, HTTP, etc.)
- **Fan-out** — one SNS publish → many SQS queues / Lambdas receive it simultaneously

### Hands-On: Basic Topic

```python
sns = boto3.client("sns",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a topic
topic = sns.create_topic(Name="user-events")
topic_arn = topic["TopicArn"]

# Subscribe an SQS queue to the topic
queue = sqs.create_queue(QueueName="user-events-processor")
queue_url = queue["QueueUrl"]
queue_arn = sqs.get_queue_attributes(
    QueueUrl=queue_url,
    AttributeNames=["QueueArn"],
)["Attributes"]["QueueArn"]

sns.subscribe(
    TopicArn=topic_arn,
    Protocol="sqs",
    Endpoint=queue_arn,
)

# Publish to the topic
sns.publish(
    TopicArn=topic_arn,
    Message=json.dumps({"event": "user.signup", "userId": "u-123"}),
    Subject="User Signup",
)

# The SQS queue now has the message
msgs = sqs.receive_message(QueueUrl=queue_url)
for msg in msgs.get("Messages", []):
    # SNS wraps the message in an envelope
    envelope = json.loads(msg["Body"])
    payload = json.loads(envelope["Message"])
    print(payload)  # {"event": "user.signup", "userId": "u-123"}
    sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=msg["ReceiptHandle"])
```

### Hands-On: Fan-Out Pattern

One event → multiple independent processors. Classic microservices pattern.

```python
# Create topic
topic = sns.create_topic(Name="order-placed")
topic_arn = topic["TopicArn"]

# Create multiple queues for different services
services = ["inventory", "billing", "notifications", "analytics"]
queues = {}
for svc in services:
    q = sqs.create_queue(QueueName=f"orders-{svc}")
    q_url = q["QueueUrl"]
    q_arn = sqs.get_queue_attributes(
        QueueUrl=q_url, AttributeNames=["QueueArn"]
    )["Attributes"]["QueueArn"]
    sns.subscribe(TopicArn=topic_arn, Protocol="sqs", Endpoint=q_arn)
    queues[svc] = q_url

# One publish reaches ALL four queues simultaneously
sns.publish(
    TopicArn=topic_arn,
    Message=json.dumps({"orderId": "ord-999", "total": 149.99}),
)

# Each service independently processes the same event
for svc, q_url in queues.items():
    msgs = sqs.receive_message(QueueUrl=q_url)
    count = len(msgs.get("Messages", []))
    print(f"{svc}: received {count} message(s)")
    # inventory: received 1 message(s)
    # billing: received 1 message(s)
    # notifications: received 1 message(s)
    # analytics: received 1 message(s)
```

### Hands-On: SNS Filter Policies

Subscribers can filter — only receive messages matching certain attributes.

```python
topic = sns.create_topic(Name="all-events")
topic_arn = topic["TopicArn"]

# Create two queues
critical_q = sqs.create_queue(QueueName="critical-alerts")
info_q = sqs.create_queue(QueueName="info-logs")

def get_arn(url):
    return sqs.get_queue_attributes(
        QueueUrl=url, AttributeNames=["QueueArn"]
    )["Attributes"]["QueueArn"]

# Subscribe with filter — critical queue only gets severity=critical
sns.subscribe(
    TopicArn=topic_arn,
    Protocol="sqs",
    Endpoint=get_arn(critical_q["QueueUrl"]),
    Attributes={
        "FilterPolicy": json.dumps({"severity": ["critical"]}),
    },
)

# Info queue gets everything
sns.subscribe(
    TopicArn=topic_arn,
    Protocol="sqs",
    Endpoint=get_arn(info_q["QueueUrl"]),
)

# Publish with message attributes
sns.publish(
    TopicArn=topic_arn,
    Message="disk almost full",
    MessageAttributes={
        "severity": {"DataType": "String", "StringValue": "critical"},
    },
)
sns.publish(
    TopicArn=topic_arn,
    Message="user logged in",
    MessageAttributes={
        "severity": {"DataType": "String", "StringValue": "info"},
    },
)

# critical queue: 1 message (only the critical one)
# info queue: 2 messages (both)
```

## What's Different on Real AWS

- **SQS long polling** — always use `WaitTimeSeconds=20` in production to reduce costs and empty responses.
- **Message size** — SQS max is 256KB. For larger payloads, store in S3 and send the S3 key.
- **SNS email subscriptions** — require confirmation click. MiniStack auto-confirms.
- **IAM** — on real AWS, the SQS queue needs a resource policy allowing SNS to send to it. MiniStack skips this.
- **FIFO SNS** — SNS also has FIFO topics for ordered fan-out. Supported in MiniStack.

## Next Steps

- Combine with **Lambda** — subscribe a Lambda to an SNS topic for serverless event processing.
- Combine with **S3** — S3 can publish events to SNS when objects are created/deleted.
- Learn **EventBridge** for more powerful event routing with content-based filtering.
