# S2 — Transform-Fidelity (REV-004) · 2026-07-10

**PR-A** `feat/sim-trust-gates` @8d1aab4. Rohbeleg: `evidence.txt`.

- **before:** Transform setzte periphere Nodes auf `onError=continueRegularOutput` (Fehlersemantik-Aenderung) + IMAP-Strip + SIM-Adapter-Mapping — unkontrolliert.
- **after:** `mark_peripheral`/`continueRegularOutput` **entfernt** → Prod-Fehlersemantik erhalten. **Deklarationsliste** (url-remap, kanon. id, IMAP→Webhook+Sim-Entries, confirm-adapterMap) in `transform.py`.
- **Semantik-Hash-Gate** (`semantic_gate.py`, `make semantic-gate`): ziel-basierte Endpoint-Kanonisierung, deklarierte Injektionen maskiert, erzwingt Node/**typeVersion**/**onError**/params/connections-Paritaet → undeklarierte Abweichung = ROT. **GRUEN fuer 6 Workflows.** 6 Negativtests (onError/typeVersion/param/added-node fallen auf).
- **Kein Regress (VM300):** #20 Routing **8/8**, #21 Effekt-Matrix unveraendert; Isolation alle Prod-IPs Timeout.

## Deklarationsliste (legitime Transform-Aenderungen)
1. **url-remap** — Prod-Host-URLs → Sandbox-Container (mehrere Prod-Hosts → 1 Stub zulaessig).
2. **canonical-id** — kanonische n8n-id fuer Sub-Ref-Aufloesung.
3. **imap→webhook** — emailReadImap entfernt + Sim-Entries (Sandbox nicht IMAP-erreichbar). SHADOW schliesst den IMAP-Polling-Pfad an Prod.
4. **confirm-map** — v04 adapterMap SIM-ONLY um imap_write erweitert.
