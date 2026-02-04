# Testing Principles

**Fundamental approaches to effective testing**

---

## 1. Test Error States First

**Rule:** Prioritize testing failure scenarios over happy paths.

**Why:**
- Most code execution goes through error paths
- Untested error paths fail in production

**Process:**
```
1. List all error conditions
2. Test each error condition
3. Verify specific error codes
4. Test error recovery
THEN test happy path
```

**Example:**
```javascript
// ❌ Bad - only happy path
it('should transfer money', () => {
    expect(transferMoney('a1', 'a2', 100).success).toBe(true);
});

// ✅ Good - errors first
describe('transferMoney errors', () => {
    it('rejects insufficient funds', () => {
        const result = transferMoney('a1', 'a2', 9999999);
        expect(result.error.code).toBe('INSUFFICIENT_FUNDS');
    });
    it('rejects invalid account', () => {
        expect(transferMoney('a1', 'invalid', 100).error.code).toBe('INVALID_ACCOUNT');
    });
    it('rollsback on partial failure', async () => {
        mockAdd.mockRejectedValue(new Error('Network error'));
        const before = getBalance('a1');
        await transferMoney('a1', 'a2', 100);
        expect(getBalance('a1')).toBe(before); // Rolled back
    });
});
// THEN test happy path
```

## 2. No Mocks in Integration Tests

**Rule:** Integration tests use real dependencies. Unit tests use mocks.

**Why:**
- Mocks diverge from real service behavior
- Integration bugs missed (real API changes break code, mocks don't)

**The Rule:**
```
Unit Tests:     Mock external dependencies
Integration:    Real databases, real APIs, real services
E2E:           Everything real, including UI
```

**Example:**
```javascript
// ❌ Bad - mocked "integration"
mockDatabase.insert.mockResolvedValue({ userId: 'user_123' });
mockEmailService.send.mockResolvedValue({ sent: true });
await registerUser({ email: 'test@example.com' });
// Just tests mocks were called

// ✅ Good - real integration
beforeAll(async () => {
    testDatabase = await createTestDatabase();
    testEmailServer = await startTestEmailServer();
});

it('registers user in real database and sends real email', async () => {
    await registerUser({ email: 'test@example.com', password: 'pass123' });

    // Verify in real database
    const userInDb = await testDatabase.query('SELECT * FROM users WHERE email = ?', ['test@example.com']);
    expect(userInDb.passwordHash).toBeDefined();

    // Verify real email sent
    const emails = await testEmailServer.getEmails();
    expect(emails[0].to).toBe('test@example.com');
});
```

**When mocks OK:** External services (payment gateways, expensive APIs) - but also test against sandbox.

## 3. Test Data Coherence

**Rule:** Test data should reflect real-world relationships and constraints.

**Why:**
- Unrealistic test data misses real bugs
- Real data tests foreign keys and validations

**Example:**
```javascript
// ❌ Bad - incoherent
const testOrder = {
    customerId: 'customer_999',  // Doesn't exist
    items: [{ productId: 'product_abc', quantity: -5, price: 0 }],  // Invalid
    total: 1000,  // Doesn't match
    updatedAt: '2020-01-01',
    createdAt: '2025-01-01'  // Updated before created?
};

// ✅ Good - coherent
beforeEach(async () => {
    testCustomer = await createTestCustomer({ email: 'test@example.com' });
    testProduct = await createTestProduct({ price: 29.99, inventory: 100 });
});

it('processes valid order', async () => {
    const order = await createOrder({
        customerId: testCustomer.customerId,  // Real reference
        items: [{ productId: testProduct.productId, quantity: 2 }]
    });

    expect(order.total).toBe(59.98);
    expect(await getProduct(testProduct.productId).inventory).toBe(98); // Decreased
});
```

## 4. Failure Scenario Coverage

**Rule:** Test what happens when dependencies fail, networks timeout, resources exhausted.

**Why:** Features work != features handle failures gracefully.

**Scenarios:**
- Dependency failures (database down, API unavailable)
- Network issues (timeouts, connection drops)
- Resource exhaustion (disk full, rate limits)
- Data issues (corrupt data, missing fields)

**Example:**
```javascript
it('handles database timeout', async () => {
    mockDatabase.query.mockImplementation(() =>
        new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000))
    );
    expect((await getUserProfile('user_123')).error.code).toBe('DATABASE_TIMEOUT');
});

it('falls back to database on cache failure', async () => {
    mockCache.get.mockRejectedValue(new Error('Cache unavailable'));
    const result = await getUserProfile('user_123');
    expect(result.success).toBe(true);
    expect(mockDatabase.query).toHaveBeenCalled();
});
```

## 5. Performance Baseline Testing

**Rule:** Establish and monitor performance baselines for critical operations.

**Why:** Detect degradation early, verify SLA compliance.

**Example:**
```javascript
it('loads user profile in < 100ms', async () => {
    const start = Date.now();
    await getUserProfile('user_123');
    expect(Date.now() - start).toBeLessThan(100);
});

it('handles 100 concurrent requests in < 5s', async () => {
    const start = Date.now();
    await Promise.all(Array.from({ length: 100 }, (_, i) => getUserProfile(`user_${i}`)));
    expect(Date.now() - start).toBeLessThan(5000);
});
```

**Test with:**
- Realistic data volumes (100, 10K, 100K+ records)
- Concurrent load (sequential, 10, 100, 1000 users)
- Monitor resource usage (memory, CPU, connections)

---

## Summary

1. **Test Error States First** → Most code execution is error handling
2. **No Mocks in Integration Tests** → Real services catch real bugs
3. **Test Data Coherence** → Realistic data reveals real problems
4. **Failure Scenario Coverage** → Test when things break
5. **Performance Baseline Testing** → Know what "normal" looks like

---

**Questions?** Contact info@happyhippo.ai
