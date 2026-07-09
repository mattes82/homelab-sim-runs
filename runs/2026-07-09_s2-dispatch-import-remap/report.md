# Sim-Sandbox Run-Report — S2: Import + Credential-Remapping (typeVersion-treu)

- **Datum:** 2026-07-09
- **Stufe:** S2 (von 4) — Commission „Sim: Dispatch-/u037-Workflow-Import + Routing-Assertion-E2E"
- **Zone:** VLAN35 / VM300 (192.168.35.60) — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sim-dispatch-import`
- **Status:** ✅ grün — 5 Workflows importiert & aktivierbar, **typeVersion-Parität nachgewiesen**, keine blockierenden toten Refs

## OP-S2.1 — Sandbox-Credentials reconcilen

Sim-n8n hatte **keine** gespeicherten Credentials (Befund S1). 5 Sandbox-Credentials
per `n8n import:credentials` angelegt — mit den **Prod-Credential-IDs**, damit die
Node-Refs ohne Node-Edit auflösen (= Remap durch Umpointen der ID auf Sandbox-Werte):

| Prod-ID | Typ | Sandbox-Wert |
|---|---|---|
| `Ltga7bRpKHD6b5ZH` | httpBearerAuth | `sim-task-writer-token` (Executor → task-writer) |
| `y0FBmZxlft6pG4hS` | httpBearerAuth | `sim-jarvis-tools-token` (known-senders) |
| `T3NR0x3i1NM3R7TM` | httpBearerAuth | Dummy (LLM-Stub ignoriert Auth) |
| `Mm8BNdiXd9Ep7xew` | httpBearerAuth | Dummy (ntfy-Sink) |
| `6pt2fQtp4Qr9jtNF` | imap | Platzhalter (IMAP-Trigger gestrippt) |

Werte kommen aus `sim.env` (VM300-lokal) — **nie** in Git/Mirror.

## OP-S2.2 — Import + Remapping

`transform.py` (im Sim-Repo, `sandbox/dispatch-import/`) normalisiert die 5 Quell-JSONs
auf Import-Form, weist kanonische IDs zu (Sub-Refs lösen auf) und **remappt alle Ziel-URLs**
auf Sandbox-Container. Keine Prod-Referenz bleibt außer n8n-Self (`localhost:5678`):

| Prod-Ziel | Sandbox-Ziel |
|---|---|
| `litellm:4000`, `litellm.lan` | `sandbox-llm-stub:4000` |
| `jarvis-action-state:8093`, `host.docker.internal:8093` | `sandbox-action-state:8093` |
| `jarvis-tools:8080`, `jarvis-tools.lan` | `sandbox-jarvis-tools:8080` |
| `jarvis-nextcloud-task-writer:8094` | `sandbox-task-writer:8094` |
| `jarvis-imap-write-adapter:8080` | `sandbox-imap-write:8080` |
| `ntfy.lan` | `sandbox-llm-stub:4000` (Sink) |
| `n8n.lan`, `127.0.0.1/localhost:5678` | n8n Self (unverändert) |

Import-Ergebnis (alle „Successfully imported"):

| ID | Name | active |
|---|---|---|
| `So3Ktjz9tZDyhmgy` | Dispatch matthias@ v1.5 (Entry, Switch) | ✅ 1 |
| `CRPVVgXZ5y6xMVsP` | u037-intent-to-proposal | ✅ 1 |
| `DlsE9BB4EwCIly5K` | jarvis-intent-extractor (Sub-WF) | 0 (executeWorkflow-Trigger, muss nicht aktiv) |
| `SimDispatch0001` | Action Dispatcher (Webhook) | ✅ 1 |
| `Drk5YMC8VKrpL7AI` | Action Executor (Sub-WF) | 0 (executeWorkflow-Trigger) |

**Sub-Workflow-Refs:** `Drk5YMC8VKrpL7AI` (Executor) und `DlsE9BB4EwCIly5K` (Extraktor)
existieren → lösen auf. `0hGHWfhc6uWcHGft` (Mail-Tracking) **nicht** importiert → der
Node „Execute Tracking" ist auf `onError=continueRegularOutput` gesetzt (peripher, kein
Routing-Einfluss). **Keine tote Credential-Ref** (alle 5 vorhanden).

## OP-S2.3 — typeVersion-Parität (Kernanforderung)

Aus der Sim-n8n **zurück-exportiert** und gegen die Quelle geprüft — **keine
Auto-Migration** beim Import:

| Workflow | Routing-Node (gespeichert) | Quelle | Parität |
|---|---|---|---|
| Dispatch matthias@ | `Dispatch: Kategorie` → **switch v3** | switch v3 | ✅ identisch |
| Dispatch matthias@ | `Domain bekannt?`/`LLM-Ergebnis?` → switch v1 | switch v1 | ✅ |
| Intent-Extraktor | `Route: Extract Only?` → **switch v3** | switch v3 | ✅ |

Der Regressionswächter-Node (`Dispatch: Kategorie`, **v3**, 8 Ausgänge) hat exakt die
Prod-typeVersion → die Sandbox spiegelt das Prod-Routing-Verhalten.

## API-Trigger-Anpassung (dokumentiert)

- **IMAP-Trigger gestrippt:** `emailReadImap` kann sich in der Sandbox nicht einloggen
  („Logging in is disabled") → würde die Workflow-Aktivierung abbrechen. Entfernt →
  **Fidelity-Gap: IMAP-Polling-Pfad nicht getestet** (API-getriggert).
- **Webhook-Entry `sim-dispatch-test`** (webhook v2) eingefügt, verdrahtet auf
  `Mail klassifizieren`. Routing-Logik/Switch **unberührt**.
- Aktivierung via `n8n update:workflow --active` + `docker restart` (kein Public-API-Key
  in der Sim-n8n — Befund S1; CLI-Weg statt `POST .../activate`).

## OP-S2.4 — Verifikation

- **Aktivierbar:** alle 3 Webhook-Workflows registriert → `POST /webhook/{sim-dispatch-test,
  u037-intent-to-proposal,jarvis-action-dispatch}` = **200** (keine 404).
- **Smoke-Kette** (Termin-Fixture → `sim-dispatch-test`): Executions 25 (Dispatch) →
  24 (Dispatcher) → 23 (u037) alle `success`. Volle Kette läuft end-to-end.
- **Isolations-Recheck nach Import:** VM300 → `.30.43:5678`/`.30.37:3000`/`.30.21:443`
  = **alle blockiert**. ✅ DROP intakt.

## Fazit S2

5 echte Prod-Workflows importiert, credential- und URL-remappt auf die Sandbox,
**typeVersion-treu** (Switch v3 nachgewiesen), aktiv und als Kette lauffähig. Bereit
für S3 (Routing-Assertion je Zweig).
