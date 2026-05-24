# A private custodian for your cloud AI: the Risk Attestation Protocol

I wanted to use frontier cloud models on my own life — my notes, my calendar, my
profile, the accumulated context that makes an assistant actually *mine* — without
shipping any of that to a provider I don't control. That tension is the whole
story. The capable models live in the cloud. The data I care about lives on my
machine and should stay there. Every obvious way to bridge that gap is bad.

This is a writeup of how I ended up resolving it: not with a clever redaction
filter, but with a small protocol where two agents negotiate, in writing, over
exactly what may cross the line — and where the side that owns the data has the
final say, including the power to forbid the cloud from answering me at all.

I'm calling the protocol RAP — Risk Attestation Protocol. The name comes from
the artifact at its center: a *risk attestation*, a structured document one agent
drafts and the other corrects. This piece explains the problem, the design, a
worked example, why I deliberately kept the exchange in plain text instead of
something cleverer, and where the whole thing does and doesn't protect you.

## The problem nobody solves well

Say you ask an assistant: *"What car should I buy, given who I am?"*

A good answer needs to know things about you. Your budget. Where you live. What
you currently drive. Maybe your stated preferences. A generic answer — "consider
a reliable hatchback" — is useless. A personal answer requires personal data.

If your assistant runs on a cloud model, that personal data has to go *somewhere*
to be reasoned about. The standard options:

- **Send it all.** The model gets your income, your address, your history, and
  gives a great answer. You've also just handed a third party a detailed profile
  of yourself. Privacy traded for capability.
- **Send nothing.** You keep your data, but the model can only give generic
  advice. Capability traded for privacy.
- **Redact on the way out.** Strip the obvious identifiers — emails, exact
  numbers — and send the rest. This feels responsible but is brittle, and worse,
  it's decided by the wrong party: the agent that *wants* the data is the one
  choosing how much to take. There's no separation of interest.

None of these is good because they all treat the question as "how much data do we
leak?" The better question is: *who decides what crosses, and at what fidelity,
and is that decider on your side?*

## Two agents, one boundary

RAP splits the system into two roles with deliberately unequal trust.

**The Cloud Agent** is the capable one. It runs on a large cloud model. It talks
to you, reasons, drafts answers. It is *untrusted* — not because it's malicious,
but because it lives outside your data boundary. It never touches your raw private
data. Ever.

**The Private Custodian** is the trusted one. It runs where your data lives — on
your own hardware. It is the only thing that reads your raw private context, and
it holds veto power over the Cloud Agent. The Custodian decides what gets
released, at what granularity, and — critically — whether the Cloud Agent is even
allowed to deliver an answer to you, or must hand its work back for private
finishing.

The two never share memory. They communicate only through one structured
document that travels between them: the attestation. There is no side channel. If
a value isn't in the attestation's approved projection, the Cloud Agent doesn't
have it and must behave as if it doesn't exist.

That last point is the design's spine. The boundary isn't enforced by hoping the
model behaves. It's enforced by the fact that the model is never given the data in
the first place, and by a hard rule about what it's allowed to do with what it
*is* given.

## How the negotiation works

The attestation goes through a fixed lifecycle.

**1. The Cloud Agent drafts.** When a request needs private context, the Cloud
Agent writes an initial attestation. It includes the *complete, verbatim user
request* — not a summary — plus a list of what it thinks it needs and why, plus
its own guess at the sensitivity of each item.

Why the full request and not a summary? Because a summary is itself a projection,
and projection decisions belong to the Custodian, not the requester. If the Cloud
Agent could pre-summarize, it could quietly smuggle framing past the only party
allowed to judge the real request. So the full text is mandatory.

**2. The Custodian corrects.** The Custodian reviews the draft against the actual
private data it holds. It produces an *approved projection*: for each requested
field, an actual value plus a **release mode** that fixes the granularity. It can
also *deny* fields outright. And it sets a **routing directive** that decides
delivery.

**3. The Cloud Agent obeys.** It validates the corrected attestation. If anything
is missing or contradictory, it fails closed — no answer. Otherwise it executes
using *only* the approved projection, and routes its output exactly as the
directive says.

The release modes are the granularity dial. A field can come back as `plain`
(as-is), `generalized` (a broader category instead of the exact value),
`range_only` (a number replaced by a band), `tokenized` (a stable label),
`summarized`, `redacted` (acknowledged but withheld), or `yes_no` (a boolean and
nothing more). The Cloud Agent must treat these as hard ceilings — it may not try
to recover finer detail than the mode allows.

## The car example, end to end

Back to *"What car should I buy, given who I am?"*

The Cloud Agent drafts an attestation requesting three things: current car
(low sensitivity, wants it exact), location (medium, wants it generalized), and
income or budget (high, wants a range). It rates the overall request as high risk.

The Custodian reviews and returns this projection:

- current car → `Golf 7`, release mode `plain`
- location → `Northern Italy`, release mode `generalized` (not the city, not the
  address — just the region)
- income → `20k–40k range`, release mode `range_only` (a band, never the figure)

And then it sets the routing directive to **return-to-custodian**: the Cloud Agent
may *not* answer me directly. It must produce a draft and hand it back.

Why? Because the projection it was given is intentionally coarse. The Custodian
wants the Cloud Agent to do the heavy lifting — survey suitable cars, reason about
trade-offs — on generalized data, and then it will privately refine that draft
against my *exact* profile before I ever see it. The expensive reasoning happens
on coarse data in the cloud; the precise, personal final step stays inside the
boundary.

So the cloud model thinks hard about "reliable cars for a Northern Italy driver in
a 20–40k income band currently driving a Golf 7," produces a solid draft, and the
Custodian — which knows my real numbers, my real city, my real preferences —
sharpens it locally. The cloud never learned where I live or what I make. It
learned a region and a band, did good work with them, and was structurally
prevented from addressing me directly.

This two-flow design is the part I'm proudest of. The simple flow — direct
delivery, where the projection is enough and the Cloud Agent just answers — is the
common case. But the return-to-custodian flow is what makes the boundary *useful*
instead of merely restrictive. It lets a powerful-but-untrusted model contribute
real work to a question it's not allowed to fully see.

## The hard invariant

There's one rule the implementation enforces absolutely:

```
If the Cloud Agent must return output to the Custodian,
then it may NOT answer the user directly,
and the output destination must be the Custodian.
```

If a corrected attestation ever violates this — says "return to custodian" but
also "you may answer the user" — the Cloud Agent fails closed and stays silent.
Contradiction is treated as danger, not as something to resolve with a guess.
This is the kind of property you want to be boring and absolute, because it's the
last line between "the cloud drafts a private answer" and "the cloud *delivers* a
private answer."

## Why plain text, and not something cleverer

Here's a question worth confronting directly, because the obvious objection to RAP
is that it's *inefficient*. The agents pass a verbose YAML document back and
forth. That costs tokens and latency. There's recent and genuinely impressive work
— for instance the RecursiveMAS line of research — showing that you can make
multi-agent systems collaborate in *latent space* instead of text: pass the
models' internal hidden states directly from one agent to the next, skip the
decode-to-text step entirely, and save enormous amounts of tokens and time. Up to
75% fewer tokens in their reported results.

So why didn't I do something like that?

Because for a *privacy* boundary, the verbosity is the feature, not the bug.

A latent state is fast but opaque. You can't read it, can't audit it, can't
sanitize it, can't log it in a way a human or a deterministic checker can later
verify. It's a vector of numbers whose contents you can't inspect. That's exactly
what you do *not* want crossing a trust boundary whose entire purpose is to control
what crosses it.

The RAP attestation is the opposite. It's slow and chatty, but every single field
that crosses is named, classified, assigned a release mode, and written down. You
can log it. You can replay it. You can audit it after the fact. You can fail it
closed when it's malformed. The whole thing is built so that "what did the cloud
get to see?" has a precise, inspectable answer.

The two approaches optimize different axes. Latent collaboration optimizes
*efficiency* — fewer tokens, faster rounds. RAP optimizes *governance* — what may
cross, at what fidelity, and who may speak to the user. On the efficiency axis,
inspectability is overhead. On the governance axis, inspectability is the entire
point. I'm not competing with the latent-space work; I'm solving a different
problem, and on my problem the plain-text artifact wins precisely because it's
plain.

(There's a hypothetical future where both sides run locally — a powerful local
model playing the Cloud Agent role on your own hardware. In that world the trust
boundary changes character and a latent exchange between *your own* models becomes
thinkable. But that's a different system, and it doesn't change the calculus for
the cloud-boundary case, which is the one that matters today.)

## What RAP does not protect against

A privacy mechanism that oversells itself is worse than none, because people lean
on guarantees that aren't there. So, plainly, the limits:

**RAP does not defend against a compromised Custodian.** The Custodian is the root
of trust for your private data — it reads everything. If it's compromised, the game
is over. RAP assumes the Custodian is trustworthy by definition; it protects your
data *from the cloud*, not from the thing that holds it.

**RAP does not eliminate volume and timing side channels.** Even if no field value
ever leaks, the *fact that an attestation exists* — its size, its frequency — can
tell an observing cloud provider that some private context was involved. In my own
deployment I accept this: I don't treat the provider as an adversary trying to
profile me through traffic analysis, on the reasoning that a provider wanting
general information about me has far easier routes. But that's a choice I made for
*my* threat model. If yours treats the cloud provider as an active adversary, you
have to address this separately, and RAP alone won't do it.

**RAP does not specify transport, auth, or encryption.** Those live below the
protocol. RAP is the negotiation contract — what crosses and who decides — not the
secure channel it rides on. You layer the usual things underneath.

**RAP is not a complete privacy system.** It's one well-defined piece: the part
that says what may cross the boundary, at what fidelity, and who may answer the
user. The data store, the classification logic, the release-mode rules, the final
delivery — those are implementation, and they're where most of the real work and
the real risk lives.

I think being explicit about this is what separates a credible design from a
hopeful one. RAP draws a clean line around a hard problem and solves *that*, and
says clearly what's outside the line.

## Where this came from, and where it's going

I built this as part of a personal system — a local AI setup where a cloud-backed
agent works against a private vault on my own hardware, mediated by exactly this
kind of attestation exchange. RAP is the boundary contract extracted from that
system and generalized so it doesn't depend on my specific stack. Anyone building
an agent that needs to touch private data without surrendering it can implement
the two roles and the document that passes between them.

The protocol spec — the full schema, the required fields, the release modes, the
invariant, the conformance criteria — is published separately as a standalone
document, deliberately neutral about implementation. This article is the *why*;
the spec is the *what*.

If you've ever wanted to point a frontier model at your own life without handing
your life over, this is one way to draw that line so that the side holding the
pen is yours.

---

*The RAP specification is available as a standalone repository. This writeup
accompanies it as the design rationale.*
