# Sustainability - AWS Well-Architected

## Overview

Sustainability focuses on minimizing the environmental impacts of running cloud workloads.

**AWS Reference**: https://docs.aws.amazon.com/wellarchitected/latest/sustainability-pillar/

---

## Design Principles

### 1. Understand Your Impact

**Standard**: Measure and track carbon footprint.

**AWS Tools**:
- AWS Customer Carbon Footprint Tool
- CloudWatch metrics for efficiency

**Metrics to Track**:
- Compute hours
- Data transfer
- Storage utilization
- Regional carbon intensity

---

### 2. Establish Sustainability Goals

**Standard**: Set measurable targets.

**Example Goals**:
- Reduce compute waste by 20%
- Achieve 80% storage efficiency
- Minimize cross-region data transfer

---

### 3. Maximize Utilization

**Standard**: Use resources efficiently.

```yaml
# RIGHT-SIZE - use what you need
Function:
  Type: AWS::Serverless::Function
  Properties:
    MemorySize: 512  # Tested optimal, not default 128 or max 10240
    Timeout: 10      # Actual need, not max 900
```

**Utilization Targets**:
| Resource | Target Utilization |
|----------|-------------------|
| Lambda memory | >70% used |
| RDS CPU | 60-80% average |
| S3 storage | Lifecycle managed |
| EC2 (if used) | >80% CPU average |

---

### 4. Anticipate and Adopt More Efficient Offerings

**Standard**: Use latest, most efficient technologies.

**Efficient Choices**:
```yaml
# Graviton2 (ARM64) - 20% more energy efficient
Architectures:
  - arm64

# Latest runtime - better performance/watt
Runtime: nodejs20.x  # Not nodejs16.x

# Managed services - AWS optimizes infrastructure
```

---

### 5. Use Managed Services

**Standard**: Leverage AWS optimization.

| Self-Managed | Managed (More Efficient) |
|--------------|-------------------------|
| EC2 + containers | Lambda |
| Self-managed DB | RDS, Aurora Serverless |
| Self-managed cache | ElastiCache |
| Self-managed search | OpenSearch Serverless |

---

### 6. Reduce Downstream Impact

**Standard**: Minimize client-side processing.

```javascript
// Compress responses
const zlib = require('zlib');

function compressResponse(data) {
  return zlib.gzipSync(JSON.stringify(data));
}

// Return only needed fields
function selectFields(data, requestedFields) {
  return requestedFields
    ? pick(data, requestedFields)
    : data;
}
```

---

## Compute Efficiency

### Serverless First

**Standard**: Use serverless for variable workloads.

**Why Serverless is Sustainable**:
- No idle compute
- Shared infrastructure
- Automatic right-sizing
- AWS manages efficiency

### ARM64/Graviton2

**Standard**: Use Graviton processors.

```yaml
Globals:
  Function:
    Architectures:
      - arm64  # More efficient than x86
```

**Benefits**:
- 20% better price/performance
- Lower energy per transaction
- Same functionality

### Right-Size Functions

**Standard**: Optimize memory and timeout.

```yaml
# Test and optimize
# Use Lambda Power Tuning
Function:
  Properties:
    MemorySize: 512    # Tested optimal
    Timeout: 10        # Actual maximum need
    EphemeralStorage:
      Size: 512        # Only if needed
```

---

## Data Efficiency

### Minimize Data Transfer

**Standard**: Keep data close to compute.

```yaml
# Same region for all resources
Database:
  Region: us-east-2

ApiLambda:
  Region: us-east-2  # Same region as DB

# Cache at edge
CloudFront:
  Properties:
    Origins:
      - DomainName: !GetAtt ApiBucket.RegionalDomainName
```

### Efficient Data Formats

**Standard**: Use compact formats.

```javascript
// JSON with compression
const response = {
  statusCode: 200,
  headers: {
    'Content-Encoding': 'gzip'
  },
  body: zlib.gzipSync(JSON.stringify(data)).toString('base64'),
  isBase64Encoded: true
};

// Consider binary formats for large data
// - Protocol Buffers
// - MessagePack
// - Parquet for analytics
```

### Data Lifecycle Management

**Standard**: Delete or archive unneeded data.

```yaml
# S3 lifecycle
Bucket:
  Properties:
    LifecycleConfiguration:
      Rules:
        # Transition to cheaper storage
        - Id: TransitionOldData
          Status: Enabled
          Transitions:
            - StorageClass: STANDARD_IA
              TransitionInDays: 30
            - StorageClass: GLACIER
              TransitionInDays: 90
        # Delete when no longer needed
        - Id: ExpireOldLogs
          Status: Enabled
          ExpirationInDays: 365
          Prefix: logs/
```

---

## Storage Efficiency

### Appropriate Storage Class

**Standard**: Match storage class to access pattern.

| Access Pattern | Storage Class |
|----------------|---------------|
| Frequent | S3 Standard |
| Infrequent (>30 days) | S3 Standard-IA |
| Rare (>90 days) | S3 Glacier |
| Archive (>180 days) | S3 Glacier Deep Archive |
| Unknown | S3 Intelligent-Tiering |

### Compression

**Standard**: Compress stored data.

```javascript
const zlib = require('zlib');

// Compress before storing
async function storeData(key, data) {
  const compressed = zlib.gzipSync(JSON.stringify(data));
  await s3.putObject({
    Bucket: bucket,
    Key: `${key}.gz`,
    Body: compressed,
    ContentEncoding: 'gzip'
  });
}
```

### Deduplication

**Standard**: Avoid storing duplicates.

```javascript
// Content-addressable storage
const crypto = require('crypto');

function getContentHash(data) {
  return crypto.createHash('sha256').update(data).digest('hex');
}

async function storeIfNew(data) {
  const hash = getContentHash(data);
  const exists = await checkExists(hash);

  if (!exists) {
    await store(hash, data);
  }

  return hash;  // Return reference
}
```

---

## Network Efficiency

### Regional Deployment

**Standard**: Deploy in efficient regions.

**Low-Carbon Regions** (varies by time):
- US West (Oregon) - Hydroelectric
- EU (Ireland) - Wind
- Canada (Montreal) - Hydroelectric

Check AWS Customer Carbon Footprint Tool for current data.

### Minimize Cross-Region Traffic

**Standard**: Keep traffic within region.

```yaml
# All resources same region
Parameters:
  Region:
    Type: String
    Default: us-west-2

# Use regional endpoints
S3Endpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
```

### Edge Caching

**Standard**: Cache at edge to reduce origin requests.

```yaml
CloudFront:
  Properties:
    DefaultCacheBehavior:
      CachePolicyId: !Ref AggressiveCachePolicy
      Compress: true
```

---

## Code Efficiency

### Efficient Algorithms

**Standard**: Use efficient implementations.

```javascript
// GOOD - O(n)
const map = new Map(items.map(i => [i.id, i]));
const found = map.get(targetId);

// BAD - O(n²)
// const found = items.find(i => i.id === targetId);
// (when called repeatedly)
```

### Minimize Dependencies

**Standard**: Only include needed dependencies.

```javascript
// GOOD - specific import
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');

// BAD - entire SDK
// const AWS = require('aws-sdk');
```

### Lazy Loading

**Standard**: Load resources when needed.

```javascript
let heavyModule;

async function processIfNeeded(data) {
  if (needsHeavyProcessing(data)) {
    // Only load if needed
    heavyModule = heavyModule || require('heavy-module');
    return heavyModule.process(data);
  }
  return simpleProcess(data);
}
```

---

## Monitoring and Improvement

### Track Efficiency Metrics

**Standard**: Monitor sustainability indicators.

```javascript
// Custom efficiency metrics
await cloudwatch.putMetricData({
  Namespace: 'Sustainability',
  MetricData: [
    {
      MetricName: 'RequestsPerComputeHour',
      Value: requests / computeHours,
      Unit: 'Count/Second'
    },
    {
      MetricName: 'DataProcessedPerDollar',
      Value: dataGB / cost,
      Unit: 'Gigabytes'
    }
  ]
});
```

### Regular Review

**Standard**: Quarterly sustainability review.

**Review Checklist**:
- [ ] Lambda memory utilization
- [ ] Storage lifecycle effectiveness
- [ ] Cache hit rates
- [ ] Idle resource identification
- [ ] Cross-region traffic patterns

---

## Sustainable Development Practices

### Efficient CI/CD

**Standard**: Minimize build waste.

```yaml
# Cache dependencies
cache:
  paths:
    - node_modules/

# Incremental builds
build:
  script:
    - npm ci --prefer-offline
    - npm run build -- --only-changed
```

### Test Efficiency

**Standard**: Run appropriate tests.

```yaml
# Staged testing - don't run everything every time
stages:
  - lint         # Fast, always
  - unit-test    # Fast, always
  - integration  # Slower, on PR
  - e2e          # Slowest, on merge to main
```

---

## Equilateral Cross-References

| Topic | Standard Document |
|-------|------------------|
| Cost Optimization | `cost-optimization/cost_optimization_standards.md` |
| Lambda Database | `serverless-saas-aws/lambda_database_standards.md` |
| Deployment | `serverless-saas-aws/deployment_standards.md` |
| Frontend Performance | `frontend-development/frontend_standards.md` |
