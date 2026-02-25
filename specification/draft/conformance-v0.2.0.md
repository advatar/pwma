# PWMA Conformance Requirements v0.2.0 (Draft)

**Status:** Draft  
**Version:** 0.2.0  
**Track:** AAIF Standards / MCP Profile  

This document defines conformance requirements for:

- PWMA servers (wallet governors),
- PWMA clients/hosts (agent runtimes invoking MCP tools),
- Relying Parties (verifiers/executors of mandates/capabilities),
- Optional auxiliary components (KCS, Vault Store integrations).

Where practical, each requirement is paired with suggested test cases.

---

## 1. Conformance levels

- **PWMA-CORE** — implements the core protocol objects + MCP tool surface.
- **PWMA-CORE+DISCOVERY** — PWMA-CORE plus `.well-known` metadata and JWKS.
- **PWMA-CORE+AUTONOMY** — PWMA-CORE plus mandates/capabilities with envelope monotonicity and action hashing.
- **PWMA-CORE+OID4VC** — PWMA-CORE plus OID4VCI/OID4VP bridging.
- **PWMA-CORE+HAPP** — PWMA-CORE plus HAPP step-up integration.

A deployment MAY claim multiple conformance levels.

---

## 2. PWMA Server requirements

### 2.1 MCP tool surface

**PWMA-SRV-MCP-01 (MUST):** Expose `aaif.pwma.request`.

**PWMA-SRV-MCP-02 (SHOULD):** Expose `aaif.pwma.get`.

**PWMA-SRV-MCP-03 (MAY):** Expose `aaif.pwma.metadata`.

Suggested tests:
- tool appears in `tools/list`
- request/response schemas validate

### 2.2 Deterministic hashing

**PWMA-SRV-HASH-01 (MUST):** Implement RFC 8785 JCS for any hashed objects.

**PWMA-SRV-HASH-02 (SHOULD):** Reject hashed objects that contain floats or non-deterministic types.

Suggested tests:
- recompute hashes from published test vectors

### 2.3 Envelope handling

**PWMA-SRV-ENV-01 (MUST):** Accept and validate PWMA-ENVELOPE v0.2 objects.

**PWMA-SRV-ENV-02 (MUST):** Enforce envelope monotonicity when issuing child mandates and capabilities (Section 9.3 of the core spec).

**PWMA-SRV-ENV-03 (MUST):** Enforce envelope constraints at capability mint time (i.e., do not mint a capability for an action instance that violates the mandate envelope).

Suggested tests (using `/test_vectors/v0.2/pwma_envelope_monotonicity_vectors_01.json`):
- child.max < parent.max => PASS
- child drops a parent key => FAIL
- child merchant allow-list not subset => FAIL
- extension type differs => FAIL
- extension data differs => FAIL

### 2.4 Mandates and subdelegation

**PWMA-SRV-MAND-01 (MUST):** Support PWMA-MANDATE in `format: jwt`.

**PWMA-SRV-MAND-02 (MUST):** Enforce monotonicity on `scope`, `aud`, `exp`, and `envelope`.

Suggested tests:
- parent scope: ["commerce.purchase"] child scope: ["commerce.purchase"] => PASS
- parent exp earlier than child exp => FAIL
- envelope monotonicity failure => FAIL

### 2.5 Capabilities

**PWMA-SRV-CAP-01 (MUST):** Support PWMA-CAP in `format: jwt`.

**PWMA-SRV-CAP-02 (MUST):** Set short `exp` (deployment policy, RECOMMENDED minutes).

**PWMA-SRV-CAP-03 (MUST):** Include `action_hash` binding.

**PWMA-SRV-CAP-04 (SHOULD):** Sender constrain capabilities (DPoP via `cnf.jkt`).

Suggested tests:
- action_hash matches computed hash in vectors
- DPoP proof validation (if enabled)

### 2.6 Metadata and discovery (NEW)

**PWMA-SRV-META-01 (MUST):** Serve a PWMA-CONFIG document at `/.well-known/pwma-configuration`.

**PWMA-SRV-META-01B (MAY):** If an alias endpoint `/.well-known/pwma` is served, its response MUST be identical to `/.well-known/pwma-configuration`.

**PWMA-SRV-META-02 (MUST):** PWMA-CONFIG MUST validate against `schemas/pwma-metadata.v0.2.schema.json`.

**PWMA-SRV-META-03 (MUST):** `issuer` in PWMA-CONFIG MUST match `iss` in issued JWT artifacts.

**PWMA-SRV-META-04 (MUST):** JWKS served from `jwks_uri` MUST contain the public keys required to validate issued JWT artifacts.

Suggested tests:
- fetch `/.well-known/pwma-configuration`
- fetch `jwks_uri`
- validate a sample mandate/capability signed with the published key

### 2.7 OID4VC bridges (optional)

**PWMA-SRV-OID-01 (MAY):** Support OID4VCI flows (offer + issuance).

**PWMA-SRV-OID-02 (MAY):** Support OID4VP flows (return and/or direct_post).

If implemented, servers MUST:
- isolate issuer/verifier interactions to provider domains (via URL-mode elicitations)
- respect least disclosure by default

### 2.8 HAPP integration (optional)

**PWMA-SRV-HAPP-01 (MAY):** Support step-up via HAPP-CC.

If implemented, servers MUST:
- bind HAPP-CC to `intent_hash` and envelope view presented to the user
- validate HAPP-CC freshness per policy

---

## 3. MCP Host/Client requirements

**PWMA-CLI-01 (MUST):** Support QR-mode elicitation display.

**PWMA-CLI-02 (MUST):** Support URL-mode navigation with explicit user confirmation.

**PWMA-CLI-03 (SHOULD):** Retry idempotently with stable `requestId` after elicitation completes.

---

## 4. Relying party (RP) requirements

### 4.1 Token verification

**PWMA-RP-VER-01 (MUST):** Verify signatures using discovery (`/.well-known/pwma-configuration` → `jwks_uri`).

**PWMA-RP-VER-02 (MUST):** Enforce `aud`, `iat`, `exp` and bounded skew.

### 4.2 Replay resistance

**PWMA-RP-REPLAY-01 (MUST):** Enforce single-use semantics for capability `jti`.

### 4.3 Action binding and envelope enforcement

**PWMA-RP-ACT-01 (MUST):** Recompute `action_hash` over the action instance and compare.

**PWMA-RP-ENV-01 (MUST):** Enforce envelope constraints at execution time.

If the RP cannot enforce envelope constraints, it MUST NOT accept PWMA capabilities for that operation.

### 4.4 ACP purchase canonicalization (optional but recommended)

**PWMA-RP-ACP-01 (SHOULD):** For ACP purchase flows, construct action instances per the normative mapping (Annex E.2 of the core spec) and validate `action_hash` accordingly.

Suggested tests:
- validate `/test_vectors/v0.2/pwma_acp_action_hash_vector_01.json`

---

## 5. Test vectors

This bundle includes:

- `pwma_intent_hash_vector_01.json`
- `pwma_capability_action_hash_vector_01.json`
- `pwma_acp_action_hash_vector_01.json`
- `pwma_envelope_monotonicity_vectors_01.json`

Implementations SHOULD use these for regression and interop testing.
