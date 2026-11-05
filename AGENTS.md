# AGENTS.md — pod-crabbox-k3s-krun

The **krun-guest k3s lease image** for the Crabbox `containerd` provider: the
`pod-crabbox-k3s-krun` image (`charly.yml`) and the `krun-lease` candy
(`candy/krun-lease/`) it composes on top of `opencharly/pod-crabbox-k3s`. This
repo owns exactly the krun delta — the `libkrunfw-crabbox` guest kernel, the
`/.krun_vm.json` metadata and the `crabbox.krun.network` label — plus the probes
that repeat the inherited k3s/lease contract at this image's boundary. The
lease contract, systemd, the inner engine and k3s belong to the base repos; a
change to any of them lands there, not here.

Canonical files:

- `charly.yml` — the image: `base: ghcr.io/opencharly/pod-crabbox-k3s` at a
  pinned tag, `entrypoint: [/sbin/init]`, `user: root`.
- `candy/krun-lease/charly.yml` — the downloaded kernel recipe and fragment,
  the `capability.oci_label` block (where `crabbox.krun.network=passt` lives),
  the VM metadata and the `check:` steps.
- `.github/workflows/image.yml` — the image build; it triggers on
  `tag-on-merge` **completion** (`workflow_run`), never on `push: tags`
  (`opencharly/.github#277`). The `tag-on-merge.yml` caller is installed on
  `main` by the org bootstrap and is not part of a PR here.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-image:image` — the `candy:` image entry (base, user, entrypoint).
- `/charly-image:layer` — the candy body: `distro:` packages, plan step verbs,
  and the `check:` surface.
- `/charly-check:check` — the check verbs, the ADE requirement, and the R10 gate.
- `/charly-internals:git-workflow` — before any git/PR action.

## What is unique here

- **The image tag IS the git tag.** `.github/workflows/image.yml` passes the
  newest `v*` tag (minus its `v`) to `charly box build --tag`.
- **The base is a pinned OCI artifact.** `base:
  ghcr.io/opencharly/pod-crabbox-k3s:<tag>` pulls the published k3s image.
  Advancing the base is a one-line tag bump, produced first by the k3s repo
  (producer-first).
- **The kernel comes from layer-krun's recipe, not from a package.** libkrunfw
  v5.6.2's own sources, patches and stock config, plus
  `config-libkrunfw-crabbox.fragment` fetched from the PINNED `layer-krun` tag,
  then `make vmlinux` only. The build asserts every required option survived
  `olddefconfig`; do not weaken those assertions — a silently reduced kernel
  surfaces later as `Diag module missing (-2)`.
- **A stock libkrunfw kernel is not a substitute.** Its config disables
  `CHECKPOINT_RESTORE` and the socket diag options, so in-guest CRIU and the k3s
  netfilter set fail on it (LIBKRUN.md §7).
- **`ARCH` must be the KERNEL's spelling for the kernel step.** The box
  bootstrap exports `ARCH=amd64`; libkrunfw's Makefile treats a non-empty `ARCH`
  as the kernel arch and copies `config-libkrunfw_amd64`, which does not exist.
  The step maps `uname -m` to `x86_64`/`arm64`/`riscv` itself.
- **`/.krun_vm.json` keys are `kernel_path` and `kernel_format`.** LIBKRUN.md §6
  is the reference; do not rename them.
- **The label is the provider's network signal.** `crabbox.krun.network=passt`
  makes the provider select the passt backend; the label and the docs must stay
  in step.

## Build / validate / test

- `charly box validate` at the repo root.
- `charly box generate pod-crabbox-k3s-krun` — inspect the emitted Containerfile,
  the label set, and the baked `ai.opencharly.description` plan.
- `charly check box pod-crabbox-k3s-krun` — the build-scope `check:` steps.
- The PR image build builds and publishes nothing; the merge-time
  `workflow_run` path publishes
  `ghcr.io/opencharly/pod-crabbox-k3s-krun:<git-tag>`.
- The kernel build is the long pole of both the local gate and CI: it downloads
  the `linux-6.12.109` tarball and compiles `vmlinux`.

## Modify this repo

- Keep the base tag in step: a base-contract, k3s or systemd fix lands in its
  owning repo first, then this repo's `base:` tag advances.
- The fragment URL carries the `layer-krun` tag; bump it together with any
  recipe change and cite the layer's CHANGELOG.
- Every claim in the plan needs a deterministic `check:`. A kernel that merely
  installs without the ELF and size probes is not proof.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly`. Do not restate its rules here.
