# S2 — Simulierte Freigabe -> tatsaechlicher IMAP-Effekt + Idempotenz (2026-07-09)

Freigabe programmatisch (Confirm-v04-Webhook, `decision=approved`, ntfy=Senke) ->
Executor -> imap-write-adapter -> **tatsaechlicher Effekt** (Dovecot-Counts / Mailpit-Fang).

| Zweig | Aktion | before -> after | Mailpit | Effekt | Executor-Terminal |
|---|---|---|---|---|---|
| werbung | move -1/+1 | {'INBOX': 3, 'INBOX.Newsletter': 0} -> {'INBOX': 2, 'INBOX.Newsletter': 1} | None->None | True | Return: Failed |
| newsletter | move -1/+1 | {'INBOX': 2, 'INBOX.Newsletter': 1} -> {'INBOX': 1, 'INBOX.Newsletter': 2} | None->None | True | Return: Failed |
| beleg | forward->Mailpit | {'INBOX': 1} -> {'INBOX': 1} | 0->1 | True | Return: Failed |

**S2 OK=True** (alle tatsaechlichen IMAP-Effekte verifiziert).
**Idempotenz:** kein Doppel-Move bei Re-Freigabe = True (Decision-Layer 409 auf Re-Approve).
**Whitelist:** Nicht-Whitelist-Ziel `INBOX.Boese` abgelehnt = True (Ordner nie angelegt).

**Befund executor-success-mismatch:** `Adapter Success?` prueft `$json.ok===true`; imap-write liefert `{moved/forwarded:true}` (kein `ok`) -> Executor `Return: Failed` TROTZ eingetretenem Effekt; `Record Idempotency` uebersprungen -> Executor-Layer-Idempotenz greift nicht (nur Decision-Layer schuetzt).
**Befund shared-bearer:** Executor nutzt EINE Bearer-Credential fuer alle Adapter; Sandbox `SIM_IMAP_WRITE_TOKEN` angeglichen (Prod-Paritaet zu pruefen).
