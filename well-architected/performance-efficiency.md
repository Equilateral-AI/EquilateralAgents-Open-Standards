# Performance Efficiency - AWS Well-Architected

## Overview

Performance Efficiency focuses on using computing resources efficiently to meet system requirements and maintaining that efficiency as demand changes.

**AWS Reference**: https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/

---

## Design Principles

### 1. Democratize Advanced Technologies

**Standard**: Use managed services over custom implementations.

**Preferred Services**:
| Need | Use | Not |
|------|-----|-----|
| Search | OpenSearch Service | Self-managed Elasticsearch |
| ML | SageMaker, Bedrock | Self-managed models |
| Caching | ElastiCache | Self-managed Redis |
| Queuing | SQS | Self-managed RabbitMQ |

---

### 2. Go Global in Minutes

**Standard**: Deploy close to users.

```yaml
# CloudFront for global distribution
Distribution:
  Type: AWS::CloudFront::Distribution
  Properties:
    DistributionConfig:
      Origins:
        - DomainName: !GetAtt ApiBucket.RegionalDomainName
          Id: S3Origin
      DefaultCacheBehavior:
        ViewerProtocolPolicy: redirect-to-https
        CachePolicyId: !Ref CachePolicy
      PriceClass: PriceClass_100  # Edge locations in most regions
```

---

### 3. Use Serverless Architectures

**Standard**: Serverless first for variable workloads.

**Benefits Realized**:
- No capacity management
- Automatic scaling
- Pay-per-use
- Built-in high availability

---

### 4. Experiment More Often

**Standard**: Test different configurations.

```javascript
// A/B test different memory configurations
const configs = [
  { memory: 512, name: 'baseline' },
  { memory: 1024, name: 'double' },
  { memory: 2048, name: 'quad' }
];

// Use Lambda Power Tuning tool
// https://github.com/alexcasalboni/aws-lambda-power-tuning
```

---

### 5. Consider Mechanical Sympathy

**Standard**: Match technology to workload.

| Workload | Best Fit |
|----------|----------|
| Compute-intensive | Higher memory Lambda (more vCPU) |
| I/O-intensive | Async patterns, concurrent requests |
| Memory-intensive | Provisioned concurrency |
| Latency-sensitive | Edge locations, caching |

---

## Lambda Optimization

### Memory Configuration

**Standard**: Right-size Lambda memory.

```yaml
# Memory affects CPU allocation
# 1769 MB = 1 vCPU equivalent

# I/O bound - lower memory sufficient
ReadFunction:
  Type: AWS::Serverless::Function
  Properties:
    MemorySize: 512
    Timeout: 30

# CPU bound - higher memory for more CPU
ProcessFunction:
  Type: AWS::Serverless::Function
  Properties:
    MemorySize: 1769  # 1 vCPU
    Timeout: 60
```

### ARM64 Architecture

**Standard**: Use Graviton2 for 20% better price/performance.

```yaml
Globals:
  Function:
    Architectures:
      - arm64  # Graviton2
```

**Note**: Verify all dependencies support ARM64.

### Cold Start Optimization

**Standard**: Minimize cold start impact.

```javascript
// Initialize outside handler
const { Client } = require('pg');
let client;

// Lazy initialization
async function getClient() {
  if (!client) {
    client = new Client(/* config */);
    await client.connect();
  }
  return client;
}

exports.handler = async (event) => {
  const db = await getClient();
  // Use warm connection
};
```

**Provisioned Concurrency** for consistent latency:
```yaml
Alias:
  Type: AWS::Lambda::Alias
  Properties:
    ProvisionedConcurrencyConfig:
      ProvisionedConcurrentExecutions: 5
```

---

## Caching Strategies

### API Gateway Caching

**Standard**: Cache GET responses.

```yaml
Api:
  Type: AWS::Serverless::Api
  Properties:
    CacheClusterEnabled: true
    CacheClusterSize: '0.5'
    MethodSettings:
      - ResourcePath: /products
        HttpMethod: GET
        CachingEnabled: true
        CacheTtlInSeconds: 300
```

### Application-Level Caching

**Standard**: Cache expensive computations.

```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 300 });

async function getProductWithCache(productId) {
  const cached = cache.get(productId);
  if (cached) {
    return cached;
  }

  const product = await db.query(
    'SELECT * FROM products WHERE id = $1',
    [productId]
  );

  cache.set(productId, product);
  return product;
}
```

### ElastiCache for Shared Cache

**Standard**: Use ElastiCache for multi-Lambda caching.

```javascript
const Redis = require('ioredis');
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379
});

async function getCachedData(key) {
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }

  const data = await fetchData();
  await redis.setex(key, 300, JSON.stringify(data));
  return data;
}
```

---

## Database Performance

### Query Optimization

**Standard**: Optimize all database queries.

```javascript
// GOOD - specific columns, indexed query
const result = await client.query(`
  SELECT id, name, price
  FROM products
  WHERE category_id = $1
  AND active = true
  ORDER BY created_at DESC
  LIMIT 20
`, [categoryId]);

// BAD - SELECT *, no limit
// const result = await client.query(`
//   SELECT * FROM products WHERE category_id = ${categoryId}
// `);
```

### Connection Pooling

**Standard**: Reuse connections efficiently.

```javascript
// For Lambda - single connection, reused across invocations
let client;

async function getConnection() {
  if (!client) {
    client = new Client({
      connectionTimeoutMillis: 5000,
      idle_in_transaction_session_timeout: 30000
    });
    await client.connect();
  }
  return client;
}
```

**Equilateral Pattern**: `serverless-saas-aws/lambda_database_standards.md`

### Read Replicas

**Standard**: Route read traffic to replicas.

```javascript
const writerEndpoint = process.env.DB_WRITER;
const readerEndpoint = process.env.DB_READER;

// Write operations
async function createOrder(order) {
  const writer = await getConnection(writerEndpoint);
  return writer.query('INSERT INTO orders...');
}

// Read operations
async function getOrders(userId) {
  const reader = await getConnection(readerEndpoint);
  return reader.query('SELECT * FROM orders WHERE user_id = $1', [userId]);
}
```

---

## Async Patterns

### Event-Driven Processing

**Standard**: Decouple with events for better throughput.

```yaml
# SQS for async processing
OrderQueue:
  Type: AWS::SQS::Queue
  Properties:
    VisibilityTimeout: 300

ProcessOrderFunction:
  Type: AWS::Serverless::Function
  Properties:
    Events:
      SQS:
        Type: SQS
        Properties:
          Queue: !GetAtt OrderQueue.Arn
          BatchSize: 10
```

### Batch Processing

**Standard**: Process in batches for efficiency.

```javascript
// Process SQS batch
exports.handler = async (event) => {
  const results = await Promise.all(
    event.Records.map(record => processRecord(record))
  );

  // Return failures for retry
  const failures = results
    .filter(r => !r.success)
    .map(r => ({ itemIdentifier: r.messageId }));

  return { batchItemFailures: failures };
};
```

### Parallel Execution

**Standard**: Parallelize independent operations.

```javascript
// GOOD - parallel
const [user, orders, recommendations] = await Promise.all([
  getUser(userId),
  getOrders(userId),
  getRecommendations(userId)
]);

// BAD - sequential
// const user = await getUser(userId);
// const orders = await getOrders(userId);
// const recommendations = await getRecommendations(userId);
```

---

## API Performance

### Compression

**Standard**: Compress API responses.

```yaml
Api:
  Type: AWS::Serverless::Api
  Properties:
    MinimumCompressionSize: 1024  # Compress > 1KB
```

### Pagination

**Standard**: Paginate all list endpoints.

```javascript
async function listOrders(userId, cursor, limit = 20) {
  const query = `
    SELECT id, created_at, total
    FROM orders
    WHERE user_id = $1
      AND id > $2
    ORDER BY id
    LIMIT $3
  `;

  const orders = await client.query(query, [userId, cursor || 0, limit + 1]);

  const hasMore = orders.length > limit;
  const items = orders.slice(0, limit);
  const nextCursor = hasMore ? items[items.length - 1].id : null;

  return { items, nextCursor };
}
```

### Response Optimization

**Standard**: Return only necessary data.

```javascript
// Client specifies fields
const fields = event.queryStringParameters?.fields?.split(',');

const order = await getOrder(orderId);

// Return only requested fields
const response = fields
  ? pick(order, fields)
  : order;
```

---

## Monitoring Performance

### Key Metrics

**Standard**: Monitor performance indicators.

```yaml
# P99 Latency Alarm
LatencyAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    MetricName: Duration
    Namespace: AWS/Lambda
    ExtendedStatistic: p99
    Period: 60
    Threshold: 1000  # 1 second
    ComparisonOperator: GreaterThanThreshold
```

**Dashboard Metrics**:
- P50, P90, P99 latency
- Cold start percentage
- Concurrent executions
- Database query time
- Cache hit ratio

### X-Ray Analysis

**Standard**: Use X-Ray for bottleneck identification.

```javascript
const AWSXRay = require('aws-xray-sdk');

async function handler(event) {
  // Automatic subsegments for AWS SDK calls
  const segment = AWSXRay.getSegment();

  // Custom subsegment for business logic
  const subsegment = segment.addNewSubsegment('processOrder');
  try {
    const result = await processOrder(event);
    return result;
  } finally {
    subsegment.close();
  }
}
```

---

## Equilateral Cross-References

| Topic | Standard Document |
|-------|------------------|
| Lambda Database | `serverless-saas-aws/lambda_database_standards.md` |
| Backend Handlers | `backend/tim_combo_lambda_pattern.md` |
| API Standards | `serverless-saas-aws/api_standards.md` |
| WebSocket Performance | `real-time-systems/websocket_postgres_pattern.md` |
| Frontend Performance | `frontend-development/frontend_standards.md` |
