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
| `sol` *(planned)* | 3× N100-class mini-PC | always-on base: fleet management-plane (Git, GitOps, provisioning), observability, DR, 24/7 apps | permanent, UPS |
| `starstuff` | 3× HP DL320 G8 on Proxmox | bootstrap/genesis + burst-compute in a flight case | powered on occasionally |
| `voyager` *(later)* | TBD | first autonomous cluster, launched/managed from `sol` | on demand |
| `moonbase`/`mars`/`earth` *(later)* | TBD | further autonomous workload clusters, managed from `sol` | on demand |

Mental model: **`starstuff` builds `sol`; `sol` launches `voyager`s.** Every
cluster has its own CA + etcd + control plane and keeps running autonomously
when `sol` is down. `sol` is management / observability / DR — a management
dependency, **not** a runtime dependency of the others. `sol` does not exist
yet; `starstuff` (CP1 `carbon`) is the current reality.

## Current state (2026-08-30)

- `starstuff` **CP1 = `carbon`**: single-node control plane, `Ready`, IP
  `10.30.4.1` (VLAN 4), Talos v1.13.9, Kubernetes v1.36.3, Flannel CNI.
  Re-validated by a clean-room bootstrap on 2026-08-30.
- GitHub repo live: `rokoter/constellation` (private). Issues #2–#6 open
  (docs/backup/bootstrap work; #1 closed — schematic ID recorded in
  `docs/from-scratch.md` §4).
- Not done yet: CP2 (`oxygen`) / CP3 (`nitrogen`), control-plane VIP, `sol`
  (the planned always-on base, bootstrapped from `starstuff`), `voyager` and
  later fleet members, any platform services.
- Next concrete steps: `docs/roadmap.md` §1 — CP2 on host 2 per
  `talos/starstuff/README.md` §5–§7, then CP3, then VIP.
- Bootstrap lessons: `docs/session-2026-08-30-cp1-bootstrap.md` (first
  bootstrap, VLAN 9) and `docs/session-2026-08-30-clean-room-cp1.md`
  (clean-room re-run, VLAN 4).

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
  The `sol` build-out is a milestone, **not a separate repo**. Separate repos
  only for genuinely standalone products (e.g. ESP32 firmware
  `constellation-eink-display`).
- **Naming**: spaceflight theme for clusters. Nodes = **chemical elements
  where the element group encodes the node type**: CNO = control plane
  (`carbon`/`oxygen`/`nitrogen`), alkaline-earth metals = base cluster `sol`
  (`beryllium`/`magnesium`/`calcium`), noble gases = autonomous managed
  clusters (`voyager` = `helium`/`neon`/`argon`, `moonbase` = `krypton`/…),
  transition metals = compute, dense metals = storage, semiconductors = GPU.
  See `docs/from-scratch.md` → "Hostname-schema".
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
  A **clean-room / fresh-machine test must start from an empty
  `~/.talos/config`** — verify with `talosctl config contexts` before `gen
  secrets`. Otherwise `config merge` silently appends `-1`/`-2`/`-3` to the
  context name (`starstuff-3` in the 2026-08-30 clean-room run) and the
  accumulation makes the stale-CA trap easy to hit.
- **Runbook network values are environment-specific**: `docs/from-scratch.md`
  now uses VLAN 4 / `10.30.4.x` (Proxmox mgmt `10.30.3.x`) from the 2026-08-30
  clean-room run; the first bootstrap used VLAN 9 / `10.3.9.x`. Gateway,
  subnet prefix and DNS were not captured — substitute your own; don't treat
  any of these as canonical infra facts.
- **`talosctl health` timeout ≠ failure**: "waiting for all k8s nodes to
  report" → `context canceled` often just means the kubelet is still
  registering. Try `kubectl get nodes` before debugging further. (Did not
  recur in the 2026-08-30 clean-room run — health passed in one go.)
- See `docs/session-2026-08-30-cp1-bootstrap.md` and
  `docs/session-2026-08-30-clean-room-cp1.md` for the CP1 bootstrap
  post-mortems.

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

Known weak spots a fresh session will likely hit: the Image Factory schematic
ID is now recorded in `docs/from-scratch.md` §4, but its exact extension set
is still unverified; `talosctl` v1.13.9 + `kubectl` 1.36 must be installed
first; a fresh clone has no `secrets.yaml` so §6 `talosctl gen secrets` really
runs; verify generated secrets do **not** show in `git status`; start from an
empty `~/.talos/config` (see the stale-CA gotcha).
`docs/from-scratch.md` is the authoritative runbook;
`talos/starstuff/README.md` is the short recipe.

## Start here

1. `docs/from-scratch.md` — reproducible runbook: Proxmox → working cluster.
2. `docs/roadmap.md` — fleet model + numbered open items in order.
3. `docs/wiki.md` — annotated worklog + troubleshooting history (the *why*).
4. `docs/session-*.md` — past session post-mortems.
