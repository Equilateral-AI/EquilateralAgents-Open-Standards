# Reliability - AWS Well-Architected

## Overview

Reliability focuses on the ability of a workload to perform its intended function correctly and consistently.

**AWS Reference**: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/

---

## Design Principles

### 1. Automatically Recover from Failure

**Standard**: Design for self-healing.

**Lambda Retry Configuration**:
```yaml
ProcessOrderFunction:
  Type: AWS::Serverless::Function
  Properties:
    EventInvokeConfig:
      MaximumRetryAttempts: 2
      DestinationConfig:
        OnFailure:
          Destination: !GetAtt DeadLetterQueue.Arn
```

**Circuit Breaker Pattern**:
```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 30000;
    this.state = 'CLOSED';
    this.failures = 0;
    this.lastFailure = null;
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailure > this.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }
}

// Usage
const breaker = new CircuitBreaker({ failureThreshold: 3 });
const result = await breaker.call(() => externalApi.getData());
```

---

### 2. Test Recovery Procedures

**Standard**: Regular failure testing.

**Testing Categories**:
```markdown
## Failure Testing

### Chaos Engineering
- Lambda cold starts under load
- Database failover
- API timeout scenarios

### Game Days
- Simulate production incidents
- Test incident response
- Validate runbooks

### Disaster Recovery
- Regular RDS snapshot restoration
- Cross-region failover
- Data recovery procedures
```

---

### 3. Scale Horizontally

**Standard**: Prefer horizontal over vertical scaling.

**Lambda Concurrency**:
```yaml
# Reserved concurrency for critical functions
ProcessPaymentFunction:
  Type: AWS::Serverless::Function
  Properties:
    ReservedConcurrentExecutions: 100

# Provisioned concurrency for consistent latency
Alias:
  Type: AWS::Lambda::Alias
  Properties:
    FunctionName: !Ref ProcessPaymentFunction
    FunctionVersion: !GetAtt ProcessPaymentFunction.Version
    ProvisionedConcurrencyConfig:
      ProvisionedConcurrentExecutions: 10
```

---

### 4. Stop Guessing Capacity

**Standard**: Use auto-scaling.

**RDS Auto Scaling**:
```yaml
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    MaxAllocatedStorage: 1000  # Auto-scale storage

ReadReplica:
  Type: AWS::RDS::DBInstance
  Properties:
    SourceDBInstanceIdentifier: !Ref Database
```

---

### 5. Manage Change Through Automation

**Standard**: Automated deployments with rollback.

**CodeDeploy Configuration**:
```yaml
DeploymentPreference:
  Type: Linear10PercentEvery1Minute
  Alarms:
    - !Ref ErrorRateAlarm
    - !Ref LatencyAlarm
  Hooks:
    PreTraffic: !Ref PreTrafficHook
    PostTraffic: !Ref PostTrafficHook
```

---

## Fault Isolation

### Multi-AZ Deployment

**Standard**: Deploy across multiple Availability Zones.

```yaml
# RDS Multi-AZ
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    MultiAZ: true
    DBSubnetGroupName: !Ref DatabaseSubnetGroup

DatabaseSubnetGroup:
  Type: AWS::RDS::DBSubnetGroup
  Properties:
    SubnetIds:
      - !Ref PrivateSubnet1  # AZ-1
      - !Ref PrivateSubnet2  # AZ-2
```

### Lambda VPC Configuration

**Standard**: Multiple subnets for Lambda.

```yaml
ProcessFunction:
  Type: AWS::Serverless::Function
  Properties:
    VpcConfig:
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref LambdaSecurityGroup
```

---

## Error Handling

### Retry with Exponential Backoff

**Standard**: Implement intelligent retries.

```javascript
async function retryWithBackoff(fn, maxRetries = 3) {
  let lastError;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Don't retry non-transient errors
      if (!isRetryable(error)) {
        throw error;
      }

      // Exponential backoff with jitter
      const delay = Math.min(
        1000 * Math.pow(2, attempt) + Math.random() * 1000,
        30000
      );

      console.warn(`Attempt ${attempt + 1} failed, retrying in ${delay}ms`, {
        error: error.message
      });

      await sleep(delay);
    }
  }

  throw lastError;
}

function isRetryable(error) {
  const retryableCodes = [
    'ECONNRESET',
    'ETIMEDOUT',
    'ENOTFOUND',
    'ThrottlingException',
    'ProvisionedThroughputExceededException'
  ];
  return retryableCodes.includes(error.code);
}
```

### Dead Letter Queues

**Standard**: Capture failed messages for analysis.

```yaml
ProcessQueue:
  Type: AWS::SQS::Queue
  Properties:
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn
      maxReceiveCount: 3

DeadLetterQueue:
  Type: AWS::SQS::Queue
  Properties:
    MessageRetentionPeriod: 1209600  # 14 days

# Alarm on DLQ messages
DLQAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Dimensions:
      - Name: QueueName
        Value: !GetAtt DeadLetterQueue.QueueName
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
```

---

## Database Reliability

### Connection Management

**Standard**: Handle connections for serverless.

```javascript
// Singleton connection for Lambda
let client = null;

async function getClient() {
  if (!client) {
    client = new Client({
      host: process.env.DB_HOST,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      connectionTimeoutMillis: 5000,
      query_timeout: 30000
    });
    await client.connect();
  }
  return client;
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  if (client) {
    await client.end();
  }
});
```

**Equilateral Pattern**: `serverless-saas-aws/lambda_database_standards.md`

### Read Replicas

**Standard**: Use read replicas for read-heavy workloads.

```javascript
// Route queries appropriately
const writeClient = createClient(process.env.DB_WRITER_HOST);
const readClient = createClient(process.env.DB_READER_HOST);

async function getOrder(orderId) {
  return readClient.query('SELECT * FROM orders WHERE id = $1', [orderId]);
}

async function createOrder(order) {
  return writeClient.query(
    'INSERT INTO orders (id, data) VALUES ($1, $2)',
    [order.id, order.data]
  );
}
```

---

## Graceful Degradation

### Feature Fallbacks

**Standard**: Degrade gracefully when dependencies fail.

```javascript
async function getProductRecommendations(userId) {
  try {
    // Try ML-based recommendations
    return await mlService.getRecommendations(userId);
  } catch (error) {
    console.warn('ML service unavailable, using fallback', { error: error.message });

    // Fall back to popular products
    return await getPopularProducts();
  }
}
```

### Bulkhead Pattern

**Standard**: Isolate failures.

```javascript
// Separate connection pools per service
const pools = {
  orders: new Pool({ max: 10 }),
  inventory: new Pool({ max: 5 }),
  analytics: new Pool({ max: 3 })
};

// Failure in analytics doesn't affect orders
async function processOrder(order) {
  const orderResult = await pools.orders.query(/*...*/);

  // Analytics failure is non-critical
  try {
    await pools.analytics.query(/*...*/);
  } catch (error) {
    console.warn('Analytics update failed', { error: error.message });
    // Continue processing
  }

  return orderResult;
}
```

---

## Monitoring for Reliability

### Key Metrics

**Standard**: Monitor availability indicators.

```yaml
# Error rate alarm
ErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    MetricName: Errors
    Namespace: AWS/Lambda
    Statistic: Average
    Period: 60
    EvaluationPeriods: 5
    Threshold: 0.01  # 1% error rate
    ComparisonOperator: GreaterThanThreshold

# Latency P99 alarm
LatencyAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    MetricName: Duration
    Namespace: AWS/Lambda
    ExtendedStatistic: p99
    Period: 60
    EvaluationPeriods: 3
    Threshold: 3000  # 3 seconds
```

### Health Checks

**Standard**: Implement comprehensive health checks.

```javascript
async function healthCheck() {
  const checks = {
    database: await checkDatabase(),
    cache: await checkCache(),
    externalApi: await checkExternalApi()
  };

  const healthy = Object.values(checks).every(c => c.healthy);

  return {
    statusCode: healthy ? 200 : 503,
    body: JSON.stringify({
      status: healthy ? 'healthy' : 'degraded',
      checks,
      timestamp: new Date().toISOString()
    })
  };
}

async function checkDatabase() {
  try {
    await client.query('SELECT 1');
    return { healthy: true };
  } catch (error) {
    return { healthy: false, error: error.message };
  }
}
```

---

## Backup and Recovery

### Automated Backups

**Standard**: Automated backup with tested recovery.

```yaml
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    BackupRetentionPeriod: 30
    PreferredBackupWindow: '03:00-04:00'
    DeleteAutomatedBackups: false
    DeletionProtection: true
```

### Point-in-Time Recovery

**Standard**: Enable PITR for critical tables.

```yaml
OrdersTable:
  Type: AWS::DynamoDB::Table
  Properties:
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
```

---

## Equilateral Cross-References

| Topic | Standard Document |
|-------|------------------|
| Database Patterns | `serverless-saas-aws/lambda_database_standards.md` |
| Deployment | `serverless-saas-aws/deployment_standards.md` |
| Agent Reliability | `multi-agent-orchestration/agent_orchestration_standards.md` |
| Error Recovery | `multi-agent-orchestration/agent_communication_protocols.md` |
| WebSocket Reliability | `real-time-systems/websocket_postgres_pattern.md` |
