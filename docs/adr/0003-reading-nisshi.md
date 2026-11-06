# ADR 0003 — Reading nisshi

Status: Proposed — 2026-10-30

## Context

nisshi's storage format (nisshi `docs/storage-format.md`) is a public contract: one
`<node>/<YYYY-MM-DD>.ndjson.zst` per node and receive day, a concatenation of zstd frames, a
manifest once the day is sealed. Its open day grows one frame per commit; a frame can be on
disk before its commit, and nisshi truncates it at restart if the commit did not happen. The
day file is recompressed when sealed, with the same decompressed content (`content_sha256`).
The stack shares contracts, not code.

## Options considered

- **Depend on `nisshi-core`**: one implementation of the format, but nenpyo's releases would
  follow nisshi's library, against the stack's rule.
- **Read through nisshi's query daemon**: no file coupling, but it does not exist yet, and a
  tail-follow API would be designed for one client.
- **Reimplement the reading side from the contract**: frames, NDJSON, manifests, the committed
  length; a few hundred lines.

## Decision

- nenpyo reads nisshi's root directly and read-only, from its own implementation of the
  contract. nisshi is a test-only flake input: its fixtures and a real `nisshi-store` in the
  nixosTests (ADR 0010).
- **Committed length.** On an open day, nenpyo reads only up to the committed length that
  nisshi's `state.json` records for the file, never past it: bytes after it may be cut at
  nisshi's next start. This makes the committed lengths of `state.json` part of nisshi's
  reader contract (they already are for nisshi's backup); nisshi's storage format is amended
  to say so (pending, with the rewrite).
- **Position.** Per node, nenpyo keeps `(day, offset)` where `offset` counts **decompressed**
  bytes: it survives the seal's recompression. On an open day it also keeps the compressed
  offset it reached, to resume without decompressing the day again; after a seal, it
  decompresses the day up to `offset` once.
- **Polling.** `nenpyo-derive` reads `state.json` every `poll_interval` (default 2 s), then
  reads what each node's open day gained. New node directories and new days are found by
  listing. A day that has a manifest is read to its end, then nenpyo moves to the next day of
  that node.
- **nisshi behind or down**: nenpyo has nothing new to read and waits; it never touches
  nisshi's files, `state.json` included. A `state.json` that does not parse (a rename under
  way is atomic, so this means corruption) is retried at the next poll and reported in
  nenpyo's status.
- **A committed length that went backwards** (nisshi restored from a backup) is a changed
  source: nenpyo drops its open output for that node and day and derives the day again from
  offset 0 (ADR 0007). A sealed day never changes; a sealed day whose `content_sha256` differs
  from the one nenpyo recorded stops nenpyo for that node, with an `error` event.
- **Retention**: a nisshi day whose data file is gone (manifest only) is skipped.

## Consequences

- nisshi's storage format gains one reader rule (committed lengths in `state.json`), to be
  written in nisshi once its rewrite is merged.
- nenpyo needs read access to nisshi's data directory, so it runs on nisshi's host.
- Latency: hikyaku's delivery plus at most one `poll_interval`.
