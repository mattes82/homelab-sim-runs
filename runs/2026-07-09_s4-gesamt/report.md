# Sim-Sandbox Run-Report — S4: Gesamtstand + Reset-/Mess-Mechanik

- **Datum:** 2026-07-09
- **Stufe:** S4 (von 4) — Abschluss
- **Zone:** VLAN35 / VM300 (192.168.35.60), Host `jarvis-sim` — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sandbox-staging-p1p2`
- **Status:** ✅ grün — DoD erfüllt

## Gesamt-Health-Matrix (echter HTTP-Pfad)

| Dienst | Container | Port | /health |
|---|---|---|---|
| Baïkal (CalDAV) | `sandbox-baikal` | 8800 | 200 (`GET /`) |
| caldav-reader | `sandbox-caldav-reader` | 8090 | 200 |
| nextcloud-task-writer | `sandbox-task-writer` | 8094 | 200 |
| caldav-writer | `sandbox-caldav-writer` | 8100 | 200 |
| action-state | `sandbox-action-state` | 8093 | 200 |
| jarvis-tools | `sandbox-jarvis-tools` | 8095 | 200 |
| Dovecot (IMAP) | `sandbox-dovecot` | 11993/11143 | imaps/imap OK |
| Mailpit (SMTP-Senke) | `sandbox-mailpit` | 8025 | 200 (API) |
| imap-write-adapter | `sandbox-imap-write` | 8110 | 200 |
| imap-backfill-reader | `sandbox-imap-reader` | 8111 | 200 |
| imap-backfill-reader-m | `sandbox-imap-reader-m` | 8112 | 200 |
| Sim-n8n | `jarvis-sim-n8n` | 5679 | 200 (healthz) |

## Referenz-E2E-Ergebnisse (S2 + S3)

| Flow | Ergebnis |
|---|---|
| `calendar.create` → Baïkal-Kalender | ✅ geschrieben + via caldav-reader rückgelesen |
| `nextcloud_task.create` → Baïkal-Tasks | ✅ 201 + rückgelesen |
| Undo `nextcloud_task`-Delete | ✅ vorhanden (kein 404-Drift), Task entfernt |
| `mail.move` → Whitelist-Ordner | ✅ Quelle −1 / Ziel +1 (in Dovecot verifiziert) |
| `mail.move` → Nicht-Whitelist | ✅ 400 abgelehnt |
| `mail.forward` → Mailpit | ✅ gefangen, **nie** extern versandt |
| Auth fail-closed (falscher Token) | ✅ 401/403 durchgängig |
| **Isolations-Recheck** (VM300→Prod) | ✅ alle `000`/Timeout nach jeder Stufe |

## Overhead-Messung (D-5) — empirische Grundlage für RAM-Entscheidungen

`docker stats --no-stream` (Ruhezustand, alle Container):

```
jarvis-sim-n8n          0.14%   280.4 MiB
sandbox-action-state    0.14%    39.0 MiB
sandbox-baikal          0.00%    26.7 MiB
sandbox-caldav-reader   0.13%    35.3 MiB
sandbox-caldav-writer   0.15%    42.3 MiB
sandbox-dovecot         0.00%    14.7 MiB
sandbox-imap-reader     0.12%    41.4 MiB
sandbox-imap-reader-m   0.14%    41.4 MiB
sandbox-imap-write      0.19%    34.9 MiB
sandbox-jarvis-tools    0.17%    38.5 MiB
sandbox-mailpit         0.00%     8.1 MiB
sandbox-task-writer     0.12%    36.8 MiB
```

- **Sandbox-Stack gesamt: ~639 MiB RAM** (12 Container), CPU im Ruhezustand ~0.
- `free -h`: total 7.8 GiB · used 1.4 GiB · **available 6.3 GiB** · **Swap 0**.
- `df -h /`: 40 GiB, 17 GiB benutzt, **22 GiB frei** (43 %).

**Bewertung:** Der komplette Staging-Stack kostet ~0,64 GiB — bequem innerhalb der 7,8 GiB.
Achtung Kontext: VM300 hat **kein Swap**; der OpenJarvis-Eval-Stack lädt bei aktivem Lauf
Multi-GB-Modelle. Für Parallelbetrieb Sandbox+Eval wäre ein RAM-Bump oder Swap die
empirisch begründete Vorsichtsmaßnahme (Sandbox selbst ist kein Treiber).

## Reset-/Seed-Mechanik (OP-S4.2) — Sekunden-Reset ohne Proxmox

- `make sim-seed` — idempotent (Baïkal-CalDAV + Dovecot-Mail). Mehrfach lauffähig.
- `make sim-sandbox-reset` — `down -v` (alle Tiers) + Daten verwerfen + seeden + hoch.
  **Gemessen: 9 s**, danach alle 12 Container up und alle /health = 200.
- Das bestehende `make sim-reset` (n8n-Routing-Sim, load-bearing für `sim-smoke`) bleibt
  **unverändert**; die Sandbox nutzt eigene, klar benannte Targets (`sim-sandbox-*`).
- `baseline-sim` (Proxmox-Snapshot) bleibt reines Disaster-Recovery (Mattes), NICHT der
  Pro-Lauf-Reset.

## Pfad-/Ziel-Mapping (Prod ↔ Sandbox)

| Domäne | Produktion | Sandbox |
|---|---|---|
| CalDAV DAV-Root | `…/remote.php/dav` | `http://sandbox-baikal/dav.php` |
| CalDAV Tasks | `/remote.php/dav/calendars/MATTES/inbox-1/` | `/dav.php/calendars/simuser/inbox-1/` |
| IMAP | echter Mailserver | `sandbox-dovecot:993` (imaps, self-signed) |
| Ausgehend/SMTP | echter MTA | `sandbox-mailpit:1025` (Senke, nie Relay) |
| Alle Tokens/Creds | echte Secrets | Wegwerf-Dummies in `sim.env` (VM300-only) |

## Bekannte Fidelity-Gaps (kein STOP)

1. **n8n-Dispatch-Workflows** sind nicht importiert — die Sim-n8n ist auf die Sandbox-Ziele
   umverdrahtet und per Container-Namen erreichbar (nachgewiesen), aber die eigentlichen
   `task`/`calendar`-Workflow-JSONs liegen in `homelab-jarvis-workflows` (Prod-Repo, nicht
   in diesem Auftrag). DoD „echter HTTP-Pfad" ist über die laufenden Container erbracht.
2. **Digest→Basic** bei Baïkal: Prod-Nextcloud nutzt App-Passwörter/Basic; Baïkal ist auf
   `dav_auth_type: Basic` gesetzt (die Adapter nutzen Basic) — formgleich, nicht identisch.
3. **Self-signed TLS** (Dovecot/Mailpit): `IMAP4_SSL` verifiziert per Default nicht — für die
   Sim ausreichend, Prod nutzt echte Zerts.
4. **deck-reader** nicht gebaut (nicht Teil der Referenz-Flows) — `jarvis-tools`
   `DECK_READER_*` zeigt auf einen nicht laufenden Namen (für die geprüften Flows irrelevant).

## Sanitierung / Sicherheit

- `gitleaks` (Pin **8.30.1**) vor **jedem** Mirror-Push — **0 Findings** durchgängig.
- `sim.env` + alle Wegwerf-Creds sind gitignored (jarvis-sim **und** Mirror) und nachweislich
  nicht im Push. VM300 bleibt credential-frei (Git/Push nur von LXC 121).

## DoD — erfüllt

✅ Alle Sandbox-Services Health 200 über echten HTTP-Pfad · ✅ Referenz-E2E S2+S3 grün ·
✅ Isolations-Recheck besteht (VM300 erreicht keine Prod-IP) · ✅ Run-Reports im Mirror ·
✅ `gitleaks` 0 Findings.
