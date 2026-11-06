# ADR 0006 — Storage, sealing and retention

Status: Proposed — 2026-10-30

## Context

Events are few, read by range, and kept as long as the logs they come from. Rebuilding them
is possible but not wanted on a restore: they are backed up like the stores' data. Their
source, nisshi, is hash-chained; nenpyo writes only what it derives from it.

## Options considered

- **A database (SQLite, an embedded key-value store)**: indexes for free, but a format no
  standard tool reads and a new dependency.
- **The stores' layout**: zstd NDJSON per node and day, sealed with a manifest; readable with
  `zstdcat | jq`, backed up incrementally, scanned fast at this volume.
- **A hash chain like nisshi's**: proves the history unchanged, but nenpyo's content is a
  function of nisshi's, which is chained already.

## Decision

- Layout, the format `nenpyo-store 1`, documented in `docs/storage-format.md`:
  ```
  <root>/FORMAT                          "nenpyo-store 1"
  <root>/state.json                      writer state
  <root>/<node>/<YYYY-MM-DD>.ndjson.zst  events derived from nisshi's file of that node and day
  <root>/<node>/<YYYY-MM-DD>.manifest.json  once sealed
  ```
  The day is nisshi's receive day of the source records: one nenpyo file per nisshi file,
  events in the order of their source records. A nisshi day that yields no event has an empty
  file and its manifest.
- **Commit** as in nisshi: append a zstd frame, `fsync`, then write `state.json` (position per
  node, committed length of each open file) through a temporary file and a rename. At start,
  open files are cut to their committed length. One writer per root (`flock`).
- **Seal**: when nisshi's day has a manifest and nenpyo has read it to the end, nenpyo
  recompresses its file and writes its manifest: `records`, `content_sha256` (decompressed
  bytes), `min_ts`, `max_ts`, counts per `type`, the `rules` versions present, and
  `source_content_sha256`, nisshi's `content_sha256` for that day. A sealed file never changes.
- **No hash chain.** Integrity is checked by recomputation: `nenpyo verify` checks each sealed
  file against its manifest and, given nisshi's root, derives the day again and compares
  (ADR 0007).
- **Retention**: the same as nisshi, default **400 days**, `0` keeps everything; the daily job
  deletes data files of sealed days older than that and keeps their manifests. Disk pressure
  never deletes: below `disk_reserve` (default 1 GiB) the deriver stops writing, keeps its
  position, and resumes when space is back; `/v1/status` and its metrics show it.
- **Backup and restore**: a backup holds the sealed days and their manifests; the open day is
  not backed up, since it is derived again from nisshi. Restore copies `FORMAT`, the sealed
  files and manifests into an empty root; without `state.json` the deriver resumes each node
  at the day after its newest sealed day, offset 0.

## Consequences

- `zstdcat <node>/<day>.ndjson.zst | jq` reads events without nenpyo.
- A range query by event time cannot prune by file day alone (late deliveries): it prunes by
  each manifest's `min_ts` and `max_ts`, and scans the open days (ADR 0008).
- Without a chain, a forged event file with a matching manifest is detected only by `verify`
  against nisshi, for days whose logs are still retained.
