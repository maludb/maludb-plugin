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
