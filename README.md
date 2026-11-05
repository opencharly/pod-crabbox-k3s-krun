# pod-crabbox-k3s-krun

The **krun-guest k3s CachyOS lease image** for the Crabbox `containerd` provider —
the [`pod-crabbox-k3s`](https://github.com/opencharly/pod-crabbox-k3s) control-plane
lease contract plus the custom `libkrunfw-crabbox` guest kernel.

## What it provides

Every lease contract property of the base image is inherited unchanged:

| Property | Value |
|---|---|
| Base | `ghcr.io/opencharly/pod-crabbox-k3s:2026.309.0459` (CachyOS) |
| SSH | `sshd` on **2222**, public-key only, user `crabbox` |
| Inputs | read-only `/crabbox`: `authorized_keys`, `lease.env`, `bootstrap.sh` |
| Readiness | `/usr/local/libexec/crabbox/ready` — polled over SSH until exit 0 |
| PID 1 | `/sbin/init` → `/usr/lib/systemd/systemd` |
| k3s | pinned `v1.37.0+k3s1` on the inner containerd, node-ip `172.31.255.1` |

This variant adds the krun guest:

| Property | Value |
|---|---|
| Guest kernel | `/krun/vmlinux` — ELF, built from libkrunfw v5.6.2 + `config-libkrunfw-crabbox.fragment` |
| Kernel config | the fragment keeps `CHECKPOINT_RESTORE`, the socket diag options and the k3s netfilter/IPVS/VXLAN set |
| VM metadata | `/.krun_vm.json` → `kernel_path: /krun/vmlinux`, `kernel_format: 1` |
| Label | `crabbox.krun.network=passt` |

The kernel is built at image build time from libkrunfw v5.6.2's own sources,
patches and stock config — the same recipe `opencharly/layer-krun` uses — and
only the `vmlinux` target is built. A stock libkrunfw kernel is not enough:
its config disables `CHECKPOINT_RESTORE` and the socket diag options, so
in-guest CRIU and the k3s netfilter set fail on it (LIBKRUN.md §7). The build
asserts every required option survived `olddefconfig` and fails otherwise.

## How to use it

The provider runs this image as a **krun** guest:

```sh
# the provider's containerd config, not a shell command
krunKernel: image
krunNetwork: passt
```

The label is why `krunNetwork: passt` is the right backend: passt is what
CRIU's own loopback self-test needs inside the guest (LIBKRUN.md §7). The host
must pass `/dev/kvm` to the VMM; the image cannot do that for itself.

## Requirements and limits

- **The image build needs network.** `libkrunfw`'s Makefile fetches the
  `linux-6.12.109` source tarball, and the pinned config fragment is fetched
  from `raw.githubusercontent.com` at the `layer-krun` tag. Both are pinned
  URLs; neither is resolved from a mutable branch.
- **The kernel is not a distro package.** No Arch/CachyOS package ships the
  `libkrunfw-crabbox` kernel this image needs, which is why the layer builds it.
  The toolchain stays in the image so the pin can be rebuilt without a new base.
- **Nothing in the image starts a VMM.** The guest boots the inherited k3s
  control plane; the VMM is the host's.

## Layout

- `charly.yml` — the `pod-crabbox-k3s-krun` image (pinned base image, label,
  systemd entrypoint).
- `candy/krun-lease/` — the kernel build, the VM metadata, and the `check:`
  steps that prove the row.
- `.github/workflows/image.yml` — builds on `pull_request` and publishes at the
  merge tag, so the image tag equals the git tag.
- `CHANGELOG/` — per-CalVer history.

## Related

- [`opencharly/pod-crabbox-k3s`](https://github.com/opencharly/pod-crabbox-k3s) —
  the k3s lease image this one imports.
- [`opencharly/layer-krun`](https://github.com/opencharly/layer-krun) — the
  `libkrunfw-crabbox` kernel recipe and its config fragment.
- [`CONTAINERD-IMAGES.md`](https://github.com/opencharly/opencharly/blob/main/CONTAINERD-IMAGES.md)
  §2 — the lease-image contract and this image's row.
- [`LIBKRUN.md`](https://github.com/opencharly/opencharly/blob/main/LIBKRUN.md) §6–§7 —
  the `.krun_vm.json` keys, passt, and why the custom kernel exists.
