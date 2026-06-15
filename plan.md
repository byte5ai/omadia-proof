# omadia Proof — Implementation Plan

> **Working name:** `omadia Proof` · **ANP spec:** v0.3-draft · **omadia:** pre-1.0 public preview
> **Repo:** `byte5ai/omadia-proof` (its own repo, a plugin bundle) + **one** kernel-side hook in `byte5ai/omadia` (planned; see §4 and R10).
> **Status of this document:** plan / draft — **Codex review incorporated (2026-06-15, see §11)**. Stored as `plan.md` in `byte5ai/omadia-proof`.
>
> **Guiding principle 1 (UX):** the user sees *proofs*, never cryptography. ANP/DLT is the engine room, not a product feature.
> **Guiding principle 2 (spec fidelity):** omadia is the **first** ANP implementation. The RFC stays the canonical source — not the code. Every schema Proof defines **MUST** flow back as a PR into `byte5ai/anp` (mandatory, see §1.4 and §11).
>
> **Reference convention:** `ANP §X` refers to a section of the ANP specification (`SPEC.md`). A **bare** `§X` (without `ANP`) refers to a section of *this* plan. The distinction is deliberate, because ANP and this plan partly share section numbers (e.g. ANP §5.3 = mandates ≠ §5.3 = UI routes of this plan).

---

## 0. What this is about (in one paragraph)

ANP is a DLT-neutral trust layer that turns every binding action into a signed, schema-valid, hash-chained, on-chain *anchored* **ANP Object** — the content stays off-chain, only a hash + status go to the chain. omadia is a self-hostable agentic OS with signed plugins, a capability registry, a vault, and an "every action leaves a receipt" philosophy. **omadia Proof** is the module that embeds ANP into omadia and capsules its entire complexity — DIDs, keys, suites, anchoring, escrow — so that a manager or employee without an IT background experiences just one thing: *"this is now bindingly recorded, verifiable, tamper-proof."*

The central tension this plan resolves: ANP assumes every actor manages keys and operates wallets (exactly the friction that sinks DLT adoption with humans) — the ANP spec addresses this explicitly via the *symbiosis thesis* (ANP §1.4): **agents absorb the wallet/key/fee friction.** In omadia that agent is already there. Proof turns "the human must operate a wallet" into "the human taps *Confirm*, the custodial layer signs."

---

## 1. Fact base (verified from the repos, not assumed)

This section separates **fact** (read from SPEC.md / omadia code) from **inference** (my conclusion from it). Inferences are marked as such.

> **Review note (Codex):** several of the claims listed below as "fact" about the ANP spec and the omadia code are load-bearing for the whole plan but not checkable from this document alone. They are listed in §11 as **assumptions-to-verify (A1–A8)**. **As of 2026-06-15: verified against the real repos → `research.md` (6/8 confirmed; A5/A7 with a caveat).**

### 1.1 What ANP actually specifies

| Concept | Fact from SPEC.md |
|---|---|
| **The three pillars** | ① Contract (binding multi-party agreement) · ② Notarize (a third party witnesses a fact) · ③ Resolve (neutral arbiter, evidence, binding ruling). ANP §1.3, ANP §7–§9. |
| **The spine artifact** | Every binding action = an **ANP Object**: schema-valid, signed (PQC-capable, agile suite), hash-chained to its predecessor, anchored on-chain by hash+status only. ANP §4.2, ANP §6.1. |
| **On-/off-chain** | Off-chain: the full object. On-chain: only `{object_hash, object_type, thread_ref, status, anchored_by, timestamp, locator?, outcome?}`. No payload, no PII. ANP §6.2, ANP §6.4. |
| **Identity** | Every actor is a **W3C DID**; all non-identity assertions are **W3C VC 2.0**. No anonymous binding. ANP §5.1, ANP §5.2. |
| **Mandate** | A principal issues an agent a **mandate VC**: `scope`, `constraints` (max_value, aggregate_value, allowed_counterparties, **escalation_threshold**), `sub_delegation`, `not_before`/`expires`, `status_list`. Actions ≥ threshold need a principal **`approve`** co-signature. ANP §5.3. |
| **Thread** | A hash-chained sequence of objects under a shared `thread_ref` (UUID). ANP §2.3. |
| **Object types** | `offer, counter_offer, accept, approve, execute, terminate, memorandum, amend, rescind, attest, witness, revoke, assert, dispute, evidence, rule, appeal, enforce, settle, receipt`. ANP §6.1. |
| **Memorandum** | *The* low-value/high-frequency case: a single object co-signed by all parties, anchored **once**, terminal at `ACCEPTED`, **no escrow**. Cost ≈ one anchor. ANP §7.2. |
| **Notarization = authorship + anchor** | An attestation is a VC (`type: attest`) with `witnessing: observed\|relayed`, optionally an M-of-N witness quorum. "VC (who says what) + on-chain anchor (independent timestamp/ordering)". ANP §8.2, ANP §5.2. |
| **Standalone notarization** | A notarization **without a contract** skips `REQUESTED` — a notary "simply issues and anchors". ANP §8.7. |
| **Dispute (optimistic)** | `assert` → challenge window → (no challenge ⇒ finalizes | `settle` co-signed ⇒ immediate | `dispute` ⇒ evidence → `rule` → optional `appeal` → `enforce`). Bonds mandatory on the assertive path. ANP §9.4. |
| **Small-claims profile** | For micro-values where arbiter fees + bonds exceed the dispute value: a pre-agreed `split` formula instead of a forum, bonds optional, deterrence purely reputational. ANP §9.4. |
| **Four enforcement paths** | `ruling` (arbiter), `uncontested_assertion` (happy path), `mutual_settlement` (unanimous waiver), `formula_split` (small-claims). Only here do numbers go on-chain (`outcome`). ANP §6.2.1. |
| **Cost goal** | "Sub-second finality and near-zero per-action cost, so that recording even trivial determinations is economically sensible." ANP §3.1 Goal 8. Reference chain: IOTA Rebased. ANP §13.2. |
| **Honest limits** | (a) No token, no marketplace function, **no legal framework** — legal recognition is explicitly an open question (ANP §16.5, ANP §3.2). (b) Escrow custody + transaction submission inherit the *native* chain crypto (not PQC today). (c) Aggregate caps are not counterparty-checkable. |

### 1.2 What omadia actually is (from the code)

| Concept | Fact from the repo |
|---|---|
| **Stack** | TypeScript monorepo. `middleware/` (Node + Postgres/pgvector), `web-ui/` (Next.js admin UI), separately distributed plugin ZIPs. |
| **Plugin model** | Everything is a plugin behind `@omadia/plugin-api`. Plugins = **signed ZIPs** (ADR-0001), `node_modules` baked in, no npm runtime trust. |
| **Capability registry** | Versioned capabilities `"<name>@<major>"`. A provider declares `provides: ["x@1"]` in its manifest, calls `ctx.services.provide("x", impl)`, a consumer calls `ctx.services.get("x")`. ADR-0003. Decoupled: `requires` matches *any* provider of the same name/major. |
| **Package naming** | Kernel/infra: `harness-*`. Feature plugins: `agent-*` / `harness-plugin-*`. (Already taken: `verifier@1` = LLM claim verifier, **not** ANP — avoid the name collision.) **→ Consequence for Proof: the package prefix is an open naming decision, see Stage A / R16.** |
| **PluginContext** | Scoped per plugin. Access to `secrets` (vault), `config`, `services`, `tools` (native tools to the orchestrator), `routes` (HTTP), `uiRoutes` (UI extensions), `jobs` (cron), `knowledgeGraph`, `notifications`, `llm`. Plugin code **never** imports the vault directly. |
| **Vault** | AES-256-GCM, `VAULT_KEY` mandatory in prod. Exactly the place for custodial key material. |
| **Privacy Shield v4** | **Architecturally critical:** raw tool results are interned server-side behind a `datasetId`; only an identity-free *digest* crosses the LLM line. Identity-/ordering-critical work runs in trusted server code. ADR + `privacyReceipt.ts`. |
| **Two-phase write** | ADR-0005: write actions are explicitly typed, two-stage (preview → confirm), "draft-by-default". "No agent-initiated write reaches a live system without a human seeing the exact change first." |
| **Audit trail** | "Every action leaves a receipt" — per-run trace + call-stack viewer. Local, traceable — but **not** tamper-proof across trust boundaries. (See §2.) |

### 1.3 The decisive inference

> **Inference:** omadia's existing "receipt" (run trace) and an ANP proof are *complementary, not redundant*. The run receipt proves *to yourself* what *your* system did (local audit, single-party-trusted). An ANP proof proves *to a mistrustful counterparty or regulator* what was agreed/witnessed *between* parties across trust boundaries — externally anchored, unalterable by either side. Proof is the bridge from the *internal log* to the *external, shared fact*. Reasoning: the run receipt lives in your Postgres DB (§1.2), which ANP §1.2 explicitly classifies as untrusted single-party storage.

---

### 1.4 MANDATORY: schema backflow into the ANP RFC

> **This requirement is non-negotiable and not a "nice-to-have".**

omadia is the first implementation of ANP. The core package (`proof.objects@1`) therefore effectively defines the JSON schemas per object type that ANP §6.1 / Appendix A only references today but does not formalize. Whoever implements first *sets* the standard — intended or not.

This implies a hard rule for every agent/developer working on Proof:

1. **No object schema is created only in Proof code.** Every schema Proof defines or changes for an ANP object type or a VC **MUST** be submitted in the same unit of work as a pull request against `byte5ai/anp` (Appendix A).
2. **The RFC is the source of truth, not the code.** On divergence between a Proof schema and the SPEC, the SPEC wins; the code is adjusted, not the other way around. Proof pins `anp_version` and references the schema version from the RFC.
3. **A schema counts as "done" only once the ANP PR is open.** A merged Proof schema without a corresponding ANP PR is a rule violation, not progress. This is a Done criterion in every phase that activates new object types.

#### 1.4.1 Schema-backflow matrix (Codex C5 — replaces the earlier prose)

This matrix is the *single* source for which object/VC type is first activated in which phase and which ANP PR must be open for it. It is finalized in **Stage A** and is a Done gate of every phase. "Activated" = the schema is first built/validated productively; a schema is defined **only once** (the earlier duplication of `memorandum` in P0 *and* P3 was a mistake — P0 *defines* it, P3 only *uses* it for the co-signing flow).

| Object/VC type | First activation | ANP PR required | Note |
|---|---|---|---|
| `memorandum` | Phase 0 (defined), Phase 1 (solo), Phase 3 (co-signed) | ✅ Phase 0 | terminal at `ACCEPTED`, no escrow |
| `attest` (+`witness`) | Phase 2 | ✅ Phase 2 | `observed`/`relayed`; M-of-N optional later |
| `offer` / `counter_offer` / `accept` | Phase 3 | ✅ Phase 3 | offer/accept cycle, no escrow yet |
| `approve` | Phase 4 | ✅ Phase 4 | principal co-signature at the escalation gate |
| **Mandate VC** | Phase 4 | ✅ Phase 4 | its own VC schema (`scope`/`constraints`/`status_list`) — was **missing** from the old gate |
| Status-list credential | Phase 4 | ✅ Phase 4 *(if Proof defines it)* | otherwise reference the ANP reference |
| `assert` / `dispute` | Phase 5 | ✅ Phase 5 | optimistic dispute entry |
| `evidence` | Phase 5 | ✅ Phase 5 | was **missing** from the old gate |
| `rule` / `appeal` / `enforce` / `settle` | Phase 5 | ✅ Phase 5 | `appeal` was **missing** from the old gate |
| `receipt` | Phase 1 (store delivery) or Phase 5 (settlement) | ✅ at first activation | DA/settlement receipt — was **missing** from the old gate |

**Reasoning:** if the first implementation and the spec drift apart, it devalues the RFC — for its author (byte5) a direct own goal. The effort of keeping schemas in sync is trivial against the damage of a spec that contradicts its own reference implementation.

---

## 2. The product promise (user first)

The user doesn't think in pillars. They think in one question: **"do I want this to be undeniable later?"** If yes, they press a button. Proof translates that button into the right ANP pillar:

| What the user wants | They see | What Proof does underneath |
|---|---|---|
| "Record that I/we said/decided this" | **Record** | Memorandum (1 party = self-attest; N parties = co-signed memorandum) |
| "Confirm that something is/was so" | **Witness** | Attestation (`witnessing: observed\|relayed`) |
| "Make an arrangement binding" | **Agree** | Contract: offer/accept (+ optional escrow/criteria) |
| "Have a dispute about it settled" | **Resolve** | Dispute: assert/dispute/rule/enforce |

The four words — **Record · Witness · Agree · Resolve** — are the *entire* user-facing surface. Everything else is capsuled.

### 2.1 The non-negotiable UX laws

These follow directly from "the users are not IT experts" + ADR-0005 + Privacy Shield v4:

1. **No crypto vocabulary in the UI.** Never "hash", "DID", "wallet", "anchor", "suite", "chain", "gas". Instead: *proof, identity, recorded, verifiable, receipt no.* The technical receipt is reachable via "View details", never forced.
2. **Identity is invisible but real.** Every omadia user and every agent automatically gets a DID + keys on first contact, held in the vault (custodial). The user signs by *Confirming*, not by handling keys. (Architecture in §4; the DID method is a Stage-A decision, R12.)
3. **Confirm is the only human gesture.** Exactly the two-phase pattern from ADR-0005, extended: a preview of the exact, human-readable action → one tap → done. Proof is conceptually "the first write connector that makes the action binding *between organizations*".
4. **The anchoring step is asynchronous and quiet.** Sub-second finality (ANP §13.2) means: the user sees "✓ Recorded" almost instantly; confirmation of the chain anchoring trickles in in the background and only updates a discreet status icon (like "sent → delivered" in messengers). *Precondition: a robust async confirmation path with retry/timeout/idempotency — see R14.*
5. **Hashes/DIDs never cross the LLM line.** Strict Privacy-Shield-v4 mandate: Proof identifiers are interned server-side; the orchestrator/LLM sees only opaque references (`proofRef`) and human-readable summaries, never raw crypto. **This has a consequence for the tool surface (§5.1):** `thread_ref`, `counterparties[]` and `evidence[]` are also *linkable identifiers / raw data* and must not cross the LLM boundary raw. The concrete dataflow per tool (what does the LLM see, what is interned server-side, which opaque IDs replace raw references) is a **mandatory Stage-A spec, R13.** (Reasoning: a hash/thread-ref in the context window is both a privacy leak via the `anchored_by` link graph ANP §6.2 and a hallucination source.)
6. **Honesty about legal force.** Where an action legally needs a notary/witness, Proof says so *clearly* and positions itself as "a technically complete, tamper-proof proof — the legal form (notary/eIDAS) is separate". No overpromise. (ANP §3.2/ANP §16.5 of the spec demands this stance.) **Wording discipline:** in the *product UI* never "legally binding"; "binding" in this plan means *technically binding between the parties per ANP*, not legally enforceable (R2).

---

## 3. The four topologies — designed as first-class citizens

We want to secure **Human-to-Nobody, Human-to-Human, Agent-to-Human, Agent-to-Agent**. ANP's mandate model (ANP §5.3) is built exactly on the `Human → Agent → Agent → Human` axis; the four topologies are points on this axis. Here's how Proof maps them:

| Topology | What it is | ANP mechanics | Who signs | UX entry point | Phase |
|---|---|---|---|---|---|
| **Human → Nobody** | Self-notarization for the record; an immutable milestone, *no* counterparty now | **Standalone attestation** (`witnessing: observed` by the user about themselves) **or** a single-party **memorandum**; anchored, terminal. ANP §8.7 allows "issue and anchor" without a contract | Only the user (custodial) | "Record" — solo | 1 |
| **Human → Human** | Two people record an arrangement | Co-signed **memorandum** (record) *or* offer/accept (binding, optional escrow) | Both users (each custodial) | "Record" / "Agree" — with a counterparty | 3 |
| **Agent → Human** | An agent proposes, a human accepts (or vice versa) | offer (agent under mandate) → accept (human) **or** an agent attestation + human `approve` on threshold exceedance ANP §5.3 | Agent (mandate) + human (`approve`/`accept`) | Agent generates a proposal → human confirm card | **4** *(see §3.2)* |
| **Agent → Agent** | Two agents close end-to-end — **the favorite** | Full offer/accept cycle, both under mandate; escalation to humans only ≥ `escalation_threshold` | Both agents (mandate), humans only on escalation | Invisible; only escalations + the audit view appear | 4 |

### 3.1 Why Agent-to-Agent is the strategic centre (and what it exclusively demands)

> **Inference, with reasoning:** Agent-to-Agent is the only one of the four cases that is *not at all* solvable otherwise today (US-1/US-4 of the ANP user stories confirm this against x402/AP2/virtual cards). The other three have workarounds (email, DocuSign, Git). A2A also grows automatically with the agentic-computing trend. Hence the prioritization in §7.

A2A demands two things the other topologies don't need, which therefore are architecture drivers:

- **Agent DIDs + mandates as runtime objects.** Every omadia agent that should be proof-capable needs its own DID (not the user's) and a principal-issued mandate VC with caps. The mandate is the "blast-radius bound" against hallucinating agents (ANP §7.5).
- **Escalation as a kernel hook, not a plugin detail.** When an agent action reaches `escalation_threshold`, a human must *reliably* be brought into the loop — the same human-checkpoint guarantee as ADR-0005, only triggered by a mandate constraint instead of by "is a write". Proof wires this into the orchestrator dispatch (R10).

### 3.2 Correction: Agent→Human belongs in Phase 4, not Phase 3 (Codex C3)

> **Fact from this plan:** a *binding* Agent→Human action requires the agent to **sign under a mandate** (§3 table, §5.1). But agent DIDs, mandate issuance, `checkAuthority` and the escalation gate are only introduced in **Phase 4**. So "an agent makes a binding proposal, a human accepts" is **not** buildable in Phase 3 — it would be a forward dependency on Phase 4.

Resolution:

- **Phase 3** stays on **Human↔Human** plus the defused case **"agent drafts, human signs as principal"** — i.e. the agent only *produces the proposal content*, the only binding signature is the human's (no agent mandate, no agent signature). This needs neither mandate nor gate.
- **Binding Agent→Human** (agent signs under mandate, human `approve`/`accept`) is switched on in **Phase 4**, together with A2A — both share the same mandate/gate substrate.

---

## 4. Architecture: how Proof sits in omadia

**Repo distribution (decided).** `byte5ai/omadia-proof` is the right home and stays: Proof is clean plugin territory for almost everything, and omadia distributes plugins as standalone ZIPs anyway (Teams, Telegram live outside the monorepo just the same). The whole bundle — object building, identity, signer, anchor adapter, store, native tools, UI — is built in the Proof repo and installed into omadia as a signed ZIP.

**Kernel changes — one planned, possibly two (Codex C2).** The *known* exception to pure plugin territory is the **escalation gate** (mandate threshold ⇒ human `approve`, §5.1): it must hook into the orchestrator *before* tool execution, and the plugin API exposes no plugin-extensible pre-dispatch hook today. Evidence: `harness-verifier` kept its orchestrator-binding wrapper deliberately kernel-side — "the Orchestrator class is ~1k LOC and not yet plugin-extractable". The gate has the same problem and therefore lands as **one small, targeted PR in `byte5ai/omadia` itself** — and only in Phase 4, when A2A needs it. **Conditional, not guaranteed sole:** whether **custodial-key scaling** (R11 / §7 Stage A) additionally needs a kernel-side custody service is an open Stage-A question. If it turns out the per-plugin vault secrets store does *not* carry per-user/-agent keys at the required scale, that is a **second, larger** kernel change. The earlier phrasing "the only cross-repo change of the whole project" is therefore softened to "the planned escalation-gate PR; a second custody PR is possible and is clarified in Stage A". A plugin that needs one or two targeted kernel hooks is still a plugin.

Proof is **not a monolith plugin**, but a small layer of capability providers plus UI extensions, exactly the pattern omadia already uses for `knowledgeGraph@1`, `verifier@1`, `privacy.redact@1` (ADR-0003). This keeps ANP swappable (DLT neutrality mirrored at the code level) and Proof testable.

> **Naming note (Stage-A decision, R16 — Codex C8):** the capability IDs (`proof.objects@1`, `proof.identity@1`, …) are stable and are used throughout this plan as the primary reference. The **package prefix**, by contrast, is *open*: §1.2 reserves `harness-*` for *kernel/infra in the monorepo* — but Proof is an external plugin bundle. Before the first package, check against the `omadia-plugin-starter` template which prefix applies there for externally distributed plugins (likely `agent-*` or an own scope such as `@omadia/plugin-proof-*`). Not cosmetic: package paths, manifests and release artifacts depend on it. Set it once in Stage A before six packages carry the wrong name.

```
┌──────────────────────────────────────────────────────────────────────┐
│  omadia UI (web-ui / canvas)                                           │
│  "Record · Witness · Agree · Resolve" cards,                           │
│  confirm dialogs, proof inbox, receipt detail view                     │
│  (uiRoutes contributed by the Proof plugin)                            │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  proofRef (opaque) + human-readable summary
                            ▼  — NEVER raw crypto (Privacy Shield v4, R13)
┌──────────────────────────────────────────────────────────────────────┐
│  Orchestrator hooks                                                    │
│  • Native tools: proof.record / proof.attest / proof.agree / proof.    │
│    resolve  (exposed to the agent)                                     │
│  • Escalation gate: mandate threshold ⇒ enforce human approve (R10)    │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  ctx.services.get("proof.*")
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PROOF CAPABILITY LAYER  (new packages; prefix = Stage-A decision)     │
│                                                                        │
│  proof.objects@1   ── ANP object build: envelope, JCS canonicalization,│
│                       schema validation, hash chaining. Pure logic,    │
│                       chain-agnostic. Mirrors ANP §6.                  │
│                                                                        │
│  proof.identity@1  ── custodial DID/key management per user+agent,     │
│                       mandate-VC issuance & checking, SD-JWT           │
│                       disclosure. Keys ONLY in the vault. Mirrors ANP §5│
│                                                                        │
│  proof.signer@1    ── signature suite (PQC-capable: ML-DSA/SLH-DSA),   │
│                       detached signatures for co-signing. ANP §6.3/§6.5│
│                       Separates "signing" from "owning the key".       │
│                                                                        │
│  proof.anchor@1    ── DLT PROFILE ADAPTER. The only package that knows │
│  (anchor-iota /      the chain. Default: IOTA-Rebased profile.         │
│   anchor-mock)       Swappable for an EVM profile with no consumer     │
│                      change (ANP §13 as a code boundary). anchor(hash),│
│                      status(), settlement/escrow hook, DA locator.     │
│                                                                        │
│  proof.store@1     ── off-chain object storage + DA layer (ANP §6.4):  │
│                       mechanism open (KG/Memory/Mirror, R15);          │
│                       + optional content-addressed mirror; void-on-    │
│                      unavailability rule; receipts; verify-link source.│
└──────────────────────────────────────────────────────────────────────┘
```

### 4.1 Why these cut lines

- **`proof.anchor@1` is the DLT seam.** Just as omadia swaps `knowledge-graph-inmemory` vs. `-neon`, Proof swaps `anchor-iota` vs. a future `anchor-evm` — without core, identity or UI noticing. That is ANP's "DLT-neutral" (ANP §3.1 Goal 3) as code architecture, not just prose. **For tests:** an `anchor-mock` (in-memory ledger) makes the entire Proof logic testable without a real chain — mandatory, because otherwise there's no CI.
- **`proof.identity@1` is the crypto capsule.** Here — and only here — the "no wallet" magic becomes real: custodial keys in the vault, signing as a service, the mandate as an issuable/checkable VC. If an enterprise later wants *its own* keys (HSM, eIDAS-QES), only this package is replaced.
- **`proof.objects@1` is pure, deterministic logic.** No I/O, no chain. This isolates the trickiest correctness requirement (JCS canonicalization, deterministic proof assembly ANP §6.3 — "every party assembles a byte-identical anchored form") for unit testing against the spec examples.

### 4.2 Data model (logical — the physical store is a Stage-A decision, R15)

> **Fact that corrects the original assumption:** an external plugin gets **no raw Postgres pool**. `MigrationContext` (the `onMigrate` hook) only grants write access to the per-plugin **secrets** and **memory** store, not the ability to create its own tables. The entities below are therefore a **logical** model — how they are physically persisted is a **blocking Stage-A decision** (R15), not a fact, and **`proof.store@1` (needed from Phase 1) depends on it** (Codex C1). My take: the knowledge graph (`knowledgeGraph@1`) is the most likely carrier because it brings ACL and retrieval; to be verified (A5) before `proof.store@1` is fixed.

The spec's strict on-/off-chain separation stays mandatory: PII lives exclusively off-chain in `body` (encrypted at rest via the existing mechanism); on the chain only hash + status.

| Entity (logical) | Purpose | Key fields |
|---|---|---|
| `proof_identities` | DID + encrypted key reference per user/agent | `subject_id` (FK user/agent), `did`, `key_vault_ref`, `kind` (`human`/`agent`), `created_at` |
| `proof_mandates` | Issued mandate VCs (agent authority) | `mandate_id`, `principal_did`, `agent_did`, `scope[]`, `constraints`, `not_before`, `expires`, `status_list_ref`, `revoked_at?` |
| `proof_objects` | Off-chain ANP objects (the full artifact) | `object_id`, `thread_ref`, `type`, `sequence`, `previous_hash`, `body` (**encrypted at rest**), `proof[]`, `anchored_hash?`, `da_locator?` |
| `proof_threads` | Derived thread state (off-chain replicated, **cache**) | `thread_ref`, `topology`, `derived_state`, `head_hash`, `pillar`, `owner_subject_id`, `counterparties[]` |
| `proof_anchors` | Mirror of the on-chain anchors (fast query without a chain round-trip) | `object_hash`, `thread_ref`, `status`, `anchored_by_did`, `ledger_timestamp`, `confirmation_state` |
| `proof_status_snapshots` | Retained status-list credentials for point-in-time proof (ANP §5.3) | `snapshot_hash`, `fetched_at`, `credential` |

`proof_threads.derived_state` is **never** treated as the source of truth — it is a cache re-derived on demand from the anchors + objects (SPEC m5: "fine state off-chain by replaying the Thread's anchored Object types"). That is the insurance against exactly the failure mode ANP §1.2 names: a manipulated local DB must not be able to falsify the state.

---

## 5. API surface

### 5.1 Native tools for the agent (exposed to the orchestrator)

Four tools, mirrored on the four user words. Each takes structured input and returns an **opaque `proofRef`** + a human-readable summary — never raw crypto:

| Tool | Pillar | Input (schema-validated) | Behavior |
|---|---|---|---|
| `proof.record` | Memorandum | `{ statement, parties[]? }` | 0/omitted `parties` ⇒ Human-to-Nobody self-memorandum; N ⇒ co-signing flow. Terminal at `ACCEPTED`. (`parties` omitted ≡ empty.) |
| `proof.attest` | Notarization | `{ subject_kind, statement, witnessing, evidence[]? }` | Standalone attestation. Enforces an honest `observed`/`relayed` declaration (ANP §8.1). |
| `proof.agree` | Contract | `{ terms, counterparties[], escrow? }` | offer→accept cycle. `escrow.required` ⇒ full settlement path; otherwise the cheaper record path. |
| `proof.resolve` | Dispute | `{ thread_ref, action: assert\|dispute\|evidence, payload }` | Optimistic dispute. Automatically selects the small-claims profile below the value threshold (ANP §9.4). |

> **Privacy-Shield consequence (R13):** the inputs above contain linkable identifiers (`thread_ref`, `counterparties[]`) and potentially raw data (`evidence[]`). These do **not** cross the LLM boundary raw, but as server-resolved opaque handles. The tool contract (LLM-visible vs. server-interned) is specified per tool in **Stage A**, *before* the tools are built.

**Mandate gate (orchestrator hook, not in the tool):** before execution the dispatch checks whether the acting agent has a valid mandate *and* the action is within `constraints`. ≥ `escalation_threshold` ⇒ tool execution pauses, a human confirm card is generated (`approve`), and only then does it continue. Exactly the ADR-0005 guarantee, triggered by mandate instead of by write type. (Kernel PR, Phase 4, R10.)

### 5.2 Capability interfaces (plugin-to-plugin, internal)

```typescript
// proof.objects@1  — pure logic, no I/O
interface ProofObjectsService {
  buildEnvelope(input: ObjectInput): UnsignedObject;       // ANP §6.1
  canonicalizeForSigning(obj: UnsignedObject): Uint8Array; // JCS, proof removed ANP §6.3
  canonicalizeAnchored(obj: SignedObject): Uint8Array;     // incl. proof[]
  assembleProofs(obj, sigs): SignedObject;                 // deterministic order ANP §6.3
  validate(obj): SchemaResult;                             // against the registered JSON schema
}

// proof.identity@1  — the crypto capsule
interface ProofIdentityService {
  ensureIdentity(subjectId: string, kind: 'human'|'agent'): Promise<Did>; // auto-provision
  issueMandate(principal: Did, agent: Did, c: MandateConstraints): Promise<MandateVc>;
  checkAuthority(agentDid, action, value): Promise<AuthorityVerdict>;      // ANP §5.3 rules
  proveDisclosure(mandate, predicate): Promise<SdJwtPresentation>;         // ANP §6.6, no cap leak
}

// proof.signer@1
interface ProofSignerService {
  sign(subjectId: string, input: Uint8Array): Promise<DetachedSignature>;  // key stays in the vault
  suite(): SuiteId;                                                        // anp-suite-2 (PQC)
}

// proof.anchor@1  — the DLT seam (default: IOTA Rebased)
interface ProofAnchorService {
  anchor(hash: TaggedHash, meta: AnchorMeta): Promise<AnchorReceipt>;      // ANP §6.2
  setStatus(hash, status): Promise<void>;
  openEscrow?(terms): Promise<EscrowId>;   // optional per profile capability ANP §10
  enforce?(directive): Promise<void>;      // ANP §6.2.1
  confirmationState(hash): Promise<'pending'|'final'>;
}

// proof.store@1
interface ProofStoreService {
  put(obj: SignedObject): Promise<void>;
  get(hash: TaggedHash): Promise<SignedObject | undefined>; // void-on-unavailability ANP §6.4
  deliver(obj, counterparties): Promise<Receipt[]>;         // DA obligation + receipt
}
```

### 5.3 UI routes (contributed by the Proof plugin via `uiRoutes`)

- `/proof` — **proof inbox**: all of the user's actions, filtered by status (recorded / waiting on counterparty / waiting on me / in resolution / closed). No crypto visible.
- `/proof/:ref` — **receipt detail view**: human-readable summary at the top; "View technical receipt" as an expandable section (here — and only here — hash/DID/anchor become visible to auditors, read-only).
- **Confirm dialog** (canvas component, no own path): preview of the exact action, one confirm button. The human signature moment.
- **Verify-link** (Phase 2/3, read-only, permissionless): an externally shareable, omadia-independent verification page for a receipt. What it exposes, how it is authorized/redacted, and how it behaves for an unavailable object is part of the DA/verify-link specification (R9).

---

## 6. The tricky parts — named honestly

No plan without the risks that can topple it. R1–R8 as originally; R9–R18 added from the Codex review; R19 from the A1–A8 verification (`research.md`, §11). Sorted by severity within the groups.

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| R1 | **Custodial keys = single point of compromise.** If the vault falls, an attacker can sign in the name of *all* users/agents. That is the price of "no wallet". | **High** | The vault is already AES-256-GCM + mandatory `VAULT_KEY`. Additionally: per-subject key derivation, mandate caps bound the blast radius per agent (ANP §7.5), escalation for high-value actions. Upgrade path to HSM/own keys by replacing only `proof.identity@1`. **Document clearly** (ADR-0001) that this is a deliberate custody decision. Operational key lifecycle separately: R11. |
| R2 | **Legal force is overestimated.** Users might believe a Proof replaces a notary. It doesn't (ANP §16.5, ANP §3.2). | **High** | UX law 6: explicit, honest framing in the UI for every high-value/form-requiring action. Proof positions itself as a *technical* proof, not a legal form. In the UI never say "legally binding"; "binding" = technically binding per ANP. |
| R3 | **Privacy leak via the on-chain link graph.** `anchored_by` DID + `thread_ref` create a permanent, un-erasable "who-with-whom-when" graph on a public chain (ANP §6.2 m6). | **Medium-high** | Pairwise DIDs per relationship/thread where a natural person is identifiable (ANP §12 makes this mandatory — A2). `proof.identity@1` generates these automatically — the user notices nothing. Aggravated by the GDPR tension: R13b. |
| R4 | **Counterparty adoption.** Every action with a counterparty needs *both* sides on ANP/omadia (ANP "shared adoption costs"). | **Medium-high** | Human-to-Nobody (self-notarization) has **no** counterparty and is therefore the adoption opener: immediate solo value with no network effect. Hence Phase 1 (§7). For counterparties without omadia: the "verify-link" (read-only verification in the browser, permissionless ANP §2.2) — verify, not co-sign (see Phase 3 scope boundary). |
| R5 | **Escrow custody inherits native chain crypto (not PQC).** A quantum attacker on the chain account could move escrow funds even though the binding signatures stay PQC-safe (ANP §6.5 "honest limitation"). | **Medium** (escrow-bearing threads only) | Bound escrow amounts + dispute windows; prefer a chain with a credible PQC migration path; disclose as a long-term risk. Most Proof actions (Record/Attest) carry *no* escrow and are unaffected. |
| R6 | **JCS canonicalization bugs ⇒ signatures don't verify across parties.** If two implementations don't canonicalize byte-identically, multi-party co-signing is dead. | **Medium** | `proof.objects@1` as pure logic unit-tested against the SPEC examples; conformance test vectors from Appendix A (A1). CI gate. |
| R7 | **ANP is v0.3-draft with open questions (ANP §16).** The spec may change before v1.0. | **Low-medium** | `proof.objects@1` pins `anp_version`. Proof modules version along. Operationalize a migration/versioning policy: R14b. Open-question areas (trust-list governance, arbiter-pool bootstrap) only go productive in later phases. |
| R8 | **Name collision `verifier@1`.** Already exists as the LLM claim verifier. | **Low** | Proof uses the `proof.*` namespace throughout. No capability is named `verify`/`verifier`. (Package prefix separately: R16.) |
| **R9** | **External verifiability without a data-availability surface** (Codex C4). A chain hash alone is not enough to independently check a receipt if the full off-chain object can't be retrieved — the "survives omadia" / verify-link promises (Phase 2/3) need a DA layer. | **Medium-high** | DA locator + a public retrieval/verify-link surface as an **explicit Phase-2 precondition** before "survives omadia" is claimed. Specify: what the link shows, authorization, redaction, locator behavior, semantics for an unavailable object. |
| **R10** | **Kernel escalation gate** (Codex C2/C3). A pre-dispatch hook in the ~1k-LOC orchestrator; the only *planned* kernel change. | **Medium** | Design as a generic pre-dispatch policy hook in the kernel (not proof-wired), Phase 4. Contract + tests before the PR; confirm the `harness-verifier` precedent (A6). |
| **R11** | **Custodial-key scaling & lifecycle** (Codex C2 + IMPORTANT). Does the per-plugin vault carry per-user/-agent keys at the required scale? Backup/restore, compromised-key revocation, rotation, recovery, operator runbooks are missing. | **High** | Stage-A verification against the real vault state (A7); possibly a kernel-side custody service (second kernel change, §4). Lifecycle runbook as its own cross-cutting work item. |
| **R12** | **DID method + resolver unresolved** (Codex C6). Determines whether identity needs a ledger lookup (`did:iota`/`did:ethr`) or is locally/custodially resolvable (`did:key`/`did:web`); must match the anchor profile; **hard to reverse** because DIDs sit in every anchored object. | **High** | Blocking Stage-A decision *before* `proof.identity@1`: method, resolver, rotation, pairwise mapping, migration. My take: `did:key` (custodial, no lookup) or `did:web` under the instance domain; `did:iota` only if the reference profile suggests it. |
| **R13** | **Privacy-Shield dataflow per tool unspecified** (Codex C7). `thread_ref`/`counterparties[]`/`evidence[]` in tool inputs could leak linkable IDs/raw data across the LLM boundary. | **Medium-high** | Mandatory Stage-A spec per tool: LLM-visible vs. server-interned vs. summarized; opaque handles instead of raw references (UX law 5). |
| **R13b** | **GDPR / right to erasure vs. immutable anchors** (Codex IMPORTANT). Hash + link graph stay on-chain even when the off-chain object is deleted. | **Medium** | Decision: what is deletable (off-chain `body`/keys), what stays (hash/anchor); how the residual hash/link is explained; does deletion break verification? As a compliance work item, before productive PII processing. |
| **R14** | **Observability & async-failure handling missing** (Codex IMPORTANT). Anchor retry, dead-letter, finality timeout, duplicate-anchor idempotency, metrics, degraded states. | **Medium** | Async anchor worker with retry/idempotency/timeout + a user-visible degraded state (UX law 4). Observability baseline as cross-cutting. |
| **R14b** | **`anp_version` migration not operationalized** (Codex IMPORTANT). Version pinned, but no schema version registry/migration policy/backward verify. | **Low-medium** | Schema version registry, migration policy, backward verification, fixture versioning, compatibility tests — couple with the schema matrix (§1.4.1). |
| **R15** | **Object-store mechanism open, Phase 1 depends on it** (Codex C1). KG vs. Memory vs. Mirror; ACL/retrieval/encryption/export/retention undefined. | **High** | Blocking Stage-A decision *before* `proof.store@1`/Phase 1: logical→physical mapping, ACL model, retrieval API, encryption, export, retention + acceptance tests (A4/A5). |
| **R16** | **Package naming not cosmetic** (Codex C8). `harness-proof-*` contradicts §1.2; wrong package paths/manifests/artifacts loom. | **Low-medium** | Stage-A naming decision + repo-layout ADR *before* the first package is created. |
| **R17** | **Multi-tenant isolation not covered** (Codex IMPORTANT). Tenant scoping for identities, mandates, objects, anchors, verify-link access, auditor rights is missing. | **Medium** | Carry as a cross-cutting requirement on `proof.identity@1`/`proof.store@1`; acceptance criteria per affected package. |
| **R18** | **Clock/timestamp trust** (Codex IMPORTANT). Local vs. ledger timestamp, skew, ordering disputes. | **Low-medium** | Ledger-timestamp precedence over local; skew handling + test fixtures; anchored in Phase 2 (anchoring). |
| **R19** | **Maturity of the IOTA first-party dependencies** (A1–A8 verification 2026-06-15, SPEC §13.2). IOTA Identity (Beta, no stable release), Gas Station (pre-1.0), Notarization/Hierarchies (Alpha) carry Phase 1 (PQC VCs/custodial signing) and Phase 2 (anchor/sponsoring). | **Medium** (from Phase 2) | `anchor-mock` keeps Phases 0–1 chain-free (already planned); IOTA dependencies only sharp in Phase 2; pin versions, plan for pre-GA API breaks; evaluate binding to IOTA Notarization (anchor) + Hierarchies (trust list) in the PoC (ANP §17 Phase 2) instead of rolling our own anchor objects. |

---

## 7. Phase plan — along the use cases, not along the tech

The sequence follows two principles: **(a) earliest solo value with no network effect** (solves R4), **(b) A2A — the strategic centre — as a clear goal, but on a load-bearing foundation.** Every phase is demonstrably useful, not just "infrastructure". **New (Codex):** an upfront **Stage A (architecture freeze)** resolves the blocking decisions *before* code exists — it replaces the former "open decision points" (old §10) and turns them into hard gates.

### Stage A — architecture freeze (no code, blocking)
**Goal:** make the hard-to-reverse decisions and verify the load-bearing assumptions against the real repos before any package is created. Each item is a precondition for at least one later phase.
**Contents (gates):**
1. **Package naming & repo layout** (R16) → ADR.
2. **Object-store mechanism** decide: KG vs. Memory vs. Mirror, incl. ACL/retrieval/encryption/export/retention spec (R15, A4/A5) — precondition for `proof.store@1`/Phase 1.
3. **DID method + resolver** decide: method, rotation, pairwise mapping, migration (R12) — precondition for `proof.identity@1`.
4. **Custodial-key scaling** check: per-plugin vault vs. kernel custody service (R11, A7) — determines whether a second kernel PR is needed.
5. **Schema-backflow matrix** finalize (§1.4.1) incl. ANP PR process + fixture/versioning policy (R14b).
6. **Privacy-Shield dataflow per tool** specify (R13).
7. **Verify assumptions A1–A8 (§11)** against `byte5ai/anp` SPEC.md and `byte5ai/omadia` code. ✅ done 2026-06-15 → `research.md`.
8. **Write the ADRs:** `0001-anp-proof-custodial-identity.md` (custody decision, R1, incl. HSM/QES upgrade path), `0002-schema-source-of-truth.md` (§1.4), `0003-repo-layout-and-naming.md` (R16), `0004-object-store.md` (R15), `0005-did-method.md` (R12).
**Done:** all five ADRs merged; the schema matrix stands; A1–A8 confirmed or accepted as a risk. Only then does Phase 0 begin.

### Phase 0 — foundation (no user feature)
**Packages:** `proof.objects@1`, `proof.anchor-mock@1`, a `proof.store@1` skeleton, plugin scaffold + CI pipeline.
**Goal:** ANP object build + canonicalization + schema validation, green against spec test vectors; an in-memory mock ledger; a signable/packageable skeleton built against the plugin-starter template with green CI. **No UI.** This is the correctness insurance (R6) and makes everything downstream CI-testable.
**CI breadth (Codex IMPORTANT — not just canonicalization):** unit (JCS/hash chain against Appendix-A vectors), schema validation, plugin packaging/signing, capability registration, later Privacy-Shield boundary tests.
**Done criterion (two hard gates):**
1. A memorandum can be built, co-signed (simulated), canonicalized and "anchored" (mock) — byte-identically reproducible.
2. **Every object schema defined here has an open PR against `byte5ai/anp` (Appendix A)** per the schema matrix (§1.4.1) — for Phase 0: `memorandum`. Without that PR, Phase 0 counts as not done (§1.4).

### Phase 1 — "Record", solo (Human-to-Nobody) · *the opener*
**Precondition:** Stage-A gates 2 (store), 3 (DID), 4 (custody) decided.
**Use case:** a manager records "I made decision X today, reason Y" — a milestone for the record. No counterpart, immediate value.
**Packages:** `proof.identity@1` (custodial DID/key, **human only**), `proof.signer@1`, `proof.store@1` (per the Stage-A decision), first UI: `proof.record` solo + proof inbox + receipt detail view. **Void-on-unavailability** behavior specified + implemented.
**Anchor:** still mock *or* the first real IOTA profile, depending on maturity (decided in this phase).
**Why first:** no adoption network effect (R4), validates the *entire* crypto capsule (laws 2+5) on a low-risk case, ships a presentable feature.
**Done:** a non-IT user records something, sees "✓ Recorded", finds it in the inbox, sees zero crypto — and an auditor can expand the technical receipt.

### Phase 2 — real anchoring + "Witness"
**Use case:** US-6-like — an action/audit result is witnessed and survives the system that produced it.
**Packages:** `proof.anchor-iota@1` productive (IOTA-Rebased profile ANP §13.2) incl. an **async anchor worker (retry/idempotency/timeout, R14)**, `proof.attest` (standalone attestation, `observed`/`relayed`), an async confirmation state in the UI (law 4), status snapshots (`proof_status_snapshots`), the **DA locator + verify-link foundation (R9)**, timestamp handling (R18).
**Precondition (Codex C4):** the "survives omadia" claim only holds once the DA/verify-link surface stands — it is part of this phase, not an assumption.
**Done:** "Witness" works end-to-end against a real chain; the confirmation state trickles in quietly; verification does not depend on omadia's continued existence (proven via DA + verify-link); the ANP PR for `attest` is open.

### Phase 3 — two parties (Human-to-Human; agent drafts, human signs)
**Use case:** a meeting record with binding force (H2H); an agent *produces the content* of a proposal, the human signs it as principal.
**Packages:** co-signing flow productive (`proof.record` with N parties, detached signatures ANP §6.3), `proof.agree` (offer/accept, **no escrow yet**), a confirm dialog for the counterparty, a "verify-link" for counterparties without omadia (read-only, permissionless, on the R9 foundation).
**Scope boundary (important, otherwise wrong expectations):** a *binding* two-party agreement requires an anchored signature (`accept`) from **every** party (ANP §7.4, m10). So both sides must be ANP-capable. The "verify-link" lets a non-omadia counterparty **verify** an action, not **co-sign** it. Binding H2H actions with a party *without* an ANP stack are **not** possible in v1 (only record one-sided + send a link to verify). A browser-based lightweight signature for external counterparties (custodial guest DID) is a conceivable later feature, not v1 scope.
**Delimitation (Codex C3):** *binding* Agent→Human (agent signs under mandate) is **not** in this phase — it needs mandate + gate and lives in Phase 4 (§3.2). In Phase 3 an agent may only *draft*; the only binding signature is the human's.
**Done:** two *omadia-/ANP-capable* parties record an arrangement byte-identically co-signed; an agent can present proposal content to a human for signature as principal; ANP PRs for `offer`/`counter_offer`/`accept` open.

### Phase 4 — Agent-to-Agent (+ binding Agent-to-Human) · *the strategic centre*
**Use case:** the favorite — AI-agent tasking with an audit trail; two agents close end-to-end; plus binding Agent→Human (pulled here from Phase 3, §3.2).
**Packages (Proof repo):** `proof.identity@1` extended with **agent DIDs + mandate issuance** (`issueMandate`, `checkAuthority`), **status-list hosting + mandate revocation lifecycle** (Codex IMPORTANT), SD-JWT disclosure for mandates (ANP §6.6, against cap leak ANP §5.3), **aggregate-cap enforcement**, a mandate management UI (the principal sets caps — "what may this agent sign without asking?").
**Kernel PR (`byte5ai/omadia`, R10):** the **escalation gate** as a small kernel-side, ideally *generic* pre-dispatch policy hook that checks the mandate at dispatch time and pauses tool execution on threshold exceedance until a human `approve` exists. Must go in the kernel because it hooks in *before* tool execution and the plugin API exposes no pre-dispatch hook (§4, verifier precedent A6). Contract + tests before the PR.
**Done:** two omadia agents of different parties close autonomously within their mandates; a binding Agent→Human runs via mandate + `approve`; an over-threshold action escalates reliably to a human; mandates are revocable (status list); the whole action is traceable as an audit trail; ANP PRs for `approve` + mandate VC open.

### Phase 5 — "Resolve" + settlement (escrow & dispute)
**Use case:** US-1/US-5 — value actions with escrow and the dispute no platform owns.
**Packages:** escrow hooks in `proof.anchor-iota@1` (`openEscrow`/`enforce` ANP §10/ANP §6.2.1) incl. bond funding + challenge windows, `proof.resolve` (optimistic dispute + automatic small-claims profile selection below the value threshold ANP §9.4), an arbiter/trust model + small-claims formula source, enforcement authorization, a dispute UI.
**Why last:** highest complexity, highest risk (R5), and economically sensible only once the cheap high-frequency cases (Phases 1–4) have traction. Until then, for micro-values: anchored evidence + reputation-based deterrence, no forum.
**Done:** an escrow-backed action can optimistically finalize, mutually settle immediately, or resolve via a dispute into an enforced outcome — without the losing side's cooperation; ANP PRs for `assert`/`dispute`/`evidence`/`rule`/`appeal`/`enforce`/`settle`/`receipt` open.

### Overview

| Phase | Topology/pillar | Delivers to the user | Core risk addressed |
|---|---|---|---|
| A | — (architecture freeze) | (decisions/ADRs) | R11, R12, R13, R15, R16 |
| 0 | — | (foundation) | R6 (canonicalization), CI breadth |
| 1 | H→Nobody / Record | "Record" solo | R4 (adoption), R1 validation |
| 2 | — / Notarize | "Witness" + real chain | R3 (link graph), R9 (DA/verify), R14, R18, R19 (IOTA maturity) |
| 3 | H→H (+ agent drafts) | "Agree" between two | R4 (verify-link) |
| 4 | **A→A (+ binding A→H)** | autonomous agents + escalation | R10 (gate), R1 (mandate caps), R2 |
| 5 | / Resolve | "Resolve" + escrow | R5 (escrow crypto) |

---

## 8. What deliberately does *not* belong in v1

- **No own marketplace/discovery function.** ANP is explicitly not a marketplace (ANP §3.2). Agents find each other via A2A/registries, not via Proof.
- **No token.** Settlement uses the native chain asset/stablecoin (ANP §3.2). Proof never introduces its own currency.
- **No multi-tier appeals, no VRF witness pools, no reputation registry** — all defined in ANP, but late/optional profiles. Only once the base holds.
- **No legal form (eIDAS-QES, notariat).** Deliberately outside. Proof is the technical proof; the legal form is a separate, later integration point (a swappable `proof.identity@1` with a QES backend would be the route).
- **No browser-based guest signature** for non-omadia counterparties (verify-link/verify only). Conceivable later, not v1.

---

## 9. Immediate next steps (concrete)

1. **Structure the Proof repo:** in `byte5ai/omadia-proof` store this document as `plan.md`, plus `data-model.md` (expand section 4.2 once the store mechanism is decided — Stage A), `research.md` (ANP spec excerpts + A1–A8 verification), `tasks.md`.
2. **Start Stage A** (§7): write the five ADRs, finalize the schema matrix, verify A1–A8. **No package code before Stage A is Done.**
3. **Clarify the schema source, then build:** since omadia is the first ANP implementation, `proof.objects@1` defines the object schemas — **with a simultaneous PR against `byte5ai/anp` Appendix A** (§1.4) per the matrix. Spec test vectors from Appendix A as fixtures where they exist; where not, they are produced here *and* played back into the RFC.

---

## 10. Decision points → moved into Stage A

> The originally open schema question is answered (§1.4: omadia defines the schemas and plays them back mandatorily). The topics carried in the first version as "three open points" (object-store mechanism, custodial-key scaling, DID method) are **no longer** just prose, but **blocking Stage-A gates** (R15, R11, R12) with mandatory ADRs. The Codex review additionally identified naming (R16) and the Privacy-Shield dataflow (R13) as decision-blocking. They all live in Stage A now (§7).

*(Note: the earlier inconsistency "§9.4 says two, §10 lists three" is resolved by this move.)*

---

## 11. Review resolution (Codex, 2026-06-15)

This document was reviewed by Codex (effort high, read-only), since it had not been reviewed before. Incorporated findings:

| Codex finding | Incorporated as |
|---|---|
| C1 store blocker, Phase 1 depends on it | R15 + Stage-A gate 2 + §4.2 note |
| C2 "single kernel PR" conditional | §4 reframed + R10/R11 + Stage-A gate 4 |
| C3 Agent→Human forward dependency | §3.2 + Phase 3/4 reordered + §3 table |
| C4 external verifiability without a DA surface | R9 + Phase-2 precondition + verify-link in §5.3 |
| C5 schema gate incomplete/duplicated | §1.4.1 schema-backflow matrix (replaces prose) |
| C6 DID method unresolved | R12 + Stage-A gate 3 + ADR-0005 |
| C7 Privacy-Shield vs. tool surface | R13 + §2.1.5/§5.1 notes + Stage-A gate 6 |
| C8 package naming not cosmetic | R16 + naming as Stage-A gate 1 + capability IDs as the primary reference |
| IMPORTANT: CI too narrow | Phase-0 CI breadth |
| IMPORTANT: key lifecycle/DR | R11 + cross-cutting |
| IMPORTANT: status-list hosting | Phase-4 package |
| IMPORTANT: `anp_version` migration | R14b |
| IMPORTANT: observability/async failure | R14 + Phase-2 worker |
| IMPORTANT: void-on-unavailability | Phase-1 Done |
| IMPORTANT: escrow/dispute too thin | Phase-5 package expanded |
| IMPORTANT: multi-tenant isolation | R17 |
| IMPORTANT: GDPR/erasure | R13b |
| IMPORTANT: clock/timestamp | R18 + Phase 2 |
| MINOR: "two vs. three" | §10 cleaned up |
| MINOR: Phase-2/5 topology cells empty | §7 overview filled |
| MINOR: "binding" wording | UX law 6 / R2 sharpened |
| MINOR: `parties[]` omitted vs. empty | §5.1 table clarified |
| MINOR: filename `plan.md` | renamed |

**Assumptions-to-verify (A1–A8)** — load-bearing claims about the ANP spec/omadia code. **Status: verified 2026-06-15 against `byte5ai/anp@main` + `byte5ai/omadia@main` → `research.md`.** Result: 6/8 confirmed; **A5** (KG without encryption-at-rest) and **A7** (vault designed for O(100) agents) with a caveat — both strengthen the Stage-A gates (R15/R11). Plus R19 (IOTA tooling maturity) and the naming finding for ADR-0003.

| # | Assumption | Depends on it |
|---|---|---|
| A1 | ANP defines the listed object types, statuses, standalone attestation, memorandum terminal behavior, deterministic proof order as described | Phase-0 schema/hash tests |
| A2 | ANP requires/recommends pairwise DIDs strongly enough to justify the R3 mitigation | identity architecture, privacy |
| A3 | ANP supports the cited PQC suites, SD-JWT mandate disclosure, IOTA-Rebased profile | Phase 1–4 sign/verify |
| A4 | omadia plugins have *no* raw Postgres access, only per-plugin Secrets/Memory via `onMigrate` | the entire storage strategy |
| A5 | `knowledgeGraph@1` fits object persistence, ACLs, retrieval, encrypted-at-rest | Phase-1 storage option |
| A6 | omadia has no plugin-extensible pre-dispatch hook; the `harness-verifier` precedent holds | kernel-PR validity (R10) |
| A7 | vault scaling/KV limits/backup/rotation suffice for per-user/-agent custody | Phase-1 identity, R11 |
| A8 | plugin signing, ZIP packaging, `provides/requires`, `uiRoutes`, native tools, Privacy-Shield internals behave as described | Phase-0 scaffold, tool wiring |

---

*Basis: ANP SPEC.md v0.3-draft (byte5ai/anp) + omadia public preview (byte5ai/omadia), read 2026-06-15. Codex review incorporated 2026-06-15 (§11). Inferences are marked as such in the text; everything else is evidenced directly from spec/code — except where A1–A8 flag it as to-be-verified.*
