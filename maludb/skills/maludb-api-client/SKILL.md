---
name: maludb-api-client
description: >-
  Integrate the MaluDB REST/JSON API into a desktop (Electron) or web application.
  Use this whenever the user is calling MaluDB over HTTP — building a client for
  subjects, verbs, episodes, statements, documents, notes, projects, memory pools,
  skills, typed attributes, the knowledge graph, or the vector-memory search pipeline;
  authenticating with a `malu_` Bearer token against `api.maludb.com` / `/v1/...`;
  handling MaluDB error codes; or writing fetch/axios/requests/cURL code that talks to
  any `/v1` endpoint. Trigger even when the user only says "the MaluDB API", "call the
  memory API", "upload a document to MaluDB", or names a `/v1` path, without saying "client".
  For writing SQL straight against the database use maludb-sql-client instead; for building
  the PHP API server itself use maludb-api-server.
---

# MaluDB API client

This skill helps you call the **MaluDB REST API** from a client application — the
desktop (Electron) or web app side of a MaluDB memory system. MaluDB is a memory /
knowledge-graph store; the API is a thin PHP+JSON layer over a PostgreSQL extension.

If the task is writing SQL directly against the database, use **maludb-sql-client**.
If the task is building the API server's PHP endpoints, use **maludb-api-server**.

## The 6 things that are always true

1. **Base URL & version.** All endpoints live under `https://api.maludb.com/v1/...`
   (host configurable per deployment). HTTPS only. Always send `Accept: application/json`.

2. **Auth is a Bearer token on every request.** Header:
   `Authorization: Bearer malu_<43 url-safe base64 chars>`. There is **no login
   endpoint** — tokens are minted admin-side (psql), not via the API. A missing or
   malformed token → `401 auth_missing`; an invalid/expired one → `401 auth_invalid`.

3. **Bodies are JSON** (`Content-Type: application/json`, UTF-8) — **except**
   `POST /v1/documents`, which is `multipart/form-data` (raw file bytes). Responses are
   always `application/json; charset=utf-8`, even for errors.

4. **Errors have one shape:** `{ "error": { "code": "...", "message": "..." } }`.
   Switch on the stable `code`, never the human `message`. The HTTP status tells you the
   category (`400/401/403/404/405/409/413/415/422/500/501`). See
   [references/auth-and-errors.md](references/auth-and-errors.md).

5. **Success bodies are wrapped under a named key**, not bare. A list returns
   `{ "subjects": [...] }`; a single resource returns `{ "subject": {...} }`; a delete
   returns `{ "deleted": true, "id": 17 }`. The key matches the resource. Read the actual
   shape per endpoint in [references/endpoints.md](references/endpoints.md).

6. **MaluDB renames things at the boundary.** What the API calls `label` the database
   calls `canonical_name`; the API resource `subjects` is the DB view `maludb_subject`.
   As a *client* you only ever see the API field names — but knowing the mapping exists
   explains why some payloads look the way they do. The map is in
   [references/data-model.md](references/data-model.md).

## Minimal request (the shape every call follows)

```ts
const res = await fetch("https://api.maludb.com/v1/subjects?q=edward&limit=20", {
  headers: {
    "Authorization": `Bearer ${process.env.MALUDB_TOKEN}`,
    "Accept": "application/json",
  },
});
const body = await res.json();
if (!res.ok) throw new MaluDbError(body.error.code, body.error.message, res.status);
const subjects = body.subjects; // wrapped under the resource key
```

A POST writes JSON and usually returns `201`:

```ts
const res = await fetch("https://api.maludb.com/v1/subjects", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${token}`,
    "Content-Type": "application/json",
    "Accept": "application/json",
  },
  body: JSON.stringify({ label: "Edward Honour", type: "person" }),
});
// 201 → { subject: { id, label, type, description, classifier_md, linked_verbs } }
```

Build **one thin wrapper** (set base URL + token once, parse `{error}` once, unwrap the
resource key once) and call it everywhere. Don't hand-roll headers per call. A complete
wrapper in PHP, TypeScript, and Python is in [references/recipes.md](references/recipes.md).

## The resource surface (what you can call)

Organized by area. Full method/body/response/status detail for each is in
[references/endpoints.md](references/endpoints.md).

- **Knowledge graph core** — `subjects`, `verbs`, the link sub-resources
  (`subjects/{id}/verbs`, `subjects/{id}/related-subjects`, `subject-relationships/{id}`),
  and the type pickers `subject-types` / `verb-types`.
- **Events** — `episodes`, `episodes/{id}/statements`, the generic `statements` layer
  (subject→verb→object edges), and `episode-types`.
- **Typed attributes** — `attributes`, `attribute-templates`, `attribute-check`
  (a typed property on any node *or* edge, driven by a template catalog).
- **Objects / unified graph** — the `(kind, id)` handle endpoints `objects/{kind}/{id}`
  and `objects/{kind}`, plus the read-only graph surface `edges`, `graph/neighbors`,
  `graph/walk`.
- **Content** — `documents` (multipart upload), `documents-backfill`, `notes`
  (with `close-issue` / `reopen-issue` actions).
- **Workspaces** — `projects` (+ `archive` / `unarchive` / `subjects` / `verbs` links),
  `pools` (+ `archive`), `skills` (+ `duplicate`).
- **Vector memory** — `memory/config`, `memory/documents` (ingest), `memory/search`.
  This is the RAG pipeline: configure a model, ingest a document (chunk → extract →
  embed → store), then search by meaning within a (subject, verb) compartment.

## Cross-cutting client behaviors worth knowing

- **Lists are cheap, detail is rich.** `GET /v1/subjects` rows carry only *count* fields
  (`linked_verbs`, `related_subjects`); the full related collections come back inline only
  on the detail `GET /v1/subjects/{id}`. Don't N+1 the list — fetch detail when the user
  opens a record.
- **`?with=attributes`** on `GET /v1/subjects|episodes|documents` adds an `attributes`
  array per row in one extra batched query.
- **Provenance is a review workflow.** Extracted statements/attributes land as
  `provenance: "suggested"`; your review UI lists `?provenance=suggested` and promotes
  them to `accepted`/`rejected` via PATCH. Explicit user input is `provided`.
- **`?debug=1`** (when the server enables it) attaches `meta.debug.queries[]` — the exact
  SQL behind the response. Invaluable when a call misbehaves.
- **The desktop client historically called `/v1/files`; the server is `/v1/documents`.**
  Use `/v1/documents`.

## Workflow when integrating an endpoint

1. Look up the endpoint in [references/endpoints.md](references/endpoints.md) — confirm
   the method, the exact request body/query params, and the response wrapper key.
2. Reuse the thin client wrapper from [references/recipes.md](references/recipes.md);
   add a typed method for the new call.
3. Map error `code`s you care about (e.g. `409 already_archived`, `422 validation_failed`)
   to user-facing behavior — see [references/auth-and-errors.md](references/auth-and-errors.md).
4. For multi-step flows (upload + link, the memory pipeline, the review queue), follow the
   worked recipes rather than reinventing the sequence.
