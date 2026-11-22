# wharf

A container registry written in Rust, built to replace Harbor in place.

> **Status: design in progress.** There is no working code yet. This README describes what wharf is being built to do, not what it does today. The design will land in `docs/design.md`.

## Why

Harbor does a lot, and it does it with a lot of moving parts: core, jobservice, registry, registryctl, portal, nginx, the Trivy adapter, Redis and PostgreSQL. Running it means sizing, upgrading and debugging all of them.

wharf aims to cover the same ground as one binary:

- **One process.** Registry, management API, web UI, background jobs and token service in a single binary. SQLite by default, PostgreSQL when you need several replicas. No Redis, no nginx.
- **Drop-in for Harbor.** The goal is to point wharf at a Harbor's existing storage and database, import everything, and switch DNS. Docker, containerd, Helm, ORAS and cosign clients work unchanged. Harbor's `/api/v2.0` REST API is the management API, so scripts, CI jobs and the Harbor Terraform provider keep working.
- **Small and fast.** Low memory at idle and under load, streaming uploads, presigned redirects for pulls from object storage. These are goals with benchmarks to back them, not results yet.

## Planned scope

- OCI Distribution Spec 1.1: push, pull, chunked uploads, cross-repository mounts, referrers, tag listing, with the OCI conformance suite in CI
- Projects, members, Harbor's roles, custom roles, quotas
- Local users, LDAP/AD, OIDC, HTTP auth proxy, robot accounts, CI workload identity
- Replication, proxy cache, retention and immutability rules, online garbage collection
- Vulnerability scanning through Harbor's Pluggable Scanner API, then a built-in scanner
- Signatures and attestations (cosign, Notation) through the referrers API, with policies on pull
- Webhooks, audit log, Prometheus metrics, OpenTelemetry tracing
- Helm charts and any OCI artifact
- A web UI and a CLI, including `wharf import harbor`

The phases and a row-by-row comparison with Harbor, zot, CNCF Distribution, Quay and Trow will be in the design document.

## License

Apache License 2.0. See [LICENSE](LICENSE).
