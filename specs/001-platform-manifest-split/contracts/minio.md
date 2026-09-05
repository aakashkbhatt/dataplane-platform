# Contract: MinIO

| | |
|---|---|
| Endpoint | `minio.platform.svc.cluster.local:9000` (S3 API), `:9090` (console — cluster-internal only, not exposed for this feature) |
| Protocol | S3-compatible HTTP API |
| Auth | Access/secret key via a `Secret` the operator applies out-of-band from `secret.example.yaml`'s key shape (Principle V) — never a value committed to this repo. |
| Stability | DNS name, API port, and the `Secret`'s key names are the stable contract. |
| Bucket ownership | This repo does not create or own the `datagen-output` bucket (or any bucket) — bucket provisioning is a client-side/operator action against the endpoint above (see `research.md` §4). datagen_speckit's repo owns the fact that it needs a bucket named `datagen-output`; this repo only guarantees the S3 endpoint exists and is reachable. |
| Consumer responsibility | datagen_speckit's own manifests/config point at the endpoint above once its embedded MinIO Deployment/Service is removed (FR-002); that removal + repointing is tracked in datagen_speckit's own repo, same boundary as Kafka (spec Assumption A1). |
| Breaking-change signal | Any change to DNS name, API port, or namespace is a breaking change and MUST be called out in the PR description per Principle VI. |
