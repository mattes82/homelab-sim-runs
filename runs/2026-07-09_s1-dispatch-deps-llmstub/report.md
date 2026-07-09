# Sim-Sandbox Run-Report — S1: Dependency-Analyse + LLM-Stub

- **Datum:** 2026-07-09
- **Stufe:** S1 (von 4) — Commission „Sim: Dispatch-/u037-Workflow-Import + Routing-Assertion-E2E"
- **Zone:** VLAN35 / VM300 (192.168.35.60), Hostname `jarvis-sim` — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sim-dispatch-import`
- **Baut auf:** #19 (Sandbox-Foundation, gemergt)
- **Status:** ✅ grün — **kein STOP** (keine harte, un-stubbare Routing-Abhängigkeit)

## OP-S1.1 — Workflow-Quellen gestagt (LXC 121 → VM300)

`homelab-jarvis-workflows` auf LXC 121 (Gitea erreichbar), read-only nach VM300
`/opt/sandbox-src/workflows/` gestagt. Die **u037-JSONs sind im aktuellen Tree nicht
mehr vorhanden** (später umstrukturiert) — aus dem von der Commission genannten
Commit-Kontext **68fae56 (PR#25)** aus der Git-Historie geholt (read-only auf LXC 121).

| Rolle | Quell-Datei | Herkunft |
|---|---|---|
| matthias@-Dispatch (Entry, Switch) | `jarvis-mail-klassifizierung-dispatch-matthias-v1.5.json` | aktueller Tree |
| u037 Intent→Proposal (Brücke) | `u037-intent-to-proposal-v01.json` | git `68fae56` |
| Intent-Extraktor (LLM-Sub-WF) | `jarvis-intent-extractor-v1.json` | git `68fae56` |
| Action-Dispatcher (Webhook `jarvis-action-dispatch`) | `jarvis-action-dispatcher.json` | aktueller Tree |
| Action-Executor | `jarvis-action-executor-v1.1.json` | aktueller Tree |

Zusätzlich aus `68fae56` mitgeholt: Gold-Fixtures `tests/fixtures/u037/{arzt_termin,
paket_abholung,newsletter_negativ,abo_frist}.json` und `docs/intent_extractor_prompt_v1_2.md`
(LLM-Contract-Referenz für den Stub) — für S3.

## OP-S1.2 — Abhängigkeitsgraph + STOP-Analyse

### Routing-Pfad (deterministisch, LLM NICHT beteiligt)

Der Switch **`Dispatch: Kategorie`** (`n8n-nodes-base.switch` **typeVersion 3**, 8 Ausgänge)
ist der Regressionswächter-Node. Die Predecessor-Kette zum Switch ist rein
**heuristisch/deterministisch** — kein LLM-Node auf dem Pfad:

```
Email Trigger (IMAP) / Manueller Teststart
  → Mail minimieren (code)
  → Mail klassifizieren (code: Keyword/Domain-Heuristik → category)
  → known-senders laden (HTTP GET jarvis-tools:8080/config/known-senders)
  → known-sender Pre-Filter (code: Override)
  → Paperless Gate IF
  → Normalisierung Kategorie (code: Whitelist 7 + Review/Sonstiges)
  → Kategorie unbekannt? (IF)
  → Dispatch: Kategorie (Switch v3, 8 Zweige)   ← ROUTING-NODE
```

Die 8 Switch-Zweige: `Lieferung / Versand`, `Newsletter / Werbung`, `Rechnung / Finanzen`,
`Account / Sicherheit`, `Vertrag`, `Termin / Kalender`, `Retoure`, `Review / Sonstiges`.

### Ziel-Dienste je Workflow (Abgleich gegen #19-Sandbox)

| Referenz (Prod-Form) | Rolle | Sandbox-Pendant | Status |
|---|---|---|---|
| `litellm:4000/v1/chat/completions` (Extractor) | LLM Intent-Extraktion | `sandbox-llm-stub:4000` (S1 gebaut) | ✅ stub |
| `litellm.lan/v1/chat/completions` (LLM-Bestätigung) | LLM JA/NEIN-Gate (**nicht** routing-entscheidend) | `sandbox-llm-stub:4000` | ✅ stub |
| `jarvis-action-state:8093/*`, `host.docker.internal:8093` | Proposals/Idempotenz/Memory | `sandbox-action-state:8093` | ✅ #19 |
| `jarvis-tools:8080/config/known-senders` | known-sender-Map (Routing-Override) | `sandbox-jarvis-tools:8080` (liefert `{}`) | ✅ #19 |
| `jarvis-tools.lan/config/known-shops` | Shop-Liste (peripher, liefert 404) | `sandbox-jarvis-tools:8080` | ⚠️ peripher, continueOnFail (S2) |
| `127.0.0.1:5678` / `localhost:5678/webhook/*` | Self (u037, jarvis-action-dispatch) | Sim-n8n selbst | ✅ intern |
| `ntfy.lan/HomeLab*` | Push-Benachrichtigung (peripher) | Stub-Sink (200) | ⚠️ peripher |
| Sub-WF `0hGHWfhc6uWcHGft` (Execute Tracking) | Mail-Tracking (peripher) | — | ⚠️ peripher (S2: continueOnFail/No-op) |

### Sub-Workflow-Referenzen (executeWorkflow — IDs für S2-Remap)

- u037 → Intent-Extraktor: `DlsE9BB4EwCIly5K`
- Action-Dispatcher → Executor: `Drk5YMC8VKrpL7AI`
- Dispatch v1.5 → Tracking: `0hGHWfhc6uWcHGft` (peripher)
- Entry-Kontext matthias@-Dispatch: `So3Ktjz9tZDyhmgy`

**STOP-Analyse (OP-S1.2):** Jede **routing-entscheidende** Abhängigkeit ist entweder
in der #19-Sandbox vorhanden (`action-state`, `jarvis-tools`) oder trivial stubbar
(LLM). Der Zweig-entscheidende Pfad braucht **keinen** LLM. `known-senders` liefert `{}`
→ reines Heuristik-Routing, keine Overrides. → **Keine harte un-stubbare Routing-
Abhängigkeit. KEIN STOP.** Periphere Refs (ntfy, known-shops, Tracking) dokumentiert,
kein STOP (Commission-Vorgabe).

### typeVersion-Bestand der Quell-JSONs (Parität-Ziel für S2)

| Workflow | Switch/IF-Nodes (typeVersion) |
|---|---|
| dispatch-matthias-v1.5 | **switch v3** (`Dispatch: Kategorie`, 8 Ausgänge) · switch v1 ×2 · if v1 ×2 |
| u037-intent-to-proposal | if v1 ×2 · webhook v2 |
| jarvis-intent-extractor | **switch v3** (`Route: Extract Only?`) |
| jarvis-action-dispatcher | if v2.2 ×3 · webhook v2 |
| jarvis-action-executor | if v2.2 ×2 |

## OP-S1.3 — sandbox-llm-stub (deterministisch, credential-frei)

- **Bau:** FastAPI (`fastapi==0.115.6`, `uvicorn==0.34.0`), `python:3.12-slim`,
  Container `sandbox-llm-stub` auf `sandbox-net`, Host-Bind `127.0.0.1:4000`,
  Netz-Alias `litellm`. Dummy-Key (Auth ignoriert). Code: `sandbox/llm-stub/`.
- **Determinismus:** fixer `created`, keine Zufallswerte, 1 Worker.
- **Contract-Weiche** (aus den echten Prod-Nodes abgelesen):
  1. **Extractor** (system=Extractor-Prompt, user=Mail-JSON) → `choices[0].message.content`
     = JSON-String `{"intents":[…]}`, am Mail-Input **gekeyt** (Betreff/Snippet →
     `appointment`/`task`/`reminder`), Confidence ≥0.70. Formgleich zum Parse-Node
     (`parsed.intents ?? []`).
  2. **LLM-Bestätigung** („… JA oder NEIN …", `max_tokens 10`) → content `JA`/`NEIN`,
     deterministisch (JA nur bei Rechnungs-/Quittungssignal). Formgleich zum
     Parse-Node (`content.includes('JA')`).
  3. **Catch-all-Sink** (200) für periphere POSTs (ntfy.lan), damit periphere Nodes
     das Routing nicht mit Fehlern stören.

### Form-Validierung (echter HTTP-Pfad) — s. `evidence.txt`

| Prüfung | Ergebnis |
|---|---|
| `GET /health` | 200 `{"status":"ok"}` |
| Extractor-POST → `parsed.intents` | ✅ valides `{"intents":[{intent_type…}]}` (keyed: „Termin"→appointment) |
| Bestätigungs-POST („Rechnung") → content | ✅ `JA` |
| Sink-POST `/HomeLab` | ✅ 200 |
| Auflösung aus `jarvis-sim-n8n` (`sandbox-llm-stub:4000`) | ✅ 200 |

## Health / Isolation (OP-S1.4)

- **13 Container up** (12 aus #19 + `sandbox-llm-stub` healthy).
- **Isolations-Recheck nach Stub-Build:** VM300 → `.30.43:5678`, `.30.37:3000`,
  `.30.21:443` = **alle blockiert**. ✅ DROP intakt.
- Ressourcen (Phase 0): `free` 6.3 GiB available, swapless; `df /` 43 %.

## Fazit S1

Abhängigkeitsgraph vollständig dokumentiert; Routing nachweislich deterministisch
(kein LLM auf dem Switch-Pfad). LLM-Stub gebaut und gegen **beide** realen Contract-
Formen form-validiert. Keine harte Routing-Abhängigkeit → kein STOP. Weiter mit S2
(Import + typeVersion-treues Remapping).
