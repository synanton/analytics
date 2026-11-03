# Analytics documentation

Documentation for the Synanton Analytics implementation.

## Architecture

- [Implementation architecture proposal](architecture/analytics-implementation-proposal.md) — goals, platform boundaries, source integration, data model, Iceberg/Flink/Spark responsibilities, security, temporal semantics, lineage, operations, and phased delivery.

## Platform contracts

The proposal is subordinate to the platform's normative designs:

- [Design 1.23 — Security](https://github.com/synanton/platform/blob/main/docs/architecture/synanton-design-1.23.md)
- [Design 1.27 — Async substrate](https://github.com/synanton/platform/blob/main/docs/architecture/synanton-design-1.27.md)
- [Design 1.32 — API surface](https://github.com/synanton/platform/blob/main/docs/architecture/synanton-design-1.32.md)
- [Design 1.34 — Temporal semantics](https://github.com/synanton/platform/blob/main/docs/architecture/synanton-design-1.34.md)

The parent design and implementation gates are tracked in [SNTP-036](https://github.com/synanton/platform/issues/474).

## Status

The architecture proposal is a draft for review. It describes a target implementation and does not imply that the proposed modules, jobs, or infrastructure are already implemented.
