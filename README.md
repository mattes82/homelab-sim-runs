# homelab-sim-runs

Public, **sanitierter** Review-Mirror für Sim-Sandbox-Läufe (VLAN35 / VM300).
Nur Run-Artefakte — **kein Code, keine Creds**. Jeder Push ist `gitleaks`-geprüft
(0 Findings Pflicht); `sim.env`/Wegwerf-Dummies werden NIE gepusht.

## Struktur

```
runs/<datum>_<stufe>/report.md     Menschlicher Report je Stufe
runs/<datum>_<stufe>/evidence.txt  Roh-Evidenz (curl/HTTP-Codes, sanitiert)
```

## Läufe (Sim-Sandbox-Aufbau P1+P2, 2026-07-09)

- `2026-07-09_s1-caldav`      — Baïkal/CalDAV
- `2026-07-09_s2-tasks-kalender` — Microservices + Wiring + Task/Kalender-E2E
- `2026-07-09_s3-mail`        — Dovecot + Mailpit-Senke, mail.move/forward-E2E
- `2026-07-09_s4-gesamt`      — Gesamtstand, Health-Matrix, Overhead, Reset-Mechanik
