# API Gateway CORS Standards

## CRITICAL: DefaultAuthorizer Anti-Pattern

**NEVER use `DefaultAuthorizer` in SAM templates.** This is a common cause of authentication failures.

### The Problem
```yaml
# THIS BREAKS CORS - DO NOT USE
Api:
  Auth:
    DefaultAuthorizer: CognitoAuthorizer  # Applies auth to OPTIONS methods!
```

**Why it breaks:**
1. Browser sends OPTIONS preflight request (no auth headers)
2. API Gateway applies DefaultAuthorizer to OPTIONS method
3. Returns 401 Unauthorized for preflight
4. Browser blocks the actual API call
5. No data loads, authentication appears broken

### The Solution - Per-Function Authorization
```yaml
# CORRECT - Proven working pattern
Api:
  Cors:
    AllowMethods: "'GET, POST, PUT, DELETE, OPTIONS'"
    AllowHeaders: "'Content-Type, X-Amz-Date, Authorization, X-Api-Key, X-Amz-Security-Token'"
    AllowOrigin: "'*'"                            # Wildcard is FINE with JWT auth
    AllowCredentials: false                       # Not needed for JWT in header
    MaxAge: 86400
  Auth:
    Authorizers:
      CognitoAuthorizer:
        UserPoolArn: !Ref CognitoUserPoolArn
        Identity:
          Header: Authorization
    AddDefaultAuthorizerToCorsPreflight: false    # CRITICAL - allows OPTIONS without auth

# Then explicitly set auth on each protected function:
SomeProtectedFunction:
  Type: AWS::Serverless::Function
  Properties:
    Events:
      ApiEvent:
        Type: Api
        Properties:
          Path: /protected
          Method: GET
          RestApiId: !Ref Api
          Auth:
            Authorizer: CognitoAuthorizer  # Explicit per-function
```

## CORS Origin Strategy: JWT vs Cookie Authentication

### JWT Authentication (Cognito, Auth0, etc.) - USE WILDCARD
```yaml
# CORRECT for JWT in Authorization header
AllowOrigin: "'*'"
AllowCredentials: false
```

**Why wildcard is fine with JWT:**
- Security is enforced by JWT validation, not CORS
- No valid JWT = 401 Unauthorized (API Gateway rejects before Lambda)
- CORS is a browser enforcement mechanism, not server-side security
- Token is sent in `Authorization` header, not cookies
- Restricting origins adds complexity without security benefit

### Cookie Authentication (Session-based) - USE SPECIFIC ORIGINS
```yaml
# CORRECT for cookie-based auth only
AllowOrigin: "'https://your-app-domain.com'"
AllowCredentials: true
```

**When to use specific origins:**
- Session cookies require `AllowCredentials: true`
- `AllowCredentials: true` requires specific origin (not `*`)
- Only applies to cookie/session-based authentication

## Required CORS Headers

| Header | JWT Auth (Cognito) | Cookie Auth |
|--------|-------------------|-------------|
| **AllowOrigin** | `'*'` (wildcard OK) | Specific domain required |
| **AllowCredentials** | `false` | `true` |
| **AllowHeaders** | Include Authorization | Include Authorization |
| **AllowMethods** | All needed methods + OPTIONS | All needed methods + OPTIONS |

## Authentication Token Standards

### Frontend Token Storage
```typescript
// Store in localStorage for JWT auth
const token = session.getAccessToken().getJwtToken();
localStorage.setItem('token', token);

// Clear on logout
localStorage.removeItem('token');
```

### API Client Token Retrieval
```typescript
// Use localStorage directly
const token = localStorage.getItem('token');

// Add to API calls
fetch('/api/endpoint', {
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  }
});
```

## Pre-deployment Testing

**MANDATORY** CORS preflight test:

```bash
# This MUST return 200 OK before considering deployment complete
curl -X OPTIONS "https://your-api.amazonaws.com/dev/endpoint" \
  -H "Origin: https://any-origin.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: authorization,content-type" \
  -I
```

Expected response for JWT auth (Cognito):
- Status: `200 OK`
- `access-control-allow-origin: *`
- `access-control-allow-methods: GET, POST, PUT, DELETE, OPTIONS`
- `access-control-allow-headers: Content-Type, Authorization, ...`

**Note**: `access-control-allow-credentials` should be absent or `false` for JWT auth.

## Troubleshooting Guide

### Symptom: API calls return 401, no data loads
1. **Check CORS preflight**: Test OPTIONS request
2. **Check for DefaultAuthorizer**: Remove from SAM template
3. **Verify token storage**: Check browser localStorage
4. **Check browser console**: Look for CORS errors

### Symptom: "Missing Authentication Token"
- Usually indicates CORS preflight failure
- Browser never sends actual request with auth header
- Fix CORS configuration first, not auth code

### Symptom: "No 'Access-Control-Allow-Origin' header"
- API Gateway CORS configuration missing
- Check SAM template `Api.Cors` section
- Verify deployment actually applied changes

## Complete SAM Template Example

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Parameters:
  CognitoUserPoolArn:
    Type: String

Resources:
  Api:
    Type: AWS::Serverless::Api
    Properties:
      StageName: dev
      Cors:
        AllowMethods: "'GET, POST, PUT, DELETE, OPTIONS'"
        AllowHeaders: "'Content-Type, X-Amz-Date, Authorization, X-Api-Key'"
        AllowOrigin: "'*'"
        AllowCredentials: false
        MaxAge: 86400
      Auth:
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !Ref CognitoUserPoolArn
            Identity:
              Header: Authorization
        AddDefaultAuthorizerToCorsPreflight: false

  GetUsersFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./src/handlers/
      Handler: getUsers.handler
      Events:
        GetUsers:
          Type: Api
          Properties:
            Path: /users
            Method: GET
            RestApiId: !Ref Api
            Auth:
              Authorizer: CognitoAuthorizer  # Explicit authorization
```

## Summary

1. **NEVER use DefaultAuthorizer** - breaks CORS preflight
2. **Use explicit per-function authorization** - more control, no CORS issues
3. **Wildcard origin is fine for JWT auth** - security is in the token
4. **Always test OPTIONS preflight** - before declaring deployment complete
5. **Store JWT in localStorage** - not cookies for API calls
