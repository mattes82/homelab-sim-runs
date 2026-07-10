# R2-S1 — Gate-Integration in den Standardpfad (DR-08) · 2026-07-10

**Branch** `feat/sim-gate-haertung-r2` @79345a5 (Basis: main @6b91fe4 = #23+#24+#25).
Rohbeleg: `evidence.txt`. Commission: `sim-gate-haertung-r2` (R2).

## Befund before → after

| Punkt | before | after |
|---|---|---|
| DR-08 Standardpfad | `make verify-offline` = nur `runner.verify_offline` (führt allein `test_eval_sink.py` aus, 9 Tests); `semantic-gate` separates Target ohne Aufrufer → neue Gates (test_semantic_gate/test_pflichtsuite/test_dataclass_canon/test_confirm_v04) laufen NIRGENDS im Standardlauf | `verify-offline: unit-test semantic-gate` — voller Discover (42 Tests) + Semantik-Gate laufen VOR verify_offline; jedes Rot bricht den Lauf |
| OP-S1.2 Staging | Quellen nicht gestaged → je Datei stilles `uebersprungen`, erst `checked==0` rot (Meldung generisch) | fehlendes/leeres SRC → **`[gate] ROT parity-not-staged`**, Exit 1 (belegt: `SEMGATE_SRC=` → rot) |
| Nebenbefund pytest | `tests/test_confirm_v04_workflows.py` importiert pytest (nicht in requirements) → unter `unittest discover` ImportError, Suite rot auf gesundem Baum | auf stdlib-unittest konvertiert (SkipTest statt pytest.skip, TestCase-Klassen; Assertions unverändert) — 42/42 grün |
| Nebenbefund stale Transformate | committete `dispatch-import/workflows/` waren pre-REV-004 (onError-Mutationen `continueRegularOutput` auf 16 Nodes, fehlendes P4-Enrichment `Forward zu Paperless`/`Move zu Newsletter`); confirm-v04-Transformat nie committet → Gate gegen frische Quellen: **18 undeklarierte Abweichungen** | aus gestagten Quellen deterministisch regeneriert (executor/intent-extractor byte-identisch zum Regenerat → Determinismus-Beleg), confirm-v04 erstmals committet → Gate **GRÜN 6/6 Workflows** |

Der Nebenbefund ist exakt die von DR-08 vorhergesagte Drift: weil das Gate nicht
im Standardpfad lag, alterten die committeten Transformate unbemerkt.

## Gegenproben (OP-S1.3, Rohbeleg in evidence.txt)

1. absichtlich gebrochener Unit-Test (`tests/test_r2_probe_broken.py`) → `make verify-offline` **EXIT=2** am `unit-test`-Target.
2. absichtlich undeklarierte Semantik-Abweichung (onError-Mutation `Push: ntfy Approval` im Dispatcher-Transformat) → **EXIT=2** am `semantic-gate` (`1 undeklarierte Abweichung`). Restore = deterministisches Regenerat (diff-identisch).
3. `SEMGATE_SRC=` (nichts gestaged) → **`ROT parity-not-staged`**, EXIT=2.
4. gesunder Baum: `make verify-offline` **EXIT=0** (42 Unit-Tests OK → Gate GRÜN 6/6 → pflichtsuite:offline GRÜN).

## Parity-Staging

Lokales Staging `.parity-src/workflows` (gitignored), Quelle VM300
`/opt/sandbox-src/workflows` (rsync). sha256 der 6 Quellen:

```
7bb1d5b47419ffc4bd4d97a8e541b80838f2008c57e294dc68ea0c983eea304d  jarvis-action-confirm-v04.json
6f7b9415e046757b5ad5715b51cdf1ce6ef405e711d13f4bb87f426001bb61cd  jarvis-action-dispatcher.json
35e5602b997017e37372189c28b1d69e37f618925d45108fe86d3516d18f6532  jarvis-action-executor-v1.1.json
8f88b76fc72272453343df576bc740eca119b6fd18bff1afdadecb5b1ecb4f25  jarvis-intent-extractor-v1.json
3198499e256231c5cb7b5f71f466029049a9f732e0c4d27b6b3da285135b0c4d  jarvis-mail-klassifizierung-dispatch-matthias-v1.5.json
e23542555bbd843f4e226663630d0f64431da08d959923696752efc33340095a  u037-intent-to-proposal-v01.json
```

## Invarianten

Isolations-Recheck VM300 → 192.168.30.43:5678 / .30.21:80 / .30.37:3000 =
**3× curl rc=28 (Timeout)** (Rohbeleg). gitleaks (staged + main..HEAD): 0 Leaks.
Kein `sim.env`/Dummy gepusht; `.parity-src/` gitignored. Git nur auf LXC 121.
