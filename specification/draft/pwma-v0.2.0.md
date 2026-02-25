# PWMA v0.2.0 — Personal Wallet Management Agent Protocol (Draft)

**Status:** Draft  
**Version:** 0.2.0  
**Track:** AAIF Standards / MCP Profile  

## 1. Abstract

PWMA (Personal Wallet Management Agent) standardizes how an agentic system can:

1) **Store and manage verifiable credentials (VCs)** in a cloud-synced encrypted wallet container,  
2) **Obtain and enforce least-privilege delegated authority** for actions,  
3) **Enable bounded autonomy** via reusable, policy-limited mandates and per-action capabilities, and  
4) **Present verifiable proofs** to relying parties using existing OpenID for Verifiable Credentials protocols.

PWMA is designed to compose with:

- **Encrypted cloud wallet containers** inspired by wwWallet’s “cloud-synced encrypted container; crypto on the client” architecture, including WebAuthn-based unlock options and end-to-end encrypted vault sync.
- **OpenID for Verifiable Credential Issuance (OpenID4VCI)** for credential issuance.
- **OpenID for Verifiable Presentations (OpenID4VP)** for credential presentation (including `direct_post`).
- **HAPP** (Human Authorization & Presence Protocol) for step-up approvals when policy requires explicit human presence and authorization.
- **Agentic commerce protocols** (e.g., OpenAI ACP), via a normative **action-instance canonicalization** profile and test vectors.

PWMA is designed to be:

- **Deployable today** as an MCP server profile (`aaif.pwma.*`),
- **Least-privilege by default** (scoped, expiring, audience-bound),
- **Autonomy-safe** (envelopes + replay protection + sender-constrained tokens),
- **Auditable** (mandates, capabilities, and receipts are verifiable artifacts),
- **Standard-ready** (well-known metadata discovery, JSON Schemas, and conformance language).

## 2. Motivation

Agents can already read and reason; next they will routinely execute high-impact actions (payments, submissions, account changes). Existing authorization patterns often prove only that *some caller* is authenticated, not that:

- the caller is a specific accountable agent,
- the action is within an authorized policy envelope,
- the authorization is fresh and replay-resistant,
- a human approved the right thing at the right time.

Most wallet/VC protocols assume direct user operation and interactive confirmation, which does not map cleanly onto delegated agents.

PWMA provides a standard “wallet governor” that can:

- keep **human root keys** out of agent runtimes,
- mint **bounded, expiring authority** for agents,
- use **HAPP** only when policy requires step-up,
- satisfy OpenID4VC flows without inventing new issuance/presentation protocols,
- make relying-party verification practical via **metadata discovery** and **canonical action binding**.

## 3. Goals and non-goals

### 3.1 Goals

PWMA v0.2.0 MUST:

- Standardize a **wallet manager** exposed as an **MCP Server**.
- Support ingesting and storing credentials obtained via **OpenID4VCI**.
- Support presenting credentials using **OpenID4VP**, including cross-device and `direct_post` flows.
- Support **mandates** (multi-use, envelope-bounded authority) and **capabilities** (short-lived, per-action authorization).
- Support **subdelegation** (User → PWMA → Task Agent → Sub-agent) with strict **monotonicity** enforcement.
- Integrate with **HAPP** for step-up approvals when required by PWMA policy or RP policy.
- Provide **verifiable receipts** for important operations.
- Provide **PWMA metadata discovery** via a `.well-known` endpoint so verifiers can obtain signing keys and supported profiles.

### 3.2 Non-goals

PWMA does NOT:

- Define biometric/liveness algorithms (delegated to HAPP Presence Providers).
- Replace OAuth/OIDC; PWMA composes with them.
- Standardize all credential formats; it transports credential formats supported by OpenID4VC ecosystems.
- Guarantee that a human understood content; it guarantees explicit approvals when required (via HAPP) and deterministic binding between approvals and actions.

## 4. Conventions and terminology

### 4.1 Requirements language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in **BCP 14** (RFC 2119 and RFC 8174).

### 4.2 Roles

- **Principal (Human Principal):** natural person who owns the wallet and is the ultimate authority.
- **Task Agent:** a general agent runtime (e.g., a ChatGPT-style agent) that calls tools.
- **Sub-agent:** a narrower-purpose agent/service invoked by the Task Agent.
- **PWMA Server:** the wallet manager + policy engine exposed over MCP.
- **Vault Store:** cloud persistence for encrypted wallet containers (untrusted storage).
- **Key Custody Service (KCS):** HSM/TEE-backed service that holds agent operational keys (RECOMMENDED).
- **Presence Provider (PP):** HAPP provider that produces portable step-up approval evidence.
- **Issuer:** VC issuer (government wallet, enterprise IdP, bank, etc.).
- **Verifier / Relying Party (RP):** asks for proofs and/or executes actions.

### 4.3 Artifacts

- **PWMA-INTENT:** deterministic JSON describing a wallet operation requested by an agent.
- **PWMA-ENVELOPE:** deterministic constraints object used for bounded autonomy.
- **PWMA-MANDATE:** a verifiable artifact encoding ongoing authority within a bounded envelope.
- **PWMA-CAP:** a short-lived, per-action capability token minted under a mandate.
- **PWMA-REC:** a verifiable receipt emitted by PWMA for auditability.
- **HAPP-CC:** a HAPP consent credential (external spec) used for step-up approvals.
- **PWMA-CONFIG:** metadata document served from a `.well-known` endpoint for discovery.

## 5. Architectural model

### 5.1 Trust boundaries (recommended)

```mermaid
flowchart LR
    subgraph UserDomain[User domain]
      U[Human Principal]
      EU[EU Wallet / HAPP profile]
    end

    subgraph AgentDomain[Agent runtime]
      TA[Task Agent]
      SA[Sub-agent]
      MCP[MCP Wallet App]
    end

    subgraph PWMADomain[PWMA domain]
      PW[PWMA Server]
      KCS[Key Custody Service\n(HSM/TEE)]
      Vault[Encrypted Vault Container\n(untrusted store)]
      Policy[Policy Engine]
      WK[.well-known\nPWMA-CONFIG + JWKS]
    end

    subgraph RPDomain[Relying parties]
      RP1[Merchant / PSP]
      RP2[Bank API]
      RP3[Tax authority]
      Ver[OID4VP Verifier]
    end

    TA -->|MCP tools/call| PW
    SA -->|MCP or A2A| TA
    PW --> Policy
    PW --> KCS
    PW --> Vault
    PW --> WK
    PW -->|OID4VCI| Iss[VC Issuers]
    PW -->|OID4VP| Ver
    PW -->|Capabilities| RP1
    PW -->|Read-only delegations| RP2
    PW -->|Submission w/ step-up| RP3
    PW -->|Step-up| EU
    U -->|Approves| EU
    MCP --> TA
    RP1 -.->|verify iss via PWMA-CONFIG| WK
```

PWMA assumes three primary trust domains:

1) **User domain**  
   - user approves step-ups (HAPP) on a provider-controlled UI (e.g., EU Wallet profile)
   - user retains high-assurance root keys

2) **PWMA domain**  
   - policy and orchestration boundary
   - stores encrypted wallet contents (via vault store)
   - holds agent operational keys (via KCS)
   - publishes verification metadata (PWMA-CONFIG) and signing keys (JWKS)

3) **Relying party domain**  
   - enforces its own policy at execution boundary
   - verifies mandates/capabilities/presentations
   - maintains replay caches / single-use consumption state

### 5.2 Key separation (MUST)

PWMA implementations MUST separate:

- **Human root keys**: used only in user-controlled systems (e.g., EU wallet / HAPP PP). PWMA MUST NOT require exporting these keys.
- **Agent operational keys**: used to identify and authenticate an acting agent and to bind proof-of-possession (PoP). These keys MUST be distinct from human root keys.
- **Vault encryption keys**: used to encrypt wallet container contents; MUST NOT be transmitted in plaintext to the vault store.

### 5.3 Two-tier wallet storage (RECOMMENDED)

PWMA SHOULD implement:

- **Cold Vault**: encrypted container holding high-sensitivity credentials and long-lived secrets. Access MAY require step-up (HAPP) and/or strong unlock factors.
- **Hot Wallet**: operational cache (active mandates, capability counters, pending receipts, ephemeral presentation state) protected by KCS/TEE policy. Hot Wallet enables autonomy without routinely decrypting the Cold Vault.

### 5.4 Vault security profiles (MUST be profile-based)

PWMA MUST support a profile-based approach for unlocking the Cold Vault. A conforming PWMA implementation MUST implement at least one of these profiles:

- **PWMA-VAULT-HAPP-RELEASE (RECOMMENDED default)**  
  Cold Vault key is sealed in KCS and released (unwrapped) only after PWMA verifies a fresh HAPP-CC meeting policy.
- **PWMA-VAULT-PRF (OPTIONAL)**  
  Cold Vault key is wrapped by a secret derived using a WebAuthn PRF-like mechanism (wwWallet style), or another PRF-like derived secret provider.
- **PWMA-VAULT-LOCAL (OPTIONAL)**  
  Cold Vault stored and decrypted locally (for on-device PWMA deployments).

A deployment MUST publish its supported vault profiles in PWMA-CONFIG (Section 12).

## 6. Data model overview

PWMA defines these primary JSON objects (schemas in `/schemas`):

1) **PWMA-INTENT** — deterministic request for a wallet operation  
2) **PWMA-ENVELOPE** — deterministic constraints object for bounded autonomy  
3) **PWMA-MANDATE** — ongoing authority within envelope  
4) **PWMA-CAP** — per-action capability token  
5) **PWMA-RESULT** — tool result envelope for MCP responses  
6) **PWMA-REC** — audit receipt envelope  
7) **PWMA-CONFIG** — metadata document for discovery  

PWMA also defines:

- **Action instances**: deterministic, profile-defined JSON objects hashed into `action_hash` for capabilities.

## 7. Canonicalization and hashing

### 7.1 Canonicalization scheme

PWMA objects that are hashed (PWMA-INTENT, action instances, envelope views) MUST be canonicalized using **RFC 8785 (JCS)** before hashing.

### 7.2 Restricted JSON types (RECOMMENDED)

To maximize cross-language determinism, implementations SHOULD restrict hashed objects to:

- JSON objects, arrays, strings, booleans, and integers
- no floating point numbers
- timestamps as RFC 3339 strings

### 7.3 Hash format

For any hashed object `X`:

`hash = "sha256:" + base64url( SHA-256( UTF8( JCS(X) ) ) )`

The prefix `sha256:` is REQUIRED.

### 7.4 Action instances and `action_hash` (NEW in v0.2)

A **PWMA action instance** is a profile-defined JSON object representing the *specific action* that a relying party will execute (e.g., “complete checkout session X for amount Y”).

- The RP MUST be able to deterministically reconstruct the same action instance (or an equivalent canonical representation) from its local execution context.
- The RP recomputes `action_hash` over that action instance and compares it to the `action_hash` claim in the capability token.

PWMA defines one normative action-instance profile in this draft:

- `aaif.pwma.action.acp.checkout_complete/v0.1` (Annex E.2)

Additional action-instance profiles can be registered over time.

## 8. PWMA-INTENT v0.2

PWMA-INTENT is a deterministic, machine-readable description of a wallet operation requested by an agent.

### 8.1 Required fields

PWMA-INTENT MUST contain:

- `version` (string, `"0.2"`)
- `profile` (string)
- `intentId` (uuid)
- `issuedAt` (RFC3339 timestamp)
- `audience` (identifier of the party that will rely on the result; often the PWMA itself)
- `agent` (identifier of requesting agent; includes `id` and MAY include `cnf.jkt`)
- `operation` (typed operation)
- `constraints` (expiry, oneTime, envelope)
- `display` (human-readable fields; for logging/UI)

### 8.2 Profiles

PWMA-INTENT MUST declare a `profile`. Profiles define:

- required operation parameters,
- envelope semantics for the operation,
- subdelegation semantics.

PWMA v0.2 defines these baseline profiles:

- `aaif.pwma.mandate.generic/v0.2`
- `aaif.pwma.capability.generic/v0.2`
- `aaif.pwma.oid4vp.bridge/v0.2`
- `aaif.pwma.oid4vci.bridge/v0.2`
- `aaif.pwma.agent.register/v0.2`

Unknown profiles MUST be rejected with `PWMA-ERR-UNSUPPORTED-PROFILE`.

### 8.3 Envelopes (bounded autonomy)

An intent MAY include `constraints.envelope` (PWMA-ENVELOPE v0.2).

If an envelope is present:

- PWMA MUST include envelope constraints in any step-up approval request (HAPP AI-INTENT conversion).
- PWMA MUST encode the envelope (or an equivalent canonical representation) in the resulting mandate.
- RPs MUST enforce the envelope at execution time if they accept PWMA capabilities for the action.

## 9. Envelopes: PWMA-ENVELOPE v0.2 (NEW in v0.2)

A PWMA-ENVELOPE is a deterministic constraints object used to express bounded autonomy in a portable, verifiable way.

### 9.1 Envelope structure (baseline)

PWMA-ENVELOPE v0.2 is a JSON object:

- `version: "0.2"`
- `constraints: { ... }`
- optional `extensions: [...]`

PWMA defines a baseline set of constraint keys whose semantics and **monotonicity rules are testable**:

- `amount_minor` — per-action amount bounds in minor units (e.g., cents)
- `max_total_amount_minor` — lifetime/counter total bounds in minor units
- `merchant_id` — allow-list of merchants
- `category` — allow-list of categories (deployment-defined vocabulary)
- `mcc` — allow-list of merchant category codes (MCC)
- `shipping_country` — allow-list of destination countries (ISO 3166-1 alpha-2)
- `audience` — allow-list of audiences (RPs)
- `payment_provider` — allow-list of payment processors/providers
- `max_uses` — maximum number of capability consumptions allowed under this mandate

Implementations MAY define additional constraint keys, but they MUST be expressed via `extensions` unless standardized.

### 9.2 Envelope evaluation (profile-defined)

Envelope constraints are applied to **an action instance**.

Each PWMA-INTENT profile MUST specify how envelope keys map to fields in its action instance.

Example (informative): for ACP checkout completion, `amount_minor` applies to `$.acp.total_amount_minor` (Annex E.2).

### 9.3 Envelope monotonicity algorithm (MUST)

When PWMA issues a **child mandate** or **capability** under a parent mandate, PWMA MUST enforce:

> **child.envelope ⊆ parent.envelope** (the child is equal or more restrictive)

This section defines a fully testable baseline algorithm for v0.2 envelopes.

#### 9.3.1 Definitions

Let:

- `P` be the parent envelope
- `C` be the child envelope

If an envelope is absent, treat it as:

- `version: "0.2"`
- `constraints: {}`

#### 9.3.2 Core rule (REQUIRED)

For every baseline constraint key present in `P.constraints`, the same key MUST be present in `C.constraints`, and `C` MUST be **equal or stricter** than `P` under the key’s comparison rule.

Child envelopes MAY introduce new constraint keys not present in the parent (additional restrictions).

#### 9.3.3 Comparison rules (baseline keys)

**A) `amount_minor` and `max_total_amount_minor`**

Object shape:

- `currency` (required, lowercase ISO 4217)
- `min` (optional integer ≥ 0)
- `max` (optional integer ≥ 0)

Monotonicity:

- `C.currency` MUST equal `P.currency` if `P.currency` is present
- if `P.max` exists then `C.max` MUST exist and `C.max ≤ P.max`
- if `P.min` exists then `C.min` MUST exist and `C.min ≥ P.min`
- if `P.max` does not exist, `C.max` MAY exist (adding a max is more restrictive)
- if `P.min` does not exist, `C.min` MAY exist (adding a min is more restrictive)

**B) `merchant_id`, `category`, `mcc`, `shipping_country`, `audience`, `payment_provider`**

Object shape:

- `in` (required array of unique strings)

Monotonicity:

- `set(C.in) ⊆ set(P.in)`

**C) `max_uses`**

Object shape:

- `le` (required integer ≥ 1)

Monotonicity:

- `C.le ≤ P.le`

#### 9.3.4 Extensions (MUST be safe)

Extensions are objects:

- `type` (string identifier)
- `data` (object)

For any extension `type` present in `P.extensions`, the child MUST include an extension of the same `type`.

Because extension semantics are not generally known, **extension monotonicity is defined as strict equality**:

- `C.extensions[type].data` MUST be deeply equal to `P.extensions[type].data`

This conservative rule guarantees that a child cannot relax an unknown restriction.

#### 9.3.5 Pseudocode (informative)

```text
function envelope_is_subset(C, P):
  assert C.version == "0.2" and P.version == "0.2"

  for each key in baseline_keys:
    if key in P.constraints:
      if key not in C.constraints: return false
      if not constraint_subset(C.constraints[key], P.constraints[key], key): return false

  // extensions
  for each extP in P.extensions:
    extC = findByType(C.extensions, extP.type)
    if extC == null: return false
    if deepEqual(extC.data, extP.data) == false: return false

  return true
```

Conformance tests for this algorithm are defined in `pwma-conformance-v0.2.0.md` and test vectors in `/test_vectors/v0.2`.

## 10. Mandates: PWMA-MANDATE v0.2

A PWMA-MANDATE is a verifiable artifact representing delegated authority within a bounded envelope.

### 10.1 Purpose

Mandates allow:

- **bounded autonomy** (repeated low-risk actions without repeated step-up),
- **policy enforcement** (scope + envelope + expiry),
- **subdelegation** (mint narrower child mandates/capabilities).

### 10.2 Required claims (logical)

A mandate MUST contain (logically) the following claims:

- `iss` — mandate issuer (PWMA issuer identifier)
- `sub` — agent subject (agent identifier)
- `aud` — allowed audience(s) (verifier/RP identifiers or classes)
- `jti` — unique identifier
- `iat`, `exp` — issuance and expiry times
- `scope` — one or more action scopes
- `envelope` — PWMA-ENVELOPE v0.2
- `delegation` — (optional) parent linkage, chain metadata
- `cnf` — proof-of-possession binding to agent key (REQUIRED unless the token is otherwise sender-constrained)

### 10.3 Format requirements

For interoperability, PWMA-MANDATE MUST support `format: "jwt"` in the PWMA result envelope.

PWMA MAY support additional formats (`vc+json`, `vc+sd-jwt`, `cwt`) as capabilities.

### 10.4 Subdelegation rules (MUST)


```mermaid
flowchart LR
  U[User / Principal] -->|delegates| PWMA[PWMA]
  PWMA -->|issues mandate| TA[Task Agent]
  TA -->|subdelegates\n(narrower scope)| SA[Sub-agent]

  note1[[Monotonicity rules:\nchild ⊆ parent (scope/aud/exp/envelope)]]
  PWMA --- note1
```


If PWMA issues a child mandate under a parent mandate, PWMA MUST enforce:

- **Scope monotonicity:** `child.scope` MUST be a subset of `parent.scope`.
- **Audience monotonicity:** `child.aud` MUST be equal to or narrower than `parent.aud`.
- **Time monotonicity:** `child.exp` MUST be ≤ `parent.exp`.
- **Envelope monotonicity:** `child.envelope` MUST be a subset of `parent.envelope` per Section 9.3.

If any monotonicity condition fails, PWMA MUST reject issuance.

## 11. Capabilities: PWMA-CAP v0.2

A PWMA-CAP is a short-lived authorization artifact minted under a mandate for a specific action instance.

### 11.1 Purpose

Capabilities provide:

- strong replay resistance (`jti` single-use + very short TTL),
- binding to a specific action instance (`action_hash`),
- sender constraint / proof of possession binding to agent keys.

### 11.2 Required claims (logical)

A capability MUST contain:

- `iss` — capability issuer (PWMA)
- `sub` — agent identifier
- `aud` — specific RP identifier
- `jti` — unique (MUST be single-use unless policy explicitly allows reuse)
- `iat`, `exp` — short validity (RECOMMENDED minutes)
- `mandate_jti` — reference to the parent mandate identifier
- `action_hash` — hash of the canonical action instance
- `cnf` — PoP binding to agent key (see Section 11.3)

### 11.3 Sender constraint (RECOMMENDED: DPoP)

PWMA deployments SHOULD sender-constrain capabilities using **DPoP** and the `cnf.jkt` confirmation method.

If a PWMA-CAP is presented as a bearer token without sender constraint, the RP MUST treat it as high-risk and SHOULD require step-up or additional proof.

### 11.4 Capability enforcement (RP requirements)

An RP that accepts PWMA capabilities MUST:

1) Verify the token signature and the issuer trust (Section 14).  
2) Enforce `aud` matching (token audience includes the RP).  
3) Enforce `iat/exp` and bounded clock skew.  
4) Enforce sender constraint (`cnf`) when present.  
5) Recompute the `action_hash` over the action instance and compare.  
6) Enforce envelope constraints (profile-defined mapping).  
7) Enforce single-use semantics for `jti` (replay cache).

## 12. PWMA Metadata and discovery: PWMA-CONFIG v0.2 (NEW in v0.2)

To enable relying parties and clients to verify PWMA-issued artifacts, each PWMA deployment MUST publish a discovery document and signing keys.

### 12.1 Well-known endpoint (MUST)

A conforming PWMA deployment with issuer identifier `https://pwma.example` MUST serve:

- `GET https://pwma.example/.well-known/pwma-configuration`

Deployments MAY also serve an alias endpoint:

- `GET https://pwma.example/.well-known/pwma`

If the alias is served, its JSON response MUST be identical (after canonical JSON serialization) to the `pwma-configuration` response.

The response body MUST be a PWMA-CONFIG JSON object (schema: `schemas/pwma-metadata.v0.2.schema.json`) and MUST include:

- `issuer` (string, URI) — the issuer identifier used in `iss` claims
- `jwks_uri` (string, URI) — location of a JSON Web Key Set used to verify PWMA-signed JWTs
- `pwma_versions_supported` (array)
- `intent_versions_supported` (array)
- `profiles_supported` (array)
- `vault_profiles_supported` (array)
- `formats_supported` (array)
- `mcp.tool_namespace` (string)

### 12.2 JWKS endpoint (MUST)

The `jwks_uri` MUST serve a valid JWKS (RFC 7517). Keys MUST be rotated safely:

- during rotation, old keys SHOULD remain published until all tokens signed with them are expired
- RPs SHOULD cache JWKS with respect to HTTP caching headers, but MUST respect key rotation

### 12.3 Optional: status / revocation (MAY)

PWMA deployments MAY publish a status endpoint (e.g., a Status List) for longer-lived mandates.

Because PWMA-CAP tokens are short-lived and single-use, deployments MAY choose to rely on:

- short expirations,
- replay prevention,
- key rotation,

instead of active revocation.

If a status endpoint is published, it MUST be advertised in PWMA-CONFIG.

### 12.4 Optional: MCP metadata tool (MAY)

A PWMA MCP server MAY implement an MCP tool `aaif.pwma.metadata` which returns the same PWMA-CONFIG object as `/.well-known/pwma-configuration`. If implemented, the response MUST be byte-for-byte identical after canonical JSON serialization.

## 13. Protocol flows

### 13.1 Agent registration (recommended)

1) Host calls PWMA MCP `aaif.pwma.request` with profile `aaif.pwma.agent.register/v0.2`.
2) PWMA either:
   - registers the agent operational key (KCS-backed), or
   - elicits step-up if registration policy requires it.
3) PWMA returns an agent registration receipt.

### 13.2 Mandate issuance (HAPP-gated when required)

1) Host submits a mandate intent.
2) PWMA evaluates policy:
   - if low-risk and already covered, it may mint a mandate immediately.
   - else it returns an elicitation requiring step-up (HAPP).
3) Once step-up evidence is verified, PWMA mints a PWMA-MANDATE.

### 13.3 OpenID4VCI bridge (credential issuance)

1) Host submits an OID4VCI offer or initiation URL to PWMA.
2) PWMA orchestrates issuance, possibly eliciting user login.
3) Resulting credentials are stored in the Cold Vault (or Hot Wallet if policy permits).

### 13.4 OpenID4VP bridge (credential presentation)

1) Host submits an OID4VP request (URL or object).
2) PWMA selects minimal credentials, applies policy, possibly requires step-up.
3) PWMA returns either:
   - a VP response (return mode) as an artifact, or
   - a receipt indicating direct_post completed.

### 13.5 HAPP step-up orchestration (external dependency)

If step-up is required, PWMA:

1) converts PWMA-INTENT + envelope into a HAPP AI-INTENT payload, including `intent_hash`.
2) requests a HAPP-CC from a Presence Provider profile (EU wallet, liveness, hardware key).
3) verifies the returned HAPP-CC and binds it to mandate/capability issuance.

### 13.6 Capability minting (autonomy within bounds)

1) Host submits a capability intent including an action instance reference or payload (profile-defined).
2) PWMA checks:
   - action is in scope,
   - action satisfies envelope constraints,
   - counters / max_uses / totals,
   - freshness and replay safety.
3) PWMA returns a PWMA-CAP token (sender-constrained if supported).

## 14. Verification requirements (Relying Parties)

### 14.1 Issuer discovery (MUST)

An RP that verifies PWMA-signed JWT artifacts MUST:

1) read `iss` from the token,
2) fetch and validate `/.well-known/pwma-configuration` for that issuer,
3) fetch `jwks_uri`,
4) verify signatures against that JWKS.

### 14.2 Audience and time checks (MUST)

RPs MUST enforce `aud`, `iat`, `exp` with bounded skew.

### 14.3 Action binding checks (MUST for capabilities)

RPs MUST reconstruct the action instance and verify `action_hash`.

For ACP commerce, use Annex E.2 mapping.

### 14.4 Envelope enforcement (MUST)

RPs MUST enforce envelope constraints at execution time.

If an RP cannot enforce envelope constraints, it MUST NOT accept PWMA capabilities for that operation.

## 15. Security considerations

- **No bearer-only high-impact capabilities.** Strongly prefer sender constraint (DPoP) for capabilities.
- **Replay resistance.** RPs MUST enforce `jti` single-use for capabilities; PWMA MUST keep issuance logs for dispute resolution.
- **Least privilege.** Mandates SHOULD be scoped, audience-restricted, and short-lived; envelopes SHOULD be narrow.
- **Key compromise.** Agent operational keys MUST be isolated from human keys; use KCS/TEE where possible.
- **Phishing resistance.** Any step-up UI MUST be on the provider domain (URL-mode or QR-mode).
- **Extension safety.** Unknown envelope extensions are equality-locked in subdelegation to avoid relaxation.

## 16. Privacy considerations

- Prefer **selective disclosure** credentials and minimal presentation sets.
- Envelopes SHOULD avoid embedding full PII (e.g., shipping address); use hashes when practical.
- PWMA SHOULD store sensitive audit data in encrypted vaults with minimal retention.

## 17. Conformance

Conformance requirements are defined in `pwma-conformance-v0.2.0.md`.

## 18. Compatibility notes (Annex)

This annex is informative (non-normative). It describes how PWMA composes with adjacent standards and ecosystems.

### A. wwWallet-inspired encrypted cloud wallets

wwWallet demonstrates a web-based wallet architecture where wallet contents remain end-to-end encrypted and cryptographic operations are performed on the client side using WebAuthn-based mechanisms, including PRF-derived keys for decrypting cloud-synced encrypted wallet contents.

PWMA adopts the same *architectural principle* (untrusted storage for encrypted containers), but adds:

- a **policy engine** for delegated authority,
- a **mandate → capability** model for autonomy,
- an MCP tool surface for agent integration,
- optional KCS/TEE-backed operational keys for agent PoP.

PWMA does not require WebAuthn PRF; it is one optional vault profile.

### B. OpenID4VCI and OpenID4VP

PWMA does not replace issuance/presentation protocols.
Instead it standardizes:

- how an agent host requests issuance/presentation operations from a wallet governor (PWMA),
- how those are executed via OpenID4VCI/OpenID4VP in a policy-aware way,
- how autonomy is expressed via mandates and capabilities.

### C. HAPP (step-up approvals)

PWMA treats step-up as an external proof layer.
When required by policy, PWMA requests HAPP-CC from certified Presence Providers and binds approvals to deterministic intent hashes (via HAPP).

This allows:

- EU-wallet based approvals,
- liveness-based approvals (e.g., iProov),
- hardware-backed approvals (e.g., YubiKey profiles),
- enterprise second factors (via appropriate PP profiles).

### D. ACK-ID (Agent Commerce Kit) compatibility

ACK-ID focuses on verifiable agent identity and ownership chains using DIDs and VCs (e.g., Controller credentials).
PWMA is complementary: it focuses on policy-limited delegated authority, wallet custody, and step-up governance.

Compatibility approach:

- Use ACK-ID identity credentials as *agent identity credentials* stored in the PWMA vault.
- Use PWMA mandates/capabilities for *action authorization*, optionally referencing the ACK-ID identity chain for accountability.

### E. OpenAI Agentic Commerce Protocol (ACP) compatibility (EXPANDED)

ACP defines commerce flows (checkout sessions, delegated payments) between agent experiences and merchants/PSPs.

PWMA can act as:

- a policy governor determining whether a purchase is allowed under a spending envelope,
- a source of verifiable proofs (OID4VP) if merchants require additional claims (age, residency, etc.),
- an audit and receipt store for purchase evidence.

#### E.1 Practical mapping (informative)

- A PWMA purchase mandate corresponds to a reusable spending envelope (e.g., “≤ $1000 per purchase, ≤ $3000 total, only these merchants”).
- For each checkout, PWMA mints a per-action capability token whose `action_hash` binds to the exact checkout session and final totals.
- The merchant/PSP still uses ACP delegated payment constraints; PWMA’s capability is *authorization evidence* and *policy enforcement*, not a replacement for payment rails.

#### E.2 Normative action instance mapping: ACP checkout completion profile (v0.1)

PWMA defines one normative action-instance profile for ACP compatibility:

- Profile: `aaif.pwma.action.acp.checkout_complete/v0.1`
- Schema: `schemas/pwma-action-instance-acp-checkout.v0.1.schema.json`

This section specifies a **field-by-field, deterministic mapping** from ACP objects to the PWMA action instance used for `action_hash`.

##### E.2.1 Inputs

Let:

- `CS` be the final ACP **Checkout Session** object.
- `AL` be the ACP **Delegated Payment Allowance** object, if delegated payment is used for this checkout.

##### E.2.2 Output: canonical PWMA action instance

The resulting action instance MUST be:

```json
{
  "version": "0.2",
  "type": "acp.checkout.complete",
  "acp": {
    "checkout_session_id": "...",
    "merchant_id": "... (optional)",
    "payment_provider": "...",
    "currency": "usd",
    "total_amount_minor": 0,
    "line_items": [{"item_id":"...","quantity":1}],
    "fulfillment": {
      "fulfillment_option_id": "... (optional)",
      "address_hash": "sha256:... (optional)",
      "country": "US (optional)",
      "postal_code": "94131 (optional)"
    },
    "delegated_payment_allowance": { "... optional ..." }
  }
}
```

All amounts in this profile MUST be **integer minor units**, and `currency` MUST be lowercase ISO‑4217.

##### E.2.3 Mapping rules (REQUIRED)

**A) Checkout binding**

- `acp.checkout_session_id = CS.id` (REQUIRED)

**B) Payment provider**

- `acp.payment_provider = CS.payment_provider.provider` (REQUIRED)

**C) Currency normalization**

- `acp.currency = lowercase(CS.currency)` (REQUIRED)

**D) Total amount**

- Find `T = the unique element in CS.totals[] where T.type == "total"`.
  - If no such element exists, mapping MUST fail.
  - If multiple such elements exist, mapping MUST fail.
- `acp.total_amount_minor = T.amount` (REQUIRED)

**E) Line items**

For each element `LI` in `CS.line_items[]`:

- `item_id = LI.item.id`
- `quantity = LI.item.quantity`

Construct:

- `acp.line_items = [ { "item_id": item_id, "quantity": quantity }, ... ]`

To ensure order-independence, `acp.line_items` MUST be sorted by:

1) `item_id` ascending (lexicographic), then  
2) `quantity` ascending.

**F) Merchant identifier**

If `AL` is present:

- `acp.merchant_id = AL.merchant_id` (REQUIRED)

Else if `CS.payment_provider.merchant_id` is present (some ACP surfaces provide it):

- `acp.merchant_id = CS.payment_provider.merchant_id` (OPTIONAL)

Else:

- omit `acp.merchant_id`

**G) Fulfillment option**

If `CS.fulfillment_option_id` is present:

- `acp.fulfillment.fulfillment_option_id = CS.fulfillment_option_id`

**H) Fulfillment address hash (privacy-preserving binding)**

If `CS.fulfillment_address` is present, construct an **address subset** object using these fields if present:

- `name`
- `line_one`
- `line_two`
- `city`
- `state`
- `country`
- `postal_code`

Compute:

- `address_hash = sha256:jcs(address_subset)` (using Section 7.3)

Then set:

- `acp.fulfillment.address_hash = address_hash`

For convenience and policy checks, the mapping MAY also copy:

- `acp.fulfillment.country = CS.fulfillment_address.country` (if present)
- `acp.fulfillment.postal_code = CS.fulfillment_address.postal_code` (if present)

**I) Delegated payment allowance (optional but recommended when present)**

If `AL` is present, include:

```json
"delegated_payment_allowance": {
  "reason": AL.reason,
  "max_amount_minor": AL.max_amount,
  "currency": lowercase(AL.currency),
  "checkout_session_id": AL.checkout_session_id,
  "merchant_id": AL.merchant_id,
  "expires_at": AL.expires_at
}
```

If `AL` is present, implementations SHOULD additionally validate:

- `AL.checkout_session_id == CS.id`
- `lowercase(AL.currency) == lowercase(CS.currency)`
- `AL.max_amount >= acp.total_amount_minor`

##### E.2.4 `action_hash` computation (REQUIRED)

Once the action instance is constructed, compute:

- `action_hash = sha256:jcs(action_instance)` (Section 7.3)

That `action_hash` MUST be the value placed in the PWMA capability token.

##### E.2.5 Interop guidance (informative)

- RPs SHOULD compute the mapping from their own ACP objects, not from agent-provided copies.
- Including the address hash binds authorization to a destination without disclosing the address in the capability token.
- Sorting `line_items` eliminates ordering-related hash mismatches.


## 19. References (informative)

- RFC 8785 — JSON Canonicalization Scheme (JCS)
- RFC 8725 — JSON Web Token Best Current Practices
- RFC 9449 — OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)
- RFC 7517 — JSON Web Key (JWK)
- OpenID4VCI — OpenID for Verifiable Credential Issuance 1.0
- OpenID4VP — OpenID for Verifiable Presentations 1.0
- wwWallet project (github)
- HAPP (AAIF draft)
