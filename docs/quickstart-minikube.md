# Quickstart: Postgres, Neo4j, and dataplane-api on a remote minikube

Two machines:

| Machine | Role |
|---|---|
| **Build machine** (your Mac) | Builds the dataplane-api image and pushes it to Docker Hub |
| **minikube host** (separate machine) | Runs minikube; pulls the image from Docker Hub and the manifests from GitHub |

On the minikube host there is one cluster. Shared platform services and the
product are kept apart by **namespace** (constitution Principle II):

```text
minikube host
└── minikube cluster
    ├── ns platform            ← platform-infra (this repo)
    │   ├── postgres-0         postgres.platform.svc.cluster.local:5432
    │   └── neo4j-0            bolt://neo4j.platform.svc.cluster.local:7687
    └── ns dataplane           ← dataPlane repo
        └── dataplane-api      image: docker.io/aakashbhatt/dataplane:<git-sha>
```

dataplane-api's ConfigMap and Secret already use those `*.platform.svc`
addresses, so nothing in its manifests changes.

## Prerequisites

**Build machine**
- Docker Desktop with `buildx` (bundled)
- `docker login` done as a Docker Hub account that can push to
  `aakashbhatt/dataplane`
- The dataPlane commit you want to ship, committed. The image tag is its git SHA.

**minikube host**
- Docker, `minikube`, `kubectl`, `git`, and outbound internet (Docker Hub,
  GitHub)
- ≥ 4 CPUs / 5 GB RAM free for minikube
- Its CPU architecture: run `uname -m`. `x86_64` means `linux/amd64`;
  `aarch64` or `arm64` means `linux/arm64`. Your Mac is arm64, so if the
  host is x86_64 the image **must** be built for amd64, or the pods crash
  with `exec format error`.

**Both:** the manifests in this repo (including `k8s/platform/neo4j/`) and in
dataPlane must be pushed to GitHub, since the host clones them from there.

---

## Part 1: Build and push the image (build machine)

Tag with the git SHA, not `:latest`. On the host, the Deployment's
`imagePullPolicy: IfNotPresent` would never re-pull a changed `:latest`, so
you'd keep running the old build (Principle III).

```bash
cd ~/PycharmProjects/dataPlane
git status                                  # commit first; the tag should mean exactly this code
export TAG=$(git rev-parse --short HEAD)
export IMAGE=docker.io/aakashbhatt/dataplane

# Set to the minikube host's arch (see Prerequisites)
export PLATFORM=linux/amd64                 # or linux/arm64

docker buildx build --platform "$PLATFORM" -t "$IMAGE:$TAG" --push .
```

Confirm it's on Docker Hub with the right architecture:

```bash
docker buildx imagetools inspect "$IMAGE:$TAG"
```

<details>
<summary>Both architectures in one image (optional)</summary>

The default `docker` builder can't push a multi-platform image, so create a
`docker-container` builder once:

```bash
docker buildx create --name multi --driver docker-container --use
docker buildx build --platform linux/amd64,linux/arm64 -t "$IMAGE:$TAG" --push .
```

</details>

**Private repo on Docker Hub?** Create an access token (Docker Hub → Account
settings → Personal access tokens, read-only is enough). The host needs it in
Part 3 step 2.

Note `$TAG`; Part 3 needs it.

---

## Part 2: Cluster and platform services (minikube host)

```bash
# 1. Get the manifests
git clone https://github.com/aakashkbhatt/dataplane-platform.git platform-infra
git clone https://github.com/aakashkbhatt/dataplane.git dataPlane
cd platform-infra

# 2. Cluster
minikube start --driver=docker --cpus=4 --memory=5120

# 3. Namespaces: platform, dataplane, datagen, familytree
kubectl apply -k k8s/namespaces

# 4. Credentials (created out-of-band, never committed: Principle V)
export PG_PW='<choose-a-postgres-password>'
export NEO4J_PW='<choose-a-neo4j-password>'

kubectl create secret generic postgres-credentials -n platform \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD="$PG_PW" \
  --from-literal=POSTGRES_DB=postgres

kubectl create secret generic neo4j-credentials -n platform \
  --from-literal=NEO4J_AUTH="neo4j/$NEO4J_PW"

# 5. Postgres and Neo4j
kubectl apply -k k8s/platform/postgres/overlays/local-minikube
kubectl apply -k k8s/platform/neo4j/overlays/local-minikube

kubectl rollout status statefulset/postgres -n platform --timeout=240s
kubectl rollout status statefulset/neo4j    -n platform --timeout=360s
```

Neo4j takes about a minute to become Ready on first start.

**Optional, Kafka:** dataplane-api's ConfigMap has `KAFKA_ENABLED: "true"`.
Either deploy it:

```bash
kubectl apply -k k8s/platform/kafka/overlays/local-minikube
```

or set `KAFKA_ENABLED: "false"` in dataPlane's `k8s/dataplane/configmap.yaml`
before Part 3. (The OTel collector in that ConfigMap has no manifest here yet.)

---

## Part 3: Deploy dataplane-api (minikube host)

```bash
cd ../dataPlane
git checkout <same commit as $TAG>          # manifests matching the image
export TAG=<tag from Part 1>
export IMAGE=docker.io/aakashbhatt/dataplane
```

### 1. App secret

`DATABASE_URL` and `NEO4J_USER`/`NEO4J_PASSWORD` must match Part 2's
credentials:

```bash
kubectl create secret generic dataplane-secrets -n dataplane \
  --from-literal=DATABASE_URL="postgresql://postgres:${PG_PW}@postgres.platform.svc.cluster.local:5432/postgres" \
  --from-literal=NEO4J_USER=neo4j \
  --from-literal=NEO4J_PASSWORD="$NEO4J_PW" \
  --from-literal=SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=ADMIN_USERNAME='<value>' \
  --from-literal=ADMIN_PASSWORD='<value>' \
  --from-literal=GOOGLE_MAPS_API_KEY='<value>'
```

New shell and lost `$PG_PW` / `$NEO4J_PW`? Read them back:

```bash
kubectl get secret postgres-credentials -n platform -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d; echo
kubectl get secret neo4j-credentials    -n platform -o jsonpath='{.data.NEO4J_AUTH}' | base64 -d; echo   # neo4j/<pw>
```

### 2. Docker Hub pull credentials (private repo only)

Skip this if `aakashbhatt/dataplane` is public.

```bash
kubectl create secret docker-registry dockerhub -n dataplane \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=aakashbhatt \
  --docker-password='<docker-hub-access-token>'

# Let every pod in the namespace use it, without editing deployment.yaml
kubectl patch serviceaccount default -n dataplane \
  -p '{"imagePullSecrets":[{"name":"dockerhub"}]}'
```

### 3. Deploy with the pushed tag

`deployment.yaml` says `aakashbhatt/dataplane:latest`. Swap in the tag while
applying; the file itself is unchanged. List files explicitly: **don't**
`apply -f k8s/dataplane/`, because `secret.yaml` there is a `REPLACE_ME`
template and would overwrite step 1's secret.

```bash
kubectl apply -f k8s/dataplane/configmap.yaml -f k8s/dataplane/service.yaml

sed "s|image: aakashbhatt/dataplane:latest|image: $IMAGE:$TAG|" k8s/dataplane/deployment.yaml \
  | kubectl apply -f -

kubectl rollout status deployment/dataplane-api -n dataplane
kubectl get deployment dataplane-api -n dataplane \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo    # should show $IMAGE:$TAG
```

### 4. Open the API

On the minikube host itself:

```bash
minikube service dataplane-api -n dataplane --url     # then <url>/docs
```

From another machine (e.g. your Mac): minikube's docker-driver IP is only
reachable on the host, so forward a port on the host instead:

```bash
# on the minikube host — listens on all interfaces, port 8000
kubectl port-forward --address 0.0.0.0 -n dataplane svc/dataplane-api 8000:80
# then from your Mac: http://<minikube-host>:8000/docs
```

Or keep it private with an SSH tunnel from your Mac:
`ssh -L 8000:localhost:8000 <minikube-host>`, with the port-forward above
using the default `--address 127.0.0.1`.

---

## Shipping a new version

1. **Build machine:** commit, then rerun Part 1 (new SHA means a new `$TAG`).
2. **minikube host:**

   ```bash
   kubectl set image deployment/dataplane-api -n dataplane dataplane-api=$IMAGE:$TAG
   kubectl rollout status deployment/dataplane-api -n dataplane
   ```

   If `configmap.yaml`/`deployment.yaml` changed too, `git pull` and redo
   Part 3 step 3 instead.

Roll back: `kubectl rollout undo deployment/dataplane-api -n dataplane`, or
`set image` to an older SHA.

## Verify

```bash
kubectl get pods -n platform     # postgres-0, neo4j-0 Running
kubectl get pods -n dataplane    # dataplane-api pods Running

# Reach both databases from the dataplane namespace, the way the app does
kubectl run pg-q --rm -i --restart=Never -n dataplane --image=postgres:16 -- \
  psql "postgresql://postgres:${PG_PW}@postgres.platform.svc.cluster.local:5432/postgres" -c 'select version()'

kubectl run neo-q --rm -i --restart=Never -n dataplane --image=neo4j:5-community --command -- \
  cypher-shell -a bolt://neo4j.platform.svc.cluster.local:7687 -u neo4j -p "$NEO4J_PW" 'RETURN 1'
```

Neo4j Browser, on the minikube host:

```bash
kubectl port-forward -n platform svc/neo4j 7474:7474 7687:7687
# open http://localhost:7474, connect to bolt://localhost:7687 as neo4j / $NEO4J_PW
```

## Day-to-day (minikube host)

| Task | Command |
|---|---|
| Stop the cluster, keep data | `minikube stop` |
| Start again | `minikube start` (secrets, PVCs, and pulled images persist) |
| Wipe everything | `minikube delete` |
| Remove only the app | `kubectl delete deployment,service,configmap -n dataplane -l app.kubernetes.io/name=dataplane-api` |

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| dataplane-api `CrashLoopBackOff`, logs say `exec format error` | Image built for the wrong CPU arch. Rebuild with `PLATFORM` matching the host's `uname -m` (Part 1). |
| dataplane-api `ImagePullBackOff`: `pull access denied` / `unauthorized` | Private repo with no pull secret, or a wrong token. Redo Part 3 step 2, then `kubectl rollout restart deployment/dataplane-api -n dataplane`. |
| dataplane-api `ImagePullBackOff`: `manifest unknown` / `not found` | `$TAG` was never pushed, or there's a typo. Check with `docker buildx imagetools inspect $IMAGE:$TAG` on the build machine. |
| Pods run an old build | The image is still `:latest`, or the same tag was re-pushed. Use a new SHA tag and `kubectl set image`. |
| `postgres-0` / `neo4j-0` stuck in `CreateContainerConfigError` | The credentials Secret is missing. Run Part 2 step 4. |
| `neo4j-0` restarting, logs mention an unrecognized setting | A non-config `NEO4J_*` key ended up in `neo4j-credentials`. It must contain only `NEO4J_AUTH`. |
| `neo4j-0` OOMKilled | Heap + page cache in `neo4j-config` exceed the container limit. Raise the limit or lower the sizes together. |
| dataplane-api: `password authentication failed` | `DATABASE_URL` doesn't match `postgres-credentials`. Read the password back (Part 3 step 1) and recreate `dataplane-secrets`. |
| dataplane-api: Neo4j `Unauthorized` | `NEO4J_PASSWORD` in `dataplane-secrets` doesn't match `NEO4J_AUTH` in `neo4j-credentials`. |
| `git clone` on the host lacks `k8s/platform/neo4j/` | platform-infra changes weren't pushed to GitHub yet. |
