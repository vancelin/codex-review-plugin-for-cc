# Attack Surface Reference

Prioritized failure categories that are expensive, dangerous, or hard to detect.

---

## 1. Auth, Permissions, and Trust Boundaries

- Missing or bypassed authentication checks
- Authorization gaps — privilege escalation, tenant isolation failures
- Token/session handling: expiry, revocation, rotation
- Trust boundary assumptions — internal services treated as trusted
- Missing CSRF protection on state-changing operations

## 2. Data Loss, Corruption, and Irreversible State

- Non-atomic multi-step operations that can partially complete
- Missing transactions where business logic requires them
- Duplicate writes on retry without idempotency keys
- Cascading deletes without soft-delete or archival
- Silent truncation or coercion of data

## 3. Rollback Safety, Retries, and Idempotency

- Rollback paths that can themselves fail
- Partial failure leaving inconsistent state
- Non-idempotent operations called from retry loops
- Missing timeout causing indefinite hangs
- Resource leaks on error paths (connections, file handles, locks)

## 4. Race Conditions, Ordering, and Stale State

- TOCTOU (time-of-check-to-time-of-use) gaps
- Shared mutable state without synchronization
- Optimistic locking without retry
- Stale cache reads that drive business decisions
- Re-entrant callbacks that modify state during iteration
- Event ordering assumptions that break under load

## 5. Empty-State, Null, and Degraded Dependencies

- Null/undefined/empty penetrating through call chains
- Timeout values — are they reasonable? what happens on timeout?
- Degraded dependency behavior (slow DB, cache miss)
- Default values that mask real failures

## 6. Version Skew, Schema Drift, and Compatibility

- Schema changes without backward-compatible migration
- API contract changes that break existing clients
- Feature flags that leave dead code paths
- Configuration changes requiring coordinated deployment
- Type changes compatible in dev but failing in production

## 7. Observability Gaps

- Errors caught and swallowed without logging
- Missing structured logging for critical operations
- Metrics gaps — no alerting on failure modes
- Health checks that don't verify actual functionality
- Missing correlation IDs for tracing
