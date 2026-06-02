# MaluDB SQL — CRUD patterns

How to read and write through the views. All examples use placeholders (`$1`, `?`) — always
parameterize; never interpolate user text.

## When you need `search_path`

- **Simple views** (`maludb_subject`, `maludb_verb`, `maludb_project`, `maludb_memory`,
  `maludb_memory_pool`, `maludb_skill`, the `*_type` pickers): resolve on the default path.
  Plain CRUD works without any `SET`.
- **Graph / facade surface** (episodes, statements, attributes, edges, the object handle,
  documents-as-graph, memory pipeline): wrap in a transaction with
  `SET LOCAL search_path TO public, maludb_core;`.

Examples below note which they are.

## Subjects & verbs — and the MAX+1 id quirk

`maludb_subject.subject_id` and `maludb_verb.verb_id` have **no sequence/default**. To
insert through the view directly, derive the id in the same statement so it's atomic:

```sql
-- INSERT a subject (simple view, no search_path needed)
INSERT INTO maludb_subject
    (subject_id, canonical_name, subject_type, description, classifier_md, created_at)
SELECT COALESCE(MAX(subject_id), 0) + 1, $1, $2, $3, $4, now()
  FROM maludb_subject
RETURNING subject_id, canonical_name, subject_type, description, classifier_md;
```

> The `SELECT ... FROM maludb_subject` in the same statement makes the id derivation and the
> insert one operation. Under concurrency, wrap it in a transaction (and accept that a unique
> key on `canonical_name` may still 23505 on a race — retry). **Prefer the facade**
> `register_svpor_subject(...)` for people/SVPOR subjects: it assigns ids and resolves
> existing rows for you (see [facades.md](facades.md)).

```sql
-- UPDATE: build the SET list from only the fields present
UPDATE maludb_subject
   SET canonical_name = COALESCE($2, canonical_name),
       subject_type   = $3,            -- pass through nulls deliberately
       description    = $4
 WHERE subject_id = $1;

-- DELETE (does NOT cascade to typed attributes — clean those first; see below)
DELETE FROM maludb_subject WHERE subject_id = $1;
```

Verbs are identical with `verb_id` / `canonical_name` / `verb_type`. Remember a `null`
`verb_type` becomes `'other'` via trigger.

### Type pickers gate writes

```sql
SELECT * FROM maludb_subject_type;   -- the registered set
SELECT * FROM maludb_verb_type;
```

An `INSERT`/`UPDATE` with an unregistered `subject_type`/`verb_type` raises (SQLSTATE
`P0001`/`23514`) → treat as validation failure.

## Links

```sql
-- Verbs linked to a subject (text-keyed join)
SELECT v.verb_id, v.canonical_name, v.verb_type
  FROM maludb_subject_verb sv
  JOIN maludb_verb v ON v.canonical_name = sv.verb_name
 WHERE sv.subject_name = $1            -- the subject's canonical_name
 ORDER BY v.canonical_name;

-- A subject's relationships (either endpoint)
SELECT relationship_id, from_subject_id, to_subject_id,
       from_subject_label, to_subject_label, relationship_type, label, valid_from, valid_to
  FROM maludb_subject_relationship
 WHERE from_subject_id = $1 OR to_subject_id = $1
 ORDER BY relationship_id;
```

## Notes (`maludb_memory`)

```sql
-- Create a note (simple view). project_id rides in payload_jsonb.
INSERT INTO maludb_memory (title, summary, memory_kind, payload_jsonb, created_at)
VALUES ($1, $2, COALESCE($3, 'note'), jsonb_build_object('project_id', $4), now())
RETURNING memory_id, title, summary, memory_kind;

-- Close an issue (only when memory_kind = 'issue')
UPDATE maludb_memory SET issue_closed_at = now()
 WHERE memory_id = $1 AND memory_kind = 'issue' AND issue_closed_at IS NULL;
-- 0 rows affected → it wasn't an open issue (the API returns 409 here)
```

## Projects

A project is a subject. Create like a subject with `subject_type='project'`, or via the
project view. Archive through the facade:

```sql
SELECT subject_id AS id, canonical_name AS name, description, archived_at
  FROM maludb_project WHERE subject_id = $1;

-- Archive / check first to distinguish 'already archived'
SELECT archived_at FROM maludb_project WHERE subject_id = $1;   -- null → not archived
SELECT maludb_project_archive($1);                              -- facade; needs search_path
```

## Pools & skills

```sql
INSERT INTO maludb_memory_pool (pool_name, task_objective, creation_kind, lifecycle_state)
VALUES ($1, $2, 'api', 'active')
RETURNING pool_id, pool_name, task_objective;

UPDATE maludb_memory_pool
   SET lifecycle_state = 'archived', archived_at = now()
 WHERE pool_id = $1;          -- pools have no DELETE in the API surface

-- Skills support DELETE; fork via the facade (needs search_path)
SELECT maludb_skill_fork($1 /*owner_schema*/, $2 /*src skill_id*/, $3 /*new name*/, $4 /*version*/);
```

## Documents (binary)

The bytes go in `maludb_source_package.content_bytes` (**bytea**), metadata in
`maludb_document`. Bind the bytes as a LOB, and compute size + sha256 yourself:

```sql
-- 1) the package (bind $1 as PDO::PARAM_LOB / bytea)
INSERT INTO maludb_source_package
    (source_type, content_bytes, media_type, content_size, content_hash, ingested_at)
VALUES ('document', $1, $2, $3, $4, now())
RETURNING source_package_id;

-- 2) the metadata row
INSERT INTO maludb_document
    (source_package_id, title, source_type, media_type, document_type, metadata_jsonb, created_at)
VALUES ($1, $2, 'document', $3, $4, $5::jsonb, now())
RETURNING document_id, title;
```

To make the document a graph node (edges to its projects/subjects), either call
`maludb_upload_document(...)` (does everything in one go) or wire the edges yourself inside
`search_path` — see [graph.md](graph.md).

## Episodes, statements, attributes (facade surface)

These must run under `search_path`. The create paths are facade functions; reads/updates/
deletes can go through the view:

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;

-- read a statement row
SELECT statement_id, subject_kind, subject_id, verb_id, object_kind, object_id,
       provenance, confidence, valid_from, valid_to, metadata_jsonb
  FROM maludb_svpor_statement WHERE statement_id = $1;

-- update an episode through the view
UPDATE maludb_episode
   SET title = $2, summary = $3, provenance = $4   -- provenance change = accept/reject
 WHERE episode_id = $1;

COMMIT;
```

Create/transition them via facades — see [facades.md](facades.md).

## No-cascade caveat (important on delete)

Deleting an episode/subject does **not** delete its typed attributes (no FK cascade). When
you hard-delete a node, delete its attributes first:

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;
-- find them, then delete each via the facade (or DELETE FROM maludb_svpor_attribute)
DELETE FROM maludb_svpor_attribute WHERE target_kind = $1 AND target_id = $2;
COMMIT;
DELETE FROM maludb_subject WHERE subject_id = $2;
```

## `RETURNING` and read-back

Most writes support `RETURNING`. For facade creates that return only an id, read the row
back in the same transaction:

```sql
SELECT maludb_register_episode(/* ... */) AS id;          -- returns episode_id
SELECT * FROM maludb_episode WHERE episode_id = <id>;     -- same txn / search_path
```
