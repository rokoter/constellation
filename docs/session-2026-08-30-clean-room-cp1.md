# Sessie 2026-08-30 — clean-room CP1 (`carbon`) bootstrap-validatie

> Losstaand van de eerste `starstuff`-bootstrap
> (`session-2026-08-30-cp1-bootstrap.md`). Doel: `docs/from-scratch.md`
> §6–§7 opnieuw doorlopen vanuit een **verse review-clone** op een **nieuw
> netwerk-schema**, en frictie in de runbook vinden.

**Wat getest / gedaan:** clean bootstrap CP1 vanaf verse clone
(`~/dev/review/constellation`) — Talos-config genereren, CP1 toepassen,
bootstrappen, verifiëren tot `Ready`.
**Omgeving:** Fedora-werkstation, `talosctl` v1.13.9, `kubectl` v1.36.3,
review-clone (niet de hoofd-werkmap), `~/.talos/config` **niet leeg** —
bevatte al `starstuff` (+ `-1`, `-2`) van eerdere runs.
**Netwerk deze run (wijkt af van de runbook):**

| wat | waarde |
|---|---|
| Talos-VLAN | **VLAN 4** |
| `carbon` / `oxygen` / `nitrogen` | `10.30.4.1` / `.2` / `.3` |
| Proxmox-mgmt `pve-dl320-1..3` | `10.30.3.1` / `.2` / `.3` |
| tijdelijke CP-endpoint | `https://10.30.4.1:6443` |
| gateway / prefix / DNS / VIP | **niet vastgelegd in deze run** |

**Talos-image:** Image Factory schematic
`ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515`
(bare-metal amd64, v1.13.9). Secure Boot gaf vooraf gedoe → teruggevallen op
dit image; Secure Boot voorlopig geparkeerd.
**Startpunt in de docs:** `docs/from-scratch.md` §6, met bovenstaande
netwerk-override.

## Verloop

Werkmap: `~/dev/review/constellation/talos/starstuff`.

- §6 `talosctl gen secrets -o secrets.yaml` — ✅
- §6 `talosctl gen config --with-secrets secrets.yaml starstuff https://10.30.4.1:6443`
  — ✅ `controlplane.yaml` / `worker.yaml` / `talosconfig` aangemaakt
- Handmatig geïnspecteerd/bewerkt: `nano talosconfig`, `nano controlplane.yaml`
  — ⚠️ exacte edits niet in het transcript; raadpleeg de werkboom-bestanden
- §7.1 `talosctl apply-config --insecure --nodes 10.30.4.1 --file controlplane.yaml`
  — `Applied configuration without a reboot` (node installeerde alsnog op `sda`)
- §7.2 wachtlus tot de mTLS-API terug is — ✅
- §7.3 `talosctl config merge ./talosconfig` —
  `renamed talosconfig context "starstuff" -> "starstuff-3"` ⚠️ (zie Problemen);
  daarna `config endpoint 10.30.4.1` + `config node 10.30.4.1`
- §7.4 pre-bootstrap checks — `systemdisk = sda`, `hostnamestatus = carbon`,
  `etcd = Preparing ("Running pre state")` — ✅ verwacht
- §7.5 `talosctl bootstrap` — ✅ (eenmalig)
- §7.6 `talosctl health --server=false` — **alle checks OK**, in één keer
  (geen "waiting for all k8s nodes to report"-timeout zoals de vorige sessie)
- §7.7 `talosctl kubeconfig --force` + `kubectl config use-context admin@starstuff`
  + `kubectl get nodes -o wide` — `carbon Ready control-plane v1.36.3`,
  Talos v1.13.9, kernel `6.18.44-talos`, containerd `2.2.7`

## Problemen

De CP1-bootstrap zélf verliep zonder harde fouten. De frictie zat eromheen.

### 1. Verouderde netwerkwaarden in de runbook

- **Symptoom:** `docs/from-scratch.md` hardcodeert overal VLAN 9 /
  `10.3.9.0/24` / gw `10.3.9.254` / CP1 `10.3.9.10` / `pve-dl320-1`
  (regels 17-18, 28, 123, 131, 139-141, 158, 203-212, 262-293, 361). Deze run
  draaide op VLAN 4 / `10.30.4.x`, Proxmox-mgmt `10.30.3.x`.
- **Oorzaak:** de runbook is geschreven rond de eerste bootstrap en gebruikt
  de concrete waarden inline i.p.v. als expliciet voorbeeld/parameter.
- **Fix:** callout bovenaan `from-scratch.md` dat de netwerkwaarden
  omgevings-specifiek zijn, met verwijzing naar dít sessiedoc voor de
  clean-room-waarden. Volledige parameterisering → issue.
- **Doc-gap:** `from-scratch.md` (intro, §0, §5, §7, §9).

### 2. Accumulerende / stille hernoeming van `talosctl`-contexts

- **Symptoom:** `talosctl config merge` →
  `renamed talosconfig context "starstuff" -> "starstuff-3"`.
- **Oorzaak:** `~/.talos/config` bevatte al `starstuff` + `-1` + `-2` van
  eerdere runs; `merge` hernoemt stil bij naam-botsing. Zelfde klasse frictie
  als de vorige sessie (`starstuff-1`) en de bestaande stale-CA-gotcha in
  `AGENTS.md`.
- **Fix:** een clean-room-run hoort te starten met een **lege
  `~/.talos/config`**; vooraf `talosctl config contexts` checken en oude
  `starstuff*`-contexts verwijderen (`talosctl config remove …`).
- **Doc-gap:** `from-scratch.md` §7 mist deze inline-waarschuwing;
  `AGENTS.md`-gotcha aanscherpen met de clean-room-eis.

### 3. Image Factory schematic-ID placeholder nog leeg

- **Symptoom:** `from-scratch.md` §4 heeft nog
  `Schematic ID hier invullen zodra bekend: ____`.
- **Oorzaak:** issue #1 — nooit ingevuld.
- **Fix:** ID van deze run:
  `ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515`.
  ⚠️ De extensie-set achter dit schematic is **niet expliciet in het
  transcript bevestigd** — verifiëren dat het het
  `siderolabs/qemu-guest-agent`-only image is (from-scratch §4 stap 2).
- **Doc-gap:** `from-scratch.md` §4 / issue #1.

### 4. Secure Boot-frictie, niet vastgelegd

- **Symptoom:** "substantial Secure Boot friction" vóór deze run; **geen
  foutregels** in het transcript.
- **Oorzaak:** onbekend (geen detail overgedragen). `from-scratch.md` §5 dekt
  al "EFI-disk zonder pre-enrolled keys" + "Secure Boot: uit".
- **Fix:** Secure Boot geparkeerd, teruggevallen op het niet-SB image.
- **Doc-gap:** geen concrete fix mogelijk zonder de foutregels → issue om het
  Secure Boot-pad óf te documenteren óf expliciet buiten scope te zetten.

### 5. (klein) `apply-config` meldt "without a reboot"

- **Symptoom:** `Applied configuration without a reboot`, terwijl §7 stap 1 in
  de doc "reboot naar de beveiligde API (mTLS)" beschrijft.
- **Oorzaak:** Talos meldt de maintenance→install-transitie niet altijd als
  reboot; de install gebeurt alsnog (systemdisk werd `sda`, API kwam terug op
  mTLS).
- **Fix:** doc-tekst nuanceren ("node verlaat maintenance mode en installeert
  op `/dev/sda`; de CLI-melding kan 'without a reboot' zijn").
- **Doc-gap:** `from-scratch.md` §7 stap 1 comment.

## Doc-gaps → issues

- [ ] **from-scratch.md: netwerkwaarden omgevings-specifiek maken** — VLAN 9 /
  `10.3.9.x` staat overal inline; clean-room-run gebruikte VLAN 4 /
  `10.30.4.x`. Parameteriseren of als expliciet voorbeeld markeren.
- [ ] **Secure Boot-pad documenteren of expliciet buiten scope** — frictie in
  deze run, geen details; §5 dekt alleen de EFI-disk zonder pre-enrolled keys.
- [ ] **from-scratch.md §7: `talosctl config merge` context-accumulatie** —
  documenteer de opruimstap vóór merge en de stille `-N`-suffix.
- [x] **Image Factory schematic-ID (issue #1)** — ingevuld in §4 met de ID van
  deze run; extensie-set nog te verifiëren.

## Lessen voor `AGENTS.md`

- Een clean-room / verse-machine-test moet starten met een **lege
  `~/.talos/config`**. Check `talosctl config contexts` vóór `gen secrets`;
  anders hernoemt `config merge` stil naar `starstuff-1/-2/-3` en kan
  `bootstrap` een oude CA pakken (bestaande stale-CA-gotcha).
- De concrete netwerkwaarden in `from-scratch.md` (VLAN 9 / `10.3.9.x`) zijn
  van de eerste bootstrap; latere runs kunnen een ander schema gebruiken (deze
  run: VLAN 4 / `10.30.4.x`, Proxmox `10.30.3.x`).
- `talosctl health --server=false` liep deze keer in één keer schoon door — de
  "waiting for all k8s nodes to report → context canceled"-timeout uit de
  vorige sessie kwam niet terug.

## Overdracht

CP1 online op het nieuwe netwerk-schema:

- hostname `carbon`, Talos-IP `10.30.4.1`, rol `control-plane`, k8s `Ready`
- systemdisk `sda`, Talos v1.13.9, Kubernetes v1.36.3, kernel `6.18.44-talos`,
  containerd `2.2.7`
- `talosctl health --server=false`: alle checks OK
- `bootstrap` is één keer gedraaid — **niet herhalen**
- cluster-generatie gebruikte de `secrets.yaml` van deze run
- tijdelijke control-plane endpoint `https://10.30.4.1:6443` (nog geen VIP)
- actieve `talosctl`-context: `starstuff-3`
- gegenereerde bestanden (in de review-clone, gitignored): `secrets.yaml`,
  `controlplane.yaml`, `worker.yaml`, `talosconfig`
- `talosconfig` + `controlplane.yaml` zijn **handmatig bewerkt** vóór apply —
  inspecteer de werkboom-bestanden, niet dit doc

**Handoff stopt hier bewust — geen CP2/CP3 in deze sessie.** Volgende werk:
verder vanaf deze CP1-online-staat.
