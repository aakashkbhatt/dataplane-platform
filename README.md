# platform-infra

Kubernetes manifests for shared platform services (Postgres, Neo4j, Kafka,
MinIO) and cluster-wide resources (namespaces, ingress controller). Products'
own Deployments, Services, and Ingresses live in each product's repo, never
here. See `.specify/memory/constitution.md` for the ownership rules.

```text
k8s/
├── namespaces/                      platform, dataplane, datagen, familytree
└── platform/<service>/
    ├── base/
    └── overlays/{local-minikube,eks-staging,eks-prod}/
```

## Setting up a cluster

- **Mac mini (runs minikube):** [docs/quickstart-macmini.md](docs/quickstart-macmini.md),
  covering the cluster, ingress controller, Postgres/Neo4j/Kafka, and dataplane-api
- **MacBook (builds and pushes):** [docs/quickstart-macbook.md](docs/quickstart-macbook.md)

Each service is applied with Kustomize, e.g.
`kubectl apply -k k8s/platform/postgres/overlays/local-minikube`. Secrets are
created out-of-band from each `secret.example.yaml`, never committed.

## Local access through the ingress controller

minikube runs one shared NGINX ingress controller (Mac mini guide, step 3b),
so every app is reached at `http://<app>.local` through a single entry point
instead of a `kubectl port-forward` per app.

### 1. Enable the controller (once; safe to re-run)

```bash
minikube addons enable ingress
kubectl wait --namespace ingress-nginx --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=300s
kubectl get ingressclass          # nginx   k8s.io/ingress-nginx
```

### 2. Run the tunnel in a separate terminal

With the Docker driver on macOS, the minikube node's IP isn't reachable from
the host. `minikube tunnel` exposes the ingress controller on `127.0.0.1`
ports 80/443:

```bash
minikube tunnel        # leave running; asks for your sudo password (ports 80/443)
```

Stopping the tunnel, sleeping the Mac, or `minikube stop` cuts access. Start
it again after each of those.

### 3. Map each app's hostname to 127.0.0.1

Add one line per app to `/etc/hosts` (`sudo nano /etc/hosts`):

```text
127.0.0.1 dataplane.local
```

macOS also resolves `.local` names over Bonjour/mDNS. If lookups are slow
(a few seconds per request), also add `::1 dataplane.local`.

Then browse to `http://dataplane.local/` (e.g. `http://dataplane.local/docs`).

### Each app owns its Ingress

This repo provides only the controller, the `nginx` IngressClass. Each app
defines its own `Ingress` **in its own namespace and repo**, next to its
Service. For example, dataplane-api's would live in the dataPlane repo:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dataplane-api
  namespace: dataplane
spec:
  ingressClassName: nginx
  rules:
    - host: dataplane.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dataplane-api
                port:
                  number: 80
```

No Ingress resources are defined in this repo.

### Reaching apps from another machine

The tunnel listens only on the minikube host's own `127.0.0.1`. To reach
`dataplane.local` from the MacBook while minikube runs on the Mac mini, use
an SSH tunnel from the MacBook:

```bash
ssh -N -L 8080:127.0.0.1:80 <user>@<mac-mini-hostname>     # keep open; minikube tunnel must be running on the Mac mini
```

Then add `127.0.0.1 dataplane.local` to the **MacBook's** `/etc/hosts` too,
and browse to `http://dataplane.local:8080/docs`. Ingress matches on the
hostname and ignores the port.
