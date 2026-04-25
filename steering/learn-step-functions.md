---
inclusion: manual
---

# Learn Step Functions with MiniStack

AWS Step Functions lets you **orchestrate** multiple services into workflows. Instead of writing complex retry/error-handling logic in your code, you define it as a state machine — a visual workflow that AWS executes and monitors.

## Core Concepts

- **State machine** — the workflow definition (JSON/YAML using Amazon States Language)
- **State** — a step in the workflow (Task, Choice, Wait, Pass, Succeed, Fail, Parallel, Map)
- **Execution** — one run of a state machine with specific input
- **Task state** — calls a Lambda, ECS task, or other AWS service
- **Choice state** — branches based on conditions (like an if/else)
- **Wait state** — pauses for a duration or until a timestamp
- **Parallel state** — runs multiple branches simultaneously
- **Map state** — runs the same steps for each item in an array

## Hands-On: Your First State Machine

```python
import boto3, json, time

sfn = boto3.client("stepfunctions",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)
lam = boto3.client("lambda",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a simple Lambda to call
import zipfile, io

def make_zip(code):
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w") as zf:
        zf.writestr("index.py", code)
    return buf.getvalue()

lam.create_function(
    FunctionName="greet",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(
        'def handler(e, c): return {"greeting": f"Hello, {e[\"name\"]}!"}'
    )},
)

fn_arn = "arn:aws:lambda:us-east-1:000000000000:function:greet"

# Define the state machine
definition = {
    "Comment": "A simple greeting workflow",
    "StartAt": "Greet",
    "States": {
        "Greet": {
            "Type": "Task",
            "Resource": fn_arn,
            "End": True,
        }
    },
}

sm = sfn.create_state_machine(
    name="greeting-workflow",
    definition=json.dumps(definition),
    roleArn="arn:aws:iam::000000000000:role/sfn-role",
)
sm_arn = sm["stateMachineArn"]

# Start an execution
execution = sfn.start_execution(
    stateMachineArn=sm_arn,
    input=json.dumps({"name": "Alice"}),
)
exec_arn = execution["executionArn"]

# Wait for completion
for _ in range(20):
    desc = sfn.describe_execution(executionArn=exec_arn)
    if desc["status"] != "RUNNING":
        break
    time.sleep(0.5)

print(desc["status"])  # SUCCEEDED
print(json.loads(desc["output"]))  # {"greeting": "Hello, Alice!"}
```

## Hands-On: Choice State (Branching)

```python
lam.create_function(
    FunctionName="approve-order",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(
        'def handler(e, c): return {**e, "approved": True}'
    )},
)
lam.create_function(
    FunctionName="reject-order",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(
        'def handler(e, c): return {**e, "approved": False, "reason": "amount too high"}'
    )},
)

definition = {
    "StartAt": "CheckAmount",
    "States": {
        "CheckAmount": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.amount",
                    "NumericLessThanEquals": 1000,
                    "Next": "ApproveOrder",
                },
            ],
            "Default": "RejectOrder",
        },
        "ApproveOrder": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:approve-order",
            "End": True,
        },
        "RejectOrder": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:reject-order",
            "End": True,
        },
    },
}

sm = sfn.create_state_machine(
    name="order-approval",
    definition=json.dumps(definition),
    roleArn="arn:aws:iam::000000000000:role/sfn-role",
)

def run(amount):
    exec = sfn.start_execution(
        stateMachineArn=sm["stateMachineArn"],
        input=json.dumps({"orderId": "ord-001", "amount": amount}),
    )
    for _ in range(20):
        desc = sfn.describe_execution(executionArn=exec["executionArn"])
        if desc["status"] != "RUNNING":
            break
        time.sleep(0.5)
    return json.loads(desc["output"])

print(run(500))   # {"approved": True, ...}
print(run(2000))  # {"approved": False, "reason": "amount too high", ...}
```

## Hands-On: Retry and Catch

```python
lam.create_function(
    FunctionName="flaky-service",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
import random
def handler(e, c):
    if random.random() < 0.7:  # 70% chance of failure
        raise Exception("Service temporarily unavailable")
    return {"result": "success"}
""")},
)

lam.create_function(
    FunctionName="fallback",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(
        'def handler(e, c): return {"result": "fallback", "error": e.get("Cause", "")}'
    )},
)

definition = {
    "StartAt": "CallFlakyService",
    "States": {
        "CallFlakyService": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:flaky-service",
            "Retry": [{
                "ErrorEquals": ["States.ALL"],
                "IntervalSeconds": 1,
                "MaxAttempts": 3,
                "BackoffRate": 2,  # 1s, 2s, 4s
            }],
            "Catch": [{
                "ErrorEquals": ["States.ALL"],
                "Next": "Fallback",
                "ResultPath": "$.error",
            }],
            "End": True,
        },
        "Fallback": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:fallback",
            "End": True,
        },
    },
}

sm = sfn.create_state_machine(
    name="resilient-workflow",
    definition=json.dumps(definition),
    roleArn="arn:aws:iam::000000000000:role/sfn-role",
)
```

## Hands-On: Parallel State

Run multiple branches simultaneously, wait for all to complete.

```python
definition = {
    "StartAt": "ProcessInParallel",
    "States": {
        "ProcessInParallel": {
            "Type": "Parallel",
            "Branches": [
                {
                    "StartAt": "SendEmail",
                    "States": {
                        "SendEmail": {
                            "Type": "Pass",
                            "Result": {"sent": True},
                            "End": True,
                        }
                    },
                },
                {
                    "StartAt": "UpdateDatabase",
                    "States": {
                        "UpdateDatabase": {
                            "Type": "Pass",
                            "Result": {"updated": True},
                            "End": True,
                        }
                    },
                },
                {
                    "StartAt": "NotifySlack",
                    "States": {
                        "NotifySlack": {
                            "Type": "Pass",
                            "Result": {"notified": True},
                            "End": True,
                        }
                    },
                },
            ],
            "End": True,
        }
    },
}

sm = sfn.create_state_machine(
    name="parallel-workflow",
    definition=json.dumps(definition),
    roleArn="arn:aws:iam::000000000000:role/sfn-role",
)

exec = sfn.start_execution(
    stateMachineArn=sm["stateMachineArn"],
    input="{}",
)
time.sleep(1)
desc = sfn.describe_execution(executionArn=exec["executionArn"])
print(json.loads(desc["output"]))
# [{"sent": True}, {"updated": True}, {"notified": True}]
```

## Hands-On: Map State (Process Arrays)

```python
lam.create_function(
    FunctionName="process-item",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(
        'def handler(e, c): return {"itemId": e["id"], "processed": True}'
    )},
)

definition = {
    "StartAt": "ProcessAll",
    "States": {
        "ProcessAll": {
            "Type": "Map",
            "ItemsPath": "$.items",
            "MaxConcurrency": 3,
            "Iterator": {
                "StartAt": "ProcessOne",
                "States": {
                    "ProcessOne": {
                        "Type": "Task",
                        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:process-item",
                        "End": True,
                    }
                },
            },
            "End": True,
        }
    },
}

sm = sfn.create_state_machine(
    name="batch-processor",
    definition=json.dumps(definition),
    roleArn="arn:aws:iam::000000000000:role/sfn-role",
)

exec = sfn.start_execution(
    stateMachineArn=sm["stateMachineArn"],
    input=json.dumps({"items": [{"id": 1}, {"id": 2}, {"id": 3}]}),
)
time.sleep(2)
desc = sfn.describe_execution(executionArn=exec["executionArn"])
print(json.loads(desc["output"]))
# [{"itemId": 1, "processed": True}, {"itemId": 2, ...}, {"itemId": 3, ...}]
```

## What's Different on Real AWS

- **Wait states** — on real AWS, a 60-second Wait actually waits 60 seconds. Use `SFN_WAIT_SCALE=0` in MiniStack to skip waits in tests: `curl -X POST localhost:4566/_ministack/config -d '{"stepfunctions._SFN_WAIT_SCALE": 0}'`
- **Express vs Standard** — Standard workflows run up to 1 year, exactly-once. Express are high-throughput, at-least-once, max 5 minutes. MiniStack treats both the same.
- **Pricing** — Standard: $0.025/1000 state transitions. Express: $1/million executions + duration.
- **Execution history** — Standard keeps full history for 90 days. Express only via CloudWatch Logs.
- **IAM** — the state machine role needs permissions to invoke Lambda, call services, etc.

## Next Steps

- Trigger Step Functions from **EventBridge** for event-driven workflows.
- Use **waitForTaskToken** for human approval steps or long-running external processes.
- Combine with **DynamoDB** to track workflow state across executions.
