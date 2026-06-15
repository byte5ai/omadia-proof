# research.md — verification of the plan's assumptions A1–A8

> **Purpose:** Stage-A artifact (plan.md §7 Stage A, §11). The plan marks eight load-bearing claims about `byte5ai/anp` (SPEC.md) and `byte5ai/omadia` (code) as **"not checkable from the plan alone"** (A1–A8). The Codex review could not read these repos. This document checks them against the real sources.
>
> **Method:** `byte5ai/anp` @ `main` (SPEC.md, 1033 lines) read directly; `byte5ai/omadia` @ `main` (TS monorepo) via three targeted code investigations. Each claim grounded in file:line or `SPEC §`. **Fact** (evidenced from source) separated from **inference** (my conclusion). Evidence read 2026-06-15.
>
> **Source commits:** anp `main` (pushed 2026-06-15T16:38Z) · omadia `main` (pushed 2026-06-15T18:49Z). SPEC author per line 1031: "Christian (byte5 GmbH)" — the RFC belongs to byte5 (relevant for §1.4).

## Verdict overview

| # | Assumption (short) | Verdict | Key evidence |
|---|---|---|---|
| A1 | ANP defines object types/status/standalone-attest/memorandum-terminal/deterministic proof order as described | ✅ **confirmed** | SPEC §6.1, §6.2, §6.3, §7.2, §8.7 |
| A2 | Pairwise DIDs strong enough for R3 | ✅ **confirmed (stronger: MUST)** | SPEC §6.2 (m6), §12 |
| A3 | PQC suites, SD-JWT mandate, IOTA-Rebased profile | ✅ **confirmed** (with pre-GA caveat) | SPEC §6.5, §5.3/§6.6, §13.2 |
| A4 | Plugins have no raw Postgres pool, only Secrets/Memory via `onMigrate` | ✅ **confirmed** | plugin-api `pluginContext.ts`, `vault.ts` |
| A5 | `knowledgeGraph@1` fits persistence/ACL/retrieval/**encrypted-at-rest** | ⚠️ **partial — encryption-at-rest missing** | KG migrations, `inMemoryKnowledgeGraph.ts` |
| A6 | No plugin-extensible pre-dispatch hook; verifier precedent | ✅ **confirmed** | `turnHooks.ts`, `orchestrator.ts`, `harness-verifier/plugin.ts` |
| A7 | Vault scales for per-user/-agent custodial keys | ⚠️ **unconfirmed — designed for O(100) agents** | `fileVault.ts`, `vault.ts` |
| A8 | Plugin signing/packaging/`provides`/`requires`/`uiRoutes`/tools/Privacy-Shield as described | ✅ **confirmed** | plugin-api, `privacyReceipt.ts`, `harness-plugin-privacy-guard` |

**Bottom line:** 6 confirmed, 2 with a substantial caveat (A5, A7). Both caveats **strengthen** the plan (R15 store decision, R11 custody scale) rather than breaking it — they make the respective Stage-A gates more urgent and more concrete. Plus four new findings below.

---

## A1 — Object model (✅ confirmed)

- **Object types.** SPEC §6.1 (lines 290–294) lists the `type` enum **exactly** as plan.md §1.1: `offer, counter_offer, accept, approve, execute, terminate, memorandum, amend, rescind, attest, witness, revoke, assert, dispute, evidence, rule, appeal, enforce, settle, receipt`.
- **Status.** SPEC §6.2 (line 352): coarse on-chain `active | superseded | revoked | disputed | enforced`; **fine states derived off-chain by replaying the object types** (m5). Matches plan.md §4.2 (`derived_state` = cache, never source of truth) exactly.
- **Standalone attestation.** SPEC §8.7 (line 660): *"A standalone notarization (no contract) skips `REQUESTED` — a Notary simply issues and anchors."* → confirms plan.md §3 table (Human→Nobody).
- **Memorandum terminal.** SPEC §7.2 (line 452): *"anchored once, reaching `ACCEPTED` on anchoring, which is terminal (no `EXECUTED`/`DISPUTED` tail) … cost ≈ one anchor, no settlement."* → matches plan.md §1.1/§5.1 exactly.
- **Deterministic proof order (R6 keystone).** SPEC §6.3 (line 396): *"`proof[]` entries MUST appear in the same order as their corresponding `signers[]` entries, so that every party assembles a byte-identical anchored form and derives the same `object_hash`."* Plus two canonical forms (signing form without `proof`; anchored form with `proof`) + JCS RFC 8785 (§6.3, lines 387–398). → fully covers the `proof.objects@1` interface (#4) and R6.

> **Nuance (for #2/#4, inference):** `accept`/`approve`/`witness`/`evidence`/`revoke` are **attachment objects** (SPEC §6.1 lines 314–322): `previous_hash: null`, target hash in `body` (`accepts_hash`/`approves_hash`/`attestation_hash`/`dispute_hash`/`revokes`), `sequence` = target's `sequence`. The rest are **chain objects**. The schema model must capture these two classes. `receipt` is **non-anchored / transport-only** (§6.1 line 329, §6.4) — it has a schema but is exempt from anchoring/DA. Also: the SPEC wobbles between `attest` (the enum token §6.1/§8.2) and "attestation" (prose §2.3) — Proof should use `attest` and flag the inconsistency in the ANP PR.

## A2 — Pairwise DIDs (✅ confirmed, even MUST)

- SPEC §6.2 (m6, line 354): *"For privacy-sensitive Threads, parties SHOULD — and where a natural person is identifiable, **MUST** (§12) — use per-relationship or per-Thread **pairwise DIDs**."*
- SPEC §12 (line 812): *"For Threads in which a natural person is identifiable … implementations **MUST** use per-relationship or per-Thread pairwise DIDs."*
- → plan.md R3 is not just justified but **normatively required**. `proof.identity@1` (#6) must generate pairwise DIDs automatically — not an optional feature.

## A3 — PQC / SD-JWT / IOTA profile (✅ confirmed, pre-GA caveat)

- **PQC suites.** SPEC §6.5 (line 418): *"Suites MAY use NIST post-quantum signatures — **ML-DSA (FIPS 204)** or **SLH-DSA (FIPS 205)** — today, or hybrid classical+PQC."* → confirms plan.md §4/§5.2 (`anp-suite-2`, ML-DSA/SLH-DSA).
- **SD-JWT mandate disclosure.** SPEC §5.3 (line 268) + §6.6 (line 427): mandates **SHOULD** be SD-JWT VCs with predicates, **strongly RECOMMENDED** for `contracting` scope (against cap leak). → confirms plan.md §5.2 `proveDisclosure`.
- **IOTA-Rebased profile.** SPEC §13.2 (lines 835–851): reference profile, Starfish consensus p99 ≈ 312 ms (sub-second), anchor ≈ 0.001 IOTA, Gas-Station sponsoring, native randomness beacon + ECVRF. **Off-chain PQC ready:** IOTA Identity (v1.7+) issues ML-DSA-44/65/87, SLH-DSA, FALCON, hybrid (line 846). → confirms plan.md §13.2/Phase 2.
- **R5 confirmation (fact):** SPEC §6.5 (line 423) + §13.2 (line 850): chain-native accounts are **Ed25519 (not PQC)** → escrow custody + tx submission inherit non-PQC crypto. Matches plan.md R5 exactly.

> **New caveat (fact, → new risk below):** the IOTA first-party tools are **pre-GA**: IOTA Identity **Beta** (v1.9.9-beta.1, "no stable release yet"), Gas Station v0.5.2 (pre-1.0), IOTA Notarization + Hierarchies **Alpha** (SPEC §13.2 lines 847–851). A real dependency-maturity question for Phase 1 (identity/PQC) and Phase 2 (anchor).

## A4 — Plugin storage isolation (✅ confirmed)

- `PluginContext` exposes **no** DB pool — full field list (`pluginContext.ts:31–147`): `secrets`, `config`, `services`, `memory`, `tools`, `routes`, `uiRoutes`, `jobs`, `knowledgeGraph`, `llm`, … no SQL/pool.
- `MigrationContext` overrides only `secrets` with `SecretsReadWriteAccessor` (`pluginContext.ts:1090–1100`); write access is **namespaced per `agentId`** (`vault.ts:30–36`, `fileVault.ts:39` `byAgent = new Map<string, Map<string,string>>()`). Memory is structurally per-plugin isolated (`pluginContext.ts:78–84`).
- → confirms plan.md §4.2 fact: a logical data model, no own Postgres. Store mechanism remains a Stage-A decision (R15).

## A5 — Knowledge Graph as object store (⚠️ partial — **encryption-at-rest missing**)

| Property | Present? | Evidence |
|---|---|---|
| Persistence | ✅ | `knowledgeGraph.ts:206–216` (`createMemorableKnowledge`/`getMemorableKnowledge`); Neon `0001_graph_init.sql:11–22` (`graph_nodes`) |
| ACL/permission | ✅ | `inMemoryKnowledgeGraph.ts:1015–1047` (`addOwner`/`removeOwner`, `acl_owners`); `0018_acl_owners.sql` (GIN index, audit table) |
| Retrieval/query | ✅ | `knowledgeGraph.ts:221–231` (`listMemorableKnowledgeFor`, ACL-gated) |
| **Encrypted-at-rest** | ❌ **missing** | `0001_graph_init.sql:8` loads `pgcrypto` only for `gen_random_uuid()`; properties stored as **plaintext JSONB**; no field/row encryption |

> **Consequence (inference, important):** plan.md §4.2 requires the PII in `body` to be **"encrypted at rest via the existing mechanism"** — and implicitly assumes the KG brings that. **It does not.** If the KG is chosen as the store (R15/ADR-0004), `body` encryption must happen at the **application layer** before it enters the KG (e.g. via `proof.signer`/vault crypto), or the store needs its own encryption layer. A concrete requirement for ADR-0004 and for `proof.store@1`/`proof.objects@1` — not a show-stopper, but to be solved explicitly.

## A6 — No pre-dispatch hook; verifier precedent (✅ confirmed)

- **Hook points are post-hoc.** `turnHooks.ts:20–27` defines exactly four `TurnHookPoint`: `onBeforeTurn` (before the first LLM inference, **not** before dispatch), `onAfterToolCall`, `onAfterTurn`, `onVerifierBlocked` — **none pre-dispatch**.
- **Dispatch is kernel-only.** `orchestrator.ts:3038–3100` (`dispatchTool`) branches straight into Privacy-Shield → sub-agent → `dispatchToolInner` — no plugin-registerable hook before it.
- **Verifier precedent, verbatim.** `harness-verifier/src/plugin.ts:17–48`: *"the Orchestrator class itself is ~1k LOC and not yet plugin-extractable … we keep the wrapper kernel-side."* The gating `VerifierService` lives kernel-side in `harness-orchestrator/src/verifierService.ts` and checks **after** the turn (post-answer), not pre-dispatch.
- **Correction (fact):** the actual `orchestrator.ts` is **3,691 lines** (not ~1k — the comment is stale); the "not plugin-extractable" rationale therefore holds *more* strongly.
- → plan.md R10 (escalation gate as a kernel-side pre-dispatch PR) is the only architecturally clean option. ADR-0005 (two-phase write) is app-level, not a kernel dispatch gate — confirming the gate genuinely has to be new kernel code.

## A7 — Vault scaling for custodial keys (⚠️ unconfirmed — designed for O(100) agents)

- **Crypto confirmed:** `fileVault.ts:146–157` AES-256-GCM, 32-byte master key, random IV; `fileVault.ts:18`.
- **But the design assumption is small:** `fileVault.ts:23` comment *"target scale (O(100) agents × O(10) keys)"* — i.e. ~1,000 secrets. Persistence serializes **the whole vault as one JSON blob** and encrypts it atomically (`fileVault.ts:132–144`).
- **No rotation, no per-secret derivation:** `fileVault.ts:186–221` (`resolveMasterKey`) — no rotation path (changing the master key ⇒ re-encryption); all secrets under one shared master key. A nightly backup to S3/Tigris exists (`vaultBackup.ts:65–224`).
- **Per-agent only, not per-user:** `vault.ts:5–6` *"Secrets are ALWAYS namespaced by agent identity.id"* — no second user tier.

> **Consequence (inference):** custodial keys **per user AND per agent** (potentially thousands) do **not obviously** fit today's file vault (single-envelope JSON, O(100)-agents design, no rotation). This **confirms R11** and makes the Stage-A custody gate (#1) concrete: either a vault extension or the **second, kernel-side custody service** considered in plan §4. This decision rightly blocks `proof.identity@1` (#6).

## A8 — Plugin packaging & runtime surface (✅ confirmed)

- **Signing:** `harness-plugin-office/src/signing.ts:1–60` (HMAC-SHA256, timing-safe). **ZIP guardrails:** `src/plugins/zipExtractor.ts:20–60` (extension allowlist, entry/byte limits, symlink/path-escape protection).
- **`provides`/`requires` + `@major`:** `pluginContext.ts:243–296` (`CapabilityRef {name, major}`, `parseCapabilityRef`); real manifests: `harness-plugin-web-search/manifest.yaml:165` (`provides: ["webSearch@1"]`). Resolver: `src/plugins/capabilityResolver.ts`.
- **Services:** `pluginContext.ts:314–338` (`get`/`provide`/`replace`). **uiRoutes:** `pluginContext.ts:504–526` (kernel injects `pluginId`). **Tools:** `pluginContext.ts:444–479` (`tools.register(spec, handler)`).
- **Privacy Shield v4:** `privacyReceipt.ts:99–116` (`datasetId` + `digestText`, the LLM never sees raw values); `harness-plugin-privacy-guard/src/service.ts:149–177` (`internAndCount`, raw rows stay server-side).
- → confirms plan.md §1.2/§2.1.5/§4. Proof can follow this pattern.

> **Naming finding (for R16/ADR-0003, fact):** external/feature plugins are named `@omadia/plugin-<domain>` in the monorepo (e.g. `@omadia/plugin-web-search`, `@omadia/plugin-office`); infra/harness `@omadia/harness-<subsystem>`; reference agents `@omadia/agent-*`. All `"private": true` (no npm publish). → concrete data for ADR-0003: the plan's hypothesis `@byte5/proof-*` is **not** the house convention; `@omadia/plugin-proof-*` is closer. Decision in ADR-0003.

---

## New findings (beyond A1–A8)

| # | Finding | Source | Consequence |
|---|---|---|---|
| F1 | **Appendix A does NOT formalize the schemas** — *"Normative JSON Schemas (to be finalized in v1.0) will cover …"* | SPEC Appendix A (line 938) | **Confirms the §1.4 thesis hard:** `proof.objects@1` is the first formalization → the schema-backflow obligation is real. → #2/#4 |
| F2 | **KG has no encryption-at-rest** (= A5) | see above | `body` encryption needed at app layer → ADR-0004, `proof.store@1`/`proof.objects@1` |
| F3 | **IOTA first-party tooling pre-GA** (Identity Beta, Gas Station pre-1.0, Notarization/Hierarchies Alpha) | SPEC §13.2 | New maturity risk for Phase 1/2 → see R19 |
| F4 | **IOTA Notarization + Hierarchies as candidate bindings** for the anchor (Locked/Dynamic) + trust list | SPEC §13.2 (line 848), §17 (line 925) | `proof.anchor-iota@1` (#8) should evaluate binding to IOTA Notarization instead of rolling its own anchor objects |
| F5 | **Appendix-A schema list richer than the plan matrix** — additionally: `execute`, `terminate`, `amend`, `rescind`, `counter_offer`, the **quorum object**, the **anchor record**, the **outcome directive**, the **suite registry** | SPEC Appendix A (lines 940–945) | Extend the schema matrix (#2) |
| F6 | **The `anp_version` registry is content-addressed & versioned** — one hash per version, verifiers MUST validate against the pinned artifact | SPEC Appendix A (line 949) | Operationalizes R14b: `proof.objects@1` pins `anp_version` + hash → #4 |
| F7 | **Conformance levels support the phase order:** "Minimal" = Core + one pillar (memorandum/attest) **without settlement**; dispute conformance **requires** a settlement profile | SPEC §14 (line 880) | Confirms: Phases 1–2 = valid Minimal conformance; escrow/dispute rightly in Phase 5 |

---

## Impact on the issues

- **#1 (Stage A):** A1–A8 done. ADR-0004 must address F2 (KG encryption); ADR-0003 uses the A8 naming finding; the custody gate is concretely justified by A7; carry the A3 pre-GA caveat (F3) as a risk.
- **#2 (schema matrix):** F1 confirms the obligation; extend the matrix with F5 (execute/terminate/amend/rescind/counter_offer/quorum/anchor-record/outcome/suite-registry); capture chain vs. attachment classes + `receipt` non-anchored + the `attest`/"attestation" token; F6 (anp_version pinning).
- **#4 (`proof.objects@1`):** deterministic proof order + two canonical forms + JCS confirmed (A1); `anp_version` pinning against the content-addressed registry (F6).
- **#6 (`proof.identity@1`):** pairwise DID = MUST (A2); custody scaling open (A7/R11) → rightly blocked.
- **#8 (`proof.anchor-iota@1`):** IOTA profile confirmed (A3); evaluate binding to IOTA Notarization (F4); pre-GA maturity (F3).

## Proposal: a new risk in plan.md §6

> **R19 — Maturity of the IOTA first-party dependencies.** IOTA Identity (Beta, no stable release), Gas Station (pre-1.0), IOTA Notarization/Hierarchies (Alpha) are the tools Phase 1 (PQC VCs, custodial signing) and Phase 2 (anchor/sponsoring) build on. **Severity: medium.** Mitigation: `anchor-mock` keeps Phases 0–1 chain-free (already planned); IOTA dependencies only sharp in Phase 2; pin versions, plan for pre-GA API breaks; evaluate the IOTA Foundation components early in the PoC (ANP §17 Phase 2).

---

*Evidence read 2026-06-15 against `byte5ai/anp@main` (SPEC.md) and `byte5ai/omadia@main`. File:line references are relative to those commits. Facts evidenced directly; inferences marked as such.*
