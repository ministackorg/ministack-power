---
inclusion: manual
---

# Learn EventBridge with MiniStack

Amazon EventBridge is a **serverless event bus**. Services publish events, rules match them, and targets receive them. It's the backbone of event-driven architectures on AWS.

## Core Concepts

- **Event bus** — a channel that receives events. `default` is always there. You can create custom ones.
- **Event** — a JSON object with `source`, `detail-type`, `detail`, and metadata
- **Rule** — matches events by pattern and routes them to targets
- **Target** — where matched events go (Lambda, SQS, SNS, Step Functions, etc.)
- **Event pattern** — a JSON filter that matches events by field values

## Hands-On: Your First Event Bus and Rule

```python
import boto3, json

eb = boto3.client("events",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)
sqs = boto3.client("sqs",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create an SQS queue as the target
q = sqs.create_queue(QueueName="order-events")
q_url = q["QueueUrl"]
q_arn = sqs.get_queue_attributes(
    QueueUrl=q_url, AttributeNames=["QueueArn"]
)["Attributes"]["QueueArn"]

# Create a rule that matches order events
eb.put_rule(
    Name="order-placed-rule",
    EventPattern=json.dumps({
        "source": ["myapp.orders"],
        "detail-type": ["OrderPlaced"],
    }),
    State="ENABLED",
)

# Add the SQS queue as a target
eb.put_targets(
    Rule="order-placed-rule",
    Targets=[{"Id": "order-queue", "Arn": q_arn}],
)

# Publish an event
eb.put_events(Entries=[{
    "Source": "myapp.orders",
    "DetailType": "OrderPlaced",
    "Detail": json.dumps({
        "orderId": "ord-001",
        "customerId": "cust-123",
        "total": 99.99,
    }),
    "EventBusName": "default",
}])

# The SQS queue now has the event
import time; time.sleep(0.5)
msgs = sqs.receive_message(QueueUrl=q_url)
for msg in msgs.get("Messages", []):
    envelope = json.loads(msg["Body"])
    detail = json.loads(envelope["detail"])
    print(f"Order received: {detail['orderId']}")
    sqs.delete_message(QueueUrl=q_url, ReceiptHandle=msg["ReceiptHandle"])
```

## Hands-On: Content-Based Filtering

EventBridge rules can filter on any field in the event detail — not just source and detail-type.

```python
# Create two queues — one for high-value orders, one for all orders
high_value_q = sqs.create_queue(QueueName="high-value-orders")
all_orders_q = sqs.create_queue(QueueName="all-orders")

def get_arn(url):
    return sqs.get_queue_attributes(
        QueueUrl=url, AttributeNames=["QueueArn"]
    )["Attributes"]["QueueArn"]

# Rule for high-value orders (total > 500)
eb.put_rule(
    Name="high-value-orders",
    EventPattern=json.dumps({
        "source": ["myapp.orders"],
        "detail-type": ["OrderPlaced"],
        "detail": {
            "total": [{"numeric": [">", 500]}],
        },
    }),
    State="ENABLED",
)
eb.put_targets(
    Rule="high-value-orders",
    Targets=[{"Id": "high-value-queue", "Arn": get_arn(high_value_q["QueueUrl"])}],
)

# Rule for all orders
eb.put_rule(
    Name="all-orders",
    EventPattern=json.dumps({
        "source": ["myapp.orders"],
        "detail-type": ["OrderPlaced"],
    }),
    State="ENABLED",
)
eb.put_targets(
    Rule="all-orders",
    Targets=[{"Id": "all-orders-queue", "Arn": get_arn(all_orders_q["QueueUrl"])}],
)

# Send a low-value order
eb.put_events(Entries=[{
    "Source": "myapp.orders",
    "DetailType": "OrderPlaced",
    "Detail": json.dumps({"orderId": "ord-001", "total": 49.99}),
}])

# Send a high-value order
eb.put_events(Entries=[{
    "Source": "myapp.orders",
    "DetailType": "OrderPlaced",
    "Detail": json.dumps({"orderId": "ord-002", "total": 999.99}),
}])

time.sleep(0.5)
# all-orders queue: 2 messages
# high-value-orders queue: 1 message (only ord-002)
```

## Hands-On: Custom Event Bus

Use custom buses to isolate events by domain or team.

```python
# Create a custom event bus for your payments domain
eb.create_event_bus(Name="payments")

# Create a rule on the custom bus
eb.put_rule(
    Name="payment-failed",
    EventBusName="payments",
    EventPattern=json.dumps({
        "source": ["payments.processor"],
        "detail-type": ["PaymentFailed"],
    }),
    State="ENABLED",
)

alert_q = sqs.create_queue(QueueName="payment-alerts")
eb.put_targets(
    Rule="payment-failed",
    EventBusName="payments",
    Targets=[{"Id": "alerts", "Arn": get_arn(alert_q["QueueUrl"])}],
)

# Publish to the custom bus
eb.put_events(Entries=[{
    "Source": "payments.processor",
    "DetailType": "PaymentFailed",
    "Detail": json.dumps({"paymentId": "pay-001", "reason": "insufficient_funds"}),
    "EventBusName": "payments",
}])
```

## Hands-On: EventBridge + Lambda

The most common pattern — Lambda reacts to events.

```python
import zipfile, io

lam = boto3.client("lambda",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

code = """
import json

def handler(event, context):
    detail = event.get("detail", {})
    print(f"Processing event: {event['detail-type']}")
    print(f"Detail: {json.dumps(detail)}")
    return {"processed": True}
"""

buf = io.BytesIO()
with zipfile.ZipFile(buf, "w") as zf:
    zf.writestr("index.py", code)

lam.create_function(
    FunctionName="event-processor",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": buf.getvalue()},
)

fn_arn = f"arn:aws:lambda:us-east-1:000000000000:function:event-processor"

# Create rule targeting Lambda
eb.put_rule(
    Name="process-user-events",
    EventPattern=json.dumps({"source": ["myapp.users"]}),
    State="ENABLED",
)
eb.put_targets(
    Rule="process-user-events",
    Targets=[{"Id": "lambda-target", "Arn": fn_arn}],
)

# Publish — Lambda is invoked automatically
eb.put_events(Entries=[{
    "Source": "myapp.users",
    "DetailType": "UserSignup",
    "Detail": json.dumps({"userId": "u-001", "email": "alice@example.com"}),
}])
```

## Hands-On: TestEventPattern

Test whether an event matches a pattern without publishing it.

```python
result = eb.test_event_pattern(
    EventPattern=json.dumps({
        "source": ["myapp.orders"],
        "detail": {"status": ["pending", "processing"]},
    }),
    Event=json.dumps({
        "source": "myapp.orders",
        "detail-type": "OrderUpdated",
        "detail": {"orderId": "ord-001", "status": "processing"},
    }),
)
print(result["Result"])  # True

result = eb.test_event_pattern(
    EventPattern=json.dumps({
        "source": ["myapp.orders"],
        "detail": {"status": ["pending", "processing"]},
    }),
    Event=json.dumps({
        "source": "myapp.orders",
        "detail-type": "OrderUpdated",
        "detail": {"orderId": "ord-002", "status": "shipped"},
    }),
)
print(result["Result"])  # False — "shipped" not in the pattern
```

## What's Different on Real AWS

- **Schedule rules** — EventBridge can trigger on a cron/rate schedule (e.g. every 5 minutes). MiniStack stores the rule but doesn't fire it on schedule.
- **Cross-account event buses** — you can send events to a bus in another account. MiniStack supports this via multi-tenancy.
- **Event replay** — archive events and replay them later. MiniStack stores archives as stubs.
- **API destinations** — send events to external HTTP endpoints. MiniStack stores the config but doesn't make HTTP calls.
- **Pricing** — $1/million custom events. First 14M events/month free.
- **IAM** — targets need resource policies allowing EventBridge to invoke them. MiniStack skips this.

## Next Steps

- Combine with **Step Functions** — trigger a workflow from an EventBridge event.
- Use **Kinesis** for high-throughput event streaming (millions of events/second).
- Learn **SNS** for simpler pub/sub without content-based filtering.
