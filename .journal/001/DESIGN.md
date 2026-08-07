# incus-bootc design

Status: draft for review. Date: 2026-08-06. Author: session 001.

Everything in this document rests on the spike recorded in
[FINDINGS.md](spike/FINDINGS.md). Facts marked "verify" need a check during
implementation; everything else was proven by running it.

## 1. What incus-bootc does

incus-bootc turns a bootc Containerfile into a virtual machine image that Incus
can import, and keeps Incus hosts supplied with those images. It is one Go
binary with two verbs:

- `incus-bootc publish` — build the Containerfile, install it into a disk,
  and push both the container image and the disk artifact to an OCI registry.
- `incus-bootc sync` — compare the registry against one or more Incus targets
  and import whatever is missing.

The lifecycle it serves: an instance boots once from the published QCOW2, then
tracks its container image with `bootc upgrade`. The disk is a bootstrap
artifact, not the update channel. Day-2 updates never pass through incus-bootc.

## 2. Scope

In scope for v1:

- Build and lint bootc images with podman.
- Produce a QCOW2 disk with `bootc install to-disk --via-loopback`.
- Publish the disk plus Incus metadata as one OCI artifact.
- Sync artifacts into Incus servers over the Incus HTTPS API, idempotently,
  with alias management and retention.
- Run sync as a long-lived agent (same binary, `--interval`).
- Native architecture only. Multi-arch publishing means running publish once
  per architecture; an OCI index tying them together can come later.

Out of scope for v1, with reasons:

- **Simplestreams serving.** The sync agent covers pull-style distribution for
  every consumer we administer, with full registry auth. A simplestreams
  façade only helps anonymous third parties, and it can be added later over
  the same artifacts without changing them.
- **osbuild/image-builder as a disk backend.** Its GPT layout cannot pass
  Incus's `sgdisk --move-second-header` check (upstream bug bib#1195,
  reproduced and root-caused in the spike).
- **User/key injection at disk-build time.** Per-instance data arrives via
  cloud-init at provision time. Baked users belong in the Containerfile.
  One image ref then produces one disk, with no side-channel inputs.
- **Signature verification of consumed images.** Deferred; see open questions
  for the signing posture of what we produce.

## 3. Facts this design stands on

From the spike (2026-08-06, aarch64, Incus 7.3, Fedora bootc 44):

- `bootc install to-disk --via-loopback` inside a privileged container
  produces a disk whose GPT passes Incus's import checks. The resulting VM
  boots, and `bootc status`, `switch`, `upgrade`, and rollback all work
  inside an Incus VM. An upgrade that changed one file pulled one 200-byte
  layer.
- Incus VM images must be qcow2. Both `incus image import` and
  `incus-simplestreams` reject raw disks ("Unsupported compression").
- The metadata tarball needs `architecture` twice: top-level for import,
  under `properties` for simplestreams tooling. Incus spells architectures
  `aarch64`/`x86_64`; OCI spells them `arm64`/`amd64`.
- Incus enforces UEFI Secure Boot by default and the Fedora bootc chain fails
  it on aarch64. Instances need `security.secureboot=false` today.
- The image fingerprint is sha256 over metadata tarball bytes plus qcow2
  bytes. Incus dedupes and caches by it.
- Conversion raw→qcow2 took about 1 second for a 2 GB image; cost scales with
  allocated bytes, not virtual size.
- Fedora bootc images ship no default root filesystem
  (`/usr/lib/bootc/install/` is empty), so the installer needs an explicit
  filesystem unless the image provides one.
- The install path needs rootful podman, loop devices, and no hypervisor. The
  sync path needs network access and an API token, nothing privileged.

## 4. Pipeline

### publish

```
Containerfile
   │ 1. podman build
   ▼
bootc image (local storage)
   │ 2. bootc container lint  (inside the image; refuse on failure)
   │ 3. podman push ──────────────────────────▶ registry: image ref
   │ 4. truncate raw file (disk size flag, default 10 GiB)
   │ 5. podman run --privileged <image> \
   │      bootc install to-disk --via-loopback \
   │      --target-imgref <registry image ref>          (verify flag; see below)
   ▼
raw disk
   │ 6. qemu-img convert -O qcow2   (containerized; no host dependency)
   ▼
qcow2
   │ 7. generate metadata.tar.gz (pure code, no I/O)
   │ 8. push both as one OCI artifact, subject-linked to the image manifest
   ▼
registry: disk artifact
```

One ordering rule: push the image (3) before the install (5), so the
installed system tracks the registry ref from first boot via
`--target-imgref`. Without it the disk tracks a local ref, and every
instance needs a manual `bootc switch`; the spike had to do exactly that by
hand. Verify `--target-imgref` behavior in milestone 2. The fallback is a
baked systemd unit that runs `bootc switch` once on first boot, which the
spike proved works.

Filesystem selection: use the image's own install config when present,
else default to ext4 (spike-proven), always overridable by flag.

Disk size: default 10 GiB. The exact number matters little because Incus
grows the root device at instance creation and `bootc-generic-growpart`
(observed active in the spike) grows the filesystem into it.

### sync

```
for each target Incus server/cluster:
  1. resolve configured refs → image digest → referrers → disk artifact
  2. read fingerprint from artifact annotations
  3. ask Incus: image with this fingerprint present?      (one API call)
  4. if missing: stream metadata blob + qcow2 blob into the images API
  5. point the alias at the new image
  6. delete synced images beyond the retention count (default: keep 2)
```

Sync is a reconcile loop: running it twice is a no-op, running it against a
half-synced target finishes the job. Rollback is pointing the alias back at
the previous fingerprint, which retention keeps around.

Agent mode is `sync --interval 15m` (or a webhook trigger later). Same
binary, same config, no privileges. It runs anywhere that can reach the
registry and the Incus API, including as an ordinary instance on the target
cluster itself.

## 5. The artifact contract

The disk artifact is the stable interface between publish and sync. Future
consumers (a simplestreams façade, other tools) read the same artifact, so
this contract changes only by version bump.

- One OCI artifact manifest, `artifactType`:
  `application/vnd.componere.incus-bootc.image.v1`
- Blob 1: the metadata tarball,
  media type `application/vnd.componere.incus-bootc.metadata.v1.tar+gzip`
- Blob 2: the disk,
  media type `application/vnd.componere.incus-bootc.disk.v1.qcow2`
- `subject`: the manifest digest of the bootc image it was installed from.
  Discovery goes image ref → digest → referrers API, filtered by
  artifactType. oras-go falls back to its tag scheme on registries without
  referrers support, and sync also accepts an explicit artifact ref.

Annotations on the manifest, all prefixed `com.componere.incus-bootc.`
(shortened below):

| Annotation | Content |
|---|---|
| `…incus-bootc.fingerprint` | Incus fingerprint (sha256 of blob1+blob2) |
| `…incus-bootc.architecture` | Incus spelling, e.g. `aarch64` |
| `…incus-bootc.alias` | default alias for sync, e.g. `repo/tag` |
| `…incus-bootc.source` | image ref + digest the disk was installed from |
| `org.opencontainers.image.created` | build timestamp |

Sync trusts annotations for skip decisions but verifies the fingerprint while
streaming (hash as bytes flow; abort on mismatch).

## 6. Incus metadata rules

The metadata generator is pure code. The first two rules and the secureboot
rule come straight from spike failures; the rest are design choices:

- Emit `architecture` top-level and under `properties` (both consumers).
- Map OCI arch names to Incus names (`arm64`→`aarch64`, `amd64`→`x86_64`).
- `creation_date` is unix seconds from the image build timestamp.
- `properties.os/release/variant/description` come from standard
  `org.opencontainers.image.*` labels when present, flags otherwise, with
  fallbacks so unlabeled images still publish.
- Emit `requirements.secureboot: "false"` as documentation, but do not rely
  on it: spike 2 tested it and Incus still enforced Secure Boot (shim failed
  with 0x1A). Instances need explicit `security.secureboot=false`. v1 states
  this in docs; sync growing an optional launch profile on targets is a
  deferred item.
- Deterministic bytes: fixed tar entry order, epoch-pinned timestamps,
  root/root ownership, fixed gzip header. Same inputs, same fingerprint,
  idempotent republish.
- No `templates/`. Instance identity is cloud-init's job.

## 7. Architecture

Hexagonal (A1): the publish and sync flows are pure orchestration over ports;
every side effect lives in an adapter. The core compiles and tests without
podman, a registry, or Incus.

### Ports

Interfaces are declared in the core package that consumes them (I2), each
scoped to one dependency's purpose (A2):

| Port | Methods (sketch) | v1 adapter |
|---|---|---|
| `Runtime` | `Build`, `Push`, `Run` (privileged helper) | podman CLI exec |
| `Registry` | `PushArtifact`, `Resolve`, `Referrers`, `FetchBlob` | oras-go |
| `Incus` | `HasImage`, `ImportImage`, `SetAlias`, `ListImages`, `DeleteImage` | incus Go client |
| `Converter` | `RawToQCOW2` | qemu-img via `Runtime` |

The podman adapter shells out to the CLI with `--format json`. The socket
API (bindings or Docker-compat client) is a second adapter behind the same
port if a concrete need appears; nothing else changes. Library choices are
mature and first-party where one exists (oras-go, the incus client,
cobra/viper), per L1.

`Converter` is a port so the pipeline can be tested without a 1-second
container run, and so a pure-Go qcow2 writer can replace qemu-img later
without touching the core. That writer would also let publish stream the
conversion during upload; not a v1 concern.

### Packages

```
cmd/incus-bootc/      CLI wiring only (cobra + viper)
internal/publish/     publish orchestration; declares Runtime, Converter,
                      and the push side of Registry
internal/sync/        sync reconciliation; declares Incus and the fetch
                      side of Registry
internal/artifact/    artifact schema, annotations, fingerprint (pure)
internal/meta/        metadata tarball generation (pure)
internal/podman/      Runtime adapter + mocks/
internal/registry/    Registry adapter (oras-go) + mocks/
internal/incus/       Incus adapter (incus client) + mocks/
internal/qemu/        Converter adapter + mocks/
```

Names follow A4: short, one obvious purpose each (A3). Every package gets
`doc.go` (D4). Files stay under 1,000 lines (R2).

### Domain types

Domain terms get types (I1). Enough to show the shape:

```go
// Fingerprint is the Incus image fingerprint: sha256 over the metadata
// tarball bytes followed by the disk bytes.
type Fingerprint string

// ImageRef is a fully qualified OCI image reference.
type ImageRef string

// Arch is an architecture in Incus spelling ("aarch64", "x86_64").
// FromOCI converts "arm64"/"amd64" forms.
type Arch string

// Artifact describes one published disk artifact.
type Artifact struct {
    Source      ImageRef    // bootc image the disk was installed from
    Fingerprint Fingerprint
    Arch        Arch
    Alias       string
    Created     time.Time
}
```

### Streaming (P1/P2)

Publish and sync sit on the critical path and move multi-GB blobs. No step
buffers a disk in memory: install writes a file, conversion reads and writes
files, publish streams the file into the registry client, sync streams
registry → hash → Incus API through an `io.TeeReader`. Peak memory stays at
buffer size regardless of image size.

### Errors and retries

Sentinels (E1) where callers branch: `ErrNotBootc` (lint failed or kernel
missing), `ErrArtifactNotFound`, `ErrFingerprintMismatch`,
`ErrTargetUnreachable`. Everything else wraps upward with context.

Retries (E3): registry and Incus API calls retry transient failures
(connection reset, 5xx, timeout) with capped exponential backoff, default 4
attempts. Local operations (build, install, convert) do not retry; their
failures are deterministic.

## 8. CLI

Cobra + viper. Precedence: flags > environment (`INCUS_BOOTC_*`) > config
file. One config file serves both verbs; sync targets live in it because
tokens and lists of clusters do not belong on a command line.

```
incus-bootc publish <context-dir>
    --file Containerfile.bootc   --tag registry.example.com/app:1.2
    --disk-size 10GiB            --filesystem ext4
    --alias app                  [--skip-image-push]

incus-bootc sync
    --config sync.yaml           [--target name]
    [--interval 15m]             [--dry-run]

incus-bootc inspect <artifact-or-image-ref>     # print artifact contract fields
incus-bootc version
```

Config sketch for sync:

```yaml
refs:
  - registry.example.com/app:1.2
targets:
  - name: homelab
    url: https://incus.lab:8443
    token_file: /run/secrets/incus-token
retention: 2
```

Registry auth uses the standard docker/containers credential chain; no
custom auth store.

## 9. Testing

Three layers (T1), mocks generated by mockery into each adapter's `mocks/`
(T2, T3):

- **Unit** (pure core): metadata bytes are golden-tested for determinism;
  fingerprint math; arch mapping; annotation round-trip; alias derivation.
- **Integration** (core + mocks): publish call order (push before install);
  sync idempotency (second run makes zero mutating calls); retention edge
  cases; retry/backoff behavior; fingerprint mismatch aborts import.
- **End-to-end** (live): GitHub Actions Linux runners provide rootful podman,
  loop devices, and `/dev/kvm`. The e2e job installs Incus (zabbly), runs
  `publish` against a registry container (testcontainers), runs `sync` into
  the local Incus, and asserts: fingerprint present, alias set, and
  `incus init` succeeds (exercises the sgdisk unpack path without booting).
  A slower boot test (`incus start` + wait for DHCP lease, as in the spike)
  runs on a schedule rather than per-PR. Local e2e uses the Lima setup from
  the spike.

The e2e suite is the real conformance test against Incus. The spike showed
implementations are stricter than their docs, so behavior gets pinned by
test, not by reading references.

## 10. Deferred items

Recorded so they are decisions, not omissions:

- **Simplestreams façade** — waits for a consumer outside our administrative
  reach. The artifact contract already carries everything it would serve.
- **Bootstrap-switch pattern** (small universal disk, `bootc switch` at
  first boot to the app ref) — proven mechanism, but v1 publishes per-image
  disks. Revisit when artifact count or size becomes a cost. Two experiments
  from the ideation log gate it: `/var` seeding semantics across switch, and
  cross-distro-family switch.
- **Pure-Go qcow2 writer** — replaces the qemu-img container behind
  `Converter` when self-containment or streamed conversion earns it.
- **OCI index for multi-arch** — publish per-arch works today.
- **Socket-API podman adapter** — CLI adapter first; swap is port-local.
- **qcow2 compression** (`-c`) — untested against Incus import; test, then
  decide by numbers.

## 11. Open questions

1. **Signing posture.** Should publish sign what it pushes (cosign on image
   and artifact), and should sync verify before import? The repo's tooling
   (goreleaser/publishing skills, SLSA targets) suggests signing is house
   style, but it adds key management to every user's setup. Options: (a) no
   signing in v1, (b) sign-if-key-present with verify-if-configured on sync,
   (c) mandatory. My lean is (b): optional at both ends, no new mandatory
   infrastructure, and the artifact contract gains nothing breaking later.
   This decides user-facing setup, so it needs your call.

## 12. Build order

1. `artifact` + `meta` with unit tests. Proves the contract and determinism
   before any I/O exists.
2. `podman` adapter + `publish` through local output (skip registry):
   Containerfile in, qcow2 + metadata out. Verify `--target-imgref` here.
3. `registry` adapter: publish end-to-end to a registry container.
4. `incus` adapter + `sync` against the Lima Incus from the spike.
5. e2e workflow in CI, agent mode, `inspect`, then user docs under `docs/`
   (Diátaxis, D5) once behavior is real (D6).

Each milestone ends with the three test layers green for what exists so far.
