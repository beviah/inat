# Inat

`Inat` is an open protocol for bilateral value transfer achieving transaction finality through zero-knowledge proofs, threshold cryptography, and distributed witness attestation — without a shared ledger, central operator, or consensus mechanism. Each wallet holds a **slot** — a value container whose liveness is anchored in a single persistent counter stored in the wallet's own P2P document. To transfer value, sender and recipient each sign a commitment; a randomly-selected quorum of witnesses attests the transfer is valid; the nullifier is reconstructed via threshold cryptography once both sides have committed, making reversal mathematically impossible; and the entire history of the slot collapses into a constant-size proof that any party can verify in under a millisecond. No blockchain, no consensus, no central operator.

## Status

Implementation is in final stages. 

You can read high level [whitepaper](https://github.com/beviah/inat/blob/main/inat-whitepaper.md). 

You can read detailed specification [specification](https://github.com/beviah/inat/blob/main/inat_protocol.md). 

## License

Inat protocol whitepaper and specification:

Apache2.0

Inat implementation:

BSL with transition to Apache2.0 within 4 years of publication
