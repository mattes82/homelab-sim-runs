# S3 — Datenklassen-Kanon (REV-003, korrigiert) · 2026-07-10

**PR-B** `feat/sim-dataclass-canon` @27b2e5e (gestapelt auf PR-A). Rohbeleg: `evidence.txt`.

**Phase-0-STOP aufgeloest:** v1.0-Praemisse „Klasse A absolut gesperrt" war **falsch** (an `homelab_jarvis.md` §2.5/§6.7 widerlegt). Verifizierter Kanon:
| Klasse | routing_policy | Intake |
|---|---|---|
| **C** (Secrets/Tokens/vertraulich-dienstlich) | `blocked` | **Hard-Block** (nie Modellinput/Cloud/Persistenz) |
| **A** (privat/personenbezogen) | `local_only` | **nicht geblockt** (lokal verarbeitbar; nie Cloud, auch degraded §2.5 Z.503) |
| **B** (technisch) | `cloud_allowed` | — |

- **before:** Klasse C nur implizit `out_of_scope`; „A absolut gesperrt" war die (falsche) Zielannahme.
- **after:** `runner/dataclass_gate.py` kodifiziert den Kanon; `replay.py` blockt C **explizit am Intake** (Status `blocked`); `out_of_scope` 20→12 (8 C-Faelle jetzt `blocked`). A bleibt local_only.
- **9 Negativtests:** C→blocked; A→local_only (nicht geblockt); no-cloud fuer A/C **inkl. degraded**; Roh-A/C nie in Persistenz/Log/Audit; E2E class_c am Intake geblockt.
- **Daten-Hygiene:** Fixtures synthetisch-only; echte PII-A/Secret = Tripwire — **ohne** A zu blocken.
