# Agent Memory Standards

**Version**: 1.0.0
**Date**: 2025-10-25
**Status**: Active
**Scope**: EquilateralAgents Open Core + Enterprise

## Overview

Agent memory enables self-learning capabilities by tracking execution history, identifying patterns, and optimizing performance over time. This standard defines how to implement and use agent memory correctly across open-core and enterprise deployments.

## Core Principles

### 1. Memory is Opt-In by Default (Enabled)

Agents have memory enabled by default, but can opt-out:

```javascript
// Memory enabled (default)
const agent = new SecurityAgent({ agentId: 'security-1' });

// Memory disabled (explicit)
const agent = new SecurityAgent({ enableMemory: false });
```

**Rationale**: Most agents benefit from learning. Explicit opt-out prevents surprises.

### 2. Single-Agent Memory (Open-Core)

Open-core memory is **isolated per agent**:
- ✅ Each agent learns from its own executions
- ❌ No cross-agent pattern sharing
- ❌ No multi-agent coordination
- ✅ File-based persistence (`.agent-memory/`)

**Storage Location**:
```
.agent-memory/
├── security-agent/
│   └── memory.json
├── deployment-agent/
│   └── memory.json
└── code-analyzer/
    └── memory.json
```

### 3. Short-Term Memory Only (Open-Core)

Open-core maintains **last 100 executions**:
- Older executions are automatically pruned
- Patterns based on recent history only
- No cumulative long-term learning
- File-based storage (not database)

**Enterprise Difference**: Unlimited history, database-backed, long-term pattern evolution.

## Implementation Standards

### Recording Executions

**Method 1: Automatic Recording** (Recommended)

Use `executeTaskWithMemory()` for automatic tracking:

```javascript
class SecurityAgent extends BaseAgent {
    async executeTask(task) {
        // Your logic here
        const findings = await this.scanCode(task.files);
        return { findings };
    }
}

// Usage
const agent = new SecurityAgent();
const result = await agent.executeTaskWithMemory({
    taskType: 'security-scan',
    description: 'Scan auth handlers',
    files: ['src/handlers/auth.js']
});

// Memory automatically records:
// - Task type
// - Duration
// - Success/failure
// - Result
```

**Method 2: Manual Recording**

For fine-grained control:

```javascript
class DeploymentAgent extends BaseAgent {
    async executeTask(task) {
        const startTime = Date.now();

        try {
            const result = await this.deploy(task.target);

            // Record success
            this.recordExecution(task, {
                success: true,
                duration: Date.now() - startTime,
                result: {
                    deployed: result.endpoint,
                    cost: result.cost
                }
            });

            return result;
        } catch (error) {
            // Record failure
            this.recordExecution(task, {
                success: false,
                duration: Date.now() - startTime,
                error: error.message
            });

            throw error;
        }
    }
}
```

### Using Memory Insights

**Before Execution: Check Success Patterns**

```javascript
// Query past experience
const patterns = agent.getSuccessPatterns('security-scan');

if (patterns && patterns.successRate < 0.7) {
    console.warn(`Low success rate (${patterns.successRate}) for this task type`);
    console.warn(`Consider alternative approach or review failure patterns`);
}
```

**Before Execution: Get Workflow Suggestions**

```javascript
const suggestion = agent.suggestOptimalWorkflow({
    taskType: 'deployment',
    environment: 'production'
});

console.log(`Estimated duration: ${suggestion.estimatedDuration}ms`);
console.log(`Confidence: ${(suggestion.confidence * 100).toFixed(0)}%`);
console.log(`Based on: ${suggestion.basedOn} prior executions`);

if (suggestion.confidence < 0.5) {
    console.log('Low confidence - consider manual review');
}
```

**After Execution: Track Performance**

```javascript
const metrics = agent.getMemoryMetrics();

console.log(`Success rate: ${(metrics.successRate * 100).toFixed(1)}%`);
console.log(`Avg duration: ${metrics.avgDuration}ms`);

if (metrics.improvement) {
    console.log(`Trend: ${metrics.improvement.trend}`);
    console.log(`Speed improvement: ${metrics.improvement.durationImprovement.toFixed(1)}%`);
}
```

### Memory Management

**Viewing Memory Stats**

```javascript
const stats = agent.getMemoryStats();

console.log(`Agent: ${stats.agentId}`);
console.log(`Executions tracked: ${stats.executionCount}/${stats.maxExecutions}`);
console.log(`Success rate: ${(stats.patterns.successRate * 100).toFixed(1)}%`);
console.log(`Memory size: ${(stats.memorySize / 1024).toFixed(1)} KB`);
```

**Resetting Memory**

```javascript
// Reset when codebase changes significantly
agent.resetMemory();
console.log('Memory reset - agent will relearn patterns');
```

**Exporting/Importing Memory**

```javascript
// Export for backup or migration
const backup = agent.memory.export();
fs.writeFileSync('agent-memory-backup.json', JSON.stringify(backup));

// Import from backup
const data = JSON.parse(fs.readFileSync('agent-memory-backup.json'));
agent.memory.import(data);
```

## Integration with .equilateral-standards

### Standards Repository Pattern

Agent memory follows `.equilateral-standards` patterns:

**Cost Optimization Integration**:
```javascript
// From .standards/cost_optimization_principles.md
// Record cost data in memory for optimization

agent.recordExecution(task, {
    success: true,
    duration: 12000,
    result: {
        deployed: true,
        cost: { lambda: 0.12, gateway: 0.03 } // Track costs
    }
});

// Later, optimize based on cost patterns
const patterns = agent.getSuccessPatterns('deploy-lambda');
console.log(`Avg cost: $${patterns.avgCost}`);
```

**Development Principles Integration**:
```javascript
// From .standards/development_principles.md
// Memory supports iterative refinement

// 1. First deployment (learning)
await agent.executeTaskWithMemory({ taskType: 'deploy', env: 'dev' });

// 2. Agent learns from execution
const suggestion = agent.suggestOptimalWorkflow({ taskType: 'deploy' });

// 3. Apply learned patterns to next deployment
if (suggestion.confidence > 0.8) {
    console.log(`High confidence - using learned patterns`);
}
```

**Testing Principles Integration**:
```javascript
// From .standards/testing_principles.md
// Track test execution patterns

agent.recordExecution(
    { taskType: 'run-tests', suite: 'integration' },
    {
        success: true,
        duration: 45000,
        result: { passed: 127, failed: 0 }
    }
);

// Identify slow tests
const testPatterns = agent.getSuccessPatterns('run-tests');
if (testPatterns.avgDuration > 60000) {
    console.warn('Integration tests averaging over 1 minute');
}
```

## Open-Core vs Enterprise Features

### Open-Core (Included)

✅ **Single-Agent Memory**
- Each agent learns independently
- File-based persistence
- Last 100 executions
- Pattern recognition
- Success rate tracking
- Performance metrics

**Use Cases**:
- Individual development (1-3 projects)
- Non-critical workflows
- Learning curve optimization
- Performance tracking

### Enterprise Only

🔒 **Multi-Agent Coordination**
- Cross-agent pattern sharing
- Team-based orchestration
- Privacy-preserved synthesis

🔒 **Long-Term Memory**
- Unlimited execution history
- Database-backed (PostgreSQL)
- Cross-project learning
- Pattern evolution tracking

🔒 **Advanced Features**
- Semantic search (150x faster)
- Patent-protected isolation
- Audit trails
- Continuous learning (24/7)

**Use Cases**:
- Enterprise scale (5+ projects)
- Regulated industries (GDPR, HIPAA)
- Mission-critical workflows
- Team collaboration

## Best Practices

### DO

✅ Use `executeTaskWithMemory()` for automatic tracking
✅ Check `suggestOptimalWorkflow()` before complex tasks
✅ Track costs and performance metrics in memory
✅ Reset memory when codebase changes significantly
✅ Export memory before major refactorings (backup)
✅ Review patterns regularly to identify bottlenecks

### DON'T

❌ Store sensitive data in memory (passwords, keys, tokens)
❌ Rely on memory for critical decision-making without validation
❌ Assume patterns from <10 executions are reliable
❌ Ignore low confidence scores
❌ Disable memory without good reason
❌ Expect cross-agent learning in open-core (enterprise only)

## Performance Considerations

### Memory Footprint

**Open-Core**: ~50 KB per agent (100 executions)
- 22 agents = ~1.1 MB total
- File-based, minimal overhead
- No database required

**Enterprise**: Database-dependent
- Unlimited history
- PostgreSQL indexes
- Query optimization required

### File System Impact

**Location**: `.agent-memory/` (gitignore recommended)

**.gitignore**:
```
# Agent memory (local learning only)
.agent-memory/
```

**Rationale**: Memory is specific to local codebase patterns. Don't commit to git unless team wants shared learning (then use enterprise with database).

### When to Reset Memory

Reset memory when:
- Codebase architecture changes significantly
- Switching between projects/branches
- Agent success rate drops unexpectedly
- Testing with different configurations

```javascript
// Example: Reset before test suite
beforeEach(() => {
    agent.resetMemory();
});
```

## Upgrade Path to Enterprise

### Migrating to Enterprise Memory

**Step 1: Export Open-Core Memory**

```bash
node -e "
const agent = require('./agents/SecurityAgent');
const backup = agent.memory.export();
require('fs').writeFileSync('memory-export.json', JSON.stringify(backup));
"
```

**Step 2: Install Enterprise**

```bash
npm install @equilateral/agents-enterprise
```

**Step 3: Import Patterns**

```bash
npm run agents:import-memory ./memory-export.json
```

**Step 4: Verify Migration**

```bash
npm run agents:status
# All agents should show historical patterns preserved
```

### Feature Comparison

| Feature | Open-Core | Enterprise |
|---------|-----------|------------|
| **Memory Type** | Single-agent | Multi-agent + synthesis |
| **Storage** | File-based | Database (PostgreSQL) |
| **History Limit** | 100 executions | Unlimited |
| **Pattern Recognition** | Basic | Semantic search |
| **Cross-Project** | No | Yes |
| **Cost** | Free | Paid |

## Compliance & Security

### Data Privacy

**Open-Core**:
- Memory stored locally in `.agent-memory/`
- No external transmission
- File permissions control access
- Recommended: Add to `.gitignore`

**Enterprise**:
- Database access controls
- Encryption at rest
- Audit trails
- GDPR/HIPAA compliant (when configured)

### Sensitive Data Handling

**DO NOT** store in memory:
- API keys, passwords, tokens
- PII (personally identifiable information)
- Secrets or credentials
- Unencrypted sensitive data

**Safe to Store**:
- Task types and descriptions (sanitized)
- Duration metrics
- Success/failure status
- Performance patterns
- Cost estimates

**Example: Sanitizing Task Data**

```javascript
// Bad: Exposes API key
agent.recordExecution(
    { taskType: 'deploy', apiKey: 'sk-1234567890' }, // ❌
    { success: true, duration: 10000 }
);

// Good: Sanitized
agent.recordExecution(
    { taskType: 'deploy', target: 'production' }, // ✅
    { success: true, duration: 10000 }
);
```

## Testing Agent Memory

### Unit Tests

```javascript
const BaseAgent = require('../equilateral-core/BaseAgent');

describe('Agent Memory', () => {
    let agent;

    beforeEach(() => {
        agent = new BaseAgent({ agentId: 'test-agent' });
    });

    afterEach(() => {
        agent.resetMemory();
    });

    test('records successful execution', async () => {
        agent.recordExecution(
            { taskType: 'test', description: 'Sample task' },
            { success: true, duration: 100 }
        );

        const patterns = agent.getSuccessPatterns('test');
        expect(patterns).not.toBeNull();
        expect(patterns.successCount).toBe(1);
    });

    test('suggests optimal workflow after learning', () => {
        // Record 10 successful executions
        for (let i = 0; i < 10; i++) {
            agent.recordExecution(
                { taskType: 'deploy' },
                { success: true, duration: 5000 + Math.random() * 1000 }
            );
        }

        const suggestion = agent.suggestOptimalWorkflow({ taskType: 'deploy' });
        expect(suggestion.hasExperience).toBe(true);
        expect(suggestion.confidence).toBeGreaterThan(0.5);
    });
});
```

## Monitoring & Debugging

### Logging Memory Stats

```javascript
// Log memory stats periodically
setInterval(() => {
    const stats = agent.getMemoryStats();
    console.log(`[${stats.agentId}] Executions: ${stats.executionCount}, Success: ${(stats.patterns.successRate * 100).toFixed(1)}%`);
}, 60000); // Every minute
```

### Debugging Low Success Rates

```javascript
const patterns = agent.getSuccessPatterns('problematic-task');

if (patterns && patterns.successRate < 0.5) {
    console.log('Failure patterns:', agent.memory.patterns.failurePatterns);

    // Analyze recent failures
    const recentFailures = Array.from(agent.memory.executions.values())
        .filter(e => !e.outcome.success && e.task.type === 'problematic-task')
        .slice(-5);

    console.log('Recent failures:', recentFailures);
}
```

## Version History

- **1.0.0** (2025-10-25): Initial release
  - SimpleAgentMemory implementation
  - Integration with BaseAgent
  - Open-core vs enterprise distinction
  - .equilateral-standards alignment

## References

- [EquilateralAgents Open Core README](../README.md)
- [Cost Optimization Principles](./cost_optimization_principles.md)
- [Development Principles](./development_principles.md)
- [Testing Principles](./testing_principles.md)
- [Agent Memory Guide](../AGENT_MEMORY_GUIDE.md)

## Questions?

- **Documentation**: https://docs.equilateral.ai/memory
- **Enterprise Features**: https://equilateral.ai/enterprise
- **Support**: support@equilateral.ai
