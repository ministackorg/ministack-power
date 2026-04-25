---
inclusion: manual
---

# Learn API Gateway with MiniStack

AWS API Gateway lets you create HTTP APIs, REST APIs, and WebSocket APIs that invoke Lambda functions or forward to HTTP backends. It's the front door for serverless applications.

## Two Flavours

- **HTTP API (v2)** — newer, simpler, cheaper. Use this for most new projects.
- **REST API (v1)** — older, more features (request validation, usage plans, API keys). Use when you need those extras.

---

## HTTP API (v2)

### Hands-On: Hello World API

```python
import boto3, json, zipfile, io, time

apigw = boto3.client("apigatewayv2",
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

# Create a Lambda handler
def make_zip(code):
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w") as zf:
        zf.writestr("index.py", code)
    return buf.getvalue()

lam.create_function(
    FunctionName="api-handler",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
import json

def handler(event, context):
    method = event["requestContext"]["http"]["method"]
    path = event["rawPath"]
    body = event.get("body") or ""
    
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "message": "Hello from Lambda!",
            "method": method,
            "path": path,
        }),
    }
""")},
)

fn_arn = "arn:aws:lambda:us-east-1:000000000000:function:api-handler"

# Create the HTTP API
api = apigw.create_api(
    Name="my-api",
    ProtocolType="HTTP",
)
api_id = api["ApiId"]

# Create integration (Lambda proxy)
integration = apigw.create_integration(
    ApiId=api_id,
    IntegrationType="AWS_PROXY",
    IntegrationUri=fn_arn,
    PayloadFormatVersion="2.0",
)
int_id = integration["IntegrationId"]

# Create routes
apigw.create_route(
    ApiId=api_id,
    RouteKey="GET /hello",
    Target=f"integrations/{int_id}",
)
apigw.create_route(
    ApiId=api_id,
    RouteKey="POST /hello",
    Target=f"integrations/{int_id}",
)

# Create a stage
apigw.create_stage(ApiId=api_id, StageName="$default", AutoDeploy=True)

# The API is now available at:
# http://{api_id}.execute-api.localhost:4566/hello
print(f"API endpoint: http://{api_id}.execute-api.localhost:4566")
```

### Hands-On: Path Parameters

```python
lam.create_function(
    FunctionName="user-handler",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
import json

def handler(event, context):
    user_id = event["pathParameters"]["userId"]
    return {
        "statusCode": 200,
        "body": json.dumps({"userId": user_id, "name": "Alice"}),
    }
""")},
)

fn_arn = "arn:aws:lambda:us-east-1:000000000000:function:user-handler"

api = apigw.create_api(Name="users-api", ProtocolType="HTTP")
integration = apigw.create_integration(
    ApiId=api["ApiId"],
    IntegrationType="AWS_PROXY",
    IntegrationUri=fn_arn,
    PayloadFormatVersion="2.0",
)

# {userId} is a path parameter
apigw.create_route(
    ApiId=api["ApiId"],
    RouteKey="GET /users/{userId}",
    Target=f"integrations/{integration['IntegrationId']}",
)
apigw.create_stage(ApiId=api["ApiId"], StageName="$default")

# Call: GET /users/u-123 → pathParameters = {"userId": "u-123"}
```

### Hands-On: Catch-All Route with {proxy+}

```python
# {proxy+} matches any path — useful for forwarding all requests to one Lambda
apigw.create_route(
    ApiId=api_id,
    RouteKey="ANY /{proxy+}",
    Target=f"integrations/{int_id}",
)
# GET /anything/you/want → all go to the same Lambda
```

---

## WebSocket API

### Hands-On: Real-Time Chat

```python
# Lambda for $connect
lam.create_function(
    FunctionName="ws-connect",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
import json

def handler(event, context):
    connection_id = event["requestContext"]["connectionId"]
    print(f"Client connected: {connection_id}")
    return {"statusCode": 200}
""")},
)

# Lambda for $disconnect
lam.create_function(
    FunctionName="ws-disconnect",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
def handler(event, context):
    connection_id = event["requestContext"]["connectionId"]
    print(f"Client disconnected: {connection_id}")
    return {"statusCode": 200}
""")},
)

# Lambda for messages — echoes back to the sender
lam.create_function(
    FunctionName="ws-message",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
import json, boto3

def handler(event, context):
    connection_id = event["requestContext"]["connectionId"]
    body = json.loads(event.get("body", "{}"))
    
    # Echo the message back
    apigw_mgmt = boto3.client("apigatewaymanagementapi",
        endpoint_url="http://localhost:4566",
        aws_access_key_id="test",
        aws_secret_access_key="test",
        region_name="us-east-1",
    )
    apigw_mgmt.post_to_connection(
        ConnectionId=connection_id,
        Data=json.dumps({"echo": body.get("message", "")}).encode(),
    )
    return {"statusCode": 200}
""")},
)

# Create WebSocket API
ws_api = apigw.create_api(
    Name="chat-api",
    ProtocolType="WEBSOCKET",
    RouteSelectionExpression="$request.body.action",
)
ws_id = ws_api["ApiId"]

def add_ws_route(route_key, fn_name):
    fn_arn = f"arn:aws:lambda:us-east-1:000000000000:function:{fn_name}"
    integ = apigw.create_integration(
        ApiId=ws_id,
        IntegrationType="AWS_PROXY",
        IntegrationUri=fn_arn,
    )
    apigw.create_route(
        ApiId=ws_id,
        RouteKey=route_key,
        Target=f"integrations/{integ['IntegrationId']}",
    )

add_ws_route("$connect", "ws-connect")
add_ws_route("$disconnect", "ws-disconnect")
add_ws_route("$default", "ws-message")

apigw.create_stage(ApiId=ws_id, StageName="prod")

# Connect with: ws://{ws_id}.execute-api.localhost:4566/prod
print(f"WebSocket: ws://{ws_id}.execute-api.localhost:4566/prod")
```

---

## REST API (v1)

Use v1 when you need request validation, usage plans, or API keys.

```python
apigw_v1 = boto3.client("apigateway",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create REST API
api = apigw_v1.create_rest_api(Name="my-rest-api")
api_id = api["id"]

# Get the root resource "/"
resources = apigw_v1.get_resources(restApiId=api_id)
root_id = resources["items"][0]["id"]

# Create /items resource
items_resource = apigw_v1.create_resource(
    restApiId=api_id,
    parentId=root_id,
    pathPart="items",
)
items_id = items_resource["id"]

# Add GET method
apigw_v1.put_method(
    restApiId=api_id,
    resourceId=items_id,
    httpMethod="GET",
    authorizationType="NONE",
)

# Add Lambda integration
fn_arn = "arn:aws:lambda:us-east-1:000000000000:function:api-handler"
apigw_v1.put_integration(
    restApiId=api_id,
    resourceId=items_id,
    httpMethod="GET",
    type="AWS_PROXY",
    integrationHttpMethod="POST",
    uri=f"arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/{fn_arn}/invocations",
)

# Deploy
apigw_v1.create_deployment(restApiId=api_id, stageName="prod")

# Available at: http://{api_id}.execute-api.localhost:4566/prod/items
```

## What's Different on Real AWS

- **Custom domains** — on real AWS you map `api.myapp.com` to your API. MiniStack uses `{apiId}.execute-api.localhost`.
- **CORS** — HTTP APIs have built-in CORS config. REST APIs need manual OPTIONS method setup. MiniStack doesn't enforce CORS.
- **Throttling** — API Gateway throttles at 10,000 req/s by default. MiniStack is unbounded.
- **IAM auth / Cognito auth** — real APIs can require authentication. MiniStack stores authorizer config but doesn't enforce it.
- **Pricing** — HTTP API: $1/million requests. REST API: $3.50/million. WebSocket: $1/million messages + $0.25/million connection-minutes.
- **Lambda permissions** — on real AWS, API Gateway needs `lambda:InvokeFunction` permission on the function. MiniStack skips this.

## Next Steps

- Add **Cognito** authentication to your API — user pools + JWT authorizers.
- Use **DynamoDB** as the backend for a full CRUD API.
- Try **WebSocket** for real-time features like chat, live dashboards, or multiplayer games.
