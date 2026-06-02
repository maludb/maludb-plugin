# MaluDB SQL — the vector-memory pipeline

MaluDB's RAG layer stores **embedded SVPO edges** in a graph-bound vector store and searches
them within a `(subject, verb)` compartment. The defining constraint: **PostgreSQL can't make
outbound HTTP calls**, so *your middleware is the model worker* — you call the LLM (extraction)
and the embedding model, then write vectors back via the facades. The DB never embeds.

All of this runs inside `BEGIN; SET LOCAL search_path TO public, maludb_core; ... COMMIT;`.

## The `malu_vector` type

Embeddings are passed as a `maludb_core.malu_vector`, cast from a bracketed float literal:

```sql
'[0.0123, -0.0456, 0.789, ...]'::maludb_core.malu_vector
```

- **One model + one dimension per namespace.** Every embedding stored under a namespace must
  share the same model and vector length. Don't mix `text-embedding-3-small` (1536) with
  another model in `'default'`.
- Format the literal locale-independently (use `.` decimals; avoid comma decimals from a
  non-C locale). Build it as `'[' || array_to_string(..., ',') || ']'` in app code.

## Step 1 — configure a model for a namespace

Use the **per-tenant self-service facades** (granted to `maludb_memory_executor`); do **not**
call the global owner-only `maludb_core.register_model_*`.

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;

-- 1) store the token encrypted (redact the value from your SQL trace!)
SELECT secret_id FROM maludb_core.secret_set(p_name => $1, p_kind => 'provider', p_value => $2);

-- 2) register the provider (secret referenced by NAME, never inlined)
SELECT maludb_register_model_provider(
         p_name => $1, p_kind => $2,          -- kind ∈ {cloud_api,local_http,local_socket,
         p_adapter_name => $3,                --         local_runtime,shell_adapter,stub}
         p_secret_ref => $4,                  -- the secret NAME from step 1
         p_data_sensitivity => $5) AS provider_id;

-- 3) register the alias (base_url rides in runtime_params)
SELECT maludb_register_model_alias(
         p_alias => $1, p_provider => $2, p_model_identifier => $3,
         p_context_length => $4,
         p_runtime_params => jsonb_build_object('base_url', $5::text)) AS alias_id;

-- 4) bind alias + prompt + embedding + defaults for this tenant/namespace
SELECT maludb_memory_set_model_config(
         p_extraction_alias     => $1,
         p_prompt_template      => $2,        -- MUST contain {{chunk}}
         p_embedding_model      => $3,
         p_namespace            => $4,
         p_generation_params    => $5::jsonb,
         p_default_subject_type => $6,        -- e.g. 'other'
         p_default_provenance   => $7) AS cfg; -- e.g. 'suggested'

-- 5) read it back
SELECT maludb_memory_model_config($4) AS cfg;   -- jsonb; secret_ref is the NAME, not the token
COMMIT;
```

Read config any time with `SELECT maludb_memory_model_config('default')`. `__secret_resolve`
turns a `secret_ref` into the plaintext token (needs the `maludb_secret_consumer` grant) —
your worker uses it to authenticate the HTTP call.

## Step 2 — ingest a document (your code does the model work)

The flow per document, all in one transaction so a partial graph never persists:

1. **In app code (before the txn):** chunk the text, call the LLM to extract candidate SVPO
   edges (or accept caller-supplied edges), and embed each edge's `source_span` → a
   `malu_vector` literal. (No creds? Use a deterministic local embedding — same text → same
   vector — so the pipeline round-trips offline, and supply pre-extracted edges.)
2. **In the txn:** create the document, then `maludb_memory_ingest_edge(...)` per edge.

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;

SELECT maludb_upload_document(
         p_title => $1, p_content_text => $2, p_source_type => 'document',
         p_media_type => $3, p_document_type => $4,
         p_projects => $5::text[], p_subjects => $6::text[],
         p_verbs => $7::text[], p_events => $8::text[],
         p_metadata_jsonb => $9::jsonb) AS document_id;

-- repeat per extracted edge:
SELECT maludb_memory_ingest_edge(
         p_source_kind      => 'document', p_source_id => $doc,
         p_subject_text     => $10, p_verb_text => $11,
         p_predicate        => $12::jsonb,                 -- [{attr_name,value_text},...]
         p_embedding        => $13::maludb_core.malu_vector,
         p_embedding_model  => $14,                        -- same model across the namespace
         p_subject_type     => $15,                        -- default from config
         p_source_span      => $16,                        -- the verbatim text embedded
         p_confidence       => $17::numeric,
         p_provenance       => $18,                        -- default 'suggested' (review queue)
         p_extraction_model => $19,
         p_namespace        => $20,
         p_document_id      => $doc) AS statement_id;
COMMIT;
```

Extracted edges default to `provenance='suggested'` — they enter the review queue and are
promoted via `maludb_svpor_statement_set_provenance(...)`.

## Step 3 — search by meaning

Embed the query **with the same model used at ingest**, then ANN-search within a compartment:

```sql
BEGIN;
SET LOCAL search_path TO public, maludb_core;
SELECT chunk_id, statement_id, document_id, source_text, distance, similarity,
       rank_no, subject_name, verb_name
  FROM maludb_memory_search(
         p_query_embedding => $1::maludb_core.malu_vector,
         p_subject         => $2,         -- subject and/or verb is REQUIRED
         p_verb            => $3,
         p_namespace       => $4,         -- 'default'
         p_limit           => $5,         -- e.g. 20
         p_metric          => $6);        -- 'cosine'
COMMIT;
```

**The `(subject, verb)` compartment pre-filter is mandatory** — pass at least one. It scopes
the ANN to the relevant slice of the graph before the vector search, which is what makes the
results graph-aware rather than a flat similarity lookup. The DB enforces this too.

## Role & capability caveats (verified, maludb_core 0.91.0)

The memory executor role is **append-only** for vectors and config:

- There are **no delete facades** for vector chunks (`malu$vector_chunk` /
  `tombstone_vector_chunk` are owner-only) or for providers/aliases/config bindings — only
  `secret_revoke` exists. Garbage-collecting the rest needs a superuser.
- The per-tenant `maludb_register_model_provider` / `maludb_register_model_alias` /
  `secret_set` / `set_model_config` / `model_config` / `ingest_edge` / `search` all work as
  `maludb_memory_executor` (+ `reader`) — **no global model-admin grant required**.
- An async extraction path (`request_extraction` / `harvest_extractions`) exists but needs a
  worker process — out of scope for direct SQL middleware.

## Common pitfalls

- **Mismatched embedding dimension/model** in a namespace → search returns garbage or errors.
  Pin one model per namespace and embed queries with it.
- **Forgetting the compartment** → `maludb_memory_search` rejects a call with neither subject
  nor verb.
- **Logging the token.** When you run `secret_set`, redact the value parameter from any query
  log — never persist the plaintext token.
