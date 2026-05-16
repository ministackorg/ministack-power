---
name: "ministack"
displayName: "MiniStack — Local AWS Emulator & Learning Environment"
description: "Test and learn AWS locally for free. MiniStack emulates 60+ AWS services on a single port — no account, no Docker required, no cost. Perfect for testing existing code or learning AWS from scratch."
keywords: ["ministack", "aws", "s3", "dynamodb", "lambda", "sqs", "sns", "kms", "cognito", "iam", "cloudwatch", "eventbridge", "step functions", "api gateway", "ecs", "eks", "rds", "elasticache", "glue", "athena", "kinesis", "firehose", "cloudformation", "secrets manager", "ssm", "localstack", "boto3", "terraform", "cdk", "local aws"]
author: "MiniStack"
---

<p align="center">
  <img src="ministack_logo.svg" alt="MiniStack" width="220"/>
</p>

# MiniStack — Local AWS Emulator & Learning Environment

MiniStack lets you develop, test, and **learn** AWS locally — no account, no cost, no risk.
It emulates 60+ AWS services on a single port and is MIT licensed, free forever.

## Onboarding

### Step 1: Install and start MiniStack

```bash
pip install ministack
ministack
```

MiniStack is now running at `http://localhost:4566`. Verify:

```bash
curl http://localhost:4566/_ministack/health
```

### Step 2: Point your AWS clients at localhost

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
aws --endpoint-url=http://localhost:4566 s3 mb s3://my-bucket
```

## When to Load Steering Files

**Testing existing AWS code:**
- Setting up clients, Terraform, or CI → `aws-testing-workflow.md`
- Resetting state between tests → `aws-testing-workflow.md`

**Learning AWS services:**
- Learning S3, object storage, buckets, presigned URLs → `learn-s3.md`
- Learning SQS, SNS, messaging, pub/sub, fanout → `learn-sqs-sns.md`
- Learning DynamoDB, NoSQL, GSI, TTL, streams → `learn-dynamodb.md`
- Learning Lambda, serverless, event-driven → `learn-lambda.md`
- Learning IAM, roles, policies, permissions → `learn-iam.md`
- Learning Step Functions, workflows, orchestration → `learn-step-functions.md`
- Learning EventBridge, event buses, rules → `learn-eventbridge.md`
- Learning API Gateway, REST APIs, WebSocket → `learn-api-gateway.md`
- Learning Cognito, user pools, authentication, sign-up, sign-in, JWT tokens, user management → `learn-cognito.md`
- Learning CloudWatch, metrics, alarms, alarm actions, log groups, observability, Lambda metrics → `learn-cloudwatch.md`

**Learning AWS patterns:**
- Building a serverless REST API, Lambda + API Gateway + DynamoDB, CRUD API → `learn-serverless-rest-api.md`

## Supported Services

60+ AWS services: S3, SQS, SNS, DynamoDB, DynamoDB Streams, Lambda, IAM, STS,
Account, Organizations, SecretsManager, SSM, AppConfig, CloudWatch, CloudWatch
Logs, EventBridge, Scheduler, Pipes, Kinesis, Firehose, Step Functions, API
Gateway v1/v2 (WebSocket included), ECS, EKS, Batch, EMR, RDS, RDS Data,
ElastiCache, OpenSearch, ECR, EFS, KMS, ACM, Route 53, Cloud Map, Cognito, Glue,
Athena, SES (v1 + v2), CloudFormation, CloudTrail, CloudFront, CloudFront
KeyValueStore, AppSync, AppSync Events, ALB, AutoScaling, WAF (v1 + v2),
Backup, CodeBuild, Transfer Family, Resource Groups, Tagging, EC2, IMDS,
ECS Metadata, S3 files. See `mcp/catalog.json` in the ministack repo for the
canonical service list.

## Links

- GitHub: https://github.com/ministackorg/ministack
- PyPI: https://pypi.org/project/ministack/
- Docker Hub: https://hub.docker.com/r/ministackorg/ministack

## License

MIT — https://github.com/ministackorg/ministack/blob/main/LICENSE

## Privacy Policy

MiniStack runs entirely locally on your machine. No data is collected or transmitted.
https://github.com/ministackorg/ministack

## Support

- GitHub Issues: https://github.com/ministackorg/ministack/issues
- GitHub Discussions: https://github.com/ministackorg/ministack/discussions
