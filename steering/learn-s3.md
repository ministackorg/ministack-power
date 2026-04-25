---
inclusion: manual
---

# Learn S3 with MiniStack

Amazon S3 (Simple Storage Service) is AWS's object storage service. Think of it as an infinitely scalable hard drive in the cloud — you store files (called **objects**) inside containers (called **buckets**).

## Core Concepts

- **Bucket** — a named container. Globally unique on real AWS. Like a top-level folder.
- **Object** — a file + its metadata. Identified by a **key** (the path/filename).
- **Key** — the full "path" of an object, e.g. `images/2024/photo.jpg`. S3 is flat — there are no real folders, just keys with `/` in them.
- **Region** — where your bucket lives. Affects latency and data residency.

## Hands-On: Your First Bucket

Make sure MiniStack is running (`ministack` or `docker run -p 4566:4566 ministackorg/ministack`), then:

```python
import boto3

s3 = boto3.client("s3",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a bucket
s3.create_bucket(Bucket="my-first-bucket")

# Upload an object
s3.put_object(
    Bucket="my-first-bucket",
    Key="hello.txt",
    Body=b"Hello, S3!",
)

# Download it back
response = s3.get_object(Bucket="my-first-bucket", Key="hello.txt")
print(response["Body"].read())  # b'Hello, S3!'

# List objects in the bucket
response = s3.list_objects_v2(Bucket="my-first-bucket")
for obj in response.get("Contents", []):
    print(obj["Key"], obj["Size"])

# Delete an object
s3.delete_object(Bucket="my-first-bucket", Key="hello.txt")

# Delete the bucket (must be empty first)
s3.delete_bucket(Bucket="my-first-bucket")
```

## Hands-On: Organising with "Folders"

S3 has no real folders — just keys with `/` separators. But SDKs and the console treat them as folders.

```python
s3.create_bucket(Bucket="my-app")

# "Upload" to different "folders"
s3.put_object(Bucket="my-app", Key="images/logo.png", Body=b"<png data>")
s3.put_object(Bucket="my-app", Key="images/banner.jpg", Body=b"<jpg data>")
s3.put_object(Bucket="my-app", Key="docs/readme.md", Body=b"# Readme")
s3.put_object(Bucket="my-app", Key="docs/api.md", Body=b"# API")

# List only the "images/" folder using Prefix
response = s3.list_objects_v2(Bucket="my-app", Prefix="images/")
for obj in response["Contents"]:
    print(obj["Key"])
# images/logo.png
# images/banner.jpg

# List "top-level folders" using Delimiter
response = s3.list_objects_v2(Bucket="my-app", Delimiter="/")
for prefix in response.get("CommonPrefixes", []):
    print(prefix["Prefix"])
# docs/
# images/
```

## Hands-On: Metadata and Content Types

```python
# Upload with metadata and content type
s3.put_object(
    Bucket="my-app",
    Key="profile.json",
    Body=b'{"name": "Alice"}',
    ContentType="application/json",
    Metadata={
        "author": "alice",
        "version": "1.0",
    },
)

# Read metadata without downloading the body
response = s3.head_object(Bucket="my-app", Key="profile.json")
print(response["ContentType"])       # application/json
print(response["Metadata"])          # {'author': 'alice', 'version': '1.0'}
print(response["ContentLength"])     # 17
```

## Hands-On: Versioning

Versioning keeps every version of an object — great for audit trails and accidental-delete protection.

```python
s3.create_bucket(Bucket="versioned-bucket")

# Enable versioning
s3.put_bucket_versioning(
    Bucket="versioned-bucket",
    VersioningConfiguration={"Status": "Enabled"},
)

# Each put creates a new version
s3.put_object(Bucket="versioned-bucket", Key="config.json", Body=b'{"v": 1}')
s3.put_object(Bucket="versioned-bucket", Key="config.json", Body=b'{"v": 2}')
s3.put_object(Bucket="versioned-bucket", Key="config.json", Body=b'{"v": 3}')

# List all versions
response = s3.list_object_versions(Bucket="versioned-bucket", Prefix="config.json")
for v in response["Versions"]:
    print(v["VersionId"], v["LastModified"])

# Get a specific version
response = s3.get_object(
    Bucket="versioned-bucket",
    Key="config.json",
    VersionId=response["Versions"][-1]["VersionId"],  # oldest version
)
print(response["Body"].read())  # b'{"v": 1}'
```

## Hands-On: Tagging

Tags are key-value pairs on buckets and objects — used for cost allocation, access control, and lifecycle rules.

```python
# Tag a bucket
s3.put_bucket_tagging(
    Bucket="my-app",
    Tagging={"TagSet": [
        {"Key": "Environment", "Value": "dev"},
        {"Key": "Team", "Value": "platform"},
    ]},
)

# Tag an object
s3.put_object_tagging(
    Bucket="my-app",
    Key="profile.json",
    Tagging={"TagSet": [{"Key": "PII", "Value": "true"}]},
)
```

## Real-World Pattern: S3 as a Data Lake Landing Zone

A common pattern: services write raw data to S3, Lambda processes it.

```python
import json, time

# Simulate an app writing events to S3
def write_event(bucket, event):
    key = f"events/{time.strftime('%Y/%m/%d')}/{event['id']}.json"
    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=json.dumps(event).encode(),
        ContentType="application/json",
    )
    return key

s3.create_bucket(Bucket="data-lake")
key = write_event("data-lake", {"id": "evt-001", "type": "purchase", "amount": 99.99})
print(f"Written to: {key}")
# events/2024/01/15/evt-001.json
```

## What's Different on Real AWS

- **Bucket names are globally unique** — `my-bucket` might already be taken. Use a prefix like `mycompany-myapp-bucket`.
- **Regions matter** — a bucket in `us-east-1` can't be accessed as if it's in `eu-west-1`. MiniStack ignores this.
- **Presigned URLs** — real AWS generates time-limited URLs for private objects. MiniStack stores them but doesn't enforce expiry.
- **S3 costs money** — storage ($0.023/GB/month), requests ($0.0004/1000 GETs), and data transfer out. MiniStack is free.
- **Eventual consistency** — real S3 is strongly consistent for new objects since 2020, but replication across regions is still eventual.
- **IAM permissions** — on real AWS, your Lambda/EC2 needs an IAM role with `s3:GetObject` etc. MiniStack skips auth.

## Next Steps

- Try **SQS + S3** together: upload a file, send a message to a queue with the S3 key, have a worker process it.
- Learn **Lambda** to automatically process S3 uploads with event triggers.
- Learn **IAM** to understand how to secure S3 buckets in production.
