# Credits

`constellation` is een homelab-project van **Koen Terstal** (`rokoter` op
GitHub), opgebouwd samen met **Claude** — Claude Code, model Sonnet 5 van
Anthropic. Deze pagina legt transparant vast hoe het werk verdeeld was, mede
als proof-of-concept van AI-geassisteerde infrastructuur-ontwikkeling.

**In één zin:** Koen bepaalt richting en beslissingen en voert alles uit wat
fysiek of onomkeerbaar is; Claude doet uitzoekwerk, schrijft documentatie en
structuur, en stelt commando's en opties voor.

**Koens inbreng.** Doel, scope en filosofie — less is more, anti-e-waste,
thematische naamgeving. Hardware- en netwerkkeuzes. Alle beslissingen:
monorepo, clusternaam, hostname-schema, en de fleet-rolverdeling met een
geplande always-on base-cluster (`sol`) als hub. Alle state-wijzigende
acties: VM's bouwen,
`talosctl apply-config` en `bootstrap`, de GitHub-push, de issues. En de
testinput met terugkoppeling naar volgende sessies.

**Waar Claude een stap maakte.** De hostname-blocker gediagnosticeerd —
oorzaak in Talos 1.12+ en de fix, met bronnen. De repo geherstructureerd naar
de monorepo-layout. Alle documentatie geschreven: runbook, roadmap, wiki,
`AGENTS.md`, de side-quest-ontwerpen en de sessie-post-mortems. De
fleet-architectuur uitgewerkt — tiers voor de base-cluster (`sol`), backup-strategie,
recovery-netwerk. CI, `.gitignore`-hardening en de GitHub-issues opgesteld.

**Transparantie.** Vrijwel alle tekst in deze repo is door Claude opgesteld
op basis van Koens input en daarna door Koen beoordeeld. AI-opgestelde
commits dragen een `Co-Authored-By: Claude Sonnet 5`- en een
`Claude-Session`-regel.
