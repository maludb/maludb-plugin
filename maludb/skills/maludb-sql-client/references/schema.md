# MaluDB SQL — schema reference

The objects you query. Three layers:

- **Updatable views** (`maludb_*`) — what you read/write. Triggers enforce integrity.
- **Facade functions** — covered in [facades.md](facades.md).
- **`malu$*` base tables** — internal; **never** reference them directly. The views and
  facades exist so you don't have to.

> Many views are **VIEWS, not tables** (e.g. `maludb_subject`, `maludb_verb` have relkind
> `v`). INSERT/UPDATE/DELETE flow through fine, but **triggers** do the integrity work —
> an unregistered `subject_type`/`verb_type` is rejected, a `null` verb type normalizes to
> `other`, etc.

## Contents

- [Graph nodes: subjects & verbs](#graph-nodes-subjects--verbs)
- [Links & relationships](#links--relationships)
- [Type pickers](#type-pickers)
- [Episodes & statements](#episodes--statements)
- [Typed attributes & templates](#typed-attributes--templates)
- [Unified graph](#unified-graph)
- [Documents](#documents)
- [Notes, projects, pools, skills](#notes-projects-pools-skills)
- [Auth & memory config](#auth--memory-config)
- [API field ↔ DB column map](#api-field--db-column-map)

---

## Graph nodes: subjects & verbs

### `maludb_subject` (view)
Columns: `subject_id` (bigint — **no sequence**, see CRUD), `canonical_name`,
`subject_type`, `description`, `classifier_md`, `created_at`, `archived_at`.
The REST API calls `canonical_name` "label" and `subject_type` "type".

### `maludb_verb` (view)
Columns: `verb_id` (**no sequence**), `canonical_name`, `verb_type`, `description`,
`classifier_md`. A `null` `verb_type` is normalized to `'other'` by a trigger.

### `maludb_subject_with_attributes` / `maludb_episode_with_attributes` / `maludb_document_with_attributes`
Same rows as the base view plus an `attributes` jsonb column. Use these to fetch a node
with its typed attributes in one query (resolves under `db_tx_core`/search_path).

## Links & relationships

### `maludb_subject_verb` (view)
Keyed by **text**, not ids: `subject_name`, `verb_name` (both = the respective
`canonical_name`). So "verbs linked to a subject" = rows where
`subject_name = <subject canonical_name>`, joined to `maludb_verb` on `canonical_name`.
The list count `linked_verbs` = `count(*) WHERE subject_name = canonical_name`;
`linked_subjects` = `count(*) WHERE verb_name = canonical_name`.

### `maludb_subject_relationship` (view)
Columns: `relationship_id`, `from_subject_id`, `to_subject_id`, `from_subject_label`,
`to_subject_label`, `relationship_type`, `label`, `valid_from`, `valid_to`. A subject's
relationships are rows where it is either endpoint. The DB enforces `valid_from <=
valid_to`.

## Type pickers

| View | Purpose |
|------|---------|
| `maludb_subject_type` | Valid `subject_type` values (writes are rejected if unregistered). |
| `maludb_verb_type` | Valid `verb_type` values. |
| `maludb_episode_type` | Advisory event-kind picker (case-insensitive unique label). |
| `maludb_attribute_template` | The attribute form catalog (see below). |

Read these before inserting subjects/verbs so you only use registered types.

## Episodes & statements

### `maludb_episode` (view, writable; create via facade)
Columns: `episode_id` (sequence), `episode_kind`, `title`, `summary`, `payload_jsonb`,
`occurred_at`, `occurred_until`, `recorded_at`, `sensitivity`, `lifecycle_state`,
`provenance`, `created_at`. Create through `maludb_register_episode(...)`; UPDATE/DELETE
through the view. `sensitivity ∈ {public,internal,restricted,prohibited}`.

### `maludb_svpor_statement` (view — the edge layer)
A statement is `(subject_kind, subject_id) --verb_id--> (object_kind, object_id)`.
Columns: `statement_id` (sequence), `subject_kind`, `subject_id`, `verb_id`, `object_kind`,
`object_id`, `predicate_id`, `valid_from`, `valid_to`, `confidence`, `provenance`,
`source_package_id`, `metadata_jsonb`, `created_at`. Create/transition via the
`maludb_svpor_statement_*` facades; idempotent on the five-tuple.
`*_kind ∈ {subject, verb, document, episode_object, memory, source_package, claim, fact,
memory_detail_object}`.

## Typed attributes & templates

### `maludb_svpor_attribute` (view, writable)
A typed property on any node **or** edge, addressed by `(target_kind, target_id)`
(`target_kind` also accepts `svpor_statement` for edge attributes). Columns: `attribute_id`,
`target_kind`, `target_id`, `attr_name`, and the typed value columns
`value_timestamp` / `value_range (tstzrange)` / `value_numeric` / `value_text` /
`value_jsonb`, plus `unit`, `provenance`, `confidence`, `valid_from`, `valid_to`,
`metadata_jsonb`, `ref_source` / `ref_entity` / `ref_key`, `created_at`. Upsert on
`(target_kind, target_id, attr_name)` via `maludb_svpor_attribute_create(...)`.

### `maludb_attribute_template` (view)
The form catalog. `applies_to ∈ {episode_type, document_type, subject_type, verb}`;
`value_type ∈ {timestamp, tstzrange, numeric, text, jsonb, reference}`;
`requirement ∈ {required, recommended, optional}`. Create/delete only (no update).

## Unified graph

### `maludb_edge` (view, read-only)
SVO statements + lineage unified into one edge view. Columns: `edge_store`, `edge_id`,
`source_kind`, `source_id`, `rel`, `target_kind`, `target_id`, `confidence`, `provenance`.
Traversed by `maludb_graph_neighbors(...)` / `maludb_graph_walk(...)` (see
[graph.md](graph.md)).

## Documents

### `maludb_document` (view, direct-INSERT)
Metadata: `document_id` (sequence), `source_package_id`, `title`, `source_type`,
`media_type`, `document_type`, `primary_project_id`, `metadata_jsonb`, `created_at`.

### `maludb_source_package` (view, direct-INSERT)
The bytes: `source_package_id` (sequence), `source_type`, `content_bytes` (**bytea** — bind
as a LOB), `media_type`, `content_size`, `content_hash` (sha256 hex), `ingested_at`.

### `maludb_document_tag` (view)
Soft tags wiring a document to subjects/projects: `tag_id`, `document_id`, `tag_kind`
(`project`/`subject`/`stakeholder`), `tag_value` (the name), `tag_object_type`,
`tag_object_id`, `provenance`. `maludb_upload_document(...)` and the document graph helpers
mirror tags into real edges.

## Notes, projects, pools, skills

| View | Notes |
|------|-------|
| `maludb_memory` | "Notes": `memory_id` (sequence), `title`, `summary` (=body), `memory_kind` (default `note`; `issue` enables `issue_closed_at`), `payload_jsonb` (carries `project_id`). |
| `maludb_project` | View of `maludb_subject WHERE subject_type='project'`. `subject_id` is the project id; `canonical_name` = name; has `archived_at`. Archive via `maludb_project_archive(id)`. |
| `maludb_memory_pool` | `pool_id` (sequence), `pool_name` (=name), `task_objective` (=description), `lifecycle_state`, `archived_at`, `creation_kind`. No DELETE granted. |
| `maludb_skill` | `skill_id` (sequence), `skill_name` (=name), `description`, `version`, `visibility`, `packaging_kind`, `enabled`, `owner_schema`, `source_skill_id`. DELETE allowed. Fork via `maludb_skill_fork(...)`. |

## Auth & memory config

### `api_tokens` (table)
The REST server validates against this; SQL middleware usually relies on its own DB role
rather than these tokens. Columns (live schema): `token_hash` (sha256 hex of the part after
`malu_`), `expires_at` (NOT NULL), `user_id`, `restaurant_id`, `device_name`. Validation:
`WHERE token_hash = ? AND expires_at > now()`. (Note: the idealized spec's `revoked_at` /
`token_prefix` / `last_used_at` columns do **not** exist in the live DB.)

### Memory model config
Not plain tables — read/written through `maludb_memory_model_config(...)` /
`maludb_memory_set_model_config(...)` and the provider/alias/secret facades. See
[vector-memory.md](vector-memory.md).

## API field ↔ DB column map

If you're also reading the maludb-api-client docs, the REST layer renames things:

| API field | DB column |
|-----------|-----------|
| subject `label` | `canonical_name` |
| subject/verb `type` | `subject_type` / `verb_type` |
| project `name` | `canonical_name` |
| pool `name` / `description` | `pool_name` / `task_objective` |
| skill `name` | `skill_name` |
| note `body` / `type` / `project_id` | `summary` / `memory_kind` / `payload_jsonb.project_id` |
| document `description` | `metadata_jsonb->>'description'` |
| episode `kind` | `episode_kind` |
| `*_id` (e.g. subject `id`) | `*_id` (e.g. `subject_id`) |
| attribute `metadata` | `metadata_jsonb` |
