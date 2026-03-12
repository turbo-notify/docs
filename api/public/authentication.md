# Authentication

> Access control contract for Turbo Notify public API.

---

## Header

All requests must include:

```http
Authorization: Bearer <access_key>
```

Requests without valid header must be rejected with:

```http
403 Forbidden
```

```json
{
  "error": "not_authorized"
}
```

---

## Access Keys

- Keys are managed in Turbo Notify dashboard.
- Key value is shown only at creation time.
- Keys can be deactivated (temporary block) or deleted (permanent block).
- Multiple keys per account are allowed for environment/team separation.

---

## Scope Model

Current contract scope is account-level:

- A valid key can operate account resources, including main sender and extra numbers.
- Per-number key scoping is not part of the current public contract.

---

## Security Requirements

- Use HTTPS in all environments.
- Never expose keys in frontend code or logs.
- Rotate keys if compromise is suspected.

---

## Related

- [Errors](errors.md)
- [Messages](messages.md)
- [Extra Numbers](extra-numbers.md)
