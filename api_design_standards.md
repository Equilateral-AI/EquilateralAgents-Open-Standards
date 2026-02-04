# API Design Standards

**Technology-agnostic principles for building great APIs**

---

## 1. Consistent Response Formats

**Rule:** Standardize success and error responses across all endpoints.

**Why:**
- Same parsing logic works for all endpoints
- Know exactly where to find error details

**Format:**
```typescript
// Success
{ success: true, data: T, metadata?: { timestamp, requestId } }

// Error
{ success: false, error: { code, message, details? }, metadata?: { ... } }
```

**Example:**
```json
// ✅ Good - Consistent
{ "success": true, "data": { "userId": "user_123" }, "metadata": { "requestId": "req_abc" } }
{ "success": false, "error": { "code": "VALIDATION_ERROR", "message": "Invalid email" } }

// ❌ Bad - Inconsistent across endpoints
{ "user": {...} }
[...]
"ok"
```

## 2. Explicit Field Naming

**Rule:** Use clear, self-documenting field names. Avoid abbreviations.

**Why:** Reduces mistakes, fewer docs needed.

**Example:**
```json
// ❌ Bad - ambiguous
{ "id": "123", "status": "active", "date": "2025-10-02", "price": 99.99, "qty": 5 }

// ✅ Good - explicit
{
    "orderId": "order_123",
    "orderStatus": "active",
    "createdAt": "2025-10-02T12:00:00Z",
    "totalPriceUSD": 99.99,
    "quantity": 5
}
```

**Conventions:**
- Booleans: `isActive`, `hasPermission`, `canEdit`
- Dates: `createdAt`, `updatedAt`, `expiresAt`
- IDs: `userId`, `orderId` (not just `id`)
- Money: `priceUSD`, `totalEUR` or `amount` + `currency`

## 3. Proper Error Handling

**Rule:** Return specific errors with what/why/how to fix.

**Why:** Faster debugging, developers can self-serve fixes.

**Format:**
```json
{
    "success": false,
    "error": {
        "code": "CATEGORY_SPECIFIC_REASON",
        "message": "Human-readable explanation",
        "details": { "field": "...", "providedValue": "...", "expectedValue": "..." }
    }
}
```

**Example:**
```json
// ❌ Bad
{ "error": "Bad request" }

// ✅ Good
{
    "success": false,
    "error": {
        "code": "INVALID_EMAIL_FORMAT",
        "message": "Email must be user@domain.com format",
        "details": { "field": "email", "providedValue": "john.doe@" }
    }
}
```

**Error codes:**
```
VALIDATION_* (400) → Input validation
AUTH_* (401/403) → Authentication/authorization
NOT_FOUND_* (404) → Resource doesn't exist
CONFLICT_* (409) → State conflicts
RATE_LIMIT_* (429) → Rate limiting
INTERNAL_* (500) → Server errors
```

## 4. Versioning Strategy

**Rule:** Use explicit versioning from day one.

**Why:** Prevents breaking clients, enables iteration.

**Approaches:**
- URL path: `/v1/users`, `/v2/users` (explicit, clear in logs)
- Header: `Accept: application/vnd.api+json; version=1` (flexible)
- Query param: `/users?version=1` (easy to test)

**Guidelines:**
- Major version for breaking changes (format, removed fields, type changes)
- No version bump for additions (new optional fields, new endpoints)
- Document lifecycle: Released → Deprecated → Sunset dates
- Provide migration guide with code examples

## 5. Pagination Standards

**Rule:** Standardize pagination across all list endpoints.

**Format:**
```json
{
    "success": true,
    "data": {
        "items": [...],
        "pagination": {
            "page": 1,
            "pageSize": 20,
            "totalCount": 150,
            "hasNextPage": true
        }
    }
}
```

**Query params:** `?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc`

---

## 6. Request Validation

**Rule:** Validate all inputs. Return specific error per validation failure.

**Example:**
```json
{
    "success": false,
    "error": {
        "code": "VALIDATION_FAILED",
        "message": "Request validation failed",
        "details": {
            "errors": [
                { "field": "email", "code": "INVALID_FORMAT", "message": "Email must be valid" },
                { "field": "age", "code": "OUT_OF_RANGE", "message": "Age 0-120", "min": 0, "max": 120 }
            ]
        }
    }
}
```

---

## Summary

1. **Consistent Response Formats** → Predictable client code
2. **Explicit Field Naming** → Self-documenting
3. **Proper Error Handling** → Specific, actionable
4. **Versioning Strategy** → Evolve without breaking
5. **Pagination Standards** → Consistent list handling
6. **Request Validation** → Clear feedback

---

**Questions?** Contact info@happyhippo.ai
