# Core Development Principles

**Universal principles for building quality software**

---

## 1. No Mocks, No Fallback Data

**Rule:** Production code never hides failures with mock data or fallback values.

**Why:**
- Hidden failures never get fixed
- Mocks don't match reality - users see failures tests didn't catch

**Example:**
```javascript
// ❌ Bad - silent fallback to mock
async function getUserProfile(userId) {
    try {
        return await api.getUser(userId);
    } catch (error) {
        return { id: userId, name: 'Test User', email: 'test@example.com' }; // Hides failure
    }
}

// ✅ Good - fail loud
async function getUserProfile(userId) {
    try {
        return await api.getUser(userId);
    } catch (error) {
        console.error('❌ Failed to fetch user:', { userId, error: error.message });
        throw new Error(`Unable to load user profile: ${error.message}`);
    }
}
```

**When fallbacks OK:** Caching with explicit staleness markers and time limits.
```javascript
// Acceptable: Cached data with staleness warning
const cached = await cache.get(userId);
if (cached && cached.timestamp > Date.now() - 300000) {
    console.warn('⚠️ Using cached data due to API failure');
    return { ...cached.data, _fromCache: true, _stale: true };
}
throw new Error('User profile unavailable');
```

## 2. Fail Fast, Fail Loud

**Rule:** Make failures obvious and immediate. Don't silently degrade.

**Why:**
- Visible failures get fixed, hidden ones persist
- Fail at source, not 10 layers downstream

**Example:**
```javascript
// ❌ Bad - silent failure
function processOrder(order) {
    if (!order.items || order.items.length === 0) {
        console.log('Warning: No items, skipping');
        return { success: true };  // Silent failure
    }
    if (!order.customerId) {
        order.customerId = 'ANONYMOUS';  // Hiding problem
    }
    return processPayment(order);
}

// ✅ Good - fail fast and loud
function processOrder(order) {
    if (!order || typeof order !== 'object') {
        throw new Error('Invalid order: must be object');
    }
    if (!order.items || order.items.length === 0) {
        throw new Error('Invalid order: no items');
    }
    if (!order.customerId) {
        throw new Error('Invalid order: customerId required');
    }
    return processPayment(order);
}

// Fail loud with monitoring
console.error('❌ Payment failed', { orderId, error: error.message });
metrics.increment('payment.failures', { errorType: error.code });
```

## 3. Error-First Design

**Rule:** Design error states before implementing happy paths.

**Why:**
- Most production code paths are error handling
- Error states define UX quality more than happy paths

**Process:**
```
1. List all possible failures
2. Design specific error response for each
3. Define recovery strategies
THEN write happy path
```

**Example:**
```javascript
// ❌ Bad - happy path only
function transferMoney(from, to, amount) {
    deduct(from, amount);
    add(to, amount);
    return { success: true };
}
// What if insufficient funds? Invalid account? Negative amount? Partial failure?

// ✅ Good - errors designed first
/**
 * Possible failures:
 * - INSUFFICIENT_FUNDS, INVALID_ACCOUNT, INVALID_AMOUNT,
 * - ACCOUNT_LOCKED, TRANSACTION_FAILED
 */
function transferMoney(from, to, amount) {
    if (!from || !to) {
        return { success: false, error: 'INVALID_ACCOUNT', recovery: 'Verify account numbers' };
    }
    if (typeof amount !== 'number' || amount <= 0) {
        return { success: false, error: 'INVALID_AMOUNT', recovery: 'Enter valid amount' };
    }
    if (getBalance(from) < amount) {
        return {
            success: false,
            error: 'INSUFFICIENT_FUNDS',
            shortfall: amount - getBalance(from),
            recovery: 'Add funds or reduce amount'
        };
    }

    try {
        const tx = beginTransaction();
        deduct(from, amount, tx);
        add(to, amount, tx);
        tx.commit();
        return { success: true, transactionId: tx.id };
    } catch (error) {
        tx.rollback();
        return { success: false, error: 'TRANSACTION_FAILED', recovery: 'Try again' };
    }
}
```

## 4. Explicit Over Implicit

**Rule:** Code should be obvious, not clever.

**Why:** Future developers understand intent immediately, fewer bugs.

**Example:**
```javascript
// ❌ Implicit - clever but unclear
const results = data.map(x => x.y || defaults[x.type]?.y ?? fallback(x));

// ✅ Explicit - clear intent
const results = data.map(item => {
    if (item.y !== null && item.y !== undefined) return item.y;
    const typeDefault = defaults[item.type];
    if (typeDefault?.y !== undefined) return typeDefault.y;
    return fallback(item);
});
```

---

## 5. Consistent Error Handling

**Rule:** Use same error handling pattern throughout application.

**Why:** Predictable debugging, consistent logging, unified monitoring.

**Example:**
```javascript
function handleError(error, context) {
    console.error('❌ Error:', { message: error.message, code: error.code, context });
    monitoring.reportError(error, context);
    return { success: false, error: error.code || 'INTERNAL_ERROR', message: error.message };
}

// Use everywhere
try {
    return await riskyOperation();
} catch (error) {
    return handleError(error, { operation: 'riskyOperation', userId });
}
```

---

## 6. Developer Experience Matters

**Rule:** Optimize for humans who will maintain this code.

**Why:** Code is read 10x more than written.

**Guidelines:**
```javascript
// Specific error messages
throw new Error(`Invalid order: expected object with 'items' array, got ${typeof order}`);

// Context in logs
console.log('Processing payment:', { orderId, amount, customerId, paymentMethod });

// Document non-obvious decisions
// Using 5min cache TTL: user data changes infrequently, API limit 100 req/min
const CACHE_TTL = 300000;

// Request IDs for debugging
const requestId = generateRequestId();
console.log('Starting operation:', { requestId, userId });
```

---

## Summary

1. **No Mocks, No Fallback Data** → Failures visible and get fixed
2. **Fail Fast, Fail Loud** → Problems obvious and immediate
3. **Error-First Design** → Error states well-designed and recoverable
4. **Explicit Over Implicit** → Code clear and maintainable
5. **Consistent Error Handling** → Debugging predictable
6. **Developer Experience Matters** → Future maintainers say thank you

---

**Questions?** Contact info@happyhippo.ai
