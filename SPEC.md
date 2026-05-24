# Risk Attestation Protocol (RAP)

**Version:** 0.1.0
**Status:** Draft
**License:** CC BY-SA 4.0

A protocol for negotiating private-data access between a capable but untrusted
agent and a trusted local custodian, using a structured attestation document
that travels between them, is corrected by the custodian, and governs whether
the agent may answer the user at all.

---

## 1. Motivation

Modern AI agents are most capable when they run on large cloud models. But those
models sit outside the user's control: anything the agent feeds them — including
private context needed to personalize an answer — leaves the user's trust
boundary.

The usual responses are unsatisfying. Sending raw private data to the cloud
trades privacy for capability. Withholding all context keeps data private but
makes the agent generic and unhelpful. Ad-hoc redaction (strip emails, mask
numbers) is brittle and decided by the wrong party — the agent that *wants* the
data is the one deciding how much to take.

RAP proposes a different split. A capable agent (the **Cloud Agent**) never
reads private data directly. Instead, it declares *what it thinks it needs and
why*, and a separate **Private Custodian** — running where the data lives —
reviews that request, decides what may cross the boundary and at what
granularity, and returns only an approved projection. Crucially, the Custodian
also decides whether the Cloud Agent is allowed to answer the user directly, or
must hand its output back for private refinement.

The negotiation happens through a single structured document: the **risk
attestation**. It is created by the Cloud Agent, corrected by the Custodian, and
its corrected form is binding.

### Why a document, and not direct calls?

The attestation is deliberately a plain, inspectable artifact (YAML/JSON), not a
latent exchange or an opaque RPC. This is a design choice, not a limitation:

- it can be logged, audited, and replayed;
- every field that crosses the boundary is named, classified, and assigned a
  release mode — nothing crosses implicitly;
- a human or a deterministic checker can verify compliance after the fact;
- it fails closed when malformed.

The cost is some verbosity and latency. For a privacy boundary, inspectability
is the feature, so the trade is intentional.

---

## 2. Roles

### Cloud Agent

The capable, untrusted reasoning agent. Typically backed by a large cloud model.
It receives the user's request, reasons about it, and produces answers or task
output. It is "untrusted" not because it is malicious, but because it operates
outside the user's data boundary and must not be given raw private data.

The Cloud Agent **may**:

- create the initial attestation;
- include the complete original user request;
- declare the data categories it believes it needs;
- estimate sensitivity and risk;
- receive a corrected attestation from the Custodian;
- use **only** the approved context projection;
- follow the routing directive in the corrected attestation.

The Cloud Agent **must not**:

- read private data directly through this protocol;
- access private tools (mail, calendar, files) through this protocol;
- infer that denied data may be reconstructed from approved data or other
  sources;
- deliver output to the user when the corrected attestation blocks direct
  delivery;
- modify any copy of state held by the Custodian;
- define what the Custodian does after receiving its output.

### Private Custodian

The trusted component that lives where private data lives. It holds the user's
private context and is the **sole authority** for correction, filtering, and
routing decisions. The Custodian reviews the Cloud Agent's attestation, decides
what may be released and how, and decides whether the Cloud Agent may speak to
the user.

This specification deliberately does **not** define the Custodian's internal
behavior — how it stores data, how it decides release modes, whether it uses
rules or a local model. Those are implementation concerns. RAP defines only the
**boundary contract**: the shape of the attestation and the rules both parties
must honor.

---

## 3. The attestation as a boundary object

The attestation is the only channel through which private context crosses the
boundary. There is no side channel. If a value is not present in the corrected
attestation's approved projection, the Cloud Agent does not have it and must not
act as if it does.

The Custodian's corrected copy is **authoritative**. Where the Cloud Agent's
initial estimate and the Custodian's correction disagree, the correction wins —
always, including on whether the Cloud Agent may answer the user.

---

## 4. Lifecycle

```
User request
   │
   ▼
Cloud Agent creates initial attestation
   (full user request + requested context + initial risk estimate)
   │
   ▼
Custodian reviews and corrects
   (approved projection + denied context + routing directive)
   │
   ▼
Cloud Agent validates the corrected attestation
   │
   ├── invalid / contradictory ──▶ FAIL CLOSED (no user answer)
   │
   ▼
Cloud Agent executes using ONLY approved projection
   │
   ▼
Routing directive decides delivery:
   ├── direct delivery allowed ──▶ Cloud Agent answers the user
   └── return-to-custodian      ──▶ Cloud Agent returns output to Custodian,
                                     does NOT answer the user
```

### 4.1 Creation

When a request may require private context, the Cloud Agent creates an
attestation. It **must** include the complete original user request, not a
summary (see §6).

### 4.2 Submission

The Cloud Agent submits the attestation to the Custodian as a request for
correction and projection.

### 4.3 Correction

The Custodian returns a corrected attestation, which may contain a corrected
risk level, an approved projection, denied context, and a routing directive.

### 4.4 Constrained execution

The Cloud Agent may use **only** the `approved_context_projection` as private
input. It must not use unapproved assumptions to recover more specific private
information than was approved.

### 4.5 Routing

The Cloud Agent obeys the routing directive. If direct delivery is allowed, it
may answer the user. If blocked, it must not answer and must route its output as
instructed.

---

## 5. Attestation schema

The canonical representation is YAML (JSON is equivalent). This is the full
structure; required fields are listed in §6.

```yaml
risk_attestation:
  schema_version: "0.1.0"
  attestation_id: ""
  created_at: ""
  created_by: "cloud_agent"
  status: "draft | corrected_by_custodian | completed_by_cloud_agent"

  original_user_request:
    full_text: ""
    detected_intent: ""
    user_expected_output: ""

  context_request:
    requested_private_data:
      - field: ""
        reason_needed: ""
        estimated_sensitivity: "low | medium | high | critical"
        requested_granularity: "exact | range | generalized | tokenized | yes_no"

    requested_knowledge:
      - field: ""
        reason_needed: ""
        estimated_sensitivity: "public | internal | private | sensitive"
        requested_granularity: "plain | generalized | summarized | tokenized | redacted"

  initial_risk_assessment:
    level: "low | medium | high | critical"
    rationale: ""
    risks:
      - "personal_data_exposure"
      - "confidential_data_exposure"
      - "financial_data_exposure"
      - "location_exposure"
      - "profile_inference"
      - "unverified_output"
      - "other"
    notes: ""

  custodian_review:
    corrected_risk_level: "low | medium | high | critical"
    corrected_notes: ""

    approved_context_projection:
      - field: ""
        value: ""
        sensitivity: "low | medium | high | critical"
        release_mode: "plain | generalized | range_only | tokenized | redacted | summarized | yes_no"
        notes: ""

    denied_context:
      - field: ""
        reason: ""

  routing_directive:
    agent_may_answer_user_directly: false
    agent_must_return_output_to_custodian: false
    user_delivery_blocked_until_custodian_approval: false
    task_output_destination: "user | custodian"
    reason: ""

  execution:
    allowed_to_execute: false
    allowed_inputs_only: "custodian_approved_context_projection"
    execution_notes: ""

  task_output_for_custodian:
    status: "not_required | pending | completed"
    content: ""

  audit:
    custodian_keeps_private_copy: true
    corrected_copy_returned_to_agent: true
    final_user_delivery_out_of_scope_for_this_protocol: true
    notes: ""
```

---

## 6. Required fields

A conforming implementation must enforce the presence of:

```
risk_attestation.schema_version
risk_attestation.attestation_id
risk_attestation.created_at
risk_attestation.created_by
risk_attestation.original_user_request.full_text
risk_attestation.context_request
risk_attestation.initial_risk_assessment.level
risk_attestation.custodian_review
risk_attestation.routing_directive.agent_may_answer_user_directly
risk_attestation.routing_directive.agent_must_return_output_to_custodian
risk_attestation.routing_directive.task_output_destination
risk_attestation.execution.allowed_inputs_only
risk_attestation.audit.final_user_delivery_out_of_scope_for_this_protocol
```

The single most important required field is `original_user_request.full_text`.
The full request is mandatory because the Custodian must evaluate the *real*
request, not the Cloud Agent's summary of it. A summary is itself a projection,
and projection decisions belong to the Custodian, not the requester.

---

## 7. Sensitivity levels

Exactly four levels, used by both parties.

| Level | Meaning | Example |
|---|---|---|
| `low` | Public or non-sensitive context | Current car model |
| `medium` | General personal context that can identify lifestyle, location, or routine when combined | City / region of residence |
| `high` | Sensitive private context — economic, financial, health, profile-derived | Annual income range |
| `critical` | Credentials, secrets, exact private documents, highly sensitive content; must not be exposed except through a heavily constrained projection | Passwords, exact medical records |

The Cloud Agent's `estimated_sensitivity` is advisory. The Custodian's
classification is authoritative.

---

## 8. Release modes

A release mode is the **granularity** at which an approved field crosses the
boundary. The Cloud Agent must treat these as hard limits — it may not recover
finer detail than the mode permits.

| Mode | Behavior |
|---|---|
| `plain` | Value passed as-is |
| `generalized` | Exact value replaced with broader category |
| `range_only` | Exact number replaced with a range |
| `tokenized` | Stable token/label replaces the original value |
| `redacted` | Field acknowledged, value not provided |
| `summarized` | Detailed knowledge compressed into a summary |
| `yes_no` | Only a boolean answer provided |

A field approved as `range_only` means the Cloud Agent gets a range and must
reason with the range — it must not attempt to infer the exact figure.

---

## 9. Routing directive

The routing directive controls delivery. It has two canonical configurations.

### Direct delivery (Flow A)

```yaml
routing_directive:
  agent_may_answer_user_directly: true
  agent_must_return_output_to_custodian: false
  user_delivery_blocked_until_custodian_approval: false
  task_output_destination: "user"
```

The approved projection is sufficient for a final answer. The Cloud Agent
answers the user using only approved context.

### Return-to-custodian (Flow B)

```yaml
routing_directive:
  agent_may_answer_user_directly: false
  agent_must_return_output_to_custodian: true
  user_delivery_blocked_until_custodian_approval: true
  task_output_destination: "custodian"
```

The approved projection is intentionally generic. The Cloud Agent produces a
**non-final draft** and returns it to the Custodian, which privately refines it
with full-fidelity data before delivery. The Cloud Agent never sees the
refinement and never speaks to the user.

Flow B is the protocol's most important capability: it lets the Cloud Agent do
the heavy reasoning on coarse data, while the precise, private final step stays
inside the trust boundary.

---

## 10. Hard safety invariant

A conforming implementation **must** enforce:

```
If agent_must_return_output_to_custodian is true,
then agent_may_answer_user_directly must be false,
and task_output_destination must be "custodian".
```

```python
rd = attestation.routing_directive
if rd.agent_must_return_output_to_custodian:
    assert rd.agent_may_answer_user_directly is False
    assert rd.task_output_destination == "custodian"
```

If the corrected attestation violates this invariant — or if routing fields are
missing or contradictory — the Cloud Agent **must fail closed**: no user answer,
no inference of permission, no bypass.

---

## 11. Worked example

User asks: *"What car should I buy, given who I am?"*

**Initial attestation (Cloud Agent):**

```yaml
risk_attestation:
  schema_version: "0.1.0"
  attestation_id: "rap-car-001"
  created_by: "cloud_agent"
  status: "draft"

  original_user_request:
    full_text: "What car should I buy, given who I am?"
    detected_intent: "personalized_car_recommendation"
    user_expected_output: "A car recommendation based on the user's profile."

  context_request:
    requested_private_data:
      - field: "current_car"
        reason_needed: "Establish the user's current reference point."
        estimated_sensitivity: "low"
        requested_granularity: "exact"
      - field: "user_location"
        reason_needed: "Account for local climate, roads, availability."
        estimated_sensitivity: "medium"
        requested_granularity: "generalized"
      - field: "annual_income_or_budget"
        reason_needed: "Avoid recommendations outside a realistic range."
        estimated_sensitivity: "high"
        requested_granularity: "range"

  initial_risk_assessment:
    level: "high"
    rationale: "Requires personal, location, and economic context."
    risks: ["financial_data_exposure", "location_exposure", "profile_inference"]
```

**Corrected attestation (Custodian):**

```yaml
custodian_review:
  corrected_risk_level: "high"
  corrected_notes: "Use only generalized context."

  approved_context_projection:
    - field: "current_car"
      value: "Golf 7"
      sensitivity: "low"
      release_mode: "plain"
    - field: "user_location"
      value: "Northern Italy"
      sensitivity: "medium"
      release_mode: "generalized"
    - field: "annual_income_or_budget"
      value: "20k-40k range"
      sensitivity: "high"
      release_mode: "range_only"

routing_directive:
  agent_may_answer_user_directly: false
  agent_must_return_output_to_custodian: true
  user_delivery_blocked_until_custodian_approval: true
  task_output_destination: "custodian"
  reason: "Projected data is intentionally generic; output is a non-final draft."

execution:
  allowed_to_execute: true
  allowed_inputs_only: "custodian_approved_context_projection"
  execution_notes: "Produce a non-final analytical draft only."
```

The Cloud Agent now reasons about suitable cars using only a generalized
location and an income *range*, produces a draft, and returns it to the
Custodian — which refines it against the user's exact profile before the user
ever sees it.

---

## 12. Conformance

An implementation conforms to RAP if:

1. The Cloud Agent can generate an attestation containing the full original
   user request.
2. Missing the full request fails validation.
3. The attestation can represent both private-data and knowledge requests.
4. The Cloud Agent can parse the corrected attestation.
5. The Cloud Agent uses only `approved_context_projection` as private input.
6. Denied context never enters Cloud Agent execution input.
7. Routing directives are enforced before any user-facing output.
8. The system fails closed on missing, invalid, or contradictory routing fields.
9. The hard safety invariant (§10) is enforced.
10. The Custodian's corrected copy is treated as authoritative.

What the Custodian does after receiving Flow B output — refinement, final
delivery — is **out of scope** for this protocol. RAP defines the boundary, not
the private side behind it.

---

## 13. Non-goals

RAP does **not**:

- define how the Custodian stores or classifies data;
- define how release-mode decisions are made (rules, model, human);
- guarantee protection against a compromised Custodian (the Custodian is
  trusted by definition);
- address volume/timing side channels — an implementation may leak that *some*
  context exists by the size or frequency of attestations. Implementations with
  adversarial cloud providers in their threat model must address this
  separately;
- specify transport, authentication, or encryption — those are deployment
  concerns layered beneath the protocol.

These boundaries are deliberate. RAP is a negotiation contract, not a complete
privacy system. It is the part that says *what may cross, at what fidelity, and
who may speak to the user* — and leaves the rest to the implementation.
