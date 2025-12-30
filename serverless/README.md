# Serverless Standards

Patterns and anti-patterns for AWS Lambda and API Gateway applications.

## Standards

| Standard | Description |
|----------|-------------|
| [lambda_database_standards.md](lambda_database_standards.md) | Database connection patterns for Lambda - SSM, pooling, caching |
| [api_gateway_cors_standards.md](api_gateway_cors_standards.md) | CORS configuration and the DefaultAuthorizer anti-pattern |

## Key Principles

### Lambda Database Connections
- **NEVER** fetch SSM parameters at runtime ($25/month per million invocations)
- **NEVER** use connection pools in Lambda (useless overhead)
- **ALWAYS** use environment variables resolved at deployment
- **ALWAYS** cache a single database client across warm invocations

### API Gateway CORS
- **NEVER** use DefaultAuthorizer (breaks CORS preflight)
- **ALWAYS** use explicit per-function authorization
- **ALWAYS** test OPTIONS preflight before declaring deployment complete
- Wildcard origin is fine for JWT authentication

## Cost Impact

Following these standards can save significant money:

| Anti-Pattern | Monthly Cost (1M invocations) |
|--------------|-------------------------------|
| Runtime SSM fetching | $25 |
| Runtime Secrets Manager | $40 |
| Connection pools | +$5 (memory overhead) |

| Correct Pattern | Monthly Cost |
|-----------------|--------------|
| Environment variables | $0 |
| Cached single client | $0 |

## Related Standards

- [Well-Architected Framework](../well-architected/) - AWS best practices
- [Cost Optimization Principles](../cost_optimization_principles.md) - General cost guidance
