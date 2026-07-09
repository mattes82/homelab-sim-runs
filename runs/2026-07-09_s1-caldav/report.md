# Sim-Sandbox Run-Report — S1: Compute-Basis + CalDAV (Baïkal)

- **Datum:** 2026-07-09
- **Stufe:** S1 (von 4) — Commission „Sim-Sandbox-Aufbau P1+P2"
- **Zone:** VLAN35 / VM300 (192.168.35.60), Hostname `jarvis-sim` — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sandbox-staging-p1p2`, Commit `8d99d3a`
- **Status:** ✅ grün

## Was aufgebaut wurde

Produktions-**geformter**, nicht produktions-**verbundener** CalDAV-Server als Basis der
Referenz-Flows (Task/Kalender in S2).

| Komponente | Wert |
|---|---|
| Image | `ckulka/baikal:0.10.1-nginx` |
| Pin (Digest, verifiziert 2026-07-09) | `sha256:434bdd162247cc6aa6f878c9b4dce6216e39e79526b980453b13812d5f8ebf4b` |
| Backend | SQLite (Bind-Mount `sandbox/caldav/data/`, gitignored) |
| Auth | `dav_auth_type: Basic` (die CalDAV-Adapter nutzen HTTP-Basic) |
| Host-Bind | `127.0.0.1:8800` → nginx:80 (keine externe Exposition) |
| Netz | `sandbox-net` (extern, Container-Name `sandbox-baikal`) |
| Zugang | Wegwerf-Dummy aus `sim.env` (schützt nichts Reales; NIE gepusht) |

Web-Installer wird übersprungen (generierte `baikal.yaml` mit `configured_version: 0.10.1`).
Seed idempotent: `baikal.yaml` + SQLite-User/Principal (`digesta1 = md5(user:realm:pass)`)
+ Kalender via **echtem MKCALENDAR** über den HTTP-DAV-Pfad.

## Geseedete Kalender (synthetisch)

| Slug | Anzeigename | Zweck |
|---|---|---|
| `privat` / `familie` / `dienstlich` / `homelab` | Privat/Familie/Dienstlich/HomeLab | `calendar.create` (VEVENT) |
| `inbox-1` | Aufgaben | Tasks-Collection (`nextcloud_task.create`, VTODO) |

## Pfad-Mapping (Prod ↔ Sandbox)

| | Produktion (Nextcloud) | Sandbox (Baïkal) |
|---|---|---|
| DAV-Root | `/remote.php/dav` | `/dav.php` |
| Tasks | `/remote.php/dav/calendars/MATTES/inbox-1/` | `/dav.php/calendars/<sim-user>/inbox-1/` |

## Verifikation (OP-S1.4) — echter HTTP-Pfad

```
## Container
sandbox-baikal | Up | image=ckulka/baikal:0.10.1-nginx
image_pin=ckulka/baikal:0.10.1-nginx@sha256:434bdd162247cc6aa6f878c9b4dce6216e39e79526b980453b13812d5f8ebf4b

## Health / Auth (echter HTTP-Pfad)
  GET /                        -> 200
  PROPFIND dav.php (auth)      -> 207   [207 = Multi-Status, Basic-Auth ok]
  PROPFIND dav.php (falsch)    -> 401   [401 = fail-closed]

## Geseedete Kalender (CalDAV PROPFIND Depth:1)
  - Aufgaben  - Dienstlich  - Familie  - HomeLab  - Privat

## Isolations-Recheck (VM300 -> Prod MUSS Timeout sein)
  n8n  192.168.30.43:5678 -> 000
  host 192.168.30.21      -> 000
  gitea 192.168.30.37:3000-> 000   [000 = unreachable/DROP]
```

**Isolations-Invariante intakt:** VM300 erreicht keine Prod-IP (alle `000`/Timeout).
Basic-Auth fail-closed (falsches Passwort → 401). Alle 5 synthetischen Kalender via
CalDAV auflistbar.

> Roh-Evidenz: [`evidence.txt`](evidence.txt). Wegwerf-Dummy-Passwörter sind **nicht**
> im Report enthalten (`sim.env` bleibt VM300-only).

## Nächste Stufe

S2 — Microservices (caldav-reader / nextcloud-task-writer / caldav-writer / action-state /
jarvis-tools) aus Quelle bauen, per `sim.env` an Baïkal verdrahten, Sim-n8n umverdrahten,
Task/Kalender-E2E.
