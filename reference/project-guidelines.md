# Project Guidelines

> Development standards and conventions for Turbo Notify.

---

## Package Management

### JavaScript/TypeScript Projects (Landing, Dashboard)

- **Use pnpm** as the package manager
- Never use `npm` or `yarn`
- Avoid versioning lock files in module-specific repos

```bash
# Install dependencies
pnpm install

# Add a package
pnpm add <package>

# Run scripts
pnpm run <script>
```

### Python Projects (API, Workers)

- **Use Poetry** as the package manager
- Never use pip, uv, pipenv, or conda
- For rate limiting, use only `rate-sync + redis` (no custom throttling counters)

```bash
# Install dependencies
poetry install

# Add a package
poetry add <package>

# Run commands
poetry run <command>
```

---

## Code Conventions

### Exports

In barrel exports, **do not use `export *`**; prefer named exports:

```typescript
// Good
export { ComponentA } from './ComponentA';
export { ComponentB } from './ComponentB';

// Bad
export * from './ComponentA';
export * from './ComponentB';
```

### File Naming

| Type | Convention | Example |
|------|------------|---------|
| TypeScript/JavaScript files | kebab-case | `user-service.ts` |
| Python files | snake_case | `user_service.py` |
| Markdown docs | kebab-case | `getting-started.md` |
| ADRs | `YYYY-MM-DD-title.md` | `2025-08-21-db-choice.md` |

---

## Related Documents

- [Documentation Standards](/docs/standards/documentation-standards.md)
- [API Standards](/docs/standards/api-standards.md)
- [CLAUDE.md](/CLAUDE.md) - AI guidelines
