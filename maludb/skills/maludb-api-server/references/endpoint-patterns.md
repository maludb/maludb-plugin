# MaluDB API server — endpoint patterns

Every endpoint is one of a handful of shapes. Find the closest, copy it, adapt the SQL.
All patterns: `require_once` the helpers, `require_auth()`, branch on the method, validate
before writing, normalize scalar types, wrap the response under a named key, end each branch
in `json_response`/`json_error` (both `exit`).

## Contents

1. [List + create (simple view)](#1-list--create-simple-view)
2. [Detail: GET / PATCH / DELETE](#2-detail-get--patch--delete)
3. [Action endpoint (archive / close-issue)](#3-action-endpoint)
4. [Sub-resource link / unlink](#4-sub-resource-link--unlink)
5. [Facade endpoint (db_tx_core)](#5-facade-endpoint-db_tx_core)
6. [Multipart upload (bytea + manual log)](#6-multipart-upload)
7. [Method-restricted endpoint](#7-method-restricted-endpoint)

---

## 1. List + create (simple view)

For resources on a plain updatable view (subjects, verbs, notes, pools, skills, projects).

```php
<?php
require_once __DIR__ . '/../../config/response.php';
require_auth();

switch ($_SERVER['REQUEST_METHOD']) {
    case 'GET': {
        $q     = query_str('q', null, 200);
        $limit = query_int('limit', 50, 200);
        $where = ''; $params = [];
        if ($q !== null && $q !== '') {
            $where = "WHERE s.canonical_name ILIKE ? OR s.description ILIKE ?";
            $params[] = "%$q%"; $params[] = "%$q%";
        }
        // $limit is server-derived + clamped, so inlining it is safe (it's an int).
        $rows = db_query(
            "SELECT s.subject_id AS id, s.canonical_name AS label, s.subject_type AS type,
                    s.description, s.classifier_md
               FROM maludb_subject s $where
              ORDER BY s.canonical_name LIMIT $limit", $params);
        foreach ($rows as &$r) { $r['id'] = (int) $r['id']; } unset($r);

        if (query_str('with', null, 40) === 'attributes') {
            attach_attributes($rows, 'maludb_subject_with_attributes', 'subject_id');
        }
        json_response(['subjects' => $rows]);
    }
    case 'POST': {
        $body  = body_json();
        $label = trim((string) ($body['label'] ?? ''));
        if ($label === '') json_error('missing_field', 'Field "label" is required.', 400);
        // subject_id has no sequence — derive MAX+1 in the same statement (atomic)
        $created = db_one(
            "INSERT INTO maludb_subject (subject_id, canonical_name, subject_type, created_at)
             SELECT COALESCE(MAX(subject_id),0)+1, ?, ?, now() FROM maludb_subject
             RETURNING subject_id AS id, canonical_name AS label, subject_type AS type",
            [$label, $body['type'] ?? null]);
        $created['id'] = (int) $created['id'];
        json_response(['subject' => $created], 201);
    }
    default:
        header('Allow: GET, POST');
        json_error('method_not_allowed', 'This endpoint supports GET and POST.', 405);
}
```

> **Why inlining `$limit` is OK but `$q` is not:** `$limit` comes from `query_int()` which
> guarantees an int; `$q` is user text and **must** be a bound `?` param. Never interpolate
> user strings into SQL.

## 2. Detail: GET / PATCH / DELETE

For `/v1/<resource>/{id}`. Note the dynamic PATCH SET-list built only from present fields.

```php
<?php
require_once __DIR__ . '/../../config/response.php';
require_auth();
$id = path_id();

switch ($_SERVER['REQUEST_METHOD']) {
    case 'GET': {
        $row = db_one("SELECT subject_id AS id, canonical_name AS label, subject_type AS type
                         FROM maludb_subject WHERE subject_id = ?", [$id]);
        if ($row === null) json_error('not_found', 'Subject not found.', 404);
        $row['id'] = (int) $row['id'];
        json_response(['subject' => $row]);
    }
    case 'PATCH': {
        if (db_one("SELECT 1 FROM maludb_subject WHERE subject_id = ?", [$id]) === null)
            json_error('not_found', 'Subject not found.', 404);
        $body = body_json();
        $fields = []; $params = [];
        if (array_key_exists('label', $body)) {
            $label = trim((string) $body['label']);
            if ($label === '') json_error('validation_failed', 'Field "label" cannot be empty.', 422);
            $fields[] = 'canonical_name = ?'; $params[] = $label;
        }
        if (array_key_exists('type', $body)) {              // pass null through deliberately
            $fields[] = 'subject_type = ?'; $params[] = $body['type'] === null ? null : (string) $body['type'];
        }
        if (!$fields) json_error('bad_request', 'No updatable fields provided.', 400);
        $params[] = $id;
        db_exec("UPDATE maludb_subject SET " . implode(', ', $fields) . " WHERE subject_id = ?", $params);
        json_response(['subject' => /* reload */ db_one("SELECT subject_id AS id, canonical_name AS label FROM maludb_subject WHERE subject_id = ?", [$id])]);
    }
    case 'DELETE': {
        $n = db_exec("DELETE FROM maludb_subject WHERE subject_id = ?", [$id]);
        if ($n === 0) json_error('not_found', 'Subject not found.', 404);
        json_response(['deleted' => true, 'id' => $id]);
    }
    default:
        header('Allow: GET, PATCH, DELETE');
        json_error('method_not_allowed', 'This endpoint supports GET, PATCH and DELETE.', 405);
}
```

Patterns to copy: `array_key_exists` (not `isset`) so an explicit `null` is distinguishable
from omitted; "must exist before update" check → `404`; affected-count → `404` on DELETE; a
`load_*_detail()` helper function when the GET response embeds related collections (see
`subjects_id.php` for the full version with `verbs[]`/`related_subjects[]`/`documents[]`).

## 3. Action endpoint

For `/v1/<resource>/{id}/<action>` (archive, unarchive, close-issue, reopen-issue). POST-only,
with a precondition check that returns `409` on a state conflict.

```php
<?php
require_once __DIR__ . '/../../config/response.php';
require_auth();
$id = path_id();

switch ($_SERVER['REQUEST_METHOD']) {
    case 'POST': {
        $project = db_one("SELECT archived_at FROM maludb_project WHERE subject_id = ?", [$id]);
        if ($project === null)            json_error('not_found', 'Project not found.', 404);
        if ($project['archived_at'] !== null) json_error('already_archived', 'Project is already archived.', 409);
        db_one("SELECT maludb_project_archive(?)", [$id]);      // facade
        $updated = db_one("SELECT subject_id AS id, canonical_name AS name, archived_at
                             FROM maludb_project WHERE subject_id = ?", [$id]);
        $updated['id'] = (int) $updated['id'];
        json_response(['project' => $updated]);
    }
    default:
        header('Allow: POST');
        json_error('method_not_allowed', 'This endpoint supports POST only.', 405);
}
```

The `409` precondition check is the signature of an action endpoint — distinguish "already
archived" / "not an issue" / "not closed" from a generic error.

## 4. Sub-resource link / unlink

For `/v1/<a>/{id}/<b>` (POST to link) and `/v1/<a>/{id}/<b>/{sub_id}` (DELETE to unlink). The
second file reads both `path_id()` and `path_sub_id()`.

```php
// subjects_id_verbs.php  — POST {verb_id}
require_auth();  $id = path_id();
case 'POST': {
    $body = body_json();
    if (!isset($body['verb_id']) || !is_int($body['verb_id']))
        json_error('validation_failed', '"verb_id" must be an integer.', 422);
    // ... INSERT the link keyed by canonical_name ...
    json_response(['linked' => true], 201);
}

// subjects_id_verbs_id.php  — DELETE
require_auth();  $id = path_id();  $vid = path_sub_id();
case 'DELETE': {
    db_exec("DELETE FROM maludb_subject_verb WHERE subject_name = (SELECT canonical_name FROM maludb_subject WHERE subject_id = ?) AND verb_name = (SELECT canonical_name FROM maludb_verb WHERE verb_id = ?)", [$id, $vid]);
    json_response(['deleted' => true]);
}
```

`PUT` for "replace the whole set" (e.g. `projects_id_subjects.php` accepts both `POST
{subject_id}` and `PUT {subject_ids:[...]}`) follows the same file with a `case 'PUT'`.

## 5. Facade endpoint (db_tx_core)

For anything touching the SVPOR facades: episodes, statements, attributes, edges, graph,
the object handle, documents-as-graph, memory. **Wrap the DB work in `db_tx_core()`** so the
`search_path` is set and multiple writes are one transaction.

```php
<?php
require_once __DIR__ . '/../../config/response.php';
require_auth();

const EPISODE_COLS = "episode_id AS id, episode_kind AS kind, title, summary,
                      payload_jsonb AS payload, occurred_at, sensitivity, provenance, created_at";
function shape_episode(array &$e): void {
    $e['id'] = (int) $e['id'];
    $e['payload'] = $e['payload'] === null ? null : json_decode($e['payload']); // object, not assoc
}

switch ($_SERVER['REQUEST_METHOD']) {
    case 'GET': {
        $limit = query_int('limit', 50, 200);
        $rows = db_tx_core(fn() => db_query(
            "SELECT " . EPISODE_COLS . " FROM maludb_episode
              ORDER BY occurred_at DESC NULLS LAST, episode_id DESC LIMIT $limit", []));
        foreach ($rows as &$r) { shape_episode($r); } unset($r);
        json_response(['episodes' => $rows]);
    }
    case 'POST': {
        $body  = body_json();
        $title = trim((string) ($body['title'] ?? ''));
        if ($title === '') json_error('missing_field', 'Field "title" is required.', 400);
        // do all validation BEFORE opening the tx
        $payload = isset($body['payload']) && is_array($body['payload']) ? json_encode($body['payload']) : '{}';
        $episode = db_tx_core(function ($pdo) use ($title, $payload, $body) {
            $row = db_one("SELECT maludb_register_episode(
                             p_episode_kind => ?, p_title => ?, p_summary => ?,
                             p_payload_jsonb => ?::jsonb, p_sensitivity => ?, p_provenance => ?) AS id",
                          [$body['kind'] ?? 'activity', $title, $body['summary'] ?? null,
                           $payload, $body['sensitivity'] ?? 'internal', $body['provenance'] ?? 'provided']);
            return db_one("SELECT " . EPISODE_COLS . " FROM maludb_episode WHERE episode_id = ?", [(int) $row['id']]);
        });
        shape_episode($episode);
        json_response(['episode' => $episode], 201);
    }
    default:
        header('Allow: GET, POST');
        json_error('method_not_allowed', 'This endpoint supports GET and POST.', 405);
}
```

Conventions in this pattern:
- **Validate before `db_tx_core()`** — a `json_error` inside the callback would throw out of
  the transaction; the helper rolls back, but doing shape checks first is cleaner and avoids
  half-resolved rows.
- **Named facade args** (`p_x => ?`) with explicit `::jsonb`/`::timestamptz`/`::numeric` casts
  on the placeholders.
- **`json_decode($col)`** (not assoc) for jsonb columns so empty objects stay `{}` not `[]`.
- For statements/attributes, prefer the prebuilt `svpor_create_statement()` /
  `svpor_create_attribute()` helpers inside `db_tx_core()` rather than re-writing the facade
  call.
- A scoped readback within the same tx (e.g. `maludb_object_get(kind,id)`) returns the
  assembled resource.

## 6. Multipart upload

`POST /v1/documents` is the one non-JSON endpoint. Bytes bind as a `PDO::PARAM_LOB` on the
raw handle, so log that one statement manually (bytes redacted); everything else uses the
helpers.

```php
case 'POST': {
    // oversize body arrives with empty $_FILES/$_POST
    if (empty($_FILES) && empty($_POST) && (int)($_SERVER['CONTENT_LENGTH'] ?? 0) > 0)
        json_error('upload_too_large', 'Upload exceeded the server size limit.', 413);
    if (!isset($_FILES['file'])) json_error('missing_field', 'Missing "file" part.', 400);
    $err = $_FILES['file']['error'];
    if ($err === UPLOAD_ERR_INI_SIZE || $err === UPLOAD_ERR_FORM_SIZE)
        json_error('upload_too_large', 'Uploaded file exceeds the size limit.', 413);
    if ($err !== UPLOAD_ERR_OK) json_error('bad_request', "File upload failed ($err).", 400);

    $bytes = file_get_contents($_FILES['file']['tmp_name']);
    $size  = strlen($bytes);
    $hash  = hash('sha256', $bytes);

    // bytea LOB → raw PDO + MANUAL sql_log (redact the bytes)
    $pdo = Database::getInstance()->getConnection();
    $t0  = microtime(true);
    $sql = "INSERT INTO maludb_source_package (source_type, content_bytes, media_type, content_size, content_hash, ingested_at)
            VALUES ('document', ?, ?, ?, ?, now()) RETURNING source_package_id";
    $stmt = $pdo->prepare($sql);
    $stmt->bindValue(1, $bytes, PDO::PARAM_LOB);
    $stmt->bindValue(2, $mime); $stmt->bindValue(3, $size, PDO::PARAM_INT); $stmt->bindValue(4, $hash);
    $stmt->execute();
    $spid = (int) $stmt->fetchColumn();
    sql_log($sql, ['<'.$size.' bytes>', $mime, $size, $hash], 1, (microtime(true)-$t0)*1000);

    $doc = db_one("INSERT INTO maludb_document (source_package_id, title, source_type, media_type, document_type, metadata_jsonb, created_at)
                   VALUES (?, ?, 'document', ?, ?, ?, now()) RETURNING document_id AS id, title", [...]);
    // optional graph wiring of comma-separated projects/subjects inside db_tx_core():
    // foreach (...) document_link_subject($doc['id'], 'project', $name);
    json_response(['document' => $doc], 201);
}
```

This is the **only** sanctioned raw-PDO path, and it still logs manually — never create an
untraceable query.

## 7. Method-restricted endpoint

When an endpoint supports a single method (most `graph/*`, `edges`, `memory/*`, `*-check`,
`*-backfill`), short-circuit instead of a `switch`:

```php
require_auth();
if ($_SERVER['REQUEST_METHOD'] !== 'GET') {
    header('Allow: GET');
    json_error('method_not_allowed', 'This endpoint supports GET.', 405);
}
$kind = query_str('kind', null, 40);
if ($kind === null || $kind === '') json_error('missing_field', 'Query param "kind" is required.', 400);
$rows = db_tx_core(fn() => db_query("SELECT ... FROM maludb_graph_neighbors(?, ?, ?, ...)", [...]));
json_response(['neighbors' => $rows]);
```
