# Sim-Sandbox Run-Report — S4: Gesamt-Abschluss (Dispatch-Import + Routing-E2E)

- **Datum:** 2026-07-09
- **Commission:** „Sim: Dispatch-/u037-Workflow-Import + Routing-Assertion-E2E"
- **Zone:** VLAN35 / VM300 (192.168.35.60) — **nicht** Prod · baut auf #19
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sim-dispatch-import`
- **Status:** ✅ **DoD erreicht** — Routing-Assertion-E2E grün (8/8), typeVersion-treu, Isolation/Health/gitleaks wie #19

## DoD-Nachweis

| Kriterium | Ergebnis |
|---|---|
| Echte Prod-Workflows in Sim-n8n importiert & lauffähig | ✅ 5 Workflows, Kette end-to-end |
| **Jeder** Test-Intent → **korrekter** Zweig | ✅ **8/8** |
| Verteilung ≠ „alle auf einen" (u037-Failure-Signatur) | ✅ 8 verschiedene Kategorien, `max(count)=1` |
| typeVersion-Parität Routing-Node | ✅ `Dispatch: Kategorie` = **switch v3** (= Prod) |
| Isolation VM300 → Prod = Timeout | ✅ nach jeder net-OP rechecked |
| gitleaks 0 Findings vor jedem Push | ✅ (8.30.1) |

## Health-Matrix (13 Container, inkl. neuer)

| Container | Rolle | Health (echter HTTP-Pfad) |
|---|---|---|
| **sandbox-llm-stub** (neu) | deterministischer LLM | `/health` 200 |
| jarvis-sim-n8n | Sim-n8n | `/healthz` 200 |
| sandbox-action-state | Proposals/Idempotenz | `/health` 200 |
| sandbox-jarvis-tools | known-senders | `/health` 200 |
| sandbox-task-writer | task.create-Adapter | `/health` 200 |
| sandbox-caldav-reader/writer | CalDAV | `/health` 200 |
| sandbox-imap-write/reader/reader-m | IMAP-Adapter | `/health` 200 |
| sandbox-dovecot | IMAP-Backend | up |
| sandbox-mailpit | SMTP-Sink | `/api/v1/info` 200 |
| sandbox-baikal | CalDAV-Server | `/dav.php` 401 (auth=up) |

## Routing-Assertion-Matrix (Kern)

| Fixture | Zweig | Aktion | Routing |
|---|---|---|---|
| lieferung → Lieferung / Versand | Switch-0 | ok | ✅ |
| newsletter → Newsletter / Werbung | Switch-1 (mail.move) | mail.move error¹ | ✅ |
| rechnung → Rechnung / Finanzen | Paperless-Dokument | task.create ok | ✅ |
| account → Account / Sicherheit | Switch-3 | ok | ✅ |
| vertrag → Vertrag | Paperless-Dokument | task.create ok | ✅ |
| termin → Termin / Kalender | Switch-5 (task) | task.create ok | ✅ |
| retoure → Retoure | Switch-6 | ok | ✅ |
| sonstiges → Review / Sonstiges | Fallback | ok | ✅ |

**Routing 8/8.** Executor-Aktion am `action-state` verifiziert (3× nextcloud_task.create,
per idempotency_key auf Fixtures rückführbar); **Idempotenz** bestätigt (Re-Run → kein
Duplikat). ¹ newsletter routet korrekt; Fehler erst im mail.move-Aktionspfad (kein reales
Sandbox-Mail) — Details s. S3-Report.

## typeVersion-Paritäts-Nachweis

Aus der Sim-n8n zurück-exportiert und gegen die Quell-JSONs geprüft — **keine
Auto-Migration**:

- `Dispatch: Kategorie` (matthias@-Dispatch) — **switch v3** = Prod
- `Route: Extract Only?` (Intent-Extraktor) — **switch v3** = Prod
- übrige Switch/IF unverändert (v1 / v2.2)

## Overhead-Delta zu #19

| Metrik | Wert |
|---|---|
| **Neuer Container** `sandbox-llm-stub` | **32.82 MiB**, 0.17 % CPU |
| Alle 13 Container (Summe, grob, `awk`) | ~744 MiB |
| `free`: available / swap | 6.2 GiB / **0 B** (swapless, s. #19-OOM-Befund) |
| Container-Count | 13 (12 aus #19 + llm-stub) |

Der Dispatch-Import kostet **~33 MiB** zusätzlich (nur der LLM-Stub; die Workflows laufen
in der bestehenden Sim-n8n). Kein weiterer Container importbedingt.

## Fidelity-Gaps (bewusst)

- **IMAP-Trigger-Polling-Pfad nicht getestet** — API-getriggert via Webhook-Entry
  `sim-dispatch-test` (IMAP-Trigger gestrippt, da Sandbox-Login deaktiviert).
- **mail.move / mail.forward-Aktion** benötigt geseedete Sandbox-Mails; hier ist das
  **Routing** in diese Zweige bewiesen, die Aktion nicht am Zielzustand (task.create-Zweige
  sind am action-state verifiziert).
- **m@-Dispatch** (`jarvis-mail-klassifizierung-dispatch-m-v1.5`, Parallel-Postfach)
  **nicht** importiert → **Fast-Follow**: trivial per selbem `transform.py` nachziehbar;
  außerhalb des Kern-Routing-Guards (matthias@) dieser Runde.
- **LLM-Qualität** bewusst außen vor (Struktur/Routing getestet).

## Mirror-Commits je Stufe

- S1 `runs/2026-07-09_s1-dispatch-deps-llmstub/`
- S2 `runs/2026-07-09_s2-dispatch-import-remap/`
- S3 `runs/2026-07-09_s3-dispatch-routing-e2e/`
- S4 `runs/2026-07-09_s4-dispatch-gesamt/` (dieser Report)

## Fazit

Die Sim-Sandbox ist ein funktionierender **Regressionswächter** gegen die u037-Switch-
Klasse: die echten Prod-Dispatch-Workflows laufen typeVersion-treu gegen Sandbox-Ziele,
und jeder Test-Intent routet nachweislich in den korrekten Zweig (nicht alle auf einen).
Eine künftige routing-brechende Änderung wird ab jetzt **vor** Prod gefangen.
