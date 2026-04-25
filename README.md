<p align="center">
  <img src="ministack_logo.svg" alt="MiniStack" width="220"/>
</p>

<h1 align="center">MiniStack — Kiro Power</h1>

<p align="center">
  Test and learn AWS locally for free. MiniStack emulates 40+ AWS services on a single port — no account, no Docker required, no cost.
</p>

---

## What this power does

When installed, this power gives Kiro instant knowledge of MiniStack — how to start it, configure AWS clients (boto3, AWS CLI, Terraform), reset state between tests, and learn any AWS service hands-on with working code examples.

Kiro will automatically activate this power whenever you mention AWS-related topics like S3, Lambda, DynamoDB, SQS, Terraform, and more.

## Install

### Option 1 — Install from a local folder (recommended for development)

1. Open Kiro
2. Click the **Powers** icon in the sidebar
3. Click **Install** → **From local folder**
4. Select this folder (`kiro-power/`)
5. Done — the power is now active

### Option 2 — Copy to your Kiro powers directory

```bash
mkdir -p ~/.kiro/powers/ministack
cp POWER.md ~/.kiro/powers/ministack/
cp ministack_logo.svg ~/.kiro/powers/ministack/
cp -r steering ~/.kiro/powers/ministack/
```

Then restart Kiro or reconnect from the Powers panel.

## What's included

| File | Purpose |
|------|---------|
| `POWER.md` | Power metadata + onboarding guide |
| `steering/aws-testing-workflow.md` | Full client setup (boto3, CLI, Terraform), reset, multi-tenancy, Docker |
| `steering/learn-s3.md` | Hands-on S3 guide |
| `steering/learn-sqs-sns.md` | Hands-on SQS & SNS guide |
| `steering/learn-dynamodb.md` | Hands-on DynamoDB guide |
| `steering/learn-lambda.md` | Hands-on Lambda guide |
| `steering/learn-iam.md` | Hands-on IAM & STS guide |
| `steering/learn-step-functions.md` | Hands-on Step Functions guide |
| `steering/learn-eventbridge.md` | Hands-on EventBridge guide |
| `steering/learn-api-gateway.md` | Hands-on API Gateway v1/v2 + WebSocket guide |
| `steering/learn-cognito.md` | Hands-on Cognito user pools + identity pools guide |
| `steering/learn-serverless-rest-api.md` | Full serverless CRUD API walkthrough |

## MiniStack links

- GitHub: https://github.com/ministackorg/ministack
- PyPI: https://pypi.org/project/ministack/
- Docker Hub: https://hub.docker.com/r/ministackorg/ministack
