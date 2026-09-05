
---

description: "Task list template for feature implementation"
---

# Tasks: Platform Manifest Ownership Split

**Input**: Design documents from `/specs/001-platform-manifest-split/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md (all present)

**Tests**: No automated test suite is requested in `spec.md` — this is a manifest-only repo. Each story's "Independent Test" from `spec.md` is executed via the matching `quickstart.md` step instead of a code test, and is captured below as a validation task at the end of each story's phase.

**Organization**: Tasks are grouped by user story (spec.md priorities P1/P2/P3) to enable independent implementation and validation of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on another incomplete task in this list)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- File paths are exact and relative to the repository root

## Path Conventions

Single manifest tree at the repo root, per `plan.md`'s Project Structure:
`k8s/namespaces/`, `k8s/platform/{kafka,postgres,minio}/{base,overlays/{local-minikube,eks-staging,eks-prod}}/`.

---

## Phase 1: Setup

**Purpose**: Scaffold the directory tree every later task writes into.

- [X] T001 Create the empty directory skeleton: `k8s/namespaces/`, and for each of `kafka`, `postgres`, `minio` under `k8s/platform/<service>/`: `base/` and `overlays/{local-minikube,eks-staging,eks-prod}/` — matches `plan.md`'s Project Structure exactly

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Namespace objects (FR-004) are the one dependency every platform service's manifest references directly (`metadata.namespace: platform`) and every product repo's own manifests will target — they block a real end-to-end deploy of any story, even though each story's *Independent Test* can substitute an ad hoc scratch namespace to stay decoupled during development.

**⚠️ CRITICAL**: Required for a real cluster deploy of any story; not required to run each story's own Independent Test (which uses a scratch namespace per `spec.md`).

- [X] T002 [P] Create `Namespace` manifest for `platform` in `k8s/namespaces/platform.yaml`
- [X] T003 [P] Create `Namespace` manifest for `dataplane` in `k8s/namespaces/dataplane.yaml`
- [X] T004 [P] Create `Namespace` manifest for `datagen` in `k8s/namespaces/datagen.yaml`
- [X] T005 [P] Create `Namespace` manifest for `familytree` in `k8s/namespaces/familytree.yaml`
- [X] T006 Create `k8s/namespaces/kustomization.yaml` listing all four namespace manifests from T002-T005 (depends on T002-T005)
- [X] T007 Validate `kubectl apply -k k8s/namespaces/ --dry-run=client` succeeds and re-running the real apply is a no-op error (quickstart.md Step 1, Edge Case 3) (depends on T006)

**Checkpoint**: Namespaces exist — every user story below can now target `namespace: platform` for real (SC-001).

---

## Phase 3: User Story 1 - One authoritative Kafka definition (Priority: P1) 🎯 MVP

**Goal**: A single, versioned Kafka StatefulSet/Service/ConfigMap in `platform-infra`, replacing the two duplicate copies in `dataPlane`.

**Independent Test**: Apply the Kafka base + `local-minikube` overlay to a scratch namespace on minikube; confirm a producer/consumer round-trips a message, independent of any product repo.

### Implementation for User Story 1

- [X] T008 [P] [US1] Create Kafka base ConfigMap (KRaft cluster ID, broker/listener config) in `k8s/platform/kafka/base/configmap.yaml`
- [X] T009 [P] [US1] Create Kafka base Service (port `9092`, headless for StatefulSet DNS) in `k8s/platform/kafka/base/service.yaml`
- [X] T010 [US1] Create Kafka base StatefulSet in `k8s/platform/kafka/base/statefulset.yaml` — image pinned `apache/kafka:4.1.2` (research.md §1), KRaft single-node, explicit `resources.requests`/`limits`, readiness+liveness probes, PVC template for the log dir with a patchable `storageClassName` (depends on T008, T009)
- [X] T011 [US1] Create `k8s/platform/kafka/base/kustomization.yaml` listing the ConfigMap, Service, and StatefulSet from T008-T010 (depends on T008, T009, T010)
- [X] T012 [P] [US1] Create the `local-minikube` overlay in `k8s/platform/kafka/overlays/local-minikube/kustomization.yaml` (+ patches: `storageClassName: standard`, single replica, minikube-sized resources) (depends on T011)
- [X] T013 [P] [US1] Create the `eks-staging` overlay in `k8s/platform/kafka/overlays/eks-staging/kustomization.yaml`, carried over from `dataPlane/k8s/eks/` as a starting point per FR-007 (depends on T011)
- [X] T014 [P] [US1] Create the `eks-prod` overlay in `k8s/platform/kafka/overlays/eks-prod/kustomization.yaml`, carried over from `dataPlane/k8s/eks/` as a starting point per FR-007 (depends on T011)
- [X] T015 [US1] Run `quickstart.md` Step 2 on minikube: apply the `local-minikube` overlay, confirm `kafka-0` reaches `Ready`, round-trip a message through `kafka.platform.svc.cluster.local:9092` (depends on T012)
- [ ] T016 [US1] Cross-repo: remove `dataPlane/k8s/kafka-statefulset.yaml`, `kafka-service.yaml`, `kafka-configmap.yaml`, and the duplicate copies under `dataPlane/k8s/eks/` in the `dataPlane` checkout — the only change this feature makes outside `platform-infra` (FR-001/FR-008), performed once T015 confirms the new manifests work (depends on T015)

**Checkpoint**: Kafka is defined exactly once, validated on minikube (SC-002).

---

## Phase 4: User Story 2 - One authoritative MinIO definition (Priority: P2)

**Goal**: A single, versioned MinIO Deployment/Service/ConfigMap in `platform-infra`, replacing the copy embedded in `datagen_speckit`.

**Independent Test**: Apply the MinIO base + `local-minikube` overlay to a scratch namespace; confirm the `datagen-output` bucket can be created and read back, independent of the `datagen_speckit` repo.

### Implementation for User Story 2

- [X] T017 [P] [US2] Create MinIO base ConfigMap (non-secret server config) in `k8s/platform/minio/base/configmap.yaml`
- [X] T018 [P] [US2] Create MinIO base Service (S3 API port `9000`, console port `9090`) in `k8s/platform/minio/base/service.yaml`
- [X] T019 [P] [US2] Create MinIO base Secret template with `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` placeholders in `k8s/platform/minio/base/secret.example.yaml` (Principle V)
- [X] T020 [US2] Create MinIO base Deployment in `k8s/platform/minio/base/deployment.yaml` — image pinned `minio/minio:RELEASE.2024-08-17T01-24-54Z` (research.md §3), explicit `resources.requests`/`limits`, readiness+liveness probes, PVC for object storage with a patchable `storageClassName` (depends on T017, T018, T019)
- [X] T021 [US2] Create `k8s/platform/minio/base/kustomization.yaml` listing the ConfigMap, Service, and Deployment from T017-T020 — `secret.example.yaml` from T019 MUST NOT appear in `resources:` (FR-005) (depends on T017, T018, T019, T020)
- [X] T022 [P] [US2] Create the `local-minikube` overlay in `k8s/platform/minio/overlays/local-minikube/kustomization.yaml` (+ patches: `storageClassName: standard`, minikube-sized resources) (depends on T021)
- [X] T023 [P] [US2] Create the `eks-staging` overlay in `k8s/platform/minio/overlays/eks-staging/kustomization.yaml`, carried over as a starting point per FR-007 (depends on T021)
- [X] T024 [P] [US2] Create the `eks-prod` overlay in `k8s/platform/minio/overlays/eks-prod/kustomization.yaml`, carried over as a starting point per FR-007 (depends on T021)
- [ ] T025 [US2] Run `quickstart.md` Step 4 on minikube: apply the `local-minikube` overlay, confirm the Deployment reaches `Available`, create the `datagen-output` bucket and read an object back (depends on T022)
- [ ] T026 [US2] Cross-repo: remove the MinIO Deployment/Service definitions from `datagen_speckit/k8s` in the `datagen_speckit` checkout — the only change this feature makes outside `platform-infra` for MinIO (FR-002/FR-008), performed once T025 confirms the new manifests work (depends on T025)

**Checkpoint**: MinIO is defined exactly once, validated on minikube (SC-003).

---

## Phase 5: User Story 3 - Deliberate namespace and Postgres ownership (Priority: P3)

**Goal**: Namespaces (delivered in Phase 2) plus a reusable Postgres StatefulSet/Service/Secret template, so standing up a namespace or a Postgres-backed service is a reviewed, versioned act.

**Independent Test**: Apply the four Namespace manifests and the Postgres base + `local-minikube` overlay to a clean minikube cluster; confirm all four namespaces exist and a fresh Postgres instance accepts connections, independent of any product being deployed.

### Implementation for User Story 3

- [X] T027 [P] [US3] Create Postgres base Secret template with `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` placeholders in `k8s/platform/postgres/base/secret.example.yaml` (Principle V)
- [X] T028 [P] [US3] Create Postgres base Service (port `5432`) in `k8s/platform/postgres/base/service.yaml`
- [X] T029 [US3] Create Postgres base StatefulSet in `k8s/platform/postgres/base/statefulset.yaml` — image pinned `postgres:16` (research.md §2), explicit `resources.requests`/`limits`, readiness+liveness probes, PVC for the data dir with a patchable `storageClassName` (depends on T027, T028)
- [X] T030 [US3] Create `k8s/platform/postgres/base/kustomization.yaml` listing the Service and StatefulSet from T028-T029 — `secret.example.yaml` from T027 MUST NOT appear in `resources:` (FR-005) (depends on T027, T028, T029)
- [X] T031 [P] [US3] Create the `local-minikube` overlay in `k8s/platform/postgres/overlays/local-minikube/kustomization.yaml` (+ patches: `storageClassName: standard`, minikube-sized resources) (depends on T030)
- [X] T032 [P] [US3] Create the `eks-staging` overlay in `k8s/platform/postgres/overlays/eks-staging/kustomization.yaml`, carried over as a starting point per FR-007 (depends on T030)
- [X] T033 [P] [US3] Create the `eks-prod` overlay in `k8s/platform/postgres/overlays/eks-prod/kustomization.yaml`, carried over as a starting point per FR-007 (depends on T030)
- [X] T034 [US3] Run `quickstart.md` Steps 1 and 3 on a clean minikube cluster: confirm all four namespaces appear in `kubectl get ns`, apply the Postgres `local-minikube` overlay, confirm `postgres-0` reaches `Ready` and accepts connections at `postgres.platform.svc.cluster.local:5432` (depends on T031, T007)

**Checkpoint**: All three platform services and all four namespaces are defined, applied, and independently validated (SC-004).

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Repo-wide guarantees that span all three services, not owned by any single story.

- [ ] T035 [P] Run `quickstart.md` Step 5: full-history secret scan across every `k8s/platform/*/base/secret.example.yaml`, confirm zero non-placeholder credential values (SC-005) (depends on T021, T030)
- [X] T036 [P] Audit every image reference under `k8s/platform/` and `k8s/namespaces/` for an explicit pinned tag — zero `:latest` or floating tags (FR-006, Principle III) (depends on T011, T021, T030)
- [ ] T037 Run the full `quickstart.md` sequence end-to-end (Steps 1-4 + teardown, then reapply) on a freshly reset minikube cluster to confirm `kubectl apply -k` succeeds with zero manual post-apply edits for every service (SC-004) (depends on T015, T025, T034)
- [ ] T038 [P] Update `CLAUDE.md`/repo docs with any operational notes that emerged during implementation (e.g. actual image tags confirmed against `dataPlane`/`datagen_speckit`, if they differed from research.md's defaults)
- [X] T039 [P] Verify `secret.example.yaml` is absent from the `resources:` list in both `k8s/platform/minio/base/kustomization.yaml` and `k8s/platform/postgres/base/kustomization.yaml` (FR-005 structural check, resolves analyze finding C1 — complements T035's value-scan) (depends on T021, T030)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately.
- **Foundational (Phase 2)**: Depends on Setup (T001). Blocks a real cluster deploy of every story (does not block running each story's own scratch-namespace Independent Test).
- **User Stories (Phase 3-5)**: Each depends on Setup; Phase 5's validation task (T034) additionally depends on Foundational (T007) for the real namespace. Stories can otherwise proceed in parallel or in priority order (P1 → P2 → P3).
- **Polish (Phase 6)**: Depends on all three stories' base+kustomization tasks being complete.

### User Story Dependencies

- **User Story 1 (P1 — Kafka)**: No dependency on US2/US3. The only outside-repo touch is T016.
- **User Story 2 (P2 — MinIO)**: No dependency on US1/US3. The only outside-repo touch is T026.
- **User Story 3 (P3 — Postgres, namespaces)**: Postgres tasks (T027-T033) have no dependency on US1/US2; final validation (T034) depends on Foundational namespaces (T007), not on US1/US2.

### Within Each Service (Kafka / MinIO / Postgres)

- ConfigMap/Service/Secret template before the StatefulSet/Deployment that references them.
- StatefulSet/Deployment before the base `kustomization.yaml`.
- Base `kustomization.yaml` before any overlay (overlays reference `../../base`).
- Overlay before that story's `quickstart.md` validation task.
- Validation before the cross-repo removal task (never delete the old copy before the new one is confirmed working).

### Parallel Opportunities

- T002-T005 (the four Namespace manifests) run in parallel.
- Within each service: the ConfigMap/Service/Secret-template tasks run in parallel (e.g. T008+T009, T017+T018+T019, T027+T028); the three overlay tasks run in parallel once the base `kustomization.yaml` exists (e.g. T012+T013+T014).
- Once Setup (T001) is done, all three services' implementation phases (Phase 3, 4, 5) can proceed in parallel if staffed — they touch entirely disjoint files.
- T035, T036, T038, T039 in Polish run in parallel; T037 does not (it's the single end-to-end sequence gating everything else in that phase).

---

## Parallel Example: User Story 1 (Kafka)

```bash
# Launch the independent base-resource tasks together:
Task: "Create Kafka base ConfigMap in k8s/platform/kafka/base/configmap.yaml"
Task: "Create Kafka base Service in k8s/platform/kafka/base/service.yaml"

# After the base kustomization (T011) exists, launch all three overlays together:
Task: "Create local-minikube overlay in k8s/platform/kafka/overlays/local-minikube/kustomization.yaml"
Task: "Create eks-staging overlay in k8s/platform/kafka/overlays/eks-staging/kustomization.yaml"
Task: "Create eks-prod overlay in k8s/platform/kafka/overlays/eks-prod/kustomization.yaml"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (namespaces)
3. Complete Phase 3: User Story 1 (Kafka) — T008 through T016
4. **STOP and VALIDATE**: T015's `quickstart.md` Step 2 round-trip, independently
5. This alone satisfies SC-002 and closes the highest-risk duplication named in the spec

### Incremental Delivery

1. Setup + Foundational → namespaces exist
2. Add User Story 1 (Kafka) → validate → remove `dataPlane`'s duplicates (MVP)
3. Add User Story 2 (MinIO) → validate → remove `datagen_speckit`'s duplicate
4. Add User Story 3 (Postgres) → validate
5. Polish: secret scan, image-pin audit, full end-to-end reapply (SC-004/SC-005)

### Parallel Team Strategy

With three implementers:

1. Together: Setup (T001) + Foundational (T002-T007)
2. Then, in parallel: Developer A → Phase 3 (Kafka), Developer B → Phase 4 (MinIO), Developer C → Phase 5 (Postgres) — no shared files between the three
3. Converge on Phase 6 (Polish) once all three base+kustomization sets exist

---

## Notes

- [P] tasks touch different files with no dependency on another incomplete task in this list.
- [Story] label maps each task to its `spec.md` user story for traceability.
- Every StatefulSet/Deployment task carries the constitution's mandatory `resources.requests`/`limits` + readiness/liveness probes as part of the task, not as a separate follow-up.
- The two cross-repo tasks (T016, T026) are the *only* changes this feature makes outside `platform-infra`, per FR-008 — everything else in this list is a file inside this repo.
- Commit after each task or logical group; stop at any checkpoint to validate a story independently.
