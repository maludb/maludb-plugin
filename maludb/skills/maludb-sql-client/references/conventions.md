# MaluDB SQL — conventions, tenancy & error handling

The cross-cutting rules that keep middleware correct.

## search_path: when and why

The single most common mistake is calling a facade without the right `search_path` and
getting a "relation does not exist" / "function does not exist" error. The rule:

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;
-- facade calls / graph / episodes / statements / attributes / memory
COMMIT;
```

- **`public` first** → `current_schema()` resolves to the tenant schema, so `owner_schema`
  stamping and Row-Level Security apply to the calling tenant. If `maludb_core` were first,
  ownership would resolve wrong.
- **`maludb_core` present** → the facades' unqualified `malu$*` base tables and RLS grant
  tables resolve.
- **`SET LOCAL`** → the change lasts only for the transaction; the connection's default path
  is untouched. Use it inside `BEGIN`/`COMMIT`.

The same PDO/JDBC connection must run the `SET LOCAL` and the facade calls (they share the
transaction). The reference API server bundles this as a `db_tx_core(fn)` helper — replicate
that pattern: open txn → set path → run → commit/rollback.

**What does NOT need it:** plain CRUD on `maludb_subject`, `maludb_verb`, `maludb_project`,
`maludb_memory`, `maludb_memory_pool`, `maludb_skill`, and the `*_type` picker views. They
resolve on the default path. (Wrapping them anyway is harmless.)

## Tenancy & Row-Level Security

MaluDB is multi-tenant by schema. Rows are owned by `owner_schema` = the tenant schema, and
RLS scopes every read/write to the connecting role's tenant. Practical consequences:

- You generally **don't filter by tenant in your SQL** — RLS does it. A `SELECT * FROM
  maludb_subject` returns only your tenant's subjects.
- Facades stamp `owner_schema` from `current_schema()` — which is why `public` must lead the
  search_path (see above).
- Cross-tenant access requires a different role/schema; a normal middleware role can't see
  another tenant's rows.

## The provenance lifecycle

`provenance ∈ { provided, suggested, accepted, rejected }` on statements and attributes:

- **`provided`** — explicit, trusted input (a user typed it, or you assert it). Default for
  direct writes.
- **`suggested`** — machine-extracted, **awaiting human review**. The default for the
  extraction pipeline. This is the review queue: `WHERE provenance = 'suggested'`.
- **`accepted` / `rejected`** — the reviewer's decision, set via
  `maludb_svpor_statement_set_provenance(...)` / `maludb_svpor_attribute_set_provenance(...)`.

To express "this was true but no longer," **close** a statement's validity instead of
rejecting it: `maludb_svpor_statement_close(id, now())`.

`sensitivity` (episodes) ∈ `{ public, internal, restricted, prohibited }`.

## Identifier assignment quirks

| Entity | Id source |
|--------|-----------|
| `maludb_subject.subject_id` | **No sequence** — derive `MAX(subject_id)+1` at insert (or use `register_svpor_subject`). |
| `maludb_verb.verb_id` | **No sequence** — same `MAX+1` rule. |
| pools, skills, documents, source packages, episodes, statements, attributes, notes | Normal **sequences** — let the DB assign, read back with `RETURNING`. |

The MAX+1 rule means a high-concurrency insert into `maludb_subject`/`maludb_verb` can race
to a `23505`; do it inside a transaction and retry on conflict, or prefer the facade which
handles resolve-or-create.

## PostgreSQL error → meaning mapping

MaluDB enforces integrity in triggers/constraints, so you'll see specific SQLSTATEs. Map
them the way the API does (this is exactly the reference server's handler):

| SQLSTATE | Cause | Treat as |
|----------|-------|----------|
| `23505` | unique violation | conflict (HTTP 409) |
| `42501` | insufficient privilege | forbidden (403) — usually a missing grant/role |
| `23502` | not-null violation | validation (422) |
| `23503` | foreign-key violation (bad target id) | validation (422) |
| `23514` | check violation | validation (422) |
| `22000` / `22023` / `22P02` | data exception / invalid value / bad cast | validation (422) |
| `P0001` | a trigger `RAISE EXCEPTION` (e.g. unregistered type, bad enum, time order) | validation (422) |

Pull the human cause from the message's `ERROR: ...` line for diagnostics. Don't surface raw
DB messages to end users — translate by SQLSTATE.

Common triggers you'll hit:
- Inserting a subject/verb with an **unregistered type** → `P0001`/`23514`. Read
  `maludb_subject_type` / `maludb_verb_type` first.
- A statement/attribute with a **bad provenance/sensitivity enum** → `22023`/`P0001`.
- `valid_from > valid_to` on a relationship/statement → `P0001`.
- A duplicate episode-type label (case-insensitive) → `23505`.

## Transaction discipline

- **One logical write = one transaction.** Upload + ingest-edges, object + attributes,
  config provider+alias+binding — group them so partial graphs never persist.
- **Do model HTTP calls *before* opening the transaction** (the DB can't, and you don't want
  a network call holding a write lock). Embed/extract first, then open `BEGIN`, write, commit.
- **Roll back on any throw** and let the SQLSTATE mapping above classify it.

## Don'ts

- Don't touch `malu$*` base tables — go through the `maludb_*` views/facades.
- Don't interpolate user text into SQL — parameterize everything (names, spans, payloads).
- Don't mix embedding models within one namespace.
- Don't assume cascade deletes — typed attributes survive their node's deletion; clean them
  up first.
