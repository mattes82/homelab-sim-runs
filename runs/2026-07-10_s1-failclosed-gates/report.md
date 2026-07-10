# S1 — Fail-closed-Gates (REV-001/002) · 2026-07-10

**PR-A** `feat/sim-trust-gates` @8d1aab4. Rohbeleg: `evidence.txt`.

| Befund | before | after |
|---|---|---|
| REV-001 `make replay` | `-$(PYTHON) -m runner.replay` schluckt Exitcode → Recipe „erfolgreich" trotz Fehler | Wrapper `sandbox/run_replay.sh`: Replay-Exitcode durchgereicht, Teardown via `trap`; **run_replay.sh Exit 1 bei Replay-Fail** (belegt) |
| REV-002 `verify-offline` | `pass:0 · pending:35 · skipped:80 · oos:20` → Exit 0 „GRUEN" | fail-closed `pflichtsuite`: leere/kaputte Suite → **ROT**; runtime-Gate 0 Pass → **ROT**; ehrliche Aussage „STRUKTUR-GRUEN … Verhalten via make replay" |

**Pflichtsuite** (`runner/pflichtsuite.py`): runtime-Pflichtkategorien = quick_confirm/undo/failure/promotion/mail_action/relevance (je ≥1 echter Pass; pending/skipped in Pflichtkategorie = Fail). Composer/Scaffolds (tageslage/open_loops/aging) = Inventar-Pflicht, nicht Verhaltens-Pflicht (dokumentiert). 8 Unit-Tests.
