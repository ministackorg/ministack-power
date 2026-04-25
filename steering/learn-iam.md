---
inclusion: manual
---

# Learn IAM with MiniStack

AWS IAM (Identity and Access Management) controls **who** can do **what** on **which** AWS resources. It's the security layer for everything in AWS.

## Core Concepts

- **User** — a person or application with long-term credentials (access key + secret)
- **Role** — an identity assumed temporarily. Used by Lambda, EC2, ECS tasks, etc.
- **Policy** — a JSON document that defines permissions (`Allow` or `Deny`)
- **Managed policy** — a reusable policy you attach to users/roles
- **Inline policy** — a policy embedded directly in a user/role (not reusable)
- **Principal** — who is making the request (user, role, service)
- **Resource** — what AWS resource the action applies to (ARN or `*`)
- **Action** — what operation (e.g. `s3:GetObject`, `dynamodb:PutItem`)

## Hands-On: Creating Users and Access Keys

```python
import boto3, json

iam = boto3.client("iam",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a user
iam.create_user(UserName="alice")

# Create access keys for the user
keys = iam.create_access_key(UserName="alice")
access_key = keys["AccessKey"]["AccessKeyId"]
secret_key = keys["AccessKey"]["SecretAccessKey"]
print(f"Access Key: {access_key}")
print(f"Secret Key: {secret_key}")

# List users
response = iam.list_users()
for user in response["Users"]:
    print(user["UserName"], user["Arn"])

# Delete access key
iam.delete_access_key(UserName="alice", AccessKeyId=access_key)

# Delete user
iam.delete_user(UserName="alice")
```

## Hands-On: Creating Roles

Roles are assumed by services (Lambda, EC2) or other accounts. They don't have long-term credentials.

```python
# Trust policy — who can assume this role
trust_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "lambda.amazonaws.com"},
        "Action": "sts:AssumeRole",
    }],
}

# Create a role for Lambda
iam.create_role(
    RoleName="lambda-execution-role",
    AssumeRolePolicyDocument=json.dumps(trust_policy),
    Description="Role for Lambda functions",
)

# Get the role ARN (used when creating Lambda functions)
response = iam.get_role(RoleName="lambda-execution-role")
role_arn = response["Role"]["Arn"]
print(role_arn)
# arn:aws:iam::000000000000:role/lambda-execution-role
```

## Hands-On: Managed Policies

```python
# Create a policy that allows reading from a specific S3 bucket
s3_read_policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "s3:GetObject",
            "s3:ListBucket",
        ],
        "Resource": [
            "arn:aws:s3:::my-data-bucket",
            "arn:aws:s3:::my-data-bucket/*",
        ],
    }],
}

response = iam.create_policy(
    PolicyName="S3ReadMyDataBucket",
    PolicyDocument=json.dumps(s3_read_policy),
    Description="Read access to my-data-bucket",
)
policy_arn = response["Policy"]["Arn"]

# Attach the policy to the Lambda role
iam.attach_role_policy(
    RoleName="lambda-execution-role",
    PolicyArn=policy_arn,
)

# List attached policies
response = iam.list_attached_role_policies(RoleName="lambda-execution-role")
for p in response["AttachedPolicies"]:
    print(p["PolicyName"], p["PolicyArn"])

# Detach and delete
iam.detach_role_policy(RoleName="lambda-execution-role", PolicyArn=policy_arn)
iam.delete_policy(PolicyArn=policy_arn)
```

## Hands-On: Inline Policies

Inline policies are embedded in a role — useful for one-off permissions.

```python
# Add an inline policy directly to the role
iam.put_role_policy(
    RoleName="lambda-execution-role",
    PolicyName="AllowDynamoDBAccess",
    PolicyDocument=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:UpdateItem",
                "dynamodb:DeleteItem",
                "dynamodb:Query",
                "dynamodb:Scan",
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:000000000000:table/Users",
        }],
    }),
)

# List inline policies
response = iam.list_role_policies(RoleName="lambda-execution-role")
print(response["PolicyNames"])  # ['AllowDynamoDBAccess']

# Get the inline policy document
response = iam.get_role_policy(
    RoleName="lambda-execution-role",
    PolicyName="AllowDynamoDBAccess",
)
print(response["PolicyDocument"])
```

## Hands-On: Groups

Groups let you manage permissions for multiple users at once.

```python
# Create a group
iam.create_group(GroupName="developers")

# Attach a policy to the group
iam.attach_group_policy(
    GroupName="developers",
    PolicyArn="arn:aws:iam::aws:policy/ReadOnlyAccess",  # AWS managed policy
)

# Add users to the group
iam.create_user(UserName="bob")
iam.create_user(UserName="carol")
iam.add_user_to_group(GroupName="developers", UserName="bob")
iam.add_user_to_group(GroupName="developers", UserName="carol")

# List group members
response = iam.get_group(GroupName="developers")
for user in response["Users"]:
    print(user["UserName"])
```

## Hands-On: AssumeRole (STS)

Services and users assume roles to get temporary credentials.

```python
sts = boto3.client("sts",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Assume a role — get temporary credentials
response = sts.assume_role(
    RoleArn="arn:aws:iam::000000000000:role/lambda-execution-role",
    RoleSessionName="my-session",
    DurationSeconds=3600,
)

creds = response["Credentials"]
print(creds["AccessKeyId"])
print(creds["SecretAccessKey"])
print(creds["SessionToken"])
print(creds["Expiration"])

# Use the temporary credentials
temp_client = boto3.client("s3",
    endpoint_url="http://localhost:4566",
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
)
```

## Key IAM Patterns

**Least privilege** — grant only the permissions needed, nothing more.

```python
# BAD: too broad
bad_policy = {
    "Statement": [{"Effect": "Allow", "Action": "*", "Resource": "*"}]
}

# GOOD: specific actions on specific resources
good_policy = {
    "Statement": [{
        "Effect": "Allow",
        "Action": ["s3:GetObject"],
        "Resource": "arn:aws:s3:::my-bucket/uploads/*",
    }]
}
```

**Deny overrides Allow** — an explicit Deny always wins.

```python
deny_policy = {
    "Statement": [
        {"Effect": "Allow", "Action": "s3:*", "Resource": "*"},
        # This Deny overrides the Allow above for delete operations
        {"Effect": "Deny", "Action": "s3:DeleteObject", "Resource": "*"},
    ]
}
```

## What's Different on Real AWS

- **MiniStack skips auth** — all API calls succeed regardless of IAM policies. On real AWS, missing permissions = `AccessDeniedException`.
- **IAM is global** — not region-specific. One IAM user/role works across all regions.
- **Service-linked roles** — some services (ECS, RDS) create their own roles automatically.
- **Permission boundaries** — an advanced feature that caps the maximum permissions a role can have.
- **AWS managed policies** — AWS provides hundreds of pre-built policies (e.g. `AmazonS3ReadOnlyAccess`). On MiniStack, you can reference them by ARN but they're not enforced.
- **IAM is free** — no cost for IAM users, roles, or policies.

## Next Steps

- Learn how **Lambda execution roles** work in practice — what permissions does a Lambda need to read from S3, write to DynamoDB, publish to SNS?
- Understand **resource-based policies** — S3 bucket policies, SQS queue policies, Lambda resource policies.
- Try **cross-account access** using MiniStack's multi-tenancy (12-digit access keys as account IDs).
