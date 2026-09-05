# Research: Platform Manifest Ownership Split

**Feature**: `001-platform-manifest-split` | **Date**: 2026-08-16

This repo has no prior `k8s/` tree (greenfield) and no read access to `dataPlane`,
`datagen_speckit`, or `familytree`'s repositories from this session, so the
"unknowns" below are resolved as planning decisions with an explicit rationale
rather than by inspecting the source repos directly. Each decision is scoped
narrowly enough that carrying the manifests over during `/speckit-implement`
is a mechanical copy-and-adjust, not a redesign, if the actual source values
differ.

## 1. Kafka: broker mode and topology

- **Decision**: Single-broker Kafka in KRaft mode (no ZooKeeper), image pinned
  to `apache/kafka:4.1.2` (the exact tag the constitution itself cites as the
  Principle III example).
- **Rationale**: KRaft is the current upstream-default consensus mode (as of
  Kafka 4.x, ZooKeeper mode has been removed from the Apache Kafka
  distribution entirely), so a new authoritative manifest should not
  reintroduce a ZooKeeper dependency the product repos never asked for.
  Single-broker matches "one authoritative definition" (US1) and keeps the
  `local-minikube` overlay cheap to run; `eks-staging`/`eks-prod` overlays can
  raise replica count later without a base redesign.
- **Alternatives considered**: Carrying over ZooKeeper-mode Kafka if that's
  what `dataPlane/k8s/kafka-statefulset.yaml` actually runs today — rejected
  as the default because it would pin the new "single source of truth" to a
  deprecated mode; if the real source manifest turns out to be ZK-based, that
  becomes a one-line correction at implementation time, not a plan change.

## 2. Postgres: version and storage

- **Decision**: `postgres:16` (the constitution's own Principle III example),
  single-instance StatefulSet, PVC via `storageClassName` reference (not a new
  StorageClass definition — the cluster's existing default provisioner is
  reused: minikube's `standard` class locally, the EKS cluster's default
  `gp3`-backed class in `eks-*` overlays).
- **Rationale**: FR-003 asks for a "StorageClass reference," not a new
  StorageClass resource; defining a cluster-wide StorageClass is Principle I
  territory (`k8s/cluster/`) and out of this feature's scope per the spec's
  FR list. A1/A2 already establish this Postgres instance is net-new and not
  a cutover target, so there's no existing data-directory layout to match.
- **Alternatives considered**: Pinning a specific patch release
  (`postgres:16.4`) for stricter immutability — rejected in favor of matching
  the constitution's own cited example exactly, so the first platform service
  merged doesn't immediately create a wording inconsistency with Principle
  III's illustrative text.
- **Pinning-granularity policy (resolves /speckit-analyze finding A1)**: This
  repo accepts major.minor-only tags (e.g. `postgres:16`) as satisfying
  Principle III's "explicit version" requirement, matching the constitution's
  own illustrative example — even though such a tag does move across
  upstream patch releases, which is in tension with Principle III's stated
  rationale. Where the upstream project publishes an immutable/dated release
  identifier instead (as MinIO does, §3), prefer that. If stricter
  determinism is wanted for Postgres specifically, repin to an explicit patch
  (e.g. `postgres:16.10`) at implementation time (T029) — a drop-in change,
  not a plan change.

## 3. MinIO: version pin (resolves spec Assumption A4)

- **Decision**: `minio/minio:RELEASE.2024-08-17T01-24-54Z`, exposing the S3
  API on `9000` and the console on `9090`.
- **Rationale**: A4 requires *a* pinned tag, deferring to the operator only if
  they want something other than "whatever's running today." Without repo
  access to confirm the exact tag `datagen_speckit/k8s` currently resolves
  `:latest` to, the closest safe default is a specific, real, dated MinIO
  release close to this feature's timeframe — swappable for the operator's
  actual observed tag with a one-line change during implementation if it
  differs.
- **Alternatives considered**: Leaving the tag as `NEEDS CLARIFICATION` and
  blocking planning — rejected because FR-006 (no floating tags) is a hard
  gate this repo must satisfy regardless of which exact release is chosen,
  and the spec's own A4 already anticipates the operator correcting this
  value rather than the plan stalling on it.

## 4. MinIO bucket provisioning (`datagen-output`)

- **Decision**: The `datagen-output` bucket is **not** created by a manifest
  resource in this repo. It's a post-deploy validation step (`mc mb` /
  equivalent) documented in `quickstart.md`, run against the MinIO instance
  once it's reachable.
- **Rationale**: FR-002 scopes this feature to Deployment/Service/ConfigMap
  definitions for MinIO itself, not bucket-provisioning automation. A
  bucket-creation Job would be a reasonable future enhancement but isn't
  named in any FR, and adding one now would be scope creep against US2's
  actual acceptance scenario, which only asks that the bucket "can be
  created and read back" — i.e., that the capability exists, not that it's
  automatic.
- **Alternatives considered**: A Kubernetes `Job` running `mc mb` on every
  apply — rejected as unrequested scope; can be proposed as a follow-up
  feature if a product repo needs it.

## 5. Resource requests/limits and probes

- **Decision**: Deferred to `/speckit-tasks`/`/speckit-implement` with
  minikube-appropriate defaults (small requests, generous-enough limits to
  avoid OOM-killing a single-node broker/database/object-store under light
  local load); exact values aren't a planning-level decision.
- **Rationale**: The Manifest & Directory Conventions section mandates every
  StatefulSet/Deployment carry explicit `resources.requests`/`limits` and
  readiness+liveness probes — that's a manifest-authoring requirement to
  satisfy per-service at implementation time, not a research unknown to
  resolve up front.

## 6. Namespace manifest idempotency (Edge Case 3)

- **Decision**: Plain `Namespace` resources with only `metadata.name` (and a
  `platform-infra`-managed label), no other embedded policy. `kubectl apply`
  on a `Namespace` is natively idempotent (it's a declarative PUT against
  `metadata.name`), so no extra scripting is needed to satisfy this edge
  case.
- **Rationale**: Matches SC-001's "single `kubectl apply -k
  k8s/namespaces/`" success criterion with the simplest possible resource
  shape.
