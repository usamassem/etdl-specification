# Event Tree Definition Language (ETDL) Specification

**Version:** 1.0.0
**Status:** Under Development — NOT YET RELEASED
**Supersedes:** 1.0.0-Standardized (Proof-of-Concept Draft)
**Profile:** IEC 61025:2006 (Fault Tree Analysis) + IEC 62502:2010 (Event Tree Analysis) + AsyncAPI 3.0 (Data & Interfaces)
**File Extension:** `.etdl`
**Media Type:** `application/vnd.etdl+yaml`

## Abstract

The Event Tree Definition Language (ETDL) is a declarative, design-time domain-specific language for reliability-aware, event-driven business processes. It models causal response sequences as **event trees**, in the sense of IEC 62502, and models the failure probability behind any point in those sequences as **fault trees**, in the sense of IEC 61025 — the same paired use the two techniques have in classical probabilistic risk assessment, where a fault tree quantifies a pivotal event that an event tree then sequences. Both tree types resolve their events against message and channel definitions owned by AsyncAPI 3.0, which ETDL delegates to entirely rather than redefining. An ETDL document compiles its event trees to service-local code, while its fault trees are resolved at build time into the probability values those event trees consume. This document defines the ETDL document format, its object model for both tree types, its condition-expression grammar, the validation rules a conforming parser must enforce, the code-generation contract a conforming compiler must satisfy, and the runtime library contract those generated artifacts depend on.

## Status of This Document

**This specification is under development and is NOT yet released.** It is not a
formal publication; normative text, schema, validation rules, and examples may
change before an official release. Implementers and adopters should treat this
document as an evolving working draft and must not rely on it as a released,
stable standard.

The document formalizes and supersedes the ETDL 1.0.0-Standardized
proof-of-concept. The prior draft was sufficient to communicate the core idea —
that event-driven flows can be modeled as causal trees with quantified branch
probabilities — but left several areas either undefined or only demonstrated by
example: the condition-expression language had no grammar, operation nodes had
no representable failure path, the reference syntax linking into AsyncAPI
documents was informal, and validation was described in three sentences rather
than enforceable rules. Appendix D itemizes every substantive change from the
draft, with rationale, so implementers of the earlier version can assess impact.

Everywhere this document uses **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, or **MAY**, that usage is normative (see Section 2.1).

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Conformance and Notation](#2-conformance-and-notation)
3. [Terminology](#3-terminology)
4. [File Format](#4-file-format)
5. [Document Structure](#5-document-structure)
   - 5.1 The ETDL Document Object
   - 5.1.1 Supplements Object
   - 5.1.2 Supplement identity
   - 5.1.3 Supplement versioning
   - 5.1.4 Supplement dependencies
   - 5.1.5 Extension boundaries
   - 5.2 Info Object
6. [The ETDL Condition Expression Language (ECEL)](#6-the-etdl-condition-expression-language-ecel)
7. [Semantic Validation Rules](#7-semantic-validation-rules)
8. [Compiler and Code Generation Semantics](#8-compiler-and-code-generation-semantics)
9. [Runtime Library Contract (etdl_core)](#9-runtime-library-contract-etdl_core)
10. [Versioning and Compatibility](#10-versioning-and-compatibility)
11. [Extensibility](#11-extensibility)
12. [Security Considerations](#12-security-considerations)
13. [Full Worked Example](#13-full-worked-example)
- [Appendix A — Consolidated Grammar Reference](#appendix-a--consolidated-grammar-reference)
- [Appendix B — Diagnostic Code Registry](#appendix-b--diagnostic-code-registry)
- [Appendix C — Reserved Words](#appendix-c--reserved-words)
- [Appendix D — Changelog: Draft to Formal Specification](#appendix-d--changelog-draft-to-formal-specification)
- [Appendix E — Companion Artifacts](#appendix-e--companion-artifacts)

---

## 1. Introduction

### 1.1 Purpose and Scope

ETDL is a design-time DSL with two complementary purposes:

1. **Event tree modeling (IEC 62502).** Expressing the causal structure of an event-driven workflow — what triggers it, what conditions branch it, what internal operations it performs, and what terminal outcomes it can reach — independently of the wire format of the messages involved.
2. **Fault tree modeling (IEC 61025).** Expressing *why* a branch or an operation fails, by deductively decomposing that failure into a logical combination of lower-level basic events, so that the probability driving an event tree's branch is a derived, auditable value rather than an asserted constant.

The two are complementary rather than redundant: an event tree is inductive (given this trigger, what sequences can follow?), while a fault tree is deductive (given this undesired outcome, what combinations of causes produce it?). Both operate over the same underlying vocabulary of AsyncAPI-defined messages and channels; an event tree and a fault tree in the same document routinely share imports. This specification defines:

- The syntax and object model of a `.etdl` document, covering both event trees and fault trees.
- The formal grammar of condition expressions used at event-tree decision points.
- The gate logic and probability-combination semantics used at fault-tree decision points.
- The resolution semantics for references into external AsyncAPI documents, and for internal references linking a fault tree's computed probability into an event tree.
- The validation rules a conforming parser **MUST** enforce before code generation.
- The minimum contract a conforming compiler and its generated code **MUST** satisfy.
- The runtime library contract (`etdl_core`) that generated code depends on for telemetry, SLA tracking, and chaos testing.

This specification does not define a wire protocol, a message broker, or a runtime orchestration engine. Those concerns belong to AsyncAPI and the underlying transport, respectively (Section 1.3).

### 1.2 Design Philosophy

ETDL follows a **"Smart Endpoints, Dumb Pipes"** philosophy: causal logic is compiled into the service that owns it, rather than interpreted centrally by an Enterprise Service Bus (ESB) or workflow engine. This is a deliberate trade-off. A central runtime engine would make cross-service causal chains easier to visualize at the cost of a shared point of failure, a shared deployment cadence, and a coupling vector between otherwise-independent services. ETDL instead treats the compiled artifact as the source of truth at runtime, and treats the `.etdl` document as a design-time contract whose enforcement happens at compile time (Section 7) and via runtime telemetry (Section 9), not via a live interpreter.

Fault trees fit this philosophy directly rather than straining it: a Fault Tree Object (Section 5.11) is resolved once, at compile time (Section 8.1), into the numeric probabilities an event tree consumes. There is no fault-tree evaluation service at runtime, and no live dependency between the generated code and the `.etdl` document that produced it — the same "no centralized runtime engine" guarantee the design philosophy already makes for event trees.

Two further consequences follow directly from this philosophy and are binding on implementations:

- A conforming ETDL compiler **MUST NOT** require a network call to an ETDL-specific service at the generated code's runtime in order to route a message through the tree, or to obtain a fault-tree-derived probability. Telemetry export (Section 9) is exempt from this restriction, since it is observational rather than decision-making.
- Core microservice logic **MUST** be modeled as a strict Directed Acyclic Graph (DAG) with no cross-tree jumps; boundary crossings happen only at the edges, through AsyncAPI channels (Section 1.3). The same DAG requirement applies to a Fault Tree Object's internal gate structure (Section 5.16).

### 1.3 Relationship to AsyncAPI, IEC 62502, and IEC 61025

ETDL is a profile that composes three standards rather than replacing any of them:

| Concern | Owned By | Notes |
|---|---|---|
| Message shape, payload schema, headers | AsyncAPI 3.0 | ETDL never redefines a schema; it only references one via an external reference (Section 5.3). |
| Channel addressing and transport binding | AsyncAPI 3.0 | ETDL's `channel` fields are references into an AsyncAPI document, not inline definitions. |
| Causal sequence: trigger → decision → operation → outcome | ETDL (IEC 62502 profile) | Event Tree Object (Section 5.5). This is ETDL's sole responsibility; AsyncAPI has no concept of sequence. |
| Failure decomposition: top event → gates → basic events | ETDL (IEC 61025 profile) | Fault Tree Object (Section 5.11). Produces the probability values event trees consume; AsyncAPI has no concept of causal decomposition. |
| Branch and operation probability | ETDL (IEC 61025 profile feeding IEC 62502 profile) | Either declared directly on a Branch/Operation, or derived from a linked Fault Tree Object's computed Top Event probability (Section 5.15). |
| Reliability and SLA tracking at runtime | ETDL (`etdl_core`) | Compares observed frequencies against the declared-or-derived probability (Section 9.3). |
| Serialization, transport binding, protocol semantics | AsyncAPI 3.0 | Out of scope for ETDL entirely. |

A conforming ETDL document **MUST** import at least one AsyncAPI document (Section 5.3); every `initiatingEvent`, `emits`, `consequence`, and — where present — Fault Tree `message` annotation resolves through an import. A document **MAY** contain zero, one, or several Fault Tree Objects; unlike AsyncAPI imports, fault trees are an optional refinement. A document with event trees but no fault trees is fully conforming and simply declares its branch probabilities directly, exactly as in ETDL 1.0.0-Standardized.

### 1.4 Non-Goals

To prevent scope creep in implementations and to prevent ETDL's probability fields from being mistaken for something they are not, this specification explicitly excludes the following from ETDL's scope:

- **Not a runtime orchestration engine.** ETDL compiles to service-local code (Section 1.2); it does not execute trees itself.
- **Not a distributed transaction coordinator.** ETDL does not provide saga compensation, two-phase commit, or exactly-once delivery. Those guarantees, if needed, are the responsibility of the underlying transport and the handlers referenced by `operation` nodes.
- **Not a general-purpose programming language.** The condition-expression language (Section 6) is intentionally non-Turing-complete.
- **Not a certified safety-case artifact.** `probability` values (Section 5.8.1) are operational engineering estimates consumed by the runtime for Service Level Objective (SLO) tracking and anomaly detection (Section 9.3). IEC 62502 is used in its source domain for certified Probabilistic Risk Assessment (PRA) in safety-critical systems; ETDL borrows its causal-modeling vocabulary but **MUST NOT** be represented as, or relied upon as, a certified PRA. A team with an actual safety-case obligation needs a dedicated PRA tool and process in addition to, not instead of, ETDL.
- **Not a Common-Cause-Failure (CCF) analysis tool.** The gate probability formulas in Section 5.16 assume statistical independence among a Gate's inputs. ETDL 1.0.0 has no dedicated construct for a shared dependency (a shared library, a shared power feed, a shared upstream provider) between nominally-independent Basic Events. An author with such a dependency **MUST** model it explicitly — typically as one shared Basic Event feeding multiple gates, or as an intermediate Gate representing the shared component — rather than relying on ETDL to detect it automatically.
- **Not a replacement for AsyncAPI.** ETDL has no mechanism for defining a message schema from scratch; see Section 1.3.

---

## 2. Conformance and Notation

### 2.1 Requirements Language

This specification uses the capitalized keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** as normative requirement levels, following the widely-adopted IETF convention (RFC 2119, as clarified by RFC 8174):

- **MUST** / **MUST NOT** — an absolute requirement or prohibition. A conforming implementation cannot deviate from these and still claim conformance.
- **SHOULD** / **SHOULD NOT** — a strong recommendation. Deviating is permitted only for a deliberate, documented reason, with the implications understood.
- **MAY** — a genuinely optional capability; implementations are free to provide it or omit it without affecting conformance.

### 2.2 Document Notation

Every object type defined in this specification (Section 5) is documented as a **Fixed Fields** table with four columns: field name, type, requirement (`REQUIRED`, `OPTIONAL`, or `DEPRECATED`), and description. Field names are exact and case-sensitive. A type written as `Map[K, V]` denotes a YAML/JSON mapping whose keys have type `K` and whose values have type `V`. A type written as `X Object` refers to another Fixed Fields table defined elsewhere in Section 5.

### 2.3 Conformance Targets

This specification defines conformance independently for four roles, since an implementation may fill only one of them (for example, an editor plugin that validates documents but does not compile them):

| Target | Conformance Requirement |
|---|---|
| **Conforming Document** | Validates against the JSON Schema in Appendix E, and satisfies every `MUST`-level rule in Section 7. |
| **Conforming Parser** | Accepts every Conforming Document, rejects every document violating a Section 7 `MUST`-level rule with the corresponding diagnostic code (Appendix B), and resolves references per Section 5.3. |
| **Conforming Compiler** | Is a Conforming Parser, and additionally satisfies the Target Language Interface Contract (Section 8.2) and Generated Code Requirements (Section 8.3) for at least one target language. |
| **Conforming Runtime** | Implements the `etdl_core` contract of Section 9 for at least one target language, including OpenTelemetry propagation (9.2), SLA anomaly alerting (9.3), and Chaos Injection Mode (9.4) with the production safeguard of Section 12. |

Supplements introduce two additional conformance levels (Section 5.1.1):

| Target | Conformance Requirement |
|---|---|
| **ETDL Supplement-Aware** | A Conforming Parser/Compiler that recognizes supplement declarations, enforces supplement identity/versioning (Section 5.1.2–5.1.3), applies the unknown-supplement policy (Section 5.1.1), and reports (W-407/E-108) that declared supplement semantics were not applied. |
| **ETDL `<domain>` Supplement-Conformant** | A Supplement-Aware implementation that additionally implements the full semantics and schema of the named supplement (e.g. **ETDL Reliability 1.0 Conformant**). |

A single package MAY implement more than one target; the Rust and TypeScript reference implementations in Section 8.4–8.5 each combine Compiler and Runtime conformance.

---

## 3. Terminology

| Term | Origin | Definition |
|---|---|---|
| **initiatingEvent** | IEC 62502 | The trigger event that originates a causal event tree sequence. Exactly one per Event Tree Object. |
| **barrier** | IEC 62502 | A conditional check or system-state condition that determines which of several branch paths a sequence takes. |
| **branch** | IEC 62502 | One labeled, conditioned path out of a barrier, carrying a probability (declared or fault-tree-derived) and a successor node. |
| **probability** | IEC 62502 | The baseline probability of a given branch being taken, either declared directly or derived from a linked Fault Tree's Top Event (Section 5.15); see `probabilityOfSuccess` / `probabilityOfFailure` for the deprecated binary-outcome declared form. |
| **operation** | AsyncAPI (adapted) | An internal execution task, command handler, or state mutation performed by the service. |
| **consequence** | IEC 62502 | The terminal node of a branch path; either emits a message via AsyncAPI or terminates silently. |
| **channel** | AsyncAPI | The transmission address or topic where a message is published, referenced rather than redefined. |
| **sequence / path** | IEC 62502 | The ordered walk of nodes from `initiatingEvent` to a terminal consequence along one chain of branch choices. |
| **node** | ETDL | Any entry in an Event Tree Object's `nodes` map: a Barrier, Operation, or Consequence. |
| **domain** | ETDL | The bounded context a `.etdl` document belongs to, declared in the Info Object. |
| **supplement / extension** | ETDL | A formally identified, versioned domain extension to ETDL Core (Section 5.1.1). Supplements add domain-specific semantics and metadata; they do not alter core ETDL semantics. |
| **ETDL Core** | ETDL | The domain-neutral language defined by Sections 4–13 of this specification, independent of any supplement. |
| **import alias** | ETDL | A short name bound to an external AsyncAPI document in `asyncapi_imports`, used as the prefix of an external reference. |
| **external reference** | ETDL | A string of the form `<import-alias>#<json-pointer>` resolving into an imported AsyncAPI document (Section 5.3). |
| **ECEL** | ETDL | The ETDL Condition Expression Language used in `condition` fields (Section 6). |
| **DAG** | Graph theory | Directed Acyclic Graph; the required topology of every Event Tree Object's `nodes`, and of every Fault Tree Object's combined `gates`/`basicEvents` graph (Section 7). |
| **topEvent** | IEC 61025 | The undesired event whose causes a fault tree analyzes; the root of the Fault Tree Object (Section 5.12). |
| **gate** | IEC 61025 (adapted) | A logical combination — AND, OR, NOT, XOR, VOTING, INHIBIT, or PRIORITY_AND — of lower gates and/or basic events (Section 5.13). ETDL's Gate Object conflates IEC 61025's separate "gate" (logic symbol) and "intermediate event" (named result) into one addressable node, following common fault-tree tooling convention. |
| **basicEvent** | IEC 61025 | A leaf cause in a fault tree, carrying its own declared probability or failure rate; not further decomposed (Section 5.14). |
| **undeveloped event** | IEC 61025 | A Basic Event explicitly marked `undeveloped: true` — not further analyzed, typically for lack of data or insignificance, rather than because it is truly atomic. |
| **cut set / minimal cut set** | IEC 61025 | A combination of Basic Events whose simultaneous occurrence causes the Top Event; *minimal* means no proper subset of it is itself a cut set (Section 8.6). |
| **fault tree** | IEC 61025 | The Fault Tree Object: a Top Event, a set of Gates, and a set of Basic Events, forming a DAG rooted at the Top Event (Section 5.11). |
| **internal reference** | ETDL | A same-document JSON Pointer of the form `#/faultTrees/<id>/topEvent`, linking a Fault Tree's computed probability into an Event Tree's Branch or Operation node (Section 5.15) — distinguished from an external reference by having no import alias before the `#`. |

---

## 4. File Format

### 4.1 Syntax

A `.etdl` document **MUST** be valid YAML 1.2, which includes any document that is also valid JSON, since JSON is a syntactic subset of YAML 1.2. Implementations **MUST** accept both forms. Examples in this specification use YAML for readability.

### 4.2 File Extension and Media Type

Files **SHOULD** use the extension `.etdl`. When a media type is needed (for example, in an HTTP `Content-Type` header when serving a document from a schema registry), implementations **SHOULD** use `application/vnd.etdl+yaml`, or `application/vnd.etdl+json` when the document is serialized as JSON.

### 4.3 Character Encoding

Documents **MUST** be encoded as UTF-8 without a byte-order mark (BOM). Parsers **MUST** reject a document containing a BOM with diagnostic code E-102 (Appendix B), since a silently-stripped BOM has historically been a source of hash and diff instability in version-controlled specification files.


---

## 5. Document Structure

This section defines every object type a `.etdl` document is built from. Sections 5.1–5.10 cover the root document and the Event Tree object model (IEC 62502 profile); Sections 5.11–5.16 cover the Fault Tree object model (IEC 61025 profile) and how it links into an Event Tree. Every object is documented as a Fixed Fields table per the notation of Section 2.2.

### 5.1 The ETDL Document Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `etdl` | string | REQUIRED | Version of the ETDL specification this document conforms to, e.g. `"1.0.0"`. |
| `info` | Info Object | REQUIRED | Document metadata (Section 5.2). |
| `asyncapi_imports` | Map[string, string] | REQUIRED | Import aliases bound to AsyncAPI document locations (Section 5.3). |
| `supplements` | Array[Supplement Object] | OPTIONAL | Declared supplements/extensions (Section 5.1.1). |
| `components` | Components Object | OPTIONAL | Reusable Barrier/Operation/Gate/Basic Event templates (Section 5.4). |
| `eventTrees` | Map[string, Event Tree Object] | REQUIRED* | Named event trees (Section 5.5). |
| `eventTree` | Event Tree Object | DEPRECATED | Legacy singular form from 1.0.0-Standardized (Section 10.2). |
| `faultTrees` | Map[string, Fault Tree Object] | OPTIONAL | Named fault trees (Section 5.11), whose computed probabilities MAY be linked into `eventTrees` (Section 5.15). |
| `x-*` | any | OPTIONAL | Specification extensions (Section 11). |

\* Exactly one of `eventTrees` or the deprecated `eventTree` MUST be present, **unless** the document declares a supplement that defines data-only semantics (Section 5.1.1). A supplement MAY define documents that carry no Event Trees or Fault Trees — for example, the ETDL Reliability Supplement 1.0 defines standalone reliability-data documents whose content lives entirely in `x-` extension fields (Section 5.1.5). Such a document MUST declare that supplement and MUST NOT carry `eventTrees`/`eventTree`/`faultTrees`. A data-only document is also exempt from the `asyncapi_imports` requirement (Section 5.3), since it references no messages or channels.

#### 5.1.1 Supplements Object

A **supplement** (also called an **extension**) is a formally identified,
versioned domain extension to ETDL Core. The core defines the mechanism;
supplements define domain semantics. This specification ships with the
**ETDL Reliability Supplement 1.0**; future supplements (safety, security,
diagnostics, performance, …) may be defined independently without modifying the
core.

A document MAY declare zero or more supplements. Declaring none is fully valid
and is the ordinary case. Declaring a supplement does not change any core ETDL
semantics; it opts into additional, domain-specific semantics defined by that
supplement.

**Declaration.** `supplements` is an array of Supplement Objects at the document
root:

```yaml
supplements:
  - id: etdl.reliability
    version: "1.0"
```

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Supplement identifier in the namespace `etdl.<domain>` (Section 5.1.2). |
| `version` | string (SemVer) | REQUIRED | Supplement version (Section 5.1.3). |
| `required` | boolean | OPTIONAL, default `false` | When `true`, the document's semantics cannot be fully applied without this supplement. |
| `metadata` | Map[string, any] | OPTIONAL | Free-form supplement metadata. `x-`-prefixed keys only, per Section 11. |

**Document-wide scope.** Supplement declarations apply to the whole document.
They are declared once at the root; individual objects do not re-declare them.

**Multiple supplements.** A document MAY declare several supplements. They
coexist independently unless one declares a dependency on another (Section
5.1.4). Order in the array is not significant.

**Unsupported supplements.** An implementation that does not implement a
declared supplement MUST NOT silently pretend the supplement's semantics were
applied. Behavior is configurable (Section 7, W-level guidance) and defaults to
a **warning**; a supplement declared `required: true` that the implementation
does not support MUST be reported as an **error** unless the implementation is
configured otherwise.

#### 5.1.2 Supplement identity

Supplement identifiers use the stable namespace `etdl.<domain>`, where `<domain>`
is a lowercase ASCII identifier:

```
supplement-id = "etdl" "." 1*(ALPHA / DIGIT / "-")
```

Reserved identifiers in this specification:

| id | Supplement |
|---|---|
| `etdl.reliability` | ETDL Reliability Supplement 1.0 |
| `etdl.safety` | (reserved for a future supplement) |
| `etdl.security` | (reserved for a future supplement) |
| `etdl.diagnostics` | (reserved for a future supplement) |
| `etdl.performance` | (reserved for a future supplement) |

An identifier not beginning with `etdl.` is not a valid supplement id.

#### 5.1.3 Supplement versioning

Supplement `version` follows Semantic Versioning and MAY be written as
`MAJOR.MINOR` or `MAJOR.MINOR.PATCH` (an omitted PATCH is treated as `0`):

- **MAJOR** — incompatible changes to the supplement's semantics or schema.
- **MINOR** — backward-compatible additions.
- **PATCH** — specification-error corrections without semantic change.

An implementation supporting supplement `etdl.<domain>` at MAJOR `N` MUST accept
documents declaring that supplement at MAJOR `N` (any MINOR/PATCH) and SHOULD
reject a declared future MAJOR it does not implement, mirroring the core
document version rule (Section 10.1).

#### 5.1.4 Supplement dependencies

A supplement MAY declare dependencies on other supplements:

```yaml
supplements:
  - id: etdl.safety-reliability
    version: "1.0"
    metadata:
      x-requires:
        - id: etdl.reliability
          range: ">=1.0 <2.0"
```

Dependency resolution is an **implementation** concern, not core compiler
business logic. The core requirement is only: an implementation that applies a
supplement with unsatisfied dependencies MUST report the conflict (Section 7)
and MUST NOT silently apply the supplement's semantics.

#### 5.1.5 Extension boundaries

A supplement MAY:

- add domain-specific metadata, entities, analysis annotations, references,
  external artifacts, validation rules, ontologies, analysis outputs, and
  semantics for extension fields;
- define **data-only documents** that carry no Event Trees or Fault Trees
  (Section 5.1), provided the document declares that supplement and its content
  lives in `x-` extension fields.

A supplement MUST NOT silently:

- change the meaning of an existing ETDL Core field or object;
- change event-tree, fault-tree, probability, or operation semantics;
- change branch ordering;
- change generated-code semantics without an explicit, declared extension;
- invalidate an otherwise valid ETDL Core document.

A supplement that requires fundamentally different tree semantics MUST define a
new supplement/version rather than overriding the core.

### 5.2 Info Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `title` | string | REQUIRED | Human-readable name of this document. |
| `version` | string | REQUIRED | The document author's own semantic version for this domain's tree(s) — independent of the `etdl` spec-version field (Section 10.1). |
| `domain` | string, matching `^[A-Za-z][A-Za-z0-9]*$` | REQUIRED | The bounded context this document belongs to. |
| `description` | string | OPTIONAL | |

### 5.3 Imports and Reference Resolution

#### 5.3.1 External References

An **External Reference** points into an AsyncAPI document declared in `asyncapi_imports`:

```
external-ref  = import-alias "#" json-pointer
import-alias  = ALPHA *( ALPHA / DIGIT / "_" )
json-pointer  = *( "/" ref-token )                 ; RFC 6901
ref-token     = *( unescaped / escaped )
unescaped     = %x00-2E / %x30-7D / %x7F-10FFFF     ; any char except "/" and "~"
escaped       = "~" ( "0" / "1" )                   ; "~0" => "~", "~1" => "/"
```

For example, `orders_api#/components/messages/OrderPlaced` resolves the JSON Pointer `/components/messages/OrderPlaced` against the AsyncAPI document at `asyncapi_imports.orders_api`.

#### 5.3.2 Internal References

An **Internal Reference** is a same-document JSON Pointer, distinguished from an External Reference by having no alias before the `#`:

```
internal-ref  = "#" json-pointer                   ; json-pointer as above
```

ETDL 1.0.0 defines a resolved meaning for the following Internal Reference shapes, and only where an Internal Reference is explicitly expected by the field that carries it:

| Shape | Resolves to | Used by |
|---|---|---|
| `#/faultTrees/<id>/topEvent` | the Top Event Object of the named Fault Tree | `probabilitySource`, `onFailureProbabilitySource` (Section 5.15) |
| `#/faultTrees/<id>/gates/<gate-id>` | the named Gate in the Fault Tree | Transfer `target` (Section 5.11.1), component `$ref` (Section 5.4) |
| `#/faultTrees/<id>/basicEvents/<event-id>` | the named Basic Event in the Fault Tree | Transfer `target` (Section 5.11.1), component `$ref` (Section 5.4) |
| `#/components/<kind>/<id>` | the named Component template | component `$ref` (Section 5.4) |

A shape is only meaningful where the carrying field explicitly expects an Internal Reference; a Transfer `target` that resolves to a gate or basic event within a Fault Tree is valid (Section 5.11.1). Any other syntactically valid Internal Reference has no defined meaning in this version of the specification, reserved for future extension. An Internal Reference that does not match one of the shapes above in a field that expects one is a validation error (Section 7, E-105).

#### 5.3.3 Resolution Algorithm

A conforming parser resolves a reference string as follows:

1. If the string starts with `#/`, it is an Internal Reference (5.3.2). Resolve the pointer against the current document. If it does not match one of the resolved Internal Reference shapes defined in 5.3.2 for a field that expects an Internal Reference, this is a validation error (Section 7).
2. Otherwise, if the string matches `<alias>#/<pointer>` where `<alias>` is a key declared in `asyncapi_imports`, it is an External Reference (5.3.1). Resolve `<pointer>` against the AsyncAPI document at `asyncapi_imports[<alias>]`.
3. A string matching neither form, or an External Reference whose `<pointer>` does not resolve within the target document, is a resolution error (Section 7).

### 5.4 Components Object

Reusable definitions, referenced from elsewhere in the document via `$ref: "#/components/<kind>/<id>"`:

| Field | Type | Requirement | Description |
|---|---|---|---|
| `barriers` | Map[string, Barrier Node Object] | OPTIONAL | Reusable barrier templates. |
| `operations` | Map[string, Operation Node Object] | OPTIONAL | Reusable operation templates. |
| `gates` | Map[string, Gate Object] | OPTIONAL | Reusable gate templates — for example, a shared "regional power loss" sub-tree referenced from several Fault Trees. |
| `basicEvents` | Map[string, Basic Event Object] | OPTIONAL | Reusable basic-event definitions, typically sourced from a component failure-rate database (OREDA- or NPRD-style) shared across many Fault Trees. |

A component reference resolves like an Internal Reference (5.3.2) but is not restricted to the `topEvent` shape; it is substituted in place, as-is, wherever it is referenced.

### 5.5 Event Tree Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `initiatingEvent` | Initiating Event Object | REQUIRED | Section 5.6. |
| `nodes` | Map[string, Node Object] | REQUIRED | Section 5.7. |
| `description` | string | OPTIONAL | |

### 5.6 Initiating Event Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Naming label; also the source for generated entry-point identifiers (Section 8.3). |
| `message` | External Reference | REQUIRED | The AsyncAPI message that starts this tree. |
| `next` | Node ID Reference | REQUIRED | |

### 5.7 Node Object

Every entry in an Event Tree's `nodes` map is one of three concrete types, discriminated by its `type` field: **Barrier** (5.8), **Operation** (5.9), or **Consequence** (5.10). A Node's own map key is its Node ID, referenced from `next`, `onFailure`, and `initiatingEvent.next` fields elsewhere in the same tree. Node IDs MUST be unique within a tree (Section 7).

### 5.8 Barrier Node Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `type` | const `"barrier"` | REQUIRED | |
| `branches` | Array[Branch Object], `≥ 2` items | REQUIRED | |
| `description` | string | OPTIONAL | |

#### 5.8.1 Branch Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `outcome` | string | REQUIRED | Free-form label; `SUCCESS` and `FAILURE` are conventional for binary barriers (Appendix C). |
| `condition` | ECEL expression, or the literal `"default"` | REQUIRED | The guard evaluated against the triggering message (Section 6). |
| `probability` | number, `0 ≤ x ≤ 1` | See note | Declared baseline probability of this branch. |
| `probabilityOfSuccess` | number, `0 ≤ x ≤ 1` | DEPRECATED | Legacy alias for `probability` when `outcome` is `SUCCESS` (Section 10.2). |
| `probabilityOfFailure` | number, `0 ≤ x ≤ 1` | DEPRECATED | Legacy alias for `probability` when `outcome` is `FAILURE` (Section 10.2). |
| `probabilitySource` | Internal Reference | See note | A `#/faultTrees/<id>/topEvent` reference (Section 5.15) deriving this branch's probability from a Fault Tree, instead of or alongside a declared value. |
| `next` | Node ID Reference | REQUIRED | Successor node if this branch is taken. |

**Note:** a Branch MUST supply at least one of `probability`, `probabilityOfSuccess`, `probabilityOfFailure`, or `probabilitySource`. When `probabilitySource` is present it is authoritative (Section 5.15); a declared value present alongside it is a non-binding cached annotation.

**Validation notes** (formalized as numbered rules in Section 7): a Barrier's `branches` MUST contain at most one entry whose `condition` is `"default"`, and if present it MUST be evaluated last. The sum of all branches' effective probabilities (declared or fault-tree-derived) MUST equal `1.0` within a tolerance of `±0.0001`.

### 5.9 Operation Node Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `type` | const `"operation"` | REQUIRED | |
| `action` | enum: `execute` | REQUIRED | Reserved for future non-`execute` action kinds; ETDL 1.0.0 defines only `execute`. |
| `handler` | string | REQUIRED | Identifier resolved by the target-language runtime to a callable (Section 8.2). |
| `emits` | External Reference | OPTIONAL | The AsyncAPI message this operation produces on success, if any. |
| `next` | Node ID Reference | REQUIRED | Successor node on success. |
| `onFailure` | Node ID Reference | OPTIONAL | Successor node if `handler` fails. See the default propagation rule below if omitted. |
| `onFailureProbabilitySource` | Internal Reference | OPTIONAL | A `#/faultTrees/<id>/topEvent` reference (Section 5.15) supplying this operation's baseline failure probability for SLA tracking (Section 9.3), independently of whether `onFailure` is set. |
| `retryPolicy` | Retry Policy Object | OPTIONAL | Section 5.9.1. |
| `timeoutMs` | integer, `> 0` | OPTIONAL | Per-attempt timeout. |
| `description` | string | OPTIONAL | |

**Default failure propagation:** if `onFailure` is omitted, a conforming compiler MUST generate code that propagates the handler's error to the caller as a typed error (preserving 1.0.0-Standardized behavior), and SHOULD emit an advisory diagnostic (Section 7) noting the unmodeled failure path. This is not merely stylistic: an operation with no `onFailure` cannot itself appear as a cause in any Fault Tree's minimal cut sets (Section 8.6), since there is no node representing its failure as a distinct, sequenced outcome.

#### 5.9.1 Retry Policy Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `maxAttempts` | integer, `≥ 1` | OPTIONAL, default `1` | |
| `backoffMs` | integer, `≥ 0` | OPTIONAL, default `0` | |
| `backoffStrategy` | enum: `fixed`, `exponential` | OPTIONAL, default `fixed` | |

### 5.10 Consequence Node Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `type` | const `"consequence"` | REQUIRED | |
| `operation` | enum: `send`, `terminate` | REQUIRED | `terminate` marks a silent terminal outcome with no message emission. |
| `channel` | External Reference | REQUIRED if `operation` is `send` | |
| `message` | External Reference | REQUIRED if `operation` is `send` | |
| `description` | string | OPTIONAL | |

---

### 5.11 Fault Tree Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `topEvent` | Top Event Object | REQUIRED | The undesired event this tree analyzes (Section 5.12). |
| `gates` | Map[string, Gate Object] | OPTIONAL | Named gates (Section 5.13). Omitted only for a trivial single-cause tree. |
| `basicEvents` | Map[string, Basic Event Object] | REQUIRED | Named leaf causes (Section 5.14). |
| `transfers` | Map[string, Transfer Object] | OPTIONAL | Named transfer points referencing another Fault Tree (or subtree) for cross-tree reuse (Section 5.11.1). |
| `description` | string | OPTIONAL | |

Unlike an Event Tree's unified `nodes` map (Section 5.5), a Fault Tree keeps Gates and Basic Events in separate maps, since the two share no fields and are never interchangeable at a given position in the tree. Their keys, however, MUST be drawn from one shared namespace: a Gate and a Basic Event within the same Fault Tree Object MUST NOT share an ID, so that any Fault Tree Node Reference (5.12) resolves unambiguously.

#### 5.11.1 Transfer Object

A transfer names a cross-tree reference (the IEC 61025 transfer-in/transfer-out symbol). Transfers are for documentation, navigation, and visualization; they do not inline the target tree into the source tree's probability computation (an explicit `rootCause` link is required for that).

| Field | Type | Requirement | Description |
|---|---|---|---|
| `target` | Internal Reference | REQUIRED | An Internal Reference of the form `#/faultTrees/<id>/...` naming the transferred Fault Tree (or a node within it). MUST resolve to an existing Fault Tree in the document. |
| `label` | string | OPTIONAL | A human-readable label (e.g. "See Subsystem A"). |

**Resolution:** a Transfer's `target` resolves like an Internal Reference (5.3.2). A `target` that does not begin with `#/faultTrees/`, or that references a Fault Tree absent from the document, is a validation error (Section 7).

### 5.12 Top Event Object

Both a Gate's `inputs` (5.13) and a Top Event's `rootCause` are **Fault Tree Node References**: a bare string that MUST match a key in the enclosing Fault Tree Object's `gates` map or `basicEvents` map. Resolution checks `gates` first, then `basicEvents`.

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Naming label; used in generated documentation and identifiers (Section 8.3). |
| `description` | string | REQUIRED | A clear statement of the undesired event, per IEC 61025 convention. Required — an undocumented top event defeats the purpose of the analysis. |
| `message` | External Reference | OPTIONAL | The AsyncAPI message that observably corresponds to this undesired event, if any. |
| `rootCause` | Fault Tree Node Reference | REQUIRED | The gate or basic event whose probability equals the Top Event's probability. |

### 5.13 Gate Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `type` | enum: `AND`, `OR`, `NOT`, `XOR`, `VOTING`, `INHIBIT`, `PRIORITY_AND` | REQUIRED | Gate logic (Section 5.16 for probability semantics). |
| `inputs` | Array[Fault Tree Node Reference] | REQUIRED | See arity rules below. |
| `k` | integer, `≥ 1` | REQUIRED if `type` is `VOTING` | Minimum number of `inputs` that must occur for the gate to occur. |
| `inhibitCondition` | string | REQUIRED if `type` is `INHIBIT` | The conditioning event that must hold for the primary input to propagate. |
| `description` | string | OPTIONAL | |

**Arity rules** (enforced in Section 7):

- `AND`, `OR`: `inputs` MUST contain at least 2 entries.
- `NOT`: `inputs` MUST contain exactly 1 entry.
- `XOR`: `inputs` MUST contain exactly 2 entries. ETDL 1.0.0 defines XOR only for the binary case; an N-ary "exactly one of N" gate is not directly expressible and is a known limitation, not an oversight.
- `VOTING`: `inputs` MUST contain at least 2 entries (`n`), and `k` MUST satisfy `1 ≤ k ≤ n`. Note `VOTING` with `k=1` is logically equivalent to `OR`, and `VOTING` with `k=n` is logically equivalent to `AND`; both are permitted, since `VOTING` can be the clearer choice when `n` is config-driven.
- `INHIBIT`: `inputs` MUST contain exactly 2 entries — the primary event and the conditioning event. The `inhibitCondition` text labels the conditioning branch.
- `PRIORITY_AND`: `inputs` MUST contain at least 2 entries, interpreted as a required sequence (first input first).

### 5.14 Basic Event Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `description` | string | REQUIRED | The specific, atomic cause this leaf represents. |
| `probability` | number, `0 ≤ x ≤ 1` | See note | Directly declared probability. |
| `failureRate` | number, `≥ 0` | See note | Constant failure rate `λ`, as an alternative to `probability` (Section 5.16). |
| `missionTime` | number, `> 0` | REQUIRED if `failureRate` is set | Exposure time `t` used with `failureRate`. Unit is implementation-defined but MUST be consistent with `failureRate`'s unit: within one document, all `failureRate`/`missionTime` pairs MUST use the same time unit (e.g. hours), so the conversion `P = 1 − e^(−λt)` is coherent. |
| `undeveloped` | boolean | OPTIONAL, default `false` | Marks this Basic Event as not further analyzed, rather than as a claim of true atomicity. Equivalent to `eventType: "undeveloped"`. |
| `eventType` | enum: `"basic"`, `"house"`, `"undeveloped"`, `"conditional"` | OPTIONAL, default `"basic"` | The fault-tree event symbol (IEC 61025). |
| `message` | External Reference | OPTIONAL | The AsyncAPI message that observably corresponds to this basic event occurring, if any. |

**Note:** a Basic Event MUST supply exactly one of `probability` or `failureRate` (with `missionTime`); supplying both, or neither, is a validation error (Section 7).

**`eventType` semantics:**

| Value | Symbol | Meaning |
|---|---|---|
| `"basic"` | circle | A basic event (default) |
| `"house"` | pentagon/house | An external event whose occurrence is a boundary condition. Its `probability` (or `failureRate`/`missionTime`) is **REQUIRED** — it is the supplied boundary value, not a computed leaf probability (W-406). |
| `"undeveloped"` | diamond | An event not further analyzed (equivalent to `undeveloped: true`) |
| `"conditional"` | ellipse | A conditional event that occurs only when a specific condition holds |

When both `undeveloped: true` and `eventType` are present, `eventType` takes precedence. `undeveloped: true` is retained as a deprecated alias for `eventType: "undeveloped"` (Section 10.3).

```yaml
faultTrees:
  PaymentGatewayFailure:
    description: "The payment gateway cannot successfully capture a charge."
    topEvent:
      id: PaymentCaptureFailed
      description: "A charge attempt against the payment gateway does not succeed."
      message: "payment_api#/components/messages/PaymentFailed"
      rootCause: GatewayUnavailableOrRejected

    gates:
      GatewayUnavailableOrRejected:
        type: OR
        description: "Either the gateway could not be reached, or it actively rejected the charge."
        inputs:
          - GatewayUnreachable
          - ChargeRejected

    basicEvents:
      GatewayUnreachable:
        description: "Stripe API did not respond within the configured timeout."
        probability: 0.008
        message: "payment_api#/components/messages/GatewayTimeout"

      ChargeRejected:
        description: "Stripe API responded with a hard decline."
        failureRate: 0.00021
        missionTime: 24
```

Note how both Basic Events carry a `message` field: the fault tree, like the event tree, is grounded in AsyncAPI's event vocabulary rather than floating free of it.

### 5.15 Linking Fault Trees to Event Trees

A Branch (5.8.1) or an Operation (5.9) MAY source its probability from a Fault Tree's computed Top Event probability instead of, or in addition to, a directly declared value, via the `probabilitySource` and `onFailureProbabilitySource` fields respectively — both Internal References of the form `#/faultTrees/<id>/topEvent`.

**Precedence:** when a probability-source field is present, its computed value (Section 5.16) is authoritative. Any directly-declared `probability`, `probabilityOfSuccess`, or `probabilityOfFailure` on the same Branch is a non-binding cached annotation of the last computed value; a compiler SHOULD recompute the Fault Tree at every build and SHOULD emit an advisory diagnostic (Section 7) if the cached value has drifted from the freshly computed one beyond a configurable tolerance, so a stale cache is visible rather than silently trusted.

**Directionality:** the linkage is one-way. A Fault Tree MUST NOT reference an Event Tree; a Basic Event's or Top Event's `message` field is a documentation-only annotation pointing at an AsyncAPI message, not an operational reference back into any Event Tree's nodes. This keeps Fault Tree resolution independent of Event Tree resolution, which a compiler is free to exploit by resolving all Fault Trees in one pass before touching any Event Tree (Section 8.1).

For example, `ProcessPaymentOperation` from the introductory example (Section 4) could be re-expressed against the Fault Tree of Section 5.14, replacing a previously-asserted constant with a derived, auditable one:

```yaml
ProcessPaymentOperation:
  type: operation
  action: execute
  handler: "stripe_charge_handler"
  emits: "payment_api#/components/messages/PaymentProcessed"
  next: FulfillmentConsequence
  onFailure: PaymentFailedConsequence
  onFailureProbabilitySource: "#/faultTrees/PaymentGatewayFailure/topEvent"
```

Section 13 develops this into a complete, self-consistent worked example.

### 5.16 Fault Tree Probability Computation

A conforming compiler MUST evaluate a Fault Tree's probability bottom-up, in topological order from Basic Events to the Top Event: every Gate's `inputs` MUST be fully resolved to numeric probabilities before that Gate's own probability is computed — the same DAG-topological-order requirement Section 1.2 makes binding, applied to `gates`/`basicEvents` instead of `nodes`.

A Basic Event's probability is either its declared `probability`, or, when `failureRate` (`λ`) and `missionTime` (`t`) are given instead, the standard exponential reliability model for a constant-failure-rate component:

```
P = 1 − e^(−λt)
```

A Gate's probability is computed from its resolved `inputs` per its `type`, under the assumption that a Gate's inputs are statistically independent (Section 1.4):

| Gate Type | Occurs iff | Probability (independent inputs `p₁ … pₙ`) |
|---|---|---|
| `AND` | every input occurs | `p₁ · p₂ · … · pₙ` |
| `OR` | at least one input occurs | `1 − (1−p₁)(1−p₂)⋯(1−pₙ)` |
| `NOT` | its single input does not occur | `1 − p₁` |
| `XOR` (2 inputs) | exactly one of the two inputs occurs | `p₁ + p₂ − 2p₁p₂` |
| `VOTING` (k-of-n) | at least `k` of the `n` inputs occur | If all `pᵢ = p`: `Σ_{j=k}^{n} C(n,j) · pʲ · (1−p)ⁿ⁻ʲ`. Otherwise, take the coefficients of `xᵏ` through `xⁿ` in the probability-generating polynomial `∏(1−pᵢ + pᵢx)`. |
| `INHIBIT` (2 inputs) | the primary input occurs while the conditioning event holds | `p₁ · p₂` (the two inputs are the primary event and the conditioning event; the `inhibitCondition` text labels the latter) |
| `PRIORITY_AND` (n ≥ 2) | all `n` inputs occur **in the listed order** (first input first) | Under the uniform-ordering assumption — each ordering of the `n` inputs equally likely — `(∏ pᵢ) / n!`. If component failure rates are available, a sequence-dependent (Markov) analysis MAY be used instead; the default static-probability model uses the uniform-ordering approximation. |

The Top Event's probability equals its `rootCause`'s resolved probability.

Probability computation is a MUST for every conforming compiler regardless of Fault Tree size, since it is a linear-time bottom-up pass. Enumerating minimal cut sets — the combinations of Basic Events that are, independent of their probabilities, sufficient to cause the Top Event — is a separate, potentially combinatorially expensive analysis, defined as a SHOULD-level compiler capability in Section 8.6.

---

## 6. The ETDL Condition Expression Language (ECEL)

### 6.1 Overview

ECEL is the expression language used in a Branch's `condition` field (Section 5.8.1). It is intentionally non-Turing-complete: no loops, no variable assignment, no calls to arbitrary code. Its only job is to evaluate a boolean guard against the message that triggered the tree. Gates (Section 5.13) do not use ECEL; their logic is structural (`type` plus `inputs`), not expression-based, since a fault tree's inputs are other named events, not arbitrary predicates over a payload.

### 6.2 Grammar

```
condition-expr   = "default" / comparison / quantifier-expr
comparison       = operand *WSP comparator *WSP operand
quantifier-expr  = quantifier "(" path-expr "," comparison ")"
quantifier       = "any" / "all"
operand          = path-expr / literal
path-expr        = root-var *member-access
root-var         = "message"
member-access    = "." identifier
                  / "[" ( "*" / index / quoted-key ) "]"
identifier       = ALPHA *( ALPHA / DIGIT / "_" )
index            = 1*DIGIT
quoted-key       = DQUOTE *(%x20-21 / %x23-7E) DQUOTE
comparator       = "==" / "!=" / ">=" / "<=" / ">" / "<" / "in" / "matches"
literal          = number / string-literal / "true" / "false" / "null" / array-literal
array-literal    = "[" [ literal *( "," literal ) ] "]"
number           = ["-"] 1*DIGIT ["." 1*DIGIT]
string-literal   = DQUOTE *(%x20-21 / %x23-7E) DQUOTE
```

String literals do not support escape sequences; a literal containing a double-quote character cannot be expressed and MUST be avoided. `array-literal` exists to support the right operand of `in` (Section 6.5); an array literal used anywhere else is a validation error. `quantifier-expr` provides explicit existential (`any`) or universal (`all`) quantification over an array-typed path (Section 6.4).

### 6.3 The Message Context

Exactly two root paths are in scope: `message.payload` (the triggering AsyncAPI message's payload) and `message.headers` (if the resolved AsyncAPI message defines any). Every `member-access` segment beyond the root MUST be checked at compile time against the resolved AsyncAPI schema for that message (Section 7, rule V-204); ECEL has no runtime-only fields.

### 6.4 Wildcard Quantification Semantics

When a `path-expr` traverses a `[*]` wildcard segment over an array-typed field, the enclosing `comparison` is evaluated once per array element and the results are combined with logical AND — universal quantification — consistent with the reference code generator's `.iter().all(...)` behavior (Section 8.4). For example, `message.payload.items[*].qty > 0` evaluates to `true` if and only if every element of `items` has `qty > 0`. An empty array satisfies universal quantification vacuously (`true`) by standard logical convention; an author who actually requires a non-empty array should pair the wildcard comparison with an explicit length check.

**Explicit quantifiers.** A condition MAY use an explicit quantifier to disambiguate or override the default universal semantics:

- `all(message.payload.items, message.payload.items[*].qty > 0)` — universal over `items` (equivalent to the default).
- `any(message.payload.items, message.payload.items[*].qty > 0)` — existential: `true` if and only if at least one element has `qty > 0`. An empty array yields `false` under `any`.

The `quantifier` is applied to the array-typed `path-expr` (the first argument); the second argument is a `comparison` whose `[*]` refers to the quantified array. Where explicit quantifiers are absent, `all` semantics MUST be assumed, for backward compatibility with 1.0.0-Standardized documents that predate this formalization.

### 6.5 Operators

| Operator | Meaning | Operand Constraint |
|---|---|---|
| `==`, `!=` | Equality / inequality | Both operands MUST resolve to the same ECEL runtime type (Section 6.7). |
| `>`, `>=`, `<`, `<=` | Ordering | Both operands MUST be `number`. |
| `in` | Membership | The right operand MUST be an array literal or an array-typed path. |
| `matches` | Pattern match | The left operand MUST be `string`; the right operand MUST be a string literal holding a valid **RE2**-syntax pattern. RE2 is mandated specifically because it guarantees linear-time matching with no catastrophic-backtracking pathway, consistent with the bounded-evaluation guarantee of Section 6.8. |

### 6.6 The `default` Literal

The literal string `"default"` used as an entire `condition` value is not evaluated as an ECEL expression; it is a sentinel recognized directly by the parser, matching only when no preceding sibling branch's condition matched (Section 7, rule V-202).

### 6.7 Type Coercion

| Declared AsyncAPI Type | ECEL Runtime Type | Rule |
|---|---|---|
| `integer`, `number` | `number` | Direct. |
| `string` | `string` | Direct. |
| `boolean` | `boolean` | Direct. |
| `array` | `array` | Direct; enables `[*]` and `in`. |
| `object` | — | Not directly comparable; MUST be narrowed with `.` member access first. |
| `null` | `null` | Comparable only via `==` / `!=`. |

ECEL performs **no implicit coercion** between runtime types — a `string` `"5"` is never equal to the `number` `5`. A `comparison` whose operands resolve to different runtime types is a compile-time type mismatch (Section 7, rule V-204), not a runtime `false`.

### 6.8 Evaluation Guarantees

A conforming ECEL evaluator MUST be:

- **Pure** — no side effects, no I/O, no mutation of the message.
- **Deterministic** — the same message always produces the same result.
- **Bounded** — evaluation time is linear in message size, with no unbounded-backtracking pathway (hence the RE2 mandate in 6.5) and no recursion.
- **Sandboxed** — no access to anything outside the `message` context: no environment variables, no filesystem, no network, no other node's state (Section 12).

---

## 7. Semantic Validation Rules

This section enumerates every rule a Conforming Parser (Section 2.3) MUST enforce. Rules are grouped by pipeline phase and by the object they apply to. **E-** codes are raised during reference resolution (Section 5.3.3); **V-** codes are raised during semantic validation, after resolution succeeds. Both are ERROR severity: per Section 2.3, a Conforming Parser MUST reject a document violating any E- or V-level rule, using the corresponding code. **W-** codes are advisory — a Conforming Parser SHOULD raise them but MAY proceed to code generation regardless. Appendix B consolidates every code below into one lookup table.

### 7.1 Document & Reference Errors (E-1xx)

| Code | Rule |
|---|---|
| E-100 | The document's `etdl` version is not a valid semantic version, or uses an unimplemented future MAJOR (Section 10.1). |
| E-101 | A reference string matches neither the External Reference nor the Internal Reference grammar (Section 5.3). |
| E-102 | The document contains a byte-order mark (BOM) (Section 4.3). |
| E-103 | An External Reference's import alias is not a key in `asyncapi_imports`, or the alias contains invalid characters (Section 5.3.1). |
| E-104 | An External Reference's JSON Pointer does not resolve within the target AsyncAPI document. |
| E-105 | An Internal Reference does not match one of the resolved shapes defined in Section 5.3.2 (`#/faultTrees/<id>/topEvent`, `#/faultTrees/<id>/gates/<gate-id>`, `#/faultTrees/<id>/basicEvents/<event-id>`, `#/components/<kind>/<id>`), or names an object absent from the document. |
| E-106 | A Supplement Object's `id` is not a valid supplement identifier (does not begin with `etdl.` or contains invalid characters) (Section 5.1.2). |
| E-107 | A Supplement Object's `version` is not valid SemVer, or declares a future MAJOR version the implementation does not support (Section 5.1.3). |
| E-108 | A declared supplement is `required: true` but is not implemented by this parser/compiler, and the configuration does not permit proceeding (Section 5.1.1). |

### 7.2 Structural Integrity — Event Trees (V-1xx)

Because Node IDs are YAML/JSON mapping keys, textually duplicate keys are typically rejected by the underlying document parser before ETDL-specific validation begins; a Conforming Parser built on a lenient YAML/JSON library (some silently keep the last of several duplicate keys) SHOULD additionally reject such input, since a silently-dropped node is a correctness hazard for causal analysis. The rules below assume that layer has already run.

| Code | Rule |
|---|---|
| V-101 | A `next`, `onFailure`, or `initiatingEvent.next` reference does not resolve to a key in the **same** Event Tree's `nodes` map. This includes an attempt to reference a node in a different Event Tree — cross-tree jumps are prohibited by Section 1.2. |
| V-102 | A cyclic reference exists among an Event Tree's nodes (DAG violation, Section 1.2). |
| V-103 | A node in `nodes` is unreachable — no `next`/`onFailure` edge, from any other node or `initiatingEvent`, targets it. |
| V-104 | A path from `initiatingEvent` does not terminate in a Consequence node. |

### 7.3 Barrier & Branch Rules (V-2xx)

| Code | Rule |
|---|---|
| V-201 | A Barrier declares fewer than two `branches`. |
| V-202 | A Barrier declares more than one `default` branch, or a `default` branch is not the last one evaluated. |
| V-203 | A Barrier's sibling branch probabilities (declared or fault-tree-derived, Section 5.15) do not sum to `1.0` within `±0.0001`, or a branch probability is outside `[0,1]` (Section 5.8.1). |
| V-204 | A `condition` references a field absent from the resolved AsyncAPI message schema, or contains a type mismatch (Section 6.7). |
| V-205 | A Branch supplies none of `probability`, `probabilityOfSuccess`, `probabilityOfFailure`, or `probabilitySource` (Section 5.8.1). |

### 7.4 Operation & Consequence Rules (V-3xx)

| Code | Rule |
|---|---|
| V-301 | An Operation's `handler` is not a syntactically valid identifier in at least one configured target language (Section 8.2). |
| V-302 | A Consequence node has `operation: send` but omits `channel` or `message`. |

### 7.5 Structural Integrity — Fault Trees (V-4xx)

These mirror 7.2's Event Tree rules in spirit, applied to a Fault Tree's `gates`/`basicEvents` graph instead of an Event Tree's `nodes`.

| Code | Rule |
|---|---|
| V-401 | A Gate's `inputs` entry, or a Top Event's `rootCause`, does not resolve to a key in that Fault Tree's `gates` or `basicEvents` maps. |
| V-402 | A Gate and a Basic Event within the same Fault Tree Object share an ID (Section 5.11's shared-namespace requirement). |
| V-403 | A cyclic reference exists among a Fault Tree's Gates (DAG violation). |
| V-404 | A Gate or Basic Event is unreachable from `topEvent.rootCause`. |

### 7.6 Gate & Basic Event Rules (V-5xx)

| Code | Rule |
|---|---|
| V-501 | A Gate's `inputs` arity violates its `type`'s rule (Section 5.13: AND/OR ≥ 2, NOT = 1, XOR = 2, VOTING ≥ 2, INHIBIT = 2, PRIORITY_AND ≥ 2). |
| V-502 | A `VOTING` gate's `k` does not satisfy `1 ≤ k ≤ n`, where `n = len(inputs)`. |
| V-503 | A Basic Event supplies both `probability` and `failureRate`, or supplies neither. |
| V-504 | A Basic Event's `failureRate` is set without a corresponding `missionTime`. |
| V-505 | An `INHIBIT` gate omits the required `inhibitCondition` field. |
| V-506 | A Transfer's `target` is not an Internal Reference of the form `#/faultTrees/<id>/...`, or references a Fault Tree absent from the document (Section 5.11.1). |

### 7.7 Advisory Diagnostics (W-4xx)

| Code | Rule |
|---|---|
| W-401 | An Operation node has no `onFailure` path (Section 5.9) — its failure is unmodeled and cannot appear in any Fault Tree's minimal cut sets (Section 8.6). |
| W-402 | A Branch's or Operation's cached declared probability has drifted from its linked Fault Tree's freshly computed probability beyond a configurable tolerance (Section 5.15). |
| W-405 | A Transfer has an empty `label`. |
| W-406 | A Basic Event of type `"house"` omits a `probability`/`failureRate` — a house event is a boundary condition whose value is supplied, not derived (Section 5.14). |
| W-407 | A declared supplement is not implemented by this parser/compiler and is not `required`; its semantics will not be applied (Section 5.1.1). |

---

## 8. Compiler and Code Generation Semantics

### 8.1 Compilation Pipeline

A Conforming Compiler processes a `.etdl` document in the following order; each stage MUST complete without error before the next begins:

1. **Parse.** Deserialize the document (Section 4.1) into an in-memory representation.
2. **Resolve Imports.** Load and parse every document listed in `asyncapi_imports`.
3. **Validate Structure.** Enforce the Section 7 rules that don't require cross-referencing computed probabilities.
4. **Resolve Fault Trees.** For every Fault Tree Object, compute Basic Event probabilities, then Gate probabilities bottom-up, then the Top Event probability (Section 5.16) — entirely independent of any Event Tree (Section 5.15).
5. **Resolve Probability Links.** Substitute every `probabilitySource` / `onFailureProbabilitySource` with the value computed in stage 4, then re-check rule V-203 (probability-sum) with the now-fully-resolved values.
6. **Type-Check Conditions.** Validate every ECEL `condition` against its resolved AsyncAPI schema (rule V-204).
7. **Generate Code.** Emit target-language artifacts per Sections 8.2–8.3, for each Event Tree.
8. **Instrument (optional).** Wire in `etdl_core` telemetry hooks (Section 9) if the target build profile requests them.

Stage 4 running before stage 7, and being entirely independent of any Event Tree, is what lets a compiler resolve all of a document's Fault Trees in a single pass regardless of how many Event Trees reference them.

### 8.2 Target Language Interface Contract

For each Event Tree, a Conforming Compiler MUST generate, in every supported target language:

- One entry-point function per `initiatingEvent`, named by transforming `initiatingEvent.id` into the target language's idiomatic convention (`snake_case`, prefixed `handle_`, for Rust; `camelCase`, prefixed `handle`, for TypeScript), taking the resolved message type as its parameter.
- One `BranchMonitor` (Section 9.1) instantiated per Barrier reached, with `recordBranch` invoked with the outcome taken and its effective probability.
- One `recordFailure` invocation per reached Operation failure, passing the probability derived from `onFailureProbabilitySource` where set, or none where absent (Section 9.1).
- Faithful reproduction of `retryPolicy` and `timeoutMs` semantics using the target language's native async/concurrency idiom.
- W3C `traceparent` propagation into every emitted Consequence message (Section 9.2).
- A typed error path through which an unhandled Operation failure (no `onFailure`) propagates to the entry point's caller, using the target language's native error idiom — `Result<T, E>` in Rust, a rejected `Promise` in TypeScript — rather than a bespoke ETDL-specific mechanism.

This contract is what makes two target languages' output comparable implementations of the *same* tree, rather than merely two programs that happen to behave similarly. Sections 8.4–8.5 generate Rust and TypeScript from one identical source tree to demonstrate it.

### 8.3 Generated Code Requirements

In addition to the interface contract above, generated code:

- MUST begin with a header comment identifying it as autogenerated and naming the compiler version: `// AUTOGENERATED BY ETDL COMPILER v<version> - DO NOT EDIT DIRECTLY`.
- MUST NOT perform network I/O to any ETDL-specific service at runtime (Section 1.2) — every value it needs, including fault-tree-derived probabilities, MUST already be resolved to a literal by compile time, not fetched from an ETDL toolchain component.
- SHOULD be deterministic for a given input message and a given set of resolved probabilities, excluding genuinely random elements such as retry jitter.
- MUST use the target language's native error-handling idiom for the unhandled-failure path (8.2).

### 8.4 Reference Implementation: Rust

Generated from the `OrderFulfillment` Event Tree of Section 13, with `ProcessPaymentOperation`'s failure probability linked to the `PaymentGatewayFailure` Fault Tree of Section 5.14 via `onFailureProbabilitySource`:

```rust
// AUTOGENERATED BY ETDL COMPILER v1.0.0 - DO NOT EDIT DIRECTLY
use std::time::Duration;
use etdl_core::telemetry::BranchMonitor;
use etdl_core::retry::{RetryPolicy, BackoffStrategy};
use orders_api::messages::OrderPlaced;
use payment_api::messages::{PaymentProcessed, PaymentFailed};

// Computed from faultTrees.PaymentGatewayFailure.topEvent at build time (Section 5.16):
// GatewayUnreachable (0.008) OR ChargeRejected (1 - e^-(0.00021 * 24) ≈ 0.005027) = 0.012987
const PROCESS_PAYMENT_FAILURE_PROBABILITY: f64 = 0.012987;

pub async fn handle_order_placed_trigger(message: OrderPlaced) -> Result<(), WorkflowError> {
    let mut monitor = BranchMonitor::new("InventoryCheckBarrier");

    if message.payload.items.iter().all(|i| i.qty > 0) {
        monitor.record_branch("SUCCESS", 0.95);

        let retry = RetryPolicy {
            max_attempts: 3,
            backoff_ms: 250,
            strategy: BackoffStrategy::Exponential,
        };

        match retry.execute(|| stripe_charge_handler(&message), Duration::from_millis(5000)).await {
            Ok(payment_result) => {
                publish_to_channel("FulfillmentChannel", payment_result).await?;
            }
            Err(err) => {
                monitor.record_failure(
                    "ProcessPaymentOperation",
                    &err,
                    Some(PROCESS_PAYMENT_FAILURE_PROBABILITY),
                );
                publish_to_channel("DeadLetterChannel", PaymentFailed::from(err)).await?;
            }
        }
    } else {
        monitor.record_branch("FAILURE", 0.05);
        publish_to_channel("DeadLetterChannel", message.into_failed()).await?;
    }

    Ok(())
}
```

### 8.5 Reference Implementation: TypeScript

The same tree, compiled to TypeScript. The `BranchMonitor` call sites, the computed constant, and the control flow are identical — only syntax differs, exactly as the Target Language Interface Contract (8.2) requires:

```typescript
// AUTOGENERATED BY ETDL COMPILER v1.0.0 - DO NOT EDIT DIRECTLY
import { BranchMonitor, RetryPolicy, BackoffStrategy } from "@etdl/core";
import type { OrderPlaced } from "./orders-api/messages";
import type { PaymentProcessed, PaymentFailed } from "./payment-api/messages";
import { stripeChargeHandler } from "./handlers/stripe-charge-handler";
import { publishToChannel } from "./runtime/channels";

// Computed from faultTrees.PaymentGatewayFailure.topEvent at build time (Section 5.16):
// GatewayUnreachable (0.008) OR ChargeRejected (1 - e^-(0.00021 * 24) ≈ 0.005027) = 0.012987
const PROCESS_PAYMENT_FAILURE_PROBABILITY = 0.012987;

export async function handleOrderPlacedTrigger(message: OrderPlaced): Promise<void> {
  const monitor = new BranchMonitor("InventoryCheckBarrier");

  const inStock = message.payload.items.every((i) => i.qty > 0);

  if (inStock) {
    monitor.recordBranch("SUCCESS", 0.95);

    const retry = new RetryPolicy({
      maxAttempts: 3,
      backoffMs: 250,
      strategy: BackoffStrategy.Exponential,
    });

    try {
      const paymentResult: PaymentProcessed = await retry.execute(
        () => stripeChargeHandler(message),
        { timeoutMs: 5000 }
      );
      await publishToChannel("FulfillmentChannel", paymentResult);
    } catch (err) {
      monitor.recordFailure("ProcessPaymentOperation", err as Error, PROCESS_PAYMENT_FAILURE_PROBABILITY);
      await publishToChannel("DeadLetterChannel", toPaymentFailed(err));
    }
  } else {
    monitor.recordBranch("FAILURE", 0.05);
    await publishToChannel("DeadLetterChannel", toInventoryFailed(message));
  }
}
```

### 8.6 Minimal Cut Set Enumeration (Fault Trees)

Minimal cut set enumeration in ETDL 1.0.0 is defined only for **coherent** Fault Trees — those built entirely from `AND`, `OR`, `VOTING`, `INHIBIT`, and `PRIORITY_AND` gates, all of which are monotonic (an additional Basic Event occurrence can only make the Top Event more, never less, likely). `NOT` and `XOR` gates are non-monotonic and produce a *non-coherent* fault tree, for which classical cut-set enumeration is not directly defined. A Conforming Compiler MUST refuse to enumerate cut sets for a Fault Tree containing a `NOT` or `XOR` gate, and MUST document that refusal clearly, rather than emit a result that silently ignores the negation. Probability computation (Section 5.16) is unaffected by this restriction and remains available for every gate type.

For a coherent Fault Tree, the classical **Method of Obtaining Cut Sets (MOCUS)** proceeds top-down by Boolean substitution:

1. Begin with a single row containing the Top Event's `rootCause`.
2. Repeatedly substitute any Gate appearing in a row with its `inputs`, until every row contains only Basic Events:
   - Substituting an `OR` gate **replaces its row with `n` new rows**, one per input — branching, since any one input alone is sufficient.
   - Substituting an `AND` gate **replaces the gate, in place, with all of its inputs in the same row** — the row grows, since every input is jointly necessary.
   - Substituting a `VOTING` (k-of-n) gate replaces its row with one new row per size-`k` combination of its `inputs`, each combination's members conjoined within that row.
3. Each finished row is a cut set. Discard any row that is a superset of another finished row — the survivors are the **minimal** cut sets.

Applied to the Fault Tree of Section 5.14 (a single `OR` gate over two Basic Events), MOCUS immediately yields two minimal cut sets: `{GatewayUnreachable}` and `{ChargeRejected}` — either alone is sufficient to cause `PaymentCaptureFailed`.

This algorithm has exponential worst-case output size in the number of `OR`/`VOTING` branch points, which is why enumeration is a SHOULD, capped at an implementation-defined tree-size threshold, rather than a MUST — unlike probability computation (8.1, stage 4), which is linear-time and therefore unconditional.

---

## 9. Runtime Library Contract (etdl_core)

### 9.1 BranchMonitor API

| Concept | Rust (`etdl_core`) | TypeScript (`@etdl/core`) | Description |
|---|---|---|---|
| Construct | `BranchMonitor::new(node_id: &str)` | `new BranchMonitor(nodeId: string)` | Bind a monitor instance to a Barrier or Operation node ID. |
| Record branch | `monitor.record_branch(outcome: &str, declared_probability: f64)` | `monitor.recordBranch(outcome: string, declaredProbability: number)` | Record which Barrier outcome occurred and its effective (declared or fault-tree-derived) baseline probability. |
| Record failure | `monitor.record_failure(operation_id: &str, error: &dyn Error, declared_probability: Option<f64>)` | `monitor.recordFailure(operationId: string, error: Error, declaredProbability?: number)` | Record that an Operation's handler failed. The probability argument is present only when `onFailureProbabilitySource` supplied one; when absent, the call still counts the failure for observability but is excluded from SLA deviation checks (9.3). |
| Flush | implementation-defined (e.g. on `Drop`) | implementation-defined (e.g. process exit hook) | Ensure telemetry is exported before process exit. |

### 9.2 OpenTelemetry Propagation

Every Consequence's `send` operation MUST inject the current W3C `traceparent` (and `tracestate`, if present) header into the outgoing message, so a distributed trace spans the full causal path from `initiatingEvent` through every Operation to the terminal Consequence, including across the AsyncAPI channel boundary into whatever service consumes it next. A `BranchMonitor` MUST attach the node ID it was constructed with as a span attribute (conventionally `etdl.node.id`) on the current span, so `recordBranch`/`recordFailure` calls correlate with the request that triggered them.

### 9.3 SLA Anomaly Alerting

A `BranchMonitor` MUST compare each `recordBranch`/`recordFailure` call's declared probability, when present, against the empirically observed frequency of that outcome over a rolling window, and MUST emit an anomaly signal when the two diverge beyond a threshold:

| Parameter | Default | Environment Variable |
|---|---|---|
| Rolling window size | 1000 evaluations | `ETDL_SLA_WINDOW` |
| Deviation threshold | `0.10` | `ETDL_SLA_THRESHOLD` |

The observed frequency of an outcome is the fraction of evaluations in the
rolling window that produced that outcome (hits / total). An anomaly is detected
when the absolute probability difference exceeds the threshold:

```
|observed − declared| > threshold
```

where `threshold` is a probability-space value in `[0, 1]` (default `0.10`). The
`ETDL_SLA_THRESHOLD` value, when set, MUST parse as a number in `[0, 1]`; an
unparseable or out-of-range value falls back to the default. An anomaly is only
reported once the window contains a minimum number of evaluations sufficient to
be statistically meaningful (at least 10).

On breach, the runtime MUST increment a counter metric (conventionally `etdl_sla_anomaly_total`, labeled by node ID and outcome) and SHOULD emit an OpenTelemetry span event on the current span, so the anomaly is visible in both metrics and trace tooling.

### 9.4 Chaos Injection Mode

When `ETDL_CHAOS` is set to a truthy value, the runtime MUST deterministically route execution through the lower-probability path for every node within scope, regardless of the condition's or handler's actual result, to exercise compensation and error-handling code under test.

The **lower-probability path** is defined deterministically:

- **Barrier:** the branch with the lowest effective probability among the non-`default` branches. Ties are broken by branch order (the first lowest-probability branch wins). The `default` branch is never selected as the chaos path unless it is the only branch.
- **Operation:** the `onFailure` path (its failure path). An Operation with no `onFailure` is unaffected by chaos.

| Variable | Default | Purpose |
|---|---|---|
| `ETDL_CHAOS` | `false` | Master switch. |
| `ETDL_CHAOS_SEED` | unset (non-deterministic) | When set, seeds branch selection so a chaos run is reproducible. |
| `ETDL_CHAOS_SCOPE` | unset (applies to every node) | Comma-separated Node IDs restricting chaos injection to specific Barriers/Operations. |

`ETDL_CHAOS` MUST be ignored — chaos injection MUST NOT activate — when the runtime detects it is running in a production environment (Section 12).

---

## 10. Versioning and Compatibility

### 10.1 Specification Versioning

The `etdl` field (Section 5.1) follows Semantic Versioning:

- **MAJOR** increments on breaking changes to the document schema or validation rules.
- **MINOR** increments on additive, backward-compatible changes (new optional fields, new gate types, etc.).
- **PATCH** increments on clarifications that don't change validated behavior.

A Conforming Parser MUST accept any document whose `etdl` MAJOR version matches its own supported MAJOR version, MAY accept documents from other MAJOR versions on a best-effort basis, and MUST reject documents from a future MAJOR version it does not implement, rather than silently misinterpreting unrecognized structure.

### 10.2 Migration from 1.0.0-Standardized

A document written against 1.0.0-Standardized (the proof-of-concept predecessor to this specification) requires **no changes** to remain parseable:

| 1.0.0-Standardized construct | 1.0.0 status | Compatibility path |
|---|---|---|
| `eventTree` (singular) | DEPRECATED | Still accepted (Section 5.1); a Conforming Parser MUST treat a bare `eventTree` exactly as `eventTrees: { default: <same content> }`. |
| `probabilityOfSuccess` / `probabilityOfFailure` | DEPRECATED | Still accepted on any Branch (Section 5.8.1) as an alias for `probability`, keyed by the branch's `outcome`. |
| Operation nodes with no `onFailure` | Unchanged | Still valid; produces advisory W-401 (Section 7.7) rather than an error. |

No 1.0.0-Standardized document uses `faultTrees`, `probabilitySource`, `onFailureProbabilitySource`, `components.gates`, or `components.basicEvents`, since none of this existed in that draft; a Conforming Parser simply finds those fields absent and proceeds with directly-declared probabilities throughout, exactly as such documents already do. Appendix D details every change between the two versions, with rationale.

### 10.3 Deprecation Policy

A field marked DEPRECATED in this specification remains supported for at least one full MAJOR version cycle beyond the version that introduced its replacement — `probabilityOfSuccess`/`probabilityOfFailure` and `eventTree`, both deprecated in 1.0.0, will not be removed before a hypothetical 2.0.0. A Conforming Parser encountering a deprecated field MUST accept it and SHOULD surface a non-blocking advisory identifying the preferred replacement.

---

## 11. Extensibility

Any object defined in Section 5 MAY carry additional fields whose names begin with `x-` (a **specification extension**). A Conforming Parser MUST preserve unrecognized `x-` fields through parsing, so downstream tooling can read them, and MUST NOT reject a document solely for containing one — the same convention used by OpenAPI, AsyncAPI, and CloudEvents. Fields not beginning with `x-` are reserved for future versions of this specification; a Conforming Parser MUST reject an unrecognized non-`x-` field (Appendix E).

---

## 12. Security Considerations

**Condition expression sandboxing.** ECEL's evaluation guarantees (Section 6.8) — purity, determinism, boundedness, and the mandatory RE2 dialect for `matches` (Section 6.5) — exist specifically to prevent a `condition` field from becoming an arbitrary-code-execution or denial-of-service vector. An implementation MUST NOT extend ECEL with a general-purpose scripting escape hatch (e.g., an `eval`-style function) without forfeiting these guarantees and this specification's conformance.

**Import resolution and path traversal.** Resolving a local-filesystem `asyncapi_imports` target MUST NOT permit a `../` (or platform-equivalent) escape outside a configured project root. A remote (`https://`) import target SHOULD be restricted to an explicit allow-list rather than fetched unconditionally, since an ETDL document's imports are effectively a supply-chain input to the compiler.

**Untrusted AsyncAPI documents.** An AsyncAPI document is data (schemas and channel definitions), not executable code; resolving one carries no code-execution risk beyond the general risk of parsing untrusted structured data. Implementations SHOULD still apply reasonable resource limits (document size, `$ref` depth) when parsing an import from an untrusted source, consistent with general defensive-parsing practice.

**Chaos Injection Mode in production.** Section 9.4's `ETDL_CHAOS` deliberately forces failure paths to execute. A runtime MUST detect a production environment — by whatever signal its ecosystem conventionally uses (e.g., a `deployment.environment` OpenTelemetry resource attribute) — and MUST ignore `ETDL_CHAOS` when one is detected, rather than trust the flag unconditionally. An implementation that cannot reliably detect its environment MUST NOT enable Chaos Injection Mode by default.

**Probabilities are not a safety case.** Reiterating Section 1.4: `probability` values, whether declared or Fault-Tree-derived, are operational SLO inputs, not a certified Probabilistic Risk Assessment. A system with an actual safety-case obligation MUST NOT cite an ETDL document's computed probabilities as that certification.

---

## 13. Full Worked Example

This section assembles a complete, valid `.etdl` document combining every construct introduced in Sections 5–9: an Event Tree with a Barrier, a retrying Operation, and three terminal Consequences; and a Fault Tree whose computed Top Event probability — rather than an asserted constant — drives that Operation's failure tracking. This is the same tree Sections 8.4 and 8.5 compile to Rust and TypeScript.

```yaml
etdl: "1.0.0"
info:
  title: "Order Fulfillment Event Tree"
  version: "2.0.0"
  domain: "FulfillmentContext"
  description: "Causal model of the order-to-fulfillment lifecycle, with payment failure quantified by a linked fault tree rather than asserted."

asyncapi_imports:
  orders_api: "./asyncapi/orders.yaml"
  payment_api: "./asyncapi/payments.yaml"

eventTrees:
  OrderFulfillment:
    description: "Primary causal path from order placement to fulfillment or terminal failure."
    initiatingEvent:
      id: OrderPlacedTrigger
      message: "orders_api#/components/messages/OrderPlaced"
      next: InventoryCheckBarrier

    nodes:
      InventoryCheckBarrier:
        type: barrier
        description: "Verifies stock availability across warehouse nodes."
        branches:
          - outcome: SUCCESS
            condition: "message.payload.items[*].qty > 0"
            probability: 0.95
            next: ProcessPaymentOperation
          - outcome: FAILURE
            condition: "default"
            probability: 0.05
            next: OutOfStockConsequence

      ProcessPaymentOperation:
        type: operation
        description: "Charges the customer's payment method via the Stripe handler."
        action: execute
        handler: "stripe_charge_handler"
        emits: "payment_api#/components/messages/PaymentProcessed"
        next: FulfillmentConsequence
        onFailure: PaymentFailedConsequence
        onFailureProbabilitySource: "#/faultTrees/PaymentGatewayFailure/topEvent"
        retryPolicy:
          maxAttempts: 3
          backoffMs: 250
          backoffStrategy: exponential
        timeoutMs: 5000

      FulfillmentConsequence:
        type: consequence
        description: "Order is successfully paid and broadcast to global fulfillment."
        operation: send
        channel: "orders_api#/channels/FulfillmentChannel"
        message: "payment_api#/components/messages/PaymentProcessed"

      OutOfStockConsequence:
        type: consequence
        description: "Order halted due to stock depletion; routed to dead-letter storage."
        operation: send
        channel: "orders_api#/channels/DeadLetterChannel"
        message: "orders_api#/components/messages/InventoryFailed"

      PaymentFailedConsequence:
        type: consequence
        description: "Payment could not be captured after retries; routed to dead-letter storage for manual review."
        operation: send
        channel: "orders_api#/channels/DeadLetterChannel"
        message: "payment_api#/components/messages/PaymentFailed"

faultTrees:
  PaymentGatewayFailure:
    description: "The payment gateway cannot successfully capture a charge."
    topEvent:
      id: PaymentCaptureFailed
      description: "A charge attempt against the payment gateway does not succeed."
      message: "payment_api#/components/messages/PaymentFailed"
      rootCause: GatewayUnavailableOrRejected

    gates:
      GatewayUnavailableOrRejected:
        type: OR
        description: "Either the gateway could not be reached, or it actively rejected the charge."
        inputs:
          - GatewayUnreachable
          - ChargeRejected

    basicEvents:
      GatewayUnreachable:
        description: "Stripe API did not respond within the configured timeout."
        probability: 0.008
        message: "payment_api#/components/messages/GatewayTimeout"

      ChargeRejected:
        description: "Stripe API responded with a hard decline."
        failureRate: 0.00021
        missionTime: 24
```

`ProcessPaymentOperation`'s failure probability is not written anywhere in this document as a number: it is `0.012987`, computed at build time (Section 5.16) by combining the two `PaymentGatewayFailure` Basic Events through their `OR` gate — exactly the value the generated code in Sections 8.4–8.5 embeds as a literal, with its provenance preserved in a comment. Changing `GatewayUnreachable`'s declared `probability`, or Stripe's actual timeout characteristics reflected in `ChargeRejected`'s `failureRate`, changes that generated constant on the next build, without touching the Event Tree at all.

---

## Appendix A — Consolidated Grammar Reference

The complete ECEL grammar (Section 6.2), consolidated:

```
condition-expr   = "default" / comparison / quantifier-expr
comparison       = operand *WSP comparator *WSP operand
quantifier-expr  = quantifier "(" path-expr "," comparison ")"
quantifier       = "any" / "all"
operand          = path-expr / literal
path-expr        = root-var *member-access
root-var         = "message"
member-access    = "." identifier
                  / "[" ( "*" / index / quoted-key ) "]"
identifier       = ALPHA *( ALPHA / DIGIT / "_" )
index            = 1*DIGIT
quoted-key       = DQUOTE *(%x20-21 / %x23-7E) DQUOTE
comparator       = "==" / "!=" / ">=" / "<=" / ">" / "<" / "in" / "matches"
literal          = number / string-literal / "true" / "false" / "null" / array-literal
array-literal    = "[" [ literal *( "," literal ) ] "]"
number           = ["-"] 1*DIGIT ["." 1*DIGIT]
string-literal   = DQUOTE *(%x20-21 / %x23-7E) DQUOTE
```

No escapes in string literals; no exponent notation in numbers; a condition is
`default`, a single `comparison`, or a `quantifier-expr` (`any`/`all`).

---

## Appendix B — Diagnostic Code Registry

Consolidated registry of every diagnostic code defined in Section 7. Codes are
stable within a MAJOR version; new codes are added, never reused.

### Document & Reference Errors (E-1xx)

| Code | Rule |
|---|---|
| E-100 | The document's `etdl` version is not a valid semantic version, or uses an unimplemented future MAJOR (Section 10.1). |
| E-101 | A reference string matches neither the External nor the Internal Reference grammar (Section 5.3). |
| E-102 | The document contains a byte-order mark (Section 4.3). |
| E-103 | An External Reference's import alias is not a key in `asyncapi_imports`, or the alias contains invalid characters (Section 5.3.1). |
| E-104 | An External Reference's JSON Pointer does not resolve within the target AsyncAPI document (Section 5.3.1). |
| E-105 | An Internal Reference does not match one of the resolved shapes defined in Section 5.3.2 (`#/faultTrees/<id>/topEvent`, `#/faultTrees/<id>/gates/<gate-id>`, `#/faultTrees/<id>/basicEvents/<event-id>`, `#/components/<kind>/<id>`), or names an object absent from the document. |
| E-106 | A Supplement Object's `id` is not a valid supplement identifier (Section 5.1.2). |
| E-107 | A Supplement Object's `version` is not valid SemVer, or declares a future MAJOR the implementation does not support (Section 5.1.3). |
| E-108 | A declared supplement is `required: true` but not implemented, and configuration does not permit proceeding (Section 5.1.1). |

### Event-Tree Structure (V-1xx)

| Code | Rule |
|---|---|
| V-101 | A `next`/`onFailure`/`initiatingEvent.next` does not resolve to a key in the same tree's `nodes` (Section 5.7). |
| V-102 | A cyclic reference among an Event Tree's nodes (Section 5.7). |
| V-103 | A node in `nodes` is unreachable from any edge (Section 5.7). |
| V-104 | A path from `initiatingEvent` does not terminate in a Consequence node (Section 7.2). |

### Barrier & Branch Rules (V-2xx)

| Code | Rule |
|---|---|
| V-201 | A barrier has fewer than two branches (Section 5.8). |
| V-202 | More than one `default` branch, or a `default` branch not evaluated last (Section 5.8.1). |
| V-203 | A Barrier's sibling branch probabilities (declared or fault-tree-derived, Section 5.15) do not sum to `1.0` within `±0.0001`, or a branch probability is outside `[0,1]` (Section 5.8.1). |
| V-204 | An ECEL `condition` references a field absent from the resolved AsyncAPI schema, or contains a type mismatch (Sections 6.3, 6.7). |
| V-205 | A Branch supplies none of `probability`, `probabilityOfSuccess`, `probabilityOfFailure`, or `probabilitySource` (Section 5.8.1). |

### Operation & Consequence Rules (V-3xx)

| Code | Rule |
|---|---|
| V-301 | An Operation's `handler` is not a syntactically valid identifier in at least one configured target language (Section 8.2). |
| V-302 | A Consequence has `operation: send` but omits `channel` or `message` (Section 5.10). |

### Fault-Tree Structure (V-4xx)

| Code | Rule |
|---|---|
| V-401 | A Gate's `inputs` entry, or a Top Event's `rootCause`, does not resolve to a key in that tree's `gates`/`basicEvents` (Section 5.12). |
| V-402 | A Gate and a Basic Event in the same tree share an ID (Section 5.11). |
| V-403 | A cyclic reference among a Fault Tree's Gates (Section 5.13). |
| V-404 | A Gate or Basic Event is unreachable from `topEvent.rootCause` (Section 5.11). |

### Gate & Basic Event Rules (V-5xx)

| Code | Rule |
|---|---|
| V-501 | A Gate's `inputs` arity violates its `type`'s rule (Section 5.13). |
| V-502 | A `VOTING` gate's `k` does not satisfy `1 ≤ k ≤ n` (Section 5.13). |
| V-503 | A Basic Event supplies both `probability` and `failureRate`, or neither (Section 5.14). |
| V-504 | A Basic Event's `failureRate` is set without a corresponding `missionTime` (Section 5.14). |
| V-505 | An `INHIBIT` gate omits the required `inhibitCondition` (Section 5.13). |
| V-506 | A Transfer's `target` is not `#/faultTrees/<id>/...` or references a Fault Tree absent from the document (Section 5.11.1). |

### Advisory Diagnostics (W-4xx)

| Code | Rule |
|---|---|
| W-401 | An Operation has no `onFailure` path (Section 5.9). |
| W-402 | A cached declared probability drifted from the freshly computed Fault Tree value (Section 5.15). |
| W-405 | A Transfer has an empty `label` (Section 5.11.1). |
| W-406 | A Basic Event of type `"house"` omits a `probability`/`failureRate` — a house event is a boundary condition whose value is supplied, not derived (Section 5.14). |
| W-407 | A declared, non-required supplement is not implemented; its semantics will not be applied (Section 5.1.1). |

---

## Appendix C — Reserved Words

The following identifiers are reserved and MUST NOT be used as `info.domain`
values (Section 5.2) or as Event Tree / Fault Tree / node names where the
grammar could be ambiguous with ECEL keywords:

- `message` — the sole ECEL root variable (Section 6.3).
- `default` — the ECEL sentinel (Section 6.6).
- `payload`, `headers` — the two ECEL member roots (Section 6.3).

An implementation MAY reserve additional identifiers internally; this list is
the normative minimum for conformance.

---

## Appendix D — Changelog: Draft to Formal Specification

Substantive changes from **1.0.0-Standardized** (Proof-of-Concept Draft) to
**1.0.0** (Formal):

| Area | Draft | 1.0.0 Formal |
|---|---|---|
| ECEL | No grammar | Full grammar (Section 6.2) + type checking (6.7) + evaluation guarantees (6.8) |
| Operation failure | No representable failure path | `onFailure` + `onFailureProbabilitySource` + retry/timeout (Sections 5.9, 5.15) |
| AsyncAPI references | Informal | Formal External/Internal Reference grammar + resolution algorithm (Section 5.3) |
| Validation | Three sentences | Full E-/V-/W- registry (Section 7, Appendix B) |
| Fault trees | Direct probabilities only | `faultTrees`, `probabilitySource`, `onFailureProbabilitySource`, `components.gates`, `components.basicEvents` (Sections 5.11, 5.15, 5.4) |
| Gate types | — | AND, OR, NOT, XOR, VOTING (formalized); INHIBIT, PRIORITY_AND (added in this revision) |
| Basic Event types | — | `eventType` (basic/house/undeveloped/conditional) added in this revision |
| Transfers | — | `transfers` (Section 5.11.1) added in this revision |
| ECEL array literals | Not in grammar | Added to support `in` (Sections 6.2, 6.5) |

---

## Appendix E — Companion Artifacts (JSON Schema)

A machine-readable JSON Schema for a **Conforming Document** is maintained as a
companion artifact in this repository (`schemas/etdl.schema.json`). The schema
implements the structural subset of Sections 4–5 and MUST accept every document
that satisfies the Section 5 MUST-level rules and the Section 6.2 grammar.

> Note: The JSON Schema validates structure and types; rules requiring computed
> values (probability sums, reference resolution, type checking) are enforced by
> a Conforming Parser/Compiler, not by the schema alone. The schema is the
> first-pass gate; Sections 7 and 8 remain normative for full conformance.
