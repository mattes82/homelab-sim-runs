# S4 — Sandbox-Integritaet Gesamt · 2026-07-10

Drei integritaetskritische Sandbox-Infra-Fixe, damit „Gruen" wieder etwas bedeutet.
**Kein Prod-Repo mutiert.** Zwei PRs, kein Auto-Merge, Merge-Reihenfolge A→B.

| Befund | Fix | before→after | PR |
|---|---|---|---|
| REV-001 | make replay Exitcode | geschluckt (`-`-Prefix) → durchgereicht (trap) | PR-A #23 |
| REV-002 | verify-offline fail-closed | GRUEN@0-Pass → Pflichtsuite-gegated (leer/0-Pass=ROT) | PR-A #23 |
| REV-004 | Transform nur Injektion | continueRegularOutput → Prod-Fehlersemantik + Semantik-Hash-Gate GRUEN | PR-A #23 |
| REV-003 | Datenklassen-Kanon | C Hard-Block am Intake, A local_only (v1.0-Praemisse korrigiert) | PR-B #24 |

**Kein Regress** #20/#21 (Routing 8/8, Effekt-Matrix unveraendert). Isolation alle Prod-IPs Timeout. gitleaks 0. Evidence-Completeness (G10): Rohbelege je Stufe in `evidence.txt`.

## Quell-SHAs
tested-base: homelab-jarvis-sim@c7ee804; PR-A tip @8d1aab4; PR-B tip @27b2e5e. Prod-Quellen (Referenz): homelab-jarvis-workflows@e9470b4, homelab-docker@7853fbe. VM300-Staging traegt kein .git (Freshness-Gap).
