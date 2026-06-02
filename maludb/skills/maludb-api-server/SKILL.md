---
name: maludb-api-server
description: >-
  Build or extend the MaluDB PHP API server — the JSON+HTTP layer that fronts the
  `maludb_core` PostgreSQL extension. Use this whenever the user is writing or modifying the
  server side of MaluDB: adding a new `/v1/...` endpoint, creating the one-file-per-endpoint
  PHP files under `v1/`, wiring `.htaccess` URL→file rewrites, using the shared
  `config/response.php` helpers (`require_auth`, `body_json`, `json_response`, `json_error`,
  `db_query`/`db_exec`/`db_one`, `db_tx_core`), implementing Bearer-token auth, SQL tracing,
  the error→HTTP-status mapping, or the multipart document upload. Trigger even when the user
  just says "add an endpoint to the MaluDB API", "the v1 PHP files", "the MaluDB server", or
  "edit response.php", without saying "API server". For calling the API from a client use
  maludb-api-client; for raw SQL against the DB use maludb-sql-client.
---

# MaluDB API server

This skill helps you build and extend the **MaluDB PHP API server** — the thin, no-framework
JSON layer that exposes the `maludb_core` PostgreSQL extension over HTTPS. It is one of the
application types this plugin helps create.

If the task is *calling* the API, use **maludb-api-client**. If it's writing raw SQL against
the database (the same SQL these endpoints run), use **maludb-sql-client**.

## The three design goals (everything follows from these)

1. **Simplicity.** Smallest possible code per endpoint. **No framework, no router, no ORM, no
   Composer dependency.** Plain PHP + PDO.
2. **SQL traceability.** From a URL, a developer reaches the SQL behind a failing request in
   two clicks: URL → file (a mechanical name transform) → query (literal text in the file).
3. **One file per endpoint.** Every `/v1/...` URL path maps to exactly one PHP file under
   `v1/`. HTTP methods are switched *inside* the file.

Keep these in mind for every change: if you're reaching for a router, a base controller, or
an abstraction layer, stop — the architecture is deliberately flat.

## Server layout

```
/var/www/
├── config/
│   ├── database.php      PDO singleton (PostgreSQL)
│   └── response.php      The ONLY shared app code: auth/response/db helpers (<~1k lines)
└── html/
    ├── .htaccess         URL → file rewrite rules
    └── v1/               one PHP file per endpoint path
/var/log/maludb/          sql.log, api.log, php.log (writeable by www-data)
```

`DocumentRoot` is `/var/www/html`; `config/` is **not** web-accessible. Every endpoint starts
with `require_once __DIR__ . '/../../config/response.php';`.

## The anatomy of an endpoint file (memorize this shape)

```php
<?php
/**
 * /v1/<path>  (one-line purpose + live-schema mapping notes)
 *   GET   ...
 *   POST  ...
 */
require_once __DIR__ . '/../../config/response.php';

require_auth();                       // 401s on a bad/missing/expired Bearer token
$id = path_id();                      // only on /{id} routes

switch ($_SERVER['REQUEST_METHOD']) {
    case 'GET': {
        $limit = query_int('limit', 50, 200);
        $rows  = db_query("SELECT ... LIMIT $limit", [...]);
        foreach ($rows as &$r) { $r['id'] = (int) $r['id']; }   // normalize scalar types
        unset($r);
        json_response(['resource_plural' => $rows]);            // wrap under a named key
    }
    case 'POST': {
        $body = body_json();
        if (trim((string)($body['x'] ?? '')) === '')
            json_error('missing_field', 'Field "x" is required.', 400);
        $created = db_one("INSERT ... RETURNING ...", [...]);
        $created['id'] = (int) $created['id'];
        json_response(['resource_singular' => $created], 201);
    }
    default:
        header('Allow: GET, POST');
        json_error('method_not_allowed', 'This endpoint supports GET and POST.', 405);
}
```

The `case` blocks end in `json_response`/`json_error`, both of which `exit` — so no `break`
is needed and execution never falls through. See
[references/endpoint-patterns.md](references/endpoint-patterns.md) for every variant (detail,
PATCH, DELETE, action endpoints, multipart upload, facade/`db_tx_core` endpoints).

## The four rules every endpoint obeys

1. **Auth first.** Call `require_auth()` (returns `user_id`, or emits `401` and exits) before
   any work. For sub-resource routes, read `path_id()` / `path_sub_id()` next.
2. **Validate before writing.** Do all shape checks (`json_error 400/422`) *before* any DB
   write, so a rejected request never half-creates rows. Required field missing → `400
   missing_field`; well-formed-but-bad value → `422 validation_failed`.
3. **Only ever use the DB helpers** — `db_query` / `db_exec` / `db_one` (and `db_tx_core`,
   `db_one_redacted`). They prepare, execute, and **log to sql.log** automatically. Dropping
   to raw PDO bypasses tracing (only do it for bytea LOB binds, and call `sql_log()` manually
   — see the documents pattern).
4. **Wrap responses under a named key and normalize scalar types.** PDO returns everything as
   strings; cast ids to `(int)`, numerics to `(float)`, decode jsonb columns. A list →
   `{plural: [...]}`; a single resource → `{singular: {...}}`; a delete → `{deleted:true,id}`.

## The shared helpers (config/response.php)

This is the only shared code. Know what's there before writing an endpoint — full signatures
in [references/shared-helpers.md](references/shared-helpers.md):

- **Auth:** `require_auth(): int`, `bearer_token(): ?string`.
- **Request:** `body_json(): array`, `path_id()`, `path_sub_id()`, `query_int()`,
  `query_str()`.
- **Response:** `json_response($data, $status=200): never`, `json_error($code,$msg,$status):
  never`.
- **DB:** `db_query()`, `db_exec()`, `db_one()`, `db_tx_core(callable)`,
  `db_one_redacted()`.
- **Domain helpers** (already written, reuse them): `svpor_create_statement()`,
  `svpor_create_attribute()`, `attach_attributes()`, the `document_link_subject()` /
  `document_unlink_subject()` / `document_neighbors()` graph helpers, and the `mem_*` memory
  pipeline helpers.
- **Global error handling:** a `set_exception_handler` maps PostgreSQL SQLSTATEs to clean
  HTTP statuses, logs to api.log, and never leaks a blank 500.

## Two things that trip people up

- **`db_tx_core()` is mandatory for the facade surface.** Episodes, statements, attributes,
  the unified graph (`edges`/`graph/*`), the object handle, documents-as-graph, and the whole
  memory pipeline call `maludb_*` facades that need
  `SET LOCAL search_path TO public, maludb_core`. `db_tx_core(fn)` does exactly that inside a
  transaction. Plain CRUD on the simple views (`maludb_subject` etc.) does **not** need it.
  See [references/endpoint-patterns.md](references/endpoint-patterns.md).
- **Live schema ≠ the draft spec.** The original requirements assumed column names
  (`subjects.label`, `api_tokens.revoked_at`) that don't exist; endpoints are built against
  the real views (`maludb_subject.canonical_name AS label`, `api_tokens.expires_at`). When in
  doubt, trust the live schema and alias in SQL. See
  [references/routing-and-auth.md](references/routing-and-auth.md).

## Adding a new endpoint — checklist

1. **Name the file** from the URL using the rewrite rules (`/` → `_`, numeric segments → the
   literal `_id`, hyphens preserved, lowercase). E.g. `/v1/subjects/{id}/verbs` →
   `subjects_id_verbs.php`. See [references/routing-and-auth.md](references/routing-and-auth.md).
2. **Confirm `.htaccess` already routes it** — the generic 1–4 segment rules cover most paths;
   only text (non-numeric) segments like the object `{kind}` handle or `/graph/<op>` need a
   bespoke rule.
3. **Copy the closest existing pattern** from
   [references/endpoint-patterns.md](references/endpoint-patterns.md) (list+create, detail
   CRUD, action, multipart, or facade/`db_tx_core`).
4. **Use the shared helpers**; never add a second shared file or a framework.
5. **Validate before writing**, normalize scalar types, wrap under the resource key.
6. **Document the methods + live-schema mapping** in the file's top comment, and add the row
   to the requirements doc's endpoint table.
