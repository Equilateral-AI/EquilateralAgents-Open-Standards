# Codex Runtime Control Standards

**Version**: 1.0.0
**Status**: Production
**Last Updated**: 2025-10-06
**Audience**: Open Source / Public

## Purpose

Define how to control Codex/AI agent runtime access, network connectivity, and operational boundaries through configuration files that the harness enforces at execution time.

## Critical Distinction

### Documentation vs Runtime Control

| File | Type | Purpose | Effect |
|------|------|---------|--------|
| `.equilateralrules/manifest.json` | **Documentation** | Standards reference for agents | None (agents read it, but harness ignores it) |
| `.codexrc.json` | **Runtime Config** | Authoritative runtime settings | Controls what harness allows |
| `harness.policy.json` | **Enforcement Policy** | Sandbox-layer network gates | Enforced at socket/process level |

**KEY PRINCIPLE**: The harness reads `.codexrc.json` and `harness.policy.json` at startup. These files are the **source of truth** for what Codex can do.

**Common Mistake**: Developers often update the standards manifest thinking it will change runtime behavior, but only the runtime config files actually control the harness.

## Runtime Configuration Architecture

### 1. `.codexrc.json` - Runtime Environment Configuration

**Location**: Project root
**Read by**: Codex CLI harness at process start
**Schema**: JSON with strict validation

#### Required Structure

```json
{
  "$schema": "https://example.com/schemas/codexrc.schema.json",
  "environment_context": {
    "name": "dev|sandbox|production",
    "network_access": "enabled|restricted|disabled",
    "outbound_policy": {
      "mode": "allowlist|denylist|open",
      "dns_allow": ["api.example.com", "db.example.com"],
      "ip_allow": [],
      "port_allow": [80, 443]
    },
    "telemetry": {
      "trace": true|false,
      "redact_headers": ["authorization", "x-api-key", "cookie"]
    }
  },
  "credentials": {
    "aws_profile": "profile-name",
    "aws_region": "us-east-1",
    "secrets": {
      "SECRET_NAME": "arn:aws:secretsmanager:region:account:secret:name"
    }
  },
  "environments": {
    "dev": { /* environment-specific config */ },
    "sandbox": { /* environment-specific config */ },
    "production": { /* environment-specific config */ }
  },
  "limits": {
    "http_max_requests": 200,
    "http_concurrency": 8,
    "timeout_seconds": 120
  }
}
```

#### Field Definitions

##### `environment_context.network_access`

**Type**: Enum (`"enabled"`, `"restricted"`, `"disabled"`)
**Default**: `"restricted"`
**Effect**: Controls whether harness allows outbound network connections

- **`"enabled"`**: Full network access (subject to outbound_policy)
- **`"restricted"`**: Only localhost and internal services
- **`"disabled"`**: No network access at all

**Example**:
```json
{
  "environment_context": {
    "network_access": "enabled"  // Required for external API calls
  }
}
```

##### `environment_context.outbound_policy.mode`

**Type**: Enum (`"allowlist"`, `"denylist"`, `"open"`)
**Default**: `"allowlist"`
**Recommendation**: Always use `"allowlist"` for production systems

- **`"allowlist"`**: Only specified domains/IPs allowed (fail-loud on unknown)
- **`"denylist"`**: All except specified domains/IPs allowed
- **`"open"`**: All network access allowed (development only, high security risk)

**Example**:
```json
{
  "outbound_policy": {
    "mode": "allowlist",
    "dns_allow": [
      "api.example.com",
      "auth.example.com",
      "*.trusted-service.com"
    ],
    "port_allow": [443]  // HTTPS only
  }
}
```

##### `credentials`

**Purpose**: Define cloud provider credentials and region
**Security**: ARNs/references only, never actual secrets (fetched at runtime)

**Example**:
```json
{
  "credentials": {
    "aws_profile": "default",
    "aws_region": "us-east-1",
    "secrets": {
      "DB_PASSWORD_ARN": "arn:aws:secretsmanager:us-east-1:123456:secret:db-pwd"
    }
  }
}
```

##### `limits`

**Purpose**: Prevent runaway resource consumption

**Example**:
```json
{
  "limits": {
    "http_max_requests": 200,     // Max HTTP requests per session
    "http_concurrency": 8,         // Max concurrent HTTP requests
    "timeout_seconds": 120         // Max request timeout
  }
}
```

### 2. `harness.policy.json` - Enforcement Policy

**Location**: Project root
**Read by**: Sandbox runner
**Purpose**: Enforced at socket/process level (fail-loud)

#### Required Structure

```json
{
  "version": "1.0",
  "network": {
    "enabled": true|false,
    "egress": {
      "mode": "allowlist|denylist|open",
      "domains": ["api.example.com", "db.example.com"],
      "ports": [80, 443]
    }
  },
  "observability": {
    "http_log_bodies": false,
    "http_log_headers": false,
    "sample_rate": 1.0
  },
  "deny_on_violation": true|false
}
```

#### Field Definitions

##### `network.enabled`

**Type**: Boolean
**Default**: `false`
**Effect**: Hard gate at sandbox level

- **`true`**: Sandbox allows network syscalls
- **`false`**: Sandbox blocks all network syscalls

**Must match** `.codexrc.json` `network_access: "enabled"`

##### `deny_on_violation`

**Type**: Boolean
**Default**: `true`
**Recommendation**: Always `true` (fail-loud)

- **`true`**: Block and throw error on policy violation
- **`false`**: Log warning but allow (insecure, development only)

### 3. Launcher Script Pattern

**Purpose**: Validate configuration before launching Codex
**Pattern**: Pre-flight checks + environment injection

#### Standard Launcher Template

```javascript
#!/usr/bin/env node
import { spawn } from "node:child_process";
import { readFileSync } from "node:fs";
import path from "node:path";

const projectRoot = process.cwd();
const codexrcPath = path.join(projectRoot, ".codexrc.json");
const policyPath = path.join(projectRoot, "harness.policy.json");

// Load and validate configs
const rc = JSON.parse(readFileSync(codexrcPath, "utf8"));
const policy = JSON.parse(readFileSync(policyPath, "utf8"));

// Pre-flight validation
if (rc?.environment_context?.network_access !== "enabled") {
  console.error("✖ Network not enabled in .codexrc.json");
  process.exit(2);
}

if (!policy?.network?.enabled) {
  console.error("✖ Network not enabled in harness.policy.json");
  process.exit(2);
}

console.log("✓ Runtime configuration validated");
console.log(`  Environment: ${rc.environment_context.name}`);
console.log(`  Network: ${rc.environment_context.network_access}`);

// Compose environment variables
const env = {
  ...process.env,
  CODEX_NETWORK_ACCESS: "enabled",
  CODEX_POLICY_FILE: policyPath,
  CODEX_RUNTIME_CONFIG: codexrcPath,
  // Cloud provider credentials (if applicable)
  AWS_PROFILE: rc.credentials?.aws_profile,
  AWS_REGION: rc.credentials?.aws_region
};

// Launch Codex with configuration
const child = spawn("codex", [
  "session",
  "--network", "enabled",
  "--policy", policyPath
], { env, stdio: "inherit" });

child.on("exit", (code) => process.exit(code ?? 1));
```

**Usage**:
```bash
chmod +x scripts/run-codex.mjs
./scripts/run-codex.mjs
```

## Access Control Patterns

### Pattern 1: Development Environment (Permissive)

**Use Case**: Local development, rapid iteration
**Security Posture**: Moderate
**Network**: Broad allowlist with wildcards acceptable

```json
{
  "environment_context": {
    "name": "dev",
    "network_access": "enabled",
    "outbound_policy": {
      "mode": "allowlist",
      "dns_allow": [
        "*.example.com",
        "*.cloudprovider.com",
        "localhost",
        "127.0.0.1"
      ],
      "port_allow": [80, 443, 5432, 3000]
    }
  },
  "limits": {
    "http_max_requests": 500,
    "http_concurrency": 10,
    "timeout_seconds": 180
  }
}
```

**harness.policy.json**:
```json
{
  "network": {
    "enabled": true,
    "egress": {
      "mode": "allowlist",
      "domains": ["*.example.com", "*.cloudprovider.com"]
    }
  },
  "deny_on_violation": false  // Development: log warnings
}
```

### Pattern 2: Sandbox Environment (Balanced)

**Use Case**: Integration testing, staging, pilot deployments
**Security Posture**: Production-like
**Network**: Explicit domains, minimal wildcards

```json
{
  "environment_context": {
    "name": "sandbox",
    "network_access": "enabled",
    "outbound_policy": {
      "mode": "allowlist",
      "dns_allow": [
        "api.sandbox.example.com",
        "db.sandbox.example.com",
        "auth.example.com"
      ],
      "port_allow": [443, 5432]
    }
  },
  "limits": {
    "http_max_requests": 200,
    "http_concurrency": 8,
    "timeout_seconds": 120
  }
}
```

**harness.policy.json**:
```json
{
  "network": {
    "enabled": true,
    "egress": {
      "mode": "allowlist",
      "domains": [
        "api.sandbox.example.com",
        "db.sandbox.example.com",
        "auth.example.com"
      ],
      "ports": [443, 5432]
    }
  },
  "deny_on_violation": true  // Fail-loud
}
```

### Pattern 3: Production Environment (Strict)

**Use Case**: Production workloads
**Security Posture**: Maximum security
**Network**: Explicit domains only, no wildcards

```json
{
  "environment_context": {
    "name": "production",
    "network_access": "enabled",
    "outbound_policy": {
      "mode": "allowlist",
      "dns_allow": [
        "api.example.com",           // No wildcards
        "db.prod-region.provider.com",
        "auth.example.com"
      ],
      "port_allow": [443]              // HTTPS only
    }
  },
  "limits": {
    "http_max_requests": 100,
    "http_concurrency": 4,
    "timeout_seconds": 60
  }
}
```

**harness.policy.json**:
```json
{
  "network": {
    "enabled": true,
    "egress": {
      "mode": "allowlist",
      "domains": [
        "api.example.com",
        "db.prod-region.provider.com",
        "auth.example.com"
      ],
      "ports": [443]
    }
  },
  "observability": {
    "http_log_bodies": false,        // Compliance: no sensitive data logs
    "http_log_headers": false,
    "sample_rate": 0.1               // 10% sampling for performance
  },
  "deny_on_violation": true          // Production: fail-loud always
}
```

### Pattern 4: Air-Gapped / Offline Mode

**Use Case**: Security-critical systems, compliance requirements
**Security Posture**: Maximum isolation
**Network**: Localhost only

```json
{
  "environment_context": {
    "name": "air-gapped",
    "network_access": "restricted",    // Only localhost
    "outbound_policy": {
      "mode": "allowlist",
      "dns_allow": ["localhost", "127.0.0.1"],
      "port_allow": [3000, 5432, 8080]
    }
  },
  "limits": {
    "http_max_requests": 50,
    "http_concurrency": 2,
    "timeout_seconds": 30
  }
}
```

**harness.policy.json**:
```json
{
  "network": {
    "enabled": false                   // Sandbox blocks all external network
  },
  "deny_on_violation": true
}
```

## Runtime Context Propagation

### Pattern: Handler Wrapper for Runtime Context

**Purpose**: Inject runtime context into every handler execution
**Applies to**: Lambda functions, serverless handlers, agent tasks

```javascript
// runtimeWrapper.js
const fs = require('fs');
const path = require('path');

/**
 * Load runtime configuration from .codexrc.json
 */
function loadRuntimeConfig() {
    const rcPath = process.env.CODEX_RUNTIME_CONFIG ||
                   path.join(process.cwd(), '.codexrc.json');

    try {
        if (fs.existsSync(rcPath)) {
            return JSON.parse(fs.readFileSync(rcPath, 'utf8'));
        }
    } catch (error) {
        console.warn('Failed to load .codexrc.json:', error.message);
    }

    // Fallback to environment variables
    return {
        environment_context: {
            name: process.env.CODEX_CONTEXT_NAME || 'dev',
            network_access: process.env.CODEX_NETWORK_ACCESS || 'restricted'
        }
    };
}

/**
 * Validate runtime environment has network access enabled
 */
function validateNetworkAccess(rc) {
    if (rc.environment_context.network_access !== 'enabled') {
        throw new Error(
            `Network disabled by runtime harness (network_access=${rc.environment_context.network_access}). ` +
            `Update .codexrc.json or launch with validated launcher script.`
        );
    }
}

/**
 * Wrap handler with runtime environment context
 */
function wrapWithRuntimeContext(handler, options = {}) {
    const rc = loadRuntimeConfig();

    // Validate network access if required
    if (options.requireNetwork) {
        validateNetworkAccess(rc);
    }

    return async (event, context) => {
        // Inject runtime environment into context
        context.runtimeEnvironment = {
            name: rc.environment_context.name,
            network_access: rc.environment_context.network_access,
            outbound_policy: rc.environment_context.outbound_policy
        };

        // Get environment-specific configuration
        const envConfig = rc.environments?.[rc.environment_context.name];
        if (envConfig) {
            context.environmentConfig = envConfig;
        }

        // Log runtime context (useful for debugging)
        console.log('Runtime Environment:', {
            name: context.runtimeEnvironment.name,
            network_access: context.runtimeEnvironment.network_access
        });

        // Call the wrapped handler with enhanced context
        return handler(event, context);
    };
}

module.exports = {
    loadRuntimeConfig,
    validateNetworkAccess,
    wrapWithRuntimeContext
};
```

**Usage in Handler**:
```javascript
const { wrapWithRuntimeContext } = require('./runtimeWrapper');

async function myHandler(event, context) {
    // Access runtime configuration
    console.log('Environment:', context.runtimeEnvironment.name);
    console.log('Network Access:', context.runtimeEnvironment.network_access);

    // Use environment-specific settings
    const apiUrl = context.environmentConfig?.api_url;

    // Handler logic...
    return { statusCode: 200, body: 'Success' };
}

// Export wrapped handler (requires network enabled)
exports.handler = wrapWithRuntimeContext(myHandler, {
    requireNetwork: true
});
```

## Validation & Testing

### Pre-Deployment Validation Checklist

```bash
#!/bin/bash
# validate-codex-config.sh

echo "🔍 Validating Codex runtime configuration..."

# 1. Validate JSON syntax
echo "→ Checking JSON syntax..."
cat .codexrc.json | python3 -m json.tool > /dev/null || exit 1
cat harness.policy.json | python3 -m json.tool > /dev/null || exit 1
echo "✓ JSON syntax valid"

# 2. Verify network access alignment
echo "→ Checking network access alignment..."
CODEXRC_NETWORK=$(jq -r '.environment_context.network_access' .codexrc.json)
POLICY_NETWORK=$(jq -r '.network.enabled' harness.policy.json)

if [ "$CODEXRC_NETWORK" = "enabled" ] && [ "$POLICY_NETWORK" = "true" ]; then
    echo "✓ Network access aligned"
elif [ "$CODEXRC_NETWORK" != "enabled" ] && [ "$POLICY_NETWORK" = "false" ]; then
    echo "✓ Network disabled (aligned)"
else
    echo "✖ Network access mismatch!"
    echo "  .codexrc.json: $CODEXRC_NETWORK"
    echo "  harness.policy.json: $POLICY_NETWORK"
    exit 1
fi

# 3. Check allowlist consistency
echo "→ Checking allowlist consistency..."
CODEXRC_DOMAINS=$(jq -r '.environment_context.outbound_policy.dns_allow[]' .codexrc.json 2>/dev/null | sort)
POLICY_DOMAINS=$(jq -r '.network.egress.domains[]' harness.policy.json 2>/dev/null | sort)

if [ "$CODEXRC_DOMAINS" = "$POLICY_DOMAINS" ]; then
    echo "✓ Allowlist domains match"
else
    echo "⚠ Allowlist domains differ (may be intentional)"
fi

# 4. Verify deny_on_violation for production
ENV_NAME=$(jq -r '.environment_context.name' .codexrc.json)
DENY_ON_VIOLATION=$(jq -r '.deny_on_violation' harness.policy.json)

if [ "$ENV_NAME" = "production" ] && [ "$DENY_ON_VIOLATION" != "true" ]; then
    echo "✖ Production must have deny_on_violation: true"
    exit 1
fi

echo "✓ All validation checks passed"
```

### Smoke Test Pattern

**Purpose**: Verify network connectivity to allowed endpoints

```javascript
// smoke-test.js
const https = require('https');
const { loadRuntimeConfig } = require('./runtimeWrapper');

function testEndpoint(name, url) {
    return new Promise((resolve) => {
        https.get(url, (res) => {
            const ok = res.statusCode === 200 || res.statusCode === 403;
            console.log(`${ok ? '✓' : '✖'} ${name}: ${res.statusCode}`);
            resolve(ok);
        }).on('error', (err) => {
            console.error(`✖ ${name}: ${err.message}`);
            resolve(false);
        });
    });
}

async function runSmokeTests() {
    console.log('🚀 Network Connectivity Smoke Tests\n');

    const rc = loadRuntimeConfig();

    if (rc.environment_context.network_access !== 'enabled') {
        console.error('✖ Network not enabled in configuration');
        process.exit(1);
    }

    console.log(`Environment: ${rc.environment_context.name}`);
    console.log(`Network Access: ${rc.environment_context.network_access}\n`);

    const results = [];

    // Test configured endpoints
    for (const [envName, envConfig] of Object.entries(rc.environments || {})) {
        if (envConfig.api_url) {
            results.push(await testEndpoint(`${envName} API`, envConfig.api_url));
        }
    }

    const passed = results.filter(r => r).length;
    const failed = results.filter(r => !r).length;

    console.log(`\n📊 Results: ${passed} passed, ${failed} failed`);
    process.exit(failed === 0 ? 0 : 1);
}

runSmokeTests();
```

**Usage**:
```bash
node smoke-test.js
```

## Security Best Practices

### 1. Always Use Allowlist Mode in Production

❌ **NEVER** (security risk):
```json
{
  "outbound_policy": {
    "mode": "open"  // Allows any network access
  }
}
```

✅ **ALWAYS**:
```json
{
  "outbound_policy": {
    "mode": "allowlist",
    "dns_allow": ["api.example.com", "db.example.com"]
  }
}
```

### 2. Fail-Loud on Violations

❌ **AVOID** (hides problems):
```json
{
  "deny_on_violation": false  // Logs warning, allows violation
}
```

✅ **PREFER**:
```json
{
  "deny_on_violation": true  // Throws error, blocks violation
}
```

### 3. Never Store Secrets in Runtime Config

❌ **NEVER**:
```json
{
  "credentials": {
    "api_key": "sk-1234567890abcdef",      // SECURITY VIOLATION
    "db_password": "actual-password-here"  // SECURITY VIOLATION
  }
}
```

✅ **ALWAYS** (use secret references):
```json
{
  "credentials": {
    "secrets": {
      "API_KEY_ARN": "arn:aws:secretsmanager:region:account:secret:api-key",
      "DB_PASSWORD_ARN": "arn:aws:secretsmanager:region:account:secret:db-pwd"
    }
  }
}
```

### 4. Explicit Domains in Production (No Wildcards)

❌ **AVOID** in production:
```json
{
  "dns_allow": ["*.example.com"]  // Too broad
}
```

✅ **PREFER** in production:
```json
{
  "dns_allow": [
    "api.example.com",
    "auth.example.com",
    "db.us-east-1.example.com"
  ]
}
```

### 5. Progressive Enhancement by Environment

**Development**: Permissive for velocity
- Wildcards acceptable
- `deny_on_violation: false` (warnings)
- Broad port access

**Sandbox**: Production-like for validation
- Minimal wildcards
- `deny_on_violation: true` (errors)
- Limited ports (443, specific services)

**Production**: Strict for security
- No wildcards
- `deny_on_violation: true` (errors)
- HTTPS only (port 443)

## Common Pitfalls & Troubleshooting

### Issue: "Network disabled by runtime harness"

**Symptom**: Error message about `network_access=restricted`

**Root Cause**: `.codexrc.json` doesn't have `network_access: "enabled"`

**Solution**:
```json
// .codexrc.json
{
  "environment_context": {
    "network_access": "enabled"
  }
}
```

### Issue: "Connection refused" or "ENOTFOUND"

**Symptom**: Cannot reach specific endpoint

**Root Cause**: Domain not in allowlist

**Solution**: Add domain to BOTH files:
```json
// .codexrc.json
{
  "outbound_policy": {
    "dns_allow": ["new-endpoint.example.com"]
  }
}

// harness.policy.json
{
  "network": {
    "egress": {
      "domains": ["new-endpoint.example.com"]
    }
  }
}
```

### Issue: Mismatch Between Files

**Symptom**: Configuration works locally but not in CI/CD

**Root Cause**: `.codexrc.json` and `harness.policy.json` out of sync

**Solution**: Use validation script before deployment:
```bash
./scripts/validate-codex-config.sh
```

### Issue: "Policy file not found"

**Symptom**: Launcher can't find `harness.policy.json`

**Root Cause**: Environment variable not set or wrong path

**Solution**:
```javascript
const env = {
  CODEX_POLICY_FILE: path.resolve(__dirname, "../harness.policy.json")
};
```

## Change Management

### Adding New Endpoint

1. **Identify**: Get full domain (e.g., `new-service.example.com`)
2. **Update `.codexrc.json`**: Add to `dns_allow` array
3. **Update `harness.policy.json`**: Add to `domains` array
4. **Validate**: Run validation script
5. **Test**: Run smoke tests
6. **Document**: Add to environment-specific config in manifest

### Changing Environment

1. **Update `.codexrc.json`**: Change `environment_context.name`
2. **Verify credentials**: Check cloud provider credentials exist
3. **Update allowlist**: Ensure domains match new environment
4. **Test**: Run smoke tests against new environment
5. **Deploy**: Restart Codex with new configuration

### Promoting Configuration

**Development → Sandbox**:
- Review allowlist, remove unnecessary wildcards
- Enable `deny_on_violation: true`
- Test thoroughly

**Sandbox → Production**:
- Remove ALL wildcards
- Review every domain (principle of least privilege)
- Reduce rate limits if appropriate
- Test failover scenarios

## Standards Compliance

These standards align with core development principles:

- **Fail Fast, Fail Loudly**: `deny_on_violation: true`
- **No Mocks, No Fallback**: Real network or fail
- **Error-First Design**: Network validation before handler execution
- **Cost-First Infrastructure**: Explicit allowlist prevents accidental expensive calls
- **Security by Default**: Allowlist mode, fail-loud violations

## Version History

- **1.0.0** (2025-10-06): Initial standards for Codex runtime control

---

**License**: Open Source
**Maintainer**: Equilateral AI Standards
**Repository**: https://github.com/equilateral-open-standards
