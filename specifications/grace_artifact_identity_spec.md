# Artifact Identity Specification

**Version:** 1.0
**Status:** Stable
**Document Type:** Public Specification

---

# Purpose

This document defines artifact identity, canonical state, version lineage, realization semantics, reconciliation rules, and persistent artifact governance for Grace.

It defines how artifacts exist, evolve, converge, and remain identifiable across runtimes and storage systems.

For the literal JSON schema implementing this model, refer to:

```text
skill/references/schemas.md
```

For how this model is enforced at runtime, refer to:

```text
specifications/grace_governance_core_spec.md
```

---

# Core Concepts

The artifact identity model is built from eight core concepts:

* Artifact
* Identity
* Canonical State
* Version Lineage
* Realization State
* Registry
* Commit
* Reconciliation

---

# Artifact

An artifact is any persistent representation of work that Grace governs — a document, a piece of source code, an architecture, a specification, a decision, a configuration, or a repository.

Artifacts are the primary unit of realization. Grace does not optimize conversations; it optimizes artifacts.

---

# Artifact Identity

Each artifact possesses:

* `artifact_id`
* `type`
* `owner_domain`
* `timestamps` (created, updated)
* `lineage_ref`
* `canonical_flag`

Identity is assigned once, at `CREATE`, and never reassigned. An artifact's identity is the one property of an artifact that is never allowed to change across its entire lifecycle.

---

# Canonical Artifact Principle

Only one canonical artifact may exist per artifact identity at any given moment.

Multiple drafts, multiple experiments, and multiple proposed revisions may coexist for the same `artifact_id`. Exactly one of them — at most — may carry `canonical_flag: true` at any instant.

---

# Lifecycle

An artifact moves through the following lifecycle states:

```text
Draft → Active → Canonical → Superseded → Archived
```

* **Draft** — newly created, not yet validated.
* **Active** — undergoing validation or in-progress realization.
* **Canonical** — the current authoritative representation of this artifact identity.
* **Superseded** — was canonical, replaced by a newer canonical version; retained for lineage.
* **Archived** — retired from active consideration; no longer eligible to become canonical again.

A `Superseded` artifact is never deleted. It remains queryable as part of the artifact's lineage.

---

# Version Lineage

Artifacts form lineage graphs while preserving continuity and traceability.

Every artifact possesses an origin, a history of mutations, relationships to prior versions, and ownership. Lineage allows work to remain understandable over time — without it, artifacts degrade into isolated snapshots with no relationship to what came before them.

A lineage graph is directional: each version points back to the version it replaced via `lineage_ref`, forming a chain that can be walked from the current canonical artifact all the way back to its `CREATE` event.

---

# State Realization

Orthogonal to the lifecycle above, an artifact also carries a realization state describing its progress toward canonicalization within the current cycle:

```text
Planned → In Progress → Verified → Canonicalized → Archived | Deprecated
```

Lifecycle describes *what the artifact currently is*. Realization state describes *where the current cycle working on it has gotten to*. An artifact can be `Active` (lifecycle) while its realization state is `In Progress`, and only becomes `Canonical` (lifecycle) once its realization state reaches `Canonicalized`.

---

# Commit Semantics

Every mutation to an artifact's canonical state is expressed as exactly one of five operations:

| Operation | Meaning |
|---|---|
| `CREATE` | New artifact identity established. |
| `REPLACE` | Canonical state updated; identity unchanged. |
| `MERGE` | Multiple revisions consolidated into one canonical state. |
| `COLLAPSE` | Multiple candidate states reduced to the single surviving canonical state. |
| `ARCHIVE` | Artifact retired; no longer eligible for canonical status. |

No other mutation path exists. An operation that does not fit one of these five categories is not a valid Grace commit.

---

# Replace Semantics

Canonical state may be updated without changing artifact identity.

A `REPLACE` operation produces a new version of the same `artifact_id`; it never produces a new `artifact_id`. The previous canonical version transitions to `Superseded` as part of the same operation — there is no intermediate state where two versions are simultaneously canonical.

---

# Version Collapse

Multiple revisions may be consolidated into a single canonical state through `MERGE` or `COLLAPSE`.

`MERGE` is used when multiple legitimate, non-conflicting revisions need to be combined into one. `COLLAPSE` is used when multiple candidate states were competing for canonical status and exactly one must win. The distinction matters for lineage: a `MERGE` records all contributing versions as ancestors; a `COLLAPSE` records the losing candidates as superseded, not as ancestors.

---

# Reconciliation Protocol

Reconciliation resolves divergence between an artifact's logical identity, as Grace understands it, and its storage-layer representation, as reported by Workspace or an external MCP.

A reconciliation cycle:

1. Compares `expected_state` (from Workspace) against `observed_state` (from the external system).
2. Classifies the `divergence_type`.
3. Selects a `resolution_action`: `retry`, `rollback`, `re_converge`, or `escalate`.
4. Marks `resolved: true` only once the canonical state and the observed state agree.

Reconciliation never resolves a divergence by silently preferring whichever state was reported most recently. It resolves toward whichever state satisfies the system's invariants, as arbitrated by Governance Core.

---

# Artifact Registry

The registry is responsible for:

* Identity resolution — given an `artifact_id`, what currently exists?
* Canonical lookup — given an `artifact_id`, what is the current canonical version?
* Lineage tracking — given an `artifact_id`, what is its full version history?
* Reconciliation records — given an `artifact_id`, what divergences have been detected and resolved against it?

The registry is the only component permitted to answer "what is canonical right now" with authority. Any other component's belief about canonical state is a cached opinion, not a source of truth.

---

# Workspace Integration

Artifacts are first-class workspace entities, shared across systems.

An artifact does not belong to the cycle that created it — once committed, it belongs to the workspace, and any future cycle, by Etta or by Grace, may reference it.

---

# Etta Interface

Etta may reference artifacts but does not govern canonicality.

Etta can read an artifact's canonical state to inform its own reasoning, and can propose that an artifact be revised. Etta cannot mark an artifact canonical, supersede it, or archive it — those operations belong exclusively to Grace.

---

# Grace Authority

Grace governs canonical state, replacement semantics, lineage, and realization.

This authority is exclusive: no other component in the ecosystem, including Etta, may directly mutate an artifact's lifecycle state, realization state, or canonical flag.

---

# MCP Mapping

CRUD systems are translated into Grace semantic artifact operations.

An external MCP typically only understands create, read, update, and delete. Grace's `ArtifactManager` is responsible for translating its five commit semantics (`CREATE`, `REPLACE`, `MERGE`, `COLLAPSE`, `ARCHIVE`) down into whatever CRUD primitives the underlying MCP actually exposes, while preserving the semantic guarantees described in this document on the Grace side of that boundary.

---

# Normative Principles

The artifact identity model rests on three irreducible statements:

* No identity, no governance.
* No canonical state, no reality.
* No lineage, no integrity.

An implementation that cannot guarantee artifact identity, canonical uniqueness, and lineage traceability is not implementing Grace's artifact model, regardless of what else it does correctly.
