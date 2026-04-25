---
inclusion: manual
---

# Learn Lambda with MiniStack

AWS Lambda is **serverless compute** — you write a function, AWS runs it when triggered. No servers to manage, no idle costs. You pay only for the milliseconds your code runs.

## Core Concepts

- **Function** — your code + runtime + configuration
- **Handler** — the entry point: `module.function_name` (e.g. `index.handler`)
- **Event** — the input passed to your function (JSON)
- **Context** — metadata about the invocation (request ID, timeout remaining, etc.)
- **Runtime** — the language environment (Python 3.12, Node.js 22, Java 21, etc.)
- **Trigger** — what invokes your function (API Gateway, SQS, S3, EventBridge, etc.)
- **Layers** — shared code/libraries attached to multiple functions

## Hands-On: Your First Lambda

```python
import boto3, zipfile, io, json

lam = boto3.client("lambda",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Package the function code as a zip
def make_zip(code: str) -> bytes:
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w") as zf:
        zf.writestr("index.py", code)
    return buf.getvalue()

# The simplest possible Lambda
code = """
def handler(event, context):
    name = event.get("name", "World")
    return {"message": f"Hello, {name}!"}
"""

lam.create_function(
    FunctionName="hello",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(code)},
)

# Invoke it
response = lam.invoke(
    FunctionName="hello",
    Payload=json.dumps({"name": "MiniStack"}),
)
result = json.loads(response["Payload"].read())
print(result)  # {"message": "Hello, MiniStack!"}
```

## Hands-On: Environment Variables

```python
code = """
import os

def handler(event, context):
    db_host = os.environ["DB_HOST"]
    env = os.environ.get("ENVIRONMENT", "dev")
    return {"db": db_host, "env": env}
"""

lam.create_function(
    FunctionName="config-demo",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(code)},
    Environment={
        "Variables": {
            "DB_HOST": "localhost:5432",
            "ENVIRONMENT": "production",
        }
    },
)

response = lam.invoke(FunctionName="config-demo", Payload=b"{}")
print(json.loads(response["Payload"].read()))
# {"db": "localhost:5432", "env": "production"}
```

## Hands-On: Error Handling

```python
code = """
def handler(event, context):
    if event.get("fail"):
        raise ValueError("Something went wrong!")
    return {"ok": True}
"""

lam.create_function(
    FunctionName="error-demo",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(code)},
)

# Successful invocation
response = lam.invoke(FunctionName="error-demo", Payload=b"{}")
print(response.get("FunctionError"))  # None
print(json.loads(response["Payload"].read()))  # {"ok": True}

# Failed invocation — Lambda returns 200 but sets FunctionError header
response = lam.invoke(
    FunctionName="error-demo",
    Payload=json.dumps({"fail": True}),
)
print(response.get("FunctionError"))  # "Unhandled"
error = json.loads(response["Payload"].read())
print(error["errorType"])    # ValueError
print(error["errorMessage"]) # Something went wrong!
```

## Hands-On: Async Invocation (Event)

Fire-and-forget — Lambda queues the event and returns immediately. AWS retries on failure.

```python
# InvocationType="Event" returns 202 immediately
response = lam.invoke(
    FunctionName="hello",
    InvocationType="Event",
    Payload=json.dumps({"name": "async"}),
)
print(response["StatusCode"])  # 202
```

## Hands-On: Versions and Aliases

Versions are immutable snapshots. Aliases point to a version — use them for blue/green deployments.

```python
code_v1 = make_zip('def handler(e, c): return {"version": 1}')
code_v2 = make_zip('def handler(e, c): return {"version": 2}')

lam.create_function(
    FunctionName="versioned",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": code_v1},
)

# Publish version 1
lam.publish_version(FunctionName="versioned")

# Update code and publish version 2
lam.update_function_code(FunctionName="versioned", ZipFile=code_v2)
lam.publish_version(FunctionName="versioned")

# Create aliases
lam.create_alias(FunctionName="versioned", Name="prod", FunctionVersion="1")
lam.create_alias(FunctionName="versioned", Name="staging", FunctionVersion="2")

# Invoke specific alias
response = lam.invoke(FunctionName="versioned:prod", Payload=b"{}")
print(json.loads(response["Payload"].read()))  # {"version": 1}

response = lam.invoke(FunctionName="versioned:staging", Payload=b"{}")
print(json.loads(response["Payload"].read()))  # {"version": 2}
```

## Hands-On: SQS Event Source Mapping

Lambda automatically polls an SQS queue and invokes your function with batches of messages.

```python
sqs = boto3.client("sqs",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create queue
q = sqs.create_queue(QueueName="jobs")
q_url = q["QueueUrl"]
q_arn = sqs.get_queue_attributes(
    QueueUrl=q_url, AttributeNames=["QueueArn"]
)["Attributes"]["QueueArn"]

# Lambda that processes SQS messages
code = """
import json

def handler(event, context):
    for record in event["Records"]:
        body = json.loads(record["body"])
        print(f"Processing job: {body}")
    return {"processed": len(event["Records"])}
"""

lam.create_function(
    FunctionName="job-processor",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(code)},
)

# Wire SQS → Lambda
lam.create_event_source_mapping(
    FunctionName="job-processor",
    EventSourceArn=q_arn,
    BatchSize=5,
    Enabled=True,
)

# Send messages — Lambda will process them automatically
for i in range(3):
    sqs.send_message(
        QueueUrl=q_url,
        MessageBody=json.dumps({"jobId": f"job-{i}", "type": "resize-image"}),
    )

import time; time.sleep(2)  # wait for poller to pick them up
```

## Hands-On: Lambda Layers

Share code across multiple functions without bundling it in each zip.

```python
# Create a layer with shared utilities
layer_code = """
def format_response(data, status=200):
    return {"statusCode": status, "body": str(data)}
"""

layer_zip = io.BytesIO()
with zipfile.ZipFile(layer_zip, "w") as zf:
    zf.writestr("python/utils.py", layer_code)
layer_zip.seek(0)

layer = lam.publish_layer_version(
    LayerName="shared-utils",
    Content={"ZipFile": layer_zip.read()},
    CompatibleRuntimes=["python3.12"],
)
layer_arn = layer["LayerVersionArn"]

# Use the layer in a function
code = """
from utils import format_response

def handler(event, context):
    return format_response({"hello": "world"})
"""

lam.create_function(
    FunctionName="uses-layer",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(code)},
    Layers=[layer_arn],
)

response = lam.invoke(FunctionName="uses-layer", Payload=b"{}")
print(json.loads(response["Payload"].read()))
# {"statusCode": 200, "body": "{'hello': 'world'}"}
```

## What's Different on Real AWS

- **Cold starts** — first invocation after idle spins up a new container (~100ms–1s). MiniStack has no cold start delay by default.
- **Timeout** — max 15 minutes. Default is 3 seconds. Always set it explicitly.
- **Memory** — 128MB to 10GB. More memory = more CPU. MiniStack ignores memory limits.
- **Concurrency** — AWS default is 1000 concurrent executions per account. MiniStack is unbounded.
- **IAM execution role** — on real AWS, your Lambda needs a role with permissions to call other services. MiniStack skips auth.
- **VPC** — Lambda can run inside a VPC to access RDS/ElastiCache. Adds ~1s cold start. MiniStack ignores VPC config.
- **Pricing** — $0.20/million requests + $0.0000166667/GB-second. First 1M requests/month free.

## Next Steps

- Connect Lambda to **API Gateway** to build a REST API.
- Use **EventBridge** to trigger Lambda on a schedule or from other services.
- Try **Step Functions** to orchestrate multiple Lambda calls into a workflow.
