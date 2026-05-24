# Privacy Architecture: How zarsOS Implements RAP

> This document describes how zarsOS instantiates the
> [Risk Attestation Protocol (RAP)](https://github.com/parifede/risk-attestation-protocol).
> RAP is the abstract boundary contract; this document is the concrete
> implementation. Read the RAP spec first for the protocol itself — here we only
> describe the zarsOS-specific mapping.

---

## 1. The three components

zarsOS separates the agentic system into three roles with distinct trust levels
(see ADR-001 for the canonical vocabulary).

| Component | RAP role | Trust | Backed by |
|---|---|---|---|
| **Zarsuit** | Cloud Agent | Untrusted on private data | Large cloud model |
| **il_segretario** | Private Custodian | Trusted; sole owner of the Vault | Local model + deterministic policy |
| **zarsOS** | (mediator — not a RAP role) | Infrastructure | Local kernel |

A note on the mapping: RAP defines two roles, Cloud Agent and Private Custodian.
zarsOS adds a third element that RAP does not name — **zarsOS itself**, the
infrastructure that carries the attestation between the two parties. RAP is
silent on transport; in zarsOS, the kernel is the transport, plus the auth,
audit, and routing enforcement around it.

- **Zarsuit** is the agent the user talks to. It has identity, character, and
  working memory, but no direct access to the Vault.
- **il_segretario** is the custodian of the Vault. It is the only component that
  reads raw private data, and it holds veto power over whether Zarsuit may answer
  the user.
- **zarsOS** is the infrastructure (gateway, kernel, permission checks, audit,
  secretary bridge) inside which Zarsuit operates. It does not talk to the user
  and has no character.

The Vault — the user's private persistent memory — is reachable only by
il_segretario. Neither Zarsuit nor zarsOS reads it directly.

---

## 2. How the attestation travels

RAP's lifecycle (spec §4) maps onto zarsOS transport as follows. Zarsuit and
il_segretario never speak directly; every hop goes through the zarsOS kernel.

```
User (device)
   │  HTTPS over tailnet
   ▼
zarsOS Gateway
   │  in-process
   ▼
zarsOS kernel ── instantiates Zarsuit (cloud model call) ──▶ Zarsuit drafts attestation
   │
   │  HTTP loopback 127.0.0.1:8722
   ▼
il_segretario (RAP Private Custodian)
   │  reviews, corrects, builds approved projection + routing directive
   ▼
zarsOS kernel ◀── corrected attestation
   │
   ▼
Zarsuit executes on approved projection (cloud model call)
   │
   ├── Flow A (direct delivery) ──▶ zarsOS Gateway ──▶ user
   └── Flow B (return-to-custodian) ──▶ il_segretario refines ──▶ Gateway ──▶ user
```

Two consequences of this topology worth stating explicitly:

- **Zarsuit never sees the Vault, il_segretario never sees the cloud.** Each
  party knows only zarsOS. il_segretario does not know Anthropic exists; Zarsuit
  does not know the Vault's internal structure. The attestation is the only
  thing that crosses, and it crosses through zarsOS.
- **The cloud model animates Zarsuit; it is not Zarsuit.** Zarsuit's identity
  lives in the character configuration that zarsOS recomposes on each turn. The
  cloud model is the substrate, not the agent (ADR-001).

---

## 3. Component mapping

The RAP attestation fields map onto concrete zarsOS modules.

| RAP concept | zarsOS implementation |
|---|---|
| Cloud Agent drafts attestation | `ZarsuitOrchestrator` builds the initial `risk_attestation` |
| Custodian reviews/corrects | il_segretario's attestation handler + policy engine |
| `approved_context_projection` | Built by il_segretario's `ContextBroker` (3-level budget) |
| Release-mode decisions | il_segretario privacy projection rules over the Vault |
| `denied_context` | Fields the permission policy refuses |
| Routing directive | il_segretario decides Flow A vs Flow B |
| Hard safety invariant (§10) | Enforced in zarsOS Flow 02 routing + il_segretario contract verifier |
| Custodian-side refinement (Flow B) | il_segretario private refinement (out of RAP scope) |
| Audit | zarsOS audit hash-chain (`risk_attestation_exchange`, `flow_02_secretary_return` events) |

The HTTP surface between zarsOS and il_segretario uses Bearer auth on loopback
(or tailnet); the attestation rides inside the request body as a structured
envelope. This transport layer is below RAP — RAP is silent on it, and zarsOS
fills it in.

---

## 4. Where zarsOS goes beyond RAP

RAP is a minimal boundary contract. zarsOS adds operational machinery around it
that the protocol does not mandate:

- **Retry loop.** When il_segretario judges Zarsuit's returned output to be off
  contract (goal mismatch, malformed schema, incomplete), it can reissue the
  request as a *new* request with a possibly widened projection — up to twice.
  The check is on contract compliance, not factual correctness. RAP says nothing
  about retries; this is a zarsOS refinement.
- **Adaptive context budget.** il_segretario fills only the context level 2
  actually needs per request category, rather than padding to a ceiling. Because
  projected data is already tokenized/generalized, the limit is performance and
  cloud-token cost, not privacy.
- **Sessions.** A session closes only after both >24h since open and >1h
  inactivity, which guarantees a session cannot close mid-retry.
- **Deterministic-first correction.** il_segretario aims to make the bulk of the
  correction deterministic (rules over the Vault's privacy map), invoking the
  local model only for genuinely ambiguous classification. RAP allows any
  mechanism; zarsOS chooses to minimize local-model use for speed.

---

## 5. Threat model notes specific to this deployment

RAP §13 lists protocol-level non-goals. zarsOS's deployment makes some explicit
choices on top of them:

- **The cloud provider is not in the adversary set.** zarsOS accepts that the
  *volume* of attestations may leak that some private context exists, to the
  cloud provider. This is an accepted side channel for this single-user
  deployment, on the reasoning that a provider wanting general user information
  could obtain it more easily by other means. A future fully-local Zarsuit would
  remove even this.
- **The Custodian is trusted by definition.** Consistent with RAP, zarsOS does
  not defend against a compromised il_segretario — it is the root of trust for
  private data.
- **Local models stay local.** No Vault data leaves the hub except as a
  sanitized projection inside an attestation.

Anyone adapting this architecture to a different threat model — for instance one
where the cloud provider *is* an adversary — must revisit these choices. They
are properties of the zarsOS deployment, not of RAP.

---

## 6. Summary

zarsOS is a concrete instantiation of RAP where the Cloud Agent is **Zarsuit**
(a character-bearing agent on a cloud model), the Private Custodian is
**il_segretario** (custodian of a local Obsidian-style Vault), and the
infrastructure carrying the attestation between them is **zarsOS** itself. The
protocol defines what may cross the boundary and who may speak to the user;
zarsOS supplies the transport, the audit, the retry loop, and the deployment
threat model around it.
