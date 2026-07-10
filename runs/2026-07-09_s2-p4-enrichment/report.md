# S2 — P4: Proposal from/subject-Enrichment Fix (Fix-Verify-Promote)

**Fix:** `homelab-jarvis-workflows` `fix/imapwrite-proposal-enrichment` @3134fcd — Dispatch-v1.5
Nodes `Move zu Newsletter`/`Forward zu Paperless` reichern `imap_write`-Proposals um
`from`(=`from_domain`, privacy-sicher)/`subject` an. typeVersion 4.2 unverändert.
**PR homelab-jarvis-workflows #40** (kein Auto-Merge).

## before → after (VM300, CLI-Re-Import typeVersion-treu)
| Zweig | before Payload | after Payload |
|---|---|---|
| werbung | account_id/uid/source/target | + `from=onlineshop-xyz.de`, `subject="Sonderangebot Aktion…"` |
| newsletter | account_id/uid/source/target | + `from=heise-medien-xyz.de`, `subject="Newsletter: …"` |
| beleg (forward) | account_id/uid/source | + `from=stromanbieter.de`, `subject="Ihre Rechnung Nr. 2026-4711"` |

`issue1_reproduced_on = []` (nach Fix). **Kein Regress**: Effekt-Matrix #21 unverändert,
Executor `Return: Executed` (P1-Fix koexistiert). Isolations-Recheck: Prod-IPs Timeout.

## Fidelity-Limit
Sandbox-Dispatch Webhook-getriggert (IMAP-Trigger gestrippt, #20); Fixture-Input trägt
`from`/`subject` wie Prods Mail-Objekt → `Mail minimieren` extrahiert sie. Echter IMAP-
Polling-Pfad bleibt SHADOW-verifiziert. `from` = Absender-**Domain** (privacy-Stance;
volle Adresse absichtlich nicht gespeichert). Forward-Zweig: Klassifikator-Route dead (P3),
Terminal-Kette via Direkt-Dispatch, der den gefixten Node spiegelt (sim PR #22).

Beleg: `s1-after.json` (from/subject je Zweig, issue1=[]).
