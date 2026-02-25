# PWMA standard-ready bundle v0.2.0

This bundle contains a standard-ready draft of **PWMA (Personal Wallet Management Agent Protocol)**.

## Contents

- `specification/draft/pwma-v0.2.0.md` — Core protocol draft
- `specification/draft/mcp-profile-v0.2.0.md` — MCP tool profile for `aaif.pwma.*`
- `specification/draft/conformance-v0.2.0.md` — Conformance requirements
- `schemas/` — JSON Schemas (Draft 2020-12)
- `test_vectors/v0.2/` — Hashing + monotonicity + ACP mapping vectors

## Notes

- This is a draft intended for review and iteration.
- The envelope monotonicity algorithm is designed to be **fully testable** with included vectors.
- The ACP action-instance mapping is provided for interoperability with agentic commerce checkouts.

