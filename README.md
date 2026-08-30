# constellation

Infrastructure-as-code voor een Talos-Linux-**fleet** thuis. Umbrella-monorepo:
alle clusters, gedeeld platform en documentatie op één plek. GitOps-bron van
waarheid. Agent-/LLM-onboarding: [`AGENTS.md`](AGENTS.md). Werkverdeling
mens/AI: [`CREDITS.md`](CREDITS.md).

## Fleet

| cluster | hardware | rol |
|---|---|---|
| `sol` *(gepland)* | 3× N100-class mini-PC, zuinig | always-on base: fleet-management, observability, DR, 24/7-platformapps |
| `starstuff` | 3× HP DL320 G8 (Proxmox) | bootstrap + burst-compute in een flight case — gaat af en toe aan |
| `voyager` *(later)* | n.t.b. | eerste autonome cluster, gelanceerd/beheerd vanuit `sol` |
| `moonbase` / `mars` / `earth` *(later)* | n.t.b. | extra autonome workload-clusters, beheerd vanuit `sol` |

Mentaal model: **`starstuff` bouwt `sol`; `sol` lanceert `voyager`s.** Elke
cluster heeft een eigen CA + etcd + control plane en draait autonoom door als
`sol` weg is. `sol` is beheer/observability/DR — een
management-afhankelijkheid, **geen** runtime-afhankelijkheid. `sol` bestaat
nog niet; `starstuff` (CP1 `carbon`) is de huidige realiteit.
Zie [`docs/roadmap.md`](docs/roadmap.md) → "Fleet-model".

## Layout

```
docs/                 alle documentatie
  from-scratch.md      reproduceerbare runbook Proxmox → werkende cluster
  roadmap.md           fleet-model, tiers, open items
  wiki.md              geannoteerde worklog + troubleshooting
  eink-status-display.md   side quest
talos/<cluster>/       machine-config bootstrap (pre-GitOps)
  patches/             node-specifieke, veilige patches (hostname per element)
  README.md            gen-config recept
clusters/<cluster>/    GitOps-roots (Argo/Flux entrypoint)          — later
platform/              gedeelde kustomize-bases                     — later
apps/                  applicatie-manifests                          — later
```

## Conventies

- **Naamgeving**: ruimtevaart-thema. Clusters = missies/plaatsen; nodes =
  elementen waarbij de elementgroep het nodetype codeert (CNO = control plane,
  aardalkali = base-cluster `sol`, edelgassen = autonome managed clusters
  (`voyager`, `moonbase`, …), overgangsmetalen = compute, dichte metalen =
  storage). Details in `docs/from-scratch.md` → "Hostname-schema".
- **Git-flow**: trunk-based op `main`, korte `feat/` `fix/` `docs/`-branches
  met PR. CI valideert Talos-configs en manifests. Milestones/epics als
  GitHub Issues.
- **Secrets**: nooit plain in Git. `secrets.yaml` / machine-configs /
  `talosconfig` / `kubeconfig` staan in `.gitignore`. Toekomst: SOPS+age voor
  versleutelde secrets die wél de repo in mogen.

## Huidige staat

`starstuff` CP1 gebootstrapt (single-node), gezond. Zie
[`docs/from-scratch.md`](docs/from-scratch.md) → "Huidige staat".
