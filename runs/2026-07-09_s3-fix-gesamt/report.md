# S3 — Fix-Verify-Promote P1+P4 Gesamt (2026-07-09/10)

Zwei in homelab-jarvis-sim#21 gefundene Prod-Bugs gefixt + in der Sandbox über den
echten HTTP-Pfad verifiziert, **ohne Regression** der #21-Effekt-Matrix. Prod-PRs offen,
**kein Auto-Merge** (Promotion = Mattes).

## Ergebnis
| Bug | Fix | before → after | PR |
|---|---|---|---|
| **P1** executor-success-mismatch | imap-write `ok:true` additiv | Executor `Return: Failed`→`Return: Executed`; Idempotenz-Key `exists:false`→`exists:true(executed)` → **zweischichtige Idempotenz** | homelab-docker **#55** |
| **P4** issue-1 (from/subject) | Dispatch-v1.5 Payload-Enrichment | Payload ohne→mit `from`/`subject` (alle 3 Zweige); `issue1=[]` | homelab-jarvis-workflows **#40** |

## Kein-Regress-Nachweis (#21-Effekt-Matrix, nach beiden Fixes)
| Zweig | Aktion | Effekt | Executor |
|---|---|---|---|
| werbung | move → INBOX.Newsletter | −1/+1 ✓ | Return: Executed |
| newsletter | move → INBOX.Newsletter | −1/+1 ✓ | Return: Executed |
| beleg | forward → paperless@sim.local | Mailpit-Fang ✓ | Return: Executed |
Idempotenz kein Doppel-Move; Whitelist-Reject intakt; Isolation alle Prod-IPs Timeout.

## Scope-Grenzen / Fidelity
- **P2** (shared-adapter-bearer) + **P3** (forward-branch-dead) NICHT in diesem Auftrag (Mattes-Entscheidung).
- Forward via Direkt-Dispatch (P3), spiegelt den gefixten Node (sim PR #22).
- IMAP-Polling-Pfad SHADOW; `from`=Absender-Domain (privacy).

## Quell-SHAs / Overhead
- siehe `source-shas.txt`. Overhead: **0 neue Container** (#19/#20-Stack, 13); imap-write-Container rebuilt (Fix-Quelle).
