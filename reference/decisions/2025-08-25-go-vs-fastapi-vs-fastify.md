# ADR: Go vs Python (FastAPI) vs Node.js (Fastify)

**Date:** 2025-08-25
**Status:** SUPERSEDED by ADR-0001 (2026-03-12)

---

## Original Decision

This document originally recommended **Go** for API and Workers based on performance benchmarks:

| Metric | Go | Python (FastAPI) | Node.js (Fastify) |
|--------|-----|------------------|-------------------|
| Startup | 10-40ms | 150-700ms | 100-500ms |
| Memory idle | 10-30 MB | 70-150 MB | 70-150 MB |
| Throughput | 20k-100k req/s | 1k-5k req/s | 10k-50k req/s |

**Original Hierarchy:** Go > Node.js (Fastify) > Python (FastAPI)

---

## Superseded Decision

**As of 2026-03-12, this ADR is SUPERSEDED.**

The project has decided to use **Python (FastAPI)** for all backend services because:

1. **Team Velocity:** Single language reduces context switching
2. **Ecosystem:** Better WhatsApp Web libraries in Python
3. **Automacity Pattern:** Existing Python workers pattern can be reused
4. **Acceptable Trade-off:** Performance difference mitigated by horizontal scaling

See **ADR-0001: Python + FastAPI Stack** for the current decision.

---

## Historical Value

This document is preserved for historical context and the detailed benchmark data, which remains accurate. The decision to override was based on team and ecosystem considerations rather than disputing the performance data.
