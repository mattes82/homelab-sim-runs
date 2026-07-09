# Sim-Sandbox Run-Report — S3: Routing-Assertion-E2E (der Kern-Test)

- **Datum:** 2026-07-09
- **Stufe:** S3 (von 4) — Commission „Sim: Dispatch-/u037-Workflow-Import + Routing-Assertion-E2E"
- **Zone:** VLAN35 / VM300 (192.168.35.60) — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sim-dispatch-import`
- **Status:** ✅ **GRÜN — Routing-Assertion 8/8, Verteilung ≠ „alle auf einen"**

## OP-S3.1/S3.2 — Fixtures + API-getriggerte Ausführung

8 Intent-Fixtures (`sandbox/dispatch-import/s3/fixtures.json`), **einer pro Switch-Zweig**.
Jede Fixture wird als Mail-JSON **API-getriggert** an den Webhook-Entry
`POST /webhook/sim-dispatch-test` geschickt → durchläuft den **echten** importierten
Pfad: Klassifikator-Heuristik → `known-senders` (jarvis-tools) → Paperless-Gate →
Normalisierung → **Switch `Dispatch: Kategorie` (v3)** → Zweig-Aktion → `jarvis-action-dispatch`
→ Dispatcher → Executor → `sandbox-action-state`. Zweig-Beobachtung: n8n-Execution-Data
(flatted) je Dispatch-Execution geparst (`s3/s3run.py`).

## OP-S3.3 — Routing-Assertion-Matrix (der Regressionswächter)

| Fixture | Erwarteter Zweig | Getroffene Kategorie | Route | Aktion | Routing |
|---|---|---|---|---|---|
| lieferung | Lieferung / Versand | **Lieferung / Versand** | Switch-Ausgang 0 | ok | ✅ PASS |
| newsletter | Newsletter / Werbung | **Newsletter / Werbung** | Switch-Ausgang 1 → Move zu Newsletter | mail.move error¹ | ✅ PASS |
| rechnung | Rechnung / Finanzen | **Rechnung / Finanzen** | Paperless-Dokumentpfad | task.create ok | ✅ PASS |
| account | Account / Sicherheit | **Account / Sicherheit** | Switch-Ausgang 3 | ok | ✅ PASS |
| vertrag | Vertrag | **Vertrag** | Paperless-Dokumentpfad | task.create ok | ✅ PASS |
| termin | Termin / Kalender | **Termin / Kalender** | Switch-Ausgang 5 → Nextcloud Task Termin | task.create ok | ✅ PASS |
| retoure | Retoure | **Retoure** | Switch-Ausgang 6 | ok | ✅ PASS |
| sonstiges | Review / Sonstiges | **Review / Sonstiges** | Switch-Fallback | ok | ✅ PASS |

- **Routing PASS: 8/8.** Jeder Intent landet im **korrekten** Zweig.
- **Negativ-Nachweis (u037-Failure-Signatur):** die Verteilung ist **8 verschiedene
  Kategorien** — `max(count)=1`, also **nicht** „alle auf einen Zweig". Der
  Regressionswächter greift.
- **typeVersion-Parität** des Routing-Nodes (`Dispatch: Kategorie` = **switch v3**,
  identisch zu Prod, s. S2) → die Sandbox testet echtes Prod-Routing.

¹ `newsletter` routet **korrekt** in den Newsletter-Zweig (Nachweis am Execution-Data:
Nodes `Dispatch: Kategorie` → `Confidence Filter Newsletter` → `Move zu Newsletter`).
Die Execution endet in `error` erst **downstream** im `mail.move`-Aktionspfad — der
workflow-**eigene** Transport-Fehlerpfad (`Dispatch-Transport-Stop`) feuert, weil in
der Sandbox **kein reales Mail** zum Verschieben existiert. Das ist eine
**Aktions**-Eigenschaft, nicht das Routing (Routing ist upstream).

### Executor-Aktion am Zielzustand (action-state)

Die geroutete Kette erzeugt echte Records in `sandbox-action-state`, per
`idempotency_key`/`title` eindeutig auf die Fixtures rückführbar:

| Proposal-Titel | tool | idempotency_key → Fixture |
|---|---|---|
| „Termin pruefen: zahnarztpraxis-xyz.de — …" | nextcloud_task.create | `mail-task-termin-matthias-fx-termin` |
| „Dokument pruefen: fitnessstudio-xyz.de" | nextcloud_task.create | `mail-task-dokument-matthias-fx-vertrag` |
| „Dokument pruefen: stromanbieter.de" | nextcloud_task.create | `mail-task-dokument-matthias-fx-rechnung` |

Status `pending_approval` (confirm-gated — echtes Prod-Verhalten; Baïkal-VTODO-Write
erfolgt erst nach Approve/execute-Mode, nicht im AUTO-Pfad).

### Idempotenz (OP-S3.3)

Re-Run `termin` mit **gleichem** `imap_uid=fx-termin` → Proposals bleiben bei 6, **kein
Duplikat** (idempotency_key dedupliziert). ✅

## Health / Isolation (OP-S3.4)

- **Alle Sandbox-Services Health 200** über echten HTTP-Pfad (llm-stub, action-state,
  jarvis-tools, task-writer, caldav-reader/writer, imap-write/reader).
- **Isolations-Recheck nach E2E:** VM300 → `.30.43:5678`/`.30.37:3000`/`.30.21:443`
  = **alle blockiert**. ✅ DROP intakt.

## Fidelity-Gaps (bewusst)

- **IMAP-Trigger-Polling-Pfad** nicht getestet (API-getriggert via Webhook-Entry).
- **mail.move/mail.forward-Aktion** benötigt geseedete Sandbox-Mails; hier wird das
  **Routing** in diese Zweige bewiesen, die Aktion selbst nicht am Zielzustand
  (task.create-Zweige sind am action-state verifiziert).
- **LLM-Qualität** bewusst außen vor (Struktur/Routing getestet, nicht Treffsicherheit).

## Fazit S3

Der Kern-Test ist **grün**: die importierten **echten** Prod-Dispatch-Workflows routen
in der Sandbox jeden der 8 Test-Intents in den **korrekten** Switch-Zweig; die Verteilung
ist nachweislich **nicht** „alle auf einen". Damit ist die Sandbox ein funktionierender
**Regressionswächter** gegen die u037-Switch-Klasse.
