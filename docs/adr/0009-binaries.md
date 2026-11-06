# ADR 0009 — Three binaries: data CLI, deriver, query daemon

Status: Proposed — 2026-10-30

## Context

masu and nisshi split into a data CLI, a datastore and a query daemon over one library
(masu ADR 0012, nisshi ADR 0016). nenpyo has the same three roles, with a deriver in place of
an ingestion server.

## Decision

A Cargo workspace:

- **`nenpyo-core`** (library): reading nisshi's format (ADR 0003), the rules (ADR 0005),
  nenpyo's storage format, commit, recovery, seal, retention, `read`, `verify`, `rebuild`.
- **`nenpyo`** (data CLI): `read <root> [--node] [--type] [--unit] --from --to` (NDJSON sorted
  by `ts`), `verify <root> [--nisshi <root>]`, `rebuild` (ADR 0007). It reads no
  configuration.
- **`nenpyo-derive`**: the only writer of a root. `nenpyo-derive run <config.toml>` derives,
  seals and applies retention; `check-config`; `--version`. Configuration: nisshi's root, its
  own root, `poll_interval`, `retention_days`, `disk_reserve`, `deploy_services`, the metrics
  listener.
- **`nenpyo-query`**: read-only. `nenpyo-query serve <config.toml>`, `check-config`,
  `--version`. Configuration: the root, listeners, TLS files, token hashes.

The NixOS module `services.nenpyo` runs both daemons as separate units, each with its own
CPU and memory bounds, and runs `check-config` on each rendered file at build time.
`nenpyo-derive` gets read access to nisshi's data directory and nothing else of nisshi's.

## Consequences

- An invalid configuration fails the system build.
- The deriver can be stopped for a rebuild while the query daemon keeps serving the old root
  until the swap.
