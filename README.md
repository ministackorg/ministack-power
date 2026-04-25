<p align="center">
  <img src="ministack_logo.svg" alt="MiniStack" height="60"/>
  &nbsp;&nbsp;×&nbsp;&nbsp;
  <img src="kiro.svg" alt="Kiro" height="60"/>
</p>

<h1 align="center">MiniStack × Kiro</h1>

<p align="center">
  Test and learn AWS locally for free.<br/>
  MiniStack emulates 40+ AWS services on a single port — no account, no Docker required, no cost.
</p>

<p align="center">
  <a href="https://github.com/ministackorg/ministack">GitHub</a> ·
  <a href="https://pypi.org/project/ministack/">PyPI</a> ·
  <a href="https://hub.docker.com/r/ministackorg/ministack">Docker Hub</a>
</p>

---

## What this power does

Once installed, Kiro gains deep knowledge of MiniStack. It will automatically help you:

- **Test AWS code locally** — configure boto3, AWS CLI, Terraform, and CDK against `localhost:4566`
- **Learn AWS services** — hands-on guides for S3, SQS, SNS, DynamoDB, Lambda, IAM, Step Functions, EventBridge, API Gateway, and Cognito
- **Build AWS patterns** — full serverless REST API walkthrough (API Gateway + Lambda + DynamoDB)
- **Reset state between tests** — clean environment for every test run
- **Run multi-tenant scenarios** — isolated accounts via access key IDs

The power activates automatically when you mention MiniStack or AWS topics in chat.

---

## Install

### From a GitHub repo

1. Open Kiro → click the **Powers** icon in the sidebar
2. Click **Add Custom Power** → **GitHub Repository**
3. Enter: `https://github.com/ministackorg/ministack-power`
4. Click **Add** — done

### From a local folder

1. Open Kiro → click the **Powers** icon in the sidebar
2. Click **Add Custom Power** → **Local Directory**
3. Enter the absolute path to this folder
4. Click **Add** — done

---

## Usage

After installing, open a **new agent chat** and ask naturally:

> "help me test S3 locally with MiniStack"
> "set up boto3 with a local AWS endpoint"
> "teach me DynamoDB"
> "how do I use Terraform with MiniStack"

The power activates on keywords — no special commands needed.

> **Note:** The "Try" button in the Powers panel sends a generic prompt that won't trigger the power. Use a regular new chat instead.

---

## What's included

| Steering file | Covers |
|---|---|
| `aws-testing-workflow.md` | boto3, AWS CLI, Terraform setup · reset · multi-tenancy · Docker |
| `learn-s3.md` | Buckets, objects, versioning, tagging, presigned URLs |
| `learn-sqs-sns.md` | Queues, DLQs, FIFO, topics, fan-out, filter policies |
| `learn-dynamodb.md` | Tables, GSI, TTL, conditional writes, scan vs query |
| `learn-lambda.md` | Functions, env vars, error handling, layers, SQS ESM |
| `learn-iam.md` | Users, roles, policies, groups, AssumeRole |
| `learn-step-functions.md` | State machines, Choice, Retry, Parallel, Map |
| `learn-eventbridge.md` | Event buses, rules, content filtering, Lambda targets |
| `learn-api-gateway.md` | HTTP API, REST API, WebSocket, path params, JWT auth |
| `learn-cognito.md` | User pools, sign-up/in, groups, token refresh, identity pools |
| `learn-serverless-rest-api.md` | Full CRUD API: API Gateway + Lambda + DynamoDB |

---

## License

MIT — https://github.com/ministackorg/ministack/blob/main/LICENSE
