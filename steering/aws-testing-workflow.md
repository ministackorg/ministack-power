---
inclusion: auto
---

# Local AWS Testing with MiniStack

When the user wants to test, develop, or debug any AWS service locally, use MiniStack.

## Starting MiniStack

```bash
pip install ministack
ministack
# Runs on http://localhost:4566
```

Verify it's up:
```bash
curl http://localhost:4566/_ministack/health
```

## Configuring AWS Clients

**boto3 (Python):**
```python
import boto3

def aws_client(service):
    return boto3.client(
        service,
        endpoint_url="http://localhost:4566",
        aws_access_key_id="test",
        aws_secret_access_key="test",
        region_name="us-east-1",
    )

s3  = aws_client("s3")
sqs = aws_client("sqs")
ddb = aws_client("dynamodb")
lam = aws_client("lambda")
# ... any of the 40+ supported services
```

**AWS CLI:**
```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
aws --endpoint-url=http://localhost:4566 <service> <command>
```

**Terraform:**
```hcl
provider "aws" {
  access_key                  = "test"
  secret_key                  = "test"
  region                      = "us-east-1"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  endpoints {
    s3             = "http://localhost:4566"
    sqs            = "http://localhost:4566"
    sns            = "http://localhost:4566"
    dynamodb       = "http://localhost:4566"
    lambda         = "http://localhost:4566"
    iam            = "http://localhost:4566"
    sts            = "http://localhost:4566"
    secretsmanager = "http://localhost:4566"
    ssm            = "http://localhost:4566"
    cloudwatch     = "http://localhost:4566"
    logs           = "http://localhost:4566"
    events         = "http://localhost:4566"
    kinesis        = "http://localhost:4566"
    stepfunctions  = "http://localhost:4566"
    apigateway     = "http://localhost:4566"
    apigatewayv2   = "http://localhost:4566"
    ecs            = "http://localhost:4566"
    ecr            = "http://localhost:4566"
    rds            = "http://localhost:4566"
    elasticache    = "http://localhost:4566"
    kms            = "http://localhost:4566"
    acm            = "http://localhost:4566"
    route53        = "http://localhost:4566"
    cognito_idp    = "http://localhost:4566"
    glue           = "http://localhost:4566"
    athena         = "http://localhost:4566"
    ses            = "http://localhost:4566"
    firehose       = "http://localhost:4566"
    cloudformation = "http://localhost:4566"
    elb            = "http://localhost:4566"
    elbv2          = "http://localhost:4566"
    wafv2          = "http://localhost:4566"
  }
}
```

## Resetting State Between Tests

```bash
# Wipe all state — call this in setUp/beforeEach
curl -X POST http://localhost:4566/_ministack/reset

# Reset and re-run init scripts
curl -X POST http://localhost:4566/_ministack/reset?init=1
```

In Python tests:
```python
import urllib.request

def reset_ministack():
    req = urllib.request.Request(
        "http://localhost:4566/_ministack/reset",
        data=b"", method="POST"
    )
    urllib.request.urlopen(req, timeout=5)
```

## Multi-Tenancy (Isolated Accounts)

Use a 12-digit number as `AWS_ACCESS_KEY_ID` to get fully isolated state per account:

```python
# Team A
client_a = boto3.client("s3",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="111111111111",
    aws_secret_access_key="test",
)

# Team B — completely isolated from Team A
client_b = boto3.client("s3",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="222222222222",
    aws_secret_access_key="test",
)
```

## Runtime Config (No Restart Needed)

```bash
# Switch Lambda to Docker execution mode
curl -X POST http://localhost:4566/_ministack/config \
  -H "Content-Type: application/json" \
  -d '{"lambda_svc.LAMBDA_EXECUTOR": "docker"}'

# Speed up Step Functions Wait states in tests
curl -X POST http://localhost:4566/_ministack/config \
  -H "Content-Type: application/json" \
  -d '{"stepfunctions._SFN_WAIT_SCALE": 0}'
```

## Docker Alternative

If you prefer Docker:
```bash
docker run -p 4566:4566 ministackorg/ministack

# With real infrastructure (RDS, ECS, Lambda containers)
docker run -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ministackorg/ministack
```

## Key Facts

- No AWS account or credentials needed
- All data is in-memory by default (resets on restart)
- MIT licensed, free forever
- Compatible with LocalStack's `/_localstack/health` endpoint
- GitHub: https://github.com/ministackorg/ministack
