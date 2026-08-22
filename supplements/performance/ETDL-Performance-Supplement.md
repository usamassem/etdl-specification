# ETDL Performance Supplement 1.0

**Supplement id:** `etdl.performance`
**Version:** 1.0.0
**Profile:** extends ETDL Core 1.0.0
**Category:** Core (Section 11.4 of the core specification)
**Status:** Under Development — NOT YET RELEASED
**Normative dependency:** ETDL Core 1.0.0

> **This supplement declares latency and throughput budgets against
> existing core nodes (Operation, Event Tree). It defines no new
> enforcement mechanism: a budget is a declared expectation for
> downstream tooling and deployment configuration (Section 6) to consult
> against the reference runtime's existing `RetryPolicy`/`timeoutMs`
> (core Section 5.9) and `SlaTracker` rolling window — it is not itself
> enforced by any Conforming Compiler.**

---

## 1. Relationship to ETDL Core

Core already carries two performance-adjacent fields on an Operation node
(Section 5.9): `timeoutMs`, a hard per-attempt timeout, and `retryPolicy`,
which core defines purely mechanically (attempt count, backoff) with no
notion of an acceptable latency *outcome*. This supplement adds that
missing piece: a declared latency budget (percentile targets) and,
optionally, a throughput expectation, for an Operation or for an entire
Event Tree end-to-end. It changes no core field's meaning and adds no new
core-level enforcement; `timeoutMs` remains the only value a Conforming
Compiler acts on mechanically.

**In-document representation.** Performance data is carried in an
`x-performance` extension field (core Section 11.1) at the document root:

```yaml
x-performance:
  budgets:
    - id: process-payment-budget
      nodeRef: "#/eventTrees/OrderFulfillment/nodes/ProcessPaymentOperation"
      p50Ms: 150
      p95Ms: 800
      p99Ms: 2000
      maxConcurrency: 200
      expectedRatePerSecond: 50
    - id: order-fulfillment-e2e-budget
      nodeRef: "#/eventTrees/OrderFulfillment"
      p50Ms: 400
      p95Ms: 2500
      p99Ms: 5000
```

---

## 2. Conformance

An implementation is **ETDL Performance 1.0 Conformant** when it:

1. is **ETDL Supplement-Aware** (core Section 2.3);
2. recognizes `etdl.performance` at version `1.0` (MAJOR `1`);
3. parses and validates every Budget Object under a declaring document's
   `x-performance.budgets` against this supplement's schema
   (`supplements/performance/ETDL-Performance-Schema.json`) and Section 4
   below;
4. enforces every `MUST`-level rule in Section 4 using the diagnostic
   codes of Section 5.

Per core Section 11.4, `etdl.performance` is a **core supplement**. This
does not change any Conforming Parser/Compiler's obligations (core Section
2.3) — a Budget Object is validated for internal consistency (Section 5)
but never enforced against measured runtime latency by anything this
specification defines.

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Budget** | A Budget Object (Section 4.1): declared latency percentile targets, and optionally a throughput expectation, for one Operation node or one whole Event Tree. |
| **`nodeRef`** | The Internal Reference (core Section 5.3.2) a Budget applies to: either `#/eventTrees/<id>/nodes/<operation-id>` (a single Operation) or `#/eventTrees/<id>` (the tree's initiating-event-to-terminal-Consequence path as a whole). |
| **Percentile target** | `p50Ms`/`p95Ms`/`p99Ms` (Section 4.1): the latency, in milliseconds, this Budget declares acceptable at the 50th/95th/99th percentile of observed calls. |

---

## 4. Data Model

### 4.1 Budget Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-performance.budgets`. |
| `nodeRef` | Internal Reference | REQUIRED | Resolves to `#/eventTrees/<id>/nodes/<operation-id>` (an Operation node) or `#/eventTrees/<id>` (a whole Event Tree). |
| `p50Ms` | number | REQUIRED | Positive, finite. |
| `p95Ms` | number | REQUIRED | Positive, finite; `>= p50Ms`. |
| `p99Ms` | number | REQUIRED | Positive, finite; `>= p95Ms`. |
| `maxConcurrency` | integer | OPTIONAL | Positive, if present. |
| `expectedRatePerSecond` | number | OPTIONAL | Positive, finite, if present. |

---

## 5. Validation Rules

| Code | Rule |
|---|---|
| E-160 | `x-performance` is present but a Budget Object's `nodeRef` does not resolve to an Operation node or an Event Tree (Section 4.1); or `p50Ms`/`p95Ms`/`p99Ms` is not finite, or is `<= 0`; or `maxConcurrency`/`expectedRatePerSecond` is present and `<= 0`. |
| E-161 | A Budget Object's percentile ordering is violated: `p50Ms > p95Ms` or `p95Ms > p99Ms`. |
| W-413 | Two Budget Objects declare the same `nodeRef` — the second is not an error, but a Conforming implementation SHOULD flag that only one is meaningfully authoritative for that node. |

`E-160`, `E-161`, and `W-413` are scoped to this supplement's own
namespace of meaning; they do not collide with core Section 7's codes or
with any other supplement's codes.

---

## 6. Relationship to Runtime Enforcement

A Budget's percentile targets are declarative, not self-enforcing. The
reference `etdl_core` runtime already has the mechanism a deployment would
configure to watch for a budget violation: `SlaTracker`'s rolling window
(`ETDL_SLA_WINDOW`, `ETDL_SLA_THRESHOLD`, core Section 9) flags an anomaly
in observed failure/latency behavior for a `BranchMonitor`-tracked node.
This supplement does not require, and a Conforming Compiler does not
perform, any automatic translation from a declared `p95Ms` into an
`ETDL_SLA_THRESHOLD` value — that translation, if a deployment wants it, is
an operational decision made outside this specification's scope, using the
Budget Object purely as the documented source of intent.

---

## 7. Compatibility

Silently ignoring `x-performance` (core Section 11.1's baseline behavior)
leaves a document fully valid under core alone — declared budgets are
additive metadata, never a precondition for parsing, validation, or code
generation.

A future `1.x` MINOR of this supplement may add fields to the Budget
Object (e.g. a memory or cost budget); it MUST NOT change the meaning of
`p50Ms`/`p95Ms`/`p99Ms` ordering (Section 4.1) without a MAJOR bump (core
Section 5.1.3).
