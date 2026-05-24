# Risk Attestation Protocol (RAP)

*🇮🇹 [Leggi in italiano](README.it.md)*

**A protocol for letting a capable cloud AI work with your private data — without ever giving it your private data.**

A powerful cloud model never reads your raw private context. Instead it declares
*what it thinks it needs and why*, and a trusted local custodian — running where
your data lives — reviews that request, decides what may cross and at what
granularity, and returns only an approved projection. The custodian even decides
whether the cloud agent may answer you at all, or must hand its work back for
private finishing.

The whole negotiation happens through one inspectable document: the **risk
attestation**.

![How the attestation travels between the two agents](assets/lifecycle.svg)

## The two roles

- **Cloud Agent** — capable, untrusted. Runs on a large cloud model. Talks to you,
  reasons, drafts answers. Never touches your raw private data.
- **Private Custodian** — trusted. Runs where your data lives. The only thing that
  reads raw private context, and the sole authority over what gets released and
  whether the Cloud Agent may answer you.

They never share memory. The attestation is the only thing that crosses the
boundary — every field named, classified, and assigned a release granularity.

## Two delivery flows

The custodian decides, per request, whether the cloud agent can answer you
directly or must return a draft for private refinement.

![Flow A direct delivery, Flow B return to custodian](assets/two-flows.svg)

Flow B is what makes the boundary useful instead of merely restrictive: it lets a
powerful-but-untrusted model do real work on a question it isn't allowed to fully
see.

## What "release granularity" means

Each approved field crosses at a fixed fidelity — `plain`, `generalized`,
`range_only`, `tokenized`, `summarized`, `redacted`, or `yes_no`. The cloud agent
must treat these as hard ceilings.

![Car example: each field crosses at a different granularity](assets/car-example.svg)

The cloud model gets enough to do good work — a region, an income band — but the
exact address and the exact salary never leave the boundary.

## Read the spec

The full protocol — schema, required fields, release modes, the hard safety
invariant, and conformance criteria — is in **[SPEC.md](SPEC.md)**.

The design rationale, including why the exchange is deliberately plain text and
not a latent/opaque channel, and an honest account of what RAP does *not* protect
against, is in the [accompanying writeup](#).

## Status

Draft (v0.1.0). The protocol is implementation-neutral: anyone building an agent
that needs to touch private data without surrendering it can implement the two
roles and the document that passes between them.

## License

Specification released under [CC BY-SA 4.0](LICENSE).
