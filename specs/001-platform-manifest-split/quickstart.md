# Quickstart: Validating the Platform Manifest Split

**Feature**: `001-platform-manifest-split`

Validates this feature end-to-end on a clean minikube cluster, independent of
any product repo, per each user story's Independent Test and the spec's
Success Criteria. Run in order — later steps assume earlier ones succeeded.

## Prerequisites

- minikube running, `kubectl` context pointed at it
- `kustomize` (or `kubectl` ≥1.14 with built-in `-k` support)
- A MinIO client for the bucket check in Step 4 (`mc`, or `aws s3` pointed at
  the MinIO endpoint) — see `research.md` §4 for why bucket creation is a
  manual validation step, not a manifest resource
- A Postgres client (`psql`, or `kubectl exec` into the pod) for Step 3
- Freshly reset cluster recommended for a true SC-004 check (no leftover
  `platform` namespace state from a prior run)

## Step 1 — Namespaces (US3, SC-001)

```bash
kubectl apply -k k8s/namespaces/
kubectl get ns platform dataplane datagen familytree
```

**Expected**: all four namespaces listed. Re-running the same `apply` MUST
succeed with no error (Edge Case 3).

## Step 2 — Kafka (US1, SC-002)

```bash
kubectl apply -k k8s/platform/kafka/overlays/local-minikube --dry-run=client
kubectl apply -k k8s/platform/kafka/overlays/local-minikube
kubectl wait --for=condition=Ready pod/kafka-0 -n platform --timeout=120s
```

Round-trip check (contract: `k8s/../contracts/kafka.md`):

```bash
kubectl run kafka-client --rm -it --restart=Never -n platform \
  --image=apache/kafka:4.1.2 -- \
  /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server kafka.platform.svc.cluster.local:9092 --topic quickstart-check
# type a message, Ctrl-D, then in a second shell:
kubectl run kafka-client-2 --rm -it --restart=Never -n platform \
  --image=apache/kafka:4.1.2 -- \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka.platform.svkc.cluster.local:9092 --topic quickstart-check --from-beginning
```

**Expected**: `kafka-0` reaches `Ready`; the consumer prints the message the
producer sent.

**Cross-repo check (manual, informational)**: confirm
`dataPlane/k8s/kafka-statefulset.yaml`, `kafka-service.yaml`,
`kafka-configmap.yaml`, and the `dataPlane/k8s/eks/` copies have been removed
in that repo (out of this repo's git history — verified in `dataPlane`'s own
PR, per FR-001/FR-008).

## Step 3 — Postgres (US3, SC-004)

```bash
kubectl apply -k k8s/platform/postgres/overlays/local-minikube --dry-run=client
kubectl create secret generic postgres-credentials -n platform \
  --from-literal=POSTGRES_USER=quickstart \
  --from-literal=POSTGRES_PASSWORD=quickstart \
  --from-literal=POSTGRES_DB=quickstart
kubectl apply -k k8s/platform/postgres/overlays/local-minikube
kubectl wait --for=condition=Ready pod/postgres-0 -n platform --timeout=120s
kubectl run pg-client --rm -it --restart=Never -n platform \
  --image=postgres:16 -- \
  psql "postgresql://quickstart:quickstart@postgres.platform.svc.cluster.local:5432/quickstart" -c '\conninfo'
```

**Expected**: `postgres-0` reaches `Ready`; `psql` connects and prints
connection info.

## Step 4 — MinIO (US2, SC-003)

```bash
kubectl apply -k k8s/platform/minio/overlays/local-minikube --dry-run=client
kubectl create secret generic minio-credentials -n platform \
  --from-literal=MINIO_ROOT_USER=quickstart \
  --from-literal=MINIO_ROOT_PASSWORD=quickstart123
kubectl apply -k k8s/platform/minio/overlays/local-minikube
kubectl wait --for=condition=Available deployment/minio -n platform --timeout=120s
kubectl port-forward -n platform svc/minio 9000:9000 &
mc alias set quickstart http://localhost:9000 quickstart quickstart123
mc mb quickstart/datagen-output
echo "hello" | mc pipe quickstart/datagen-output/check.txt
mc cat quickstart/datagen-output/check.txt
```

**Expected**: MinIO Deployment becomes `Available`; the `datagen-output`
bucket is created and the round-tripped object reads back `hello`.

**Cross-repo check (manual, informational)**: confirm
`datagen_speckit/k8s`'s embedded MinIO Deployment/Service definitions have
been removed in that repo (per FR-002/FR-008, verified in `datagen_speckit`'s
own PR).

## Step 5 — Full-history secret scan (SC-005)

```bash
git log -p -- 'k8s/platform/*/base/secret.example.yaml' 'k8s/platform/*/overlays/*/secret.example.yaml' \
  | grep -Ev 'CHANGEME|example|placeholder|<.*>' \
  | grep -E 'password|secret|key' -i
```

**Expected**: no output — every Secret in history is placeholder-only
(Principle V / FR-005).

## Teardown

```bash
kubectl delete -k k8s/platform/kafka/overlays/local-minikube
kubectl delete -k k8s/platform/postgres/overlays/local-minikube
kubectl delete -k k8s/platform/minio/overlays/local-minikube
kubectl delete secret postgres-credentials minio-credentials -n platform
kubectl delete -k k8s/namespaces/
```
