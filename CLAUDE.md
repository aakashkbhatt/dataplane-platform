<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan:
`specs/001-platform-manifest-split/plan.md`
<!-- SPECKIT END -->

This repo owns Kubernetes manifests for shared platform services (Kafka,
Postgres, MinIO) and cluster-wide resources (Namespaces, StorageClass, the
eksctl cluster config) — never a product's own Deployment/Service, per
`.specify/memory/constitution.md` Principle I. See that file for the full
ownership boundary and manifest conventions this repo follows.
