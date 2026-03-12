# Contributing Guide

> How to contribute to Turbo Notify development.

---

## Getting Started

1. Read the [Onboarding Guide](guides/onboarding.md)
2. Set up your development environment
3. Familiarize yourself with the [Architecture](architecture/ecosystem-architecture.md)
4. Review the [Documentation Standards](standards/documentation-standards.md)

---

## Development Workflow

### Branch Strategy

```
main           # Production-ready code
├── develop    # Integration branch
├── feature/*  # New features
├── fix/*      # Bug fixes
├── docs/*     # Documentation
└── refactor/* # Code improvements
```

### Branch Naming

```
feature/add-session-warmup
fix/webhook-timeout-handling
docs/update-onboarding-guide
refactor/extract-session-state-machine
```

---

## Code Standards

### Python

- **Version:** 3.13+
- **Package Manager:** Poetry
- **Linter:** Ruff
- **Type Checker:** mypy
- **Test Framework:** pytest

```bash
# Install dependencies
poetry install

# Run linter
poetry run ruff check .

# Run formatter
poetry run ruff format .

# Run type checker
poetry run mypy src/

# Run tests
poetry run pytest
```

### API Standards

Follow [API Standards](standards/api-standards.md) for:
- URL structure
- Request/response format
- Error handling
- Versioning

---

## Commit Messages

Use conventional commits:

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code change, no feature/fix |
| `perf` | Performance improvement |
| `test` | Adding/updating tests |
| `chore` | Maintenance tasks |

### Examples

```
feat(sessions): add session warmup functionality

Implement session warmup to pre-connect sessions before
high-volume periods. Reduces latency for first message.

Closes #123
```

```
fix(webhooks): handle timeout correctly

Previously, webhook timeouts were not properly tracked,
causing retries to fail silently.
```

---

## Pull Request Process

### Before Creating PR

- [ ] Code compiles without errors
- [ ] All tests pass
- [ ] Linter passes
- [ ] Type checker passes
- [ ] Documentation updated (if needed)
- [ ] Commits are well-structured

### PR Template

```markdown
## Summary
Brief description of changes.

## Changes
- Change 1
- Change 2

## Testing
How to test these changes.

## Related Issues
Closes #123
```

### Review Process

1. Create PR with description
2. Request review from team member
3. Address review comments
4. Get approval
5. Squash and merge

---

## Testing

### Unit Tests

```bash
poetry run pytest tests/unit/
```

### Integration Tests

```bash
# Start dependencies
docker-compose up -d postgres nats

# Run tests
poetry run pytest tests/integration/
```

### Test Coverage

```bash
poetry run pytest --cov=src --cov-report=html
open htmlcov/index.html
```

### Writing Tests

- Test one thing per test
- Use descriptive test names
- Mock external dependencies
- Include edge cases

```python
# Good test name
def test_session_reconnect_increments_failure_count():
    ...

# Bad test name
def test_reconnect():
    ...
```

---

## Documentation

### When to Update Docs

- New feature added
- Behavior changed
- API modified
- New configuration option
- Error handling changed

### Documentation Checklist

- [ ] README updated if needed
- [ ] API docs reflect changes
- [ ] Runbooks updated for new operations
- [ ] Glossary includes new terms

---

## Code Review Guidelines

### As Author

- Keep PRs small and focused
- Respond to feedback promptly
- Explain non-obvious decisions
- Be open to suggestions

### As Reviewer

- Review within 24 hours
- Be constructive
- Explain the "why" for changes
- Approve when ready, don't block on style

### What to Check

- [ ] Logic is correct
- [ ] Tests are adequate
- [ ] Error handling is proper
- [ ] Performance is acceptable
- [ ] Security is considered
- [ ] Documentation is updated

---

## Release Process

### Versioning

Use semantic versioning: `MAJOR.MINOR.PATCH`

- **MAJOR:** Breaking changes
- **MINOR:** New features, backwards compatible
- **PATCH:** Bug fixes

### Release Checklist

- [ ] All tests pass
- [ ] Version bumped
- [ ] CHANGELOG updated
- [ ] Documentation updated
- [ ] Release notes written
- [ ] Tagged in git

---

## Getting Help

- Check existing documentation
- Ask in team chat
- Create issue for discussion

---

## Related Documentation

- [Onboarding Guide](guides/onboarding.md) - Environment setup
- [Architecture](architecture/ecosystem-architecture.md) - System design
- [API Standards](standards/api-standards.md) - API conventions
- [Documentation Standards](standards/documentation-standards.md) - Doc conventions
