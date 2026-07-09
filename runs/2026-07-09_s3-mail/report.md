# Sim-Sandbox Run-Report — S3: Mail (Dovecot + Mailpit-Senke)

- **Datum:** 2026-07-09
- **Stufe:** S3 (von 4)
- **Zone:** VLAN35 / VM300 (192.168.35.60) — **nicht** Prod
- **Repo (Code):** `homelab-jarvis-sim` Branch `feat/sandbox-staging-p1p2`, Commit `c0c8c07`
- **Status:** ✅ grün

## Was aufgebaut wurde

| Komponente | Pin (Tag + Digest, verifiziert 2026-07-09) |
|---|---|
| Dovecot (IMAP/imaps 993) | `dovecot/dovecot:2.4.4@sha256:723e3392…89b2d4` |
| Mailpit (SMTP-Senke) | `axllent/mailpit:v1.30.4@sha256:5a49a77c…82d4f6` |
| imap-write-adapter / imap-backfill-reader(+ -m) | aus Quelle gebaut (`/opt/sandbox-src`) |

- **Dovecot:** `passdb static` (ein gemeinsames Wegwerf-Passwort), Maildir, self-signed
  Zert (imaps/993 — der Adapter nutzt `IMAP4_SSL`). Explizite imap/imaps-Listener +
  Namespace-Separator `.` (Prod-geformte Ordner `INBOX.Newsletter`, `Archiv.2025`).
- **Mailpit** = reine **Senke**: STARTTLS + `--smtp-auth-accept-any` (der Adapter macht
  `smtp.starttls()`+`login()`), fängt ausgehende Mail, **relayed NIE** nach draußen.
- Konten **matthias** und **m** auf Sandbox-Postfächer gemappt.
- Seed: Ordner `INBOX/INBOX.Newsletter/INBOX.Werbung/Archiv.2025` + synthetische Mails.

## Verifikation — echter HTTP-Pfad

```
## Health-Matrix
  imap-write-adapter       /health -> 200
  imap-backfill-reader     /health -> 200
  imap-backfill-reader-m   /health -> 200
  mailpit                  SMTP-Senke API -> 200

## IMAP-Read-Wiring: imap-backfill-reader /mail/mailboxes (matthias)
  [Archiv.2025, Archiv, INBOX.Werbung, INBOX.Newsletter, INBOX, ...]

## E2E mail.move (Whitelist-Ziel) matthias INBOX uid1 -> INBOX.Newsletter
  before INBOX=3 Newsletter=1
  resp   {"moved":true, ...}
  after  INBOX=2 Newsletter=2

## E2E mail.move NICHT-Whitelist -> HTTP 400 (abgelehnt)

## E2E mail.forward matthias INBOX uid2 -> Mailpit (gefangen, NICHT versandt)
  resp   {"forwarded":true,"to":"paperless@sim.local"}
  Mailpit total=1  from=forwarder@sim.local to=[paperless@sim.local] subj=Sonderangebot Aktion

## Auth fail-closed
  imap-write falscher Bearer -> 401

## Isolations-Recheck (VM300 -> Prod)
  n8n .30.43:5678 -> 000 | .30.21 -> 000 | gitea .37:3000 -> 000
```

**Ergebnis:** `mail.move` verschiebt in Whitelist-Ordner (in Dovecot verifiziert:
Quelle −1 / Ziel +1), Nicht-Whitelist-Ziel wird mit **400** abgelehnt, `mail.forward`
wird **in Mailpit gefangen** (kein externer Versand), **kein DELETE** (Adapter hat keinen).
Auth fail-closed (401). Isolation intakt.

## Bau-Notizen (in der Sim-Schleife behoben, kein STOP)

1. Dovecot 2.4.4-Minimal-Image aktiviert per Default **keine** inet_listener → imap/imaps
   explizit gesetzt (`10-sandbox.conf`).
2. Maildir-Seed brauchte `docker exec -i` (ohne `-i` leere Mails, `size=0`).
3. `SMTP_USER` dient als Envelope-From → muss gültige RFC-5321-Adresse sein
   (`forwarder@sim.local`), sonst Mailpit-553.

## Nächste Stufe
S4 — Review-Mirror-Finalisierung, `make sim-seed`/`sim-reset`, Overhead-Messung.
