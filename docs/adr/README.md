# Architecture Decision Records

MADR-lite: one decision per file, `NNNN-slug.md`, sections Context / Options considered /
Decision / Consequences. A decision is changed by a new ADR that supersedes the old one; before
the first deployment, ADRs are amended in place.

| ADR | Title |
|---|---|
| [0001](0001-scope.md) | Scope and boundaries |
| [0002](0002-language-and-dependencies.md) | Language and dependencies |
| [0003](0003-reading-nisshi.md) | Reading nisshi |
| [0004](0004-event-schema.md) | Event schema and vocabulary |
| [0005](0005-rules.md) | Rules |
| [0006](0006-storage.md) | Storage, sealing and retention |
| [0007](0007-determinism-and-rebuild.md) | Determinism, resume and rebuild |
| [0008](0008-query-api.md) | Query API and authentication |
| [0009](0009-binaries.md) | Three binaries: data CLI, deriver, query daemon |
| [0010](0010-testing.md) | Testing |
