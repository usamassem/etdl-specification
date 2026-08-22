# ETDL Security Supplement 1.0

**Supplement id:** `etdl.security`
**Version:** 1.0.0
**Profile:** extends ETDL Core 1.0.0
**Category:** Core (Section 11.4 of the core specification)
**Status:** Under Development — NOT YET RELEASED
**Normative dependency:** ETDL Core 1.0.0, ETDL Tree Event Supplement 1.0 (`etdl.tree-event`)

> **This supplement classifies threats and maps mitigating controls. It
> defines no new tree structure of its own: an attack tree is structurally
> identical to any other tree of OR/AND-combined causes, so this supplement
> reuses the Tree Event Supplement's generic Tree Object (its own Section 4)
> under a security interpretation, exactly the "separate, consuming layer"
> pattern that supplement's Section 1 describes and explicitly permits.**

---

## 1. Relationship to ETDL Core and to the Tree Event Supplement

This supplement declares an explicit dependency on `etdl.tree-event`
(core Section 5.1.4):

```yaml
supplements:
  - id: etdl.security
    version: "1.0"
    metadata:
      x-requires:
        - id: etdl.tree-event
          range: ">=1.0 <2.0"
  - id: etdl.tree-event
    version: "1.0"
```

An **attack tree** — a threat decomposed into the sub-conditions that
achieve it, combined through AND/OR logic — has exactly the shape the Tree
Event Supplement already validates: a root, named nodes, gates with arity
rules, no cycles. This supplement does not re-define that structure; it
attaches a security interpretation (a STRIDE category per leaf) to an
already-validated `x-tree-event` Tree, and separately maps mitigating
controls onto core Barrier nodes (Section 5.8) — the same "give existing
core structure a domain meaning" pattern the Safety Supplement
(`etdl.safety`) uses for the same Barrier node under a different
interpretation. A document MAY declare both; they read the same Barrier
node independently and do not conflict (core Section 11.2).

**In-document representation.** Security data is carried in an
`x-security` extension field (core Section 11.1) at the document root:

```yaml
x-security:
  threatModels:
    - id: payment-gateway-attack-tree
      treeRef: "gateway-compromise"
      leafCategories:
        CredentialStuffing: spoofing
        ApiKeyLeak: information-disclosure
        RateLimitBypass: denial-of-service
  controls:
    - id: gateway-rate-limiter
      nodeRef: "#/eventTrees/OrderFulfillment/nodes/RateLimitBarrier"
      framework: "NIST-800-53"
      controlId: "SC-5"
      mitigates: ["RateLimitBypass"]
```

---

## 2. Conformance

An implementation is **ETDL Security 1.0 Conformant** when it:

1. is **ETDL Supplement-Aware** (core Section 2.3);
2. recognizes `etdl.security` at version `1.0` (MAJOR `1`), and enforces
   its declared dependency on `etdl.tree-event` (core Section 5.1.4);
3. parses and validates every Threat Model Object and Control Object under
   a declaring document's `x-security` against this supplement's schema
   (`supplements/security/ETDL-Security-Schema.json`) and Section 4 below;
4. enforces every `MUST`-level rule in Section 4 using the diagnostic
   codes of Section 5.

Per core Section 11.4, `etdl.security` is a **core supplement**. This does
not change any Conforming Parser/Compiler's obligations (core Section
2.3) — no conformance target requires implementing this supplement, and an
implementation MAY implement `etdl.tree-event` without `etdl.security`.

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Threat Model** | A Threat Model Object (Section 4.1): a `etdl.tree-event` Tree reinterpreted as an attack tree, with a STRIDE category assigned to each leaf that has one. |
| **STRIDE Category** | One of `spoofing`, `tampering`, `repudiation`, `information-disclosure`, `denial-of-service`, `elevation-of-privilege` — the standard six-category threat classification this supplement adopts without modification. |
| **Control** | A Control Object (Section 4.2): a core Barrier node given a security-control identity and a list of the threats it mitigates. |
| **Framework** | A free-form string naming the control catalog a `controlId` is drawn from (e.g. `"NIST-800-53"`, `"ISO-27001"`) — this specification does not own or validate any such catalog's contents (Section 6). |

---

## 4. Data Model

### 4.1 Threat Model Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-security.threatModels`. |
| `treeRef` | string | REQUIRED | The `id` of a Tree Object (`etdl.tree-event` Section 4.1) declared in this document's `x-tree-event.trees`. |
| `leafCategories` | Map[string, STRIDE Category] | REQUIRED | Keys are leaf node ids (`etdl.tree-event` Section 4.2) of the referenced Tree; not every leaf needs an entry — an uncategorized leaf is a structural node with no assigned threat category, which this supplement does not treat as an error. |

### 4.2 Control Object

| Field | Type | Requirement | Description |
|---|---|---|---|
| `id` | string | REQUIRED | Unique within `x-security.controls`. |
| `nodeRef` | Internal Reference | REQUIRED | Resolves to `#/eventTrees/<id>/nodes/<barrier-id>` (a Barrier node, core Section 5.8). |
| `framework` | string | OPTIONAL | The control catalog `controlId` is drawn from (Section 3). |
| `controlId` | string | REQUIRED | The catalog identifier within `framework` (e.g. `"SC-5"`). |
| `mitigates` | Array[string] | REQUIRED, non-empty | Leaf node ids (from some Threat Model's `treeRef` tree) this control mitigates. |

---

## 5. Validation Rules

| Code | Rule |
|---|---|
| E-140 | `x-security` is present but a Threat Model Object's `leafCategories` value is not one of Section 3's six STRIDE categories, or `treeRef` does not name an `id` present in this document's `x-tree-event.trees` (`etdl.tree-event` Section 4.1). |
| E-141 | A `leafCategories` key, or a Control Object's `mitigates` entry, is not a leaf node id of the referenced Tree (`etdl.tree-event` Section 4.2); or a Control Object's `nodeRef` does not resolve to a Barrier node. |
| W-411 | A Control Object's `mitigates` list contains a leaf id that no declared Threat Model's `leafCategories` assigns a category to — a control mapped against an uncategorized threat. |

`E-140`, `E-141`, and `W-411` are scoped to this supplement's own
namespace of meaning; they do not collide with core Section 7's codes,
`etdl.tree-event`'s `E-12x` codes, or any other supplement's codes.

---

## 6. Scope and Non-Goals

This supplement records *that* a control is claimed to mitigate a threat;
it does not verify the claim is true, does not validate `controlId` against
the named `framework`'s actual catalog (this specification is not the
authority for NIST 800-53, ISO 27001, or any other external standard), and
performs no automated, formal, or AI-assisted threat analysis of its own.
An attack tree's structural soundness (no cycles, correct gate arity) is
already fully guaranteed by `etdl.tree-event`'s own validation (its Section
5); this supplement adds classification metadata on top of an
already-valid tree, nothing more.

---

## 7. Compatibility

Silently ignoring `x-security` (core Section 11.1's baseline behavior)
leaves a document fully valid under core and `etdl.tree-event` alone —
threat classification and control mapping are additive metadata, never a
precondition for parsing, validation, or code generation of the underlying
tree or event tree.

A future `1.x` MINOR of this supplement may add fields to the Threat Model
or Control Object; it MUST NOT change the meaning of the six STRIDE
categories in Section 3 without a MAJOR bump (core Section 5.1.3).
