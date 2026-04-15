# Verifiable Credentials

Archon implements the W3C Verifiable Credentials (VC) standard to enable trust without central authority.

## The Flow of Trust

1. **Issuance:** An issuer signs a claim about a subject and anchors the proof.
2. **Holding:** The subject stores the VC in their Archon wallet.
3. **Presentation:** The subject presents the VC to a verifier.

[[def: Verifiable Credential]]:
~ A digitally signed statement that can be cryptographically verified to be authentic and untampered.

::: warning
Credentials should never be stored in plain text; they must be encrypted to the subject's DID to ensure privacy.
:::

### Example: Clinical Credentialing
In a virtual nursing scenario, a nurse presents a VC issued by a State Board. The verifier checks the signature against the Board's `did:cid` anchor.
