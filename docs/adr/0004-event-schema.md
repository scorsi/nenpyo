# ADR 0004 — Event schema and vocabulary

Status: Proposed — 2026-10-30

## Context

Consumers (yagura's timelines, kenbun, hansho) filter events by interval, node, type and unit,
and every event must lead back to the log line it came from. Events are rare (a few hundred a
day on the whole stack) and kept as long as the logs.

## Options considered

- **Free-form events** (any type, any fields): every consumer guesses what a type means.
- **A closed vocabulary owned by nenpyo**, each type with its fields, changed by ADR.

## Decision

One event = one JSON object, one line, members in this order:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `v` | integer | yes | `1` |
| `ts` | string | yes | event time: the source record's `ts`, nisshi's format (9 fractional digits, `Z`) |
| `node` | string | yes | the source record's `node` |
| `type` | string | yes | from the vocabulary below |
| `unit` | string | no | the systemd unit or house service the event is about, full unit name when known (`annai.service`) |
| `attrs` | object | no | the type's fields below, nothing else |
| `src` | object | yes | `{"day": "<nisshi receive day>", "stream": "<32 hex>", "seq": <n>}`: the source record, unique in nisshi |
| `rules` | integer | yes | version of the rule set that derived it (ADR 0005) |

An event's identity is `(src.stream, src.seq, type)`. One source record yields at most one
event per type.

Vocabulary v1:

| Type | From | `unit` | `attrs` |
|---|---|---|---|
| `boot` | systemd's start of a new boot | — | `boot_id` |
| `unit-failed` | systemd: a unit entered the failed state | yes | `result` |
| `unit-exited` | systemd: a unit's main process exited abnormally (killed, dumped, or a non-zero status) | yes | `exit_code`, `exit_status` |
| `unit-restart` | systemd: a restart of the unit was scheduled | yes | `restarts` (systemd's counter) |
| `unit-oom` | systemd: a process of the unit was killed by the OOM killer | yes | — |
| `oom-kill` | kernel: the OOM killer killed a process | when the kernel line names the cgroup's unit | `pid`, `comm` |
| `coredump` | systemd-coredump: a process dumped core | yes, `COREDUMP_UNIT` | `signal` |
| `agent-start` | hikyaku `agent-start` | — | `version`, `fresh_spool` |
| `journal-gap` | hikyaku `journal-gap` | — | `from_ts`, `to_ts` |
| `spool-full`, `spool-resumed` | hikyaku | — | `signal` |
| `seq-gap` | nisshi `seq-gap` | — | `gap_stream`, `from_seq`, `to_seq` |
| `deploy` | a `deploy` line (below) | — | `revision` |

The `deploy` line is the contract for sources that record an event through the logs: a house
JSON line in nisshi's schema with `event:"deploy"` and a string `revision`, from a service
named in nenpyo's configuration (`deploy_services`, empty by default). Later sources (hansho's
alerts) get their own types the same way, by amending this ADR.

## Consequences

- A new type, or a new field of a type, amends this ADR; a changed meaning is a new type.
- `rules` lets a consumer tell events of two rule sets apart in a history that was not rebuilt
  (ADR 0007).
- Error bursts are not events (ADR 0001).
