# Quickstart: MacBook (build machine)

Everything in this file runs on **your MacBook**. Nothing here touches
minikube; that's all in [quickstart-macmini.md](quickstart-macmini.md).

The MacBook's two jobs:

1. Build the dataplane-api image and push it to Docker Hub
2. Push both repos to GitHub, so the Mac mini can `git pull` the manifests

---

## 1. One-time setup

```bash
docker login            # Docker Hub user: aakashbhatt
```

Ask the Mac mini once for its CPU type (run `uname -m` **there**):

- `arm64` (Apple Silicon): use the plain build in step 2. This is the usual
  case, and matches the MacBook.
- `x86_64` (old Intel Mac mini): use the "Intel" line in step 2 instead.

Find the Mac mini's name under System Settings → General → Sharing →
**Local hostname** (e.g. `aakash-mac-mini.local`). You'll use it in step 5.

---

## 2. Build and push the image

Do this every time you want the Mac mini to run new dataplane-api code.

```bash
cd ~/PycharmProjects/dataPlane
git add -A && git commit -m "..."       # the image tag is the commit, so commit first
TAG=$(git rev-parse --short HEAD)

docker build -t aakashbhatt/dataplane:$TAG -t aakashbhatt/dataplane:latest .
docker push aakashbhatt/dataplane:$TAG
docker push aakashbhatt/dataplane:latest

echo "TAG=$TAG"                         # ← send this value to the Mac mini
```

Intel Mac mini only, instead of the build and push lines:
`docker buildx build --platform linux/amd64 -t aakashbhatt/dataplane:$TAG -t aakashbhatt/dataplane:latest --push .`

Why two tags: `:latest` is what `deployment.yaml` names for the first
deploy. `:$TAG` is what the Mac mini uses for every update after that,
because once minikube has a `:latest` it never downloads a newer `:latest`
again.

> You already pushed `aakashbhatt/dataplane:latest` (arm64, private repo).
> For the **first** deploy you can skip this step. Come back to it for the
> next release.

---

## 3. Push dataPlane's Kubernetes files to GitHub

The Mac mini clones dataPlane only for `k8s/dataplane/` (configmap,
deployment, service). The app code reaches it through the image instead.
`main` doesn't have them. Today they're on `entity_enhancement_apis`, which
is the branch the Mac mini checks out.

When you change any of the 3 files, push your current branch, then check out
that same branch on the Mac mini:

```bash
cd ~/PycharmProjects/dataPlane
git add k8s/dataplane && git commit -m "..."
git push -u dataplane "$(git branch --show-current)"
git branch --show-current          # ← the branch to check out on the Mac mini
```

Never put real values in `k8s/dataplane/secret.yaml`. It stays a
`REPLACE_ME` template, and the Mac mini creates the real secret itself.

---

## 4. Push platform-infra to GitHub

The Mac mini gets Postgres, Neo4j, Kafka, and namespace manifests by cloning
this repo from GitHub
(`https://github.com/aakashkbhatt/dataplane-platform`, public).

```bash
cd ~/PycharmProjects/platform-infra
git add -A && git commit -m "..."
git push
```

Re-run this whenever you change anything in this repo. Then on the Mac mini,
`git pull` (see that guide's "Updates" section).

---

## 5. Open the API from the MacBook

Once the Mac mini has started its port-forward (Mac mini guide, step 8):

```text
http://<mac-mini-hostname>:8000/docs        e.g. http://aakash-mac-mini.local:8000/docs
```

---

## 6. Export your database for the Mac mini

Your local Postgres (docker container `psql-postgre-sql-1`, port 5432) has
the data. dataplane-api uses only the **`dataplane` schema** (~33 tables,
under 1 MB compressed), so export just that. The ~4 GB in `public`
(`products`, `profile_`, …) isn't used by the app.

```bash
docker exec psql-postgre-sql-1 \
  pg_dump -U postgres -d postgres -n dataplane -Fc --no-owner --no-privileges \
  > ~/dataplane.dump

ls -lh ~/dataplane.dump
```

Copy `~/dataplane.dump` to the Mac mini's home folder (AirDrop, or
`scp ~/dataplane.dump <mac-mini-hostname>:~/` if Remote Login is on). Then
follow the Mac mini guide's step 7b.

This is real data: keep the file out of both git repos, and delete it from
both machines once it's restored.

---

## Summary: what to do when

| You changed… | On the MacBook | Then on the Mac mini |
|---|---|---|
| dataplane-api code | Step 2 → send `TAG` | Updates → "New app version" |
| `configmap.yaml` / `deployment.yaml` / `service.yaml` | Step 3 (push) | Updates → "Changed dataplane files" |
| Anything in platform-infra | Step 4 | Updates → "Changed platform-infra" |
| Want the Mac mini to have your current local data | Step 6 → copy `dataplane.dump` | Step 7b |
