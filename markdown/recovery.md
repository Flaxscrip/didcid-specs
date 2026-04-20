## DID Recovery

For security reasons, this method provides no support for storing private keys. We recommend that clients use BIP-39 to generate a master seed phrase consisting of at least 12 words, and that users safely store the recovery phrase.

If a user loses a device that contains their wallet, they should be able to install the wallet software on a new device, initialize it with their seed phrase, and recover their DID along with all their credentials. This implies that a [[def: vault, An encrypted archive of credentials stored with the agent DID document, encrypted with the DID's current private key]] of the credentials should be stored with the agent DID document, though it should be encrypted with the DID's current private key for privacy.

### Recovery Process

```mermaid
graph LR
    A["Lost device"] --> B["Install wallet"]
    B --> C["Enter BIP-39 seed"]
    C --> D["Derive HD keys"]
    D --> E["Recover DID"]
    E --> F["Resolve DID doc"]
    F --> G["Key rotation update"]
    G --> H["DID recovered"]
```

::: warning
The vault pattern requires the user to retain access to their seed phrase. If both the device and the seed phrase are lost simultaneously, DID recovery is not possible — this is an intentional security property, not a bug.
:::

### Security Considerations for Recovery

1. **Seed phrase secrecy** — The BIP-39 seed phrase is the single point of failure for key recovery. It must never be stored digitally in plaintext.
2. **Vault encryption** — The credential vault is encrypted with the DID's current key. After key rotation, the vault must be re-encrypted with the new key.
3. **No controller delegation** — Because an asset DID is controlled by exactly one agent, there is no multi-sig recovery path. The agent's seed phrase is the sole recovery mechanism.