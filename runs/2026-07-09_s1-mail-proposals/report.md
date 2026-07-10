# S1 — Konsistenter Seed + Dispatch -> Proposal (2026-07-09)

Sandbox VLAN35/VM300. Je Mail-Zweig eine Mail konsistent in Dovecot INBOX **und**
als IMAP-roh-geformter Dispatch-Input (gleiche UID/Message-ID) -> Klassifikator
v1.5 -> Proposal in action-state.

| Zweig | via | Proposal-Typ | Route | UID-Match | Idem-Key | Issue#1 (from/subj fehlt) |
|---|---|---|---|---|---|---|
| werbung | classifier | imap_write.move | confirm | True | True | True |
| newsletter | classifier | imap_write.move | confirm | True | True | True |
| beleg | direct_forward_dispatch | imap_write.forward | confirm | True | True | True |

**S1 OK=True.** Issue#1 (imap_write-Proposals ohne from/subject) reproduziert auf: ['werbung', 'newsletter', 'beleg'] — Move braucht uid/folder, **blockiert Execution nicht**.
Befund: der Forward-Zweig des Klassifikators ist in v1.5 unerreichbar (Dead Branch) -> Terminal-Kette via exaktem `Forward zu Paperless`-Dispatch bewiesen. Isolations-Recheck: alle Prod-IPs Timeout.
