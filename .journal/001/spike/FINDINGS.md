# Spike: Containerfile → bootc QCOW2 → Incus (2026-08-06)

Environment: macOS 26.4 / M4 Max, everything aarch64. Build side: podman machine
(Fedora CoreOS 43, rootful). Incus side: Lima vz VM (Ubuntu 24.04,
`nestedVirtualization: true` → /dev/kvm works) running Zabbly Incus 7.3.
Test image: `FROM quay.io/fedora/fedora-bootc:44` + root password + SSH key
drop-in; passes `bootc container lint` (13 checks).

## Verdict matrix

| # | Builder | Build result | `sgdisk --move-second-header` | `incus image import` + launch | `incus-migrate` | Boot |
|---|---------|--------------|-------------------------------|-------------------------------|-----------------|------|
| A | bootc-image-builder container (archived wrapper) | qcow2 (1.1 GB) — needs `--rootfs ext4` on Fedora | **exit 4** (backup GPT overlaps last partition) | **FAILS** at instance creation (bib#1195, reproduced today) | works (no sgdisk step) | boots, secureboot off |
| B | unified `image-builder` CLI v77 (`--bootc-ref`) | qcow2 — needs real volume at `/var/cache/image-builder/store` (chcon fails on overlay) | **exit 4** — byte-for-byte same osbuild GPT as A | **FAILS** (same defect) | works — booted, SSH-verified `bootc status` | boots, secureboot off |
| C | bcvk 0.18.0 `to-disk` | see notes — blocked on aarch64/F44 today | n/a | n/a | n/a | n/a |
| D | `bootc install to-disk --via-loopback` + `qemu-img convert` | raw→qcow2 (2.1 GB) | **exit 0** — clean 128-entry GPT, 2015-sector tail gap | **WORKS** | (not needed) | boots; full in-guest verification |

## The osbuild GPT defect (A/B)

osbuild ends the last partition at the last usable sector with a backup-table
geometry sgdisk reads as a 2014-sector overlap ("reduce table by 8056
entries"). Incus runs `sgdisk --move-second-header` on every VM image unpack →
instance creation fails. `sgdisk --resize-table=128` does NOT remediate; a real
fix needs a partition shrink or full GPT rewrite upstream. The disk itself
boots fine (firmware ignores the backup table) — this is builder-vs-importer
strictness, tracked as bootc-image-builder#1195, alive on latest everything as
of today. `incus-migrate` succeeds with the same artifact because it converts
qcow2→raw itself and never invokes sgdisk.

## bcvk (C) findings

- `to-disk` is not a container wrapper: it boots an **ephemeral helper VM**
  (QEMU direct-kernel-boot of the target image's kernel) and runs
  `bootc install to-disk` inside it. Requires /dev/kvm.
- Undocumented host deps found empirically: `/dev/vhost-vsock`, `objcopy`
  (binutils). Failures before those were fixed were silent (detached `--rm`
  container, stderr swallowed; diagnosed by re-running its generated podman
  command attached).
- Terminal blocker in this environment: Fedora 44 aarch64 kernels are
  zstd-compressed EFI zboot images; host QEMU 10.1 (Fedora 43) cannot
  direct-load them ("unable to handle EFI zboot image with zstd compression")
  and bcvk hangs forever instead of erroring.
- F43-base retry: identical failure ("unable to handle EFI zboot image with
  zstd compression" / "could not load kernel") — both current Fedora kernel
  streams are zstd zboot. **Method C is conclusively blocked on aarch64 today**
  with host QEMU ≤ 10.1; bcvk hangs silently instead of surfacing the QEMU
  error (upstream-bug-worthy). Untested: x86_64, or a host with newer QEMU.
- Requires `bwrap` inside the target image (fedora-bootc ships it).

## Incus-side facts (aarch64, Incus 7.3)

- Split image import is trivial: `metadata.yaml` (architecture +
  creation_date + properties) tarred up + qcow2; `incus image import meta.tar.gz
  disk.qcow2 --alias x`; fingerprint = sha256 over both.
- `security.secureboot` defaults to true and rejects BOTH distrobuilder stock
  images and the Fedora bootc chain (shim loads, next stage fails 0x1A) on
  aarch64. `security.secureboot=false` currently mandatory for these guests.
- No incus-agent in bootc images → no `incus exec`/IP-in-list via agent; DHCP
  lease still shows in `incus list`, serial console shows login, SSH works.
  Agent injection was intentionally NOT explored (would be a distribution
  question for the real design).
- `incus admin init --auto` can fail subnet auto-pick in dense environments;
  manual `incus network create incusbr0 ipv4.address=...` fixes.
- Ubuntu-archive `incus-tools`/`incus-migrate` packages conflict-remove the
  Zabbly daemon; Zabbly's equivalent is `incus-extra`.
- `incus-migrate` is TUI-only but fully scriptable via stdin (prompt order:
  local-target y/n → type menu → name → path → UEFI y/n → secureboot y/n →
  action menu).

## Upgrade lifecycle (proven end-to-end on Method D instance)

Local `registry:3` on the Incus host bridge (10.171.99.1:5000); insecure
registries.conf drop-in written in-guest (persists via bootc /etc merge);
`bootc switch 10.171.99.1:5000/spike-bootc:spike` → reboot → tracking registry.
Pushed v2 (marker change) to same tag; in-guest `bootc upgrade` pulled
**1 layer / 200 bytes**; reboot → v2 live, previous deployment retained as
rollback. The "one-time QCOW2 boot + standard bootc upgrades + rollback"
model is fully validated inside Incus VMs.

## Operational notes

- Nested-virt guests boot slowly (2–5 min to sshd) and flake under load (one
  systemd pid1 freeze, one journald timeout with 3 concurrent nested VMs) —
  environmental, not pipeline defects.
- Fedora bootc images declare no default root filesystem: every builder needs
  an explicit ext4/xfs choice (`--rootfs` / `--bootc-default-fs` /
  `--filesystem`).
- `incus console` on the incus-migrate-created VM errors ("Chardev user does
  not support chardev hotswap") while a stock VM consoles fine — unexplained,
  low priority.
