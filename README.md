# maludb-plugin

A Claude Code **plugin marketplace** for building [MaluDB](https://github.com/maludb)
memory applications. MaluDB is a memory / knowledge-graph system implemented as a set of
PostgreSQL extensions (`maludb_core`).

This repo is both the marketplace catalog (`.claude-plugin/marketplace.json`) and the host
of the `maludb` plugin (in [`maludb/`](maludb/)).

## The plugin: `maludb`

Three skills, one per way of building on MaluDB:

| Skill | For | Covers |
|-------|-----|--------|
| `maludb-api-client` | desktop / web app developers | Integrating the MaluDB REST/JSON API — auth, endpoints, error codes, recipes (PHP, TS/JS, Python). |
| `maludb-sql-client` | middleware developers | Querying the `maludb_core` PostgreSQL extension directly — views, facade functions, `search_path`, RLS, the vector-memory pipeline. |
| `maludb-api-server` | backend developers | Scaffolding the PHP API server — one-file-per-endpoint, shared helpers, routing, SQL tracing. |

## Install (in Claude Code)

```text
/plugin marketplace add maludb/maludb-plugin
/plugin install maludb@maludb
```

`maludb@maludb` means *plugin `maludb`* from *marketplace `maludb`* (the marketplace name
is the `name` field in `.claude-plugin/marketplace.json`). After installing, the three
skills trigger automatically based on what you're working on; manage them with `/plugin`.

## Updating

Bump the `version` in both `maludb/.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json`, commit, and push. Users then run:

```text
/plugin marketplace update maludb
```

and update the plugin from the `/plugin` menu.

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json     # marketplace catalog (source: ./maludb)
└── maludb/                  # the plugin
    ├── .claude-plugin/
    │   └── plugin.json
    └── skills/
        ├── maludb-api-client/
        ├── maludb-sql-client/
        └── maludb-api-server/
```

---

## API reference (REST endpoints)

> The full REST endpoint catalog, inlined here for convenience. The canonical copy lives in [`maludb/skills/maludb-api-client/references/endpoints.md`](maludb/skills/maludb-api-client/references/endpoints.md) — keep the two in sync when the API changes.

Every `/v1` endpoint, grouped by area. For each: HTTP methods, request body / query
params, and the response shape (always JSON, always wrapped under a named key). All
endpoints require `Authorization: Bearer malu_...` unless noted.

Conventions used below:
- `{id}` / `{sid}` / `{vid}` are numeric path segments.
- `field?` = optional; `field(req)` = required.
- Response shown is the success body; errors are always `{error:{code,message}}`.
- "Returns 201" means a created resource; otherwise 200.

### Contents

- [1. Subjects](#1-subjects)
- [2. Verbs](#2-verbs)
- [3. Type pickers](#3-type-pickers)
- [4. Episodes & statements](#4-episodes--statements)
- [5. Typed attributes](#5-typed-attributes)
- [6. Objects & unified graph](#6-objects--unified-graph)
- [7. Documents](#7-documents)
- [8. Notes](#8-notes)
- [9. Projects](#9-projects)
- [10. Memory pools](#10-memory-pools)
- [11. Skills](#11-skills)
- [12. Vector memory pipeline](#12-vector-memory-pipeline)

---

### 1. Subjects

A subject is a node in the graph (a person, concept, project, software, …). The API field
`label` maps to the DB `canonical_name`; `type` to `subject_type`.

#### `GET /v1/subjects`
Query params: `q?` (search over name + description), `limit?` (default 50, max 200),
`with?=attributes`.
Returns `{ subjects: [ { id, label, type, description, classifier_md, linked_verbs,
related_subjects, attributes? } ] }`. `linked_verbs` / `related_subjects` are **counts**,
not arrays — fetch the detail endpoint for the full collections.

#### `POST /v1/subjects`
Body: `{ label(req), type?, description?, classifier_md? }`.
Returns `201 { subject: { id, label, type, description, classifier_md, linked_verbs } }`.

#### `GET /v1/subjects/{id}`
Returns `{ subject: { id, label, type, description, classifier_md, verbs[],
related_subjects[], documents[] } }`. Here the full collections are embedded:
- `verbs[]` = `{ id, canonical_name, type }`
- `related_subjects[]` = `{ relationship_id, id, label, relationship_type,
  relationship_label, direction: "outgoing"|"incoming", valid_from, valid_to }`
- `documents[]` = `{ id, title, rel }` (documents wired to this subject via the graph)

404 `not_found` if absent.

#### `PATCH /v1/subjects/{id}`
Body (any subset): `{ label?, type?, description?, classifier_md? }`. Sending `null`
clears a nullable field. Empty `label` → `422`. Returns the full detail object.

#### `DELETE /v1/subjects/{id}`
Returns `{ deleted: true, id }`. 404 if absent. (Does **not** delete the subject's typed
attributes — see the no-cascade caveat in §5.)

#### `GET /v1/subjects/{id}/verbs`
Returns `{ ... }` listing the verbs linked to this subject (same data as the detail
embedding). Backward-compat sub-resource.

#### `POST /v1/subjects/{id}/verbs`
Body: `{ verb_id(req) }`. Links a verb to the subject.

#### `DELETE /v1/subjects/{id}/verbs/{vid}`
Unlinks the verb from the subject.

#### `GET|POST /v1/subjects/{id}/related-subjects`
GET lists related subjects. POST body: `{ related_subject_id(req), valid_from?, valid_to? }`
to create a relationship.

#### `DELETE /v1/subjects/{id}/related-subjects/{otherId}`
Pair-level unlink: removes any relationship between the two subjects.

#### `GET|PATCH|DELETE /v1/subject-relationships/{relationship_id}`
Row-level access to a single relationship (the `relationship_id` surfaced on POST/GET).
PATCH body: `{ relationship_type?, label?, valid_from?, valid_to? }` (`null` clears the
`valid_*` bounds; the DB enforces time order → `422`).

---

### 2. Verbs

A verb is a graph node naming a relation/action. Verbs expose `canonical_name` directly
(no `label` alias); `type` maps to `verb_type` (a null type normalizes to `other`).

#### `GET /v1/verbs`
Query params: `q?`, `limit?`. Returns `{ verbs: [ { id, canonical_name, type,
description, classifier_md, linked_subjects } ] }`. `linked_subjects` is a count.

#### `POST /v1/verbs`
Body: `{ canonical_name(req), type?, description?, classifier_md? }`. Returns `201`.

#### `GET|PATCH|DELETE /v1/verbs/{id}`
PATCH body: `{ canonical_name?, type?, description?, classifier_md? }`.

#### `GET /v1/verbs/{id}/subjects`
Read-only listing of the subjects linked to this verb.

---

### 3. Type pickers

#### `GET /v1/subject-types`
Returns the distinct allowed `subject_type` values. Use to populate a type dropdown.

#### `GET /v1/verb-types`
Returns the distinct allowed `verb_type` values.

These exist because the DB **rejects** an unregistered type on subject/verb writes
(trigger → `422`). Offer the user only registered types, or expect a `422`.

---

### 4. Episodes & statements

Episodes are first-class events; statements are the normalized
`(subject) --verb--> (object)` edges that link people/documents/decisions to events.

#### `GET /v1/episodes`
Query params: `q?`, `kind?`, `provenance?`, `limit?`, `with?=attributes`. Ordered newest
occurrence first. Returns `{ episodes: [ { id, kind, title, summary, payload,
occurred_at, occurred_until, recorded_at, sensitivity, lifecycle_state, provenance,
created_at, attributes? } ] }`.

#### `POST /v1/episodes`
Body: `{ title(req), kind?(default "activity"), summary?, payload?(object),
occurred_at?, occurred_until?, sensitivity?(default "internal"), provenance?(default
"provided") }`. `sensitivity ∈ {public,internal,restricted,prohibited}` and
`provenance ∈ {provided,suggested,accepted,rejected}` are DB-enforced → `422`. `kind` is
free text. Returns `201 { episode: {...} }`.

#### `GET /v1/episodes/{id}`
Returns `{ episode: { episode, statements[], details[] } }` — the episode plus its linked
statements and detail rows, labels resolved.

#### `PATCH /v1/episodes/{id}`
Body (any subset): `{ title?, summary?, kind?, payload?, occurred_at?, occurred_until?,
sensitivity?, provenance?, lifecycle_state? }`. Patching `provenance` is the accept/reject
transition.

#### `DELETE /v1/episodes/{id}`
Removes the episode.

#### `GET|POST /v1/episodes/{id}/statements`
Event-scoped statements. GET lists statements whose object is this episode. POST creates a
statement with the object defaulted to this episode (see the statement body below). 404 if
the episode is missing.

#### `GET /v1/statements`
Query params: `provenance?`, `object_kind?`, `object_id?`, `subject_kind?`, `subject_id?`,
`verb_id?`, `limit?`. The **review queue** is `?provenance=suggested`. Returns
`{ statements: [...] }`.

#### `POST /v1/statements`
**Statement body** (shared with the episode-scoped POST). A statement is
`(subject_kind, subject_id) --verb_id--> (object_kind, object_id)`. The endpoint resolves
names so you needn't pre-fetch ids:
```jsonc
{
  "verb": "upgraded",          // OR "verb_id": 42
  "subject_kind": "subject",   // default "subject"
  "subject": "Edward",         // name → create-or-resolve a person (only when kind=subject)
  "subject_id": 17,            // OR an explicit id (wins over "subject")
  "object_kind": "episode_object", // required (defaulted on the scoped route)
  "object_id": 100,            // required
  "predicate": "role",         // OR "predicate_id"; optional
  "valid_from": null, "valid_to": null,
  "confidence": 0.9,           // optional number
  "provenance": "provided",    // default "provided"
  "source_package_id": null,
  "metadata": {}               // optional object
}
```
`*_kind ∈ {subject, verb, document, episode_object, memory, source_package, claim, fact,
memory_detail_object}`. Idempotent on the five-tuple (re-linking returns the existing id).
Unknown verb/predicate name, bad kind, or FK violation on a bad id → `422`. A `document`
is a valid kind, so an uploaded document links straight to an event by `document_id`.

#### `GET|PATCH|DELETE /v1/statements/{id}`
PATCH `{ provenance? }` → set provenance (suggested → accepted|rejected);
`{ valid_to? }` or `{ close: true }` → close the statement's validity (`close:true` uses
`now()`). DELETE removes it.

#### `GET|POST /v1/episode-types`
Advisory event-kind picker. POST creates a label (case-insensitive unique → `409`).

#### `PATCH|DELETE /v1/episode-types/{id}`
Update/remove a picker entry; deleting does not affect episodes already tagged (no FK).

---

### 5. Typed attributes

A typed property on any node **or** edge, addressed by `(target_kind, target_id)`. A
template catalog drives forms; an advisory check reports completeness. Attributes follow
the same provenance review workflow as statements.

#### `GET /v1/attributes`
Query params: `target_kind?`, `target_id?`, `attr_name?`, `provenance?`, `limit?`. The
review queue is `?provenance=suggested`. Returns `{ attributes: [ { id, target_kind,
target_id, attr_name, value_timestamp, value_range, value_numeric, value_text, value_jsonb,
unit, provenance, confidence, valid_from, valid_to, metadata, created_at, ref_source,
ref_entity, ref_key } ] }`.

#### `POST /v1/attributes`
Create/**upsert** (idempotent on `target_kind+target_id+attr_name`). Body:
```jsonc
{
  "target_kind": "subject", "target_id": 17, "attr_name": "birthday",
  "value_timestamp": null, "value_range": null, "value_numeric": null,
  "value_text": null, "value_jsonb": null,        // set the one that fits the type
  "unit": null, "provenance": "provided", "confidence": null,
  "valid_from": null, "valid_to": null, "metadata": {},
  "ref_source": null, "ref_entity": null, "ref_key": null
}
```
`target_kind` = any node kind **or** `svpor_statement` (to attach an attribute to an edge).
Returns `201`.

#### `GET|PATCH|DELETE /v1/attributes/{id}`
GET one row. PATCH `{ provenance }` is the **only** mutation (the accept/reject transition);
to change a *value*, re-POST the attribute. DELETE removes it.

#### `GET|POST /v1/attribute-templates`
The form catalog. GET filter: `?applies_to=&type_value=`. POST body:
`{ applies_to, value_type, requirement, ... }` where
`applies_to ∈ {episode_type, document_type, subject_type, verb}`,
`value_type ∈ {timestamp, tstzrange, numeric, text, jsonb, reference}`,
`requirement ∈ {required, recommended, optional}`. Bad enum → `422`.

#### `GET|DELETE /v1/attribute-templates/{id}`
Read/remove one template. **No PATCH** → `405`.

#### `GET /v1/attribute-check`
Query params: `target_kind(req)`, `target_id(req)`. Returns
`{ check: { applies_to, type_value, missing_required[], fields[] } }`. **Advisory only** —
the DB never rejects on missing attributes; use this to validate a form on submit.

---

### 6. Objects & unified graph

The `(object_kind, object_id)` **handle** is the canonical identifier across the
graph/attribute surface. `object_kind ∈ {subject, verb, document, episode_object, memory,
source_package, claim, fact, memory_detail_object, svpor_statement}`.

#### `GET /v1/objects/{kind}/{id}`
Returns `{ object: { kind, id, object, attributes, [statements, details] } }` — one handle
read inline with its typed attributes (and, for episodes, statements + details). 404 on an
unknown handle.

#### `POST /v1/objects/{kind}`
**Atomic create**: register the object then apply its attributes in one transaction; either
both land or neither. Supported kinds:
- `subject`: `{ canonical_name|name|label(req), subject_type?(="other"), description?,
  classifier_md?, attributes?[] }`
- `episode_object`: `{ title(req), kind?(="activity"), summary?, payload?, occurred_at?,
  occurred_until?, sensitivity?(="internal"), provenance?, attributes?[] }`

Each `attributes[]` element: `{ attr_name, value_text?|value_numeric?|value_timestamp?|
value_range?|value_jsonb?, unit?, provenance?, confidence?, ref_source?, ref_entity?,
ref_key? }`. Other kinds → `422`. Returns `201 { object: {...} }` (a full `object_get`).

#### `GET /v1/edges`
Read-only unified edge view (SVO statements + lineage). Query params: `source_kind?`,
`source_id?`, `target_kind?`, `target_id?`, `rel?`, `edge_store?`, `limit?` (default 100,
max 500). Returns `{ edges: [ { edge_store, edge_id, source_kind, source_id, rel,
target_kind, target_id, confidence, provenance } ] }`.

#### `GET /v1/graph/neighbors`
Query params: `kind(req)`, `id(req)`, `direction?(both|out|in, default both)`, `rel?`
(comma-separated filter). One labeled hop. Returns `{ kind, id, direction, neighbors: [
{ neighbor_kind, neighbor_id, rel, edge_store, confidence, provenance, label } ] }`.

#### `GET /v1/graph/walk`
Query params: `kind(req)`, `id(req)`, `max_depth?(default 4, max 20)`, `direction?`,
`rel?`. Cycle-safe breadth-first multi-hop walk. Returns `{ ..., walk: [ { object_kind,
object_id, depth, rel, edge_store, label, path:[ids] } ] }`.

---

### 7. Documents

Metadata lives in one view; the bytes in another (bytea). Documents are first-class graph
nodes — naming projects/subjects on upload wires real `document→subject` edges.

#### `GET /v1/documents`
Query params: `q?` (title search), `limit?`, `with?=attributes`. Returns
`{ documents: [ { id, title, source_type, media_type, document_type, primary_project_id,
description, content_size, created_at, attributes? } ] }`. GET returns metadata only —
binary download is out of v1.

#### `POST /v1/documents` — multipart, NOT JSON
`Content-Type: multipart/form-data`. Parts:
- `file` (req) — the raw bytes
- `filename?`, `mime_type?`, `description?`, `document_type?` (free-text label)
- `projects?`, `subjects?` — **comma-separated names**; each becomes a `document→subject`
  graph edge + soft tag, and the first project sets `primary_project_id`.

Returns `201 { document: { id, title, source_type, media_type, document_type,
description, content_size, primary_project_id, created_at } }`. Over the size limit →
`413 upload_too_large` (default ceiling 25 MB, set by PHP `upload_max_filesize` /
`post_max_size`).

#### `GET|PATCH|DELETE /v1/documents/{id}`
GET returns metadata + `primary_project_id` + `tags[]`. PATCH body:
`{ link?: {projects[], subjects[]}, unlink?: {projects[], subjects[]} }` adds/removes graph
links. DELETE removes the document **and** its graph edges.

#### `POST /v1/documents-backfill`
Runs the tenant graph backfill (onboarding; idempotent). Returns `{ linked: <int> }`.

---

### 8. Notes

Notes are free-form memory rows. `type: "issue"` enables the close/reopen actions.

#### `GET|POST /v1/notes`
POST body: `{ title(req), body(req), type?(default "note"), project_id? }`.

#### `GET|PATCH|DELETE /v1/notes/{id}`
Standard CRUD.

#### `POST /v1/notes/{id}/close-issue`
Sets `issue_closed_at = now()` on an `issue` note. `409` if the note is not an issue.

#### `POST /v1/notes/{id}/reopen-issue`
Clears `issue_closed_at`. `409` if not an issue, or not currently closed.

---

### 9. Projects

A project IS a subject (`subject_type='project'`); its id is the subject id. API field
`name` maps to `canonical_name`.

#### `GET|POST /v1/projects`
Standard list/create.

#### `GET|PATCH|DELETE /v1/projects/{id}`
Standard CRUD. Detail returns `{ project: { id, name, description, classifier_md,
archived_at, ... } }`.

#### `POST /v1/projects/{id}/archive`
`409 already_archived` if already archived. Returns the updated project.

#### `POST /v1/projects/{id}/unarchive`
`409 not_archived` if not archived.

#### `POST|PUT /v1/projects/{id}/subjects`
POST links one: `{ subject_id }`. PUT replaces the whole set: `{ subject_ids: [...] }`.

#### `DELETE /v1/projects/{id}/subjects/{sid}`
Unlink one subject.

#### `POST|PUT /v1/projects/{id}/verbs`
Same shape with `verb_id` / `verb_ids`.

#### `DELETE /v1/projects/{id}/verbs/{vid}`
Unlink one verb.

---

### 10. Memory pools

#### `GET|POST /v1/pools`
Create body uses `name` → pool_name, `description` → task_objective.

#### `GET|PATCH /v1/pools/{id}`
**No DELETE** in v1.

#### `POST /v1/pools/{id}/archive`
Archive via `lifecycle_state='archived'`.

(`join`/`leave`/`tags`/`DELETE` are intentionally not in v1.)

---

### 11. Skills

#### `GET /v1/skills`
Query param: `visibility?` (optional filter). Create body uses `name` → skill_name.

#### `GET|PATCH|DELETE /v1/skills/{id}`
Standard CRUD (DELETE is allowed here).

#### `POST /v1/skills/{id}/duplicate`
Fork a skill. Body: `{ name?, version?(default "1.0.0") }`. Only published/forkable skills
can be forked — a non-forkable source → `422`. Returns `201 { skill: {...} }`.

---

### 12. Vector memory pipeline

The RAG layer. The **API is the model worker**: PostgreSQL cannot make outbound HTTP
calls, so the server chunks text, calls the LLM (extraction) and the embedding model, then
writes back. Every embedding in a namespace must share one model + dimension.

#### `GET /v1/memory/config`
Query param: `namespace?(default "default")`. Returns `{ namespace, config: {
extraction_alias, model_identifier, provider_kind, base_url, secret_ref, embedding_model,
prompt_template, generation_params, default_subject_type, default_provenance } }`.
`secret_ref` is the secret **name**, never the token value.

#### `POST|PUT /v1/memory/config`
Configures a tenant+namespace in one transaction: store the token encrypted, register the
provider + alias, bind the embedding model + prompt + defaults. Body:
```jsonc
{
  "namespace": "default",
  "secret_name": "openai-key", "token": "sk-...",   // token stored encrypted, never logged
  "provider": { "name": "openai", "kind": "cloud_api", "adapter_name": null,
                "data_sensitivity": "internal" },
  "alias":    { "name": "gpt-extract", "model_identifier": "gpt-4o-mini",
                "context_length": null, "base_url": "https://api.openai.com/v1" },
  "prompt_template": "... {{chunk}} ...",   // MUST contain {{chunk}} → else 422
  "embedding_model": "text-embedding-3-small",
  "generation_params": {},
  "default_subject_type": "other", "default_provenance": "suggested"
}
```
`provider.kind ∈ {cloud_api, local_http, local_socket, local_runtime, shell_adapter, stub}`.
Returns the read-back config. `secret_name` is required when a `token` is provided.

#### `POST /v1/memory/documents` — ingest
Upload → chunk → extract SVPO edges → embed → store, atomically per document. Body:
```jsonc
{
  "title": "Q2 standup", "text": "...full text...",   // both required
  "source_type": "document", "media_type": null, "document_type": null,
  "projects": [], "subjects": [], "verbs": [], "events": [],   // names → graph links
  "metadata": {}, "namespace": "default", "embedding_model": null,
  "chunk": { "max": 2000, "overlap": 200 },
  "edges": [ /* OPTIONAL pre-extracted candidate_edges → bypass the LLM */ ]
}
```
Extracted edges default to `provenance: "suggested"`. With no model creds, the server falls
back to a deterministic embedding and you can supply `edges` directly — so
upload→ingest→search round-trips offline. Returns `201 { document_id, namespace,
embedding_model, extractor, chunk_count, edges:[...] }`.

A `candidate_edge` (when you supply your own) looks like:
```jsonc
{ "subject_text": "Edward", "subject_type": "person", "verb_text": "upgraded",
  "predicate": [ { "attr_name": "status", "value_text": "done" } ],
  "source_span": "...", "confidence": 0.9, "provenance": "suggested" }
```

#### `POST /v1/memory/search`
Embed the query (same model as ingest) → ANN search within a `(subject, verb)` compartment.
Body: `{ query(req), subject?, verb?, namespace?(default "default"), limit?(default 20,
max 200), metric?(default "cosine"), embedding_model? }`. **At least one of `subject` /
`verb` is required** (the compartment pre-filter). Returns `{ namespace, embedding_model,
results: [ { chunk_id, statement_id, document_id, source_text, distance, similarity,
rank_no, subject_name, verb_name } ] }`.
