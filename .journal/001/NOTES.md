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
