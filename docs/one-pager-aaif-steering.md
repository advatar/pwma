# Proposal: PWMA — Personal Wallet Management Agent Protocol (AAIF)

## The problem

Agentic systems (LLMs + tools) increasingly need to **hold, present, and exchange verifiable credentials** while acting across many domains (commerce, banking, tax, enterprise, healthcare). Today:

- Wallets are typically **human-operated** and **interactive**.
- Agents need to act with **least privilege**, **auditable consent**, and **bounded autonomy**.
- Credential exchange protocols exist (OpenID4VCI/OpenID4VP), but there is no standard way to expose a **personal wallet manager** to agents as a portable tool boundary.
- Key custody is ambiguous: agents must not inherit the human’s high-assurance signing keys, yet must still be able to **prove delegated authority** when acting autonomously.

## The proposal

**PWMA (Personal Wallet Management Agent)** is an AAIF protocol + reference architecture that standardizes a **personal wallet manager** exposed to agents via **MCP Tools**, while composing with:

- **Encrypted cloud wallet storage** inspired by wwWallet (cloud-synced encrypted container; client-side crypto; multiple vault unlock profiles—HAPP-gated by default, PRF/passkey optional). 
- **OpenID4VCI** for credential acquisition.
- **OpenID4VP** for credential presentation to relying parties.
- **HAPP** for human presence / explicit approvals when policy requires step-up.
- **Agentic commerce protocols** (e.g., ACP) via an action-instance canonicalization profile and test vectors.

PWMA’s central idea:

- The agent never holds the human’s root keys.
- The agent receives **mandates** (verifiable, scoped, expiring) and uses **agent operational keys** (HSM/TEE-backed) to act autonomously *within those bounds*.
- When a relying party or policy requires it, PWMA obtains a **HAPP Consent Credential** and upgrades the agent’s authority.

## Why MCP

MCP is the neutral tool interoperability layer for agents, but it intentionally does not define:

- wallet semantics,
- credential lifecycle,
- delegation policy,
- step-up interaction patterns.

PWMA fills this gap with a **standard MCP profile** (similar to HAPP over MCP), so any compliant agent host can integrate once and work with many wallet implementations.

## What gets standardized

1) **A Wallet Operation Intent DSL** (deterministic JSON + canonical hashing), enabling WYSIWYS-style audit and bounded envelopes.

2) **A portable Envelope object** with a **testable monotonicity algorithm** for safe multi-hop delegation.

3) **Mandate / capability artifacts** (scope + envelope + expiry + audience binding + action binding).

4) **Discovery metadata** for verifiers via `/.well-known/pwma-configuration` and JWKS.

5) **Challenge handling patterns**:
- OID4VP requests (present credentials)
- OID4VCI offers (receive credentials)
- HAPP challenges (step-up authorization)

6) **An MCP tool profile** for PWMA:
- discovery via `tools/list`
- invocation via `tools/call`
- URL/QR elicitation patterns for human interaction

## Deliverables for incubation

- PWMA specification + schemas + test vectors
- MCP profile for `aaif.pwma.request`
- Conformance requirements + interop harness
- Reference implementations:
  - PWMA MCP server
  - verifier/validator libraries (envelopes, mandates, capabilities)
  - OpenID4VCI/OpenID4VP adapters
  - HAPP integration module

## Ask

- Incubate PWMA as an AAIF standardization effort
- Define a governance model for:
  - issuer trust lists / registries
  - presence provider registries (via HAPP)
  - conformance testing
