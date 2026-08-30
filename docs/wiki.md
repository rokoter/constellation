# Talos-op-Proxmox bootstrap-cluster — wiki

## ⚡ Overdracht naar Claude Code — lees dit eerst

We zijn overgestapt van de Claude.ai-chatinterface naar Claude Code
(`~/talos-cluster`) om direct te kunnen troubleshooten zonder copy-paste.

### ✅ OPGELOST — blocker "static hostname is already set in v1alpha1 config"

`talosctl apply-config --insecure --nodes 10.3.9.10 --file controlplane.yaml`
gaf consequent:
```
error applying new configuration: rpc error: code = InvalidArgument desc = 1 error occurred:
        * static hostname is already set in v1alpha1 config
```

**Diagnose (2026-08-29):** niets mis met de node-runtime-state.
`talosctl get systemdisk --insecure` was leeg → geen half-geïnstalleerde
disk. `hostnamespec`/`hostnamestatus` leeg. De fout kwam puur uit het
config-bestand: dat zette de hostname op **twee wederzijds uitsluitende
manieren tegelijk**:

1. `machine.network.hostname: controleplane1` — legacy v1alpha1-veld,
   toegevoegd door onze `patches/carbon.yaml`
2. een los `HostnameConfig`-document (`auto: stable`) — sinds Talos 1.12
   genereert `talosctl gen config` dit automatisch aan het eind van
   `controlplane.yaml`

Talos 1.12+ weigert een config met allebei. Zie
[siderolabs/talos#13405](https://github.com/siderolabs/talos/discussions/13405)
en [#12573](https://github.com/siderolabs/talos/issues/12573).

**Fix (toegepast):**
- `machine.network.hostname` uit `controlplane.yaml` verwijderd
- in het `HostnameConfig`-document `auto: stable` vervangen door
  `hostname: controlplane1` (meteen de typefout `controleplane1` → `controlplane1`
  rechtgezet)
- `talosctl validate --config controlplane.yaml --mode metal` → **valid**

**Afgehandeld:** `apply-config --insecure` + `talosctl bootstrap` gedraaid.
CP1 is nu een gezonde single-node control plane — zie sectie "Control-plane
node 1 → Bootstrap voltooid" verderop.

**Talos-versie**: v1.13.9 (node én talosctl-client, matchend)
**Node**: CP1, IP 10.3.9.10, Proxmox host pve-dl320-1, VM ID 100
**Werkmap**: `~/talos-cluster`, bevat `controlplane.yaml`, `worker.yaml`,
`secrets.yaml`, `talosconfig` (alle 4 in `.gitignore`, nooit committen),
en `patches/carbon.yaml` (wél veilig, referentie voor de gewenste
`HostnameConfig`-eindstaat)

---

## Architectuur

- 3x HP ProLiant DL320 G8 v2, elk Proxmox VE 9.2 bare-metal
- Elke host draait 1 Talos control-plane VM (+ later een lichte worker-VM)
- **Rolverschuiving (zie `roadmap.md` → "Fleet-model"):** `starstuff` wordt
  **bootstrap/genesis + burst-compute** in een flight case die af en toe aan
  gaat. De always-on management-/monitoring-/git-laag verhuist naar een apart
  zuinig cluster **`sol`** (N100-nodes; in eerdere notities `voyager` genoemd
  — die naam is nu de eerste autonome fleet-member, gelanceerd vanuit `sol`).
  Elke cluster is autonoom (eigen CA + etcd); `sol` is de beheer-/DR-hub, een
  management-afhankelijkheid, geen runtime-afhankelijkheid.
- Oorspronkelijk doel van `starstuff` (semi-stabiele bootstrap-laag met
  monitoring, security, patch management, alerting, Gitea) = nu de rol van
  `sol`.

## Hardware per host

- CPU: Intel Xeon E3-1270 v3 @ 3.50GHz (8 threads)
- RAM: ~15.58 GiB
- SAS-controller: LSI SAS2308 (IT mode)
- Storage: 2x SAS HDD in ZFS mirror (`rpool`) → wordt vervangen door 2x Samsung
  PM1633a 480GB SAS SSD in ZFS mirror

## Proxmox

- Versie: 9.2.11 (Debian 13 Trixie)
- Kernel: 7.0.14-14-pve (na dist-upgrade)
- Repository: production-ready enterprise repo actief, subscription-check faalt (verwacht, no-subscription)

## Netwerk

- Unifi UCG Fiber, meerdere VLANs beschikbaar
- Talos-cluster-VLAN: **VLAN 9 — `k3s` — `10.3.9.0/24`**, gateway `10.3.9.254`
  (bestaand VLAN hergebruikt, naam wordt mogelijk nog aangepast)
- IP-toewijzing: DHCP-reservering op **MAC-adres** (niet hostname — Talos
  stuurt in maintenance-mode geen DHCP-hostname mee)
- Hostname wordt apart gezet in de Talos machine-config
  (`machine.network.hostname`), onafhankelijk van DHCP
- Geplande firewall-regel: alleen management-VLAN → VLAN 9 op poorten 6443
  (K8s API) en 50000/50001 (Talos API)
- Gepland (vervolgsessie, zie `roadmap.md` §6): iLO4-management-VLAN zonder
  internet-route, alleen inbound via Unifi-firewall; en een geïsoleerd
  recovery-VLAN ("code red") met dnsmasq/PXE voor bare-metal bootstrap
  vanaf 0

## VM-specificaties (per control-plane node)

- vCPU: 2, type `host`
- RAM: 4096 MB
- Disk: 32GB
- Machine type: `q35`
- BIOS: OVMF (UEFI) + EFI-disk
- SCSI-controller: VirtIO SCSI (**niet** "Single" — kan bootstrap-hangs geven)
- Netwerkmodel: VirtIO
- QEMU Guest Agent: aan (extensie zit in custom Talos-ISO)

## Talos image

- Gebouwd via Talos Image Factory
- System extensions: alleen `siderolabs/qemu-guest-agent`
  (bewust minimaal gehouden; overige extensies zoals zfs/iscsi-tools/nfs-utils
  pas toevoegen zodra de storage-oplossing voor persistent volumes bekend is)

## Werkstation-tooling (Fedora)

- `talosctl` geïnstalleerd via `curl -sL https://talos.dev/install | sh`
  → geïnstalleerd in `/usr/local/bin/talosctl`
- Versie: v1.13.9 — matcht exact met de Talos-node-versie (belangrijk om
  gelijk te houden)

## Config-opslag (GitHub repo)

- **Leernotitie — secrets-hygiëne**: `controlplane.yaml`/`worker.yaml` bevatten
  de root CA private key (`machine.ca.key`) en het cluster join-token
  (`machine.token`) in plaintext. Nooit plakken in publieke tools/chats/tickets
  zonder reden, en nooit naar git. Bij twijfel of secrets ergens beland
  zijn (chat-logs, tickets): goedkoop om `talosctl gen config` opnieuw te
  draaien vóórdat het cluster gebootstrapt is — na bootstrap is roteren
  duurder


- **Repo-herstructurering (2026-08-30)**: van losse `~/talos-cluster` naar
  een **umbrella-monorepo `constellation`** voor de hele fleet. Layout:
  `docs/` (deze wiki + from-scratch + roadmap + side quests),
  `talos/<cluster>/` (machine-config bootstrap, met `patches/` + README),
  later `clusters/` / `platform/` / `apps/` voor GitOps. Branch `master` →
  `main`. Zie `../README.md`. `starstuff`'s config staat nu in
  `talos/starstuff/`.
- Clusternaam: **`starstuff`** (Sagan-referentie — "we are made of starstuff")
- Hostname-schema: **elementen**, elementgroep codeert het nodetype
  (CNO = control plane, overgangsmetalen = compute, dichte metalen = storage,
  aardalkali = base-cluster `sol`, edelgassen = autonome managed clusters).
  Volledig in `from-scratch.md` → "Hostname-schema"
- `.gitignore` bevat: `secrets.yaml`, `controlplane*.yaml`, `worker*.yaml`,
  `talosconfig`, `kubeconfig`, `*.kubeconfig`, `*.secret.yaml`
  → `secrets.yaml` is het belangrijkste bestand: bevat de gedeelde
  cluster-CA die alle 3 CP-nodes met elkaar verbindt. Verlies = cluster
  opnieuw moeten bootstrappen. Lek = cluster volledig gecompromitteerd
- `patches/`-map aangemaakt voor node-specifieke, wél-veilige patches
  (hostname + IP per node) die we wél naar GitHub pushen
- Base-config gegenereerd met:
  ```
  talosctl gen config starstuff https://10.3.9.10:6443
  ```

## Proxmox Backup Server (PBS)

- PBS-storage toegevoegd aan pve-dl320-1 (Datacenter → Storage → Add → PBS)
- Proxmox storage-ID: **`pbs-ugreen`**
- Server: `10.3.2.14`
- Datastore: `HDD`
- Namespace: `pve-dl320` (per-host namespace, logisch voor straks 3 hosts)
- Content: backup
- Auth: `root@pam!pve-dl320` API-token (niet het root-wachtwoord zelf)
- Permissions op PBS-server (Access Control → Permissions):
  `/datastore/HDD` → token `root@pam!pve-dl320` → rollen `DatastoreBackup`
  + `DatastoreReader`, propagate: yes
- Backup-job aangemaakt (Datacenter → Backup → Add):
  - Storage: **`pbs-ugreen`**
  - Schedule: dagelijks 3:00
  - Mode: Snapshot, compressie: ZSTD (fast and good), encryptie: aan
  - Guest: VM 100 (`controlplane1` — Proxmox VM-naam, los van Talos-hostname)
  - Notificatie: sendmail (legacy) naar `koen@terstal.it`, always
- Eerste testrun succesvol: 32GB VM, sparse/thin data, ~10 s, ~1MB archief
- Latere run (2026-08-30): incrementeel, 97% zero data, 36 s, 910 MiB/s.
  EFI-disk-warning verholpen door de BIOS/EFI-disk-fix — run toont nu
  `include disk 'efidisk0' ...`, bevestigd correct
- ⚠️ **Backup-log meldt** `skipping guest filesystem freeze - agent
  configured but not running?` → de **QEMU guest-agent-extensie draait niet**
  in het Talos-image (staat niet in `talosctl services`). Backups zijn nu
  crash-consistent i.p.v. filesystem-consistent. Fix: image via Image Factory
  herbouwen mét `siderolabs/qemu-guest-agent` en `talosctl upgrade`. Zie
  roadmap.md.
- Nog te doen: PBS-storage + backup-job herhalen op host 2 en 3, elk met
  eigen namespace (bv. `pve-dl320-2`, `pve-dl320-3`)

## Control-plane node 1 — `carbon` (gebootstrapt als `controlplane1`)

- Proxmox host: pve-dl320-1, VMID 100
- Status: Talos v1.13.9, **geïnstalleerd op /dev/sda, gebootstrapt en gezond**
  (single-node control plane).
- Hostname: **hernoemd naar `carbon`** (2026-08-30) volgens het elementen-schema
  (zie `from-scratch.md` → "Hostname-schema"). Node `carbon` is `Ready`; oude
  Node `controlplane1` verwijderd.
- IP: 10.3.9.10/24 (via Unifi fixed-IP-reservering op MAC-adres, bevestigd
  actief in console)
- Secure Boot: uit (False) — bevestigd correct

### Bootstrap voltooid (2026-08-29)

- `talosctl apply-config --insecure` → node uit maintenance mode, install op
  `/dev/sda`, reboot naar secure API (mTLS via `talosconfig`)
- `talosctl bootstrap` (eenmalig) → etcd `Preparing` → `Running`
- `talosctl health --server=false` → **alle checks OK**: etcd gezond,
  kube-apiserver/controller-manager/scheduler static pods Running, node Ready
  + schedulable, kube-proxy + coredns ready. Flannel CNI draait (default, niet
  uitgezet in config).
- etcd member: `controlplane1` (4e389bc734de4205), geen learner
- kubeconfig opgehaald + gemerged in `~/.kube/config`, context `admin@starstuff`
  (lokale kopie `~/talos-cluster/kubeconfig` — nu in `.gitignore`)
- `kubectl` geïnstalleerd via `kubernetes1.36-client` (Fedora) — matcht
  k8s-server v1.36.3. Node Ready, kube-system pods Running.
- talosconfig-context: endpoint + node op 10.3.9.10 gezet

### CP1-rename + bootstrap-lessen (2026-08-30)

Bij het opnieuw doorlopen van §6–§7 (rename naar `carbon`) sloeg `bootstrap`
aan op een **stale gecachte CA** → `x509: certificate signed by unknown
authority`. Opgelost met expliciete `--talosconfig`/`-e`/`-n`-flags; oude
`starstuff`-context verwijderd (nieuwe heet `starstuff-1`). `talosctl health`
liep vast op een timeout die géén fout was — `kubectl get nodes` bevestigde
`carbon` Ready. Volledig: `session-2026-08-30-cp1-bootstrap.md`. Gotchas
staan ook in `../AGENTS.md`.

### Opgeloste boot-issue

- **Symptoom**: VM kwam niet voorbij UEFI boot-manager,
  `Boot0002 UEFI QEMU DVD-ROM ... Access Denied`
- **Oorzaak**: EFI-disk was aangemaakt met `pre-enrolled-keys=1` (Secure Boot
  met Microsoft-keys aan) — Talos-ISO is niet met die keys gesigneerd
- **Fix**:
  1. EFI-disk verwijderd en opnieuw toegevoegd zonder pre-enrolled keys
  2. BIOS teruggezet naar `OVMF (UEFI)` (was tijdelijk op SeaBIOS gezet als
     verkeerde work-around)
  3. SCSI-controller teruggezet naar gewoon `VirtIO SCSI` (niet "single" —
     bekende oorzaak van bootstrap-hangs volgens Sidero-docs)
- **Geverifieerde eindstaat hardware**: BIOS OVMF (UEFI), machine q35, SCSI
  VirtIO SCSI, EFI disk zonder pre-enrolled keys, netwerk VirtIO op VLAN 9

### Hostname configureren — geleerde les

- **Talos 1.12+ heeft hostname verplaatst** van `machine.network.hostname`
  (v1alpha1) naar een apart `HostnameConfig`-document. `talosctl gen config`
  zet dat document (met `auto: stable`) automatisch onderaan
  `controlplane.yaml`/`worker.yaml`.
- `machine.network.hostname` én dat `HostnameConfig`-document samen →
  `static hostname is already set in v1alpha1 config`. Zie de opgeloste
  blocker bovenaan deze wiki.
- `HostnameConfig` heeft `auto` (`stable`/`off`) en `hostname` als
  **wederzijds uitsluitende** velden. Een strategic-merge patch
  (`talosctl machineconfig patch` / `--config-patch`) verwijdert de
  gegenereerde `auto: stable` NIET
  ([talos#12573](https://github.com/siderolabs/talos/issues/12573)) — je
  krijgt dan een ongeldig document met beide velden.
- **Werkende methode voor een vaste hostname per node**: na
  `talosctl gen config` het `HostnameConfig`-document in de output direct
  bewerken:
  ```yaml
  apiVersion: v1alpha1
  kind: HostnameConfig
  hostname: carbon   # was: auto: stable
  ```
  `patches/carbon.yaml` bewaart de gewenste eindstaat als referentie.
- `--config-patch` voor `machine.network.hostname` gaf
  `error: static hostname is already set in v1alpha1 config` (zelfde oorzaak)
- **Belangrijkere correctie**: officiële Sidero-productiedocs
  (docs.siderolabs.com/talos/v1.13/getting-started/prodnotes) voorschrijven
  een **gedeelde secrets-bundle** voor multi-node clusters, en het officiële
  `talosctl machineconfig patch`-commando i.p.v. losse `sed`-bewerkingen:
  1. `talosctl gen secrets -o secrets.yaml` — eenmalig, gedeelde CA/tokens
     voor het hele cluster (**ook in `.gitignore`**, minstens zo gevoelig
     als `controlplane.yaml`)
  2. `talosctl gen config --with-secrets secrets.yaml starstuff https://10.3.9.10:6443`
  3. Patch-bestand per node in `patches/`
  4. `talosctl machineconfig patch controlplane.yaml --patch @patches/<naam>.yaml --output controlplane.yaml`
- **Waarom dit belangrijk is**: elke `talosctl gen config` zonder
  `--with-secrets` genereert een **nieuwe** CA. Bij 3 CP-nodes moeten die
  allemaal dezelfde CA delen om samen een etcd-quorum te vormen — vandaar nu
  de vaste `secrets.yaml` als eenmalige bron van waarheid
- **Les over secrets-hygiëne blijft staan**: gebruik gerichte `grep` i.p.v.
  `sed`-regelbereiken bij het verifiëren van wijzigingen in configs die
  secrets bevatten

### Volgende stap

- ✅ CP1 gebootstrapt en gezond, `kubectl` werkt (zie "Bootstrap voltooid")
- **CP1 hernoemen** `controlplane1` → `carbon` (config staat klaar):
  `talosctl apply-config --nodes 10.3.9.10 --file controlplane.yaml` →
  wachten tot Node `carbon` Ready → `kubectl delete node controlplane1`
- CP2/CP3 (`oxygen` / `nitrogen`): VM's bouwen (zelfde specs),
  `controlplane.yaml` per node met eigen `HostnameConfig`, zelfde
  `secrets.yaml`. `apply-config --insecure` per node; **geen** extra
  `bootstrap` — die nodes joinen het bestaande etcd-quorum automatisch
- PBS-storage + `talosctl etcd snapshot`-cron (zie `from-scratch.md` →
  "Backupstrategie" — VM-snapshots van Talos-nodes hebben weinig waarde)
- Worker-VM's plannen zodra storage-oplossing voor persistent volumes bekend is
- Zie `roadmap.md` voor de geplande always-on base-cluster `sol` op zuinige
  nodes (in eerdere notities "edge-cluster" / `voyager`)

## Troubleshooting-log: pve-dl320-1 dist-upgrade freeze

- Tijdens `apt dist-upgrade` (nieuwe kernel + dubbele initramfs/grub op oude
  SAS-HDD's) bevroor de host — bereikbaar via ping, geen GUI/login meer
- Diagnose: geen ECC/MCE-fouten, geen hung-task-meldingen, SMART OK op beide
  schijven → waarschijnlijk zware IO-wait, geen hardware-defect
- Na hard reset via iLO: dpkg was interrupted → opgelost met
  `dpkg --configure -a`
- Aanleiding om versneld over te stappen op SSD's (2x Samsung PM1633a 480GB
  SAS, refurbished, ~€50/stuk)
