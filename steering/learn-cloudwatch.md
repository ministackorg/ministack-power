---
inclusion: manual
---

# Learn CloudWatch with MiniStack

CloudWatch is AWS's **observability service**: metrics (time-series numbers),
alarms (rules that fire on metrics), and logs (structured log streams). It's
how every other AWS service tells you what's going on.

**When to use it:** monitor application health, alert on thresholds, debug
Lambda/ECS/ALB issues, drive autoscaling, write custom dashboards.

MiniStack emulates all three pieces locally: `cloudwatch` (metrics + alarms),
`cloudwatch-logs` (log groups and streams). Other services publish their
metrics into the same store, so you can see Lambda invocation counts, SQS
queue depth, etc., just like in real AWS.

---

## Part 1 — Metrics

A **metric** is a stream of (timestamp, value) datapoints in a **namespace**
with optional **dimensions**.

### Hands-On: publish + read custom metrics

```python
import boto3
import time

cw = boto3.client("cloudwatch",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test", aws_secret_access_key="test",
    region_name="us-east-1",
)

# Publish two datapoints
cw.put_metric_data(
    Namespace="MyApp",
    MetricData=[
        {"MetricName": "Requests", "Value": 1.0, "Unit": "Count",
         "Dimensions": [{"Name": "Service", "Value": "api"}]},
        {"MetricName": "Latency", "Value": 123.0, "Unit": "Milliseconds",
         "Dimensions": [{"Name": "Service", "Value": "api"}]},
    ],
)

# Read it back
end = time.time()
stats = cw.get_metric_statistics(
    Namespace="MyApp",
    MetricName="Requests",
    Dimensions=[{"Name": "Service", "Value": "api"}],
    StartTime=end - 600, EndTime=end,
    Period=60, Statistics=["Sum", "Average"],
)
print(stats["Datapoints"])
```

### Service-published metrics

When you invoke a Lambda or push to SQS, MiniStack automatically records the
canonical AWS metrics — exactly like real AWS. You can read them without
publishing anything yourself:

| Namespace | Metrics |
|---|---|
| `AWS/Lambda` | `Invocations`, `Errors`, `Duration`, `Throttles` |
| `AWS/SQS` | `ApproximateNumberOfMessagesVisible`, `NumberOfMessagesSent`, `NumberOfMessagesReceived` |
| `AWS/S3` | `BucketSizeBytes`, `NumberOfObjects` |
| `AWS/ApplicationELB` | `RequestCount`, `TargetResponseTime` |

```python
# After a few Lambda invocations, query the canonical metric:
stats = cw.get_metric_statistics(
    Namespace="AWS/Lambda",
    MetricName="Invocations",
    Dimensions=[{"Name": "FunctionName", "Value": "my-fn"}],
    StartTime=end - 600, EndTime=end,
    Period=60, Statistics=["Sum"],
)
```

---

## Part 2 — Alarms

An **alarm** watches a metric and changes state (`OK` ↔ `ALARM` ↔
`INSUFFICIENT_DATA`) when a threshold is crossed. On state transition it
dispatches **actions** — typically SNS notifications, but also AutoScaling
policies, EC2 actions, or SSM automations.

### Hands-On: alarm fires an SNS notification

```python
import boto3, json

cw  = boto3.client("cloudwatch",  endpoint_url="http://localhost:4566",
                   aws_access_key_id="test", aws_secret_access_key="test",
                   region_name="us-east-1")
sns = boto3.client("sns",         endpoint_url="http://localhost:4566",
                   aws_access_key_id="test", aws_secret_access_key="test",
                   region_name="us-east-1")
sqs = boto3.client("sqs",         endpoint_url="http://localhost:4566",
                   aws_access_key_id="test", aws_secret_access_key="test",
                   region_name="us-east-1")

# 1. Set up SNS → SQS plumbing so we can see the alarm message arrive.
topic_arn = sns.create_topic(Name="alerts")["TopicArn"]
queue_url = sqs.create_queue(QueueName="alerts-q")["QueueUrl"]
queue_arn = sqs.get_queue_attributes(
    QueueUrl=queue_url, AttributeNames=["QueueArn"],
)["Attributes"]["QueueArn"]
sns.subscribe(TopicArn=topic_arn, Protocol="sqs", Endpoint=queue_arn)

# 2. Create an alarm that fires when "Errors" sum > 0, action = publish to SNS.
cw.put_metric_alarm(
    AlarmName="too-many-errors",
    MetricName="Errors", Namespace="MyApp",
    Statistic="Sum", Period=60, EvaluationPeriods=1,
    Threshold=0.0, ComparisonOperator="GreaterThanThreshold",
    ActionsEnabled=True,
    AlarmActions=[topic_arn],
    OKActions=[topic_arn],
)

# 3. Force the alarm into ALARM state.
cw.set_alarm_state(
    AlarmName="too-many-errors",
    StateValue="ALARM",
    StateReason="forced for test",
)

# 4. Read the SNS notification off the queue.
msgs = sqs.receive_message(QueueUrl=queue_url, WaitTimeSeconds=2)["Messages"]
envelope = json.loads(msgs[0]["Body"])
payload  = json.loads(envelope["Message"])
print(payload["NewStateValue"], "—", payload["NewStateReason"])
# → ALARM — forced for test
```

The payload shape matches real AWS: `AlarmName`, `NewStateValue`,
`NewStateReason`, `StateChangeTime`, `Region`, `OldStateValue`, and a `Trigger`
sub-object with the metric/threshold/comparison.

### Disabling actions

`ActionsEnabled=False` (or calling `disable_alarm_actions`) stops dispatch
without deleting the alarm. The state still transitions and history still
records, but no SNS notification fires.

---

## Part 3 — Logs

CloudWatch Logs stores structured logs in **log groups** (one per
application/service) containing **log streams** (one per instance/run).

### Hands-On: write and read logs

```python
import boto3, time

logs = boto3.client("logs",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test", aws_secret_access_key="test",
    region_name="us-east-1",
)

logs.create_log_group(logGroupName="/my/app")
logs.create_log_stream(logGroupName="/my/app", logStreamName="run-001")

logs.put_log_events(
    logGroupName="/my/app",
    logStreamName="run-001",
    logEvents=[
        {"timestamp": int(time.time() * 1000), "message": "starting up"},
        {"timestamp": int(time.time() * 1000), "message": "ready"},
    ],
)

events = logs.get_log_events(
    logGroupName="/my/app", logStreamName="run-001",
)["events"]
for e in events:
    print(e["message"])
```

### Lambda auto-logs

When you invoke a Lambda, its `print()` / `console.log` output is captured
into `/aws/lambda/<FunctionName>` automatically — no setup needed. After an
invocation:

```python
logs.filter_log_events(logGroupName="/aws/lambda/my-fn")
```

### Filter patterns

```python
# Find error lines from the last 5 minutes
result = logs.filter_log_events(
    logGroupName="/my/app",
    filterPattern="ERROR",
    startTime=int((time.time() - 300) * 1000),
)
for e in result["events"]:
    print(e["message"])
```

---

## Common Gotchas

- **Metrics are aggregated by Period**: `Period=60` buckets datapoints into
  1-minute bins. If you publish two values in the same minute and ask for
  `Sum`, you get one bucket with both summed; ask for `Average` and you get
  the mean. This matches real CloudWatch.
- **`StateValue` changes only fire actions on transition**: setting `ALARM`
  twice in a row dispatches once. Recovery (`ALARM → OK`) fires `OKActions`.
- **Alarms evaluate on `put_metric_data`**: when you publish a metric that
  an alarm watches, the alarm re-evaluates immediately. In real AWS this is
  asynchronous (~minute latency); in MiniStack it's synchronous.
- **Log retention is not enforced**: `put_retention_policy` is accepted but
  events are kept until reset.

---

## Reset

```python
import requests
requests.post("http://localhost:4566/_ministack/reset")
```

Clears all alarms, metrics, log groups, and history.
