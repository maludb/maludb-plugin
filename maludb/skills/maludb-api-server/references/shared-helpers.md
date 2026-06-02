# MaluDB API server — shared helpers (config/response.php)

The only shared application code. Endpoints `require_once __DIR__ .
'/../../config/response.php';` and use **only** these functions. Knowing what already exists
prevents you from re-implementing auth, logging, or transaction handling per endpoint.

## Auth

### `require_auth(): int`
Validates the `Authorization: Bearer malu_...` header and returns the `user_id`, or emits a
`401` and exits. Call it first in every endpoint. Internally: extract the token, require the
`malu_` prefix, sha256 the remainder, look up
`api_tokens WHERE token_hash = ? AND expires_at > now()`. Also sets
`$GLOBALS['__auth_user_id']` (used in the SQL/api logs).

### `bearer_token(): ?string`
Pulls the raw token from `getallheaders()` / `$_SERVER['HTTP_AUTHORIZATION']` /
`REDIRECT_HTTP_AUTHORIZATION`, matching `Bearer <token>`. You rarely call this directly.

## Request parsing

### `body_json(): array`
Decodes `php://input` as JSON (UTF-8). Empty body → `[]`. Malformed → `400 body_invalid_json`.
Non-object JSON → `400 bad_request`. Use for every JSON `POST`/`PATCH`/`PUT`.

### `path_id(): int` / `path_sub_id(): int`
Return `(int)$_GET['id']` / `$_GET['sub_id']` (populated by the `.htaccess` rewrite), or
`400` if absent/non-numeric. Use on `/{id}` and `/{id}/.../{sub_id}` routes.

### `query_int(string $name, ?int $default = null, ?int $max = null): ?int`
Read+validate a query int. Non-digit → `400`. Clamps to `$max`. Use for `limit`, ids in
query string, etc.: `$limit = query_int('limit', 50, 200);`.

### `query_str(string $name, ?string $default = null, int $max_len = 200): ?string`
Read a query string, truncated to `$max_len`. Use for `q`, `kind`, `with`, etc.

## Responses (both `exit` — no `break`/`return` needed after)

### `json_response($data, int $status = 200): never`
Emits `$data` as JSON with the status, adds `meta.debug` when `?debug=1` + `DEBUG_ENABLED`.
Always wrap under a resource key: `json_response(['subjects' => $rows]);`.

### `json_error(string $code, string $message, int $status): never`
Emits `{error:{code,message}}` with the status. Use a **stable** `code` and a developer
`message`. Common: `json_error('missing_field', 'Field "x" is required.', 400)`,
`json_error('not_found', 'Subject not found.', 404)`,
`json_error('validation_failed', '...', 422)`.

## Database — use these, never raw PDO (except bytea)

All three prepare, execute, time, and append to `sql.log` automatically.

### `db_query(string $sql, array $params = []): array`
SELECT → `fetchAll()` (array of assoc rows). Empty result → `[]`.

### `db_one(string $sql, array $params = []): ?array`
First row or `null`. Use for single-row reads, and for facade calls that
`RETURNING`/`SELECT facade(...) AS id`.

### `db_exec(string $sql, array $params = []): int`
INSERT/UPDATE/DELETE → affected row count. Use the count to detect not-found
(`if ($n === 0) json_error('not_found', ...)`).

### `db_tx_core(callable $fn)`
**Run `$fn` inside a transaction with `SET LOCAL search_path TO public, maludb_core`.** The
callback receives the shared PDO handle; any `db_query`/`db_one`/`db_exec` inside it shares
the transaction and the search_path. Commits on success, rolls back and rethrows on any
throw (the global handler then maps the SQLSTATE). **Required** for every facade-backed
endpoint (episodes, statements, attributes, graph, object handle, documents-as-graph,
memory). Example:

```php
$episode = db_tx_core(function ($pdo) use ($title, ...) {
    $row = db_one("SELECT maludb_register_episode(p_title => ?, ...) AS id", [$title, ...]);
    return db_one("SELECT " . EPISODE_COLS . " FROM maludb_episode WHERE episode_id = ?",
                  [(int) $row['id']]);
});
```

### `db_one_redacted(string $sql, array $params, array $redact): ?array`
Like `db_one`, but the 1-based indexes in `$redact` are logged as `<redacted>` instead of
their value. Use when a param is a secret (e.g. the model token in `memory/config`):
`db_one_redacted("SELECT ... secret_set(p_value => ?)", [$name, $token], [2]);`.

## Domain helpers (already written — reuse, don't re-implement)

These encapsulate multi-step graph logic. They assume the right search_path, so call them
inside `db_tx_core()`.

### Statements & attributes
- `svpor_create_statement(array $body, ?array $force_object = null): array` — full parse +
  validate + name-resolve + create + read-back for a statement. `$force_object =
  ['kind'=>…,'id'=>…]` pins the object (episode-scoped route). All validation runs before any
  write. Used by `statements.php` and `episodes_id_statements.php`.
- `svpor_create_attribute(array $body, ?array $force_target = null): array` — same for an
  attribute (upsert on target+attr_name). Used by `attributes.php`.
- `svpor_statement_cols()` / `shape_statement(&$r)` and
  `svpor_attribute_cols()` / `shape_attribute(&$r)` — the read-side column lists and
  scalar-normalizers for those rows.
- `attach_attributes(array &$rows, string $view, string $pk_col)` — for `?with=attributes`:
  batch-fetch the `attributes` jsonb from a `maludb_*_with_attributes` view and attach to each
  row by id. `$view`/`$pk_col` are endpoint constants, never user input.

### Documents-as-graph
- `document_link_subject(int $doc, string $tag_kind, string $name, $prov='provided'): ?int`
  — resolve-or-create a subject (without clobbering its type), create the
  `document→subject` edge, and record the soft tag. `tag_kind ∈
  {project→concerns, subject→mentions, stakeholder→involves}`.
- `document_unlink_subject(...)` — inverse; also repoints `primary_project_id` if needed.
- `document_neighbors(int $subject_id): array` — `[{id,title,rel}]` documents linked to a
  subject/project via the graph.

### Memory pipeline (the API is the model worker)
- `mem_chunk($text, $max=2000, $overlap=200): array` — boundary-aware text chunking.
- `mem_embed($text, $cfg=[]): array` — embed via HTTP if configured, else a **deterministic**
  local vector (so the pipeline round-trips with no creds). `mem_embed_dim()` is the length.
- `mem_vector_literal(array $floats): string` — render floats as a `'[..]'` body to cast to
  `::maludb_core.malu_vector`.
- `mem_extract($chunk, $cfg): array` — LLM extraction → `candidate_edges`.
- `mem_resolve_token(?string $ref): ?string` — DB-stored secret (via `__secret_resolve`) →
  plaintext, env fallback.
- `mem_http_post($url, $headers, $json): string` — JSON POST over cURL with a timeout; maps
  transport/upstream errors to `502 upstream_error`.

## Constants & globals

- `DEBUG_ENABLED` — `getenv('MALUDB_DEBUG') === '1'`.
- `$GLOBALS['__auth_user_id']` — `'anon'` until `require_auth()` sets the int user id (used in
  logs).
- `$GLOBALS['__sql_trace']` — per-request query buffer feeding `meta.debug`.
- `maludb_log_dir()` — resolves the writeable log directory.
- `endpoint_file()`, `iso_now_ms()`, `api_log()`, `pg_error_message()` — logging utilities.
