# Documentation Navigation

> Find the right documentation for your needs.

---

## By Role

### I'm a new developer

Start here to get your environment set up and understand the codebase:

1. [Onboarding Guide](guides/onboarding.md) - Environment setup and first steps
2. [Ecosystem Architecture](architecture/ecosystem-architecture.md) - Understand the system
3. [Glossary](reference/glossary.md) - Learn the terminology
4. [Session Lifecycle](architecture/session-lifecycle.md) - Core concept

### I'm a Product Owner

Understand the business context and technical constraints:

1. [Product Owners Guide](guides/product-owners.md) - Business overview
2. [README](README.md) - Platform capabilities
3. [Glossary](reference/glossary.md) - Domain terminology

### I'm an operator / SRE

Operational guides and troubleshooting:

1. [Runbooks Index](runbooks/README.md) - All operational procedures
2. [Observability Overview](observability/README.md) - Monitoring stack
3. [Metrics Guide](observability/metrics-guide.md) - Key metrics

---

## By Task

### I need to understand...

| Topic | Document |
|-------|----------|
| Product vision | [Vision](product/vision.md) |
| How the system works | [Ecosystem Architecture](architecture/ecosystem-architecture.md) |
| User-facing vs internal API | [API Contract Alignment](architecture/api-contract-alignment.md) |
| Global API change workflow | [Public API Governance](reference/public-api-governance.md) |
| Public API endpoints | [Public API Specification](api/README.md) |
| Session management | [Session Lifecycle](architecture/session-lifecycle.md) |
| Worker design | [Worker Architecture](architecture/worker-architecture.md) |
| Authentication system | [Authentication Flow](architecture/authentication-flow.md) |
| Event messaging | [NATS Events](architecture/nats-events.md) |
| Database structure | [Database Schema](architecture/database-schema.md) |
| Rate-limit standard | [ADR: rate-sync as Mandatory Rate-Limiting Engine](reference/decisions/2026-03-12-rate-sync-rate-limiting.md) |
| Business terminology | [Glossary](reference/glossary.md) |
| Architecture decisions | [ADRs](reference/decisions/) |

### I need to set up...

| Task | Document |
|------|----------|
| Development environment | [Onboarding Guide](guides/onboarding.md) |
| Monitoring | [Observability Overview](observability/README.md) |

### I need to fix...

| Problem | Document |
|---------|----------|
| Session not connecting | [Session Troubleshooting](runbooks/session-troubleshooting.md) |
| Worker failures | [Worker Recovery](runbooks/worker-recovery.md) |
| Message delivery issues | [Message Troubleshooting](runbooks/message-troubleshooting.md) |
| Webhook failures | [Webhook Troubleshooting](runbooks/webhook-troubleshooting.md) |

### I need to implement...

| Feature Area | Reference |
|--------------|-----------|
| New NATS event | [NATS Events](architecture/nats-events.md) |
| New API endpoint | [API Standards](standards/api-standards.md) |
| Rate-limit policy | [Rate Limits](api/rate-limits.md) |
| New metric | [Metrics Guide](observability/metrics-guide.md) |

---

## Quick Reference

### Common Commands

```bash
# Start development environment
docker-compose up -d

# Run API locally
cd api && poetry run uvicorn src.main:app --reload

# Run tests
cd api && poetry run pytest

# Check NATS streams
nats stream ls
```

### Key Files

| Purpose | Location |
|---------|----------|
| Public API contract (source of truth) | `docs/api/README.md` |
| FastAPI entry point (target) | `api/src/main.py` |
| Landing customer docs | `landing/src/content/docs/` |
| Landing code templates | `landing/src/features/docs/lib/code-templates/` |

### Important URLs

| Service | URL |
|---------|-----|
| API (local) | `http://localhost:8000` |
| NATS Monitor | `http://localhost:8222` |
| Dashboard (local) | `http://localhost:3000` |

---

## Documentation Map

```
docs/
├── README.md              ← Start here
├── NAVIGATION.md          ← You are here
├── CONTRIBUTING.md        ← How to contribute
│
├── product/               ← Business context
│   ├── vision.md
│   ├── pitch.md
│   └── overview.md
│
├── api/                   ← Public API contract
│   ├── README.md
│   ├── authentication.md
│   ├── messages.md
│   ├── extra-numbers.md
│   ├── reactions.md
│   ├── typing-indicator.md
│   ├── webhooks.md
│   ├── errors.md
│   ├── rate-limits.md
│   └── changelog.md
│
├── architecture/          ← System design
│   ├── ecosystem-architecture.md
│   ├── worker-architecture.md
│   ├── authentication-flow.md
│   ├── api-contract-alignment.md
│   ├── nats-events.md
│   ├── session-lifecycle.md
│   └── database-schema.md
│
├── guides/                ← Tutorials
│   ├── onboarding.md
│   └── product-owners.md
│
├── observability/         ← Monitoring
│   ├── README.md
│   └── metrics-guide.md
│
├── runbooks/              ← Operations
│   ├── README.md
│   ├── session-troubleshooting.md
│   ├── worker-recovery.md
│   ├── message-troubleshooting.md
│   └── webhook-troubleshooting.md
│
├── standards/             ← Conventions
│   ├── documentation-standards.md
│   ├── api-standards.md
│   └── analytics.md
│
├── templates/             ← Doc templates
│   └── README.md
│
└── reference/             ← Lookup
    ├── glossary.md
    ├── public-api-governance.md
    ├── project-guidelines.md
    └── decisions/         ← ADRs
```
