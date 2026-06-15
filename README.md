<div align="center">

# omadia-proof

### Make any binding action provable across trust boundaries — without anyone touching a key, a wallet, or a chain.

omadia Proof embeds **[ANP](https://github.com/byte5ai/anp)** — the Agent
Notarization &amp; Negotiation Protocol — into the
**[omadia](https://github.com/byte5ai/omadia)** agentic OS. Every binding
action — a decision, an agreement, a witnessed fact — becomes a signed,
schema-valid, hash-chained, on-chain-*anchored* receipt. The content stays
off-chain; only a hash + status reach the ledger. DIDs, keys, signature suites,
anchoring, escrow — all of it is capsuled, so a manager or employee without an
IT background experiences just one thing: *"this is now recorded, verifiable,
and tamper-proof."*

[![License: MIT](https://img.shields.io/badge/License-MIT-black.svg)](LICENSE)
[![Status: design](https://img.shields.io/badge/status-design-orange.svg)](#status--roadmap)
[![Implements: ANP v0.3-draft](https://img.shields.io/badge/implements-ANP%20v0.3--draft-6f42c1.svg)](https://github.com/byte5ai/anp)
[![Plugin for: omadia](https://img.shields.io/badge/plugin%20for-omadia-2496ED.svg)](https://github.com/byte5ai/omadia)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

[**Website**](https://omadia.ai) · [**Plan**](plan.md) · [**Fact base**](research.md) · [**Issues**](https://github.com/byte5ai/omadia-proof/issues) · [**ANP spec**](https://github.com/byte5ai/anp/blob/main/SPEC.md) · [**Contributing**](CONTRIBUTING.md)

</div>

---

> **Status — design / pre-implementation.** This repository currently holds the
> reviewed implementation [plan](plan.md) and the verified [fact base](research.md)
> (the plan's load-bearing assumptions checked against the real ANP spec and
> omadia code). Package code lands per the phase plan below — **nothing here is
> runnable yet.**

## The whole user surface is four words

The user never thinks in protocols or pillars. They think in one question —
*"do I want this to be undeniable later?"* — and if yes, they tap a button.
Proof translates that into the right ANP pillar:

| What the user wants | They see | What Proof does underneath |
|---|---|---|
| Record that we said / decided this | **Record** | Memorandum (self- or co-signed) |
| Confirm that something is / was so | **Witness** | Attestation (`observed` / `relayed`) |
| Make an arrangement binding | **Agree** | Contract: offer / accept (+ optional escrow) |
| Have a disagreement settled | **Resolve** | Optimistic dispute: assert / dispute / rule / enforce |

No crypto vocabulary ever reaches the UI — no *hash*, *DID*, *wallet*, *anchor*.
The technical receipt is one click away under *View details*, never pushed.

## Why it exists

omadia already gives every run an audit *receipt* — but that receipt lives in
your own database, which proves to *you* what *your* system did. A Proof receipt
is different: it proves to a **mistrustful counterparty or regulator** what was
agreed or witnessed **across** a trust boundary — externally anchored,
unalterable by either side. Proof is the bridge from the internal log to the
external, shared fact.

ANP normally assumes every actor runs keys and wallets — the exact friction that
sinks DLT adoption with humans. omadia already has the agent that absorbs it:
Proof turns *"the human must operate a wallet"* into *"the human taps Confirm,
the custodial layer signs."* It secures all four topologies —
**Human→Nobody, Human→Human, Agent→Human, Agent→Agent** — with agent-to-agent
as the strategic centre.

## Architecture

Proof is **not a monolith**. It is a small layer of capability providers plus UI
extensions, shipped as a signed plugin bundle and installed into omadia, with a
single kernel-side hook for the escalation gate.

```
  omadia UI  ──  Record · Witness · Agree · Resolve cards, confirm dialogs, proof inbox
       │   proofRef (opaque) + human-readable summary — never raw crypto (Privacy Shield)
       ▼
  Orchestrator hooks  ──  native tools: proof.record / attest / agree / resolve
       │                  + escalation gate: mandate threshold ⇒ human approve   ← one kernel PR
       ▼
  Proof capability layer  (signed plugin bundle)
    proof.objects    ANP object build · JCS canonicalization · schema · hash chain   (pure logic)
    proof.identity   custodial DID/key per user + agent · mandates · SD-JWT disclosure (the crypto capsule)
    proof.signer     PQC-capable suite (ML-DSA / SLH-DSA), detached signatures
    proof.anchor     the DLT seam — IOTA Rebased profile (swappable); a mock for CI
    proof.store      off-chain object store + data availability; void-on-unavailability
```

The full design — risks, cut lines, data model, API surface — is in
[`plan.md`](plan.md).

## Relationship to the family

- **[omadia](https://github.com/byte5ai/omadia)** — Proof is an omadia plugin
  bundle. It lives 100% in this repo except one small kernel-side hook (the
  pre-dispatch escalation gate), needed only once agent-to-agent lands.
- **[ANP](https://github.com/byte5ai/anp)** — omadia is the **first** ANP
  implementation, so Proof effectively formalizes the per-object JSON schemas the
  spec only references today. **Every schema Proof defines or changes MUST ship
  as a PR back to `byte5ai/anp` in the same unit of work** — the RFC is
  canonical, not the code ([plan §1.4](plan.md)).

## Status &amp; roadmap

Work is tracked as [issues](https://github.com/byte5ai/omadia-proof/issues)
across seven milestones:

| Milestone | Delivers |
|---|---|
| **Stage A — Architecture freeze** | blocking ADRs (naming, store, DID method, custody, schema matrix, privacy dataflow) before any code |
| **Phase 0 — Foundation** | `proof.objects`, mock anchor, plugin scaffold + CI; byte-identical canonicalization |
| **Phase 1 — Record, solo** | Human→Nobody self-record, custodial identity, proof inbox — the adoption opener |
| **Phase 2 — Real anchoring + Witness** | IOTA-Rebased anchoring, attestation, data availability / verify-link |
| **Phase 3 — Two parties** | Human↔Human co-signing, `agree` (offer / accept) |
| **Phase 4 — Agent-to-Agent** | agent DIDs, mandates, the kernel escalation gate — the strategic centre |
| **Phase 5 — Resolve + escrow** | optimistic dispute, small-claims profile, settlement |

## Honest limitations

Stated up front, per the plan's own framing:

- **Not a legal form.** Proof is a *technical*, tamper-proof receipt. Notarial /
  eIDAS-qualified form is a separate, later integration — Proof never claims
  legal force.
- **Custodial keys.** The "no wallet" magic means keys are held custodially in
  omadia's vault. A deliberate custody trade-off (mandate caps bound the blast
  radius); the upgrade path to HSM / own keys is a single-package swap.
- **Escrow inherits the chain's native crypto.** Binding signatures are
  post-quantum-capable; escrow custody is only as quantum-safe as the chain.

## Documents

| File | What |
|---|---|
| [`plan.md`](plan.md) | The implementation plan (Codex-reviewed) — architecture, API surface, risks, phases |
| [`research.md`](research.md) | Verified fact base — the plan's assumptions checked against the real ANP spec + omadia code |
| [`AGENTS.md`](AGENTS.md) · [`CONTRIBUTING.md`](CONTRIBUTING.md) | Engineering standards, worktree + PR workflow, schema-backflow rule |

## License

[MIT](LICENSE) — Copyright (c) 2026 byte5 GmbH

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the dev setup, the worktree + PR
workflow, and the mandatory ANP schema-backflow rule: every change that defines
or alters an ANP object schema must open a matching PR against
[`byte5ai/anp`](https://github.com/byte5ai/anp).

## Maintainership

omadia Proof is maintained by [byte5 GmbH](https://byte5.de) under the GitHub
organisation [`byte5ai`](https://github.com/byte5ai).
