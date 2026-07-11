# R2 — Sandbox-Gate-Härtung gesamt: rigoros, integriert, ehrlich · 2026-07-11

**Commission** `sim-gate-haertung-r2` (2026-07-10, homelab-docs-tooling PR #51).
**Branch** `feat/sim-gate-haertung-r2`, 4 Commits (1 je Stufe):
S1 `79345a5` → S2 `f5fda2d` → S3 `5a688e8` → S4 `d045b26`. Basis: main `6b91fe4`
(= #23 + #24 via Nachlande-PR #25 — #24 war durch Merge-Reihenfolge-Unfall
7 s nach #23 in die schon gemergte Base gelandet, Block-0-Befund).
**PR offen, kein Auto-Merge** (Commission-Vorgabe).

Stufen-Reports (je mit Rohbeleg): `2026-07-10_r2-s1-gate-integration/`,
`_r2-s2-semantik-rigor/`, `_r2-s3-pflichtsuite-szenario/`, `_r2-s4-plumbing/`.

## before → after je DR

| DR | before | after (Beleg) |
|---|---|---|
| DR-08 | verify-offline führte neue Tests NICHT aus, semantic-gate ohne Aufrufer → committete Transformate alterten still (18 undeklarierte Abweichungen gegen frische Quellen!) | `verify-offline: unit-test semantic-gate`; 62-Tests-Discover + Gate im Standardpfad; `parity-not-staged` eindeutig rot; stale Transformate regeneriert, confirm-v04 erstmals committet. 3 Gegenproben rot (S1) |
| DR-04 | `node_sig` = (type, typeVersion, onError, params) — credentials/disabled/retryOnFail/… unverglichen | Voll-Vergleich außer Ignorable-Allowlist; Mutations-Gegenproben credentials/retryOnFail/disabled am echten Transformat → je rot (S2) |
| DR-05 | ungestagte Workflows still übersprungen (`checked==0`-Logik) | `WORKFLOW_MANIFEST`: alle 6 required, beidseitig; fehlender Required → Exit 1 (Gegenprobe belegt) |
| DR-06 | `DECLARED_ADDED_NODES` global, IMAP-Strip überall | `DECLARED_INJECTIONS` pro Workflow: exakte Signaturen, exakte Kanten, Anzahl; Strip nur im Dispatch-Workflow, vollständig (Unit-Mutationstests) |
| DR-02 (Sandbox) | `unmask_confirm_map` demaskierte die Sim-only-Map im Gate → „GRÜN 6 WF" verdeckte die Prod-Divergenz | Parity/Harness getrennt; `unmask_confirm_map` entfernt. **Parity = EHRLICH ROT (Exit 3)** mit exakt 1 deklarierter Lücke, 0 undeklarierte (keine weitere Divergenz → kein STOP-Fall); Harness = grün, ausdrücklich **NICHT promotion-fähig** |
| DR-07 | Pflichtsuite kategorienbasiert — „6 Pass + 19 oos" konnte grün sein | `RUNTIME_REQUIRED_SCENARIOS`: 12 Szenarioklassen / 27 Fixture-IDs, je Fall NUR `pass`; unknown/oos/skipped/pending/blocked/fail hart rot (alle Modi sinngemäß: oos/blocked/fail auch offline rot). Gegenprobe Pass+oos → rot (Unit + live) |
| DR-10 | `$UP_CMD` ungeprüft, trap nach Startpfad | fail-fast mit eigenen Exitcodes, trap vor Startpfad, Mock-Readiness+Liveness; `REPLAY_UP_CMD=false` → Exit 2 + teardown (Beleg) |
| DR-11 | `blocked` außerhalb der Summen; `write_summary` hartkodiert | Intake-Hard-Block = beobachteter pass mit Assertion `blocked=true` (OP-S4.2-Option 2); Summen dynamisch + Assertion `Summe==total`; Beleg 89+0+35+0+16=140 |
| DR-13 | nackte `open(...)`, ResourceWarnings | `Path.read_text/write_text`/`with open`; `-W error::ResourceWarning` im unit-test-Gate |

## Ignorable-Allowlist (DR-04)

Node-Ebene: `id`, `position`, `notes`, `notesInFlow`, `color`. Workflow-Ebene: keine.
Endpoint-Kanonisierung (URL_REMAP → Platzhalter) = deklarierter Injektionspunkt.

## Required-Workflow-Manifest (DR-05)

Alle 6 Workflows **required** (executor v1.1, intent-extractor v1, dispatch
matthias v1.5, u037, dispatcher, confirm-v04); assert-gekoppelt an `transform.IDMAP`;
keine optionalen (Optionalität müsste eng begründet deklariert werden).

## Parity/Harness (DR-02) — ehrlich rote Confirm-Map

`make semantic-gate` (Standardpfad) = Harness hart + Parity gemeldet;
`make semantic-gate-parity` = promotionsrelevant, **Exit ≠ 0 bis der Prod-Fix
(separater Track) die adapterMap ergänzt**. Beleg beidseitig (LXC 121 + VM300):
Harness GRÜN/„NICHT promotion-fähig", Parity `EHRLICH ROT — 1 deklarierte
Prod-Lücke (DR-02 confirm-adapterMap); undeklarierte Abweichungen: 0`.

## Pflichtsuite-Szenarioliste (DR-07) — 12 Klassen / 27 Pflichtfälle

doppel_confirm (qc-double-01..03) · verlorene_antwort_nach_write
(qc-lost-response-01, neu) · token_falsch (undo-invalid-token-01) · token_fehlend
(prom-apply-no-token) · token_leer (undo-empty-token-01, neu, Quell-Orakel 422) ·
write_fail_500 (fail-target_write_fail-01..04) · http_409/429/503
(fail-target_http_*-01, neu inkl. Mock-Fehlermodi mit echten Codes) ·
undo_lifecycle (undo-lifecycle-01..03) · degradierung (prom-degrade-policy/undos) ·
datenklassen_hardblock (fail-class_c_pattern-01..04 + fail-dienstlich-01..04).
Fixture-Lücken wurden per Mattes-Freigabe (AskUser 2026-07-10) GEBAUT statt
STOP — echte beobachtete Antworten, kein Faken.

## Kein Regress #20/#21 (Harness-Modus, VM300)

Transformate regeneriert + alle 6 re-importiert: **#20 Routing 8/8 PASS**;
**#21 Effekt-Matrix `[('werbung', True), ('newsletter', True), ('beleg', True)]`**
unverändert, idem + whitelist grün (Rohbeleg S2).

## Keystone: erster grüner `make replay`

VM300, frische Sandbox-DB: **Exit 0 — 140 Fälle, 89 pass / 0 fail / 35 pending /
0 skipped / 16 oos**, 27/27 Pflichtfälle, relevance 14/14 via `sandbox-lifeops`.
„Grün" ist jetzt fail-closed verdient: jede Stufe trägt belegte Gegenproben, die
es brechen würden.

## Quell-SHAs / Pins

- homelab-jarvis-sim main-Basis: `6b91fe4`; R2-Commits `79345a5`/`f5fda2d`/`5a688e8`/`d045b26`.
- Parity-Quellen (VM300 `/opt/sandbox-src/workflows`, sha256, S1-Report): confirm-v04 `7bb1d5b4…`, dispatcher `6f7b9415…`, executor `35e5602b…`, intent-extractor `8f88b76f…`, dispatch v1.5 `3198499e…`, u037 `e2354255…`.
- jas-sandbox-Image: `jarvis-action-state:sim-64a2d19d` (Digest-Pin des Sandbox-Builds 0.12.0; altes Pin `sim-029dd8b` hat kein /promotion+/mail → dokumentierter Wechsel).
- Lifeops-Quelle: homelab-docker Checkout `241a47f` (Branch fix/imap-write-ok-contract; jarvis-lifeops dort unverändert zu main).
- Mirror-SHAs (homelab-sim-runs): S1 `cdcaabc`, S2 `c6798fb`, S3 `2dae187`, S4 `85e9972`.

## Invarianten (gesamt)

Isolations-Recheck nach jeder Netz-OP: durchgehend 3× rc=28 Timeout (je Stufe im
Rohbeleg). gitleaks vor jedem Push: 0. `sim.env`/Dummies nie gepusht; Admin-/
Lifeops-Token = dokumentierte Sim-Dummies. Git nur auf LXC 121; VM300 rein
rsync-Ziel. A0 nicht nötig (kein `homelab_*.md` berührt).
