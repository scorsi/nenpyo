# ADR 0005 — Rules

Status: Proposed — 2026-10-30

## Context

A rule turns a log record into an event (ADR 0004). Most rules match systemd and kernel lines,
whose structured fields hikyaku copies into wrapped records under `journal` (hikyaku ADR 0014):
`MESSAGE_ID`, `UNIT`, `_BOOT_ID`, `_TRANSPORT`, `_SYSTEMD_UNIT`… Any journald client can set
`MESSAGE_ID` and `UNIT` on its own lines; `_TRANSPORT` and `_SYSTEMD_UNIT` are written by
journald and cannot be forged by a client. The vocabulary is closed and owned by nenpyo.

## Options considered

- **Declarative rules in the Nix configuration** (match fields, extract captures): a language
  to design, document and test, for a vocabulary that changes by ADR anyway.
- **Rules in Rust, versioned, defined by fixtures.**

## Decision

- Rules are Rust code in `nenpyo-core`. The rule set has an integer version, bumped by any
  change to what it derives; every event carries it (`rules`).
- **Stateless**: a rule sees one record and yields zero or one event. No windows, no memory of
  earlier records: the output depends on each record alone, whatever the arrival order
  (ADR 0007).
- **Trusted fields only**:
  - systemd rules require `journal._SYSTEMD_UNIT = "init.scope"` (PID 1), then match
    `journal.MESSAGE_ID` and read `journal.UNIT` and the result fields;
  - kernel rules require `journal._TRANSPORT = "kernel"`;
  - `coredump` requires `journal._SYSTEMD_UNIT` to start with `systemd-coredump@`;
  - stack rules match house JSON lines by `service` and `event` (hikyaku's events from service
    `hikyaku`, nisshi's `seq-gap` from nisshi's reserved stream of 32 zeros);
  - `deploy` requires a `service` listed in `deploy_services` (ADR 0004).
- **The rules are the fixtures**: `fixtures/rules/` holds records taken from a real journal of
  the systemd and kernel versions nenpyo is tested on, each with the event it must yield or
  none. A `MESSAGE_ID` or a kernel message format is pinned by a fixture, never trusted from
  documentation. A rule without a fixture is not a rule.
- Kernel lines carry no `MESSAGE_ID`: the OOM rule parses the message text
  (`Out of memory: Killed process <pid> (<comm>)` and the `oom-kill:` line for the cgroup),
  pinned by fixtures on each tested kernel.
- The only configuration of the rules is `deploy_services`.

## Consequences

- Adding or changing a rule is a release of nenpyo; it applies from its deployment on. Older
  events keep their `rules` version until a manual rebuild (ADR 0007).
- A systemd or kernel upgrade that changes a message is caught by the fixtures recorded on the
  new version, not in production.
