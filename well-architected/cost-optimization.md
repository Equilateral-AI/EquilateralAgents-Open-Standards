# Cost Optimization - AWS Well-Architected

## Overview

Cost Optimization focuses on avoiding unnecessary costs and understanding where money is being spent.

**AWS Reference**: https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/

---

## Design Principles

### 1. Implement Cloud Financial Management

**Standard**: Track and allocate costs.

**Tagging Strategy**:
```yaml
Tags:
  Project: !Ref ProjectName
  Environment: !Ref Environment
  Owner: !Ref OwnerEmail
  CostCenter: !Ref CostCenter
```

**Required Tags**:
- `Project` - Application/service name
- `Environment` - dev/staging/prod
- `Owner` - Team or individual
- `CostCenter` - Billing allocation

---

### 2. Adopt a Consumption Model

**Standard**: Pay only for what you use.

**Serverless First**:
```yaml
# Lambda - pay per invocation
# API Gateway - pay per request
# DynamoDB On-Demand - pay per operation
# S3 - pay per storage and request
```

**Avoid Fixed Costs in Dev/Test**:
```yaml
# DEV - use smaller/on-demand
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: !If [IsProd, db.r6g.large, db.t4g.micro]
    MultiAZ: !If [IsProd, true, false]
```

---

### 3. Measure Overall Efficiency

**Standard**: Track cost per transaction.

```javascript
// Custom metric for cost efficiency
await cloudwatch.putMetricData({
  Namespace: 'BusinessMetrics',
  MetricData: [{
    MetricName: 'CostPerOrder',
    Value: totalCost / orderCount,
    Unit: 'None'
  }]
});
```

---

### 4. Stop Spending on Undifferentiated Heavy Lifting

**Standard**: Use managed services.

| Build | Buy (Managed) |
|-------|---------------|
| Custom auth | Cognito |
| Custom search | OpenSearch |
| Custom monitoring | CloudWatch |
| Custom deployment | CodePipeline |

---

### 5. Analyze and Attribute Expenditure

**Standard**: Understand cost drivers.

**Cost Analysis Tools**:
- AWS Cost Explorer
- AWS Budgets
- Cost Allocation Tags
- Resource-level cost data

---

## Lambda Cost Optimization

### Right-Size Memory

**Standard**: Use Lambda Power Tuning.

```bash
# Deploy power tuning tool
# Test function with different memory configurations
# Find optimal price/performance balance
```

**Memory/Duration Tradeoff**:
| Memory | Duration | Cost per 1M invocations |
|--------|----------|-------------------------|
| 512 MB | 2000 ms | $16.67 |
| 1024 MB | 1000 ms | $16.67 |
| 2048 MB | 500 ms | $16.67 |

Higher memory often reduces cost due to faster execution.

### ARM64 Architecture

**Standard**: Use Graviton2 for 20% savings.

```yaml
Globals:
  Function:
    Architectures:
      - arm64  # 20% cheaper than x86
```

### Avoid Runtime SSM Calls

**Standard**: NEVER fetch SSM parameters at runtime.

```yaml
# CORRECT - resolve at deploy time ($0)
Environment:
  Variables:
    DB_HOST: '{{resolve:ssm:/prod/db/host:1}}'

# WRONG - fetch at runtime ($25/month per million calls)
# const param = await ssm.getParameter({ Name: '/prod/db/host' });
```

**Equilateral Pattern**: `serverless-saas-aws/lambda_database_standards.md`

---

## Database Cost Optimization

### Instance Sizing

**Standard**: Environment-appropriate sizing.

```yaml
Mappings:
  EnvironmentConfig:
    dev:
      InstanceClass: db.t4g.micro
      AllocatedStorage: 20
      MultiAZ: false
    prod:
      InstanceClass: db.r6g.large
      AllocatedStorage: 100
      MultiAZ: true

Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: !FindInMap [EnvironmentConfig, !Ref Environment, InstanceClass]
```

### Reserved Instances

**Standard**: Reserved capacity for production.

- 1-year reserved: ~30% savings
- 3-year reserved: ~60% savings

Only commit for stable, production workloads.

### Storage Optimization

**Standard**: Clean up unused data.

```sql
-- Archive old data
INSERT INTO orders_archive
SELECT * FROM orders WHERE created_at < NOW() - INTERVAL '2 years';

DELETE FROM orders WHERE created_at < NOW() - INTERVAL '2 years';

-- VACUUM to reclaim space
VACUUM FULL orders;
```

---

## S3 Cost Optimization

### Storage Classes

**Standard**: Use appropriate storage class.

```yaml
# Lifecycle policy
BucketLifecycle:
  Type: AWS::S3::Bucket
  Properties:
    LifecycleConfiguration:
      Rules:
        - Id: TransitionToIA
          Status: Enabled
          Transitions:
            - StorageClass: STANDARD_IA
              TransitionInDays: 30
            - StorageClass: GLACIER
              TransitionInDays: 90
        - Id: ExpireOldVersions
          Status: Enabled
          NoncurrentVersionExpiration:
            NoncurrentDays: 30
```

### Intelligent Tiering

**Standard**: Use for unpredictable access patterns.

```yaml
IntelligentTieringBucket:
  Type: AWS::S3::Bucket
  Properties:
    IntelligentTieringConfigurations:
      - Id: EntireBucket
        Status: Enabled
        Tierings:
          - AccessTier: ARCHIVE_ACCESS
            Days: 90
```

---

## API Gateway Cost Optimization

### Caching

**Standard**: Cache responses to reduce Lambda invocations.

```yaml
Api:
  Type: AWS::Serverless::Api
  Properties:
    CacheClusterEnabled: true
    CacheClusterSize: '0.5'  # $0.02/hour vs Lambda invocations
```

### Request Validation

**Standard**: Validate at API Gateway to avoid Lambda costs.

```yaml
# Reject invalid requests before Lambda
RequestValidator:
  Type: AWS::ApiGateway::RequestValidator
  Properties:
    ValidateRequestBody: true
    ValidateRequestParameters: true
```

---

## CloudWatch Cost Optimization

### Log Retention

**Standard**: Set appropriate retention.

```yaml
LogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    RetentionInDays: 30  # Not infinite
```

| Environment | Retention |
|-------------|-----------|
| Dev | 7 days |
| Staging | 14 days |
| Prod | 30-90 days |

### Metric Filters vs Custom Metrics

**Standard**: Prefer metric filters.

```yaml
# FREE - metric filter from logs
ErrorMetricFilter:
  Type: AWS::Logs::MetricFilter
  Properties:
    FilterPattern: 'ERROR'
    MetricTransformations:
      - MetricName: ErrorCount
        MetricNamespace: App
        MetricValue: '1'

# COSTS - custom metric via SDK
# cloudwatch.putMetricData()
```

---

## Development Cost Optimization

### Dev/Test Resources

**Standard**: Smaller resources for non-prod.

```yaml
# Use conditions
Parameters:
  Environment:
    Type: String

Conditions:
  IsProd: !Equals [!Ref Environment, prod]

Resources:
  Database:
    Properties:
      DBInstanceClass: !If [IsProd, db.r6g.large, db.t4g.micro]
      MultiAZ: !If [IsProd, true, false]
      BackupRetentionPeriod: !If [IsProd, 30, 1]
```

### Scheduled Scaling

**Standard**: Scale down non-prod during off-hours.

```yaml
# Stop dev RDS at night
ScheduledStopAction:
  Type: AWS::Events::Rule
  Condition: IsDev
  Properties:
    ScheduleExpression: 'cron(0 22 ? * MON-FRI *)'
    Targets:
      - Id: StopDB
        Arn: !Sub 'arn:aws:rds:${AWS::Region}:${AWS::AccountId}:db:${Database}'
        Input: '{"action": "stop"}'
```

---

## Cost Monitoring

### AWS Budgets

**Standard**: Set budgets with alerts.

```yaml
MonthlyBudget:
  Type: AWS::Budgets::Budget
  Properties:
    Budget:
      BudgetName: !Sub '${ProjectName}-monthly'
      BudgetLimit:
        Amount: 100
        Unit: USD
      TimeUnit: MONTHLY
      BudgetType: COST
    NotificationsWithSubscribers:
      - Notification:
          NotificationType: ACTUAL
          ComparisonOperator: GREATER_THAN
          Threshold: 80
        Subscribers:
          - SubscriptionType: EMAIL
            Address: !Ref AlertEmail
```

### Cost Allocation

**Standard**: Tag resources for attribution.

```javascript
// Tag all resources created
const params = {
  ResourceArn: functionArn,
  Tags: {
    Project: 'OrderSystem',
    Feature: 'Payments',
    CostCenter: 'Engineering'
  }
};
```

---

## Cost Review Checklist

**Weekly**:
- [ ] Check Cost Explorer for anomalies
- [ ] Review running dev/test resources
- [ ] Verify auto-scaling configurations

**Monthly**:
- [ ] Review detailed cost breakdown
- [ ] Identify optimization opportunities
- [ ] Update cost forecasts

**Quarterly**:
- [ ] Reserved instance coverage analysis
- [ ] Architecture efficiency review
- [ ] Savings Plans evaluation

---

## Equilateral Cross-References

| Topic | Standard Document |
|-------|------------------|
| Lambda Database | `serverless-saas-aws/lambda_database_standards.md` |
| Cost Standards | `cost-optimization/cost_optimization_standards.md` |
| Deployment | `serverless-saas-aws/deployment_standards.md` |
| Development Principles | `serverless-saas-aws/development_principles.md` |
