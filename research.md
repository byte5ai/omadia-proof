# research.md — Verifikation der Plan-Annahmen A1–A8

> **Zweck:** Stufe-A-Artefakt (plan.md §7 Stufe A, §11). Der Plan markiert acht tragende Aussagen über `byte5ai/anp` (SPEC.md) und `byte5ai/omadia` (Code) als **„aus dem Plan allein nicht prüfbar"** (A1–A8). Der Codex-Review konnte diese Repos nicht lesen. Dieses Dokument prüft sie gegen die echten Quellen.
>
> **Methode:** `byte5ai/anp` @ `main` (SPEC.md, 1033 Z.) direkt gelesen; `byte5ai/omadia` @ `main` (TS-Monorepo) über drei gezielte Code-Recherchen. Jede Aussage geerdet an Datei:Zeile bzw. `SPEC §`. **Fakt** (aus Quelle belegt) von **Inferenz** (mein Schluss) getrennt. Belege gelesen am 2026-06-15.
>
> **Quell-Commits:** anp `main` (pushed 2026-06-15T16:38Z) · omadia `main` (pushed 2026-06-15T18:49Z). SPEC-Autor laut Z. 1031: „Christian (byte5 GmbH)" — der RFC gehört byte5 (relevant für §1.4).

## Verdikt-Übersicht

| # | Annahme (Kurzform) | Verdikt | Kernbeleg |
|---|---|---|---|
| A1 | ANP definiert Object-Typen/Status/Standalone-Attest/Memorandum-terminal/det. Proof-Ordnung wie beschrieben | ✅ **bestätigt** | SPEC §6.1, §6.2, §6.3, §7.2, §8.7 |
| A2 | Pairwise-DIDs stark genug für R3 | ✅ **bestätigt (stärker: MUST)** | SPEC §6.2 (m6), §12 |
| A3 | PQC-Suites, SD-JWT-Mandate, IOTA-Rebased-Profil | ✅ **bestätigt** (mit Pre-GA-Caveat) | SPEC §6.5, §5.3/§6.6, §13.2 |
| A4 | Plugins ohne rohen Postgres-Pool, nur Secrets/Memory via `onMigrate` | ✅ **bestätigt** | plugin-api `pluginContext.ts`, `vault.ts` |
| A5 | `knowledgeGraph@1` taugt für Persistenz/ACL/Retrieval/**encrypted-at-rest** | ⚠️ **teilweise — Encryption-at-rest fehlt** | KG-Migrations, `inMemoryKnowledgeGraph.ts` |
| A6 | Kein plugin-erweiterbarer Pre-Dispatch-Hook; Verifier-Präzedenz | ✅ **bestätigt** | `turnHooks.ts`, `orchestrator.ts`, `harness-verifier/plugin.ts` |
| A7 | Vault skaliert für per-User/-Agent-Custodial-Keys | ⚠️ **unbestätigt — auf O(100) Agents ausgelegt** | `fileVault.ts`, `vault.ts` |
| A8 | Plugin-Signing/Packaging/`provides`/`requires`/`uiRoutes`/Tools/Privacy-Shield wie beschrieben | ✅ **bestätigt** | plugin-api, `privacyReceipt.ts`, `harness-plugin-privacy-guard` |

**Fazit:** 6× bestätigt, 2× mit substanzieller Einschränkung (A5, A7). Beide Einschränkungen **bestärken** den Plan (R15-Store-Entscheidung, R11-Custody-Skalierung) statt ihn zu kippen — sie machen die jeweiligen Stufe-A-Gates dringlicher und konkreter. Zusätzlich vier neue Funde unten.

---

## A1 — Object-Modell (✅ bestätigt)

- **Object-Typen.** SPEC §6.1 (Z. 290–294) listet die `type`-Enum **exakt** wie plan.md §1.1: `offer, counter_offer, accept, approve, execute, terminate, memorandum, amend, rescind, attest, witness, revoke, assert, dispute, evidence, rule, appeal, enforce, settle, receipt`.
- **Status.** SPEC §6.2 (Z. 352): coarse on-chain `active | superseded | revoked | disputed | enforced`; **feine Zustände off-chain durch Replay der Object-Typen abgeleitet** (m5). Deckt plan.md §4.2 (`derived_state` = Cache, nie Source-of-Truth) exakt.
- **Standalone-Attestation.** SPEC §8.7 (Z. 660): *„A standalone notarization (no contract) skips `REQUESTED` — a Notary simply issues and anchors."* → plan.md §3-Tabelle (Human→Nobody) bestätigt.
- **Memorandum terminal.** SPEC §7.2 (Z. 452): *„anchored once, reaching `ACCEPTED` on anchoring, which is terminal (no `EXECUTED`/`DISPUTED` tail) … cost ≈ one anchor, no settlement."* → plan.md §1.1/§5.1 exakt.
- **Deterministische Proof-Ordnung (R6-Keystone).** SPEC §6.3 (Z. 396): *„`proof[]` entries MUST appear in the same order as their corresponding `signers[]` entries, so that every party assembles a byte-identical anchored form and derives the same `object_hash`."* Plus zwei kanonische Formen (Signing-Form ohne `proof`; Anchored-Form mit `proof`) + JCS RFC 8785 (§6.3, Z. 387–398). → deckt `proof.objects@1`-Interface (#4) und R6 vollständig.

> **Nuance (für #2/#4, Inferenz):** `accept`/`approve`/`witness`/`evidence`/`revoke` sind **Attachment-Objects** (SPEC §6.1 Z. 314–322): `previous_hash: null`, Ziel-Hash im `body` (`accepts_hash`/`approves_hash`/`attestation_hash`/`dispute_hash`/`revokes`), `sequence` = Ziel-`sequence`. Die übrigen sind **Chain-Objects**. Das Schema-Modell muss diese zwei Klassen abbilden. `receipt` ist **non-anchored / transport-only** (§6.1 Z. 329, §6.4) — es hat ein Schema, ist aber von Anchoring/DA ausgenommen. Außerdem: die SPEC schwankt zwischen `attest` (Enum-Token §6.1/§8.2) und „attestation" (Prosa §2.3) — Proof sollte `attest` setzen und die Inkonsistenz im ANP-PR flaggen.

## A2 — Pairwise-DIDs (✅ bestätigt, sogar MUST)

- SPEC §6.2 (m6, Z. 354): *„For privacy-sensitive Threads, parties SHOULD — and where a natural person is identifiable, **MUST** (§12) — use per-relationship or per-Thread **pairwise DIDs**."*
- SPEC §12 (Z. 812): *„For Threads in which a natural person is identifiable … implementations **MUST** use per-relationship or per-Thread pairwise DIDs."*
- → plan.md R3 ist nicht nur gerechtfertigt, sondern **normativ erzwungen**. `proof.identity@1` (#6) muss Pairwise-DIDs automatisch erzeugen — kein optionales Feature.

## A3 — PQC / SD-JWT / IOTA-Profil (✅ bestätigt, Pre-GA-Caveat)

- **PQC-Suites.** SPEC §6.5 (Z. 418): *„Suites MAY use NIST post-quantum signatures — **ML-DSA (FIPS 204)** or **SLH-DSA (FIPS 205)** — today, or hybrid classical+PQC."* → plan.md §4/§5.2 (`anp-suite-2`, ML-DSA/SLH-DSA) bestätigt.
- **SD-JWT-Mandate-Disclosure.** SPEC §5.3 (Z. 268) + §6.6 (Z. 427): Mandates **SHOULD** als SD-JWT-VCs mit Prädikaten, für `contracting`-Scope **strongly RECOMMENDED** (gegen Cap-Leak). → plan.md §5.2 `proveDisclosure` bestätigt.
- **IOTA-Rebased-Profil.** SPEC §13.2 (Z. 835–851): Reference-Profil, Starfish-Consensus p99 ≈ 312 ms (sub-second), Anker ≈ 0.001 IOTA, Gas-Station-Sponsoring, nativer Randomness-Beacon + ECVRF. **Off-chain-PQC ready:** IOTA Identity (v1.7+) stellt ML-DSA-44/65/87, SLH-DSA, FALCON, Hybrid aus (Z. 846). → plan.md §13.2/Phase 2 bestätigt.
- **R5-Bestätigung (Fakt):** SPEC §6.5 (Z. 423) + §13.2 (Z. 850): Chain-native Accounts sind **Ed25519 (nicht PQC)** → Escrow-Custody + TX-Submission erben Nicht-PQC-Krypto. plan.md R5 exakt.

> **Neuer Caveat (Fakt, → neues Risiko unten):** Die IOTA-First-Party-Tools sind **pre-GA**: IOTA Identity **Beta** (v1.9.9-beta.1, „no stable release yet"), Gas Station v0.5.2 (pre-1.0), IOTA Notarization + Hierarchies **Alpha** (SPEC §13.2 Z. 847–851). Das ist eine reale Abhängigkeits-Reife-Frage für Phase 1 (Identity/PQC) und Phase 2 (Anchor).

## A4 — Plugin-Storage-Isolation (✅ bestätigt)

- `PluginContext` exponiert **keinen** DB-Pool — vollständige Feldliste (`pluginContext.ts:31–147`): `secrets`, `config`, `services`, `memory`, `tools`, `routes`, `uiRoutes`, `jobs`, `knowledgeGraph`, `llm`, … kein SQL/Pool.
- `MigrationContext` überschreibt nur `secrets` mit `SecretsReadWriteAccessor` (`pluginContext.ts:1090–1100`); Schreibzugriff ist **per `agentId` namespaced** (`vault.ts:30–36`, `fileVault.ts:39` `byAgent = new Map<string, Map<string,string>>()`). Memory ist per-Plugin strukturell isoliert (`pluginContext.ts:78–84`).
- → plan.md §4.2 Faktum bestätigt: logisches Datenmodell, kein eigener Postgres. Store-Mechanismus bleibt Stufe-A-Entscheidung (R15).

## A5 — Knowledge Graph als Object-Store (⚠️ teilweise — **Encryption-at-rest fehlt**)

| Eigenschaft | Vorhanden? | Beleg |
|---|---|---|
| Persistenz | ✅ | `knowledgeGraph.ts:206–216` (`createMemorableKnowledge`/`getMemorableKnowledge`); Neon `0001_graph_init.sql:11–22` (`graph_nodes`) |
| ACL/Permission | ✅ | `inMemoryKnowledgeGraph.ts:1015–1047` (`addOwner`/`removeOwner`, `acl_owners`); `0018_acl_owners.sql` (GIN-Index, Audit-Tabelle) |
| Retrieval/Query | ✅ | `knowledgeGraph.ts:221–231` (`listMemorableKnowledgeFor`, ACL-gated) |
| **Encrypted-at-rest** | ❌ **fehlt** | `0001_graph_init.sql:8` lädt `pgcrypto` nur für `gen_random_uuid()`; Properties liegen als **plaintext JSONB**; keine Field-/Row-Encryption |

> **Konsequenz (Inferenz, wichtig):** plan.md §4.2 verlangt, dass PII im `body` **„verschlüsselt at rest über den existierenden Mechanismus"** liegt — und nimmt implizit an, der KG bringe das mit. **Tut er nicht.** Wird der KG als Store gewählt (R15/ADR-0004), muss die Verschlüsselung des `body` **auf Applikationsebene** geschehen, bevor er in den KG geht (z. B. über `proof.signer`/Vault-Krypto), oder der Store braucht eine eigene Verschlüsselungsschicht. Das ist eine konkrete Anforderung an ADR-0004 und an `proof.store@1`/`proof.objects@1` — kein Show-Stopper, aber explizit zu lösen.

## A6 — Kein Pre-Dispatch-Hook; Verifier-Präzedenz (✅ bestätigt)

- **Hook-Punkte sind post-hoc.** `turnHooks.ts:20–27` definiert exakt vier `TurnHookPoint`: `onBeforeTurn` (vor erster LLM-Inferenz, **nicht** vor Dispatch), `onAfterToolCall`, `onAfterTurn`, `onVerifierBlocked` — **keiner pre-dispatch**.
- **Dispatch ist kernel-only.** `orchestrator.ts:3038–3100` (`dispatchTool`) verzweigt direkt in Privacy-Shield → Sub-Agent → `dispatchToolInner` — kein plugin-registrierbarer Hook davor.
- **Verifier-Präzedenz wörtlich.** `harness-verifier/src/plugin.ts:17–48`: *„the Orchestrator class itself is ~1k LOC and not yet plugin-extractable … we keep the wrapper kernel-side."* Der gatende `VerifierService` liegt kernel-seitig in `harness-orchestrator/src/verifierService.ts` und prüft **nach** dem Turn (post-answer), nicht pre-dispatch.
- **Korrektur (Fakt):** Die tatsächliche `orchestrator.ts` hat **3.691 Zeilen** (nicht ~1k — der Kommentar ist veraltet); die „nicht plugin-extrahierbar"-Begründung gilt dadurch *stärker*.
- → plan.md R10 (Eskalations-Gate als kernel-seitiger Pre-Dispatch-PR) ist die einzige architektonisch saubere Option. ADR-0005 (Two-Phase-Write) ist App-Level, kein Kernel-Dispatch-Gate — bestätigt, dass das Gate genuin neu in den Kernel muss.

## A7 — Vault-Skalierung für Custodial-Keys (⚠️ unbestätigt — auf O(100) Agents ausgelegt)

- **Krypto bestätigt:** `fileVault.ts:146–157` AES-256-GCM, 32-Byte-Masterkey, random IV; `fileVault.ts:18`.
- **Aber Design-Annahme klein:** `fileVault.ts:23` Kommentar *„target scale (O(100) agents × O(10) keys)"* — d. h. ~1.000 Secrets. Die Persistenz serialisiert **den gesamten Vault als ein JSON-Blob** und verschlüsselt ihn atomar (`fileVault.ts:132–144`).
- **Keine Rotation, keine per-Secret-Derivation:** `fileVault.ts:186–221` (`resolveMasterKey`) — kein Rotations-Pfad (Masterkey-Wechsel ⇒ Re-Encryption nötig); alle Secrets unter einem geteilten Masterkey. Nightly-Backup nach S3/Tigris existiert (`vaultBackup.ts:65–224`).
- **Nur per-Agent, nicht per-User:** `vault.ts:5–6` *„Secrets are ALWAYS namespaced by agent identity.id"* — kein zweiter User-Tier.

> **Konsequenz (Inferenz):** Custodial-Keys **pro User UND pro Agent** (potenziell Tausende) passen **nicht offensichtlich** in den heutigen File-Vault (Single-Envelope-JSON, O(100)-Agents-Design, keine Rotation). Das **bestätigt R11** und macht das Stufe-A-Custody-Gate (#1) konkret: entweder Vault-Erweiterung oder der im Plan §4 erwogene **zweite, kernel-seitige Custody-Service**. Diese Entscheidung blockiert `proof.identity@1` (#6) zu Recht.

## A8 — Plugin-Packaging & Runtime-Surface (✅ bestätigt)

- **Signing:** `harness-plugin-office/src/signing.ts:1–60` (HMAC-SHA256, timing-safe). **ZIP-Guardrails:** `src/plugins/zipExtractor.ts:20–60` (Extension-Allowlist, Entry-/Byte-Limits, Symlink-/Path-Escape-Schutz).
- **`provides`/`requires` + `@major`:** `pluginContext.ts:243–296` (`CapabilityRef {name, major}`, `parseCapabilityRef`); echte Manifeste: `harness-plugin-web-search/manifest.yaml:165` (`provides: ["webSearch@1"]`). Resolver: `src/plugins/capabilityResolver.ts`.
- **Services:** `pluginContext.ts:314–338` (`get`/`provide`/`replace`). **uiRoutes:** `pluginContext.ts:504–526` (Kernel injiziert `pluginId`). **Tools:** `pluginContext.ts:444–479` (`tools.register(spec, handler)`).
- **Privacy Shield v4:** `privacyReceipt.ts:99–116` (`datasetId` + `digestText`, LLM sieht nie Rohwerte); `harness-plugin-privacy-guard/src/service.ts:149–177` (`internAndCount`, Rohzeilen bleiben server-seitig).
- → plan.md §1.2/§2.1.5/§4 bestätigt. Proof kann sich an dieses Muster halten.

> **Naming-Fund (für R16/ADR-0003, Fakt):** Externe/Feature-Plugins heißen im Monorepo `@omadia/plugin-<domain>` (z. B. `@omadia/plugin-web-search`, `@omadia/plugin-office`); Infra/Harness `@omadia/harness-<subsystem>`; Referenz-Agents `@omadia/agent-*`. Alle `"private": true` (kein npm-Publish). → Konkrete Datenbasis für ADR-0003: die Plan-Hypothese `@byte5/proof-*` ist **nicht** die Hauskonvention; `@omadia/plugin-proof-*` läge näher. Entscheidung in ADR-0003.

---

## Neue Funde (über A1–A8 hinaus)

| # | Fund | Quelle | Konsequenz |
|---|---|---|---|
| F1 | **Appendix A formalisiert die Schemas NICHT** — *„Normative JSON Schemas (to be finalized in v1.0) will cover …"* | SPEC Appendix A (Z. 938) | **Bestätigt die §1.4-Kernthese hart:** `proof.objects@1` ist die erste Formalisierung → Schema-Backflow-Pflicht real. → #2/#4 |
| F2 | **KG ohne Encryption-at-rest** (= A5) | s. o. | `body`-Verschlüsselung auf App-Ebene nötig → ADR-0004, `proof.store@1`/`proof.objects@1` |
| F3 | **IOTA-First-Party-Tooling pre-GA** (Identity Beta, Gas Station pre-1.0, Notarization/Hierarchies Alpha) | SPEC §13.2 | Neues Reife-Risiko für Phase 1/2 → siehe R19 |
| F4 | **IOTA Notarization + Hierarchies als Kandidaten-Bindings** für Anker (Locked/Dynamic) + Trust-List | SPEC §13.2 (Z. 848), §17 (Z. 925) | `proof.anchor-iota@1` (#8) sollte Binding an IOTA Notarization prüfen statt eigene Anker-Objekte zu rollen |
| F5 | **Appendix-A-Schemaliste reicher als Plan-Matrix** — zusätzlich: `execute`, `terminate`, `amend`, `rescind`, `counter_offer`, **quorum-Object**, **Anchor-Record**, **outcome-Directive**, **suite-registry** | SPEC Appendix A (Z. 940–945) | Schema-Matrix #2 ergänzen |
| F6 | **`anp_version`-Registry ist content-addressed & versioniert** — ein Hash pro Version, Verifier MUSS gegen gepinntes Artefakt validieren | SPEC Appendix A (Z. 949) | Operationalisiert R14b: `proof.objects@1` pinnt `anp_version` + Hash → #4 |
| F7 | **Conformance-Levels stützen die Phasen-Reihenfolge:** „Minimal" = Core + ein Pillar (Memorandum/Attest) **ohne Settlement**; Dispute-Conformance **verlangt** ein Settlement-Profil | SPEC §14 (Z. 880) | Bestätigt: Phasen 1–2 = valide Minimal-Conformance; Escrow/Dispute zu Recht Phase 5 |

---

## Auswirkungen auf die Issues

- **#1 (Stufe A):** A1–A8 abgearbeitet. ADR-0004 muss F2 adressieren (KG-Encryption); ADR-0003 nutzt den A8-Naming-Fund; das Custody-Gate ist durch A7 konkret begründet; A3-Pre-GA-Caveat (F3) als Risiko führen.
- **#2 (Schema-Matrix):** F1 bestätigt die Pflicht; Matrix um F5 (execute/terminate/amend/rescind/counter_offer/quorum/anchor-record/outcome/suite-registry) ergänzen; Chain- vs. Attachment-Klassen + `receipt`-non-anchored + `attest`/„attestation"-Token aufnehmen; F6 (anp_version-Pinning).
- **#4 (`proof.objects@1`):** Det. Proof-Ordnung + zwei kanonische Formen + JCS bestätigt (A1); `anp_version`-Pinning gegen content-addressed Registry (F6).
- **#6 (`proof.identity@1`):** Pairwise-DID = MUST (A2); Custody-Skalierung offen (A7/R11) → blockiert zu Recht.
- **#8 (`proof.anchor-iota@1`):** IOTA-Profil bestätigt (A3); Binding an IOTA Notarization prüfen (F4); Pre-GA-Reife (F3).

## Vorschlag: ein neues Risiko in plan.md §6

> **R19 — Reife der IOTA-First-Party-Abhängigkeiten.** IOTA Identity (Beta, kein Stable-Release), Gas Station (pre-1.0), IOTA Notarization/Hierarchies (Alpha) sind die Tools, auf denen Phase 1 (PQC-VCs, custodial Signing) und Phase 2 (Anchor/Sponsoring) aufsetzen. **Schwere: Mittel.** Mitigation: `anchor-mock` hält Phase 0–1 chain-frei (bereits geplant); IOTA-Abhängigkeiten erst in Phase 2 scharf; Versionen pinnen, Pre-GA-API-Brüche einplanen; im PoC (ANP §17 Phase 2) die IOTA-Foundation-Komponenten früh evaluieren.

---

*Belege gelesen 2026-06-15 gegen `byte5ai/anp@main` (SPEC.md) und `byte5ai/omadia@main`. Datei:Zeile-Referenzen beziehen sich auf den Stand dieser Commits. Fakten direkt belegt; Inferenzen als solche markiert.*
