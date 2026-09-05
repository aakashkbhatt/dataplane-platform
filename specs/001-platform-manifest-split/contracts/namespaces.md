# Contract: Namespaces

| | |
|---|---|
| Resources | `platform`, `dataplane`, `datagen`, `familytree` — plain `Namespace` objects, `k8s/namespaces/` |
| Apply | `kubectl apply -k k8s/namespaces/` — single command, all four namespaces, idempotent on repeat application (Edge Case 3). |
| Stability | Namespace names are the stable contract every product repo's own manifests target via `metadata.namespace`. Renaming or removing a namespace here is a breaking change for every product deployed into it. |
| Isolation rule | Per Principle II: `platform` MUST NOT depend on any product namespace; `dataplane`/`datagen`/`familytree` MUST NOT reference each other directly. This repo enforces the *existence* of the boundary — enforcing that no product's manifest violates it (e.g. via NetworkPolicy) is future cross-cutting scope, not this feature's FR list. |
| Consumer responsibility | Each product repo's own manifests set `metadata.namespace` to their assigned namespace; this repo does not template product Deployments/Services into these namespaces (Principle I). |
