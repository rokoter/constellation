# Talos-op-Proxmox — van scratch tot werkende bootstrap-laag

Reproduceerbare runbook: van "Proxmox VE net geïnstalleerd" tot de huidige
staat (1 werkende Talos control-plane node, cluster `starstuff`).

Voor **achtergrond, rationale en troubleshooting-historie** zie
[`wiki.md`](./wiki.md). Dit bestand is de kale checklist; de wiki legt uit
*waarom*. Repo-layout: zie [`../README.md`](../README.md).

> Machine-config van `starstuff` staat in `talos/starstuff/`. Draai de
> `talosctl`-commando's hieronder vanuit die map (`cd talos/starstuff`), of
> geef het volledige pad mee met `--file`.

- **Talos**: v1.13.9 (node én `talosctl`-client — gelijk houden)
- **Kubernetes**: v1.36.3 (komt mee met Talos 1.13.9)
- **Clusternaam**: `starstuff`
- **VLAN 9** — `10.3.9.0/24`, gateway `10.3.9.254`
- **CP1**: `10.3.9.10`, Proxmox-host `pve-dl320-1`, VMID 100

> Hostname-schema: **elementen** (zie "Hostname-schema" onderaan). Control
> plane = CNO: `carbon` (CP1), `oxygen` (CP2), `nitrogen` (CP3).

---

## 0. Vereisten

- 3× HP ProLiant DL320 G8 v2, elk met Proxmox VE 9.2 bare-metal geïnstalleerd
- Unifi-netwerk met VLAN 9 beschikbaar, gateway `10.3.9.254`
- Fedora-werkstation met internettoegang
- (optioneel) Proxmox Backup Server op `10.3.2.14`

---

## 1. Proxmox-host voorbereiden (per host)

1. **No-subscription repo** (of enterprise zonder subscription — check faalt,
   dat is verwacht).
2. **Systeem bijwerken**:
   ```bash
   apt update && apt dist-upgrade
   ```
   > ⚠️ Op de trage SAS-HDD's kan `dist-upgrade` (nieuwe kernel + dubbele
   > initramfs/grub-generatie) de host laten bevriezen door IO-wait. Host
   > blijft pingbaar, GUI/login weg. Oplossing: hard reset via iLO, daarna:
   > ```bash
   > dpkg --configure -a
   > ```
   > Reden om versneld naar SSD's te gaan (2× Samsung PM1633a 480GB SAS,
   > ZFS mirror).
3. **Reboot** in de nieuwe kernel.

---

## 2. Proxmox Backup Server koppelen (per host, optioneel)

Datacenter → Storage → Add → Proxmox Backup Server:

| veld | waarde |
|---|---|
| Server | `10.3.2.14` |
| Datastore | `HDD` |
| Namespace | `pve-dl320` (per host: `pve-dl320-2`, `-3`) |
| Content | backup |
| Auth | API-token `root@pam!pve-dl320` (niet het root-wachtwoord) |

Op de PBS-server (Access Control → Permissions):
`/datastore/HDD` → token `root@pam!pve-dl320` → rollen `DatastoreBackup` +
`DatastoreReader`, propagate: yes.

> **Zin van VM-backups van Talos-nodes?** Beperkt — zie "Backupstrategie"
> onderaan. De echte DR-artefact is een `talosctl etcd snapshot`, geen
> VM-snapshot.

---

## 3. Werkstation-tooling (Fedora)

```bash
# talosctl — pin op de node-versie
curl -sL https://talos.dev/install | sh          # installeert in /usr/local/bin
talosctl version --client                        # moet v1.13.9 zijn

# kubectl — moet binnen ±1 minor van de k8s-server (v1.36.3) vallen
sudo dnf install kubernetes1.36-client
kubectl version --client                         # v1.36.x
```

> Fedora 44 heeft `kubernetes1.33/1.34/1.35/1.36-client`. Pak altijd de
> versie die matcht met wat Talos draait (nu 1.36). Bij een Talos-upgrade
> die k8s meebumpt: ook kubectl mee-swappen met `sudo dnf swap`.

---

## 4. Talos-image bouwen (Image Factory)

1. Ga naar <https://factory.talos.dev>, kies **Bare-metal**, Talos **v1.13.9**,
   arch amd64.
2. System extensions: **alleen** `siderolabs/qemu-guest-agent`.
   (zfs-/iscsi-tools/nfs-utils pas toevoegen zodra de storage-oplossing voor
   persistent volumes bekend is.)
3. Noteer de **schematic ID** en download de **ISO** (`metal-amd64.iso`).
4. Upload de ISO naar Proxmox: elke host → local storage → ISO Images →
   Upload (of `wget` in `/var/lib/vz/template/iso/`).

> Schematic ID hier invullen zodra bekend: `________________________`

---

## 5. Control-plane VM aanmaken (per node)

Op de Proxmox-host, VM met **exact** deze instellingen:

| instelling | waarde | let op |
|---|---|---|
| VMID / naam | 100 / `carbon` | `oxygen`=101, `nitrogen`=102 op resp. host 2/3 |
| Machine type | `q35` | |
| BIOS | `OVMF (UEFI)` + EFI-disk | **EFI-disk zonder pre-enrolled keys** — anders `Access Denied` bij boot (Talos-ISO is niet met MS-keys gesigneerd) |
| Secure Boot | uit | |
| SCSI-controller | `VirtIO SCSI` | **niet** "VirtIO SCSI single" — geeft bootstrap-hangs |
| Disk | 32 GB | |
| vCPU | 2, type `host` | |
| RAM | 4096 MB | |
| Netwerk | VirtIO, bridge op **VLAN 9** | |
| QEMU Guest Agent | aan | extensie zit in de custom ISO |
| CD/DVD | de Talos-ISO | |

### DHCP-reservering (Unifi)

- Reserveer per node een vast IP **op MAC-adres** (Talos stuurt in
  maintenance-mode geen DHCP-hostname mee).
- CP1 → `10.3.9.10`, CP2 → `.11`, CP3 → `.12`.

### Booten

1. Start de VM, boot van de ISO.
2. Wacht tot de console **"maintenance mode"** toont en een IP heeft.
3. Verifieer:
   ```bash
   ping -c2 10.3.9.10
   talosctl get disks --insecure --nodes 10.3.9.10        # sda ~34 GB zichtbaar
   talosctl get systemdisk --insecure --nodes 10.3.9.10   # LEEG = nog niks op disk
   ```

---

## 6. Machine-config genereren

In `talos/starstuff/` (in de repo):

```bash
cd talos/starstuff

# Eenmalig — gedeelde cluster-CA/tokens voor ALLE nodes. NOOIT committen.
talosctl gen secrets -o secrets.yaml

# Basis-configs met die gedeelde secrets
talosctl gen config --with-secrets secrets.yaml \
  starstuff https://10.3.9.10:6443
# -> controlplane.yaml, worker.yaml, talosconfig
```

> **Waarom `--with-secrets`?** Elke `gen config` zónder genereert een nieuwe
> CA. De 3 CP-nodes moeten dezelfde CA delen om samen een etcd-quorum te
> vormen. `secrets.yaml` is de enige bron van waarheid — verlies = cluster
> opnieuw bootstrappen, lek = cluster gecompromitteerd.

### Hostname zetten (Talos 1.12+)

`talosctl gen config` zet onderaan `controlplane.yaml` een los
`HostnameConfig`-document met `auto: stable`. **Bewerk dat direct**:

```yaml
---
apiVersion: v1alpha1
kind: HostnameConfig
hostname: carbon             # was: auto: stable  (CP2 -> oxygen, CP3 -> nitrogen)
```

> `machine.network.hostname` (v1alpha1) **niet** gebruiken — dat naast het
> `HostnameConfig`-document geeft `static hostname is already set in v1alpha1
> config`. `auto` en `hostname` zijn wederzijds uitsluitend, en een
> strategic-merge patch verwijdert `auto: stable` niet
> ([talos#12573](https://github.com/siderolabs/talos/issues/12573)). Dus:
> handmatig editen in de output. `patches/carbon.yaml` bewaart de gewenste
> eindstaat als referentie (idem `patches/oxygen.yaml`, `nitrogen.yaml`).

### Valideren

```bash
talosctl validate --config controlplane.yaml --mode metal
# -> "controlplane.yaml is valid for metal mode"
```

---

## 7. Toepassen & bootstrappen (alleen CP1)

```bash
cd talos/starstuff

# 1. Config toepassen — node gaat uit maintenance mode, installeert op
#    /dev/sda, reboot naar de beveiligde API (mTLS).
talosctl apply-config --insecure --nodes 10.3.9.10 --file controlplane.yaml

# 2. Wacht tot de beveiligde API terug is (~1-2 min)
until talosctl --talosconfig ./talosconfig -e 10.3.9.10 -n 10.3.9.10 \
      version >/dev/null 2>&1; do sleep 5; done

# 3. talosconfig als default zetten
talosctl config merge ./talosconfig
talosctl config endpoint 10.3.9.10
talosctl config node 10.3.9.10

# 4. Verifieer pre-bootstrap staat
talosctl get systemdisk          # DISK = sda  -> geïnstalleerd
talosctl get hostnamestatus      # HOSTNAME = carbon
talosctl services                # etcd = Preparing ("Running pre state")

# 5. Bootstrap — EENMALIG, ALLEEN op CP1. Nooit opnieuw, nooit op CP2/CP3.
talosctl bootstrap

# 6. Wachten tot alles gezond is
talosctl health --server=false   # exit 0, alle checks OK

# 7. kubeconfig ophalen (merge in ~/.kube/config)
talosctl kubeconfig --force
kubectl config use-context admin@starstuff
kubectl get nodes -o wide        # carbon  Ready  control-plane  v1.36.3
```

Verwacht na `bootstrap`:
- `etcd` `Preparing` → `Running`
- kube-apiserver / controller-manager / scheduler als static pods (Ready)
- Flannel CNI deployt automatisch (niet uitgezet in de config)
- coredns (2×), kube-proxy Running
- controller-manager/scheduler tonen kort `RESTARTS 2-3` tijdens bootstrap —
  normaal, stabiliseert daarna

---

## 8. Huidige staat (bijgewerkt 2026-08-30)

- **CP1 = `carbon`** — Talos v1.13.9 op `/dev/sda`, gebootstrapt, `Ready`,
  control-plane. Single-node control plane.
- etcd: 1 member, geen learner.
- Kubernetes v1.36.3, node Ready + schedulable, Flannel CNI.
- `~/.kube/config` context `admin@starstuff`; actieve talosctl-context
  `starstuff-1` (oude `starstuff`-context verwijderd — zie
  `session-2026-08-30-cp1-bootstrap.md`).
- Bootstrap-lessen: `session-2026-08-30-cp1-bootstrap.md`.

---

## 9. Volgende stappen

### CP2 en CP3 toevoegen

1. VM's bouwen op host 2/3 (§5), DHCP-reservering `.11` (`oxygen`) /
   `.12` (`nitrogen`).
2. `controlplane.yaml` per node — **zelfde `secrets.yaml`**, alleen het
   `HostnameConfig`-document aanpassen (`hostname: oxygen` / `nitrogen`).
   Endpoint in de config blijft `https://10.3.9.10:6443` (of later een VIP).
3. `talosctl apply-config --insecure --nodes 10.3.9.11 --file controlplane-oxygen.yaml`
   (idem `.12` / `nitrogen`).
4. **Geen** `talosctl bootstrap` — deze nodes joinen het bestaande
   etcd-quorum automatisch.
5. Controle: `talosctl -n 10.3.9.10 etcd members` (3 members),
   `kubectl get nodes` (3× Ready).

> Overweeg daarna een control-plane VIP (`machine.network.interfaces[].vip`)
> zodat de API niet aan CP1 hangt.

### Overig

- PBS-storage + `talosctl etcd snapshot`-cron op host 2/3 (§Backupstrategie).
- Worker-VM's — pas plannen zodra de persistent-storage-oplossing bekend is.
- Firewall-regel Unifi: alleen management-VLAN → VLAN 9 op 6443, 50000, 50001.

---

## Backupstrategie — heeft een PBS-backup van Talos-nodes zin?

**Kort: VM-snapshots van Talos-nodes hebben weinig waarde. Maak
`talosctl etcd snapshot`s.**

- Een Talos-node is volledig herbouwbaar uit `controlplane.yaml` +
  `secrets.yaml` (minuten werk). Een VM-image toevoegt daar weinig aan.
- Een oude VM-snapshot van één etcd-member terugzetten in een draaiend
  cluster is juist **gevaarlijk** (stale member, revisie-mismatch). Sidero
  raadt dit expliciet af.
- De ondersteunde DR-weg is een consistente etcd-snapshot:
  ```bash
  talosctl -n 10.3.9.10 etcd snapshot etcd-$(date +%F).snapshot
  ```
  Restore via `talosctl bootstrap --recover-from=<snapshot>`.
- **Nu (single-node CP):** etcd-snapshots zijn essentieel — geen redundantie.
  Zet een cron op het werkstation of een externe host, bewaar de snapshots
  buiten het cluster.
- **Straks (3 CP-nodes):** etcd is HA. Eén node kwijt = herbouwen uit config,
  geen restore nodig. Snapshots blijven nuttig voor "hele cluster kwijt" /
  logische fouten (per ongeluk namespace verwijderd).
- **Stateful workloads** (Gitea, monitoring-DB's): back-uppen via hun eigen
  mechanisme of via CSI-volume-snapshots zodra er persistent storage is —
  niet via VM-snapshots.

Praktisch advies:
- VM-backup van de Talos-guests: hooguit een wekelijkse, of overslaan.
- Wél: dagelijkse `talosctl etcd snapshot` naar een externe locatie.
- PBS-backupjobs bewaren voor Proxmox-host-config en toekomstige
  stateful-VM's.

---

## Hostname-schema — elementen (VASTGESTELD)

De letterlijke invulling van "starstuff": elementen die in sterren gesmeed
zijn. De **elementgroep codeert het soort node** — de exacte rol staat
daarnaast als k8s-label (`node-class=…`) en Talos `machine.type`.

| soort node | elementgroep | voorbeelden | metafoor |
|---|---|---|---|
| control plane | CNO (leven) | `carbon` (CP1), `oxygen` (CP2), `nitrogen` (CP3) | de kern / het brein |
| general compute worker | overgangsmetalen | `iron`, `nickel`, `cobalt`, `titanium`, `chromium`, `vanadium` | de werkpaarden — sterk, structureel |
| storage worker | dichte / zware metalen | `tungsten`, `osmium`, `iridium`, `lead`, `gold`, `platinum` | dicht = houdt veel vast |
| GPU / accelerator worker | halfgeleiders | `silicon`, `germanium`, `gallium`, `arsenic` | halfgeleiders = compute-versnelling |
| edge / standalone cluster | edelgassen | `helium`, `neon`, `argon`, `krypton`, `xenon` | inert, stabiel, self-contained — draait alleen door (zie ROADMAP) |
| ephemeral / burst (optioneel) | alkalimetalen | `lithium`, `sodium`, `potassium` | reactief, kortlevend |

Waarom dit schema:
- Letterlijke band met `starstuff`.
- Elementgroep = zelf-documenterend nodetype, zonder rol in de hostname te
  hardcoden (rol = label + `machine.type`, dat blijft de bron van waarheid).
- Periodiek systeem = ~118 namen, nooit op.
- Kort, lowercase, DNS-safe, makkelijk te typen.

Bij elke node ook een label zetten, bv.:
```bash
kubectl label node carbon node-class=control-plane
kubectl label node iron   node-class=compute
kubectl label node tungsten node-class=storage
```
(of via `machine.nodeLabels` in de Talos-config, dan overleeft het reboots.)

### Overwogen alternatieven

- **Planeten vanaf de zon**: `mercury/venus/earth` (CP) /
  `jupiter/saturn/…` (workers). Ingebouwde volgorde, maar minder ruimte om
  nodetypes te coderen.
- **Sondes**: `voyager1`, `pioneer10`, `newhorizons` — nummers ingebouwd,
  maar ~5 bekende namen.
- **Sterren**: `betelgeuse`, `rigel`, `sirius` — onbeperkt, geen structuur.

### CP1 hernoemen (`controlplane1` → `carbon`)

CP1 is al toegepast als `controlplane1`. Hernoemen:

```bash
cd talos/starstuff
# 1. HostnameConfig in controlplane.yaml staat al op `hostname: carbon`
talosctl apply-config --nodes 10.3.9.10 --file controlplane.yaml

# 2. Talos zet de hostname live. De k8s Node-objectnaam volgt de hostname:
#    er verschijnt een nieuwe Node `carbon`, de oude blijft als NotReady staan.
kubectl get nodes                      # wacht tot `carbon` Ready is
kubectl delete node controlplane1      # oude opruimen
```

DHCP-reservering staat op MAC — die hoef je niet aan te passen. Doe dit
terwijl het nog een wegwerp-single-node is; met workloads erop is het duurder.
