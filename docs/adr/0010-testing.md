# ADR 0010 — Testing

Status: Proposed — 2026-10-30

## Decision

- **Rust tests** (`cargo test` through crane in `nix flake check`): every rule fixture
  (ADR 0005); reading nisshi's format against nisshi's fixtures and against files written by a
  real `nisshi-store` (a committed length mid-file, a truncated last frame, a seal between two
  reads); commit and recovery with a crash injected at every step; seal idempotence; the query
  API's filters, sorting, cursor and counts.
- **Determinism**: deriving the same nisshi root twice, and once with a `kill -9` at random
  points, gives identical `content_sha256` for every day.
- **nixosTests** are the definition of done, with nisshi and hikyaku as test-only flake
  inputs and standard tools as the oracle (`zstdcat`, `jq`, `curl`): a VM runs hikyaku,
  `nisshi-store`, `nenpyo-derive` and `nenpyo-query`; a unit is killed with `SIGSEGV`, another
  restarted by systemd, a third exceeds its `MemoryMax`, the VM reboots; each event is
  queryable through `/v1/events` within 10 s, with its `src` pointing at the right nisshi
  record; nisshi stopped then restarted loses nothing; a `nenpyo verify --nisshi` passes.
- **Rule fixtures are recorded** from the journal of the test VMs (the systemd and kernel
  versions of the pinned nixpkgs), not written by hand; a nixpkgs bump re-records them and a
  diff shows any changed message.
- A breakage seen on real hosts becomes a test before it is fixed.
