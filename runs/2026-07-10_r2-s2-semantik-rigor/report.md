# R2-S2 — Semantik-Gate rigoros + ehrlich (DR-02/04/05/06) · 2026-07-10

**Branch** `feat/sim-gate-haertung-r2` @f5fda2d. Rohbeleg: `evidence.txt`.

## before → after

| DR | before | after |
|---|---|---|
| DR-04 | `node_sig` = nur (type, typeVersion, onError, params) — credentials/disabled/retryOnFail/maxTries/waitBetweenTries/alwaysOutputData/continueOnFail/settings/staticData unverglichen | **Voll-Vergleich**: ganzer Node außer Ignorable-Allowlist `{id, position, notes, notesInFlow, color}`; Workflow-Rahmen (name/settings/staticData) verglichen. Mutations-Gegenproben credentials/retryOnFail/disabled am ECHTEN Dispatcher-Transformat → je **ROT** (Rohbeleg); Unit-Mutationstests je Feld (7 Felder + Feld-Entfernung + Rahmen) |
| DR-05 | ungestagte Workflows still übersprungen, nur `checked==0` rot | **WORKFLOW_MANIFEST**: alle 6 Workflows `required` (keine optionalen — Optionalität müsste eng begründet deklariert werden), beidseitig Pflicht; Gegenprobe confirm-v04-Transformat entfernt → `required-Workflow fehlt: Transformat` **Exit ≠ 0** |
| DR-06 | `DECLARED_ADDED_NODES` global, IMAP-Strip überall erlaubt, keine Signatur-/Kanten-Bindung | **DECLARED_INJECTIONS pro Workflow**: exakte Node-Signaturen (Literal-Deklaration als Review-Anker), exakte Anzahl, exakt deklarierte Kanten, keine Fremd-Referenzen auf Sim-Nodes; emailReadImap-Strip nur im Dispatch-Workflow und dort vollständig. Tests: Sim-Node in fremdem Workflow / falsche Signatur / fehlende deklarierte Node / abweichende Kante → je rot |
| DR-02 | `unmask_confirm_map` demaskierte die SIM-ONLY confirm-adapterMap im Gate-Pfad → Divergenz unsichtbar, „GRÜN 6 WF" | `unmask_confirm_map` **entfernt**. `GATE_MODE=parity` (Default): Map **ehrlich rot** als deklarierte Prod-Lücke, **Exit 3**; jede weitere Divergenz Exit 1 (Abbruchkriterium-Wächter). `GATE_MODE=harness`: exakt diese Injektion akzeptiert, Ausgabe **„NICHT promotion-fähig"**. Real belegt: Parity = EHRLICH ROT mit exakt 1 Lücke, **0 undeklarierte** — keine weitere Divergenz aufgedeckt (kein STOP-Fall) |

## Modus-Architektur (Entscheidung, review-relevant)

`make semantic-gate` (Standardpfad, Dependency von `verify-offline`) = **Harness-Lauf
(hart)** + **Parity-Lauf**: Parity-Exit 3 (= ehrlich rote, deklarierte Lücke) bricht den
Standardlauf nicht, wird aber in jedem Lauf gemeldet („NICHT promotion-fähig"); jeder
andere Parity-Fehler bricht. `make semantic-gate-parity` = promotionsrelevante Aussage,
**Exit ≠ 0 bis der Prod-Fix die Map ergänzt** (separater Track). Damit bleibt der
Standardlauf als Regressionsgate nutzbar UND die Prod-Divergenz in jedem Lauf sichtbar —
„alles grün" wird nicht mehr vorgetäuscht.

## Ignorable-Allowlist (DR-04, deklariert)

Node-Ebene: `id`, `position`, `notes`, `notesInFlow`, `color` (technische ID + Canvas/UI).
Workflow-Ebene: keine (kanonische `id` beidseitig identisch normalisiert).
Endpoint-Kanonisierung (URL_REMAP → Platzhalter) bleibt der deklarierte
Endpoint-Injektionspunkt.

## Required-Workflow-Manifest (DR-05)

`jarvis-action-executor-v1.1`, `jarvis-intent-extractor-v1`,
`jarvis-mail-klassifizierung-dispatch-matthias-v1.5`, `u037-intent-to-proposal-v01`,
`jarvis-action-dispatcher`, `jarvis-action-confirm-v04` — **alle required**.
Manifest ist mit `transform.IDMAP` assert-gekoppelt (kein still unklassifizierter Workflow).

## Kein Regress #20/#21 (Harness-Modus, VM300)

Transformate auf VM300 regeneriert + alle 6 re-importiert (inkl. Aktivierung + Restart):
- **#20 Routing: 8/8 PASS** (s3run, Switch-Matrix).
- **#21 Effekt-Matrix: `[('werbung', True), ('newsletter', True), ('beleg', True)]`** — unverändert; idem (kein Doppel-Move, Key gebrannt) und whitelist-Reject grün.

## Invarianten

Isolations-Recheck VM300 → .30.43:5678/.30.21:80/.30.37:3000 = **3× rc=28 Timeout**.
gitleaks staged: 0. Unit-Suite: 56 Tests grün (20 Gate-Tests). `make verify-offline`
Ende-zu-Ende grün mit sichtbarer EHRLICH-ROT-Meldung des Parity-Laufs.
