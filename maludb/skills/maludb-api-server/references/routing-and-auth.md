# MaluDB API server — routing, auth & live-schema mapping

## URL → file rewriting

A single `.htaccess` in `/var/www/html/` rewrites every `/v1/...` URL to exactly one PHP
file in `v1/`. Rules are matched **most-specific first** (longest path → shortest). Numeric
`{id}` segments become named query params (`id` for the first, `sub_id` for the second);
other query params flow through via `[QSA]`.

```apache
RewriteEngine On

# Handle routes (NON-numeric kind segment) — MUST precede the generic numeric rules:
#   /v1/objects/<kind>/<id>  →  objects_id.php?kind=<kind>&id=<id>
RewriteRule ^v1/objects/([a-zA-Z_][a-zA-Z0-9_-]*)/([0-9]+)$ v1/objects_id.php?kind=$1&id=$2 [QSA,L]
#   /v1/objects/<kind>       →  objects.php?kind=<kind>
RewriteRule ^v1/objects/([a-zA-Z_][a-zA-Z0-9_-]*)$ v1/objects.php?kind=$1 [QSA,L]

# 4-segment: /v1/<a>/<id>/<b>/<id>  →  <a>_id_<b>_id.php?id=…&sub_id=…
RewriteRule ^v1/([a-zA-Z][a-zA-Z0-9-]*)/([0-9]+)/([a-zA-Z][a-zA-Z0-9-]*)/([0-9]+)$ \
            v1/$1_id_$3_id.php?id=$2&sub_id=$4 [QSA,L]
# 3-segment: /v1/<a>/<id>/<b>  →  <a>_id_<b>.php?id=…
RewriteRule ^v1/([a-zA-Z][a-zA-Z0-9-]*)/([0-9]+)/([a-zA-Z][a-zA-Z0-9-]*)$ \
            v1/$1_id_$3.php?id=$2 [QSA,L]
# 2-segment: /v1/<a>/<id>  →  <a>_id.php?id=…
RewriteRule ^v1/([a-zA-Z][a-zA-Z0-9-]*)/([0-9]+)$ v1/$1_id.php?id=$2 [QSA,L]
# 1-segment: /v1/<a>  →  <a>.php
RewriteRule ^v1/([a-zA-Z][a-zA-Z0-9-]*)$ v1/$1.php [QSA,L]
```

### The naming rule (URL → file)

- Segment separator `/` → `_` in the file name.
- A numeric id position → the literal token `_id` in the file name, captured as `id` (first)
  / `sub_id` (second) in `$_GET`.
- Hyphens are **preserved** (`related-subjects` stays `related-subjects`).
- All file names lowercase.

| URL | File | `$_GET` |
|-----|------|---------|
| `/v1/subjects` | `subjects.php` | — |
| `/v1/subjects/17` | `subjects_id.php` | `id=17` |
| `/v1/subjects/17/verbs` | `subjects_id_verbs.php` | `id=17` |
| `/v1/subjects/17/verbs/42` | `subjects_id_verbs_id.php` | `id=17, sub_id=42` |
| `/v1/subjects/17/related-subjects/99` | `subjects_id_related-subjects_id.php` | `id=17, sub_id=99` |
| `/v1/notes/5/close-issue` | `notes_id_close-issue.php` | `id=5` |
| `/v1/subject-types` | `subject-types.php` | — |
| `/v1/objects/subject/17` | `objects_id.php` | `kind=subject, id=17` |

### When you need a bespoke rewrite rule

The generic 1–4-segment rules handle any path whose dynamic segments are **numeric ids**.
Add a hand-written rule (before the generic block) only when a segment is **text**:

- The object handle `/v1/objects/<kind>/<id>` (kind is text) — already present above.
- The graph ops `/v1/graph/<op>` → `graph_<op>.php` (e.g. `graph/walk` → `graph_walk.php`).
  Add: `RewriteRule ^v1/graph/([a-zA-Z]+)$ v1/graph_$1.php [QSA,L]`.

When you add a new endpoint, first check whether an existing generic rule already routes it
(most do) — you usually only create the PHP file.

## Authentication

**Bearer token on every request.** `require_auth()` (in `config/response.php`) does it:

1. Read `Authorization: Bearer <token>` (via `getallheaders()` / `$_SERVER`). Missing or not
   `Bearer <token>` → `401 auth_missing`.
2. Require the `malu_` prefix → else `401 auth_invalid`.
3. `sha256(substr($token, 5))` and look up:
   ```sql
   SELECT user_id FROM api_tokens WHERE token_hash = ? AND expires_at > now();
   ```
4. No row → `401 auth_invalid`. Found → set `$GLOBALS['__auth_user_id']` and return the int.

**Token format:** `malu_<43 url-safe base64 chars>` (~32 bytes entropy).

**Issuance/revocation is admin-only** (psql / a separate tool), **not** an API endpoint.

### `api_tokens` — live schema vs the draft spec

The idealized spec described `revoked_at` / `token_prefix` / `last_used_at`. **Those columns
do not exist** in the live DB. The live table has `token_hash`, `expires_at` (NOT NULL),
`user_id`, `restaurant_id`, `device_name`. So:

- Validation is `WHERE token_hash = ? AND expires_at > now()` (expiry, not revocation).
- There is no `last_used_at` update (the column is absent).
- Tokens are **never** logged in full — only redacted; the SQL trace strips tokens from the
  request URI too.

## SQLSTATE → HTTP status mapping

The global handler (`handle_uncaught` in `config/response.php`) turns DB integrity errors
into clean statuses, so endpoint code can let trigger/constraint violations propagate:

| SQLSTATE | PostgreSQL meaning | HTTP | `code` |
|----------|--------------------|------|--------|
| `23505` | unique_violation | 409 | `conflict` |
| `42501` | insufficient_privilege | 403 | `insufficient_privilege` |
| `23502` | not_null_violation | 422 | `validation_failed` |
| `23503` | foreign_key_violation | 422 | `validation_failed` |
| `23514` | check_violation | 422 | `validation_failed` |
| `22000` / `22023` / `22P02` | data exception / invalid value / bad cast | 422 | `validation_failed` |
| `P0001` | trigger `RAISE EXCEPTION` | 422 | `validation_failed` |
| (other) | — | 500 | `internal_error` |

Catch a `PDOException` explicitly only when you want a *different* result than this default —
e.g. to attach a precise message (`skills_id_duplicate.php` catches a "not forkable" error and
returns `422 validation_failed` with `pg_error_message($e)`).

## Live-schema mapping (build against reality)

The original requirements assumed names that the live DB (`zozocal`) doesn't have. Endpoints
are built against the **real** views; the public JSON contract is preserved by **aliasing in
SQL**. The key mappings (the full list is in api-server-requirements §4.0):

| API concept | Live DB source |
|-------------|----------------|
| `subjects` resource | view `maludb_subject` |
| subject `id` / `label` / `type` | `subject_id` / `canonical_name AS label` / `subject_type` |
| subject id assignment | **no sequence** — `MAX(subject_id)+1` at insert |
| `verbs` | view `maludb_verb` (`verb_id`, `canonical_name`, `verb_type`); no `label` alias; `MAX+1` ids; null type → `other` |
| subject↔verb links | `maludb_subject_verb`, keyed by **text** (`subject_name`, `verb_name`) |
| `projects` | `maludb_project` (= subjects where type='project'); `name`→`canonical_name`; archive via `maludb_project_archive` |
| `pools` | `maludb_memory_pool`; `name`→`pool_name`, `description`→`task_objective`; sequence id; no DELETE |
| `skills` | `maludb_skill`; `name`→`skill_name`; fork via `maludb_skill_fork`; DELETE ok |
| `documents` | metadata `maludb_document` + bytes `maludb_source_package.content_bytes` (bytea); ids sequenced; size+sha256 computed by the API |
| `notes` | `maludb_memory`; `body`→`summary`, `type`→`memory_kind` (`issue` enables close/reopen), `project_id`→`payload_jsonb.project_id` |
| episodes/statements/attributes | facade views/functions → require `db_tx_core()` |
| Auth | `api_tokens` validated on `expires_at` (no `revoked_at`) |
| Logs | `/var/log/maludb/` if writeable, else `/var/www/var/log/` |

**Rule of thumb:** when the spec and the live schema disagree, the live schema wins, and you
preserve the public field name by aliasing (`canonical_name AS label`). Document the mapping
in the endpoint file's top comment.
