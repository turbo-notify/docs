# Developer Onboarding Guide

> Complete guide to set up your development environment and understand Turbo Notify.

---

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.13+ | API and workers |
| Poetry | 1.8+ | Python dependency management |
| Docker | 24+ | Local services |
| Docker Compose | 2.20+ | Service orchestration |
| Git | 2.40+ | Version control |
| Node.js | 20+ | Dashboard/Landing (optional) |

### Recommended Tools

- VS Code with Python extension
- DBeaver or pgAdmin (PostgreSQL client)
- NATS CLI (`nats`)
- HTTPie or curl

---

## Environment Setup

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd turbo-notity
```

### Step 2: Install Python Dependencies

```bash
# Landing API
cd landing-api
poetry install

# Dashboard API
cd ../dashboard-api
poetry install
```

### Step 3: Configure Environment

```bash
cp .env.example .env
```

Edit `.env` with your local settings:

```env
# Database
DATABASE_URL=postgresql+asyncpg://turbo:turbo@localhost:5432/turbo_notify

# NATS
NATS_URL=nats://localhost:4222

# Redis (required for local integration flows)
REDIS_URL=redis://localhost:6379

# API
API_HOST=0.0.0.0
API_PORT=8000
DEBUG=true

# Security
SECRET_KEY=your-secret-key-for-development
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=1440
```

### Step 4: Start Infrastructure

```bash
docker-compose up -d
```

This starts:
- PostgreSQL (port 5432)
- NATS + JetStream (ports 4222, 8222)
- Redis (port 6379)

Redis is required for shared `rate-sync` limiter behavior in API/workers/webhook dispatcher.
In-memory limiter backend is only for isolated local unit tests.

### Step 5: Run Migrations

```bash
cd landing-api
poetry run alembic upgrade head
```

### Step 6: Start the APIs

```bash
# Landing API (port 8010)
cd landing-api
poetry run python -m interfaces.http

# Dashboard API (port 8020, in another terminal)
cd dashboard-api
poetry run python -m interfaces.http
```

Landing API: `http://localhost:8010` (Swagger: `http://localhost:8010/docs`)
Dashboard API: `http://localhost:8020` (Swagger: `http://localhost:8020/docs`)

---

## Project Structure

```
turbo-notity/
├── landing-api/           # Landing Page API (Python + FastAPI)
│   ├── src/
│   │   ├── application/   # Use cases, DTOs
│   │   ├── domain/        # Business entities
│   │   ├── infrastructure/# DB, Redis, NATS
│   │   └── interfaces/    # HTTP routes
│   │       └── http/      # FastAPI app
│   ├── tests/
│   └── pyproject.toml
│
├── dashboard-api/         # Dashboard API (Python + FastAPI)
│   ├── src/
│   │   ├── application/   # Use cases, DTOs
│   │   ├── domain/        # Business entities
│   │   ├── infrastructure/# DB, Redis, NATS
│   │   └── interfaces/    # HTTP routes
│   │       └── http/      # FastAPI app
│   ├── tests/
│   └── pyproject.toml
│
├── public-api/            # Control Plane API (Go - legacy)
│
├── shared-core/           # Shared domain library
│   └── src/shared_core/   # Core entities, infrastructure
│
├── shared-api/            # Shared FastAPI library
│   └── src/shared_api/    # Common API infrastructure
│
├── workers/               # Session Workers (future)
├── orchestrator/          # Session Orchestrator (future)
│
├── dashboard-web/         # Admin Dashboard (Next.js)
├── landing-web/           # Marketing Website (Next.js)
├── ops/                   # Infrastructure
│   ├── docker/
│   └── scripts/
│
├── docs/                  # Documentation
└── docker-compose.yml
```

---

## Architecture Understanding

### Core Concepts

Read these documents in order:

1. **[Ecosystem Architecture](../architecture/ecosystem-architecture.md)**
   - Complete system overview
   - Component responsibilities
   - Data flow diagrams

2. **[Session Lifecycle](../architecture/session-lifecycle.md)**
   - Session states (cold, warm, hot)
   - State transitions
   - Lease management

3. **[NATS Events](../architecture/nats-events.md)**
   - Message subjects
   - Event contracts
   - Consumer patterns

4. **[Database Schema](../architecture/database-schema.md)**
   - Table structures
   - Relationships
   - Common queries

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Tenant** | A customer organization using the platform |
| **Session** | A WhatsApp Web connection for a phone number |
| **Worker** | A process that manages multiple sessions |
| **Lease** | Temporary ownership of a session by a worker |
| **Control Plane** | The API that manages everything |

---

## First Development Task

### Understanding the Codebase

1. **Explore the API structure**
   ```bash
   cd landing-api/src
   tree -L 2
   ```

2. **Read the main entry point**
   - `landing-api/src/interfaces/http/app.py` - FastAPI app setup
   - `landing-api/src/interfaces/http/routes/` - Route definitions

3. **Understand the domain layer**
   - `landing-api/src/domain/entities/` - Core entities
   - `landing-api/src/domain/services/` - Business logic

### Running Tests

```bash
cd landing-api
poetry run pytest
```

With coverage:

```bash
poetry run pytest --cov=src --cov-report=html
```

### Adding a New Endpoint

1. Create the route in `src/interfaces/http/routes/`
2. Add business logic in `src/domain/services/`
3. Write tests in `tests/`
4. Run linting: `poetry run ruff check .`
5. Run tests: `poetry run pytest`

---

## Development Workflow

### Branch Naming

```
feature/description
fix/description
docs/description
refactor/description
```

### Commit Messages

Use conventional commits:

```
feat: add session reconnection logic
fix: handle webhook timeout correctly
docs: update onboarding guide
refactor: extract session state machine
test: add integration tests for messages
```

### Code Style

- **Linter**: Ruff
- **Formatter**: Ruff format
- **Type checking**: mypy

```bash
# Check code
poetry run ruff check .

# Format code
poetry run ruff format .

# Type check
poetry run mypy src/
```

### Pre-commit Hooks

```bash
poetry run pre-commit install
```

---

## Common Tasks

### View NATS Messages

```bash
# Install NATS CLI
brew install nats-io/nats-tools/nats

# Subscribe to all events
nats sub "turbo.>"

# View streams
nats stream ls

# View stream info
nats stream info MESSAGES
```

### Database Operations

```bash
# Connect to PostgreSQL
psql postgresql://turbo:turbo@localhost:5432/turbo_notify

# Create migration
poetry run alembic revision --autogenerate -m "add column"

# Apply migrations
poetry run alembic upgrade head

# Rollback
poetry run alembic downgrade -1
```

### Docker Commands

```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f api

# Restart service
docker-compose restart api

# Stop all
docker-compose down
```

---

## Troubleshooting

### API won't start

1. Check PostgreSQL is running: `docker-compose ps`
2. Verify `.env` settings
3. Run migrations: `poetry run alembic upgrade head`

### NATS connection refused

1. Check NATS is running: `docker-compose ps`
2. Verify port 4222 is not in use
3. Check NATS logs: `docker-compose logs nats`

### Database connection error

1. Verify PostgreSQL is running
2. Check credentials in `.env`
3. Ensure database exists

---

## Useful Links

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [NATS JetStream](https://docs.nats.io/nats-concepts/jetstream)
- [Poetry](https://python-poetry.org/docs/)

---

## Getting Help

- Check the [Glossary](../reference/glossary.md) for terminology
- Read [Runbooks](../runbooks/README.md) for common issues
- Ask in team chat

---

## Checklist

- [ ] Repository cloned
- [ ] Python 3.13+ installed
- [ ] Poetry installed
- [ ] Dependencies installed (`poetry install`)
- [ ] `.env` configured
- [ ] Docker services running
- [ ] Migrations applied
- [ ] API running locally
- [ ] Read architecture documentation
- [ ] Tests passing
- [ ] Pre-commit hooks installed
