# ETDL Versioning and Language Stability

This document defines the ETDL language evolution policy: specification
versions, compiler/parser compatibility, generated-code compatibility, backward
compatibility, deprecation, and feature stability levels.

## Specification versions

- The `etdl` field in a document is **SemVer** (`MAJOR.MINOR.PATCH`).
  - **MAJOR** — breaking changes to schema or validation rules.
  - **MINOR** — additive, backward-compatible changes (new optional fields, new
    gate types, ...).
  - **PATCH** — clarifications, non-behavioral edits.
- The current specification is **1.0.0**.
- `info.version` is the author's own document version and is independent of the
  `etdl` field.

## Compiler / parser compatibility

- A parser **MUST** accept any document whose `etdl` MAJOR matches its supported
  MAJOR.
- A parser **MUST** reject unimplemented future MAJORs (reference behavior:
  `E-100`).
- A parser **MAY** accept other MAJORs best-effort.
- `x-*` extension fields are preserved; unknown non-`x-` fields are rejected.

## Generated-code compatibility

- The Rust backend's generated-code **contract** (handler signature, message
  serialization, publisher seam) is part of the compiler's public surface. A
  change to it is a MINOR/breaking change for `etdl-compiler`, documented in the
  changelog.
- Generated output is validated by a compile-check harness
  (`etdl-compiler/tests/codegen_test.rs`) before release.
- Generated code is deterministic for identical input + compiler version.

## Backward compatibility

Within a MAJOR version:

- Valid documents keep validating (except where a spec correction explicitly
  requires otherwise, which is a MAJOR change).
- Deprecated fields remain accepted for at least one full MAJOR cycle.

## Deprecation policy

1. A feature is marked DEPRECATED in the spec and emits an advisory (W-).
2. It remains fully functional for ≥ 1 MAJOR cycle.
3. It may be removed only in a MAJOR release, with a migration note.

Currently deprecated:

| Feature | Replacement | Introduced |
|---|---|---|
| `eventTree` (singular) | `eventTrees` (map) | always |
| `probabilityOfSuccess` / `probabilityOfFailure` | `probability` keyed by outcome | always |
| `undeveloped: true` on a Basic Event | `eventType: "undeveloped"` | always |

## Breaking changes

Breaking changes are only permitted in a MAJOR release and require:

- a changelog entry,
- a migration section,
- updated conformance cases,
- updated generated-code contract documentation.

## Experimental features

Experimental language features MUST be labeled (e.g. documented as
EXPERIMENTAL in the spec and, where feasible, only accepted when an
implementation opts in). They are not covered by compatibility guarantees.

## Feature stability levels

| Level | Meaning | Change policy |
|---|---|---|
| STABLE | Relied upon; fully tested | Only via MINOR/breaking releases with notes |
| EXPERIMENTAL | May change; opt-in | Any release; marked in docs |
| DEPRECATED | Keep ≥ 1 MAJOR cycle | Removed only in MAJOR |
| INTERNAL | Not part of the contract | Any time |

## Reference implementation versioning

All ETDL crates share one workspace version. The language `etdl` field (1.x)
and the compiler crate version (0.x pre-1.0) are deliberately decoupled: the
compiler tracks the spec MAJOR it implements, while its own crate version tracks
implementation maturity.

## Changelog convention

- `CHANGELOG.md` at the repository root.
- Entries grouped: `Added`, `Changed`, `Fixed`, `Deprecated`, `Removed`.
- Each entry links to the spec requirement it implements, if applicable.
