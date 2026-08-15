  # Cloud (AWS) — Lead .NET Interview Prep

## Table of Contents

1. [Compute: ECS vs EKS vs Fargate vs Lambda](#1-compute-ecs-vs-eks-vs-fargate-vs-lambda)
2. [Storage & CDN: S3 + CloudFront](#2-storage--cdn-s3--cloudfront)
3. [IAM — Identity and Access Management](#3-iam--identity-and-access-management)
4. [Secrets Manager vs Parameter Store](#4-secrets-manager-vs-parameter-store)
5. [SQS vs SNS](#5-sqs-vs-sns)
6. [DynamoDB](#6-dynamodb)
7. [RDS](#7-rds)
8. [CloudWatch](#8-cloudwatch)
9. [Auto Scaling](#9-auto-scaling)
10. [ALB — Application Load Balancer](#10-alb--application-load-balancer)
11. [VPC & Networking](#11-vpc--networking)
12. [Route 53](#12-route-53)
13. [Scenario Questions & Answers](#13-scenario-questions--answers)

---

## 1. Compute: ECS vs EKS vs Fargate vs Lambda

### ECS (Elastic Container Service)

AWS-native container orchestration service. You define **Task Definitions** (container image, CPU, memory, env vars, IAM role) and run them as **Services** (long-running) or standalone **Tasks** (one-off jobs).

**Launch Types:**

| Feature | EC2 Launch Type | Fargate Launch Type |
|---|---|---|
| Server management | You manage EC2 instances | AWS manages all infra |
| Cost model | Pay for reserved EC2 capacity | Pay per vCPU/memory/second |
| Startup time | Fast (containers on running EC2) | Slightly slower (~10-30s cold start) |
| Best for | High throughput, cost optimization at scale | Simplicity, variable workloads |

### EKS (Elastic Kubernetes Service)

Managed Kubernetes control plane. AWS handles etcd, API server, upgrades. You manage worker nodes (EC2 or Fargate profiles). Best when:
- Team already knows Kubernetes
- Need Kubernetes-native features (custom CRDs, Helm charts, complex networking)
- Multi-cloud portability is a concern

### Fargate

Serverless compute engine for containers — works with **both ECS and EKS**. No EC2 instances to patch. Tasks are isolated at the VM boundary (better security). Charged per vCPU-second and GB-second.

### Lambda

Event-driven serverless functions. No container management.

**Key limits:**
- 15-minute max execution time
- 10 GB max memory
- 512 MB - 10 GB ephemeral storage
- 1,000 default concurrent executions per region (soft limit)
- Cold start problem: first invocation after idle spins up a new execution environment (~100ms-2s for .NET)

**Cold Start Mitigation for .NET:**
- Use **Provisioned Concurrency** (keeps N instances warm — costs money)
- Use .NET **Native AOT** (ahead-of-time compilation, drastically cuts cold start)
- Minimize package size (trim unused assemblies)
- Use Lambda SnapStart (for Java; .NET support limited — check current docs)

### Decision Matrix

| Scenario | Best Choice |
|---|---|
| Containerized .NET API, team owns infra | ECS Fargate |
| Need Kubernetes ecosystem (Helm, Istio) | EKS |
| Max cost efficiency, heavy steady traffic | ECS on EC2 |
| Event-driven, short-lived processing | Lambda |
| Batch jobs, infrequent heavy processing | ECS Fargate Tasks |
| Sub-100ms latency, no cold start tolerance | ECS Fargate (always-on) |

### Real Scenario: .NET API on ECS Fargate

```yaml
# ecs-task-definition.json (simplified)
{
  "family": "orders-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/orders-api-task-role",
  "containerDefinitions": [
    {
      "name": "orders-api",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/orders-api:latest",
      "portMappings": [{ "containerPort": 8080 }],
      "environment": [
        { "name": "ASPNETCORE_ENVIRONMENT", "value": "Production" }
      ],
      "secrets": [
        {
          "name": "ConnectionStrings__OrdersDb",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/orders/db-conn"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/orders-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```
Architecture: .NET API on ECS Fargate

   Internet
      |
   [Route 53]
      |
   [ALB]  (public subnet)
      |
 [ECS Service]
 [Fargate Task - .NET API]  (private subnet)
      |          |
   [RDS]      [Secrets Manager]
 (private subnet)
```

---

## 2. Storage & CDN: S3 + CloudFront

### S3 Storage Classes

| Class | Use Case | Min Duration | Retrieval |
|---|---|---|---|
| Standard | Hot data, frequent access | None | Immediate |
| Intelligent-Tiering | Unknown or variable access | None | Immediate |
| Standard-IA | Infrequent access, still fast | 30 days | Immediate |
| One Zone-IA | IA data, loss-tolerant | 30 days | Immediate |
| Glacier Instant | Archive with ms retrieval | 90 days | Milliseconds |
| Glacier Flexible | Archive | 90 days | Minutes-hours |
| Glacier Deep Archive | Long-term archive | 180 days | 12-48 hours |

### Lifecycle Policies

Automatically transition or delete objects:

```json
{
  "Rules": [
    {
      "ID": "archive-old-invoices",
      "Status": "Enabled",
      "Filter": { "Prefix": "invoices/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 2555 }
    }
  ]
}
```

### Presigned URLs

Grant temporary access to private S3 objects without exposing credentials:

```csharp
using Amazon.S3;
using Amazon.S3.Model;

public class S3Service
{
    private readonly IAmazonS3 _s3Client;

    public S3Service(IAmazonS3 s3Client)
    {
        _s3Client = s3Client;
    }

    public string GeneratePresignedDownloadUrl(string bucket, string key, int expiryMinutes = 15)
    {
        var request = new GetPreSignedUrlRequest
        {
            BucketName = bucket,
            Key = key,
            Verb = HttpVerb.GET,
            Expires = DateTime.UtcNow.AddMinutes(expiryMinutes)
        };
        return _s3Client.GetPreSignedURL(request);
    }

    public string GeneratePresignedUploadUrl(string bucket, string key, int expiryMinutes = 10)
    {
        var request = new GetPreSignedUrlRequest
        {
            BucketName = bucket,
            Key = key,
            Verb = HttpVerb.PUT,
            Expires = DateTime.UtcNow.AddMinutes(expiryMinutes),
            ContentType = "application/octet-stream"
        };
        return _s3Client.GetPreSignedURL(request);
    }
}
```

### Multipart Upload

Required for objects > 5 GB. Recommended for objects > 100 MB. Allows parallel upload, resume on failure:

```csharp
public async Task UploadLargeFileAsync(string bucket, string key, Stream fileStream)
{
    var transferUtility = new TransferUtility(_s3Client);
    var uploadRequest = new TransferUtilityUploadRequest
    {
        BucketName = bucket,
        Key = key,
        InputStream = fileStream,
        PartSize = 6_291_456, // 6 MB per part
        CannedACL = S3CannedACL.Private
    };
    await transferUtility.UploadAsync(uploadRequest);
}
```

### CloudFront

Global CDN with 400+ edge locations. Origins can be: S3 bucket, ALB, EC2, custom HTTP server.

**Key Concepts:**
- **Distribution:** CloudFront configuration with one or more origins
- **Origin:** where CloudFront fetches content from
- **Cache Behavior:** rules per path pattern (`/api/*` → no-cache, `/*.js` → cache 1 year)
- **Signed URLs / Cookies:** restrict access to authenticated users

**Cache Behaviors Example:**

```
CloudFront Distribution
├── /api/*          → Origin: ALB         (no cache, forward all headers)
├── /assets/*       → Origin: S3 bucket   (cache 1 year, compress)
├── /             → Origin: S3 bucket   (cache 5 min)
└── Default (*)    → Origin: ALB
```

### Real Example: S3 + CloudFront Static Site + Private Downloads

```
Architecture: Static Site + Private File Download

User browser
    |
[CloudFront Distribution]
    |                    |
[S3 - static site]    [ALB → .NET API]
                           |
                       [S3 - private docs]
                       (OAC - Origin Access Control)

Flow for private download:
1. User hits /api/files/{id}/download (.NET API)
2. API checks authorization
3. API generates presigned S3 URL (15 min expiry)
4. API returns 302 redirect to presigned URL
5. User downloads directly from S3
```

**CloudFront + S3 Origin Access Control (OAC):**
- S3 bucket is completely private (no public access)
- CloudFront assumes an IAM role to access the bucket
- Users cannot access S3 URLs directly — only through CloudFront

---

## 3. IAM — Identity and Access Management

### Core Entities

| Entity | Description |
|---|---|
| **User** | Human or service with long-term credentials (access key + secret) |
| **Group** | Collection of users — attach policies to group, not individual users |
| **Role** | Assumed by services/users temporarily — no long-term credentials |
| **Policy** | JSON document defining Allow/Deny on Actions and Resources |

### Policy Types

**Identity-Based Policy** — attached to User/Group/Role:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

**Resource-Based Policy** — attached to the resource (e.g., S3 bucket policy):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789:role/orders-api-task-role" },
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::private-docs/*"
    }
  ]
}
```

### IAM Roles for ECS Tasks

Every ECS task should have **two roles**:

| Role | Purpose |
|---|---|
| **Execution Role** | ECS agent pulls image from ECR, writes logs to CloudWatch, reads secrets from Secrets Manager at startup |
| **Task Role** | Your application code uses this to call AWS services at runtime (S3, SQS, DynamoDB, etc.) |

```json
// Task Role Policy for orders-api
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::orders-documents/*"
    },
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/orders/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage"],
      "Resource": "arn:aws:sqs:us-east-1:123456789:orders-queue"
    }
  ]
}
```

### Principle of Least Privilege

- Grant **minimum permissions** required to perform the function
- Use **conditions** to further restrict (e.g., only allow from specific VPC)
- Regularly **audit** unused permissions with IAM Access Analyzer
- Prefer **roles** over users for service-to-service communication

### Cross-Account Access

```
Account A (production) wants to read from Account B (shared services):

Account B creates a role with trust policy:
{
  "Principal": { "AWS": "arn:aws:iam::ACCOUNT-A-ID:root" },
  "Action": "sts:AssumeRole"
}

Account A code calls:
var stsClient = new AmazonSecurityTokenServiceClient();
var response = await stsClient.AssumeRoleAsync(new AssumeRoleRequest {
    RoleArn = "arn:aws:iam::ACCOUNT-B:role/SharedServicesRole",
    RoleSessionName = "orders-api-session"
});
```

---

## 4. Secrets Manager vs Parameter Store

### Comparison

| Feature | Secrets Manager | Parameter Store (SSM) |
|---|---|---|
| Cost | ~$0.40/secret/month | Free (standard), $0.05/10k API calls advanced |
| Auto-rotation | Yes — built-in Lambda rotation | No (manual or custom Lambda) |
| Max size | 65 KB | 4 KB (standard), 8 KB (advanced) |
| Versioning | Yes | Yes |
| Cross-account | Yes | Yes (with resource policy) |
| Best for | DB passwords, API keys, certificates | App config, feature flags, non-secret config |

### Parameter Store Types

| Type | Use |
|---|---|
| String | Plaintext config values |
| StringList | Comma-separated values |
| SecureString | Encrypted with KMS — treat as secret |

### When to Use What

- **Secrets Manager:** Database passwords, OAuth client secrets, TLS private keys — anything that should rotate
- **Parameter Store (SecureString):** Semi-sensitive config that doesn't need rotation
- **Parameter Store (String):** Environment-specific config (feature flags, URLs, timeouts)

### Real Example: .NET Reading from Secrets Manager

**NuGet:** `AWSSDK.SecretsManager`

```csharp
// Program.cs — inject secrets into IConfiguration
using Amazon.SecretsManager;
using Amazon.SecretsManager.Model;
using System.Text.Json;

public static class SecretsManagerExtensions
{
    public static IConfigurationBuilder AddSecretsManager(
        this IConfigurationBuilder builder,
        string secretArn)
    {
        var client = new AmazonSecretsManagerClient();
        var request = new GetSecretValueRequest { SecretId = secretArn };

        var response = client.GetSecretValueAsync(request).GetAwaiter().GetResult();
        var secretJson = response.SecretString;

        // Secret stored as JSON: {"username":"admin","password":"p@ssw0rd","host":"db.internal"}
        var secretDict = JsonSerializer.Deserialize<Dictionary<string, string>>(secretJson)!;

        // Map to ASP.NET Core configuration keys
        var configData = new Dictionary<string, string?>
        {
            ["ConnectionStrings:OrdersDb"] =
                $"Host={secretDict["host"]};Database=orders;Username={secretDict["username"]};Password={secretDict["password"]}"
        };

        builder.AddInMemoryCollection(configData);
        return builder;
    }
}

// Usage in Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddSecretsManager(
    secretArn: Environment.GetEnvironmentVariable("DB_SECRET_ARN")!
);
```

**Alternative: Use AWS .NET Configuration Extension:**

```csharp
// NuGet: Amazon.Extensions.Configuration.SystemsManager
// Works for both Secrets Manager and Parameter Store

builder.Configuration.AddSystemsManager(config =>
{
    config.Path = "/prod/orders-api";  // Parameter Store hierarchy
    config.ReloadAfter = TimeSpan.FromMinutes(5);
    config.Optional = false;
});
```

**Reading at Runtime (typed client):**

```csharp
public class DatabaseConnectionFactory
{
    private readonly IAmazonSecretsManager _secretsManager;
    private readonly IMemoryCache _cache;

    public DatabaseConnectionFactory(IAmazonSecretsManager secretsManager, IMemoryCache cache)
    {
        _secretsManager = secretsManager;
        _cache = cache;
    }

    public async Task<string> GetConnectionStringAsync()
    {
        return await _cache.GetOrCreateAsync("db-conn-string", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);

            var response = await _secretsManager.GetSecretValueAsync(new GetSecretValueRequest
            {
                SecretId = "prod/orders/db",
                VersionStage = "AWSCURRENT"
            });

            var secret = JsonSerializer.Deserialize<DbSecret>(response.SecretString)!;
            return $"Host={secret.Host};Database={secret.DbName};Username={secret.Username};Password={secret.Password}";
        });
    }
}
```

---

## 5. SQS vs SNS

### SQS (Simple Queue Service) — Pull-Based

Consumers **poll** the queue. Messages persist until consumed or TTL expires.

| Feature | Standard Queue | FIFO Queue |
|---|---|---|
| Ordering | Best-effort | Strict FIFO |
| Deduplication | Possible duplicates | 5-min dedup window |
| Throughput | Unlimited | 300 TPS (3000 with batching) |
| Use case | Max throughput, order not critical | Financial transactions, ordered events |

**Key Concepts:**
- **Visibility Timeout:** Message hidden while being processed (default 30s). If not deleted, reappears.
- **DLQ (Dead Letter Queue):** After N failed processing attempts, message moved to DLQ for investigation
- **Long Polling:** `WaitTimeSeconds=20` reduces empty responses, lowers cost
- **Batch Operations:** Process up to 10 messages per API call

### SNS (Simple Notification Service) — Push-Based

Publisher sends to a **topic**. SNS fans out to all **subscribers** (SQS queues, Lambda, HTTP endpoints, email, SMS).

**Fan-Out Pattern:**

```
[Order Service] → [SNS: order-placed-topic]
                        |
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
   [SQS: inventory] [SQS: billing] [SQS: notifications]
          |             |             |
   [Inventory       [Billing      [Email/SMS
    Service]         Service]      Service]
```

### Real Example: Order Placed Flow

```csharp
// NuGet: AWSSDK.SimpleNotificationService, AWSSDK.SQS

// 1. Publish to SNS
public class OrderEventPublisher
{
    private readonly IAmazonSimpleNotificationService _sns;
    private readonly string _topicArn;

    public OrderEventPublisher(IAmazonSimpleNotificationService sns, IConfiguration config)
    {
        _sns = sns;
        _topicArn = config["Aws:OrderPlacedTopicArn"]!;
    }

    public async Task PublishOrderPlacedAsync(OrderPlacedEvent orderEvent)
    {
        var message = JsonSerializer.Serialize(orderEvent);
        await _sns.PublishAsync(new PublishRequest
        {
            TopicArn = _topicArn,
            Message = message,
            Subject = "OrderPlaced",
            MessageAttributes = new Dictionary<string, MessageAttributeValue>
            {
                ["eventType"] = new MessageAttributeValue
                {
                    DataType = "String",
                    StringValue = "OrderPlaced"
                }
            }
        });
    }
}

// 2. Consume from SQS (Inventory Service)
public class InventoryQueueConsumer : BackgroundService
{
    private readonly IAmazonSQS _sqs;
    private readonly string _queueUrl;
    private readonly ILogger<InventoryQueueConsumer> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var response = await _sqs.ReceiveMessageAsync(new ReceiveMessageRequest
            {
                QueueUrl = _queueUrl,
                MaxNumberOfMessages = 10,
                WaitTimeSeconds = 20,  // Long polling
                VisibilityTimeout = 60
            }, stoppingToken);

            foreach (var message in response.Messages)
            {
                try
                {
                    // SNS wraps message in an envelope
                    var snsEnvelope = JsonSerializer.Deserialize<SnsEnvelope>(message.Body)!;
                    var orderEvent = JsonSerializer.Deserialize<OrderPlacedEvent>(snsEnvelope.Message)!;

                    await ProcessInventoryAsync(orderEvent);

                    // Delete only after successful processing
                    await _sqs.DeleteMessageAsync(_queueUrl, message.ReceiptHandle, stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to process message {MessageId}", message.MessageId);
                    // Do NOT delete — visibility timeout expires, message reappears
                    // After maxReceiveCount failures → moves to DLQ
                }
            }
        }
    }
}
```

### DLQ Strategy

```
SQS Queue
  maxReceiveCount: 3
  ↓ (after 3 failures)
DLQ (Dead Letter Queue)
  ↓ (CloudWatch Alarm on ApproximateNumberOfMessagesVisible > 0)
[SNS Alert → PagerDuty]
  ↓ (manual investigation or DLQ replay)
[Lambda: replay-dlq-messages → original queue]
```

---

## 6. DynamoDB

### Core Concepts

- **Partition Key (PK):** Determines which physical partition stores the item. Must be unique for a given table (unless combined with SK).
- **Sort Key (SK):** Secondary dimension within a partition. Enables range queries.
- **Item:** A record (like a row). Max 400 KB.
- **Attribute:** A field. Schema-less — not all items need the same attributes.

### GSI vs LSI

| Feature | GSI (Global Secondary Index) | LSI (Local Secondary Index) |
|---|---|---|
| Partition Key | Can be ANY attribute | Must be same as table PK |
| Sort Key | Optional | Required (different from table SK) |
| When defined | Anytime | Only at table creation |
| Storage | Separate partition | Same partition as base table |
| Consistency | Eventually consistent only | Strongly consistent available |
| Limit | 20 per table | 5 per table |

### Capacity Modes

| Mode | Use Case | Cost |
|---|---|---|
| Provisioned | Predictable traffic, significant scale | Set RCU/WCU, cheaper at steady load |
| On-Demand | Unpredictable traffic, new apps | Pay per request, no capacity planning |

**RCU/WCU:**
- 1 RCU = 1 strongly consistent read OR 2 eventually consistent reads per second for items up to 4 KB
- 1 WCU = 1 write per second for items up to 1 KB

### Single-Table Design

Design all entity types in ONE table using generic `PK`/`SK` and entity-type prefixes:

```
Single-Table Design: Order System

PK                  | SK                    | Type    | Data
--------------------|----------------------|---------|------------------
CUSTOMER#cust-001   | METADATA             | Customer| name, email, ...
CUSTOMER#cust-001   | ORDER#2024-001       | Order   | total, status, ...
CUSTOMER#cust-001   | ORDER#2024-002       | Order   | total, status, ...
ORDER#2024-001      | METADATA             | Order   | customerId, total
ORDER#2024-001      | ITEM#item-a          | OrderItem| productId, qty
ORDER#2024-001      | ITEM#item-b          | OrderItem| productId, qty
PRODUCT#prod-001    | METADATA             | Product | name, price, stock

Access Patterns:
- Get customer:                  PK=CUSTOMER#cust-001, SK=METADATA
- Get all customer orders:       PK=CUSTOMER#cust-001, SK begins_with ORDER#
- Get order with items:          PK=ORDER#2024-001 (all SK)
- Get specific order item:       PK=ORDER#2024-001, SK=ITEM#item-a
```

### DynamoDB C# Example

```csharp
// NuGet: AWSSDK.DynamoDBv2

// Using DynamoDB Document Model
public class OrderRepository
{
    private readonly IAmazonDynamoDB _dynamoDb;
    private readonly DynamoDBContext _context;

    public OrderRepository(IAmazonDynamoDB dynamoDb)
    {
        _dynamoDb = dynamoDb;
        _context = new DynamoDBContext(dynamoDb);
    }

    // Using low-level API for single-table design
    public async Task<Order?> GetOrderAsync(string orderId)
    {
        var response = await _dynamoDb.GetItemAsync(new GetItemRequest
        {
            TableName = "OrdersTable",
            Key = new Dictionary<string, AttributeValue>
            {
                ["PK"] = new AttributeValue { S = $"ORDER#{orderId}" },
                ["SK"] = new AttributeValue { S = "METADATA" }
            },
            ConsistentRead = true
        });

        if (!response.IsItemSet) return null;

        return new Order
        {
            OrderId = orderId,
            CustomerId = response.Item["CustomerId"].S,
            Total = decimal.Parse(response.Item["Total"].N),
            Status = response.Item["Status"].S
        };
    }

    // Query all items for an order (metadata + line items)
    public async Task<OrderDetail> GetOrderDetailAsync(string orderId)
    {
        var response = await _dynamoDb.QueryAsync(new QueryRequest
        {
            TableName = "OrdersTable",
            KeyConditionExpression = "PK = :pk",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>
            {
                [":pk"] = new AttributeValue { S = $"ORDER#{orderId}" }
            }
        });

        var order = new OrderDetail();
        foreach (var item in response.Items)
        {
            var sk = item["SK"].S;
            if (sk == "METADATA")
                order.Metadata = MapToOrder(item);
            else if (sk.StartsWith("ITEM#"))
                order.Items.Add(MapToOrderItem(item));
        }
        return order;
    }
}
```

---

## 7. RDS

### Multi-AZ vs Read Replicas

| Feature | Multi-AZ | Read Replicas |
|---|---|---|
| Purpose | High availability / failover | Read scalability / offload reads |
| Replication | Synchronous | Asynchronous |
| Standby accessible? | No (standby only for failover) | Yes (read traffic) |
| Failover time | ~1-2 minutes (automatic) | Manual promotion |
| Cross-region | No (same region, different AZ) | Yes (cross-region possible) |
| Cost | 2x instance cost | Additional instance cost per replica |

### RDS Proxy

Connection pooling layer between application and RDS. Critical for serverless scenarios (Lambda opens/closes connections aggressively — overwhelms RDS).

```
Lambda functions (100s of concurrent)
        ↓
[RDS Proxy] ← maintains pool of 10-20 DB connections
        ↓
[RDS PostgreSQL]
```

**Benefits:**
- Reduces connection overhead for serverless
- Handles failover transparently (faster than direct RDS failover)
- Integrates with IAM and Secrets Manager

### .NET API with RDS PostgreSQL

```csharp
// NuGet: Npgsql.EntityFrameworkCore.PostgreSQL

// appsettings.Production.json — connection string loaded from Secrets Manager
// "ConnectionStrings": { "OrdersDb": "<injected at startup>" }

// Program.cs
builder.Services.AddDbContext<OrdersDbContext>(options =>
    options.UseNpgsql(
        builder.Configuration.GetConnectionString("OrdersDb"),
        npgsqlOptions =>
        {
            // Resilience for transient failures
            npgsqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorCodesToAdd: null
            );
            npgsqlOptions.CommandTimeout(30);
        }
    )
);

// Health check
builder.Services.AddHealthChecks()
    .AddNpgSql(
        builder.Configuration.GetConnectionString("OrdersDb")!,
        name: "orders-db",
        tags: new[] { "db", "postgresql" }
    );
```

### Backup Strategy

- **Automated Backups:** Enabled by default (1-35 day retention). Stored in S3. Enables point-in-time recovery (PITR).
- **Manual Snapshots:** On-demand, persist until deleted (survive instance deletion).
- **Restore:** Creates a new RDS instance — you update your connection string.

---

## 8. CloudWatch

### Core Components

| Component | Description |
|---|---|
| **Metrics** | Time-series data (CPU, request count, latency) |
| **Logs** | Log streams aggregated into log groups |
| **Alarms** | Trigger on metric threshold → SNS, Auto Scaling, etc. |
| **Dashboards** | Visual metric displays |
| **Log Insights** | SQL-like query language for logs |
| **Container Insights** | Enhanced metrics for ECS/EKS |

### Log Insights Queries

```sql
-- Find all 5xx errors in last hour
fields @timestamp, @message
| filter @message like /HTTP\/[0-9\.]+ 5[0-9][0-9]/
| sort @timestamp desc
| limit 100

-- Count errors by endpoint
fields @timestamp, requestPath, statusCode
| filter statusCode >= 500
| stats count(*) as errorCount by requestPath
| sort errorCount desc

-- P99 latency per endpoint (structured JSON logs)
fields @timestamp, path, durationMs
| stats pct(durationMs, 99) as p99, avg(durationMs) as avg by path
| sort p99 desc
```

### Custom Metrics from .NET App

```csharp
// NuGet: AWSSDK.CloudWatch

public class CloudWatchMetricsService
{
    private readonly IAmazonCloudWatch _cloudWatch;
    private const string Namespace = "OrdersApi";

    public CloudWatchMetricsService(IAmazonCloudWatch cloudWatch)
    {
        _cloudWatch = cloudWatch;
    }

    public async Task RecordOrderProcessedAsync(string region, double processingTimeMs)
    {
        await _cloudWatch.PutMetricDataAsync(new PutMetricDataRequest
        {
            Namespace = Namespace,
            MetricData = new List<MetricDatum>
            {
                new MetricDatum
                {
                    MetricName = "OrdersProcessed",
                    Value = 1,
                    Unit = StandardUnit.Count,
                    Timestamp = DateTime.UtcNow,
                    Dimensions = new List<Dimension>
                    {
                        new Dimension { Name = "Environment", Value = "Production" },
                        new Dimension { Name = "Region", Value = region }
                    }
                },
                new MetricDatum
                {
                    MetricName = "OrderProcessingTime",
                    Value = processingTimeMs,
                    Unit = StandardUnit.Milliseconds,
                    Timestamp = DateTime.UtcNow
                }
            }
        });
    }
}

// Better approach: use EMF (Embedded Metrics Format) via structured logs
// CloudWatch automatically extracts metrics from structured log entries
// NuGet: Amazon.CloudWatch.EMF

// In a log message (Serilog/structured logging):
logger.Information("OrderProcessed {@Metrics}", new {
    OrderId = orderId,
    ProcessingTimeMs = sw.ElapsedMilliseconds,
    // EMF format — CloudWatch extracts these as metrics automatically
    _aws = new {
        Namespace = "OrdersApi",
        Metrics = new[] { new { Name = "OrdersProcessed", Unit = "Count" } }
    }
});
```

### Real Example: 5xx Alarm → PagerDuty

```
CloudWatch Alarm:
  Metric: AWS/ApplicationELB → HTTPCode_Target_5XX_Count
  Threshold: > 10 errors in 1 minute
  Actions:
    → SNS Topic: prod-alerts
      → PagerDuty (HTTPS subscriber)
      → Slack (Lambda subscriber)
      → Email (direct subscriber)
```

---

## 9. Auto Scaling

### EC2 Auto Scaling Groups

Scale EC2 instances in/out based on policies:

| Policy Type | Trigger | Use Case |
|---|---|---|
| Target Tracking | Keep metric at target (e.g., CPU=60%) | Most common, self-adjusting |
| Step Scaling | Step changes per alarm threshold | Precise control over scale steps |
| Scheduled | Fixed time/cron | Predictable traffic patterns |
| Predictive | ML-based forecast | Variable but patterned traffic |

### ECS Service Auto Scaling

**Target Tracking — keep CPU at 70%:**

```json
{
  "PolicyType": "TargetTrackingScaling",
  "TargetTrackingScalingPolicyConfiguration": {
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }
}
```

### Scale on SQS Queue Depth

Scale worker ECS tasks based on number of messages in queue:

```json
{
  "PolicyType": "TargetTrackingScaling",
  "TargetTrackingScalingPolicyConfiguration": {
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [
        { "Name": "QueueName", "Value": "orders-processing-queue" }
      ],
      "Statistic": "Average"
    },
    "TargetValue": 100.0,
    "ScaleInCooldown": 120,
    "ScaleOutCooldown": 30
  }
}
```

**Interpretation:** Maintain 1 ECS task per 100 messages in queue. If queue has 500 messages → scale to 5 tasks.

### Real Example: Scale API Workers on Queue Length

```
SQS Queue depth → Custom Metric
        |
CloudWatch Alarm (> 100 messages per task)
        |
Application Auto Scaling
        |
ECS Service (min: 1, max: 50 tasks)
        |
Orders Processing Workers (.NET BackgroundService)

Scale-out cooldown: 30s  (scale fast when busy)
Scale-in cooldown:  120s (drain carefully before removing)
```

---

## 10. ALB — Application Load Balancer

### Core Components

- **Listener:** Port + protocol (HTTP:80, HTTPS:443) with SSL certificate
- **Rules:** Conditions (path, host, header) → Actions (forward, redirect, fixed-response)
- **Target Group:** Registered targets (ECS tasks, EC2, Lambda, IPs) + health check config
- **Health Check:** HTTP probe against each target. Unhealthy targets removed from rotation.

### Path-Based Routing

```
ALB Listener (HTTPS:443)
├── Rule: path-pattern /api/v1/*   → Target Group: api-v1-tg  (ECS service v1)
├── Rule: path-pattern /api/v2/*   → Target Group: api-v2-tg  (ECS service v2)
├── Rule: path-pattern /health     → Fixed response: 200 OK
└── Default:                       → Target Group: api-v2-tg
```

### Blue/Green with Weighted Target Groups

```
Deployment Progress:

Day 0:  api-blue  100%  |  api-green  0%
Day 1:  api-blue   90%  |  api-green  10%   (canary — watch errors)
Day 2:  api-blue   50%  |  api-green  50%   (ramp up)
Day 3:  api-blue    0%  |  api-green  100%  (fully shifted)
Day 4:  Decommission blue environment
```

```json
// ALB Rule Action for weighted routing
{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      { "TargetGroupArn": "arn:...api-blue-tg", "Weight": 10 },
      { "TargetGroupArn": "arn:...api-green-tg", "Weight": 90 }
    ],
    "StickinessDuration": 0
  }
}
```

### ECS Rolling Update

```json
{
  "deploymentConfiguration": {
    "minimumHealthyPercent": 100,
    "maximumPercent": 200
  }
}
// ECS launches new tasks (200% capacity) → validates health → drains old tasks
// At minimumHealthyPercent=100: always full capacity during deployment
```

### Sticky Sessions

ALB can route a user to the same target using a **duration-based cookie** (`AWSALB`). Useful for stateful apps, but avoid if possible — prefer stateless (JWT, Redis session).

---

## 11. VPC & Networking

### VPC Components

| Component | Description |
|---|---|
| **VPC** | Virtual private cloud with CIDR block (e.g., 10.0.0.0/16) |
| **Subnet** | Sub-range of VPC CIDR, tied to one AZ (e.g., 10.0.1.0/24) |
| **Internet Gateway** | Allows internet traffic for public subnets |
| **NAT Gateway** | Allows private subnet resources to reach internet (outbound only) |
| **Route Table** | Rules for how to route traffic (associated with subnets) |
| **Security Group** | Stateful firewall at ENI level (allows return traffic automatically) |
| **NACL** | Stateless firewall at subnet level (must explicitly allow return traffic) |

### Security Groups vs NACLs

| Feature | Security Groups | NACLs |
|---|---|---|
| Level | Instance / ENI | Subnet |
| Stateful | Yes (return traffic auto-allowed) | No (must allow both directions) |
| Rules | Allow only (no explicit deny) | Allow AND Deny |
| Evaluation | All rules evaluated together | Rules evaluated in order by number |
| Default | Deny all inbound | Allow all (default NACL) |

### VPC Peering vs Transit Gateway

| Feature | VPC Peering | Transit Gateway |
|---|---|---|
| Scale | Point-to-point (grows complex) | Hub-and-spoke (scales to 1000s of VPCs) |
| Cross-account | Yes | Yes |
| Cross-region | Yes (inter-region peering) | Yes |
| Transitive routing | No | Yes |
| Cost | Data transfer only | Per attachment + data transfer |
| Use case | 2-5 VPCs | Large organization, many VPCs |

### PrivateLink

Access AWS services (S3, Secrets Manager, SQS) from private subnets **without traffic leaving AWS network** (no NAT Gateway needed):

```
ECS Task (private subnet)
    |
[VPC Endpoint - Gateway type for S3]
    |
S3 (no internet traversal)

[VPC Endpoint - Interface type for Secrets Manager]
    |
Secrets Manager (via private IP in your VPC)
```

### Real Example: 3-Tier Architecture

```
                    Internet
                       |
              [Internet Gateway]
                       |
         ┌─────────────────────────┐
         │     Public Subnet       │
         │  [ALB]                  │
         └───────────┬─────────────┘
                     │
         ┌───────────▼─────────────┐
         │     Private Subnet      │
         │  [ECS Fargate Tasks]    │
         │  [NAT Gateway]          │
         └───────────┬─────────────┘
                     │
         ┌───────────▼─────────────┐
         │  Data/Isolated Subnet   │
         │  [RDS Multi-AZ]         │
         │  [ElastiCache]          │
         └─────────────────────────┘

Security Groups:
  ALB SG:     Inbound 443 from 0.0.0.0/0
  ECS SG:     Inbound 8080 from ALB SG only
  RDS SG:     Inbound 5432 from ECS SG only
  Cache SG:   Inbound 6379 from ECS SG only
```

---

## 12. Route 53

### Record Types

| Type | Purpose | Example |
|---|---|---|
| **A** | Domain → IPv4 | `api.example.com → 1.2.3.4` |
| **AAAA** | Domain → IPv6 | `api.example.com → 2001:db8::1` |
| **CNAME** | Domain → another domain | `www → example.com` (cannot use at zone apex) |
| **ALIAS** | Like CNAME but works at zone apex, free for AWS resources | `example.com → alb.aws.com` |
| **MX** | Mail exchange | Email routing |
| **TXT** | Text records | SPF, DKIM, domain verification |

**ALIAS vs CNAME:** Use ALIAS for AWS resources (ALB, CloudFront, S3 static website). ALIAS has no charge for queries and works at apex.

### Routing Policies

| Policy | Description | Use Case |
|---|---|---|
| **Simple** | Returns single value (or random from multiple) | Basic single-region setup |
| **Weighted** | Split traffic by percentage | A/B testing, blue/green |
| **Latency** | Route to lowest latency region | Multi-region performance |
| **Failover** | Primary/secondary with health check | Disaster recovery |
| **Geolocation** | Route based on user's geographic location | Content localization, compliance |
| **Geoproximity** | Route based on distance, with bias adjustment | Fine-tune geographic routing |
| **Multivalue** | Returns multiple IPs with health checking | Simple load balancing |

### Real Example: Multi-Region Failover

```
Route 53 Health Check → api.example.com (primary: us-east-1)
    |
If health check fails (3 consecutive failures over 30s):
    |
Automatic failover to secondary: us-west-2

DNS Records:
  api.example.com  ALIAS  us-east-1 ALB  Failover=PRIMARY  HealthCheck=enabled
  api.example.com  ALIAS  us-west-2 ALB  Failover=SECONDARY

TTL: 60s (low TTL so failover propagates quickly)
```

---

## 13. Scenario Questions & Answers

---

### Q: How do you deploy a .NET API with zero downtime?

**Strategy 1 — ECS Rolling Update:**

```
1. New task definition registered (new Docker image)
2. ECS starts new tasks (maximumPercent: 200 → double capacity)
3. New tasks pass health check (ALB confirms /health returns 200)
4. ECS drains old tasks (stops sending new requests, waits for in-flight to complete)
5. Old tasks terminated
6. Result: Zero downtime, no user requests dropped
```

**Strategy 2 — Blue/Green with ALB:**

```
1. Deploy "green" ECS service with new code alongside "blue" (current)
2. Register green tasks in a new target group
3. Add ALB rule: 10% → green, 90% → blue
4. Monitor error rates, latency, business metrics for green
5. If healthy: gradually shift traffic (10→50→100%)
6. If problem detected: shift 100% back to blue instantly (< 1 minute)
7. Decommission blue after confidence period
```

**Database Migration Strategy:**

```
NEVER do breaking schema changes in one deployment.

Pattern: Expand-Contract
Phase 1 (backward compatible change):
  - Add new_column (nullable) alongside old_column
  - Deploy code that writes to BOTH columns, reads from old_column

Phase 2:
  - Backfill: UPDATE table SET new_column = old_column WHERE new_column IS NULL
  - Deploy code that writes to BOTH, reads from new_column

Phase 3:
  - Deploy code that only writes to new_column
  - Drop old_column (in a future release)
```

---

### Q: How do you scale your .NET API for high traffic?

**Full Architecture Answer:**

```
Scaling Strategy — Layered Approach:

Layer 1: CDN (CloudFront)
  - Cache static assets, reduce origin load
  - Eliminate requests that don't need backend

Layer 2: Load Balancing (ALB)
  - Distribute across multiple AZs
  - Health check removes unhealthy instances

Layer 3: Compute (ECS Auto Scaling)
  - Target CPU at 70%
  - Scale out in 60s, scale in with 5-min cooldown
  - Min: 2 tasks (HA across AZs), Max: 50 tasks

Layer 4: Caching (ElastiCache Redis)
  - Cache: reference data, user sessions, expensive queries
  - Reduce DB load by 80%+

Layer 5: Database
  - Read Replicas: route SELECT queries to replicas
  - RDS Proxy: connection pooling (critical for burst traffic)
  - Query optimization + proper indexing

Layer 6: Async Processing (SQS)
  - Move non-critical work off request path
  - Order confirmation email → SQS → worker service
  - Improves API response time, user experience
```

**ECS Auto Scaling Config:**

```json
{
  "MinCapacity": 2,
  "MaxCapacity": 50,
  "Policies": [
    {
      "Name": "cpu-target-tracking",
      "TargetValue": 70,
      "Metric": "ECSServiceAverageCPUUtilization",
      "ScaleOutCooldown": 60,
      "ScaleInCooldown": 300
    },
    {
      "Name": "memory-target-tracking",
      "TargetValue": 80,
      "Metric": "ECSServiceAverageMemoryUtilization",
      "ScaleOutCooldown": 60,
      "ScaleInCooldown": 300
    }
  ]
}
```

---

### Q: How do you secure secrets in AWS?

**Answer — Never in code, always in managed services:**

```
BAD:
  appsettings.json: { "DbPassword": "p@ssw0rd" }  ← NEVER
  Environment variable: DB_PASSWORD=p@ssw0rd       ← better but still risky
  Dockerfile ENV:  ENV DB_PASSWORD=p@ssw0rd        ← NEVER (in image layers)

GOOD:
  1. Store in Secrets Manager
  2. ECS Task Role has GetSecretValue permission (IAM role, no credentials)
  3. App reads at startup via AWS SDK
  4. Secret rotated automatically by Secrets Manager Lambda
  5. Old secret version available for 24h during rotation window
```

**Defense in Depth:**

```
Layer 1: No secrets in code/config files (git history prevention)
  → .gitignore secrets files
  → git-secrets pre-commit hook

Layer 2: Secrets Manager with auto-rotation
  → DB password rotated every 30 days
  → API keys rotated every 90 days

Layer 3: IAM Least Privilege
  → ECS task role ONLY has access to its own secrets
  → Cannot access other services' secrets

Layer 4: KMS Encryption
  → Secrets Manager uses KMS CMK for encryption
  → Separate KMS keys per environment (dev/staging/prod)

Layer 5: Network
  → VPC Endpoint for Secrets Manager
  → Traffic never traverses internet
  → Security Group restricts which ECS tasks can reach endpoint

Layer 6: Audit
  → CloudTrail logs every GetSecretValue call
  → Alert on unexpected access patterns
```

---

### Q: Walk me through your autoscaling strategy for a worker service processing an SQS queue.

**Answer:**

```
Goal: Process SQS queue quickly without over-provisioning.

Metric: ApproximateNumberOfMessagesVisible (queue depth)
Target: 100 messages per worker task

Algorithm:
  Current tasks = max(minCapacity, ceil(queueDepth / 100))

Example:
  Queue depth: 0     → 1 task (minimum)
  Queue depth: 50    → 1 task (ceil(50/100) = 1)
  Queue depth: 150   → 2 tasks
  Queue depth: 1000  → 10 tasks
  Queue depth: 5001  → 50 tasks (capped at maximum)

Scale-Out:
  Cooldown: 30s (scale fast, messages are piling up)
  Behavior: Add multiple tasks at once (step, not one at a time)

Scale-In:
  Cooldown: 120s (don't scale in too fast — messages take time to process)
  Behavior: Drain task gracefully (complete current batch before terminating)

Drain pattern in .NET:
  protected override async Task ExecuteAsync(CancellationToken stoppingToken)
  {
      while (!stoppingToken.IsCancellationRequested)
      {
          // Process batch
          await ProcessBatchAsync(stoppingToken);
      }
      // stoppingToken cancelled → ECS sends SIGTERM
      // Task completes current batch, exits cleanly
      // ECS waits stopTimeout (30s) before SIGKILL
  }
```

**Full Architecture:**

```
[SQS Queue] ← Orders placed by API
     |
     | (ApproximateNumberOfMessagesVisible metric)
     |
[CloudWatch] → [Application Auto Scaling]
                       |
               [ECS Service: order-workers]
                       |
               [.NET BackgroundService]
               (processes 10 msgs/batch, 20s long-poll)
                       |
               [RDS / DynamoDB / SNS]
```

---

## Quick Reference Cheat Sheet

```
Service Selection:
  Run containers (simple):  ECS Fargate
  Run containers (k8s):     EKS
  Event-driven functions:   Lambda
  Object storage:           S3
  Block storage (EC2):      EBS
  File storage:             EFS
  Queue:                    SQS
  Pub/Sub:                  SNS
  Key-value DB:             DynamoDB
  Relational DB:            RDS
  Cache:                    ElastiCache (Redis/Memcached)
  CDN:                      CloudFront
  DNS:                      Route 53
  Load balancer:            ALB (HTTP) / NLB (TCP)
  Secrets:                  Secrets Manager
  Config:                   Parameter Store (SSM)
  Monitoring:               CloudWatch
  Tracing:                  X-Ray
  CI/CD:                    CodePipeline + CodeBuild

IAM Rule of Thumb:
  Service-to-service:       IAM Role (no access keys)
  Human access:             IAM User → Group → Policy
  Cross-account:            Role with trust policy + STS AssumeRole
  Temp credentials:         STS

Networking Rule of Thumb:
  Public-facing:            Public subnet + Internet Gateway + Elastic IP
  Internal service:         Private subnet + NAT Gateway (for outbound)
  DB tier:                  Isolated subnet (no route table internet entry)
  AWS service access:       VPC Endpoint (Gateway for S3/DynamoDB, Interface for others)
```