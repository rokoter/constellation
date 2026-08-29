# constellation — project brief for AI coding agents

Model-agnostic onboarding context. Any coding agent (Claude Code, or a local
LLM via aider/opencode/continue/etc.) should read this first, then `docs/`.
Keep this file short and factual; depth lives in `docs/`.

## What this is

Infrastructure-as-code for a home **Talos Linux fleet**. Umbrella-monorepo:
all clusters + shared platform + docs in one place. Goals: reproducible,
GitOps-driven, and deliberately **anti-e-waste** (reuse old hardware, but
isolate it safely).

Guiding principle throughout: **start simple ("less is more"); add complexity
as an explicit, separate step.**

## Fleet model

| cluster | hardware | role | uptime |
|---|---|---|---|
| `voyager` *(planned)* | 3× N100-class mini-PC | always-on hub: management-plane (Git, GitOps, provisioning), observability, 24/7 apps | permanent, UPS |
| `starstuff` | 3× HP DL320 G8 on Proxmox | burst-compute in a flight case | powered on occasionally |
| `moonbase`/`mars`/`earth` *(later)* | TBD | workload spokes, provisioned from `voyager` | on demand |

Every cluster has its own CA + etcd and keeps running autonomously when
`voyager` is down. `voyager` is management / observability / DR — **not** a
runtime dependency of the others.

## Current state (2026-08-30)

- `starstuff` **CP1 bootstrapped**: single-node, healthy (`talosctl health`
  all-green), Talos v1.13.9, Kubernetes v1.36.3, Flannel CNI.
- CP1 still named `controlplane1`; rename to `carbon` is staged in
  `talos/starstuff/controlplane.yaml` but **not yet applied**.
- Not done yet: CP2/CP3, `voyager`, any platform services, GitHub remote.
- Next concrete steps: `docs/roadmap.md` §0 (GitHub repo) and §1 (rename CP1 →
  add CP2/CP3 → control-plane VIP).

## Repo layout

```
docs/            from-scratch.md · roadmap.md · wiki.md · eink-status-display.md
talos/<cluster>/ machine-config bootstrap (pre-GitOps) + patches/ + README
clusters/        GitOps roots per cluster (Argo/Flux)          — not created yet
platform/        shared kustomize bases                         — not created yet
apps/            application manifests                          — not created yet
.github/workflows/validate.yml   CI: yamllint + talosctl + patch check
```

## Conventions

- **Git flow**: trunk-based on `main`. Short `feat/`/`fix/`/`docs/` branches +
  PR (even solo — CI + a review moment). Milestones/epics = GitHub Issues.
  "Project Voyager" is a milestone, **not a separate repo**. Separate repos
  only for genuinely standalone products (e.g. ESP32 firmware
  `constellation-eink-display`).
- **Naming**: spaceflight theme for clusters. Nodes = **chemical elements
  where the element group encodes the node type**: CNO = control plane
  (`carbon`/`oxygen`/`nitrogen`), noble gases = `voyager`
  (`helium`/`neon`/`argon`), transition metals = compute, dense metals =
  storage, semiconductors = GPU. See `docs/from-scratch.md` → "Hostname-schema".
- **Secrets**: never committed in plaintext. Gitignored: `secrets.yaml`,
  `controlplane*.yaml`, `worker*.yaml`, `talosconfig`, `kubeconfig`. Future:
  SOPS+age for encrypted secrets that may enter the repo.

## Gotchas

- **Use `HostnameConfig`, not `machine.network.hostname`**: Talos 1.12+ puts
  hostname in a separate `HostnameConfig` document. Having both → `static
  hostname is already set in v1alpha1 config`. `auto` and `hostname` are
  mutually exclusive; a strategic-merge patch does not remove `auto: stable`,
  so edit the generated config directly. See `docs/wiki.md` → "Hostname
  configureren".
- **Keep versions aligned**: `talosctl` client == Talos node (now v1.13.9);
  `kubectl` within ±1 minor of the k8s server (now v1.36.3).
- **QEMU guest-agent is not running** in the current `starstuff` image →
  Proxmox backups are crash-consistent only. Fix via Talos Image Factory +
  `talosctl upgrade`. `docs/roadmap.md` §5.
- **State-changing `talosctl` commands** (`apply-config`, `bootstrap`,
  `upgrade`) are run by the human, not the agent. Read-only `talosctl get …` /
  `health` are fine for an agent. (Claude Code's auto-mode enforces this; other
  agents should follow the same rule.)
- Run `talosctl` commands from `talos/starstuff/`, or pass the full `--file`
  path. Proxmox PBS storage-ID = `pbs-ugreen`.

## Start here

1. `docs/from-scratch.md` — reproducible runbook: Proxmox → working cluster.
2. `docs/roadmap.md` — fleet model + numbered open items in order.
3. `docs/wiki.md` — annotated worklog + troubleshooting history (the *why*).
