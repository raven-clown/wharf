# wharf design

Status: draft for review, 2026-11-22.

## 1. Summary

wharf is an OCI container registry written in Rust. It covers everything Harbor does, ships as one binary, and replaces a running Harbor in place: same storage, same management API, same users, robots and policies, with nothing to re-push and no credentials to rotate.

The project wins on four things, each backed by a test or a benchmark rather than a slogan:

1. **Drop-in.** Harbor's `/api/v2.0` is wharf's management API. `wharf import harbor` reads a Harbor's database and reuses its blob storage as is.
2. **One process.** Registry, API, web UI, token service and background jobs in one binary. SQLite by default, PostgreSQL for several replicas. No Redis, no nginx, no sidecars.
3. **Light and fast.** Streaming uploads, presigned redirects for pulls, reference-counted garbage collection, exact quotas.
4. **Short setup.** `wharf serve` starts with no config. A production config fits in ten lines.

## 2. Goals and non-goals

Goals:

- Pass the OCI Distribution Spec conformance suite, every release.
- Reach Harbor feature parity by the end of phase 4, and go past it in phase 5.
- Keep every Harbor client and tool working unchanged: Docker, Podman, containerd, Helm, ORAS, cosign, Notation, crane, skopeo, the Harbor Terraform provider, harbor-cli, scripts against `/api/v2.0`, and other Harbors replicating to or from wharf.
- Publish reproducible benchmarks against Harbor, zot, CNCF Distribution, Quay and Trow for every performance claim.

Non-goals:

- Building container images from Dockerfiles (Quay has this; it belongs in CI).
- Notary v1 and ChartMuseum, which Harbor itself removed in 2.6 and 2.8.
- Compatibility with Harbor's internal component APIs (registryctl, jobservice, the scanner adapter's internals). Only the public `/v2`, `/api/v2.0` and `/service/token` surfaces count.
- Harbor's UAA auth mode, until someone asks for it. The importer stops with a clear message when it finds UAA.

## 3. Who it is for and what success looks like

- **Teams running Harbor** who are tired of sizing and upgrading nine services plus PostgreSQL and Redis. Success: they run `wharf import harbor --check`, see a clean report, switch DNS, and their CI, Terraform and Kubernetes clusters keep working.
- **Teams on zot or Distribution** who want projects, quotas, replication and a real UI without moving to a multi-service stack. Success: one binary gives them Harbor-level features.
- **Contributors** who want a registry they can build with `cargo build` and test on a laptop.

## 4. The field

Facts as of November 2026. Where a fact is not confirmed it is marked "unverified" in the matrix.

- **Harbor** (CNCF graduated, Go). Latest line 2.14 upstream, with 2.15 reported by a downstream distribution. Runs as core, jobservice, registry (CNCF Distribution), registryctl, portal, nginx, trivy-adapter, plus PostgreSQL and Redis. Documented minimum 2 CPU and 4 GB RAM. Common complaints: the number of moving parts, database and Redis memory under replication and GC, upgrade paths that break, quota usage that lags under concurrent pushes.
- **zot** (CNCF sandbox, Go). Single binary, OCI native storage, no external database. Embeds Trivy as a library, has a UI (zui), sync (pull mirroring and on-demand proxy), retention, dedupe, LDAP, OIDC, API keys, and signature verification with uploaded cosign keys and Notation certificates. No projects or quotas in Harbor's sense, no push replication, no Harbor API.
- **CNCF Distribution** (Go). The reference registry, v3 since April 2025. Storage drivers, token auth, a single-upstream pull-through proxy, notifications. Offline garbage collection. No UI, RBAC, scanning or replication.
- **Quay** (Red Hat, Python). Organizations, teams, robots, Clair scanning, repository and organization mirroring, proxy cache, quotas, auto-prune, immutable tags (3.17), builds, geo-replication at the storage layer. Needs PostgreSQL, Redis, Clair and object storage.
- **Trow** (Rust). Registry and image cache built to run inside Kubernetes, with an admission webhook for image allow and deny lists. SQLite since 0.7. Last release found: v0.7.2, February 2025. No UI, RBAC or scanning. It is the only other Rust registry with real use, and its scope is narrow.

## 5. Architecture

### 5.1 One binary, many crates

wharf is a modular monolith: one binary, a Cargo workspace inside, each crate with one job and a narrow interface.

```
crates/
  wharf          the binary: CLI, wiring, config
  http           axum router: /v2, /api/v2.0, /service/token, /ui, /metrics, /healthz
  oci            distribution spec: uploads, manifests, referrers, tags, mounts
  storage        blob store over object_store: fs, S3, GCS, Azure
  meta           sqlx repositories and migrations: SQLite and PostgreSQL
  auth           users, groups, robots, tokens, LDAP, OIDC, RBAC checks
  policy         retention, immutability, quota, signature and vulnerability gates
  jobs           durable queue, scheduler, workers
  replication    registry adapters and the copy engine
  scan           Pluggable Scanner API client, later the built-in scanner
  webhook        event fan-out and delivery
  harbor         /api/v2.0 handlers and models, and the Harbor importer
  ui             compiled templates, htmx, static assets
```

Rules for the boundaries:

- `oci`, `policy`, `replication` and `scan` never touch SQL directly. They call traits in `meta`.
- `harbor` is a translation layer over the same services the UI and CLI use. There is no second business logic.
- `http` handlers stay thin: parse, authorize, call a service, render.

Building blocks: tokio, axum, sqlx, object_store, oci-spec, rustls, tracing with OpenTelemetry export.

### 5.2 Storage

Blobs use the exact layout of CNCF Distribution, the same one Harbor writes:

```
docker/registry/v2/blobs/sha256/<first two hex>/<digest>/data
docker/registry/v2/repositories/<name>/_manifests/...
```

So wharf can open a Harbor's volume or bucket in place. wharf keeps the authoritative manifest, tag and reference data in its database, and writes the repository link files too, so a Harbor or Distribution instance pointed at the same storage still sees a consistent tree during a migration window.

Content is addressed by digest and deduplicated across projects. Each blob has a reference count in the database, updated in the same transaction as the manifest that adds or drops it.

### 5.3 Metadata

- SQLite (WAL mode) by default, one file under the data directory.
- PostgreSQL for HA, from phase 3.
- One set of migrations per backend, run at startup inside a transaction. Each release can roll back to the previous minor version: migrations that drop data wait one release.

### 5.4 Push

1. The client gets a token from `/service/token`. `auth` checks push permission on the project.
2. `oci` streams the blob to storage, hashing as it goes. Nothing is buffered whole in memory.
3. On commit it checks the digest, then the quota, in one transaction with the blob record.
4. A manifest push validates the manifest, checks immutability rules, writes manifest, tag and references, and updates quota usage, all in one transaction.
5. An event goes into the job queue: webhooks, scan on push, event-based replication.

### 5.5 Pull

1. `auth` checks pull permission, then `policy` checks the signature and vulnerability gates for the artifact.
2. On object storage, wharf answers a blob GET with a redirect to a short-lived presigned URL. On a filesystem it streams with Range support.
3. Pull time and count are recorded in batches for retention rules and the UI, never on the request's critical path.

### 5.6 Proxy cache

A proxy-cache project maps to an upstream registry, with Harbor's path model (`/v2/<project>/library/nginx/...`).

- A cache miss streams from upstream to the client and to storage at once.
- Concurrent requests for the same digest share one upstream fetch.
- When upstream is down, cached content is served as is, and tag lookups fall back to the last known digest.
- Tag freshness has a TTL. Docker Hub's rate-limit headers are honored and shown in the UI.
- Deletions upstream can be synced locally, as in Harbor 2.14.

### 5.7 Jobs

Garbage collection, retention, replication, scans, webhook deliveries, preheat and imports all run through one durable queue in the metadata database.

- PostgreSQL claims jobs with `FOR UPDATE SKIP LOCKED`; SQLite uses leases with a heartbeat.
- Each job has a type, a payload, attempts with exponential backoff, a log, and progress that the UI and API can read.
- Concurrency limits per job type. Cron schedules live in the same tables.
- In phase 3, `wharf serve --role api|worker|all` splits request handling from job processing with the same binary.

### 5.8 Garbage collection

Harbor runs GC without blocking pushes since 2.1. wharf matches that and changes how it works: because blob reference counts are exact, GC deletes blobs whose count reached zero longer than a grace window ago (two hours by default, like Harbor), instead of marking the whole registry each run. A full mark-and-sweep stays available as a repair command. Dry runs list what would go.

## 6. Tenancy, identity and access

### 6.1 Harbor's model, exactly

- Projects: public or private, normal or proxy cache, with metadata such as auto scan, severity gate, signature gate, and a CVE allowlist.
- Members: users and groups, with Harbor's roles: Project Admin, Maintainer, Developer, Guest, Limited Guest. System administrators above that.
- The permission matrix matches Harbor's documented table cell for cell, and a test walks every cell.

### 6.2 Authentication

- Local database users, LDAP and Active Directory (with group search), OIDC (with group claims, admin group, and Harbor's CLI secret for `docker login`), and HTTP auth proxy.
- Unlike Harbor, several modes can be active at once: a local break-glass admin next to OIDC and LDAP, and the mode can change while users exist.
- Passwords imported from Harbor keep working: wharf verifies Harbor's PBKDF2 hashes, then rehashes with Argon2id at the next login.

### 6.3 Tokens and robots

- Registry tokens follow the Docker token specification, signed JWTs (ES256) with repository scopes and a short lifetime. `/service/token` keeps Harbor's path.
- Robot accounts at system and project level, with Harbor's fine-grained permissions, expiry, secret refresh, and the `robot$` name prefix.

### 6.4 What goes past Harbor

- **Workload identity:** CI systems exchange their OIDC token (GitHub Actions, GitLab CI, Kubernetes service accounts, any issuer you trust) for a short-lived registry token, under rules that match issuer, subject and claims. No long-lived robot secret stored in CI.
- **Custom roles:** build a role from Harbor's permission set. Harbor's five roles stay as built-ins.
- **Audit log:** every write and every pull, with Harbor's audit log API, export to syslog and OTLP, and retention.

## 7. Artifacts, policies and jobs

- **Artifacts:** images, multi-arch indexes, Helm charts, cosign and Notation signatures, SBOMs, attestations, WASM, CNAI models, and any OCI artifact. Related artifacts are linked through the referrers API. Labels and search.
- **Retention:** Harbor's rules (most recent N, pushed or pulled within N days, untagged, tag and repository patterns), on a schedule, with a dry run that lists the exact artifacts it would delete.
- **Immutability:** Harbor's tag and repository pattern rules.
- **Quota:** storage per project, counted in the push transaction, so usage is always exact and an over-quota push fails at once. Harbor's documentation says usage can lag by minutes under concurrent pushes and pushes may not be rejected immediately.
- **Replication:** push and pull, manual, scheduled or event based, filters on name, tag, label and resource type, bandwidth limits. Adapters: Harbor, Distribution, Docker Hub, GHCR, ECR, Google Artifact Registry and GCR, ACR, Quay, GitLab, JFrog Artifactory, Alibaba ACR, Huawei SWR, zot, wharf. The copy engine skips blobs the target has (HEAD, then cross-repository mount), resumes partial copies, and copies blobs in parallel.
- **Vulnerability scanning:** Harbor's Pluggable Scanner API, so existing adapters for Trivy and Grype plug in. Scan on push, scheduled scan of everything, SBOM generation, system and project CVE allowlists, and a pull gate by severity. Phase 5 adds a built-in scanner.
- **Signatures:** cosign and Notation signatures stored as referrers. A project policy can require signatures for pulls. Beyond Harbor, the policy can name trusted keys, certificates or keyless identities (issuer and subject), and wharf verifies the signature before serving the pull. Harbor's documented check is that a signature verified by the chosen tool exists; its docs do not describe pinning keys.
- **Webhooks:** Harbor's event types and payloads, default and CloudEvents formats, Slack. Retries with backoff and a delivery log per endpoint.
- **P2P preheat:** Dragonfly and Kraken providers through Harbor's preheat API.

## 8. Harbor compatibility

### 8.1 Surfaces

| Surface | Promise |
| --- | --- |
| `/v2` OCI distribution API | Full spec 1.1, conformance suite in CI |
| `/service/token` | Same path and token format as Harbor |
| `/api/v2.0` | Harbor's API, endpoint by endpoint, checked against Harbor's swagger. Each feature ships with its endpoints in the same phase |
| `/api/v2.0/x-wharf/...` | Endpoints for features Harbor lacks (workload identity, custom roles, signature trust) |
| Webhook payloads | Byte-compatible with Harbor's default and CloudEvents formats |
| Storage layout | Identical to Harbor's registry, opened in place |
| Database | Not shared. `wharf import harbor` reads Harbor's PostgreSQL once |

### 8.2 Drop-in by phase

A Harbor install can move to wharf once wharf supports every feature it uses. `wharf import harbor --check` reports that before anything changes.

| After phase | Drop-in for a Harbor that uses | Not yet |
| --- | --- | --- |
| 1 | Projects, local users, members, robots, push and pull, tags, artifacts, quotas, GC, Helm OCI charts, signatures stored as artifacts | LDAP, OIDC, policies, replication, proxy cache, scanning, webhooks |
| 2 | Phase 1 plus LDAP, OIDC, groups, auth proxy, retention, immutability, labels, audit log, webhooks | HA, replication, proxy cache, scanning |
| 3 | Phase 2 plus PostgreSQL and several replicas, replication, proxy cache, preheat, the Kubernetes Helm chart | Scanning and signature gates |
| 4 | Everything Harbor 2.14 offers, apart from the non-goals above | Nothing for a supported Harbor |

### 8.3 Clients that must work unchanged

Docker, Podman, containerd and nerdctl, CRI-O, Helm, ORAS, cosign, Notation, crane, skopeo, regctl, the Harbor Terraform provider, harbor-cli, and Harbor's own replication adapter pointed at wharf.

### 8.4 Terraform provider coverage

| Resources | Phase |
| --- | --- |
| `harbor_project`, `harbor_user`, `harbor_project_member_user`, `harbor_robot_account`, `harbor_garbage_collection`, `harbor_config_system` (quota parts) | 1 |
| `harbor_project_member_group`, `harbor_group`, `harbor_config_auth`, `harbor_retention_policy`, `harbor_immutable_tag_rule`, `harbor_label`, `harbor_webhook`, `harbor_purge_audit_log` | 2 |
| `harbor_registry`, `harbor_replication`, `harbor_preheat_instance` | 3 |
| `harbor_interrogation_services`, `harbor_tasks` | 4 |

The provider's acceptance tests run against wharf in CI, limited to the resources of finished phases.

## 9. Migration from Harbor

`wharf import harbor` takes Harbor's `harbor.yml` or a database URL plus storage settings.

1. `--check` connects read-only and writes a report: Harbor version, features in use, anything wharf does not support yet, storage reachable, counts of every object.
2. The import reads Harbor's PostgreSQL and writes wharf's database: projects and metadata, users with their password hashes, groups, members, robots with their secret hashes, OIDC CLI secrets, registries, replication rules, retention, immutability, quotas, labels, webhooks, scanner registrations, preheat instances, artifacts, tags and accessories. Scan reports are re-run, not copied.
3. wharf opens Harbor's storage in place. No blob is copied.
4. Cutover: stop pushes to Harbor (read-only mode), run the import again to pick up the delta, point DNS or the load balancer at wharf. Rollback is pointing DNS back, since Harbor's data was never written.

Supported sources: Harbor 2.11 and later at first, widened as tests cover more versions. The importer grows each phase to cover that phase's features.

## 10. Operations

### 10.1 Config

- `wharf serve` with no flags: data in `./data`, SQLite, filesystem storage, a generated admin password printed once.
- A TOML file for the rest, every key overridable with `WHARF_*` environment variables. A production setup:

```toml
url = "https://registry.example.com"

[storage]
s3 = { bucket = "registry", region = "eu-west-1" }

[database]
url = "postgres://wharf@db/wharf"

[tls]
acme = { email = "ops@example.com" }
```

- Settings Harbor keeps in its database (auth mode, OIDC, LDAP, project creation rights) stay editable in the UI and through `/api/v2.0/configurations`. A setting present in the file is pinned and shown read-only.
- Built-in TLS with ACME, so no reverse proxy is needed.

### 10.2 Web UI

Server-rendered templates compiled into the binary, with htmx. No Node toolchain in the build. Page layout follows Harbor's, so people who move find things where they expect. Dark mode, keyboard navigation, every page under 100 ms on a reference machine (a target, measured in CI).

### 10.3 CLI

`wharf serve`, `wharf admin` (users, password reset, robots), `wharf gc`, `wharf config check`, `wharf import harbor [--check]`, `wharf bench`.

### 10.4 Observability

Prometheus metrics with Harbor's metric names where they exist, OpenTelemetry traces, structured JSON logs, `/healthz` and `/readyz`.

### 10.5 Deployment

Static binaries for Linux (x86_64, arm64), macOS and Windows, a distroless container image, a Helm chart (phase 3), and a systemd unit example.

## 11. Feature matrix

Legend: yes, partial, no, unverified (not confirmed from sources yet). The wharf column gives the phase that delivers the row. "Edge" says what wharf does better, or "parity" where it only matches the best.

| Feature | Harbor | zot | Distribution | Quay | Trow | wharf | Edge |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Deployment** | | | | | | | |
| Processes to run | 9 to 11 | 1 | 1 | 4+ | 1 | 1 | One process with Harbor's feature set; zot, Distribution and Trow are single too but do far less |
| External database needed | PostgreSQL, Redis | no | no | PostgreSQL, Redis | no | no (PostgreSQL optional) | Same features without Redis |
| Zero-config start | no | partial | yes | no | partial | 1 | Starts with defaults and a generated admin |
| Built-in TLS and ACME | via nginx certs | TLS only | TLS, ACME | TLS | TLS | 1 | No proxy needed |
| HA with several replicas | yes | yes | yes | yes | no | 3 | Same binary, role flag, no Redis |
| Upgrade | migrator, version hops | binary swap | binary swap | operator | binary swap | 1 | Migrations at startup, one-minor rollback |
| Windows and macOS binaries | no | partial | yes | no | no | 1 | Runs on a laptop for development |
| **Distribution** | | | | | | | |
| OCI Distribution 1.1 | yes | yes | partial (referrers unverified) | yes | partial | 1 | Parity, conformance in CI |
| Chunked and resumable uploads | yes | yes | yes | yes | yes | 1 | Parity |
| Cross-repository mount | yes | yes | yes | yes | unverified | 1 | Parity |
| Presigned redirects for pulls | yes (via Distribution) | partial | yes | yes | no | 1 | Parity |
| Storage: fs, S3, GCS, Azure | yes | fs, S3 | yes | yes | fs | fs, S3 in 1; GCS, Azure in 3 | Parity |
| Dedup across projects | yes | yes | yes | yes | yes | 1 | Parity |
| Helm charts as OCI | yes | yes | yes | yes | yes | 1 | Parity |
| Any OCI artifact | yes | yes | yes | yes | partial | 1 | Parity |
| **Tenancy and access** | | | | | | | |
| Projects or organizations | yes | no | no | yes | no | 1 | Parity with Harbor |
| Built-in roles | 5 plus admin | per-repo policies | no | 4 team roles | no | 1 | Parity, same matrix |
| Custom roles | no | no | no | no | no | 2 | New |
| Robot accounts | yes | API keys | no | yes | no | 1 | Parity, Harbor's names and permissions |
| CI workload identity (OIDC token exchange) | no | no | no | partial (unverified) | no | 2 | New for Harbor users |
| Local users | yes | htpasswd | htpasswd | yes | one user | 1 | Parity |
| LDAP and AD | yes | yes | no | yes | no | 2 | Parity |
| OIDC | yes | yes | no | yes | no | 2 | Parity |
| Several auth modes at once | no | yes | no | partial | no | 2 | Harbor users get it without a re-setup |
| HTTP auth proxy | yes | no | no | no | no | 2 | Parity |
| Quotas | yes, can lag | no | no | yes | no | 1 | Exact, enforced in the push transaction |
| Audit log | yes | yes | no | yes | no | 2 | Adds pulls, OTLP export |
| **Policies** | | | | | | | |
| Retention rules | yes | yes | no | auto-prune | no | 2 | Dry run lists exact deletions |
| Tag immutability | yes | no | no | yes | no | 2 | Parity |
| Garbage collection | online since 2.1 | yes | offline | yes | unverified | 1 | Reference counted, no full mark per run (benchmark) |
| Labels | yes | no | no | yes | no | 2 | Parity |
| **Distribution between registries** | | | | | | | |
| Push replication | yes | no | no | no | no | 3 | Skips present blobs, resumes, parallel (benchmark) |
| Pull replication and mirroring | yes | yes | no | yes | no | 3 | Same |
| Proxy cache | yes | yes | one upstream | yes | yes | 3 | Coalesced fetches, serve stale when upstream is down |
| P2P preheat | yes | no | no | no | no | 3 | Parity |
| Geo-replication of storage | no | no | no | yes | no | 5 | Active-active sites |
| **Supply chain** | | | | | | | |
| Pluggable scanners | yes | no | no | no | no | 4 | Parity, same API |
| Built-in scanner | no | yes (Trivy library) | no | Clair service | no | 5 | Better than Harbor; parity with zot |
| SBOM generation | yes | partial (unverified) | no | partial (unverified) | no | 4 | Parity |
| CVE allowlist | yes | no | no | no | no | 4 | Parity |
| Block pulls by severity | yes | no | no | no | no | 4 | Parity |
| cosign and Notation storage | yes | yes | yes | yes | partial | 1 | Parity |
| Signature gate on pull | yes, presence | partial (verifies, gate unverified) | no | no | no | 4 | Verifies against pinned keys or keyless identities |
| Kubernetes admission control | no | no | no | no | yes | 5 | Optional webhook using wharf's policies |
| **Integration** | | | | | | | |
| Harbor `/api/v2.0` | yes | no | no | no | no | 1 to 4 | Only drop-in alternative |
| Harbor Terraform provider | yes | no | no | no | no | 1 to 4 | Same |
| Import from Harbor in place | n/a | no | no | no | no | 1 to 4 | No blob copy, no credential rotation |
| Webhooks | yes | partial (unverified) | yes | yes | no | 2 | Delivery log, retries, Harbor payloads |
| Prometheus metrics | yes | yes | yes | yes | partial | 1 | Parity, Harbor metric names |
| OpenTelemetry tracing | yes | unverified | yes | unverified | no | 1 | Parity |
| Web UI | yes | yes | no | yes | no | 1 | Server rendered, no Node build, page time target |
| Image builds | no | no | no | yes | no | no | Non-goal |
| Lazy-pull indexes (SOCI, eStargz) | partial (separate service) | no | no | no | no | 5 | Built in |
| Per-project usage reports | partial | no | no | partial | no | 5 | New |

The "unverified" cells get checked against each project's current docs before the matrix is published in the README, and the first benchmark round fills in the numbers below.

## 12. Benchmarks

"Better" on performance means a number from `bench/`, run the same way against every registry. Until a number exists, the claim is called a target.

Method: one VM size (4 vCPU, 8 GB RAM, local NVMe), one MinIO for object storage, each registry at its documented default production setup, the same image set (small, medium, a 5 GB layer, a 200-arch index), the client load generated by `wharf bench` and checked with k6. Scripts and raw results are committed.

| Claim | Metric | Target |
| --- | --- | --- |
| Lighter | RSS at idle, all processes summed | under 50 MB for wharf; Harbor measured, not assumed |
| Lighter under load | RSS at 200 concurrent pulls | flat within 2x of idle |
| Faster pulls | manifest GET p50 and p99 at 2,000 req/s | p99 under 5 ms on SQLite |
| Faster pushes | throughput for the 5 GB layer, 32 parallel pushes | at disk or network limit |
| Faster setup | time from download to first successful push | under 1 minute |
| Shorter config | lines of config for TLS, S3 and PostgreSQL | under 10 |
| Smaller | binary and container image size | under 40 MB image |
| Quicker GC | time to collect 100k orphaned blobs among 1M | measured against Harbor and zot |
| Quicker replication | time to replicate 1,000 images that share base layers | measured against Harbor |
| Faster cold start | process start to ready | under 1 second on SQLite |

## 13. Testing

- Unit tests with every change, written first. Integration tests against real SQLite and PostgreSQL. No network outside test containers.
- OCI Distribution conformance suite on every pull request.
- `/api/v2.0` checked against Harbor's swagger file for paths, parameters and response shapes.
- Harbor Terraform provider acceptance tests for finished resources.
- Client matrix end to end: docker, podman, nerdctl, helm, oras, cosign, notation, crane, skopeo, regctl.
- Importer tests against real Harbor databases for each supported version, built in CI from Harbor's installer.
- Fuzzing for manifest, index and token parsing.
- Build and unit tests on Linux, macOS and Windows; integration and end-to-end tests on Linux.

## 14. Phases

Each phase ends with a release, its exit checks green, and the importer covering its features.

### Phase 1: registry core (MVP)

OCI 1.1 with conformance, Distribution storage layout on filesystem and S3, SQLite, token service, local users, projects, members, Harbor's roles, robot accounts, quotas, reference-counted GC, `/api/v2.0` for these, the web UI for browsing and project administration, the CLI, metrics and tracing, `wharf import harbor` for projects, users, members, robots, artifacts and tags.

Exit: conformance suite passes; the client matrix passes; phase 1 Terraform resources pass; a Harbor 2.14 with projects, users and robots imports in place and serves pulls with the old credentials.

### Phase 2: identity and policy

LDAP and AD, OIDC with CLI secrets, groups, HTTP auth proxy, several modes at once, workload identity, custom roles, retention, immutability, labels, audit log, webhooks, configuration API.

Exit: phase 2 Terraform resources pass; webhook payloads match Harbor fixtures byte for byte; an OIDC-backed Harbor imports and logs users in without changes.

### Phase 3: scale and distribution

PostgreSQL, `--role api|worker|all`, GCS and Azure storage, the Helm chart, replication with every adapter, proxy cache, P2P preheat.

Exit: three replicas behind a load balancer survive a rolling restart under load; Harbor replicates to wharf and back with Harbor's own adapter; the replication and proxy benchmarks are published.

### Phase 4: supply chain and parity

Pluggable Scanner API, scan on push and on schedule, SBOMs, allowlists, severity gate, signature gate with verification.

Exit: every row in section 11 that Harbor has is "yes" for wharf; the full Terraform provider suite passes for every resource outside the non-goals; `import harbor --check` reports nothing unsupported for a Harbor 2.14 using all features.

### Phase 5: beyond

Built-in scanner (SBOM matched against OSV), SOCI and eStargz indexes, Kubernetes admission webhook, per-project usage reports, active-active geo-replication.

Exit: the built-in scanner's findings match Trivy on a published corpus within a stated tolerance; the geo-replication design gets its own spec before work starts.

## 15. Risks

- **Harbor's API is large.** Several hundred operations. Mitigation: generate request and response types from Harbor's swagger, and gate each phase only on the endpoints its features need.
- **Harbor's schema moves between versions.** Mitigation: the importer has one reader per supported Harbor minor version, each tested against a real database from that version.
- **Notation has no mature Rust library.** Verifying Notation signatures (JWS and COSE envelopes, X.509 trust) may need an implementation of our own. sigstore-rs covers cosign but has to be checked for keyless support.
- **SQLite and many concurrent pushes.** One writer at a time. Mitigation: short transactions, blob writes outside the transaction, and PostgreSQL for heavy installs. The benchmark shows where the line is.
- **A built-in scanner is a product of its own.** It is kept to phase 5, and the pluggable API covers scanning before then.
- **"Better" claims could be wrong.** Every performance claim is a target until the benchmark shows it, and results are published even when wharf loses.

## 16. Sources

- Harbor releases and support: https://endoflife.date/harbor, https://github.com/goharbor/harbor
- Harbor GC: https://goharbor.io/docs/2.2.0/administration/garbage-collection/
- Harbor quota lag: https://github.com/goharbor/perf/wiki/Quota-Update, https://goharbor.io/docs/2.0.0/administration/configure-project-quotas
- Harbor content trust: https://goharbor.io/docs/edge/working-with-projects/project-configuration/implementing-content-trust/
- Harbor memory reports: https://knowledge.broadcom.com/external/article/440690/harbor-replication-is-failing-with-harbo.html, https://phabricator.wikimedia.org/T342563
- Harbor upgrade failure: https://github.com/goharbor/harbor/issues/15538
- zot: https://github.com/project-zot/zot/releases
- Distribution v3: https://github.com/distribution/distribution/releases
- Quay 3.17: https://docs.redhat.com/en/documentation/red_hat_quay/3/html/red_hat_quay_release_notes/release-notes-317
- Trow: https://github.com/Trow-Registry/trow
- OCI 1.1: https://opencontainers.org/posts/blog/2024-03-13-image-and-distribution-1-1/
- OCI conformance redesign: https://opencontainers.org/posts/blog/2026-04-04-distribution-spec-conformance/
