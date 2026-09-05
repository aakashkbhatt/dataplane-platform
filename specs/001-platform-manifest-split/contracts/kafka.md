# Contract: Kafka

What a product namespace (`dataplane`, `datagen`, `familytree`) may rely on
when depending downward on this platform service (Principle II).

| | |
|---|---|
| Endpoint | `kafka.platform.svc.cluster.local:9092` |
| Protocol | Kafka wire protocol (KRaft mode, no ZooKeeper) |
| Auth | None at this feature's scope — cluster-internal only, no external Ingress. A follow-up feature may add SASL if a product requires it. |
| Stability | DNS name and port are the stable contract. Broker count, storage size, and resource sizing may change per overlay without notice to consumers. |
| Consumer responsibility | Each product repo owns its own client config pointing at the address above (spec Assumption A1) — this repo does not template product-side ConfigMaps. |
| Breaking-change signal | Any change to the DNS name, port, or namespace this Service lives in is a breaking change and MUST be called out in the PR description per Principle VI. |
