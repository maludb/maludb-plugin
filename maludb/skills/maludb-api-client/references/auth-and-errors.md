# MaluDB API — authentication & errors

## Authentication

**Mechanism:** Bearer token on **every** request. There is no login/session endpoint.

```
Authorization: Bearer malu_pQYTIRdzGeGaoX4u-1uw3u4Ozbq61gPu5aKPbi3_Nmw
```

- **Token format:** the literal prefix `malu_` followed by ~43 URL-safe base64 chars
  (≈32 bytes of entropy). Treat the whole string as opaque.
- **Issuance & revocation are admin operations** (done in `psql` against the `api_tokens`
  table), **not** API endpoints. Your client receives a token out of band (config,
  keychain, env var) — it cannot mint or rotate one over the API in v1.
- **Validation, server-side:** the server strips `malu_`, sha256-hashes the remainder, and
  looks it up in `api_tokens WHERE token_hash = ? AND expires_at > now()`. So tokens
  **expire** — an expired token returns `401 auth_invalid` just like a wrong one.
- **Storage:** keep the token out of source control and logs. In Electron use the OS
  keychain / `safeStorage`; in a browser app the token must come from your own backend
  session (do not ship a long-lived MaluDB token to a browser).

### 401 cases

| Situation | `code` |
|-----------|--------|
| No `Authorization` header, or not `Bearer <token>` | `auth_missing` |
| Token doesn't start with `malu_` | `auth_invalid` |
| Hash not found, or token expired | `auth_invalid` |

On any `401`, surface a "session expired / not signed in" state and re-acquire a token;
do not retry the same token.

## Error responses

Every error — regardless of status — has this exact shape:

```json
{ "error": { "code": "stable_string", "message": "Human-readable explanation." } }
```

Switch your client logic on `code` (stable). The `message` is for developers/logs; never
render it verbatim to end users.

### Status codes & common codes

| Status | Meaning | Common `code` values |
|--------|---------|----------------------|
| `400` | Malformed request: bad JSON, missing required field, bad query param | `bad_request`, `body_invalid_json`, `missing_field` |
| `401` | Missing/invalid/expired token | `auth_missing`, `auth_invalid` |
| `403` | Authenticated but not permitted | `forbidden`, `insufficient_privilege` |
| `404` | Resource not found (after auth) | `not_found` |
| `405` | Method not supported by this endpoint | `method_not_allowed` (check the `Allow` header) |
| `409` | Conflict | `conflict`, `already_archived`, `not_archived` |
| `413` | Upload too large (`POST /v1/documents`) | `upload_too_large` |
| `415` | Unsupported `Content-Type` | `unsupported_media_type` |
| `422` | Well-formed but invalid values; **includes DB trigger/constraint rejects** | `validation_failed` |
| `500` | Unhandled server error | `internal_error` |
| `501` | Feature needs a DB change not yet available | `not_implemented` |
| `502` | Upstream model/embedding call failed (memory endpoints) | `upstream_error`, `model_not_configured` |

### Why `422` shows up a lot

MaluDB enforces a great deal of integrity in the database via triggers and constraints
(unregistered subject/verb types, bad enum values for `sensitivity`/`provenance`,
time-order on `valid_from`/`valid_to`, etc.). The server maps these PostgreSQL violations
to clean HTTP statuses:

- unique violation → `409 conflict`
- not-null / FK / check / bad-cast / trigger `RAISE` → `422 validation_failed`
- insufficient privilege → `403 insufficient_privilege`

So a `422` usually means "the value you sent is real JSON but the DB rejected it" — show
the field-level problem and let the user correct it. Populate dropdowns from the type-picker
endpoints (`/v1/subject-types`, `/v1/verb-types`, `/v1/episode-types`,
`/v1/attribute-templates`) to avoid most `422`s before they happen.

### Recommended client error handling

```ts
class MaluDbError extends Error {
  constructor(public code: string, message: string, public status: number) {
    super(message);
  }
}

// In your wrapper, after parsing the JSON body:
if (!res.ok) {
  const code = body?.error?.code ?? "unknown";
  const msg  = body?.error?.message ?? res.statusText;
  throw new MaluDbError(code, msg, res.status);
}
```

Then branch on `err.code`:
- `auth_invalid` / `auth_missing` → re-auth.
- `validation_failed` → show the message near the offending field.
- `already_archived` / `not_archived` / `conflict` → it's a state conflict, refresh and
  inform the user.
- `not_found` → the record was deleted elsewhere; refresh the list.
- `upload_too_large` → tell the user the size cap.
- `internal_error` (500) → retry-with-backoff is reasonable once, then surface.

## Debug mode

When the server has debug enabled, append `?debug=1` to any request and the response gains:

```json
{ "...": "...", "meta": { "debug": { "file": "subjects_id.php",
  "queries": [ { "sql": "...", "params": [17], "rows": 1, "dur_ms": 1.8 } ] } } }
```

This is the exact SQL behind the response — the fastest way to understand why a call
returned what it did. Production deployments typically leave debug off, so don't depend on
`meta.debug` being present.
