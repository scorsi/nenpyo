# ADR 0007 — Determinism, resume and rebuild

Status: Proposed — 2026-10-30

## Context

nenpyo must survive its own crashes and nisshi's restores without losing or doubling events,
and its output must be checkable against its source. When rules change, older events keep the
old ones unless someone decides otherwise; consumers may already have acted on them.

## Options considered

- **Recompute history automatically on every rule change**: events change under consumers,
  and every release rewrites files already backed up.
- **Rules apply forward; a rebuild is a manual step into a new root.**

## Decision

- **Determinism**: a nenpyo file is a function of the nisshi file of the same node and day
  and of the rule set (ADR 0005: stateless rules, events in source order). Deriving it twice
  gives the same bytes, before compression.
- **Resume**: the position per node (ADR 0003) is committed with the events (ADR 0006).
  After a crash, the open file is cut to its committed length and reading resumes at the
  committed position: no event lost, none doubled.
- **Changed source**: when nisshi's committed length of an open day went backwards, nenpyo
  empties its file for that day and derives it again from offset 0.
- **New rules apply from their deployment on.** Events already written keep their `rules`
  version.
- **Rebuild is manual**: `nenpyo rebuild --nisshi <root> --out <empty root> [--from <day>]
  [--to <day>]` derives into a new root with the current rules; the operator stops the
  deriver, swaps the roots and starts it again. A rebuild never rewrites a root in place.
  Days whose nisshi data was deleted by retention cannot be rebuilt: `rebuild` copies their
  sealed files and manifests unchanged from the old root when given `--keep <old root>`.
- **Verify by recomputation**: `nenpyo verify <root> [--nisshi <root>]` checks every sealed
  file against its manifest; with `--nisshi`, it derives each sealed day whose source is still
  there and whose events carry a single `rules` version equal to the current one, and compares
  `content_sha256`. A day of an older version is reported as not comparable, not as a mismatch.

## Consequences

- A rule change is visible in the data: the `rules` field changes from its deployment on.
- A rebuild needs the disk for a second root until the swap.
