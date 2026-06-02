# MaluDB SQL — facade function catalog

The facade functions are the safe, supported way to create and transition MaluDB graph
data. They resolve names, assign ids, enforce idempotency, and stamp ownership/provenance.
All take **named arguments** (`p_name => value`). Unless noted, call them **inside** a
transaction with `SET LOCAL search_path TO public, maludb_core;`.

Calling convention: `SELECT facade(p_x => $1, ...) AS id;` — they're scalar functions
returning an id (or a jsonb), so wrap in `SELECT`.

## Contents

- [Subjects & verbs](#subjects--verbs)
- [Episodes](#episodes)
- [Statements (edges)](#statements-edges)
- [Typed attributes](#typed-attributes)
- [Objects (the handle)](#objects-the-handle)
- [Documents](#documents)
- [Projects & skills](#projects--skills)
- [Memory pipeline](#memory-pipeline)

---

## Subjects & verbs

### `register_svpor_subject(...) → bigint subject_id`
Create-or-resolve a subject. Reuses an existing subject by `canonical_name` rather than
duplicating (and does **not** override an existing one's type).
```sql
SELECT register_svpor_subject(
         p_canonical_name => $1,
         p_description    => $2,      -- optional
         p_subject_type   => $3,      -- e.g. 'person', 'project', 'other'
         p_classifier_md  => $4       -- optional
       ) AS subject_id;
```
The 2-arg form `register_svpor_subject(p_canonical_name => $1, p_subject_type => 'person')`
is common for "ensure a person exists".

### `maludb_core.resolve_svpor_verb(text) → bigint verb_id`
Resolve a verb id by name (use when creating statements by verb name). Returns NULL for an
unknown verb — treat as validation failure.
```sql
SELECT maludb_core.resolve_svpor_verb($1) AS verb_id;
```

### `maludb_core.resolve_svpor_predicate(text) → bigint predicate_id`
Same, for predicates.

## Episodes

### `maludb_register_episode(...) → bigint episode_id`
```sql
SELECT maludb_register_episode(
         p_episode_kind   => $1,             -- free text, default 'activity'
         p_title          => $2,             -- required
         p_summary        => $3,
         p_payload_jsonb  => $4::jsonb,       -- '{}'::jsonb if none
         p_occurred_at    => $5::timestamptz,
         p_occurred_until => $6::timestamptz,
         p_sensitivity    => $7,             -- {public,internal,restricted,prohibited}
         p_provenance     => $8              -- {provided,suggested,accepted,rejected}
       ) AS episode_id;
```
SECURITY INVOKER → that's exactly why it needs `search_path public, maludb_core` (public
first for tenant ownership, maludb_core for base tables). Read back from `maludb_episode`.

### `maludb_episode_get(episode_id) → jsonb`
Aggregate read: `{ episode, statements[], details[] }` with labels resolved.

## Statements (edges)

A statement is `(subject_kind, subject_id) --verb_id--> (object_kind, object_id)`.

### `maludb_svpor_statement_create(...) → bigint statement_id`
**Idempotent** on the five-tuple (re-linking returns the existing id).
```sql
SELECT maludb_svpor_statement_create(
         p_subject_kind      => $1,            -- e.g. 'subject', 'document'
         p_subject_id        => $2,
         p_verb_id           => $3,            -- resolve via resolve_svpor_verb()
         p_object_kind       => $4,            -- {subject,verb,document,episode_object,...}
         p_object_id         => $5,
         p_predicate_id      => $6,            -- optional
         p_valid_from        => $7::timestamptz,
         p_valid_to          => $8::timestamptz,
         p_confidence        => $9::numeric,
         p_provenance        => $10,           -- default 'provided'
         p_source_package_id => $11,
         p_metadata_jsonb    => $12::jsonb
       ) AS statement_id;
```

### `maludb_svpor_statement_set_provenance(statement_id, provenance)`
The accept/reject transition (`suggested` → `accepted`|`rejected`). DB-enforced enum.

### `maludb_svpor_statement_close(statement_id, valid_to timestamptz)`
Close the statement's validity (it was true, now it isn't). `COALESCE($2, now())` is the
common pattern.

### `maludb_svpor_statement_delete(statement_id)`
Remove it.

## Typed attributes

A typed property on any node or edge, addressed by `(target_kind, target_id)`. Upsert on
`(target_kind, target_id, attr_name)`.

### `maludb_svpor_attribute_create(...) → bigint attribute_id`
```sql
SELECT maludb_svpor_attribute_create(
         p_target_kind     => $1,             -- node kind OR 'svpor_statement' (edge attr)
         p_target_id       => $2,
         p_attr_name       => $3,
         p_value_timestamp => $4::timestamptz, -- set the ONE column that matches the type
         p_value_range     => $5::tstzrange,
         p_value_numeric   => $6::numeric,
         p_value_text      => $7,
         p_value_jsonb     => $8::jsonb,
         p_unit            => $9,
         p_provenance      => $10,            -- default 'provided'
         p_confidence      => $11::numeric,
         p_valid_from      => $12::timestamptz,
         p_valid_to        => $13::timestamptz,
         p_metadata_jsonb  => $14::jsonb,
         p_ref_source      => $15, p_ref_entity => $16, p_ref_key => $17   -- 'reference' type
       ) AS attribute_id;
```

### `maludb_svpor_attribute_set_provenance(attribute_id, provenance)`
Accept/reject transition (the only mutation — re-create to change a value).

### `maludb_svpor_attribute_delete(attribute_id)`

### `maludb_attribute_template_create(...)` / `maludb_attribute_template_delete(id)`
Manage the form catalog. Bad enum on `applies_to`/`value_type`/`requirement` raises → 422.

### `maludb_attribute_check(target_kind, target_id) → jsonb`
Advisory completeness: `{ applies_to, type_value, missing_required[], fields[] }`. The DB
never rejects on missing attributes — this is for your form layer to validate on submit.

### `maludb_attributes_apply(kind, id, attributes jsonb) → int`
Apply an array of attribute objects to a target in one call (used by the atomic
object-create path). Each element: `{ attr_name, value_*?, unit?, provenance?, confidence?,
ref_* }`.

## Objects (the handle)

The `(object_kind, object_id)` handle is the canonical identifier across the graph/attribute
surface. `object_kind ∈ {subject, verb, document, episode_object, memory, source_package,
claim, fact, memory_detail_object, svpor_statement}`.

### `maludb_object_get(kind, id) → jsonb`
`{ kind, id, object, attributes, [statements, details] }`. NULL (or a null-object envelope)
for an unknown handle.

### Atomic object + attributes (pattern, not a single function)
```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;
-- 1) register the object via its register_* helper (subject or episode_object)
SELECT register_svpor_subject(p_canonical_name => $1, p_subject_type => 'person') AS id;
-- 2) apply attributes atomically
SELECT maludb_attributes_apply('subject', <id>, $2::jsonb);
-- 3) return the assembled handle
SELECT maludb_object_get('subject', <id>) AS obj;
COMMIT;
```

## Documents

### `maludb_upload_document(...) → bigint document_id`
One-call document ingest that also wires graph edges from the names you pass.
```sql
SELECT maludb_upload_document(
         p_title         => $1, p_content_text => $2, p_source_type => $3,
         p_media_type    => $4, p_document_type => $5,
         p_projects      => $6::text[], p_subjects => $7::text[],
         p_verbs         => $8::text[], p_events   => $9::text[],
         p_metadata_jsonb => $10::jsonb
       ) AS document_id;
```
(For binary uploads you insert `maludb_source_package` + `maludb_document` yourself — see
[crud.md](crud.md) — then optionally link via the document graph helpers.)

### `maludb_document_graph_backfill() → int`
Onboarding/idempotent: wires existing documents into the graph. Returns the count linked.

## Projects & skills

### `maludb_project_archive(project_id)`
Sets `archived_at`. Check `maludb_project.archived_at` first to distinguish
"already archived".

### `maludb_skill_fork(source_owner_schema, source_skill_id, new_name, new_version) → bigint`
Fork a publishable/forkable skill. A non-forkable source raises → 422.

## Memory pipeline

See [vector-memory.md](vector-memory.md) for the full pipeline. The functions:

- `maludb_memory_model_config(namespace) → jsonb` — read the bound model config.
- `maludb_memory_set_model_config(p_extraction_alias, p_prompt_template, p_embedding_model,
  p_namespace, p_generation_params, p_default_subject_type, p_default_provenance)` — bind it.
- `maludb_register_model_provider(p_name, p_kind, p_adapter_name, p_secret_ref,
  p_data_sensitivity) → id` — per-tenant self-service provider registration.
- `maludb_register_model_alias(p_alias, p_provider, p_model_identifier, p_context_length,
  p_runtime_params jsonb) → id` — base_url rides in `runtime_params`.
- `maludb_core.secret_set(p_name, p_kind, p_value) → secret_id` — store a token encrypted
  (redact the value from logs). `maludb_core.__secret_resolve(name) → text` reads it back
  (needs the secret-consumer grant).
- `maludb_memory_ingest_edge(...) → statement_id` — store one embedded SVPO edge.
- `maludb_memory_search(...) → TABLE` — ANN search within a `(subject, verb)` compartment.
