# R2-S4 — Wrapper fail-fast, blocked-Reporting, Hygiene (DR-10/11/13) · 2026-07-11

**Branch** `feat/sim-gate-haertung-r2` @d045b26. Rohbeleg: `evidence.txt`.

## before → after

| DR | before | after |
|---|---|---|
| DR-10 | `run_replay.sh`: `$UP_CMD` frei geparst und UNGEPRÜFT (kein `set -e`), `trap teardown` erst NACH Up+Mock-Start, keinerlei Readiness-Checks | trap (EXIT INT TERM) **vor** dem Startpfad; jeder Schritt explizit geprüft: sandbox_up → Exit 2, Mock-Readiness (Health-Poll 30 s) + Prozess-Liveness (`kill -0`) → Exit 3, Lifeops → Exit 4; Overrides nur als dokumentierte `bash -c`-Hooks. **Gegenprobe `REPLAY_UP_CMD=false` → Exit 2, teardown lief via trap** (Rohbeleg) |
| DR-09-Minimal | relevance-Pflichtfälle skippten still („lifeops not reachable") | `sandbox-lifeops` als Sandbox-Service (compose, Build aus `/opt/sandbox-src/homelab-docker/jarvis-lifeops` @241a47f, Sim-Dummy-Token, Volume statt `/mnt/raid`); `run_replay.sh` startet + health-checkt ihn, **kein stilles Überspringen** (Exit 4). Volle Replay-Profil-Aufteilung bleibt bewusst Lifecycle-Arbeit |
| DR-11 | `write_summary` hartkodierte 5 Status — `blocked` fehlte in der Kopfzeile, `total` zählte ihn (135 ≠ 127 sichtbar) | Summen **dynamisch über alle vorkommenden Status** + Assertion `Summe == total`; zusammen mit S3 (Intake-Block = beobachteter pass) existiert kein Sonderstatus mehr. Beleg: `89+0+35+0+16 = 140` exakt |
| DR-13 | nackte `open(...)` in runner/transform/mtrun/s3run/tests → ResourceWarnings | `Path.read_text/write_text` bzw. `with open`; `unit-test`-Target läuft mit `-W error::ResourceWarning` — Warnungen sind Gate-Fehler. 62 Tests grün unter der Regel |
| Nebenentscheid | `classification_unclear` = `skipped` (needs-runtime) → hielt die runtime-Pflichtsuite strukturell dauerhaft rot | ehrlich **out-of-seam** (`externe Fault-Injektion (manuell, SIM-VM)`, OP-9-Mechanik): in-code nicht stellbar; `skipped` behält die scharfe Semantik „hätte laufen müssen" |

## Keystone-Beleg: erster grüner `make replay`

VM300, `JAS_IMAGE_TAG=jarvis-action-state:sim-64a2d19d`, frische Sandbox-DB, Lifeops
frisch gebaut+gestartet: **Exit 0** — 140 Fälle: **89 pass · 0 fail · 35 pending ·
0 skipped · 16 oos** (Summe == total), runtime-Pflichtsuite **GRÜN**
(6 Pflichtkategorien, 12 Pflicht-Szenarien / 27 Pflichtfälle), relevance 14/14
über `sandbox-lifeops` (healthy). Vorher gab es keinen grünen Runtime-Lauf —
und der jetzige ist fail-closed verdient, nicht erschlichen: jede Stufe hat
belegte Gegenproben, die ihn brechen.

## Invarianten

Isolations-Recheck VM300 → 3× rc=28 Timeout (Rohbeleg). gitleaks staged: 0.
Lifeops-Token/Admin-Token = dokumentierte Sim-Dummies, kein Prod-Secret;
sim.env unangetastet.
