# ADR 0001 — Scope and boundaries

Status: Proposed — 2026-10-30

## Context

masu archives scrapes taken every 30 s. A unit that crashes and is restarted between two
scrapes, an OOM kill, a reboot under a minute: the metrics show two healthy points and nothing
in between. The logs in nisshi hold what happened, but answering "which units restarted on
which node this month" means decompressing and parsing every line of the range.

The usual answer is to derive data from logs at ingestion: Loki recording rules, Vector and
Fluent Bit `log_to_metric`, Datadog metrics from logs, the OpenTelemetry count connector,
mtail. Events are usually kept apart from metrics (Grafana annotations, Kubernetes Events).

Two places were ruled out by the owner: hikyaku runs on every node under a small memory bound
and gets no derivation work; nisshi stays a log store with one job, ingestion and query.

## Options considered

- **An ETL pushing counters to masu and events to a separate event store with its own push
  API**: two contracts and two writers for one derivation. Counters also fight the time axis:
  nisshi files by receive day and a node may deliver weeks late, so a counter dated at event
  time would need rewriting the past in an append-only archive, and one dated at receive time
  dumps weeks of restarts at once. The `node` label is reserved by masu's metric convention,
  and a node token pushes only its own node.
- **Events pushed directly by their sources** (deployments, alerts) beside the derived ones:
  a second entry path, and events that are no longer rebuildable from the logs.
- **One product, one writer, logs as the only input**: every event comes from a log line in
  nisshi; the deriver is the store's only writer, so no push protocol is needed.

## Decision

- nenpyo derives dated events from nisshi's files and stores them; that is its whole input.
  A source that wants an event recorded (a deployment, hansho's alerts, kajiba) writes a log
  line in nisshi's schema, which reaches nisshi through hikyaku like any other line. The logs
  stay the single source of truth: hikyaku's spool, authentication and delivery, and nisshi's
  deduplication and hash chain, cover events too.
- No push API: `nenpyo-derive` is the only writer of its root, as `nisshi-store` is of its
  own; `nenpyo-query` only reads it (ADR 0009).
- Nothing is written to masu. Counts over time (restarts per unit per day) are queries on
  nenpyo (ADR 0008). A metric derived from events comes back only if hansho needs metric rules
  that cannot query nenpyo.
- Out of scope: error bursts, error rates and log volumes per service and level (counts over
  nisshi, a job for nisshi's query daemon); alerting (hansho); views (yagura).

## Consequences

- nenpyo runs on the same host as nisshi, with read access to its data directory.
- The journald fields the rules need (`MESSAGE_ID`, `UNIT`, `_BOOT_ID`, `_TRANSPORT`…) must
  reach nisshi: hikyaku ADR 0014 copies them into wrapped lines, and nisshi's log schema
  documents them (amendment pending). Logs stored before that change carry none, and yield
  only the events of house JSON lines.
- An event recorded through the logs costs a few seconds more than a direct push: hikyaku's
  delivery, then the deriver's poll.
- An event about nisshi being down reaches nenpyo only once nisshi is back; nenpyo is a
  history, not an alerting path.
