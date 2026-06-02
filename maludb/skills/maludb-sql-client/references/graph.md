# MaluDB SQL — the unified graph

MaluDB unifies SVO statements and lineage into one **edge** view, with set-returning
functions for one-hop and multi-hop traversal. All of this is facade surface → run inside
`BEGIN; SET LOCAL search_path TO public, maludb_core; ... COMMIT;`.

## The handle

Every graph node is identified by an `(object_kind, object_id)` **handle**:
`object_kind ∈ {subject, verb, document, episode_object, memory, source_package, claim,
fact, memory_detail_object, svpor_statement}`. The traversal functions take and return
handles, so you can pivot from any hit to `maludb_object_get(kind, id)` for the full record.

## `maludb_edge` — the unified edge view (read-only)

```sql
SELECT edge_store, edge_id, source_kind, source_id, rel, target_kind, target_id,
       confidence, provenance
  FROM maludb_edge
 WHERE source_kind = $1 AND source_id = $2     -- any subset of filters
 ORDER BY edge_store, edge_id DESC
 LIMIT $3;
```
Filterable columns: `source_kind`, `source_id`, `target_kind`, `target_id`, `rel`,
`edge_store`. `edge_store` tells you which underlying store the edge came from (statements
vs lineage).

## `maludb_graph_neighbors(...)` — one hop

```sql
SELECT neighbor_kind, neighbor_id, rel, edge_store, confidence, provenance, label
  FROM maludb_graph_neighbors(
         $1,        -- kind   (text)
         $2,        -- id     (bigint)
         $3,        -- direction: 'both' | 'out' | 'in'  (default 'both')
         $4         -- rel_filter text[]  (NULL = all rels)
       );
```
Pass a rel filter as a real array, or NULL. From a comma-separated string:
```sql
... maludb_graph_neighbors($1, $2, $3,
      CASE WHEN $4 = '' THEN NULL ELSE string_to_array($4, ',') END) ...
```
`label` is the human-readable name of the neighbor — handy for UI without a second lookup.

## `maludb_graph_walk(...)` — multi-hop, cycle-safe

```sql
SELECT object_kind, object_id, depth, rel, edge_store, label, path
  FROM maludb_graph_walk(
         $1,        -- kind
         $2,        -- id
         $3,        -- max_depth  (default 4; keep modest — breadth grows fast)
         $4,        -- direction  ('both'|'out'|'in')
         $5         -- rel_filter text[]  (NULL = all)
       );
```
Breadth-first from the start handle. Each row is a reached object with its `depth`, the
`rel` that reached it, and `path` — a PostgreSQL `text[]` / int array of the object ids
walked to get there (e.g. `{17,42,99}`). Decode `path` in your driver (it comes back as a
`{...}` literal over some drivers).

## Documents as graph nodes

A document is wired into the graph by mirroring each project/subject/stakeholder tag into a
real edge:

| tag_kind | resolved subject_type | edge verb |
|----------|----------------------|-----------|
| `project` | `project` | `concerns` |
| `subject` | `concept` | `mentions` |
| `stakeholder` | `person` | `involves` |

So a document's projects/subjects become `document --concerns|mentions|involves--> subject`
edges, and the document is then reachable from `maludb_graph_neighbors/_walk` and
`maludb_edge`. To wire one yourself (resolve-or-create the subject **without** clobbering an
existing type, then create the edge):

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;

-- resolve-or-create the subject (reuse existing as-is)
SELECT subject_id FROM maludb_subject WHERE canonical_name = $1;          -- if found, use it
-- else: SELECT register_svpor_subject(p_canonical_name => $1, p_subject_type => 'project');

-- the edge
SELECT maludb_svpor_statement_create(
         p_subject_kind => 'document', p_subject_id => $doc,
         p_verb_id      => maludb_core.resolve_svpor_verb('concerns'),
         p_object_kind  => 'subject',  p_object_id  => $subject,
         p_provenance   => 'provided');

-- the soft tag carrying the resolved object (so a UI can deep-link)
INSERT INTO maludb_document_tag
    (document_id, tag_kind, tag_value, tag_object_type, tag_object_id, provenance)
VALUES ($doc, 'project', $1, 'subject', $subject, 'provided');
COMMIT;
```

To find documents related to a subject/project:
```sql
SELECT neighbor_id, label, rel
  FROM maludb_graph_neighbors('subject', $1, 'both', ARRAY['concerns','mentions','involves'])
 WHERE neighbor_kind = 'document'
 ORDER BY neighbor_id;
```

## Practical notes

- **Provenance on edges.** Explicit links are `provenance='provided'`; machine-suggested
  ones are `'suggested'` and flow through the same accept/reject review as statements.
- **Edge attributes.** Attributes can hang on an edge: set `target_kind='svpor_statement'`,
  `target_id=<statement_id>` in `maludb_svpor_attribute_create(...)`.
- **Keep `max_depth` small.** The walk is breadth-first and cycle-safe, but fan-out grows
  quickly on a dense graph — start at 2–3 and a rel filter, widen only if needed.
