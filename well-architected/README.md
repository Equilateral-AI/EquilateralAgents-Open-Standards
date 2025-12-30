# AWS Well-Architected Framework Standards

Actionable guidance mapped to the AWS Well-Architected Framework pillars for building secure, high-performing, resilient, and efficient serverless applications.

## Pillars

| Pillar | Document | Key Focus |
|--------|----------|-----------|
| Operational Excellence | [operational-excellence.md](operational-excellence.md) | Observability, IaC, automation, runbooks |
| Security | [security.md](security.md) | IAM, encryption, secrets, OWASP compliance |
| Reliability | [reliability.md](reliability.md) | Multi-AZ, retries, circuit breakers, graceful degradation |
| Performance Efficiency | [performance-efficiency.md](performance-efficiency.md) | Right-sizing, caching, async patterns |
| Cost Optimization | [cost-optimization.md](cost-optimization.md) | Pay-per-use, ARM64, efficient queries |
| Sustainability | [sustainability.md](sustainability.md) | Carbon footprint, efficient compute, cold storage |

## Usage

1. **During Design**: Review relevant pillars before architecture decisions
2. **During Development**: Follow patterns linked in each pillar
3. **During Review**: Use as checklist for Well-Architected reviews
4. **For Audits**: Map your implementation to AWS-recognized best practices

## AWS Well-Architected Tool Integration

These standards align with the AWS Well-Architected Tool questions. Use them to:
- Prepare for Well-Architected reviews
- Identify improvement areas
- Document compliance with best practices

## Related Standards

- `serverless/` - Lambda and API Gateway patterns
- Root standards - Development principles, testing, cost optimization
