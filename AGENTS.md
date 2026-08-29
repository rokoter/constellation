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

- `starstuff` **CP1 = `carbon`**: single-node control plane, `Ready`, Talos
  v1.13.9, Kubernetes v1.36.3, Flannel CNI.
- GitHub repo live: `rokoter/constellation` (private). Issues #1–#4 open
  (docs/backup work).
- Not done yet: CP2 (`oxygen`) / CP3 (`nitrogen`), control-plane VIP,
  `voyager`, any platform services.
- Next concrete steps: `docs/roadmap.md` §1 — CP2 on host 2 per
  `talos/starstuff/README.md` §5–§7, then CP3, then VIP.
- Bootstrap lessons from the manual CP1 run:
  `docs/session-2026-08-30-cp1-bootstrap.md`.

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
- **Definition of done** (hard rule): a task is done only when (1) it works
  and is verified, (2) the relevant docs (README/wiki/from-scratch) are
  updated to reflect the new state, (3) lessons/gotchas are written into the
  docs themselves, not just the commit message. The doc update is part of the
  same task/PR — never a "later" action. Applies to small tasks too.

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
- **Stale talosconfig/CA before `bootstrap`**: after any `gen secrets`/`gen
  config` cycle, run `talosctl config contexts` and remove old contexts with
  the same cluster name (`talosctl config remove <name>`). Otherwise
  `bootstrap` picks a cached CA → `x509: certificate signed by unknown
  authority`. Pass explicit `--talosconfig ./talosconfig -e <ip> -n <ip>` to
  be safe. Follow §6→§7 in exact order; status-check after every step.
- **`talosctl health` timeout ≠ failure**: "waiting for all k8s nodes to
  report" → `context canceled` often just means the kubelet is still
  registering. Try `kubectl get nodes` before debugging further.
- See `docs/session-2026-08-30-cp1-bootstrap.md` for the full CP1 bootstrap
  post-mortem.

## Logging a test / bootstrap session

When a session tests the repo by walking a runbook (esp. a clean bootstrap on
a fresh machine), capture friction **as you go**:

1. `cp docs/session-TEMPLATE.md docs/session-<YYYY-MM-DD>-<slug>.md` and fill
   it in step by step — do not reconstruct at the end.
2. For every problem: exact error line → cause → fix → which doc is missing
   what.
3. At the end: turn the doc-gaps into `gh issue create` calls, and fold any
   cross-version gotchas into this file's "Gotchas" section (that's a DoD
   doc-update, part of the same task).
4. Commit the session file on a branch `session/<YYYY-MM-DD>` and push, so it
   comes back as a PR — or the human pastes it into the main chat.

Known weak spots a fresh session will likely hit: the empty Image Factory
schematic-ID placeholder (issue #1); `talosctl` v1.13.9 + `kubectl` 1.36 must
be installed first; a fresh clone has no `secrets.yaml` so §6 `talosctl gen
secrets` really runs; verify generated secrets do **not** show in
`git status`. `docs/from-scratch.md` is the authoritative runbook;
`talos/starstuff/README.md` is the short recipe.

## Start here

1. `docs/from-scratch.md` — reproducible runbook: Proxmox → working cluster.
2. `docs/roadmap.md` — fleet model + numbered open items in order.
3. `docs/wiki.md` — annotated worklog + troubleshooting history (the *why*).
4. `docs/session-*.md` — past session post-mortems.
