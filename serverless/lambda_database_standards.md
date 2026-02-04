# Lambda Database Connection Standards

**Core Principle**: Lambda functions require specific database patterns that minimize cost and maximize performance. These patterns are fundamentally different from traditional server applications.

## MANDATORY PATTERNS - NO EXCEPTIONS

### 1. Configuration: ALWAYS Use Environment Variables

**CORRECT - Resolved at deployment, zero runtime cost**
```yaml
# SAM Template - Environment Variables
Environment:
  Variables:
    DB_HOST: '{{resolve:ssm:app-db-host:1}}'
    DB_NAME: '{{resolve:ssm:app-db-name:1}}'
    DB_USER: '{{resolve:ssm:app-db-user:1}}'
    DB_PASSWORD: '{{resolve:ssm-secure:app-db-pass:1}}'
    DB_PORT: '{{resolve:ssm:app-db-port:1}}'
```

**WRONG - Fetches every invocation, costs money**
```yaml
# NEVER DO THIS
Environment:
  Variables:
    SSM_DB_HOST_PARAM: 'app-db-host'  # Then fetch in code
```

```javascript
// ANTI-PATTERN - Runtime SSM Fetching
const ssm = new AWS.SSM();
const param = await ssm.getParameter({Name: 'app-db-host'}).promise();
// This costs $25/month per million invocations!
```

### 2. Connections: NEVER Use Pools in Lambda

**CORRECT - Single cached client**
```javascript
let client = null;
let isConnected = false;

async function getClient() {
  if (client && isConnected) {
    try {
      await client.query('SELECT 1');
      return client;
    } catch (e) {
      isConnected = false;
    }
  }

  client = new Client({
    host: process.env.DB_HOST,  // Free, instant
    database: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    port: process.env.DB_PORT
  });

  await client.connect();
  isConnected = true;
  return client;
}
```

**WRONG - Pool adds overhead with no benefit**
```javascript
// NEVER DO THIS IN LAMBDA
const pool = new Pool({
  max: 10,  // Lambda can't use 10 connections!
  min: 2    // Unnecessary in Lambda environment
});
```

### 3. Why This Matters

#### Cost Impact
- **Runtime SSM fetch**: $25/month per million invocations
- **Environment variables**: $0
- **Connection pool overhead**: Wasted memory allocation
- **New connection per request**: Unnecessary latency

#### Performance Impact
- **SSM fetch**: +50-200ms latency per invocation
- **Environment variable**: 0ms (already in memory)
- **Connection pool overhead**: +memory, +complexity, zero benefit
- **Cached client reuse**: Saves 20-100ms on warm starts

### 4. Lambda Fundamentals

These are core Lambda concepts that drive our database patterns:

- **One container = one request at a time** - Lambda containers handle requests sequentially
- **Pools are for concurrency** - Lambda has none per container
- **Warm containers reuse globals** - Cache your database client
- **Environment variables are free and instant** - Always use them
- **Cold starts are inevitable** - Optimize for both cold and warm

## The Standard Pattern

### dbClient.js

```javascript
const { Client } = require('pg');

let client = null;
let isConnected = false;

async function getClient() {
  // Reuse existing connection if valid
  if (client && isConnected) {
    try {
      await client.query('SELECT 1');
      return client;
    } catch (e) {
      console.log('Connection lost, reconnecting...');
      isConnected = false;
    }
  }

  // Create new connection
  client = new Client({
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    port: process.env.DB_PORT || 5432,
    // Lambda-optimized settings
    connectionTimeoutMillis: 5000,
    query_timeout: 10000,
    statement_timeout: 10000,
    idle_in_transaction_session_timeout: 10000
  });

  await client.connect();
  isConnected = true;
  console.log('Database connected successfully');
  return client;
}

async function executeQuery(sql, params = []) {
  const dbClient = await getClient();
  try {
    const result = await dbClient.query(sql, params);
    return result;
  } catch (error) {
    console.error('Query execution error:', error);
    // Reset connection on certain errors
    if (error.code === '57P01' || error.code === 'ECONNRESET') {
      isConnected = false;
    }
    throw error;
  }
}

async function executeTransaction(queries) {
  const dbClient = await getClient();
  try {
    await dbClient.query('BEGIN');
    const results = [];
    for (const { sql, params } of queries) {
      results.push(await dbClient.query(sql, params));
    }
    await dbClient.query('COMMIT');
    return results;
  } catch (error) {
    await dbClient.query('ROLLBACK');
    throw error;
  }
}

module.exports = {
  executeQuery,
  executeTransaction,
  getClient
};
```

### Lambda Handler Pattern

```javascript
const { executeQuery } = require('./dbClient');

exports.handler = async (event) => {
  try {
    const result = await executeQuery(
      'SELECT * FROM users WHERE company_id = $1',
      [event.queryStringParameters?.companyId]
    );

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        records: result.rows,
        count: result.rowCount
      })
    };
  } catch (error) {
    console.error('Handler error:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' })
    };
  }
};
```

## Anti-Patterns to Fix Immediately

### Anti-Pattern 1: Runtime SSM/Secrets Fetching
```javascript
// NEVER DO THIS - Costs money and adds latency
const getDbConfig = async () => {
  const ssm = new AWS.SSM();
  const params = await Promise.all([
    ssm.getParameter({Name: 'db-host'}).promise(),
    ssm.getParameter({Name: 'db-password', WithDecryption: true}).promise()
  ]);
  return { host: params[0].Value, password: params[1].Value };
};
```

### Anti-Pattern 2: Connection Pools
```javascript
// NEVER DO THIS - Pools are useless in Lambda
const pool = new Pool({
  max: 20,           // Lambda can't use 20 connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});
```

### Anti-Pattern 3: New Connection Every Request
```javascript
// NEVER DO THIS - Wastes time on every invocation
exports.handler = async (event) => {
  const client = new Client(config);
  await client.connect();  // Cold start penalty every time!
  const result = await client.query('SELECT NOW()');
  await client.end();      // Throwing away warm connection!
  return result;
};
```

### Anti-Pattern 4: Not Handling Connection Errors
```javascript
// INCOMPLETE - No connection validation
let client = new Client(config);
await client.connect();
// Using client without checking if still connected
```

## Cost Comparison Table

| Pattern | Monthly Cost (1M invocations) | Latency | Recommendation |
|---------|-------------------------------|---------|----------------|
| Environment Variables | $0 | 0ms | ALWAYS USE |
| Runtime SSM Fetch | $25 | +50-200ms | NEVER USE |
| Runtime Secrets Manager | $40 | +100-300ms | NEVER USE |
| Connection Pool | +$5 (memory) | +10ms | NEVER USE |
| New Connection/Request | +$10 (compute) | +20-100ms | NEVER USE |
| Cached Single Client | $0 | 0ms (warm) | ALWAYS USE |

## SAM Template Configuration

### Required Environment Variables
```yaml
Globals:
  Function:
    Runtime: nodejs18.x
    Environment:
      Variables:
        # PostgreSQL Configuration (resolved at deployment)
        DB_HOST: '{{resolve:ssm:app-db-host:1}}'
        DB_NAME: '{{resolve:ssm:app-db-name:1}}'
        DB_USER: '{{resolve:ssm:app-db-user:1}}'
        DB_PASSWORD: '{{resolve:ssm-secure:app-db-pass:1}}'
        DB_PORT: '{{resolve:ssm:app-db-port:1}}'
        # Application Configuration
        NODE_ENV: !Ref Environment
        AWS_NODEJS_CONNECTION_REUSE_ENABLED: 1
```

### IAM Permissions
```yaml
Policies:
  - Version: '2012-10-17'
    Statement:
      # NO SSM permissions needed - resolved at deployment!
      - Effect: Allow
        Action:
          - 'rds:DescribeDBInstances'  # Only if using RDS
        Resource: '*'
```

## Migration Checklist

For existing Lambda functions not following these patterns:

- [ ] **Remove all runtime SSM/Secrets Manager calls**
- [ ] **Add environment variables to SAM template** with `{{resolve:ssm:}}`
- [ ] **Replace connection pools with single cached client**
- [ ] **Implement connection validation** in getClient()
- [ ] **Remove client.end() calls** - keep connections alive
- [ ] **Add connection error handling** with reconnection logic
- [ ] **Update IAM policies** - remove SSM permissions
- [ ] **Test cold and warm starts** - verify connection reuse

## Summary

**Every Lambda function touching a database MUST:**
1. Use environment variables resolved at deployment time
2. Cache a single database client (never use pools)
3. Reuse connections across warm invocations
4. Never fetch configuration at runtime
5. Handle connection errors with reconnection logic

**This is not optional. This is the standard.**

---

**Remember**: In Lambda, simplicity wins. One cached client, environment variables, and proper error handling. That's it.
