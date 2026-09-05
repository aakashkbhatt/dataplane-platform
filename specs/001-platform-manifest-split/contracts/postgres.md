# Contract: Postgres

| | |
|---|---|
| Endpoint | `postgres.platform.svc.cluster.local:5432` |
| Protocol | PostgreSQL wire protocol |
| Auth | Credentials via a `Secret` the operator applies out-of-band from `secret.example.yaml`'s key shape (Principle V) — never a value committed to this repo. |
| Stability | DNS name, port, and the `Secret`'s key names (e.g. `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`) are the stable contract; the values behind those keys are operator-managed and out of scope here. |
| Consumer responsibility | dataplane-api currently uses an external/local Postgres container, not this instance (spec Assumption A2) — cutting over is a separate, later feature in dataplane-api's own repo, and includes a real data migration this feature does not perform. |
| Breaking-change signal | Any change to DNS name, port, namespace, or a `Secret` key name a consumer already reads is a breaking change and MUST be called out in the PR description per Principle VI. |
