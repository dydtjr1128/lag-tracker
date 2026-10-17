# Document Index

These are design-stage documents. Once implementation starts, each document becomes the source of truth for the behavior it describes and is updated together with the code.

## Reading order

| Order | Document | Owns |
| --- | --- | --- |
| 1 | [Overview and requirements](design/overview.md) | Background, goals and non-goals, users, functional and non-functional requirements, success criteria, terms |
| 2 | [Architecture](design/architecture.md) | Structure, run modes, threads and tick plan, crates, time base, degradation, error handling, build and distribution |
| 3 | [Metrics](design/metrics.md) | Source, interval, permission, and meaning of each metric, Chrome roles, process selection |
| 4 | [Detection and likely causes](design/detection.md) | Incident model, ring buffer and capture, automatic rules, marks, contribution, P2P suspicion, external logs |
| 5 | [Storage and privacy](design/storage.md) | Files, schema, size budget, data never collected, sharing |
| 6 | [UI/UX](design/ui-ux.md) | User flow, live screen, keys, exit, report, copy principles, command line |
| 7 | [Roadmap](design/roadmap.md) | Phase scope, acceptance criteria and validation, risks, open questions |

## Conventions

- "Initial value" next to a threshold is an assumption made before measurement. When a measurement changes it, record the evidence in the same document.
- "Needs verification" marks behavior that has not been confirmed yet. The acceptance criteria of the relevant phase confirm it.
- Work, writing, and Git rules are in [AGENTS.md](../AGENTS.md).
