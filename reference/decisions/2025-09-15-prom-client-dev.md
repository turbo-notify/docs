# ADR: Using eval('require') for prom-client

**Date:** 2025-09-15
**Status:** Accepted
**Context:** Next.js + prom-client compatibility

---

## Problem

When running the project with `pnpm dev`, Next.js tried to resolve the internal `cluster` module inside `prom-client`, resulting in:

```
Module not found: Can't resolve 'cluster'
```

## Decision

Remove direct imports of `prom-client` and load it only at runtime using `eval('require')`. This prevents the bundler from analyzing the package during the development build.

## Implementation

```javascript
// Instead of:
import client from 'prom-client';

// Use:
const client = eval('require')('prom-client');
```

## Consequences

- Prevents future refactorings from reintroducing the error
- Any use of `prom-client` must follow this pattern
- Only affects development mode bundling

## History

| Commit | Change |
|--------|--------|
| `7985ba2` (2025-09-03) | Introduced `eval('require')` approach |
| `a690691` | Replaced with direct `require`, reactivating bug |
| `cad1e14` | Re-established dynamic loading |
