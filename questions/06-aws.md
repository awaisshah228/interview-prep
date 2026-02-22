# AWS Cloud Services - Interview Q&A

> 15+ questions covering SQS/SNS, ECS, DynamoDB, S3, Lambda, VPC, and AWS architecture

---

## Table of Contents

- [Messaging (SQS, SNS)](#messaging-sqs-sns)
- [Compute (ECS, Lambda)](#compute-ecs-lambda)
- [Storage & Database](#storage--database)
- [Networking & Security](#networking--security)
- [Architecture Patterns](#architecture-patterns)

---

## Messaging (SQS, SNS)

### Q1: SQS Standard vs FIFO.

**Answer:**

| Feature | Standard | FIFO |
|---|---|---|
| Throughput | Unlimited | 300 msg/s (3,000 with batching) |
| Ordering | Best-effort | Strict FIFO (per message group) |
| Delivery | At-least-once (may duplicate) | Exactly-once |
| Deduplication | None | 5-min dedup window |
| Naming | Any name | Must end with `.fifo` |
| Cost | Lower | ~25% more |

```
When to use Standard:
✅ High throughput needs (analytics, logging)
✅ Order doesn't matter
✅ Idempotent consumers (can handle duplicates)

When to use FIFO:
✅ Strict ordering required (trade execution sequence)
✅ No duplicates allowed (payment processing)
✅ Lower throughput acceptable

Message Groups (FIFO):
- Messages in SAME group → strictly ordered
- Messages in DIFFERENT groups → processed in parallel
- Use userId as MessageGroupId → user's trades are in order
  while different users' trades process in parallel
```

---

### Q2: SQS DLQ (Dead Letter Queue) and Visibility Timeout.

**Answer:**

```
DLQ Flow:
┌──────────┐     process     ┌──────────┐
│ Producer │────────────────▶│  Main Q  │
└──────────┘                 └────┬─────┘
                                  │ fails 3 times
                                  ▼
                             ┌──────────┐
                             │   DLQ    │ ← inspect, fix, replay
                             └──────────┘

Visibility Timeout:
┌──────────┐     receive     ┌──────────┐
│ Consumer │◀────────────────│   SQS    │
│          │                 │  Queue   │
│processing│                 │          │
│  ...     │  message hidden │ (30s     │
│  done!   │  from others    │  default)│
│  delete  │────────────────▶│          │
└──────────┘                 └──────────┘

If consumer crashes before delete:
- Message becomes visible again after timeout
- Another consumer picks it up (retry)
- After maxReceiveCount → moves to DLQ
```

```typescript
// SQS Configuration
const queueConfig = {
  QueueName: 'trade-processing',
  Attributes: {
    VisibilityTimeout: '60',        // 60 seconds to process
    MessageRetentionPeriod: '1209600', // 14 days max retention
    ReceiveMessageWaitTimeSeconds: '20', // long polling (reduce costs)
    RedrivePolicy: JSON.stringify({
      deadLetterTargetArn: DLQ_ARN,
      maxReceiveCount: 3,            // after 3 failures → DLQ
    }),
  },
};

// DLQ monitoring with CloudWatch alarm
// Alert when messages appear in DLQ
{
  AlarmName: 'trade-dlq-messages',
  MetricName: 'ApproximateNumberOfMessagesVisible',
  Namespace: 'AWS/SQS',
  Dimensions: [{ Name: 'QueueName', Value: 'trade-processing-dlq' }],
  Threshold: 0,
  ComparisonOperator: 'GreaterThanThreshold',
  AlarmActions: [SNS_ALERT_TOPIC_ARN],
}

// Replaying DLQ messages
// 1. Read from DLQ
// 2. Fix the issue (code fix, data fix)
// 3. Send back to main queue
// 4. Delete from DLQ
// AWS now supports "DLQ redrive" in console (one-click replay)
```

---

### Q3: SNS - Topics, Subscriptions, and Filtering.

**Answer:**

```typescript
// Create topic
const { TopicArn } = await sns.createTopic({ Name: 'trade-events' });

// Subscribe different endpoints
await sns.subscribe({
  TopicArn,
  Protocol: 'sqs',
  Endpoint: analyticsQueueArn,
  Attributes: {
    // Filter: only receive SOL trades
    FilterPolicy: JSON.stringify({
      symbol: ['SOL', 'ETH'],
      eventType: ['TRADE_EXECUTED'],
      amount: [{ numeric: ['>=', 1000] }], // trades >= 1000
    }),
  },
});

await sns.subscribe({
  TopicArn,
  Protocol: 'sqs',
  Endpoint: allTradesQueueArn,
  // No filter → receives everything
});

await sns.subscribe({
  TopicArn,
  Protocol: 'https',
  Endpoint: 'https://webhook.myapp.com/trades',
  // HTTP endpoint subscription
});

await sns.subscribe({
  TopicArn,
  Protocol: 'lambda',
  Endpoint: lambdaArn,
  // Direct Lambda invocation
});

// Publish with attributes (for filtering)
await sns.publish({
  TopicArn,
  Message: JSON.stringify(tradeData),
  MessageAttributes: {
    symbol: { DataType: 'String', StringValue: 'SOL' },
    eventType: { DataType: 'String', StringValue: 'TRADE_EXECUTED' },
    amount: { DataType: 'Number', StringValue: '5000' },
  },
});

// SNS supports: SQS, Lambda, HTTP/S, Email, SMS, Kinesis Firehose
// Filter policy reduces unnecessary message delivery (cost + performance)
```

---

## Compute (ECS, Lambda)

### Q4: ECS Fargate vs EC2 Launch Type.

**Answer:**

| Feature | Fargate | EC2 |
|---|---|---|
| Management | Serverless (no servers) | You manage instances |
| Pricing | Per vCPU + memory/second | EC2 instance pricing |
| Startup | 30-60 seconds | Faster (if instances running) |
| GPU | Limited | Full support |
| SSH access | No | Yes |
| Custom AMI | No | Yes |
| Cost (steady) | Higher | Lower (reserved/spot) |
| Cost (bursty) | Lower | Higher (idle resources) |

```typescript
// Task Definition (Fargate)
{
  family: 'trade-api',
  networkMode: 'awsvpc',              // required for Fargate
  requiresCompatibilities: ['FARGATE'],
  cpu: '512',                          // 0.5 vCPU
  memory: '1024',                     // 1 GB
  executionRoleArn: 'arn:aws:iam::role/ecsTaskExecutionRole',
  taskRoleArn: 'arn:aws:iam::role/tradeApiRole', // for AWS SDK calls
  containerDefinitions: [{
    name: 'trade-api',
    image: `${ECR_REPO}:${IMAGE_TAG}`,
    portMappings: [{ containerPort: 3000, protocol: 'tcp' }],
    environment: [
      { name: 'NODE_ENV', value: 'production' },
    ],
    secrets: [
      // From Secrets Manager or Parameter Store
      { name: 'DB_PASSWORD', valueFrom: 'arn:aws:secretsmanager:...' },
    ],
    logConfiguration: {
      logDriver: 'awslogs',
      options: {
        'awslogs-group': '/ecs/trade-api',
        'awslogs-region': 'us-east-1',
        'awslogs-stream-prefix': 'ecs',
      },
    },
    healthCheck: {
      command: ['CMD-SHELL', 'curl -f http://localhost:3000/health || exit 1'],
      interval: 30, timeout: 5, retries: 3, startPeriod: 60,
    },
  }],
}

// Auto Scaling
{
  ServiceName: 'trade-api',
  ScalableDimension: 'ecs:service:DesiredCount',
  MinCapacity: 2,
  MaxCapacity: 20,
  // Scale on CPU
  TargetTrackingScalingPolicy: {
    TargetValue: 70, // target 70% CPU
    PredefinedMetricSpecification: {
      PredefinedMetricType: 'ECSServiceAverageCPUUtilization',
    },
    ScaleInCooldown: 300,  // wait 5 min before scaling in
    ScaleOutCooldown: 60,  // wait 1 min before scaling out
  },
}
```

---

### Q5: Lambda - When to use, cold starts, and patterns.

**Answer:**

```
When to use Lambda:
✅ Event-driven processing (S3 uploads, SQS messages, API Gateway)
✅ Scheduled tasks (crons)
✅ Lightweight APIs (< 15 min execution)
✅ Data transformation pipelines
✅ Webhook handlers

When NOT to use Lambda:
❌ Long-running processes (>15 min)
❌ Consistent low-latency requirements
❌ WebSocket connections
❌ Heavy computation (use ECS/EC2)
❌ Large memory/storage needs (>10 GB / 10 GB)

Cold Start Optimization:
1. Provisioned Concurrency (keep instances warm)
2. Smaller package size (less to load)
3. Use Node.js or Python (fastest cold starts)
4. Lazy-load SDK clients (don't import all of AWS SDK)
5. Keep handler function lean
```

```typescript
// Lambda handler pattern
import { SQSHandler, SQSEvent } from 'aws-lambda';

// Initialize outside handler (reused across invocations)
const db = new Pool({ connectionString: process.env.DATABASE_URL });

export const handler: SQSHandler = async (event: SQSEvent) => {
  const results = await Promise.allSettled(
    event.Records.map(async (record) => {
      const trade = JSON.parse(record.body);
      await db.query(
        'INSERT INTO trades (symbol, amount, price) VALUES ($1, $2, $3)',
        [trade.symbol, trade.amount, trade.price]
      );
    })
  );

  // Partial batch failure handling
  const failures = results
    .map((r, i) => r.status === 'rejected' ? event.Records[i].messageId : null)
    .filter(Boolean);

  return {
    batchItemFailures: failures.map(id => ({ itemIdentifier: id })),
  };
};

// Lambda layers: shared dependencies across functions
// Lambda destinations: route success/failure to SQS/SNS/Lambda
// Lambda@Edge: run at CloudFront edge locations
```

---

## Storage & Database

### Q6: S3 - Key features and patterns.

**Answer:**

```
S3 Storage Classes:
Standard        → Frequent access, low latency (~$0.023/GB)
Standard-IA     → Infrequent, min 30 days (~$0.0125/GB)
One Zone-IA     → Single AZ, non-critical (~$0.01/GB)
Glacier Instant → Archive, ms retrieval (~$0.004/GB)
Glacier Flexible→ Archive, min-hours retrieval (~$0.0036/GB)
Glacier Deep    → Long-term archive, 12-48h retrieval (~$0.00099/GB)
Intelligent     → Auto-moves between tiers
```

```typescript
// Pre-signed URLs (secure temporary access)
const url = await s3.getSignedUrl('putObject', {
  Bucket: 'trade-documents',
  Key: `uploads/${userId}/${filename}`,
  ContentType: 'application/pdf',
  Expires: 300, // 5 minutes
});
// Client uploads directly to S3 (no server proxy needed)

// Lifecycle rules
{
  Rules: [{
    ID: 'archive-old-trades',
    Status: 'Enabled',
    Transitions: [
      { Days: 90, StorageClass: 'STANDARD_IA' },
      { Days: 365, StorageClass: 'GLACIER' },
    ],
    Expiration: { Days: 2555 }, // delete after 7 years
  }],
}

// S3 Event Notifications
// Upload → Lambda → process → store result
{
  LambdaFunctionConfigurations: [{
    LambdaFunctionArn: processUploadLambdaArn,
    Events: ['s3:ObjectCreated:*'],
    Filter: { Key: { FilterRules: [{ Name: 'prefix', Value: 'uploads/' }] } },
  }],
}

// Versioning: Enable for audit trails, accidental delete protection
// Encryption: SSE-S3 (default), SSE-KMS (custom key), SSE-C (client key)
// CORS: Configure for browser direct uploads
```

---

### Q7: DynamoDB Design Patterns.

**Answer:**

```
DynamoDB Core Concepts:
- Primary Key: Partition Key (PK) or PK + Sort Key (SK)
- GSI: Global Secondary Index (different PK+SK, eventually consistent)
- LSI: Local Secondary Index (same PK, different SK, strongly consistent)
- Capacity: On-Demand (pay per request) or Provisioned (set RCU/WCU)

Single Table Design:
═══════════════════

PK              SK                    Attributes
────────────────────────────────────────────────────
USER#123        PROFILE               name, email, role
USER#123        TRADE#2024-01-15#001  symbol:SOL, amount:100
USER#123        TRADE#2024-01-16#002  symbol:ETH, amount:50
USER#123        BALANCE#SOL           available:1000, locked:100
SYMBOL#SOL      METADATA              name, decimals, ...
SYMBOL#SOL      PRICE#LATEST          price:95.50, volume:...

GSI1:
GSI1PK          GSI1SK
SYMBOL#SOL      TRADE#2024-01-15      → find all SOL trades by date
STATUS#active   TRADE#2024-01-15      → find active trades by date

Access Patterns:
1. Get user profile     → PK=USER#123, SK=PROFILE
2. Get user trades      → PK=USER#123, SK begins_with TRADE#
3. Get trades by date   → PK=USER#123, SK between TRADE#2024-01 and TRADE#2024-02
4. Get trades by symbol → GSI1: PK=SYMBOL#SOL, SK begins_with TRADE#
5. Get user balance     → PK=USER#123, SK=BALANCE#SOL
```

```typescript
// Query examples
// 1. Get user's recent trades
const result = await dynamodb.query({
  TableName: 'TradingApp',
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': 'USER#123',
    ':sk': 'TRADE#',
  },
  ScanIndexForward: false, // newest first
  Limit: 20,
});

// 2. Conditional write (optimistic locking)
await dynamodb.update({
  TableName: 'TradingApp',
  Key: { PK: 'USER#123', SK: 'BALANCE#SOL' },
  UpdateExpression: 'SET available = available - :amount',
  ConditionExpression: 'available >= :amount', // fails if insufficient
  ExpressionAttributeValues: { ':amount': 100 },
});

// 3. Transactions (ACID across items)
await dynamodb.transactWrite({
  TransactItems: [
    {
      Update: {
        TableName: 'TradingApp',
        Key: { PK: 'USER#123', SK: 'BALANCE#USD' },
        UpdateExpression: 'SET available = available - :cost',
        ConditionExpression: 'available >= :cost',
        ExpressionAttributeValues: { ':cost': 9500 },
      },
    },
    {
      Put: {
        TableName: 'TradingApp',
        Item: {
          PK: 'USER#123',
          SK: `TRADE#${Date.now()}`,
          symbol: 'SOL',
          amount: 100,
          price: 95,
        },
      },
    },
  ],
});

// DynamoDB Streams → Lambda (CDC - Change Data Capture)
// Use for: sync to Elasticsearch, trigger notifications, audit log
```

---

## Networking & Security

### Q8: VPC, Subnets, Security Groups.

**Answer:**

```
VPC Architecture:
═══════════════════════════════════════════════

┌─── VPC (10.0.0.0/16) ──────────────────────────────┐
│                                                      │
│  ┌─── Public Subnet (10.0.1.0/24) ─────────────┐   │
│  │  ┌──────┐  ┌──────┐  ┌──────────────────┐   │   │
│  │  │ ALB  │  │ NAT  │  │ Bastion Host     │   │   │
│  │  │      │  │ GW   │  │ (jump server)    │   │   │
│  │  └──────┘  └──────┘  └──────────────────┘   │   │
│  │  Route: 0.0.0.0/0 → Internet Gateway        │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─── Private Subnet (10.0.2.0/24) ────────────┐   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │  │ ECS Task │  │ ECS Task │  │ ECS Task │   │   │
│  │  │ (API)    │  │ (API)    │  │ (Worker) │   │   │
│  │  └──────────┘  └──────────┘  └──────────┘   │   │
│  │  Route: 0.0.0.0/0 → NAT Gateway             │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─── Private Subnet (10.0.3.0/24) ────────────┐   │
│  │  ┌──────────┐  ┌──────────┐                  │   │
│  │  │ RDS      │  │ Redis    │                  │   │
│  │  │ Primary  │  │ Cluster  │                  │   │
│  │  └──────────┘  └──────────┘                  │   │
│  │  No internet access                           │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘

Security Groups (stateful firewall):
─────────────────────────────────────
ALB SG:         Inbound: 80, 443 from 0.0.0.0/0
API SG:         Inbound: 3000 from ALB SG only
DB SG:          Inbound: 5432 from API SG only
Redis SG:       Inbound: 6379 from API SG only

NACLs (stateless, subnet-level):
Additional layer, usually keep default (allow all)
```

---

### Q9: IAM - Policies and Least Privilege.

**Answer:**

```json
// Least privilege: Only grant permissions that are needed

// Example: ECS task role for trade-api service
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456:trade-*"
    },
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456:trade-events"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::trade-documents/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456:secret:trade-api/*"
    }
  ]
}

// IAM best practices:
// 1. Use roles, not access keys (for services)
// 2. Use least privilege (only needed permissions)
// 3. Use conditions (restrict by IP, time, MFA)
// 4. Use resource-level permissions (specific ARNs, not *)
// 5. Rotate credentials regularly
// 6. Use AWS Organizations SCPs for account-level restrictions
```

---

## Architecture Patterns

### Q10: Well-Architected Framework - 6 Pillars.

**Answer:**

```
1. OPERATIONAL EXCELLENCE
   - Automate deployments (CI/CD)
   - Monitor with CloudWatch, New Relic
   - Runbooks for incidents
   Your example: GitHub Actions CI/CD, Prometheus monitoring

2. SECURITY
   - IAM least privilege
   - Encrypt at rest and in transit
   - Security groups, VPC
   Your example: VPC isolation, secrets management

3. RELIABILITY
   - Multi-AZ deployments
   - Auto-scaling
   - DLQ for message failures
   Your example: SNS+SQS with DLQ, ECS auto-scaling

4. PERFORMANCE EFFICIENCY
   - Right-sizing resources
   - Caching (Redis, CloudFront)
   - Use serverless where appropriate
   Your example: Redis caching, CloudFront CDN

5. COST OPTIMIZATION
   - Reserved instances for steady workloads
   - Spot instances for batch processing
   - S3 lifecycle policies
   - Right-sizing and monitoring costs

6. SUSTAINABILITY
   - Efficient resource usage
   - Minimize data transfer
   - Use managed services
```

---

### Q11: Multi-Region and Disaster Recovery.

**Answer:**

```
DR Strategies (increasing cost and speed):
──────────────────────────────────────────

1. BACKUP & RESTORE (RPO: hours, RTO: hours)
   - S3 cross-region replication
   - Automated DB snapshots
   - Cheapest, slowest recovery

2. PILOT LIGHT (RPO: minutes, RTO: minutes)
   - Core infrastructure running (DB replicas)
   - Compute off, turn on when needed
   - Moderate cost

3. WARM STANDBY (RPO: seconds, RTO: seconds)
   - Scaled-down version running in DR region
   - Scale up on failover
   - Higher cost

4. ACTIVE-ACTIVE (RPO: ~0, RTO: ~0)
   - Full stack in multiple regions
   - Route 53 latency-based routing
   - Most expensive, fastest recovery

RPO = Recovery Point Objective (how much data can you lose?)
RTO = Recovery Time Objective (how fast must you recover?)

For trading platform:
- Use Active-Active or Warm Standby (financial data is critical)
- RDS Multi-AZ for automatic failover
- S3 cross-region replication for documents
- DynamoDB global tables for multi-region
- Route 53 health checks for automatic DNS failover
```

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [System Design](./05-system-design.md) | **Next**: [DevOps](./07-devops.md)
