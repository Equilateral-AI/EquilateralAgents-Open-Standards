# C++ Geometric Algorithms Standards

**Version**: 1.0.0
**Author**: Equilateral AI
**License**: Apache-2.0
**Repository**: https://github.com/Equilateral-AI/equilateral-open-standards

## Overview

This standard defines best practices for implementing geometric algorithms in C++, with focus on correctness, performance, and numerical stability.

## Core Principles

1. **Boundary Inclusion**: Be explicit about boundary handling
2. **Numerical Stability**: Handle floating-point edge cases
3. **Degenerate Cases**: Always consider zero-size/zero-length inputs
4. **Coordinate Systems**: Document assumptions (screen vs world, 2D vs 3D)
5. **Test Coverage**: Every edge case must have a test

---

## Pattern: Point-in-Shape Testing

### Inclusive vs Exclusive Boundaries

**Problem**: Does a point on the boundary count as "inside"?

**Decision**: Use **inclusive** boundaries (≤, ≥) by default for containment tests.

**Rationale**:
- Matches user expectations (clicking edge of button should work)
- Prevents gaps in spatial partitioning (quadtrees, grids)
- Consistent with mathematical definitions (closed sets)

**Exception**: Use exclusive boundaries when explicitly modeling "open" regions.

### Implementation

```cpp
// ✅ CORRECT: Inclusive boundary test
bool Rectangle::contains(const Vec2& point) const {
    return point.x >= minX && point.x <= maxX &&
           point.y >= minY && point.y <= maxY;
}

// ❌ WRONG: Exclusive boundaries (creates edge case bugs)
bool Rectangle::contains(const Vec2& point) const {
    return point.x > minX && point.x < maxX &&
           point.y > minY && point.y < maxY;
}
```

### Test Cases Required

```cpp
TEST(Rectangle, ContainmentTests) {
    Rectangle rect(0, 0, 100, 100);

    // Interior point
    EXPECT_TRUE(rect.contains(Vec2(50, 50)));

    // Boundary points (edges)
    EXPECT_TRUE(rect.contains(Vec2(0, 50)));   // left edge
    EXPECT_TRUE(rect.contains(Vec2(100, 50))); // right edge
    EXPECT_TRUE(rect.contains(Vec2(50, 0)));   // top edge
    EXPECT_TRUE(rect.contains(Vec2(50, 100))); // bottom edge

    // Corner points
    EXPECT_TRUE(rect.contains(Vec2(0, 0)));       // top-left
    EXPECT_TRUE(rect.contains(Vec2(100, 0)));     // top-right
    EXPECT_TRUE(rect.contains(Vec2(100, 100)));   // bottom-right
    EXPECT_TRUE(rect.contains(Vec2(0, 100)));     // bottom-left

    // Exterior points
    EXPECT_FALSE(rect.contains(Vec2(-1, 50)));
    EXPECT_FALSE(rect.contains(Vec2(101, 50)));
}
```

---

## Pattern: Line-Shape Intersection

### Complete Intersection Coverage

**Problem**: Line-rectangle intersection can occur in multiple ways:
1. Endpoints inside shape
2. Line crosses shape edges
3. Line segment fully contained within shape

**Trap**: Only checking (1) and (2) misses case where both endpoints are on boundaries.

### Implementation

```cpp
// ✅ CORRECT: Handles all intersection cases
bool Rectangle::intersects(const Vec2& p0, const Vec2& p1) const {
    // Case 1: Either endpoint is inside (or on boundary)
    if (contains(p0) || contains(p1)) {
        return true;
    }

    // Case 2: Line crosses any edge
    Vec2 topLeft(minX, minY);
    Vec2 topRight(maxX, minY);
    Vec2 bottomRight(maxX, maxY);
    Vec2 bottomLeft(minX, maxY);

    Vec2 intersection;
    return lineSegmentIntersection(p0, p1, topLeft, topRight, intersection) ||
           lineSegmentIntersection(p0, p1, topRight, bottomRight, intersection) ||
           lineSegmentIntersection(p0, p1, bottomRight, bottomLeft, intersection) ||
           lineSegmentIntersection(p0, p1, bottomLeft, topLeft, intersection);
}

// ❌ WRONG: Only checks endpoints strictly inside + edge crossings
bool Rectangle::intersects(const Vec2& p0, const Vec2& p1) const {
    // Misses case where endpoints are ON boundaries but line crosses interior
    if (strictlyInside(p0) || strictlyInside(p1)) {
        return true;
    }
    // ... edge crossing checks
}
```

### Degenerate Cases

```cpp
TEST(Rectangle, LineIntersectionEdgeCases) {
    Rectangle rect(0, 0, 100, 100);

    // Both endpoints on boundaries
    EXPECT_TRUE(rect.intersects(Vec2(0, 50), Vec2(50, 0)));

    // Line segment entirely on edge
    EXPECT_TRUE(rect.intersects(Vec2(0, 0), Vec2(0, 50)));

    // Line segment touching corner
    EXPECT_TRUE(rect.intersects(Vec2(-10, -10), Vec2(0, 0)));

    // Zero-length segment (point) on boundary
    EXPECT_TRUE(rect.intersects(Vec2(0, 0), Vec2(0, 0)));

    // Parallel to edge, just outside
    EXPECT_FALSE(rect.intersects(Vec2(-1, 0), Vec2(-1, 100)));
}
```

---

## Pattern: Floating-Point Tolerance

### Epsilon Comparisons

**Problem**: Direct equality tests fail due to floating-point precision.

```cpp
// ❌ WRONG: Exact comparison
bool almostEqual(float a, float b) {
    return a == b;  // Fails for 0.1 + 0.2 != 0.3
}

// ✅ CORRECT: Epsilon comparison
bool almostEqual(float a, float b, float epsilon = 1e-6f) {
    return std::abs(a - b) < epsilon;
}

// ✅ BETTER: Relative epsilon for large values
bool almostEqual(float a, float b, float epsilon = 1e-6f) {
    float absA = std::abs(a);
    float absB = std::abs(b);
    float diff = std::abs(a - b);

    // Handle exact equality and near-zero values
    if (a == b) return true;
    if (a == 0.0f || b == 0.0f || (absA + absB < epsilon)) {
        return diff < epsilon;
    }

    // Use relative error for larger values
    return diff / std::min(absA + absB, std::numeric_limits<float>::max()) < epsilon;
}
```

### When to Use Epsilon

- ✅ **Do** use epsilon for: intersection points, distance tests, colinearity checks
- ❌ **Don't** use epsilon for: integer grid coordinates, pixel positions, array indices

---

## Pattern: Coordinate System Documentation

### Document Assumptions

Every geometric function should document:
1. **Origin location**: Top-left vs bottom-left vs center
2. **Axis directions**: +Y up or down?
3. **Units**: Pixels, meters, normalized?
4. **Handedness**: Left-handed (screen) vs right-handed (world)?

```cpp
/// Tests if a point is inside this rectangle.
///
/// Coordinate system: Screen space (origin top-left, +Y down)
/// Boundary behavior: Inclusive (points on edges are considered inside)
///
/// @param point Point in screen coordinates (pixels)
/// @return true if point is inside or on boundary, false otherwise
bool Rectangle::contains(const Vec2& point) const;
```

---

## Pattern: Degenerate Case Handling

### Always Test Zero-Size Inputs

```cpp
TEST(Circle, DegenerateCases) {
    // Zero radius
    Circle zeroCircle(Vec2(50, 50), 0);
    EXPECT_FALSE(zeroCircle.contains(Vec2(51, 50))); // Outside
    EXPECT_TRUE(zeroCircle.contains(Vec2(50, 50)));  // Center

    // Negative radius (should assert or clamp to zero)
    EXPECT_DEATH(Circle(Vec2(0, 0), -10), "radius");
}

TEST(Line, DegenerateCases) {
    // Zero-length line segment
    Vec2 point(50, 50);
    EXPECT_TRUE(lineSegmentDistance(point, point, Vec2(50, 50)) == 0);
    EXPECT_TRUE(lineSegmentDistance(point, point, Vec2(51, 50)) == 1);
}
```

---

## Pattern: Performance Considerations

### Early Exit Optimization

```cpp
// ✅ CORRECT: Check cheap tests first
bool Circle::intersects(const Rectangle& rect) const {
    // Early reject: Circle center too far from rectangle
    Vec2 closestPoint = rect.closestPointTo(center);
    if (center.distanceSquared(closestPoint) > radius * radius) {
        return false;
    }

    // More expensive containment tests
    // ...
}

// ❌ WRONG: Expensive test first
bool Circle::intersects(const Rectangle& rect) const {
    // Checks all 4 edges even if center is far away
    return intersectsEdge(rect.topEdge()) ||
           intersectsEdge(rect.rightEdge()) ||
           // ...
}
```

### Avoid Unnecessary sqrt()

```cpp
// ✅ CORRECT: Compare squared distances
float distSq = (p1 - p0).lengthSquared();
if (distSq < radiusSq) {
    // Point inside circle
}

// ❌ WRONG: Unnecessary sqrt
float dist = (p1 - p0).length();  // sqrt() is expensive
if (dist < radius) {
    // Point inside circle
}
```

---

## Anti-Patterns

### ❌ Mixing Coordinate Systems

```cpp
// BAD: Mixing screen and world coordinates
void draw() {
    Vec2 screenPos = worldToScreen(entity.position);

    // BUG: worldSize is in world units, screenPos is in pixels
    Rectangle bounds(screenPos.x, screenPos.y, entity.worldSize, entity.worldSize);
}
```

### ❌ Ignoring Edge Cases in Documentation

```cpp
// BAD: Doesn't specify boundary behavior
bool contains(Vec2 point);

// GOOD: Explicit about edges
/// Returns true if point is inside or on the boundary of this rectangle.
bool contains(Vec2 point);
```

### ❌ Inconsistent Boundary Handling

```cpp
// BAD: Different functions use different conventions
bool Rectangle::contains(Vec2 p) { return p.x >= minX && p.x < maxX; }  // Inclusive left, exclusive right
bool Circle::contains(Vec2 p) { return distance(p) <= radius; }         // Inclusive boundary

// GOOD: Consistent convention across all shapes
bool Rectangle::contains(Vec2 p) { return p.x >= minX && p.x <= maxX; } // Inclusive
bool Circle::contains(Vec2 p) { return distance(p) <= radius; }         // Inclusive
```

---

## Testing Checklist

For every geometric algorithm, test:

- [ ] Interior points (clearly inside)
- [ ] Exterior points (clearly outside)
- [ ] Boundary points (on edges)
- [ ] Corner points (vertices)
- [ ] Degenerate inputs (zero-size shapes, zero-length segments)
- [ ] Negative dimensions (should assert or clamp)
- [ ] Floating-point precision (almost-equal cases)
- [ ] Performance (large inputs, worst-case scenarios)

---

## Real-World Example

**Case Study**: OpenFrameworks ofRectangle::intersects bug ([issue #4871](https://github.com/openframeworks/openFrameworks/issues/4871))

**Problem**: `ofRectangle(0, 0, 100, 100).intersects(ofPoint(0, 50), ofPoint(50, 0))` returned `false`.

**Root Cause**: Function only checked if endpoints were *strictly inside* the rectangle. It missed the case where both endpoints are on boundaries but the line crosses the interior.

**Fix**: Ensure `contains()` uses inclusive boundaries, so boundary points are detected.

**Lesson**: Boundary conditions are the #1 source of geometric bugs. Test them exhaustively.

---

## References

- Christer Ericson, "Real-Time Collision Detection" (Morgan Kaufmann, 2004)
- Joseph O'Rourke, "Computational Geometry in C" (Cambridge, 1998)
- Bruce Dawson, "Comparing Floating Point Numbers" (2012)
- OpenFrameworks geometric utilities: https://openframeworks.cc/documentation/types/ofRectangle/

---

## Version History

- **1.0.0** (2025-10-12): Initial release
  - Boundary inclusion patterns
  - Line-shape intersection patterns
  - Floating-point tolerance guidelines
  - Degenerate case handling

---

**License**: Apache-2.0
**Maintained by**: [Equilateral AI](https://github.com/Equilateral-AI)
**Contributions**: PRs welcome at https://github.com/Equilateral-AI/equilateral-open-standards
