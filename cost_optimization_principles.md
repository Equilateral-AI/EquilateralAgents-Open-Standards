# Cost Optimization Principles

**Universal patterns for building cost-effective systems**

---

## 1. Cost-First Infrastructure Design

**Rule:** Perform cost analysis during planning, before implementation.

**Why:**
- Prevents expensive surprises
- Catches expensive patterns early

**Process:**
```
1. Estimate monthly costs per component
2. Compare cost vs value
3. Document cost implications
4. Get approval for costs above threshold
THEN implement
```

**Example:**
```yaml
# ❌ Bad - deploy first, discover costs later
Database:
  DBInstanceClass: db.r5.24xlarge  # $13,000/month - no review

# ✅ Good - cost analysis before deployment
# COST ANALYSIS:
# - Dev: db.t3.micro @ $15/month
# - Prod: db.t3.small @ $30/month (start small)
# - Scale trigger: 80% CPU sustained > 1 hour
# - Annual: $360-$600

Database:
  DBInstanceClass: !If [IsProduction, db.t3.small, db.t3.micro]
  MultiAZ: !If [IsProduction, true, false]
```

## 2. Pay-Per-Use Default

**Rule:** Default to pay-per-use. Add fixed costs only when measured need exists.

**Why:**
- No monthly bills when not used
- Costs grow with usage/revenue

**Strategy:**
```
Start with pay-per-use
Monitor actual usage
IF usage consistent & high for 3+ months
THEN consider reserved capacity
```

**Examples:**
```yaml
# ❌ Bad - always-on EC2 for batch jobs
EC2Instance:
  InstanceType: t3.large  # $60/month even if used 1 hour/day

# ✅ Good - Lambda pay-per-invocation
BatchFunction:
  Runtime: python3.11
  # Cost: $0.20/hour when running → $6/month for 1 hour/day

# ❌ Bad - provisioned RDS always on
Database:
  DBInstanceClass: db.t3.small  # $30/month always

# ✅ Good - Aurora Serverless scales to zero
Database:
  Engine: aurora-postgresql
  ServerlessV2ScalingConfiguration:
    MinCapacity: 0.5  # Scales down when idle
  # Cost: ~$40/month prod, ~$5/month dev
```

**Switch to reserved when:**
- Running 24/7 for 30+ days
- Pay-per-use > reserved for 3+ months
- Usage predictable

## 3. Environment-Appropriate Sizing

**Rule:** Don't use production resources for development.

**Why:** Dev/staging can be 90% cheaper.

**Strategy:**
```
Development: Smallest tier that works, single-AZ, < $50/month
Staging: Small tier, production data subset
Production: Sized for actual load + headroom, multi-AZ
```

**Example:**
```yaml
# ❌ Bad - same size everywhere
Database:
  DBInstanceClass: db.r5.xlarge  # $365/month × 3 envs = $1,095/month

# ✅ Good - environment-appropriate
Database:
  DBInstanceClass: !If [IsProd, db.r5.xlarge, !If [IsStaging, db.t3.small, db.t3.micro]]
  MultiAZ: !If [IsProd, true, false]
  # Dev: $15/month, Staging: $30/month, Prod: $365/month
  # Total: $410/month (62% savings)
```

## 4. Cost Analysis Gates

**Rule:** Validate costs before deployment. Require approval above threshold.

**Why:** Catches expensive mistakes before production.

**Process:**
```
estimate_monthly_cost()
IF cost > threshold
THEN require_approval() + document_justification()
validate_environment_sizing()
THEN deploy
```

**Example CI/CD gate:**
```yaml
# GitHub Actions
cost-check:
  steps:
    - run: infracost breakdown --path . > costs.json
    - run: |
        monthly_cost=$(jq '.totalMonthlyCost' costs.json)
        if (( $(echo "$monthly_cost > 100" | bc -l) )); then
          echo "❌ Cost $monthly_cost exceeds threshold"
          exit 1
        fi
```

**Thresholds:**
- < $50/month: Auto-approve
- $50-200: Team lead approval
- $200-1000: Manager approval
- > $1000: Director + business justification

## 5. Right-Size for Actual Load

**Rule:** Size based on measured load, not guesses. Monitor and adjust continuously.

**Why:** Avoid over/under-provisioning, adapt to change.

**Process:**
```
Monitor actual usage
IF consistently < 20% utilized THEN downsize
IF consistently > 80% utilized THEN upsize
Document rationale
```

**Example:**
```
Initial: db.t3.small ($30/month)

After 30 days:
- CPU: 15% avg, 45% peak
- Storage: 3GB used / 20GB allocated
→ Action: Downsize to db.t3.micro ($15/month, 50% savings)

After 6 months:
- CPU: 65% avg, 92% peak
- Performance degrading during peaks
→ Action: Upsize to db.t3.medium ($60/month, needed for growth)
```

**Monitor:**
- Utilization (CPU, memory, storage)
- Performance (latency, throughput)
- Cost trends

**Review frequency:**
- New resources: Weekly for first month
- Established: Monthly
- Large costs (>$500/mo): Weekly

## 6. Eliminate Waste

**Rule:** Actively find and eliminate wasteful spending.

**Why:** Hidden waste accumulates quickly.

**Common waste sources:**
```bash
# 1. Orphaned resources (unattached EBS volumes)
aws ec2 describe-volumes --filters Name=status,Values=available
# Cost: $0.10/GB/month (50GB = $5/month wasted)

# 2. Over-provisioned (instances with <10% CPU)
# → Downsize or consider serverless

# 3. Forgotten test resources
# → Tag with auto-expiration: AutoDelete: "2025-10-30"

# 4. Excessive data transfer
# → Add CDN/caching, reduce cross-region calls
```

**Monthly review checklist:**
- [ ] Review all resources > $10/month
- [ ] Check for orphaned/unused resources
- [ ] Verify utilization (< 20% = candidate for downsizing)
- [ ] Review data transfer costs
- [ ] Check for forgotten test/dev resources

---

## Summary

1. **Cost-First Infrastructure Design** → Know costs before commit
2. **Pay-Per-Use Default** → Start variable, go fixed when proven
3. **Environment-Appropriate Sizing** → Don't use prod resources for dev
4. **Cost Analysis Gates** → Prevent expensive mistakes
5. **Right-Size for Actual Load** → Monitor and adjust
6. **Eliminate Waste** → Actively remove unnecessary costs

---

**Questions?** Contact info@happyhippo.ai
