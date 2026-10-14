# Quickstart: Mac mini (runs minikube)

Everything in this file runs on **the Mac mini**. The MacBook side (building
and pushing the image) is in [quickstart-macbook.md](quickstart-macbook.md).

What ends up running here, in one minikube cluster:

```text
minikube
├── namespace platform    postgres-0, neo4j-0, kafka-0     (from ~/platform-infra)
└── namespace dataplane   dataplane-api                    (image from Docker Hub,
                                                            YAML from ~/dataPlane/k8s/dataplane)
```

Before starting, make sure the MacBook has pushed both repos to GitHub.

---

## 1. Install tools (once)

```bash
brew install minikube kubectl git
uname -m          # tell the MacBook: arm64 or x86_64
```

Install **Docker Desktop**, open it, and set **Settings → Resources →
Memory ≥ 8 GB, CPUs ≥ 4**. If the Mac mini only has 8 GB of RAM in total,
use 6 GB there and `--memory=4608` in step 3.

---

## 2. Get the two repos (once)

```bash
cd ~
git clone https://github.com/aakashkbhatt/dataplane-platform.git platform-infra
git clone https://githusb.com/aakashkbhatt/dataplane.git dataPlane

cd ~/dataPlane
git checkout entity_enhancement_apis      # main has no k8s/dataplane/ folder
ls k8s/dataplane/                        # configmap.yaml deployment.yaml secret.yaml service.yaml
```

- **platform-infra** (public) provides the Postgres, Neo4j, Kafka, and
  namespace manifests.
- **dataPlane** (private, so `git clone` asks for your GitHub login) is
  needed only for its 3 YAML files in `k8s/dataplane/`. The app code itself
  comes from the Docker Hub image.

---

## 3. Start minikube

```bash
minikube start --driver=docker --cpus=4 --memory=6144
```

After a reboot, run just `minikube start`. Everything below survives
restarts, so skip to step 8.

---

## 3b. Ingress controller (once; safe to re-run)

One shared NGINX entry point for every app's Ingress. It's cluster-wide, so
it lives here, not in any app repo.

```bash
minikube addons enable ingress            # no-op if already enabled
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s
kubectl get ingressclass                  # expect: nginx   k8s.io/ingress-nginx
```

The addon stays enabled across `minikube stop`/`start`. To reach apps through
it, see "Local access through the ingress controller" in the
[README](../README.md).

---

## 4. Namespaces, Postgres, Neo4j, Kafka (once)

```bash
cd ~/platform-infra
kubectl apply -k k8s/namespaces
```

Pick two passwords. They live only inside the cluster:

```bash
PG_PW='<choose-a-postgres-password>'
NEO4J_PW='<choose-a-neo4j-password>'

kubectl create secret generic postgres-credentials -n platform \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD="$PG_PW" \
  --from-literal=POSTGRES_DB=postgres

kubectl create secret generic neo4j-credentials -n platform \
  --from-literal=NEO4J_AUTH="neo4j/$NEO4J_PW"
```

Deploy and wait (Neo4j takes about a minute):

```bash
kubectl apply -k k8s/platform/postgres/overlays/local-minikube
kubectl apply -k k8s/platform/neo4j/overlays/local-minikube
kubectl apply -k k8s/platform/kafka/overlays/local-minikube

kubectl rollout status statefulset/postgres -n platform --timeout=300s
kubectl rollout status statefulset/neo4j    -n platform --timeout=300s
kubectl rollout status statefulset/kafka    -n platform --timeout=300s
```

---

## 5. Docker Hub login for the cluster (once)

The image repo is private, so minikube needs a token to download it. Create
one at hub.docker.com → Account settings → Personal access tokens →
**Read-only**.

```bash
kubectl create secret docker-registry dockerhub -n dataplane \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=aakashbhatt \
  --docker-password=''

kubectl patch serviceaccount default -n dataplane \
  -p '{"imagePullSecrets":[{"name":"dockerhub"}]}'
```

---

## 6. dataplane-api secret (once)

This reuses the passwords from step 4. If you're in a new terminal, reload
them first:

```bash
PG_PW=$(kubectl get secret postgres-credentials -n platform -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)
NEO4J_PW=$(kubectl get secret neo4j-credentials -n platform -o jsonpath='{.data.NEO4J_AUTH}' | base64 -d | cut -d/ -f2)
```

```bash
kubectl create secret generic dataplane-secrets -n dataplane \
  --from-literal=DATABASE_URL="postgresql://postgres:${PG_PW}@postgres.platform.svc.cluster.local:5432/postgres" \
  --from-literal=NEO4J_USER=neo4j \
  --from-literal=NEO4J_PASSWORD="$NEO4J_PW" \
  --from-literal=SECRET_KEY="$(openssl rand -hex 32)" \
  --from-literal=ADMIN_USERNAME='tesuser4' \
  --from-literal=ADMIN_PASSWORD='newpassword' \
  --from-literal=GOOGLE_MAPS_API_KEY='<value>'
```

---

## 7. Deploy dataplane-api (first time)

Uses 3 of the files in `~/dataPlane/k8s/dataplane/`. **Don't** apply
`secret.yaml` from that folder: it's a `REPLACE_ME` template and would
overwrite the step 6 secret.

`~/dataPlane/k8s/dataplane` is just a folder on disk. Each file says
`namespace: dataplane` inside it, which is what puts the app in the
`dataplane` namespace.

```bash
cd ~/dataPlane/k8s/dataplane
kubectl apply -f configmap.yaml -f service.yaml -f deployment.yaml
kubectl rollout status deployment/dataplane-api -n dataplane --timeout=300s
kubectl get pods -n dataplane
```

All pods should show `Running` and `1/1`.

---

## 7b. Load data from the MacBook (optional)

A fresh Postgres is empty. To copy the MacBook's data in, the MacBook first
runs its step 6 and copies `dataplane.dump` to `~/` here.

```bash
# Stop the app so nothing writes during the restore
kubectl scale deployment/dataplane-api -n dataplane --replicas=0

# Restore. --clean --if-exists replaces the empty tables the app created on first start.
kubectl exec -i -n platform postgres-0 -- \
  pg_restore -U postgres -d postgres --clean --if-exists --no-owner --no-privileges \
  < ~/dataplane.dump

# Check: should print 33 (the number of tables in the dataplane schema)
kubectl exec -n platform postgres-0 -- \
  psql -U postgres -d postgres -Atc "select count(*) from pg_tables where schemaname='dataplane'"

# Start the app again
kubectl scale deployment/dataplane-api -n dataplane --replicas=2
kubectl rollout status deployment/dataplane-api -n dataplane

rm ~/dataplane.dump
```

Re-running this later replaces the Mac mini's `dataplane` schema with the
MacBook's. Anything created only on the Mac mini is lost. To keep a copy
first:
`kubectl exec -n platform postgres-0 -- pg_dump -U postgres -d postgres -n dataplane -Fc > ~/macmini-backup.dump`

---

## 8. Open the API

Once dataplane-api ships its own Ingress (in the dataPlane repo), use the
ingress workflow in the [README](../README.md) instead of the port-forward
below.

On the Mac mini:

```bash
minikube service dataplane-api -n dataplane --url       # open <url>/docs
```

From the MacBook: run this here and leave the terminal open:

```bash
kubectl port-forward --address 0.0.0.0 -n dataplane svc/dataplane-api 8000:80
```

Then on the MacBook, browse to `http://<mac-mini-hostname>:8000/docs`. If
macOS asks whether `kubectl` may accept incoming connections, click Allow.

---

## Updates

**New app version.** The MacBook sends you a `TAG`:

```bash
kubectl set image deployment/dataplane-api -n dataplane \
  dataplane-api=aakashbhatt/dataplane:<TAG>
kubectl rollout status deployment/dataplane-api -n dataplane
```

Undo: `kubectl rollout undo deployment/dataplane-api -n dataplane`

**Changed dataplane files.** The MacBook pushed a change to
`k8s/dataplane/` in dataPlane:

```bash
cd ~/dataPlane
git fetch && git checkout <branch the MacBook pushed> && git pull
cd k8s/dataplane
kubectl apply -f configmap.yaml -f service.yaml
kubectl rollout restart deployment/dataplane-api -n dataplane
```

If `deployment.yaml` changed, also run
`kubectl apply -f deployment.yaml`, then repeat the `set image` command
above with the latest `TAG`. Applying the file resets the image to `:latest`.

**Changed platform-infra.** The MacBook pushed to GitHub:

```bash
cd ~/platform-infra && git pull
kubectl apply -k k8s/namespaces
kubectl apply -k k8s/platform/postgres/overlays/local-minikube
kubectl apply -k k8s/platform/neo4j/overlays/local-minikube
kubectl apply -k k8s/platform/kafka/overlays/local-minikube
```

---

## Everyday commands

| Task | Command |
|---|---|
| Is everything up? | `kubectl get pods -n platform; kubectl get pods -n dataplane` |
| App logs | `kubectl logs -n dataplane deploy/dataplane-api --tail=100` |
| Which image is running? | `kubectl get deploy dataplane-api -n dataplane -o jsonpath='{.spec.template.spec.containers[0].image}'; echo` |
| Stop (keeps all data) | `minikube stop` |
| Start again | `minikube start` |
| Wipe everything | `minikube delete`, then redo steps 3–7 |

## Troubleshooting

| You see | Fix |
|---|---|
| App pod `ImagePullBackOff`, `unauthorized` / `pull access denied` | Step 5 missing or wrong token. Fix it, then `kubectl rollout restart deployment/dataplane-api -n dataplane`. |
| App pod `ImagePullBackOff`, `not found` | That `TAG` wasn't pushed. Ask the MacBook to check its step 2 output. |
| App pod `CrashLoopBackOff`, logs say `exec format error` | The image was built for the wrong CPU. Tell the MacBook what `uname -m` prints here. |
| App pod `CrashLoopBackOff`, other errors | Read `kubectl logs -n dataplane deploy/dataplane-api`. Usually a wrong value in step 6. |
| `password authentication failed` / Neo4j `Unauthorized` in logs | Step 6 passwords don't match step 4. Run `kubectl delete secret dataplane-secrets -n dataplane`, redo step 6, then `rollout restart`. |
| `postgres-0` / `neo4j-0` in `CreateContainerConfigError` | Step 4 secrets missing. Re-run its `create secret` lines. |
| `neo4j-0` `OOMKilled` | Not enough memory. Raise Docker Desktop / minikube memory (`minikube delete`, then step 3 with a larger `--memory`). |
| Pods `Pending` | Not enough CPU/memory in minikube. `kubectl describe pod <name> -n <ns>` shows why. |
| MacBook can't open `:8000/docs` | Is the step 8 port-forward still running, with `--address 0.0.0.0`? Was the macOS prompt allowed? |
