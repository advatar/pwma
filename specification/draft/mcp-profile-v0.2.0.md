# PWMA over MCP v0.2.0 (Profile)

**Status:** Draft  
**Version:** 0.2.0  
**Tool namespace:** `aaif.pwma.*`

## 1. Summary

This profile standardizes how a Personal Wallet Management Agent (PWMA) is exposed as an **MCP Server**.

It enables an MCP Host/Client (Task Agent runtime) to:

- request mandates and per-action capabilities,
- ingest credentials using OpenID4VCI offers,
- present credentials using OpenID4VP requests,
- trigger step-up approvals via HAPP (URL/QR-mode elicitations),
- retrieve stored artifacts and receipts by reference,
- optionally fetch PWMA discovery metadata (PWMA-CONFIG).

This profile uses:

- MCP `tools/list` for discovery,
- MCP `tools/call` for invocation,
- **URL-mode** and **QR-mode** elicitations for any sensitive user interactions.

## 2. Required capabilities

### 2.1 PWMA MCP Server

A conforming PWMA MCP server:

- MUST implement MCP tools.
- MUST expose tool `aaif.pwma.request`.
- SHOULD expose tool `aaif.pwma.get` to fetch artifacts by reference.
- MAY expose tool `aaif.pwma.metadata` to return PWMA-CONFIG (Section 12 of the core spec).

### 2.2 MCP Host/Client

A conforming MCP host/client:

- MUST support displaying QR code payloads provided by tool errors (QR-mode elicitation).
- MUST support safe navigation to provider/issuer-controlled UI (URL-mode elicitation), including:
  - clear domain display,
  - explicit user confirmation before navigation.

## 3. Tool: `aaif.pwma.request`

### 3.1 Purpose

Perform a wallet operation (mandates, capabilities, OpenID4VCI, OpenID4VP) and return verifiable artifacts and/or elicitations when interaction is required.

### 3.2 Input arguments (logical model)

`aaif.pwma.request` MUST accept exactly one of the following request shapes:

A) **Intent Mode** (agent-initiated)

- `walletIntent` (PWMA-INTENT; schema: `schemas/pwma-intent.v0.2.schema.json`)

B) **OpenID4VP Mode** (verifier request)

- `oid4vpRequest` (URL string or parsed object)
- optional `execution` hints:
  - `mode: "return" | "direct_post"`
  - when `direct_post`, include `response_uri` and any required metadata

C) **OpenID4VCI Mode** (issuer offer)

- `oid4vciOffer` (URL string or parsed offer object)

D) **HAPP forwarding** (rare; advanced)

- `happChallenge` (HAPP-CHAL object, if an RP provided one and the host wants PWMA to orchestrate)

All requests SHOULD include:

- `requestId` (string, stable across retries)
- `return` formatting preference:
  - `artifacts.inline` boolean (default false for large artifacts)
  - `formatPreference` array (e.g., `["jwt","vc+json"]`)

### 3.3 Output

If the operation can complete without user interaction, the tool MUST return:

- `structuredContent` as a **PWMA Result Envelope** (schema: `schemas/pwma-result-envelope.v0.1.schema.json`)

If user interaction is required, PWMA SHOULD return a JSON-RPC error:

- `code: -32042`
- `data.elicitations[]`

After the user completes the elicited step, the host SHOULD retry the tool call with the same `requestId`.

### 3.4 Elicitations

Elicitations MUST be represented as an array.

Each elicitation object MUST include:

- `elicitationId` (string)
- `mode` (`"url"` or `"qr"`)
- `message` (string)
- one of:
  - `url` (when mode `"url"`)
  - `qrPayload` (when mode `"qr"`)

Security note:

- URL-mode elicitations MUST be hosted on the issuer/provider domain (not embedded forms).
- QR payloads SHOULD be treated as opaque; hosts MUST render as QR without interpretation.

### 3.5 Idempotency (RECOMMENDED)

PWMA servers SHOULD be idempotent on `requestId`:

- repeating the same request after elicitation SHOULD produce the same result, or a stable reference to the same produced artifacts.

### 3.6 Errors

PWMA defines the following error codes:

- `-32040` — policy denied (PWMA-ERR-POLICY-DENY)
- `-32041` — malformed request / unsupported profile (PWMA-ERR-BAD-REQUEST)
- `-32042` — user interaction required (PWMA-ERR-ELICIT)
- `-32043` — external protocol error (issuer/verifier failure) (PWMA-ERR-UPSTREAM)
- `-32044` — vault locked / unlock required (PWMA-ERR-VAULT-LOCKED)

## 4. Tool: `aaif.pwma.get` (optional)

### 4.1 Purpose

Retrieve an artifact by reference when the artifact is too large to return inline (e.g., VP tokens, receipts, credential blobs).

### 4.2 Input arguments

- `ref` (string)

### 4.3 Output

- `structuredContent` containing the referenced artifact, in the same envelope format used for artifacts in PWMA Result Envelope.

## 5. Tool: `aaif.pwma.metadata` (optional)

### 5.1 Purpose

Return the PWMA discovery document (PWMA-CONFIG) to an MCP host without requiring an extra HTTP fetch.

### 5.2 Output

- `structuredContent` containing the PWMA-CONFIG object (schema: `schemas/pwma-metadata.v0.2.schema.json`)

If implemented, this tool’s result MUST be semantically identical to the JSON served at:

- `/.well-known/pwma-configuration`

## 6. Message flow examples (non-normative)

### 6.1 Mandate request requiring step-up

1) Host calls `aaif.pwma.request` with a mandate PWMA-INTENT → server returns `-32042` with QR elicitation.

2) User completes approval on a Presence Provider / EU Wallet.

3) Host retries `aaif.pwma.request` with same `requestId` → server returns `pwma.mandate` artifact + receipt.

### 6.2 OpenID4VP request fulfilled autonomously

1) Host calls `aaif.pwma.request` with `oid4vpRequest`.

2) Server returns:

- an artifact containing the authorization response (return mode), or
- a receipt indicating a successful `direct_post` execution.

## 7. Notes

- Sensitive interactions (issuer login, biometrics, enterprise authentication) MUST occur on issuer/provider domains via URL-mode.
- PWMA SHOULD minimize disclosures by default and enforce local policy regardless of host hints.
- PWMA SHOULD support OpenID4VP `direct_post` where feasible (cross-device flows).
