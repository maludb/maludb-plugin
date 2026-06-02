# MaluDB API — client recipes

Copy-paste starting points. First a **thin wrapper** (set base URL + token once, parse the
`{error}` shape once, unwrap the resource key once) in three languages, then worked
multi-step flows. Build on the wrapper — don't hand-roll headers per call.

## Contents

- [Thin client wrapper](#thin-client-wrapper) — TypeScript, Python, PHP
- [Recipe: create a subject and link a verb](#recipe-create-a-subject-and-link-a-verb)
- [Recipe: upload a document and link it to a project](#recipe-upload-a-document-and-link-it-to-a-project)
- [Recipe: the vector-memory pipeline](#recipe-the-vector-memory-pipeline)
- [Recipe: the provenance review queue](#recipe-the-provenance-review-queue)
- [Recipe: walk the graph](#recipe-walk-the-graph)

---

## Thin client wrapper

### TypeScript / JavaScript (fetch — works in Electron renderer/main and modern browsers)

```ts
export class MaluDbError extends Error {
  constructor(public code: string, message: string, public status: number) {
    super(message);
    this.name = "MaluDbError";
  }
}

export class MaluDb {
  constructor(
    private baseUrl: string,        // e.g. "https://api.maludb.com/v1"
    private token: string,          // "malu_..."
  ) {}

  // Core request. `key` unwraps the resource ({subjects:[...]} -> [...]).
  async request<T = any>(
    method: string,
    path: string,
    opts: { query?: Record<string, any>; body?: any; key?: string } = {},
  ): Promise<T> {
    const url = new URL(this.baseUrl + path);
    for (const [k, v] of Object.entries(opts.query ?? {})) {
      if (v !== undefined && v !== null) url.searchParams.set(k, String(v));
    }
    const headers: Record<string, string> = {
      Authorization: `Bearer ${this.token}`,
      Accept: "application/json",
    };
    let body: string | undefined;
    if (opts.body !== undefined) {
      headers["Content-Type"] = "application/json";
      body = JSON.stringify(opts.body);
    }
    const res = await fetch(url, { method, headers, body });
    const json = res.status === 204 ? {} : await res.json();
    if (!res.ok) {
      throw new MaluDbError(json?.error?.code ?? "unknown",
                            json?.error?.message ?? res.statusText, res.status);
    }
    return opts.key ? json[opts.key] : json;
  }

  // Typed conveniences
  listSubjects(q?: string, limit = 50) {
    return this.request("GET", "/subjects", { query: { q, limit }, key: "subjects" });
  }
  createSubject(input: { label: string; type?: string; description?: string }) {
    return this.request("POST", "/subjects", { body: input, key: "subject" });
  }
  getSubject(id: number) {
    return this.request("GET", `/subjects/${id}`, { key: "subject" });
  }

  // multipart upload needs raw FormData (no JSON Content-Type)
  async uploadDocument(file: Blob, fields: Record<string, string> = {}) {
    const fd = new FormData();
    fd.append("file", file);
    for (const [k, v] of Object.entries(fields)) fd.append(k, v);
    const res = await fetch(this.baseUrl + "/documents", {
      method: "POST",
      headers: { Authorization: `Bearer ${this.token}`, Accept: "application/json" },
      body: fd, // browser sets the multipart boundary; do NOT set Content-Type yourself
    });
    const json = await res.json();
    if (!res.ok) throw new MaluDbError(json.error.code, json.error.message, res.status);
    return json.document;
  }
}
```

### Python (requests)

```python
import requests

class MaluDbError(Exception):
    def __init__(self, code, message, status):
        super().__init__(f"{status} {code}: {message}")
        self.code, self.status = code, status

class MaluDb:
    def __init__(self, base_url: str, token: str):
        self.base = base_url.rstrip("/")          # "https://api.maludb.com/v1"
        self.s = requests.Session()
        self.s.headers.update({"Authorization": f"Bearer {token}",
                               "Accept": "application/json"})

    def request(self, method, path, *, query=None, body=None, key=None):
        kw = {"params": query}
        if body is not None:
            kw["json"] = body                      # sets Content-Type: application/json
        r = self.s.request(method, self.base + path, **kw)
        data = {} if r.status_code == 204 else r.json()
        if not r.ok:
            err = data.get("error", {})
            raise MaluDbError(err.get("code", "unknown"),
                              err.get("message", r.reason), r.status_code)
        return data[key] if key else data

    def list_subjects(self, q=None, limit=50):
        return self.request("GET", "/subjects", query={"q": q, "limit": limit}, key="subjects")

    def create_subject(self, label, type=None, description=None):
        return self.request("POST", "/subjects",
                            body={"label": label, "type": type, "description": description},
                            key="subject")

    def upload_document(self, path, **fields):
        with open(path, "rb") as f:
            r = self.s.post(self.base + "/documents",
                            files={"file": f}, data=fields)  # multipart; no json=
        data = r.json()
        if not r.ok:
            raise MaluDbError(data["error"]["code"], data["error"]["message"], r.status_code)
        return data["document"]
```

### PHP (cURL)

```php
<?php
final class MaluDbError extends RuntimeException {
    public function __construct(public string $code, string $message, public int $status) {
        parent::__construct($message);
    }
}

final class MaluDb {
    public function __construct(private string $base, private string $token) {} // base e.g. https://api.maludb.com/v1

    public function request(string $method, string $path, array $opts = []): mixed {
        $url = $this->base . $path;
        if (!empty($opts['query'])) {
            $q = array_filter($opts['query'], fn($v) => $v !== null);
            if ($q) $url .= '?' . http_build_query($q);
        }
        $headers = ["Authorization: Bearer {$this->token}", "Accept: application/json"];
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST  => $method,
            CURLOPT_RETURNTRANSFER => true,
        ]);
        if (array_key_exists('body', $opts)) {
            $headers[] = "Content-Type: application/json";
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($opts['body']));
        }
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        $raw    = curl_exec($ch);
        $status = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        $json = $raw === '' ? [] : json_decode($raw, true);
        if ($status >= 400) {
            throw new MaluDbError($json['error']['code'] ?? 'unknown',
                                  $json['error']['message'] ?? 'HTTP error', $status);
        }
        return isset($opts['key']) ? $json[$opts['key']] : $json;
    }

    public function listSubjects(?string $q = null, int $limit = 50): array {
        return $this->request('GET', '/subjects', ['query' => ['q' => $q, 'limit' => $limit], 'key' => 'subjects']);
    }
    public function createSubject(array $input): array {
        return $this->request('POST', '/subjects', ['body' => $input, 'key' => 'subject']);
    }
    public function uploadDocument(string $filePath, array $fields = []): array {
        $headers = ["Authorization: Bearer {$this->token}", "Accept: application/json"];
        $post = $fields + ['file' => new CURLFile($filePath)];   // multipart; cURL sets boundary
        $ch = curl_init($this->base . '/documents');
        curl_setopt_array($ch, [CURLOPT_POST => true, CURLOPT_POSTFIELDS => $post,
                                CURLOPT_HTTPHEADER => $headers, CURLOPT_RETURNTRANSFER => true]);
        $json = json_decode(curl_exec($ch), true);
        $status = (int) curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        if ($status >= 400) throw new MaluDbError($json['error']['code'], $json['error']['message'], $status);
        return $json['document'];
    }
}
```

---

## Recipe: create a subject and link a verb

A statement-free quick link: create a subject, then attach an existing verb to it.

```ts
const malu = new MaluDb("https://api.maludb.com/v1", token);

const subject = await malu.createSubject({ label: "Edward Honour", type: "person" });
// link verb id 42 to the subject
await malu.request("POST", `/subjects/${subject.id}/verbs`, { body: { verb_id: 42 } });

// or model a real edge with the statement layer (resolves names for you):
await malu.request("POST", "/statements", {
  body: {
    subject: "Edward Honour",       // resolved/created as a person
    verb: "maintains",              // resolved by name
    object_kind: "document", object_id: 7,
    provenance: "provided",
  },
  key: "statement",
});
```

`POST /v1/statements` is idempotent on the `(subject_kind, subject_id, verb_id,
object_kind, object_id)` five-tuple — calling it twice returns the same edge.

---

## Recipe: upload a document and link it to a project

One multipart call wires the document into the graph: each name in `projects` / `subjects`
becomes a `document→subject` edge, and the first project becomes `primary_project_id`.

```python
doc = malu.upload_document(
    "/path/to/spec.pdf",
    filename="spec.pdf",
    mime_type="application/pdf",
    description="Architecture spec",
    document_type="spec",
    projects="MaluDB,Platform",     # comma-separated names
    subjects="Edward Honour",
)
# doc -> { id, title, content_size, primary_project_id, ... }

# Later: every document for a project shows up on the project/subject detail page,
# and is reachable from the graph endpoints:
neighbors = malu.request("GET", "/graph/neighbors",
                         query={"kind": "subject", "id": doc["primary_project_id"],
                                "rel": "concerns,mentions,involves"},
                         key="neighbors")
```

Add or remove links later with `PATCH /v1/documents/{id}`:
```jsonc
{ "link":   { "subjects": ["New Person"] },
  "unlink": { "projects": ["Platform"] } }
```

---

## Recipe: the vector-memory pipeline

Three steps: configure a model for a namespace, ingest documents, then search by meaning.
The pipeline round-trips **without live model creds** — the server falls back to a
deterministic embedding and you can hand it pre-extracted `edges`.

```ts
// 1) Configure the namespace once (token stored encrypted server-side).
await malu.request("POST", "/memory/config", {
  body: {
    namespace: "default",
    secret_name: "openai-key", token: process.env.OPENAI_KEY,
    provider: { name: "openai", kind: "cloud_api", data_sensitivity: "internal" },
    alias:    { name: "extract", model_identifier: "gpt-4o-mini",
                base_url: "https://api.openai.com/v1" },
    prompt_template: "Extract SVPO edges. {{chunk}}",   // MUST contain {{chunk}}
    embedding_model: "text-embedding-3-small",
    default_provenance: "suggested",
  },
});

// 2) Ingest a document (server chunks -> extracts -> embeds -> stores).
const ingest = await malu.request("POST", "/memory/documents", {
  body: { title: "Q2 standup", text: longTranscript, namespace: "default",
          projects: ["MaluDB"] },
});
// ingest -> { document_id, chunk_count, extractor, edges:[{statement_id, subject_text, ...}] }

// --- OR, with no model: supply edges yourself ---
await malu.request("POST", "/memory/documents", {
  body: { title: "notes", text: "Edward upgraded the server.", subjects: ["Edward"],
          edges: [{ subject_text: "Edward", subject_type: "person", verb_text: "upgraded",
                    source_span: "Edward upgraded the server.", confidence: 0.9 }] },
});

// 3) Search within a (subject, verb) compartment — at least one is required.
const hits = await malu.request("POST", "/memory/search", {
  body: { query: "what did Edward change?", subject: "Edward", namespace: "default", limit: 10 },
  key: "results",
});
// hits -> [{ chunk_id, document_id, source_text, similarity, subject_name, verb_name }]
```

Key constraints: the **search query is embedded with the same model as ingest** (don't mix
models in one namespace), and search **requires a subject and/or verb** as the compartment
pre-filter before the ANN.

---

## Recipe: the provenance review queue

Machine-extracted statements/attributes land as `suggested`. Build a review screen that
lists them and promotes each.

```python
# 1) The queue
pending = malu.request("GET", "/statements", query={"provenance": "suggested", "limit": 100},
                       key="statements")

# 2) Accept or reject each (PATCH the provenance)
for s in pending:
    decision = "accepted"  # or "rejected", from the reviewer
    malu.request("PATCH", f"/statements/{s['id']}", body={"provenance": decision})

# Attributes work the same way:
pending_attrs = malu.request("GET", "/attributes", query={"provenance": "suggested"},
                             key="attributes")
malu.request("PATCH", f"/attributes/{pending_attrs[0]['id']}", body={"provenance": "accepted"})
```

To **close** a statement's validity instead of rejecting it (it was true, now it isn't):
`PATCH /v1/statements/{id}` with `{ "close": true }` (uses `now()`) or
`{ "valid_to": "2026-06-01T00:00:00Z" }`.

---

## Recipe: walk the graph

Start from a handle and explore. `neighbors` is one hop; `walk` is multi-hop.

```php
// One hop out of a subject, following only "concerns"/"mentions" edges:
$neighbors = $malu->request('GET', '/graph/neighbors', ['query' => [
    'kind' => 'subject', 'id' => 17, 'direction' => 'out', 'rel' => 'concerns,mentions',
], 'key' => 'neighbors']);

// Multi-hop walk, depth 3:
$walk = $malu->request('GET', '/graph/walk', ['query' => [
    'kind' => 'subject', 'id' => 17, 'max_depth' => 3, 'direction' => 'both',
], 'key' => 'walk']);
// each row: { object_kind, object_id, depth, rel, label, path:[ids walked] }

// Or read raw edges directly:
$edges = $malu->request('GET', '/edges', ['query' => [
    'source_kind' => 'document', 'source_id' => 7, 'limit' => 100,
], 'key' => 'edges']);
```

The `(kind, id)` handle is the same identifier used by `GET /v1/objects/{kind}/{id}` (which
returns the object inline with its attributes), so you can pivot from a graph hit straight
to its full record.
