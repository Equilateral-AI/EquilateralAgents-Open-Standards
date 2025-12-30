# Operational Excellence - AWS Well-Architected

## Overview

Operational Excellence focuses on running and monitoring systems to deliver business value and continually improving processes and procedures.

**AWS Reference**: https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/

---

## Design Principles

### 1. Perform Operations as Code

**Standard**: All infrastructure MUST be defined as code (IaC).

**Implementation**:
```yaml
# SAM template example
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: nodejs20.x
    Architectures: [arm64]
    Tracing: Active
    Environment:
      Variables:
        LOG_LEVEL: INFO
```

**Equilateral Pattern**: `serverless-saas-aws/sam_deployment_standards.md`

---

### 2. Make Frequent, Small, Reversible Changes

**Standard**: Deploy small incremental changes with rollback capability.

**Practices**:
- Feature flags for gradual rollouts
- Blue/green deployments via CodeDeploy
- Database migrations with rollback scripts
- Version all API endpoints

**Equilateral Pattern**: `serverless-saas-aws/deployment_standards.md`

---

### 3. Refine Operations Procedures Frequently

**Standard**: Maintain runbooks for common operational tasks.

**Required Runbooks**:
```markdown
## Runbook Categories

1. **Deployment Runbooks**
   - Standard deployment procedure
   - Rollback procedure
   - Database migration procedure

2. **Incident Response**
   - High error rate response
   - Database connection issues
   - Third-party integration failures

3. **Maintenance**
   - Certificate rotation
   - Secret rotation
   - Log retention cleanup
```

---

### 4. Anticipate Failure

**Standard**: Design for failure and test failure scenarios.

**Implementation**:
```javascript
// Graceful degradation pattern
async function getDataWithFallback(primarySource, fallbackSource) {
  try {
    return await primarySource.getData();
  } catch (error) {
    console.warn('Primary source failed, using fallback', { error: error.message });
    return await fallbackSource.getData();
  }
}
```

---

### 5. Learn from All Operational Failures

**Standard**: Conduct post-incident reviews and implement improvements.

**Post-Incident Template**:
```markdown
## Incident Report: [TITLE]

### Timeline
- Detection: [TIME]
- Response: [TIME]
- Mitigation: [TIME]
- Resolution: [TIME]

### Impact
- Duration: [X minutes]
- Users affected: [COUNT]
- Revenue impact: [ESTIMATE]

### Root Cause
[Description]

### Action Items
- [ ] Immediate fix
- [ ] Monitoring improvement
- [ ] Process improvement
```

---

## Observability Standards

### Structured Logging

**Standard**: All logs MUST be structured JSON.

```javascript
// REQUIRED log format
const log = {
  timestamp: new Date().toISOString(),
  level: 'INFO',
  requestId: context.awsRequestId,
  service: 'payment-service',
  action: 'processPayment',
  userId: event.userId,
  duration: 234,
  success: true
};

console.log(JSON.stringify(log));
```

**Log Levels**:
- `ERROR`: Requires immediate attention
- `WARN`: Degraded but functional
- `INFO`: Business events
- `DEBUG`: Development only

### CloudWatch Metrics

**Standard**: Custom metrics for business KPIs.

```javascript
const { CloudWatch } = require('@aws-sdk/client-cloudwatch');
const cloudwatch = new CloudWatch();

async function publishMetric(name, value, unit = 'Count') {
  await cloudwatch.putMetricData({
    Namespace: 'EquilateralApp',
    MetricData: [{
      MetricName: name,
      Value: value,
      Unit: unit,
      Dimensions: [
        { Name: 'Environment', Value: process.env.ENVIRONMENT }
      ]
    }]
  });
}
```

**Required Metrics**:
- `OrdersProcessed` - Business throughput
- `PaymentSuccess` / `PaymentFailure` - Conversion
- `ApiLatency` - Performance
- `ErrorCount` - Reliability

### X-Ray Tracing

**Standard**: Enable X-Ray for all Lambda functions and API Gateway.

```yaml
# SAM template
Globals:
  Function:
    Tracing: Active
  Api:
    TracingEnabled: true
```

```javascript
// Custom subsegments
const AWSXRay = require('aws-xray-sdk');

async function processOrder(order) {
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment('processOrder');

  try {
    // Process order
    subsegment.addAnnotation('orderId', order.id);
    return result;
  } finally {
    subsegment.close();
  }
}
```

---

## Alerting Standards

### Alarm Configuration

**Standard**: Alarms for all critical metrics with appropriate thresholds.

```yaml
# CloudWatch Alarm
HighErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub '${AWS::StackName}-high-error-rate'
    MetricName: Errors
    Namespace: AWS/Lambda
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref AlertSNSTopic
```

**Alarm Categories**:

| Category | Threshold | Action |
|----------|-----------|--------|
| Error Rate | >1% | Alert + Investigate |
| Latency P99 | >3s | Alert + Scale |
| DLQ Messages | >0 | Alert + Process |
| Concurrent Executions | >80% limit | Alert + Request increase |

---

## Deployment Standards

### CI/CD Pipeline

**Standard**: Automated pipeline with quality gates.

```yaml
# Required pipeline stages
stages:
  - lint          # ESLint, security scan
  - test          # Unit + integration tests
  - build         # SAM build
  - deploy-dev    # Auto-deploy to dev
  - integration   # E2E tests
  - deploy-prod   # Manual approval + deploy
```

### Deployment Verification

**Standard**: Verify deployments with smoke tests.

```javascript
// Post-deployment verification
async function verifyDeployment() {
  const checks = [
    checkApiHealth(),
    checkDatabaseConnectivity(),
    checkExternalIntegrations()
  ];

  const results = await Promise.all(checks);

  if (results.some(r => !r.healthy)) {
    await rollback();
    throw new Error('Deployment verification failed');
  }
}
```

---

## Documentation Standards

### API Documentation

**Standard**: OpenAPI spec for all APIs.

```yaml
openapi: 3.0.0
info:
  title: Payment API
  version: 1.0.0
paths:
  /payments:
    post:
      summary: Process payment
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PaymentRequest'
```

### Architecture Decision Records (ADRs)

**Standard**: Document significant decisions.

```markdown
# ADR-001: Use PostgreSQL over DynamoDB

## Status
Accepted

## Context
Need to store relational data with complex queries.

## Decision
Use RDS PostgreSQL.

## Consequences
- Pro: Rich query capabilities
- Con: Higher operational overhead
```

---

## Equilateral Cross-References

| Topic | Standard Document |
|-------|------------------|
| SAM Deployment | `serverless-saas-aws/sam_deployment_standards.md` |
| Lambda Patterns | `backend/tim_combo_lambda_pattern.md` |
| Database Operations | `database/migration_deployment_order.md` |
| CloudFormation | `deployment/cloudformation_resource_management.md` |
| Agent Monitoring | `multi-agent-orchestration/agent_orchestration_standards.md` |
