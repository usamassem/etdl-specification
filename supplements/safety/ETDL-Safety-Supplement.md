# ETDL Safety Supplement 1.0

**Supplement id:** `etdl.safety`
**Version:** 1.0.0
**Profile:** extends ETDL Core 1.0.0
**Category:** Core (Section 11.4 of the core specification)
**Status:** Under Development — NOT YET RELEASED
**Normative dependency:** ETDL Core 1.0.0

> **This supplement classifies hazards and gives safety meaning — Safety
> Integrity Level, independence, common-cause grouping — to structures ETDL
> Core already defines: the Consequence node (Section 5.10) and the Barrier
> node (Section 5.8). It defines no new probability mathematics; residual
> risk is read from the core Fault-Tree evaluation (Section 5.16) this
> supplement's own compiler obligations do not duplicate.**

---

## 1. Relationship to ETDL Core

ETDL Core already has the two structures a safety case is built from: a
**Barrier** node (Section 5.8), which core defines only as a branch point
with declared or derived probabilities, and a **Fault Tree** (Section 5.11),
which core evaluates to a Top Event probability (Section 5.16) with no
safety-specific interpretation attached. This supplement gives that
existing structure safety meaning — which barriers are safety barriers,
what hazard each protects against, what Safety Integrity Level each is
assigned, and which barriers must not share a failure cause — without
changing how core parses, validates, or evaluates either structure.

This supplement has no dependency on the Reliability Supplement
(`etdl.reliability`) or the Tree Event Supplement (`etdl.tree-event`). A
document MAY combine all three — a Barrier's SIL assignment (this
supplement) and its branch probability (core, possibly reliability-derived)
describe the same node from two independent, composable angles, exactly the
composition pattern core Section 11.2 describes for supplements in general.

**In-document representation.** Safety data is carried in an `x-safety`
extension field (core Section 11.1) at the document root:

```yaml
x-safety:
  hazards:
    - id: gateway-unavailable-during-payment
      description: "payment cannot be captured while the gateway is down"
      severity: critical
      likelihood: remote
      riskIndex: 2
      consequenceRef: "#/eventTrees/OrderFulfillment/nodes/PaymentFailedConsequence"
  barriers:
    - id: retry-barrier
      nodeRef: "#/eventTrees/OrderFulfillment/nodes/RetryBarrier"
      sil: 2
      independentOf: ["fallback-gateway-barrier"]
      commonCauseGroup: "primary-network-path"
    - id: fallback-gateway-barrier
      nodeRef: "#/eventTrees/OrderFulfillment/nodes/FallbackBarrier"
      sil: 1
      independentOf: ["retry-barrier"]
      commonCauseGroup: "secondary-network-path"
```

---

## 2. Conformance

An implementation is **ETDL Safety 1.0 Conformant** when it:

1. is **ETDL Supplement-Aware** (core Section 2.3);
2. recognizes `etdl.safety` at version `1.0` (MAJOR `1`);
3. parses and validates every Hazard Object and Safety Barrier Object under
   a declaring document's `x-safety` against this supplement's schema
   (`supplements/safety/ETDL-Safety-Schema.json`) and Section 4 below;
4. enforces every `MUST`-level rule in Section 4 using the diagnostic codes
   of Section 5.

Per core Section 11.4, `etdl.safety` is a **core supplement**: its
normative specification is authored and maintained by this
specification's own maintainers, cross-referenced from core Appendix E.2.
This does not change any Conforming Parser/Compiler's obligations (core
Section 2.3) — no conformance target requires implementing this
supplement.

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Hazard** | A Hazard Object (Section 4.1): a named source of harm, classified by severity and likelihood, tied to a specific Consequence. |
| **Severity** | One of `catastrophic`, `critical`, `marginal`, `negligible` (Section 4.1) — how bad the hazard's outcome is if it occurs, independent of how likely it is. |
| **Likelihood** | One of `frequent`, `probable`, `occasional`, `remote`, `improbable` (Section 4.1) — how often the hazard is expected to occur, independent of severity. |
| **Risk Index** | An integer 1–4 read from the Section 4.1 risk matrix for a given severity/likelihood pair; lower is worse. |
| **Safety Barrier** | A Safety Barrier Object (Section 4.2): a core Barrier node given a Safety Integrity Level and an independence declaration. |
| **Safety Integrity Level (SIL)** | An integer 1–4 (IEC 61508 scale; 4 is the strongest) a Safety Barrier is assigned, describing the rigor expected of it — this supplement records the assignment; it does not itself certify or derive one. |
| **Common-Cause Group** | A free-form string tag (Section 4.2) identifying a shared failure cause; two barriers sharing a tag are not independent regardless of an `independentOf` declaration. |

---

## 4. Data Model

### 4.1 Hazard Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-safety.hazards`. |
| `description` | string | REQUIRED | Human-readable description of the harm. |
| `severity` | enum | REQUIRED | One of `catastrophic`, `critical`, `marginal`, `negligible`. |
| `likelihood` | enum | REQUIRED | One of `frequent`, `probable`, `occasional`, `remote`, `improbable`. |
| `riskIndex` | integer | REQUIRED | Declared risk index, 1–4; SHOULD equal the matrix lookup below. A mismatch is not rejected (Section 5, W-410) — declaring an index intentionally more conservative than the matrix is a legitimate, common practice this supplement does not forbid. |
| `consequenceRef` | Internal Reference | REQUIRED | Resolves to `#/eventTrees/<id>/nodes/<consequence-id>` (a Consequence node, core Section 5.10). |

**Risk matrix** (severity × likelihood → Risk Index), the reference this
supplement's `riskIndex` field is checked against:

| | frequent | probable | occasional | remote | improbable |
|---|---|---|---|---|---|
| **catastrophic** | 1 | 1 | 1 | 2 | 2 |
| **critical** | 1 | 1 | 2 | 2 | 3 |
| **marginal** | 1 | 2 | 3 | 3 | 4 |
| **negligible** | 2 | 3 | 4 | 4 | 4 |

### 4.2 Safety Barrier Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-safety.barriers`. |
| `nodeRef` | Internal Reference | REQUIRED | Resolves to `#/eventTrees/<id>/nodes/<barrier-id>` (a Barrier node, core Section 5.8). |
| `sil` | integer | REQUIRED | Safety Integrity Level, 1–4. |
| `independentOf` | Array[string] | OPTIONAL, default `[]` | `id`s of other Safety Barrier Objects this one is claimed to share no common cause with. |
| `commonCauseGroup` | string | OPTIONAL | A shared-failure-cause tag (Section 3). |

---

## 5. Validation Rules

| Code | Rule |
|---|---|
| E-130 | `x-safety` is present but a Hazard Object's `severity`/`likelihood` is not one of Section 4.1's enumerated values, or a Safety Barrier Object's `sil` is not an integer in `[1,4]`. |
| E-131 | A Hazard Object's `consequenceRef`, or a Safety Barrier Object's `nodeRef`, does not resolve to a node of the required kind (Section 4.1/4.2). |
| E-132 | Two Safety Barrier Objects list each other (or are transitively connected through further mutual `independentOf` declarations) in `independentOf` while sharing the same non-empty `commonCauseGroup` — a self-contradictory independence claim. |
| W-410 | A Hazard Object's declared `riskIndex` does not equal Section 4.1's matrix value for its `severity`/`likelihood` pair. |

`E-130`–`E-132` and `W-410` are scoped to this supplement's own namespace of
meaning; they do not collide with core Section 7's codes or with any other
supplement's codes.

---

## 6. Residual Risk — Relationship to Fault-Tree Evaluation

A Hazard's `consequenceRef` (Section 4.1) typically names a Consequence
reached through an Operation whose failure path is protected by one or more
Safety Barriers (Section 4.2), which in turn may derive their branch
probability from a Fault Tree's Top Event (core Section 5.15–5.16). This
supplement defines no new computation over that value: the residual
probability of a hazard occurring, after its protecting barriers, is
exactly the core-computed branch probability already reachable through the
Event Tree — this supplement only adds the hazard classification (Section
4.1) and barrier metadata (Section 4.2) an external safety-case tool needs
to interpret that number, and does not recompute, re-derive, or override it.

---

## 7. Compatibility

Silently ignoring `x-safety` (core Section 11.1's baseline behavior for an
unrecognized extension field) leaves a document fully valid under core
alone — hazard classification and SIL assignment are additive metadata,
never a precondition for core parsing, validation, or code generation.

A future `1.x` MINOR of this supplement may add hazard/barrier fields
(e.g. a mitigation-tracking field); it MUST NOT change the meaning of
`severity`, `likelihood`, `sil`, or the risk matrix in Section 4.1 without a
MAJOR bump (core Section 5.1.3's supplement versioning rule).
