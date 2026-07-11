# R2-S3 — Runtime-Pflichtsuite szenariobasiert (DR-07) · 2026-07-11

**Branch** `feat/sim-gate-haertung-r2` @5a688e8. Rohbeleg: `evidence.txt`.

## before → after

| Punkt | before | after |
|---|---|---|
| Pflichtmanifest | kategorienbasiert (`RUNTIME_REQUIRED`: je Kategorie ≥1 pass) — „6 Pass + 19 oos" konnte grün sein | **`RUNTIME_REQUIRED_SCENARIOS`**: 12 Szenarioklassen → 27 konkrete Fixture-IDs; je Pflichtfall NUR `pass` zulässig; **unknown/oos/skipped/pending/blocked/fail = hart rot**; Kategorien-Floor bleibt als Untergrenze |
| offline | oos/blocked entzogen Fälle still der Verhaltens-Pflicht | Pflichtfall muss existieren (unknown rot); oos/blocked/fail auch offline rot (laufzeitunabhängig); pending/skipped offline strukturell ok |
| `blocked`-Status | eigener Status außerhalb der Summen (Statuszeile addierte nicht zu total) | Intake-Hard-Block = **beobachteter pass mit Assertion blocked=true** (OP-S4.2-Option 2, für S3-DoD vorgezogen): expected wird asserted; `out_of_scope`-Fehl-Label der class_c/dienstlich-Fixtures entfernt; Summen == total (offline: 140 = 8+35+85+12, Rohbeleg) |

## Pflichtsuite-Szenarioliste (12 Klassen / 27 Pflichtfälle)

| Szenarioklasse | Fixture-IDs |
|---|---|
| doppel_confirm | qc-double-01/02/03 |
| verlorene_antwort_nach_write | qc-lost-response-01 *(neu)* |
| token_falsch | undo-invalid-token-01 |
| token_fehlend | prom-apply-no-token |
| token_leer | undo-empty-token-01 *(neu)* |
| write_fail_500 | fail-target_write_fail-01..04 |
| http_409 / http_429 / http_503 | fail-target_http_{409,429,503}-01 *(neu)* |
| undo_lifecycle | undo-lifecycle-01/02/03 |
| degradierung | prom-degrade-policy, prom-degrade-undos |
| datenklassen_hardblock | fail-class_c_pattern-01..04, fail-dienstlich-01..04 |

**Neue Fähigkeiten** (Freigabe Mattes statt STOP-Abbruch, AskUser 2026-07-10):
Mock-Adapter-Fehlermodi `error_409/429/503` (echte HTTP-Codes, Seam prüft den
KONKRETEN Code — 409 ≠ 500); `empty_token`-Szenario (leerer String unverändert
auf die Leitung; Quell-Orakel `Field(min_length=1)` → 422, empirisch bestätigt);
`lost_response_retry` (Write-Ebenen-Idempotenz via action_id-Dedupe, getrennt
vom Doppel-Confirm auf Decision-Ebene).

## Belege (VM300)

- jas-sandbox: `jarvis-action-state:sim-64a2d19d` (Digest-Pin des aktuellen
  Sandbox-Builds 0.12.0 — hat /promotion+/mail; das alte Pin sim-029dd8b hat
  beides nicht). **Frische Sandbox-DB je Lauf ist Pflicht** (feste proposal_ids
  → 409 bei DB-Wiederverwendung; `run_replay.sh` macht das ohnehin).
- **27/27 Pflichtfälle = pass in einem Lauf** (75 pass gesamt; 12 oos =
  caldav/ntfy/litellm_down out-of-seam; 4 skipped = classification_unclear,
  braucht externe Fault-Injektion → S4-Anmerkung).
- Befund + Fix dabei: action-state P3-Hardening schützt /promotion-Writes per
  Bearer (ohne Token fail-closed **503**, Rohbeleg) → sandbox_up.sh setzt
  Sandbox-Dummy `ACTION_STATE_ADMIN_TOKEN=sim-admin-token`, Seams senden ihn
  bei genau den 7 geschützten Write-Aufrufen. Danach promotion 10/10 pass.
- **Gegenprobe „Pass + oos ist rot"**: Unit-Test `test_dr07_gegenprobe_pass_plus_oos_is_red`
  (+ je verbotener Status ein subTest, + „jeder entfernte Pflichtfall → unknown → rot").
  Live: enforce=runtime-Subset-Lauf **Exit 1**, FAIL-Zeilen je Pflichtfall
  szenario-genau (Rohbeleg enforce-subset2); mit frischer DB bleibt der Subset-Lauf
  prinzipiell rot (relevance/lifeops fehlt → S4; Composer-Inventar; classification_unclear).
- 62 Unit-Tests grün; `make verify-offline` Ende-zu-Ende grün (Intake-Block-
  Fixtures offline = 8× pass).

## Invarianten

Isolations-Recheck VM300 → 3× rc=28 Timeout (Rohbeleg). gitleaks staged: 0.
Kein sim.env/Secret berührt; Admin-Token = dokumentierter Sim-Dummy.
