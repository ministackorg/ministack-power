---
inclusion: manual
---

# Learn DynamoDB with MiniStack

DynamoDB is AWS's fully managed NoSQL database. It's designed for applications that need single-digit millisecond latency at any scale — from 1 request/second to millions.

## Core Concepts

- **Table** — like a database table, but schema-less (except for the key)
- **Item** — a row. Each item can have different attributes.
- **Partition key (PK)** — required. Determines which partition stores the item. Like a hash key.
- **Sort key (SK)** — optional. Combined with PK, makes a composite key. Enables range queries.
- **Attribute** — a field. Can be String (S), Number (N), Binary (B), Boolean (BOOL), List (L), Map (M), Null (NULL), or Set.
- **GSI** — Global Secondary Index. Query on non-key attributes.
- **LSI** — Local Secondary Index. Alternate sort key on the same partition.

## Hands-On: Your First Table

```python
import boto3

ddb = boto3.client("dynamodb",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a table with just a partition key
ddb.create_table(
    TableName="Users",
    KeySchema=[
        {"AttributeName": "userId", "KeyType": "HASH"},
    ],
    AttributeDefinitions=[
        {"AttributeName": "userId", "AttributeType": "S"},
    ],
    BillingMode="PAY_PER_REQUEST",
)

# Put an item — notice: no schema, add any attributes you want
ddb.put_item(
    TableName="Users",
    Item={
        "userId": {"S": "u-001"},
        "name": {"S": "Alice"},
        "email": {"S": "alice@example.com"},
        "age": {"N": "30"},
        "active": {"BOOL": True},
        "tags": {"L": [{"S": "admin"}, {"S": "beta"}]},
    },
)

# Get an item by its key
response = ddb.get_item(
    TableName="Users",
    Key={"userId": {"S": "u-001"}},
)
print(response["Item"]["name"]["S"])  # Alice

# Update an item
ddb.update_item(
    TableName="Users",
    Key={"userId": {"S": "u-001"}},
    UpdateExpression="SET age = :a, #st = :s",
    ExpressionAttributeValues={
        ":a": {"N": "31"},
        ":s": {"S": "premium"},
    },
    ExpressionAttributeNames={"#st": "status"},  # 'status' is a reserved word
)

# Delete an item
ddb.delete_item(
    TableName="Users",
    Key={"userId": {"S": "u-001"}},
)
```

## Hands-On: Composite Key (PK + SK)

The most powerful DynamoDB pattern. One table, many access patterns.

```python
ddb.create_table(
    TableName="Orders",
    KeySchema=[
        {"AttributeName": "customerId", "KeyType": "HASH"},
        {"AttributeName": "orderId", "KeyType": "RANGE"},
    ],
    AttributeDefinitions=[
        {"AttributeName": "customerId", "AttributeType": "S"},
        {"AttributeName": "orderId", "AttributeType": "S"},
    ],
    BillingMode="PAY_PER_REQUEST",
)

# Add orders for two customers
import time
for i in range(3):
    ddb.put_item(TableName="Orders", Item={
        "customerId": {"S": "cust-001"},
        "orderId": {"S": f"ord-{i:03d}"},
        "total": {"N": str(10 * (i + 1))},
        "status": {"S": "pending"},
        "createdAt": {"S": time.strftime("%Y-%m-%dT%H:%M:%SZ")},
    })

ddb.put_item(TableName="Orders", Item={
    "customerId": {"S": "cust-002"},
    "orderId": {"S": "ord-100"},
    "total": {"N": "99"},
    "status": {"S": "shipped"},
    "createdAt": {"S": time.strftime("%Y-%m-%dT%H:%M:%SZ")},
})

# Query all orders for a customer (uses partition key)
response = ddb.query(
    TableName="Orders",
    KeyConditionExpression="customerId = :c",
    ExpressionAttributeValues={":c": {"S": "cust-001"}},
)
print(f"cust-001 has {response['Count']} orders")  # 3

# Query with sort key range — orders starting with "ord-00"
response = ddb.query(
    TableName="Orders",
    KeyConditionExpression="customerId = :c AND begins_with(orderId, :prefix)",
    ExpressionAttributeValues={
        ":c": {"S": "cust-001"},
        ":prefix": {"S": "ord-00"},
    },
)
print(f"Matching orders: {response['Count']}")  # 3
```

## Hands-On: Global Secondary Index (GSI)

Query on non-key attributes. Essential for real applications.

```python
ddb.create_table(
    TableName="Products",
    KeySchema=[
        {"AttributeName": "productId", "KeyType": "HASH"},
    ],
    AttributeDefinitions=[
        {"AttributeName": "productId", "AttributeType": "S"},
        {"AttributeName": "category", "AttributeType": "S"},
        {"AttributeName": "price", "AttributeType": "N"},
    ],
    GlobalSecondaryIndexes=[{
        "IndexName": "category-price-index",
        "KeySchema": [
            {"AttributeName": "category", "KeyType": "HASH"},
            {"AttributeName": "price", "KeyType": "RANGE"},
        ],
        "Projection": {"ProjectionType": "ALL"},
    }],
    BillingMode="PAY_PER_REQUEST",
)

# Add products
products = [
    ("p-001", "electronics", 299),
    ("p-002", "electronics", 99),
    ("p-003", "books", 15),
    ("p-004", "books", 25),
    ("p-005", "electronics", 599),
]
for pid, cat, price in products:
    ddb.put_item(TableName="Products", Item={
        "productId": {"S": pid},
        "category": {"S": cat},
        "price": {"N": str(price)},
    })

# Query electronics under $300 — using the GSI
response = ddb.query(
    TableName="Products",
    IndexName="category-price-index",
    KeyConditionExpression="category = :c AND price < :p",
    ExpressionAttributeValues={
        ":c": {"S": "electronics"},
        ":p": {"N": "300"},
    },
)
for item in response["Items"]:
    print(item["productId"]["S"], item["price"]["N"])
# p-002 99
# p-001 299
```

## Hands-On: TTL (Time to Live)

Automatically delete items after a timestamp — great for sessions, caches, temporary data.

```python
import time

ddb.create_table(
    TableName="Sessions",
    KeySchema=[{"AttributeName": "sessionId", "KeyType": "HASH"}],
    AttributeDefinitions=[{"AttributeName": "sessionId", "AttributeType": "S"}],
    BillingMode="PAY_PER_REQUEST",
)

# Enable TTL on the "expiresAt" attribute
ddb.update_time_to_live(
    TableName="Sessions",
    TimeToLiveSpecification={
        "Enabled": True,
        "AttributeName": "expiresAt",
    },
)

# Create a session that expires in 1 hour
ddb.put_item(
    TableName="Sessions",
    Item={
        "sessionId": {"S": "sess-abc123"},
        "userId": {"S": "u-001"},
        "expiresAt": {"N": str(int(time.time()) + 3600)},  # epoch timestamp
    },
)
```

## Hands-On: Conditional Writes

Prevent overwriting existing items or enforce business rules atomically.

```python
from botocore.exceptions import ClientError

# Only create if item doesn't exist (idempotent create)
try:
    ddb.put_item(
        TableName="Users",
        Item={"userId": {"S": "u-001"}, "name": {"S": "Alice"}},
        ConditionExpression="attribute_not_exists(userId)",
    )
    print("Created!")
except ClientError as e:
    if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
        print("User already exists")

# Atomic counter — increment only if value is below limit
ddb.update_item(
    TableName="Users",
    Key={"userId": {"S": "u-001"}},
    UpdateExpression="SET loginCount = loginCount + :one",
    ConditionExpression="loginCount < :max",
    ExpressionAttributeValues={":one": {"N": "1"}, ":max": {"N": "100"}},
)
```

## Hands-On: Scan vs Query

**Query** — efficient, uses the index. Always prefer this.
**Scan** — reads every item. Expensive on large tables. Use only for admin/analytics.

```python
# Query (efficient) — requires knowing the partition key
response = ddb.query(
    TableName="Orders",
    KeyConditionExpression="customerId = :c",
    ExpressionAttributeValues={":c": {"S": "cust-001"}},
)

# Scan (expensive) — reads everything, then filters
response = ddb.scan(
    TableName="Orders",
    FilterExpression="#s = :status",
    ExpressionAttributeNames={"#s": "status"},
    ExpressionAttributeValues={":status": {"S": "pending"}},
)
# FilterExpression runs AFTER reading all items — you still pay for the full scan
```

## What's Different on Real AWS

- **Provisioned vs On-Demand** — `PAY_PER_REQUEST` (on-demand) is easiest to start with. Provisioned capacity is cheaper at predictable load.
- **Hot partitions** — if all traffic hits one partition key, you'll get throttled. Design keys to distribute load.
- **Item size limit** — 400KB per item. For larger data, store in S3 and keep the S3 key in DynamoDB.
- **GSI eventual consistency** — GSI updates are asynchronous. A write may not be immediately visible in a GSI query.
- **Costs** — charged per read/write unit and storage. On-demand is ~$1.25/million writes, $0.25/million reads.
- **DynamoDB Streams** — real-time change feed. MiniStack supports it — Lambda ESMs can poll it.

## Next Steps

- Connect **Lambda** to DynamoDB Streams to react to data changes in real time.
- Learn the **single-table design** pattern — one table, many entity types, using PK/SK prefixes.
- Try **TransactWriteItems** for multi-item atomic operations (like a bank transfer).
