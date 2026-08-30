# Side quest — e-ink statusscherm voor `starstuff`

**Status:** brainstorm / niet ingepland. Doel van dit document: het idee
vastleggen zodat het niet verdwijnt, met een voorkeursrichting.

## Idee

Een klein e-ink-schermpje (ESP32) aan de muur dat de gezondheid van het
cluster toont: nodes Ready, etcd-status, versies, aantal pods, actieve
alerts. Alleen **read-only** data. Optioneel ook via Home Assistant, zodat
het meteen historie + remote-toegang krijgt.

## Kernprincipe — nooit de cluster-API blootstellen

De ESP32 (en zeker niet "op internet deelbaar") praat **niet** rechtstreeks
met de Kubernetes- of Talos-API. Daartussen zit een klein **aggregator-
service in het cluster** dat één afgeleide, gesaneerde JSON publiceert. Zo
blijft alle auth binnen het cluster en lekt er geen token of node-detail
naar buiten.

```
 k8s API  ─┐
           ├─►  cluster-status (SA, read-only RBAC)  ──►  /status.json  ──┬──►  ESP32 (ESPHome), LAN
 Talos API ─┘   (scoped talosconfig secret, os:reader)                     └──►  Home Assistant (rest sensor)
```

`/status.json` bevat bijvoorbeeld:

```json
{
  "updated_at": "2026-08-30T12:00:00Z",
  "cluster": "starstuff",
  "k8s_version": "v1.36.3",
  "talos_version": "v1.13.9",
  "nodes_ready": "3/3",
  "nodes": [{"name": "carbon", "ready": true, "role": "control-plane"}],
  "etcd_healthy": true,
  "etcd_members": 3,
  "pods_running": 42,
  "alerts": []
}
```

## Componenten

### 1. `cluster-status` aggregator (in-cluster)

- Kleine service (Go/Python, ~100 regels) of zelfs een `CronJob` die elke
  minuut een `ConfigMap` vult die nginx serveert.
- **ServiceAccount** met strak RBAC: `get/list` op `nodes`, `pods`,
  `componentstatuses`; lezen van de `kube-system`-events. Niets schrijven.
- Talos-data (optioneel): een `talosconfig`-secret met rol **`os:reader`**
  (`talosctl config new --roles os:reader`), voor `etcd members`, service-
  health, versies. Als dit te veel gedoe is: k8s-API alleen is genoeg voor
  een eerste versie.
- Exposen als `ClusterIP` (voor LAN via de router/reverse-proxy) — geen
  auth op LAN, of een statische bearer-token als je dat wilt.

### 2. Scherm — ESP32 + e-ink

Voorkeur: iets dat **ESPHome** goed ondersteunt, dan is de firmware een YAML.

| optie | scherm | opmerking |
|---|---|---|
| Waveshare 7.5" e-Paper v2 + ESP32 driverboard | 800×480 mono | Beste ESPHome-support (`waveshare_epaper`), goedkoop |
| LilyGo T5 4.7" | 960×540 grayscale | Alles-in-één, USB-C, accu-connector; ESPHome deels (soms PlatformIO) |
| M5Paper / M5Stack Paper S3 | 4.7" 16-grey | Nette behuizing, ESP32-S3; ESPHome-support wisselend |

Firmware-aanpak:
- **ESPHome** met `http_request` → `/status.json`, `json` parsen, `display`-
  `lambda` voor de layout. `deep_sleep` tussen updates (elke 2–5 min) —
  e-ink + deep-sleep = jaren op een accu.
- Of Arduino/PlatformIO als het scherm ESPHome niet trekt.

### 3. Home Assistant (optioneel maar aantrekkelijk)

- HA `rest`-sensor leest dezelfde `/status.json` → entities + historie +
  alerting ("carbon NotReady > 5 min").
- ESP32 kan dan i.p.v. het endpoint **HA** bevragen (native API), of HA
  rendert een dashboard.
- **Remote/deelbaar:** via Nabu Casa of een authenticerende reverse-proxy
  (Cloudflare Tunnel + Access). Exposeer alleen de HA-view of de status-
  JSON — nooit de k8s/Talos-API.
- Trade-off: koppelt het scherm aan HA-uptime. Voor een *status*-scherm
  acceptabel (HA is doorgaans beschikbaarder dan wat het monitort).

## Voorkeursrichting

1. Begin met **`cluster-status` aggregator, k8s-API-only**, `/status.json`
   op een ClusterIP.
2. **HA `rest`-sensor** erop → historie + alerting + remote-view "gratis".
3. **Waveshare 7.5" + ESP32 in ESPHome**, pollt `/status.json` (of HA)
   elke 5 min, deep-sleep ertussen.
4. Later: Talos-data toevoegen via een `os:reader`-talosconfig-secret.

## Hoe past dit in de git-flow?

Een brainstorm zoals dit doorloopt drie stadia:

| stadium | waar | vorm |
|---|---|---|
| **idee** | `docs/roadmap.md` (1 regel) + dit doc in `docs/` | direct op `main` committen — alleen tekst, geen branch nodig |
| **uitwerking** | GitHub Issue in `constellation`, label `side-quest`, onder de `sol`-milestone | discussie, checklist, linkt naar de PR('s) |
| **bouwen** | feature branch(es) | `feat/cluster-status-service` voor het in-cluster deel (map `platform/cluster-status/`); de **ESP32-firmware in een aparte repo** (`constellation-eink-display`) — firmware hoort niet in de fleet-repo |

Concreet nu: dit document + een pointer in `docs/roadmap.md`, verder niets.
Het in-cluster `cluster-status`-manifest komt t.z.t. in `platform/` (Gitea,
zie ROADMAP §3), niet los hierin.
