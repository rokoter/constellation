# Sessie 2026-08-30 — CP1 (`carbon`) bootstrap, lessen

Handmatig doorlopen van `talos/starstuff/README.md` §6–§7. CP1 draait nu als
`carbon`, `Ready`, control-plane, k8s v1.36.3, cluster `starstuff`.

## Wat misging (en de les)

### `bootstrap` faalde: `x509: certificate signed by unknown authority`

Stap 3 van §7 (`talosctl config merge` + `endpoint` + `node`) was
overgeslagen. `bootstrap` pakte een **oude, gecachte `talosconfig`/CA** uit
`~/.talos/config` van een eerdere poging (andere CA).

- **Fix:** `talosctl bootstrap --talosconfig ./talosconfig -e 10.3.9.10 -n 10.3.9.10`
  (expliciete flags, stale default-context omzeild) → geslaagd.
- **Les:** §6–§7 **in exacte volgorde**. Na elke `gen secrets`/`gen config`:
  eerst `talosctl config contexts` checken op een verouderde context met
  dezelfde clusternaam vóór `bootstrap`.

### Stale/dubbele talosctl-contexts

`talosctl config merge` hernoemde de nieuwe context stil naar `starstuff-1`
omdat er al een (ongeldige) `starstuff`-context bestond.

- **Fix:** na bevestiging dat `starstuff-1` werkt: `talosctl config remove starstuff`.
- **Les:** vóór een nieuwe `gen secrets`-cyclus oude contexts met dezelfde
  clusternaam actief verwijderen i.p.v. laten ophopen.

### `talosctl health` liep vast op "waiting for all k8s nodes to report" → `context canceled`

Geen echte fout — de kubelet was nog bezig zich te registreren. `talosctl
kubeconfig --force` + `kubectl get nodes` bevestigde binnen ~40 s dat de node
Ready werd.

- **Les:** bij deze timeout niet meteen debuggen — eerst `kubectl get nodes`.

## Werkwijze-tips die hieruit volgen

- Multi-stap CLI-bootstrap: na **élke** stap een korte statuscheck
  (`talosctl config contexts`, `talosctl get hostnamestatus`) i.p.v.
  doorrennen.
- Zie ook `AGENTS.md` → "Definition of done" (vanaf nu harde regel).

## Openstaand

- `starstuff-1`-context evt. hernoemen naar iets zonder `-1`-suffix
  (`talosctl config rename` indien beschikbaar).
