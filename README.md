# EquilateralAgents Open Standards

**Universal Development Principles for Quality Software**

A curated collection of battle-tested development standards that work across any technology stack, framework, or language. These standards represent fundamental principles for building reliable, maintainable software.

**MIT Licensed - Use in any project**

---

## YAML Format (New)

All standards are now available in **machine-readable YAML format** in the `yaml/` directory. This enables:

- **[MindMeld.dev](https://mindmeld.dev)** integration - automatic injection of relevant standards into AI coding sessions
- **AI agent consumption** - Claude, Cursor, Copilot can parse rules directly without regex extraction
- **Automated validation** - lint/check tools consume YAML natively

```yaml
# Example: yaml/development-principles.yaml
id: development-principles
category: core
priority: 10
rules:
  - action: NEVER
    rule: "Return mock data or fallback values from production code on failure"
  - action: ALWAYS
    rule: "Fail fast and loud -- make failures obvious and immediate"
anti_patterns:
  - "Returning mock/default objects in catch blocks to hide API failures"
```

The original markdown standards remain as human-readable reference. YAML is the canonical format for tooling.

**Schema**: See [equilateral-standards-yaml/SCHEMA.yaml](https://github.com/Equilateral-AI/equilateral-standards-yaml/blob/main/SCHEMA.yaml)

---

## 🎯 Philosophy

Good software development isn't about following trends—it's about applying timeless principles that lead to quality outcomes. These standards represent lessons learned from building production systems at scale.

**Core Beliefs:**
- **Fail Fast, Fail Loud** - Make problems obvious immediately
- **No Mocks, No Fallbacks** - Real failures teach real lessons
- **Errors Are First-Class** - Design error states before happy paths
- **Explicit Over Implicit** - Code should be obvious, not clever
- **Cost-Conscious Design** - Always consider resource implications

---

## 📚 Standards Included

### Core Development Principles
Fundamental principles that apply to all software development:
- Error-first design methodology
- Fail-fast architecture patterns
- No mock data / no fallback philosophy
- Explicit error handling standards
- Developer experience optimization

### Cost Optimization Principles
Universal patterns for resource-efficient software:
- Cost-first infrastructure planning
- Pay-per-use default patterns
- Resource sizing strategies
- Environment-appropriate architecture
- Cost analysis gates

### API Design Standards
Technology-agnostic API design principles:
- Consistent error handling
- Standard response formats
- Field naming conventions
- Versioning strategies
- Documentation requirements

### Testing Principles
Fundamental testing approaches:
- Error state validation
- Real integration testing (no mocks)
- Test data coherence
- Failure scenario coverage
- Performance baseline testing

### AI Assistant Configuration
Configuring AI coding assistants for automatic enforcement:
- Personality prompts with war stories
- Role-based framing
- Specific examples (good vs bad)
- Living documentation
- Behavior verification

### Serverless Standards
AWS serverless patterns extracted from production:
- Lambda database connection management
- API Gateway CORS configuration

### Well-Architected Standards
AWS Well-Architected Framework applied to real systems:
- Cost Optimization, Reliability, Security
- Operational Excellence, Performance Efficiency, Sustainability

### Agent & Runtime Standards
AI agent and runtime control patterns:
- Agent memory and context management
- Codex runtime control and sandboxing

### YAML Directory Structure

```
yaml/
  development-principles.yaml
  api-design-standards.yaml
  cost-optimization-principles.yaml
  testing-principles.yaml
  ai-assistant-configuration.yaml
  agent-memory-standards.yaml
  codex-runtime-control-standards.yaml
  cpp-geometric-algorithms-standards.yaml
  serverless/
    api-gateway-cors-standards.yaml
    lambda-database-standards.yaml
  well-architected/
    cost-optimization.yaml
    operational-excellence.yaml
    performance-efficiency.yaml
    reliability.yaml
    security.yaml
    sustainability.yaml
```

---

## 🚀 Quick Start

### Step 1: Add Standards to Your Project

```bash
# Option A: Add as git submodule (recommended)
git submodule add https://github.com/Equilateral-AI/EquilateralAgents-Open-Standards.git .standards

# Option B: Clone directly
git clone https://github.com/Equilateral-AI/EquilateralAgents-Open-Standards.git

# Option C: Use with MindMeld (automatic injection)
# Sign up at https://mindmeld.dev - standards are injected into your AI coding sessions automatically
```

### Step 2: Configure Your AI Coding Assistant

**Make standards enforcement automatic** by adding a "personality prompt" to your AI assistant:

```bash
# Copy the template to your project
cp .standards/CLAUDE.md.template ./CLAUDE.md

# Or for Cursor
cp .standards/CLAUDE.md.template ./.cursorrules

# Customize for your project (add project-specific rules)
# Your AI assistant will now automatically follow standards!
```

**What This Does:**
- AI assistant reads standards before every code change
- Catches violations early (mocks, vague errors, silent failures)
- Enforces error-first design automatically
- References existing patterns instead of reinventing

**Supported AI Assistants:**
- Claude Code (CLAUDE.md)
- Cursor (.cursorrules)
- Continue (.continuerc.json)
- GitHub Copilot (via .github/copilot-instructions.md)
- Any assistant that reads project context files

### Step 3: Reference in Documentation

Add to your project's README or CONTRIBUTING guide:

```markdown
## Development Standards

This project follows the [EquilateralAgents Open Standards](./.standards):

- **No Mock Data** - Failures must be visible for root cause fixes
- **Fail Fast, Fail Loud** - Make failures obvious and immediate
- **Error-First Design** - Design error states before happy path
- **Cost-Conscious** - Consider resource implications in design

See [CLAUDE.md](./CLAUDE.md) for AI assistant configuration.
```

---

## 📖 Standards Overview

### 1. Core Development Principles
**File:** [development_principles.md](./development_principles.md)

Fundamental principles for quality software development:

- **No Mocks, No Fallback Data** - Production code should never hide failures with fallback values
- **Fail Fast, Fail Loud** - Make failures obvious and immediate, don't mask problems
- **Consistent Error Handling** - Use established patterns across entire application
- **Error-First Design** - Design error states before implementing happy path
- **Explicit Over Implicit** - Prefer obvious code over clever abstractions
- **Developer Experience Matters** - Optimize for the humans maintaining the code

**When to Use:** Every project, every language, every framework.

---

### 2. Cost Optimization Principles
**File:** [cost_optimization_principles.md](./cost_optimization_principles.md)

Universal patterns for building cost-effective systems:

- **Cost-First Infrastructure Design** - Always perform cost analysis during planning
- **Pay-Per-Use Default** - Only add fixed costs when measured need exists
- **Environment-Appropriate Sizing** - Dev templates for dev, production for production
- **Cost Analysis Gates** - Validate costs before deployment
- **Resource Right-Sizing** - Match resources to actual needs

**When to Use:** Any project with infrastructure costs (cloud, SaaS, compute).

---

### 3. API Design Standards
**File:** [api_design_standards.md](./api_design_standards.md)

Technology-agnostic principles for building great APIs:

- **Consistent Response Formats** - Standardize success and error responses
- **Explicit Field Naming** - Use clear, self-documenting field names
- **Proper Error Handling** - Return meaningful errors with context
- **Versioning Strategy** - Plan for API evolution from day one
- **Documentation First** - Document endpoints before implementation

**When to Use:** Building any API (REST, GraphQL, gRPC, etc.).

---

### 4. Testing Principles
**File:** [testing_principles.md](./testing_principles.md)

Fundamental approaches to effective testing:

- **Test Error States First** - Failures are more important than success cases
- **No Mocks in Integration Tests** - Test against real integrations
- **Test Data Coherence** - Test data should reflect real-world relationships
- **Failure Scenario Coverage** - Test what happens when things go wrong
- **Performance Baselines** - Establish and monitor performance expectations

**When to Use:** Every project that needs to be maintained.

---

### 5. AI Assistant Configuration
**File:** [ai_assistant_configuration.md](./ai_assistant_configuration.md)

Principles for configuring AI coding assistants to automatically follow standards:

- **Use Personality Prompts** - Context about standards, conventions, war stories
- **Frame as a Role** - "Quality-focused developer" or "Technical team lead"
- **Include Specific Examples** - Good vs bad code, specific patterns
- **Choose Right Format** - CLAUDE.md, .cursorrules, etc.
- **Make It Living Documentation** - Update when learning new patterns
- **Balance Detail with Readability** - Specific but readable (500-800 lines)
- **Test and Iterate** - Verify behavior actually changes

**When to Use:** Any project using AI coding assistants (Claude Code, Cursor, Copilot, etc.).

**Template Available:** [CLAUDE.md.template](./CLAUDE.md.template) - Copy and customize for your project.

---

## 🎓 Philosophy in Practice

### Example: The "No Mocks" Principle

**Bad Practice (Hidden Failures):**
```javascript
// API call fails, falls back to mock data
async function fetchUserData(userId) {
    try {
        return await api.getUser(userId);
    } catch (error) {
        console.log('API failed, using mock');
        return { id: userId, name: 'Test User' };  // ❌ Masks the real problem
    }
}
```

**Good Practice (Fail Fast, Fail Loud):**
```javascript
// API call fails, failure is obvious
async function fetchUserData(userId) {
    try {
        return await api.getUser(userId);
    } catch (error) {
        console.error('❌ Failed to fetch user:', error);
        throw new Error(`User data unavailable: ${error.message}`);  // ✅ Obvious failure
    }
}
```

**Why This Matters:**
- Mock data hides API problems until production
- Developers get false confidence that features work
- Root causes are never fixed because failures are invisible
- Users see broken features when mocks don't match reality

---

### Example: Error-First Design

**Bad Practice (Happy Path First):**
```javascript
// Designed for success, errors are afterthought
function processPayment(amount) {
    const result = chargeCard(amount);
    return { success: true, transactionId: result.id };
    // What about failures? ❌
}
```

**Good Practice (Errors Designed First):**
```javascript
// Designed with failures in mind
function processPayment(amount) {
    // What can go wrong?
    // - Insufficient funds
    // - Invalid card
    // - Network timeout
    // - Fraud detection trigger

    try {
        const result = chargeCard(amount);
        return {
            success: true,
            transactionId: result.id
        };
    } catch (error) {
        // Explicit error handling for each case
        if (error.code === 'INSUFFICIENT_FUNDS') {
            return {
                success: false,
                error: 'INSUFFICIENT_FUNDS',
                message: 'Card declined: insufficient funds',
                userAction: 'Try a different payment method'
            };
        }
        // ... handle other specific errors
    }
}
```

**Why This Matters:**
- Error states are the majority of code paths in production
- Users experience errors, not happy paths
- Well-designed errors enable recovery
- Specific errors enable better UX

---

## 🔧 How to Use These Standards

### 1. Team Onboarding
Include standards in your onboarding documentation:

```markdown
## Code Quality Standards

We follow the EquilateralAgents Open Standards for development:

1. **No Mock Data** - [Read why](./standards/development_principles.md#no-mocks)
2. **Fail Fast** - [Read why](./standards/development_principles.md#fail-fast)
3. **Error-First Design** - [Read why](./standards/development_principles.md#error-first)
```

### 2. Code Review Checklist
Reference standards in PR templates:

```markdown
## PR Checklist

- [ ] Follows fail-fast principle (no silent failures)
- [ ] No mock/fallback data in production code
- [ ] Error states designed and tested
- [ ] Cost implications reviewed
```

### 3. Architecture Decisions
Use standards as decision framework:

```markdown
## Architecture Decision: User Data Caching

**Standards Applied:**
- Cost Optimization: Cache reduces API calls 80%
- Error-First Design: Cache miss strategy defined
- Fail Fast: Cache errors don't fall back to stale data

**Decision:** Implement Redis cache with explicit error handling
```

---

## 🤝 Contributing

These standards are deliberately minimal and universal. Contributions should:

- ✅ Apply to any technology stack
- ✅ Be based on production experience
- ✅ Focus on principles, not implementations
- ✅ Include practical examples
- ❌ Not be framework-specific
- ❌ Not be language-specific
- ❌ Not be vendor-specific

**To contribute:** Submit PR with standard + real-world examples

---

## 📄 License

MIT License - Use freely in any project, commercial or open source.

**Built with ❤️ by HappyHippo.ai**

**Questions?** Contact info@happyhippo.ai

---

## 🔗 Related Projects

- **[EquilateralAgents Open Core](https://github.com/Equilateral-AI/equilateral-agents-open-core)** - 22 production-ready AI agents implementing these standards
- **EquilateralAgents Enterprise** - 60+ specialized agents with advanced capabilities

**Using these standards?** We'd love to hear about it! Contact info@happyhippo.ai
