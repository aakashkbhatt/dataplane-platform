# Implementation Plan: Platform Manifest Ownership Split

**Branch**: `001-platform-manifest-split` | **Date**: 2026-08-16 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-platform-manifest-split/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

## Summary

Stand up `platform-infra` as the single authoritative home for the three
shared, pulled-image platform services (Kafka, Postgres, MinIO) and the
cluster's four namespaces, replacing the Kafka duplication across
`dataPlane/k8s` + `dataPlane/k8s/eks` and the MinIO definition embedded in
`datagen_speckit/k8s`. Each service is expressed as `base/` +
`overlays/{local-minikube,eks-staging,eks-prod}/` per Kustomize (Principle
IV), with images pinned (Principle III) and Secrets committed only as
placeholder templates (Principle V). This feature defines the manifests and
validates them on minikube; it does not touch any product repo's own
Deployment/Service, and it does not cut dataplane-api's traffic over to the
new in-cluster Postgres (spec Assumptions A1/A2).

## Technical Context

**Language/Version**: N/A — this repo is Kubernetes YAML manifests only, no
application code.

**Primary Dependencies**: Kustomize (bundled with `kubectl` ≥1.14, invoked as
`kubectl apply -k`); no Helm, no controllers/operators introduced by this
feature.

**Storage**: The three platform services *are* the storage/messaging layer
this feature defines (Kafka topics, Postgres database, MinIO object storage)
— see `data-model.md`. No separate application storage exists in this repo.

**Testing**: No unit/integration test suite (manifest-only repo). Validation
is `kubectl apply -k ... --dry-run=client` followed by a real apply to a
scratch namespace on minikube, per the constitution's Development Workflow
section; concrete steps and expected outcomes are in `quickstart.md`.

**Target Platform**: Kubernetes — minikube (validated this feature, per spec
Assumption A3) and EKS via `eks-staging`/`eks-prod` overlays (carried over
from existing content per FR-007, not exercised against a real cluster this
feature).

**Project Type**: Infrastructure / Kubernetes manifest repository (single
`k8s/` tree, no frontend/backend split).

**Performance Goals**: N/A — this feature defines correctness and ownership
(single source of truth, idempotent apply), not throughput/latency targets.
Each service's own performance characteristics (Kafka throughput, Postgres
query latency, MinIO I/O) are unchanged by *where* the manifest lives.

**Constraints**: Every image reference pinned, no `:latest` (Principle III /
FR-006) — see research.md §2 for this repo's major.minor-vs.-dated-release
pinning-granularity policy. Every Secret is a placeholder-only template
excluded from its `kustomization.yaml` (Principle V / FR-005). Zero
modification to any product repo's manifests beyond removing the
now-duplicated Kafka/MinIO definitions (FR-008). No real EKS cluster required
as part of Done (Assumption A3).

**Scale/Scope**: 3 platform services × (`base` + 3 overlays) + 4 Namespace
manifests. No new product-facing surface — consumers are the 3 existing
product repos referenced in the spec.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Result |
|---|---|---|
| I. Build-vs-Pull Ownership Boundary | Only Kafka/Postgres/MinIO (pulled images) + Namespaces are in scope; FR-008 explicitly forbids touching product repos' own Deployment/Service beyond removing the duplicated definitions. | ✅ Pass |
| II. Namespace-per-Product Isolation | FR-004 defines exactly `platform` + the 3 product namespaces; `contracts/namespaces.md` states the no-cross-reference rule; no platform service depends on a product namespace. | ✅ Pass |
| III. Immutable Image References | FR-006 + research.md §1–3 pin Kafka/Postgres/MinIO to explicit tags, no `:latest`. | ✅ Pass |
| IV. Kustomize as the Only Manifest Pattern | FR-007 + Project Structure below: every service is `base/` + `overlays/{local-minikube,eks-staging,eks-prod}/`, no hand-duplicated per-env folders. | ✅ Pass |
| V. Secrets Are Never Committed With Real Values | FR-005 + data-model.md: every Secret is `secret.example.yaml`, excluded from `resources:`; quickstart.md Step 5 scans full history to confirm. | ✅ Pass |
| VI. Independent Change Cadence | This feature is infra-only manifests; FR-008 keeps product-repo changes (removing duplicated definitions) out of this repo's own commits. | ✅ Pass |

No violations — Complexity Tracking table below is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-platform-manifest-split/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md         # Phase 1 output (/speckit-plan command)
├── quickstart.md         # Phase 1 output (/speckit-plan command)
├── contracts/             # Phase 1 output (/speckit-plan command)
│   ├── kafka.md
│   ├── postgres.md
│   ├── minio.md
│   └── namespaces.md
├── checklists/
│   └── requirements.md
└── tasks.md              # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
k8s/
├── namespaces/
│   ├── kustomization.yaml
│   ├── platform.yaml
│   ├── dataplane.yaml
│   ├── datagen.yaml
│   └── familytree.yaml
│
└── platform/
    ├── kafka/
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   ├── statefulset.yaml
    │   │   ├── service.yaml
    │   │   └── configmap.yaml
    │   └── overlays/
    │       ├── local-minikube/
    │       │   └── kustomization.yaml (+ patches)
    │       ├── eks-staging/
    │       │   └── kustomization.yaml (+ patches)
    │       └── eks-prod/
    │           └── kustomization.yaml (+ patches)
    │
    ├── postgres/
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   ├── statefulset.yaml
    │   │   ├── service.yaml
    │   │   └── secret.example.yaml   # placeholder only, NOT in kustomization resources:
    │   └── overlays/
    │       ├── local-minikube/
    │       ├── eks-staging/
    │       └── eks-prod/
    │
    └── minio/
        ├── base/
        │   ├── kustomization.yaml
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   ├── configmap.yaml
        │   └── secret.example.yaml   # placeholder only, NOT in kustomization resources:
        └── overlays/
            ├── local-minikube/
            ├── eks-staging/
            └── eks-prod/
```

**Structure Decision**: Follows the constitution's Manifest & Directory
Conventions verbatim: `k8s/platform/<service>/{base,overlays/...}` per
service, plus `k8s/namespaces/` at the repo root. `k8s/cluster/`
(StorageClass, eksctl `cluster.yaml`) is named in the constitution as future
cluster-wide scope but is **not** created by this feature — no FR in
`spec.md` calls for a new StorageClass or eksctl config, and Postgres's PVC
only *references* an existing StorageClass by name (research.md §2), so
standing up `k8s/cluster/` is left to whichever future feature actually owns
that resource.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations — table intentionally left empty.
