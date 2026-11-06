# nenpyo (年表)

> Event store of the sekkeizu observability stack: derives dated events (boots, unit failures,
> restarts, OOM kills, agent gaps, deploys) from the logs held by [nisshi](../nisshi), keeps
> them in an open format and answers range queries on them fast. Rust.

*年表 (nenpyō): a chronological table, the list of dated events beside a history.*

**Status:** design (ADRs proposed, no code).

## Why

Metrics are sampled every 30 s: a service that crashes and restarts between two scrapes leaves
two healthy points. The logs saw it. nenpyo reads them as they arrive and keeps what happened as
events, small enough to query in milliseconds where a log scan takes seconds.

## Role

- `nenpyo-derive`: the only writer. Reads nisshi's files read-only, applies the rules, writes
  one event file per node and receive day, seals and keeps them.
- `nenpyo-query`: read-only query daemon over the same files, for [yagura](../yagura),
  [kenbun](../kenbun) and [hansho](../hansho).
- `nenpyo`: data CLI (`read`, `verify`, `rebuild`).

## Not its job

Storing logs ([nisshi](../nisshi)) or metrics ([masu](../masu)), error rates and log volumes
(queries on nisshi), alerting ([hansho](../hansho)), views ([yagura](../yagura)).

## Documentation

- [ADRs](docs/adr/README.md)
