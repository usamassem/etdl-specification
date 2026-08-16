# Pass 1 — ETDL Specification Extension Framework Design Review

**Status:** Design review (read-only) — no specification text was edited during
this pass. This document records the review that precedes the implementation
pass (Pass 2).

**Scope:** introducing a generic **Supplement / Extension** mechanism into ETDL
Core, and designing **ETDL Reliability Supplement 1.0** as its first
domain-specific supplement.

**Design rule:** ETDL Core stays domain-neutral. The core learns only that "a
document may declare or use one or more formally identified supplements, and
each supplement may add validated semantics and metadata." Reliability-specific
ontology, probability models, statistics, databases, and AI never enter the core.

---

## 1. Repository state inspected

- `ETDL-Specification.md` (1142 lines, version 1.0.0)
- `schemas/etdl.schema.json` (197 lines)
- `VERSIONING.md`
- `examples/order-fulfillment.etdl`
- Compiler repo `etdl` (for implementation assumptions)

The specification is clean on `main` (`usamassem/etdl-specification`).

---

## 2. Conflicts with ETDL 1.0.0 identified (pre-existing, in scope for core fixes)

| # | Conflict | Location A | Location B | Nature |
|---|---|---|---|---|
| C1 | **Internal Reference shape** — only `#/faultTrees/<id>/topEvent` has "resolved meaning," but Transfers and Components need wider shapes | §5.3.2 | §5.11.1 (`#/faultTrees/<id>/...`), §5.4 (`$ref` "not restricted to topEvent shape") | Direct contradiction; E-105 also hard-codes the narrow shape |
| C2 | **`any()`/`all()` have no grammar** — §6.4 says MAY support them, but §6.2/Appendix A have no function-call production | §6.4 | §6.2 / Appendix A | Documented feature is unparseable by the normative grammar |
| C3 | **E-100 missing from §7.1** — the MAJOR-version/invalid-semver code exists only in Appendix B | §7.1 | Appendix B | Registry and body disagree |
| C4 | **E-103 / E-105 text drift** — §7.1 and Appendix B differ on wording and scope | §7.1 | Appendix B | Body vs registry inconsistency |
| C5 | **V-203 scope drift** — §7.3 defines sum-only; Appendix B adds "or branch probability outside [0,1]" | §7.3 | Appendix B | Two definitions of one code |
| C6 | **`x-*` vs schema** — §11 mandates `x-*` on any §5 object; schema `additionalProperties:false` rejects nested `x-*`, while root accepts unknown non-`x-` fields | §11 | `schemas/etdl.schema.json` | Schema contradicts §11 in both directions |
| C7 | **Schema misses MUST-level constraints** — branch probability presence, `outcome`, `action`, send channel/message, exactly-one prob/failureRate, VOTING `k`, INHIBIT `inhibitCondition`, per-type arity, non-empty `eventTrees`, strict SemVer | §5.8.1, 5.9, 5.10, 5.13, 5.14 | schema `$defs` | Schema is looser than §5 MUSTs |
| C8 | **§5.8.1 branch-probability MUST has no enforcement** | §5.8.1 | §7, schema | Unenforceable MUST |
| C9 | **House-event W-406 self-contradiction** | §5.14 | §7.7 W-406 | Advisory contradicts the object's semantics |
| C10 | **`undeveloped: true` deprecation not in VERSIONING.md** | §5.14, §10.3 | VERSIONING.md | Versioning doc incomplete |

These conflicts are pre-existing and **independent of the Supplement mechanism**.
They are fixed in Pass 2 so the core is internally consistent before the
extension mechanism is layered on.

---

## 3. Ambiguities (underspecified, non-blocking)

| # | Item | Location | Question |
|---|---|---|---|
| A1 | missionTime units | §5.14 | per-pair or global? no unit declaration field |
| A2 | §9.3 threshold | §9.3 | "10 percentage points" vs probability in [0,1]; threshold format unspecified |
| A3 | §9.4 lower-probability path | §9.4 | undefined for >2 branches / ties / default |
| A4 | retry jitter | §8.3 | referenced, never defined |
| A5 | any/all nesting | §6.4 | informal |
| A6 | transfer target exactness | §5.11.1 | `topEvent` vs any sub-path |

---

## 4. Proposed core modification — generic Supplement mechanism

### 4.1 Declaration (document-wide)

```yaml
etdl: "1.0.0"
info: { ... }
supplements:
  - id: etdl.reliability
    version: "1.0"
```

- Zero or more supplements may be declared.
- Declaration is **document-wide** (applies to the whole document).
- Documents that declare no supplements remain valid (backward compatible).
- `id` uses the namespace `etdl.<domain>`; `version` is SemVer.
- Optional `required: true` marks a supplement whose semantics are required for
  the document to be meaningful (e.g. external probability sources).

### 4.2 Where declared

At the document root, alongside `info` and `asyncapi_imports`. This is the
natural ETDL-style location and matches the "document-level metadata" pattern.

### 4.3 Unknown-supplement policy

Behavior for a supplement the implementation does not know:

- Configurable: `error` | `warning` | `ignore`.
- Default: **`warning`** (deterministic and documented).
- An implementation that does not implement a declared supplement MUST NOT
  silently pretend its semantics were applied; it reports per policy.

### 4.4 Core-only knowledge

The core validator recognizes the declaration shape and enforces identity /
versioning rules, but does **not** interpret domain-specific semantics. That is
the supplement's job.

### 4.5 Extension boundaries

A supplement MAY:

- add domain-specific metadata, entities, analysis annotations, references,
  external artifacts, validation rules, ontologies, analysis outputs, and
  semantics for extension fields.

A supplement MUST NOT silently:

- change the meaning of an existing ETDL field;
- change event-tree / fault-tree / probability / operation semantics;
- change branch ordering or generated-code semantics without an explicit
  extension;
- invalidate an otherwise valid ETDL Core document.

A supplement that needs fundamentally different tree semantics must define a
new extension/version, not override the core.

---

## 5. Proposed supplement identity & versioning

- Namespace: `etdl.<domain>` (e.g. `etdl.reliability`, `etdl.safety`,
  `etdl.security`, `etdl.diagnostics`).
- Version: SemVer (`MAJOR.MINOR.PATCH`).
  - MAJOR: incompatible semantic changes.
  - MINOR: backward-compatible additions.
  - PATCH: specification-error fixes without semantic change.
- Dependencies (optional, forward-looking): a supplement may declare
  `requires: [ { id, range } ]`. Dependency resolution is the implementation's
  concern, not core compiler business logic, unless it must reject an unknown
  required supplement.

---

## 6. Proposed conformance levels

| Level | Meaning |
|---|---|
| **ETDL Core Conformant** | implements core parse/validate/compile per §7/§8; recognizes but does not interpret supplements |
| **ETDL Supplement-Aware** | recognizes supplements, enforces identity/versioning/unknown policy, reports unapplied semantics |
| **ETDL Reliability 1.0 Conformant** | implements the Reliability Supplement 1.0 data model and semantics |

Distinguish syntax support / semantic support / analysis support / runtime
support.

---

## 7. Reliability Supplement 1.0 — design summary (Pass 2 scope)

The supplement is a **separate document** with its own schema. It:

- defines reliability terminology, ontology, failure modes/causes/effects/
  mechanisms, dependencies, common-cause failures, conditions;
- defines `ProbabilityEstimate` (id, value, metric, population, unit/time basis,
  conditions, source, method, uncertainty, confidence, assumptions, provenance,
  version, timestamp, status);
- distinguishes probability states: declared, assumed, measured, estimated,
  predicted, inferred, imported, unknown (unknown is explicit, never `0`);
- defines provenance, uncertainty, distributions (as a generic representation),
  reliability artifacts (data-only `.etdl` documents), external probability
  sources;
- defines runtime observations (immutable), statistical-update lifecycle,
  FMEA/FMECA as an **optional** profile, sensitivity/importance analysis
  outputs, cause/consequence traversal;
- separates **reliability** (probability/frequency of failure) from **risk**
  (probability × consequence/impact) — severity/impact never enter the core
  probability model;
- defines `ReliabilityBuildProvenance` for reproducibility (which ETDL, which
  supplement, which ontology/artifact/model/dataset/compiler);
- **does not** put algorithms in ETDL syntax: Bayesian/Monte-Carlo/ML/LLM/
  database/CRATE operations are implementation concerns; the supplement defines
  only the representation and semantics of results;
- marks future/optional capabilities explicitly rather than implying they are
  implemented.

---

## 8. Open decisions to confirm before Pass 2

1. **`any()`/`all()`:** add grammar (Option A) or demote out of 1.0.0 (Option B)?
   → Recommendation: keep 1.0.0 stable, add to a MINOR/next revision; do not
   expand core scope for this pass.
2. **Version field:** keep `1.0.0` (fold into the same draft→formal release) or
   bump to `1.1.0`? → Recommendation: keep `1.0.0`; the supplement mechanism is
   additive and backward compatible, and this pass does not change validated
   behavior of existing documents.
3. **House events:** require a probability (recommended) or keep advisory?
   → Recommendation: house-event probability is REQUIRED (it is the boundary
   condition); adjust W-406 to flag its absence, not its presence.
4. **Transfer target strictness:** require `topEvent` (strict) or allow any
   `#/faultTrees/<id>/...`? → Recommendation: allow any sub-path (matches
   current implementation), and widen E-105 wording accordingly.
5. **missionTime units:** add `info.units` field, or document "consistent per
   document" without a field? → Recommendation: document "consistent per
   document" for 1.0.0; a units field is a MINOR follow-up, not this pass.

These decisions were recorded; Pass 2 applies the recommended resolutions
unless superseded by the design review.

## 10. Conflict resolution status (updated after implementation)

All conflicts C1–C10 and ambiguities A1–A6 identified in this review have been
resolved in the core specification (commit `a7dfec6` supplement mechanism,
followed by the conflict-resolution pass):

| Item | Resolution |
|---|---|
| C1 / A6 | §5.3.2 now defines four resolved Internal Reference shapes; E-105 updated; §5.11.1/§5.4 consistent |
| C2 / A5 | `any`/`all` quantifier grammar added to §6.2 and Appendix A; §6.4 semantics defined |
| C3 | E-100 added to §7.1 |
| C4 | E-103 wording unified |
| C5 | V-203 wording unified (sum + range) |
| C6 | Schema root now rejects unknown non-`x-` (patternProperties `^x-` + additionalProperties false); nested `$defs` allow `x-*` |
| C7 | Schema enforces branch `outcome`, operation `action`, send channel/message, exactly-one probability/failureRate (with `x-*` deferral), VOTING `k`, INHIBIT `inhibitCondition`, non-empty `eventTrees`, strict SemVer |
| C8 | New V-205: branch supplies no probability field |
| C9 | House-event probability REQUIRED; W-406 now flags its absence |
| C10 | `undeveloped: true` deprecation added to VERSIONING.md |
| A1 | missionTime unit clarified as globally consistent per document |
| A2 | §9.3 threshold restated as probability-space `\|observed − declared\| > T`, default `0.10`; `ETDL_SLA_THRESHOLD` parse rule defined |
| A3 | §9.4 lower-probability-path selection defined (lowest non-default branch, ties by order; operation `onFailure`) |

The specification is internally consistent after this pass.

---

## 9. Pass 2 file plan (what will be produced)

```
ETDL-Specification.md            (core: supplement mechanism + conflict fixes)
schemas/etdl.schema.json         (core schema + supplements)
supplements/
  reliability/
    ETDL-Reliability-Supplement.md
    ETDL-Reliability-Schema.json
examples/
  extensions/
    reliability/
      core-only/
      basic-probability/
      external-probability/
      measured-probability/
      common-cause/
      observations/
      migration/
    hypothetical-future-supplement/
README.md                         (core vs supplement vs implementation)
```
