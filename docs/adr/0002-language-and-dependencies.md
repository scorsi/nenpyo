# ADR 0002 — Language and dependencies

Status: Proposed — 2026-10-30

## Context

The deriver decompresses and parses every log line nisshi receives, and a rebuild reads the
whole retained history (about 16 GB of JSON at the current estimates). The query daemon serves
HTTPS. The target is a modest NAS beside nisshi and masu, both written in Rust (masu ADR 0002,
nisshi ADR 0015).

## Options considered

| | For | Against |
|---|---|---|
| Rust | the stores' language and crates: one dependency register; no GC, bounded memory; fast zstd and JSON | more dependencies than a large stdlib |
| Go | stdlib HTTP and TLS | zstd from a third-party crate; nisshi just left Go for the same machine |
| Zig | full control | no TLS server in the standard library: the query daemon would sit behind a proxy or carry its own TLS |
| Nim, Elixir | — | a new toolchain for one component; the BEAM's memory on a small NAS |

## Decision

Rust, built with crane, as a Cargo workspace (ADR 0009). Direct dependencies, all from the
stores' register (masu ADR 0002, nisshi ADR 0015):

| Crate | Why |
|---|---|
| `zstd` (vendored libzstd) | reading nisshi's files, writing nenpyo's |
| `serde` (`derive`), `serde_json` | log records, events, `state.json`, manifests, query responses |
| `sha2` | content hashes of manifests, token hashes |
| `toml` | configuration files, read with `#[serde(deny_unknown_fields)]` |
| `tokio` (`rt-multi-thread` with 2 workers, `net`, `time`, `signal`, `macros`) | runtime of `nenpyo-query` |
| `hyper` (`server`, `http1`), `hyper-util` (`tokio`) | query API, HTTP/1.1 only |
| `rustls` (`ring` provider), `tokio-rustls`, `rustls-pemfile` | TLS |

`nenpyo-derive` is synchronous (one thread reads and writes); only `nenpyo-query` uses tokio.
No protobuf: the query API is JSON. Written by hand, as in the stores: the `ts` format and
calendar days, argument parsing, own logs in nisshi's schema, the metrics exposition, `flock`,
`statvfs` and `sd_notify` through `std` and `extern "C"`.

Log records are parsed with `serde_json`. A record it refuses (a lone surrogate escape in a
service field is valid in nisshi's schema, not in `serde_json`) is skipped, counted in
`nenpyo_unreadable_records_total` and yields no event: the rules match wrapped lines from
systemd and the kernel, which never carry one.

Any other crate needs a new ADR; a crate already a transitive dependency may become a direct
one, listed here.

## Consequences

- No crate beyond the stores' register.
- A house JSON line with a lone surrogate escape is invisible to the rules; if a rule ever
  needs such lines, nisshi's scanner is rewritten here (no shared code).
