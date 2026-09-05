# Feature Specification: Platform Manifest Ownership Split

**Feature Branch**: `001-platform-manifest-split`

**Created**: 2026-08-16

**Status**: Draft

**Input**: User description: "Stand up platform-infra as the shared-infrastructure repository and perform the ownership split: move the Kubernetes manifests for things nobody's own Dockerfile builds — Kafka, Postgres, and MinIO — into this repo, owned once, under a `platform` namespace. Also define the namespace objects themselves (platform, dataplane, datagen, familytree). Kafka and Postgres manifests are currently duplicated/scattered across dataPlane's own repo (`k8s/` and `k8s/eks/`); MinIO is currently embedded inside datagen_speckit's repo. Product manifests (dataplane-api, datagen-triggering-service, familytree) stay in their own repos — out of scope here."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - One authoritative Kafka definition (Priority: P1)

As the platform operator, I want Kafka defined exactly once in `platform-infra`, so that dataplane-api can depend on a single, versioned source of truth instead of two manifests (`dataPlane/k8s/kafka-statefulset.yaml` and the copy under `dataPlane/k8s/eks/`) that can silently drift apart.

**Why this priority**: This is the one duplication already causing active risk today — the two Kafka manifest copies have no mechanism keeping them in sync, and the next hand-edit to either one goes unnoticed by the other.

**Independent Test**: Apply `platform-infra`'s Kafka overlay to a scratch namespace on minikube; confirm a test producer/consumer can round-trip a message through it, independent of any product repo being touched.

**Acceptance Scenarios**:

1. **Given** a clean minikube cluster, **When** the Kafka base + local-minikube overlay is applied to the `platform` namespace, **Then** a `kafka-0` pod reaches Ready and is reachable at `kafka.platform.svc.cluster.local:9092`.
2. **Given** the Kafka manifest now lives only in `platform-infra`, **When** `dataPlane/k8s/kafka-statefulset.yaml`, `kafka-service.yaml`, `kafka-configmap.yaml`, and the duplicate copies under `dataPlane/k8s/eks/` are removed, **Then** no Kafka resource definition remains in any product repository.

---

### User Story 2 - One authoritative MinIO definition (Priority: P2)

As the platform operator, I want MinIO defined exactly once in `platform-infra` instead of embedded in datagen_speckit's own manifest folder, so that a MinIO configuration change never requires opening the datagen product repository.

**Why this priority**: Second-highest duplication risk, same shape as Kafka, currently affecting one product rather than the shared broker every product will eventually use.

**Independent Test**: Apply `platform-infra`'s MinIO overlay to a scratch namespace; confirm the bucket used by datagen (`datagen-output`) can be created and read back independent of the datagen repo being open or modified.

**Acceptance Scenarios**:

1. **Given** a clean minikube cluster, **When** the MinIO base + local-minikube overlay is applied to the `platform` namespace, **Then** MinIO is reachable at `minio.platform.svc.cluster.local:9000`.
2. **Given** MinIO now lives only in `platform-infra`, **When** the MinIO Deployment/Service definitions are removed from `datagen_speckit/k8s`, **Then** no MinIO resource definition remains in that repository.

---

### User Story 3 - Deliberate namespace and Postgres ownership (Priority: P3)

As the platform operator, I want the cluster's namespaces (`platform`, `dataplane`, `datagen`, `familytree`) and a reusable Postgres manifest defined once in `platform-infra`, so that creating a namespace or standing up a new Postgres-backed service is a reviewed, versioned act rather than an ad hoc `kubectl create` typed by hand.

**Why this priority**: Lower urgency than Kafka/MinIO because it doesn't fix an active duplication, but it closes the gap the constitution's Principle II depends on — today's cluster has no namespace boundaries at all.

**Independent Test**: Apply the four Namespace manifests and the Postgres base + local-minikube overlay to a clean minikube cluster; confirm all four namespaces exist and a fresh Postgres instance in `platform` accepts connections, independent of whether any product is deployed yet.

**Acceptance Scenarios**:

1. **Given** a clean minikube cluster, **When** `k8s/namespaces/` is applied, **Then** `platform`, `dataplane`, `datagen`, and `familytree` all appear in `kubectl get ns`.
2. **Given** a clean `platform` namespace, **When** the Postgres base + local-minikube overlay is applied, **Then** a `postgres-0` pod reaches Ready and accepts connections at `postgres.platform.svc.cluster.local:5432`.

---

### Edge Cases

- What happens to dataplane-api's existing Kafka connection (`kafka.default.svc.cluster.local`) once Kafka moves to the `platform` namespace? It stops resolving. Updating dataplane-api's own `dataplane-config` ConfigMap to the new address is a change to *dataplane-api's own repo*, tracked as a follow-up feature there — out of scope for this spec (Assumption A1 below).
- What happens to the Postgres database dataplane-api currently uses (an external/local container, not an in-cluster StatefulSet today)? This spec only prepares the in-cluster manifest; it does not migrate data or cut traffic over (Assumption A2 below).
- What happens if a namespace already exists (re-applying this feature on a cluster that was partially set up by hand)? Namespace manifests MUST be safe to re-apply (`kubectl apply` is idempotent) without erroring or altering existing resources inside that namespace.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `platform-infra` MUST define a single authoritative Kafka StatefulSet, Service, and ConfigMap, replacing the duplicate definitions currently split across `dataPlane/k8s` and `dataPlane/k8s/eks`.
- **FR-002**: `platform-infra` MUST define a single authoritative MinIO Deployment, Service, and ConfigMap, replacing the definition currently embedded in `datagen_speckit/k8s`.
- **FR-003**: `platform-infra` MUST define a Postgres StatefulSet, Service, Secret template, and StorageClass reference, available for any product namespace to depend on. Standing this up does NOT require migrating dataplane-api's existing data or cutting its traffic over as part of this feature.
- **FR-004**: `platform-infra` MUST define Namespace resources for `platform`, `dataplane`, `datagen`, and `familytree`.
- **FR-005**: Every Secret in `platform-infra` MUST exist only as a template file with placeholder values, and MUST be excluded from its `kustomization.yaml`'s `resources:` list, per the project constitution.
- **FR-006**: Every image reference in `platform-infra` MUST be pinned to an explicit version (no `:latest`, no floating tags).
- **FR-007**: Each platform service (Kafka, MinIO, Postgres) MUST be expressed as `base/` plus at least a `local-minikube` overlay. `eks-staging`/`eks-prod` overlays MAY be carried over from the existing content in `dataPlane/k8s/eks/` as a starting point, without requiring a real EKS cluster to validate against as part of this feature.
- **FR-008**: This feature MUST NOT modify any manifest inside `dataPlane`'s, `datagen_speckit`'s, or `familytree`'s own repositories beyond removing the now-duplicated Kafka/MinIO definitions named in FR-001/FR-002. Updating those products' own DNS/endpoint references to point at the new `platform` namespace is a separate, later follow-up feature in each product's own repo.

### Key Entities

- **Namespace**: A cluster-scoped boundary (`platform`, `dataplane`, `datagen`, `familytree`) that groups related resources and scopes `kubectl` operations.
- **Platform Service**: A shared, stateful dependency this repo owns end-to-end (Kafka, Postgres, MinIO) — expressed as a StatefulSet or Deployment, a Service, a ConfigMap, and (where credentials are needed) a Secret template.
- **Overlay**: An environment-specific Kustomize patch (`local-minikube`, `eks-staging`, `eks-prod`) applied on top of a service's `base/` manifest set.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All four namespace objects exist in the cluster after a single `kubectl apply -k k8s/namespaces/`.
- **SC-002**: Exactly one copy of Kafka's manifest set exists across every repository the operator owns (down from two today).
- **SC-003**: Exactly zero MinIO resource definitions remain in `datagen_speckit`'s repository.
- **SC-004**: `kubectl apply -k` of each platform service's `local-minikube` overlay succeeds with zero manual post-apply edits, on a freshly reset minikube cluster.
- **SC-005**: A full-history secret scan of `platform-infra` finds zero non-placeholder credential values.

## Assumptions

- **A1**: Updating each product's own manifests to point at the new `platform`-namespace DNS names (`kafka.platform.svc.cluster.local`, `minio.platform.svc.cluster.local`, `postgres.platform.svc.cluster.local`) is out of scope here and tracked as a follow-up feature in each product's own repository, per the constitution's ownership boundary (Principle I).
- **A2**: Postgres is newly defined in `platform-infra` by this feature, but dataplane-api keeps pointing at its existing external Postgres container until a separate, deliberate data-migration feature — this feature prepares the manifest, it does not cut traffic over.
- **A3**: minikube remains the only environment this feature is validated against; `eks-staging`/`eks-prod` overlays are carried forward from existing content as a starting point but are not exercised against a real EKS cluster as part of this feature's Done criteria.
- **A4**: MinIO's version is pinned to whatever tag is already running today (`minio/minio:latest` observed in the current image cache) unless the operator specifies otherwise during planning — this spec does not choose a new version, it only requires *a* pinned one (FR-006).
