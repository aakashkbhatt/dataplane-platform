<!--
Sync Impact Report
- Version change: (none — template) → 1.0.0
- Modified principles: n/a (initial ratification)
- Added sections: Core Principles (I–VI), Manifest & Directory Conventions,
  Development Workflow, Governance
- Removed sections: none
- Templates requiring updates:
  - .specify/templates/plan-template.md → ✅ no edit needed (generic
    "Constitution Check" gate already reads this file at plan time)
  - .specify/templates/spec-template.md → ✅ no edit needed (generic
    user-story structure applies to operator-facing infra stories as-is)
  - .specify/templates/tasks-template.md → ✅ no edit needed (generic
    phase/story structure applies; task authors should map
    src/models/services examples to this repo's k8s/ layout per Principle IV)
  - .specify/templates/commands/*.md → ✅ no agent-specific references found
- Follow-up TODOs: none
-->

# platform-infra Constitution

## Core Principles

### I. Build-vs-Pull Ownership Boundary
This repository owns the Kubernetes manifests for things nobody's own
Dockerfile produces: shared, stateful platform services (Postgres, Kafka,
MinIO, Neo4j) plus true cluster-wide resources (Namespace definitions,
StorageClass, the eksctl `cluster.yaml`, and any future cross-cutting
ResourceQuota or NetworkPolicy). If a product's own Dockerfile builds the
image, that product's Deployment, Service, ConfigMap, and Secret template
MUST live in that product's own repository, never here. Rationale: a build
step and its manifest changing together, in the same commit, is what keeps a
code change from ever needing a second repository open; a pulled image has
no such coupling and belongs with the infra that operates it.

### II. Namespace-per-Product Isolation
The cluster is divided into a `platform` namespace and one namespace per
product (`dataplane`, `datagen`, `familytree`, and others as they're added).
Products MAY depend downward on `platform`. `platform` MUST NOT depend on
any product. Products MUST NOT reference each other's namespaces directly.
Rationale: this makes blast radius a namespace-scoped fact enforceable by
`kubectl apply -n`/`delete -n`, not a convention someone has to remember.

### III. Immutable Image References
No manifest in this repository may reference `:latest` or a floating tag.
Platform images (things this repo owns per Principle I) MUST be pinned to an
explicit upstream version (e.g. `postgres:16`, `apache/kafka:4.1.2`).
Product images are referenced only via the tag each product's own overlay
supplies (a git short SHA) — this repository never hardcodes a product's
image tag. Rationale: floating tags make rollout and rollback
non-deterministic; this project has already been bitten by exactly this
(see the `imagePullPolicy` history on dataplane-api).

### IV. Kustomize as the Only Manifest Pattern
Every deployable unit in this repository is expressed as `base/` plus
`overlays/{local-minikube,eks-staging,eks-prod}/`, using Kustomize to patch
the base per environment. Hand-duplicated per-environment folders (copies of
manifests instead of overlays) are NOT permitted — they drift the moment one
copy is edited and the other isn't. Rationale: one pattern, applied
uniformly, is what makes "runs on minikube" vs. "runs on EKS" a flag instead
of a second set of files to keep in sync by hand.

### V. Secrets Are Never Committed With Real Values
Every Secret resource in this repository exists only as `secret.example.yaml`
with placeholder values, and MUST be excluded from `kustomization.yaml`'s
`resources:` list. Real secret values are applied out-of-band
(`kubectl create secret ...`, or a secrets manager) and are never present in
git history at any point. Rationale: matches the pattern already
correctly used in `dataplane-api/k8s/secret.yaml` and
`datagen_speckit/k8s/base/secret.example.yaml` — this just makes it
non-negotiable everywhere.

### VI. Independent Change Cadence
Changes in this repository are infra-only: version bumps, resource limits,
storage sizing, namespace/quota policy. They are reviewed and merged on
their own schedule, decoupled from any product's feature or release
cadence, and MUST NOT bundle an unrelated product-repo change in the same
commit or PR.

## Manifest & Directory Conventions

- Directory shape: `k8s/platform/<service>/{base,overlays/...}` for each
  platform service, plus `k8s/namespaces/` and `k8s/cluster/` (StorageClass,
  eksctl config) at the repository root.
- Every StatefulSet/Deployment in this repo carries explicit `resources.requests`
  and `resources.limits`, and a readiness + liveness probe, matching the
  standard already set by `dataplane-api` and `datagen-triggering-service`.
- ConfigMaps hold non-secret, environment-editable config only; anything
  that would leak credentials, tokens, or PII goes in a Secret (Principle V).

## Development Workflow

- Feature work in this repo follows the Spec Kit flow: `/speckit-specify` →
  `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`, same as
  `dataplane-api` and `datagen_speckit`.
- A change here is validated locally against minikube
  (`kubectl apply -k ... --dry-run=client`, then a real apply to a scratch
  namespace) before being treated as ready for the `eks-staging` overlay.
- No product repository's CI/deploy step may modify a file in this
  repository, and this repository's CI/deploy step may not modify a file in
  a product repository — Principle I is enforced at the repo boundary, not
  just by convention.

## Governance

This constitution supersedes ad hoc practice for this repository. Amendments
require: a proposed diff to this file, an explicit version bump per the
policy below, and an updated Sync Impact Report at the top of this file.
Any exception to a Core Principle must be recorded in the relevant
`specs/<feature>/plan.md`'s Complexity Tracking table with a stated
rationale — silent exceptions are not permitted.

**Versioning policy**: MAJOR for a backward-incompatible principle removal
or redefinition (e.g. abandoning namespace-per-product isolation); MINOR for
a new principle or materially expanded guidance; PATCH for wording or
clarification only.

**Version**: 1.0.0 | **Ratified**: 2026-08-16 | **Last Amended**: 2026-08-16
