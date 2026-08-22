# ETDL Tree Event Supplement 1.0

**Supplement id:** `etdl.tree-event`
**Version:** 1.0.0
**Profile:** extends ETDL Core 1.0.0
**Category:** Core (Section 11.4 of the core specification)
**Status:** Under Development — NOT YET RELEASED
**Normative dependency:** ETDL Core 1.0.0 (the supplement mechanism, Section 5.1.1 of the core specification)

> **This supplement defines a domain-neutral tree-of-events structure: nodes,
> logical gates, structural validation, and deterministic traversal. It
> defines no probability, reliability, failure, or any other domain
> semantics — those belong to a domain that *interprets* a tree this
> supplement validates, never to this supplement itself.**

---

## 1. Relationship to ETDL Core

ETDL Core already defines two tree-shaped structures for one specific purpose each: the Event Tree Object (Section 5.5, a causal sequence toward a Consequence) and the Fault Tree Object (Section 5.11, an IEC 61025 cause structure evaluated to a probability). This supplement defines a third, deliberately more general structure — a tree of named nodes combined through logical gates, with no probability, causal, or reliability meaning attached to any node or gate by this supplement itself.

**This supplement does not depend on the Reliability Supplement** (`etdl.reliability`, Section 12 of that supplement's own document) or on any other domain supplement. A tree declared under this supplement is meaningful — structurally valid or not — independently of whether any consumer ever interprets it as a fault tree, a safety case, an attack tree, or anything else. A domain wishing to give tree nodes a probability, a failure mode, or any other domain meaning does so as a **separate, consuming layer** that reads a validated `Tree` and produces its own derived result; it does not modify this supplement's schema or validation rules to do so (Section 11.2 of the core specification: a supplement's output is a derived view, and no supplement depends on another supplement's output).

**In-document representation.** Tree data is carried in an `x-tree-event` extension field (the `x-*` convention of core Section 11.1) at the document root:

```yaml
x-tree-event:
  trees:
    - schema: "etdl.tree-event/1.0"
      id: "payment-gateway-availability"
      version: "1"
      root: "GatewayUnavailable"
      nodes:
        NetworkTimeout:
          kind: leaf
          eventRef: "std.events.NetworkTimeout"
        DatabaseFailure:
          kind: leaf
          description: "the database did not respond"
        GatewayUnavailable:
          kind: gate
          gate: OR
          children: ["NetworkTimeout", "DatabaseFailure"]
```

`x-tree-event.trees` is an array; a document MAY declare more than one tree. Unlike the Reliability Supplement (Section 1 of that document), this supplement defines no standalone, data-only document form — a tree is always metadata attached to an ordinary ETDL document, never the whole of one, since a tree has no meaning as a top-level artifact independent of *some* consuming context declaring what it represents.

---

## 2. Conformance

An implementation is **ETDL Tree Event 1.0 Conformant** (per the general pattern core Section 2.3 establishes for named-supplement conformance) when it:

1. is **ETDL Supplement-Aware** (core Section 2.3);
2. recognizes `etdl.tree-event` at version `1.0` (MAJOR `1`);
3. parses and validates every Tree Object under a declaring document's `x-tree-event.trees` against this supplement's schema (`supplements/tree-event/ETDL-Tree-Event-Schema.json`) and Section 4–5 below;
4. enforces every `MUST`-level structural rule in Section 5, using the diagnostic codes of Section 8;
5. implements the traversal operations of Section 6 with the ordering guarantees stated there.

This supplement, per core Section 11.4, is a **core supplement**: a Conforming Compiler implementing supplement conformance at all is expected to implement this one, since it is domain-neutral infrastructure other core supplements (including the Reliability Supplement) and future ones build on, not an optional add-on comparable to a non-core supplement (core Section 5.1.2's reserved `etdl.chain`, for example).

---

## 3. Terminology

| Term | Definition |
|---|---|
| **Tree** | The Tree Object defined in Section 4: a root, a set of named nodes, and the schema/version/identity metadata that identify it. |
| **Node** | A single entry in a Tree's `nodes` map — either a Leaf or a Gate (Section 4.2). |
| **Leaf** | A node with no children, optionally carrying a stable reference (`eventRef`) to something external this supplement does not interpret. |
| **Gate** | A node that combines its children through a Gate Kind (Section 4.3). |
| **Gate Kind** | One of `AND`, `OR`, `NOT`, `XOR`, `K_OF_N` (Section 4.3) — a structural label for how a gate's children combine, carrying no probability or evaluation semantics of its own. |
| **Root** | The single node from which every other node in a Tree must be reachable (Section 5.5). |
| **Tree, not DAG** | This supplement's node graph requires each non-root node to have exactly one parent (Section 5.4) — a design decision distinguishing it from a general directed acyclic graph, explained in Section 5.4. |

---

## 4. Tree Object

### 4.1 Fixed Fields

| Field | Type | Requirement | Description |
|---|---|---|---|
| `schema` | string | REQUIRED | MUST be `"etdl.tree-event/1.0"` for this version. |
| `id` | string | REQUIRED | Identifier for this tree, unique within the declaring document's `x-tree-event.trees` array (Section 8, E-122). |
| `version` | string | REQUIRED | The tree author's own version for this tree's content — independent of `schema`'s supplement-schema version and of the document's own `info.version` (core Section 5.2). |
| `root` | string | REQUIRED | The node id (a key of `nodes`) every other node must be reachable from (Section 5.5). |
| `nodes` | Map[string, Node Object] | REQUIRED | Every node in the tree, keyed by node id. |
| `description` | string | OPTIONAL | |
| `metadata` | Map[string, string] | OPTIONAL | Free-form string metadata, not interpreted by this supplement. |

**Node identity is the map key.** A Node Object carries no `id` field of its own; a node's identity is exclusively the key it is stored under in `nodes`. This mirrors the identity convention ETDL Core already uses for `FaultTree.basicEvents` and `FaultTree.gates` (Section 5.11) — identity is never array position and never duplicated onto the value.

### 4.2 Node Object

A Node Object is one of two shapes, discriminated by its `kind` field:

**Leaf** (`kind: leaf`):

| Field | Type | Requirement | Description |
|---|---|---|---|
| `kind` | `"leaf"` | REQUIRED | |
| `eventRef` | string | OPTIONAL | A stable reference to something external — for example a `std.events`-qualified id (core Section 5.1.7), or any other identifier meaningful to a consumer of this tree. This supplement stores and validates `eventRef` only as an opaque string; it does not resolve, dereference, or assign meaning to it. |
| `description` | string | OPTIONAL | |
| `status` | string | OPTIONAL | One of `discovered`, `candidate`, `accepted`, `rejected` (Section 7). |
| `metadata` | Map[string, string] | OPTIONAL | |

**Gate** (`kind: gate`):

| Field | Type | Requirement | Description |
|---|---|---|---|
| `kind` | `"gate"` | REQUIRED | |
| `gate` | Gate Kind (Section 4.3) | REQUIRED | |
| `children` | Array[string] | REQUIRED | Node ids of this gate's inputs, in declaration order. |
| `description` | string | OPTIONAL | |
| `status` | string | OPTIONAL | Section 7. |
| `metadata` | Map[string, string] | OPTIONAL | |

### 4.3 Gate Kind and arity

| Gate Kind | Arity requirement | Meaning (structural label only — Section 1) |
|---|---|---|
| `AND` | ≥ 2 children | All children combined. |
| `OR` | ≥ 2 children | Any child combined. |
| `NOT` | exactly 1 child | Complement of the single child. |
| `XOR` | exactly 2 children | Exactly one of the two children. |
| `K_OF_N` | ≥ 1 children, with an explicit `k` satisfying `1 ≤ k ≤ n` (`n = len(children)`) | At least `k` of the `n` children. |

`K_OF_N`'s `k` is carried as part of the Gate Kind value (e.g. `{ gate: K_OF_N, k: 2, children: [...] }`); a Gate Kind other than `K_OF_N` carries no additional parameter.

A gate whose `children` violates its Gate Kind's arity requirement, or a `K_OF_N` gate whose `k` does not satisfy `1 ≤ k ≤ n`, MUST be reported per Section 8 (E-121) and MUST NOT be treated as a smaller- or larger-arity gate of a different kind.

---

## 5. Structural Validation Rules

A Conforming Tree Event 1.0 implementation MUST validate every declared Tree Object against all of the following rules, and MUST report every violation found (not merely the first) as a single validation pass produces a complete diagnostic set, mirroring core Section 7's discipline for Event Trees and Fault Trees.

### 5.1 Non-emptiness and root resolution

- A Tree with an empty `nodes` map MUST be rejected.
- A Tree whose `root` is empty or is not a key of `nodes` MUST be rejected.

### 5.2 Child references

- Every id in a Gate node's `children` MUST be a key of the same Tree's `nodes`. A reference to a node id absent from `nodes` MUST be rejected.

### 5.3 Gate arity

- Section 4.3's arity table MUST be enforced for every Gate node.

### 5.4 Tree, not DAG

This supplement's node graph is a **strict tree**: every non-root node MUST have exactly one parent (be referenced as a child by exactly one Gate). A node referenced as a child by more than one Gate MUST be rejected, not silently accepted as an implicit directed acyclic graph (DAG).

This is a deliberate 1.0 design decision, not an oversight: a shared node (the same sub-structure meaning "the same thing happened" in two different gates' evaluations) has genuinely different semantics from two independent copies of an equivalent sub-structure, and which semantics is intended is a **domain** decision (for example, whether a reliability interpretation should apply an independence assumption across the shared node or treat it as a common cause) that this domain-neutral supplement is not positioned to make. A future version of this specification MAY introduce an explicit, opt-in DAG mode with defined shared-node semantics; this version does not, and an implementation MUST NOT treat a shared node as valid on the theory that "DAG" is a reasonable relaxation of "tree" — it is a different structure with different semantics, requiring a different, not-yet-defined, contract.

### 5.5 Reachability

- Every key in `nodes` MUST be reachable from `root` by following Gate `children` edges. A node not reachable from `root` MUST be rejected as orphaned.

### 5.6 Cycles

- The graph formed by `root` and the `children` edges MUST be acyclic. A cycle MUST be rejected, and the reported diagnostic (Section 8, E-121) SHOULD include the cyclic chain of node ids to aid diagnosis.

---

## 6. Traversal

A Conforming Tree Event 1.0 implementation MUST provide the following read-only queries over a validated Tree, each with the stated determinism guarantee. These queries assume a Tree has already passed Section 5's validation; their behavior on an invalid Tree (for example, one containing a cycle) is undefined by this supplement.

| Operation | Result | Determinism |
|---|---|---|
| `children(node)` | The direct children of `node` (empty for a Leaf) | Same order as declared in `children` |
| `leaves()` | Every Leaf node's id | Sorted (lexicographic by node id) |
| `ancestors(node)` | `node`'s ancestor chain, nearest first, root last | Deterministic given the Tree (parentage is unique per Section 5.4) |
| `depth(node)` | `0` for `root`, else the length of `ancestors(node)` | Deterministic |
| `descendants(node)` | Every descendant of `node`, in preorder | Deterministic |
| `preorder()` | Every node from `root`, each node before its children | Deterministic |
| `postorder()` | Every node from `root`, each node after its children | Deterministic |

**Determinism is a conformance requirement, not an implementation detail.** Two Conforming Tree Event 1.0 implementations given the same valid Tree MUST produce identical `preorder()`/`postorder()`/`leaves()` sequences — a consumer (for example, an artifact generator, Section 9) that depends on stable ordering for reproducible output MUST be able to rely on this without additional implementation-specific agreement.

**Stack safety.** An implementation MUST NOT rely on unbounded recursion to compute Section 6's operations in a way that risks exhausting the process call stack on a deep-but-valid Tree; a Tree's depth is bounded only by Section 5's structural rules, not by any depth limit this supplement defines, so an implementation encountering a several-thousand-node-deep valid Tree MUST still complete these operations rather than crash.

---

## 7. Node Status (optional)

A node's `status` field, when present, is one of:

| Value | Meaning |
|---|---|
| `discovered` | Proposed by an automated process (for example, source-code analysis), not yet reviewed. |
| `candidate` | Under engineering review. |
| `accepted` | Reviewed and accepted into the tree's model. |
| `rejected` | Reviewed and rejected — SHOULD NOT be treated as structurally present for a consuming domain's purposes, though this supplement itself does not remove a `rejected` node from validation or traversal. |

This field exists primarily to support integration with a source-code discovery process producing candidate nodes for engineering review (analogous to, but independent of, the Reliability Supplement's discovery-candidate concept, Section 5.2 of that document) before they are treated as an accepted part of a tree's model. This supplement assigns `status` no validation or traversal consequence beyond recording it — a domain consuming the tree decides what, if anything, to do with a `discovered` or `rejected` node.

---

## 8. Diagnostics

This supplement's diagnostics extend the existing `E-1xx` family (the same family core Sections 5.1.2–5.1.3, 5.1.7–5.1.9 already use for supplement- and library-declaration errors) rather than introducing a new prefix, per core Section 11.5's requirement that a supplement's diagnostic codes not collide with the core registry (Appendix B of the core specification) or another supplement's:

| Code | Rule |
|---|---|
| `E-120` | `x-tree-event.trees` is absent from a document declaring this supplement, or a Tree Object's manifest is malformed (fails to parse against Section 4's schema). |
| `E-121` | A Tree Object fails one or more of Section 5's structural validation rules (non-emptiness, root resolution, child references, gate arity, tree-not-DAG, reachability, cycles). One `E-121` diagnostic MUST be reported per distinct violation found. |
| `E-122` | Two or more Tree Objects in the same document's `x-tree-event.trees` declare the same `id`. |

`E-120`/`E-121`/`E-122` are scoped to this supplement's own namespace of meaning (they are not reused by, and do not collide with, core Section 7's `E-1xx` codes, which this supplement's Sections 5–6 do not overlap with).

---

## 9. Relationship to Consuming Domains

This supplement defines structure and validation only. A domain giving tree nodes probability, failure, safety, or any other meaning is a **separate, consuming layer**:

- It reads a validated `Tree` (Section 4–5) as an input.
- It supplies, by whatever mechanism is appropriate to that domain, the additional information a Leaf's `AND`/`OR`/`NOT`/`XOR`/`K_OF_N` combination requires to be evaluated in that domain's terms (for example, a probability value per leaf, an independence or dependence assumption for how a Gate combines them).
- It produces its own result (for example, a probability, per the Reliability Supplement) as its own derived output — never by modifying this supplement's Tree Object, its validation rules, or this document.

This supplement does not define, and is not the normative authority for, any such consuming interpretation, including the Reliability Supplement's own tree-consuming adapter. That interpretation — treating `AND`/`OR` as independent-probability composition, `NOT` as complement, `K_OF_N` as an at-least-`k`-of-`n` probability threshold — is documented as part of the Reliability Supplement (Section 12 of that document), stated there explicitly as *one* possible domain interpretation of a Tree Object, not the only legitimate one.

---

## 10. Compatibility

This supplement's `version` (Section 4.1's `schema` field encodes it as `etdl.tree-event/1.0`) follows the same MAJOR/MINOR/PATCH compatibility rule core Section 5.1.3 already establishes for every supplement: an implementation supporting `etdl.tree-event` at MAJOR `1` MUST accept a Tree Object declaring `schema: "etdl.tree-event/1.0"` and MUST reject a future MAJOR it does not implement.

This supplement introduces no core ETDL semantic change (core Section 5.1.5); a document declaring no `etdl.tree-event` supplement, or declaring it but containing no `x-tree-event` field, is unaffected by this document in every respect.
