# MaluDB API — data model (client view)

As a client you work with **API field names**. The database underneath uses different
names; the server aliases between them. You never send DB names — but knowing the mapping
exists explains the payloads and helps when you also read the maludb-sql-client docs.

## The conceptual model

MaluDB is a knowledge graph plus a memory pipeline:

- **Subjects** and **verbs** are graph nodes. A subject is a person/concept/project/etc.;
  a verb names a relation or action.
- **Statements** are directed edges: `(subject) --verb--> (object)`. The `object` can be
  another subject, a verb, a document, an episode, etc. — addressed by a `(kind, id)`
  **handle**. Statements carry temporal bounds, confidence, and provenance.
- **Episodes** are events (an activity, a meeting, a decision) with time bounds and a JSON
  payload. Statements link participants/artifacts to an episode.
- **Attributes** are typed properties hung on any node *or* edge — addressed by the same
  `(target_kind, target_id)` handle. A template catalog defines which attributes apply to
  which kinds, and an advisory check reports completeness.
- **Documents** are content (files/transcripts). On upload they become graph nodes, wired
  to the projects/subjects you name.
- **Projects**, **pools**, **skills**, **notes** are higher-level containers. A *project is
  itself a subject* (`subject_type='project'`).
- **Vector memory** is the RAG layer: a document's text is chunked, SVPO edges are
  extracted, each is embedded, and search runs an ANN query inside a `(subject, verb)`
  compartment.

## API field ↔ DB column map

You send/receive the left column. The right is what the server stores (and what
maludb-sql-client uses). This is why, e.g., you POST `label` but a SQL user updates
`canonical_name`.

| Resource | API field | DB column / source |
|----------|-----------|--------------------|
| subjects | `id` | `maludb_subject.subject_id` |
| subjects | `label` | `canonical_name` |
| subjects | `type` | `subject_type` |
| subjects | `description`, `classifier_md` | same |
| verbs | `id` | `maludb_verb.verb_id` |
| verbs | `canonical_name` | `canonical_name` (no alias) |
| verbs | `type` | `verb_type` (null → `other`) |
| projects | resource `projects` | view `maludb_project` (= subjects where type='project') |
| projects | `name` | `canonical_name` |
| projects | `archived_at` | `maludb_subject.archived_at` |
| pools | `name`, `description` | `pool_name`, `task_objective` |
| skills | `name` | `skill_name` |
| documents | `id` | `maludb_document.document_id` |
| documents | `content_size` | `maludb_source_package.content_size` |
| documents | `description` | `metadata_jsonb->>'description'` |
| notes | `title` | `title` |
| notes | `body` | `summary` |
| notes | `type` | `memory_kind` (default `note`; `issue` enables close/reopen) |
| notes | `project_id` | `payload_jsonb.project_id` |
| episodes | `id`, `kind` | `episode_id`, `episode_kind` |
| statements | `id` | `statement_id` |
| attributes | `id`, `metadata` | `attribute_id`, `metadata_jsonb` |

## Controlled vocabularies (enums)

The DB rejects out-of-set values → `422`. Offer these in your UI:

- **`provenance`** ∈ `{ provided, suggested, accepted, rejected }` — the review lifecycle.
  - `provided` = explicit user input (trusted).
  - `suggested` = machine-extracted, awaiting review (the review queue).
  - `accepted` / `rejected` = the human decision (set via PATCH).
- **`sensitivity`** (episodes) ∈ `{ public, internal, restricted, prohibited }`.
- **`subject_type` / `verb_type`** — open sets, but each value must be **registered**.
  Fetch the allowed values from `/v1/subject-types` and `/v1/verb-types`.
- **object/target `kind`** ∈ `{ subject, verb, document, episode_object, memory,
  source_package, claim, fact, memory_detail_object, svpor_statement }`.
- **attribute `value_type`** (templates) ∈ `{ timestamp, tstzrange, numeric, text, jsonb,
  reference }`; `applies_to` ∈ `{ episode_type, document_type, subject_type, verb }`;
  `requirement` ∈ `{ required, recommended, optional }`.
- **model provider `kind`** (memory config) ∈ `{ cloud_api, local_http, local_socket,
  local_runtime, shell_adapter, stub }`.

## Typed attribute values

An attribute carries exactly one value column matched to its type — set the one that fits
and leave the rest null:

| `value_type` | field to set | example |
|--------------|--------------|---------|
| `timestamp` | `value_timestamp` | `"2026-01-15T09:00:00Z"` |
| `tstzrange` | `value_range` | `"[2026-01-01,2026-02-01)"` |
| `numeric` | `value_numeric` | `42.5` |
| `text` | `value_text` | `"on hold"` |
| `jsonb` | `value_jsonb` | `{ "tags": ["a","b"] }` |
| `reference` | `ref_source` / `ref_entity` / `ref_key` | external pointer |

## List vs detail (the denormalization rule)

To keep lists cheap, **list rows carry counts, detail carries collections**:

- `GET /v1/subjects` → each row has `linked_verbs` and `related_subjects` as **integers**.
- `GET /v1/subjects/{id}` → the same names are **arrays** (`verbs[]`,
  `related_subjects[]`), plus `documents[]`.
- `GET /v1/verbs` → each row has `linked_subjects` (count).

So: render lists from the list endpoint, and lazy-load the detail endpoint when the user
opens a record. Don't loop the list firing per-row detail calls.

## Identity quirks (rarely visible, occasionally surprising)

- Subject and verb ids have **no sequence** — the server assigns `MAX(id)+1` at insert.
  Practically invisible to a client, but it means ids are dense and you should treat the
  `id` from the create response as authoritative (don't predict it).
- Pools, skills, documents, episodes, statements, and attributes use normal sequences.
