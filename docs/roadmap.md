# Roadmap — Talos-fleet

Losse items, grofweg op volgorde. Huidige staat: [`from-scratch.md`](./from-scratch.md).
Achtergrond/troubleshooting: [`wiki.md`](./wiki.md). Repo-opzet: [`../README.md`](../README.md).

## Fleet-model

Mentaal model: **`starstuff` bouwt `sol`; `sol` lanceert `voyager`s.**

| cluster | hardware | rol | uptime |
|---|---|---|---|
| **`sol`** *(gepland)* | 3× N100-class mini-PC, zuinig | **always-on base / fleet-hub**: management-plane (Git, GitOps, provisioning), observability, DR-coördinatie, en de "echte" 24/7-platformapps | permanent aan, UPS |
| **`starstuff`** | 3× HP DL320 G8 | **bootstrap/genesis + burst-compute** in een flight case — bouwt `sol`, daarna zware/bulk workloads en dev/test | gaat af en toe aan |
| `voyager` *(later)* | n.t.b. | eerste **autonome** cluster, gelanceerd/beheerd vanuit `sol` — gewoon fleet-lid, geen hub | naar behoefte |
| `moonbase` / `mars` / `earth` (later) | n.t.b. | extra autonome workload-clusters, beheerd vanuit `sol` | naar behoefte |

`sol` bestaat nog niet — `starstuff` (CP1 `carbon`) is de huidige realiteit.
Doel-architectuur: `starstuff` → bootstrapt `sol` → `sol` beheert de fleet.
De rollen zijn hiermee omgedraaid t.o.v. de oorspronkelijke wiki (waar
`starstuff` zelf de always-on laag was).

Elk cluster heeft een **eigen control plane + etcd + eigen CA** en draait
autonoom door als `sol` weg is (laatst toegepaste Talos-config + laatst
gesyncte GitOps-staat; reconcile bij herstel). `sol` is een
**management-afhankelijkheid, geen runtime-afhankelijkheid**: managed clusters
draaien hun Kubernetes-workloads gewoon door als `sol` offline is.

"Een nieuwe laag starten": config toevoegen aan Git op `sol` → de
fleet-tooling op `sol` past Talos-config toe op de nieuwe nodes → GitOps
bootstrapt de workloads. `starstuff` speelt geen rol (mag uit staan).

### Levenscyclus

1. Er bestaat nog niets.
2. Bouw / start `starstuff`.
3. Gebruik `starstuff` + de werkstation-/bootstrap-tooling om `sol` te maken.
4. Breng `sol` naar een volledig zelfstandige, gezonde staat.
5. `starstuff` mag nu uit — het is geen runtime-afhankelijkheid meer.
6. `sol` is voortaan de normale always-on fleet-management/base-cluster.
7. Maak `voyager` en latere clusters aan vanuit de fleet-tooling op `sol`.
8. `voyager` en alle latere clusters draaien zelfstandig door als `sol`
   onbereikbaar is.

`sol`'s pre-GitOps Talos-config komt t.z.t. in `talos/sol/` (nog niet
aangemaakt), de GitOps-root in `clusters/sol/`.

---

## 0. Git-flow opzetten

- [x] Umbrella-monorepo `constellation`, monorepo-layout (`docs/`, `talos/`,
      later `clusters/`, `platform/`, `apps/`).
- [x] GitHub-repo `rokoter/constellation` (private) live, `main` gepusht.
- [x] Eerste docs-issues aangemaakt: #1–#4 (Image Factory schematic ID,
      config-naamgeving, multi-node recept, secrets.yaml YubiKey-backup).
- [ ] `main` beschermen: overgeslagen voor nu (solo/private) — aanzetten via
      Settings → Rules → Rulesets zodra een 2e persoon meedoet.
- [ ] GitHub Project + `sol`-milestone (voorheen "Voyager" genoemd); overige
      roadmap-items → issues.
- [ ] Renovate aanzetten (chart-/image-/Talos-versies).
- [ ] Later: SOPS+age zodat versleutelde `secrets.yaml` per cluster wél de
      repo in kan (nu nog gitignored).

## 1. `starstuff` control plane afmaken

- [x] **CP1 hernoemd** `controlplane1` → `carbon` (2026-08-30). Node Ready.
      Lessen: `session-2026-08-30-cp1-bootstrap.md`. Clean-room-herbootstrap op
      VLAN 4 (`10.30.4.1`): `session-2026-08-30-clean-room-cp1.md`.
- [ ] **CP2 `oxygen`** (`10.30.4.2`, host `pve-dl320-2`, VMID 101) — zelfde
      `secrets.yaml`, `HostnameConfig: oxygen`, `apply-config --insecure`,
      **geen** `bootstrap`.
- [ ] **CP3 `nitrogen`** (`10.30.4.3`, host `pve-dl320-3`, VMID 102) — idem.
- [ ] **Control-plane VIP** (`machine.network.interfaces[].vip`) zodat de API
      niet aan `carbon` hangt.
- [ ] Verify: 3 etcd members, 3× Ready, VIP-failover getest.
- [ ] **Graceful power-on/off** van de flight case: shutdown-volgorde
      (workloads → drain → `talosctl shutdown`), en een health-gate bij
      opstarten voor GitOps weer mag reconcilen.

## 2. Hardware

- [ ] **`sol`-nodes**: 3× N100/N150/N305 mini-PC (Beelink EQ, GMKtec,
      Topton/CWWK). 16 GB min, 32 GB beter (VM + Grafana + Gitea + HA + Argo
      tikt aan). NVMe 500 GB–1 TB, liefst een 2e NVMe voor replicated storage.
      ~30 W always-on voor het drietal. Geen ECC op de meeste N100 — accepteren
      of Ryzen-embedded als het moet.
- [ ] **UPS** op de `sol`-stack + NUT voor nette shutdown.
- [ ] **Eigen VLAN/subnet** voor `sol`, los van VLAN 4 (`starstuff`).
- [ ] **SSD-swap DL320-hosts**: 2× Samsung PM1633a 480 GB SAS per host in
      ZFS mirror (`rpool`), i.p.v. de trage HDD's.

## 3. Platform-diensten — **op `sol`**

> Dit is het "eigenlijke doel" uit de oude wiki, verplaatst van `starstuff`
> naar `sol` (de geplande always-on base-cluster). Alles GitOps vanuit Git op
> `sol`.

### Tier 0 — Talos/k8s-basis

- [ ] 3-node combined control-plane+worker (`allowSchedulingOnControlPlanes`),
      eigen `secrets.yaml`.
- [ ] CNI: Flannel (simpel) of Cilium (NetworkPolicy + observability, zwaarder).
- [ ] **Replicated storage** over de 3 NVMe: Longhorn (pragmatisch op kleine
      hw) of OpenEBS replicated-lvm. `local-path` alleen voor wegwerp.
- [ ] LoadBalancer-IP's: MetalLB of Cilium L2.
- [ ] cert-manager (Let's Encrypt).

### Tier 1 — management-plane (de reden dat `sol` bestaat)

- [ ] **Gitea/Forgejo** — dé git-bron van waarheid voor álle clusters
      (platform-repo + repo/dir per cluster). Push-mirror naar GitHub voor
      een off-site kopie.
- [ ] **GitOps-controller** — Argo CD (multi-cluster UI, "app of apps",
      registreert `starstuff`/`moonbase`/… als remote clusters). Flux als
      lichter alternatief.
- [ ] **Secrets** — SOPS + age; encrypted secrets (incl. elke cluster-CA /
      `secrets.yaml`) in Git. OpenBao later voor dynamische secrets.
- [ ] **Cluster-provisioning** (hoe een nieuwe laag "vanaf `sol` start"):
      - **Start:** `talosctl` + configs in Git + een Gitea-Actions-runner die
        `apply-config` / `upgrade` doet. Simpel, weinig magie.
      - **Groeipad:** self-hosted **Omni** op `sol` — machine-onboarding
        via SideroLink (WireGuard), config-templates, image factory,
        cluster-lifecycle. Managed clusters blijven draaien als Omni down is.
        Beste fit voor "spin up moonbase/mars van hieruit".
      - **Alternatief:** Cluster API + Talos-providers (`CABPT`/`CACPPT`) —
        GitOps-native, zwaarder in beheer.

### Tier 2 — observability (belangrijk, always-on)

- [ ] **VictoriaMetrics** i.p.v. vanilla Prometheus — veel lichter op N100,
      lange retentie op kleine disk. + **vmalert**/Alertmanager → e-mail
      (`koen@terstal.it`) + ntfy/push. Alerts vuren vanaf `sol`, dus ook
      als `starstuff` uit is.
- [ ] **Grafana** — per-cluster dashboards + een fleet-overzicht.
- [ ] **`starstuff` → `sol` metrics**: `starstuff` doet `remote_write`
      naar VM op `sol` als het aan staat (push, overleeft intermitterend
      + NAT). Historie blijft over de aan/uit-cycli heen.
- [ ] **Blackbox / Uptime Kuma** — probes op iLO's, cluster-API-VIP's,
      externe diensten, ISP.
- [ ] Loki (single-binary) — centrale logs, optioneel, v2.

### Tier 3 — netwerk/platform waar alles op leunt

- [ ] **Interne DNS** — Blocky of AdGuard/Pi-hole (adblock + `*.lab`-zones).
      Mag niet van `starstuff` afhangen.
- [ ] **Ingress** — Traefik of ingress-nginx.
- [ ] **Remote access** — Cloudflare Tunnel (geen port-forward, auth via CF
      Access) of een WireGuard-gateway. Exposeert Grafana/Gitea/statuspagina/
      HA — **nooit** cluster-API's.
- [ ] **SSO** (later) — Authentik / Pocket-ID / Keycloak: één login voor
      Grafana/Gitea/Argo.

### Tier 4 — de "echte" 24/7-apps

- [ ] **Home Assistant** — always-on, en het knooppunt voor het e-ink-scherm
      + "deelbaar op internet".
- [ ] **`cluster-status` aggregator** (zie `docs/eink-status-display.md`) —
      leest alle clusters, één JSON, voedt HA + het scherm.
- [ ] **Statuspagina** — Gatus / Uptime Kuma (publieke view).
- [ ] Vaultwarden / Paperless / dashboards — wat 24/7 up moet en in het
      N100-budget past.

## 4. Backups & DR

- [ ] **Backup-orchestrator op `sol`**: restic/Velero + een MinIO- of
      NAS-target. Dekt etcd-snapshots van **alle** clusters + PV-backups.
      Off-site kopie via restic → B2/S3.
- [ ] **etcd-snapshots** per cluster via cron
      (`talosctl -n <VIP> etcd snapshot`). `sol`'s eigen snapshot = het
      kroonjuweel — off-site.
- [ ] **PBS**: storage-ID `pbs-ugreen`, backup-job op DL320-host 2 en 3
      (namespaces `pve-dl320-2/3`). VM-snapshots van Talos-guests laag
      prioriteren — herbouw uit config is sneller/veiliger.

## 5. Operationeel

- [ ] **QEMU guest-agent** draait niet in het huidige `starstuff`-image
      (backup-log `agent configured but not running`; niet in
      `talosctl services`). Image via Image Factory herbouwen mét
      `siderolabs/qemu-guest-agent` + `talosctl upgrade`. Meenemen bij de
      storage-extensions (`zfs`/`iscsi-tools`/`nfs-utils` zodra storage
      gekozen is).
- [ ] **Firewall** (Unifi): mgmt-VLAN → VLAN 4 op 6443/50000/50001;
      `sol`-VLAN ↔ VLAN 4 alleen voor beheer. iLO-isolatie: zie §6.
- [ ] **Upgrade-runbook**: `talosctl upgrade` + `talosctl upgrade-k8s`,
      volgorde, kubectl mee-swappen, per cluster.
- [ ] **Security** — PSA staat al op `baseline/restricted`; verder
      NetworkPolicies, image-scanning, SSO.
- [ ] **GitHub/remote** — zie §0. Tracked: `docs/`, `talos/**/patches/`,
      `talos/**/README.md`, `.github/`, `README.md`, `.gitignore`. Nooit:
      `secrets.yaml`, `controlplane*.yaml`, `worker*.yaml`, `talosconfig`,
      `kubeconfig`.

## 6. Bare-metal bootstrap & recovery-netwerk (vervolgsessie)

> Vangnet om **vanaf 0** te kunnen beginnen (geen werkende Proxmox/cluster).
> Bewust **niet** nu bouwen — de huidige flow (ISO van Proxmox' lokale
> storage + `apply-config --insecure`) volstaat. Less is more om te starten.

> Gerelateerde issues: **#5** custom retro live-ISO (archiso, amber console,
> Ventoy) voor clean-room tests · **#6** PXE-netboot van diezelfde omgeving op
> een geïsoleerd "rode-netwerk"-VLAN (vervolg op #5, aparte sessie).

### Recovery-VLAN ("code red")

- [ ] Geïsoleerd VLAN **achter een proxy** (OPNsense of vergelijkbaar):
      - **uitgaand**: alleen gecontroleerd — GitHub (+ wat de tooling nodig
        heeft: `talos.dev`, GitHub-releases/CDN, distro-mirror). Alles anders
        droppen.
      - **inkomend**: nooit. Geen enkele poort vanaf buiten of andere VLANs.
      - **oost-west**: alleen terminal-pc / PXE-omgeving → de Proxmox-hosts +
        de Talos-VM's (voor `talosctl`/console). Verder niets.
- [ ] Eén klein, cluster-onafhankelijk apparaat (Raspberry Pi / mini-PC —
      niet op `sol`, want dat kan juist weg zijn):
      - **dnsmasq** = DHCP + TFTP + DNS
      - **nginx** = iPXE-script + Talos-assets (Image Factory `metal`) +
        statuspagina
- [ ] iPXE chainload → Talos-installer. Twee smaken:
      - baken de config in: `talos.config=http://boot.recovery/<node>.yaml`
      - of boot kaal en doe `apply-config --insecure` met de hand (past bij
        de huidige runbook — begin hiermee).
- [ ] "Captive-portal-achtig" = **een statische statuspagina** op een vast IP
      (`http://boot.recovery`): node↔IP-tabel, links naar de iLO's, de
      bootstrap-runbook, config-URL's + checksums. Geen echte captive portal
      (te complex).
- [ ] Werknaam voor de dienst: `launchpad` (waar je vandaan start).
- [ ] Volgorde: eerst fleet werkend (starstuff 3-node + `sol`), dán dit als
      los vangnet.

### iLO-isolatie (anti-e-waste: veilig oud spul gebruiken)

- [ ] iLO4 (DL320 G8) in een **eigen management-VLAN**, **geen route naar
      internet** — iLO4-firmware is oud/kwetsbaar, nooit blootstellen.
- [ ] Unifi-firewall: **alleen inbound** vanaf mgmt-VLAN / werkstation-IP →
      iLO op 443 (+ 17988/17990 remote console, 623 IPMI alleen indien nodig).
      Rest droppen. iLO's zonder default gateway (of gateway zonder
      internet-route).
- [ ] iLO4-hardening: laatste firmware (→ 2.x), TLS 1.2 forceren, zwakke
      ciphers uit, default creds weg, account per persoon, IPMI-over-LAN +
      SNMP uit indien ongebruikt. NTP naar interne bron; syslog → `sol`'s
      log-stack (als die er is).
- [ ] Later (optioneel, **niet** voor bootstrap): reverse proxy met auth
      (SSO of basic-auth + IP-allowlist) vóór de iLO's, zodat je van buiten
      via één geauthenticeerde hop bij de console kunt.

## 7. Naamgeving

- **Fleet-thema**: ruimtevaart. Clusters: `sol` (always-on base/hub),
      `starstuff` (bootstrap/genesis + burst), `voyager` (eerste autonome
      fleet-member), `moonbase` / `mars` / `earth` (latere autonome clusters).
- **Naamsemantiek** (zonder architectuur-voorkennis leesbaar):
      - `starstuff` — oermaterie / genesis, waar de fleet uit ontstaat
      - `sol` — stabiel centrum / thuis / basis
      - `voyager` — iets dat vanaf die basis naar buiten wordt gelanceerd
- **Node-thema**: elementen, elementgroep = nodetype (zie
      `from-scratch.md` → "Hostname-schema"):
      - control plane = CNO: `carbon`, `oxygen`, `nitrogen` (`starstuff`)
      - base-cluster / fleet-hub = aardalkalimetalen: `beryllium`,
        `magnesium`, `calcium` (`sol`) — stabiel, structureel, de vaste kern
      - autonome managed clusters = edelgassen (inert, self-contained):
        `voyager` = `helium`/`neon`/`argon`, `moonbase` =
        `krypton`/`xenon`/`radon`
      - compute-workers = overgangsmetalen, storage = dichte metalen, GPU =
        halfgeleiders
- Eerder overwogen voor de hub: een grondstation-naam (`houston` /
      `mission-control`). `sol` gekozen — drukt "stabiel centrum" uit zonder
      extra context en blijft binnen het ruimtevaart-thema.
- Elke cluster een eigen repo/submap met eigen `.gitignore` voor zijn
      `secrets.yaml` / machine-configs.

## 8. Side quests

- [ ] **E-ink statusscherm** (ESP32 + read-only aggregator, optioneel via
      Home Assistant). Brainstorm/ontwerp: `docs/eink-status-display.md`.
- [ ] **Lokaal LLM als dev-assistent**: `AGENTS.md` + `docs/` als
      start-context voeren aan een lokaal model (ollama/llama.cpp) met een
      agent-tool (aider/opencode/continue) op het Fedora-werkstation.
      Testen of een cold-start sessie het project kan oppakken uit alleen de
      repo-docs.
