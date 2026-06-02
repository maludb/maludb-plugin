# MaluDB API server — architecture & operations

The non-functional contract: hosting, logging, SQL tracing, debug mode, and the deployment
assumptions. These are what make the "two-click URL → SQL" traceability goal real.

## Hosting & transport

| | |
|---|---|
| Host | `api.maludb.com` (configurable) |
| TLS | HTTPS only at the edge |
| OS | Ubuntu 24.04 LTS |
| Web server | Apache 2.4 + `mod_rewrite`, `mod_headers`, `mod_deflate` |
| PHP | 8.2+ with `pdo`, `pdo_pgsql`, `mbstring`, `json`, `fileinfo` |
| PostgreSQL | 14+ with the `maludb_core` extension installed |

- `DocumentRoot` = `/var/www/html`. `config/` lives at `/var/www/config` — **outside** the
  docroot, so it is never web-served.
- No Composer, no autoloader, no namespace. The only shared file is `config/response.php`
  (which `require`s `config/database.php`).
- **CORS is not configured** in v1 — the canonical client is Electron, not a browser. If you
  add a browser client, you'll need to add CORS headers (and rethink token handling).

## Request/response conventions

- **Request body:** `application/json`, UTF-8 — **except** `POST /v1/documents`
  (`multipart/form-data`).
- **Response:** always `application/json; charset=utf-8`, including errors.
- **Method handling:** each file branches on `$_SERVER['REQUEST_METHOD']` and returns
  `405 method_not_allowed` (with an `Allow:` header) for unsupported verbs.
- **Success bodies** are wrapped under a named key (`{subjects:[...]}`, `{subject:{...}}`).
- **Error bodies** are `{ "error": { "code": "...", "message": "..." } }`.

## Logging

Three files under `/var/log/maludb/` (the dir must be writeable by `www-data`; the helper
falls back to `/var/www/var/log` then `sys_get_temp_dir()` in dev without root):

- **`sql.log`** — every query run through `db_query`/`db_exec`/`db_one`, one block each:
  timestamp (ISO-8601 UTC ms), endpoint file, method, request URI (**tokens stripped**),
  `user_id`, the literal SQL, bound params (JSON), row count, duration ms.
- **`api.log`** — one line per request plus full stack traces for any `500`.
- **`php.log`** — PHP `error_log` destination.

**Token redaction is absolute:** full bearer tokens are never written to any log. Secrets
passed as SQL params (e.g. a model token in `memory/config`) are logged via
`db_one_redacted()`, which replaces the secret positions with `<redacted>`.

Rotation: daily via `logrotate`, retain 14 days.

## SQL tracing — the mechanism

This is goal #2 (traceability) in action. `db_query`/`db_exec`/`db_one` each:

1. `prepare()` the SQL on the shared PDO handle.
2. `execute()` with the positional params.
3. Time the call, count rows (returned for SELECT, affected for write).
4. Append a block to `sql.log` **and** push the same record onto an in-request buffer
   (`$GLOBALS['__sql_trace']`) for debug mode.

Because *every* endpoint uses only these helpers, the trace is complete. The rare exception
(binary `bytea` LOB binds, which need raw PDO) calls `sql_log()` manually with the bytes
redacted — see the documents pattern. **Never** drop to raw PDO without a manual `sql_log()`;
that creates an untraceable query and breaks the contract.

## Debug mode

When `?debug=1` is present **and** the server constant `DEBUG_ENABLED` is `true`, every JSON
response gains:

```json
"meta": { "debug": { "file": "subjects_id.php",
  "queries": [ { "sql": "...", "params": [17], "rows": 1, "dur_ms": 1.8 } ] } }
```

`DEBUG_ENABLED` derives from `getenv('MALUDB_DEBUG') === '1'` (off by default in production —
set via Apache `SetEnv MALUDB_DEBUG 1` or a `config/env.php`). `json_response()` only injects
`meta.debug` when both conditions hold and the payload is an array.

## Global error handling

`config/response.php` installs a `set_exception_handler('handle_uncaught')` and a shutdown
function so an uncaught throw never produces a blank 500. The handler:

- Logs `summary + stack` to `api.log` (stack only for `>= 500`).
- For `PDOException`, reads the SQLSTATE and maps it (see
  [routing-and-auth.md](routing-and-auth.md) for the full table): `23505 → 409 conflict`,
  `42501 → 403`, `23502/23503/23514/22000/22023/22P02/P0001 → 422 validation_failed`, else
  `500 internal_error`.
- Emits the standard `{error:{code,message}}` body with the mapped status.

This is why endpoint code can simply *let a DB trigger violation propagate* — the handler
turns it into a clean `422` rather than a crash. You only catch explicitly when you want a
**different** code than the default mapping (e.g. `skills_id_duplicate.php` catches a
"not forkable" `PDOException` to return `422 validation_failed` with the precise message).

## Database connection

`config/database.php` is a PDO singleton (`Database::getInstance()->getConnection()`):

- DSN: `pgsql:host=...;port=5432;dbname=...;sslmode=...`.
- Options: `ERRMODE_EXCEPTION` (so failures throw → the global handler),
  `DEFAULT_FETCH_MODE = FETCH_ASSOC`, `EMULATE_PREPARES = false` (real prepared statements),
  non-persistent.
- One connection per request, shared across all helper calls — which is what lets
  `db_tx_core()` wrap multiple `db_*` calls in one transaction with one `search_path`.

> Credentials currently live as class constants in `database.php`. For any real deployment,
> move them to environment variables / a non-committed config and never commit secrets.
