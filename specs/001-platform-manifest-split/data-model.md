# Resource Model: Platform Manifest Ownership Split

**Feature**: `001-platform-manifest-split` | **Date**: 2026-08-16

This repo has no application data model — its "entities" are Kubernetes
resources. This document maps the spec's Key Entities to the concrete
resource kinds each produces, their config surface, and the relationships
between them, per `base/` + `overlays/` (Principle IV).

## Namespace

| Field | Value |
|---|---|
| Kind | `Namespace` |
| Count | 4 (`platform`, `dataplane`, `datagen`, `familytree`) |
| Location | `k8s/namespaces/<name>.yaml`, no `base`/`overlays` split (no per-environment variance) |
| Config surface | `metadata.name`, `metadata.labels` (ownership marker only) |
| Relationships | Every Platform Service resource below is scoped `metadata.namespace: platform`. Product namespaces (`dataplane`, `datagen`, `familytree`) are declared here but own no resources in this repo (Principle I) — they exist so product repos have a namespace to deploy *into*. |
| Validation rule | `kubectl apply` MUST be idempotent (Edge Case 3) — plain resource, no admission-time side effects. |

## Platform Service

A Platform Service is the repeating shape used for Kafka, Postgres, and
MinIO. Each instantiation follows the same resource set; per-service
specifics are in the Service Instances table below.

| Field | Value |
|---|---|
| Kinds | `StatefulSet` (Kafka, Postgres) or `Deployment` (MinIO) + `Service` + `ConfigMap` + `Secret` template (where credentials apply) |
| Location | `k8s/platform/<service>/base/` + `k8s/platform/<service>/overlays/{local-minikube,eks-staging,eks-prod}/` |
| Config surface | Image tag (pinned, Principle III), replica count, resource requests/limits, readiness/liveness probe, PVC `storageClassName` + size (stateful services only), ConfigMap key/value pairs, Secret template keys (placeholder values only) |
| Relationships | `Service` selects the `StatefulSet`/`Deployment`'s pod labels within the same namespace. `ConfigMap` and `Secret` are mounted/env-referenced by the `StatefulSet`/`Deployment`. The `Secret` is never listed in the owning `kustomization.yaml`'s `resources:` (Principle V / FR-005) — it exists on disk as `secret.example.yaml` documentation only. |
| Validation rule | Every image reference pinned to an explicit version, no `:latest` (Principle III / FR-006). Every `StatefulSet`/`Deployment` carries explicit `resources.requests`/`limits` and readiness + liveness probes (Manifest & Directory Conventions). |

### Service Instances

| Service | Kind | DNS (in `platform` ns) | Port(s) | Pinned Image | Persistent Storage |
|---|---|---|---|---|---|
| Kafka | `StatefulSet` | `kafka.platform.svc.cluster.local` | `9092` (broker) | `apache/kafka:4.1.2` | Yes — log dir PVC |
| Postgres | `StatefulSet` | `postgres.platform.svc.cluster.local` | `5432` | `postgres:16` | Yes — data dir PVC |
| MinIO | `Deployment` | `minio.platform.svc.cluster.local` | `9000` (S3 API), `9090` (console) | `minio/minio:RELEASE.2024-08-17T01-24-54Z` | Yes — object storage PVC |

## Overlay

| Field | Value |
|---|---|
| Kind | Kustomize overlay (`kustomization.yaml` + patches), not a distinct K8s resource kind |
| Instances | `local-minikube` (built and validated this feature), `eks-staging`/`eks-prod` (carried over from existing `dataPlane/k8s/eks/` content as a starting point, per FR-007 — not validated against a real EKS cluster in this feature) |
| Config surface | Replica count patches, resource sizing patches, `storageClassName` patch (`standard` on minikube vs. the EKS cluster's default class), any environment-specific ConfigMap value overrides |
| Relationships | Each overlay references exactly one service's `base/` via `resources: [../../base]`; overlays never reference another service's or another environment's overlay. |

## State / Lifecycle

None of these resources have an application-level state machine — their
lifecycle is Kubernetes' own (`Pending` → `Running`/`Ready`, or `Terminating`
on delete). The only "transition" the spec cares about is captured as
acceptance scenarios in `spec.md` (pod reaches `Ready`, namespace appears in
`kubectl get ns`) and re-validated in `quickstart.md`.
