# Extra Numbers API

> Manage additional WhatsApp sender numbers.

---

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/extra-numbers` | Add extra number |
| `GET` | `/extra-numbers` | List extra numbers |
| `GET` | `/extra-numbers/{alias}/status` | Get number status |
| `POST` | `/extra-numbers/activate` | Activate extra number |
| `POST` | `/extra-numbers/deactivate` | Deactivate extra number |
| `DELETE` | `/extra-numbers/{alias}` | Remove extra number |

---

## Add Extra Number

```http
POST /extra-numbers
```

```json
{
  "phone": "+5511988888888",
  "alias": "clinica_tal"
}
```

Success (`200`):

```json
{
  "code": "AAAA-BBBB"
}
```

Common errors:

- `400 invalid_json`
- `402 not_available_in_plan`
- `403 not_authorized`
- `409 number_already_exists`
- `500 internal_error`
- `503 whatsapp_unavailable`

---

## List Extra Numbers

```http
GET /extra-numbers
```

Success (`200`):

```json
[
  {
    "alias": "condominio_1",
    "number": "+5511999999997",
    "status": {
      "state": "active",
      "at": "2025-07-16T10:01:00Z"
    }
  }
]
```

Common errors:

- `402 not_available_in_plan`
- `403 not_authorized`
- `500 internal_error`

---

## Extra Number Status

```http
GET /extra-numbers/{alias}/status
```

Success (`200`):

```json
{
  "state": "active",
  "at": "2025-07-16T10:01:00Z"
}
```

Possible status values:

- `active`
- `inactive`
- `pending`
- `failed`

Failure `reason` examples:

- `invalid_code`
- `whatsapp_unavailable`

Common errors:

- `402 not_available_in_plan`
- `403 not_authorized`
- `404 number_not_found`
- `500 internal_error`

---

## Activate / Deactivate

### Activate

```http
POST /extra-numbers/activate
```

```json
{
  "alias": "clinica_tal"
}
```

Success (`200`):

```json
{
  "state": "active",
  "at": "2025-07-16T10:01:00Z"
}
```

### Deactivate

```http
POST /extra-numbers/deactivate
```

```json
{
  "alias": "clinica_tal"
}
```

Success (`200`):

```json
{
  "state": "inactive",
  "at": "2025-07-16T10:05:00Z"
}
```

Common errors:

- `400 invalid_json`
- `403 not_authorized`
- `409 invalid_status`
- `422 number_not_found`
- `500 internal_error`

---

## Remove Extra Number

```http
DELETE /extra-numbers/{alias}
```

Success (`200`):

```json
{
  "number": "+5511988888888",
  "alias": "clinica_tal"
}
```

Common errors:

- `402 not_available_in_plan`
- `403 not_authorized`
- `404 number_not_found`
- `500 internal_error`
- `503 whatsapp_unavailable`

---

## Webhook Behavior

- Extra numbers use the same account-level webhook configured in dashboard.
- Per-number webhook configuration is not part of the public contract.

## Related

- [Messages](messages.md)
- [Webhooks](webhooks.md)
- [Errors](errors.md)
