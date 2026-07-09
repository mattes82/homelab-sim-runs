# Sim-Sandbox Run-Report — S2: Microservices + Wiring + Task/Kalender-E2E

- **Datum:** 2026-07-09
- **Stufe:** S2 (von 4)
- **Zone:** VLAN35 / VM300 (192.168.35.60) — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sandbox-staging-p1p2`, Commit `c4b04c4`
- **Status:** ✅ grün

## Was aufgebaut wurde

Fünf Referenz-Microservices **aus Quelle** gebaut (aus `/opt/sandbox-src/homelab-docker`,
per rsync von LXC121 gestagt — `homelab-docker` ist nicht public), per `sim.env` an
Baïkal (S1) verdrahtet. Alle im Netz `sandbox-net`, Ziel-URLs = Container-Namen, KEINE
Prod-IP. Host-Ports nur `127.0.0.1` für die Verifikation.

| Service | Container | Port | Inbound-Token | Anpassung |
|---|---|---|---|---|
| caldav-reader | `sandbox-caldav-reader` | 8090→8080 | `READER_TOKEN` | `NC_CALENDAR_URL=…/dav.php` |
| nextcloud-task-writer | `sandbox-task-writer` | 8094 | `JARVIS_TASK_WRITER_BEARER` | `NC_TASK_CALENDAR_PATH=/dav.php/…/inbox-1` |
| caldav-writer | `sandbox-caldav-writer` | 8100→8080 | `JARVIS_CALDAV_WRITER_TOKEN` | `CALDAV_URL=…/dav.php/calendars/<user>` |
| action-state | `sandbox-action-state` | 8093 | `ACTION_STATE_ADMIN_TOKEN` | SQLite **von /mnt/raid → sandbox-Volume**; `tool-registry.yaml` read-only gemountet |
| jarvis-tools | `sandbox-jarvis-tools` | 8095→8080 | `JARVIS_TOOLS_READER_TOKEN` | Peer-URLs auf sandbox-DNS |

## Verifikation — echter HTTP-Pfad (nicht TestClient)

```
## Health-Matrix
  caldav-reader          /health -> 200
  nextcloud-task-writer  /health -> 200
  caldav-writer          /health -> 200
  action-state           /health -> 200
  jarvis-tools           /health -> 200

## E2E (Referenz-Flows)
  calendar.create -> caldav-writer -> ok=True cal=dienstlich   (Baikal privat/dienstlich)
  task.create     -> task-writer   -> ok=True status=201       (Baikal inbox-1)
  readback caldav-reader: events=2 tasks=1
  undo task.delete-> task-writer   -> ok=True ; tasks_after=0

## Auth fail-closed
  task-writer falscher Bearer -> 403

## Sim-n8n Rewire (OP-S2.3)
  restart-policy=unless-stopped  netz=sandbox-net  healthz=200
  n8n -> sandbox-task-writer:8094/health (by name) -> {"status":"ok"}

## Isolations-Recheck (VM300 -> Prod MUSS Timeout sein)
  n8n .30.43:5678 -> 000 | .30.21 -> 000 | gitea .37:3000 -> 000
```

**Ergebnis:** Voller Round-Trip über echte HTTP-Pfade grün — `calendar.create` und
`nextcloud_task.create` schreiben nach Baïkal, `caldav-reader` liest beide zurück, der
**Undo-Pfad** (`/tasks/delete`) ist im gebauten Image vorhanden (kein 404-Drift) und
entfernt die Aufgabe. Auth fail-closed (403 bei falschem Token). Isolation intakt.

## Fidelity-Gap (kein STOP)

Die Sim-n8n ist auf die Sandbox-Ziele **umverdrahtet und per Container-Namen erreichbar**
(nachgewiesen), aber die eigentlichen Dispatch-Workflow-JSONs (`task`/`calendar`) liegen in
`homelab-jarvis-workflows` (Prod-Repo, in diesem Auftrag nicht mutiert) und sind noch nicht
in die Sim-n8n importiert. Der DoD-relevante Nachweis „echter HTTP-Pfad" ist über die
laufenden Container erbracht; der voll workflow-getriebene n8n-Lauf braucht den Import
dieser Workflows (analog zum bestehenden u037-„rsync-zur-Smoke-Zeit"-Muster).

## Nächste Stufe
S3 — Mail (Dovecot + Mailpit-Senke), imap-Adapter aus Quelle, mail.move/forward-E2E.
