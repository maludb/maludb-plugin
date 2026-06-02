# MaluDB plugin

A Claude Code plugin for building **MaluDB memory applications**.

MaluDB is a memory / knowledge-graph system implemented as a set of extensions to
PostgreSQL (the `maludb_core` schema). It stores **subjects**, **verbs**, **objects**,
**episodes** (events), **SVPO statements** (graph edges), **typed attributes**,
**documents**, **memory pools**, **skills**, **projects**, and **notes**, plus a
graph-bound **vector-memory pipeline** (`malu_vector`) and **graph traversal**
(`neighbors` / `walk`).

There are two ways to consume MaluDB, and one way to build the surface that exposes it.
This plugin ships a skill for each:

| Skill | Audience | What it covers |
|-------|----------|----------------|
| **maludb-api-client** | Desktop (Electron) & web app developers | Integrating the MaluDB REST/JSON API — base URL, Bearer auth, error codes, every `/v1` endpoint, and end-to-end recipes (PHP, TypeScript/JavaScript, Python). |
| **maludb-sql-client** | Middleware developers | Querying the `maludb_core` PostgreSQL extension directly — updatable views, facade functions, the `search_path` rule, RLS/tenancy, the provenance lifecycle, graph traversal, and the vector-memory pipeline. |
| **maludb-api-server** | Backend / API developers | Scaffolding the PHP API server itself — the one-file-per-endpoint architecture, shared response/auth/db helpers, `.htaccess` routing, SQL tracing, and the error-mapping contract. |

## Layout

```
maludb/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    ├── maludb-api-client/
    │   ├── SKILL.md
    │   └── references/{endpoints,auth-and-errors,data-model,recipes}.md
    ├── maludb-sql-client/
    │   ├── SKILL.md
    │   └── references/{schema,crud,facades,graph,vector-memory,conventions}.md
    └── maludb-api-server/
        ├── SKILL.md
        └── references/{architecture,shared-helpers,endpoint-patterns,routing-and-auth}.md
```

Each `SKILL.md` is the always-loaded entry point: the overview, the rules you always
need, and pointers into `references/`, which load only when relevant. All content is
grounded in the **live behavior** of the reference API server (the `v1/*.php` endpoints
and `config/response.php`), not just the idealized spec — where the two disagree, the
live schema wins (see the api-server-requirements §4.0 live-schema mapping).

## Installing

Add this plugin directory to a Claude Code plugin marketplace, or point Claude Code at
it directly. The three skills auto-register from `skills/` and trigger on the contexts
described in each skill's `description`.
