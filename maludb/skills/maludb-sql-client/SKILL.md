---
name: maludb-sql-client
description: >-
  Query and write the MaluDB PostgreSQL extension (`maludb_core`) directly with SQL from
  middleware — no REST API in between. Use this whenever the user is writing SQL against
  MaluDB: SELECT/INSERT/UPDATE/DELETE through the `maludb_*` updatable views; calling
  facade functions like `register_svpor_subject`, `maludb_register_episode`,
  `maludb_svpor_statement_create`, `maludb_svpor_attribute_create`, `maludb_memory_search`,
  `maludb_graph_walk`/`_neighbors`, `maludb_upload_document`; using the `malu_vector` type;
  setting the `search_path` for facade access; dealing with RLS/tenant ownership, the
  provenance lifecycle, or PostgreSQL constraint/trigger errors from MaluDB. Trigger even
  when the user just says "the MaluDB database", "query maludb_core", "the maludb_subject
  view", or names any `maludb_*` object, without saying "SQL client". For calling the HTTP
  API instead use maludb-api-client; for building the PHP API server use maludb-api-server.
---

# MaluDB SQL client

This skill helps middleware talk to MaluDB **directly in SQL** — against the `maludb_core`
PostgreSQL extension, with no REST layer. MaluDB is a memory / knowledge-graph store:
subjects & verbs (nodes), SVPO statements (edges), episodes (events), typed attributes,
documents, and a graph-bound vector-memory pipeline.

If the task is calling the HTTP API, use **maludb-api-client**. If it's building the PHP
endpoints, use **maludb-api-server**.

## The one rule you cannot skip: search_path

The MaluDB facade views and functions reference their internal `malu$*` base tables and RLS
grant tables **unqualified**. They only resolve when `maludb_core` is on the `search_path`,
and tenant ownership / RLS only works when the tenant (`public`) schema is **first**:

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;
-- ... your facade calls / view writes here ...
COMMIT;
```

- `public` first → `current_schema()` = the tenant schema, so `owner_schema` resolution and
  row-level security apply to the caller's tenant.
- `maludb_core` in the path → the facade's base tables resolve.
- `SET LOCAL` scopes the change to the transaction; use it inside `BEGIN`/`COMMIT`.

**Which operations need it?** Everything that touches the SVPOR graph, episodes, statements,
attributes, the unified edge/graph functions, the object handle, documents-as-graph, and the
whole memory pipeline. **Plain CRUD on the simple views** (`maludb_subject`, `maludb_verb`,
`maludb_project`, `maludb_memory_pool`, `maludb_skill`, the `*_type` pickers) resolves on the
default path and does **not** need it. When unsure, wrap it — it's harmless on the simple
views and required for the rest. See [references/conventions.md](references/conventions.md).

## The shapes you write against

MaluDB exposes three kinds of SQL objects (never touch the `malu$*` base tables directly):

1. **Updatable views** — `maludb_subject`, `maludb_verb`, `maludb_project`, `maludb_memory`
   (notes), `maludb_memory_pool`, `maludb_skill`, `maludb_document`, `maludb_episode`,
   `maludb_svpor_statement`, `maludb_svpor_attribute`, `maludb_edge`, …. You can
   `SELECT`/`INSERT`/`UPDATE`/`DELETE` through most of them; triggers enforce integrity.
2. **Facade functions** — the safe way to create/transition graph data:
   `register_svpor_subject(...)`, `maludb_register_episode(...)`,
   `maludb_svpor_statement_create(...)` / `_set_provenance` / `_close` / `_delete`,
   `maludb_svpor_attribute_create(...)`, `maludb_attributes_apply(...)`,
   `maludb_object_get(...)`, `maludb_graph_neighbors/_walk(...)`, `resolve_svpor_verb(...)`,
   the `maludb_memory_*` pipeline, `maludb_upload_document(...)`. Signatures use **named
   args** (`p_foo => ?`). Full catalog in [references/facades.md](references/facades.md).
3. **Type pickers & catalogs** — `maludb_subject_type`, `maludb_verb_type`,
   `maludb_episode_type`, `maludb_attribute_template`. The DB **rejects** an unregistered
   subject/verb type, so read these to know the valid set.

Where to look:
- View names, columns, and the API↔DB name map → [references/schema.md](references/schema.md)
- INSERT/UPDATE/DELETE patterns (incl. the MAX+1 id quirk) → [references/crud.md](references/crud.md)
- Function signatures → [references/facades.md](references/facades.md)
- Edges, neighbors, walk, the `(kind,id)` handle → [references/graph.md](references/graph.md)
- `malu_vector`, model config, ingest, search → [references/vector-memory.md](references/vector-memory.md)
- RLS, provenance, id quirks, SQLSTATE meanings → [references/conventions.md](references/conventions.md)

## Three quick examples

**Read subjects (no facade needed — simple view):**
```sql
SELECT subject_id, canonical_name, subject_type, description
FROM maludb_subject
WHERE canonical_name ILIKE '%edward%'
ORDER BY canonical_name LIMIT 50;
```

**Create a subject + an episode + link them (facades → needs search_path):**
```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;

SELECT register_svpor_subject(p_canonical_name => 'Edward', p_subject_type => 'person') AS subject_id;

SELECT maludb_register_episode(
         p_episode_kind => 'activity', p_title => 'Server upgrade',
         p_summary => 'Upgraded prod', p_payload_jsonb => '{}'::jsonb,
         p_occurred_at => now(), p_sensitivity => 'internal', p_provenance => 'provided'
       ) AS episode_id;

-- link the subject to the episode via a statement (resolves the verb by name)
SELECT maludb_svpor_statement_create(
         p_subject_kind => 'subject', p_subject_id => <subject_id>,
         p_verb_id      => maludb_core.resolve_svpor_verb('performed'),
         p_object_kind  => 'episode_object', p_object_id => <episode_id>,
         p_provenance   => 'provided');
COMMIT;
```

**Semantic search within a compartment (vector memory):**
```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;
SELECT chunk_id, document_id, source_text, similarity
FROM maludb_memory_search(
       p_query_embedding => '[0.12, -0.04, ...]'::maludb_core.malu_vector,
       p_subject => 'Edward', p_verb => NULL,
       p_namespace => 'default', p_limit => 10, p_metric => 'cosine');
COMMIT;
```

## Working principles

- **Always use parameterized queries** (`$1`/`?`), never string interpolation — the same SQL
  injection rules apply, and several facades take user-derived text (names, spans).
- **Prefer facades over raw view writes** for anything in the SVPOR graph (statements,
  attributes, episodes, subjects-as-people): they resolve names, assign ids, enforce
  idempotency, and stamp provenance/ownership correctly. Raw `INSERT`s into the simple views
  are fine for plain subject/verb/project/note/pool/skill rows.
- **Expect a `RAISE`/constraint, not a crash.** MaluDB pushes integrity into triggers — an
  unregistered type, a bad enum, an out-of-order time range all surface as a PostgreSQL
  error with a clear SQLSTATE. Map them like the API does (see conventions): `23505` →
  conflict, `23502/23503/23514/22*/P0001` → validation, `42501` → privilege.
- **One document = one transaction.** Multi-write flows (upload + ingest edges; object +
  attributes) belong in a single `BEGIN`/`COMMIT` so partial graphs never persist.
