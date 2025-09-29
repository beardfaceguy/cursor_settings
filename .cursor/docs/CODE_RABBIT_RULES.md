# CodeRabbit Rules Reference

This document captures the coding standards and rules enforced by CodeRabbit AI code reviews. These rules should be followed in all future implementations to ensure consistent code quality and avoid review comments.

## Security Rules

### 1. Cryptographically Secure Random Number Generation
**Rule**: Never use `Math.random()` for generating IDs or tokens that could be exposed externally.

**Enforcement**: CodeRabbit flags `Math.random()` usage as a security vulnerability.

**Solution**: Use Node.js `crypto.randomInt()` for cryptographically secure random number generation.

**Example**:
```typescript
// ❌ BAD - Predictable and insecure
const id = chars.charAt(Math.floor(Math.random() * chars.length));

// ✅ GOOD - Cryptographically secure
import { randomInt } from "crypto";
const id = chars.charAt(randomInt(0, chars.length));
```

**Files Affected**: Any utility that generates IDs, tokens, or random values

## Database & Error Handling Rules

### 2. Unique Constraint Violation Handling
**Rule**: Always implement retry logic for unique constraint violations (P2002 errors) when generating unique identifiers.

**Enforcement**: CodeRabbit expects robust error handling for race conditions.

**Solution**: Implement retry mechanism with exponential backoff or simple retry loop.

**Example**:
```typescript
// ✅ GOOD - Retry logic for unique constraint violations
let attempts = 0;
const maxAttempts = 5;
while (attempts < maxAttempts) {
  try {
    const result = await db.create({ data: { uniqueId: generateId() } });
    break;
  } catch (error: any) {
    if (error.code === 'P2002' && error.meta?.target?.includes('uniqueId')) {
      attempts++;
      if (attempts >= maxAttempts) {
        throw new Error(`Failed to generate unique ID after ${maxAttempts} attempts`);
      }
      continue; // Retry with new ID
    }
    throw error; // Re-throw non-unique constraint errors
  }
}
```

**Files Affected**: Services that create entities with auto-generated unique fields

### 3. Client Input Validation
**Rule**: Always validate and sanitize client-provided input, especially for unique identifiers.

**Enforcement**: CodeRabbit expects input validation for fields that could cause database constraint violations.

**Solution**: Implement format validation and normalize input (e.g., empty strings to null).

**Example**:
```typescript
// ✅ GOOD - Validate client input
const validateScanBoxId = (id: string): boolean => {
  const pattern = /^[A-Z0-9]{8}$/;
  return pattern.test(id);
};

// Normalize empty strings to null to prevent unique constraint violations
const normalizedId = clientId?.trim() || null;
```

**Files Affected**: Services that accept client-provided unique identifiers

## API Design Rules

### 4. Immutable Unique Identifiers
**Rule**: Unique identifiers that are exposed externally should be immutable after creation.

**Enforcement**: CodeRabbit flags update operations on unique external identifiers.

**Solution**: Remove unique identifiers from update inputs or implement dedicated rotation endpoints.

**Example**:
```typescript
// ❌ BAD - Allows mutation of unique external ID
input EstateUpdateInput {
  scanBoxId: String  // Should not be mutable
}

// ✅ GOOD - Immutable unique identifier
input EstateUpdateInput {
  // scanBoxId removed - use dedicated rotation endpoint if needed
}

// ✅ GOOD - Dedicated rotation endpoint
mutation rotateScanBoxId(estateId: ID!): Estate
```

**Files Affected**: GraphQL input types for entities with unique external identifiers

### 5. Server-Generated Field Handling
**Rule**: Server-generated fields should not be exposed in create inputs unless explicitly needed.

**Enforcement**: CodeRabbit questions why server-generated fields appear in create inputs.

**Solution**: Either remove from create inputs or implement proper validation.

**Example**:
```typescript
// ❌ BAD - Server-generated field in create input without validation
input EstateCreateInput {
  scanBoxId: String  // Server generates this, why is it in create input?
}

// ✅ GOOD - Server-generated field not in create input
input EstateCreateInput {
  // scanBoxId generated server-side, not in input
}

// ✅ GOOD - Server-generated field with validation if needed
input EstateCreateInput {
  scanBoxId: String  // Only if client needs to provide specific value
}
```

**Files Affected**: GraphQL create input types

## Database Schema Rules

### 6. Database-Level Constraints
**Rule**: Enforce data integrity at the database level, not just application level.

**Enforcement**: CodeRabbit suggests database-level constraints for format validation.

**Solution**: Use CHECK constraints and appropriate column types.

**Example**:
```sql
-- ✅ GOOD - Database-level format constraint
ALTER TABLE "Estate" ADD CONSTRAINT "Estate_scanBoxId_format" 
CHECK ("scanBoxId" ~ '^[A-HJ-NP-Z2-9]{8}$');

-- ✅ GOOD - Fixed length column type
ALTER TABLE "Estate" ALTER COLUMN "scanBoxId" TYPE CHAR(8);
```

**Files Affected**: Database migrations and Prisma schema

### 7. Index Strategy for Unique Fields
**Rule**: Ensure proper indexing for unique fields and query patterns.

**Enforcement**: CodeRabbit checks for appropriate indexes on unique fields.

**Solution**: Create unique indexes and consider covering indexes for common query patterns.

**Example**:
```sql
-- ✅ GOOD - Unique index for uniqueness constraint
CREATE UNIQUE INDEX "Estate_scanBoxId_key" ON "Estate"("scanBoxId");

-- ✅ GOOD - Covering index for common queries
CREATE INDEX "Estate_scanBoxId_status_idx" ON "Estate"("scanBoxId", "status") 
WHERE "scanBoxId" IS NOT NULL;
```

**Files Affected**: Database migrations

## Test Data Rules

### 8. Test Fixture Consistency
**Rule**: Test fixtures should match the actual data format and constraints.

**Enforcement**: CodeRabbit flags test data that violates stated format rules.

**Solution**: Ensure test fixtures follow the same format as production data.

**Example**:
```typescript
// ✅ GOOD - Test fixture matches format
const mockEstate = {
  scanBoxId: "ABC12345",  // Follows A-Z, 0-9 format
};
```

**Files Affected**: Test fixtures and mock data

## Documentation Rules

### 9. Field Documentation
**Rule**: Add documentation for unique fields to clarify their purpose and format.

**Enforcement**: CodeRabbit suggests adding field descriptions for clarity.

**Solution**: Add doc comments to Prisma models and GraphQL schema.

**Example**:
```prisma
model Estate {
  /// External "Scan Box" identifier (8 chars; A-Z, 0-9). Not secret; unique when present.
  scanBoxId String? @unique
}
```

**Files Affected**: Prisma schema, GraphQL schema, TypeScript interfaces

## Operational Rules

### 10. Migration Safety
**Rule**: Database migrations should be safe for production environments.

**Enforcement**: CodeRabbit flags potentially disruptive migration operations.

**Solution**: Use CONCURRENTLY for index creation on large tables.

**Example**:
```sql
-- ✅ GOOD - Concurrent index creation for large tables
CREATE UNIQUE INDEX CONCURRENTLY "Estate_scanBoxId_key" ON "Estate"("scanBoxId");
```

**Files Affected**: Database migrations

## Implementation Checklist

When implementing new features with unique identifiers, ensure:

- [ ] Use `crypto.randomInt()` for ID generation
- [ ] Implement retry logic for unique constraint violations
- [ ] Validate client input format
- [ ] Make unique identifiers immutable after creation
- [ ] Add database-level constraints
- [ ] Create appropriate indexes
- [ ] Update test fixtures to match format
- [ ] Add field documentation
- [ ] Use safe migration practices

## Common CodeRabbit Comments

- **Security**: "Use crypto.randomInt for stronger randomness"
- **Error Handling**: "Add retry logic for P2002 unique constraint violations"
- **Validation**: "Add format validation for scanBoxId"
- **Immutability**: "Make scanBoxId immutable post-creation"
- **Database**: "Consider enforcing format/length at DB level"
- **Tests**: "Fixture violates stated charset"
- **Documentation**: "Add field description to propagate via codegen"

## References

- [CodeRabbit Review of PR #633](https://github.com/meetalix/alix-api/pull/633)
- [Node.js crypto.randomInt() documentation](https://nodejs.org/api/crypto.html#cryptorandomintmin-max-callback)
- [Prisma Error Codes](https://www.prisma.io/docs/reference/api-reference/error-reference)
- [PostgreSQL CREATE INDEX CONCURRENTLY](https://www.postgresql.org/docs/current/sql-createindex.html)
