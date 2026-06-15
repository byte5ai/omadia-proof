# omadia Proof — Implementierungsplan

> **Arbeitsname:** `omadia Proof` · **ANP-Spec:** v0.3-draft · **omadia:** pre-1.0 public preview
> **Repo:** `byte5ai/omadia-proof` (eigenes Repo, Plugin-Bündel) + **ein** kernel-seitiger Hook in `byte5ai/omadia` (geplant; siehe §4 und R10).
> **Status dieses Dokuments:** Plan / Entwurf — **Codex-Review eingearbeitet (2026-06-15, siehe §11)**. Abgelegt als `plan.md` im Repo `byte5ai/omadia-proof`.
>
> **Leitprinzip 1 (UX):** Der Nutzer sieht *Nachweise*, nie Kryptografie. ANP/DLT ist Maschinenraum, kein Produktmerkmal.
> **Leitprinzip 2 (Spec-Treue):** omadia ist die **erste** ANP-Implementierung. Der RFC bleibt die kanonische Quelle — nicht der Code. Jedes Schema, das Proof definiert, **MUSS** als PR zurück in `byte5ai/anp` (zwingend, siehe §1.4 und §11).
>
> **Referenz-Konvention:** `ANP §X` verweist auf einen Abschnitt der ANP-Spezifikation (`SPEC.md`). Ein **nacktes** `§X` (ohne `ANP`) verweist auf einen Abschnitt *dieses* Plans. Diese Trennung ist bewusst, weil ANP und dieser Plan teils dieselben Nummern tragen (z. B. ANP §5.3 = Mandates ≠ §5.3 = UI-Routen dieses Plans).

---

## 0. Worum es geht (in einem Absatz)

ANP ist ein DLT-neutraler Trust-Layer, der jede verbindliche Handlung in ein signiertes, schema-valides, hash-verkettetes und on-chain *verankertes* **ANP Object** verwandelt — Inhalt bleibt off-chain, nur Hash + Status gehen auf die Kette. omadia ist ein selbst-hostbares Agenten-OS mit signierten Plugins, Capability-Registry, Vault und einer „jede Aktion hinterlässt eine Quittung"-Philosophie. **omadia Proof** ist das Modul, das ANP in omadia einbettet und seine gesamte Komplexität — DIDs, Keys, Suites, Anchoring, Escrow — so kapselt, dass ein Manager oder Mitarbeiter ohne IT-Hintergrund nur noch eine Sache erlebt: *„Das ist jetzt verbindlich festgehalten, überprüfbar, manipulationssicher."*

Die zentrale Spannung, die dieser Plan auflöst: ANP setzt voraus, dass jeder Akteur Schlüssel verwaltet und Wallets bedient (genau die Friktion, an der DLT-Adoption bei Menschen scheitert) — die ANP-Spec adressiert das explizit über die *Symbiose-These* (ANP §1.4): **Agenten absorbieren die Wallet-/Key-/Fee-Friktion.** In omadia ist dieser Agent bereits da. Proof macht aus „der Mensch muss eine Wallet bedienen" ein „der Mensch tippt auf *Bestätigen*, der Custodial-Layer signiert".

---

## 1. Faktenbasis (verifiziert aus den Repos, nicht angenommen)

Diese Sektion trennt **Faktum** (aus SPEC.md / omadia-Code gelesen) von **Inferenz** (mein Schluss daraus). Inferenzen sind als solche markiert.

> **Review-Hinweis (Codex):** Mehrere der unten als „Faktum" geführten Aussagen über die ANP-SPEC und den omadia-Code sind für den ganzen Plan tragend, aber aus diesem Dokument allein nicht prüfbar. Sie sind in §11 als **Annahmen-zu-verifizieren (A1–A8)** gelistet und werden in **Stufe A** (§7) gegen die echten Repos bestätigt, bevor die abhängigen Issues finalisiert werden.

### 1.1 Was ANP tatsächlich spezifiziert

| Konzept | Fakt aus SPEC.md |
|---|---|
| **Die drei Säulen** | ① Contract (verbindliche Mehrparteien-Vereinbarung) · ② Notarize (Dritter bezeugt eine Tatsache) · ③ Resolve (neutraler Arbiter, Beweise, bindendes Urteil). ANP §1.3, ANP §7–§9. |
| **Das Spine-Artefakt** | Jede bindende Aktion = ein **ANP Object**: schema-valid, signiert (PQC-fähige, agile Suite), hash-verkettet zum Vorgänger, on-chain verankert nur per Hash+Status. ANP §4.2, ANP §6.1. |
| **On-/Off-chain** | Off-chain: das volle Object. On-chain: nur `{object_hash, object_type, thread_ref, status, anchored_by, timestamp, locator?, outcome?}`. Kein Payload, keine PII. ANP §6.2, ANP §6.4. |
| **Identität** | Jeder Akteur ist ein **W3C DID**; alle Nicht-Identitäts-Aussagen sind **W3C VC 2.0**. Kein anonymes Binden. ANP §5.1, ANP §5.2. |
| **Mandate** | Ein Principal stellt einem Agent ein **Mandate VC** aus: `scope`, `constraints` (max_value, aggregate_value, allowed_counterparties, **escalation_threshold**), `sub_delegation`, `not_before`/`expires`, `status_list`. Aktionen ≥ Threshold brauchen eine Principal-**`approve`**-Co-Signatur. ANP §5.3. |
| **Thread** | Hash-verkettete Sequenz von Objects unter gemeinsamem `thread_ref` (UUID). ANP §2.3. |
| **Object-Typen** | `offer, counter_offer, accept, approve, execute, terminate, memorandum, amend, rescind, attest, witness, revoke, assert, dispute, evidence, rule, appeal, enforce, settle, receipt`. ANP §6.1. |
| **Memorandum** | *Der* Low-Value-/High-Frequency-Fall: ein einzelnes, von allen Parteien co-signiertes Object, **einmal** verankert, terminal bei `ACCEPTED`, **kein Escrow**. Kosten ≈ ein Anchor. ANP §7.2. |
| **Notarization = Autorschaft + Anker** | Attestation ist ein VC (`type: attest`) mit `witnessing: observed\|relayed`, optional M-of-N-Witness-Quorum. „VC (wer sagt was) + on-chain-Anker (unabhängiger Zeitstempel/Ordnung)". ANP §8.2, ANP §5.2. |
| **Standalone-Notarisierung** | Eine Notarisierung **ohne Vertrag** überspringt `REQUESTED` — ein Notar „simply issues and anchors". ANP §8.7. |
| **Dispute (optimistisch)** | `assert` → Challenge-Window → (kein Einspruch ⇒ finalisiert | `settle` co-signiert ⇒ sofort | `dispute` ⇒ Evidence → `rule` → optional `appeal` → `enforce`). Bonds Pflicht auf dem assertiven Pfad. ANP §9.4. |
| **Small-Claims-Profil** | Für Mikrowerte, wo Arbiter-Gebühren + Bonds den Streitwert übersteigen: vor-vereinbarte `split`-Formel statt Forum, Bonds optional, Abschreckung rein reputationsbasiert. ANP §9.4. |
| **Vier Enforcement-Pfade** | `ruling` (Arbiter), `uncontested_assertion` (Happy Path), `mutual_settlement` (einstimmiger Verzicht), `formula_split` (Small-Claims). Nur hier gehen Zahlen on-chain (`outcome`). ANP §6.2.1. |
| **Kostenziel** | „Sub-second finality and near-zero per-action cost, so that recording even trivial determinations is economically sensible." ANP §3.1 Goal 8. Referenz-Chain: IOTA Rebased. ANP §13.2. |
| **Ehrliche Grenzen** | (a) Kein Token, keine Marktplatz-Funktion, **kein Rechtsrahmen** — rechtliche Anerkennung ist explizit offene Frage (ANP §16.5, ANP §3.2). (b) Escrow-Custody + Transaktions-Submission erben die *native* Chain-Krypto (heute nicht PQC). (c) Aggregate-Caps sind nicht gegenparteien-prüfbar. |

### 1.2 Was omadia tatsächlich ist (aus dem Code)

| Konzept | Fakt aus dem Repo |
|---|---|
| **Stack** | TypeScript-Monorepo. `middleware/` (Node + Postgres/pgvector), `web-ui/` (Next.js Admin-UI), separat distribuierte Plugin-ZIPs. |
| **Plugin-Modell** | Alles ist Plugin hinter `@omadia/plugin-api`. Plugins = **signierte ZIPs** (ADR-0001), `node_modules` eingebacken, keine npm-Runtime-Trust. |
| **Capability-Registry** | Versionierte Capabilities `"<name>@<major>"`. Provider deklariert `provides: ["x@1"]` im Manifest, ruft `ctx.services.provide("x", impl)`, Consumer holt `ctx.services.get("x")`. ADR-0003. Entkoppelt: `requires` matcht *jeden* Provider gleichen Namens/Majors. |
| **Paket-Naming** | Kernel/Infra: `harness-*`. Fachliche Plugins: `agent-*` / `harness-plugin-*`. (Bereits belegt: `verifier@1` = LLM-Claim-Verifier, **nicht** ANP — Namenskollision vermeiden.) **→ Folge für Proof: das Paket-Präfix ist eine offene Naming-Entscheidung, siehe Stufe A / R16.** |
| **PluginContext** | Pro Plugin gescoped. Zugriff auf `secrets` (Vault), `config`, `services`, `tools` (native Tools an den Orchestrator), `routes` (HTTP), `uiRoutes` (UI-Erweiterungen), `jobs` (Cron), `knowledgeGraph`, `notifications`, `llm`. Plugin-Code importiert **nie** den Vault direkt. |
| **Vault** | AES-256-GCM, `VAULT_KEY` Pflicht in Prod. Genau der Ort für Custodial-Key-Material. |
| **Privacy Shield v4** | **Architektonisch kritisch:** rohe Tool-Ergebnisse werden serverseitig hinter einer `datasetId` interniert; nur ein identitätsfreies *Digest* überquert die LLM-Leitung. Identitäts-/ordnungskritische Arbeit läuft in vertrautem Servercode. ADR + `privacyReceipt.ts`. |
| **Two-Phase-Write** | ADR-0005: Schreibende Aktionen sind explizit getypt, zweistufig (Preview → Bestätigung), „draft-by-default". „Kein agenten-initiierter Write erreicht ein Live-System, ohne dass ein Mensch die exakte Änderung zuerst sieht." |
| **Audit-Trail** | „Jede Aktion hinterlässt eine Quittung" — Per-Run-Trace + Call-Stack-Viewer. Lokal, nachvollziehbar — aber **nicht** grenzüberschreitend manipulationssicher. (Siehe §2.) |

### 1.3 Die entscheidende Inferenz

> **Inferenz:** omadias bestehende „Quittung" (Run-Trace) und ein ANP-Proof sind *komplementär, nicht redundant*. Die Run-Quittung beweist *dir selbst*, was *dein* System getan hat (lokales Audit, einseitig vertrauenswürdig). Ein ANP-Proof beweist *einer misstrauischen Gegenpartei oder Aufsicht*, was *zwischen* Parteien über Vertrauensgrenzen hinweg vereinbart/bezeugt wurde — extern verankert, von keiner Seite manipulierbar. Proof ist die Brücke vom *internen Log* zum *externen, geteilten Faktum*. Begründung: Die Run-Quittung lebt in deiner Postgres-DB (§1.2), die ANP §1.2 explizit als nicht-vertrauenswürdige Single-Party-Storage einstuft.

---

### 1.4 ZWINGEND: Schema-Rückfluss in den ANP-RFC

> **Diese Anforderung ist nicht verhandelbar und kein „nice-to-have".**

omadia ist die erste Implementierung von ANP. Damit definiert das Core-Paket (`proof.objects@1`) faktisch die JSON-Schemas pro Object-Typ, die ANP §6.1 / Appendix A heute nur referenziert, aber nicht ausformuliert. Wer als Erster implementiert, *setzt* den Standard — ob gewollt oder nicht.

Daraus folgt eine harte Regel für jeden Agenten/Entwickler, der an Proof arbeitet:

1. **Kein Object-Schema entsteht nur im Proof-Code.** Jedes Schema, das Proof für einen ANP-Object-Typ oder ein VC definiert oder ändert, **MUSS** in derselben Arbeitseinheit als Pull Request gegen `byte5ai/anp` (Appendix A) eingereicht werden.
2. **Der RFC ist die Quelle der Wahrheit, nicht der Code.** Bei Divergenz zwischen Proof-Schema und SPEC gewinnt die SPEC; der Code wird angepasst, nicht umgekehrt. Proof pinnt `anp_version` und referenziert die Schema-Version aus dem RFC.
3. **Ein Schema gilt erst als „fertig", wenn der ANP-PR offen ist.** Ein gemergter Proof-Schema-Stand ohne zugehörigen ANP-PR ist ein Regelverstoß, kein Fortschritt. Dies ist Done-Kriterium in jeder Phase, die neue Object-Typen aktiviert.

#### 1.4.1 Schema-Backflow-Matrix (Codex C5 — ersetzt die frühere Prosa)

Diese Matrix ist die *einzige* Quelle dafür, welcher Object-/VC-Typ in welcher Phase erstmals aktiviert wird und welcher ANP-PR dazu offen sein muss. Sie wird in **Stufe A** finalisiert und ist Done-Gate jeder Phase. „Aktiviert" = das Schema wird erstmals produktiv gebaut/validiert; ein Schema wird **nur einmal** definiert (frühere Doppelung `memorandum` in P0 *und* P3 war ein Fehler — P0 *definiert*, P3 *nutzt* es nur für den Co-Signing-Flow).

| Object/VC-Typ | Erst-Aktivierung | ANP-PR Pflicht | Bemerkung |
|---|---|---|---|
| `memorandum` | Phase 0 (definiert), Phase 1 (solo), Phase 3 (co-signed) | ✅ Phase 0 | terminal bei `ACCEPTED`, kein Escrow |
| `attest` (+`witness`) | Phase 2 | ✅ Phase 2 | `observed`/`relayed`; M-of-N optional später |
| `offer` / `counter_offer` / `accept` | Phase 3 | ✅ Phase 3 | offer/accept-Zyklus, noch ohne Escrow |
| `approve` | Phase 4 | ✅ Phase 4 | Principal-Co-Signatur am Eskalations-Gate |
| **Mandate VC** | Phase 4 | ✅ Phase 4 | eigenes VC-Schema (`scope`/`constraints`/`status_list`) — war im alten Gate **vergessen** |
| Status-List-Credential | Phase 4 | ✅ Phase 4 *(falls Proof es definiert)* | sonst Verweis auf ANP-Referenz |
| `assert` / `dispute` | Phase 5 | ✅ Phase 5 | optimistischer Dispute-Einstieg |
| `evidence` | Phase 5 | ✅ Phase 5 | war im alten Gate **vergessen** |
| `rule` / `appeal` / `enforce` / `settle` | Phase 5 | ✅ Phase 5 | `appeal` war im alten Gate **vergessen** |
| `receipt` | Phase 1 (Store-Delivery) bzw. Phase 5 (Settlement) | ✅ bei Erst-Aktivierung | DA-/Settlement-Quittung — war im alten Gate **vergessen** |

**Begründung:** Driften erste Implementierung und Spec auseinander, entwertet das den RFC — für dessen Autor (byte5) ein direktes Eigentor. Der Aufwand, Schemas synchron zu halten, ist trivial gegenüber dem Schaden einer Spec, der ihre eigene Referenzimplementierung widerspricht.

---

## 2. Das Produktversprechen (User zuerst)

Der Nutzer denkt nicht in Säulen. Er denkt in einer einzigen Frage: **„Will ich, dass das hier später unbestreitbar ist?"** Wenn ja, drückt er einen Knopf. Proof übersetzt diesen Knopf in die richtige ANP-Säule:

| Was der Nutzer will | Was er sieht | Was Proof darunter tut |
|---|---|---|
| „Festhalten, dass ich/wir das gesagt/entschieden haben" | **Festhalten** | Memorandum (1 Partei = self-attest; N Parteien = co-signiertes Memorandum) |
| „Bestätigen, dass etwas so ist/war" | **Bezeugen** | Attestation (`witnessing: observed\|relayed`) |
| „Eine Abmachung verbindlich machen" | **Vereinbaren** | Contract: offer/accept (+ optional Escrow/Kriterien) |
| „Einen Streit darüber klären lassen" | **Klären** | Dispute: assert/dispute/rule/enforce |

Die vier Worte — **Festhalten · Bezeugen · Vereinbaren · Klären** — sind die *gesamte* nutzerseitige Oberfläche. Alles andere ist gekapselt.

### 2.1 Die nicht verhandelbaren UX-Gesetze

Diese leiten sich direkt aus „die Nutzer sind keine IT-Experten" + ADR-0005 + Privacy Shield v4 ab:

1. **Kein Krypto-Vokabular im UI.** Niemals „Hash", „DID", „Wallet", „Anchor", „Suite", „Chain", „Gas". Stattdessen: *Nachweis, Identität, festgehalten, überprüfbar, Beleg-Nr.* Der technische Beleg ist über „Details ansehen" erreichbar, nie aufgedrängt.
2. **Identität ist unsichtbar, aber real.** Jeder omadia-User und jeder Agent bekommt beim ersten Kontakt automatisch ein DID + Keys, im Vault verwahrt (custodial). Der User signiert durch *Bestätigen*, nicht durch Key-Handling. (Architektur in §4; DID-Methode ist Stufe-A-Entscheidung, R12.)
3. **Bestätigen ist die einzige menschliche Geste.** Genau das Two-Phase-Muster von ADR-0005, erweitert: Preview des exakten, menschenlesbaren Vorgangs → ein Tap → fertig. Proof ist konzeptionell „der erste Write-Connector, der die Aktion *zwischen Organisationen* verbindlich macht".
4. **Der Anker-Vorgang ist asynchron und still.** Sub-Sekunden-Finalität (ANP §13.2) heißt: der User sieht „✓ Festgehalten" quasi sofort; die Bestätigung der Chain-Verankerung trudelt im Hintergrund nach und aktualisiert nur ein dezentes Status-Icon (analog „gesendet → zugestellt" bei Messengern). *Voraussetzung: ein robuster Async-Confirmation-Pfad mit Retry/Timeout/Idempotenz — siehe R14.*
5. **Hashes/DIDs überqueren nie die LLM-Leitung.** Striktes Privacy-Shield-v4-Mandat: Proof-Identifikatoren werden serverseitig interniert; der Orchestrator/LLM sieht nur opake Referenzen (`proofRef`) und menschenlesbare Zusammenfassungen, nie Roh-Krypto. **Das hat eine Konsequenz für die Tool-Surface (§5.1):** auch `thread_ref`, `counterparties[]` und `evidence[]` sind *linkbare Identifikatoren bzw. Rohdaten* und dürfen nicht roh über die LLM-Grenze. Der konkrete Datenfluss pro Tool (was sieht das LLM, was wird serverseitig interniert, welche opaken IDs ersetzen Roh-Referenzen) ist **Stufe-A-Pflicht-Spec, R13.** (Begründung: ein Hash/Thread-Ref im Kontextfenster ist sowohl ein Privacy-Leak via `anchored_by`-Linkgraph ANP §6.2 als auch eine Halluzinationsquelle.)
6. **Ehrlichkeit über Rechtskraft.** Wo ein Vorgang rechtlich einen Notar/Zeugen braucht, sagt Proof das *klar* und positioniert sich als „technisch fertiger, manipulationssicherer Nachweis — die rechtliche Form (Notar/eIDAS) ist separat". Kein Overpromise. (ANP §3.2/ANP §16.5 der Spec verlangt diese Haltung.) **Wording-Disziplin:** im *Produkt-UI* nie „rechtsverbindlich"; „verbindlich" in diesem Plan meint *technisch bindend zwischen den Parteien gemäß ANP*, nicht rechtlich durchsetzbar (R2).

---

## 3. Die vier Topologien — als First-Class-Bürger entworfen

Du willst **Human-to-Nobody, Human-to-Human, Agent-to-Human, Agent-to-Agent** absichern. ANPs Mandate-Modell (ANP §5.3) ist exakt auf die `Human → Agent → Agent → Human`-Achse gebaut; die vier Topologien sind Punkte auf dieser Achse. So mappt Proof sie:

| Topologie | Was es ist | ANP-Mechanik | Wer signiert | UX-Einstiegspunkt | Phase |
|---|---|---|---|---|---|
| **Human → Nobody** | Self-Notarisierung für die Nachwelt; unveränderlicher Milestone, *keine* Gegenpartei jetzt | **Standalone Attestation** (`witnessing: observed` durch den User selbst über sich) **oder** Single-Party-**Memorandum**; anchored, terminal. ANP §8.7 erlaubt „issue and anchor" ohne Vertrag | Nur der User (custodial) | „Festhalten" — solo | 1 |
| **Human → Human** | Zwei Menschen halten eine Abmachung fest | Co-signiertes **Memorandum** (Record) *oder* offer/accept (verbindlich, optional Escrow) | Beide User (je custodial) | „Festhalten" / „Vereinbaren" — mit Gegenpartei | 3 |
| **Agent → Human** | Ein Agent proponiert, ein Mensch nimmt an (oder umgekehrt) | offer (Agent unter Mandate) → accept (Human) **oder** Attestation des Agent + Human-`approve` bei Threshold-Überschreitung ANP §5.3 | Agent (Mandate) + Human (`approve`/`accept`) | Agent erzeugt Vorschlag → Human-Bestätigungskarte | **4** *(siehe §3.2)* |
| **Agent → Agent** | Zwei Agenten schließen end-to-end ab — **dein Favorit** | Voller offer/accept-Zyklus, beide unter Mandate; Eskalation an Menschen nur ≥ `escalation_threshold` | Beide Agenten (Mandate), Menschen nur bei Eskalation | Unsichtbar; nur Eskalationen + Audit-Sicht erscheinen | 4 |

### 3.1 Warum Agent-to-Agent das strategische Zentrum ist (und was es exklusiv verlangt)

> **Inferenz, mit Begründung:** Agent-to-Agent ist der einzige der vier Fälle, der heute *gar nicht* anders lösbar ist (US-1/US-4 der ANP-User-Stories bestätigen das gegen x402/AP2/Virtual-Cards). Die anderen drei haben Notlösungen (E-Mail, DocuSign, Git). A2A wächst zudem mit dem Agentic-Computing-Trend automatisch. Daraus folgt die Priorisierung in §7.

A2A verlangt zwei Dinge, die die anderen Topologien nicht brauchen und die deshalb Architektur-Treiber sind:

- **Agenten-DIDs + Mandate als Laufzeit-Objekt.** Jeder omadia-Agent, der proof-fähig sein soll, braucht ein eigenes DID (nicht das des Users) und ein vom Principal ausgestelltes Mandate VC mit Caps. Das Mandate ist die „Blast-Radius-Grenze" gegen halluzinierende Agenten (ANP §7.5).
- **Eskalation als Kernel-Hook, nicht als Plugin-Detail.** Wenn eine Agenten-Aktion `escalation_threshold` erreicht, muss *zuverlässig* ein Mensch in die Schleife — das ist dieselbe Human-Checkpoint-Garantie wie ADR-0005, nur ausgelöst durch einen Mandate-Constraint statt durch „ist ein Write". Proof verdrahtet das in den Orchestrator-Dispatch (R10).

### 3.2 Korrektur: Agent→Human gehört in Phase 4, nicht Phase 3 (Codex C3)

> **Faktum aus diesem Plan:** Ein *verbindlicher* Agent→Human-Vorgang verlangt, dass der Agent **unter Mandate signiert** (§3-Tabelle, §5.1). Agenten-DIDs, Mandate-Ausstellung, `checkAuthority` und das Eskalations-Gate werden aber erst in **Phase 4** eingeführt. Damit ist „Agent schlägt verbindlich vor, Mensch nimmt an" in Phase 3 **nicht** baubar — es wäre eine Vorwärts-Abhängigkeit auf Phase 4.

Auflösung:

- **Phase 3** bleibt auf **Human↔Human** plus den entschärften Fall **„Agent draftet, Mensch signiert als Principal"** — d. h. der Agent *erzeugt nur den Vorschlagsinhalt*, die einzige bindende Signatur ist die des Menschen (kein Agenten-Mandate, keine Agenten-Signatur). Das braucht weder Mandate noch Gate.
- **Verbindliches Agent→Human** (Agent signiert unter Mandate, Human `approve`/`accept`) wird in **Phase 4** scharfgeschaltet, zusammen mit A2A — beide teilen denselben Mandate-/Gate-Unterbau.

---

## 4. Architektur: wie Proof in omadia sitzt

**Repo-Verteilung (entschieden).** `byte5ai/omadia-proof` ist das richtige Zuhause und bleibt: Proof ist zu fast allem sauberes Plugin-Territorium, und omadia distribuiert Plugins ohnehin als eigenständige ZIPs (Teams, Telegram leben genauso außerhalb des Monorepos). Das gesamte Bündel — Object-Bau, Identity, Signer, Anchor-Adapter, Store, native Tools, UI — wird im Proof-Repo gebaut und als signiertes ZIP in omadia installiert.

**Kernel-Eingriffe — geplant einer, möglicherweise zwei (Codex C2).** Die *bekannte* Ausnahme zum reinen Plugin-Territorium ist das **Eskalations-Gate** (Mandate-Threshold ⇒ Human-`approve`, §5.1): es muss *vor* der Tool-Ausführung in den Orchestrator greifen, und die Plugin-API gibt heute keinen plugin-erweiterbaren Pre-Dispatch-Hook her. Beleg: `harness-verifier` ließ seinen orchestrator-bindenden Wrapper bewusst kernel-seitig — „the Orchestrator class is ~1k LOC and not yet plugin-extractable". Das Gate hat dasselbe Problem und landet deshalb als **ein kleiner, gezielter PR in `byte5ai/omadia` selbst** — und zwar erst in Phase 4, wenn A2A es braucht. **Konditional, nicht garantiert einzig:** ob die **Custodial-Key-Skalierung** (R11 / §7 Stufe A) zusätzlich einen kernel-seitigen Custody-Service braucht, ist eine offene Stufe-A-Frage. Bestätigt sich, dass der per-plugin Vault-Secrets-Store per-User/-Agent-Keys *nicht* in der nötigen Größenordnung trägt, ist das ein **zweiter, größerer** Kernel-Eingriff. Die frühere Formulierung „die einzige Cross-Repo-Änderung des ganzen Projekts" ist daher zu „der geplante Eskalations-Gate-PR; ein zweiter Custody-PR ist möglich und wird in Stufe A geklärt" abgeschwächt. Ein Plugin, das ein bis zwei gezielte Kernel-Hooks braucht, ist immer noch ein Plugin.

Proof ist **kein Monolith-Plugin**, sondern eine kleine Schicht von Capability-Providern plus UI-Erweiterungen, exakt nach dem Muster, das omadia für `knowledgeGraph@1`, `verifier@1`, `privacy.redact@1` bereits nutzt (ADR-0003). Das hält ANP austauschbar (DLT-Neutralität auf Code-Ebene gespiegelt) und Proof testbar.

> **Naming-Hinweis (Stufe-A-Entscheidung, R16 — Codex C8):** Die Capability-IDs (`proof.objects@1`, `proof.identity@1`, …) sind stabil und werden in diesem Plan durchgängig als primäre Referenz benutzt. Das **Paket-Präfix** ist dagegen *offen*: §1.2 reserviert `harness-*` für *Kernel/Infra im Monorepo* — Proof ist aber ein externes Plugin-Bündel. Vor dem ersten Paket gegen das `omadia-plugin-starter`-Template prüfen, welches Präfix dort für extern distribuierte Plugins gilt (vermutlich `agent-*` oder ein eigener Scope wie `@byte5/proof-*`). Keine Kosmetik: Paketpfade, Manifeste und Release-Artefakte hängen daran. Einmal in Stufe A festlegen, bevor sechs Pakete den falschen Namen tragen.

```
┌──────────────────────────────────────────────────────────────────────┐
│  omadia UI (web-ui / canvas)                                           │
│  „Festhalten · Bezeugen · Vereinbaren · Klären"-Karten,                │
│  Bestätigungs-Dialoge, Proof-Inbox, Beleg-Detailsicht                  │
│  (uiRoutes vom Proof-Plugin beigesteuert)                              │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  proofRef (opak) + menschenlesbare Summary
                            ▼  — NIE Roh-Krypto (Privacy Shield v4, R13)
┌──────────────────────────────────────────────────────────────────────┐
│  Orchestrator-Hooks                                                    │
│  • Native Tools: proof.record / proof.attest / proof.agree / proof.    │
│    resolve  (an den Agenten exponiert)                                 │
│  • Eskalations-Gate: Mandate-Threshold ⇒ Human-approve erzwingen (R10) │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  ctx.services.get("proof.*")
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PROOF-CAPABILITY-SCHICHT (neue Pakete; Präfix = Stufe-A-Entscheidung) │
│                                                                        │
│  proof.objects@1   ── ANP-Object-Bau: Envelope, JCS-Canonicalization,  │
│                       Schema-Validierung, Hash-Verkettung. Reine       │
│                       Logik, chain-agnostisch. Spiegelt ANP §6.        │
│                                                                        │
│  proof.identity@1  ── Custodial DID/Key-Verwaltung pro User+Agent,     │
│                       Mandate-VC-Ausstellung & -Prüfung, SD-JWT-       │
│                       Disclosure. Keys NUR im Vault. Spiegelt ANP §5.  │
│                                                                        │
│  proof.signer@1    ── Signatur-Suite (PQC-fähig: ML-DSA/SLH-DSA),      │
│                       detached signatures für Co-Signing. ANP §6.3/§6.5│
│                       Trennt „signieren" von „Schlüssel besitzen".     │
│                                                                        │
│  proof.anchor@1    ── DLT-PROFIL-ADAPTER. Das einzige Paket, das die   │
│  (anchor-iota /      Kette kennt. Default: IOTA-Rebased-Profil.        │
│   anchor-mock)       Austauschbar gegen EVM-Profil ohne Consumer-      │
│                      Änderung (ANP §13 als Code-Grenze). anchor(hash), │
│                      status(), settlement/escrow-Hook, DA-Locator.     │
│                                                                        │
│  proof.store@1     ── Off-chain-Object-Storage + DA-Layer (ANP §6.4):  │
│                       Mechanismus offen (KG/Memory/Mirror, R15);       │
│                       + optional content-addressed Mirror; Void-on-    │
│                      unavailability-Regel; receipts; Verify-Link-Quelle.│
└──────────────────────────────────────────────────────────────────────┘
```

### 4.1 Warum diese Schnitt-Linien

- **`proof.anchor@1` ist die DLT-Naht.** Genau wie omadia `knowledge-graph-inmemory` vs. `-neon` swappt, swappt Proof `anchor-iota` vs. einen künftigen `anchor-evm` — ohne dass Core, Identity oder UI es merken. Das ist ANPs „DLT-neutral" (ANP §3.1 Goal 3) als Code-Architektur, nicht nur als Prosa. **Für Tests:** ein `anchor-mock` (In-Memory-Ledger) macht die gesamte Proof-Logik ohne echte Chain testbar — Pflicht, weil sonst kein CI.
- **`proof.identity@1` ist die Krypto-Kapsel.** Hier — und nur hier — wird die „kein Wallet"-Magie real: Custodial Keys im Vault, Signieren als Service, Mandate als ausstellbares/prüfbares VC. Wenn ein Enterprise später *eigene* Schlüssel will (HSM, eIDAS-QES), wird nur dieses Paket ersetzt.
- **`proof.objects@1` ist reine, deterministische Logik.** Kein I/O, keine Kette. Damit ist die heikelste Korrektheits-Anforderung (JCS-Canonicalization, deterministische Proof-Assembly ANP §6.3 — „every party assembles a byte-identical anchored form") isoliert unit-testbar gegen die Spec-Beispiele.

### 4.2 Datenmodell (logisch — physischer Store ist Stufe-A-Entscheidung, R15)

> **Faktum, das die ursprüngliche Annahme korrigiert:** Ein externes Plugin bekommt **keinen rohen Postgres-Pool**. `MigrationContext` (der `onMigrate`-Hook) erlaubt nur Schreibzugriff auf den per-plugin **Secrets**- und **Memory**-Store, nicht das Anlegen eigener Tabellen. Die unten genannten Entitäten sind deshalb ein **logisches** Modell — wie sie physisch persistiert werden, ist eine **blockierende Stufe-A-Entscheidung** (R15), kein Faktum, und **`proof.store@1` (ab Phase 1 gebraucht) hängt daran** (Codex C1). Mein Stand: der Knowledge Graph (`knowledgeGraph@1`) ist der wahrscheinlichste Träger, weil er ACL und Retrieval mitbringt; zu verifizieren (A5), bevor `proof.store@1` festgelegt wird.

Strikte On-/Off-chain-Trennung der Spec bleibt zwingend: PII lebt ausschließlich off-chain in `body` (verschlüsselt at rest über den existierenden Mechanismus); auf der Kette nur Hash + Status.

| Entität (logisch) | Zweck | Schlüsselfelder |
|---|---|---|
| `proof_identities` | DID + verschlüsselte Key-Referenz pro User/Agent | `subject_id` (FK user/agent), `did`, `key_vault_ref`, `kind` (`human`/`agent`), `created_at` |
| `proof_mandates` | Ausgestellte Mandate-VCs (Agent-Autorität) | `mandate_id`, `principal_did`, `agent_did`, `scope[]`, `constraints`, `not_before`, `expires`, `status_list_ref`, `revoked_at?` |
| `proof_objects` | Off-chain ANP Objects (das volle Artefakt) | `object_id`, `thread_ref`, `type`, `sequence`, `previous_hash`, `body` (**verschlüsselt at rest**), `proof[]`, `anchored_hash?`, `da_locator?` |
| `proof_threads` | Abgeleiteter Thread-Zustand (off-chain repliziert, **Cache**) | `thread_ref`, `topology`, `derived_state`, `head_hash`, `pillar`, `owner_subject_id`, `counterparties[]` |
| `proof_anchors` | Spiegel der On-chain-Anker (schnelle Abfrage ohne Chain-Roundtrip) | `object_hash`, `thread_ref`, `status`, `anchored_by_did`, `ledger_timestamp`, `confirmation_state` |
| `proof_status_snapshots` | Retained Status-List-Credentials für Point-in-time-Beweis (ANP §5.3) | `snapshot_hash`, `fetched_at`, `credential` |

`proof_threads.derived_state` wird **nie** als Quelle der Wahrheit behandelt — es ist ein Cache, der bei Bedarf aus den Ankern + Objects neu abgeleitet wird (SPEC m5: „fine state off-chain by replaying the Thread's anchored Object types"). Das ist die Versicherung gegen genau den Failure-Mode, den ANP §1.2 nennt: eine manipulierte lokale DB darf den Zustand nicht verfälschen können.

---

## 5. API-Surface

### 5.1 Native Tools für den Agenten (an den Orchestrator exponiert)

Vier Tools, gespiegelt an den vier Nutzer-Worten. Jedes nimmt strukturierte Eingaben, gibt einen **opaken `proofRef`** + menschenlesbare Summary zurück — nie Roh-Krypto:

| Tool | Säule | Eingabe (schema-validiert) | Verhalten |
|---|---|---|---|
| `proof.record` | Memorandum | `{ statement, parties[]? }` | 0/fehlende `parties` ⇒ Human-to-Nobody-Self-Memorandum; N ⇒ Co-Signing-Flow. Terminal bei `ACCEPTED`. (`parties` weggelassen ≡ leer.) |
| `proof.attest` | Notarization | `{ subject_kind, statement, witnessing, evidence[]? }` | Standalone-Attestation. Erzwingt ehrliche `observed`/`relayed`-Deklaration (ANP §8.1). |
| `proof.agree` | Contract | `{ terms, counterparties[], escrow? }` | offer→accept-Zyklus. `escrow.required` ⇒ voller Settlement-Pfad; sonst günstiger Record-Pfad. |
| `proof.resolve` | Dispute | `{ thread_ref, action: assert\|dispute\|evidence, payload }` | Optimistischer Dispute. Wählt automatisch small-claims-Profil unterhalb der Wertschwelle (ANP §9.4). |

> **Privacy-Shield-Konsequenz (R13):** Die obigen Eingaben enthalten linkbare Identifikatoren (`thread_ref`, `counterparties[]`) und potenziell Rohdaten (`evidence[]`). Über die LLM-Grenze gehen diese **nicht roh**, sondern als serverseitig aufgelöste opake Handles. Der Tool-Vertrag (LLM-sichtbar vs. server-interniert) wird in **Stufe A** pro Tool spezifiziert, *bevor* die Tools gebaut werden.

**Mandate-Gate (Orchestrator-Hook, nicht im Tool):** Vor Ausführung prüft der Dispatch, ob der handelnde Agent ein gültiges Mandate hat *und* die Aktion in `constraints` liegt. ≥ `escalation_threshold` ⇒ Tool-Ausführung pausiert, eine Human-Bestätigungskarte wird erzeugt (`approve`), erst danach läuft es weiter. Genau die ADR-0005-Garantie, getriggert durch Mandate statt durch Write-Typ. (Kernel-PR, Phase 4, R10.)

### 5.2 Capability-Interfaces (Plugin-zu-Plugin, intern)

```typescript
// proof.objects@1  — reine Logik, kein I/O
interface ProofObjectsService {
  buildEnvelope(input: ObjectInput): UnsignedObject;       // ANP §6.1
  canonicalizeForSigning(obj: UnsignedObject): Uint8Array; // JCS, proof entfernt ANP §6.3
  canonicalizeAnchored(obj: SignedObject): Uint8Array;     // inkl. proof[]
  assembleProofs(obj, sigs): SignedObject;                 // deterministische Reihenfolge ANP §6.3
  validate(obj): SchemaResult;                             // gegen registriertes JSON-Schema
}

// proof.identity@1  — die Krypto-Kapsel
interface ProofIdentityService {
  ensureIdentity(subjectId: string, kind: 'human'|'agent'): Promise<Did>; // auto-provision
  issueMandate(principal: Did, agent: Did, c: MandateConstraints): Promise<MandateVc>;
  checkAuthority(agentDid, action, value): Promise<AuthorityVerdict>;      // ANP §5.3 Regeln
  proveDisclosure(mandate, predicate): Promise<SdJwtPresentation>;         // ANP §6.6, kein Cap-Leak
}

// proof.signer@1
interface ProofSignerService {
  sign(subjectId: string, input: Uint8Array): Promise<DetachedSignature>;  // Key bleibt im Vault
  suite(): SuiteId;                                                        // anp-suite-2 (PQC)
}

// proof.anchor@1  — die DLT-Naht (default: IOTA Rebased)
interface ProofAnchorService {
  anchor(hash: TaggedHash, meta: AnchorMeta): Promise<AnchorReceipt>;      // ANP §6.2
  setStatus(hash, status): Promise<void>;
  openEscrow?(terms): Promise<EscrowId>;   // optional je Profil-Capability ANP §10
  enforce?(directive): Promise<void>;      // ANP §6.2.1
  confirmationState(hash): Promise<'pending'|'final'>;
}

// proof.store@1
interface ProofStoreService {
  put(obj: SignedObject): Promise<void>;
  get(hash: TaggedHash): Promise<SignedObject | undefined>; // Void-on-unavailability ANP §6.4
  deliver(obj, counterparties): Promise<Receipt[]>;         // DA-Pflicht + receipt
}
```

### 5.3 UI-Routen (vom Proof-Plugin via `uiRoutes` beigesteuert)

- `/proof` — **Proof-Inbox**: alle Vorgänge des Users, gefiltert nach Status (festgehalten / wartet auf Gegenpartei / wartet auf mich / in Klärung / abgeschlossen). Kein Krypto sichtbar.
- `/proof/:ref` — **Beleg-Detailsicht**: menschenlesbare Zusammenfassung oben; „Technischen Beleg ansehen" als Aufklapp-Sektion (hier — und nur hier — werden Hash/DID/Anker für Auditoren sichtbar, read-only).
- **Bestätigungs-Dialog** (Canvas-Komponente, kein eigener Pfad): Preview des exakten Vorgangs, ein Bestätigen-Button. Das menschliche Signatur-Moment.
- **Verify-Link** (Phase 2/3, read-only, permissionless): eine extern teilbare, omadia-unabhängige Prüfseite für einen Beleg. Was sie exponiert, wie autorisiert/redigiert wird und wie sie sich bei nicht verfügbarem Object verhält, ist Teil der DA-/Verify-Link-Spezifikation (R9).

---

## 6. Die heiklen Stellen — ehrlich benannt

Kein Plan ohne die Risiken, die ihn kippen können. R1–R8 wie ursprünglich; R9–R16 aus dem Codex-Review ergänzt. Nach Schwere sortiert innerhalb der Gruppen.

| # | Risiko | Schwere | Mitigation |
|---|---|---|---|
| R1 | **Custodial Keys = Single Point of Compromise.** Wenn der Vault fällt, kann ein Angreifer im Namen *aller* User/Agenten signieren. Das ist der Preis für „kein Wallet". | **Hoch** | Vault ist bereits AES-256-GCM + `VAULT_KEY`-Pflicht. Zusätzlich: per-subject Key-Derivation, Mandate-Caps begrenzen den Blast-Radius pro Agent (ANP §7.5), Eskalation für Hochwert-Aktionen. Upgrade-Pfad zu HSM/eigenen Keys über Austausch nur von `proof.identity@1`. **Klar dokumentieren** (ADR-0001), dass dies eine bewusste Custody-Entscheidung ist. Operatives Key-Lifecycle separat: R11. |
| R2 | **Rechtskraft wird überschätzt.** Nutzer könnten glauben, ein Proof ersetze einen Notar. Tut er nicht (ANP §16.5, ANP §3.2). | **Hoch** | UX-Gesetz 6: explizite, ehrliche Einordnung im UI bei jedem Hochwert-/formbedürftigen Vorgang. Proof positioniert sich als *technischer* Nachweis, nicht als Rechtsform. Im UI nie „rechtsverbindlich" texten; „verbindlich" = technisch bindend gemäß ANP. |
| R3 | **Privacy-Leak via On-chain-Linkgraph.** `anchored_by`-DID + `thread_ref` erzeugen einen permanenten, un-löschbaren „wer-mit-wem-wann"-Graph auf öffentlicher Kette (ANP §6.2 m6). | **Mittel-hoch** | Pairwise-DIDs pro Beziehung/Thread, wo eine natürliche Person identifizierbar ist (ANP §12 macht das zur Pflicht — A2). `proof.identity@1` generiert diese automatisch — der User merkt nichts. Verschärft durch GDPR-Spannung: R13b. |
| R4 | **Gegenparteien-Adoption.** Jeder Vorgang mit Gegenpartei braucht *beide* Seiten auf ANP/omadia (ANP „shared adoption costs"). | **Mittel-hoch** | Human-to-Nobody (Self-Notarisierung) hat **keine** Gegenpartei und ist deshalb der Adoptions-Türöffner: sofortiger Solo-Nutzen ohne Netzwerkeffekt. Daraus folgt Phase 1 (§7). Für Gegenparteien ohne omadia: der „Verify-Link" (read-only Verifikation im Browser, permissionless ANP §2.2) — Prüfen, nicht Mitzeichnen (siehe Phase 3 Scope-Grenze). |
| R5 | **Escrow-Custody erbt native Chain-Krypto (nicht PQC).** Ein Quanten-Angreifer auf das Chain-Konto könnte Escrow-Mittel bewegen, obwohl die Binding-Signaturen PQC-sicher bleiben (ANP §6.5 „honest limitation"). | **Mittel** (nur escrow-tragende Threads) | Escrow-Beträge + Dispute-Fenster bounden; Chain mit glaubwürdigem PQC-Migrationspfad bevorzugen; als Langzeit-Risiko offenlegen. Die meisten Proof-Vorgänge (Record/Attest) tragen *kein* Escrow und sind davon unberührt. |
| R6 | **JCS-Canonicalization-Bugs ⇒ Signaturen verifizieren nicht über Parteien hinweg.** Wenn zwei Implementierungen nicht byte-identisch kanonisieren, ist Multi-Party-Co-Signing tot. | **Mittel** | `proof.objects@1` als reine Logik gegen die SPEC-Beispiele unit-getestet; Konformitäts-Testvektoren aus Appendix A (A1). CI-Gate. |
| R7 | **ANP ist v0.3-draft mit offenen Fragen (ANP §16).** Die Spec kann sich vor v1.0 ändern. | **Niedrig-mittel** | `proof.objects@1` pinnt `anp_version`. Proof-Module versionieren mit. Migrations-/Versionierungs-Policy operationalisieren: R14b. Offene-Fragen-Bereiche (Trust-List-Governance, Arbiter-Pool-Bootstrap) erst in späteren Phasen produktiv. |
| R8 | **Namenskollision `verifier@1`.** Existiert bereits als LLM-Claim-Verifier. | **Niedrig** | Proof nutzt `proof.*`-Namespace durchgängig. Kein Capability heißt `verify`/`verifier`. (Paket-Präfix separat: R16.) |
| **R9** | **Externe Verifizierbarkeit ohne Data-Availability-Fläche** (Codex C4). Ein Chain-Hash allein reicht nicht, um einen Beleg unabhängig zu prüfen, wenn das volle Off-chain-Object nicht abrufbar ist — die „überlebt omadia"-/Verify-Link-Versprechen (Phase 2/3) brauchen eine DA-Schicht. | **Mittel-hoch** | DA-Locator + öffentliche Abruf-/Verify-Link-Fläche als **explizite Phase-2-Vorbedingung**, bevor „überlebt omadia" behauptet wird. Spezifizieren: was der Link zeigt, Autorisierung, Redaction, Locator-Verhalten, Semantik bei nicht verfügbarem Object. |
| **R10** | **Kernel-Eskalations-Gate** (Codex C2/C3). Pre-Dispatch-Hook im ~1k-LOC-Orchestrator; einzige *geplante* Kernel-Änderung. | **Mittel** | Als generischer Pre-Dispatch-Policy-Hook im Kernel entwerfen (nicht proof-spezifisch verdrahtet), Phase 4. Kontrakt + Tests vor dem PR; `harness-verifier`-Präzedenz (A6) bestätigen. |
| **R11** | **Custodial-Key-Skalierung & -Lifecycle** (Codex C2 + IMPORTANT). Trägt der per-plugin Vault per-User/-Agent-Keys in nötiger Größenordnung? Backup/Restore, Kompromittierungs-Revocation, Rotation, Recovery, Operator-Runbooks fehlen. | **Hoch** | Stufe-A-Verifikation gegen den echten Vault-Stand (A7); ggf. kernel-seitiger Custody-Service (zweiter Kernel-Eingriff, §4). Lifecycle-Runbook als eigenes Cross-Cutting-Workitem. |
| **R12** | **DID-Methode + Resolver ungelöst** (Codex C6). Bestimmt, ob Identität Ledger-Lookup braucht (`did:iota`/`did:ethr`) oder lokal/custodial auflösbar ist (`did:key`/`did:web`); muss zum Anchor-Profil passen; **schwer reversibel**, weil DIDs in jedem anchored Object stehen. | **Hoch** | Blockierende Stufe-A-Entscheidung *vor* `proof.identity@1`: Methode, Resolver, Rotation, Pairwise-Mapping, Migration. Mein Stand: `did:key` (custodial, kein Lookup) oder `did:web` unter Instanz-Domain; `did:iota` erst, wenn das Reference-Profil es nahelegt. |
| **R13** | **Privacy-Shield-Datenfluss pro Tool unspezifiziert** (Codex C7). `thread_ref`/`counterparties[]`/`evidence[]` in Tool-Inputs könnten linkbare IDs/Rohdaten über die LLM-Grenze leaken. | **Mittel-hoch** | Stufe-A-Pflicht-Spec pro Tool: LLM-sichtbar vs. server-interniert vs. summarized; opake Handles statt Roh-Referenzen (UX-Gesetz 5). |
| **R13b** | **GDPR / Recht auf Löschung vs. unlöschbare Anker** (Codex IMPORTANT). Hash + Linkgraph bleiben on-chain, auch wenn off-chain gelöscht wird. | **Mittel** | Entscheidung: was ist löschbar (off-chain `body`/Keys), was bleibt (Hash/Anker); wie wird der verbleibende Hash/Link erklärt; bricht Löschung die Verifikation? Als Compliance-Workitem, vor produktiver PII-Verarbeitung. |
| **R14** | **Observability & Async-Failure-Handling fehlen** (Codex IMPORTANT). Anchor-Retry, Dead-Letter, Finality-Timeout, Duplicate-Anchor-Idempotenz, Metriken, degraded states. | **Mittel** | Async-Anchor-Worker mit Retry/Idempotenz/Timeout + user-sichtbarem degraded state (UX-Gesetz 4). Observability-Baseline als Cross-Cutting. |
| **R14b** | **`anp_version`-Migration nicht operationalisiert** (Codex IMPORTANT). Version gepinnt, aber kein Schema-Versions-Registry/Migrations-Policy/Backward-Verify. | **Niedrig-mittel** | Schema-Versions-Registry, Migrations-Policy, Backward-Verification, Fixture-Versionierung, Kompatibilitäts-Tests — mit der Schema-Matrix (§1.4.1) koppeln. |
| **R15** | **Object-Store-Mechanismus offen, Phase 1 hängt daran** (Codex C1). KG vs. Memory vs. Mirror; ACL/Retrieval/Encryption/Export/Retention undefiniert. | **Hoch** | Blockierende Stufe-A-Entscheidung *vor* `proof.store@1`/Phase 1: logisch→physisch-Mapping, ACL-Modell, Retrieval-API, Verschlüsselung, Export, Retention + Akzeptanztests (A4/A5). |
| **R16** | **Paket-Naming nicht kosmetisch** (Codex C8). `harness-proof-*` widerspricht §1.2; falsche Paketpfade/Manifeste/Artefakte drohen. | **Niedrig-mittel** | Stufe-A-Naming-Entscheidung + Repo-Layout-ADR, *bevor* das erste Paket angelegt wird. |
| **R17** | **Multi-Tenant-Isolation nicht abgedeckt** (Codex IMPORTANT). Tenant-Scoping für Identities, Mandates, Objects, Anchors, Verify-Link-Zugriff, Auditor-Rechte fehlt. | **Mittel** | Als Querschnitts-Anforderung an `proof.identity@1`/`proof.store@1` führen; Akzeptanzkriterien je betroffenem Paket. |
| **R18** | **Clock-/Timestamp-Trust** (Codex IMPORTANT). Lokaler vs. Ledger-Timestamp, Skew, Ordering-Disputes. | **Niedrig-mittel** | Präzedenz Ledger-Timestamp vor lokalem; Skew-Handling + Test-Fixtures; in Phase 2 (Verankerung) verankern. |

---

## 7. Phasenplan — entlang der Use-Cases, nicht entlang der Technik

Die Sequenz folgt zwei Prinzipien: **(a) frühester Solo-Nutzen ohne Netzwerkeffekt** (löst R4), **(b) A2A — dein strategisches Zentrum — als klares Ziel, aber auf tragfähigem Fundament.** Jede Phase ist demonstrierbar nützlich, nicht nur „Infrastruktur". **Neu (Codex):** eine vorgeschaltete **Stufe A (Architektur-Freeze)** löst die blockierenden Entscheidungen, *bevor* Code entsteht — sie ersetzt die früheren „offenen Entscheidungspunkte" (§10 alt) und macht sie zu harten Gates.

### Stufe A — Architektur-Freeze (kein Code, blockierend)
**Ziel:** Die schwer reversiblen Entscheidungen treffen und die tragenden Annahmen gegen die echten Repos verifizieren, bevor irgendein Paket angelegt wird. Jeder Punkt ist Voraussetzung für mindestens eine spätere Phase.
**Inhalte (Gates):**
1. **Paket-Naming & Repo-Layout** festlegen (R16) → ADR.
2. **Object-Store-Mechanismus** entscheiden: KG vs. Memory vs. Mirror, inkl. ACL/Retrieval/Encryption/Export/Retention-Spec (R15, A4/A5) — Voraussetzung für `proof.store@1`/Phase 1.
3. **DID-Methode + Resolver** entscheiden: Methode, Rotation, Pairwise-Mapping, Migration (R12) — Voraussetzung für `proof.identity@1`.
4. **Custodial-Key-Skalierung** prüfen: per-plugin Vault vs. Kernel-Custody-Service (R11, A7) — bestimmt, ob ein zweiter Kernel-PR nötig ist.
5. **Schema-Backflow-Matrix** finalisieren (§1.4.1) inkl. ANP-PR-Prozess + Fixture-/Versionierungs-Policy (R14b).
6. **Privacy-Shield-Datenfluss pro Tool** spezifizieren (R13).
7. **Annahmen A1–A8 (§11) verifizieren** gegen `byte5ai/anp` SPEC.md und `byte5ai/omadia`-Code.
8. **ADRs schreiben:** `0001-anp-proof-custodial-identity.md` (Custody-Entscheidung, R1, inkl. HSM/QES-Upgrade-Pfad), `0002-schema-source-of-truth.md` (§1.4), `0003-repo-layout-and-naming.md` (R16), `0004-object-store.md` (R15), `0005-did-method.md` (R12).
**Done:** Alle fünf ADRs gemerged; Schema-Matrix steht; A1–A8 bestätigt oder als Risiko quittiert. Erst dann beginnt Phase 0.

### Phase 0 — Fundament (kein Nutzer-Feature)
**Pakete:** `proof.objects@1`, `proof.anchor-mock@1`, Skeleton `proof.store@1`, Plugin-Scaffold + CI-Pipeline.
**Ziel:** ANP-Object-Bau + Canonicalization + Schema-Validierung, gegen Spec-Testvektoren grün; ein In-Memory-Mock-Ledger; ein gegen das Plugin-Starter-Template gebautes, signier-/paketierbares Skeleton mit grüner CI. **Kein UI.** Dies ist die Korrektheits-Versicherung (R6) und macht alles Folgende CI-testbar.
**CI-Breite (Codex IMPORTANT — nicht nur Canonicalization):** Unit (JCS/Hash-Chain gegen Appendix-A-Vektoren), Schema-Validierung, Plugin-Packaging/Signing, Capability-Registrierung, später Privacy-Shield-Boundary-Tests.
**Done-Kriterium (zwei harte Gates):**
1. Ein Memorandum lässt sich bauen, co-signieren (simuliert), kanonisieren und „verankern" (mock) — byte-identisch reproduzierbar.
2. **Jedes hier definierte Object-Schema hat einen offenen PR gegen `byte5ai/anp` (Appendix A)** gemäß Schema-Matrix (§1.4.1) — für Phase 0: `memorandum`. Ohne diesen PR gilt Phase 0 als nicht abgeschlossen (§1.4).

### Phase 1 — „Festhalten", solo (Human-to-Nobody) · *der Türöffner*
**Voraussetzung:** Stufe-A-Gates 2 (Store), 3 (DID), 4 (Custody) entschieden.
**Use-Case:** Manager hält fest „Ich habe heute Entscheidung X getroffen, Grund Y" — Milestone für die Nachwelt. Kein Gegenüber, sofortiger Nutzen.
**Pakete:** `proof.identity@1` (custodial DID/Key, **nur Human**), `proof.signer@1`, `proof.store@1` (gemäß Stufe-A-Entscheidung), erstes UI: `proof.record` solo + Proof-Inbox + Beleg-Detailsicht. **Void-on-unavailability**-Verhalten spezifiziert + implementiert.
**Anchor:** noch Mock *oder* erstes echtes IOTA-Profil, je nach Reife (Entscheidung in dieser Phase).
**Warum zuerst:** Kein Adoptions-Netzwerkeffekt (R4), validiert die *gesamte* Krypto-Kapselung (Gesetz 2+5) an einem risikoarmen Fall, liefert ein vorzeigbares Feature.
**Done:** Ein Nicht-IT-User hält etwas fest, sieht „✓ Festgehalten", findet es in der Inbox, sieht null Krypto — und ein Auditor kann den technischen Beleg aufklappen.

### Phase 2 — Echte Verankerung + „Bezeugen"
**Use-Case:** US-6-artig — ein Vorgang/Audit-Ergebnis wird bezeugt und überlebt das System, das es erzeugt hat.
**Pakete:** `proof.anchor-iota@1` produktiv (IOTA-Rebased-Profil ANP §13.2) inkl. **asynchronem Anchor-Worker (Retry/Idempotenz/Timeout, R14)**, `proof.attest` (Standalone-Attestation, `observed`/`relayed`), asynchroner Confirmation-State im UI (Gesetz 4), Status-Snapshots (`proof_status_snapshots`), **DA-Locator + Verify-Link-Foundation (R9)**, Timestamp-Handling (R18).
**Vorbedingung (Codex C4):** Die „überlebt omadia"-Behauptung gilt erst, wenn die DA-/Verify-Link-Fläche steht — sie ist Teil dieser Phase, nicht Annahme.
**Done:** „Bezeugen" funktioniert end-to-end gegen eine echte Kette; der Confirmation-State trudelt still nach; Verifikation hängt nicht von omadias Weiterexistenz ab (über DA + Verify-Link nachgewiesen); ANP-PR für `attest` offen.

### Phase 3 — Zwei Parteien (Human-to-Human; Agent draftet, Mensch signiert)
**Use-Case:** Meeting-Protokoll mit Verbindlichkeit (H2H); ein Agent *erstellt den Inhalt* eines Vorschlags, der Mensch signiert ihn als Principal.
**Pakete:** Co-Signing-Flow produktiv (`proof.record` mit N Parteien, detached signatures ANP §6.3), `proof.agree` (offer/accept, **noch ohne** Escrow), Bestätigungs-Dialog für die Gegenpartei, „Verify-Link" für Gegenparteien ohne omadia (read-only, permissionless, auf R9-Fundament).
**Scope-Grenze (wichtig, sonst falsche Erwartung):** Ein *verbindliches* zweiseitiges Agreement verlangt eine anchored Signatur (`accept`) **jeder** Partei (ANP §7.4, m10). Beide Seiten müssen also ANP-fähig sein. Der „Verify-Link" lässt eine Nicht-omadia-Gegenpartei einen Vorgang **prüfen**, nicht **mitzeichnen**. Verbindliche H2H-Vorgänge mit einer Partei *ohne* ANP-Stack sind in v1 **nicht** möglich (nur einseitig festhalten + Link zur Prüfung schicken). Eine browserbasierte Leichtsignatur für externe Gegenparteien (custodial guest-DID) ist denkbares späteres Feature, nicht v1-Scope.
**Abgrenzung (Codex C3):** *Verbindliches* Agent→Human (Agent signiert unter Mandate) ist **nicht** in dieser Phase — es braucht Mandate + Gate und liegt in Phase 4 (§3.2). In Phase 3 darf ein Agent nur *draften*; die einzige bindende Signatur ist die des Menschen.
**Done:** Zwei *omadia-/ANP-fähige* Parteien halten eine Abmachung byte-identisch co-signiert fest; ein Agent kann einem Menschen einen Vorschlagsinhalt zur Signatur als Principal vorlegen; ANP-PRs für `offer`/`counter_offer`/`accept` offen.

### Phase 4 — Agent-to-Agent (+ verbindliches Agent-to-Human) · *das strategische Zentrum*
**Use-Case:** Dein Favorit — KI-Agenten-Beauftragung mit Audittrail; zwei Agenten schließen end-to-end ab; zusätzlich verbindliches Agent→Human (aus Phase 3 hierher gezogen, §3.2).
**Pakete (Proof-Repo):** `proof.identity@1` erweitert um **Agenten-DIDs + Mandate-Ausstellung** (`issueMandate`, `checkAuthority`), **Status-List-Hosting + Mandate-Revocation-Lifecycle** (Codex IMPORTANT), SD-JWT-Disclosure für Mandate (ANP §6.6, gegen Cap-Leak ANP §5.3), **Aggregate-Cap-Enforcement**, Mandate-Verwaltungs-UI (Principal setzt Caps — „Wofür darf dieser Agent ohne Rückfrage unterschreiben?").
**Kernel-PR (`byte5ai/omadia`, R10):** das **Eskalations-Gate** als kleiner kernel-seitiger, möglichst *generischer* Pre-Dispatch-Policy-Hook, der zur Dispatch-Zeit das Mandate prüft und bei Threshold-Überschreitung die Tool-Ausführung pausiert, bis ein Human-`approve` vorliegt. Muss in den Kernel, weil es *vor* der Tool-Ausführung greift und die Plugin-API keinen Pre-Dispatch-Hook hergibt (§4, Verifier-Präzedenz A6). Kontrakt + Tests vor dem PR.
**Done:** Zwei omadia-Agenten verschiedener Parteien schließen autonom innerhalb ihrer Mandate ab; ein verbindliches Agent→Human läuft über Mandate + `approve`; eine Aktion über der Schwelle eskaliert zuverlässig an einen Menschen; Mandate sind widerrufbar (Status-List); der gesamte Vorgang ist als Audittrail nachvollziehbar; ANP-PRs für `approve` + Mandate-VC offen.

### Phase 5 — „Klären" + Settlement (Escrow & Dispute)
**Use-Case:** US-1/US-5 — Wertvorgänge mit Escrow und der Streit, den keine Plattform besitzt.
**Pakete:** Escrow-Hooks in `proof.anchor-iota@1` (`openEscrow`/`enforce` ANP §10/ANP §6.2.1) inkl. Bond-Funding + Challenge-Windows, `proof.resolve` (optimistischer Dispute + automatische Small-Claims-Profil-Wahl unterhalb der Wertschwelle ANP §9.4), Arbiter-/Trust-Modell + Small-Claims-Formel-Quelle, Enforcement-Autorisierung, Dispute-UI.
**Warum zuletzt:** Höchste Komplexität, höchste Risiken (R5), und ökonomisch erst sinnvoll, wenn die günstigen High-Frequency-Fälle (Phasen 1–4) Traktion haben. Bis dahin gilt für Mikrowerte: anchored Evidence + reputationsbasierte Abschreckung, kein Forum.
**Done:** Ein escrow-getragener Vorgang kann optimistisch finalisieren, einvernehmlich sofort settlen oder über einen Dispute zu einem durchgesetzten Ergebnis führen — ohne Kooperation der unterlegenen Seite; ANP-PRs für `assert`/`dispute`/`evidence`/`rule`/`appeal`/`enforce`/`settle`/`receipt` offen.

### Übersicht

| Phase | Topologie/Säule | Liefert dem Nutzer | Kern-Risiko adressiert |
|---|---|---|---|
| A | — (Architektur-Freeze) | (Entscheidungen/ADRs) | R11, R12, R13, R15, R16 |
| 0 | — | (Fundament) | R6 (Canonicalization), CI-Breite |
| 1 | H→Nobody / Record | „Festhalten" solo | R4 (Adoption), R1-Validierung |
| 2 | — / Notarize | „Bezeugen" + echte Kette | R3 (Linkgraph), R9 (DA/Verify), R14, R18 |
| 3 | H→H (+ Agent draftet) | „Vereinbaren" zu zweit | R4 (Verify-Link) |
| 4 | **A→A (+ verbindl. A→H)** | autonome Agenten + Eskalation | R10 (Gate), R1 (Mandate-Caps), R2 |
| 5 | / Resolve | „Klären" + Escrow | R5 (Escrow-Krypto) |

---

## 8. Was bewusst *nicht* in v1 gehört

- **Keine eigene Marktplatz-/Discovery-Funktion.** ANP ist explizit kein Marktplatz (ANP §3.2). Agenten finden einander über A2A/Registries, nicht über Proof.
- **Kein Token.** Settlement nutzt den nativen Chain-Asset/Stablecoin (ANP §3.2). Proof führt nie eine eigene Währung ein.
- **Keine Multi-Tier-Appeals, keine VRF-Witness-Pools, keine Reputations-Registry** — alles in ANP definiert, aber späte/optionale Profile. Erst wenn die Basis trägt.
- **Keine rechtliche Form (eIDAS-QES, Notariat).** Bewusst außerhalb. Proof ist der technische Nachweis; die Rechtsform ist ein separater, späterer Integrationspunkt (austauschbares `proof.identity@1` mit QES-Backend wäre der Weg).
- **Keine browserbasierte Gast-Signatur** für Nicht-omadia-Gegenparteien (nur Verify-Link/Prüfen). Denkbar später, nicht v1.

---

## 9. Sofort-Nächste-Schritte (konkret)

1. **Proof-Repo strukturieren:** In `byte5ai/omadia-proof` dieses Dokument als `plan.md` ablegen, dazu `data-model.md` (Sektion 4.2 ausbauen, sobald der Store-Mechanismus entschieden ist — Stufe A), `research.md` (ANP-Spec-Auszüge + A1–A8-Verifikation), `tasks.md`.
2. **Stufe A starten** (§7): die fünf ADRs schreiben, Schema-Matrix finalisieren, A1–A8 verifizieren. **Kein Paket-Code vor Stufe-A-Done.**
3. **Schema-Quelle klären, dann bauen:** Da omadia die erste ANP-Implementierung ist, definiert `proof.objects@1` die Object-Schemas — **mit gleichzeitigem PR gegen `byte5ai/anp` Appendix A** (§1.4) gemäß Matrix. Spec-Testvektoren aus Appendix A als Fixtures, soweit vorhanden; wo nicht, werden sie hier erzeugt *und* in den RFC gespielt.

---

## 10. Entscheidungspunkte → in Stufe A überführt

> Die ursprünglich offene Schema-Frage ist beantwortet (§1.4: omadia definiert die Schemas und spielt sie zwingend zurück). Die in der ersten Fassung als „drei offene Punkte" geführten Themen (Object-Store-Mechanismus, Custodial-Key-Skalierung, DID-Methode) sind **nicht** mehr nur Prosa, sondern **blockierende Stufe-A-Gates** (R15, R11, R12) mit ADR-Pflicht. Der Codex-Review hat zusätzlich Naming (R16) und den Privacy-Shield-Datenfluss (R13) als entscheidungs-blockierend identifiziert. Alle leben jetzt in Stufe A (§7).

*(Hinweis: Die frühere Inkonsistenz „§9.4 sagt zwei, §10 listet drei" ist mit dieser Überführung erledigt.)*

---

## 11. Review-Resolution (Codex, 2026-06-15)

Dieses Dokument wurde von Codex (effort high, read-only) reviewt, da es zuvor ungeprüft war. Eingearbeitete Befunde:

| Codex-Befund | Eingearbeitet als |
|---|---|
| C1 Store-Blocker, Phase 1 hängt daran | R15 + Stufe-A-Gate 2 + §4.2-Hinweis |
| C2 „Single Kernel PR" konditional | §4 reframed + R10/R11 + Stufe-A-Gate 4 |
| C3 Agent→Human Vorwärts-Abhängigkeit | §3.2 + Phase 3/4 umsortiert + §3-Tabelle |
| C4 Externe Verifizierbarkeit ohne DA-Fläche | R9 + Phase-2-Vorbedingung + Verify-Link in §5.3 |
| C5 Schema-Gate unvollständig/doppelt | §1.4.1 Schema-Backflow-Matrix (ersetzt Prosa) |
| C6 DID-Methode ungelöst | R12 + Stufe-A-Gate 3 + ADR-0005 |
| C7 Privacy-Shield vs. Tool-Surface | R13 + §2.1.5/§5.1-Hinweise + Stufe-A-Gate 6 |
| C8 Paket-Naming nicht kosmetisch | R16 + Naming als Stufe-A-Gate 1 + Capability-IDs als primäre Referenz |
| IMPORTANT: CI zu eng | Phase-0 CI-Breite |
| IMPORTANT: Key-Lifecycle/DR | R11 + Cross-Cutting |
| IMPORTANT: Status-List-Hosting | Phase-4-Paket |
| IMPORTANT: `anp_version`-Migration | R14b |
| IMPORTANT: Observability/Async-Failure | R14 + Phase-2-Worker |
| IMPORTANT: Void-on-unavailability | Phase-1-Done |
| IMPORTANT: Escrow/Dispute zu dünn | Phase-5-Paket ausgebaut |
| IMPORTANT: Multi-Tenant-Isolation | R17 |
| IMPORTANT: GDPR/Erasure | R13b |
| IMPORTANT: Clock/Timestamp | R18 + Phase 2 |
| MINOR: „zwei vs. drei" | §10 bereinigt |
| MINOR: Phase-2/5-Topologie-Zellen leer | §7-Übersicht gefüllt |
| MINOR: „verbindlich"-Wording | UX-Gesetz 6 / R2 präzisiert |
| MINOR: `parties[]` weggelassen vs. leer | §5.1-Tabelle präzisiert |
| MINOR: Dateiname `plan.md` | umbenannt |

**Annahmen-zu-verifizieren (A1–A8)** — load-bearing Aussagen über ANP-SPEC/omadia-Code, die in Stufe A gegen die echten Repos zu bestätigen sind, bevor abhängige Issues finalisiert werden:

| # | Annahme | Hängt dran |
|---|---|---|
| A1 | ANP definiert die gelisteten Object-Typen, Status, Standalone-Attestation, Memorandum-Terminalverhalten, deterministische Proof-Ordnung wie beschrieben | Phase-0 Schema-/Hash-Tests |
| A2 | ANP verlangt/empfiehlt Pairwise-DIDs stark genug, um R3-Mitigation zu rechtfertigen | Identity-Architektur, Privacy |
| A3 | ANP unterstützt die zitierten PQC-Suites, SD-JWT-Mandate-Disclosure, IOTA-Rebased-Profil | Phase 1–4 Sign/Verify |
| A4 | omadia-Plugins haben *keinen* rohen Postgres-Zugriff, nur per-plugin Secrets/Memory via `onMigrate` | gesamte Storage-Strategie |
| A5 | `knowledgeGraph@1` taugt für Object-Persistenz, ACLs, Retrieval, encrypted-at-rest | Phase-1 Storage-Option |
| A6 | omadia hat keinen plugin-erweiterbaren Pre-Dispatch-Hook; `harness-verifier`-Präzedenz gilt | Kernel-PR-Validität (R10) |
| A7 | Vault-Skalierung/KV-Limits/Backup/Rotation reichen für per-User/-Agent-Custody | Phase-1 Identity, R11 |
| A8 | Plugin-Signing, ZIP-Packaging, `provides/requires`, `uiRoutes`, native Tools, Privacy-Shield-Internals verhalten sich wie beschrieben | Phase-0 Scaffold, Tool-Wiring |

---

*Grundlage: ANP SPEC.md v0.3-draft (byte5ai/anp) + omadia public preview (byte5ai/omadia), gelesen am 15.06.2026. Codex-Review eingearbeitet 15.06.2026 (§11). Inferenzen sind im Text als solche markiert; alles andere ist direkt aus Spec/Code belegt — soweit nicht in A1–A8 als zu verifizieren geführt.*
