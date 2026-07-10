# S1 — P1: imap-write `ok`-Contract Fix (Fix-Verify-Promote)

**Fix:** `homelab-docker` `fix/imap-write-ok-contract` @241a47f — imap-write-Adapter liefert
bei `move`/`forward`-Erfolg **additiv `ok:true`** (Contract wie task-writer `/tasks/create`).
**PR homelab-docker #55** (kein Auto-Merge).

## Contract-Feststellung (Phase 0)
Executor nutzt EINEN generischen Node `Adapter Success?` (`$json.ok===true`) für alle Adapter.
task-writer `/tasks/create` liefert `{ok: ...}` → Contract; imap-write lieferte nur
`{moved/forwarded:true}` → **Ausreißer**. Fix = angleichen (bestätigt, kein STOP).

## before → after (VM300, echter HTTP-Pfad, Mail-Terminal-E2E)
| Signal | before (current-main) | after (Fix) |
|---|---|---|
| Adapter-Response move/forward | `{moved/forwarded:true}` (kein `ok`) | `{ok:true, moved/forwarded:true}` |
| Executor-Terminal-Node (3 Zweige) | `Return: Failed` | **`Return: Executed`** |
| `Record Idempotency` | übersprungen | **läuft** |
| `/idempotency/check` (Move-Key) | `exists:false` (Guard inert) | **`exists:true, status:executed`** (Guard armiert) |
| Effekt-Matrix #21 | move −1/+1, forward→Mailpit | **unverändert** (kein Regress) |

## Zweischichtige Idempotenz nachgewiesen
- **Decision-Layer:** Re-Approve → 409 „proposal is not pending: current status is approved".
- **Executor-Layer:** Idempotenz-Key gebrannt (`status:executed`) → `Check Idempotency` → `Already Executed?` → `Return: Skipped` bei jedem Re-Execute (vorher inert).
- kein Doppel-Move bestätigt; Whitelist-Reject intakt; Isolations-Recheck: alle Prod-IPs Timeout.

Belege: `s2-after.json` (Executor `Return: Executed`, effect true), `idem-after.json` (beide Layer).
