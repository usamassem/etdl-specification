# ETDL Diagnostics Supplement 1.0

**Supplement id:** `etdl.diagnostics`
**Version:** 1.0.0
**Profile:** extends ETDL Core 1.0.0
**Category:** Core (Section 11.4 of the core specification)
**Status:** Under Development — NOT YET RELEASED
**Normative dependency:** ETDL Core 1.0.0

> **This supplement is structural metadata only: it declares which
> runtime telemetry attribute a document's author expects to correlate
> with which Fault-Tree cause, for a human or an external tool doing
> post-incident triage to consult. It defines no new runtime behavior of
> its own, adds no new obligation to the reference `etdl_core` runtime
> (`BranchMonitor`/`SlaTracker`/telemetry), and performs no automated
> root-cause inference — see Section 6.**

---

## 1. Relationship to ETDL Core

The reference ETDL implementation's runtime already produces the raw
material a root-cause correlation needs, without this supplement's help:
`BranchMonitor` records branch and failure outcomes; the W3C `traceparent`
span carries a node-id attribute (core Section 9, conventionally
`etdl.node.id`); an `SlaTracker` rolling window flags an anomaly. None of
that is normatively defined by core beyond Section 9's telemetry
requirements — core says a span attribute exists and is populated; it says
nothing about which Fault-Tree cause a given attribute value is *evidence
for*. This supplement adds exactly that mapping, as declared metadata a
document's author supplies, not as something a compiler computes.

**In-document representation.** Diagnostics data is carried in an
`x-diagnostics` extension field (core Section 11.1) at the document root:

```yaml
x-diagnostics:
  correlations:
    - id: gateway-timeout-correlation
      spanAttribute: "etdl.node.id"
      spanValue: "ProcessPaymentOperation"
      causeRef: "#/faultTrees/PaymentGatewayFailure/basicEvents/GatewayUnreachable"
      description: "an anomaly on this span most often traces back to gateway unreachability"
  anomalyRules:
    - id: payment-operation-anomaly
      monitors: "#/eventTrees/OrderFulfillment/nodes/ProcessPaymentOperation"
      description: "watch ETDL_SLA_THRESHOLD/ETDL_SLA_WINDOW on this node's BranchMonitor for elevated failure rate"
```

---

## 2. Conformance

An implementation is **ETDL Diagnostics 1.0 Conformant** when it:

1. is **ETDL Supplement-Aware** (core Section 2.3);
2. recognizes `etdl.diagnostics` at version `1.0` (MAJOR `1`);
3. parses and validates every Correlation Object and Anomaly Rule Object
   under a declaring document's `x-diagnostics` against this supplement's
   schema (`supplements/diagnostics/ETDL-Diagnostics-Schema.json`) and
   Section 4 below;
4. enforces every `MUST`-level rule in Section 4 using the diagnostic
   codes of Section 5.

Per core Section 11.4, `etdl.diagnostics` is a **core supplement**. This
does not change any Conforming Parser/Compiler's obligations (core Section
2.3), and does not require the reference `etdl_core` runtime, or any
runtime, to change its telemetry behavior — Section 6 makes this explicit.

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Correlation** | A Correlation Object (Section 4.1): a declared association between a runtime telemetry span attribute/value and a Fault-Tree cause. |
| **Cause** | A Fault-Tree Gate or Basic Event (core Section 5.11), referenced by a Correlation's `causeRef`. |
| **Anomaly Rule** | An Anomaly Rule Object (Section 4.2): a declaration that a specific node's runtime behavior is worth monitoring for anomalies, without specifying a new detection mechanism (core `etdl_core::sla::SlaTracker` already exists for that). |

---

## 4. Data Model

### 4.1 Correlation Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-diagnostics.correlations`. |
| `spanAttribute` | string | REQUIRED | The telemetry span attribute name (e.g. `"etdl.node.id"`, core Section 9) this correlation is keyed on. |
| `spanValue` | string | REQUIRED | The attribute value that indicates this cause is implicated. |
| `causeRef` | Internal Reference | REQUIRED | Resolves to `#/faultTrees/<id>/gates/<gate-id>` or `#/faultTrees/<id>/basicEvents/<event-id>` (core Section 5.3.2). |
| `description` | string | OPTIONAL | Human-readable rationale for the correlation. |

### 4.2 Anomaly Rule Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-diagnostics.anomalyRules`. |
| `monitors` | Internal Reference | REQUIRED | Resolves to `#/eventTrees/<id>/nodes/<node-id>` (any node kind, core Section 5.7). |
| `description` | string | OPTIONAL | Human-readable note on what an anomaly at this node would mean; MAY reference the reference runtime's `ETDL_SLA_THRESHOLD`/`ETDL_SLA_WINDOW` configuration (core Section 9), but this supplement does not require or interpret those variables itself. |

---

## 5. Validation Rules

| Code | Rule |
|---|---|
| E-150 | `x-diagnostics` is present but a Correlation Object's `causeRef` does not resolve to a Gate or Basic Event, or an Anomaly Rule Object's `monitors` does not resolve to a node (Section 4.1/4.2). |
| E-151 | Two Correlation Objects declare the same `id`, or two Anomaly Rule Objects declare the same `id`, within the same document. |
| W-412 | An Anomaly Rule Object's `monitors` node is an Operation with neither `onFailureProbabilitySource` nor any Fault Tree reachable from this document that a Correlation Object's `causeRef` could plausibly connect it to — a monitored node with no correlated cause on record. |

`E-150`, `E-151`, and `W-412` are scoped to this supplement's own
namespace of meaning; they do not collide with core Section 7's codes or
with any other supplement's codes.

---

## 6. Scope and Non-Goals

This supplement performs no automated correlation, root-cause inference,
anomaly detection, or telemetry ingestion of any kind. It is a static,
author-declared lookup table: `spanAttribute`/`spanValue` pairs a human
expects to see, and the Fault-Tree cause they expect that observation to
mean. Whether a Correlation Object's claim is actually accurate — whether
that span attribute really does correlate with that cause in production —
is an empirical question this specification does not answer and no
Conforming Parser/Compiler is expected to verify. Consuming this metadata
at runtime (to power an actual triage dashboard, for example) is left
entirely to tooling outside this specification's scope; no change to the
reference `etdl_core` runtime is required, or implied, by declaring
`x-diagnostics` in a document.

---

## 7. Compatibility

Silently ignoring `x-diagnostics` (core Section 11.1's baseline behavior)
leaves a document fully valid under core alone — correlation and
anomaly-rule metadata are additive, never a precondition for parsing,
validation, code generation, or runtime telemetry behavior.

A future `1.x` MINOR of this supplement may add fields to either object;
it MUST NOT change the meaning of `spanAttribute`/`spanValue` matching
(Section 4.1) without a MAJOR bump (core Section 5.1.3).
