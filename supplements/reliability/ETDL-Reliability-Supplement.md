# ETDL Reliability Supplement 1.0

**Supplement id:** `etdl.reliability`
**Version:** 1.0.0
**Profile:** extends ETDL Core 1.0.0 (IEC 61025:2006 + IEC 62502:2010 + AsyncAPI 3.0)
**Status:** Under Development — NOT YET RELEASED
**Normative dependency:** ETDL Core 1.0.0 (the supplement mechanism, Section 5.1.1 of the core specification)

> **This supplement extends ETDL Core for reliability engineering, probabilistic
> analysis, failure-mode modeling, reliability evidence, probability provenance,
> uncertainty, and related engineering analysis. It does not redefine the core
> ETDL language.**

---

## 1. Relationship to ETDL Core

ETDL Core defines a domain-neutral language: event trees, fault trees,
operations, basic events, probabilities, and compilation semantics. This
supplement adds a **reliability ontology** and a **probability data model** on
top of that core, using only the generic Supplement mechanism.

- **ETDL Core ≠ Reliability Supplement.** Core is domain-neutral; the
  supplement is domain-specific.
- **Reliability Supplement ≠ Reliability implementation.** The supplement
  defines the *representation and semantics* of reliability information. The
  statistical engines, databases, Bayesian/Monte-Carlo/ML/LLM tooling, and
  source-code analyzers that produce or consume that information are separate
  implementation concerns.

Nothing in this supplement requires ETDL itself to execute Bayesian inference,
MCMC, Monte Carlo, regression, machine learning, neural networks, LLMs,
external databases, or any particular programming language or statistical crate.

**In-document representation.** Reliability metadata is carried in an
`x-reliability` extension field (the `x-*` convention of core Section 11). This
applies in two places:

- **Inside an ordinary ETDL document** that models event/fault trees, the
  `x-reliability` block adds reliability metadata to that document.
- **In a standalone reliability-data document** (a `.etdl` file), the whole
  document is a data-only artifact: it declares `etdl.reliability`, carries no
  Event/Fault Trees, and holds its reliability content entirely in its
  `x-reliability` block (core §5.1 data-only documents).

Both forms use the same `x-reliability` structure and the same reliability
schema, so no second file format is needed — one language (`.etdl`) covers the
whole reliability concern.

---

## 2. Conformance

An implementation is **ETDL Reliability 1.0 Conformant** when it:

1. is **ETDL Supplement-Aware** (core Section 2.3);
2. recognizes `etdl.reliability` at version `1.0` (MAJOR `1`);
3. validates documents against this supplement's schema
   (`supplements/reliability/ETDL-Reliability-Schema.json`);
4. applies the reliability semantics defined here, including resolving external
   probability artifacts deterministically or failing clearly;
5. never silently substitutes a value for an unknown probability.

Conformance levels distinguish:

- **syntax support** — the supplement's fields parse;
- **semantic support** — the fields are interpreted per this document;
- **analysis support** — sensitivity/importance/cut-set analysis is produced;
- **runtime support** — runtime observations are collected.

An implementation may claim syntax/semantic support without analysis or runtime
support.

---

## 3. Reliability terminology

| Term | Definition |
|---|---|
| **Failure** | An event in which a system, service, or function does not perform its required function. |
| **Failure mode** | The manner in which a failure manifests (e.g. timeout, unavailable, capacity-exhaustion). |
| **Failure cause** | The condition or event that gives rise to a failure mode. |
| **Failure mechanism** | The physical, logical, or temporal process by which a cause produces a failure. |
| **Failure effect** | The consequence of a failure mode on the enclosing system, process, or business function. |
| **Condition** | A state or constraint under which an event or probability applies. |
| **Dependency** | A relationship in which one element depends on another. |
| **Common-cause failure (CCF)** | Multiple failures arising from a single shared cause; invalidates naive independence assumptions. |
| **Reliability** | The probability/frequency that a system performs its required function for a stated period/usage. |
| **Risk** | Reliability (probability/frequency of failure) combined with consequence/impact. Risk is NOT part of the core probability model. |
| **Reliability evidence** | Immutable recorded observations or measurements. |
| **Reliability model** | A versioned model that transforms evidence and assumptions into probability estimates. |
| **Provenance** | The recorded source, method, dataset, and model that produced an estimate. |

---

## 4. Reliability ontology

### 4.1 Core concepts

| Concept | Stability | Notes |
|---|---|---|
| Event | Stable | |
| Failure | Stable | |
| FailureMode | Stable | canonical identifiers (Section 7) |
| FailureCause | Stable | |
| FailureEffect | Stable | |
| FailureMechanism | Stable | |
| Condition | Stable | |
| Dependency | Stable | |
| Resource | Stable | |
| Barrier | Stable | maps to core Barrier |
| Mitigation | Stable | |
| Observation | Stable | immutable evidence |
| Evidence | Stable | |
| Assumption | Stable | |
| Prediction | Stable | |
| ProbabilityEstimate | Stable | Section 9 |
| ReliabilityModel | Versioned | |
| Failure taxonomy / categories | Versioned but extensible | |
| Failure mechanisms (domain vocabulary) | Versioned but extensible | |
| probability / failure rate / MTBF / MTTR / confidence / distribution parameters / observed frequency / predicted probability | Mutable knowledge | change with evidence |

### 4.2 Ontology stability rules

- **Stable** concepts are semantic identities and should rarely change.
- **Versioned but extensible** concepts may gain entries without changing
  existing semantics.
- **Mutable knowledge** (probabilities, rates, distributions) changes whenever
  new evidence arrives.
- **Immutable historical evidence** is never rewritten; new observations are
  appended.

A new observation does NOT change the ontology. The ontology changes only when
the conceptual classification changes (e.g. refining `failure.database.timeout`
into `failure.database.connection_timeout`, `query_timeout`, `lock_timeout`).
Such refinement is versioned and traceable.

---

## 5. Failure modes

### 5.1 FailureMode object

A `FailureMode` is identified by a stable canonical id (Section 7) and carries
metadata:

```yaml
failureModes:
  - id: failure.database.timeout
    label: "Database timeout"
    description: "A database operation exceeded its time budget."
    category: failure-category.database
    causes: [ failure.database.connection_timeout ]
    effects: [ failure.operation.degraded ]
```

| Field | Type | Requirement |
|---|---|---|
| `id` | FailureModeId | REQUIRED (Section 7) |
| `label` | string | OPTIONAL |
| `description` | string | OPTIONAL |
| `category` | string | OPTIONAL (ontology category id) |
| `causes` | Array[FailureModeId] | OPTIONAL |
| `effects` | Array[FailureModeId] | OPTIONAL |
| `x-*` | any | OPTIONAL |

### 5.2 Discovery candidates

A discovery tool (source-code analyzer, exception monitor) may propose
**candidate** failure modes. A candidate is NOT an authoritative FailureMode
until reviewed.

```yaml
candidateFailureModes:
  - id: candidate.exception.ETIMEDOUT
    proposed: failure.network.timeout
    status: candidate       # candidate | reviewed | accepted | rejected | merged | deprecated
    evidence: { source: "analyzer", detail: "socket.connect() ETIMEDOUT" }
```

Status values: `candidate`, `reviewed`, `accepted`, `rejected`, `merged`,
`deprecated`. Human/engineering review SHOULD be possible before acceptance.

### 5.3 Universal service/server failure ontology (optional, generic)

The following identifiers are **ontology concepts, not probability values**. A
common failure mode NEVER carries a universal probability.

```
failure.compute.memory_exhaustion
failure.compute.cpu_saturation
failure.network.timeout
failure.network.dns_failure
failure.network.unreachable
failure.storage.capacity_exhaustion
failure.storage.io_error
failure.database.unavailable
failure.database.connection_timeout
failure.database.query_timeout
failure.database.lock_timeout
failure.messaging.unavailable
failure.messaging.publish_failed
failure.configuration.invalid
failure.configuration.missing
failure.deployment.version_incompatibility
failure.runtime.process_crash
failure.security.unauthorized
failure.dependency.unavailable
failure.resource.exhaustion
```

Implementations MAY extend this vocabulary. Probability is never part of the
identifier (bad: `database.timeout.p=0.002`; good: `failure.database.timeout`).

---

## 6. Dependencies, common-cause failures, conditions

### 6.1 Dependency model

```yaml
dependencies:
  - id: payment_gateway_dep
    from: order-service
    to: payment-gateway
    kind: causal                 # dependency | causal | statistical | common-cause
```

`kind` distinguishes **dependency** (structural), **causal dependency**,
**statistical dependency**, and **common-cause relationship**. They are not
interchangeable.

### 6.2 Common-cause failure (CCF)

A CommonCauseFailure is a single cause producing multiple failures:

```yaml
commonCauseFailures:
  - id: ccf.region-outage
    cause: RegionOutage
    failures: [ failure.database.unavailable, failure.network.unreachable, failure.cache.unavailable, failure.messaging.unavailable ]
```

**Critical:** standard AND/OR probability formulas assume statistical
independence (core Section 5.16). A common-cause relationship **invalidates
naive independence**. This supplement does NOT force a specific common-cause
probability model in version 1.0; it requires that CCF be represented and that
tools which compute probabilities under CCF document the model used.

### 6.3 Conditions

Conditional probability is expressed as `P(event | condition)`:

```yaml
probabilityConditions:
  - id: cond.high-load
    expression: "message.payload.load >= 0.8"   # ECEL, optional
    label: "High load"
```

Conditions are identified; the statistical engine determines the actual
probability. The supplement defines how conditions are named and referenced, not
how they are estimated.

---

## 7. Canonical failure-mode identifiers

Identifiers use the stable namespace:

```
failure.<domain>.<name>
```

Examples:

```
failure.network.timeout
failure.network.unreachable
failure.database.unavailable
failure.database.connection_timeout
failure.database.query_timeout
failure.storage.capacity_exhaustion
failure.compute.memory_exhaustion
failure.configuration.invalid
failure.deployment.incompatible_version
```

Rules:

- probability is NEVER part of the identifier;
- refinement (e.g. timeout → connection/query/lock timeout) is versioned and
  traceable;
- a candidate discovery never silently becomes an authoritative identifier.

---

## 8. Probability states

Every probability value has a **state**, distinguished explicitly. `unknown` is
never represented as `probability: 0`.

| State | Meaning |
|---|---|
| `declared` | Stated in the document as an assertion (core `probability`). |
| `assumed` | Adopted as a working assumption without direct measurement. |
| `measured` | Derived from direct observation. |
| `estimated` | Produced by a statistical model from data. |
| `predicted` | Produced by a model for a future period/population. |
| `inferred` | Derived indirectly (e.g. from related quantities). |
| `imported` | Taken from an external artifact/provider. |
| `unknown` | Not known; MUST be represented explicitly, never as `0`. |

---

## 9. ProbabilityEstimate

A `ProbabilityEstimate` is the formal representation of a probability with
provenance, uncertainty, and conditions.

```yaml
probabilityEstimates:
  - id: est.gateway-timeout
    event: failure.database.timeout
    state: estimated
    value: 0.0021
    metric: probability
    population: production-orders
    unitTimeBasis: per-request
    conditions: [ cond.high-load ]
    source: statistical-model
    method: bayesian-beta-binomial
    uncertainty:
      kind: confidence-interval
      level: 0.95
      lower: 0.0015
      upper: 0.0030
    assumptions: [ "independence of requests" ]
    provenance:
      sourceType: measurement
      dataset: production-2026-Q2
      model: reliability-model-v3
      modelVersion: "3.1"
      generatedAt: "2026-08-16T12:00:00Z"
    status: reviewed
```

| Field | Type | Requirement |
|---|---|---|
| `id` | string | REQUIRED |
| `event` | FailureModeId | REQUIRED (what event) |
| `state` | ProbabilityState | REQUIRED (Section 8) |
| `value` | number | OPTIONAL* |
| `metric` | string | OPTIONAL (`probability`, `failure-rate`, `frequency`, `availability`, …) |
| `population` | string | OPTIONAL |
| `unitTimeBasis` | string | OPTIONAL (`per-request`, `per-operation`, `per-hour`, `per-mission`, `per-transaction`) |
| `conditions` | Array[ConditionId] | OPTIONAL |
| `source` | string | OPTIONAL |
| `method` | string | OPTIONAL |
| `uncertainty` | Uncertainty | OPTIONAL (Section 11) |
| `confidence` | number | OPTIONAL |
| `assumptions` | Array[string] | OPTIONAL |
| `provenance` | Provenance | OPTIONAL (Section 12) |
| `version` | string | OPTIONAL (knowledge version) |
| `timestamp` | string (RFC 3339) | OPTIONAL |
| `status` | string | OPTIONAL (engineering review status) |
| `x-*` | any | OPTIONAL |

\* An estimate may be an **uncertain** estimate without a single `value` (see
Section 11). A `state: unknown` estimate MUST omit `value` or mark it absent
explicitly; it MUST NOT be `0`.

### 9.1 Backward compatibility

The core syntax `probability: 0.01` (a Branch or Basic Event field) remains
valid and means `state: declared`. The richer `ProbabilityEstimate` is an
**extension**; it does not replace or alter the core field.

---

## 10. Failure rate vs probability vs frequency

The supplement distinguishes:

| Quantity | Meaning | Unit example |
|---|---|---|
| Failure rate λ | Instantaneous rate | per hour |
| Probability P(F) | Probability of failure in a mission/period | per mission |
| Frequency | Rate of occurrence | per request, per year |
| Availability | Uptime fraction | dimensionless |
| MTBF / MTTF / MTTR | Time-based measures | hours |

Units MUST be explicit and unambiguous (`per-request`, `per-operation`,
`per-hour`, `per-mission`, `per-transaction`). The core conversion
`P = 1 − e^(−λt)` (core Section 5.16) is unchanged.

---

## 11. Uncertainty and distributions

### 11.1 Uncertainty representation

An estimate may be:

- a **point estimate** (single `value`), or
- an **uncertain estimate** with an `uncertainty` object:

```yaml
uncertainty:
  kind: confidence-interval      # confidence-interval | credible-interval | distribution | lower-bound | upper-bound
  level: 0.95
  lower: 0.0015
  upper: 0.0030
```

### 11.2 Probability distributions

A generic distribution representation:

```yaml
distribution:
  type: beta
  parameters: { alpha: 3.2, beta: 150.0 }
  domain: [0, 1]
  unit: probability
```

Supported types (conceptually): Bernoulli, Binomial, Beta, Exponential, Gamma,
Normal, LogNormal, Weibull, Poisson. Not all must be implemented; unsupported
types are rejected or reported per implementation policy. The supplement defines
the representation, not the algorithm.

---

## 12. Provenance

Every non-literal reliability estimate SHOULD carry provenance:

| Field | Type |
|---|---|
| `sourceType` | `engineering-assumption` \| `measurement` \| `historical-data` \| `prediction` \| `statistical-model` \| `external-calculation` \| `vendor-data` \| `simulation` \| `inference` \| `unknown` |
| `dataset` | string |
| `model` | string |
| `modelVersion` | string |
| `generatedAt` | RFC 3339 |

Provenance records semantics, not implementation.

---

## 13. Reliability artifacts (external probability sources)

A **Reliability Artifact** is an external, non-executable **`.etdl` document**
that provides reliability information produced by an external calculation. It is
a data-only document (core §5.1): it declares `etdl.reliability`, carries no
Event/Fault Trees, and holds its reliability content in its `x-reliability`
block. Using `.etdl` for the artifact means a project manages one extension
language rather than a second file format.

```yaml
# payment-gateway.etdl
etdl: "1.0.0"
info:
  title: "Payment gateway reliability data"
  version: "1.0.0"
  domain: "PaymentsContext"
supplements:
  - id: etdl.reliability
    version: "1.0"
x-reliability:
  probabilityEstimates:
    - id: est.gateway-timeout
      event: failure.database.timeout
      state: estimated
      value: 0.0021
      provenance:
        sourceType: external-calculation
        model: vendor-reliability-model-v3
```

The artifact MAY contain estimates, distributions, confidence, provenance,
assumptions, model version, dataset information, failure modes, and runtime
observations. It MUST NOT be executable.

### 13.1 Declaring an external source

```yaml
supplements:
  - id: etdl.reliability
    version: "1.0"

x-reliability:
  sources:
    - id: payment_gateway
      type: external
      file: "./reliability/payment-gateway.etdl"
```

### 13.2 Referencing a source from a Basic Event

```yaml
basicEvents:
  GatewayTimeout:
    description: "Gateway did not respond in time."
    x-reliability:
      source: payment_gateway
      estimate: est.gateway-timeout
```

This integrates with, and does not duplicate, the core `probability`,
`failureRate`, `missionTime`, and `onFailureProbabilitySource` fields.

**V-503 interaction.** Core rule V-503 requires a Basic Event to supply exactly
one of `probability`/`failureRate`. When a Basic Event obtains its probability
from an external source via `x-reliability.source`, that is an explicit
extension of the probability semantics declared by this supplement (core
Section 5.1.5 permits supplements to add semantics to extension fields). Such an
event does not carry a core `probability`/`failureRate`; the external value is
resolved deterministically at build time (Section 14). If the external source
cannot be resolved, compilation MUST fail rather than inventing a value
(Section 15).

---

## 14. Deterministic compilation

Rich reliability information resolves deterministically to an ETDL Core
probability:

```
Rich reliability information
      ↓
resolved probability
      ↓
ETDL Core probability calculation
      ↓
deterministic compiled result
```

- Ordinary ETDL execution must NOT require a Bayesian engine, Monte Carlo
  engine, database, or AI service.
- If the compiler cannot resolve an external source, compilation MUST fail
  clearly rather than silently invent a value (Section 15).

---

## 15. Unknown probability policy

Handling of unknown probabilities is configurable, not fixed in syntax:

| Policy | Behavior |
|---|---|
| `error` | Compilation fails on any unresolved probability (safety-critical). |
| `warning` | Compilation proceeds with a W-level diagnostic (exploratory). |
| `allow` | Compilation proceeds; the value is treated as unresolved. |

Default: **`warning`**. Unknown MUST never become `0` silently. The exact
configuration mechanism follows the compiler/CLI architecture
(e.g. `[reliability] unknown_policy = "error"`).

---

## 16. Runtime observations

A `ReliabilityObservation` is immutable evidence. Observations may be carried
either in an ordinary document's `x-reliability` block or in a standalone
reliability-data `.etdl` artifact (Section 13):

```yaml
reliabilityObservations:
  - id: obs-2026-08-16-0001
    event: failure.database.timeout
    timestamp: "2026-08-16T12:00:00Z"
    service: order-service
    operation: process-payment
    environment: production
    outcome: failed
    conditions: [ cond.high-load ]
    durationMs: 2500
    x-metadata: { trace_id: "..." }
```

Observations are appended, never rewritten (Section 4.2). The statistical model
may change as observations accumulate.

---

## 17. Statistical updating lifecycle

The recommended lifecycle (Bayesian updating is ONE supported implementation,
not a requirement):

```
prior/assumption
      ↓
observations
      ↓
statistical estimation
      ↓
updated probability model
      ↓
engineering review
      ↓
new build
```

Section 33 makes this lifecycle normative for the specific case of comparing
a predicted probability against accumulated observations (**calibration**)
and states the non-mutation guarantee an implementation MUST uphold while
doing so.

---

## 18. FMEA / FMECA mapping (optional)

FMEA/FMECA is an **optional analysis profile**, not a requirement. It maps onto
supplement concepts:

| FMEA concept | Supplement concept |
|---|---|
| Failure Mode | FailureMode |
| Cause | FailureCause |
| Effect | FailureEffect |
| Detection | (analysis output) |
| Mitigation | Mitigation |
| Severity / Occurrence / Detectability | analysis annotations; NOT part of the core probability model |

No single industry scoring methodology is prescribed.

---

## 19. Cause and consequence traversal

The supplement defines relationships enabling:

- cause → event
- event → consequence

and tools to traverse "what can cause this?" and "what can this cause?". Core
tree semantics are unchanged.

---

## 20. Analysis outputs

Standard analysis result concepts (outputs, not necessarily ETDL executable
objects):

- `ProbabilityResult`
- `SensitivityResult`
- `ImportanceResult`
- `MinimalCutSetResult`
- `UncertaintyResult`
- `FailureDiscoveryResult`
- `CauseAnalysisResult`
- `ConsequenceAnalysisResult`

### 20.1 Sensitivity and importance

```yaml
sensitivity:
  topFailure: failure.payment
  contributors:
    - event: failure.database.timeout
      contribution: 0.71
    - event: failure.network.timeout
      contribution: 0.23
    - event: failure.storage.capacity_exhaustion
      contribution: 0.06
```

The supplement defines the semantics of **probability contribution, sensitivity,
importance, and risk contribution** but does not force one mathematical
implementation where multiple accepted methods exist.

---

## 21. Risk vs reliability

- **Reliability** = probability/frequency of failure.
- **Risk** = probability × consequence/impact.

Severity or financial impact is NEVER part of the core probability model. Risk
is a derived, analysis-layer concept.

---

## 22. Reliability build provenance

A compiled ETDL artifact SHOULD be traceable to:

```
Which ETDL?
Which supplement?
Which ontology?
Which reliability artifact?
Which probability model?
Which dataset?
Which model version?
Which compiler?
```

Represented as:

```yaml
reliabilityBuildProvenance:
  etdlVersion: "1.0.0"
  supplementVersion: "1.0"
  ontology: "core, postgresql"
  artifacts: [ "./reliability/payment-gateway.etdl" ]
  models: [ "reliability-model-v3" ]
  datasets: [ "production-2026-Q2" ]
  compiler: "etdl 0.2.0"
```

---

## 23. Built-in vs installable functionality

**Built-in (kept small):**

- basic probability resolution
- basic reliability schema validation
- core distributions if already needed
- artifact loading
- provenance validation

**Installable (optional providers):**

- Bayesian engines
- Monte Carlo
- advanced statistics
- database connectors
- source-code analyzers
- AI/LLM analyzers
- vendor reliability databases
- domain ontologies

---

## 24. Other core supplements

The core mechanism supports additional supplements without core changes, and
four now exist alongside this one: `etdl.safety`, `etdl.security`,
`etdl.diagnostics`, and `etdl.performance` (each its own companion
specification, core Appendix E.2). None of the four depend on this
supplement, and this supplement depends on none of them — each reads its own
`x-<domain>` extension field and, where relevant, the same core Barrier/Fault
Tree structures this supplement reads, independently (core Section 11.2). A
document MAY declare any combination of them alongside `etdl.reliability`.
The generic supplement mechanism, not any one supplement, is the
architectural feature.

---

## 25. Compatibility

- ETDL Core documents (no supplements) remain valid and unchanged.
- `probability: 0.01` remains valid and means `state: declared`.
- An implementation without this supplement recognizes `etdl.reliability`,
  reports it as unsupported per policy (W-407), and NEVER silently applies
  reliability semantics.
- This supplement does not change core event/fault tree semantics, probability
  formulas, branch ordering, or generated-code semantics.

---

## 26. Normative vs informative

**Normative** (this supplement): syntax, schema, validation, semantics,
compatibility, identifiers, versioning, probability states, provenance
structure, the reliability-data `.etdl` artifact format, unknown-probability
policy, observation structure, the predictive quantities and non-mutation
guarantees of Section 32, and the calibration discipline and non-mutation
guarantee of Section 33.

**Informative** (not normative): recommended Bayesian/Monte-Carlo methods,
FMEA scoring methodology, AI-assisted discovery, statistical methodology
recommendations, the specific statistical test used for calibration (Section
33.3 requires disclosure of the method, not one particular method).

---

## 27. Source-code analysis boundary

The supplement represents reliability information; a source-code analyzer is a
separate tool.

```
Source Code
    ↓
ETDL Discovery Tool
    ↓
Candidate Failure Modes
    ↓
Ontology Mapping
    ↓
Engineering Review
    ↓
Reliability Model
    ↓
ETDL
```

The supplement defines the objects exchanged between these stages, not the
analyzer.

---

## 28. AI/LLM boundary

AI/LLM may be used by implementations to classify exceptions, discover failure
modes, suggest causes/consequences, or map code concepts to ontology. The
specification does not depend on AI; everything is representable without it.

---

## 29. Mathematical correctness

- Assumptions (independence, mutual exclusivity, conditional dependence,
  stationarity, mission time, population, time basis) are documented with every
  estimate.
- `P(A AND B) = P(A)P(B)` is only valid under independence; the supplement never
  implies it without that assumption.
- A probability is never inferred from an exception count without specifying
  population, observation period, exposure, and sampling assumptions.

---

## 30. Reliability estimate identity

A probability value is not identified by its numeric value. Identity is based on:

- what event
- under what conditions
- for what population
- using what evidence/model

`failure.database.timeout` is the identity; `P = 0.0021` is current knowledge
about it.

---

## 31. Versioning of knowledge

| Item | Versioning |
|---|---|
| FailureMode ID | Stable |
| ReliabilityEstimate | Versioned |
| Observation | Immutable |
| Model | Versioned |
| Ontology | Versioned |
| Probability artifact | Versioned |

This enables reproducible engineering builds.

---

## 32. Predictive Reliability

Every quantity defined elsewhere in this supplement — `ProbabilityEstimate`
(Section 9), a Reliability Artifact's estimates (Section 13) — is a
**time-independent** statement: an inference about a quantity from available
evidence, with no future time horizon. This section defines the distinct
concept of a **prediction**: an expected future outcome over a specified
future time or exposure interval, computed from a time-to-failure model
whose parameters typically originate from a `ProbabilityEstimate`. A
prediction is never collapsed into, and never silently substitutes for, an
estimate or an observation (Section 3's existing distinction between
`Reliability` and `Reliability evidence` already establishes the estimate/
evidence boundary this section extends to cover time).

### 32.1 Predictive quantities

| Quantity | Definition | Notes |
|---|---|---|
| `S(t)` — survival | `P(T > t)` | |
| `R(t)` — reliability | `P(T > t)` | Identical formula to `S(t)`; named separately because, for a non-repairable system, "the system survives past `t`" and "the system remains reliable through `t`" are the same statement read in two domains. This supplement makes no claim about a repairable system beyond this section's stated scope. |
| `F(t)` — failure probability | `P(T <= t) = 1 - S(t)` | |
| `h(t)` — hazard | Instantaneous failure rate given survival to `t` | NOT a probability; not bounded to `[0,1]` in general (Section 10's failure-rate-vs-probability distinction applies to `h(t)` exactly as it does to a Basic Event's `failureRate`). |
| `H(t)` — cumulative hazard | `-ln(S(t))`, or an equivalent closed form where one exists | |
| `f(t)` — density | `h(t) * S(t)` | Defined for continuous models only. |

A prediction result MUST identify which of these six quantities it reports,
using a closed, named set — an implementation MUST NOT report a bare,
unlabeled numeric value for a predictive quantity, precisely so that a
hazard is never mistaken for a probability and a survival value is never
silently reinterpreted as something else.

### 32.2 Mission time

A prediction is always made **over an explicit mission time or exposure
interval** — an implementation MUST NOT assume a time horizon that was not
explicitly supplied. A mission time carries a numeric value and a unit; this
supplement, consistent with Section 10's treatment of `TimeBasis`, does not
define a checked unit-conversion system (core Section 5.1.7 notes `std.units`
is not yet part of ETDL Core) — matching a rate's declared unit to a mission
time's declared unit is the caller's responsibility, stated explicitly rather
than silently assumed compatible.

### 32.3 Time-to-failure models

This version of the supplement defines two model families:

**Constant-hazard (exponential) model** — one parameter, `lambda` (the
constant failure rate):

```
S(t) = exp(-lambda * t)
h(t) = lambda                    (constant for all t)
H(t) = lambda * t
```

**Weibull model** — two parameters, shape `k` and scale `lambda`:

```
S(t) = exp(-(t/lambda)^k)
h(t) = (k/lambda) * (t/lambda)^(k-1)
H(t) = (t/lambda)^k
mean  = lambda * Gamma(1 + 1/k)
```

The Weibull model's shape parameter `k` determines the hazard's direction
over time: `k < 1` is decreasing hazard (infant mortality/burn-in), `k = 1`
is constant hazard (mathematically equivalent to the exponential model with
the same `lambda`), `k > 1` is increasing hazard (wear-out/aging). An
implementation MUST NOT force a system exhibiting aging behavior into the
constant-hazard model on the theory that it is a simpler default — the two
models make different, non-interchangeable claims.

A model descriptor accompanying a prediction MUST identify: the model family,
its parameters, and its assumptions stated explicitly (for example,
"constant hazard," "non-repairable"). This supplement does not require any
particular set of model families beyond these two; a future version, or a
non-core supplement, MAY define additional families without requiring a
change to Sections 32.1–32.2's quantity/mission-time model.

### 32.4 Extrapolation

A model MAY declare the time range it is asserted valid over. A prediction
MUST record whether the requested mission time falls outside that declared
range (`extrapolated: true`); when no range was declared, an implementation
MUST NOT report `extrapolated: true` on the theory that an undeclared range
implies invalidity — the absence of a declared range is not itself evidence
either way, and an implementation MUST NOT invent one.

### 32.5 Provenance and the calibration boundary

A prediction's parameters typically originate from a `ProbabilityEstimate`
(Section 9) inside a Reliability Artifact (Section 13). Constructing a
prediction from an artifact is a **read-only** operation: it MUST NOT modify
the source artifact, MUST NOT invoke or duplicate the calibration process of
Section 33, and MUST record, as part of the prediction's provenance, which
artifact and which estimate within it the model's parameters were read from.
A prediction whose parameters were supplied directly (not read from an
artifact) carries no artificial artifact provenance — Section 12's provenance
requirement is "SHOULD," not fabricated where genuinely absent.

### 32.6 Determinism

Every quantity in Section 32.1, for the model families of Section 32.3, is
computed by a closed-form, deterministic formula. This supplement defines no
sampling, simulation, or randomized method for computing a predictive
quantity; Monte Carlo or Bayesian posterior-predictive simulation, if ever
introduced, is out of scope for this version and MUST be introduced, if at
all, as an explicitly labeled, opt-in alternative — never as a silent
replacement for the closed-form calculation.

### 32.7 Numerical stability

An implementation MUST return `S(0) = 1`, `H(0) = 0`, `F(0) = 0` exactly, by
direct evaluation of the boundary case rather than by evaluating the general
formula at a point where floating-point rounding could produce a value other
than the exact boundary. An implementation SHOULD compute `H(t)` from a
model's own closed form (Section 32.3) rather than as `-ln(S(t))`, since the
latter loses precision as `S(t)` approaches zero. `S(t)` MUST remain finite,
non-negative, and never `NaN` for any finite, non-negative `t`, including
values of `t` for which `S(t)` is smaller than the floating-point
representation can distinguish from zero.

---

## 33. Runtime Feedback and Calibration

This section makes Section 17's lifecycle diagram normative for
**calibration**: the specific act of comparing a `ProbabilityEstimate`'s
predicted value against accumulated `ReliabilityObservation`s (Section 16)
and reporting whether they are statistically consistent.

### 33.1 The discipline

```
observe  ->  analyze (calibrate)  ->  engineering review  ->  publish a NEW artifact  ->  rebuild
```

An implementation performing calibration MUST NOT, as a consequence of that
calibration, modify the `ProbabilityEstimate`, the Reliability Artifact
containing it, or any prior `ReliabilityObservation`, in place. A calibration
result that indicates the predicted and observed values have diverged MUST
be surfaced as a **report for engineering review** — never as an automatic
rewrite of the estimate. If a human reviewer decides the estimate should
change, that decision is realized by publishing a **new** Reliability
Artifact (a new `version`, Section 25's versioning discipline), never by
editing the artifact the calibration was run against. This is a `MUST`-level
requirement, not a recommendation: an implementation that silently updates a
probability in response to observed data, without this human-in-the-loop
publish step, does not conform to this supplement regardless of how
statistically sound its update rule is.

### 33.2 Calibration result

A calibration comparison MUST report, at minimum:

| Field | Meaning |
|---|---|
| the predicted value | Snapshotted from the artifact at comparison time — not a live reference that could change if the artifact is later edited (which, per Section 33.1, it MUST NOT be for this event/conditions combination without becoming a new artifact version). |
| the observed value | The aggregated observation's proportion (failures / exposure) under the same conditions. |
| the statistical method used | Stated explicitly, by name and version, so an engineer reviewing the result can judge whether it applies — an implementation MUST NOT present a bare p-value or status with no stated method. |
| a status | At minimum distinguishing: consistent with prediction; a potential deviation; a significant deviation ("drift"); insufficient data to judge; or the comparison is unsupported (for example, because the predicted and observed values describe different metrics, conditions, or time bases and comparing them would attribute a difference to the model that may only reflect a difference in circumstances). |

A comparison MUST be refused (reported as unsupported, not silently
performed anyway) when the predicted estimate's metric, declared conditions,
or time basis does not match the observation's — Section 10's metric
distinctions (probability vs. rate vs. frequency) are not interchangeable
inputs to a single statistical test.

### 33.3 Statistical method

This supplement does not mandate one statistical test; an implementation
MUST state which one it used (Section 33.2). Where the predicted quantity is
a probability (Section 9's `state`-bearing estimate, not a rate), an exact
binomial test comparing observed failures against exposure at the predicted
rate is an accepted method; a normal approximation MUST NOT be silently
substituted for an exact test without disclosure, since the approximation's
error is largest exactly in the low-failure-count regime reliability
engineering most often operates in.

### 33.4 Insufficient data

A comparison based on an exposure below a stated minimum MUST be reported as
insufficient data, not silently treated as a confident result merely because
a p-value happened to compute. The minimum exposure threshold is an
implementation configuration detail this supplement does not fix a value
for, but its presence and its value MUST be disclosed alongside any
comparison result it gated.

### 33.5 Relationship to Predictive Reliability

Section 32's predictions and this section's calibration compose without
either depending on the other's internal mechanism: a calibration result
(Section 33.2) MAY itself become the trigger for a human review that
publishes a new artifact (Section 33.1), whose revised estimate a Section
32 prediction MAY then be constructed from (Section 32.5) — but Section 32's
model construction is read-only with respect to any artifact, and this
section's calibration does not construct or consume a Section 32 prediction
directly. Each remains independently meaningful: a document may use Section
32's predictive quantities with no calibration ever having run, and
calibration (this section) is meaningful for any predicted probability
regardless of whether it is ever used to construct a Section 32 model.
