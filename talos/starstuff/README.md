# talos/starstuff — machine-config bootstrap

Pre-GitOps Talos-config voor cluster **`starstuff`** (3× HP DL320 G8 op
Proxmox). Volledige stap-voor-stap: [`../../docs/from-scratch.md`](../../docs/from-scratch.md).

> `starstuff` is de **bootstrap/genesis-cluster** van de fleet: hieruit wordt
> straks de always-on base-cluster `sol` gebouwd (config komt in `talos/sol/`,
> nog niet aangemaakt). Zie
> [`../../docs/roadmap.md`](../../docs/roadmap.md) → "Fleet-model".

## Bestanden

| bestand | in Git? | inhoud |
|---|---|---|
| `patches/carbon.yaml` | ✅ tracked | referentie-eindstaat `HostnameConfig` voor CP1 (`carbon`) |
| `patches/oxygen.yaml`, `nitrogen.yaml` | ✅ (nog aan te maken) | idem CP2/CP3 |
| `secrets.yaml` | ❌ gitignored | gedeelde cluster-CA + tokens — enige bron van waarheid |
| `controlplane.yaml` / `worker.yaml` | ❌ gitignored | volledige machine-config, bevat CA private keys |
| `talosconfig` / `kubeconfig` | ❌ gitignored | client-credentials |

## Config (her)genereren

```bash
cd talos/starstuff

# Eenmalig — gedeelde secrets voor het hele cluster
talosctl gen secrets -o secrets.yaml

# Basis-configs met die secrets
talosctl gen config --with-secrets secrets.yaml starstuff https://10.3.9.10:6443

# HostnameConfig-document per node: `auto: stable` -> `hostname: <element>`
#   CP1 = carbon, CP2 = oxygen, CP3 = nitrogen   (zie patches/ voor de eindstaat)

talosctl validate --config controlplane.yaml --mode metal
```

> Talos 1.13+: hostname hoort in het losse `HostnameConfig`-document, niet in
> `machine.network.hostname`. Zie `../../docs/wiki.md` → "Hostname configureren".

## Toepassen

```bash
cd talos/starstuff
talosctl apply-config --insecure --nodes 10.3.9.10 --file controlplane.yaml
# CP1 alleen, eenmalig:
talosctl bootstrap
```
