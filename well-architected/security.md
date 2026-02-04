# Security - AWS Well-Architected

## Overview

Security focuses on protecting information, systems, and assets while delivering business value through risk assessments and mitigation strategies.

**AWS Reference**: https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/

---

## Design Principles

### 1. Implement a Strong Identity Foundation

**Standard**: Least privilege access for all identities.

**IAM Policy Pattern**:
```yaml
# Lambda execution role - minimal permissions
LambdaExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: MinimalAccess
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            # Specific table, not *
            - Effect: Allow
              Action:
                - dynamodb:GetItem
                - dynamodb:PutItem
                - dynamodb:Query
              Resource: !GetAtt OrdersTable.Arn
            # Specific parameter path
            - Effect: Allow
              Action: ssm:GetParameter
              Resource: !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/${Environment}/app/*'
```

**Never Allow**:
```yaml
# BAD - Never do this
- Effect: Allow
  Action: '*'
  Resource: '*'
```

---

### 2. Enable Traceability

**Standard**: Log all security-relevant events.

**Required Logging**:
```javascript
// Security event logging
function logSecurityEvent(event) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    eventType: 'SECURITY',
    action: event.action,
    userId: event.userId,
    sourceIp: event.sourceIp,
    userAgent: event.userAgent,
    resource: event.resource,
    outcome: event.outcome,
    reason: event.reason
  }));
}

// Log authentication attempts
logSecurityEvent({
  action: 'AUTHENTICATION',
  userId: username,
  sourceIp: event.requestContext.identity.sourceIp,
  outcome: 'FAILURE',
  reason: 'Invalid credentials'
});
```

**CloudTrail Configuration**:
```yaml
CloudTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    IsLogging: true
    IsMultiRegionTrail: true
    EnableLogFileValidation: true
    S3BucketName: !Ref AuditLogsBucket
```

---

### 3. Apply Security at All Layers

**Standard**: Defense in depth.

**Layer Security**:

| Layer | Controls |
|-------|----------|
| Edge | WAF, CloudFront, Shield |
| API | API Gateway authorization, throttling |
| Compute | Lambda VPC, security groups |
| Data | Encryption, access policies |
| Network | Private subnets, NACLs |

---

### 4. Automate Security Best Practices

**Standard**: Security as code.

**AWS Config Rules**:
```yaml
# Automated compliance checking
ConfigRuleEncryptionEnabled:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: s3-bucket-server-side-encryption-enabled
    Source:
      Owner: AWS
      SourceIdentifier: S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED
```

---

### 5. Protect Data in Transit and at Rest

**Standard**: Encrypt everything.

**In Transit**:
```yaml
# API Gateway - enforce HTTPS
Api:
  Type: AWS::Serverless::Api
  Properties:
    StageName: prod
    EndpointConfiguration: REGIONAL
    # No HTTP, only HTTPS
```

**At Rest**:
```yaml
# S3 bucket encryption
DataBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: aws:kms
            KMSMasterKeyID: !Ref DataEncryptionKey

# RDS encryption
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    StorageEncrypted: true
    KmsKeyId: !Ref DatabaseEncryptionKey
```

---

### 6. Keep People Away from Data

**Standard**: Minimize direct access.

**Practices**:
- No production database access without approval
- Use Systems Manager Session Manager, not SSH
- Automated data access through Lambda, not manual queries
- Audit all data access

---

### 7. Prepare for Security Events

**Standard**: Incident response plan.

**Required Components**:
```markdown
## Security Incident Response Plan

### Severity Levels
- P1: Data breach, active attack
- P2: Vulnerability discovered, suspicious activity
- P3: Policy violation, failed security scan

### Response Steps
1. Detect - CloudWatch alarms, GuardDuty
2. Contain - Revoke access, isolate resources
3. Eradicate - Remove threat, patch vulnerability
4. Recover - Restore services, validate security
5. Learn - Post-incident review, improve controls
```

---

## Authentication Standards

### Cognito Configuration

**Standard**: Use Cognito for user authentication.

```yaml
UserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    UserPoolName: !Sub '${AWS::StackName}-users'
    Policies:
      PasswordPolicy:
        MinimumLength: 12
        RequireLowercase: true
        RequireUppercase: true
        RequireNumbers: true
        RequireSymbols: true
    MfaConfiguration: OPTIONAL
    EnabledMfas:
      - SOFTWARE_TOKEN_MFA
    AccountRecoverySetting:
      RecoveryMechanisms:
        - Name: verified_email
          Priority: 1
```

**Equilateral Pattern**: `serverless-saas-aws/cognito_authentication_standards.md`

### API Authorization

**Standard**: Authorize all API endpoints.

```yaml
# API Gateway authorizer
CognitoAuthorizer:
  Type: AWS::ApiGateway::Authorizer
  Properties:
    RestApiId: !Ref Api
    Name: CognitoAuthorizer
    Type: COGNITO_USER_POOLS
    ProviderARNs:
      - !GetAtt UserPool.Arn
    IdentitySource: method.request.header.Authorization

# Function with authorizer
GetOrderFunction:
  Type: AWS::Serverless::Function
  Properties:
    Events:
      Api:
        Type: Api
        Properties:
          Path: /orders/{id}
          Method: GET
          Auth:
            Authorizer: CognitoAuthorizer
```

---

## Secrets Management

### SSM Parameter Store

**Standard**: Never hardcode secrets.

```yaml
# Reference secrets in SAM
Environment:
  Variables:
    DB_PASSWORD: '{{resolve:ssm-secure:/prod/db/password:1}}'
    API_KEY: '{{resolve:ssm-secure:/prod/integrations/stripe-key:1}}'
```

**Equilateral Pattern**: `serverless-saas-aws/lambda_database_standards.md`

### Secret Rotation

**Standard**: Rotate secrets automatically.

```yaml
SecretRotation:
  Type: AWS::SecretsManager::RotationSchedule
  Properties:
    SecretId: !Ref DatabaseSecret
    RotationRules:
      AutomaticallyAfterDays: 30
    RotationLambdaARN: !GetAtt RotationFunction.Arn
```

---

## Input Validation

### OWASP Top 10 Protection

**Standard**: Validate all inputs.

```javascript
const Joi = require('joi');

// Strict input validation
const orderSchema = Joi.object({
  productId: Joi.string().uuid().required(),
  quantity: Joi.number().integer().min(1).max(100).required(),
  email: Joi.string().email().required(),
  notes: Joi.string().max(500).allow('')
});

async function handler(event) {
  const body = JSON.parse(event.body);

  const { error, value } = orderSchema.validate(body, {
    stripUnknown: true  // Remove unexpected fields
  });

  if (error) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'Invalid input' })
      // Don't expose validation details to client
    };
  }

  // Use validated 'value', not raw 'body'
  return processOrder(value);
}
```

### SQL Injection Prevention

**Standard**: Always use parameterized queries.

```javascript
// CORRECT - parameterized query
const result = await client.query(
  'SELECT * FROM orders WHERE user_id = $1 AND status = $2',
  [userId, status]
);

// WRONG - string interpolation (SQL injection risk)
// const result = await client.query(
//   `SELECT * FROM orders WHERE user_id = '${userId}'`
// );
```

### XSS Prevention

**Standard**: Sanitize all output.

```javascript
const DOMPurify = require('dompurify');

function sanitizeUserContent(content) {
  return DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href']
  });
}
```

---

## WAF Configuration

**Standard**: Protect APIs with WAF.

```yaml
WebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    DefaultAction:
      Allow: {}
    Rules:
      - Name: AWSManagedRulesCommonRuleSet
        Priority: 1
        OverrideAction:
          None: {}
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesCommonRuleSet
        VisibilityConfig:
          CloudWatchMetricsEnabled: true
          MetricName: CommonRuleSet
          SampledRequestsEnabled: true
      - Name: RateLimitRule
        Priority: 2
        Action:
          Block: {}
        Statement:
          RateBasedStatement:
            Limit: 2000
            AggregateKeyType: IP
```

---

## Equilateral Cross-References

| Topic | Standard Document |
|-------|------------------|
| Authentication | `serverless-saas-aws/cognito_authentication_standards.md` |
| Credentials | `compliance-security/unified_credential_management_standards.md` |
| Tech Stack | `compliance-security/tech_stack_security.md` |
| Lambda Database | `serverless-saas-aws/lambda_database_standards.md` |
| API CORS | `serverless-saas-aws/api_gateway_cors_standards.md` |
