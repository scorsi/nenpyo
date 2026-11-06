# ADR 0008 — Query API and authentication

Status: Proposed — 2026-10-30, rewritten 2026-11-01 on the shared query document

## Context

yagura draws events on its timelines and lists them per node and unit; kenbun and hansho read
them too. The stack has no query language: each store answers a structured API (yagura ADR
0002). The stores share one request contract, the query document (masu ADR 0013): a batch of
named queries over one `[from, to)` range, answered in one pass, protobuf or protojson over
HTTPS POST. Readers authenticate like the stores' readers (masu ADR 0007, nisshi ADR 0008).

## Options considered

- **`GET` routes with query parameters** (`/v1/events`, `/v1/counts`): the first draft of this
  ADR. Simple, but a shape of its own next to masu's and nisshi's, and one request per tile.
- **The query document**: the same envelope, errors, paging and frames as the other stores;
  nenpyo adds only its two operators. Chosen.

## Decision

`nenpyo-query` serves `POST /v1/query` as masu ADR 0013 specifies (HTTPS, HTTP/1.1 only,
`application/x-protobuf` or `application/json`, identity `reader`, envelope messages and
`docs/query-document.md` mirrored byte-identical), with `proto/nenpyo/v1/query.proto`:

```proto
message Query {
  oneof body {
    EventsQuery events = 1;
    CountsQuery counts = 2;
  }
}
message EventsQuery {
  repeated string node = 1;   // any of; empty = all
  repeated string type = 2;
  repeated string unit = 3;
  uint32 limit = 4;           // default 1000, at most 10 000
  string after = 5;           // cursor of a previous result
}
message CountsQuery {
  repeated string node = 1;
  repeated string type = 2;
  repeated string unit = 3;
  google.protobuf.Duration step = 4;  // at least 1 min; at most 10 000 buckets in the range
  repeated string by = 5;             // node, type, unit
}
```

- **`events`**: a `TableFrame` whose rows are the event objects of ADR 0004, `from <= ts < to`,
  sorted by `ts`, then `node`, `src.stream`, `src.seq`; `cursor` while more remain, bound to
  the query's filters and range (ADR 0013 of masu).
- **`counts`**: a `CountsFrame`, one `CountGroup` per distinct combination of the `by` fields,
  `counts[i]` the events in bucket `i`. This is what replaces counters in masu (ADR 0001).
- `GET /v1/status`: per node, nenpyo's position, nisshi's committed position and the lag
  between them in bytes and seconds; the rules version; free disk and reserve; whether the
  deriver is stopped (disk, a changed sealed source) and why.
- Times are RFC 3339 only in the API; the `-1h` offsets of `nenpyo read` are resolved by the
  CLI. Initial budgets: 64 queries and 31 days per request, 10 s wall time, 2 concurrent
  requests, 2 000 000 rows and buckets accumulated across results, 16 MiB response; per query
  the `limit` and bucket ceilings above. A per-request scan budget hit is 422 `scan-limit`,
  an aggregate result one 422 `response-limit` counted while frames are built, a per-query
  one that query's `result-limit`. `step > 0`; the last bucket is cut at `to`.

Execution: the daemon reads manifests at start and when new ones appear, keeps `min_ts`,
`max_ts` and type counts per sealed file in memory, decompresses only the files whose range
meets the request, reads open files up to their committed length, and feeds every query of
the request from that one traversal.

Authentication, the stores' scheme: TLS, bearer token, the configuration maps hex
`sha256(token)` to `reader`; constant-time comparison; 401 without valid credentials, 403 for
a valid token without the `reader` role, both before the body is decoded. Then hanko
certificates and mTLS with SPIFFE IDs, in the same steps as the stores.

Own metrics on a loopback listener, masu's convention, for both daemons: `nenpyo_records_read_total{source_node}`,
`nenpyo_events_total{type}`, `nenpyo_unreadable_records_total`, `nenpyo_lag_seconds{source_node}`,
`nenpyo_seals_total{result}`, `nenpyo_queries_total{code}`, `nenpyo_disk_free_bytes`,
`nenpyo_build_info{version}`.

## Consequences

- yagura and kenbun never read nenpyo's files, and use the same client core as for masu and
  nisshi; a timeline tile and a whole dashboard are the same request shape.
- `prost-types` becomes a direct dependency (well-known types), recorded in ADR 0002; it is
  already transitive through `prost-build`.
- A query over a long range on a node with many late deliveries decompresses more files; at
  the expected volume this is milliseconds. An index comes only if measurements ask for it.
