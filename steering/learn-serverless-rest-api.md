---
inclusion: manual
---

# Build a Serverless REST API with MiniStack

This is the most common serverless architecture on AWS: **API Gateway → Lambda → DynamoDB**. No servers, no idle costs, scales automatically from 0 to millions of requests.

```
Client → API Gateway → Lambda → DynamoDB
```

We'll build a complete CRUD API for a **Tasks** app from scratch.

---

## Step 1: Set Up the Database (DynamoDB)

```python
import boto3, json, zipfile, io, time

# Clients
def client(service):
    return boto3.client(service,
        endpoint_url="http://localhost:4566",
        aws_access_key_id="test",
        aws_secret_access_key="test",
        region_name="us-east-1",
    )

ddb = client("dynamodb")
lam = client("lambda")
apigw = client("apigatewayv2")

# Create the Tasks table
ddb.create_table(
    TableName="Tasks",
    KeySchema=[
        {"AttributeName": "userId", "KeyType": "HASH"},
        {"AttributeName": "taskId", "KeyType": "RANGE"},
    ],
    AttributeDefinitions=[
        {"AttributeName": "userId", "AttributeType": "S"},
        {"AttributeName": "taskId", "AttributeType": "S"},
    ],
    BillingMode="PAY_PER_REQUEST",
)
print("✓ DynamoDB table created")
```

---

## Step 2: Write the Lambda Handler

One Lambda handles all routes — a common pattern called a "monolambda".

```python
def make_zip(code: str) -> bytes:
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w") as zf:
        zf.writestr("index.py", code)
    return buf.getvalue()

handler_code = """
import json
import boto3
import uuid
import time
from boto3.dynamodb.conditions import Key

ddb = boto3.resource("dynamodb",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)
table = ddb.Table("Tasks")


def response(status, body):
    return {
        "statusCode": status,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
        },
        "body": json.dumps(body),
    }


def handler(event, context):
    method = event["requestContext"]["http"]["method"]
    path   = event["rawPath"]
    params = event.get("pathParameters") or {}
    body   = json.loads(event.get("body") or "{}")

    # In a real app, get userId from the JWT token in the Authorizer context.
    # Here we read it from a header for simplicity.
    user_id = (event.get("headers") or {}).get("x-user-id", "user-default")

    # POST /tasks — create a task
    if method == "POST" and path == "/tasks":
        task_id = str(uuid.uuid4())
        item = {
            "userId": user_id,
            "taskId": task_id,
            "title": body.get("title", "Untitled"),
            "done": False,
            "createdAt": int(time.time()),
        }
        if body.get("description"):
            item["description"] = body["description"]
        table.put_item(Item=item)
        return response(201, item)

    # GET /tasks — list all tasks for the user
    if method == "GET" and path == "/tasks":
        result = table.query(
            KeyConditionExpression=Key("userId").eq(user_id),
        )
        return response(200, {"tasks": result["Items"], "count": result["Count"]})

    # GET /tasks/{taskId} — get one task
    if method == "GET" and "/tasks/" in path:
        task_id = params.get("taskId")
        result = table.get_item(Key={"userId": user_id, "taskId": task_id})
        item = result.get("Item")
        if not item:
            return response(404, {"error": "Task not found"})
        return response(200, item)

    # PUT /tasks/{taskId} — update a task
    if method == "PUT" and "/tasks/" in path:
        task_id = params.get("taskId")
        updates = []
        values = {}
        names = {}
        if "title" in body:
            updates.append("title = :t")
            values[":t"] = body["title"]
        if "done" in body:
            updates.append("#d = :d")
            values[":d"] = body["done"]
            names["#d"] = "done"
        if "description" in body:
            updates.append("description = :desc")
            values[":desc"] = body["description"]
        if not updates:
            return response(400, {"error": "Nothing to update"})
        kwargs = {
            "Key": {"userId": user_id, "taskId": task_id},
            "UpdateExpression": "SET " + ", ".join(updates),
            "ExpressionAttributeValues": values,
            "ReturnValues": "ALL_NEW",
            "ConditionExpression": "attribute_exists(taskId)",
        }
        if names:
            kwargs["ExpressionAttributeNames"] = names
        try:
            result = table.update_item(**kwargs)
            return response(200, result["Attributes"])
        except Exception as e:
            if "ConditionalCheckFailed" in str(e):
                return response(404, {"error": "Task not found"})
            raise

    # DELETE /tasks/{taskId} — delete a task
    if method == "DELETE" and "/tasks/" in path:
        task_id = params.get("taskId")
        table.delete_item(Key={"userId": user_id, "taskId": task_id})
        return response(204, {})

    return response(400, {"error": f"Unknown route: {method} {path}"})
"""

lam.create_function(
    FunctionName="tasks-api",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip(handler_code)},
    Timeout=30,
)
print("✓ Lambda function created")
```

---

## Step 3: Create the API Gateway

```python
# Create HTTP API
api = apigw.create_api(
    Name="tasks-api",
    ProtocolType="HTTP",
    CorsConfiguration={
        "AllowOrigins": ["*"],
        "AllowMethods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
        "AllowHeaders": ["Content-Type", "X-User-Id"],
    },
)
api_id = api["ApiId"]

# Create Lambda integration
fn_arn = "arn:aws:lambda:us-east-1:000000000000:function:tasks-api"
integration = apigw.create_integration(
    ApiId=api_id,
    IntegrationType="AWS_PROXY",
    IntegrationUri=fn_arn,
    PayloadFormatVersion="2.0",
)
int_id = integration["IntegrationId"]
target = f"integrations/{int_id}"

# Create routes
routes = [
    "POST /tasks",
    "GET /tasks",
    "GET /tasks/{taskId}",
    "PUT /tasks/{taskId}",
    "DELETE /tasks/{taskId}",
]
for route_key in routes:
    apigw.create_route(ApiId=api_id, RouteKey=route_key, Target=target)

# Deploy
apigw.create_stage(ApiId=api_id, StageName="$default", AutoDeploy=True)

base_url = f"http://{api_id}.execute-api.localhost:4566"
print(f"✓ API deployed at: {base_url}")
```

---

## Step 4: Test the Full API

```python
import urllib.request, urllib.error

def api_call(method, path, body=None, user_id="user-alice"):
    url = f"{base_url}{path}"
    data = json.dumps(body).encode() if body else None
    req = urllib.request.Request(url, data=data, method=method)
    req.add_header("Content-Type", "application/json")
    req.add_header("X-User-Id", user_id)
    try:
        with urllib.request.urlopen(req) as resp:
            return resp.status, json.loads(resp.read())
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read())

# Create tasks
status, task1 = api_call("POST", "/tasks", {"title": "Buy groceries", "description": "Milk, eggs, bread"})
print(f"Created: {status} → {task1['taskId']}")  # 201

status, task2 = api_call("POST", "/tasks", {"title": "Write tests"})
print(f"Created: {status} → {task2['taskId']}")  # 201

# List all tasks
status, data = api_call("GET", "/tasks")
print(f"List: {status} → {data['count']} tasks")  # 200, 2 tasks

# Get one task
task_id = task1["taskId"]
status, data = api_call("GET", f"/tasks/{task_id}")
print(f"Get: {status} → {data['title']}")  # 200, "Buy groceries"

# Update — mark as done
status, data = api_call("PUT", f"/tasks/{task_id}", {"done": True})
print(f"Update: {status} → done={data['done']}")  # 200, done=True

# Delete
status, _ = api_call("DELETE", f"/tasks/{task_id}")
print(f"Delete: {status}")  # 204

# Verify deleted
status, data = api_call("GET", f"/tasks/{task_id}")
print(f"After delete: {status}")  # 404

# Different user — isolated data
status, data = api_call("GET", "/tasks", user_id="user-bob")
print(f"Bob's tasks: {data['count']}")  # 0 — Bob has no tasks
```

---

## Step 5: Add Input Validation

Real APIs validate input. Add this to the Lambda:

```python
# In the POST /tasks handler, add validation:
validation_code = """
def validate_task(body):
    errors = []
    if not body.get("title"):
        errors.append("title is required")
    if len(body.get("title", "")) > 200:
        errors.append("title must be 200 characters or less")
    return errors

# In handler, before creating:
errors = validate_task(body)
if errors:
    return response(400, {"error": "Validation failed", "details": errors})
"""
```

---

## Step 6: Add a Health Check Endpoint

```python
# Add to the Lambda handler:
health_code = """
# GET /health — no auth required
if method == "GET" and path == "/health":
    return response(200, {"status": "ok", "service": "tasks-api"})
"""

# Add the route
apigw.create_route(ApiId=api_id, RouteKey="GET /health", Target=target)

# Test it
status, data = api_call("GET", "/health")
print(f"Health: {status} → {data}")  # 200, {"status": "ok"}
```

---

## The Complete Architecture

```
                    ┌─────────────────────────────────┐
                    │         API Gateway v2           │
                    │                                  │
                    │  POST   /tasks                   │
                    │  GET    /tasks                   │
                    │  GET    /tasks/{taskId}          │
                    │  PUT    /tasks/{taskId}          │
                    │  DELETE /tasks/{taskId}          │
                    └──────────────┬──────────────────┘
                                   │ AWS_PROXY
                    ┌──────────────▼──────────────────┐
                    │         Lambda Function          │
                    │         (tasks-api)              │
                    │                                  │
                    │  Routes requests                 │
                    │  Validates input                 │
                    │  Calls DynamoDB                  │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │           DynamoDB               │
                    │           (Tasks)                │
                    │                                  │
                    │  PK: userId                      │
                    │  SK: taskId                      │
                    └─────────────────────────────────┘
```

---

## What to Add Next (Production Checklist)

**Authentication** — add a Cognito JWT authorizer to API Gateway so only logged-in users can call the API. The `userId` comes from the JWT token, not a header.

**Error handling** — wrap the Lambda handler in try/except and return structured errors.

**Pagination** — `GET /tasks` should support `?limit=20&nextToken=...` for large datasets.

**Observability** — Lambda automatically logs to CloudWatch. Add structured logging (`json.dumps({"level": "info", "message": "..."})`) for easier querying.

**Environment variables** — pass the table name as `TABLE_NAME` env var instead of hardcoding it.

**Infrastructure as code** — define all of this in Terraform or CDK so you can deploy to real AWS with one command.

---

## What's Different on Real AWS

- **IAM** — Lambda needs `dynamodb:GetItem`, `dynamodb:PutItem`, etc. on the Tasks table. API Gateway needs `lambda:InvokeFunction`.
- **Cold starts** — first request after idle takes ~200ms extra. Use Provisioned Concurrency for latency-sensitive APIs.
- **DynamoDB costs** — $1.25/million writes, $0.25/million reads. For a small app, essentially free.
- **Lambda costs** — first 1M requests/month free, then $0.20/million. Essentially free for most apps.
- **API Gateway costs** — $1/million requests for HTTP API. Essentially free for most apps.
- **Total cost for a small app** — often under $1/month. This architecture scales to millions of users before you need to worry about cost.
