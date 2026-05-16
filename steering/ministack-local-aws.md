---
inclusion: manual
---

# Local AWS Testing with MiniStack

When the user wants to test, develop, or debug AWS code locally, use MiniStack as the local AWS endpoint.

## Starting MiniStack

If MiniStack is not already running, start it:

```bash
pip install ministack
ministack
```

MiniStack runs at `http://localhost:4566`. Verify with:

```bash
curl http://localhost:4566/_ministack/health
```

## Configuring AWS Clients

Always point AWS clients at `http://localhost:4566`:

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
```

**AWS CLI:**
```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
# Add --endpoint-url=http://localhost:4566 to every command
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
    s3       = "http://localhost:4566"
    sqs      = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
    lambda   = "http://localhost:4566"
    iam      = "http://localhost:4566"
    sts      = "http://localhost:4566"
    # add other services as needed
  }
}
```

## Resetting State Between Tests

```bash
curl -X POST http://localhost:4566/_ministack/reset
```

Call this in `setUp`, `beforeEach`, or at the start of a test session to get a clean environment.

## Supported Services

MiniStack supports 60+ AWS services including:
S3, SQS, SNS, DynamoDB, DynamoDB Streams, Lambda, IAM, STS, Account, Organizations, SecretsManager, SSM, AppConfig, CloudWatch, CloudWatch Logs, EventBridge, Scheduler, Pipes, Kinesis, Firehose, Step Functions, API Gateway v1/v2 (WebSocket included), ECS, EKS, Batch, EMR, RDS, RDS Data, ElastiCache, OpenSearch, ECR, EFS, KMS, ACM, Route 53, Cloud Map, Cognito, Glue, Athena, SES (v1 + v2), CloudFormation, CloudTrail, CloudFront, CloudFront KeyValueStore, AppSync, AppSync Events, ALB, AutoScaling, WAF (v1 + v2), Backup, CodeBuild, Transfer Family, Resource Groups, Tagging, and more.

## When to Use MiniStack

- Writing unit/integration tests for AWS-dependent code
- Local development without AWS credentials or costs
- CI/CD pipelines that need AWS services
- Prototyping new AWS integrations
- Testing Terraform/CDK/Pulumi infrastructure code locally

## Notes

- No AWS account or credentials needed — use `test`/`test` as access/secret key
- All data is in-memory by default — resets on restart
- MIT licensed, free forever: https://github.com/ministackorg/ministack
