# Multi-Stack Architecture Standards

**Version:** 1.0.0
**Last Updated:** 2025-11-16
**Status:** Active Production Architecture

## Overview

Tim-Combo/HappyHippo uses a **multi-stack, multi-API Gateway architecture** across all environments. This is a **CloudFormation best practice** that separates concerns and enables:

✅ **Independent deployment** of different application domains
✅ **Separation of persistent infrastructure** (DB, VPC, S3) from ephemeral compute (Lambda)
✅ **Easier rollbacks** - roll back one domain without affecting others
✅ **Team autonomy** - different teams can own different stacks
✅ **Better organization** - each stack has clear responsibility

## Architecture Philosophy

### Persistent vs. Ephemeral Resources

**Persistent Infrastructure (Separate Stacks):**
- Database (RDS instances)
- VPC, Subnets, Security Groups
- S3 Buckets
- Cognito User Pools
- API Gateways (created once, referenced by many stacks)

**Ephemeral Compute (Domain Stacks):**
- Lambda functions (can be recreated quickly)
- Lambda permissions
- Lambda layers
- Function-specific IAM roles

### Why Multiple Stacks?

**Business Reasons:**
1. **Domain Separation** - Integrations, communications, core business logic are separate concerns
2. **Independent Deployment** - Deploy integration updates without touching core business logic
3. **Blast Radius Containment** - Failed deployment of one domain doesn't affect others
4. **Team Ownership** - Integration team owns flux-integrations-stack

**Technical Reasons:**
1. **AWS CloudFormation limits:** 200KB template size, 500 resources per stack
2. **API Gateway limits:** 300 routes per API Gateway
3. **Deployment speed:** Smaller stacks deploy faster
4. **Change management:** Easier to review smaller, focused templates

### Multi-Stack Architecture Pattern

Instead of one monolithic stack, we have:
- **Persistent infrastructure stacks** - Database, VPC, API Gateways (rarely change)
- **Domain-specific stacks** - Integrations, Communications, Core Business (frequent updates)
- **Cross-stack references** - Stacks import values from persistent infrastructure

## Architecture by Environment

### Sandbox Environment (Account: 455510265254)

**4 Active CloudFormation Stacks:**

1. **fluxSystems** (Main Stack)
   - Template: `IAC/sam/dev_stuff/SB_Flux/lambda_with_auth_updated_MAXED.yaml`
   - Size: 195K (**MAXED OUT** - do not add more functions)
   - API Gateway: `k0y33bw7t8` (primary)
   - Purpose: Core business logic, authentication, main workflows
   - Status: ⚠️ At AWS limits - route new functions to domain stacks

2. **flux-integrations-stack**
   - Templates: `IAC/cloudformation/lambda-stacks/lambda-integrations-*.yaml`
   - API Gateway: TBD (may use API_BASE2)
   - Purpose: All integration-related functions (ADP, QuickBooks, BambooHR, etc.)
   - Includes:
     - `lambda-integrations-adp-marketplace-additions.yaml` (ADP Marketplace webhooks)
     - `lambda-integrations-adp.yaml` (ADP WFN integration)
     - `lambda-integrations-*.yaml` (other integrations)

3. **flux-communications-stack**
   - Templates: `IAC/cloudformation/communication-functions.yaml`, `tim-dev-communication-functions.yaml`
   - Purpose: Email, SMS, notifications, alerts
   - API Gateway: Likely API_BASE2

4. **flux-main-stack**
   - Purpose: Additional core functions that don't fit in fluxSystems
   - Details: TBD (need to document)

### Development Environment (Account: 532595801838)

**Active Stacks:**
- **Clarity2025** (or similar) - uses `IAC/sam/dev_stuff/deployAPI/lambda_with_auth.yaml`
- Additional stacks TBD (need to document)

## API Gateway Architecture

### Primary API Gateway (API_BASE)
- **Sandbox ID:** `k0y33bw7t8`
- **Dev ID:** `n2eqji12v4` (Clarity2025)
- **Base URL Sandbox:** `https://k0y33bw7t8.execute-api.us-east-2.amazonaws.com/prod`
- **Base URL Dev:** `https://n2eqji12v4.execute-api.us-east-2.amazonaws.com/prod`
- **Purpose:** Primary business logic routes
- **Managed by:** `fluxSystems` (sandbox) or `Clarity2025` (dev) stack

### Secondary API Gateway (API_BASE2)
- **Status:** Planned/In Progress
- **Purpose:** Overflow routes when primary API Gateway hits limits
- **Will be used for:**
  - Integration endpoints
  - Communication endpoints
  - Additional domain-specific routes
- **Configuration:** TBD

## When to Use Which Stack

### Decision Tree

```
New Lambda Function Needed?
│
├─ Is it an integration with external system?
│  └─ YES → Use flux-integrations-stack
│     └─ Template: IAC/cloudformation/lambda-stacks/lambda-integrations-{system}.yaml
│
├─ Is it communication/notification related?
│  └─ YES → Use flux-communications-stack
│     └─ Template: IAC/cloudformation/communication-functions.yaml
│
├─ Is it core business logic?
│  └─ YES → Check if fluxSystems has space
│     ├─ Space available? → Add to fluxSystems (SAM template)
│     └─ MAXED OUT? → Add to flux-main-stack or create new domain stack
│
└─ New domain not covered above?
   └─ Create new stack: flux-{domain}-stack
      └─ Template: IAC/cloudformation/lambda-stacks/lambda-{domain}.yaml
```

### Stack Selection Guidelines

| Function Type | Stack | Template Location | API Gateway |
|--------------|-------|------------------|-------------|
| ADP Integration | flux-integrations-stack | `lambda-stacks/lambda-integrations-adp*.yaml` | API_BASE or API_BASE2 |
| QuickBooks Integration | flux-integrations-stack | `lambda-stacks/lambda-integrations-*.yaml` | API_BASE2 |
| Email/SMS/Notifications | flux-communications-stack | `communication-functions.yaml` | API_BASE2 |
| Core Business Logic | fluxSystems | `lambda_with_auth_updated_MAXED.yaml` ⚠️ | API_BASE |
| New Core Functions | flux-main-stack | TBD | API_BASE |
| Other Domains | Create new stack | `lambda-stacks/lambda-{domain}.yaml` | API_BASE2 |

## Template Naming Conventions

### SAM Templates (Monolithic)
- **Pattern:** `lambda_with_auth[_environment][_MAXED].yaml`
- **Examples:**
  - `lambda_with_auth_updated_MAXED.yaml` (Sandbox - maxed out)
  - `lambda_with_auth.yaml` (Dev - may also be near limits)
- **Location:** `IAC/sam/dev_stuff/{environment}/`
- **When to use:** Legacy - avoid adding to these templates when possible

### CloudFormation Modular Templates
- **Pattern:** `lambda-{domain}[-{subdomain}].yaml`
- **Examples:**
  - `lambda-integrations-adp-marketplace-additions.yaml`
  - `lambda-integrations-oauth.yaml`
  - `lambda-user-management.yaml`
  - `lambda-system-monitoring.yaml`
- **Location:** `IAC/cloudformation/lambda-stacks/`
- **When to use:** Preferred for all new functions

### Stack Naming
- **Pattern:** `flux-{domain}-stack` (sandbox) or `{environment}-{domain}-stack` (dev)
- **Examples:**
  - `flux-integrations-stack`
  - `flux-communications-stack`
  - `clarity-integrations-stack` (hypothetical dev equivalent)

## Adding New Functions: Step-by-Step

### Option 1: Add to Existing Domain Stack (Preferred)

1. **Identify the correct domain stack**
   ```bash
   # List existing stacks
   aws cloudformation list-stacks --profile sandbox-sso --region us-east-2 \
     --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE
   ```

2. **Find or create the modular template**
   ```bash
   # Check if domain template exists
   ls IAC/cloudformation/lambda-stacks/lambda-{domain}*.yaml
   ```

3. **Add function definition following CloudFormation pattern**
   ```yaml
   # Example from lambda-integrations-adp-marketplace-additions.yaml
   MonitorADPCredentialsFunction:
     Type: AWS::Lambda::Function
     Properties:
       FunctionName: !Sub '${StackName}-monitorADPCredentials'
       Runtime: 'nodejs22.x'
       Handler: 'index.handler'
       Role: !Ref LambdaExecutionRoleArn
       Timeout: 30
       MemorySize: 512
       Code:
         S3Bucket: !Ref TIMBucketName
         S3Key: 'monitorADPCredentials_js.zip'
       Architectures:
         - 'arm64'  # 20% cost savings for ARM64
   ```

4. **Add Lambda permission**
   ```yaml
   MonitorADPCredentialsPermission:
     Type: AWS::Lambda::Permission
     Properties:
       FunctionName: !Ref MonitorADPCredentialsFunction
       Action: 'lambda:InvokeFunction'
       Principal: 'apigateway.amazonaws.com'
       SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ApiGatewayId}/*'
   ```

5. **Add output export**
   ```yaml
   Outputs:
     MonitorADPCredentialsFunctionArn:
       Description: 'ARN of the monitorADPCredentials function'
       Value: !GetAtt MonitorADPCredentialsFunction.Arn
       Export:
         Name: !Sub '${StackName}-MonitorADPCredentialsFunctionArn'
   ```

6. **Deploy the stack**
   ```bash
   cd IAC/cloudformation
   ./deploy-cloudformation.sh sandbox us-east-2 integrations-adp-marketplace
   ```

### Option 2: Create New Domain Stack (When Needed)

1. **Create new modular template**
   ```bash
   cp IAC/cloudformation/lambda-stacks/lambda-template-example.yaml \
      IAC/cloudformation/lambda-stacks/lambda-{new-domain}.yaml
   ```

2. **Define stack in deployment script**
   - Add to `IAC/cloudformation/deploy-cloudformation.sh`
   - Define stack name: `flux-{new-domain}-stack`

3. **Follow Option 1 steps to add functions**

## MAXED OUT Templates: What To Do

### Identifying Maxed Out Templates

Templates marked with `_MAXED` in filename:
- `lambda_with_auth_updated_MAXED.yaml` (Sandbox - 195K)
- Any template >190KB should be considered near limits

### Warning Signs
- File size >190KB
- CloudFormation deployment errors about template size
- >450 resources in template
- Deployment takes >10 minutes

### DO NOT:
- ❌ Add more functions to `_MAXED` templates
- ❌ Try to "compress" YAML to fit more
- ❌ Remove comments/whitespace to add functions
- ❌ Split single function across multiple resources

### DO:
- ✅ Add functions to appropriate domain stack instead
- ✅ Create new modular template if needed
- ✅ Move functions from MAXED template to domain stacks (refactoring)
- ✅ Document which template is maxed in `.myWorld.json`

## API Gateway Route Management

### Route Distribution Strategy

**Primary API Gateway (API_BASE):**
- Core authentication routes (`/tim/auth/*`)
- Core business routes (`/tim/clients/*`, `/tim/companies/*`)
- High-traffic routes
- User-facing routes

**Secondary API Gateway (API_BASE2):**
- Integration routes (`/tim/integrations/*`)
- Communication routes (`/tim/communications/*`)
- Webhook endpoints (external systems calling us)
- Low-traffic administrative routes

### Adding Routes

Routes are defined in the API Gateway integration, not in Lambda function definitions.

**For SAM templates**, routes are defined in Events:
```yaml
Events:
  monitorCredentialsGet:
    Type: Api
    Properties:
      RestApiId: !Ref ApiGateway  # References API_BASE
      Path: /tim/marketplace/credentials/monitor
      Method: GET
      Auth:
        Authorizer: CognitoAuthorizer
```

**For CloudFormation modular templates**, routes are managed separately or via API Gateway definitions.

## Environment-Specific Configuration

### Parameters to Parameterize

Always parameterize these values (never hardcode):
- Database connection (`DBHost`, `DBUser`, `DBPass`, `DBName`)
- S3 bucket names (`TIMBucketName`)
- API Gateway IDs (`ApiGatewayId`, `ApiGateway2Id`)
- Cognito User Pool ARNs
- Environment-specific URLs
- AWS Account IDs (use `AWS::AccountId` pseudo-parameter)

### Multi-Environment Templates

Same template should work in dev, sandbox, and production by:
1. Using Parameters for environment-specific values
2. Using CloudFormation pseudo-parameters (`AWS::Region`, `AWS::AccountId`)
3. Using Stack Name in resource naming: `!Sub '${StackName}-functionName'`

## Cross-Stack References and Shared Resources

### Persistent Infrastructure Stack

**Purpose:** Houses resources that are:
- Long-lived (databases, VPCs)
- Expensive to replace
- Referenced by multiple domain stacks
- Change infrequently

**Resources:**
```yaml
# Example: persistent-infrastructure.yaml
Resources:
  # Database
  TimDatabase:
    Type: AWS::RDS::DBInstance
    # ... configuration ...

  # Primary API Gateway
  TimApiGateway:
    Type: AWS::ApiGateway::RestApi
    # ... configuration ...

  # Cognito User Pool
  TimUserPool:
    Type: AWS::Cognito::UserPool
    # ... configuration ...

  # Shared Lambda Execution Role
  TimLambdaExecutionRole:
    Type: AWS::IAM::Role
    # ... configuration ...

Outputs:
  DatabaseEndpoint:
    Description: RDS database endpoint
    Value: !GetAtt TimDatabase.Endpoint.Address
    Export:
      Name: !Sub '${AWS::StackName}-DatabaseEndpoint'

  ApiGatewayId:
    Description: Primary API Gateway ID
    Value: !Ref TimApiGateway
    Export:
      Name: !Sub '${AWS::StackName}-ApiGatewayId'

  LambdaExecutionRoleArn:
    Description: Shared Lambda execution role ARN
    Value: !GetAtt TimLambdaExecutionRole.Arn
    Export:
      Name: !Sub '${AWS::StackName}-LambdaExecutionRoleArn'
```

### Domain Stacks Import Shared Resources

**Pattern:** Use `!ImportValue` to reference exported values from persistent stack

```yaml
# Example: lambda-integrations-adp.yaml
Parameters:
  PersistentStackName:
    Type: String
    Default: 'flux-persistent-infrastructure'
    Description: Name of the persistent infrastructure stack

Resources:
  AdpIntegrationFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${AWS::StackName}-adpIntegration'
      Role: !ImportValue
        Fn::Sub: '${PersistentStackName}-LambdaExecutionRoleArn'
      Environment:
        Variables:
          DB_HOST: !ImportValue
            Fn::Sub: '${PersistentStackName}-DatabaseEndpoint'

  AdpIntegrationPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref AdpIntegrationFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub
        - 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ApiGatewayId}/*'
        - ApiGatewayId: !ImportValue
            Fn::Sub: '${PersistentStackName}-ApiGatewayId'
```

### Benefits of Cross-Stack References

✅ **Single Source of Truth** - Database endpoint defined once, used everywhere
✅ **Automatic Updates** - If RDS endpoint changes, all stacks get new value automatically
✅ **Dependency Management** - CloudFormation prevents deleting exported resources
✅ **Type Safety** - Import failures caught at deployment time

### Shared Resource Patterns

| Resource Type | Where Defined | How Referenced | Update Frequency |
|--------------|---------------|----------------|------------------|
| RDS Database | Persistent stack | `!ImportValue ${Stack}-DatabaseEndpoint` | Never (scale in place) |
| API Gateway | Persistent stack | `!ImportValue ${Stack}-ApiGatewayId` | Rarely |
| Cognito Pool | Persistent stack | `!ImportValue ${Stack}-CognitoPoolArn` | Never |
| S3 Buckets | Persistent stack | `!ImportValue ${Stack}-LambdaBucket` | Never |
| VPC/Subnets | Persistent stack | `!ImportValue ${Stack}-VPCId` | Never |
| Lambda Role | Persistent stack | `!ImportValue ${Stack}-LambdaRoleArn` | Occasionally |
| Lambda Functions | Domain stacks | N/A (not shared) | Frequently |

### Current Shared Resource Exports (Sandbox)

**From fluxSystems stack (acts as persistent stack currently):**
```yaml
Exports:
  - Name: fluxSystems-ApiGatewayId
    Value: k0y33bw7t8
  - Name: fluxSystems-LambdaExecutionRoleArn
    Value: arn:aws:iam::455510265254:role/fluxSystems-LambdaExecutionRole
  - Name: fluxSystems-DatabaseEndpoint
    Value: fluxcapacitor2325.c9ncobp8exjr.us-east-2.rds.amazonaws.com
  - Name: fluxSystems-CognitoUserPoolArn
    Value: arn:aws:cognito-idp:us-east-2:455510265254:userpool/us-east-2_57sEtr0xp
```

**Domain stacks should import these values instead of receiving as parameters.**

### Migration Strategy: Parameters → Imports

**Current (Parameter Passing):**
```yaml
# Domain stack receives parameters
Parameters:
  DBHost:
    Type: String
  ApiGatewayId:
    Type: String
```

**Target (Import Values):**
```yaml
# Domain stack imports from persistent stack
Resources:
  MyFunction:
    Environment:
      Variables:
        DB_HOST: !ImportValue fluxSystems-DatabaseEndpoint
```

**Benefits:**
- No need to manually pass 20+ parameters to each domain stack
- Changes to shared resources propagate automatically
- Reduced human error in parameter values

## Deployment Workflows

### Sandbox Deployment

```bash
# 1. Sync Lambda artifacts from dev to sandbox
aws s3 sync s3://tim-dev-lambda/ s3://tim-sb-be-live/ --profile sandbox-sso

# 2. Deploy main stack (if not maxed)
cd IAC/sam/dev_stuff/SB_Flux
sam deploy --template-file lambda_with_auth_updated_MAXED.yaml \
  --stack-name fluxSystems \
  --capabilities CAPABILITY_IAM \
  --profile sandbox-sso

# 3. Deploy domain stacks
cd IAC/cloudformation
./deploy-cloudformation.sh sandbox us-east-2 integrations-adp-marketplace
./deploy-cloudformation.sh sandbox us-east-2 communications

# 4. Force update all Lambda functions
bash scripts/force-update-lambdas.sh
```

### Development Deployment

Similar to sandbox, but using dev account profile and appropriate stack names.

## Frontend Configuration

### API Endpoint Configuration

Frontend must be configured with correct API Gateway URLs:

**Sandbox (`.env.sandbox`):**
```
REACT_APP_API_URL=https://k0y33bw7t8.execute-api.us-east-2.amazonaws.com/prod
REACT_APP_API_URL_2=https://{api-gateway-2-id}.execute-api.us-east-2.amazonaws.com/prod  # When implemented
```

**Dev (`.env.development`):**
```
REACT_APP_API_URL=https://n2eqji12v4.execute-api.us-east-2.amazonaws.com/prod
REACT_APP_API_URL_2=https://{api-gateway-2-id}.execute-api.us-east-2.amazonaws.com/prod  # When implemented
```

### API Client Multi-Gateway Configuration

The frontend `apiClient.ts` handles multiple API Gateways transparently to application code.

#### Configuration (`src/frontend/src/api/core/apiClient.ts`)

```typescript
// Environment-driven API base URLs
const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:3001';
const API_BASE2 = process.env.REACT_APP_API_URL_2 || API_BASE;  // Fallback to primary if not configured

// Validate configuration on load
if (API_BASE === API_BASE2 && process.env.NODE_ENV === 'production') {
  console.warn('API_BASE2 not configured - all requests routing to primary API Gateway');
}

// Endpoint routing - transparently directs to appropriate API Gateway
export const API_ENDPOINTS = {
  // Core business endpoints → Primary API Gateway (API_BASE)
  AUTH: {
    LOGIN: `${API_BASE}/tim/auth/login`,
    LOGOUT: `${API_BASE}/tim/auth/logout`,
    REFRESH: `${API_BASE}/tim/auth/refresh`
  },
  CLIENTS: {
    BASE: `${API_BASE}/tim/clients`,
    GET: `${API_BASE}/tim/clients/:clientId`,
    LIST: `${API_BASE}/tim/clients`
  },
  COMPANIES: {
    BASE: `${API_BASE}/tim/companies`,
    GET: `${API_BASE}/tim/companies/:companyId`
  },

  // Integration endpoints → Secondary API Gateway (API_BASE2)
  INTEGRATIONS: {
    BASE: `${API_BASE2}/tim/integrations`,
    INSTANCES: `${API_BASE2}/tim/integrations/instances`,
    TEMPLATES: `${API_BASE2}/tim/integrations/templates`,
    CREDENTIALS: {
      MONITOR: `${API_BASE2}/tim/marketplace/credentials/monitor`,
      UPDATE: `${API_BASE2}/tim/marketplace/credentials/update`
    }
  },

  // Communication endpoints → Secondary API Gateway (API_BASE2)
  COMMUNICATIONS: {
    SEND_EMAIL: `${API_BASE2}/tim/communications/email`,
    SEND_SMS: `${API_BASE2}/tim/communications/sms`,
    NOTIFICATIONS: `${API_BASE2}/tim/communications/notifications`
  },

  // Marketplace webhooks → Secondary API Gateway (API_BASE2)
  MARKETPLACE: {
    OAUTH_TOKEN: `${API_BASE2}/tim/marketplace/oauth/token`,
    SUBSCRIPTION_CREATE: `${API_BASE2}/tim/marketplace/subscription/create`,
    SUBSCRIPTION_CANCEL: `${API_BASE2}/tim/marketplace/subscription/cancel`
  }
};
```

#### Routing Strategy

**Primary API Gateway (API_BASE):**
- Authentication and session management
- Core business entities (clients, companies, employees)
- High-traffic, user-facing routes
- Real-time operations
- Super admin functions

**Secondary API Gateway (API_BASE2):**
- External system integrations (ADP, QuickBooks, etc.)
- Webhooks (incoming from partners)
- Communication services (email, SMS)
- Batch operations
- Background processing endpoints

#### Why This Approach Works

✅ **Transparent to Application Code** - Components don't need to know which API Gateway
✅ **Environment-Driven** - Same code works in dev/sandbox/production
✅ **Graceful Degradation** - Falls back to primary if secondary not configured
✅ **Route Capacity Management** - Automatically distributes load across gateways
✅ **Single Source of Truth** - All endpoints defined in one place

#### Making API Calls

Application code uses the same `Make_Authorized_API_Call` function regardless of which API Gateway:

```typescript
// Example: Call integration endpoint (automatically routes to API_BASE2)
const result = await Make_Authorized_API_Call(
  API_ENDPOINTS.INTEGRATIONS.INSTANCES,
  'GET'
);

// Example: Call client endpoint (automatically routes to API_BASE)
const client = await Make_Authorized_API_Call(
  API_ENDPOINTS.CLIENTS.GET.replace(':clientId', clientId),
  'GET'
);

// Example: Call marketplace monitoring (automatically routes to API_BASE2)
const health = await Make_Authorized_API_Call(
  API_ENDPOINTS.INTEGRATIONS.CREDENTIALS.MONITOR,
  'GET'
);
```

#### Adding New Endpoints

**Step 1:** Determine which API Gateway based on domain
```
Is it an integration? → API_BASE2
Is it communication? → API_BASE2
Is it a webhook? → API_BASE2
Is it core business logic? → API_BASE
```

**Step 2:** Add to appropriate section in `API_ENDPOINTS`
```typescript
export const API_ENDPOINTS = {
  // ... existing endpoints ...

  NEW_DOMAIN: {
    BASE: `${API_BASE2}/tim/new-domain`,  // Use API_BASE2 for new integrations
    ACTION: `${API_BASE2}/tim/new-domain/action`
  }
};
```

**Step 3:** Use in component as normal
```typescript
const result = await Make_Authorized_API_Call(
  API_ENDPOINTS.NEW_DOMAIN.ACTION,
  'POST',
  { data: 'value' }
);
```

#### Migration: Single → Multi-Gateway

If currently all endpoints use `API_BASE`, migrate gradually:

**Phase 1: Add API_BASE2 (same as API_BASE)**
```bash
# .env.sandbox
REACT_APP_API_URL=https://k0y33bw7t8.execute-api.us-east-2.amazonaws.com/prod
REACT_APP_API_URL_2=https://k0y33bw7t8.execute-api.us-east-2.amazonaws.com/prod  # Same initially
```

**Phase 2: Update endpoint definitions**
```typescript
// Change integration endpoints to use API_BASE2
INTEGRATIONS: {
  BASE: `${API_BASE2}/tim/integrations`,  // Changed from API_BASE
}
```

**Phase 3: Deploy secondary API Gateway**
```bash
# Create new API Gateway for integrations
# Update .env with new API Gateway ID
REACT_APP_API_URL_2=https://{new-api-id}.execute-api.us-east-2.amazonaws.com/prod
```

**Phase 4: Redeploy frontend**
```bash
npm run build:sandbox
# Endpoints automatically route to new API Gateway
```

#### Troubleshooting Multi-Gateway Setup

**Problem:** 404 errors on integration endpoints

**Diagnosis:**
```typescript
// Check which URL is being called
console.log('API_BASE:', API_BASE);
console.log('API_BASE2:', API_BASE2);
console.log('Integration endpoint:', API_ENDPOINTS.INTEGRATIONS.BASE);
```

**Solution:** Ensure `REACT_APP_API_URL_2` is set correctly in `.env`

**Problem:** CORS errors only on integration endpoints

**Diagnosis:** API_BASE2 may not have CORS configured

**Solution:**
```bash
# Check API Gateway CORS settings
aws apigateway get-resources --rest-api-id {api-base2-id} --profile sandbox-sso
```

#### Best Practices

✅ **DO:**
- Use `API_BASE` for high-traffic, user-facing routes
- Use `API_BASE2` for integrations, webhooks, background jobs
- Always fallback: `API_BASE2 = API_BASE2 || API_BASE`
- Log which API Gateway is being used in development
- Test both API Gateways independently

❌ **DON'T:**
- Hardcode API Gateway URLs in components
- Mix API_BASE and API_BASE2 arbitrarily
- Assume API_BASE2 is always configured (handle fallback)
- Skip CORS configuration on API_BASE2
- Forget to update both API Gateways when adding routes

## Monitoring and Observability

### Identifying Which Stack a Lambda Belongs To

**By function name prefix:**
```bash
# List all Lambda functions with stack prefix
aws lambda list-functions --profile sandbox-sso \
  --query 'Functions[?starts_with(FunctionName, `fluxSystems-`)].FunctionName'

aws lambda list-functions --profile sandbox-sso \
  --query 'Functions[?starts_with(FunctionName, `flux-integrations-`)].FunctionName'
```

**By CloudFormation stack:**
```bash
# List all resources in a stack
aws cloudformation list-stack-resources \
  --stack-name fluxSystems \
  --profile sandbox-sso \
  --query 'StackResourceSummaries[?ResourceType==`AWS::Lambda::Function`].PhysicalResourceId'
```

### Stack Health Monitoring

```bash
# Check stack status
aws cloudformation describe-stacks \
  --stack-name fluxSystems \
  --profile sandbox-sso \
  --query 'Stacks[0].StackStatus'

# Get recent stack events (for troubleshooting)
aws cloudformation describe-stack-events \
  --stack-name fluxSystems \
  --profile sandbox-sso \
  --max-items 20
```

## Common Mistakes to Avoid

### 1. Adding to Maxed Templates
**Mistake:** Adding functions to `lambda_with_auth_updated_MAXED.yaml`
**Impact:** Deployment fails due to CloudFormation size limits
**Solution:** Use appropriate domain stack instead

### 2. Hardcoding API Gateway IDs
**Mistake:** Hardcoding `k0y33bw7t8` in Lambda permission source ARN
**Impact:** Template won't work in other environments
**Solution:** Use parameter `!Ref ApiGatewayId`

### 3. Inconsistent Naming
**Mistake:** Using different naming patterns across stacks
**Impact:** Difficult to find and manage resources
**Solution:** Follow naming convention: `${StackName}-functionName`

### 4. Not Using ARM64
**Mistake:** Using x86_64 architecture for new Lambda functions
**Impact:** 20% higher costs unnecessarily
**Solution:** Always use `Architectures: ['arm64']` for new functions

### 5. Forgetting Lambda Permissions
**Mistake:** Adding Lambda function but forgetting `AWS::Lambda::Permission`
**Impact:** API Gateway can't invoke the function (403 errors)
**Solution:** Always add both Function and Permission resources

## Troubleshooting

### Template Size Issues

**Symptom:** `Template format error: template size exceeds limit`

**Diagnosis:**
```bash
# Check template size
ls -lh IAC/sam/dev_stuff/SB_Flux/lambda_with_auth_updated_MAXED.yaml
# If >190KB, it's near limits
```

**Solution:** Move functions to domain-specific stack

### API Gateway Route Conflicts

**Symptom:** Routes not accessible or returning 403/404

**Diagnosis:**
```bash
# Check which API Gateway a route is registered to
aws apigateway get-resources \
  --rest-api-id k0y33bw7t8 \
  --profile sandbox-sso
```

**Solution:** Ensure route is registered to correct API Gateway

### Stack Update Failures

**Symptom:** CloudFormation stack update fails midway

**Diagnosis:**
```bash
# Get failure reason
aws cloudformation describe-stack-events \
  --stack-name fluxSystems \
  --profile sandbox-sso \
  --query 'StackEvents[?ResourceStatus==`UPDATE_FAILED`]' \
  --output table
```

**Solution:** Fix the specific resource causing failure, rollback if needed

## Future Architecture Considerations

### When to Create API_BASE3

If API_BASE2 approaches 250+ routes:
1. Create third API Gateway
2. Move lowest-traffic domains to API_BASE3
3. Update frontend API client with API_BASE3 configuration
4. Document in this standard

### Stack Consolidation Strategy

As architecture matures, consider:
1. Merging small domain stacks into larger consolidated stacks
2. Creating "uber-templates" that reference multiple modular templates
3. Using CloudFormation nested stacks for better organization

## References

- **CloudFormation Limits:** https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html
- **API Gateway Limits:** https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html
- **Project Deployment Manifest:** `.myWorld.json`
- **Backend Handler Standards:** `.clinerules/backend_handler_standards.md`

## Changelog

### 2025-11-16
- Initial documentation of multi-stack architecture
- Documented 4-stack sandbox architecture
- Added MAXED template guidelines
- Documented API_BASE / API_BASE2 concept (planned)
