# Phase 1a: Registry Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A single `wharf` binary that passes the OCI Distribution Spec conformance suite and works with every client in the phase 1 matrix, with one local admin user, SQLite metadata, filesystem or S3 storage in Distribution's layout, reference-counted garbage collection, TLS with ACME, metrics and tracing.

**Architecture:** A Cargo workspace of small crates (`oci`, `storage`, `meta`, `auth`, `jobs`, `http`, `wharf`) as laid out in section 5.1 of the spec. Uploads spool to local disk while hashing, then move to the blob store. SQLite holds repositories, blobs with reference counts, manifests, tags and users; storage holds blob bytes and Distribution's link files.

**Tech Stack:** Rust 1.94 (edition 2024), tokio 1.54, axum 0.8, tower-http 0.7, sqlx 0.9 (runtime queries, SQLite), object_store 0.14, oci-spec 0.10, sha2 0.11, argon2 0.6, jsonwebtoken 12, p256 0.14, figment 0.10, clap 4.6, prometheus-client 0.26, tracing-opentelemetry 0.34, axum-server 0.9 with rustls, rustls-acme.

**Spec:** `docs/design.md`. Sections 5.1 to 5.5, 5.8, 6.3, 10 and 13 apply to this plan.

**Scope:** This is the first of four plans for phase 1. Projects, members, roles, robots, quotas and `/api/v2.0` come in 1b; the web UI in 1c; `wharf import harbor` in 1d. Until 1b, the only principal with access is the local admin; anonymous requests get nothing.

## Global Constraints

- Blob paths are exactly `docker/registry/v2/blobs/<algo>/<first two hex>/<hex>/data` (spec 5.2).
- Link files are exactly Distribution's: `docker/registry/v2/repositories/<name>/_layers/<algo>/<hex>/link`, `.../_manifests/revisions/<algo>/<hex>/link`, `.../_manifests/tags/<tag>/current/link`, `.../_manifests/tags/<tag>/index/<algo>/<hex>/link`. Each holds the digest string, for example `sha256:abc...`, with no newline.
- The token endpoint is `/service/token` (spec 6.3). Tokens are ES256 JWTs.
- GC grace window default: 2 hours (spec 5.8).
- No `query!` macros: builds must not need `DATABASE_URL`.
- Every test passes on Linux, macOS and Windows. Paths go through `std::path` or `object_store::path::Path`, never string concatenation with `/` for local files.
- Library crates never print and never read environment variables; only the `wharf` crate does.
- Tests first, no network outside the e2e and conformance jobs, no bare sleeps (use `tokio::time::pause` or channels).
- No comments that restate code. Comments only for why.
- Commit messages follow Conventional Commits. One branch and one pull request per task, named `feat/<topic>`, `ci/<topic>` or `chore/<topic>`, closing that task's issue.
- Never mention AI, assistants or code generation anywhere. No `Co-Authored-By` or `Generated with` trailers. No em-dashes.

## Review Focus

1. A client that drops the connection halfway through a PATCH: the upload status must report exactly the bytes on disk, and a later PATCH from that offset plus PUT must succeed with the right digest. Test in Task 4.
2. Two pushes of the same tag or the same blob at the same moment: no duplicate rows, reference counts equal to the real number of references. Test in Task 5.
3. The process restarts between PATCH and PUT: a new `Uploads` over the same spool directory finishes the upload by rehashing the spooled file. Test in Task 4.
4. A manifest that references a blob pushed only to another repository: rejected with `MANIFEST_BLOB_UNKNOWN`, so content does not leak across repositories without a mount. Test in Task 5.
5. GC racing a push: a blob listed as collectable that a new manifest references before deletion must survive. Test in Task 9.

---

### Task 1: Workspace, toolchain and CI

**Files:**
- Create: `Cargo.toml` (workspace), `rust-toolchain.toml`, `rustfmt.toml`, `clippy.toml`, `deny.toml`, `.gitignore`, `.gitattributes`, `CHANGELOG.md` (Keep a Changelog, with an `Unreleased` section)
- Create: `crates/wharf/Cargo.toml`, `crates/wharf/src/main.rs`, `crates/wharf/tests/cli.rs`
- Create: empty library crates `crates/{oci,storage,meta,auth,jobs,http}` with `src/lib.rs`
- Create: `.github/workflows/ci.yml`
- Modify: `AGENTS.md` (commands section)

**Interfaces:**
- Produces: the workspace every later task adds to; CI jobs named `fmt`, `clippy`, `test (ubuntu-latest)`, `test (macos-latest)`, `test (windows-latest)`, `deny`, which the ruleset on `main` requires.

- [ ] **Step 1: Write the failing test** in `crates/wharf/tests/cli.rs`

```rust
#[test]
fn version_prints_name_and_version() {
    let out = assert_cmd::Command::cargo_bin("wharf").unwrap().arg("--version").output().unwrap();
    assert!(out.status.success());
    assert_eq!(String::from_utf8(out.stdout).unwrap(), format!("wharf {}\n", env!("CARGO_PKG_VERSION")));
}
```

- [ ] **Step 2: Run it and see it fail**

Run: `cargo test -p wharf --test cli`
Expected: FAIL, the binary does not exist yet.

- [ ] **Step 3: Create the workspace**

Workspace `resolver = "3"`, `edition = "2024"`, `rust-version = "1.94"`, `license = "Apache-2.0"`, shared `[workspace.dependencies]` with the versions in the header, `[workspace.lints]` with `unsafe_code = "forbid"` and clippy `pedantic` at warn. `rust-toolchain.toml` pins `1.94` with `rustfmt` and `clippy`. `.gitattributes` sets `* text=auto eol=lf`. `main.rs` uses clap with `#[command(version)]` and one `serve` subcommand stub that returns an error "not implemented".

- [ ] **Step 4: Run it and see it pass**

Run: `cargo test -p wharf --test cli`
Expected: PASS.

- [ ] **Step 5: Add CI**

`ci.yml` on `pull_request` and `push` to `main`, `permissions: contents: read`, `concurrency` cancelling older runs per ref. Jobs: `fmt` (`cargo fmt --all --check`), `clippy` (`cargo clippy --workspace --all-targets -- -D warnings`), `test` matrix over `ubuntu-latest`, `macos-latest`, `windows-latest` (`cargo test --workspace`), `deny` (`cargo deny check` via `EmbarkStudios/cargo-deny-action`). Use `Swatinem/rust-cache`. Pin every action to a full commit SHA with the version in a trailing comment. Also add `.github/workflows/codeql.yml` with one job named `analyze` for `rust`, on pull requests, pushes to main and weekly.

- [ ] **Step 6: Record the commands in `AGENTS.md`, commit, open the PR**

```bash
git add Cargo.toml rust-toolchain.toml rustfmt.toml clippy.toml deny.toml .gitignore .gitattributes CHANGELOG.md crates .github/workflows/ci.yml .github/workflows/codeql.yml AGENTS.md
git commit -m "chore: set up the cargo workspace and CI"
```

After the PR merges, add the six CI checks and `analyze` to the `main` ruleset as required status checks, next to `pr title` and `dependency review`.

---

### Task 2: Config

**Files:**
- Create: `crates/wharf/src/config.rs`
- Test: `crates/wharf/src/config.rs` (unit tests module)

**Interfaces:**
- Produces:

```rust
pub struct Config {
    pub url: Option<url::Url>,           // public base URL; defaults to http://<listen>
    pub listen: std::net::SocketAddr,    // default 0.0.0.0:5000
    pub data_dir: std::path::PathBuf,    // default ./data
    pub storage: StorageConfig,
    pub database: DatabaseConfig,
    pub tls: Option<TlsConfig>,
    pub gc: GcConfig,
    pub telemetry: TelemetryConfig,
}
pub enum StorageConfig { Fs { root: Option<PathBuf> }, S3 { bucket: String, region: Option<String>, endpoint: Option<Url>, prefix: Option<String> } }
pub struct DatabaseConfig { pub url: Option<String> }            // default sqlite://<data_dir>/wharf.db
pub enum TlsConfig { Files { cert: PathBuf, key: PathBuf }, Acme { email: String, domains: Vec<String>, directory: Option<Url> } }
pub struct GcConfig { pub grace: humantime_serde::Duration, pub interval: humantime_serde::Duration } // 2h, 1h
pub struct TelemetryConfig { pub otlp_endpoint: Option<Url> }
pub fn load(path: Option<&Path>, env: &[(String, String)]) -> Result<Config, ConfigError>;
```

`env` is passed in so tests never touch the process environment. Keys map as `WHARF_STORAGE__S3__BUCKET` (figment `Env::prefixed("WHARF_").split("__")`). Unknown keys are an error naming the key. `WHARF_ADMIN_PASSWORD` is not a config key: the env provider skips it, and `serve` reads it separately.

- [ ] **Step 1: Write the failing tests**

```rust
#[test] fn no_file_no_env_gives_defaults() {
    let c = load(None, &[]).unwrap();
    assert_eq!(c.listen, "0.0.0.0:5000".parse().unwrap());
    assert_eq!(c.data_dir, PathBuf::from("data"));
    assert!(matches!(c.storage, StorageConfig::Fs { root: None }));
    assert_eq!(*c.gc.grace, Duration::from_secs(7200));
}
#[test] fn env_overrides_file() { /* file sets listen 127.0.0.1:6000, env WHARF_LISTEN=127.0.0.1:7000, expect 7000 */ }
#[test] fn s3_table_parses() { /* [storage] s3 = { bucket = "registry", region = "eu-west-1" } */ }
#[test] fn unknown_key_is_rejected() { /* file with `lisen = "x"` -> Err whose Display contains "lisen" */ }
#[test] fn spec_example_parses() { /* the TOML block from spec section 10.1 verbatim */ }
#[test] fn admin_password_env_is_not_a_config_key() { /* env WHARF_ADMIN_PASSWORD=x -> Ok */ }
```

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p wharf config`
Expected: FAIL to compile, `load` missing.

- [ ] **Step 3: Implement `load` in `config.rs`** with figment: `Serialized::defaults`, then `Toml::file` when given, then the env provider built from the passed pairs. Serde `deny_unknown_fields` on every struct.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p wharf config`
Expected: PASS, 6 tests.

- [ ] **Step 5: Commit**

```bash
git add crates/wharf/src/config.rs crates/wharf/src/main.rs crates/wharf/Cargo.toml
git commit -m "feat: load config from TOML and WHARF_ variables"
```

---

### Task 3: OCI names, digests and manifests

**Files:**
- Create: `crates/oci/src/{lib.rs,digest.rs,name.rs,reference.rs,manifest.rs,error.rs}`
- Create: `crates/oci/tests/fixtures/` (manifests: OCI image, OCI index, Docker schema2, Docker list, artifact with subject and artifactType, image with a foreign layer, schema1)
- Test: `crates/oci/tests/{digest.rs,name.rs,manifest.rs}`

**Interfaces:**
- Produces:

```rust
pub enum Algorithm { Sha256, Sha512 }
pub struct Digest { /* algorithm + lowercase hex */ }
impl Digest {
    pub fn sha256_of(bytes: &[u8]) -> Digest;
    pub fn algorithm(&self) -> Algorithm;
    pub fn hex(&self) -> &str;
}
impl FromStr for Digest   // Err = DigestError
impl Display for Digest   // "sha256:<hex>"
pub struct RepoName(String);   // FromStr, Display, as_str()
pub struct Tag(String);        // FromStr, Display, as_str()
pub enum Reference { Tag(Tag), Digest(Digest) }   // FromStr: digest if it contains ':'
pub enum ManifestKind { Image, Index }
pub struct Descriptor { pub media_type: String, pub digest: Digest, pub size: u64, pub artifact_type: Option<String> }
pub struct ParsedManifest {
    pub kind: ManifestKind,
    pub media_type: String,
    pub artifact_type: Option<String>,
    pub subject: Option<Descriptor>,
    pub references: Vec<Descriptor>,    // config, layers or child manifests; foreign layers left out
    pub annotations: serde_json::Map<String, serde_json::Value>,
}
pub const MAX_MANIFEST_BYTES: usize = 4 * 1024 * 1024;
pub fn parse_manifest(content_type: &str, body: &[u8]) -> Result<ParsedManifest, ManifestError>;
pub enum ErrorCode { BlobUnknown, BlobUploadInvalid, BlobUploadUnknown, DigestInvalid, ManifestBlobUnknown, ManifestInvalid, ManifestUnknown, NameInvalid, NameUnknown, SizeInvalid, Unauthorized, Denied, Unsupported, TooManyRequests, RangeInvalid }
impl ErrorCode { pub fn as_str(&self) -> &'static str; pub fn status(&self) -> u16; }
```

Values the spec pins: repository names match `[a-z0-9]+((\.|_|__|-+)[a-z0-9]+)*(\/[a-z0-9]+((\.|_|__|-+)[a-z0-9]+)*)*` and are at most 255 bytes; tags match `[a-zA-Z0-9_][a-zA-Z0-9._-]{0,127}`; sha256 hex is 64 chars, sha512 is 128, lowercase only. Accepted manifest media types: `application/vnd.oci.image.manifest.v1+json`, `application/vnd.oci.image.index.v1+json`, `application/vnd.docker.distribution.manifest.v2+json`, `application/vnd.docker.distribution.manifest.list.v2+json`. Schema1 types are `Unsupported`. When the body's `mediaType` field is present and differs from the Content-Type, it is `ManifestInvalid`. Foreign layer types left out of `references`: `application/vnd.docker.image.rootfs.foreign.diff.tar.gzip` and the three `application/vnd.oci.image.layer.nondistributable.*` types. `ErrorCode::status`: 404 for the `*Unknown` codes, 401 `Unauthorized`, 403 `Denied`, 405 `Unsupported`, 416 `RangeInvalid`, 429 `TooManyRequests`, 400 for the rest.

- [ ] **Step 1: Write the failing tests**

```rust
#[test] fn digest_round_trips() { let s = format!("sha256:{}", "a".repeat(64)); assert_eq!(s.parse::<Digest>().unwrap().to_string(), s); }
#[test] fn digest_rejects_uppercase_short_and_unknown_algo() { for s in ["sha256:ABC...", "sha256:abc", "md5:..."] { assert!(s.parse::<Digest>().is_err()) } }
#[test] fn sha256_of_empty() { assert_eq!(Digest::sha256_of(b"").to_string(), "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"); }
#[test] fn names() { for ok in ["a", "library/nginx", "a.b_c__d-e/f--g"] { assert!(ok.parse::<RepoName>().is_ok()) } for bad in ["A", "a/", "/a", "a..b", "a___b", &"a".repeat(256)] { assert!(bad.parse::<RepoName>().is_err()) } }
#[test] fn tags() { assert!("v1.0_rc-1".parse::<Tag>().is_ok()); assert!(".x".parse::<Tag>().is_err()); assert!("a".repeat(129).parse::<Tag>().is_err()); }
#[test] fn image_manifest_references_config_then_layers() { /* fixture, expect kind Image and references in order */ }
#[test] fn index_references_children() { /* fixture */ }
#[test] fn artifact_subject_and_type() { /* fixture, subject Some, artifact_type Some("application/vnd.example+type") */ }
#[test] fn foreign_layer_is_left_out() { /* fixture with one foreign layer, references.len() == 2 */ }
#[test] fn schema1_is_unsupported() { /* Err(ManifestError::Unsupported) */ }
#[test] fn media_type_mismatch_is_invalid() { /* OCI body, docker content type */ }
#[test] fn oversized_manifest_is_invalid() { /* MAX_MANIFEST_BYTES + 1 bytes */ }
```

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p oci`
Expected: FAIL to compile.

- [ ] **Step 3: Implement the modules.** Parse manifests with `oci-spec` types for OCI and small serde structs for the two Docker types. Regexes compiled once with `std::sync::LazyLock`.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p oci`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/oci
git commit -m "feat: parse OCI names, digests and manifests"
```

---

### Task 4: Blob store and upload spool

**Files:**
- Create: `crates/storage/src/{lib.rs,layout.rs,blobs.rs,uploads.rs}`
- Test: `crates/storage/tests/{layout.rs,blobs.rs,uploads.rs}`

**Interfaces:**
- Consumes: `oci::{Digest, RepoName, Tag}`.
- Produces:

```rust
pub mod layout {
    pub fn blob_data(d: &Digest) -> object_store::path::Path;
    pub fn layer_link(repo: &RepoName, d: &Digest) -> Path;
    pub fn revision_link(repo: &RepoName, d: &Digest) -> Path;
    pub fn tag_current_link(repo: &RepoName, tag: &Tag) -> Path;
    pub fn tag_index_link(repo: &RepoName, tag: &Tag, d: &Digest) -> Path;
}
pub struct BlobStore { /* Arc<dyn ObjectStore>, presign capability */ }
impl BlobStore {
    pub fn new(store: Arc<dyn ObjectStore>, signer: Option<Arc<dyn object_store::signer::Signer>>) -> Self;
    pub async fn stat(&self, d: &Digest) -> Result<Option<u64>, StorageError>;
    pub async fn get(&self, d: &Digest, range: Option<Range<u64>>) -> Result<BlobBody, StorageError>; // BlobBody { size: u64, stream: BoxStream<'static, Result<Bytes, StorageError>> }
    pub async fn presign_get(&self, d: &Digest, ttl: Duration) -> Result<Option<url::Url>, StorageError>;
    pub async fn put_bytes(&self, d: &Digest, bytes: Bytes) -> Result<(), StorageError>;
    pub async fn delete(&self, d: &Digest) -> Result<(), StorageError>;
    pub async fn write_link(&self, path: &Path, d: &Digest) -> Result<(), StorageError>;
    pub async fn delete_link(&self, path: &Path) -> Result<(), StorageError>;
}
pub struct UploadId(uuid::Uuid);   // FromStr, Display
pub struct Uploads { /* spool dir + Mutex<HashMap<UploadId, (Sha256, u64)>> */ }
impl Uploads {
    pub fn new(spool_dir: PathBuf) -> std::io::Result<Self>;
    pub async fn start(&self) -> Result<UploadId, StorageError>;
    pub async fn append(&self, id: &UploadId, offset: u64, body: impl Stream<Item = Result<Bytes, std::io::Error>> + Send) -> Result<u64, StorageError>;
    pub async fn status(&self, id: &UploadId) -> Result<u64, StorageError>;
    pub async fn finish(&self, id: &UploadId, expected: &Digest, store: &BlobStore) -> Result<u64, StorageError>;
    pub async fn cancel(&self, id: &UploadId) -> Result<(), StorageError>;
    pub async fn sweep_stale(&self, older_than: Duration) -> Result<usize, StorageError>;
}
pub enum StorageError { NotFound, UploadUnknown, RangeMismatch { expected: u64 }, DigestMismatch { actual: Digest }, Io(..), Store(..) }
```

`append` checks `offset == current size`, else `RangeMismatch`. If the body stream fails partway, the bytes written so far stay, the file is flushed, and the in-memory hasher for that upload is dropped, so `finish` rehashes the file. `finish` with no in-memory hasher rehashes the file. On a `LocalFileSystem` store, `finish` renames when the spool and store share a volume and copies otherwise; on any other store it streams the file as a multipart upload. Only sha256 uploads are accepted in phase 1a; a sha512 `expected` returns `DigestMismatch` after hashing with sha512.

- [ ] **Step 1: Write the failing tests**

```rust
#[test] fn blob_path_matches_distribution() {
    let d: Digest = format!("sha256:ab{}", "c".repeat(62)).parse().unwrap();
    assert_eq!(layout::blob_data(&d).as_ref(), format!("docker/registry/v2/blobs/sha256/ab/ab{}/data", "c".repeat(62)));
}
#[test] fn tag_links_match_distribution() { /* exact strings for current and index links of repo "library/nginx", tag "1.27" */ }
#[tokio::test] async fn chunked_upload_then_finish() { /* append 0..3 then 3..6 of "abcdef", finish with sha256_of("abcdef"), stat == Some(6), file exists at blob_data path in a LocalFileSystem tempdir */ }
#[tokio::test] async fn wrong_offset_is_range_mismatch() { /* append at 2 when size 0 -> RangeMismatch { expected: 0 } */ }
#[tokio::test] async fn wrong_digest_is_rejected_and_nothing_is_stored() { /* finish with sha256_of("x") -> DigestMismatch, stat None */ }
#[tokio::test] async fn dropped_connection_keeps_written_bytes() { /* stream yields "abc" then io error -> append Err, status == 3, append "def" at 3, finish sha256_of("abcdef") ok */ }
#[tokio::test] async fn finish_after_restart_rehashes_spool() { /* append with one Uploads, build a new Uploads on the same dir, finish ok */ }
#[tokio::test] async fn range_get() { /* put "abcdef", get 2..4 yields "cd", size 6 */ }
#[tokio::test] async fn presign_is_none_for_local_fs() {}
#[tokio::test] async fn sweep_removes_only_old_uploads() { /* use filetime to age one spool file */ }
```

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p storage`
Expected: FAIL to compile.

- [ ] **Step 3: Implement `layout`, `BlobStore` and `Uploads`.** Spool files live at `<spool_dir>/<uuid>`; hashing with `sha2::Sha256` as chunks are written.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p storage`
Expected: PASS on all three systems in CI.

- [ ] **Step 5: Commit**

```bash
git add crates/storage
git commit -m "feat: blob store in distribution layout with a hashing upload spool"
```

---

### Task 5: Metadata store

**Files:**
- Create: `crates/meta/src/{lib.rs,repos.rs,blobs.rs,manifests.rs,tags.rs,users.rs,gc.rs}`
- Create: `crates/meta/migrations/sqlite/0001_init.sql`
- Test: `crates/meta/tests/{manifests.rs,tags.rs,users.rs,gc.rs}`

**Interfaces:**
- Consumes: `oci::{Digest, RepoName, Tag, Reference}`.
- Produces:

```rust
#[derive(Clone)] pub struct Meta { /* SqlitePool */ }
impl Meta {
    pub async fn open(url: &str) -> Result<Meta, MetaError>;            // creates the file, WAL, foreign_keys=ON, runs migrations
    pub async fn ensure_repository(&self, name: &RepoName) -> Result<RepoId, MetaError>;
    pub async fn repository(&self, name: &RepoName) -> Result<Option<RepoId>, MetaError>;
    pub async fn repositories(&self, last: Option<&str>, n: Option<u32>) -> Result<Vec<String>, MetaError>;
    pub async fn record_blob(&self, d: &Digest, size: u64) -> Result<(), MetaError>;           // insert or ignore; new rows get zero_since = now
    pub async fn link_blob(&self, repo: RepoId, d: &Digest) -> Result<(), MetaError>;
    pub async fn linked_blob(&self, repo: RepoId, d: &Digest) -> Result<Option<u64>, MetaError>; // size if linked
    pub async fn put_manifest(&self, repo: RepoId, m: &NewManifest, tag: Option<&Tag>) -> Result<(), MetaError>;
    pub async fn manifest(&self, repo: RepoId, r: &Reference) -> Result<Option<ManifestRow>, MetaError>;
    pub async fn delete_manifest(&self, repo: RepoId, d: &Digest) -> Result<bool, MetaError>;
    pub async fn delete_tag(&self, repo: RepoId, tag: &Tag) -> Result<bool, MetaError>;
    pub async fn tags(&self, repo: RepoId, last: Option<&str>, n: Option<u32>) -> Result<Vec<String>, MetaError>;
    pub async fn referrers(&self, repo: RepoId, subject: &Digest, artifact_type: Option<&str>) -> Result<Vec<ReferrerRow>, MetaError>;
    pub async fn collectable_blobs(&self, zero_before: time::OffsetDateTime) -> Result<Vec<(Digest, u64)>, MetaError>;
    pub async fn forget_blob(&self, d: &Digest, zero_before: time::OffsetDateTime) -> Result<bool, MetaError>;
    pub async fn create_user(&self, name: &str, password_hash: &str, admin: bool) -> Result<UserId, MetaError>;
    pub async fn user_by_name(&self, name: &str) -> Result<Option<UserRow>, MetaError>;
    pub async fn set_password(&self, name: &str, password_hash: &str) -> Result<bool, MetaError>;
    pub async fn user_count(&self) -> Result<u64, MetaError>;
}
pub struct NewManifest { pub digest: Digest, pub media_type: String, pub artifact_type: Option<String>, pub subject: Option<Digest>, pub size: u64, pub kind: oci::ManifestKind, pub references: Vec<Digest>, pub annotations: serde_json::Map<String, serde_json::Value> }
pub struct ManifestRow { pub digest: Digest, pub media_type: String, pub size: u64, pub subject: Option<Digest> }
pub struct ReferrerRow { pub digest: Digest, pub media_type: String, pub artifact_type: Option<String>, pub size: u64, pub annotations: serde_json::Map<String, serde_json::Value> }
pub struct UserRow { pub id: UserId, pub name: String, pub password_hash: String, pub admin: bool }
pub enum MetaError { MissingReferences(Vec<Digest>), Db(sqlx::Error), .. }
```

Schema for `0001_init.sql`:

```sql
CREATE TABLE repositories (id INTEGER PRIMARY KEY, name TEXT NOT NULL UNIQUE, created_at INTEGER NOT NULL);
CREATE TABLE blobs (digest TEXT PRIMARY KEY, size INTEGER NOT NULL, refcount INTEGER NOT NULL DEFAULT 0, zero_since INTEGER, created_at INTEGER NOT NULL);
CREATE TABLE repository_blobs (repository_id INTEGER NOT NULL REFERENCES repositories(id) ON DELETE CASCADE, digest TEXT NOT NULL REFERENCES blobs(digest), PRIMARY KEY (repository_id, digest));
CREATE TABLE manifests (id INTEGER PRIMARY KEY, repository_id INTEGER NOT NULL REFERENCES repositories(id) ON DELETE CASCADE, digest TEXT NOT NULL REFERENCES blobs(digest), media_type TEXT NOT NULL, artifact_type TEXT, subject TEXT, size INTEGER NOT NULL, annotations TEXT NOT NULL DEFAULT '{}', pushed_at INTEGER NOT NULL, UNIQUE (repository_id, digest));
CREATE INDEX manifests_subject ON manifests (repository_id, subject);
CREATE TABLE manifest_refs (manifest_id INTEGER NOT NULL REFERENCES manifests(id) ON DELETE CASCADE, digest TEXT NOT NULL REFERENCES blobs(digest), PRIMARY KEY (manifest_id, digest));
CREATE TABLE tags (repository_id INTEGER NOT NULL REFERENCES repositories(id) ON DELETE CASCADE, name TEXT NOT NULL, manifest_id INTEGER NOT NULL REFERENCES manifests(id) ON DELETE CASCADE, updated_at INTEGER NOT NULL, PRIMARY KEY (repository_id, name));
CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL UNIQUE, password_hash TEXT NOT NULL, admin INTEGER NOT NULL DEFAULT 0, created_at INTEGER NOT NULL);
```

Reference counting, all inside the transaction that changes the references: a blob's `refcount` is the number of `manifests` rows with that digest plus the number of `manifest_refs` rows with that digest. When it goes to 0, set `zero_since = now`; when it leaves 0, set `zero_since = NULL`. `subject` is not counted. `put_manifest` checks, in the same transaction, that every reference of an `Image` is in `repository_blobs` for that repository and every reference of an `Index` is a manifest in that repository; missing ones return `MissingReferences` with the list. Pushing an existing manifest digest again is a no-op except for the tag. Moving a tag updates its `manifest_id` and `updated_at`. `delete_manifest` removes the tags pointing at it. `tags` and `repositories` sort by byte order and page with `last` (exclusive) and `n`. `forget_blob` deletes the row only if `refcount = 0 AND zero_since < zero_before`, in one statement, and returns whether it did.

- [ ] **Step 1: Write the failing tests** (each on a fresh `sqlite::memory:` or tempfile database)

```rust
#[tokio::test] async fn image_needs_its_blobs_linked() { /* put_manifest with an unlinked layer -> MissingReferences(vec![layer]) */ }
#[tokio::test] async fn blob_in_another_repo_is_not_enough() { /* link layer in repo a, put manifest in repo b -> MissingReferences */ }
#[tokio::test] async fn index_needs_child_manifests() {}
#[tokio::test] async fn refcounts_follow_manifests() { /* layer shared by manifests in two repos: refcount 2; delete one: 1; delete other: 0 and zero_since set */ }
#[tokio::test] async fn same_tag_pushed_concurrently() { /* 16 tasks put_manifest different digests on tag "latest" with try_join; exactly one tag row, every blob refcount equals its real reference count */ }
#[tokio::test] async fn same_blob_recorded_concurrently() { /* 16 record_blob calls, one row */ }
#[tokio::test] async fn tag_listing_pages() { /* tags b,a,c -> n=2: [a,b]; last=b: [c] */ }
#[tokio::test] async fn referrers_filter_by_artifact_type() {}
#[tokio::test] async fn delete_manifest_drops_its_tags() {}
#[tokio::test] async fn forget_blob_only_when_still_zero() { /* collectable, then put a manifest referencing it, forget_blob -> false and row remains */ }
#[tokio::test] async fn users_round_trip() {}
```

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p meta`
Expected: FAIL to compile.

- [ ] **Step 3: Implement `Meta`** with `sqlx::query` and `query_as` plus `FromRow`, `sqlx::migrate!("migrations/sqlite")`, `BEGIN IMMEDIATE` for write transactions so concurrent writers queue instead of failing with `SQLITE_BUSY`, and `busy_timeout` of 5 seconds.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p meta`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/meta
git commit -m "feat: sqlite metadata with reference-counted blobs"
```

---

### Task 6: Users, passwords and tokens

**Files:**
- Create: `crates/auth/src/{lib.rs,password.rs,token.rs,scope.rs,key.rs}`
- Test: `crates/auth/tests/{password.rs,token.rs,scope.rs}`

**Interfaces:**
- Consumes: `meta::Meta` (`user_by_name`, `create_user`, `user_count`, `set_password`).
- Produces:

```rust
pub fn hash_password(plain: &str) -> String;                 // Argon2id, PHC string
pub fn verify_password(plain: &str, phc: &str) -> bool;
pub struct SigningKey { /* p256 ES256 key + kid */ }
impl SigningKey { pub fn load_or_create(path: &Path) -> std::io::Result<SigningKey>; }
pub enum Principal { Anonymous, User { name: String, admin: bool } }
pub struct Scope { pub kind: String, pub name: String, pub actions: Vec<String> }   // FromStr "repository:library/nginx:pull,push"
pub struct TokenResponse { pub token: String, pub access_token: String, pub expires_in: u64, pub issued_at: String }
pub struct Claims { pub sub: String, pub access: Vec<Scope>, /* iss, aud, exp, nbf, iat, jti */ }
#[derive(Clone)] pub struct Auth { /* meta, key, issuer, service, ttl */ }
impl Auth {
    pub fn new(meta: Meta, key: SigningKey, issuer: String, service: String, ttl: Duration) -> Auth;
    pub async fn authenticate_basic(&self, authorization: &str) -> Result<Principal, AuthError>;   // "Basic base64(user:pass)"; bad credentials -> Err(Unauthorized)
    pub fn grant(&self, who: &Principal, requested: &[Scope]) -> Vec<Scope>;                     // 1a: admin gets what it asks for, everyone else gets nothing
    pub fn issue(&self, who: &Principal, granted: Vec<Scope>) -> TokenResponse;
    pub fn verify(&self, bearer: &str) -> Result<Claims, AuthError>;
    pub async fn bootstrap_admin(&self, password: Option<String>) -> Result<Option<String>, AuthError>; // creates "admin" when there are no users; returns the generated password if it made one
}
```

Values: issuer and service are both `wharf` unless configured; token lifetime 300 seconds; the JWT header carries `kid`; generated admin passwords are 24 characters from `[A-Za-z0-9]`.

- [ ] **Step 1: Write the failing tests**

```rust
#[test] fn password_round_trip() { let h = hash_password("s3cret"); assert!(h.starts_with("$argon2id$")); assert!(verify_password("s3cret", &h)); assert!(!verify_password("nope", &h)); }
#[test] fn scope_parses() { let s: Scope = "repository:library/nginx:pull,push".parse().unwrap(); assert_eq!(s.actions, ["pull", "push"]); }
#[test] fn scope_with_port_in_name_is_rejected() { assert!("repository:host:5000/x:pull".parse::<Scope>().is_err()); }
#[tokio::test] async fn admin_gets_requested_scope() {}
#[tokio::test] async fn anonymous_gets_nothing() {}
#[tokio::test] async fn token_round_trip_and_expiry() { /* issue, verify ok; with tokio::time::pause and a clock injected through Auth, after 301 s verify fails */ }
#[tokio::test] async fn token_signed_by_other_key_is_rejected() {}
#[tokio::test] async fn bootstrap_creates_admin_once() { /* first call Some(pw) of len 24, second call None */ }
#[test] fn signing_key_persists() { /* load_or_create twice on the same path gives the same kid */ }
```

For the expiry test, `Auth::new` takes the clock through a `with_clock(Arc<dyn Fn() -> OffsetDateTime + Send + Sync>)` builder method; production uses `OffsetDateTime::now_utc`.

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p auth`
Expected: FAIL to compile.

- [ ] **Step 3: Implement the modules.** The key file is a PKCS#8 PEM written with mode 0600 on Unix. `kid` is the base32 of the first 30 bytes of the SHA-256 of the public key in DER, grouped by four with `:`, the format the Docker token spec uses.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p auth`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/auth
git commit -m "feat: argon2 passwords and ES256 registry tokens"
```

---

### Task 7: HTTP base, errors, token endpoint and auth middleware

**Files:**
- Create: `crates/http/src/{lib.rs,state.rs,error.rs,token.rs,authn.rs,base.rs}`
- Test: `crates/http/tests/{common/mod.rs,base.rs,token.rs}`

**Interfaces:**
- Consumes: `auth::Auth`, `meta::Meta`, `storage::{BlobStore, Uploads}`, `oci::ErrorCode`.
- Produces:

```rust
#[derive(Clone)] pub struct AppState { pub meta: Meta, pub blobs: BlobStore, pub uploads: Arc<Uploads>, pub auth: Auth, pub base_url: url::Url, pub presign_ttl: Option<Duration> }
pub fn router(state: AppState) -> axum::Router;
pub struct RegistryError { pub code: ErrorCode, pub message: String, pub detail: Option<serde_json::Value> }
impl IntoResponse for RegistryError   // {"errors":[{"code":"BLOB_UNKNOWN","message":"...","detail":...}]}
pub struct Access { pub principal: Principal, pub granted: Vec<Scope> }   // extractor: from Basic or Bearer; checks the route's repository and action, else 401 with challenge
// tests/common: pub async fn test_app() -> (axum::Router, TempDir, String /*admin password*/)
```

Behavior the tests pin:
- Every `/v2` response carries `Docker-Distribution-Api-Version: registry/2.0`.
- `GET /v2/` with no credentials: 401 with `WWW-Authenticate: Bearer realm="<base_url>/service/token",service="wharf"`. With valid Basic or Bearer: 200 and body `{}`.
- A repository route without credentials: 401 with the same header plus `,scope="repository:<name>:<actions>"`, where actions is `pull` for GET and HEAD and `pull,push` for the rest, and `delete` for DELETE.
- `/v2` accepts Basic credentials directly as well as Bearer tokens, as Harbor does.
- `GET /service/token?service=wharf&scope=repository:a/b:pull,push` with Basic: 200 with `TokenResponse` JSON. Without credentials: a token with no access, as Distribution does. Bad credentials: 401.
- `POST /service/token` with `grant_type=password` form fields: the same as GET (OAuth2 password grant used by some clients).

- [ ] **Step 1: Write the failing tests** in `base.rs` and `token.rs`, using `tower::ServiceExt::oneshot` on `test_app()`, one test per bullet above.

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p http --test base --test token`
Expected: FAIL to compile.

- [ ] **Step 3: Implement** the router skeleton with `tower-http` `SetResponseHeader` for the API version header, the token handlers, and the `Access` extractor that reads the repository name from the path and the action from the method.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p http`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/http
git commit -m "feat: token service, auth challenges and registry errors"
```

---

### Task 8: Blob, upload, manifest, tag, referrer and catalog endpoints

**Files:**
- Create: `crates/http/src/{blobs.rs,uploads.rs,manifests.rs,tags.rs,referrers.rs,catalog.rs}`
- Test: `crates/http/tests/{blobs.rs,uploads.rs,manifests.rs,tags.rs,referrers.rs}`

**Interfaces:**
- Consumes: everything in `AppState`; `oci::parse_manifest`; `Meta` methods from Task 5; `storage::layout` for link files.
- Produces: the full distribution API that Tasks 11 and 12 test from outside.

Routes and the behavior each test pins (all paths under `/v2/<name>/`, where `<name>` may contain slashes):

| Route | Behavior |
| --- | --- |
| `HEAD/GET blobs/<digest>` | 200 with `Content-Length`, `Docker-Content-Digest`; `Range: bytes=a-b` gives 206 with `Content-Range`, a range past the end gives 416 `RANGE_INVALID`; with `presign_ttl` set and a signer, 307 to the presigned URL; unknown or not linked in this repository: 404 `BLOB_UNKNOWN` |
| `DELETE blobs/<digest>` | 202 and unlinks from the repository; unknown: 404 |
| `POST blobs/uploads/` | 202 with `Location: /v2/<name>/blobs/uploads/<uuid>`, `Range: 0-0`, `Docker-Upload-UUID`; with `?digest=` and a body: monolithic upload, 201 with `Location: /v2/<name>/blobs/<digest>`; with `?mount=<digest>&from=<repo>` and the blob linked in `from` and the caller allowed to pull `from`: 201 mount, otherwise falls back to 202 new session |
| `PATCH blobs/uploads/<uuid>` | `Content-Range: a-b` must start at the current size, else 416; streams the body; 202 with `Range: 0-<size-1>` |
| `PUT blobs/uploads/<uuid>?digest=` | optional final chunk, verify digest (400 `DIGEST_INVALID` on mismatch), `record_blob`, `link_blob`, write the layer link file; 201 with `Location` and `Docker-Content-Digest` |
| `GET blobs/uploads/<uuid>` | 204 with `Range`; unknown: 404 `BLOB_UPLOAD_UNKNOWN` |
| `DELETE blobs/uploads/<uuid>` | 204 |
| `PUT manifests/<reference>` | body up to `MAX_MANIFEST_BYTES`; parse; store bytes as a blob; `put_manifest` with the tag when the reference is a tag; when the reference is a digest it must match the body (400 `DIGEST_INVALID`); `MissingReferences` maps to 400 `MANIFEST_BLOB_UNKNOWN` with the digests in `detail`; write revision and tag link files; 201 with `Location`, `Docker-Content-Digest`, and `OCI-Subject` when the manifest has a subject |
| `HEAD/GET manifests/<reference>` | 200 with `Content-Type` set to the stored media type, `Docker-Content-Digest`, `Content-Length`; unknown: 404 `MANIFEST_UNKNOWN` |
| `DELETE manifests/<reference>` | by digest: 202 and delete link files; by tag: 202 and delete the tag only |
| `GET tags/list?n=&last=` | `{"name": ..., "tags": [...]}`; when more remain, `Link: </v2/<name>/tags/list?n=<n>&last=<last>>; rel="next"`; unknown repository: 404 `NAME_UNKNOWN` |
| `GET referrers/<digest>?artifactType=` | 200 OCI index JSON with `Content-Type: application/vnd.oci.image.index.v1+json`, descriptors with `artifactType` and `annotations`; with a filter, header `OCI-Filters-Applied: artifactType`; unknown subject: 200 with empty `manifests` |
| `GET /v2/_catalog?n=&last=` | admin only; `{"repositories": [...]}` with the same `Link` paging |

- [ ] **Step 1: Write the failing tests**, one per row and outcome in the table, using `test_app()` and a helper `push_image(&app, name, tag) -> (manifest_digest, layer_digests)` in `tests/common`.

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p http`
Expected: the new tests FAIL with 404 from the router.

- [ ] **Step 3: Implement the handlers.** Repository names with slashes: one wildcard route `/v2/{*rest}` dispatched by a small parser that splits on the last `/blobs/`, `/manifests/`, `/tags/` or `/referrers/` segment, because axum cannot match a wildcard in the middle of a path. Streams go straight from the request body to `Uploads::append`; no `to_bytes` on blob bodies.

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p http`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/http
git commit -m "feat: OCI distribution endpoints"
```

---

### Task 9: Garbage collection

**Files:**
- Create: `crates/jobs/src/{lib.rs,gc.rs}`
- Modify: `crates/wharf/src/main.rs` (add `gc` subcommand)
- Test: `crates/jobs/tests/gc.rs`, `crates/wharf/tests/cli.rs`

**Interfaces:**
- Consumes: `Meta::collectable_blobs`, `Meta::forget_blob`, `BlobStore::delete`, `Uploads::sweep_stale`.
- Produces:

```rust
pub struct GcOptions { pub grace: Duration, pub dry_run: bool, pub stale_uploads: Duration }   // stale_uploads default 24 h
pub struct GcReport { pub blobs: Vec<(Digest, u64)>, pub bytes: u64, pub stale_uploads: usize }
pub async fn run_gc(meta: &Meta, blobs: &BlobStore, uploads: &Uploads, opts: &GcOptions, now: OffsetDateTime) -> Result<GcReport, JobError>;
pub fn spawn_periodic_gc(meta: Meta, blobs: BlobStore, uploads: Arc<Uploads>, opts: GcOptions, interval: Duration, shutdown: CancellationToken) -> JoinHandle<()>;
```

Order inside `run_gc`: list collectable blobs with `zero_before = now - grace`, then for each call `forget_blob` first and delete the stored bytes only when it returns true. So a blob that gained a reference between listing and deletion stays. `dry_run` lists and deletes nothing. `wharf gc [--dry-run] [--config <path>]` prints one line per blob (`<digest>\t<size>`) and a summary line `collected <n> blobs, <bytes> bytes, <m> stale uploads`.

- [ ] **Step 1: Write the failing tests**

```rust
#[tokio::test] async fn orphan_blob_collected_after_grace() { /* record_blob at t0, run at t0 + 1h: nothing; at t0 + 2h + 1s: collected and stat None */ }
#[tokio::test] async fn referenced_blob_survives() {}
#[tokio::test] async fn dry_run_deletes_nothing() {}
#[tokio::test] async fn blob_referenced_after_listing_survives() { /* call collectable_blobs, then put_manifest referencing it, then run the forget-then-delete step: blob remains in meta and storage */ }
#[test] fn gc_cli_dry_run_prints_summary() { /* assert_cmd on a temp data dir, stdout ends with "collected 0 blobs, 0 bytes, 0 stale uploads\n" */ }
```

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p jobs && cargo test -p wharf --test cli`
Expected: FAIL to compile.

- [ ] **Step 3: Implement `run_gc`, `spawn_periodic_gc` and the subcommand.**

- [ ] **Step 4: Run and see them pass**

Run: `cargo test -p jobs && cargo test -p wharf`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add crates/jobs crates/wharf
git commit -m "feat: reference-counted garbage collection"
```

---

### Task 10: `wharf serve`, admin command, TLS, metrics and tracing

**Files:**
- Create: `crates/wharf/src/{serve.rs,admin.rs,telemetry.rs,tls.rs}`
- Create: `crates/http/src/{health.rs,metrics.rs}`
- Test: `crates/wharf/tests/serve.rs`, `crates/http/tests/metrics.rs`

**Interfaces:**
- Consumes: `config::load`, every crate above.
- Produces:
  - `wharf serve [--config <path>]`: loads config, opens `Meta`, builds the store (`LocalFileSystem` at `storage.root` or `<data_dir>/storage`, or `AmazonS3Builder::from_env` plus the config fields), spool at `<data_dir>/uploads`, key at `<data_dir>/token-key.pem`, bootstraps the admin (password from `WHARF_ADMIN_PASSWORD` when set), prints a generated password once to stderr as `admin password: <pw>`, starts periodic GC, serves until SIGINT or SIGTERM (Ctrl+C on Windows), then drains in-flight requests for up to 30 seconds.
  - `wharf admin reset-password <user>`: prints a new 24-character password.
  - `GET /healthz`: 200 `ok` while the process runs. `GET /readyz`: 200 when the database answers `SELECT 1` and the store answers a `head` on a missing key without an error other than not found, else 503.
  - `GET /metrics`: OpenMetrics text with `registry_http_requests_total{code,handler,method}` and `registry_http_request_duration_seconds{handler,method}`, the names CNCF Distribution and Harbor's registry use. `handler` is one of `base`, `blob`, `blob_upload`, `blob_upload_chunk`, `manifest`, `tags`, `referrers`, `catalog`, `token`.
  - TLS: `[tls] cert/key` serves HTTPS with `axum-server` and rustls; `[tls] acme` uses `rustls-acme` with TLS-ALPN-01 and caches certificates under `<data_dir>/acme`.
  - Tracing: JSON logs to stdout; with `telemetry.otlp_endpoint`, spans go to OTLP over gRPC.

- [ ] **Step 1: Write the failing tests**

```rust
#[tokio::test] async fn metrics_count_requests_by_handler() { /* GET /v2/ twice via test_app, /metrics contains registry_http_requests_total{code="401",handler="base",method="GET"} 2 */ }
#[test] fn serve_starts_and_answers_healthz() { /* spawn the binary with WHARF_LISTEN=127.0.0.1:0 is not discoverable, so use WHARF_LISTEN=127.0.0.1:<free port from a bound-then-dropped TcpListener>, WHARF_DATA_DIR=<tempdir>, WHARF_ADMIN_PASSWORD=pw; poll /healthz with a 10 s deadline; kill */ }
#[test] fn first_start_prints_generated_password_once() { /* no WHARF_ADMIN_PASSWORD: stderr has one "admin password: " line of 24 chars; second start in the same data dir: none */ }
#[test] fn reset_password_prints_new_password() {}
#[test] fn serve_with_tls_files_answers_https() { /* rcgen self-signed cert in tempdir; reqwest with the cert added as root */ }
```

- [ ] **Step 2: Run and see them fail**

Run: `cargo test -p wharf --test serve && cargo test -p http --test metrics`
Expected: FAIL.

- [ ] **Step 3: Implement** `serve`, `admin`, `tls`, `telemetry`, `health` and `metrics` (a tower layer that records both metrics, labelled by the handler name the router sets in a request extension).

- [ ] **Step 4: Run and see them pass**

Run: `cargo test --workspace`
Expected: PASS on all three systems.

- [ ] **Step 5: Commit**

```bash
git add crates/wharf crates/http
git commit -m "feat: serve command with TLS, ACME, metrics and tracing"
```

---

### Task 11: OCI conformance in CI

**Files:**
- Create: `conformance/VERSION`, `conformance/run.sh`
- Modify: `.github/workflows/ci.yml` (new job `conformance`)

**Interfaces:**
- Consumes: the `wharf` binary.
- Produces: a required check `conformance` that runs push, pull, content discovery and content management workflows.

- [ ] **Step 1: Pin the suite.** Write the commit of `opencontainers/distribution-spec` that the job builds to `conformance/VERSION`, and check the suite's README at that commit for its environment variables and how it is run (the suite was redesigned in April 2026).

- [ ] **Step 2: Write `conformance/run.sh`** (bash, Linux only): build wharf in release mode, start it on `127.0.0.1:5000` with a temp data dir and `WHARF_ADMIN_PASSWORD`, wait for `/healthz`, build and run the pinned suite with all four workflows enabled, a namespace and a cross-mount namespace, and the admin credentials, then upload the suite's HTML and JUnit reports as artifacts.

- [ ] **Step 3: Run it locally on Linux or in a draft PR and see it fail where wharf is incomplete.** Fix wharf in the crate the failure points to, with a test in that crate first that reproduces the failure, until the suite passes.

Run: `bash conformance/run.sh`
Expected: the suite reports zero failures.

- [ ] **Step 4: Add the `conformance` job** to `ci.yml` on `ubuntu-latest`, with `actions/setup-go` pinned by SHA.

- [ ] **Step 5: Commit**

```bash
git add conformance .github/workflows/ci.yml
git commit -m "ci: run the OCI distribution conformance suite"
```

After the PR merges, add `conformance` to the required checks.

---

### Task 12: Client matrix end to end

**Files:**
- Create: `e2e/run.sh`, `e2e/lib.sh`, `e2e/clients/{docker,podman,nerdctl,helm,oras,cosign,notation,crane,skopeo,regctl}.sh`, `e2e/VERSIONS`
- Modify: `.github/workflows/ci.yml` (new job `e2e`)

**Interfaces:**
- Consumes: the `wharf` binary served over HTTPS with a self-signed CA (clients refuse plain HTTP to non-localhost by default, and some refuse it on localhost too).
- Produces: a required check `e2e`.

What each client script proves, against `wharf.test:5443` mapped to 127.0.0.1 in `/etc/hosts` with the CA trusted system-wide:

| Client | Proves |
| --- | --- |
| docker | login, push, pull of a multi-arch image built with buildx, delete by digest through the API |
| podman | login, push, pull |
| nerdctl | login, push, pull through containerd |
| helm | `helm registry login`, `helm push` a chart, `helm pull`, `helm show chart oci://...` |
| oras | push an artifact with `--artifact-type`, attach a referrer, `oras discover` lists it |
| cosign | `cosign sign --key`, `cosign verify --key`, `cosign attest`, `cosign tree` |
| notation | `notation sign` with a test key, `notation verify` with a trust policy |
| crane | `crane copy` between two repositories (cross-repository mount), `crane ls`, `crane catalog` |
| skopeo | `skopeo copy` from an OCI layout, `skopeo inspect` |
| regctl | `regctl tag ls` with paging, `regctl manifest get --list`, `regctl blob get` with a range |

- [ ] **Step 1: Write `e2e/lib.sh`** with `start_wharf`, `stop_wharf`, `assert_eq`, and the CA setup, and `e2e/VERSIONS` with a pinned version of every client.

- [ ] **Step 2: Write the client scripts** from the table, each failing the run on the first wrong result.

- [ ] **Step 3: Run them and fix what fails**, each fix with a unit or HTTP test in the crate it touches first.

Run: `bash e2e/run.sh`
Expected: `10 clients passed`.

- [ ] **Step 3b: Run against S3.** `e2e/run.sh --s3` starts MinIO (pinned in `e2e/VERSIONS`), serves wharf with `[storage] s3` pointed at it and `presign_ttl` on, and runs the docker, crane and regctl scripts again, plus a check that a blob GET answers 307 to a MinIO URL.

Run: `bash e2e/run.sh --s3`
Expected: `3 clients passed`, `presigned redirect ok`.

- [ ] **Step 4: Add the `e2e` job** to `ci.yml` on `ubuntu-latest`.

- [ ] **Step 5: Commit**

```bash
git add e2e .github/workflows/ci.yml
git commit -m "ci: end-to-end tests with ten registry clients"
```

After the PR merges, add `e2e` to the required checks, tag `v0.1.0-alpha.1`, and publish a release with notes from `CHANGELOG.md`.
