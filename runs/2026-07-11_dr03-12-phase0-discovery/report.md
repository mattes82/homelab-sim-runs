# Phase-0-Discovery — Datenklassen-Realitätsprüfung (DR-03/DR-12)

**Datum:** 2026-07-11 · **Commission:** `homelab-docs-tooling/commissions/inbox/multi/2026-07-10_commission_dr03-12-datenklassen-realitaet.md` (Ablage-PR docs-tooling#64, gemergt)
**Modus:** read-only auf LXC 121; alle Aussagen gegen `origin/main` der Prod-Repo-Clones via `git grep`/`git show` (kein Checkout, kein Branch, kein Commit in Prod-Repos).
**Quell-SHAs:** `homelab-docker` origin/main **002acea** · `homelab-jarvis-workflows` origin/main **579633c** · (Randfundort `homelab-scripts` origin/main für secret-scan.sh)

---

## Befund (OP-0.5): **Branch B — kein realer Seam im Sinne der Kern-Frage**

**Kern-Frage:** Gibt es irgendeinen Live-Codepfad, der `privacy_class` ableitet **und** `routing_policy` durchsetzt (Cloud-Block A/C, Log/Persistenz-Block C)?

**Antwort: Nein — belegt.** Es existiert kein Codepfad in beiden Repos, der ein `privacy_class`-Label an einer echten Senke durchsetzt: kein Cloud-Call wird anhand privacy_class/routing_policy blockiert, kein Log-/Persistenz-Write anhand Klasse C verweigert. Der einzige Cloud-LLM-Call im exportierten Dispatch läuft **ungegated** (s. OP-0.1).

**ABER — für Mattes' Einordnung wichtig (keine erzwungene Selbst-Einordnung):** Es existieren mehrere **echte, absichtsvolle Teil-Mechanismen**, die Teile der §2.5-Policy tragen, ohne die Kern-Frage zu erfüllen. Sie sind unten einzeln belegt (§ Teil-Mechanismen). Die Commission antizipiert diese Kategorie selbst als „verwandter, engerer Mechanismus" → Branch B. Konsequenz gemäß Commission: **STOP nach Phase 0, keine Seam gebaut**, Ersatzarbeit nur nach Freigabe.

---

## OP-0.1 — u037/Dispatch-Workflow (homelab-jarvis-workflows @ 579633c)

1. **Der u037-Workflow ist NICHT im Repo exportiert.** Einzige Vorkommen von `u037` (Rohbeleg: Grep-Läufe, s. `op01_dispatch_v15_nodes.txt` Kopf + Session): `monitoring/uptime-kuma-relevance-failsafe.yaml` und die beiden Dispatch-v1.5-JSONs, die den Webhook `http://127.0.0.1:5678/webhook/u037-intent-to-proposal` **aufrufen** (dispatch-m-v1.5 Zeile 1110). `body_for_llm` kommt **in keinem der beiden Repos** vor (`grep_workflows_sensitivity.txt`, `grep_docker_sensitivity.txt`). ⇒ **Das behauptete u037-Sensitivity-Gate ist aus den Prod-Repos code-seitig weder bestätigbar noch widerlegbar** — es liegt nur im Live-n8n. Kein Raten: als Verifikations-Lücke gemeldet.
2. **Dispatch übergibt den Body ungegated an u037:** Node „Trigger u037 Intent Extraction" sendet `body_text: $json.snippet` bedingungslos, dazu `data_class` und ein krudes `sender_class` (Newsletter/Tracking→'privat', **alles andere**→'dienstlich') — Label wird übergeben, im Repo nirgends ausgewertet (`op01_dispatch_v15_kernnodes.txt`).
3. **`data_class` ist im Klassifikator eine Konstante:** einzige Zuweisung `let data_class = 'S2';` (Node „Mail klassifizieren", Zeile 314 des extrahierten Codes; Gegenprobe: keine weitere Zuweisung in 627 Zeilen). **Keine Ableitung** — weder S-Level noch A/B/C.
4. **Direkter Cloud-LLM-Call ohne Datenklassen-Gate:** Node „LLM-Bestätigung" → `http://litellm.lan/v1/chat/completions`, `model: 'cloud-fast-claude'`, sendet **Betreff + Snippet**. Vorgeschaltete Gates prüfen nur `paperless_candidate === true` (Paperless Gate) und Domain-bekannt — **keine** privacy_class/sensitivity-Prüfung in der Kette `Paperless Gate → Shop-Liste laden → Domain-Check → Domain bekannt? → LLM-Bestätigung` (Verkettung jq-belegt in `op01_dispatch_v15_kernnodes.txt`). Da der Klassifikator Bank-/Steuer-Kategorien vergibt (Zeilen 440–457, 544–566) und diese `paperless_candidate=true` setzen können, **können Mails der Kategorien Bank/Steuer (nach §2.5-Kopplung in relevance.py: Klasse A/local_only) den Cloud-Call erreichen.**
5. **Implizite A/B/C-artige Fänger im selben Workflow:** known-sender Pre-Filter = reiner Kategorie-Override (kein Privacy-Fang); Bank/Steuer-Keyword-Listen steuern nur Kategorie/Paperless-Routing, blocken nichts. Zwillings-Check: matthias-v1.5 identisch auf beiden Kern-Nodes.

## OP-0.2 — Cortex-Artefakte (beide Repos)

`git grep -iE 'cortex|u039|/classify|routing_log'` → **0 Treffer in homelab-docker UND homelab-jarvis-workflows** (`op02_cortex_suche.txt`, exit=1). Compose-Service-Inventar aller `compose*.yml` in homelab-docker: kein cortex-, classifier- oder extractor-Service (vollständige Liste im Rohbeleg). **Erwartung laut Roadmap bestätigt: Cortex/`/classify`/`routing_log` existiert nicht deployt.**

## OP-0.3 — Adapter/Executor

`git grep -iE 'privacy_class|routing_policy|data_class|sensitivity'` über `jarvis-imap-write-adapter`, `jarvis-nextcloud-task-writer`, `jarvis-caldav-writer`, `jarvis-tools`, `jarvis-inbox-receiver`, `jarvis-artifact-intake` (docker) sowie `jarvis-action-executor-v1.1.json` + `jarvis-action-dispatcher.json` (workflows) → **0 Treffer** (`op03_adapter_executor.txt`, exit=1). **Keine Datenklassen-Prüfung vor Write/Log in irgendeinem Adapter/Executor.**

## OP-0.4 — secret-scan.sh

**0 Referenzen** in homelab-docker und homelab-jarvis-workflows (`op04_secretscan.txt`, exit=1). Das Skript liegt in `homelab-scripts` (top-level, Selbstbeschreibung: „Klasse-C-Pattern-Erkennung **vor RAG-Import**", manueller Aufruf `bash secret-scan.sh <Pfad>`), einzige Repo-Referenzen sind .gitignore-Ausnahmen. ⇒ **Ausschließlich manuelles Pre-RAG-Import-Tool; keine Laufzeit-Einbindung auf Nachrichten-/Content-Ebene.** (Die Commit-Zeit-Gates in docs-tooling/CI nutzen gitleaks — ein anderes Werkzeug; secret-scan.sh ist auch dort nicht eingebunden.)

---

## Teil-Mechanismen (real existierend, aber ≠ Kern-Frage) — Einordnung durch Mattes

1. **`jarvis-lifeops/app/relevance.py`** (deployter Service; Volltext `cat_lifeops_relevance_py.txt`): leitet `privacy_class` C/A/B deterministisch ab (dienstlich→C, persönlich/Bank/Steuer→A, sonst B; Fail-Safe: Persona-Ausfall→C) und liefert die **komplette §2.5-Kopplung** `routing_policy/learning_policy/content_logging` (`_COUPLING`, Zeilen 44–49, 150–156). **Aber:** reiner Decision-Helper via `POST /relevance/evaluate` — er setzt nichts durch. Einziger Konsument im Repo: SHADOW-Workflow (s. 2.). Der Haupt-Dispatch v1.5 ruft ihn **nicht** auf (Node-Inventar `op01_dispatch_v15_nodes.txt`: keine lifeops-URL).
2. **`jarvis-action-state/app/mail_move.py`** (`cat_actionstate_mail_move_py.txt`): `/mail/move/eligibility` enforced eine deterministische AND-Regel inkl. Pflicht-Verdict der Relevanz-Engine (degrade-to-restrictive, Zeilen 70–77) und leitet `privacy_class` B/C aus Markern ab (Zeile 88). **Aber:** selbstdokumentiert „pure decision helpers (no state mutation); the n8n dispatch calls them in SHADOW" (Zeilen 15–16); einzige Konsum-Kette = `jarvis-mail-move-eligibility-shadow-v0.1` → **ntfy-Signal, kein Executor-Pfad** (`op05_verdict_konsumenten.txt`). Zudem dokumentierter GAP: der Live-Klassifikator emittiert die Marker persönlich/dienstlich gar nicht (Zeilen 36–41) — konsistent mit Befund OP-0.1.3.
3. **`jarvis-action-state/app/promotion.py`**: harter Promotion-Block `if class_c > 0: blocking` („class-C data touches (need 0)") — enforced, aber (a) auf `memory_decisions.data_class`, das client-geliefert ist (Default 'B', Ableitung nirgends) und selbstdokumentiert **PROXY**, (b) Gegenstand ist **Autonomie-Promotion**, nicht Cloud-Routing/Log je Nachricht (`op05_actionstate_dataclass.txt`).
4. **`u018pre` Notiz-Intent-Klassen-Gate** (workflows: `tools/u018pre_note_intent.py` + `u018pre-note-intent-v01.json`; `op01_randbefund_u018pre.txt`): **echtes, durchsetzendes Cloud-Gate** — Keyword-Heuristik (mandant, kunde, vertrag, gehalt, lohn, kündigung …) ⇒ Treffer → **KEINE Cloud-Extraktion**, REVIEW-Proposal. Fängt faktisch C-artige Inhalte **ohne** privacy_class-Label — der von der Commission benannte „verwandte, engere Mechanismus", nur für den **Notiz-Pfad** (nicht Mail).

**Warum trotzdem eindeutig Branch B:** Keiner dieser vier Mechanismen leitet `privacy_class` ab **und** setzt `routing_policy` an echten Senken durch. 1+2 leiten ab, setzen aber nicht durch (SHADOW/ntfy-only); 3 setzt durch, leitet aber nicht ab (Proxy-Feld) und betrifft Promotion; 4 setzt durch, nutzt aber kein Klassen-Label und betrifft nur Notizen. Eine Sandbox-Seam-Anbindung hätte kein reales Vorbild — exakt die Scheinsicherheit, die DR-03/DR-12 kritisieren.

## Explizit geprüfte Orte (Branch-B-Nachweis „an X/Y/Z geprüft, dort nichts")

- workflows: beide Dispatch-v1.5 (m+matthias, Node-für-Node), Intake v0.2–0.4, Executor v1.1, Dispatcher, Confirm-Workflows, alle 49 Workflow-JSONs via Repo-weitem Grep; docs/workflow-index; monitoring/.
- docker: alle Compose-Services (Inventar), alle Adapter (imap-write, nextcloud-task-writer, caldav-writer, tools, inbox-receiver, artifact-intake), action-state (main.py: data_class nur Schema/Storage/Read-back, Zeilen 182/193/357/529/1353), lifeops (relevance/persona/store), litellm-Ordner.
- Muster: `privacy_class|routing_policy|data_class|datenklasse|sensitivity|body_for_llm|cortex|u039|/classify|routing_log|secret-scan|gitleaks` — jeweils Repo-weit gegen origin/main.

## Limitationen

- **Live-n8n nicht eingesehen** (Phase-0-Scope = Repo-Clones): u037-intern (Sensitivity-Gate, Extractor) nicht verifizierbar; der u037-Workflow ist nicht exportiert. Falls gewünscht: read-only n8n-Export-Abgleich als Folgeauftrag.
- Laufende Container/ENV auf VM200 nicht geprüft (gleicher Grund); Aussagen gelten für den in Git versionierten Stand.

## Rohbelege in diesem Ordner

`grep_docker_privacyclass.txt`, `grep_docker_sensitivity.txt`, `grep_workflows_privacyclass.txt`, `grep_workflows_sensitivity.txt`, `op01_dispatch_v15_nodes.txt`, `op01_dispatch_v15_kernnodes.txt`, `op01_randbefund_u018pre.txt`, `op02_cortex_suche.txt`, `op03_adapter_executor.txt`, `op04_secretscan.txt`, `op05_verdict_konsumenten.txt`, `op05_actionstate_dataclass.txt`, `cat_lifeops_relevance_py.txt`, `cat_actionstate_mail_move_py.txt`.

## Konsequenz (gemäß Commission, Branch B)

STOP nach Phase 0. **Keine Sandbox-Seam gebaut.** Kein Schreibzugriff auf homelab-jarvis-sim erfolgt. Ersatzarbeit (u037-Sensitivity-Gate in Sandbox-Verify-Kette — setzt ohnehin zuerst einen n8n-Export von u037 voraus, s. Limitationen; alternativ u018pre-Klassen-Gate, das repo-versioniert und damit sofort treu testbar wäre) **nur nach Mattes' Freigabe**.
