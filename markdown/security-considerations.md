## Security Considerations

This section describes the security properties of the `did:cid` method and addresses the security requirements of [[ref: DID-CORE]] Section 10.

### Threat Model

The `did:cid` method is designed to resist the following classes of adversary:

- **Passive observers** — parties who can read all IPFS data, registry contents, and network traffic but cannot forge cryptographic signatures
- **Active network adversaries** — parties who can intercept and modify messages between a client and a node
- **Malicious or compromised nodes** — nodes that may serve stale, incorrect, or selectively withheld DID data
- **Minority registry attackers** — a minority coalition of registry validators acting in bad faith

The method does **not** claim security against:

- An adversary who obtains the controller's private key
- An adversary who controls a majority of the DID's specified [[ref: registry]] (e.g., a blockchain majority attack)
- A global IPFS outage in which no node retains a copy of the creation operation

---

### Self-Certifying Identifiers

[[def: self-certifying identifier, A DID whose suffix is derived deterministically from its own creation data, making the identifier itself cryptographic proof of the initial state]]

The DID suffix of a `did:cid` DID is the [[ref: CID]] (Content Identifier) of the JSON-canonicalized creation operation. This makes every `did:cid` DID a [[ref: self-certifying identifier]]: a resolver can independently verify that a given creation operation corresponds to a claimed DID by computing its CID and comparing it to the DID suffix. No trusted third party is required to validate the binding between the DID and its initial public key.

Any modification to the creation operation — including the public key, timestamp, or registration metadata — produces a different CID, and therefore a different DID. This property prevents silent substitution of the creation anchor.

---

### Operation Chain Integrity

[[def: operation chain, The hash-linked sequence of create, update, and delete operations associated with a DID, where each operation after creation references the CID of its predecessor via the `previd` field]]

All update and delete operations include a `previd` field containing the [[ref: CID]] of the previous operation in the sequence. This forms an [[ref: operation chain]] — a tamper-evident hash chain anchored at the [[ref: self-certifying identifier]]. An adversary who does not control the signing key cannot:

- **Reorder** operations without invalidating the `previd` chain
- **Insert** a forged operation without producing a valid signature and a known `previd`
- **Replay** a prior update operation, as the current `previd` will have advanced beyond the replayed operation's target

Node implementations MUST reject any update or delete operation whose `previd` does not exactly match the CID of the most recently accepted operation for that DID.

---

### Registry Security and Finality

The integrity of DID update history depends on the [[ref: registry]] specified in the creation operation. Different registries provide different finality and Byzantine fault-tolerance guarantees:

| Registry | Finality Model | Considerations |
|----------|----------------|----------------|
| **Bitcoin mainnet** | Probabilistic; ~6 confirmations (~60 min) | Highest economic security; reorganization risk decreases exponentially with block depth |
| **Bitcoin Signet / Testnet** | Same model; lower economic stake | Suitable for development and testing; not appropriate for production identity |
| **Hyperswarm** | P2P DHT-based ordering | Faster settlement; weaker Byzantine fault tolerance; suitable for lower-stakes updates |

The `registry` field in the creation operation is **immutable** — once a DID is created with a specified registry, that registry cannot be changed without creating an entirely new DID. This prevents registry-switching attacks where an adversary attempts to redirect update resolution to a weaker registry after DID establishment.

::: note
Node operators SHOULD document the registries they support and their trusted peer node policies. Resolvers that do not support a DID's specified registry MUST forward the resolution request to a trusted node rather than returning a partial or stale result.
:::

---

### Cryptographic Operations

All operations MUST apply the JSON Canonicalization Scheme (JCS) to the operation object before computing its CID and before signing. Failure to apply canonicalization consistently may cause the same logical operation to produce different CIDs depending on JSON serialization order, leading to resolution failures or operation rejection.

The current specification requires `EcdsaSecp256k1Signature2019` for all proofs. Implementing nodes MUST:

1. Verify the proof signature is cryptographically valid before accepting any create, update, or delete operation.
2. Verify the signing key was the active controller key at the time the operation was submitted.
3. Reject operations with unknown or unsupported `proof.type` values.

---

### Key Management

The `did:cid` method makes the following key management design decisions:

- **Client-side signing only** — Private keys are never transmitted to nodes. Clients sign operations locally and submit only the signed operation and the corresponding public key.
- **HD key derivation** — Keys are derived using BIP-32 hierarchical deterministic derivation from a BIP-39 seed phrase. Implementations SHOULD use hardened derivation paths to prevent child key exposure from compromising the parent key.
- **No server-side key custody** — Nodes have no access to private keys and cannot sign on behalf of DID controllers.

After a key rotation (via an update operation), credentials signed with the previous key remain verifiable through [[ref: temporal resolution]] — the historical key state is preserved and resolvable at any prior version time.

Clients SHOULD:

- Encrypt private key material at rest using a passphrase-derived key.
- Implement key rotation promptly when a key may have been exposed.
- Store the BIP-39 seed phrase in a physically secure, offline location.

---

### Proof Verification Requirements

The `did:cid` method **requires** the `proof.created` field in all signed objects. While the W3C Data Integrity specification treats `proof.created` as optional, `did:cid` mandates it because [[ref: temporal resolution]] requires a creation timestamp to determine which historical key state to use for verification.

Verifiers MUST resolve the signer's DID **at the time the proof was created** (`versionTime = proof.created`) rather than at the current time. Resolving at the current time after a key rotation may produce a different active key, causing valid historical proofs to fail verification.

---

### Revocation Finality

DID revocation (via a `delete` operation) is **permanent and irreversible**. After a revocation is confirmed on the DID's [[ref: registry]]:

- The DID resolves with `didDocumentMetadata.deactivated: true`.
- `didDocument` and `didDocumentData` are returned as empty objects.
- No further update or delete operations are accepted for that DID.
- Possession of the original BIP-39 seed phrase does not enable recovery.

This finality is an intentional security property. It prevents scenarios where an attacker who later recovers an old key attempts to "un-revoke" a DID and take control of its associated credentials and assets.

---

### Node Trust Model

DID resolution may involve forwarding requests to trusted peer nodes when a resolver does not directly support the DID's registry. The following risks apply to this trust model:

- **Stale data**: A peer node that lags behind the registry may return outdated DID documents, causing verifiers to use superseded keys.
- **Malicious forwarding**: A compromised node may return incorrect or fabricated DID data to the requesting client.
- **Availability dependency**: If no reachable node supports a given registry, resolution fails for DIDs using that registry.

Mitigations:

- Resolvers in high-security contexts SHOULD run their own node directly connected to the DID's specified registry.
- Clients SHOULD validate that the returned document's `versionId` is consistent with the expected [[ref: operation chain]].
- Node operators SHOULD monitor registry sync status and alert on significant lag.

---

### Availability and Denial of Service

**IPFS availability**: DID resolution for a newly created DID requires that its creation operation be retrievable from IPFS. Node operators MUST pin creation operations for all DIDs they are responsible for. Operators SHOULD also arrange for redundant pinning (e.g., via Filecoin or a pinning service) to protect against single-node failure.

**Registry unavailability**: Temporary registry unavailability causes resolution to return the most recently known state rather than failing. This is a graceful degradation, not a hard failure, and is consistent with the method's design.

**Creation spam**: DID creation requires only an IPFS pin and a valid signature; there is no on-chain transaction required at creation time. Node operators SHOULD implement rate limiting on creation endpoints to prevent resource exhaustion from spam creation.

**Update queue costs**: For registries with non-trivial transaction costs (e.g., Bitcoin mainnet), nodes may batch update operations. Operators SHOULD implement queue management policies that prevent unbounded accumulation of pending updates.

---

### Cryptographic Algorithm Agility

The `proof.type` field in all operations specifies the cryptographic algorithm used. The current method version defines `EcdsaSecp256k1Signature2019`. Future versions of the method specification may introduce additional proof types (e.g., Ed25519Signature2020, BLS12-381 for threshold schemes).

Existing DIDs using `EcdsaSecp256k1Signature2019` are unaffected by the introduction of new proof types. Nodes MUST continue to support all historically accepted proof types to preserve backward compatibility of [[ref: temporal resolution]] for existing DIDs.

---

### Residual Risks

The following risks remain after the mitigations described in this section:

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Private key compromise | Low (with secure local storage) | High — attacker can update or revoke the DID | Key rotation; monitor for unauthorized update operations |
| BIP-39 seed phrase exposure | Low (with physical security) | Critical — unrecoverable if device is also lost | Hardware wallet; physically separate offline backup |
| Blockchain reorganization | Very low (mainnet, ≥6 confirmations) | Medium — brief resolution inconsistency | Await sufficient confirmations before relying on an update |
| IPFS content unavailability | Low (with active pinning) | High — resolution failure for affected DIDs | Multi-provider pinning; redundant node infrastructure |
| Trusted node compromise | Low | Medium — stale or incorrect DID data returned | Multi-node resolution; independent registry verification |
