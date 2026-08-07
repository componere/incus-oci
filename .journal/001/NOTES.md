---
id: 001
title: Bootstrap incus-oci project
started: 2026-08-06
---

## 2026-08-06 16:10 — Kickoff
Goal for the session: not yet stated in detail; the user opened a session immediately after bootstrapping the repo, so the working assumption is initial development of `incus-oci`.
Current state of the world: repo `componere/incus-oci` freshly created from the `meigma/template-go` template (default branch `master`, single "Initial commit"). Session framework set up: journal branch `journal/jmgilman` created and pushed, `.journal/` scaffolded. Template ships mise/moon tooling, melange/apko image pipeline, and repo skills under `.agents/skills`. No project code written yet; `DELETE_ME.md` template placeholder still present.
Plan: await the user's actual request for this session.

## 2026-08-06 16:14 — Goal stated: build familiarity
User defined the project space: `incus-oci` aims at unified management of container and disk images, with Incus (specifically IncusOS) as the primary virtualization layer and bootc-generated images as the focus. Envisioned flow: dual-variant Containerfile (normal + bootc) → podman build/publish to registry → image-builder produces QCOW2 wrapped with Incus metadata → QCOW2 is a one-time boot artifact; subsequent upgrades happen in-place via standard bootc process. Instruction: build familiarity only — no design, no strong opinions yet. Launched four parallel research agents: (1) bootc + bootc-image-builder, (2) Incus image format/VM model, (3) IncusOS, (4) ecosystem prior art (OCI artifacts, containerDisk, simplestreams, etc.).

## 2026-08-06 16:35 — Research sweep complete (4 agents)
Key facts gathered (details in agent reports, distilled here):
- bootc: OCI image + kernel at /usr/lib/modules; ostree/composefs backend; CNCF sandbox, weekly releases (v1.16.7). /etc 3-way merge, /var machine-local, /usr read-only. upgrade/switch/rollback track a registry ref. Logically bound images via Quadlet for app payloads.
- bootc-image-builder ARCHIVED June 2026 → merged into unified osbuild/image-builder CLI (`image-builder build qcow2 --bootc-ref=...`); container wrapper remains compatible for now. podman-bootc also archived → replaced by bcvk (ephemeral VMs, to-disk, libvirt).
- Incus images: split format = metadata tarball (metadata.yaml: architecture + creation_date + optional properties/templates) + disk.qcow2 for VMs. `incus image import meta.tar.gz disk.qcow2 --alias x`. Fingerprint = sha256(meta+data). VMs: UEFI default, security.secureboot default TRUE (MS keys), csm option. incus-agent via 9p/virtiofs config drive or `agent:config` ISO; needed for exec/file ops; console works without. cloud-init only in /cloud variants; agent-less path = `cloud-init:config` NoCloud seed ISO.
- simplestreams: static HTTPS tree; incus-simplestreams CLI (add/list/remove/generate-metadata); qcow2 published as disk-kvm.img.
- Incus OCI app containers since 6.3 (skopeo+umoci), containers only; open issue #1360 for OCI-as-VM.
- IncusOS: GA Nov 2025, Debian 13 + mkosi, A/B sysupdate, Secure Boot + TPM required, ZFS, no shell — API only via incus-osd; apps as signed sysext (Incus, Operations Center, Migration Manager). FuturFusion stack rebrands it HypervisorOS. Incus 7.0 LTS May 2026.
- Prior art bootc+Incus: essentially greenfield. One forum thread (Feb-Mar 2026): bib qcow2 fails Incus import — sgdisk --move-second-header exit 4, GPT layout issue, upstream bug bootc-image-builder#1195. No dedicated projects exist.
- Registry-side disk shipping: OCI 1.1 artifacts (artifactType/Referrers), ORAS, podman artifact stable since 5.6, podman machine distributes its own qcow2 via OCI artifacts, KubeVirt containerDisk (/disk/ in scratch image) as closest analog.
- First-boot config landscape: cloud-init (Incus-native keys), Ignition, systemd credentials via SMBIOS (Incus only through raw.qemu overrides).
Next: awaiting user direction; design phase comes later.

## 2026-08-06 16:58 — New goal: spike Containerfile → Incus bootc QCOW2
Goal directive: prove one or more viable shapes for Containerfile → bootc QCOW2 → Incus import, empirically, no hacks; compare all viable methods. Full autonomy for podman/OrbStack/Lima locally; ask before installing host tools. rk1 remote is dead — local only.
Environment: macOS 26.4, M4 Max (nested virt capable), podman 5.7.1, lima 2.0.3 (vz+nested), incus client 7.2, qemu-img 10.2, orbstack, docker, oras. No skopeo on host.
Planned methods: A) bootc-image-builder container B) unified image-builder --bootc-ref C) bcvk to-disk D) bootc install to-disk --via-loopback. Import via incus image import (split) and incus-migrate. Watch for GPT sgdisk bug (bib#1195).

## 2026-08-06 17:15 — Spike progress: Method A built; import bug reproduced + root-caused
- Task1: minimal Containerfile FROM quay.io/fedora/fedora-bootc:44 (aarch64), root:spike pw; `bootc container lint` 13 checks pass; kernel 7.1.6. Built rootful in podman machine (FCOS 43).
- Method A (bib container quay.io/centos-bootc/bootc-image-builder:latest): FAILS without `--rootfs` on Fedora images ("missing required info: DefaultRootFs" — Fedora sets no default rootfs). With --rootfs ext4: SUCCESS → 1.1GB qcow2 (10GiB virtual) + vmdk/vpc/gce/ovf checksums in manifest.
- Incus host: Lima 2.0.3 vz VM w/ nestedVirtualization=true on M4 Max → /dev/kvm present; zabbly incus 7.3 (server) installed. GOTCHA: `incus admin init --auto` failed "no unused IPv4 subnet" (Lima net density); manual `incus network create incusbr0 ipv4.address=10.171.99.1/24` + profile root disk fixed. GOTCHA: images:fedora/43 VM fails default secureboot (unsigned distrobuilder bootloader, "Access Denied"); security.secureboot=false → boots, agent up, IP on bridge. Nested-virt boot is slow (~60-90s to agent).
- Import: `incus image import metadata.tar.gz spike-bib.qcow2` SUCCEEDS (fingerprint 4b69437e). LAUNCH fails: `sgdisk --move-second-header root.img` exit 4 — bib#1195 reproduced today on latest everything.
- ROOT CAUSE: sgdisk: "Secondary partition table overlaps the last partition by 2014 blocks... Aborting". osbuild GPT ends last partition flush to disk end, no room for backup table where sgdisk wants it; `sgdisk -v` alone reports "No problems, 0 free sectors". Builder-vs-importer strictness collision; disk itself boots in plain qemu.
- Next: incus-migrate path w/ same qcow2; Method D (bootc install to-disk, different partitioner) predicted to differ; B likely same osbuild GPT.
