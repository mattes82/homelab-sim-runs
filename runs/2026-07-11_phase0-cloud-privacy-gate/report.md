# Phase-0-Report — Cloud-Privacy-Gate vor „LLM-Bestätigung" (Dispatch v1.5): **STOP nach Abbruchkriterium**

**Datum:** 2026-07-11 · **Commission:** `homelab-docs-tooling/commissions/inbox/homelab-jarvis-workflows/2026-07-11_commission_dispatch-cloud-privacy-gate.md` (Ablage-PR docs-tooling#66, GEMERGT @9c073af)
**Block 0:** Punkt 1 anfangs nicht erfüllt (sim#26 OPEN) → gemeldet; Mattes hat #26 gemergt (sim main @e4d98d7). Punkt 2 erfüllt.
**Quell-SHAs:** workflows origin/main **579633c** · docker origin/main **002acea** · sim main **e4d98d7** · VM300-Staged-Source byte-identisch zu origin/main (git-blob `c821cfcc…`, s. u.).

---

## Ergebnis: STOP in Phase 0 — der Anlass-Pfad ist im realen Graph **strukturell unerreichbar** (Dead Branch)

Greifendes Abbruchkriterium (wörtlich aus der Commission): *„OP-0.4/0.5 reproduziert das erwartete Vorher-Verhalten nicht (z. B. weil der Graph anders verzweigt als in der Discovery gelesen) → STOP + Befund, Discovery-Annahme war dann unvollständig."* — Genau das ist eingetreten, **statisch UND live belegt**. Es wurde **kein Gate gebaut, kein Fix-Branch, kein Prod-Repo mutiert.**

### Kernbefund

Die Kette `Paperless Gate → Shop-Liste laden → Domain-Check → Domain bekannt? → LLM-Bestätigung` existiert im Workflow-JSON (Discovery korrekt), ist aber vom Mail-Eingang aus **nicht erreichbar** — in **beiden Zwillingen** (Topologie identisch, s. Zwillings-Volldiff):

1. Realer Fluss: `Mail klassifizieren → known-senders laden → known-sender Pre-Filter → Paperless Gate IF` (`op02b_dead_branch_pruefung.txt`).
2. `Paperless Gate IF` (paperless_candidate==true) → **out 0 → „Nextcloud Task Dokument"** (konservativer Task-Node, kein Cloud-Call). **Nur** der false-Zweig führt weiter Richtung Kategorien-Switch.
3. Auf dem false-Zweig verlangt der Code-Node `Paperless Gate` erneut `paperless_candidate === true` → gibt `[]` zurück → **die Kette stirbt exakt dort** (= #21-Befund 2 „forward-branch-dead", eine Ebene über dem in der Discovery gelesenen Segment).
4. Niemand restauriert `paperless_candidate` dazwischen (`Normalisierung Kategorie` fasst nur category/Flags an; der known-sender Pre-Filter forciert candidate sogar auf false).
5. **Bank/Steuer erreichen nicht einmal den Rechnungs-Zweig:** Der Switch `Dispatch: Kategorie` kennt weder `Bank` noch `Steuer`; `Normalisierung Kategorie` mappt beide (WHITELIST-Miss) auf `Review / Sonstiges` → ntfy (+ u037-Trigger).

### Live-Nicht-Reproduktion (VM300, matthias-Zwilling importiert @579633c; `op04_live_repro_probe.json.txt`, Probe-Quelle `pg_probe.py.txt`)

| Fixture | Kategorie (live) | candidate | LLM-Bestätigung exekutiert? | Terminal |
|---|---|---|---|---|
| fx-bank-att (Kontoauszug+Anhang) | Bank | true | **NEIN** | Nextcloud Task Dokument |
| fx-steuer-noatt (Steuerbescheid) | Steuer | true | **NEIN** | Nextcloud Task Dokument |
| fx-shop-rechnung (**Gegen-Fixture** der Commission) | Rechnung / Finanzen | true | **NEIN** | Nextcloud Task Dokument |
| fx-bank-noatt | Bank | false | **NEIN** | ntfy Sonstige Push (Review / Sonstiges) |
| fx-zahlungserinnerung-v2 (Rechnung, candidate=false) | Rechnung / Finanzen | false | **NEIN** | **`Paperless Gate` exekutiert mit 0 Output-Items** — Kette endet, Shop-Liste/LLM nie erreicht |

Beide OP-Erwartungen scheitern: OP-0.4 (Bank/Steuer erreichen LLM) **und** OP-0.5 (Gegen-Fixture erreicht LLM als Baseline). Die fünfte Fixture belegt das Sterben der Kette am exakten Node (einzige eingehende Kante der LLM-Strecke verwirft jedes Item). Erste Probe-Iteration mit „Lieferung" im Betreff wurde als ungültig verworfen (Kategorie-Kollision, im Rohbeleg dokumentiert und wiederholt).

### Vollständig erledigte Phase-0-OPs (verwertbar für jede Folge-Entscheidung)

- **OP-0.1 (`op01_taxonomie.txt`):** Vollständige Taxonomie beider Zwillinge (identisch, 13 Kategorien inkl. `Account / Sicherheit`); `PERSONAL_CATEGORIES` frisch = `{"Account / Sicherheit", "Bank", "Steuer"}` → Gate-Menge wäre exakt diese drei gewesen.
- **OP-0.2 (`op02_graph_domain_llm.txt` + `op02b`):** Switch-Semantik **invertiert** gegenüber der Commission-Vermutung: `domain_known==true` → out 0 → **LLM-Bestätigung**; unbekannt → out 1 → Nextcloud Task Rechnung. Downstream: LLM-Ergebnis? JA→Forward zu Paperless→ntfy; NEIN→Nextcloud Task Rechnung→ntfy „Rechnung unbekannt".
- **OP-0.3 (`op03_fallback_route.txt`):** Konservative Bestandsroute identifiziert (wäre „Nextcloud Task Rechnung" bzw. der IF-true-Pfad „Nextcloud Task Dokument" gewesen — kein neuer Proposal-Typ nötig). Für den STOP nicht mehr angewandt.
- **Zwillings-Volldiff (`op02c_zwillings_volldiff.txt`):** Topologie/Verbindungen identisch; Unterschiede nur (a) `Dispatch: Kategorie` typeVersion 1 (m) vs. 3 (matthias) bei gleichen 8 Regeln/Fallback, (b) **P4-Enrichment (#40) fehlt im m-Zwilling** (`Move zu Newsletter`/`Forward zu Paperless` ohne from/subject, account_id 'm') — Nebenbefund.

### Invarianten

Isolations-Recheck nach allen Netz-OPs: 4/4 Prod-Ziele timeout/unreachable (`isolation_recheck.txt`). gitleaks 8.30.1 über den Report-Ordner: 0 Findings. VM300-Wegwerf-Probe gelöscht (Quelle als `pg_probe.py.txt` archiviert). Kein Schreibzugriff auf Prod-Repos; sim-Repo unverändert (nur rsync des bereits gemergten main-Stands nach VM300 + Standard-Re-Import).

---

## Konsequenz & Entscheidungsvorlage (Mattes)

**Der beauftragte Fix hätte eine tote Kante bewacht.** Ein Gate vor „LLM-Bestätigung" wäre nach aktuellem Graph wirkungslos (Kette unerreichbar) und im Sinne der Sandbox-Reviews Scheinsicherheit — die S1.3-Assertions („Gegen-Fixture erreicht LLM weiterhin") wären unbeweisbar, weil schon die Baseline nicht existiert.

Reale Ist-Lage für personenbezogene Kategorien (Bank/Steuer/Account) heute: candidate=true → Nextcloud Task Dokument (konservativ, kein Cloud-Call); candidate=false → Review / Sonstiges → ntfy **+ „Trigger u037 Intent Extraction" mit `body_text`** (u037-internes Gate weiterhin nicht repo-verifizierbar — bekannte Discovery-Limitation, außerhalb dieses Auftrags).

Optionen (Entscheidung, keine CC-Erfindung):
1. **Zusammen mit P3 entscheiden:** Der Dead Branch ist derselbe wie P3 (#21-Befund 2). Falls P3 die Kette je reaktiviert, gehört das Privacy-Gate **in denselben Fix** (sonst geht die Kette „scharf" ohne Gate live). Empfehlung: Commission umwidmen zu „P3 + Privacy-Gate gemeinsam" oder das Gate als Bedingung an jeden P3-Fix knüpfen.
2. **Gate trotzdem jetzt einziehen** (Defense-in-Depth auf toter Kante): möglich, aber Vorher/Nachher-DoD der Commission ist damit nicht erfüllbar (kein beweisbares Vorher-Verhalten) — bräuchte angepasstes DoD.
3. **Commission schließen** (Anlass entfällt), Discovery-Korrektur genügt (Addendum liegt bei der Discovery).

## Rohbelege in diesem Ordner

`op01_taxonomie.txt`, `op02_graph_domain_llm.txt`, `op02b_dead_branch_pruefung.txt`, `op02c_zwillings_volldiff.txt`, `op03_fallback_route.txt`, `op04_live_repro_probe.json.txt`, `pg_probe.py.txt`, `isolation_recheck.txt`.
