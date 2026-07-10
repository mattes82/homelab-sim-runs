# S3 — Mail-Terminal-Gap Gesamt (2026-07-09)

Volle Mail-Aktions-Schleife E2E in der Sandbox geschlossen: Seed -> Dispatch ->
Proposal -> simulierte Freigabe -> Executor -> imap-write-adapter -> **tatsaechlicher
IMAP-Effekt** (Dovecot/Mailpit) + Idempotenz + Whitelist-Reject.

## Effekt-Matrix
| Zweig | Aktion | Effekt verifiziert |
|---|---|---|
| werbung | imap_write.move -> INBOX.Newsletter | ja (INBOX -1 / Newsletter +1) |
| newsletter | imap_write.move -> INBOX.Newsletter | ja (INBOX -1 / Newsletter +1) |
| beleg | imap_write.forward -> paperless@sim.local | ja (Mailpit-Fang, nie extern) |

Proposal->Freigabe->Effekt je Zweig gruen; Idempotenz bestaetigt (kein Doppel-Move);
Whitelist-Reject bestaetigt. Isolations-Recheck nach jeder Netz-OP: alle Prod-IPs Timeout.

## Befunde
- **issue-1**: imap_write-Proposals ohne from/subject — Move braucht uid/folder (vorhanden) — from/subject sind Display-/Kontext-Enrichment, fehlen im Payload. Blockiert die Ausfuehrung NICHT.
- **forward-branch-dead**: Forward-zu-Paperless-Zweig in v1.5 unerreichbar — Paperless Gate IF (paperless_candidate==true) -> Nextcloud Task Dokument; Switch-Rechnung-Zweig re-gated 'Paperless Gate' erneut auf true, das nach dem IF nur false sein kann -> Dead Branch. Terminal-Kette via exaktem Forward-Dispatch bewiesen (wie 'Forward zu Paperless' ihn absetzt).
- **shared-adapter-bearer**: Executor nutzt eine geteilte Bearer-Credential fuer alle Adapter — 'Jarvis Task Writer Bearer' fuer task-writer UND imap-write. Sandbox: SIM_IMAP_WRITE_TOKEN an den Task-Writer-Bearer angeglichen. Prod-Paritaet verifizieren, sonst Executor->imap-write 401 auch in Prod.
- **executor-success-mismatch**: Executor wertet imap-write-Erfolg als Fehler — 'Adapter Success?' prueft $json.ok===true; imap-write liefert {moved:true}/{forwarded:true} (kein 'ok') -> Return: Failed TROTZ eingetretenem IMAP-Effekt. Folge: 'Record Idempotency' (Key-Verbrennung) uebersprungen -> Executor-Layer-Idempotenz greift NICHT; Doppel-Schutz nur ueber Decision-Layer (409 auf Re-Approve). from/subject-Enrichment wuerde das nicht heilen.
- **proposal-full-absent**: /proposals/{id}/full im Sandbox-action-state-Build nicht vorhanden — Spaeterer Commission-Stand nicht in die Sandbox gebaut — Ausfuehrungsnachweis hier ueber Executor-Terminal-Node statt /full.

## Overhead / Freshness
- Overhead-Delta: **0 neue Container** (mail-terminal nutzt den #19/#20-Stack, 13 Container); Mem ~1.6Gi/7.8Gi.
- Quell-SHAs (Freshness-Praxis): siehe `source-shas.txt` — `homelab-docker@b6a520b4`, `workflows@e9470b4f`; VM300-Staging traegt kein `.git` (Freshness-Gap, motiviert die Praxis).

## Rest-Gaps
- IMAP-Trigger-Polling weiterhin API-getriggert (wie #20).
- `/proposals/{id}/full` im Sandbox-action-state-Build nicht vorhanden.
- Forward-Klassifikator-Zweig unerreichbar (Dead Branch) — Terminal-Kette via Direkt-Dispatch bewiesen.
