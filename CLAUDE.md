# CLAUDE.md

Project onboarding voor AI-agents staat in **[`AGENTS.md`](./AGENTS.md)** —
model-agnostisch, zodat ook een lokaal LLM (aider/opencode/continue/…) ermee
kan starten. Lees dat eerst, daarna `docs/`.

Claude-Code-specifiek:
- State-wijzigende `talosctl`-commando's (`apply-config`, `bootstrap`,
  `upgrade`) draait de gebruiker zelf — auto-mode blokkeert die. Read-only
  `talosctl get …` / `health` mag Claude wel.
