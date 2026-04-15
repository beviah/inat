# Inat Protocol Specification

LoudWhisper is an p2p framework implementing inat protocol

Double-spend prevention for peer-to-peer game economies — issuer-gated identity, decentralized execution. No blockchain, no consensus, no global ledger. Need-to-know state and content-addressed lookups.

Double-spend is deterministically impossible during live validation and probabilistically infeasible under bounded Sybil conditions due to VRF-selected threshold nullifier decryption over folded atomic state.

The protocol doesn't have a throughput problem. It has a demand problem. 

Fiat-backed issuance is structurally supported but out of scope: any licensed entity (EMI, MSB, or equivalent) may serve as issuer, performing KYC at mint and redeem while the protocol's ZK layer preserves in-flight privacy — compliance architecture, licensing, and gate-level controls are the issuer's responsibility, not a protocol concern.

Inat tokens are closed-loop instruments with no fiat redemption promise. If secondary markets emerge for fiat conversion, the exchange operators — not the issuer or the protocol — bear the regulatory burden. This follows the established pattern of virtual currency secondary markets (game gold, digital goods).


---

# Part I: Overview & Design Principles

# 1. Key Properties

| Property | Mechanism |
|----------|-----------|
| Issuer-attested | Genesis validity verified in fold circuit; O(1) regardless of depth |
| No double-spend | Sequence oracle advancement (authoritative liveness) + Iroh-blob nullifier (in-flight race detection) |
| GC-resistant | Sequence oracle (~200 bytes/identity) persists liveness; proof folding carries validated history; Iroh-blob nullifiers GC-safe after oracle advances |
| No sender griefing | Threshold encryption (can't reveal nullifier alone) |
| No receiver griefing | Bilateral signatures required |
| No witness griefing | VRF selection + credit requirements |
| Witness legitimacy | VRF proof + shard membership verified in fold circuit (§33 constraint 6a–6e) — wrong-shard or non-VRF witnesses cannot produce valid folds |
| Decentralized Validation | 21-of-29 random witness quorum |
| Grind-resistant | Per-phase witness rotation (recipient entropy) |
| Privacy | ZK proofs hide all transaction details from witnesses |
| Collusion-resistant | Beacon entropy in seed₂₃/seed₄ unpredictable even to colluding parties |
| Issuer-gated witness identity (not any wallet) | VRF eligibility requires issuer-signed eligible header + funded wallet (≥MIN_WITNESS_BALANCE). Headers delivered directly to eligible wallets (not published as blobs). Sybil resistance via administrative scarcity, not computational cost. |
| Fixed witness count | All transactions use WITNESS_TOTAL=29 witnesses, WITNESS_QUORUM=21 threshold. Shard sizing derived from these constants: SHARD_MIN_DENSITY = ceil(WITNESS_QUORUM × SHARD_TARGET_SIZE / WITNESS_TOTAL) = 750. |
| Temporal integrity | Beacon recency chain — consecutive fold proofs must advance at real-time rate |
| Symmetric participation | All funded, issuer-approved wallets are VRF-eligible — witnessing incentivized via fee shares |
| O(1) verification | Proof folding (~22KB, ~200ms). Bilateral signatures (σ_s, σ_r) verified natively from fold data — not inside ZK circuits (avoids ~400k non-native BN254↔Curve25519 constraints) |
| Per-phase seq stride | SEQ_STRIDE=4: one doc entry per phase transition per party. Output at base+4, forward at base+8. protocol_seq = Iroh entry seq (by construction). Entry model: one doc.set() per phase, supplementary data in blobs via CID. |
| Data availability | 7 Tit-for-Tat confirmations |
| No Global State | Need-to-know state distribution |
| Multi-Asset Support | Asset ID in slot structure |
| Key-rotation safe | Asset identity via asset_genesis_cid; issuer keys rotatable via AssetKeyRegistry without token migration |
| Domain | P2P game economies, virtual goods, network-native point systems |

## 2. Transaction Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│              TRANSACTION STATE MACHINE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SENDER SLOT                     RECIPIENT SLOT
  ──────────                      ──────────────
  READY (slot at seq=N)            READY (slot at seq=M)
      │                               │
      ▼ [Phase 1: sender writes       │
         entry at seq=N+1,            │
         W₁ attests + holds shares]   │
  PENDING_SEND (seq=N+1)             │
      │                               ▼ [Recipient validates π₁,
      │                          verifies amount, signs σ_r,
      │                          writes entry at seq=M+1]
      │                      PENDING_RECEIVE (seq=M+1)
      │                               │
      ▼ [sender entry N+2]            ▼ [recipient entry M+2,
                                          W₂₃ quorum = LOCKED]
  COMMITTED (seq=N+2) ◄────── COMMITTED (seq=M+2)
      │                               │
      ▼ [sender entry N+3:            │
         shares released,             ▼ [recipient entry M+3:
         nullifier published]            observed nullifier]
  SPENT (seq=N+3)              SPENT (seq=M+3)
      │                               │
      ▼ [sender entry N+4:           ▼ [recipient entry M+4:
         fold, output slot]             output slot materializes]
  FINALIZED (seq=N+4)          READY (new slot at seq=M+4)
  output slot at N+4           forward commit → M+8
  forward commit → N+8

  SEQ STRIDE:
  ─────────
  SEQ_STRIDE = 4. Each party writes 4 entries per transaction.
  Output slot at base + SEQ_STRIDE. Forward commit at base + 2*SEQ_STRIDE.
  Recovery from PENDING (base+1) uses 3 entries to reach base+4.
  Gaps from abort/skip are harmless — reuse is impossible.

  ENTRY MODEL:
  ────────────
  Each entry = one protocol_seq increment = one doc.set() call.
  Supplementary data (proofs, attestations, etc.) in blobs via CID.
  protocol_seq value == Iroh entry sequence number (by construction).
      
│                                                                 │
│  WITNESS SETS (different at each phase, sized by tx tier):      │
│  • W₁ (Phase 1): Holds encrypted shares, releases after W₂₃    │
│  • W₂₃ (Phase 2): Attests bilateral binding (uses σ_r entropy) │
│  • W₄ (Phase 4): DA confirmation (7 validators)                │
│                                                                 │
│  ISSUER-GATED ELIGIBILITY:                                      │
│  • All witnesses must hold issuer-signed eligibility cert       │
│  • All witnesses must hold ≥MIN_WITNESS_BALANCE of asset        │
│  • Eligibility verified in fold circuit via Merkle proof        │
└─────────────────────────────────────────────────────────────────┘
```

## 3. Design Principles

- **Symmetric Privacy**: Both sender and recipient have identical privacy guarantees via private-key-derived identifiers and session randomness.
- **Need-to-Know State**: No global transaction broadcast. State shared only with participants. Only nullifiers published globally.
- **Counterparty Verification**: Parties verify intended counterparty via ECDH-derived session-bound commitments. Witnesses verify pairing without learning identities.
- **Minimal Trust**: No single trusted party. Distributed 13-of-19 witness quorum via VRF random selection with economic incentives.
- **Identity vs Value Separation**: Identity layer (keys, sequences, ownership) is orthogonal to value layer (slots, balances, assets, lineage).
- **Gossip-ZK Parity**: Every gossip-level routing constraint (shard membership, witness selection) is also enforced at the ZK circuit level (§33 constraint 6). No security property relies solely on network-layer behavior.
- **Issuer-Gated, Decentralized-Executed**: Identity admission is permissioned (issuer gates VRF eligibility). Transaction execution is fully decentralized (p2p gossip, VRF witness selection, ZK proofs). The issuer is the bouncer, not the bartender. The issuer cannot retroactively censor finalized fold proofs. Asset identity is derived from `asset_genesis_cid` (immutable), not from the issuer's signing key — key rotation updates the AssetKeyRegistry without affecting token identity or requiring migration.
- **Symmetric Participation**: Every funded, issuer-approved wallet is eligible to witness. Witnessing is incentivized via fee shares.


- **`inat-protocol`**: §4–§6 (crypto), §11–§21 (identity/slots/pairing/threshold), §28 SELF_AUTHENTICATING, §31–§35 (circuits), §49.1–§49.3 (data structures)
- **`inat-network`**: §19 (witness selection/shards), §20 (phases), §21 (attestation), §24–§27 (protocols), §36–§41 (storage), §42–§44 (oracle), §47–§48 (economics/witness economics), §49 (data structures)


---

# Part II: Cryptographic Primitives

## 4. Hash Function

```python
# NOTATION: Throughout this spec, || denotes byte concatenation.
# In implementation, use: b"prefix" + value (Python) or 
# [prefix, value].concat() (Rust)

H: {0,1}* → {0,1}^256
Implementation: BLAKE3

# Domain separation
H_slot(x) = H(b"inat_slot:" || x)
H_null(x) = H(b"inat_nullifier:" || x)
H_pair(x) = H(b"inat_pairing:" || x)
H_comm(x) = H(b"inat_commitment:" || x)

# Additional domain-separated hashes (used in implementation)
H_slot_addr(x) = H(b"inat_slot_addr:" || x)     # §5.1 slot addressing
H_ptr(x)       = H(b"inat_pointer:" || x)        # Pointer name derivation
H_pk_addr(x)   = H(b"inat_pubkey_addr:" || x)    # Public key → short address
H_vss_root(x)  = H(b"inat_feldman_vss_root:" || x)  # VSS commitment root
H_key_derive(x)= H(b"inat_key_derivation:" || x) # Child key derivation
# All balance/amount commitments MUST use Pedersen (§6) for
# homomorphic properties and range proof compatibility.
```

## 4.1 Scalar Field Reduction

When reducing arbitrary data to a Ristretto scalar, SHA-512 is used
(not BLAKE3) to produce a 64-byte intermediate for uniform reduction:

    scalar = from_bytes_mod_order_wide(SHA-512(data))

This is required because uniform scalar sampling needs 2× the field
size as input (512 bits for a ~253-bit field). BLAKE3's 32-byte output
would produce biased scalars. This applies ONLY to scalar reduction — 
all other hash operations use BLAKE3 as specified in §4.

## 4.2 Authenticated Encryption (AEAD)

```python
# XChaCha20-Poly1305 (RFC 8439 extended-nonce variant)
# Consistent with NaCl/libsodium dependency.
# 24-byte nonce eliminates collision risk for random nonces.

def aead_encrypt(key: bytes, nonce: bytes, plaintext: bytes, aad: bytes) -> bytes:
    """XChaCha20-Poly1305 authenticated encryption.
    key: 32 bytes (from HKDF or ECDH derivation)
    nonce: 24 bytes (XChaCha20) or 12 bytes (truncated contexts)
    aad: additional authenticated data (bound but not encrypted)
    Returns: ciphertext || 16-byte Poly1305 tag
    """
    return xchacha20poly1305_encrypt(key, nonce, plaintext, aad)

def aead_decrypt(key: bytes, nonce: bytes, ciphertext: bytes, aad: bytes) -> bytes:
    """Raises AuthenticationError on tag mismatch."""
    return xchacha20poly1305_decrypt(key, nonce, ciphertext, aad)
```

AEAD contexts and their constructions:

| Context              | AEAD variant          | Nonce   |
|----------------------|-----------------------|---------|
| Share encryption     | ChaCha20-Poly1305     | 12B rng |
| Amount blinding      | XChaCha20-Poly1305    | 24B derived |
| Spot reveal          | XChaCha20-Poly1305    | 24B derived |
| Slot-at-rest         | XChaCha20-Poly1305    | 24B rng |

Rule: If key is ephemeral (single-use DH), ChaCha20 with 12-byte
random nonce is sufficient. If key may be reused or is long-lived,
XChaCha20 with 24-byte nonce is required.


Usage contexts:
- Share encryption (§10): ephemeral ECDH key + HKDF → XChaCha20-Poly1305
- Amount blinding encryption (§20.1 step 21): bilateral ECDH → XChaCha20-Poly1305
- Spot reveal (§18.5): bilateral ECDH + ephemeral → XChaCha20-Poly1305

All AEAD contexts derive keys via HKDF-SHA256 (§7) with
context-specific `info` strings. Nonces are either random
(24 bytes) or deterministically derived via `H(context)[:24]`
when replay is structurally impossible (e.g., single-use sessions).


## 4.3 Merkle Tree Construction

```python

EMPTY_ROOT = H(b"inat_merkle_empty:")

def merkle_root(leaves: List[bytes]) -> bytes:
    """
    Binary Merkle tree over BLAKE3.
    - Leaves hashed with MERKLE_LEAF_DOMAIN.
    - Internal nodes hashed with MERKLE_NODE_DOMAIN.
    - Canonical left-right ordering (smaller hash on left).
    - Odd layer count: promote last node to next layer.
    - Empty list: H(MERKLE_EMPTY_DOMAIN).
    """
    if not leaves:
        return H(b"inat_merkle_empty:")
    nodes = [merkle_leaf_hash(leaf) for leaf in leaves]
    while len(nodes) > 1:
        next_layer = []
        for i in range(0, len(nodes), 2):
            if i + 1 < len(nodes):
                next_layer.append(merkle_node_hash(nodes[i], nodes[i + 1]))
            else:
                next_layer.append(nodes[i])  # Promote unpaired
        nodes = next_layer
    return nodes[0]

def merkle_proof(leaves: List[bytes], index: int) -> List[Tuple[bytes, str]]:
    """Returns list of (sibling_hash, direction) pairs from leaf to root."""
    ...
```

All protocol Merkle trees (witness_root, slot_set_root,
attestation_cids_root, confirmation_root) MUST use this
construction. Implementations MUST NOT use raw `H(leaf)`
without the domain prefix.

## 5. Elliptic Curve

```
Group: Ristretto255 (prime-order group over Curve25519)
  - Eliminates cofactor-related attacks present in raw Ed25519/Curve25519
  - All point operations use the Ristretto encoding (32 bytes compressed)
  - Implemented via curve25519-dalek's RistrettoPoint

G: Ristretto basepoint (RISTRETTO_BASEPOINT_POINT)
Order: L = 2^252 + 27742317777372353535851937790883648493

Signing (§5.1): Ed25519 (for protocol signatures σ_s, σ_r, attestations)
  sk ← random(32 bytes)
  pk = Ed25519_derive_public(sk)
  σ = Ed25519_Sign(sk, message)
  valid = Ed25519_Verify(pk, message, σ)

Commitments & ZK (§5.2): Ristretto255 (for Pedersen, bulletproofs, Feldman VSS)
  scalar ← random mod L
  point = scalar × G
  All Pedersen commitments, range proofs, and VSS operate on Ristretto points.

NOTE: Ed25519 public keys are NOT Ristretto points. The two are used in
separate contexts and must not be mixed:
  - Ed25519: identity, signing, signature verification
  - Ristretto: commitments, proofs, threshold secret sharing

ECDH Key Agreement (§7) requires X25519 keys. Since identity keys
are Ed25519, conversion is required before any DH operation:

    x25519_sk = clamp(SHA-512(ed25519_seed)[:32])
    x25519_pk = edwards_to_montgomery(ed25519_pk)

Where:
    - clamp: RFC 7748 bit-clamping (clear bits 0,1,2,255; set bit 254)
    - edwards_to_montgomery: u = (1 + y) / (1 - y) mod p

All ECDH contexts (pairing §15, share encryption §10, spot reveal
§18) MUST apply this conversion. Passing raw Ed25519 bytes to
X25519 produces a valid but unrelated keypair.

```

## 6. Pedersen Commitment

```python
# Generators (fixed, deterministic):
g = RISTRETTO_BASEPOINT_POINT          # curve25519-dalek
h = PedersenGens::default().B_blinding  # bulletproofs crate

# IMPORTANT: We use the bulletproofs crate's default generators, NOT a
# custom HashToCurve derivation. This is required because range proofs
# are verified against these same generators — using different g/h for
# commitments vs proofs would break verification.

def pedersen_commit(value: int, blinding: bytes) -> Point:
    r = Scalar::from_bytes_mod_order(blinding)
    return value * g + r * h

# Properties: Hiding, Binding, Homomorphic: C(a,r1) + C(b,r2) = C(a+b, r1+r2)
# Range proof: Bulletproofs over Ristretto255, value in [0, 2^64)
```

## 7. ECDH Key Agreement

```python
def ecdh(my_ed_sk: bytes, their_ed_pk: bytes, context: bytes = b"") -> bytes:
        """X25519 key agreement from Ed25519 identity keys."""
        my_x_sk = ed25519_sk_to_x25519(my_ed_sk)
        their_x_pk = ed25519_pk_to_x25519(their_ed_pk)
        shared_point = X25519(my_x_sk, their_x_pk)
        return HKDF-SHA256(
            ikm=shared_point, salt=None,
            info=b"ecdh_shared:" || context, length=32
        )
```

## 8. Zero-Knowledge Proof System

```python
# PLONK on BN254
# Universal trusted setup: Powers-of-tau (Phase 1) only.
# No per-circuit Phase 2 ceremony. Deterministic from ptau + R1CS.
#
# Sub-proof layer: Sigma protocols (Schnorr proofs, Bulletproofs range
# proofs) operate as standalone sub-proofs for balance conservation
# and commitment opening within the protocol flow.

π = Prove(proving_key, public_inputs, private_witness)
valid = Verify(verification_key, public_inputs, π)
# Verification: O(1), ~5ms per proof
# Proof size: ~2.5KB (PLONK on BN254)
```


### 8.1 Trusted Setup (PLONK — Universal, No Ceremony)

PLONK uses a universal trusted setup. Only Phase 1 (powers-of-tau)
is required. There is NO per-circuit Phase 2 ceremony and NO
circuit-specific toxic waste.

**Phase 1 (powers-of-tau, universal, one-time):**
Downloaded from Hermez ceremony (54 contributors, audited).
Security: requires at least ONE honest contributor.

**SRS sizing:** 2^18 (262,144) constraints. Covers largest circuit
(SWEEP at ~200k constraints) with headroom.

**Key generation (deterministic, per circuit):**
Given Phase 1 ptau + circuit R1CS, PLONK setup produces proving
key and verification key deterministically. No MPC. No ceremony.
Adding new circuits requires only rerunning `snarkjs plonk setup`
against the existing ptau.

**Verification keys** are embedded in the protocol as constants.
**Proving keys** are distributed to provers.

**Circuit set:**

| Circuit | Constraints | Setup |
|---------|------------|-------|
| SENDER (π₁) | ~50k | PLONK from universal ptau |
| COMBINED (π₂₃) | ~65k | PLONK from universal ptau |
| SWEEP (π_sweep) | ~200k (variable) | PLONK from universal ptau |
| RECOVERY | ~120k | PLONK from universal ptau |
| CAPACITY | ~15k | PLONK from universal ptau |
| DONATION_SENDER (π₁_d) | ~51k | PLONK from universal ptau |
| DONATION_COMBINED (π₂₃_d) | ~66k | PLONK from universal ptau |
| FOLD | N/A (attested hash-chain, signature-based; +~21K constraints for registry proof in future SNARK variant) | No setup required |
| WITNESS_ELIGIBILITY | ~100k (Merkle proofs + issuer sig per witness) | PLONK from universal ptau |

**Implementation:** circom + snarkjs (PLONK backend). Native bridge
via rapidsnark for production proving performance.

---

## 9. Verifiable Random Function (VRF)

```python
def vrf_prove(sk: bytes, input: bytes) -> tuple[bytes, bytes]:
    output = H(sk × HashToCurve(input))
    proof = ...  # ECVRF proof
    return output, proof

def vrf_verify(pk: Point, input: bytes, output: bytes, proof: bytes) -> bool:
    return ECVRF_verify(pk, input, output, proof)
```

## 10. Threshold Cryptography (Feldman VSS)

```python
class FeldmanVSS:
    def split(self, secret: bytes, n: int = 29, m: int = 21
    ) -> Tuple[List[bytes], List[bytes]]:
        coefficients = [secret] + [random_scalar() for _ in range(m - 1)]
        shares = [evaluate_polynomial(coefficients, i) for i in range(1, n + 1)]
        commitments = [g ** coef for coef in coefficients]
        return shares, commitments

    def verify_share(self, share: bytes, index: int, commitments: List[bytes]) -> bool:
        lhs = g ** share
        rhs = product(C_j ** (index ** j) for j, C_j in enumerate(commitments))
        return lhs == rhs

    def reconstruct(self, shares: List[Tuple[int, bytes]], m: int = 21) -> bytes:
        assert len(shares) >= m
        return lagrange_interpolate(shares[:m], x=0)

    def encrypt_share_for_witness(self, share: bytes, witness_pk: bytes) -> bytes:
        """Ephemeral ECDH per-share encryption (§7 HKDF key extraction).
        
        Wire format: ephemeral_pk (32) || nonce (12) || AEAD ciphertext || tag (16)
        AEAD: ChaCha20-Poly1305 (IETF, 12-byte nonce)
        """
        ephemeral_sk = random_scalar()
        ephemeral_pk = derive_pk(ephemeral_sk)
        # HKDF extracts uniform key from DH output (§7)
        encryption_key = ecdh(ephemeral_sk, witness_pk, context=b"inat_share_encryption")
        nonce = random_bytes(12)
        ciphertext = aead_encrypt(key=encryption_key, nonce=nonce,
                                plaintext=share, aad=b"inat_share_encryption")
        return ephemeral_pk || nonce || ciphertext
```

Threshold Public Key (TPK):
    TPK is a binding identifier for the witness set, NOT an encryption key.
    Derived deterministically from sorted witness public keys:

    tpk = H(b"TPK:" || sort(witness_pk_1 || ... || witness_pk_N))

    Used for: session identification, commitment binding.
    NOT used for: encryption, signing, or DH.

VSS Commitment Root (for ZK proof public inputs):
    vss_root = H(b"inat_feldman_vss_root:" ||
                count.to_bytes(2, 'big') ||
                C_0 || C_1 || ... || C_{m-1})

This compact root lets ZK circuits reference the full
polynomial commitment set with a single 32-byte input.


| Property | Guarantee |
|----------|-----------|
| Sender can't reveal early | Needs 21 witness shares |
| Sender can't withhold | Shares already distributed |
| Witnesses can't forge | Feldman commitments verify |
| Partial reveal is useless | Need exactly 21 shares |
| Finalization is inevitable | Once W₂₃ quorum, W₁ releases |

---

# Part III: Identity & Slot Model

## 11. Identity

```python
def generate_identity() -> tuple[bytes, bytes]:
    sk = random_bytes(32)
    pk = derive_public_key(sk)  # Ed25519
    return sk, pk

pk_commit = H(pk)  # For sequence oracle
```

### 11.1 Mint Protocol (Genesis Slot Creation)

Creates the first slot for an identity under a given issuer.
This is the ONLY way value enters the system.

**Roles:**
- **Issuer**: Holds issuer_sk. Registered via out-of-band trust
  establishment. Signs the genesis record to attest value creation.
- **Recipient**: The identity receiving the minted slot.

**Issuer-side flow (`issue_genesis`):**

0. **Publish AssetDefinition (first mint for this asset only).**
   If no AssetDefinition exists for this asset:

       asset_def = AssetDefinition(
           issuer_pk   = issuer_pk,
           name        = name,
           symbol      = symbol,
           decimals    = decimals,
           max_supply  = max_supply,
           description = description,
           metadata_cid = metadata_cid,
       )
       asset_genesis_cid = await content_store.publish(asset_def.serialize())
       asset_id = H("inat_asset:" || asset_genesis_cid || name || params)

   Subsequent mints reuse the existing asset_genesis_cid and asset_id.

1. **Validate request.** Verify recipient_pk is a valid Ed25519
   public key. Verify amount > 0 and amount ≤ max_supply (if capped).
   Verify asset_id matches issuer's registered asset.

2. **Derive recipient slot identifiers at seq=0.**

       slot_commit = H("inat_slot:" || recipient_sk || (0).to_bytes(8))
       nullifier = H("inat_nullifier:" || recipient_sk || (0).to_bytes(8))
       nullifier_commit = H(nullifier)

   NOTE: The recipient provides slot_commit and nullifier_commit
   (derived from their sk). The issuer never learns recipient_sk.

3. **Compute forward commit.**

       forward_commit = H("inat_slot:" || recipient_sk || (1).to_bytes(8))

   Provided by recipient. Binds genesis slot to recipient's next seq.

4. **Build balance commitment.**

       balance_blinding = random_bytes(32)
       balance_commit = pedersen_commit(amount, balance_blinding)

   Blinding factor is shared with recipient via ECDH (§7).

5. **Build GenesisRecord.**

       genesis_record = GenesisRecord(
           slot_commit        = slot_commit,
           forward_commit     = forward_commit,
           balance_commit     = balance_commit,
           asset_id           = asset_id,
           asset_genesis_cid  = asset_genesis_cid,
           issuer_pk          = issuer_pk,
           recipient_pk       = recipient_pk,
           amount             = amount,           # Plaintext for issuer records
           timestamp          = current_timestamp(),
       )

6. **Issuer signs.**

       content_hash = H(genesis_record.serialize())
       issuer_sig = Ed25519_Sign(issuer_sk, content_hash)

7. **Recipient countersigns.**

       recipient_sig = Ed25519_Sign(recipient_sk, content_hash)

8. **Publish to content store.**

       genesis_cid = await content_store.publish(
           genesis_record.serialize() || issuer_sig || recipient_sig)

9. **Create SlotLineage.**

       lineage = SlotLineage(
           origin              = LineageOrigin.MINT,
           mint_proof          = genesis_cid,
           issuer_pk           = issuer_pk,
           issuer_attestation  = issuer_sig,
           asset_id            = asset_id,
           lineage_commit      = H(lineage.serialize()),
       )

9a. **Create initial AssetKeyRegistry (first mint for this asset only).**

        If this is the first genesis record for this asset_id
        (no prior AssetKeyRegistry exists for asset_genesis_cid):

            initial_registry = AssetKeyRegistry(
                asset_id           = asset_id,
                asset_genesis_cid  = asset_genesis_cid,
                authorized_keys    = [issuer_pk],
                registry_root      = merkle_root([issuer_pk]),
                epoch              = 0,
                previous_root      = merkle_root([issuer_pk]),
                signatures         = [(issuer_pk,
                    Ed25519_Sign(issuer_sk,
                        initial_registry._signing_payload()))],
            )
            registry_cid = await content_store.publish(
                initial_registry.serialize())

            # Announce on registry topic
            asset_prefix = asset_id[:4].hex()
            registry_topic = f"inat/registry/v1/{asset_prefix}".encode()
            await transport.broadcast(
                registry_topic,
                RegistryAnnouncement(
                    asset_id=asset_id, epoch=0,
                    registry_cid=registry_cid).serialize(),
                privacy=PROTOCOL_PRIVACY_CONFIGS["heartbeat"])

        Subsequent mints for the same asset reuse the existing
        registry. The issuer_pk used for signing MUST be in the
        current AssetKeyRegistry.


10. **Create OwnedSlot.**

        owned_slot = OwnedSlot(
            owner_sk          = recipient_sk,
            seq               = 0,
            balance            = amount,
            balance_blinding   = balance_blinding,
            nullifier          = nullifier,
            # Public fields:
            slot_commit        = slot_commit,
            nullifier_commit   = nullifier_commit,
            asset_id           = asset_id,
            balance_commit     = balance_commit,
            lineage            = lineage,
            status             = SlotStatus.READY,
            created_at         = current_timestamp(),
        )

10a. **Grant VRF eligibility (if balance ≥ MIN_WITNESS_BALANCE).**

        If the minted amount meets the witnessing threshold, the
        issuer includes the recipient's wallet_pk in the next
        epoch's IssuerEligibleHeader (§19.9) and delivers the
        signed header plus the recipient's Merkle proof directly
        to the recipient via the same channel as mint delivery.

        Headers are NOT published as Iroh blobs. Each eligible
        wallet receives its own (signed_header, merkle_proof)
        tuple from the issuer at epoch rollover.

        No separate eligibility request is needed for mint
        recipients — funded wallets are eligible by default.

**Post-conditions:**
- Recipient has a READY slot at seq=0 with minted balance.
- Genesis record is content-addressed and immutable.
- Issuer signature is verifiable by any party (fold circuit §33).
- Recipient receives signed IssuerEligibleHeader + Merkle proof directly (if balance ≥ MIN_WITNESS_BALANCE).

### 11.1.1 Supply Cap Enforcement (Optional)

If AssetDefinition.max_supply is set (non-None), the issuer MUST
generate a MintProof with every GenesisRecord. Recipients MUST
verify this proof before countersigning — an unverified or missing
proof is grounds for refusing the mint.

**IssuerMintAccumulator** (stored in issuer's own wallet document):

| Field | Type | Description |
|-------|------|-------------|
| cumulative_minted | uint64 | Total minted to date |
| mint_root | bytes(32) | Merkle root of all GenesisRecord CIDs |
| mint_count | uint64 | Number of mints to date |

**MintCircuit (π_mint):**

```
PUBLIC INPUTS:                 PRIVATE INPUTS:
• asset_genesis_cid            • mint_amount
• previous_mint_root           • recipient_pk
• new_mint_root                • genesis_record (full)
• previous_cumulative          • issuer_sk
• new_cumulative
• max_supply
• mint_count

PROVES:
1. new_cumulative = previous_cumulative + mint_amount
2. new_cumulative ≤ max_supply
3. new_mint_root = insert(previous_mint_root, H(genesis_record))
4. H(genesis_record).amount == mint_amount
5. H(genesis_record).recipient_pk == recipient_pk
6. H(derive_pk(issuer_sk)) ∈ current AssetKeyRegistry
7. max_supply matches AssetDefinition.max_supply from asset_genesis_cid
```

**Circuit cost:** ~8k constraints (range check + Merkle insert + signature verification).

**Properties:**
- Supply cap is cryptographically enforced, not honor-system.
- max_supply is bound to asset_genesis_cid — changing the cap would
  change the CID and create a different asset.
- Secret inflation is impossible: recipients refuse mints without
  valid MintProofs, and concurrent mints require issuer-side
  serialization (naturally rate-limiting).
- Fold circuit chains each slot's lineage back to a genesis that
  includes the MintProof, so every slot's validity depends on
  supply cap compliance.

**For uncapped assets (max_supply == None):** MintProof generation
is optional. The fold circuit uses the existing genesis path and
does not require supply accounting. This preserves the unlimited-
issuance model for loyalty points, game currencies without caps,
and issuer-discretion economies.

**GenesisRecord structure:**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| slot_commit | bytes | 32 | Recipient's slot commitment at seq=0 |
| forward_commit | bytes | 32 | Recipient's slot commitment at seq=1 |
| balance_commit | bytes | 32 | Pedersen(amount, blinding) |
| asset_id | bytes | 32 | Per §13.6 |
| asset_genesis_cid | bytes | 32 | CID of the AssetDefinition record (§13.6), published before first mint. NOT self-referential. |
| issuer_pk | bytes | 32 | Ed25519 (must be in AssetKeyRegistry) |
| recipient_pk | bytes | 32 | Ed25519 |
| amount | uint64 | 8 | Plaintext amount |
| timestamp | uint64 | 8 | Creation time |
| issuer_sig | bytes | 64 | Over content_hash |
| recipient_sig | bytes | 64 | Over content_hash |

**Validation (`validate_genesis`):**

| # | Check | Reject if |
|---|-------|-----------|
| 1 | Issuer is trusted | issuer_pk not in trusted registry |
| 2 | Asset matches issuer | asset_id ≠ registered asset for issuer |
| 3 | Issuer signature | Ed25519_Verify(issuer_pk, content_hash, issuer_sig) fails |
| 4 | Recipient signature | Ed25519_Verify(recipient_pk, content_hash, recipient_sig) fails |
| 5 | Balance commitment | pedersen_commit(amount, blinding) ≠ balance_commit |
| 6 | Slot derivation | slot_commit ≠ H("inat_slot:" \|\| recipient_sk \|\| 0) |


## 12. Canonical Slot Derivation

```python
def derive_slot_identifiers(sk: bytes, seq: int) -> Tuple[bytes, bytes, bytes]:
    """
    CANONICAL slot identifier derivation. All other code MUST reference this.
    """
    slot_commit = H(b"inat_slot:" || sk || seq.to_bytes(8, 'big'))
    nullifier = H(b"inat_nullifier:" || sk || seq.to_bytes(8, 'big'))
    nullifier_commit = H(nullifier)
    return slot_commit, nullifier, nullifier_commit
```

## 13. Slot Structure
### 13.1 Slot States


```python
class SlotStatus(Enum):
    READY = "READY"
    PENDING_SEND = "PENDING_SEND"
    PENDING_RECEIVE = "PENDING_RECEIVE"
    COMMITTED = "COMMITTED"
    SPENT = "SPENT"
    FINALIZED = "FINALIZED"
    RECOVERED = "RECOVERED"
    BURNED = "BURNED"

TERMINAL_STATES = frozenset({
    SlotStatus.SPENT,
    SlotStatus.FINALIZED,
    SlotStatus.RECOVERED,
    SlotStatus.BURNED,
})


class Phase(Enum):
    INITIATE = "INITIATE"    # Sender proof + W₁ attestation + share hold
    COMMIT = "COMMIT"        # Bilateral binding + W₂₃ attestation
    FINALIZE = "FINALIZE"    # Share release + nullifier publication
    CONFIRM = "CONFIRM"      # DA confirmation + proof folding
```

| State | Terminal | Description |
|-------|----------|-------------|
| READY | No | Spendable |
| PENDING_SEND | No | Sender committed (seq incremented, σ_s signed), awaiting recipient |
| PENDING_RECEIVE | No | Recipient committed (seq incremented, σ_r signed, sender proof verified), awaiting W₂₃ quorum |
| COMMITTED | No | W₂₃ quorum achieved. Point of no return. |
| SPENT | Yes | Nullifier published |
| FINALIZED | Yes | Proof folded, O(1) verifiable |
| RECOVERED | Yes | Abort from PENDING_SEND or PENDING_RECEIVE via 21/29 quorum |
| BURNED | Yes | Permanent value destruction |

### 13.2 Lineage Origins

| Origin | Source | Genesis proof |
|--------|--------|---------------|
| MINT | Issuer attestation | issuer_sig over mint_record |
| TRANSFER | Previous spend | collapsed_proof from fold circuit |
| SWEEP | Fee claim | sweep_proof from sweep circuit |
| RECOVERY | Witnessed abort | recovery_proof from recovery circuit |

### 13.3 Slot (Public View)

Shared with counterparties, witnesses, stored in documents.
Contains only commitments — no secrets.

| Field | Type | Size | Derivation |
|-------|------|------|------------|
| slot_commit | bytes | 32 | H("inat_slot:" \|\| sk \|\| seq) per §12 |
| nullifier_commit | bytes | 32 | H(nullifier) — safe to share |
| asset_id | bytes | 32 | H("inat_asset:" \|\| issuer_pk \|\| name \|\| params) |
| balance_commit | bytes | 32 | Pedersen(balance, blinding) per §6 |
| lineage | SlotLineage | var | See §13.5 |
| status | SlotStatus | 1 | One of §13.1 values |
| created_at | uint64 | 8 | Unix timestamp |
| spent_at | uint64? | 8 | Unix timestamp, present if spent |

A slot is **spendable** iff `status == READY`.

### 13.4 OwnedSlot (Private View)

Held only by slot owner. MUST NOT appear in any wire message,
document store, or witness request.

Extends Slot with:

| Field | Type | Size | Sensitivity |
|-------|------|------|-------------|
| owner_sk | bytes | 32 | SECRET — signing/derivation key |
| seq | uint64 | 8 | SECRET — sequence counter |
| balance | uint64 | 8 | SECRET — plaintext balance |
| balance_blinding | bytes | 32 | SECRET — Pedersen blinding factor |
| nullifier | bytes | 32 | SECRET — preimage of nullifier_commit |

**Invariant**: All public fields MUST be derivable from (owner_sk, seq)
via §12. Implementations MUST verify consistency on construction.

**Invariant**: For MINT-origin slots, `asset_id == lineage.asset_id`.

**Invariant**: Single-issuer-per-identity. Each identity (wallet) holds
slots of exactly one asset_id. This is not a convention — it is enforced
by the circuit constraints. Balance conservation (§31 constraint 5) binds
all commitments to a single asset_id, and forward_commit (§31 constraint 7)
chains seq=N to seq=N+1. Injecting a different asset_id at any sequence
number would break either conservation or the forward commit chain.
Consequence: sweep destinations are inherently per-issuer.

**Projection**: OwnedSlot → Slot by dropping all SECRET fields.
This projection MUST be applied before any external serialization.

### 13.5 SlotLineage

| Field | Present when | Type | Size |
|-------|-------------|------|------|
| origin | Always | LineageOrigin | 1 |
| mint_proof | MINT | bytes | var |
| issuer_pk | MINT | bytes | 32 |
| issuer_attestation | MINT | bytes | 64 |
| asset_id | MINT | bytes | 32 |
| source_nullifier_cid | TRANSFER | bytes | 32 |
| spend_record_cid | TRANSFER | bytes | 32 |
| collapsed_proof | TRANSFER, depth>0 | CollapsedProof | ~22KB |
| lineage_commit | Always | bytes | 32 |
| sweep_record_cid | SWEEP | bytes | 32 |
| sweep_proof | SWEEP | bytes | var |
| recovery_record_cid | RECOVERY | bytes | 32 |
| recovery_proof | RECOVERY | bytes | var |

### 13.6 Asset Definition

| Field | Type | Description |
|-------|------|-------------|
| asset_id | bytes(32) | H("inat_asset:" || asset_genesis_cid || name || params) |
| asset_genesis_cid | bytes(32) | CID of the AssetDefinition record at creation time (immutable identity anchor) |
| issuer_pk | bytes(32) | Ed25519 public key |
| name | string | Human-readable name |
| symbol | string | Ticker symbol |
| decimals | uint8 | Decimal places |
| max_supply | uint64? | Optional supply cap |
| description | string | Asset description |
| metadata_cid | bytes(32)? | Optional metadata blob CID |

**Intended asset classes**: Game currencies, virtual goods credits,
loyalty points, in-app economies, and other network-native point
systems where participants transact peer-to-peer within a shared
application context. The protocol provides double-spend prevention
for these economies without requiring a blockchain or global ledger.
Issuers define assets for their application ecosystem; the protocol
is agnostic to external valuation or fiat convertibility.


## 14. Recipient Identifiers

```python
RECIPIENT_TAG_DOMAIN = b"inat_receive_tag:"

def derive_recipient_identifiers(
    recipient_sk: bytes, seq: int, session_id: bytes,
) -> tuple[bytes, bytes, bytes]:
    slot_commit, nullifier, nullifier_commit = derive_slot_identifiers(recipient_sk, seq)
    receive_tag = H(RECIPIENT_TAG_DOMAIN || recipient_sk || seq.to_bytes(8, 'big') || session_id)
    receive_tag_commit = H(receive_tag)
    return slot_commit, receive_tag_commit, nullifier_commit
```

## 15. Pairing Commitment

```python
PAIRING_DOMAIN = b"inat_pairing"

@dataclass
class PairingData:
    pairing_tag: bytes           # H(shared_secret || session_id) — PRIVATE
    my_pairing_commit: bytes     # H(pairing_tag || my_role)
    their_pairing_commit: bytes
    pairing_bind: bytes          # H(sender_commit || recipient_commit)


def compute_pairing_data(
    my_sk: bytes, their_pk: bytes, session_id: bytes, my_role: Role
) -> PairingData:
    shared_secret = ecdh(my_sk, their_pk, context=b"pairing")
    pairing_tag = H(PAIRING_DOMAIN || b":" || shared_secret || session_id)
    sender_pairing_commit = H(b"inat_pairing_role:sender:" || pairing_tag)
    recipient_pairing_commit = H(b"inat_pairing_role:recipient:" || pairing_tag)
    pairing_bind = H(sender_pairing_commit || recipient_pairing_commit)

    if my_role == Role.SENDER:
        return PairingData(pairing_tag, sender_pairing_commit, recipient_pairing_commit, pairing_bind)
    else:
        return PairingData(pairing_tag, recipient_pairing_commit, sender_pairing_commit, pairing_bind)


def verify_counterparty(their_published: bytes, expected: bytes) -> bool:
    return constant_time_compare(their_published, expected)

def verify_pairing_as_witness(sender_commit: bytes, recipient_commit: bytes, claimed_bind: bytes) -> bool:
    return constant_time_compare(H(sender_commit || recipient_commit), claimed_bind)

# SHARD MANIPULATION RESISTANCE:
# Colluding parties who grind σ_r to steer seed₂₃ toward a
# Sybil-packed shard face two barriers:
# 1. Fold circuit enforces shard prefix membership (§33 constraint
#    6d) — Sybils must have NodeIDs matching the target shard
#    prefix, costing O(2^depth) key-grinds per Sybil.
# 2. Cross-phase depth consistency (constraint 6e) combined with
#    shard hopping requires Sybil coverage across three independent
#    shards. At depth=17, placing 1000 Sybils in three shards
#    costs ~400M keypair grinds — and the target shards shift
#    with each beacon round.
```

## 16. Threshold Data Structures

```python
@dataclass
class ThresholdEncryptedNullifier:
    encrypted_shares: List[bytes]
    polynomial_commitments: List[bytes]
    nullifier_commit: bytes
    threshold_m: int = 21
    total_n: int = 29


@dataclass
class PublishedShare:
    session_id: bytes
    witness_index: int
    share_value: bytes
    witness_pk: bytes
    beacon_round: int
    signature: bytes

    def verify(self, poly_commits: List[bytes]) -> bool:
        return feldman_vss.verify_share(self.share_value, self.witness_index, poly_commits)
```

---

# Part IV: Transaction State Machine

## 17. State Transitions

```
READY (seq=N)
    │ [Phase 1: sender signs, seq++, W₁ attests + holds shares]
PENDING_SEND (seq=N+1, σ_s committed)
    │ [Phase 2: recipient signs, W₂₃ quorum = POINT OF NO RETURN]
COMMITTED (bilateral binding proven)
    │ [Phase 3: W₁ releases shares, nullifier published]
SPENT (nullifier on Iroh-blob)
    │ [Phase 4: 7 DA confirmations, fold into proof]
FINALIZED (O(1) proof)

WITNESS SETS (rotated per phase, sized by tx tier §19.11):
  W₁ (Phase 1): Holds encrypted shares. Size = tier.witness_total.
  W₂₃ (Phase 2): Attests bilateral binding. Size = tier.witness_total.
  W₄  (Phase 4): DA confirmation (7 validators)
  All witnesses must be in current IssuerEligibleRoot (§19.9).

SHARE LIFECYCLE:
  Phase 1: Sender splits nullifier → 19 shares, encrypts to W₁
  Phase 2: W₂₃ quorum → point of no return
  Phase 3: W₁ sees W₂₃ quorum → release shares → reconstruct nullifier
```

```
# π₁ (sender proof) — verified by W₁ and recipient
async def verify_sender_proof(proof, public_inputs_bytes) -> bool:
    public_inputs = SenderPublicInputs.deserialize(public_inputs_bytes)
    return groth16_verify(VERIFICATION_KEYS["sender"], proof, public_inputs.serialize())

# π₂₃ (combined proof) — verified by W₂₃ and anyone auditing
async def verify_combined_proof(proof, public_inputs_bytes) -> bool:
    public_inputs = CombinedPublicInputs.deserialize(public_inputs_bytes)
    return groth16_verify(VERIFICATION_KEYS["combined"], proof, public_inputs.serialize())
```

# Part V: Transaction Protocol


## 18. Discovery Protocol

Bridges "I want to pay someone" → Phase 1. Layered reveal with
anti-sniffing gates.

### 18.1 Overview

| Step | Direction | Content | Anti-sniffing property |
|------|-----------|---------|----------------------|
| 0 | Public | Payment Token | No slot, no pk, no balance |
| 1 | Recipient → Payer | Capacity Proof (ZK) | Reveals nothing about slots |
| 2 | Payer → Recipient | Locked Funds Proof | Costs seq increment + lock |
| 3 | Recipient → Payer | Spot Reveal (ECDH) | Only payer can decrypt |
| 4 | — | Phase 1 begins | — |

### 18.2 Payment Token

Safe to share as QR/URL/NFC. Contains NO sensitive data.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| token_id | bytes | 16 | Random identifier |
| recipient_pk_commit | bytes | 32 | H(recipient_pk) — NOT raw pk |
| issuer_family_id | bytes | 32 | |
| amount | uint64? | 8 | Optional requested amount |
| memo | string | var | |
| expires_at | uint64 | 8 | 0 = no expiry |
| creator_signature | bytes | 64 | Sign(recipient_sk, token_id \|\| issuer \|\| amount \|\| expires) |

The signature proves the token creator controls recipient_pk.

### 18.3 Capacity Proof

Recipient proves they CAN receive without revealing which slot
or how much they hold.

**ZK proof (Groth16, ~192 bytes, ~3ms verify):**

Public inputs:

| Field | Type | Description |
|-------|------|-------------|
| slot_set_root | bytes(32) | Merkle root of owner's slot commitments |
| issuer_family_id | bytes(32) | |
| min_amount | uint64 | Minimum acceptable balance |
| owner_pk_commit | bytes(32) | H(owner_pk) |
| beacon_round | uint64 | Replay prevention |

Private inputs:

| Field | Type | Description |
|-------|------|-------------|
| owner_sk | bytes(32) | |
| chosen_slot_commit | bytes(32) | Which slot satisfies the proof |
| chosen_slot_seq | uint64 | |
| chosen_slot_balance | uint64 | |
| chosen_slot_balance_blinding | bytes(32) | |
| chosen_slot_asset_id | bytes(32) | |
| merkle_index | uint32 | Position in slot set |
| merkle_proof | List[(bytes, str)] | Merkle path |
| all_slot_commits | List[bytes] | Full slot set (for Merkle tree) |

Proves: ∃ slot ∈ owner's set with correct issuer and balance ≥ min.

**Verification rule:** Beacon round MUST be within
MAX_CAPACITY_PROOF_AGE_ROUNDS of current round.

### 18.4 Locked Funds Proof

Payer proves they locked funds. Cost per probe: 1 sequence
increment + slot lock + recovery requires 21/29 quorum.

**Prerequisite:** Payer's slot MUST already be PENDING_SEND
with sequence already incremented.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| payer_pk_commit | bytes | 32 | H(payer_pk) |
| amount_commit | bytes | 32 | Pedersen(amount, blinding) |
| locked_slot_commit_hash | bytes | 32 | H(slot_commit) — double-hashed |
| lock_signature | bytes | 64 | Sign(payer_sk, lock_payload) |
| locked_at | uint64 | 8 | |
| expires_at | uint64 | 8 | |
| nonce | bytes | 16 | Random |

**lock_payload derivation:**

    lock_payload = H("locked_funds_proof:"
        || amount_commit || locked_slot_commit_hash
        || nonce || locked_at || expires_at)

**Verification rules:**
- current_timestamp ≤ expires_at
- H(payer_pk) = payer_pk_commit
- verify_signature(payer_pk, lock_payload, lock_signature)

Note: verifier must know payer_pk (received during discovery handshake).

### 18.5 Spot Reveal

Recipient reveals receiving slot address, encrypted so only
payer can decrypt.

**Encryption:**

    shared_secret = ECDH(recipient_sk, payer_pk, context=b"spot_reveal")
    encryption_key = H("spot_reveal:" || shared_secret || session_id)
    ephemeral_sk = random(32)                    # Forward secrecy
    final_key = H(encryption_key || ECDH(ephemeral_sk, payer_pk))
    nonce = H("spot_nonce:" || session_id)[:12]
    ciphertext = AEAD_encrypt(final_key[:32], nonce, slot_commit, session_id)

**Wire format:**

| Field | Type | Size |
|-------|------|------|
| ciphertext | bytes | var |
| ephemeral_pk | bytes | 32 |
| session_id | bytes | 32 |

**Decryption** (payer side): reverse derivation using
ECDH(payer_sk, recipient_pk) and ECDH(payer_sk, ephemeral_pk).

    shared_secret = ECDH(payer_sk, recipient_pk, context=b"spot_reveal")

### 18.6 Discovery Session

**Session ID derivation:**

    discovery_session_id = H(payment_token.token_id || lock_proof.nonce)

**Discovery handshake (recipient side):**

1. Generate Capacity Proof. Send to payer via MessageTransport.
2. Receive Locked Funds Proof from payer. Verify per §18.4.
3. Select receiving slot. Encrypt via Spot Reveal (§18.5).
   Send to payer via MessageTransport.

**Discovery handshake (payer side):**

1. Receive Capacity Proof. Verify per §18.3.
   Verify: capacity_proof.owner_pk_commit = payment_token.recipient_pk_commit.
2. Lock funds: increment oracle seq, set source slot to PENDING_SEND.
   Build Locked Funds Proof. Send to recipient via MessageTransport.
3. Receive Spot Reveal. Decrypt per §18.5 to obtain recipient_slot_commit.

**Discovery complete â†' proceed to Phase 1 (§20.1).**

**Phase 1 entry from discovery:** When Phase 1 is preceded by the
discovery protocol, steps 1â€"2 (dual check and oracle increment) have
already been performed during the Locked Funds Proof (§18.4). The
slot is already PENDING_SEND at seq N+1. Phase 1 resumes at step 3
(session setup) using the already-incremented new_seq. The Phase 1
precondition "Source slot status = READY" applies only to the
direct-entry path (no discovery).


### 18.7 Discovery Failure

| Failure | State | Resolution |
|---------|-------|------------|
| Capacity proof invalid | No state change | Abort. No cost. |
| Payer locked but never received spot | Slot stuck in PENDING_SEND | Recovery via §23 (21/29 quorum). Seq increment is NOT recoverable (anti-sniffing cost). |
| Spot reveal timeout | Slot stuck in PENDING_SEND | Same as above. |
| Session expired | — | DISCOVERY_TIMEOUT = 120s. DISCOVERY_STEP_TIMEOUT = 30s. |

### 18.8 Privacy Requirements

All discovery messages MUST use MessageTransport with:

| Message type | encryption | padding | timing_jitter | relay |
|-------------|-----------|---------|---------------|-------|
| capacity_proof | ✓ | ✓ | ✓ (max 0.2s) | ✓ |
| locked_funds_proof | ✓ | ✓ | ✓ | ✓ |
| spot_reveal | ✓ | ✓ | ✓ (max 0.1s) | ✓ |

Spot reveal padding is critical: prevents slot address
length-based fingerprinting.

---


### Discovery Abort/Timeout

- If payer locked funds but never received spot reveal: slot stuck in PENDING_SEND
- Recovery via Section 23 (witnessed abort, 21/29 quorum). Sequence increment is NOT recoverable (anti-sniffing cost).

---

## 19. Witness Selection — SHARD-TARGETED SELF-SELECTION

### 19.1 Shard Targeting

Transactions target a gossip shard — a prefix-defined subset of
Inat nodes. The shard is derived deterministically from the phase seed.

**Constants:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| SHARD_TARGET_SIZE | 1000 | Target Inat nodes per shard. At this density, expected VRF self-selections = WITNESS_TOTAL = 29. |
| SHARD_MIN_DENSITY | 750 | Minimum safe shard population. Derived: ceil(WITNESS_QUORUM × SHARD_TARGET_SIZE / WITNESS_TOTAL) = 725, +25 margin = 750. Below this, expected self-selections approach WITNESS_QUORUM. Depth hysteresis merges shards before this is reached. |
| SHARD_MAX_DENSITY | 2000 | Split threshold |
| SHARD_MIN_DEPTH | 4 | Minimum prefix bits |
| SHARD_MAX_DEPTH | 24 | Maximum prefix bits |

def derive_shard_topic(seed: bytes, depth: int, asset_id: bytes) -> bytes:
    """Shard within an asset-scoped gossip namespace.
    
    NOTE: Shard membership is enforced at TWO levels:
    1. Gossip level: only nodes in the shard hear the envelope
    2. Fold level: §33 constraint 6d proves each witness's NodeID
       has the correct prefix for the phase seed's shard.
    Level 2 closes the gap where a Sybil in a different shard
    presents a valid VRF proof for a seed outside its shard.
    """
    seed_hash = H(b"inat_shard:" || asset_id || seed)
    seed_bits = int.from_bytes(seed_hash[:4], 'big')
    prefix = seed_bits >> (32 - depth)
    asset_prefix = asset_id[:4].hex()
    return f"inat/tx/v1/{asset_prefix}/{prefix:0{depth}b}".encode()

def witness_heartbeat_topic(asset_id: bytes) -> bytes:
    """Top-level heartbeat topic for asset supporter discovery."""
    asset_prefix = asset_id[:4].hex()
    return f"inat/witness/v1/{asset_prefix}".encode()

### 19.2 Shard Density

Only Inat-enabled nodes count toward shard density. Each node
tracks density within its own shard via signed heartbeats.

**Heartbeat interval:** 60 seconds.
**Stale threshold:** 600 seconds (peers not seen within this are evicted).

Density is now per-asset per-shard. A node participating in 5 assets tracks density independently for each:

@dataclass
class ShardHeartbeat:
    node_pk: bytes              # 32
    asset_id: bytes             # 32 — which asset this heartbeat is for
    shard_prefix: bytes         # var
    timestamp: int              # 8
    observed_density: int       # 4 — this node's count of supporters in THIS asset in THIS shard
    recommended_depth: int      # 2
    signature: bytes            # 64

# VRF THRESHOLD MODEL:
# Threshold is a hardcoded protocol constant:
#   VRF_THRESHOLD = (WITNESS_TOTAL / SHARD_TARGET_SIZE) × 2^256
#
# Expected self-selections = actual_shard_population × (WITNESS_TOTAL / SHARD_TARGET_SIZE)
# SHARD_MIN_DENSITY is derived to keep expected self-selections > WITNESS_QUORUM:
#   SHARD_MIN_DENSITY = ceil(WITNESS_QUORUM × SHARD_TARGET_SIZE / WITNESS_TOTAL) + margin
#                     = ceil(21 × 1000 / 29) + 26 = 750
# Shard depth adaptation maintains population ≥ SHARD_MIN_DENSITY.
# No density measurement enters threshold computation.
# Heartbeats are used ONLY for shard depth consensus (§19.2).
#
# VRF selection is verified inside ZK circuits (fold §33 constraint 6)
# to prevent fake witness injection.

A node whitelisting 5 assets sends 5 heartbeats per interval (one per asset-shard it belongs to). At 60s intervals and ~150 bytes each, that's 750 bytes/min — negligible.

#### 19.2.1 — Depth Hysteresis

```python
SHARD_SPLIT_THRESHOLD = 1.2 * SHARD_MAX_DENSITY    # 2400
SHARD_MERGE_THRESHOLD = 0.8 * SHARD_MIN_DENSITY     # 600
# NOTE: SHARD_MIN_DENSITY = 750 is derived, not arbitrary:
#   floor = ceil(WITNESS_QUORUM × SHARD_TARGET_SIZE / WITNESS_TOTAL) = 724
#   At SHARD_MIN_DENSITY the expected self-selections =
#   750 × (29/1000) = 21.75 — just above WITNESS_QUORUM = 21.
#   MERGE_THRESHOLD = 600 → expected = 17.4 → triggers merge before
#   quorum becomes unreachable.
DEPTH_CHANGE_COOLDOWN = 600                          # 10 minutes
```

Depth increases only when density exceeds SPLIT for COOLDOWN duration
Depth decreases only when density drops below MERGE for COOLDOWN duration
Prevents oscillation at boundary

### 19.3 Shard Depth Consensus

Per-asset depth consensus. Nodes track median recommended depth from heartbeats within each asset namespace:

class AssetShardConsensus:
    def __init__(self):
        self._depths: Dict[bytes, DepthTracker] = {}  # asset_id → tracker
    
    def compute_consensus(self, asset_id: bytes) -> int:
        tracker = self._depths.get(asset_id)
        if not tracker or tracker.supporter_count < ASSET_MIN_SUPPORTERS:
            raise AssetBelowThresholdError(asset_id, tracker.supporter_count)
        return tracker.median_depth

Shard depth is a network consensus property. Nodes converge on
the **median** of their neighborhood's depth recommendations.

**Verification rule:** Witnesses MUST reject transaction intents
where `|claimed_depth - consensus_depth| > 1`.

**Fold-level enforcement:** The fold circuit (§33 constraint 6d-6e)
additionally proves that each witness's NodeID prefix matches the
shard derived from the phase seed, and that declared depth is
consistent across all three phases. This closes the gap between
gossip-level shard routing and ZK-level shard membership proof.

**Convergence guarantee:** All honest nodes in a shard converge
to the same depth within 2 heartbeat intervals (10 minutes).

### 19.4 VRF Self-Selection

Every Inat node in the target shard evaluates the VRF lottery independently.

**Selection test (distance-weighted):**

    # local_density = this node's own heartbeat count for this asset
    # in this shard. Never transmitted. Never accepted from a peer.
    # Every node independently derives the same threshold (±20%)
    # from the same gossip heartbeats.

    (output, proof) = VRF_prove(node_sk, seed)
    VRF_THRESHOLD = (WITNESS_TOTAL / SHARD_TARGET_SIZE) × 2^256
    # Hardcoded ratio. Expected self-selections at population N:
    #   E[selected] = N × (WITNESS_TOTAL / SHARD_TARGET_SIZE)
    #
    # At SHARD_TARGET_SIZE (1000): E = 29  ← full witness set
    # At SHARD_MIN_DENSITY  (750): E = 21.75 ← just above WITNESS_QUORUM
    # At SHARD_MERGE_THRESH (600): E = 17.4  ← merge triggered before this
    #
    # SHARD_MIN_DENSITY is therefore NOT arbitrary — it is derived as:
    #   ceil(WITNESS_QUORUM × SHARD_TARGET_SIZE / WITNESS_TOTAL) + safety margin
    # = ceil(21 × 1000 / 29) + 25 = 725 + 25 = 750
    #
    # Shard depth adaptation (§19.2) maintains population ≥ SHARD_MIN_DENSITY,
    # which guarantees E[selected] > WITNESS_QUORUM at all operating densities.
    # No density measurement enters threshold computation.
    # Heartbeats are used ONLY for shard depth consensus (§19.2).
    # SHARD_MIN_DENSITY prevents undersized shards.
    # ASSET_MIN_SUPPORTERS prevents undersized networks.

    # Distance weighting: nodes farther from anchor pass more easily
    anchor = H("inat_diversity_anchor:" || seed)
    xor_dist = max(uint128(xor(node_id, anchor) >> 128), 1)
    adjusted_threshold = base_threshold × min(xor_dist, MAX_DIVERSITY_BOOST)

    selected = uint256(output) < adjusted_threshold

    # Where adjusted_threshold applies distance weighting to VRF_THRESHOLD
    # (not density-derived). See distance weighting below.

| Parameter | Value |
|-----------|-------|
| VRF_THRESHOLD_RATIO | WITNESS_TOTAL / SHARD_TARGET_SIZE = 29/1000 |
| MAX_DIVERSITY_BOOST | 2^64 |

**Verification:** Anyone can verify a claimed selection given
(node_pk, seed, output, proof). The verifier computes threshold
from its OWN local heartbeat density — never from any wire
parameter. There is no density oracle and no claimed_density
input to verification.

    valid = vrf_verify(node_pk, seed, output, proof)
            AND uint256(output) < adjusted_threshold
    # Threshold is hardcoded (VRF_TARGET_WITNESSES / SHARD_TARGET_SIZE),
    # same for selector and verifier. No density input.

Since all honest nodes in the same asset-scoped shard observe
the same signed heartbeats, their density estimates converge
naturally (±20%). This means a candidate selected at density=950
will also pass verification at density=1050. The protocol
tolerates this: any ≥13 valid attestations form a quorum.

An attacker cannot inflate selection probability by lying about
density because density never crosses the wire — verifiers
derive it independently from signed, NodeID-bound heartbeats.
The only way to inflate density is to inject real signed
heartbeats, which requires real nodes, at which point the
distance-weighted scoring (§19.5) penalizes clustering.

If real density is genuinely too low (< SHARD_MIN_DENSITY = 750),
depth hysteresis triggers a merge before expected self-selections
fall below WITNESS_QUORUM. If the merge cannot restore density
(total asset supporters < ASSET_MIN_SUPPORTERS), fewer than
WITNESS_QUORUM witnesses self-select → transaction fails. This is
correct behavior — the network is too thin for safety. No
oracle is needed; it falls out of the math.

### 19.5 Diversity Ranking (Post-Selection)

After collecting distance-weighted VRF winners (§19.4), the sender
ranks them by diversity score and caps at WITNESS_TOTAL_CAP. This
is an ordering step — all VRF winners are valid, but only the top
19 are assigned indices. The ranking uses the same anchor and
distance preference as selection, ensuring consistency.

    anchor = H("inat_diversity_anchor:" || seed)
    score(node) = uint128(vrf_output[:16]) × max(uint128(xor(node_id, anchor) >> 128), 1)

Sender takes top-ranked WITNESS_TOTAL_CAP witnesses. Rankings are
deterministic and verifiable from public data.

### 19.6 Seed Chain

Each phase derives its seed from the previous phase's entropy,
causing transactions to hop to different shards.

# Phase 1 — includes asset_id
seed_1 = H(slot_commit || beacon_round || beacon_randomness || asset_id)

# Phase 2 — recipient entropy
seed_23 = H(seed_1 || H(σ_r) || beacon_round_23 || beacon_randomness_23)

# Phase 4
seed_4 = H(seed_23 || spend_cid || beacon_round_4 || beacon_randomness_4)

**Beacon per phase:** Each phase fetches a fresh beacon round at
broadcast time. beacon₁ is fetched during Phase 1 (§20.1 step 3).
beacon₂₃ is fetched during Phase 2 (§20.2 step 12). beacon₄ is
fetched during Phase 4 (§20.4 step 1). These MAY be the same round
if phases execute within one beacon interval (30s), but are typically
different.

**Verification rule:** Witnesses in Phase 2 and Phase 4 MUST
recompute the expected seed from its inputs and reject mismatches.

**Grinding resistance (ZK-enforced):** Shard hopping is not merely
a gossip-level convention — the fold circuit (§33 constraint 6d)
verifies that each witness's NodeID has the correct prefix for
the shard derived from that phase's seed. A Sybil with a valid
VRF proof but wrong shard prefix cannot appear in a valid fold
proof. Combined with cross-phase depth consistency (constraint 6e),
an attacker must grind O(2^depth) keypairs per Sybil per phase.
At depth=17 with three phase hops, placing 1000 Sybils across
all three target shards costs ~400M keypair grinds — and the
target shards shift with each beacon round.

**Colluding sender+recipient grinding σ_r:** Colluding parties
control σ_r and can grind the nonce to steer seed₂₃ toward a
Sybil-packed shard (~131K attempts at depth=17). This requires
pre-placed Sybils in the target shard. Since the target changes
per beacon round (30s), the attacker must either: (a) cover all
131K shards with Sybils (131M nodes), or (b) grind σ_r within
a single beacon window to hit their Sybil shard. Constraint 6d
ensures this grinding must produce real NodeID prefix matches,
not just valid VRF proofs.

### 19.7 Witness Index Assignment

Witnesses do not have pre-assigned indices. After collection,
indices are assigned deterministically:

    sorted = sort witnesses by vrf_output ascending
    capped = sorted[:WITNESS_TOTAL_CAP]
    index(witness) = position_in_capped + 1

| Parameter | Value |
|-----------|-------|
| WITNESS_TOTAL_CAP | Tier-dependent (13–49, default 29) |
| WITNESS_QUORUM | Tier-dependent (9–33, default 21) |

### 19.8 Shadow DHT Health Map

Background registration — NOT used for witness selection
Used only for pre-transaction density estimation

SHADOW_DHT_INTERVAL = 600  # Same as heartbeat

async def publish_witness_availability(node_pk, asset_id, iroh_node):
    """Background task. Publishes to DHT for health estimation.
    NOT used for transaction routing."""
    position = H(b"inat_witness_health:" || asset_id || node_pk)
    record = WitnessHealthRecord(
        node_pk=node_pk,
        asset_id=asset_id,
        timestamp=current_timestamp(),
        signature=sign(node_sk, ...),
    )
    await iroh_node.dht_provide(
        key=H(b"inat_health_bucket:" || asset_id),
        value=record.serialize(),
        ttl=SHADOW_DHT_INTERVAL * 2,
    )

async def estimate_asset_health(asset_id, iroh_node) -> int:
    """Pre-transaction check. Returns estimated supporter count."""
    providers = await iroh_node.dht_get_providers(
        H(b"inat_health_bucket:" || asset_id))
    # Scale by sampling ratio
    return len(providers) * estimated_scaling_factor(providers)

The wallet uses this before attempting a transaction:

estimated = await estimate_asset_health(asset_id, iroh)
if estimated < ASSET_MIN_SUPPORTERS:
    warn_user("This asset has insufficient network support")
    return
Proceed with Phase 1

### 19.9 Issuer Eligible Header

VRF eligibility is gated by the asset issuer. The issuer
periodically signs a Merkle root of all wallet public keys
eligible for VRF participation and delivers the signed header
plus per-wallet Merkle proofs directly to eligible wallets.

**Headers are NOT published as Iroh blobs.** There is no
gossip topic for eligible roots. Each eligible wallet receives
its own (signed_header, merkle_proof) tuple directly from the
issuer. Witnesses carry the signed header inline in their
WitnessEligibility responses, making attestation bundles
self-contained.

**Eligibility criteria (enforced by issuer):**
- Wallet holds ≥ MIN_WITNESS_BALANCE of the issuer's asset
- Wallet is not blacklisted (equivocation, inactivity)
- Wallet meets issuer's admission policy (per-issuer discretion)

**Epoch header construction:**

Every ISSUER_ELIGIBLE_ROOT_EPOCH, the issuer:

1. Collects all wallet PKs meeting eligibility criteria.
2. Applies issuer-defined admission criteria (balance check,
   blacklist check, issuer discretion).
3. Computes `eligible_root = Merkle(sort(eligible_wallet_pks))`.
4. Signs the header payload (see IssuerEligibleHeader below).
5. Signs with any authorized key in the asset's AssetKeyRegistry.
   Fold circuit verifies signing key ∈ issuer_registry_root.
6. For each eligible wallet: computes Merkle proof and sends
   (signed_header, merkle_proof) directly to the wallet via
   the same transport channel as mint delivery.

**Wallet-side behavior on header receipt:**

1. Verify issuer signature on the header payload.
2. Verify issuer_pk ∈ current AssetKeyRegistry (cached).
3. Verify own Merkle proof against header.eligible_root.
4. Store (signed_header, merkle_proof) locally.
5. Include the full bundle in subsequent WitnessEligibility
   responses.

**Revocation latency:** One epoch. Removed wallets will not
appear in the next epoch's header, and the previous header
expires after ELIGIBLE_ROOT_MAX_AGE_EPOCHS.

```python
@dataclass
class IssuerEligibleHeader:
    """Issuer-signed snapshot of VRF-eligible wallet PKs.
    Self-contained — delivered directly to eligible wallets
    and carried inline in WitnessEligibility responses.
    NOT published as an Iroh blob."""
    epoch_number: int
    eligible_root: bytes            # Merkle(sort(eligible_pks))
    asset_id: bytes
    issuer_pk: bytes
    eligible_count: int
    issued_at: int                  # Unix timestamp
    registry_root: bytes            # AssetKeyRegistry.registry_root at signing time
    registry_epoch: int             # AssetKeyRegistry.epoch at signing time
    issuer_signature: bytes

    def signing_payload(self) -> bytes:
        return msgpack.packb({
            "epoch_number": self.epoch_number,
            "eligible_root": self.eligible_root,
            "asset_id": self.asset_id,
            "eligible_count": self.eligible_count,
            "issued_at": self.issued_at,
            "registry_root": self.registry_root,
            "registry_epoch": self.registry_epoch,
        })

    def verify(self, registry: 'AssetKeyRegistry' = None) -> bool:
        """Verify signature. If registry provided, also verify
        issuer_pk is an authorized key and registry_root matches."""
        sig_valid = Ed25519_Verify(
            self.issuer_pk,
            self.signing_payload(),
            self.issuer_signature)
        if not sig_valid:
            return False
        if registry:
            if self.issuer_pk not in registry.authorized_keys:
                return False
            if self.registry_root != registry.registry_root:
                return False
            if self.registry_epoch != registry.epoch:
                return False
        return True
```

**Per-witness eligibility proof:**

Each witness carries the signed header plus their own Merkle
proof inline. No external blob lookup is required.

```python
@dataclass
class EligibilityProof:
    """Witness proves VRF eligibility via self-contained signed header
    plus Merkle membership proof. Fully self-authenticating given a
    cached AssetKeyRegistry."""
    witness_pk: bytes
    header: IssuerEligibleHeader    # Full signed header (not a CID)
    merkle_proof: List[Tuple[bytes, str]]  # Inclusion proof against header.eligible_root

    def verify(self, registry: 'AssetKeyRegistry') -> bool:
        if not self.header.verify(registry):
            return False
        leaf = merkle_leaf_hash(self.witness_pk)
        return verify_merkle_proof(leaf, self.merkle_proof, self.header.eligible_root)
```

**Verification in fold circuit (§33):**
- Issuer signed the header payload for this asset_id
- Signing issuer_pk ∈ issuer_registry_root (Merkle proof)
- Each witness_pk is in header.eligible_root (Merkle proof)
- header.epoch_number is recent (within ELIGIBLE_ROOT_MAX_AGE_EPOCHS)
- Header payload signature is verified natively at fold-verify time
  (not inside the circuit — bound via hash chain per §33.1)

**Storage:** Each eligible wallet caches its own (signed_header, merkle_proof) tuple locally (~300 bytes header + ~640 bytes proof at 1M eligible wallets). This is the ONLY persistent copy — no gossip, no blob, no shared cache. The wallet discards headers older than ELIGIBLE_ROOT_MAX_AGE_EPOCHS + 1. During transaction participation, the header travels inline in WitnessEligibility responses to the transaction's sender, then gets absorbed into the CollapsedProof via §33.1 hash binding. After the fold exists, the header can be discarded entirely — the proof is self-authenticating.

**Revocation latency:** One epoch. Removed wallets do not receive a new header for the next epoch; their previous header expires after ELIGIBLE_ROOT_MAX_AGE_EPOCHS. Worst case: misbehaving witness participates for one more epoch.

**Forced finalization (§22):** Because the header travels inline in the WitnessEligibility bundle and is captured in the TransactionStateRecord at attestation time, transactions stuck in COMMITTED for up to 72 hours retain all data needed to generate a fold proof. No "stale header blob" edge case exists — the header is already in the transaction state, owned by both parties via wallet document replication. Even if the issuer revokes the witness and stops publishing new headers, the in-flight transaction can still finalize using the header that was valid at attestation time.

**Privacy:** Headers travel point-to-point from issuer to eligible
wallet and inline inside WitnessEligibility responses. The issuer
learns wallet PKs (already known from mint). Broader network
visibility is scoped to transactions where the witness actually
participates — not a global publication.


| Parameter | Value | Description |
|-----------|-------|-------------|
| ISSUER_ELIGIBLE_ROOT_EPOCH | 3600s (1 hour) | Epoch duration |
| ELIGIBLE_ROOT_MAX_AGE_EPOCHS | 2 | Grace period for in-flight txns |
| MIN_WITNESS_BALANCE | Application-defined | Minimum balance for eligibility |

### 19.10 Issuer Transparency Log

The issuer maintains an append-only Merkle tree of ALL identities
ever approved (not just currently eligible). This allows anyone
to audit total issuance.

**Structure:** Merkle Mountain Range (MMR) of all wallet PKs ever
issued a genesis slot. Published as an Iroh document, replicated
to REDUNDANCY_TARGET (50) peers.

**Auditable properties:**
- **Inclusion proof:** "My wallet PK is in the issuance log."
- **Consistency proof:** "Log version N is a prefix of version M"
  (proves no deletions).
- **Size attestation:** "Total issued identities = X."

**Growth velocity limit:** If `eligible_count` increases by more
than ISSUER_MAX_GROWTH_RATE per epoch, nodes MAY flag the issuer
and refuse new attestations until a social/manual audit occurs.

```python
@dataclass
class IssuerTransparencyRecord:
    """Periodic snapshot of total issuance for audit."""
    asset_id: bytes
    issuer_pk: bytes
    total_issued: int               # All-time count
    current_eligible: int           # Currently in eligible root
    mmr_root: bytes                 # Merkle Mountain Range root
    epoch_number: int
    issuer_signature: bytes

    def verify(self) -> bool:
        payload = msgpack.packb({
            "asset_id": self.asset_id,
            "total_issued": self.total_issued,
            "current_eligible": self.current_eligible,
            "mmr_root": self.mmr_root,
            "epoch_number": self.epoch_number,
        })
        return Ed25519_Verify(self.issuer_pk, payload, self.issuer_signature)
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| ISSUER_MAX_GROWTH_RATE | 0.05 | Max 5% growth per epoch |
| ISSUER_LOG_REPLICAS | 50 | = REDUNDANCY_TARGET |

### 19.11 Witness Count

All transactions use a fixed witness set: WITNESS_TOTAL witnesses,
requiring WITNESS_QUORUM attestations.

## 20. Transaction Phases

### 20.1 Phase 1: Sender Initiates (READY → PENDING_SEND)

**Preconditions:**
- Source slot status = READY (or PENDING_SEND if preceded by discovery â€" see §18.6)
- Source slot balance â‰¥ amount + fee
- Source slot asset_id matches intended transfer

NOTE: When entered from discovery protocol (§18.6), the slot is
already PENDING_SEND with seq incremented. Steps 1â€"2 are skipped;
execution begins at step 3. All subsequent steps are identical.

**Steps:**

1. **Dual check.**
   Verify oracle seq matches expected AND nullifier not published.
   Both checks use independent fetches (§21.2 tiered lookup).

       current_seq = await oracle.get_seq(H(sender_pk))
       assert current_seq == source_slot.seq
       assert not await nullifier_store.exists(source_slot.nullifier_commit)

2. **Increment sequence (deferred to step 21).**
   The protocol_seq increment happens as part of the Phase 1 doc
   entry write (step 21). The entry model guarantees exactly one
   increment per phase transition. Oracle propagation (gossip
   SequenceUpdate) is broadcast simultaneously with the doc write.

   At this point, compute the target seq:

       base_seq = source_slot.seq                    # N
       phase1_seq = base_seq + 1                     # N+1

   The actual doc write occurs in step 21 after all proof generation
   and witness collection completes. This ensures the entry contains
   all CID references (proof blob, attestation blob, etc.) in a
   single atomic write.

3. **Session setup.**
   Generate random session_id (32 bytes). Record current beacon round
   and randomness from drand (§22).

       session_id = random_bytes(32)
       beacon_1 = await beacon_client.get_latest()
       beacon_round_1 = beacon_1.round
       beacon_randomness_1 = beacon_1.randomness

4. **Derive identifiers for the consumed slot.**
   Uses source slot's ORIGINAL seq (the seq it was created at).
   slot_commit is already known from the source slot.

       slot_commit = source_slot.slot_commit
       nullifier = H("inat_nullifier:" || sender_sk || source_slot.seq.to_bytes(8))
       nullifier_commit = H(nullifier)

   NOTE: `slot_commit` uses source_slot.seq (= N). This is DIFFERENT
   from sender_output_commit which uses new_seq (= N+1). See step 5.

5. **Derive sender output slot.**
   The sender's remaining balance goes to a new output slot at
   base_seq + SEQ_STRIDE. This reserves SEQ_STRIDE seq positions
   for this transaction's phase entries (one per state transition).

       base_seq = source_slot.seq                    # N (consumed slot's seq)
       sender_output_seq = base_seq + SEQ_STRIDE     # N+4
       sender_output_commit = H("inat_slot:" || sender_sk || sender_output_seq.to_bytes(8))
       sender_output_amount = source_slot.balance - amount - total_fee
       sender_output_blinding = random_bytes(32)
       sender_output_amount_commit = pedersen_commit(sender_output_amount, sender_output_blinding)

6. **Derive sender forward commit.**
   Binds this transaction's output to the sender's next predetermined
   slot. The forward commit is SEQ_STRIDE past the output slot,
   reserving space for the next transaction's phase entries.
   Fold circuit (§33) verifies without revealing sk.

       sender_forward_commit = H("inat_slot:" || sender_sk
                                  || (base_seq + 2 * SEQ_STRIDE).to_bytes(8))

   Sequence semantics:

   | Slot / Entry | Seq | Role |
   |--------------|-----|------|
   | Consumed (spent) | N (base_seq) | Created in prior tx or mint |
   | Entry: PENDING_SEND | N+1 | Phase 1 state transition |
   | Entry: COMMITTED | N+2 | Phase 2 state transition |
   | Entry: SPENT | N+3 | Phase 3 state transition |
   | Entry: FINALIZED + output | N+4 (base_seq+SEQ_STRIDE) | Phase 4, output slot materializes |
   | Sender forward | N+8 (base_seq+2×SEQ_STRIDE) | Bound for next tx's output |

   Recovery sequence (abort from PENDING_SEND at N+1):

   | Entry | Seq | Role |
   |-------|-----|------|
   | Entry: R₁ quorum | N+2 | Recovery investigation |
   | Entry: R₂ quorum | N+3 | Recovery confirmation |
   | Entry: RECOVERED + output | N+4 | Output slot, same target as normal |

7. **Compute pairing.**
   ECDH-derived session-bound commitments per §15.

       pairing = compute_pairing_data(sender_sk, recipient_pk, session_id, Role.SENDER)

8. **Compute fee.**
   Based on witness quorum. Donation adds +1.

       witness_count = (WITNESS_QUORUM + 1) if donation else WITNESS_QUORUM
       total_fee = witness_count * FEE_SHARE + priority_fee

9. **Build Pedersen commitments.**
   Amount, fee, sender output. All over Ristretto255 (§6).

       amount_commit = pedersen_commit(amount, amount_blinding)
       fee_commit = pedersen_commit(total_fee, fee_blinding)

10. **Generate nullifier polynomial and commitments.**
    Polynomial and commitments are generated BEFORE broadcast.
    Per-witness share evaluation happens after collection (step 18).

        coefficients = [scalar_from(nullifier)] + [random_scalar() for _ in range(WITNESS_QUORUM - 1)]
        poly_commits = [coef * G for coef in coefficients]
        poly_commits_root = H_vss_root(
            WITNESS_QUORUM.to_bytes(2, 'big') + b"".join(c.compress() for c in poly_commits))

11. **Build partial SenderCommitment.**
    All fields populated EXCEPT witness_tpk (unknown until step 16).
    σ_s is deferred to step 17.

        sender_commitment = SenderCommitment(
            owner_pk_commit           = H(sender_pk),
            seq                       = new_seq,
            slot_commit               = slot_commit,
            nullifier_commit          = nullifier_commit,
            amount_commit             = amount_commit,
            sender_output_commit      = sender_output_commit,
            sender_output_amount_commit = sender_output_amount_commit,
            fee_commit                = fee_commit,
            recipient_pk_commit       = H(recipient_pk),
            lineage_commit            = H(source_slot.lineage.serialize()),
            asset_id                  = asset_id,
            session_id                = session_id,
            beacon_round              = beacon_round_1,
            witness_tpk               = b"",       # Set in step 16
            donation                  = donation,
            sender_forward_commit     = sender_forward_commit,
            eligible_root_epoch       = current_eligible_root.epoch_number,
        )

12. **Generate π₁.**
    ZK proof per §31. Proves: slot derivation, nullifier derivation,
    balance conservation, lineage, sender_forward_commit, poly binding.
    Does NOT prove σ_s or witness binding (deferred to π₂₃ in Phase 2).

        public_inputs = SenderPublicInputs(
            slot_commit                 = slot_commit,
            nullifier_commit            = nullifier_commit,
            amount_commit               = amount_commit,
            sender_output_commit        = sender_output_commit,
            sender_output_amount_commit = sender_output_amount_commit,
            fee_commit                  = fee_commit,
            asset_id                    = asset_id,
            owner_pk_commit             = H(sender_pk),
            session_id                  = session_id,
            beacon_round                = beacon_round_1,
            lineage_commit              = H(source_slot.lineage.serialize()),
            poly_commits_root           = poly_commits_root,
            sender_forward_commit       = sender_forward_commit,
        )

        private_witness = SenderWitness(
            sk                     = sender_sk,
            seq                    = source_slot.seq,
            input_balance          = source_slot.balance,
            input_blinding         = source_slot.balance_blinding,
            amount                 = amount,
            amount_blinding        = amount_blinding,
            sender_output_amount   = sender_output_amount,
            sender_output_blinding = sender_output_blinding,
            fee                    = total_fee,
            fee_blinding           = fee_blinding,
            lineage_proof          = source_slot.lineage.serialize(),
            nullifier              = nullifier,
        )

        if donation:
            proof_1 = await zk_prover.prove_phase1_donation(public_inputs, private_witness)
        else:
            proof_1 = await zk_prover.prove_phase1(public_inputs, private_witness)

13. **Compute shard target.**
    seed₁ and shard_topic per §19.6. Depth from network consensus.

        consensus_depth = shard_depth_consensus.compute_consensus()
        seed_1 = H(slot_commit || beacon_round_1.to_bytes(8) || beacon_randomness_1)
        shard_bits = int.from_bytes(H("inat_shard:" || seed_1)[:4], 'big')
        prefix = shard_bits >> (32 - consensus_depth)
        shard_topic = f"inat/tx/v1/{prefix:0{consensus_depth}b}".encode()

14. **Broadcast envelope to shard.**
    Via MessageTransport with phase1_envelope privacy config.
    Envelope contains ONLY VRF selection data — no identity,
    asset, or proof information.

        envelope = IntentEnvelope(
            phase              = Phase.INITIATE,
            session_id         = session_id,
            seed               = seed_1,
            shard_depth        = consensus_depth,
            reply_path         = transport.create_reply_path(),
            beacon_round       = beacon_round_1,
            nullifier_commit   = nullifier_commit,
            observed_density   = density_tracker.current_density,
        )
        await transport.broadcast(
            shard_topic, envelope.serialize(),
            privacy=PROTOCOL_PRIVACY_CONFIGS["phase1_envelope"])

15. **Collect witness eligibility.**
    Determine witness tier from amount (§19.11). Wait for
    ≥tier.witness_total VRF-verified eligibility responses.
    Each eligibility response MUST include an EligibilityProof
    (§19.9). Reject witnesses not in current IssuerEligibleRoot.

        raw_eligible = []
        deadline = current_timestamp() + ELIGIBILITY_COLLECTION_TIMEOUT

        while len(raw_eligible) < WITNESS_TOTAL_CAP and current_timestamp() < deadline:
            remaining = deadline - current_timestamp()
            msg = await transport.receive(session_filter, timeout=remaining)
            if msg is None:
                break
            elig = WitnessEligibility.deserialize(msg.payload)

            if elig.node_pk in local_blacklist:                       continue
            if not vrf_verify(elig.node_pk, seed_1,
                              elig.vrf_output, elig.vrf_proof):       continue
            if not vrf_threshold_check(elig.vrf_output):              continue
            if elig.session_id != session_id:                         continue
            # Issuer eligibility check
            if not elig.eligibility_proof:                            continue
            if not elig.eligibility_proof.verify(
                    current_eligible_root.eligible_root):             continue
            if elig.eligibility_proof.epoch_number < (
                    current_epoch - ELIGIBLE_ROOT_MAX_AGE_EPOCHS):    continue
            raw_eligible.append(elig)

        if len(raw_eligible) < WITNESS_QUORUM:
            raise InsufficientWitnessesError(len(raw_eligible), WITNESS_QUORUM)

16. **Rank by diversity, assign indices, set witness_tpk.**
    Rank per §19.5. Cap at WITNESS_TOTAL. Assign Feldman VSS
    indices 1..N per §19.7. Finalize SenderCommitment with witness_tpk.
    
        anchor = H("inat_diversity_anchor:" || seed_1)
        ranked = sorted(raw_eligible, key=lambda e:
            uint128(e.vrf_output[:16]) * max(uint128(xor(e.node_pk, anchor) >> 128), 1),
            reverse=True)
        capped = ranked[:WITNESS_TOTAL_CAP]

        index_map = {}
        for i, elig in enumerate(capped):
            index_map[elig.node_pk] = i + 1

        witness_pks = sorted([e.node_pk for e in capped])
        witness_tpk = H(b"TPK:" + b"".join(witness_pks))
        sender_commitment.witness_tpk = witness_tpk

17. **Sign σ_s.**
    Sign over the FINALIZED commitment (with witness_tpk populated).

        commitment_hash = H(sender_commitment.serialize())
        sigma_s = Ed25519_Sign(sender_sk, commitment_hash)

18. **Deliver payload to selected witnesses.**
    Send full IntentPayload individually to each selected witness
    via their reply_path. Only these ~19 nodes learn oracle_key,
    asset_id, proof, poly_commits, and public_inputs.

        payload = IntentPayload(
            session_id         = session_id,
            slot_doc_id        = wallet_doc.id(),
            oracle_key         = H(sender_pk),
            claimed_slot_commit = slot_commit,
            claimed_asset_id   = asset_id,
            zk_proof           = proof_1,
            public_inputs      = public_inputs.serialize(),
            poly_commits       = [c.compress() for c in poly_commits],
            expected_seq       = new_seq,
        )

        for elig in capped:
            await transport.send(
                elig.reply_path, payload.serialize(),
                privacy=PROTOCOL_PRIVACY_CONFIGS["intent_payload"])

19. **Collect attestations.**
    Selected witnesses verify proof + mutable state, then attest.

        attestation_responses = []
        deadline = current_timestamp() + PAYLOAD_ATTESTATION_TIMEOUT

        while len(attestation_responses) < WITNESS_QUORUM and current_timestamp() < deadline:
            remaining = deadline - current_timestamp()
            msg = await transport.receive(session_filter, timeout=remaining)
            if msg is None:
                break
            response = WitnessResponse.deserialize(msg.payload)
            if not response.attestation.verify():                     continue
            if response.attestation.session_id != session_id:         continue
            attestation_responses.append(response)

        if len(attestation_responses) < WITNESS_QUORUM:
            raise InsufficientWitnessesError(len(attestation_responses), WITNESS_QUORUM)

        attestations_1 = [r.attestation for r in attestation_responses[:WITNESS_QUORUM]]

20. **Split nullifier and distribute shares.**
    Evaluate polynomial at each witness index. Encrypt each share to
    the assigned witness's public key (§10 ephemeral ECDH). Deliver
    individually via MessageTransport with share_delivery privacy.

        shares = [evaluate_polynomial(coefficients, idx)
                  for idx in range(1, len(capped) + 1)]

        for elig in capped:
            idx = index_map[elig.node_pk]
            encrypted_share = feldman_vss.encrypt_share_for_witness(
                shares[idx - 1], elig.node_pk)
            payload = ShareDelivery(
                session_id      = session_id,
                witness_index   = idx,
                encrypted_share = encrypted_share,
            ).serialize()
            await transport.send(
                elig.reply_path, payload,
                privacy=PROTOCOL_PRIVACY_CONFIGS["share_delivery"])

    # Encrypt amount_blinding for recipient
    blinding_key = derive_amount_blinding_key(sender_sk, recipient_pk)
    nonce = derive_amount_blinding_nonce(session_id)
    encrypted_blinding = aead_encrypt(
        key=blinding_key, 
        plaintext=amount_blinding,
        aad=session_id,
        nonce=nonce,  # 24-byte deterministic nonce
    )

    **Note:** This uses XChaCha20-Poly1305 (from `aead.py`'s general-purpose `aead_encrypt`), NOT ChaCha20-Poly1305 (from share encryption). The bilateral ECDH key is derived from long-term identity keys, so XChaCha20's 24-byte nonce provides extra safety against nonce collision across sessions.


21. **Write Phase 1 entry to wallet document.**
    Single doc entry at protocol_seq = base_seq + 1. All phase data
    goes in a blob; the entry carries the CID. One doc.set() call.

        # Encrypt amount_blinding for recipient
        blinding_encryption_key = ecdh(sender_sk, recipient_pk,
                                        context=b"amount_blinding")
        nonce = H("amount_blinding_nonce:" || session_id)[:12]
        encrypted_blinding = aead_encrypt(
            key=blinding_encryption_key, nonce=nonce,
            plaintext=amount_blinding, aad=session_id)

        # Pack all phase data into a single blob
        phase1_blob = msgpack.packb({
            "role": Role.SENDER.value,
            "sender_commitment": sender_commitment.serialize(),
            "sigma_s": sigma_s,
            "proof_1": proof_1,
            "public_inputs_1": public_inputs.serialize(),
            "poly_commits": [c.compress() for c in poly_commits],
            "witness_seed_1": seed_1,
            "witnesses_1": [e.node_pk for e in capped],
            "w1_reply_paths": [e.reply_path for e in capped],
            "attestations_1": [a.serialize() for a in attestations_1],
            "pairing": pairing.serialize(),
            "encrypted_amount_blinding": encrypted_blinding,
        })
        phase1_blob_cid = await storage.blob_put(phase1_blob)

        # Read previous entry hash for chain binding
        prev_entry = await wallet_doc.get(
            WalletDocSchema.entry_key(base_seq))
        previous_entry_hash = H(prev_entry) if prev_entry else b"\x00" * 32

        # Single atomic doc entry — protocol_seq = base_seq + 1
        entry = WalletDocSchema.entry_content(
            protocol_seq=base_seq + 1,
            event_type="PENDING_SEND",
            slot_commit=slot_commit,
            session_id=session_id,
            beacon_round=beacon_round_1,
            phase_data_cid=phase1_blob_cid,
            previous_entry_hash=previous_entry_hash,
        )
        await wallet_doc.set(
            WalletDocSchema.entry_key(base_seq + 1), entry)

        # Gossip propagation (parallel with doc replication)
        await broadcast_sequence_update(
            identity_commit=H(sender_pk),
            new_seq=base_seq + 1,
            owner_sk=sender_sk)

22. **Notify recipient.**
    Send TransactionInvite via MessageTransport. Includes document read
    ticket. Recipient reads finalized commitment + σ_s from document.

        read_ticket = await wallet_doc.share(
            mode=ShareMode.READ,
            key_filter=[
                WalletDocSchema.slot_status(slot_hex),
                WalletDocSchema.slot_data(slot_hex),
                WalletDocSchema.slot_lineage(slot_hex),
                WalletDocSchema.tx_state(session_hex),
                WalletDocSchema.tx_role(session_hex),
                WalletDocSchema.tx_amount_blinding(session_hex),
            ])

        invite = TransactionInvite(
            session_id    = session_id,
            sender_doc_id = wallet_doc.id(),
            sender_pk     = sender_pk,
            asset_id      = asset_id,
            amount_hint   = amount,
            read_ticket   = read_ticket.to_string(),
        )
        await transport.send(
            recipient_reply_path, invite.serialize(),
            privacy=PROTOCOL_PRIVACY_CONFIGS["tx_invite"])

**Post-conditions:**
- Source slot status = PENDING_SEND
- Oracle seq = previous + 1
- ~19 VRF-selected witnesses received IntentPayload and attested (Round 2)
- W₁ witnesses hold encrypted shares (delivered individually, step 20)
- σ_s and finalized SenderCommitment (with witness_tpk) are in wallet document
- Encrypted amount_blinding available in wallet document for recipient
- Recipient has TransactionInvite with read ticket
- ~981 remaining shard nodes saw only IntentEnvelope (no identity/asset data)

**Failure modes:**

| Failure | State after | Resolution |
|---------|------------|------------|
| Dual check fails (step 1) | READY (unchanged) | Abort. No cost. |
| Oracle increment fails (step 2) | READY (unchanged) | Retry or abort. |
| Insufficient eligibility (step 15) | PENDING_SEND (seq incremented) | Recovery via §23 (21/29 quorum). Seq increment is sunk cost. |
| Insufficient attestations (step 19) | PENDING_SEND (seq incremented) | Recovery via §23. Selected witnesses learned oracle_key but did not attest — no state change on their side. |
| Share delivery partial failure | PENDING_SEND | Acceptable if ≥13 shares delivered. Otherwise recovery. |
| Recipient never notified (step 22) | PENDING_SEND | Recovery after RECOVERY_TIMEOUT. |


---

### 20.2 Phase 2: Recipient Accepts + Commits (PENDING_RECEIVE → COMMITTED)

**Preconditions:**
- Received valid TransactionInvite via MessageTransport
- Sender slot status = PENDING_SEND (verified from sender's document)

**Steps:**

1. **Fetch sender state.**
   Join sender's document via read ticket from TransactionInvite.

       sender_doc = await storage.doc_join(invite.sender_doc_id, invite.read_ticket)
       session_hex = invite.session_id.hex()
       state_data = await sender_doc.get(WalletDocSchema.tx_state(session_hex))
       sender_state = TransactionStateRecord.deserialize(state_data)

2. **Verify phase.**
   Sender slot must be PENDING_SEND.

       slot_hex = sender_state.sender_commitment.slot_commit.hex()
       status_bytes = await sender_doc.get(WalletDocSchema.slot_status(slot_hex))
       assert SlotStatus(status_bytes.decode()) == SlotStatus.PENDING_SEND

3. **Dual revalidation.**
   Independent oracle seq check + nullifier existence check.

       current_seq = await oracle.get_seq(sender_state.sender_commitment.owner_pk_commit)
       assert current_seq in (sender_state.sender_commitment.seq - 1,
                              sender_state.sender_commitment.seq)
       assert not await nullifier_store.exists(
           sender_state.sender_commitment.nullifier_commit)

4. **Verify π₁.**

       assert await verify_zk_proof(sender_state.proof_1,
                                     sender_state.public_inputs_1)

5. **Verify σ_s over finalized commitment.**
   Recipient has the finalized SenderCommitment (with witness_tpk)
   from the sender's document.

       commitment_hash = H(sender_state.sender_commitment.serialize())
       assert Ed25519_Verify(invite.sender_pk, commitment_hash,
                              sender_state.sigma_s)

6. **Verify pairing.**
   Counterparty verification per §15.

       pairing = compute_pairing_data(recipient_sk, invite.sender_pk,
                                       invite.session_id, Role.RECIPIENT)
       assert verify_counterparty(
           sender_state.sender_commitment.recipient_pk_commit,
           H(derive_pk(recipient_sk)))

7. **Verify lineage.**
   Chain to genesis per §35.

       assert verify_lineage(sender_state.lineage_proof,
                              sender_state.sender_commitment.lineage_commit,
                              sender_state.sender_commitment.slot_commit)

8. **Compute recipient seq targets.**
   Recipient's Phase 2 entry will be at recv_base + 1. The actual
   doc write (single entry) happens in step 11a after all validation,
   proof generation, and σ_r signing.

       recipient_pk = derive_pk(recipient_sk)
       # Read current protocol_seq from recipient doc
       recv_latest = await recipient_wallet_doc.get(
           WalletDocSchema.entry_key(recv_current_seq))
       recv_base_seq = WalletDocSchema.parse_entry(recv_latest)["protocol_seq"]
       recv_phase2_seq = recv_base_seq + 1                # M+1

9. **Derive recipient slot identifiers.**
   Output slot at recv_base + SEQ_STRIDE, reserving 4 entries for
   phase transitions.

       recv_output_seq = recv_base_seq + SEQ_STRIDE     # M+4
       recv_slot_commit = H("inat_slot:" || recipient_sk || recv_output_seq.to_bytes(8))
       recv_nullifier = H("inat_nullifier:" || recipient_sk || recv_output_seq.to_bytes(8))
       recv_nullifier_commit = H(recv_nullifier)

10. **Derive recipient forward commit.**

        recipient_forward_commit = H("inat_slot:" || recipient_sk
                                      || (recv_base_seq + 2 * SEQ_STRIDE).to_bytes(8))

10b. **Decrypt amount blinding.**

        session_hex = invite.session_id.hex()
        blinding_decryption_key = ecdh(recipient_sk, invite.sender_pk,
                                        context=b"amount_blinding")
        nonce = H("amount_blinding_nonce:" || invite.session_id)[:12]
        encrypted_blinding = await sender_doc.get(
            WalletDocSchema.tx_amount_blinding(session_hex))
        amount_blinding = aead_decrypt(
            key=blinding_decryption_key, nonce=nonce,
            ciphertext=encrypted_blinding, aad=invite.session_id)

10c. **Verify amount.**
    Confirm the plaintext amount_hint matches the sender's Pedersen
    commitment using the decrypted blinding factor. Reject if mismatch.

        amount = invite.amount_hint
        assert amount is not None, "Transfer requires amount"
        assert pedersen_commit(amount, amount_blinding) == \
            sender_state.sender_commitment.amount_commit, \
            "Amount commitment mismatch"

10d. **Extract lineage data.**
    Read sender's slot lineage for CombinedWitness (Ï€â‚‚â‚ƒ constraint 13).

        slot_hex = sender_state.sender_commitment.slot_commit.hex()
        lineage_data = await sender_doc.get(
            WalletDocSchema.slot_lineage(slot_hex))
        lineage = SlotLineage.deserialize(lineage_data)
        lineage_proof = lineage.serialize()
        previous_collapsed_proof = (
            lineage.collapsed_proof.serialize()
            if lineage.origin == LineageOrigin.TRANSFER and lineage.collapsed_proof
            else None)

11. **Build RecipientCommitment and sign σ_r.**

```python
    recipient_commitment = RecipientCommitment(
        slot_commit               = recv_slot_commit,
        amount_commit             = sender_state.sender_commitment.amount_commit,
        sender_nullifier_commit   = sender_state.sender_commitment.nullifier_commit,
        sender_commitment_hash    = H(sender_state.sender_commitment.serialize()),
        session_id                = invite.session_id,
        recipient_forward_commit  = recipient_forward_commit,
    )
    sigma_r = Ed25519_Sign(recipient_sk, H(recipient_commitment.serialize()))
```

11a. **Write PENDING_RECEIVE entry to recipient wallet document.**

Single doc entry at protocol_seq = recv_base_seq + 1. All phase
data in a blob. Recipient has validated sender's proof, verified
amount commitment, signed σ_r. W₂₃ quorum NOT yet attempted.

```python
    # Pack phase data into blob
    recv_phase2_blob = msgpack.packb({
        "role": Role.RECIPIENT.value,
        "sender_commitment": sender_state.sender_commitment.serialize(),
        "recipient_commitment": recipient_commitment.serialize(),
        "sigma_s": sender_state.sigma_s,
        "sigma_r": sigma_r,
        "proof_1": sender_state.proof_1,
        "public_inputs_1": sender_state.public_inputs_1,
        "witness_seed_1": sender_state.witness_seed_1,
        "witnesses_1": sender_state.witnesses_1,
        "w1_reply_paths": sender_state.w1_reply_paths,
        "attestations_1": [a.serialize() for a in sender_state.attestations_1],
        "pairing": pairing.serialize(),
    })
    recv_phase2_cid = await storage.blob_put(recv_phase2_blob)

    prev_entry = await recipient_wallet_doc.get(
        WalletDocSchema.entry_key(recv_base_seq))
    prev_hash = H(prev_entry) if prev_entry else b"\x00" * 32

    entry = WalletDocSchema.entry_content(
        protocol_seq=recv_base_seq + 1,
        event_type="PENDING_RECEIVE",
        slot_commit=recv_slot_commit,
        session_id=invite.session_id,
        beacon_round=sender_state.beacon_round,
        phase_data_cid=recv_phase2_cid,
        previous_entry_hash=prev_hash,
    )
    await recipient_wallet_doc.set(
        WalletDocSchema.entry_key(recv_base_seq + 1), entry)

    await broadcast_sequence_update(
        identity_commit=H(recipient_pk),
        new_seq=recv_base_seq + 1,
        owner_sk=recipient_sk)
```

**Post-condition:** Recipient protocol_seq = recv_base_seq + 1.
Event type = PENDING_RECEIVE. Recoverable via §23.1 if W₂₃ fails.

12. **Fetch Phase 2 beacon.**
    Fresh beacon for seed₂₃ derivation. May differ from Phase 1 beacon.

        beacon_23 = await beacon_client.get_latest()
        beacon_round_23 = beacon_23.round
        beacon_randomness_23 = beacon_23.randomness

13. **Compute Phase 2 shard.**
    seed₂₃ hops to a DIFFERENT neighborhood. Includes H(σ_r) for
    anti-grinding (§19.6).

        seed_23 = H(sender_state.witness_seed_1
                     || H(sigma_r)
                     || beacon_round_23.to_bytes(8)
                     || beacon_randomness_23)
        shard_bits = int.from_bytes(H("inat_shard:" || seed_23)[:4], 'big')
        consensus_depth = shard_depth_consensus.compute_consensus()
        prefix = shard_bits >> (32 - consensus_depth)
        shard_topic_23 = f"inat/tx/v1/{prefix:0{consensus_depth}b}".encode()

14. **Generate π₂₃.**
    Combined circuit per §32. Proves BOTH signatures valid (σ_s and σ_r),
    bilateral binding, W₁ witness set binding, seed chain, lineage,
    forward commits.

        public_inputs_23 = CombinedPublicInputs(
            sender_commitment_hash    = H(sender_state.sender_commitment.serialize()),
            recipient_commitment_hash = H(recipient_commitment.serialize()),
            amount_commit             = sender_state.sender_commitment.amount_commit,
            nullifier_commit          = sender_state.sender_commitment.nullifier_commit,
            session_id                = invite.session_id,
            beacon_round_1            = sender_state.beacon_round,
            beacon_round_23           = beacon_round_23,
            witness_set_root_1        = sender_state.sender_commitment.witness_tpk,
            witness_seed_23           = seed_23,
            lineage_commit            = sender_state.sender_commitment.lineage_commit,
            sender_forward_commit     = sender_state.sender_commitment.sender_forward_commit,
            recipient_forward_commit  = recipient_forward_commit,
        )

        private_witness_23 = CombinedWitness(
            sender_commitment         = sender_state.sender_commitment,
            recipient_commitment      = recipient_commitment,
            sigma_r                   = sigma_r,
            recipient_sk              = recipient_sk,
            recipient_base_seq        = recv_base_seq,
            amount                    = amount,
            amount_blinding_sender    = amount_blinding,
            amount_blinding_recipient = amount_blinding,
            lineage_proof             = lineage_proof,
            previous_collapsed_proof  = previous_collapsed_proof,
            seed_1                    = sender_state.witness_seed_1,
            beacon_randomness_23      = beacon_randomness_23,
        )

        if sender_state.sender_commitment.donation:
            proof_23 = await zk_prover.prove_phase2_donation(
                public_inputs_23, private_witness_23)
        else:
            proof_23 = await zk_prover.prove_phase2(
                public_inputs_23, private_witness_23)

15. **Broadcast envelope to Phase 2 shard.**

        envelope_23 = IntentEnvelope(
            phase              = Phase.COMMIT,
            session_id         = invite.session_id,
            seed               = seed_23,
            shard_depth        = consensus_depth,
            reply_path         = transport.create_reply_path(),
            beacon_round       = beacon_round_23,
            nullifier_commit   = sender_state.sender_commitment.nullifier_commit,
            observed_density   = density_tracker.current_density,
        )
        await transport.broadcast(
            shard_topic_23, envelope_23.serialize(),
            privacy=PROTOCOL_PRIVACY_CONFIGS["phase2_envelope"])

16. **Collect W₂₃ eligibility, deliver payload, collect attestations.**
    Same two-phase flow as Phase 1 (§20.1 steps 15–19).
    Phase 2 payload includes BOTH wallet doc IDs so witnesses
    can independently verify both parties' state.

        # Round 1: collect eligibility
        raw_eligible_23 = []
        deadline = current_timestamp() + ELIGIBILITY_COLLECTION_TIMEOUT
        while len(raw_eligible_23) < WITNESS_QUORUM and current_timestamp() < deadline:
            remaining = deadline - current_timestamp()
            msg = await transport.receive(session_filter_23, timeout=remaining)
            if msg is None: break
            elig = WitnessEligibility.deserialize(msg.payload)
            if elig.node_pk in local_blacklist:                       continue
            if not vrf_verify(elig.node_pk, seed_23,
                              elig.vrf_output, elig.vrf_proof):       continue
            if elig.session_id != invite.session_id:                  continue
            raw_eligible_23.append(elig)

        if len(raw_eligible_23) < WITNESS_QUORUM:
            raise InsufficientWitnessesError(len(raw_eligible_23), WITNESS_QUORUM)

        ranked_23 = rank_witnesses_by_diversity(raw_eligible_23, seed_23)

        # Round 2: deliver payload with both doc IDs, collect attestations
        payload_23 = IntentPayload(
            session_id          = invite.session_id,
            slot_doc_id         = recipient_wallet_doc.id(),
            oracle_key          = H(recipient_pk),
            claimed_slot_commit = recv_slot_commit,
            claimed_asset_id    = sender_state.sender_commitment.asset_id,
            zk_proof            = proof_23,
            public_inputs       = public_inputs_23.serialize(),
            committed_seq       = new_recv_seq,
            sender_slot_doc_id  = invite.sender_doc_id,
            sender_oracle_key   = sender_state.sender_commitment.owner_pk_commit,
        )

        for elig in ranked_23[:WITNESS_TOTAL_CAP]:
            await transport.send(
                elig.reply_path, payload_23.serialize(),
                privacy=PROTOCOL_PRIVACY_CONFIGS["intent_payload"])

        attestation_responses_23 = []
        deadline = current_timestamp() + PAYLOAD_ATTESTATION_TIMEOUT
        while len(attestation_responses_23) < WITNESS_QUORUM and current_timestamp() < deadline:
            remaining = deadline - current_timestamp()
            msg = await transport.receive(session_filter_23, timeout=remaining)
            if msg is None: break
            response = WitnessResponse.deserialize(msg.payload)
            if not response.attestation.verify():                     continue
            if response.attestation.session_id != invite.session_id:  continue
            attestation_responses_23.append(response)

        if len(attestation_responses_23) < WITNESS_QUORUM:
            raise InsufficientWitnessesError(len(attestation_responses_23), WITNESS_QUORUM)

        attestations_23 = [r.attestation for r in attestation_responses_23[:WITNESS_QUORUM]]

17. **Write COMMITTED entry for recipient.**

Single doc entry at protocol_seq = recv_base_seq + 2. W₂₃ quorum
achieved. Point of no return.

```python
    committed_blob = msgpack.packb({
        "proof_23": proof_23,
        "public_inputs_23": public_inputs_23.serialize(),
        "witness_seed_23": seed_23,
        "witnesses_23": [r.node_pk for r in ranked_23[:WITNESS_QUORUM]],
        "attestations_23": [a.serialize() for a in attestations_23],
    })
    committed_cid = await storage.blob_put(committed_blob)

    prev_entry = await recipient_wallet_doc.get(
        WalletDocSchema.entry_key(recv_base_seq + 1))

    entry = WalletDocSchema.entry_content(
        protocol_seq=recv_base_seq + 2,
        event_type="COMMITTED",
        slot_commit=recv_slot_commit,
        session_id=invite.session_id,
        beacon_round=beacon_round_23,
        phase_data_cid=committed_cid,
        previous_entry_hash=H(prev_entry),
    )
    await recipient_wallet_doc.set(
        WalletDocSchema.entry_key(recv_base_seq + 2), entry)

    await broadcast_sequence_update(
        identity_commit=H(recipient_pk),
        new_seq=recv_base_seq + 2,
        owner_sk=recipient_sk)
```

**Post-condition:** Recipient protocol_seq = recv_base_seq + 2.
Point of no return. Finalization is inevitable.

18. **Notify sender of W₂₃ quorum.**
    Send attestation bundle so sender can notify W₁ witnesses.

        w23_bundle = W23AttestationBundle(
            session_id   = invite.session_id,
            attestations = attestations_23,
            proof_23     = proof_23,
        )
        await transport.send(
            sender_reply_path, w23_bundle.serialize(),
            privacy=PROTOCOL_PRIVACY_CONFIGS["w23_notification"])

19. **Sender forwards W₂₃ bundle to each W₁ witness.**
    (Executed by sender upon receiving W₂₃ notification.)
    Uses reply_paths stored from Phase 1 (§20.1 step 19 w1_reply_paths).

        for w1_reply_path in sender_state.w1_reply_paths:
            await transport.send(
                w1_reply_path, w23_bundle.serialize(),
                privacy=PROTOCOL_PRIVACY_CONFIGS["w23_notification"])

    W₁ witnesses verify bundle → decrypt share → publish → Phase 3.

20. **Recipient independently forwards W₂₃ bundle if sender is unresponsive.**
    The recipient holds all data required for this step: the W₂₃
    attestation bundle (generated in step 16) and w1_reply_paths
    (readable from the sender's wallet document via the read ticket
    obtained in step 1).

    If Phase 3 share publication has not begun within
    W1_FORWARD_TIMEOUT seconds of W₂₃ quorum, the recipient
    SHOULD independently forward the bundle to all W₁ witnesses:

        # Recipient reads W₁ reply paths from sender's wallet doc
        tx_state_data = await sender_doc.get(
            WalletDocSchema.tx_state(session_hex))
        sender_tx_state = TransactionStateRecord.deserialize(tx_state_data)

        for w1_reply_path in sender_tx_state.w1_reply_paths:
            await transport.send(
                w1_reply_path, w23_bundle.serialize(),
                privacy=PROTOCOL_PRIVACY_CONFIGS["w23_notification"])

    **Correctness:** The W₂₃ bundle is self-authenticating (all
    attestation signatures verifiable by W₁ witnesses independently
    of who delivers it). W₁ witnesses perform identical verification
    regardless of whether the bundle arrives from the sender or
    recipient. No new trust assumption is introduced.

    **Privacy:** The recipient already knows the sender's doc_id
    (from the TransactionInvite) and w1_reply_paths are opaque
    reply paths — no network identity is leaked by this step.

    W1_FORWARD_TIMEOUT = 120  # seconds after W₂₃ quorum

**Post-conditions:**
- Sender slot status = COMMITTED
- Recipient slot status = COMMITTED (new slot at recv_seq)
- Recipient oracle seq = previous + 1
- W₂₃ quorum achieved — abort is impossible
- W₁ witnesses notified — share release imminent (sender primary, recipient fallback after W1_FORWARD_TIMEOUT)
- Finalization is inevitable; does not depend on sender remaining online after W₂₃ quorum

**Failure modes:**

| Failure | State after | Resolution |
|---------|------------|------------|
| π₁ verification fails (step 4) | No state change | Abort. No cost to recipient. |
| σ_s verification fails (step 5) | No state change | Abort. Sender may be malicious. |
| Lineage invalid (step 7) | No state change | Abort. |
| Amount commitment mismatch (step 10c) | No state change | Abort. Sender may be malicious. |
| Insufficient W₂₃ eligibility (step 16) | **PENDING_RECEIVE** (recipient seq incremented) | Recovery via §23.1 (21/29 quorum). Sender: recovery after timeout. |
| Insufficient W₂₃ attestations (step 16) | **PENDING_RECEIVE** (recipient seq incremented) | Recovery via §23.1. |
| Sender never receives W₂₃ notification (step 18) | Both COMMITTED | Resumption (§23.2): recipient retries. |


### 20.3 Phase 3: Share Release + Nullifier (COMMITTED → SPENT)


```python
def _extract_forward_commits(session: 'TransactionStateRecord') -> List[bytes]:
    commits = [session.recipient_commitment.recipient_forward_commit]
    # Sender output always exists (sender gets remaining balance)
    commits.append(session.sender_commitment.sender_forward_commit)
    return commits
```

```python
def _extract_output_commits(session: 'TransactionStateRecord') -> List[bytes]:
    """Both outputs always exist: recipient + sender remainder."""
    return [
        session.recipient_commitment.slot_commit,
        session.sender_commitment.sender_output_commit,
    ]
```

**Trigger:** W₁ witnesses observe W₂₃ quorum (≥WITNESS_QUORUM valid Phase 2 attestations).

**W₁ witness behavior upon observing W₂₃ quorum:**

1. Decrypt own share.
2. Verify share against polynomial commitments (Feldman).
3. Publish share as content-addressed blob.
4. Broadcast share CID on shares_shard_topic(seed_1, shard_depth).

**Nullifier reconstruction** (permissionless among participants - any node with read access to the transaction's wallet document can execute):

1. Collect ≥WITNESS_QUORUM published shares from shares_shard_topic(seed_1, shard_depth).
   Reconstructor must know seed_1 (available from SpendRecord lineage
   or wallet document). Read access is available to: the sender, the recipient, all W₁ witnesses (via read tickets from Phase 1), and any node they shared access with. seed₁ is derivable from public data in the wallet document (slot_commit, beacon_round, beacon_randomness). No special role is required beyond document read access.
2. Feldman-verify each share against polynomial commitments.
3. Lagrange-interpolate nullifier from ≥WITNESS_QUORUM shares.
4. Verify: H(reconstructed_nullifier) = nullifier_commit from Phase 1.
5. Publish nullifier via NullifierStore (§40).


```python
# ShareReleaseClaim IS the Phase 3 attestation.
# Rename for clarity and add to AttestationSummary.

@dataclass
class ShareReleaseAttestation:
    """Phase 3 attestation: W₁ witness confirms share release."""
    session_id: bytes
    witness_index: int
    share_cid: bytes                # CID of published share blob
    nullifier_commit: bytes
    publisher_pk: bytes
    beacon_round: int
    signature: bytes
```

**SpendRecord construction:**

Build SpendRecord (§49.3) containing:
- Consumed slot: spent_commit, nullifier_commit
- Output slots: output_commits (recipient + change), forward_commits
- Bilateral attestation: SenderCommitment, RecipientCommitment, σ_s, σ_r, π₂₃
- Witness evidence: witness_root (Merkle of first 13 witness PKs), witness_bundle_cid
- Lineage: previous_spend_cid, depth, issuer_family_id

### SpendRecord Field Derivation Rules

**output_commits:**

    output_commits = [
        recipient_slot_commit,      # H("inat_slot:" || recipient_sk || recv_seq)
        sender_output_commit,       # H("inat_slot:" || sender_sk || new_seq)
    ]

Both outputs always exist. The sender output carries the remaining
balance (input - amount - fee). If sender_output_amount == 0, the
slot still exists as a zero-balance slot.

**forward_commits:**

    forward_commits = [
        recipient_commitment.recipient_forward_commit,
        sender_commitment.sender_forward_commit,
    ]

Forward commits bind each SpendRecord output to the owner's next
predetermined slot. They are carried in the commitment structures:
sender_forward_commit in SenderCommitment (§20.1 step 6),
recipient_forward_commit in RecipientCommitment (§20.2 step 10).
Each proof (π₁, π₂₃) takes the forward commit as a public input
and verifies it matches H("inat_slot:" || owner_sk || (base_seq + 2 * SEQ_STRIDE))
without revealing sk — placing it SEQ_STRIDE positions past the
output slot to reserve space for the next transaction's phase
entries. The SpendRecord reads them from the commitment structures.

**depth:**

    if lineage.origin == MINT:
        depth = 1
    else:
        depth = lineage.collapsed_proof.depth + 1

**previous_spend_cid:**

    if lineage.origin == MINT:
        previous_spend_cid = None    # fold circuit uses genesis_cid
    elif lineage.origin == TRANSFER:
        previous_spend_cid = lineage.spend_record_cid

**issuer_family_id:**

    if lineage.collapsed_proof exists:
        issuer_family_id = lineage.collapsed_proof.issuer_family_id
    else:
        issuer_family_id = sender_commitment.asset_id

Consistent across the entire lineage chain.

**witness_root:**

    witness_root = Merkle(first WITNESS_QUORUM witness PKs, ordered by attestation timestamp)

Only the first WITNESS_QUORUM witnesses (by timestamp) are included in the
Merkle root. These are the fee recipients. Additional attestations
are stored in the WitnessBundle but not in the root.

**WitnessBundle construction order:**

1. Collect all Phase 1 and Phase 2 attestations.
2. Sort all attestations by timestamp ascending.
3. First WITNESS_QUORUM (21) attestations → project via
   `att.to_fee_attestation()` → fee_attestations: List[FeeAttestation].
4. Remaining → additional_attestations: List[Attestation].
5. Compute witness_root from fee_attestations.
6. Publish SpendRecord as blob → obtain spend_cid.
   SpendRecord contains witness_root (Merkle of fee-earning PKs)
   but does NOT contain witness_bundle_cid. No circular dependency.
7. Create WitnessBundle with spend_record_cid = spend_cid.
8. Publish WitnessBundle as blob → obtain witness_bundle_cid.

Dependency is one-directional: WitnessBundle → SpendRecord.
SpendRecord is self-contained. The fold circuit binds to
witness_root from SpendRecord; the WitnessBundle is
supplementary evidence for fee claims and auditing.


```python
# In finalize_phase3, after collecting attestations:

if session.sender_commitment.donation:
        witness_bundle = WitnessBundle(
            ...
            witness_count=WITNESS_QUORUM + 1,
            pool_pk=HARDCODED_POOL_PK,
        )
        # witness_root includes quorum real + 1 ghost
    else:
        witness_bundle = WitnessBundle(
            ...
            witness_count=WITNESS_QUORUM,
            pool_pk=None,
        )
```

### 20.3.1 — Donation receipt publication (donation only)


```python
# After SpendRecord is published, if donation:
if session.sender_commitment.donation:
    receipt = DonationReceipt(
        spend_record_cid=spend_cid,
        donation_commit=session.sender_commitment.fee_commit,
        issuer_family_id=issuer_family_id,
        epoch=current_epoch(),
    )
    inbox_key = H(b"inat_donation_inbox:" +
                  HARDCODED_POOL_PK + issuer_family_id)
    await push_to_responsible_peers(
        iroh_node, inbox_key, receipt.serialize(),
        peer_count=DONATION_RECEIPT_RESPONSIBLE_PEERS,
        privacy=DONATION_PRIVACY_CONFIG["inat.donation.store"])
```


**Verification rule:** SpendRecord.verify_binding() MUST hold:
- session_ids match between sender and recipient commitments
- recipient.sender_nullifier_commit = sender.nullifier_commit
- recipient.amount_commit = sender.amount_commit
- recipient.sender_commitment_hash = H(sender_commitment)
- len(output_commits) > 0
- len(output_commits) = len(forward_commits)

**Post-condition:** Sender slot status = SPENT. Nullifier is published.

**Stall handling:** If shares remain unavailable for 72 hours after
COMMITTED, Forced Finalization (§22) activates.


### 20.4 Phase 4: DA Confirmation + Fold (SPENT → FINALIZED)

**Preconditions:**
- Sender slot status = SPENT
- Nullifier published (Phase 3 complete)
- SpendRecord published as content-addressed blob

**Steps:**

1. **Fetch Phase 4 beacon.**

       beacon_4 = await beacon_client.get_latest()
       beacon_round_4 = beacon_4.round
       beacon_randomness_4 = beacon_4.randomness

2. **Compute Phase 4 shard.**
   seed₄ per §19.6 (third shard hop). Uses spend_cid as entropy.

       seed_4 = H(seed_23 || spend_cid || beacon_round_4.to_bytes(8)
                  || beacon_randomness_4)
       shard_bits = int.from_bytes(H("inat_shard:" || seed_4)[:4], 'big')
       consensus_depth = shard_depth_consensus.compute_consensus()
       prefix = shard_bits >> (32 - consensus_depth)
       shard_topic_4 = f"inat/tx/v1/{prefix:0{consensus_depth}b}".encode()

3. **Broadcast DA check request.**
   IntentEnvelope (Phase CONFIRM) to Phase 4 shard.

       envelope_4 = IntentEnvelope(
           phase            = Phase.CONFIRM,
           session_id       = session_id,
           seed             = seed_4,
           shard_depth      = consensus_depth,
           reply_path       = transport.create_reply_path(),
           beacon_round     = beacon_round_4,
           nullifier_commit = nullifier_commit,
           observed_density = density_tracker.current_density,
       )
       await transport.broadcast(
           shard_topic_4, envelope_4.serialize(),
           privacy=PROTOCOL_PRIVACY_CONFIGS["phase4_envelope"])

4. **Collect W₄ eligibility and deliver payload.**
   Same two-phase flow as earlier phases. W₄ target: 7 validators.

       # Round 1: collect VRF-eligible witnesses
       # Round 2: deliver IntentPayload with spend_cid + nullifier_cid

       payload_4 = IntentPayload(
           session_id      = session_id,
           slot_doc_id     = wallet_doc.id(),
           oracle_key      = H(sender_pk),
           claimed_slot_commit = slot_commit,
           claimed_asset_id = asset_id,
           spend_cid       = spend_cid,
           nullifier_cid   = nullifier_cid,
       )

5. **W₄ witness verification.**
   Each W₄ witness independently:
   b. Fetches SpendRecord by spend_cid from content store.
   a. If SpendRecord.is_forced == false:
        Fetches nullifier blob by nullifier_cid from content store.
        Verifies H(fetched_nullifier) = nullifier_commit from SpendRecord.
      If SpendRecord.is_forced == true:
        Fetches ForcedFinalizationRecord by forced_finalization_cid
        from content store.
        Verifies ForcedFinalizationRecord.nullifier_commit ==
        SpendRecord.nullifier_commit.
        Verifies ForcedFinalizationRecord contains ≥13 valid W_FF
        attestations.
        NOTE: No preimage exists or is required. The W_FF quorum
        and Π₂₃ together constitute sufficient spend evidence.
   d. Verifies SpendRecord.verify_binding() (§49.3).
   e. Verifies bilateral signatures (σ_s, σ_r) in SpendRecord.
   f. Returns Confirmation with status DA_AVAILABLE, DA_UNAVAILABLE,
      or DA_INVALID.

6. **Collect 7 DA confirmations.**

       confirmations = []
       deadline = current_timestamp() + PAYLOAD_ATTESTATION_TIMEOUT
       while len(confirmations) < DA_VALIDATORS and current_timestamp() < deadline:
           msg = await transport.receive(session_filter_4, timeout=remaining)
           if msg is None: break
           conf = Confirmation.deserialize(msg.payload)
           if conf.status != DA_AVAILABLE: continue
           if not verify_signature(conf.validator_pk, conf.payload(), conf.signature):
               continue
           confirmations.append(conf)

       if len(confirmations) < DA_VALIDATORS:
           raise InsufficientConfirmationsError(len(confirmations), DA_VALIDATORS)

7. **Fold.**
   Generate CollapsedProof per §33. v1: attested hash-chain.

       collapsed = await zk_prover.fold(
           previous_fold   = source_slot.lineage.collapsed_proof,
           spend_record    = spend_record,
           phase_proofs    = (proof_1, proof_23),
           confirmations   = confirmations,
           witness_set_root = witness_root,
           beacon_round    = beacon_round_4,
       )

8. **Write FINALIZED entries.**
   Sender: entry at base_seq+4. Recipient: entry at recv_base_seq+4.
   Output slots materialize. Each is a single doc.set().

   NOTE: Sender's SPENT entry (base_seq+3) was written during Phase 3
   when the nullifier was published and SpendRecord was created.

       # Sender FINALIZED entry (base_seq + 4)
       finalized_blob = msgpack.packb({
           "collapsed_proof_cid": collapsed_proof_cid,
           "confirmation_root": confirmation_root,
           "spend_record_cid": spend_cid,
       })
       finalized_cid = await storage.blob_put(finalized_blob)

       prev_entry = await wallet_doc.get(
           WalletDocSchema.entry_key(base_seq + 3))
       entry = WalletDocSchema.entry_content(
           protocol_seq=base_seq + SEQ_STRIDE,
           event_type="FINALIZED",
           slot_commit=sender_output_commit,
           session_id=session_id,
           beacon_round=beacon_round_4,
           phase_data_cid=finalized_cid,
           previous_entry_hash=H(prev_entry),
       )
       await wallet_doc.set(
           WalletDocSchema.entry_key(base_seq + SEQ_STRIDE), entry)

       # Recipient FINALIZED entry (recv_base_seq + 4)
       # (written by recipient on their own wallet doc)
       # Output slot is now READY with collapsed proof as lineage

**Post-conditions:**
- Sender slot status = FINALIZED (terminal)
- Recipient slot status = READY (new spendable slot)
- CollapsedProof available for O(1) third-party verification
- SpendRecord and nullifier are DA-confirmed by 7 validators

**Failure modes:**

| Failure | State after | Resolution |
|---------|------------|------------|
| Insufficient DA confirmations | SPENT (unchanged) | Retry with fresh W₄ selection. |
| SpendRecord blob unavailable | SPENT | Re-publish SpendRecord. Retry. |
| Fold generation fails | SPENT | Retry fold. No state rollback needed. |

---


## 21. Witness Attestation (self-selection + direct reply)

### Node Startup

# Node subscribes to shards for each whitelisted asset
async def start_inat_node(secret_key, admission_policy):
    for asset_id in admission_policy.accepted_issuers:
        depth = await get_asset_depth(asset_id)
        my_shard = derive_my_shard(node_id, asset_id, depth)
        await subscribe(my_shard)
        await subscribe(witness_heartbeat_topic(asset_id))
        # Subscribe to registry updates for this asset
        asset_prefix = asset_id[:4].hex()
        await subscribe(f"inat/registry/v1/{asset_prefix}".encode())

A conforming Inat node MUST, on startup:

1. **Determine shard membership**: Compute own shard prefix
   from node_id and current consensus depth.

       node_hash = H("inat_shard:" || node_id)
       prefix = uint32(node_hash[:4]) >> (32 - depth)
       shard_topic = "inat/tx/v1/{prefix as binary}"

2. **Subscribe to shard topic**: Listen for IntentEnvelope
   messages on own shard_topic. Process per §21.1.

3. **Subscribe to oracle topic**: Listen for SequenceUpdate
   messages on INAT_ORACLE_TOPIC. Process per §45.8.

4. **Subscribe to shares topics dynamically**: When participating
   in a session (as W₁ witness or as a node performing Phase 3
   reconstruction), subscribe to shares_shard_topic(seed_1, shard_depth)
   for that session. Unsubscribe after session completes or times out.

   Nodes do NOT subscribe to a global shares topic. Share
   publications are scoped to the Phase 1 shard neighborhood.

5. **Start heartbeat**: Broadcast ShardHeartbeat (§19.2) to
   own shard topic every DENSITY_HEARTBEAT_INTERVAL.

6. **Start Pulse loop**: Execute anti-entropy exchange (§45.9)
   every PULSE_INTERVAL. Additionally track shard density and
   update consensus depth.

7. **Resubscribe on depth change**: If consensus depth changes,
   recompute shard prefix and re-subscribe to new shard topic.
   Unsubscribe from old shard topic.

**Gossip topic summary:**

| Topic | Content | Subscribe at |
|-------|---------|-------------|
| inat/tx/v1/{prefix} | IntentEnvelope (all phases) | Startup + depth change |
| inat/oracle/v1 | SequenceUpdate | Startup |
| inat/shares/v1/{prefix} | Published share CIDs | Per-session (W₁ participants + reconstructors) |
| inat/registry/v1/{asset_prefix} | RegistryAnnouncement (new AssetKeyRegistry CID) | Startup (per accepted asset) |

There is no gossip topic for eligible headers. Each wallet
receives its own header directly from the issuer. Witnesses
carry headers inline in WitnessEligibility responses.

---

### 21.1 Attestation Lifecycle (Two-Phase)

**Round 1: Eligibility (on IntentEnvelope via shard gossip)**

Upon receiving an IntentEnvelope via shard gossip subscription:

1. **Shard depth validation**: Reject if `|envelope.shard_depth - consensus_depth| > 1`.
2. **Shard membership**: Reject if `derive_shard_topic(envelope.seed, envelope.shard_depth) ≠ my_shard_topic`.
3. **Beacon freshness**: Reject if `current_round - envelope.beacon_round > MAX_BEACON_AGE_ROUNDS`.
4. **VRF self-selection**: Evaluate lottery per §19.4. Exit silently if not selected.
5. **Reply with eligibility**: Send WitnessEligibility directly to envelope.reply_path.

        # self-selection threshold uses THIS NODE's local heartbeat density.
        # observed_density is included as a hint for sender-side sanity
        # filtering only — it is never used by any party for threshold math.
        # Verify own eligibility before responding
        if not self._has_eligible_wallet(envelope.asset_id):
            return  # Not eligible for this asset — silent exit

        eligibility = WitnessEligibility(
            session_id       = envelope.session_id,
            node_pk          = self.node_pk,
            vrf_output       = vrf_output,
            vrf_proof        = vrf_proof,
            reply_path       = transport.create_reply_path(),
            observed_density = density_tracker.current_density,
            eligibility_proof = self._get_eligibility_proof(envelope.asset_id),
        )
        await transport.send(
            envelope.reply_path, eligibility.serialize(),
            privacy=PROTOCOL_PRIVACY_CONFIGS["witness_eligibility"])

**Round 2: Attestation (on IntentPayload via direct delivery)**

Upon receiving an IntentPayload via reply_path:

6. **Session match**: Reject if payload.session_id doesn't match a pending eligibility.
7. **Verification pipeline**: Execute full checks in §21.2 using payload fields.
8. **Reply with attestation**: Send WitnessResponse to sender's reply_path.

**Privacy property:** Only ~19 VRF-selected witnesses receive the
IntentPayload. The remaining ~981 shard nodes see only the
IntentEnvelope, which contains no identity, asset, or proof data.


### Sender-Side Response Validation

**Round 1 — WitnessEligibility validation:**

| # | Check | Reject if |
|---|-------|-----------|
| 1 | Blacklist | elig.node_pk ∈ local_blacklist |
| 2 | VRF proof | vrf_verify(node_pk, seed, output, proof) fails |
| 3 | VRF threshold | uint256(output) ≥ computed threshold |
| 4 | Density sanity filter | \|observed_density - local_density\| > 20% of local_density. Catches stale/wrong-shard nodes. NOT used for VRF threshold — threshold always uses verifier's own local_density. |
| 5 | Session match | elig.session_id ≠ expected session_id |

Collection terminates when ≥WITNESS_TOTAL_CAP valid eligibility
responses are received, or ELIGIBILITY_COLLECTION_TIMEOUT expires.

**Round 2 — WitnessResponse validation (post-payload):**

| # | Check | Reject if |
|---|-------|-----------|
| 1 | Attestation signature | attestation.verify() fails |
| 2 | Session match | attestation.session_id ≠ expected session_id |
| 3 | Known witness | response.node_pk not in selected set |

Collection terminates when ≥WITNESS_QUORUM valid attestations
are received, or PAYLOAD_ATTESTATION_TIMEOUT expires.


### 21.2 Verification Pipeline

All checks below execute in Round 2, after the witness receives
IntentPayload. Round 1 (IntentEnvelope) performs only: shard
depth/membership validation, beacon freshness, and VRF self-selection.

Checks are ordered fail-fast. Expensive operations execute
only after cheap checks pass.

**Phase A — Independent Fetch & Cheap Checks (~60-110ms):**

| # | Check | Source | Fail condition |
|---|-------|--------|----------------|
| A0a | Asset support | self.admission_policy | asset_id ∉ accepted_issuers |
| A0b | Own eligibility | Local eligible root cache | This witness not in current IssuerEligibleRoot for asset_id |
| A0c | Own balance | Local wallet | Balance < MIN_WITNESS_BALANCE for asset_id |
| A1 | Beacon freshness | drand | current_round - envelope.beacon_round > MAX_BEACON_AGE_ROUNDS (checked in Round 1) |
| A2 | Fetch slot state | Wallet document(s) (independent) | Document unreachable |
| A3 | Fetch sequence | Oracle (independent) | Oracle unreachable |
| A4 | Nullifier check | NullifierStore | Nullifier already exists |
| A5 | Terminal state | Fetched slot status | Status ∈ TERMINAL_STATES (Phase 1, 2 only) |
| A6 | Session binding | Fetched document | `doc[tx:{session_id}:state]` exists AND `doc[tx:{session_id}:role]` matches expected role AND beacon_round is within MAX_BEACON_AGE_ROUNDS of current round |
| A7 | Asset consistency | Fetched document | `doc[slot:{slot_commit}:data].asset_id == payload.claimed_asset_id` |
| A8 | Cross-check: fetched vs ZK-bound | Fetched state vs public inputs | ALL of: (a) fetched slot_commit == payload.claimed_slot_commit, (b) fetched asset_id == public_inputs.asset_id, (c) fetched oracle seq is consistent with phase-specific check (§21.3), (d) fetched slot status is consistent with expected phase |
| A9 | Sequence check | See §21.3 | Sequence violation |

**CRITICAL**: Witnesses MUST independently fetch all mutable state
(slot status, sequence number, nullifier existence) from the
storage layer. Witnesses MUST NOT trust requester-supplied values
for any mutable state. If any fetch fails, the attestation MUST
be rejected. All checks use fields from the IntentPayload
(received in Round 2), NOT the IntentEnvelope (Round 1).


Phase 2 witnesses performing dual document fetch MUST check:
- Sender slot: status = PENDING_SEND (not terminal, not READY)
- Recipient slot: status = PENDING_RECEIVE (not terminal, not READY)


**Cross-binding verification:** Phase 2 witnesses MUST verify
that both pending states reference each other:

| Check | Source | Binding |
|-------|--------|---------|
| Recipient → Sender | RecipientCommitment (recipient doc) | `sender_commitment_hash == H(sender_commitment)` fetched from sender doc |
| Recipient → Sender | RecipientCommitment (recipient doc) | `sender_nullifier_commit == sender_commitment.nullifier_commit` |
| Sender → Recipient | SenderCommitment (sender doc) | `recipient_pk_commit == H(recipient_pk)` consistent with recipient doc owner |
| Session match | Both docs | `sender tx_state.session_id == recipient tx_state.session_id` |

These bindings are already proven inside π₂₃ (§32 constraints
7, 8), but witnesses MUST verify them from fetched document
state as a mutable-state cross-check. A PENDING_RECEIVE that
does not reference the correct PENDING_SEND (or vice versa)
indicates a stale or forged document and MUST be rejected.

If recipient slot is still READY (recipient hasn't persisted
PENDING_RECEIVE yet), the witness MAY retry after a short delay
(propagation tolerance) or reject.

Each party binds to counterparty with a hash!

**Phase B — Expensive Crypto (~200ms, only if Phase A passes):**

| # | Check | Fail condition |
|---|-------|----------------|
| B1 | ZK proof verification | Proof invalid |
| B1b | σ_s signature (Phase 2 only) | Ed25519_Verify(sender_pk, H(sender_commitment), σ_s) fails. sender_pk and σ_s fetched from sender wallet document. |
| B1c | σ_r signature (Phase 2 only) | Ed25519_Verify(recipient_pk, H(recipient_commitment), σ_r) fails. recipient_pk and σ_r fetched from recipient wallet document. |
| B2 | Phase-specific check | See §21.4 |

NOTE: σ_s and σ_r are verified natively (not inside ZK) because
Ed25519 signature verification over Curve25519 inside BN254
circuits requires non-native field arithmetic (~200-400k
constraints per signature). Native verification costs ~microseconds.
The ZK circuit (π₂₃) proves commitment structure, bilateral
binding, and proof-of-knowledge of both parties' secret keys.
Signature validity is witness-attested and fold-bound (§33.1).

### 21.3 Per-Phase Sequence Checks


```python
@staticmethod
def check_sequence_phase1(expected_seq: int, fetched_seq: int):
    """
    Sender claims expected_seq = N+1 (just incremented).
    Oracle may return N (not yet propagated) or N+1 (propagated).
    
    Valid:   fetched_seq == expected_seq     → oracle caught up
    Valid:   fetched_seq == expected_seq - 1 → propagation delay
    Invalid: fetched_seq > expected_seq      → seq advanced past claim (stale/replay)
    Invalid: fetched_seq < expected_seq - 1  → claimed seq too far ahead
    """
    if fetched_seq == expected_seq:
        return  # Oracle propagated
    if fetched_seq == expected_seq - 1:
        return  # Propagation delay — acceptable
    if fetched_seq > expected_seq:
        raise SequenceCheckError(
            f"Oracle seq {fetched_seq} > expected {expected_seq}: "
            f"slot already advanced past this transaction")
    raise SequenceCheckError(
        f"Oracle seq {fetched_seq} too far behind expected {expected_seq}: "
        f"suspicious gap")
```

**Phase 2 check — no further increment since Phase 1:**

```python
@staticmethod
def check_sequence_phase2(committed_seq: int, fetched_seq: int):
    """
    committed_seq was set during Phase 1 (= N+1).
    No increment happens between Phase 1 and Phase 2 for same slot.
    
    Valid:   fetched_seq == committed_seq     → consistent
    Valid:   fetched_seq == committed_seq - 1 → Phase 1 propagation still delayed
    Invalid: fetched_seq > committed_seq      → another tx incremented (conflict)
    Invalid: fetched_seq < committed_seq - 1  → impossible gap
    """
    if fetched_seq == committed_seq:
        return
    if fetched_seq == committed_seq - 1:
        return  # Phase 1 increment still propagating
    if fetched_seq > committed_seq:
        raise SequenceCheckError(
            f"Oracle seq {fetched_seq} > committed {committed_seq}: "
            f"slot was used in another transaction")
    raise SequenceCheckError(
        f"Oracle seq {fetched_seq} too far behind committed {committed_seq}")
```

| Phase | Doc | Rule | Rationale |
|-------|-----|------|-----------|
| Phase 1 | Sender | fetched_seq ∈ {base_seq, base_seq+1} | Sender wrote entry at base+1. May not have propagated yet (returns base) or has (returns base+1). fetched > base+1 → conflict. |
| Phase 2 | Sender | fetched_seq ∈ {base_seq+1, base_seq+2} | Sender may have written COMMITTED entry (base+2). fetched > base+2 → conflict. |
| Phase 2 | Recipient | fetched_seq ∈ {recv_base, recv_base+1, recv_base+2} | Recipient wrote PENDING_RECEIVE (base+1) and may have written COMMITTED (base+2). |

NOTE: With per-phase seq increments, the acceptable range is wider
because each party advances seq at every state transition. The
fundamental invariant is: fetched_seq must be consistent with the
transaction's progress — not behind the entry that should exist,
not ahead due to a conflicting transaction.

### 21.4 Phase-Specific Behavior

**Phase 1 (W₁):**
- Round 1: Respond with WitnessEligibility (VRF proof only).
- Round 2: Receive IntentPayload. Execute verification pipeline (§21.2). Attest.
- Await encrypted share delivery from sender (timeout: SHARE_DELIVERY_TIMEOUT).
- Store encrypted share locally.
- Await W₂₃ quorum notification from sender via MessageTransport.
    Timeout: RECOVERY_TIMEOUT. On timeout: discard stored share.


**Failure cases:**

| Condition | Behavior |
|-----------|----------|
| Share delivery times out (SHARE_DELIVERY_TIMEOUT) | Discard session context. No share to release. Session is abandoned from this witness's perspective. |
| W₂₃ quorum never observed AND session expires | Discard stored share. Session expiry is determined by: `current_time - session_created_at > RECOVERY_TIMEOUT` |
| W₂₃ quorum observed but share decryption fails | Log error. Do not publish. Other W₁ witnesses may still provide 13 shares. |
| Feldman verification fails after decryption | Do not publish. Sender may have distributed corrupted shares. |

**Phase 2 (W₂₃):**
- Round 1: Respond with WitnessEligibility (VRF proof only).
- Round 2: Receive IntentPayload. Execute verification pipeline (§21.2).
- Verify π₂₃ (which internally proves seed chain validity per §32 constraint 10).
- Verify seed₂₃ matches the shard this witness is in:
  `derive_shard_topic(payload.public_inputs.witness_seed_23, shard_depth) == my_shard_topic`.
- Fetch BOTH wallet documents (sender's + recipient's). Verify non-terminal.
- Attest.

NOTE: W₂₃ witnesses do NOT independently recompute the seed chain.
seed_1 is not available to them. The ZK proof guarantees seed_23
was correctly derived from seed_1, H(σ_r), and the beacon. The
witness only verifies that seed_23 corresponds to their shard
and that π₂₃ is valid.

**Phase 4 (W₄):**
- Verify data availability: spend_cid and nullifier_cid are retrievable from content store.
- Return Confirmation with status DA_AVAILABLE, DA_UNAVAILABLE, or DA_INVALID.

### 21.5 W₁ Share Release

**Trigger:** W₁ witness receives W₂₃AttestationBundle from sender containing ≥WITNESS_QUORUM valid Phase 2 attestations.

**Release procedure:**
1. Verify attestation set (all signatures valid, all from Phase 2, correct session_id).
2. Decrypt own share.
3. Verify share against polynomial commitments (Feldman VSS §10).
4. Publish verified share as content-addressed blob.
5. Broadcast share CID on INAT_SHARES_TOPIC.

**Expiry:** If session expires before W₂₃ quorum, discard share.

### 21.6 Attestation Response

Attestation payload (signed by witness):

| Field | Type | Size |
|-------|------|------|
| session_id | bytes | 32 |
| phase | Phase | 1 |
| nullifier_commit | bytes | 32 |
| beacon_round | uint64 | 8 |
| timestamp | uint64 | 8 |

WitnessResponse (sent to sender):

| Field | Type |
|-------|------|
| attestation | Attestation |
| vrf_output | bytes(32) |
| vrf_proof | bytes(var) |
| node_pk | bytes(32) |
| reply_path | bytes(var) |
| observed_density | uint32 |

### 21.7 Post-Attestation

After attesting, witnesses SHOULD propagate the witnessed state
to the wallet's home shard (REDUNDANCY_TARGET peers) for
durability. This is a liveness optimization, not a correctness
requirement.

### 21.8 Share Distribution

After collecting attestations and assigning indices (§19.7),
the sender delivers encrypted shares individually to each W₁
witness via MessageTransport.

**Share delivery payload:**

| Field | Type | Size |
|-------|------|------|
| session_id | bytes | 32 |
| witness_index | uint16 | 2 |
| encrypted_share | bytes | var |

Encrypted share wire format per §10:
`ephemeral_pk (32) || nonce (12) || AEAD ciphertext`


### Verification Checks (both W₁ and W₂₃)

Executed in Round 2 only, after receiving IntentPayload. Round 1
(IntentEnvelope) performs only shard/VRF/beacon validation.

Session binding (IDs match, beacon valid), roles correct, pairing bind match, proofs valid, asset match, no double-spend (dual check: Iroh seq + Iroh-blob nullifier), signatures valid (Iroh docs self-authenticating). Phase 2 additionally verifies both wallet documents (sender's + recipient's) are non-terminal. On success, cache state locally (feeds Pulse).


## 22. Beacon, Randomness & Forced Finalization

```python
class BeaconClient:
    # drand mainnet, 30s round duration
    # get_round(n), get_latest(), round_at_time(ts), BLS signature verification
    # Cache rounds locally

MAX_BEACON_AGE_ROUNDS = 20  # ~10 minutes
```

### 22.1 Trigger

Any node MAY submit a ForcedFinalizationRequest when:
- Slot is in COMMITTED state
- 72 hours (FORCED_FINALIZATION_TIMEOUT) have elapsed since committed_at
- Nullifier has not been published
- W₂₃ quorum evidence (≥13 valid attestations + π₂₃) is available

### 22.2 W_FF Witness Selection

Fresh VRF selection (WITNESS_TOTAL witnesses, WITNESS_QUORUM threshold).
MUST be disjoint from W₁, W₂₃, and W₄.

### 22.3 W_FF Verification

Each W_FF witness MUST verify:

| # | Check |
|---|-------|
| 1 | VRF self-selection into W_FF |
| 2 | W₂₃ attestation signatures valid (≥13) |
| 3 | π₂₃ valid |
| 4 | Timeout elapsed (72 hours) |
| 5 | Nullifier not already published |
| 6 | Attempt share collection (confirm unavailability) |

If shares ARE available (step 6 succeeds), W_FF MUST respond
with status SHARES_AVAILABLE and decline to attest forced finalization.

### 22.4 Execution

With ≥13 W_FF APPROVED attestations:

1. Publish nullifier_commit (NOT preimage — shares unavailable).
   Push to NULLIFIER_RESPONSIBLE_PEERS via publish_forced (§40.2).
2. Build ForcedFinalizationRecord containing: session_id, nullifier_commit,
   W₂₃ attestations, W_FF attestations, forced_at timestamp.
   Publish as Iroh blob → forced_finalization_cid.
   Push forced_finalization_cid to FORCED_FINALIZATION_REPLICATION_PEERS
   using XOR routing key:
       forced_record_key = H("inat_forced_finalization:" || session_id)
   Peers closest by XOR distance to forced_record_key store the blob
   and answer fetch requests. W₄ witnesses and fold provers locate
   the blob by computing forced_record_key from session_id (available
   in the SpendRecord) and querying XOR-nearest peers.
   Each W_FF witness that approved MUST independently store the blob
   locally for at least FORCED_FINALIZATION_RETENTION_WINDOW.
3. Build SpendRecord with is_forced = true,
   forced_finalization_cid = forced_finalization_cid.
   Publish SpendRecord via §41 archive (pin=True).
4. Announce W_FF completion to recipient.
   W_FF witnesses broadcast a ForcedFinalizationAnnouncement on the
   recipient's wallet document gossip topic (INAT_ORACLE_TOPIC),
   keyed by the recipient's identity_commit from the RecipientCommitment
   in Π₂₃. The recipient's app-level recovery daemon (subscribed to
   their own wallet document) receives this and triggers Phase 4.

       announcement = ForcedFinalizationAnnouncement(
           session_id             = session_id,
           nullifier_commit       = nullifier_commit,
           forced_finalization_cid = forced_finalization_cid,
           spend_cid              = spend_cid,
           recipient_identity_commit = recipient_commitment.slot_commit,
           timestamp              = current_timestamp(),
       )
       await transport.broadcast(
           INAT_ORACLE_TOPIC,
           announcement.serialize(),
           privacy=PROTOCOL_PRIVACY_CONFIGS["heartbeat"])

5. Proceed to Phase 4 (§20.4).

NOTE: The ForcedFinalizationRecord is only required until Phase 4
completes and a CollapsedProof exists. After folding, it MAY be
garbage collected — the fold absorbs its evidence. Nodes MUST NOT
GC it before the recipient slot reaches FINALIZED.

**Properties:** Decentralized (fresh VRF), non-griefable (timeout + proof required),
permissionless trigger, different witness set from all other phases.

| Parameter | Value |
|-----------|-------|
| FORCED_FINALIZATION_TIMEOUT | 259,200s (72 hours) |
| FORCED_FINALIZATION_WITNESSES | 19 |
| FORCED_FINALIZATION_THRESHOLD | 13 |

---

---

## 23. Recovery & Resumption


### 23.1 Recovery (abort stuck PENDING_SEND or PENDING_RECEIVE)

**Precondition:** Slot status ∈ {PENDING_SEND, PENDING_RECEIVE}.
NOT available for COMMITTED or later states (point of no return).

**Procedure:**

1. Owner signs RecoveryRequest (identity_commit, slot_commit,
   stuck_seq, session_id, stuck_state).

2. Fresh VRF witness selection (NOT original W₁ or W₂₃).


3. Each recovery witness investigates:

| Check | Reject recovery if |
|-------|--------------------|
| Owner signature | Invalid |
| Slot status | ∉ {PENDING_SEND, PENDING_RECEIVE} |
| Slot status | = COMMITTED (point of no return — must resume, not recover) |
| Nullifier | Already published |
| Oracle seq | Doesn't match expected |
| Timeout | Not elapsed (unless bilateral signatures provided) |

**Delete the paragraph:**
> PENDING_RECEIVE-specific check: Recovery witnesses MUST also
> verify that no W₂₃ quorum exists for this session. [...]

**Replace with:**

**Safety under ambiguous W₂₃ state:** Recovery does NOT require
verifying whether the original W₂₃ quorum was achieved. Recovery
is safe regardless, because:

1. Both parties recover independently to new sequence numbers
   (stuck_seq + 1). Oracle advances past the stuck seq.
2. If W₂₃ quorum WAS achieved and shares release later
   (publishing the orphaned nullifier), the nullifier maps
   to a RECOVERED (terminal) slot. No value can be spent
   from it.
3. The abandoned SpendRecord's output_commits point to slots
   at the stuck seq — both now RECOVERED (terminal). The fold
   chain dead-ends.

4. **Recovery Quorum R₁** (21/29, fresh VRF):

   seed_R1 = H(slot_commit || beacon_round || beacon_randomness)

   Each R₁ witness investigates per the table above.
   With ≥21/29 APPROVED attestations, compute:

   R1_attestation_root = Merkle(R₁ attestation signatures)

5. **Recovery Quorum R₂** (21/29, fresh VRF, different shard):

   seed_R2 = H(seed_R1 || H(R1_attestation_root)
               || beacon_round_2 || beacon_randomness_2)

   Each R₂ witness verifies:
   - R₁ attestation signatures valid (≥13)
   - R₁ investigation results consistent
   - Timeout still valid
   - Nullifier still unpublished

   With ≥21/29 R₂ APPROVED attestations:
   - recovery_seq = base_seq + SEQ_STRIDE
     (base_seq = consumed slot's creation seq, proven via slot_commit)
   - Write recovery entries (R₁ at base+2, R₂ at base+3, RECOVERED
     at base+4). Each is one doc entry, one seq increment.
   - Create new OwnedSlot at recovery_seq with identical balance
     NOTE: recovery_seq == intended output seq from aborted tx.
     Whether normal completion or recovery, the output always
     materializes at base_seq + SEQ_STRIDE.
   - Generate recovery proof (§34)
   - Publish RecoveryRecord
   - Old slot status → RECOVERED (terminal). New slot → READY.

**Race analysis:** If nullifier publishes DURING recovery
(between investigation and completion), recovery witnesses
will observe it in their nullifier check and reject. If it
publishes AFTER recovery completes, the slot is already
RECOVERED — the orphaned nullifier is harmless. Oracle
monotonicity is the fundamental safety guarantee.

**Anti-grinding:** Recovery uses two sequential witness quorums
(R₁ and R₂), following the same pattern as the main transaction:

| Quorum | Seed | Grindable? |
|--------|------|------------|
| R₁ (21/29) | H(slot_commit \|\| beacon_round \|\| beacon_randomness) | Yes (owner controls slot) |
| R₂ (21/29) | H(seed_R₁ \|\| H(R₁_attestation_root) \|\| beacon_round_2 \|\| beacon_randomness_2) | No (R₁ attestation root is unpredictable) |

R₁ investigates (oracle, nullifier, slot status, timeout).
R₂ confirms based on R₁'s findings. Different shards.
Both quorums are fresh VRF selections, independent of the
original transaction's W₁ and W₂₃.

---

### 23.2 Resumption (continue interrupted transaction)

| Stuck state | Action |
|-------------|--------|
| PENDING_SEND | Try resume if counterparty reachable. Recover after timeout if not. |
| PENDING_RECEIVE | Try resume W₂₃ collection (re-broadcast Phase 2 envelope). Recover after timeout if not. |
| COMMITTED | MUST resume. Either party SHOULD notify W₁ of W₂₃ quorum (§20.2 step 19–20). Recipient acts independently after W1_FORWARD_TIMEOUT if sender unresponsive. Collect shares. Finalize Phase 3. |
| SPENT | Resume Phase 4 only — fresh W₄ selection for DA confirmation. |

### State Preservation (Iroh must persist)

| Data | Required For | Availability |
|------|-------------|--------------|
| witness_seed_1, encrypted_shares, poly_commits | Resumption / Phase 3 | Iroh doc + Pulse + W₁ local |
| sender/recipient_commitment, σ_s, σ_r, proof_1 | All phases | Iroh doc + 50x push |
| attestations_1, attestations_23 | Phase triggers | Iroh-blob + Pulse + 50x push |
| session_id, witness_seed_23 | Correlation / Phase 4 | Iroh doc + Pulse |

Fallback: query responsible XOR neighborhood (same tiered fetch as Section 15.2).


### 23.3 Burn (READY → BURNED)

Permanent value destruction. Two types: owner-voluntary and
issuer-initiated recall. Both require witness attestation (21/29).

**Burn types:**

| Type | Initiator | Use case |
|------|-----------|----------|
| VOLUNTARY | Slot owner | Deflationary burn, key compromise response |
| ISSUER_RECALL | Issuer | Regulatory action, asset sunset |

**Precondition:** Slot status = READY.

**Procedure:**

1. **Initiator signs BurnIntent.**

       burn_intent = BurnIntent(
           slot_commit    = slot.slot_commit,
           identity_commit = H(owner_pk),
           burn_type      = BurnType.VOLUNTARY,  # or ISSUER_RECALL
           reason         = "voluntary",
           timestamp      = current_timestamp(),
       )

   VOLUNTARY: signed by owner_sk.
   ISSUER_RECALL: signed by issuer_sk. Must include
   `issuer_attestation` proving issuer authority over asset_id.

2. **Increment oracle sequence.**

       new_seq = await oracle.increment_and_propagate(
           identity_commit=H(owner_pk),
           owner_sk=owner_sk,
           wallet_doc=wallet_doc)

3. **Fresh VRF witness selection** (19 witnesses, 21/29 quorum).
   Seed: `H(slot_commit || beacon_round || beacon_randomness)`.

4. **Each burn witness investigates:**

   | Check | Reject burn if |
   |-------|----------------|
   | Initiator signature | Invalid |
   | Slot status | ≠ READY |
   | Nullifier | Already published |
   | Oracle seq | Doesn't match |
   | Issuer authority (RECALL only) | issuer_pk ≠ asset issuer |

5. **With ≥21/29 APPROVED attestations:**
   a. Publish nullifier (prevents future spends of this slot).
   b. Build BurnRecord. Publish as content-addressed blob.
   c. Slot status → BURNED (terminal).

**BurnRecord:**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| slot_commit | bytes | 32 | Burned slot |
| identity_commit | bytes | 32 | H(owner_pk) |
| nullifier_commit | bytes | 32 | Published on burn |
| burn_type | BurnType | 1 | VOLUNTARY or ISSUER_RECALL |
| asset_id | bytes | 32 | |
| balance_commit | bytes | 32 | Pedersen commitment (value destroyed) |
| burn_seed | bytes | 32 | VRF seed for witness selection |
| witness_attestation_root | bytes | 32 | Merkle root of ≥13 attestations |
| issuer_attestation | bytes? | 64 | Present for ISSUER_RECALL only |
| timestamp | uint64 | 8 | |
| burn_cid | bytes | 32 | CID of this record |

**Post-conditions:**
- Slot status = BURNED (terminal, no recovery possible)
- Nullifier published (blocks any spend attempt)
- Oracle seq advanced (burn consumes a sequence number)
- Value is permanently destroyed

**BURNED is irreversible.** Unlike RECOVERED, there is no path
from BURNED to any other state. The balance is gone.

---

# Part VI: Protocol Interfaces

## 24. Content Store Protocol

```python
class ContentStoreProtocol(Protocol):
    """Deterministic content-addressed storage (Iroh-blob). cid = H(content)."""
    async def publish(self, content: bytes) -> bytes:
    async def retrieve(self, cid: bytes) -> Optional[bytes]: ...
    async def exists(self, cid: bytes) -> bool: ...
    def compute_cid(self, content: bytes) -> bytes: ...
```



## 25. Transport Protocols

class MessageTransport(Protocol):
    """
    Abstract message-plane transport. All inter-node protocol
    messages MUST route through this interface.

    Implementations SHOULD provide:
      - Sender anonymity (observers cannot link message to originator)
      - Timing decorrelation (message timing does not leak identity)
      - Payload indistinguishability (padded to uniform size classes)

    Implementations MAY provide:
      - Relay-based routing
      - Multi-hop anonymization
    """

    async def broadcast(self, topic: bytes, payload: bytes,
                        privacy: 'SecurityConfig') -> None:
        """Broadcast to all nodes subscribed to topic."""
        ...

    async def send(self, reply_path: bytes, payload: bytes,
                   privacy: 'SecurityConfig') -> None:
        """Send to a specific node via its reply path."""
        ...

    async def rpc(self, reply_path: bytes, method: str,
                  payload: bytes, privacy: 'SecurityConfig',
                  timeout: float = 5.0) -> bytes:
        """Request-response to a specific node."""
        ...

    async def receive(self, topic_or_filter: bytes,
                      timeout: float) -> Optional['TransportMessage']:
        """Receive next message matching filter."""
        ...

    def create_reply_path(self) -> bytes:
        """
        Generate an opaque reply path for this node.
        Callers embed this in outgoing messages so responders
        can reach them without knowing their network identity.

        The reply path MUST NOT contain the node's real
        network address in cleartext. Format is implementation-defined.
        """
        ...


class StorageTransport(Protocol):
    """
    Abstract data-plane transport for content-addressed
    and document storage. Direct access is acceptable here
    because content is commitments/CIDs/proofs — identity
    is not embedded in the data or the access pattern.

    Implementations MAY add caching, replication strategies,
    or access control beyond what is specified here.
    """

    # --- Content-addressed blobs ---
    async def blob_put(self, content: bytes) -> bytes:
        """Store content, return CID. Immutable."""
        ...

    async def blob_get(self, cid: bytes) -> Optional[bytes]:
        """Retrieve by CID."""
        ...

    async def blob_exists(self, cid: bytes) -> bool: ...

    # --- Mutable documents ---
    async def doc_create(self) -> str:
        """Create new document. Caller is sole writer."""
        ...

    async def doc_open(self, doc_id: str) -> Optional['DocHandle']:
        """Open existing document."""
        ...

    async def doc_join(self, doc_id: str, ticket: str) -> 'DocHandle':
        """Join document with read ticket."""
        ...

    async def doc_sync_to(self, doc: 'DocHandle', peer: bytes) -> None:
        """Push document state to peer."""
        ...

    async def doc_sync_from(self, doc: 'DocHandle', peer: bytes) -> None:
        """Pull document state from peer."""
        ...


@dataclass
class SecurityConfig:
    """Per-message-type security requirements.
    Implementations map these to concrete mechanisms."""
    encryption: bool = True
    padding: bool = False
    timing_jitter: bool = False
    max_delay: float = 0.0          # Max jitter delay in seconds
    relay: bool = False             # Request relay routing


@dataclass
class TransportMessage:
    """Received message. reply_path is opaque — pass it to send()."""
    payload: bytes
    reply_path: bytes               # Opaque sender return address
    topic: Optional[bytes] = None


# §25.1 Privacy Configurations

Protocol-level privacy requirements per message type.
Implementations decide how to fulfill these.

PROTOCOL_PRIVACY_CONFIGS = {
    # Phase broadcasts — sender identity is privacy-critical
    # Envelopes — broadcast to shard, contain no identity data
    "phase1_envelope":       SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, max_delay=0.2,
                                            relay=True),
    "phase2_envelope":       SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, max_delay=0.2,
                                            relay=True),

    # Payloads — delivered to selected witnesses only
    "intent_payload":        SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, max_delay=0.1,
                                            relay=True),

    # Eligibility — witness → sender, lightweight
    "witness_eligibility":   SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, relay=True),

    # Transaction invite — sender to recipient, contains identity
    "tx_invite":             SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, max_delay=0.2,
                                            relay=True),
    "phase4_envelope":       SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, relay=True),

    "w23_notification":  SecurityConfig(encryption=True, padding=True,
                                     timing_jitter=True, relay=True),
                                     
    # Witness responses — witness identity is privacy-critical
    "witness_response":      SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, relay=True),

    # Discovery — all steps are privacy-critical
    "capacity_proof":        SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, max_delay=0.2,
                                            relay=True),
    "locked_funds_proof":    SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, relay=True),
    "spot_reveal":           SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, max_delay=0.1,
                                            relay=True),

    # Infrastructure — lower sensitivity
    "nullifier_push":        SecurityConfig(encryption=True, padding=False,
                                            timing_jitter=False, relay=False),
    "nullifier_query":       SecurityConfig(encryption=True, padding=False,
                                            timing_jitter=False, relay=False),
    "oracle_query":          SecurityConfig(encryption=True, padding=False,
                                            timing_jitter=False, relay=False),
    "share_delivery":        SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, relay=True),
    "share_publish":         SecurityConfig(encryption=True, padding=False,
                                            timing_jitter=False, relay=False),
    "donation_receipt":      SecurityConfig(encryption=True, padding=True,
                                            timing_jitter=True, relay=False),

    # Heartbeat / pulse — not sensitive
    "heartbeat":             SecurityConfig(encryption=False, padding=False,
                                            timing_jitter=False, relay=False),
    "pulse":                 SecurityConfig(encryption=False, padding=False,
                                            timing_jitter=False, relay=False),
}


## 25.2 Transport Selection Rules

MUST use MessageTransport (§25):
  - All transaction intents (§20.1, §20.2, §20.4)
  - All witness responses (§21)
  - All RPC calls: nullifier push/query (§40), oracle query (§45)
  - Share delivery to W₁ witnesses (§21.3)
  - Share CID broadcast (§20.3)
  - Discovery messages (§18)
  - Transaction invite/accept (§49.8)
  - Donation receipt push/query (§47.5.5)
  - Heartbeat broadcast (§19.2)
  - Pulse summary exchange (§45)

MUST use StorageTransport (§25):
  - Blob add/get (spend records, proofs, nullifiers, bundles)
  - Document create/open/join/get/set
  - Document sync (replication)
  - IntentEnvelopes are broadcast (shard gossip)
  - IntentPayloads are unicast (reply_path to selected witnesses)
  - WitnessEligibility is unicast (reply_path to sender)
  - WitnessResponse (attestation) is unicast (reply_path to sender)

The boundary is: if the operation carries a message FROM one node
TO another, use MessageTransport. If it reads/writes content to
a shared store, use StorageTransport.


# --- Node wiring ---

class InatNodeTransports:
    """
    Constructed at node startup. Protocol code receives these
    via dependency injection — never touches Iroh directly for
    message-plane operations.
    """

    def __init__(self, iroh_node: 'IrohNode', config: 'LoudWhisperConfig'):
        self.messages: MessageTransport = LoudWhisperTransport(iroh_node, config)
        self.storage: StorageTransport = IrohStorageTransport(iroh_node)

## 26. Sequence Protocol

The sequence counter is the `identity:protocol_seq` field in the
wallet's Iroh document (§36.2). There is no separate oracle system.

The wallet owner is the sole writer. Iroh's per-author monotonic
sequencing prevents rollback. Peers subscribed to the wallet
document (§45) can read the current sequence directly.

**Trust model:**
- Authoritative for LIVENESS: `protocol_seq > creation_seq` → slot is dead.
- NOT authoritative for legitimacy — ZK proofs validate balance,
  witness history, and issuer chain independently.
- A corrupted document cannot forge valid proofs — only cause liveness issues.
- An unreachable wallet document halts transaction processing for that identity.

**Operations (all map to wallet document reads/writes):**

| Operation | Implementation |
|-----------|---------------|
| get_seq | Read `identity:protocol_seq` from wallet doc (local subscription, peer doc, or gossip cache). Returns 0 if never seen. |
| increment | Owner writes `protocol_seq = old + 1` to own wallet doc. Monotonic by Iroh CRDT. |
| verify_not_spent | `current_seq > creation_seq` → definitely spent. `current_seq == creation_seq` → possibly unspent (verify proof). |


```

## 27. Slot State Protocol

```python
class SlotStateProtocol(Protocol):
    async def publish(self, slot_commit: bytes, status: SlotStatus,
                      metadata: SlotMetadata, proof: bytes) -> bool:
        """
        Enforced transitions:
        READY → PENDING_SEND
        READY → PENDING_RECEIVE
        READY → BURNED
        PENDING_SEND → COMMITTED
        PENDING_SEND → RECOVERED (21/29 recovery quorum)
        PENDING_RECEIVE → COMMITTED (W₂₃ quorum)
        PENDING_RECEIVE → RECOVERED (21/29 recovery quorum)
        COMMITTED → SPENT
        SPENT → FINALIZED
        """
        ...
    async def resolve(self, slot_commit: bytes) -> Optional[SlotState]: ...
```

## Summary: Seq Arithmetic

| Party | Base | Entry 1 | Entry 2 | Entry 3 | Entry 4 = Output | Forward |
|-------|------|---------|---------|---------|-------------------|---------|
| Sender | N | N+1 PENDING_SEND | N+2 COMMITTED | N+3 SPENT | N+4 FINALIZED | N+8 |
| Recipient | M | M+1 PENDING_RECEIVE | M+2 COMMITTED | M+3 (observed) | M+4 READY | M+8 |

Recovery from PENDING (at base+1):

| Entry | Seq | Content |
|-------|-----|---------|
| R₁ investigation | base+2 | R₁ attestation CIDs |
| R₂ confirmation | base+3 | R₂ attestation CIDs |
| RECOVERED output | base+4 | New slot, balance preserved |

**Invariant:** output always at `base + SEQ_STRIDE` regardless of path (normal or recovery).

**Recipient M+3 entry:** The recipient writes an entry at M+3
corresponding to the SPENT phase (observed nullifier publication).
This entry is an acknowledgment, not an action — the recipient
does not publish the nullifier (the sender/W₁ witnesses do).
The entry serves two purposes: (1) it advances the recipient's
protocol_seq monotonically through all four phase positions,
maintaining the SEQ_STRIDE invariant, and (2) it records the
nullifier_cid in the recipient's wallet document for local
auditability. Omitting this entry would create a seq gap
(M+1, M+2, M+4) that violates the one-entry-per-phase model
and complicates recovery reasoning.

**Entry model:** one `doc.set()` = one `protocol_seq` increment. Blobs carry the data. No multi-set per phase. No CID circularity (WitnessBundle → SpendRecord, never the reverse).

## 28. Evidence Categories

```python

SELF_AUTHENTICATING_EVIDENCE = {
    "zk_proofs", "signatures", "pedersen_commitments",
    "feldman_poly_commits", "seed_chain_values", "vrf_proofs",
    "beacon_round_and_value",
}

MUTABLE_STATE_EVIDENCE = {
    "slot_status", "sequence_number", "nullifier_existence",
    "session_binding", "asset_id",
}
```

## 29. Witness Protocol

```python
@dataclass
class AttestationRequest:
    """Internal representation of an IntentPayload after Round 1 eligibility.
    Constructed by the witness from a received IntentPayload (§44).
    Witness fetches mutable state independently via slot_doc_id and oracle_key.

    NOTE: This is NOT broadcast. It is the witness's internal view
    after receiving IntentPayload in Round 2 of the two-phase
    witness engagement (§21.1)."""
    session_id: bytes
    phase: Phase
    beacon_round: int               # From IntentEnvelope (Round 1)
    # Pointers for independent fetch (from IntentPayload)
    slot_doc_id: str                # Primary wallet doc — witness fetches independently
    oracle_key: bytes               # H(pk) of primary party — witness fetches independently
    # Self-authenticating cryptographic material (from IntentPayload)
    proof: bytes
    public_inputs: bytes
    nullifier_commit: bytes         # From IntentEnvelope (Round 1)
    witness_seed: bytes
    # Claimed values (cross-checked against fetched state)
    claimed_slot_commit: bytes
    claimed_asset_id: bytes
    # Phase 1
    expected_sequence: Optional[int] = None
    poly_commits: Optional[List[bytes]] = None
    # Phase 2 (dual doc fetch)
    committed_sequence: Optional[int] = None
    sender_commitment_hash: Optional[bytes] = None
    sender_slot_doc_id: Optional[str] = None    # Sender's wallet doc (Phase 2 only)
    sender_oracle_key: Optional[bytes] = None   # H(sender_pk) (Phase 2 only)
    # Phase 4
    spend_cid: Optional[bytes] = None
    nullifier_cid: Optional[bytes] = None
    # Evidence
    evidence_cid: Optional[bytes] = None

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self), default=enum_encoder)

    @classmethod
    def deserialize(cls, data: bytes) -> 'AttestationRequest':
        d = msgpack.unpackb(data)
        d['phase'] = Phase(d['phase'])
        return cls(**d)


@dataclass
class MutableStateBundle:
    slot_status: SlotStatus
    current_seq: int
    nullifier_exists: bool
    session_binding: Optional[bytes]
    asset_id: Optional[bytes]


class WitnessProtocol(Protocol):
    async def attest(self, request: AttestationRequest) -> Attestation:...

    # _fetch_and_verify_mutable_state: see §21 InatNode._fetch_mutable_state
    # MutableStateBundle: see §21
    
    def _check_terminal_state(self, status: SlotStatus, phase: Phase):
        """REJECT for TERMINAL_STATES in Phase 1 and 2. Phase 4 exempt (slot expected SPENT)."""
        if phase in (Phase.INITIATE, Phase.COMMIT):
            if status in TERMINAL_STATES:
                raise TerminalStateError(f"Slot in terminal state {status}")

    async def request_attestations(self, phase: Phase, session_id: bytes, proof: bytes, witnesses: List[WitnessInfo]) -> List[Attestation]: ...
    def verify_quorum(self, attestations: List[Attestation], phase: Phase, threshold_m: int = 13) -> bool: ...
    async def collect_shares(self, session_id: bytes, threshold_m: int = 21, timeout: float = 30.0) -> List[PublishedShare]: ...
    async def reconstruct_nullifier(self, shares: List[PublishedShare], poly_commits: List[bytes], expected_nullifier_commit: bytes, threshold_m: int = 21) -> bytes: ...
```

## 30. ZK Prover Protocol

```python
class ZKProverProtocol(Protocol):
    async def prove_phase1(self, public: Phase1PublicInputs, private: Phase1PrivateInputs) -> bytes: ...
    async def prove_phase2(self, public: Phase2PublicInputs, private: Phase2PrivateInputs) -> bytes: ...
    async def fold(self, previous_fold: Optional[bytes], spend_record: SpendRecord,
                   phase_proofs: Tuple[bytes, bytes], confirmations: List[Confirmation],
                   witness_set_root: bytes, beacon_round: int) -> CollapsedProof:
        """v1: Attested hash-chain fold (~1KB, <1ms verification, owner-signed).
        v2 (future): Nova IVC fold (~22KB, ~200ms verification, trustless).
        Output format: CollapsedProof with opaque fold_proof field."""
        ...
    async def verify_collapsed(self, proof: CollapsedProof) -> bool: ...
```

---

# Part VII: ZK Circuit Definitions

## 31. Phase 1 Circuit (π₁)

```
PUBLIC INPUTS:                 PRIVATE INPUTS:
• slot_commit                  • sender_sk
• nullifier_commit             • sender_seq (source slot seq)
• amount_commit                • nullifier
• sender_output_commit         • input_balance, input_blinding
• sender_output_amount_commit  • amount, amount_blinding
• fee_commit                   • sender_output_amount, sender_output_blinding
• asset_id                     • fee, fee_blinding
• owner_pk_commit              • lineage_proof
• session_id
• beacon_round
• lineage_commit
• poly_commits_root
• sender_forward_commit

PROVES:
1. slot_commit = H("inat_slot:" || sk || base_seq)
   Where base_seq is the consumed slot's creation seq.
2. nullifier = H("inat_nullifier:" || sk || base_seq)
3. nullifier_commit = H(nullifier)
4. owner_pk_commit = H(derive_pk(sk))
5. Balance conservation:
   input_balance = amount + sender_output_amount + fee
   Each verified via Pedersen commitment opening.
6. sender_output_commit = H("inat_slot:" || sk || (base_seq + SEQ_STRIDE))
   Output slot is SEQ_STRIDE positions past consumed slot.
7. sender_forward_commit = H("inat_slot:" || sk || (base_seq + 2 * SEQ_STRIDE))
   Next tx's output, another SEQ_STRIDE past this output.
   GENESIS SPECIAL CASE: For depth=1 (first spend of a mint-origin
   slot), base_seq=0 and forward_commit = H(sk || 8). There is no
   "previous forward_commit" to chain from — the fold circuit (§33
   constraint 9) verifies forward_commit consistency starting at
   depth=2. At depth=1, the forward_commit is accepted as the
   chain's initial value, analogous to the self-referential
   previous_registry_root at genesis (§33 constraint 7c).
8. poly_commits_root matches polynomial with nullifier as constant term
9. Lineage valid (chain to genesis per §35)
10. All commitments use correct asset_id
11. SEQ_STRIDE == 4 (hardcoded constant, verified in circuit)

DOES NOT PROVE (deferred to π₂₃):
• σ_s validity (σ_s doesn't exist at π₁ generation time)
• witness_set_root binding (witnesses unknown at π₁ time)
• sender_commitment_hash binding (commitment not finalized at π₁ time)
```


```python
@dataclass
class SenderPublicInputs:
    """Public inputs for π₁."""
    slot_commit: bytes
    nullifier_commit: bytes
    amount_commit: bytes
    sender_output_commit: bytes
    sender_output_amount_commit: bytes
    fee_commit: bytes
    asset_id: bytes
    owner_pk_commit: bytes
    session_id: bytes
    beacon_round: int
    lineage_commit: bytes
    poly_commits_root: bytes
    sender_forward_commit: bytes
    seq_stride: int = 4             # SEQ_STRIDE (hardcoded, verified in circuit)
    
    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

@dataclass
class SenderWitness:
    """Private witness for π₁."""
    sk: bytes
    seq: int                        # Source slot seq (NOT new_seq)
    input_balance: int
    input_blinding: bytes
    amount: int
    amount_blinding: bytes
    sender_output_amount: int
    sender_output_blinding: bytes
    fee: int
    fee_blinding: bytes
    lineage_proof: bytes
    nullifier: bytes                # For poly_commits_root verification

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))
```

## 32. Phase 2 Circuit (π₂₃)

```
PUBLIC INPUTS:                 PRIVATE INPUTS:
• H(sender_commitment)         • sender_commitment (full)
• H(recipient_commitment)      • recipient_commitment (full)
• amount_commit                • sigma_s, sigma_r
• nullifier_commit             • sender_pk, recipient_sk
• session_id                   • recipient_seq
• beacon_round_1               • amount, blindings
• beacon_round_23              • lineage_proof
• witness_set_root_1           • previous_collapsed_proof
• witness_seed_23              • seed_1
• lineage_commit               • beacon_randomness_23
• sender_forward_commit        • vrf_proofs_23
• recipient_forward_commit

NOTE: witness_seed_1 and H(σ_r) are PRIVATE inputs. The circuit
proves seed_23 = H(seed_1 || H(σ_r) || beacon_round_23 || randomness_23)
without exposing seed_1 or H(σ_r) to verifiers. This prevents
cross-phase linking via public inputs serialization.

PROVES:
1. [WITNESS-VERIFIED, NOT CIRCUIT-PROVEN] σ_s validity is verified
   natively by W₂₃ witnesses (§21.2 B1b) and bound into the fold
   hash chain (§33.1). Removed from circuit to avoid ~200-400k
   non-native constraints (Ed25519 over Curve25519 inside BN254).
2. [WITNESS-VERIFIED, NOT CIRCUIT-PROVEN] σ_r validity is verified
   natively by W₂₃ witnesses (§21.2 B1c) and bound into the fold
   hash chain (§33.1). σ_r remains as private input for seed chain
   derivation (constraint 10: H(σ_r) in seed₂₃).
3. Recipient owns sk: H(derive_pk(recipient_sk)) = recipient_pk_commit
4. Recipient slot: slot_commit = H("inat_slot:" || recipient_sk || (recipient_base_seq + SEQ_STRIDE))
   Where recipient_base_seq is the recipient's protocol_seq before this transaction.
5. amount_commits match (sender = recipient)
6. session_ids match (sender_commitment.session_id = recipient_commitment.session_id)
7. recipient_commitment.sender_nullifier_commit = nullifier_commit
8. recipient_commitment.sender_commitment_hash = H(sender_commitment)
9. witness_set_root_1 = sender_commitment.witness_tpk    [W₁ binding]
10. witness_seed_23 = H(seed_1 || H(σ_r) || beacon_round_23 || beacon_randomness_23)
11. sender_forward_commit = sender_commitment.sender_forward_commit
12. recipient_forward_commit = H("inat_slot:" || recipient_sk || (recipient_base_seq + 2 * SEQ_STRIDE))
13. Lineage valid (chain to genesis per §35)

DOES NOT PROVE:
- σ_s, σ_r signature validity (witness-verified natively per §21.2
  B1b/B1c; bound into fold hash chain per §33.1; third parties
  verify from fold public data using native Ed25519)
- W₂₃ witness set binding (W₂₃ verified by VRF + seed chain;
  fold circuit binds retroactively)
```

```python
@dataclass
class CombinedPublicInputs:
    """Public inputs for π₂₃. NOTE: witness_seed_1 and H(σ_r) are
    intentionally absent — they are private inputs to prevent
    cross-phase linking."""
    sender_commitment_hash: bytes
    recipient_commitment_hash: bytes
    amount_commit: bytes
    nullifier_commit: bytes
    session_id: bytes
    beacon_round_1: int
    beacon_round_23: int
    witness_set_root_1: bytes
    witness_seed_23: bytes
    lineage_commit: bytes
    sender_forward_commit: bytes
    recipient_forward_commit: bytes

    def serialize(self) -> bytes:
        return msgpack.packb({
            "sender_commitment_hash": self.sender_commitment_hash,
            "recipient_commitment_hash": self.recipient_commitment_hash,
            "amount_commit": self.amount_commit,
            "nullifier_commit": self.nullifier_commit,
            "session_id": self.session_id,
            "beacon_round_1": self.beacon_round_1,
            "beacon_round_23": self.beacon_round_23,
            "witness_set_root_1": self.witness_set_root_1,
            "witness_seed_23": self.witness_seed_23,
            "lineage_commit": self.lineage_commit,
            "sender_forward_commit": self.sender_forward_commit,
            "recipient_forward_commit": self.recipient_forward_commit,
        })

@dataclass
class CombinedWitness:
    """Private witness for π₂₃.
    NOTE: sigma_s and sender_pk are NOT circuit inputs — signature
    validity is witness-verified (§21.2 B1b/B1c) and fold-bound (§33.1).
    sigma_r is retained as private input for seed chain (constraint 10)."""
    sender_commitment: 'SenderCommitment'
    recipient_commitment: 'RecipientCommitment'
    sigma_r: bytes
    recipient_sk: bytes
    recipient_base_seq: int         # Recipient's protocol_seq BEFORE this tx
    amount: int
    amount_blinding_sender: bytes
    amount_blinding_recipient: bytes
    lineage_proof: bytes
    previous_collapsed_proof: Optional[bytes]
    seed_1: bytes
    beacon_randomness_23: bytes

    def serialize(self) -> bytes:
        return msgpack.packb({
            "sender_commitment": self.sender_commitment.serialize(),
            "recipient_commitment": self.recipient_commitment.serialize(),
            "sigma_r": self.sigma_r,
            "recipient_sk": self.recipient_sk,
            "recipient_base_seq": self.recipient_base_seq,
            "amount": self.amount,
            "amount_blinding_sender": self.amount_blinding_sender,
            "amount_blinding_recipient": self.amount_blinding_recipient,
            "lineage_proof": self.lineage_proof,
            "previous_collapsed_proof": self.previous_collapsed_proof,
            "seed_1": self.seed_1, "beacon_randomness_23": self.beacon_randomness_23,
        })
```

## 33. Fold Circuit

```
INPUTS:
- previous_fold (None for depth=1)
- spend_record, phase_proofs (π₁, π₂₃), confirmations (7)
- witness_seeds (seed_1, seed_23, seed_4)
- witness_set_roots, beacon_rounds, beacon_randomnesses
- issuer_pk (current signing key for this transaction)
- asset_genesis_cid (immutable asset identity anchor)
- issuer_registry_root (Merkle root of authorized issuer PKs)
- issuer_registry_epoch (epoch of the registry used)
- issuer_registry_sig: bytes # (M-of-N signature over registry, from keys
  in previous_registry_root — or self-signed if depth=1)
- issuer_pk_registry_proof: List[Tuple[bytes, str]] # (Merkle proof: issuer_pk ∈ registry)
- previous_registry_root (from previous_fold.issuer_registry_root,
  or == issuer_registry_root if depth=1)
- genesis_record + genesis_issuer_sig (depth=1 only)
- sigma_s: bytes                   # Sender signature (not ZK-proven; fold-bound for third-party verification)
- sigma_r: bytes                   # Recipient signature (not ZK-proven; fold-bound for third-party verification)
- sender_pk: bytes                 # For third-party native Ed25519 verification of σ_s
- recipient_pk: bytes              # For third-party native Ed25519 verification of σ_r
- w1_vrf_proofs: List[(witness_pk, vrf_output, vrf_proof)]  # quorum entries
• w23_vrf_proofs: List[(witness_pk, vrf_output, vrf_proof)] # quorum entries
• w4_vrf_proofs: List[(witness_pk, vrf_output, vrf_proof)]  # 7 entries
• declared_depth: uint16  # Shard depth, consistent across all phases
• issuer_eligible_root: bytes                  # Issuer-signed Merkle root
• issuer_eligible_root_sig: bytes              # Issuer signature
• issuer_eligible_root_epoch: uint64           # Epoch number
- w1_eligibility_proofs: List[List[bytes]]     # Per-witness Merkle proofs
- w23_eligibility_proofs: List[List[bytes]]    # Per-witness Merkle proofs

NOTE: VRF verification inside BN254 circuits over Ristretto255
requires non-native field arithmetic (~50-100k constraints per
verification). For 33 total witnesses this adds ~1.5-3.3M
constraints. Mitigation options:
  a. Circuit-native VRF (BN254-based ECVRF) — eliminates
     non-native overhead, ~10-20k per verify, ~660k total
  b. Batched VRF proof — aggregate N VRF proofs into single
     verification, amortized cost
  c. Recursive composition — separate VRF-validity SNARK
     whose proof is verified inside fold (~200k fixed)
Option (c) is recommended: a dedicated VRF-batch circuit proves
"these 33 witnesses all passed VRF" and the fold circuit verifies
that single proof. Adds one verification key but keeps fold
circuit manageable.

PROVES:
1. Previous fold valid (or genesis if depth=1)
2. Both phase proofs valid
3. 7 DA confirmations valid
4. Lineage connects to previous (spend_record.previous_spend_cid)
5. Witness seed chain correctly derived
6. Witness VRF legitimacy and shard membership (per phase):
   For each witness_pk in witness_root:
     a. vrf_verify(witness_pk, phase_seed, vrf_output, vrf_proof) == true
     b. uint256(vrf_output) < VRF_THRESHOLD (hardcoded constant)
     c. witness_pk is unique within the set
     d. prefix_bits(H(witness_pk), declared_depth) ==
        prefix_bits(H("inat_shard:" || asset_id || phase_seed), declared_depth)
   e. declared_depth is consistent across all three phases
   Verified for W₁ (seed_1), W₂₃ (seed_23), W₄ (seed_4).

   Constraint 6d binds witnesses to the correct shard —
   a valid VRF proof alone is insufficient; the witness must
   also be in the shard derived from the phase seed.
   Without this, a Sybil in shard 0xAA could present a valid
   VRF proof for a seed mapping to shard 0xFF.
   Cost: bit comparison only — negligible constraint overhead.

   Constraint 6e prevents depth manipulation across phases.
   Combined with shard hopping (seed₂₃ depends on σ_r and
   beacon₂₃), an attacker must grind O(2^depth) NodeIDs per
   Sybil per shard — at depth=17, ~131K grinds per Sybil.
7. ISSUER ATTESTATION (Registry Root Model):
   - asset_id = H("inat_asset:" || asset_genesis_cid || name
     || params)  — stable identity, decoupled from issuer key
   - If depth=1:
       a. verify(issuer_pk, genesis_issuer_sig,
                 genesis_record) = true
       b. issuer_pk ∈ issuer_registry_root (Merkle proof)
       c. previous_registry_root == issuer_registry_root
          (no prior registry at genesis — self-referential)
       d. issuer_registry_sig is valid under issuer_pk
          (self-signed bootstrap — the genesis key authorizes
          the initial registry)
   - If depth>1:
       a. asset_id == previous_fold.asset_id
       b. issuer_pk ∈ issuer_registry_root (Merkle proof)
       c. previous_registry_root == previous_fold.issuer_registry_root
          (chain continuity — the circuit knows what the prior
          registry was)
       d. issuer_registry_sig contains ≥ REGISTRY_ROTATION_THRESHOLD
          valid signatures from keys in previous_registry_root
          (trust chain — new registry authorized by old registry)
          WHERE REGISTRY_ROTATION_THRESHOLD =
            max(REGISTRY_MIN_SIGNERS,
                ceil(previous_registry_key_count
                     * REGISTRY_ROTATION_THRESHOLD_NUMERATOR
                     / REGISTRY_ROTATION_THRESHOLD_DENOMINATOR))
       e. If issuer_registry_root == previous_registry_root:
          signature check (d) is trivially satisfied (no rotation).
          The circuit SHOULD still verify at least one valid
          signature to confirm registry liveness.
       f. issuer_registry_epoch >= previous_fold.issuer_registry_epoch
          (monotonic — registries cannot regress)
       g. issuer_registry_epoch is within REGISTRY_MAX_AGE_EPOCHS
          of current beacon epoch (staleness bound)
   NOTE: issuer_pk no longer must equal previous_fold.issuer_pk.
   Key rotation and multi-issuer are the same mechanism:
   registry with keys added/removed. Cost: ~21K constraints
   (~500-1000 for Merkle proof + ~20K for registry sig verify
   + ~2K for threshold check and epoch comparison).
8. issuer_family_id consistent across chain
9. FORWARD COMMIT CONSISTENCY:
   - spend_record.forward_commits[0] == recipient_commitment.recipient_forward_commit
   - spend_record.forward_commits[1] == sender_commitment.sender_forward_commit
   - Each verified as H("inat_slot:" || owner_sk || (base_seq + 2 * SEQ_STRIDE))
     inside the respective phase circuits (π₁ for sender, π₂₃ for recipient).
     Forward commit is SEQ_STRIDE past output, reserving space for
     the next transaction's phase entries.
10. OUTPUT CONSERVATION:
    - sender_output_amount + recipient_amount + fee == input_balance
    - output_commits[0] == recipient_slot_commit
    - output_commits[1] == sender_output_commit

11. ISSUER-GATED ELIGIBILITY: The signed eligible header is sourced from the attestation bundle (carried inline in WitnessEligibility responses), NOT from an Iroh blob. Like σ_s/σ_r, the header signature is witness-verified during attestation and bound into the fold hash chain (§33.1) rather than verified in-circuit — this avoids non-native BN254↔Ed25519 constraints.

    Witness-verified (during attestation, before fold):
    - Ed25519_Verify(issuer_pk, header_signing_payload, eligible_header_signature) == true
    - issuer_pk ∈ current AssetKeyRegistry (cached locally)

    Circuit-proven (inside fold):
    - header.registry_root == issuer_registry_root (same registry as this fold step)
    - header.registry_epoch == issuer_registry_epoch
    - header.epoch_number is within ELIGIBLE_ROOT_MAX_AGE_EPOCHS of beacon_round
    - header.asset_id == this fold's asset_id
    - For each witness_pk in w1_witness_root: Merkle membership proof valid against header.eligible_root
    - For each witness_pk in w23_witness_root: Merkle membership proof valid against header.eligible_root

    Third-party verification: At fold-verify time, the verifier natively checks the header signature (constant cost, ~microseconds) using header fields bound via §33.1 hash chain. The circuit-proven constraints are guaranteed by the fold SNARK verification itself.


12. WITNESS COUNT ENFORCEMENT:
    - len(w1_witness_root leaves) == WITNESS_TOTAL
    - len(w1_quorum attestations) >= WITNESS_QUORUM
    - Same enforced for w23

13. SIGNATURE BINDING (not verified in ZK — bound by hash):
    - H(sigma_s || sigma_r || sender_pk || recipient_pk) is included
      in fold_value (§33.1). This commits the signatures into the
      hash chain without performing non-native Ed25519 verification.
    - Third-party verifiers check σ_s and σ_r natively from fold
      public data (~microseconds per Ed25519 verify).
    - Security argument: witnesses verified signatures in real-time
      (§21.2 B1b/B1c). Fold binds the exact bytes they verified.
      A forged signature would require either (a) 13+ colluding
      witnesses who skip verification, or (b) replacing the
      signature bytes after fold — prevented by hash chain binding.

14. BEACON RECENCY CHAIN:
    - If previous_fold exists:
      beacon_round - previous_fold.final_beacon_round
      ≤ MAX_FOLD_BEACON_GAP_ROUNDS
    - Prevents temporal gaps in fold chain. Consecutive folds
      must advance at a rate consistent with real time.
    - If depth == 1 (genesis): no previous beacon to chain from.

PROVES (additional, for SWEEP-origin lineage):
15. SWEEP ACCUMULATOR CHAIN:
    If lineage.origin == SWEEP:
      a. sweep_record.previous_sweep_root is a valid Merkle root
      b. sweep_record.new_sweep_root incorporates all claimed leaves
      c. Each claimed leaf has a valid non-membership proof against
         previous_sweep_root
      d. sweep_record.protocol_seq == slot.seq
         (binds sweep to the seq that created this slot)
      e. sweep_record.sweep_epoch > previous sweep_epoch
         (monotonic)

OUTPUT: ~560 bytes proof (v1 attested hash-chain with eligibility data
+ registry membership proof), O(1) verification (<1ms)
```

### 33.1 Fold Implementation — v1 (Attested Hash-Chain)

v1 uses an attested hash-chain instead of recursive SNARKs.
The slot owner signs each fold step. Verification recomputes
the hash chain and checks the signature.

**Fold computation:**

    sig_binding = H(sigma_s || sigma_r || sender_pk || recipient_pk)
    header_binding = H(eligible_header_signature
                       || header.epoch_number
                       || header.eligible_root
                       || header.asset_id
                       || header.issuer_pk
                       || header.eligible_count
                       || header.issued_at
                       || header.registry_root
                       || header.registry_epoch)
    fold_value = H("inat_fold:" || H(prev_proof) || ownership_commit
                   || spend_binding || balance_commit || depth
                   || issuer_eligible_root || witness_tier
                   || issuer_registry_root || issuer_registry_epoch
                   || sig_binding || header_binding)
    signing_payload = H(fold_value || genesis_cid || issuer_family_id
                        || balance_commit || is_genesis
                        || previous_registry_root)
    fold_proof = b"INAT_FOLDED_PROOF:" || Sign(owner_sk, signing_payload)

For genesis (depth=1):

    fold_proof = b"INAT_GENESIS_PROOF:" || Sign(owner_sk, genesis_payload)

**Verification:**

    prefix = fold_proof[:18]  # or [:19]
    if prefix == b"INAT_GENESIS_PROOF:":
        # Recompute genesis payload, verify owner signature
    elif prefix == b"INAT_FOLDED_PROOF:":
        # Recompute hash chain from public inputs, verify owner signature

**Trust model:** The security assumption is the owner's signing
key — the same key that signs bilateral commitments (σ_s, σ_r).
Since a valid slot owner must sign all protocol phases anyway,
the fold attestation adds no new trust assumption.

This is weaker than SNARK-based IVC (where verification is
trustless), but equivalent to the protocol's existing trust
model for individual transactions.

**Wire format compatibility:** The `fold_proof` field in
CollapsedProof is opaque bytes. The verifier dispatches on prefix:

| Prefix | Version | Verification |
|--------|---------|-------------|
| `b"INAT_GENESIS_PROOF:"` | v1 | Recompute genesis hash, check signature |
| `b"INAT_FOLDED_PROOF:"` | v1 | Recompute hash chain, check signature |
| `b"INAT_IVC_PROOF:"` | v2 (future) | Nova IVC SNARK verification |
| FOLD | v1: N/A (attested hash-chain, no circuit). v2 (not planned): ~700k (includes VRF batch verify) | v1: no setup. v2: Phase 2 from shared Phase 1 SRS |
| VRF_BATCH | ~200k (v2 only, not planned) | Phase 2 from shared Phase 1 SRS |

v2 proofs can be introduced without breaking CollapsedProof
serialization. Old v1 proofs remain verifiable indefinitely.

### 33.2 Recursive Proofs (Not Planned)

v1's attested hash-chain fold is inductively secure: each step is
ZK-proven and witnessed by 21/29 quorum, so the chain is valid by
induction. Verification is <1ms (one Ed25519 signature check).

Recursive SNARKs (Nova IVC, etc.) would replace this with a single
SNARK covering all history — but at 200ms verification vs <1ms,
with an entirely different proving stack, for no security benefit.

The `fold_proof` field remains opaque bytes with prefix dispatch
(§33.1) if a future use case justifies recursive proofs. The wire
format is forward-compatible. But there is no planned v2.

## 34. Recovery Circuit

```
PUBLIC INPUTS:                 PRIVATE INPUTS:
• identity_commit              • owner_sk
• recovery_seq                 • base_seq (consumed slot's creation seq)
• balance_commit               • balance, blinding_factor
• R1_attestation_root          • lineage_proof
• R2_attestation_root          • R₁ attestations
• seed_R1                      • R₂ attestations
• seed_R2
• beacon_round_1
• beacon_round_2

PROVES:
1. Owner knows sk for identity_commit
2. balance_commit = Pedersen(balance, blinding_factor)
3. recovery_seq = base_seq + SEQ_STRIDE
   Recovery output lands at the same seq as a normal tx output.
   base_seq is the consumed slot's creation seq (= N).
   recovery_seq = N + SEQ_STRIDE = N + 4.
4. slot_commit = H("inat_slot:" || sk || base_seq)
   Proves ownership of the consumed slot.
5. Lineage valid up to base_seq
6. R₁ attestations valid (≥13 valid signatures)
7. R₂ attestations valid (≥13 valid signatures)
8. Each R₁ witness VRF-selected using seed_R1
9. Each R₂ witness VRF-selected using seed_R2
10. seed_R2 = H(seed_R1 || H(R1_attestation_root) 
              || beacon_round_2 || beacon_randomness_2)
11. Value preserved: balance(new) = balance(old)
```

## 35. Sweep Circuit, Collapsed Proof, Capacity Inputs & Circuit Summary

### Sweep Circuit

Consolidates multiple slots. For each input: verify slot derivation,
nullifier, amount commitment. Verify output slot, conservation:
total_input == output_balance + fee.

### Unified Sweep Circuit (π_sweep) — WITH ACCUMULATOR

"""
Unified circuit for both witness fee claims and donation guardian claims.
The circuit cannot distinguish the two claim types — both prove Merkle
membership in witness_root and claim 1 share per SpendRecord.

PUBLIC INPUTS:                      PRIVATE INPUTS:
• claimant_pk_commit                • claimant_sk
• claimed_amount                    • N spend_record_cids
• issuer_family_id                  • N merkle_proofs (pk ∈ witness_root)
• sweep_nullifier                   • N fee_commits
• new_slot_commit                   • fee_share_amount
• previous_sweep_root       [NEW]   • N sweep_leaf_preimages    [NEW]
• new_sweep_root            [NEW]   • N non-membership_proofs   [NEW]
• sweep_epoch               [NEW]   • previous_sweep_leaves     [NEW]
• protocol_seq              [NEW]

PROVES (existing):
1. claimant owns sk: H(derive_pk(sk)) = claimant_pk_commit
2. For each SpendRecord:
   a. pk ∈ witness_root (Merkle membership proof)
   b. SpendRecord is finalized
   c. issuer_family_id matches across all records
   d. my_share = fee_share_amount (1 share per record)
3. sum(my_shares) = claimed_amount
4. sweep_nullifier = H(pk || sweep_epoch || nonce)
5. Per-record nullifiers: H(pk || spend_record_cid) are fresh
6. new_slot_commit = H("inat_slot:" || claimant_sk || protocol_seq)
   Output MUST go to claimant's own wallet (accumulator co-location)
7. Sweep accumulator non-membership: each H(claimant_pk ||
   spend_record_cid) absent from previous sweep_root
8. New sweep_root = insert(previous_sweep_root, new_claim_leaves)
9. protocol_seq binding: sweep_root written atomically with seq
10. Epoch monotonicity: sweep_epoch > previous sweep_epoch

PROVES (new — accumulator chain):
7. PREVIOUS ROOT BINDING:
   previous_sweep_root is a valid Merkle root over
   previous_sweep_leaves. (If sweep_epoch == 0, must equal
   EMPTY_ROOT.)

8. NON-MEMBERSHIP — for each new claim:
   sweep_leaf_i = H("inat_sweep_leaf:" || claimant_pk
                     || spend_record_cid_i)
   Prove: sweep_leaf_i ∉ previous_sweep_root
   (via sorted-tree non-membership proof — leaf would be
   between two adjacent existing leaves, or outside bounds)

9. NEW ROOT CONSTRUCTION:
   new_sweep_root = Merkle(sort(previous_sweep_leaves
                                ∪ {sweep_leaf_0, ..., sweep_leaf_N-1}))
   The circuit recomputes the root from the full sorted leaf
   set and constrains it to equal the public new_sweep_root.

10. SEQ BINDING:
    protocol_seq matches the claimant's post-increment seq.
    Binds this sweep to a specific point in the wallet's
    monotonic history — cannot be replayed at a different seq.

11. EPOCH MONOTONICITY:
    sweep_epoch > previous_sweep_epoch
    (previous_sweep_epoch is derived from previous_sweep_root
    context, or 0 if first sweep)

12. OUTPUT WALLET BINDING:
    new_slot_commit = H("inat_slot:" || claimant_sk || protocol_seq)
    The output MUST be derived from claimant_sk — the same key
    that proves witness_root membership. No free target choice.

    RATIONALE: If the output could go to an arbitrary wallet,
    the accumulator (in the claimant's wallet doc) and the
    output (in some other wallet doc) would be decoupled.
    A guardian sweeping into wallet A, then wallet B after
    NullifierStore GC, would bypass the accumulator — because
    the accumulator lives in the pool_pk wallet doc, not in
    wallet A or B.

    By binding output to claimant_sk:
    • Accumulator, oracle seq, and output all live in ONE doc
    • Third-party verification reads ONE doc
    • Guardian/witness can still transfer out via normal tx

    This supersedes the earlier design where claimants could choose
    an arbitrary target wallet. Free target was a convenience
    feature that created a durability hole — the accumulator and
    output could end up in different documents, enabling re-sweep
    after NullifierStore GC.

OUTPUT:
• New slot with claimed_amount IN CLAIMANT'S OWN WALLET
• sweep_nullifiers published to NullifierStore (fast path)
• new_sweep_root written to wallet document (durable path)
• protocol_seq incremented in wallet document
"""

@dataclass
class SweepPublicInputs:
    """Public inputs for unified sweep circuit — with accumulator."""
    claimant_pk_commit: bytes       # H(claimant_pk)
    claimed_amount: int
    issuer_family_id: bytes
    sweep_nullifier: bytes          # H(pk || sweep_epoch || nonce)
    new_slot_commit: bytes          # MUST = H("inat_slot:" || claimant_sk || protocol_seq)
                                    # Output bound to claimant's own wallet (accumulator co-location)
    # --- Accumulator (new) ---
    previous_sweep_root: bytes      # Current sweep_root from wallet doc
    new_sweep_root: bytes           # After incorporating new claims
    sweep_epoch: int                # Monotonic sweep batch counter
    protocol_seq: int               # Post-increment seq

    def serialize(self) -> bytes:
        return msgpack.packb({
            "claimant_pk_commit": self.claimant_pk_commit,
            "claimed_amount": self.claimed_amount,
            "issuer_family_id": self.issuer_family_id,
            "sweep_nullifier": self.sweep_nullifier,
            "new_slot_commit": self.new_slot_commit,
            "previous_sweep_root": self.previous_sweep_root,
            "new_sweep_root": self.new_sweep_root,
            "sweep_epoch": self.sweep_epoch,
            "protocol_seq": self.protocol_seq,
        })

@dataclass
class SweepPrivateInputs:
    """Private witness for unified sweep circuit — with accumulator."""
    claimant_sk: bytes
    spend_record_cids: List[bytes]
    merkle_proofs: List[List[Tuple[bytes, str]]]
    fee_commits: List[bytes]
    fee_share_amount: int
    # --- Accumulator (new) ---
    previous_sweep_leaves: List[bytes]  # Full sorted leaf set of previous_sweep_root
    non_membership_proofs: List[        # Per new claim: adjacent leaves proving absence
        Tuple[Optional[bytes], Optional[bytes]]  # (left_neighbor, right_neighbor)
    ]

    def serialize(self) -> bytes:
        return msgpack.packb({
            "claimant_sk": self.claimant_sk,
            "spend_record_cids": self.spend_record_cids,
            "merkle_proofs": self.merkle_proofs,
            "fee_commits": self.fee_commits,
            "fee_share_amount": self.fee_share_amount,
            "previous_sweep_leaves": self.previous_sweep_leaves,
            "non_membership_proofs": self.non_membership_proofs,
        })


### Donation Circuit Variant

"""
Optional variant of π₁ and π₂₃ that adds pool_pk as 14th entry in
witness_root. Sender pays 14 × fee_share instead of 13. Witnesses
earn identically. User opts in via wallet UX.

PHASE 1 DONATION VARIANT (π₁_d):
  All standard π₁ constraints, plus:
  • witness_count = 14 (not 13)
  • witness_set[13] = pool_pk
  • H(pool_pk) == HARDCODED_POOL_ID (constant in circuit)
  • pool_pk is NOT VRF-verified (exempt from selection check)
  • total_fee = 14 × fee_share (not 13)
  • balance check: input >= amount + change + (14 × fee_share)

PHASE 2 DONATION VARIANT (π₂₃_d):
  All standard π₂₃ constraints, plus:
  • Same 14-entry witness_root binding
  • fee_commit covers 14 shares

FOLD CIRCUIT:
  Accepts either variant. The fold verifies "previous proof valid +
  current transition valid" regardless of whether the witness_root
  contains 13 or 14 entries. A slot's collapsed proof may contain a
  mix of donation and standard transactions in its folded history.

CIRCUIT OVERHEAD:
  • ~200 additional constraints for pool_pk verification
  • ~1% circuit size increase
  • Zero overhead when standard circuit is used
"""


### CollapsedProof

```python
@dataclass
class CollapsedProof:
    version: int = 3                # Bumped for final fixes
    current_slot_commit: bytes
    current_seq: int
    owner_pk_commit: bytes          # H(owner_pk) for oracle lookup
    nullifier_commit: bytes         # H(H("inat_nullifier:" || sk || seq))
    genesis_cid: bytes
    asset_genesis_cid: bytes            # Immutable asset identity anchor
    issuer_family_id: bytes
    issuer_pk: bytes                    # Current signing key (not identity anchor)
    issuer_registry_root: bytes         # Merkle root of authorized issuer PKs
    issuer_registry_epoch: int          # Epoch of the registry used for this fold step
    previous_registry_root: bytes       # Registry root from previous fold (or == issuer_registry_root at depth=1)
    depth: int
    witness_seed_chain: bytes
    final_beacon_round: int
    confirmation_root: bytes
    fold_proof: bytes
    # Bilateral signatures (not ZK-proven; bound into fold hash chain
    # for third-party native Ed25519 verification)
    sigma_s: bytes                      # Sender signature over H(sender_commitment)
    sigma_r: bytes                      # Recipient signature over H(recipient_commitment)
    sender_pk: bytes                    # For σ_s verification
    recipient_pk: bytes                 # For σ_r verification
    # Issuer eligibility (self-contained header, not a blob CID)
    issuer_eligible_root: bytes           # header.eligible_root for this fold step
    eligible_root_epoch: int              # header.epoch_number
    eligible_header_asset_id: bytes       # header.asset_id
    eligible_header_issuer_pk: bytes      # header.issuer_pk (signer)
    eligible_header_eligible_count: int   # header.eligible_count
    eligible_header_issued_at: int        # header.issued_at
    eligible_header_signature: bytes      # Issuer signature over header payload
    
    def verify(self) -> bool:
        """O(1) verification regardless of depth.
        Dispatches on fold_proof prefix (§33.1)."""
        if self.fold_proof.startswith(b"INAT_GENESIS_PROOF:"):
            return self._verify_genesis_proof()
        elif self.fold_proof.startswith(b"INAT_FOLDED_PROOF:"):
            return self._verify_attested_fold()
        elif self.fold_proof.startswith(b"INAT_IVC_PROOF:"):
            return self._verify_ivc_fold()  # v2 future
        return False

    def _verify_attested_fold(self) -> bool:
        """v1: Recompute hash chain from public inputs, verify owner signature,
        verify bilateral signatures natively."""
        signature = self.fold_proof[len(b"INAT_FOLDED_PROOF:"):]
        expected_payload = self._compute_fold_signing_payload()
        if not Ed25519_Verify(self._owner_pk_from_commit(), expected_payload, signature):
            return False
        # Native bilateral signature verification (~microseconds each)
        # These were witness-verified at attestation time (§21.2 B1b/B1c)
        # and hash-bound into fold_value. Verify here for third-party assurance.
        sender_commit_hash = H(self._sender_commitment_bytes())
        if not Ed25519_Verify(self.sender_pk, sender_commit_hash, self.sigma_s):
            return False
        recipient_commit_hash = H(self._recipient_commitment_bytes())
        if not Ed25519_Verify(self.recipient_pk, recipient_commit_hash, self.sigma_r):
            return False
        return True

    def public_inputs(self) -> bytes:
        return msgpack.packb({
            "slot_commit": self.current_slot_commit, "seq": self.current_seq,
            "owner_pk_commit": self.owner_pk_commit,
            "nullifier_commit": self.nullifier_commit,
            "genesis": self.genesis_cid,
            "asset_genesis_cid": self.asset_genesis_cid,
            "issuer": self.issuer_pk,
            "issuer_registry_root": self.issuer_registry_root,
            "depth": self.depth, "seed_chain": self.witness_seed_chain,
            "beacon": self.final_beacon_round, "conf_root": self.confirmation_root,
        })
```

### Third-Party Verification

Phase 1 (liveness, ~10ms): binding check → nullifier check → protocol_seq check
Phase 2 (value, <1ms): verify fold_proof — recompute hash chain + Ed25519 owner signature check + native Ed25519 verification of σ_s and σ_r from fold public data (~2 additional microseconds)

### Capacity Proof Inputs

```python
@dataclass
class CapacityPublicInputs:
    """Public inputs for capacity proof circuit."""
    slot_set_root: bytes
    issuer_family_id: bytes
    min_amount: int
    owner_pk_commit: bytes
    beacon_round: int

    def serialize(self) -> bytes:
        return msgpack.packb({
            "slot_set_root": self.slot_set_root,
            "issuer_family_id": self.issuer_family_id,
            "min_amount": self.min_amount,
            "owner_pk_commit": self.owner_pk_commit,
            "beacon_round": self.beacon_round,
        })

@dataclass
class CapacityPrivateInputs:
    """Private witness for capacity proof circuit."""
    owner_sk: bytes
    chosen_slot_commit: bytes
    chosen_slot_seq: int
    chosen_slot_balance: int
    chosen_slot_balance_blinding: bytes
    chosen_slot_asset_id: bytes
    merkle_index: int
    merkle_proof: List[Tuple[bytes, str]]
    all_slot_commits: List[bytes]

    def serialize(self) -> bytes:
        return msgpack.packb({
            "owner_sk": self.owner_sk,
            "chosen_slot_commit": self.chosen_slot_commit,
            "chosen_slot_seq": self.chosen_slot_seq,
            "chosen_slot_balance": self.chosen_slot_balance,
            "chosen_slot_balance_blinding": self.chosen_slot_balance_blinding,
            "chosen_slot_asset_id": self.chosen_slot_asset_id,
            "merkle_index": self.merkle_index,
            "merkle_proof": self.merkle_proof,
            "all_slot_commits": self.all_slot_commits,
        })
```

### Proof System Summary

Phase circuits: PLONK (BN254), ~2.5KB each, ~5ms verification.
Universal trusted setup: powers-of-tau only. No per-circuit ceremony.
Fold: attested hash-chain (~364 bytes, <1ms verification).

Constraint counts (Groth16 circuits):
sender: ~55k (includes tier range proof), combined: ~65k (signatures witness-verified, not circuit-proven),
sweep: ~200k (variable), recovery: ~120k, capacity: ~15k,
donation_sender: ~56k, donation_combined: ~66k (signatures witness-verified).
Fold (v1): no circuit (attested hash-chain, <1ms verification).
Fold (v2, not planned): ~200k constraints (includes eligibility Merkle proofs + issuer sig).

VERIFICATION_KEYS / PROVING_KEYS exist for:
  sender, combined, sweep, recovery, capacity,
  donation_sender, donation_combined

Fold: no setup required (signature-based).

### Circuit Relationship Diagram

```
Phase 1 (INITIATE):
  Sender generates π₁ (sender_circuit)
    → proves: sk derives slot_commit/nullifier, balance conservation, lineage valid,
      forward commit, poly binding. Does NOT prove σ_s or witness binding.
  W₁ verifies π₁ → Attests (holds shares, doesn't release)

Phase 2 (COMMIT):
  Recipient generates π₂₃ (combined_circuit)
    → proves: same amount_commit (no inflation), recipient binds to sender's
      nullifier, seed₂₃ correctly derived with H(σ_r), lineage verified,
      bilateral binding via hash linkage and proof-of-knowledge
  W₂₃ verifies π₂₃ + verifies σ_s and σ_r natively (§21.2 B1b/B1c) → Attests
  ╔═══════════════════════════════════════════════╗
  ║  POINT OF NO RETURN: W₂₃ quorum achieved      ║
  ╚═══════════════════════════════════════════════╝

Phase 3 (FINALIZE):
  W₁ releases shares → Nullifier reconstructed → Published
  (No new ZK proof — uses Feldman VSS verification)

Phase 4 (CONFIRM):
  Anyone generates collapsed proof (fold_circuit)
    → proves: previous fold valid (or genesis), π₁ and π₂₃ valid,
      7 DA confirmations, seed chain correct, O(1) proof regardless of depth
  W₄ confirms DA → Fold generated → FINALIZED

┌────────────┬─────────────────────────────────────────────────────────┐
│ Circuit    │ Guarantees                                              │
├────────────┼─────────────────────────────────────────────────────────┤
│ π₁ sender  │ Sender owns slot, balance conserved, lineage valid,    │
│            │ forward commit, poly binding. No σ_s, no witness bind. │
│ π₂₃ comb. │ Same amount (no inflation), bilateral binding,         │
│            │ W₁ set binding, seed chain (H(σ_r)), recipient forward │
│            │ commit. σ_s/σ_r witness-verified + fold-bound (§33.1)  │
│ fold       │ Entire history valid, O(1) regardless of depth,        │
│            │ issuer-attested genesis, all witnesses legitimate       │
│ sweep      │ Witness earned fees legitimately, no double-sweep       │
│ recovery   │ Witnessed abort, balance preserved, seq advanced        │
│ capacity   │ Owner has slot with correct issuer, balance ≥ min,     │
│            │ slot in owner's set (without revealing which/how many)  │
│ π_sweep   │ Claimant pk ∈ witness_root of N SpendRecords,       │
│            │ accumulated shares, issuer consistency, deferred mint │
│ π₁_d      │ Standard π₁ + pool_pk as 14th witness entry,        │
│            │ 14 × fee_share balance conservation                  │
│ π₂₃_d     │ Standard π₂₃ + 14-entry witness_root binding        │
└────────────┴─────────────────────────────────────────────────────────┘
```

---

# Part VIII: Storage

### 36.1 Storage Layers

```
┌──────────────────────────────────────────────────────────┐
│  IROH DOCUMENTS (per-wallet, append-only, monotonic)     │
│  ─────────────────────────────────────────────────────── │
│  • One document per wallet (identity)                    │
│  • Owner is sole writer (per-author monotonic sequencing)│
│  • Witnesses + counterparties get read tickets           │
│  • protocol_seq stored as value, incremented by owner    │
│  • Doc replication via XOR-sharded subscriptions         │
│                                                          │
│  IROH BLOBS (content-addressed, immutable)               │
│  ─────────────────────────────────────────────────────── │
│  • SpendRecords, collapsed proofs, witness bundles       │
│  • Written once, referenced by CID from doc entries      │
│  • Consumed by fold circuit, GC-safe after folding       │
│                                                          │
│  NULLIFIER SET (ephemeral, real-time)                    │
│  ─────────────────────────────────────────────────────── │
│  • Nullifiers during active transactions (race detection)|
|  • XOR-sharded across responsible peers (§40)            │
│  • GC-safe once oracle seq advances                      │
└──────────────────────────────────────────────────────────┘
```

### 36.2 Wallet Document Key Schema


### Witness Fetch Key Paths

When a witness independently fetches mutable state (§21.2),
it reads doc entries from the wallet document:

| Check | Method | Source |
|-------|--------|--------|
| Protocol sequence | Read latest entry's `protocol_seq` field | Entry at max `seq:*` key |
| Slot status | Infer from latest entry's `event_type` matching `session_id` | Scan recent entries |
| Slot data (asset_id) | Fetch phase blob via `phase_data_cid` from relevant entry | Blob store |
| Session binding | Find entry with matching `session_id` | Scan entries |
| Amount blinding (enc) | Read from PENDING_SEND phase blob | Blob store via `phase_data_cid` |

**Phase 2 dual document fetch:** In Phase 2, witnesses read entries
from BOTH the sender's and recipient's wallet documents. The sender's
document provides: latest entry (protocol_seq, event_type for slot
status), phase blob (sender commitment, asset_id). The recipient's
document provides: latest entry (protocol_seq, event_type).

**Protocol_seq verification rule:** Readers MUST verify that the
`protocol_seq` value in entry content equals the Iroh entry's
sequence position. Mismatch → reject (corrupted or malicious doc).

Oracle sequence is the `protocol_seq` from the latest entry.
Accessible via doc subscription (local), gossip cache, or RPC
to XOR-nearest peers (§45.6).


```python
class WalletDocSchema:
    """
    Single Iroh document per wallet. Owner is sole writer.

    ENTRY MODEL: Each doc entry is exactly one protocol_seq increment.
    protocol_seq value = Iroh entry sequence number (by construction).
    Rollback is structurally impossible — the append-only hash-chained
    log makes it unforgeable. Readers verify protocol_seq == entry
    position from Iroh entry metadata.

    RULE: ONE doc.set() call per phase transition. All supplementary
    data (proofs, commitments, attestations, session state, encrypted
    blinding factors) goes in Iroh blobs, referenced by CID from the
    entry content. This guarantees protocol_seq increments exactly
    once per meaningful state transition.

    Entry content is msgpack-encoded with the following schema:

        {
            "protocol_seq": int,           # == Iroh entry sequence number
            "event_type": str,             # MINT | PENDING_SEND | PENDING_RECEIVE |
                                           #   COMMITTED | SPENT | FINALIZED |
                                           #   RECOVERED | BURNED | SWEEP
            "slot_commit": bytes(32),      # H(sk || output_seq) for slot-creating events
            "session_id": bytes(32),       # Links entries across parties
            "beacon_round": int,
            "phase_data_cid": bytes(32),   # → blob: phase-specific payload (see below)
            "previous_entry_hash": bytes(32),  # H(entry N-1 content) — ZK-bindable chain
        }

    Phase-specific blobs (referenced by phase_data_cid):

        PENDING_SEND blob: {
            sender_commitment, sigma_s, proof_1, public_inputs_1,
            poly_commits, witness_seed_1, witnesses_1, w1_reply_paths,
            attestations_1, pairing, encrypted_amount_blinding
        }

        PENDING_RECEIVE blob: {
            sender_commitment, recipient_commitment, sigma_s, sigma_r,
            proof_1, public_inputs_1, witness_seed_1, witnesses_1,
            w1_reply_paths, attestations_1, pairing
        }

        COMMITTED blob: {
            proof_23, public_inputs_23, witness_seed_23,
            witnesses_23, attestations_23
        }

        SPENT blob: {
            spend_record_cid, nullifier_cid, witness_bundle_cid
        }

        FINALIZED blob: {
            collapsed_proof_cid, confirmation_root
        }

        RECOVERED blob: {
            recovery_record_cid, R1_attestation_root, R2_attestation_root
        }

        SWEEP blob: {
            sweep_record_cid, new_sweep_root, sweep_epoch
        }

    Status is inferred from the latest entry's event_type, not stored
    as a separate key. Witnesses read the latest entry + fetch the
    phase blob for verification material.

    SWEEP ACCUMULATOR: sweep_root, sweep_count, sweep_epoch are
    fields in SWEEP-type entry blobs, not separate doc keys.
    The current values are derived from the latest SWEEP entry.
    """

    # No per-key statics. All state is in entries.
    # The following helpers exist for entry content construction only.

    @staticmethod
    def entry_content(
        protocol_seq: int,
        event_type: str,
        slot_commit: bytes,
        session_id: bytes,
        beacon_round: int,
        phase_data_cid: bytes,
        previous_entry_hash: bytes,
    ) -> bytes:
        """Construct a single doc entry. One call to doc.set() per phase."""
        return msgpack.packb({
            "protocol_seq": protocol_seq,
            "event_type": event_type,
            "slot_commit": slot_commit,
            "session_id": session_id,
            "beacon_round": beacon_round,
            "phase_data_cid": phase_data_cid,
            "previous_entry_hash": previous_entry_hash,
        })

    @staticmethod
    def parse_entry(data: bytes) -> dict:
        return msgpack.unpackb(data)

    @staticmethod
    def entry_key(seq: int) -> str:
        """Doc key for entry at this seq. Enables direct lookup."""
        return f"seq:{seq}"

    # --- Convenience accessors ---
    # These derive state from entries. They are NOT separate doc keys.
    # Code referencing slot_status(), tx_state(), etc. MUST use these
    # helpers, which scan entries to reconstruct the requested view.

    @staticmethod
    def slot_status(slot_hex: str) -> str:
        """Lookup key alias. Callers scan recent entries for matching
        slot_commit and return the latest event_type."""
        return f"_derived:slot_status:{slot_hex}"

    @staticmethod
    def slot_data(slot_hex: str) -> str:
        return f"_derived:slot_data:{slot_hex}"

    @staticmethod
    def slot_lineage(slot_hex: str) -> str:
        return f"_derived:slot_lineage:{slot_hex}"

    @staticmethod
    def tx_state(session_hex: str) -> str:
        return f"_derived:tx_state:{session_hex}"

    @staticmethod
    def tx_role(session_hex: str) -> str:
        return f"_derived:tx_role:{session_hex}"

    @staticmethod
    def tx_amount_blinding(session_hex: str) -> str:
        return f"_derived:tx_amount_blinding:{session_hex}"

    @staticmethod
    def sweep_leaf_set(epoch: str) -> str:
        return f"_derived:sweep_leaf_set:{epoch}"

    @staticmethod
    def sweep_proof_cid(epoch: str) -> str:
        return f"_derived:sweep_proof_cid:{epoch}"

    # NOTE ON IMPLEMENTATION: These "_derived:" prefixes are placeholders.
    # The implementation MUST resolve them by scanning doc entries
    # (filtering by slot_commit or session_id from entry content)
    # and fetching the relevant phase_data_cid blob. There are no
    # separate doc keys for slot status, tx state, or sweep state —
    # all state is reconstructed from the append-only entry log.
    #
    # Sweep accumulator state (sweep_root, sweep_count, sweep_epoch)
    # is derived from the latest SWEEP-type entry's phase blob.
    # The §37 genesis blob initializes these to zero values;
    # subsequent SWEEP entries advance them.
```


## 37. Document Access & Sync

```python
DOCUMENT_ACCESS = {
    "owner": DocumentAccess.OWNER,          # Read + Write
    "counterparty": DocumentAccess.READ_ONLY,
    "witnesses": DocumentAccess.READ_ONLY,
    "public": DocumentAccess.NONE,
}


class DocumentSyncManager:
    """Manages wallet Iroh document lifecycle."""

    def __init__(self, iroh_node: IrohNode):
        self.iroh = iroh_node

    async def create_wallet_doc(self, pk: bytes) -> Doc:
        """Create wallet document. Owner is sole writer.
        Genesis entry at seq=0 establishes identity."""
        doc = await self.iroh.docs.create()
        identity_commit = H(pk)

        # Genesis blob: identity metadata
        # Genesis blob establishes initial sweep accumulator state.
        # These values are the starting point; subsequent SWEEP-type
        # entries advance them. There are no separate doc keys —
        # current sweep state is derived from the latest SWEEP entry
        # (or from this genesis blob if no sweeps have occurred).
        genesis_blob = msgpack.packb({
            "pk_commit": identity_commit,
            "created_at": current_timestamp(),
            "sweep_root": EMPTY_ROOT,
            "sweep_count": 0,
            "sweep_epoch": 0,
        })
        genesis_blob_cid = await self.iroh.blobs.add(genesis_blob)

        # Single entry at seq=0
        entry = WalletDocSchema.entry_content(
            protocol_seq=0,
            event_type="GENESIS",
            slot_commit=b"\x00" * 32,
            session_id=b"\x00" * 32,
            beacon_round=0,
            phase_data_cid=genesis_blob_cid,
            previous_entry_hash=b"\x00" * 32,  # No predecessor
        )
        await doc.set(WalletDocSchema.entry_key(0), entry)
        return doc

    async def share_read_access(self, doc: Doc, keys: List[str]) -> str:
        """Read ticket for counterparty/witnesses."""
        ticket = await doc.share(mode=ShareMode.READ, key_filter=keys)
        return ticket.to_string()

    async def join_document(self, doc_id: str, ticket: str) -> Doc:
        return await self.iroh.docs.join(doc_id, ticket=Ticket.from_string(ticket))
```

## 38. Content Store Layout (Iroh Blobs)

# Canonical blob path schema. All immutable, referenced by CID from wallet docs.
```
BLOB_PATHS = {
    # Nullifiers (Phase 3 output, GC-safe after oracle advances)
    "nullifier/{nullifier_cid}":                       "nullifier preimage (32 bytes)",

    # Transaction records (input to fold circuit, GC-safe after folding)
    "spend/{spend_cid}":                               "SpendRecord (~1.8KB)",

    # Witness data
    "share/{session_id}/{witness_index}":              "PublishedShare",
    "witness_bundle/{session_id}":                     "WitnessBundle (refs spend_cid; SpendRecord does NOT back-ref)",

    # Phase data blobs (referenced by phase_data_cid from doc entries)
    "phase/{session_id}/{event_type}":                 "Phase-specific data blob (~1-10KB, GC-safe after fold)",

    # Proofs
    "proof/{proof_cid}":                               "ZK proof bytes",
    "collapsed/{slot_commit}":                         "CollapsedProof (~22KB)",

    # DA confirmations
    "confirmation/{session_id}/{validator_pk_commit}": "Confirmation",

    # Issuer key registries
    "registry/{asset_id}/{epoch}":                     "AssetKeyRegistry (~300 bytes)",
    "registry_rotation/{asset_id}/{epoch}":            "IssuerKeyRotation (variable)",
}
```

## 39. Persistence Hierarchy

```
MUST PERSIST (Loss = Permanent Fund Loss):
  • Wallet Document             ~1KB/identity (Iroh doc, k=50 replicas) + sweep accumulator
                                Contains: protocol_seq, slot state, lineage.
                                Owner is sole writer. Monotonic.
        Contains: protocol_seq, slot state, lineage,
        sweep_root, sweep_count, sweep_epoch.
        Sweep accumulator grows at ~32 bytes per swept SpendRecord.
        At 1000 sweeps: +32KB. At 10000: +320KB.
        Within MAX_STORAGE budget.
  • Current Slot Proofs         ~364 bytes each (v1 collapsed proof, Iroh blobs)
  • User's secret key           32 bytes

REAL-TIME ONLY (Active Transactions):
  • Iroh-blob Nullifiers             ~32 bytes/tx
    Race detection during attestation window.
    GC-safe once oracle sequence advances past creation_seq.

SHOULD PERSIST (Loss = Verification Delay):
  • Recent Spend Records        ~2KB/tx
  • Collapsed Proofs            ~22KB each

MAY GARBAGE COLLECT (After Proof Folding):
  • Old spend records, old witness bundles, intermediate proofs
  • Old Iroh-blob nullifiers (oracle has advanced past them)
  • ForcedFinalizationRecords — GC-safe once recipient slot reaches
    FINALIZED (CollapsedProof absorbs all evidence). MUST NOT GC
    before FINALIZED. W_FF witnesses retain for
    FORCED_FINALIZATION_RETENTION_WINDOW as a safety margin.
  • Session state (after finalization)
  • Far-shard oracle records evicted by LRU
  • Archived sweep leaf sets older than SWEEP_ACCUMULATOR_GC_EPOCHS
    (current sweep_root is self-sufficient for proofs)
  • NullifierStore sweep entries (now a fast-path cache, not
    the sole gate — accumulator is authoritative)

RECENT ONLY (Crash Recovery):
  • Write-ahead log for in-flight transactions (NOT replicated)
```

---

# Part IX: Iroh-blob Integration


### 40.1 Purpose

Nullifiers provide race detection during the attestation window.
The NullifierStore is an XOR-sharded distributed set of
nullifier_commits.

**Trust model:** The NullifierStore is NOT authoritative for
spend status. The oracle is authoritative for liveness (§26).
A negative NullifierStore lookup does NOT guarantee unspent.
A positive lookup is strong evidence of prior spend.

### 40.2 Operations

| Operation | Input | Output | Description |
|-----------|-------|--------|-------------|
| publish | nullifier (32 bytes) | cid | Store blob of nullifier preimage. Push H(nullifier) to responsible peers. |
| publish_forced | nullifier_commit (32 bytes), proof_cid | cid | Forced publication (no preimage). Push nullifier_commit to responsible peers. |
| exists | nullifier_commit (32 bytes) | bool | Check existence across tiers. |

### 40.3 Replication

On publish, the publisher pushes nullifier_commit to the
NULLIFIER_RESPONSIBLE_PEERS closest nodes (by XOR distance
to nullifier_commit). These peers store the nullifier_commit
in a local set and answer exists() queries.

| Parameter | Value |
|-----------|-------|
| NULLIFIER_RESPONSIBLE_PEERS | 20 |
| NULLIFIER_QUERY_PEERS | 5 |
| NULLIFIER_QUERY_TIMEOUT | 3.0s |

### 40.4 Existence Check

Tiered lookup:

| Tier | Source | Latency |
|------|--------|---------|
| 1 | Local set (this node was pushed to, or published) | <1ms |
| 2 | Local positive cache (previous remote hit) | <1ms |
| 3 | RPC to XOR-responsible peers | ≤3s |

First positive result at any tier → return true.
All tiers negative → return false (subject to trust model above).

### 40.5 RPC Interface

| Method | Request | Response | Description |
|--------|---------|----------|-------------|
| inat/nullifier/exists | nullifier_commit (32) | 0x01 or 0x00 | Peer checks local set |
| inat/nullifier/store | nullifier_commit (32) | 0x01 | Peer stores in local set |


## 41. Spend Record Archive

```python
class SpendRecordArchive:
    async def publish(self, record: SpendRecord) -> str:
        return await self.Iroh_blob.add(content=record.serialize(), pin=True)

    async def fetch(self, cid: str) -> Optional[SpendRecord]:
        try:
            data = await self.Iroh_blob.get(cid)
            return SpendRecord.deserialize(data)
        except NotFoundError: return None

    async def verify_lineage(self, slot: Slot, depth: int = 10) -> LineageVerificationResult:
        """
        Verify slot's lineage chain back to mint or max depth.
        """
        current = slot.lineage
        chain = []

        for _ in range(depth):
            if current.origin == LineageOrigin.MINT:
                valid = await self._verify_mint(current)
                return LineageVerificationResult(valid=valid, chain=chain, origin=LineageOrigin.MINT)

            spend_record = await self.fetch(current.spend_record_cid)
            if not spend_record:
                return LineageVerificationResult(valid=False, error="Missing spend record")

            bundle = await self._fetch_witness_bundle(current.witness_bundle_cid)
            if not bundle or not bundle.is_valid:
                return LineageVerificationResult(valid=False, error="Invalid witness bundle")

            chain.append(spend_record)
            # continue chain...

        return LineageVerificationResult(valid=True, chain=chain, truncated=True)
```
---

# Part X: Sequence Oracle Implementation


## 42. Constants

```python
# --- Sequence Oracle ---
REDUNDANCY_TARGET = 50              # Wallet doc subscriptions per node (XOR nearest)
NEIGHBORHOOD_SIZE = 20              # XOR neighbors for Pulse
MAX_SUBSCRIBED_DOCS = 5_000_000     # Upper bound on doc subscriptions per node
MAX_STORAGE = 1 * GB                # ~1 GB per node
PULSE_INTERVAL = 300                # 5 minutes
GOSSIP_FAN_OUT = 8
GOSSIP_TTL = 6

# --- Transaction Protocol ---
WITNESS_TOTAL = 29                  
WITNESS_QUORUM = 21                 
DA_VALIDATORS = 7
MAX_BEACON_AGE_ROUNDS = 20          # ~10 minutes
BEACON_ROUND_DURATION = 30          # seconds
RECOVERY_TIMEOUT = 3600             # 1 hour
ATTESTATION_TIMEOUT = 30            # seconds
ELIGIBILITY_COLLECTION_TIMEOUT = 10 # seconds (envelope → eligibility round)
PAYLOAD_ATTESTATION_TIMEOUT = 20    # seconds (payload → attestation round)
SHARE_DELIVERY_TIMEOUT = 30         # seconds (sender → W₁ share delivery)
W1_FORWARD_TIMEOUT = 120            # seconds after W₂₃ quorum before recipient
                                    # independently forwards W₂₃ bundle to W₁ (§20.2 step 20)
N_MIN = 100_000                      # Minimum network size for activation.
                                    # Below N_MIN: bootstrap mode, transactions rejected.
                                    # Used by: BootstrapManager (activation gate),
                                    # VRFDensityGate (threshold computation),
                                    # DynamicParameters (first network tier floor).
FEE_SHARE = 10                      # Per-witness. Total fee = tier.quorum × FEE_SHARE
SEQ_STRIDE = 4                      # Protocol_seq increments per party per transaction.
                                    # One entry per phase transition:
                                    #   +1: initiate (PENDING_SEND / PENDING_RECEIVE)
                                    #   +2: commit (COMMITTED)
                                    #   +3: finalize (SPENT / observed nullifier)
                                    #   +4: confirm (FINALIZED → output slot materializes)
                                    # Output slot at base + SEQ_STRIDE.
                                    # Forward commit at base + 2*SEQ_STRIDE.
                                    # Recovery uses remaining entries (max 3) to reach
                                    # base + SEQ_STRIDE regardless of abort phase.
                                    # INVARIANT: Every doc write = exactly one seq increment.
                                    # Supplementary data goes in blobs; doc entry carries CIDs.
VRF_THRESHOLD = (VRF_TARGET_WITNESSES / SHARD_TARGET_SIZE)  # × 2^256 in field
# Hardcoded. Shard depth absorbs network growth.
# Never recomputed. Never density-dependent.

# --- Issuer-Gated Eligibility ---
ISSUER_ELIGIBLE_ROOT_EPOCH = 3600   # 1 hour
ELIGIBLE_ROOT_MAX_AGE_EPOCHS = 2    # Grace period for in-flight txns.
                                    # Bounds header freshness at verification:
                                    # header.epoch_number must be within this
                                    # many epochs of current beacon epoch.
MIN_WITNESS_BALANCE = 10            # 0.1 tokens (in base units, assuming 100 base = 1 token)
ISSUER_MAX_GROWTH_RATE = 0.05       # Max 5% identity growth per epoch

# --- Issuer Key Registry ---
REGISTRY_ROTATION_THRESHOLD_NUMERATOR = 1    # M in M-of-N: ceil(N * NUM / DEN)
REGISTRY_ROTATION_THRESHOLD_DENOMINATOR = 2  # Majority by default
REGISTRY_MIN_SIGNERS = 1                     # Floor (single-key registries)
REGISTRY_MAX_AGE_EPOCHS = 3         # Max registry staleness in eligible root epochs.
                                    # In-flight folds may reference a registry up to
                                    # REGISTRY_MAX_AGE_EPOCHS old. After that, the fold
                                    # circuit rejects it. Must be >= ELIGIBLE_ROOT_MAX_AGE_EPOCHS.
REGISTRY_PUBLISH_TOPIC_PREFIX = "inat/registry/v1/"  # + asset_prefix hex

# --- Beacon Recency Chain ---
MAX_FOLD_BEACON_GAP_ROUNDS = 2400   # Max rounds between consecutive folds (~20 hours)

# --- Forced Finalization ---
FORCED_FINALIZATION_TIMEOUT = 259200        # 72 hours (seconds)
FORCED_FINALIZATION_WITNESSES = 29
FORCED_FINALIZATION_THRESHOLD = 21
FORCED_FINALIZATION_REPLICATION_PEERS = 20  # Peers to push ForcedFinalizationRecord blob to
FORCED_FINALIZATION_RETENTION_WINDOW = 86400 # 24h after W_FF; covers Phase 4 + fold window

# --- Discovery Protocol ---
PAYMENT_TOKEN_DEFAULT_TTL = 3600
MAX_CAPACITY_PROOF_AGE_ROUNDS = 40
MAX_SLOT_SET_SIZE = 256
LOCK_PROOF_TTL = 300
DISCOVERY_TIMEOUT = 120
DISCOVERY_STEP_TIMEOUT = 30

# --- Gossip Topics ---

# Replace N_MIN with per-asset minimum
ASSET_MIN_SUPPORTERS = 100_000   # Per-asset activation threshold

# Remove global N_MIN (no longer meaningful — activation is per-asset)
# N_MIN = 100_000  # REMOVED

# Shard depth hysteresis
SHARD_SPLIT_THRESHOLD = 2400     # 1.2 × SHARD_MAX_DENSITY
SHARD_MERGE_THRESHOLD = 600      # 0.8 × SHARD_MIN_DENSITY  
DEPTH_CHANGE_COOLDOWN = 600      # seconds

# INAT_SHARES_TOPIC is NOT a single global topic.
# Share publications are scoped to a session-derived shard.

def shares_shard_topic(seed_1: bytes, shard_depth: int) -> bytes:
    """Derive shares topic from Phase 1 seed.
    Shares publish to the same shard as Phase 1 witnesses (W₁),
    since W₁ witnesses are the publishers."""
    shard_bits = int.from_bytes(H("inat_shares_shard:" || seed_1)[:4], 'big')
    prefix = shard_bits >> (32 - shard_depth)
    return f"inat/shares/v1/{prefix:0{shard_depth}b}".encode()
INAT_ORACLE_TOPIC = b"inat/oracle/v1"

# --- Donation System ---
DONATION_RECEIPT_SIZE = 104             # bytes
DONATION_RECEIPT_RESPONSIBLE_PEERS = 20
DONATION_RECEIPT_QUERY_TIMEOUT = 5.0    # seconds
# HARDCODED_POOL_ID set at protocol genesis

# --- Sweep ---
SWEEP_COUNT_THRESHOLD = 1000
SWEEP_TIME_THRESHOLD = 604800           # 7 days (seconds)

# --- Sweep Accumulator ---
SWEEP_LEAF_DOMAIN = b"inat_sweep_leaf:"
MAX_SWEEP_BATCH_SIZE = 5000        # Max claims per sweep proof
                                    # (circuit constraint count scales
                                    # linearly — ~40 constraints per
                                    # non-membership proof)
SWEEP_ACCUMULATOR_GC_EPOCHS = 100  # Archive leaf sets older than this
                                    # can be pruned. The current
                                    # sweep_root is sufficient for
                                    # non-membership proofs — archived
                                    # leaf sets are convenience only.

```

## 43. Identity Commit (Wallet Lookup Key)

```python
def identity_commit(pk: bytes) -> bytes:
    """Canonical lookup key for wallet documents. H(pk) is the wallet's
    identity in the XOR keyspace — determines which peers subscribe
    to this wallet's document."""
    return H(pk)
```

## 44. Wire Format Structures


### TransactionIntent

### IntentEnvelope

Broadcast to gossip shard. Received by all ~1000 Inat nodes in shard.
Contains ONLY the fields needed for VRF self-selection and basic
deduplication. No identity, asset, or proof data.

**Fields (all phases):**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| phase | Phase | 1 | INITIATE, COMMIT, FINALIZE, or CONFIRM |
| session_id | bytes | 32 | Random per transaction |
| seed | bytes | 32 | VRF lottery seed for this phase |
| asset_id | bytes | 32 | asset being witnessed |
| shard_depth | uint16 | 2 | Network-consensus depth |
| reply_path | bytes | var | Opaque return address (§25) |
| beacon_round | uint64 | 8 | |
| nullifier_commit | bytes | 32 | Unique per tx — safe for dedup |
| observed_density | uint32 | 4 | Sender's local density hint — used ONLY for sender-side ±20% sanity filtering of eligibility responses (§20.1 step 15). NEVER used for VRF threshold computation by any party. Verifiers derive threshold from their own heartbeat count. |


asset_id lets a node that somehow receives an envelope for an asset it doesn't support reject it immediately — before VRF evaluation.

**Privacy property:** No field in IntentEnvelope is stable across
transactions from the same identity. session_id and nullifier_commit
are unique per transaction. An observer learns only "a transaction
is happening in this shard at this time."


### IntentPayload

Delivered individually to VRF-selected witnesses ONLY, via their
reply_path from WitnessEligibility. NOT broadcast.

**Common fields (all phases):**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| session_id | bytes | 32 | Must match envelope |
| slot_doc_id | string | var | Primary wallet document ID — witness fetches independently |
| oracle_key | bytes | 32 | H(pk) of primary party — witness fetches independently |
| claimed_slot_commit | bytes | 32 | Cross-checked against fetched state |
| claimed_asset_id | bytes | 32 | Cross-checked against fetched state |

**Phase 1 additional fields:**

| Field | Type | Description |
|-------|------|-------------|
| zk_proof | bytes | π₁ per §31 |
| public_inputs | bytes | Serialized SenderPublicInputs |
| poly_commits | List[bytes(32)] | Feldman VSS polynomial commitments |
| expected_seq | uint64 | Expected oracle seq after increment |

Note: σ_s and finalized SenderCommitment (with witness_tpk) are
available via the wallet document read ticket delivered in Phase 1
step 22. Witnesses read them from the document after payload
delivery, not from the payload itself.

Note: encrypted_shares are NOT included. Delivered post-attestation (§21.8).

**Phase 2 additional fields:**

| Field | Type | Description |
|-------|------|-------------|
| zk_proof | bytes | π₂₃ per §32 |
| public_inputs | bytes | Serialized CombinedPublicInputs |
| committed_seq | uint64 | Oracle seq at commit time |
| sender_slot_doc_id | string | Sender's wallet document ID (Phase 2 only) |
| sender_oracle_key | bytes(32) | H(sender_pk) (Phase 2 only) |

NOTE: seed_1 is intentionally excluded. The seed chain
(seed_23 = H(seed_1 || H(σ_r) || beacon_round_23 || randomness_23))
is proven valid inside π₂₃ (§32 constraint 10). Witnesses verify
the seed chain by verifying the ZK proof, not by recomputing it
from plaintext inputs. Including seed_1 would allow cross-phase
linking by any observer present in both shards.

NOTE: sigma_r_hash is intentionally excluded. The seed chain is
proven valid inside π₂₃ using σ_r as a private input. Exposing
H(σ_r) would allow cross-phase correlation if an observer also
sees the Phase 2 seed derivation inputs.

Phase 2 witnesses MUST independently fetch BOTH wallet documents
(sender's via sender_slot_doc_id, recipient's via slot_doc_id)
and verify BOTH are in valid (non-terminal) states. Either doc
being unreachable or in a terminal state → REJECT.

**Phase 4 additional fields:**

| Field | Type | Description |
|-------|------|-------------|
| spend_cid | bytes | CID of SpendRecord |
| nullifier_cid | bytes | CID of published nullifier |
```


```python

@dataclass
class WitnessEligibility:
    """Round 1 response: witness proves VRF selection + issuer eligibility.
    Sent directly to sender's reply_path. NOT broadcast.

    The eligibility_proof carries the full signed IssuerEligibleHeader
    inline (not a CID). Attestation bundles are self-contained — a
    verifier needs only a cached AssetKeyRegistry to verify eligibility,
    no external blob fetch."""
    session_id: bytes
    node_pk: bytes
    vrf_output: bytes
    vrf_proof: bytes
    reply_path: bytes
    observed_density: int           # Hint only — never used for threshold
    eligibility_proof: EligibilityProof  # Includes full IssuerEligibleHeader inline

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'WitnessEligibility':
        d = msgpack.unpackb(data)
        d['eligibility_proof'] = EligibilityProof(**d['eligibility_proof'])
        return cls(**d)


@dataclass
class WitnessResponse:
    """Round 2 response: witness attests after verifying full payload.
    Sent directly to sender's reply_path. NOT broadcast."""
    attestation: Attestation
    node_pk: bytes
    reply_path: bytes

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'WitnessResponse':
        d = msgpack.unpackb(data)
        d['attestation'] = Attestation(**d['attestation'])
        return cls(**d)

NOTE: There is no separate StateRecord. Peers subscribed to a wallet
document (REDUNDANCY_TARGET = 50 XOR-nearest) hold the full Iroh
document replica. Sequence reads are served directly from the
document's `identity:protocol_seq` key. The wallet document IS the
oracle record.


@dataclass
class SequenceUpdate:
    """Gossip notification that a wallet's protocol_seq has advanced."""
    identity_commit: bytes          # H(pk) — wallet doc lookup key
    new_seq: int
    proof: bytes                    # Owner signature over (identity_commit || new_seq)
    originator_sig: bytes
    timestamp: int
    ttl: int

    def decrement_ttl(self) -> 'SequenceUpdate':
        return SequenceUpdate(self.identity_commit, self.new_seq, self.proof,
                              self.originator_sig, self.timestamp, self.ttl - 1)


@dataclass
class PulseSummary:
    """Summary of wallet docs this node tracks, for anti-entropy exchange."""
    key_range: Tuple[bytes, bytes]
    doc_count: int
    cuckoo_filter: bytes            # Bloom-like filter of tracked wallet IDs
    seq_digest: bytes               # Merkle root of (wallet_id, protocol_seq) pairs

    @classmethod
    def from_docs(cls, tracked_docs: Dict[bytes, int],
                  key_range: Tuple[bytes, bytes]) -> 'PulseSummary': ...

    def ids_not_in(self, other: 'PulseSummary') -> List[bytes]: ...
    def ids_with_higher_seq(self, other: 'PulseSummary') -> List[bytes]: ...
```

## 45. Sequence Propagation

### 45.1 Purpose

Each identity's `protocol_seq` is a monotonically increasing
counter stored in the wallet's Iroh document at
`identity:protocol_seq`. It is the authoritative source for
**liveness**: `protocol_seq > creation_seq` → slot is definitely spent.

There is no separate oracle system. The wallet document IS the oracle.

### 45.2 Data Model

Each identity's sequence lives in one place: the wallet Iroh document.
Peers subscribed to the document (REDUNDANCY_TARGET = 50 XOR-nearest)
hold a full replica via Iroh's document replication.

Permanent wallet document footprint (idle, one active slot): **~1KB**.
Peak during active transaction: **~10KB** (tx state keys present).
After finalization and GC of tx keys: back to **~1KB**.

### 45.3 Operations

All operations map to wallet document reads and writes:

| Operation | Implementation | Rule |
|-----------|---------------|------|
| get_seq | Read `identity:protocol_seq` from wallet doc or gossip cache | Returns 0 if unknown. Take max of all observed values. |
| increment | Owner writes `protocol_seq + 1` to own wallet doc | Monotonic by Iroh per-author sequencing. Irreversible. |
| verify_not_spent | Compare `protocol_seq` against `creation_seq` | `protocol_seq > creation_seq` → definitely spent |

### 45.4 Propagation

Two paths to the same value, both monotonic:

**Document replication (authoritative):** Owner writes protocol_seq.
Iroh CRDT replication propagates to 50 subscribed peers.
Witnesses read directly via doc_id from IntentPayload.

**Gossip (speed path):** Owner broadcasts SequenceUpdate on
INAT_ORACLE_TOPIC. Subscribed nodes update local cache.
Faster propagation but lossy (no delivery guarantee).

**Consistency rule:** Both paths only increase. Local reads return
`max(gossip_cached_seq, document_seq)`. Always safe because
monotonic.

### 45.5 Replication


Wallet documents are replicated via Iroh document subscriptions
to the REDUNDANCY_TARGET nodes closest (by XOR distance) to the
identity_commit (H(pk)).

| Parameter | Value |
|-----------|-------|
| REDUNDANCY_TARGET | 50 |
| NEIGHBORHOOD_SIZE | 20 |

### 45.6 Read Path

Tiered lookup for protocol_seq:

| Tier | Source | Freshness |
|------|--------|-----------|
| 1 | Local wallet doc subscription (if this node subscribes) | Real-time |
| 2 | Gossip cache (SequenceUpdate received) | Near real-time |
| 3 | RPC to XOR-nearest peers (they hold doc replicas) | ≤5s |
| 4 | Return 0 (identity never seen) | N/A |

First response wins. Take max of all observed values.

### 45.7 Write Path

On increment:
1. Owner writes `protocol_seq = old + 1` to own wallet document.
   Iroh per-author monotonic sequencing prevents rollback.
2. Owner signs `(identity_commit || new_seq)` for gossip proof.
3. Propagation (parallel):
   a. Iroh document replication to subscribed peers (authoritative).
   b. SequenceUpdate broadcast via gossip (speed path).

### 45.8 SequenceUpdate Message

| Field | Type | Size | Description |
|-------|------|------|-------------|
| identity_commit | bytes | 32 | H(pk) |
| new_seq | uint64 | 8 | |
| proof | bytes | 64 | Owner signature over (identity_commit \|\| new_seq) |
| originator_sig | bytes | 64 | |
| timestamp | uint64 | 8 | |
| ttl | uint8 | 1 | Decremented on re-broadcast. Max: GOSSIP_TTL. |

**Validation rules:**
- Reject if ttl < 0 or ttl > GOSSIP_TTL
- Reject if timestamp > current_time + 60s
- Reject if signature invalid

### 45.9 Anti-Entropy (Pulse)

Every PULSE_INTERVAL, nodes exchange document summaries with
XOR-nearest neighbors.

**PulseSummary:**

| Field | Type | Description |
|-------|------|-------------|
| key_range | (bytes, bytes) | XOR range this node covers |
| doc_count | uint32 | Number of tracked wallets |
| cuckoo_filter | bytes | Probabilistic set of tracked wallet_ids |
| seq_digest | bytes(32) | Merkle root of (wallet_id, protocol_seq) pairs |

**Reconciliation rules:**
- IDs in remote summary not in local → subscribe if responsible
- IDs with higher remote seq → pull document update

| Parameter | Value |
|-----------|-------|
| PULSE_INTERVAL | 300s (5 minutes) |
| GOSSIP_FAN_OUT | 8 |
| GOSSIP_TTL | 6 |

### 45.10 Capacity Limits

| Parameter | Value |
|-----------|-------|
| MAX_SUBSCRIBED_DOCS | 5,000,000 |
| MAX_STORAGE | 1 GB |

At ~1KB per idle wallet document, 1GB supports ~1M subscriptions
with headroom for active transactions (~10KB peak each).

**Eviction policy:** Under storage pressure, unsubscribe from
wallet documents with greatest XOR distance from this node.
MUST NOT evict documents this node is responsible for
(within REDUNDANCY_TARGET of identity_commit).

### 45.11 Churn Handling

**Node join:**

When a new node appears in this node's XOR neighborhood
(detected via DHT or Pulse):

For each wallet document where the new node is within
REDUNDANCY_TARGET closest to the wallet_id:
  → Push document sync to the new node.

This ensures the new node receives wallet documents it is
now responsible for replicating.

**Node departure:**

When a node leaves the XOR neighborhood (detected via stale
heartbeat or DHT churn notification):

For each wallet document where the departed node was within
REDUNDANCY_TARGET closest to the wallet_id:
  → Identify next-closest peer not already subscribed.
  → Push document sync to that peer.

This maintains the replication invariant of REDUNDANCY_TARGET
copies for each wallet document.

---

## 46. Verification Helpers

```python
def verify_sequence_proof(identity_commit: bytes, seq: int,
                          proof: bytes, owner_pk: bytes) -> bool:
    """Verify owner Ed25519 signature over (identity_commit || seq)."""
    if not proof or len(proof) < 64:
        return False
    expected_payload = identity_commit + seq.to_bytes(8, 'big')
    return Ed25519_Verify(owner_pk, expected_payload, proof)

def verify_gossip_update(message: SequenceUpdate, originator_pk: bytes) -> bool:
    if message.ttl < 0 or message.ttl > GOSSIP_TTL:
        return False
    if message.timestamp > current_timestamp() + 60:
        return False
    payload = message.identity_commit + message.new_seq.to_bytes(8, 'big')
    return Ed25519_Verify(originator_pk, payload, message.originator_sig)
```

---

# Part XI: Consistency & Scale


### Glossary

| Term | Definition |
|------|------------|
| Home Shard | k=50 nodes with smallest XOR distance to identity_commit; responsible for subscribing to that wallet document |
| identity_commit | Canonical wallet lookup key: H(pk). Determines XOR keyspace position. Previously called oracle_key. |
| Pulse | Periodic anti-entropy protocol (5min) for summary exchange and gap healing |
| Wallet Document | Iroh document per identity (~1KB idle, ~10KB active). Contains protocol_seq, slot state, transaction state, lineage. Owner is sole writer. Replicated to 50 XOR-nearest peers. IS the sequence oracle. |
| Terminal State | Slot state from which no further spend is permitted: SPENT, FINALIZED, RECOVERED |
| asset_genesis_cid | CID of the asset's creation record. Immutable identity anchor for asset_id derivation, decoupled from any signing key. `asset_id = H("inat_asset:" \|\| asset_genesis_cid \|\| name \|\| params)`. |
| AssetKeyRegistry | Merkle tree of currently authorized issuer public keys for an asset. Chained: each epoch's registry is signed by M-of-N keys from the previous epoch's registry (majority threshold, §42). Enables key rotation and multi-issuer without token migration. Bootstrap at genesis: single self-signed key (§11.1 step 9a). Published as Iroh blob and announced on `inat/registry/v1/{asset_prefix}` topic. Fold circuit verifies chain of trust back to genesis (§33 constraint 7). |
| issuer_registry_root | `Merkle(sort(authorized_keys))` — the root of the AssetKeyRegistry. Verified in fold circuit via Merkle membership proof (~500–1000 constraints). |
| IssuerKeyRotation | Registry update adding/removing authorized keys. Signed by M-of-N keys from the PREVIOUS registry (threshold = majority per §42). Epochs must be sequential. Cannot reduce key count below REGISTRY_MIN_SIGNERS. Removed keys cannot sign new IssuerEligibleRoots after effective_epoch. Rotation cost: ~21K fold circuit constraints (~5% overhead). |
| RegistryAnnouncement | Gossip message announcing a new AssetKeyRegistry CID on the `inat/registry/v1/{asset_prefix}` topic. Contains asset_id, epoch, registry_cid. Nodes fetch and cache the registry blob. |
| REGISTRY_MAX_AGE_EPOCHS | Maximum staleness for a registry referenced in a fold proof. Fold circuit rejects registries older than this. Default: 3 (§42). |
| REGISTRY_ROTATION_THRESHOLD | M-of-N threshold for registry updates: `max(1, ceil(N/2))` by default. Protocol constants in §42. |
| Witness Redundancy Push | Active replication triggered by witness events to 50 home-shard peers (wallet doc subscribers) |
| XOR Distance | Bitwise XOR between identifiers; determines DHT keyspace proximity |
| SELF_AUTHENTICATING | Evidence verified locally from submitted data (ZK proofs, signatures, commitments) |
| MUTABLE_STATE | Evidence fetched independently by witnesses from Iroh (slot status, sequence, nullifiers) |
| SpendRecord | Finalized transaction receipt: consumed slot commit, output commits, forward commits, bilateral signatures, proof, witness evidence, lineage. Input to fold circuit. GC-safe once folded. |
| Forward Commit | `H(owner_sk \|\| seq+1)` — binds a SpendRecord output to the owner's next predetermined slot. Verified by fold circuit without revealing sk. |
| Output Commit | Slot commitment for a transaction output: recipient's new slot and/or sender's change slot. |
| SpendRecordLite | Lightweight spend-only projection of SpendRecord
|                 | for spend_store / archive. Defined in §49.3.
|                 | Distinct from full SpendRecord. |
| ShareReleaseClaim | Phase 3 claim by W₁ witness when releasing their share. |
| BurnRecord | Finalized record of permanent value destruction. Contains slot commitment, nullifier (published on burn), witness attestation root, and optional issuer attestation. Terminal — no recovery. See §23.3. |
| BurnType | VOLUNTARY (owner-initiated) or ISSUER_RECALL (issuer-initiated). Both require 21/29 witness quorum. |
| GenesisRecord | Issuer-signed record creating a new slot at seq=0. Dual-signed by issuer and recipient. Published as content-addressed blob. See §11.1. |
| Deferred Mint | Mechanism by which fee gaps
| Donation Circuit | Optional variant of π₁/π₂₃ that adds pool_pk as 14th entry in witness_root. Sender pays 14 × fee_share instead of 13. Witnesses earn identically. User opts in via wallet UX. |
| Donation Receipt | 104-byte record pushed to issuer-sharded inbox during finalization of donation-circuit transactions. Contains spend_record_cid, donation_commit, issuer_family_id, epoch. Enables guardian discovery of claimable SpendRecords. |
| Fee Gap | Difference between transaction inputs and outputs. Exists as a proven deduction with no output slot. Claimed later via sweep. |
| Ghost Witness | Donation guardian's pool_pk when included as 14th entry in witness_root. Not VRF-selected. Circuit-hardcoded. Sweeps using same unified circuit as real witnesses. |
| Guardian | Holder of pool_sk corresponding to the hardcoded pool_id. Collects donation claims via unified sweep. Chooses target wallet freely at sweep time. Has no special protocol privileges. |
| Issuer-Sharded Inbox | Receipt routing destination at H(pool_pk || issuer_family_id). Separates donation receipts by issuer so guardian queries only issuers worth sweeping. |
| pool_id | H(pool_pk) — hardcoded identity tag in the donation circuit. Not a wallet address. Not a destination. A claim marker that the guardian can prove ownership of at sweep time. |
| Unified Sweep | Single sweep circuit serving both witnesses and donation guardian. Both prove Merkle membership in witness_root, both claim 1 share per SpendRecord, both mint into freely chosen wallets. Protocol cannot distinguish the two claim types. |
| IssuerEligibleHeader | Self-contained, issuer-signed snapshot of VRF-eligible wallet PKs. Contains eligible_root, epoch, asset_id, registry binding, and issuer signature. Delivered directly from issuer to each eligible wallet at epoch rollover (NOT published as an Iroh blob, NOT gossiped). Witnesses carry the full header inline in WitnessEligibility responses, making attestation bundles self-contained. Key rotation is transparent — the new key signs the next epoch's header. §19.9. |
| EligibilityProof | Per-witness proof bundle: (witness_pk, full IssuerEligibleHeader, Merkle proof against header.eligible_root). Fully self-authenticating given only a cached AssetKeyRegistry. Carried inline in WitnessEligibility responses. No external lookup required. |
| Issuer Transparency Log | Append-only MMR of all identities ever issued. Replicated to 50 peers. Auditable for inclusion, consistency, and growth rate. §19.10. |
| Beacon Recency Chain | Fold constraint: consecutive folds' beacons must be within MAX_FOLD_BEACON_GAP_ROUNDS. Prevents temporal gaps. |
| Symmetric Participation | All funded, issuer-approved wallets are eligible to witness. Witnessing incentivized via fee shares. |
| MIN_WITNESS_BALANCE | Minimum token balance required for VRF eligibility. Default: 0.1 tokens. |
| IntentEnvelope | Broadcast to gossip shard (~1000 nodes). Contains only VRF selection data: seed, session_id, nullifier_commit, beacon_round, shard_depth. No identity, asset, or proof data. |
| IntentPayload | Delivered individually to VRF-selected witnesses (~19 nodes) after eligibility round. Contains oracle_key, asset_id, ZK proof, public_inputs, poly_commits (Phase 1), and all verification material. Phase 2 additionally carries sender_slot_doc_id for dual wallet doc verification. |
| WitnessEligibility | Round 1 response from VRF-selected witness. Contains VRF proof and reply_path. No attestation — witness has not yet seen transaction details. |
| Two-Phase Witness Engagement | Privacy mechanism: shard broadcast (IntentEnvelope) reveals only VRF lottery data. Full transaction metadata (IntentPayload) delivered only to selected witnesses. Adds ~2-5s latency per phase. |
| sender_wallet_doc_id / recipient_wallet_doc_id | Iroh document IDs carried in attestation requests. Phase 1 carries only sender's. Phase 2 carries both — witnesses independently fetch both and verify non-terminal state. |


## 47. FEE MECHANISM

### 47.1 Fee Principle

All mandatory fees go to witnesses. There are no mandatory protocol
fund allocations, no hardcoded system wallets, and no percentage-based
creator/developer splits.

  Mandatory fee: 100% to attesting witnesses
  Protocol fund: 0%
  Creator allocation: 0%
  Optional donation: sender pays extra, witnesses unaffected

RATIONALE:
• Mandatory fees to hardcoded wallets create economic interest
  in transaction flow → money transmission risk
• Witnesses perform computational work → fees are payment
  for labor, not rent extraction
• Protocol developer has zero economic interest in tx volume
• Donation is a voluntary gift, not a protocol requirement


### 47.2 Fee Schedule

@dataclass
class FeeSchedule:
    fee_share: int = 10                     # Per-witness share (base unit)
    sweep_base_fee: int = 50
    sweep_per_input_fee: int = 25
    mint_fee: int = 200
    min_priority_fee: int = 0
    max_priority_fee: int = 10000

DEFAULT_SCHEDULE = FeeSchedule()

def calculate_transfer_fee(
    priority: int = 0,
    donation: bool = False,
    schedule: FeeSchedule = DEFAULT_SCHEDULE,
) -> int:
    """Donation adds +1 witness share."""
    witness_count = (WITNESS_QUORUM + 1) if donation else WITNESS_QUORUM
    base = witness_count * schedule.fee_share
    return base + min(priority, schedule.max_priority_fee)

def calculate_sweep_fee(
    num_inputs: int,
    schedule: FeeSchedule = DEFAULT_SCHEDULE,
) -> int:
    return (schedule.sweep_base_fee
            + schedule.sweep_per_input_fee * num_inputs
            + schedule.fee_share * WITNESS_QUORUM)


### 47.3 Fee Lifecycle

Phase 1 (INITIATE):
  Standard circuit:
    Sender commits: amount + (WITNESS_QUORUM × fee_share)
    fee_commit = Pedersen(WITNESS_QUORUM × FEE_SHARE, fee_blinding)
  Donation circuit:
    Sender commits: amount + ((WITNESS_QUORUM + 1) × fee_share)
    fee_commit = Pedersen((WITNESS_QUORUM + 1) × FEE_SHARE, fee_blinding)

Phase 4 (FINALIZE):
  Sender balance:    old - amount - total_fee
  Recipient balance: old + amount
  Fee gap:           deducted from sender, not yet anywhere
  Witness bundle:    published to content store

Sweep (whenever):
  Witness/guardian proves: participation in N transactions
  Witness/guardian balance: 0 + (N × FEE_SHARE)
  Deferred mint: new slot created from proven gaps

@dataclass
class FeeCommitment:
    """Commitment to fee amount (standard or donation variant)."""
    fee_commit: bytes           # Pedersen(N × FEE_SHARE, blinding)
    witness_count: int          # 13 (standard) or 14 (donation)
    # In ZK proof: input_commit = output_commit + change_commit + fee_commit


### 47.5.1 Sweep Triggers

| Trigger | Threshold | Description |
|---------|-----------|-------------|
| Count-based | 1000 attestations | Amortize sweep cost |
| Time-based | 7 days | Periodic redemption |
| Manual | On demand | Claimant discretion |

### 47.5.2 Sweep Procedure

async def execute_sweep(
    claimant_sk: bytes,
    spend_record_cids: List[bytes],
    issuer_family_id: bytes,
    wallet_doc: Doc,
    content_store: ContentStoreProtocol,
    nullifier_store: NullifierStore,
    oracle: SequenceProtocol,
    zk_prover: ZKProverProtocol,
) -> SweepResult:
    """Sweep with durable accumulator. Double-sweep prevention
    is as robust as transfer double-spend prevention."""

    claimant_pk = derive_pk(claimant_sk)
    claimant_pk_commit = H(claimant_pk)

    # ── Step 1: Read current accumulator state ──

    previous_sweep_root = await wallet_doc.get(
        WalletDocSchema.SWEEP_ROOT)
    previous_sweep_count = int.from_bytes(
        await wallet_doc.get(WalletDocSchema.SWEEP_COUNT), 'big')
    previous_sweep_epoch = int.from_bytes(
        await wallet_doc.get(WalletDocSchema.SWEEP_EPOCH), 'big')

    # Load previous leaf set (needed for non-membership proofs
    # and new root construction).
    # For first sweep: empty list.
    if previous_sweep_epoch > 0:
        prev_leaves_data = await wallet_doc.get(
            WalletDocSchema.sweep_leaf_set(
                str(previous_sweep_epoch)))
        previous_sweep_leaves = sorted(
            msgpack.unpackb(prev_leaves_data))
    else:
        previous_sweep_leaves = []

    # Verify local state consistency
    assert merkle_root(previous_sweep_leaves) == previous_sweep_root \
        or (len(previous_sweep_leaves) == 0
            and previous_sweep_root == EMPTY_ROOT)

    # ── Step 2: Validate claims + compute leaves ──

    validated_cids = []
    new_leaves = []
    non_membership_proofs = []

    for cid in spend_record_cids:
        # 2a. Fetch and validate SpendRecord
        record = await content_store.retrieve(cid)
        if record is None:
            continue
        spend = SpendRecord.deserialize(record)

        # 2b. Verify claimant ∈ witness_root
        merkle_proof = compute_merkle_proof_for_pk(
            claimant_pk, spend)
        if merkle_proof is None:
            continue

        # 2c. Verify issuer consistency
        if spend.issuer_family_id != issuer_family_id:
            continue

        # 2d. Compute sweep leaf
        sweep_leaf = H(b"inat_sweep_leaf:"
                       + claimant_pk + cid)

        # 2e. Check NullifierStore (fast path — race detection)
        sweep_null = H(b"inat_sweep_nullifier:"
                       + claimant_pk + cid)
        if await nullifier_store.exists(sweep_null):
            continue  # Already swept (fast reject)

        # 2f. Check accumulator (durable path — authoritative)
        if sweep_leaf in previous_sweep_leaves:
            continue  # Already swept (durable reject)

        # 2g. Build non-membership proof
        #     (adjacent leaves in sorted order proving absence)
        left, right = find_adjacent(
            previous_sweep_leaves, sweep_leaf)
        non_membership_proofs.append((left, right))

        validated_cids.append(cid)
        new_leaves.append(sweep_leaf)

    if not validated_cids:
        return SweepResult(success=False,
                           reason="No valid claims")

    # ── Step 3: Compute new accumulator root ──

    all_leaves = sorted(previous_sweep_leaves + new_leaves)
    new_sweep_root = merkle_root(all_leaves)
    new_sweep_epoch = previous_sweep_epoch + 1
    claimed_amount = len(validated_cids) * FEE_SHARE

    # ── Step 4: Increment oracle seq ──

    new_seq = await oracle.increment_and_propagate(
        identity_commit=H(claimant_pk),
        owner_sk=claimant_sk,
        wallet_doc=wallet_doc)

    # ── Step 5: Derive output slot ──

    new_slot_commit = H(b"inat_slot:"
                        + claimant_sk
                        + new_seq.to_bytes(8, 'big'))
    output_blinding = random_bytes(32)
    balance_commit = pedersen_commit(claimed_amount,
                                     output_blinding)

    # ── Step 6: Generate sweep proof ──

    sweep_nonce = random_bytes(16)
    sweep_nullifier = H(claimant_pk
                        + new_sweep_epoch.to_bytes(8, 'big')
                        + sweep_nonce)

    public_inputs = SweepPublicInputs(
        claimant_pk_commit=claimant_pk_commit,
        claimed_amount=claimed_amount,
        issuer_family_id=issuer_family_id,
        sweep_nullifier=sweep_nullifier,
        new_slot_commit=new_slot_commit,
        previous_sweep_root=previous_sweep_root,
        new_sweep_root=new_sweep_root,
        sweep_epoch=new_sweep_epoch,
        protocol_seq=new_seq,
    )

    private_inputs = SweepPrivateInputs(
        claimant_sk=claimant_sk,
        spend_record_cids=validated_cids,
        merkle_proofs=[
            compute_merkle_proof_for_pk(claimant_pk,
                SpendRecord.deserialize(
                    await content_store.retrieve(c)))
            for c in validated_cids
        ],
        fee_commits=[
            SpendRecord.deserialize(
                await content_store.retrieve(c)
            ).sender_commitment.fee_commit
            for c in validated_cids
        ],
        fee_share_amount=FEE_SHARE,
        previous_sweep_leaves=previous_sweep_leaves,
        non_membership_proofs=non_membership_proofs,
    )

    sweep_proof = await zk_prover.prove_sweep(
        public_inputs, private_inputs)

    # ── Step 7: Publish nullifiers (fast path) ──

    for cid in validated_cids:
        sweep_null = H(b"inat_sweep_nullifier:"
                       + claimant_pk + cid)
        await nullifier_store.publish(sweep_null)

    # ── Step 8: Atomic wallet document update ──
    #
    # All writes happen together. Iroh per-author monotonic
    # sequencing guarantees that if sweep_root advances,
    # protocol_seq has also advanced. Both are bound to the
    # same wallet document write batch.

    epoch_str = str(new_sweep_epoch)
    slot_hex = new_slot_commit.hex()

    # Accumulator state
    await wallet_doc.set(
        WalletDocSchema.SWEEP_ROOT, new_sweep_root)
    await wallet_doc.set(
        WalletDocSchema.SWEEP_COUNT,
        (previous_sweep_count + len(validated_cids)
         ).to_bytes(8, 'big'))
    await wallet_doc.set(
        WalletDocSchema.SWEEP_EPOCH,
        new_sweep_epoch.to_bytes(8, 'big'))

    # Archive leaf set (for future non-membership proofs)
    await wallet_doc.set(
        WalletDocSchema.sweep_leaf_set(epoch_str),
        msgpack.packb(all_leaves))

    # Output slot
    sweep_record = SweepRecord(
        witness_pk_commit=claimant_pk_commit,
        new_slot_commit=new_slot_commit,
        amount=claimed_amount,
        attestation_count=len(validated_cids),
        sweep_proof=sweep_proof,
        sweep_nullifier=sweep_nullifier,
        attestation_cids_root=H(b"".join(validated_cids)),
        issuer_family_id=issuer_family_id,
        timestamp=current_timestamp(),
    )
    sweep_cid = await content_store.publish(
        sweep_record.serialize())

    await wallet_doc.set(
        WalletDocSchema.sweep_proof_cid(epoch_str),
        sweep_cid)

    # Slot state
    lineage = SlotLineage(
        origin=LineageOrigin.SWEEP,
        sweep_record_cid=sweep_cid,
        sweep_proof=sweep_proof,
        asset_id=issuer_family_id,
        lineage_commit=H(b"sweep_lineage:" + sweep_cid),
    )
    owned_slot = OwnedSlot(
        owner_sk=claimant_sk,
        seq=new_seq,
        balance=claimed_amount,
        balance_blinding=output_blinding,
        nullifier=H(b"inat_nullifier:"
                    + claimant_sk
                    + new_seq.to_bytes(8, 'big')),
        slot_commit=new_slot_commit,
        nullifier_commit=H(H(b"inat_nullifier:"
                              + claimant_sk
                              + new_seq.to_bytes(8, 'big'))),
        asset_id=issuer_family_id,
        balance_commit=balance_commit,
        lineage=lineage,
        status=SlotStatus.READY,
        created_at=current_timestamp(),
    )

    await wallet_doc.set(
        WalletDocSchema.slot_status(slot_hex),
        SlotStatus.READY.value.encode())
    await wallet_doc.set(
        WalletDocSchema.slot_data(slot_hex),
        owned_slot.to_public().serialize())
    await wallet_doc.set(
        WalletDocSchema.slot_lineage(slot_hex),
        lineage.serialize())

    return SweepResult(
        success=True,
        new_slot=owned_slot,
        sweep_cid=sweep_cid,
        swept_count=len(validated_cids),
    )
    

## Sweep Leaf Derivation

```python
SWEEP_LEAF_DOMAIN = b"inat_sweep_leaf:"

def sweep_leaf(claimant_pk: bytes, spend_record_cid: bytes) -> bytes:
    """Deterministic leaf for the sweep accumulator.
    Same (claimant, record) pair always produces the same leaf."""
    return H(SWEEP_LEAF_DOMAIN + claimant_pk + spend_record_cid)
```

---

## Non-Membership Proof Construction

The sweep accumulator uses a **sorted Merkle tree**. Leaves are
inserted in canonical byte order. Non-membership is proven by
showing the two adjacent leaves between which the queried leaf
would be inserted — both are present, and the gap between them
contains no other leaves.

```python
def find_adjacent(
    sorted_leaves: List[bytes],
    target: bytes,
) -> Tuple[Optional[bytes], Optional[bytes]]:
    """Find left and right neighbors of target in sorted list.
    Returns (None, first) if target < all leaves.
    Returns (last, None) if target > all leaves.
    Returns (left, right) otherwise.
    Raises if target is IN the list (already swept)."""
    idx = bisect.bisect_left(sorted_leaves, target)
    if idx < len(sorted_leaves) and sorted_leaves[idx] == target:
        raise AlreadySweptError(target)
    left = sorted_leaves[idx - 1] if idx > 0 else None
    right = sorted_leaves[idx] if idx < len(sorted_leaves) else None
    return (left, right)


def verify_non_membership(
    target: bytes,
    left: Optional[bytes],
    right: Optional[bytes],
    root: bytes,
    left_proof: List[Tuple[bytes, str]],
    right_proof: List[Tuple[bytes, str]],
) -> bool:
    """Verify target is NOT in the sorted Merkle tree.
    Proves: left < target < right, and both left/right are
    valid leaves with valid Merkle proofs to root."""
    if left is not None:
        assert left < target
        assert verify_merkle_proof(
            merkle_leaf_hash(left), left_proof, root)
    if right is not None:
        assert target < right
        assert verify_merkle_proof(
            merkle_leaf_hash(right), right_proof, root)
    # If both are adjacent in the tree (no leaves between them),
    # then target is provably absent.
    return True
```

---

## Guardian Wallet Doc Requirement

The donation guardian (holder of pool_sk) MUST maintain a wallet
document keyed by `identity_commit = H(pool_pk)`, just like any
other protocol identity. This doc holds:

- `identity:protocol_seq` — incremented on each guardian sweep
- `identity:sweep_root` — accumulator of all guardian-swept pairs
- Sweep-origin slots (guardian's balance, spendable via normal tx)

The guardian's wallet doc is replicated to k=50 XOR-nearest peers
like any other wallet. `pool_pk` is a real Ed25519 public key,
not just a hash tag — the guardian signs oracle increments and
sweep proofs with `pool_sk`.

**Guardian fund flow:**

```
SpendRecord (donation tx)
    ↓ guardian proves pool_pk ∈ witness_root[13]
Guardian sweep → new slot in guardian's wallet (pool_pk doc)
    ↓ normal transfer (§20)
Guardian sends to final destination wallet
```

The extra transfer is the cost of durability. Without it, the
accumulator and the output are in different docs, and GC of
NullifierStore entries opens a re-sweep window.

**Witness fund flow (unchanged):**

```
SpendRecord (any tx witness attested)
    ↓ witness proves witness_pk ∈ witness_root[0..12]
Witness sweep → new slot in witness's wallet
    ↓ spend directly, or transfer
```

Both flows are structurally identical. The circuit doesn't
distinguish them — both prove Merkle membership and sweep into
their own wallet.


### 47.5.3 Sweep Nullifier

sweep_nullifier = H("inat_sweep_nullifier:" || claimant_pk
                     || spend_record_cid)

TRUST MODEL (updated):
• NullifierStore: FAST-PATH race detection during concurrent sweeps.
  GC-safe once sweep_root incorporates the leaf.
• Sweep accumulator (wallet doc): AUTHORITATIVE durable prevention.
  sweep_leaf ∈ sweep_root ↔ already claimed. Replicated k=50.

This matches the transfer model:
  Transfer: NullifierStore (fast) + oracle seq (durable)
  Sweep:    NullifierStore (fast) + sweep_root (durable)
  
One nullifier per (claimant, SpendRecord) pair. Second claim for
same SpendRecord → same nullifier → rejected. Uses same NullifierStore
as spend nullifiers.

Two-layer double-claim prevention (mirrors transfer double-spend):

| Layer | Transfer | Sweep |
|-------|----------|-------|
| Fast-path (ephemeral) | NullifierStore | NullifierStore (sweep_nullifier) |
| Durable (authoritative) | Oracle seq kills input slot | sweep_root accumulator in wallet doc |
| Replication | Wallet doc k=50 | Same wallet doc k=50 |
| ZK binding | Fold circuit binds nullifier to seq | Sweep circuit binds leaf to sweep_root + seq |

Sweep outputs MUST go to claimant's own wallet so the accumulator
and the output live in the same document, preventing re-sweep
attacks after NullifierStore GC.

## Verification — Third-Party Sweep Verification

```python
async def verify_sweep_slot(
    slot: Slot,
    sweep_record: SweepRecord,
    wallet_doc: Doc,
) -> bool:
    """Third-party verification of a sweep-origin slot."""

    # 1. Verify sweep proof
    public_inputs = SweepPublicInputs(
        claimant_pk_commit=sweep_record.witness_pk_commit,
        claimed_amount=sweep_record.amount,
        issuer_family_id=sweep_record.issuer_family_id,
        sweep_nullifier=sweep_record.sweep_nullifier,
        new_slot_commit=sweep_record.new_slot_commit,
        previous_sweep_root=...,  # from proof public inputs
        new_sweep_root=...,       # from proof public inputs
        sweep_epoch=...,
        protocol_seq=...,
    )
    if not await zk_prover.verify_sweep(
            sweep_record.sweep_proof, public_inputs):
        return False

    # 2. Verify accumulator state matches wallet doc
    doc_sweep_root = await wallet_doc.get(
        WalletDocSchema.SWEEP_ROOT)
    doc_seq = int.from_bytes(
        await wallet_doc.get(
            WalletDocSchema.PROTOCOL_SEQ), 'big')

    # The wallet doc's sweep_root must be >= the sweep's
    # new_sweep_root (it may have advanced further via
    # subsequent sweeps)
    # Protocol seq must be >= the sweep's bound seq
    if doc_seq < public_inputs.protocol_seq:
        return False  # Wallet doc hasn't caught up

    # 3. Slot commit matches
    if slot.slot_commit != sweep_record.new_slot_commit:
        return False

    return True
```

---

## Circuit Constraint Cost Impact

```
Existing sweep circuit:    ~200k constraints
Non-membership proofs:     +40 per claim (Merkle proof pair)
Root recomputation:        +N×log(N) hashing constraints
Seq/epoch binding:         +~500 constraints

At 1000 claims per batch:  ~200k + 40k + 20k + 0.5k ≈ 260k
At 100 claims per batch:   ~200k + 4k + 2k + 0.5k ≈ 206k

Manageable within 2^18 SRS.
```

---

## Security Argument

| Attack | Prevention |
|--------|-----------|
| Re-sweep same SpendRecord | sweep_leaf ∈ previous_sweep_root → circuit rejects (constraint 8) |
| Replay sweep proof at different seq | protocol_seq binding (constraint 10) — proof is seq-specific |
| NullifierStore wipe | Accumulator in wallet doc (k=50) is authoritative |
| Forge sweep_root | Owner is sole writer; Iroh monotonic sequencing prevents rollback |
| Race between concurrent sweeps | NullifierStore provides fast-path; both would produce same leaf → second sweep's non-membership proof fails |
| Wallet doc corruption | Same risk as transfer — loss of wallet doc = loss of funds. No new assumption. |
| Sweep into wallet A, re-sweep into wallet B | Output bound to claimant_sk (constraint 12) — accumulator and output always in same doc |
| Guardian re-sweeps after NullifierStore GC | Guardian's pool_pk wallet doc holds sweep_root — durable across GC cycles |

**Symmetry achieved:**

| Mechanism | Transfer | Sweep |
|-----------|----------|-------|
| Fast-path (ephemeral) | NullifierStore | NullifierStore |
| Durable (authoritative) | Oracle seq kills input slot | sweep_root accumulates claimed pairs |
| Replication | Wallet doc k=50 | Wallet doc k=50 (same doc) |
| ZK binding | Fold circuit binds nullifier to seq | Sweep circuit binds leaf to sweep_root + seq |

### 47.5.4 SweepRecord

| Field | Type | Description |
|-------|------|-------------|
| witness_pk_commit | bytes(32) | H(claimant_pk) |
| new_slot_commit | bytes(32) | Derived slot for claimed balance |
| amount | uint64 | Total claimed |
| attestation_count | uint32 | Number of validated SpendRecords |
| sweep_proof | bytes(var) | ZK proof per §35 |
| sweep_nullifier | bytes(32) | H(pk \|\| batch_epoch \|\| nonce) |
| attestation_cids_root | bytes(32) | H(all claimed spend_record_cids) |
| issuer_family_id | bytes(32) | Per-issuer sweep |
| timestamp | uint64 | Creation time |

### 47.5.5 Donation Receipt (for guardian discovery)

Published during finalization of donation-circuit transactions.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| spend_record_cid | bytes | 32 | |
| donation_commit | bytes | 32 | Pedersen(fee_share, blinding) |
| issuer_family_id | bytes | 32 | |
| epoch | uint64 | 8 | |

Total: 104 bytes.

**Routing:** Pushed to XOR-responsible peers for inbox key:
`H("inat_donation_inbox:" || pool_pk || issuer_family_id)`

**Discovery:** Guardian queries inbox peers for receipts since last sweep epoch. Deduplicates by spend_record_cid.

---

### 47.6 Fee Failure Modes

| Failure                | Impact                     | Resolution                    |
|------------------------|----------------------------|-------------------------------|
| Witness never sweeps   | Fee stays as unclaimed gap | Witness's loss, no impact     |
| SpendRecord lost       | Witness has local copy     | Backup: scan content store    |
| Double-sweep attempt   | sweep_nullifier collision  | Rejected                      |
| Invalid Merkle proof   | Sweep proof fails          | Cannot claim                  |
| Guardian ignores issuer| Donations permanently burned| Deflationary, protocol indiff.|


### 47.7 — OPTIONAL DONATION SYSTEM (NEW)

#### 47.7.1 Design Principle

The donation system is a voluntary, user-initiated mechanism that
has zero impact on protocol operation, witness compensation, or
transaction processing. It exists as an optional circuit variant
that adds the donation guardian's public key as a 14th entry in the
witness root. The protocol is indifferent to its use.

PRINCIPLES:
1. VOLUNTARY: User actively opts in. Off by default.
2. WITNESS-NEUTRAL: Witnesses earn identically regardless of variant.
3. NO HARDCODED WALLET: Protocol contains a pool identity tag
   H(pool_pk), not a destination wallet.
4. IDENTICAL MECHANISM: Same deferred mint and sweep circuit as
   witness fees. The guardian IS a ghost witness.
5. ZERO OVERHEAD FOR NON-DONORS: Standard circuit unchanged.


#### 47.7.2 The Ghost Witness Model

TWO CIRCUIT VARIANTS:

STANDARD (WITNESS_QUORUM witnesses):
  witness_set = [w1, w2, ... w21]
  witness_root = Merkle(21 entries)
  total_fee = 21 × fee_share

DONATION (WITNESS_QUORUM witnesses + 1 ghost):
  witness_set = [w1, w2, ... w21, pool_pk]
  witness_root = Merkle(22 entries)
  total_fee = 22 × fee_share
  Circuit additionally proves:
    witness_set[21] == pool_pk
    H(pool_pk) == HARDCODED_POOL_ID
    (pool_pk is NOT VRF-selected — special case)

WITNESS PERSPECTIVE:
  Standard: I earn fee_share. 20 others earn same.
  Donation: I earn fee_share. 20 others + ghost earn same.
  My pay is identical. I don't care.

Hardcoded pool identity tag (NOT a wallet address)
HARDCODED_POOL_ID: bytes = b""  # H(pool_pk) — set at protocol genesis


#### 47.7.5 Donation Network Overhead

PER DONATION TRANSACTION:
  Extra witness_root entry:  +32 bytes in SpendRecord
  Donation receipt:          104 bytes pushed to 20 peers
  Circuit:                   ~200 extra constraints

PER NON-DONATION TRANSACTION:
  Zero. Standard circuit completely unchanged.

AMORTIZED ACROSS ALL TRANSACTIONS (1% donation rate):
  +0.32 bytes average SpendRecord overhead
  +1.04 bytes average receipt overhead
  Total: ~1.4 bytes amortized. Protocol rounding error.

# Privacy config for donation receipts (add to §29.2 / §18 configs)
DONATION_PRIVACY_CONFIG = {
    "inat.donation.store": SecurityConfig(
        encryption=True, padding=True, timing_jitter=True, relay=False),
    "inat.donation.query": SecurityConfig(
        encryption=True, padding=True, timing_jitter=True, relay=False),
}


## 48. WITNESS ECONOMICS

### 48.1 Fee Model Summary

Witnesses earn fees denominated in the issuer of the transaction
they attested. No stake required. No registration. Fees are
deferred mints collected via the unified sweep circuit (§47.5).

### 48.2 Node Startup: InatNode.start()

async def start_inat_node(secret_key: bytes):
    """
    Minimal startup. No registration. No stake. No credits.
    Join DHT → subscribe shard → start Pulse + density → done.
    """
    iroh = await iroh_connect()
    node = InatNode(secret_key, iroh)
    await node.start()
    return node


### 48.3 Witness Sovereignty

Witnesses have full discretion over which issuers they validate.
Each witness only earns fees denominated in issuers they chose to attest.

class WitnessAdmissionPolicy:
    """
    Each witness defines their own issuer acceptance policy.
    The protocol does NOT mandate which issuers are valid.
    Witnesses self-select based on economic incentive.
    """

    def __init__(self, config: 'WitnessConfig'):
        self.accepted_issuers: Set[bytes] = set()


### 48.4 Slashing

Without stake, consequences are soft but economically effective.

class ViolationType(Enum):
    DOUBLE_SPEND_ATTESTATION = "double_spend"
    INVALID_PROOF_ATTESTATION = "invalid_proof"
    CONFLICTING_ATTESTATIONS = "conflicting"
    LAZY_ATTESTATION = "lazy"

| Violation                  | Evidence                              | Consequence              |
|----------------------------|---------------------------------------|--------------------------|
| Double-spend attestation   | Attested tx with existing nullifier   | Excluded from shard gossip |
| Invalid proof attestation  | Signed valid for provably invalid proof| Excluded from shard gossip |
| Conflicting attestations   | Two signatures for same session       | Excluded from shard gossip |

Detection is permissionless. Without stake, "slashing" means:
1. Fee forfeiture: slashed node loses accumulated fee attestations.
   sweep_proof verification checks slashing status.
2. Shard exclusion: detecting nodes propagate SlashingRecord via
   gossip. Nodes maintain local blacklists of slashed PKs.
   Transacting parties skip attestations from blacklisted witnesses.
3. "Soft" slashing — no protocol-level enforcement, but
   economically effective: fee earnings are the only reward.

Sybil resistance comes from VRF threshold math, not slashing economics.

@dataclass
class SlashingRecord:
    witness_pk: bytes
    evidence: 'SlashingEvidence'
    detected_by: bytes
    timestamp: int
    signature: bytes

    def verify(self) -> bool:
        return verify_equivocation(self.evidence)

Equivocation: same witness signs attestations for different
sessions → SlashingEvidence propagated via gossip


## 49. DATA STRUCTURES

```python
### 49.1 Enums

class OracleUnavailableError(Exception):
    """Callers MUST treat as attestation failure (fail closed)."""
    pass

# SlotStatus: see §13 (canonical definition)
# LineageOrigin: see §13 (canonical definition)
# TERMINAL_STATES: see §13 (canonical definition)

class Role(Enum):
    SENDER = "SENDER"
    RECIPIENT = "RECIPIENT"

class RecoveryType(Enum):
    TIMEOUT = "TIMEOUT"
    BILATERAL = "BILATERAL"

class BurnType(Enum):
    VOLUNTARY = "VOLUNTARY"
    ISSUER_RECALL = "ISSUER_RECALL"
```

### 49.1.1 Serialization Helpers

```python
def enum_encoder(obj):
    """msgpack default encoder for Enum values and bytes-like objects."""
    if isinstance(obj, Enum):
        return obj.value
    if isinstance(obj, bytes):
        return obj
    raise TypeError(f"Object of type {type(obj)} is not msgpack serializable")


# Versioned serialization policy:
# - All msgpack payloads SHOULD include a "_v" field (integer).
# - Decoders MUST reject _v values they do not recognize.
# - Field additions are backward-compatible (use dict.get with defaults).
# - Field removals require a _v bump.
# - Current versions: SenderCommitment=1, RecipientCommitment=1,
#   SpendRecord=2, CollapsedProof=2, TransactionStateRecord=1.
```

### 49.2 Commitment Structures


@dataclass
class SenderCommitment:
    """What sender commits to in Phase 1. Signed AFTER witness collection
    (witness_tpk requires knowing the witness set). σ_s signs the finalized
    commitment including witness_tpk."""
    owner_pk_commit: bytes              # H(owner_pk)
    seq: int                            # new_seq (post-increment)
    slot_commit: bytes                  # Consumed slot: H("inat_slot:" || sk || source_seq)
    nullifier_commit: bytes             # H(nullifier)
    amount_commit: bytes                # Pedersen(amount to recipient, blinding)
    sender_output_commit: bytes         # H("inat_slot:" || sk || new_seq)
    sender_output_amount_commit: bytes  # Pedersen(remaining balance, blinding)
    fee_commit: bytes                   # Pedersen(total_fee, blinding)
    recipient_pk_commit: bytes          # H(recipient_pk)
    lineage_commit: bytes
    asset_id: bytes
    session_id: bytes
    beacon_round: int
    witness_tpk: bytes                  # H("TPK:" || sort(W₁ witness PKs))
                                        # Set after attestation collection (§20.1 step 16)
    donation: bool = False              # True → +1 witness circuit variant
    sender_forward_commit: bytes = b""  # H("inat_slot:" || sk || (new_seq + 1))
    eligible_root_epoch: int = 0       # IssuerEligibleRoot epoch used

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'SenderCommitment':
        return cls(**msgpack.unpackb(data))

@dataclass
class RecipientCommitment:
    """What recipient commits to in Phase 2. Signed AFTER verifying sender's commitment."""
    slot_commit: bytes                  # H("inat_slot:" || recipient_sk || recv_seq)
    amount_commit: bytes                # Must match sender's amount_commit
    sender_nullifier_commit: bytes      # Must match sender's nullifier_commit
    sender_commitment_hash: bytes       # H(sender_commitment.serialize())
    session_id: bytes                   # Must match sender's
    recipient_forward_commit: bytes     # H("inat_slot:" || recipient_sk || (recv_seq + 1))

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'RecipientCommitment':
        return cls(**msgpack.unpackb(data))

### 49.3 Spend Record

@dataclass
class SpendRecord:
    """Finalized transaction record. Input to fold circuit (Phase 4). 
    GC-safe once folded into output slot proofs."""
    session_id: bytes
    beacon_round: int

    # Consumed slot
    spent_commit: bytes                 # Slot commitment being consumed
    nullifier_commit: bytes             # H(nullifier) for consumed slot

    # Output slots
    output_commits: List[bytes]         # [recipient_slot_commit, sender_output_commit?]
    forward_commits: List[bytes]        # H(owner_sk || output_seq + 1) per output
                                        # Binds each output to owner's next predetermined slot
                                        # Verified by fold circuit (§33)

    # Bilateral attestation
    sender_commitment: SenderCommitment
    recipient_commitment: RecipientCommitment
    sigma_s: bytes
    sigma_r: bytes
    proof_23: bytes

    # Witness evidence
    witness_root: bytes                 # Merkle root of first 13 witness PKs
    # NOTE: witness_bundle_cid intentionally absent. WitnessBundle
    # references SpendRecord (one-directional). No CID circularity.

    # Forced finalization metadata
    is_forced: bool = False
    forced_finalization_cid: Optional[bytes] = None

    # Lineage
    previous_spend_cid: Optional[bytes]   # Previous SpendRecord CID (None → genesis_cid)
    depth: int                          # Distance from genesis
    issuer_family_id: bytes

    # Metadata
    nullifier_cid: bytes
    fee_amount: int
    created_at: int

    @property
    def combined_hash(self) -> bytes:
        return H(
            H(self.sender_commitment.serialize()) +
            H(self.recipient_commitment.serialize()) +
            self.sigma_s + self.sigma_r + self.witness_root
        )

    def verify_binding(self) -> bool:
        """Structural binding checks. Full verification requires π₂₃."""
        return (
            self.recipient_commitment.session_id == self.sender_commitment.session_id and
            self.recipient_commitment.sender_nullifier_commit == self.sender_commitment.nullifier_commit and
            self.recipient_commitment.amount_commit == self.sender_commitment.amount_commit and
            self.recipient_commitment.sender_commitment_hash == H(self.sender_commitment.serialize()) and
            len(self.output_commits) > 0 and
            len(self.output_commits) == len(self.forward_commits)
        )

    def serialize(self) -> bytes:
        return msgpack.packb({
            "session_id": self.session_id, "beacon_round": self.beacon_round,
            "spent_commit": self.spent_commit,
            "nullifier_commit": self.nullifier_commit,
            "output_commits": self.output_commits,
            "forward_commits": self.forward_commits,
            "sender_commitment": self.sender_commitment.serialize(),
            "recipient_commitment": self.recipient_commitment.serialize(),
            "sigma_s": self.sigma_s, "sigma_r": self.sigma_r,
            "proof_23": self.proof_23,
            "witness_root": self.witness_root,
            "previous_spend_cid": self.previous_spend_cid,
            "depth": self.depth,
            "issuer_family_id": self.issuer_family_id,
            "nullifier_cid": self.nullifier_cid,
            "fee_amount": self.fee_amount, "created_at": self.created_at,
            "is_forced": self.is_forced,
            "forced_finalization_cid": self.forced_finalization_cid,
        })

    @classmethod
    def deserialize(cls, data: bytes) -> 'SpendRecord':
        d = msgpack.unpackb(data)
        d['sender_commitment'] = SenderCommitment.deserialize(d['sender_commitment'])
        d['recipient_commitment'] = RecipientCommitment.deserialize(d['recipient_commitment'])
        return cls(**d)

NOTE: SpendRecord (above) is the canonical bilateral record (§49.3).
    SpendRecordLite (spend_record.py) is a lightweight projection for
    spend_store and content archive. They share some fields but are
    DISTINCT types with different field names and purposes.
    Do not alias one to the other.

@dataclass
class SpendRecordLite:
    """Lightweight spend-only projection for spend_store / archive.
    Omits bilateral commitment details, witness bundles, and proofs."""
    session_id: bytes
    spent_commit: bytes
    nullifier_commit: bytes
    output_commits: List[bytes]
    issuer_family_id: bytes
    depth: int
    previous_spend_cid: Optional[bytes]
    nullifier_cid: bytes
    created_at: int

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'SpendRecordLite':
        return cls(**msgpack.unpackb(data))



SlotState composes a public Slot with transaction-phase-specific
fields. The Slot captures intrinsic identity (persists across
transactions). The remaining fields capture ephemeral transaction
machinery (relevant only during an active transaction).

@dataclass
class SlotState:
    """
    Slot state stored in Iroh Documents. Third-party visibility
    of commitment status plus active transaction context.

    The `slot` field carries the intrinsic public identity.
    Phase-specific fields are populated/cleared as the transaction
    state machine advances.
    """

    # === INTRINSIC (persists across transactions) ===
    slot: Slot

    # === PHASE 1: PENDING_SEND ===
    sender_commitment: Optional[SenderCommitment] = None
    sigma_s: Optional[bytes] = None
    encrypted_shares: Optional[List[bytes]] = None
    poly_commits: Optional[List[bytes]] = None
    proof_1: Optional[bytes] = None
    witness_seed_1: Optional[bytes] = None
    attestations_1: Optional[List['Attestation']] = None

    # === PHASE 2: COMMITTED ===
    recipient_commitment: Optional[RecipientCommitment] = None
    sigma_r: Optional[bytes] = None
    proof_23: Optional[bytes] = None
    witness_seed_23: Optional[bytes] = None
    attestations_23: Optional[List['Attestation']] = None
    combined_hash: Optional[bytes] = None

    # === PHASE 3: SPENT ===
    nullifier_cid: Optional[bytes] = None
    spend_cid: Optional[bytes] = None
    witness_bundle_cid: Optional[bytes] = None

    # === PHASE 4: FINALIZED ===
    collapsed_proof: Optional[bytes] = None
    confirmation_root: Optional[bytes] = None

    # === RECOVERY ===
    recovery_cid: Optional[bytes] = None

    # === TIMESTAMPS ===
    updated_at: float = 0.0
    pending_since: Optional[float] = None

    # --- Convenience accessors delegated to inner Slot ---

    @property
    def slot_commit(self) -> bytes:
        return self.slot.slot_commit

    @property
    def status(self) -> SlotStatus:
        return self.slot.status

    @status.setter
    def status(self, value: SlotStatus):
        self.slot.status = value

    @property
    def asset_id(self) -> bytes:
        return self.slot.asset_id

    @classmethod
    def from_slot(cls, slot: Slot) -> 'SlotState':
        """Create SlotState from a public Slot with no active transaction."""
        return cls(slot=slot)

    @classmethod
    def from_owned_slot(cls, owned: OwnedSlot) -> 'SlotState':
        """Create SlotState from an OwnedSlot by projecting to public view."""
        return cls(slot=owned.to_public())


# TransactionStateRecord (unchanged structure, updated type references)

@dataclass
class TransactionStateRecord:
    """Complete transaction state for a party's view."""
    session_id: bytes
    beacon_round: int
    role: Role
    phase: Phase
    sender_commitment: Optional[SenderCommitment] = None
    recipient_commitment: Optional[RecipientCommitment] = None
    sigma_s: Optional[bytes] = None
    sigma_r: Optional[bytes] = None
    proof_1: Optional[bytes] = None
    proof_23: Optional[bytes] = None
    # Threshold crypto (sender only)
    encrypted_shares: Optional[List[bytes]] = None
    poly_commits: Optional[List[bytes]] = None
    # Witness data
    witness_seed_1: Optional[bytes] = None
    witness_seed_23: Optional[bytes] = None
    witnesses_1: Optional[List[bytes]] = None
    witnesses_23: Optional[List[bytes]] = None
    w1_reply_paths: Optional[List[bytes]] = None    # Reply paths for Wâ‚ share release trigger
    attestations_1: Optional[List['Attestation']] = None
    attestations_23: Optional[List['Attestation']] = None
    # Serialized public inputs (for counterparty/witness verification)
    public_inputs_1: Optional[bytes] = None
    public_inputs_23: Optional[bytes] = None
    # Pairing
    pairing: Optional['PairingData'] = None
    # Timestamps
    created_at: float = 0.0
    updated_at: float = 0.0

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self), default=enum_encoder)

    @classmethod
    def deserialize(cls, data: bytes) -> 'TransactionStateRecord':
        d = msgpack.unpackb(data)
        d['role'] = Role(d['role'])
        d['phase'] = Phase(d['phase'])
        if d.get('sender_commitment'):
            d['sender_commitment'] = SenderCommitment.deserialize(d['sender_commitment'])
        if d.get('recipient_commitment'):
            d['recipient_commitment'] = RecipientCommitment.deserialize(d['recipient_commitment'])
        return cls(**d)


### 49.5 Attestation Structures

@dataclass
class Attestation:
    session_id: bytes
    witness_pk: bytes
    witness_index: int              # 0 to WITNESS_TOTAL-1
    phase: Phase
    nullifier_commit: bytes
    beacon_round: int
    timestamp: int
    vrf_proof: Optional[bytes] = None
    witness_seed: Optional[bytes] = None
    signature: bytes = b""

    def signing_payload(self) -> bytes:
        return msgpack.packb({
            "session_id": self.session_id, "phase": self.phase.value,
            "nullifier_commit": self.nullifier_commit,
            "beacon_round": self.beacon_round, "timestamp": self.timestamp,
        })

    def verify(self) -> bool:
        return verify_signature(self.witness_pk, self.signing_payload(), self.signature)

    def to_fee_attestation(self) -> 'FeeAttestation':
        """Project to fee-relevant fields for WitnessBundle inclusion."""
        return FeeAttestation(
            witness_pk=self.witness_pk,
            witness_index=self.witness_index,
            session_id=self.session_id,
            beacon_round=self.beacon_round,
            timestamp=self.timestamp,
            signature=self.signature,
        )

@dataclass
class FeeAttestation:
    """First 13 by timestamp get fees."""
    witness_pk: bytes
    witness_index: int
    session_id: bytes
    beacon_round: int
    timestamp: int
    signature: bytes

    def leaf_hash(self) -> bytes:
        return H(self.witness_pk || self.session_id || self.beacon_round.to_bytes(8, 'big'))

@dataclass
class AttestationSummary:
    """Compact representation of witness attestations for finalized records."""
    initiate_quorum: bool               # ≥13 W₁ attestations
    commit_quorum: bool                 # ≥13 W₂₃ attestations
    shares_sufficient: bool             # ≥13 shares published (Phase 3)
    da_confirmed: bool                  # ≥7 W₄ confirmations (Phase 4)
    witness_set_root: bytes
    beacon_round: int
    initiate_count: int                 # W₁ attestation count
    commit_count: int                   # W₂₃ attestation count
    share_release_count: int            # Published + verified shares
    confirm_count: int                  # DA confirmation count
    attestation_roots: Dict[Phase, bytes]
    confirmation_root: bytes

@dataclass
class WitnessBundle:
    """References SpendRecord via spend_record_cid. One-directional:
    SpendRecord does NOT back-reference WitnessBundle."""
    session_id: bytes
    spend_record_cid: bytes             # Always set (published after SpendRecord)
    beacon_round: int
    fee_attestations: List[FeeAttestation]      # Always exactly 13 real attestations
    additional_attestations: List[Attestation]
    donation: bool = False                      # True → 14-entry witness_root
    pool_pk: Optional[bytes] = None             # Present iff donation=True

    @property
    def is_valid(self) -> bool:
        return len(self.fee_attestations) >= WITNESS_QUORUM  # Always 13

    @property
    def witness_count(self) -> int:
        """Fee-earning entries: 13 (standard) or 14 (donation)."""
        return 14 if self.donation else WITNESS_QUORUM

    def get_witness_root(self) -> bytes:
        """Merkle root of fee-earning entries.
        Standard: 13 leaves (witness PKs from fee_attestations).
        Donation: 14 leaves (13 witness PKs + pool_pk as 14th leaf).
        pool_pk is NOT a FeeAttestation — it never attested."""
        leaves = [a.witness_pk for a in self.fee_attestations[:WITNESS_QUORUM]]
        if self.donation and self.pool_pk:
            leaves.append(self.pool_pk)
        return merkle_root(leaves)

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))


@dataclass
class W23AttestationBundle:
    """Sent from sender to each W₁ witness after W₂₃ quorum."""
    session_id: bytes
    attestations: List[Attestation]  # ≥13 Phase 2 attestations
    proof_23: bytes                   # For independent verification
    
@dataclass
class ShareReleaseClaim:
    """First 5 publishers get +1 credit bonus."""
    session_id: bytes
    nullifier_cid: bytes
    spend_cid: bytes
    publisher_pk: bytes
    beacon_round: int
    signature: bytes

@dataclass
class Confirmation:
    status: str     # DA_AVAILABLE | DA_UNAVAILABLE | DA_INVALID
    nullifier_cid: Optional[bytes] = None
    spend_cid: Optional[bytes] = None
    validator_pk: Optional[bytes] = None
    beacon_round: Optional[int] = None
    signature: Optional[bytes] = None

    def payload(self) -> bytes:
        return msgpack.packb({
            "nullifier_cid": self.nullifier_cid,
            "spend_cid": self.spend_cid,
            "beacon_round": self.beacon_round,
        })

DA_AVAILABLE = "DA_AVAILABLE"
DA_UNAVAILABLE = "DA_UNAVAILABLE"
DA_INVALID = "DA_INVALID"

### 49.6 Recovery Structures

@dataclass
class RecoveryRequest:
    identity_commit: bytes
    slot_commit: bytes
    stuck_seq: int
    stuck_state: SlotStatus         # PENDING_SEND or PENDING_RECEIVE
    session_id: bytes
    request_type: RecoveryType
    timestamp: int
    owner_signature: bytes
    recipient_signature: Optional[bytes] = None  # BILATERAL only

Recovery witnesses use stuck_state to determine investigation
checks. Both states follow the same recovery procedure; the
field is informational for logging and investigation routing.


@dataclass
class InvestigationResult:
    """What witness found during recovery investigation."""
    slot_status: SlotStatus
    nullifier_exists: bool
    oracle_seq: int
    w23_quorum: bool                # True if W₂₃ quorum achieved
    timeout_elapsed: bool

@dataclass
class RecoveryAttestation:
    status: str                     # APPROVED, ALREADY_SPENT, COMMITTED_NO_RECOVERY, etc.
    identity_commit: Optional[bytes] = None
    recovered_seq: Optional[int] = None
    investigation: Optional[InvestigationResult] = None
    witness_pk: Optional[bytes] = None
    beacon_round: Optional[int] = None
    signature: Optional[bytes] = None
    details: Optional[str] = None

@dataclass
class RecoveryRecord:
    identity_commit: bytes
    old_seq: int
    new_seq: int                    # old_seq + 1
    old_slot_commit: bytes
    new_slot_commit: bytes
    balance_commit: bytes           # UNCHANGED
    session_id: bytes
    witness_attestations: List[RecoveryAttestation]  # ≥13 APPROVED
    recovery_seed: bytes
    proof: bytes
    timestamp: int

    def verify_quorum(self) -> bool:
        approved = [a for a in self.witness_attestations if a.status == "APPROVED"]
        return len(approved) >= WITNESS_QUORUM

@dataclass
class RecoveryResult:
    success: bool
    reason: Optional[str] = None
    new_slot: Optional[OwnedSlot] = None
    recovery_cid: Optional[bytes] = None

### 49.6.1 Burn Structures

@dataclass
class BurnIntent:
    slot_commit: bytes
    identity_commit: bytes          # H(owner_pk)
    burn_type: BurnType
    reason: str
    asset_id: bytes
    timestamp: int
    owner_signature: bytes          # Always present
    issuer_attestation: Optional[bytes] = None  # ISSUER_RECALL only

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self), default=enum_encoder)

@dataclass
class BurnRecord:
    slot_commit: bytes
    identity_commit: bytes
    nullifier_commit: bytes
    burn_type: BurnType
    asset_id: bytes
    balance_commit: bytes           # Value destroyed (Pedersen)
    burn_seed: bytes                # VRF seed for witness selection
    witness_attestation_root: bytes # Merkle of ≥13 attestations
    issuer_attestation: Optional[bytes]
    timestamp: int
    burn_cid: bytes                 # CID of this record

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self), default=enum_encoder)

    @classmethod
    def deserialize(cls, data: bytes) -> 'BurnRecord':
        d = msgpack.unpackb(data)
        d['burn_type'] = BurnType(d['burn_type'])
        return cls(**d)

### 49.7 Sweep Structures

@dataclass
class SweepAttestation:
    """Reference to a single fee claim (witness or guardian)."""
    spend_record_cid: bytes
    witness_bundle_cid: bytes
    merkle_proof: List[bytes]       # Proves claimant_pk ∈ witness_root
    witness_index: int              # Position in witness_root (0-12 for witness, 13 for ghost)

@dataclass
class SweepRequest:
    claimant_pk: bytes              # Witness PK or pool_pk
    attestations: List[SweepAttestation]
    total_amount: int               # len(attestations) × FEE_SHARE
    issuer_family_id: bytes         # Sweep is per-issuer

@dataclass
class SweepRecord:
    witness_pk_commit: bytes        # H(claimant_pk) — witness or guardian
    new_slot_commit: bytes
    amount: int
    attestation_count: int
    sweep_proof: bytes
    sweep_nullifier: bytes          # H(claimant_pk || batch_epoch || nonce)
    attestation_cids_root: bytes    # H(all claimed spend_record_cids)
    issuer_family_id: bytes         # Per-issuer sweep
    timestamp: int

Sweep produces an OwnedSlot (the claimant knows their own sk).
The public Slot projection is written to Iroh via .to_public().

Sweep-origin slots use SlotLineage (§13.5) with origin=SWEEP.
No separate SweepLineage type is needed.

### 49.8 Message Structures

@dataclass
class TransactionInvite:
    session_id: bytes
    sender_doc_id: str
    sender_pk: bytes
    asset_id: bytes
    amount_hint: Optional[int]
    read_ticket: str

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

@dataclass
class TransactionAccepted:
    session_id: bytes
    recipient_doc_id: str
    recipient_pk: bytes
    read_ticket: str

@dataclass
class MintInvite:
    session_id: bytes
    issuer_pk: bytes
    issuer_doc_id: str
    asset_id: bytes
    amount: int

### 49.9 Issuer Eligibility Structures

```python
# IssuerEligibleHeader: defined in §19.9
# EligibilityProof: defined in §19.9 (carries full IssuerEligibleHeader inline)
# IssuerTransparencyRecord: defined in §19.10
# WITNESS_TIERS: defined in §19.11
```

### 49.9.1 Issuer Key Management Structures
```python
@dataclass
class AssetKeyRegistry:
    """Merkle tree of currently authorized issuer PKs for an asset.
    Enables key rotation and multi-issuer without token migration.
    Key rotation and multi-issuer are the same mechanism: add/remove
    keys from the registry.

    BOOTSTRAP (genesis): The first registry is created during asset
    definition (§11.1). It contains exactly one key: the genesis
    issuer_pk. It is self-signed by that key. epoch=0.

        initial_registry = AssetKeyRegistry(
            asset_id           = asset_id,
            asset_genesis_cid  = asset_genesis_cid,
            authorized_keys    = [genesis_issuer_pk],
            registry_root      = Merkle([genesis_issuer_pk]),
            epoch              = 0,
            previous_root      = Merkle([genesis_issuer_pk]),  # self-referential at genesis
            signatures         = [(genesis_issuer_pk,
                                   Sign(genesis_issuer_sk, signing_payload))],
        )

    This is published as an Iroh blob alongside the GenesisRecord.
    The CID is announced on the registry gossip topic (§42).
    Subsequent rotations chain from this initial registry.

    PUBLICATION: On creation or rotation, the registry is:
    1. Serialized and published as an Iroh blob → registry_cid.
    2. Announced on gossip topic:
       REGISTRY_PUBLISH_TOPIC_PREFIX + asset_id[:4].hex()
    3. The next IssuerEligibleRoot references registry_root
       and registry_epoch (§19.9).
    Nodes cache current + previous registry (~300 bytes each).
    Registries older than REGISTRY_MAX_AGE_EPOCHS + 1 are GC'd.
    """
    asset_id: bytes
    asset_genesis_cid: bytes
    authorized_keys: List[bytes]        # Current valid issuer PKs (sorted)
    registry_root: bytes                # Merkle(sort(authorized_keys))
    epoch: int
    previous_root: bytes                # Registry root of the prior epoch.
                                        # == registry_root at epoch=0 (genesis).
                                        # Enables fold circuit trust chain (§33 constraint 7).
    signatures: List[Tuple[bytes, bytes]]  # [(pk, sig), ...] M-of-N
                                        # Signatures from keys in PREVIOUS registry
                                        # (or self-signed at epoch=0).

    @staticmethod
    def rotation_threshold(key_count: int) -> int:
        """Compute required M for M-of-N registry authorization.
        Uses protocol constants from §42."""
        return max(
            REGISTRY_MIN_SIGNERS,
            -(-key_count * REGISTRY_ROTATION_THRESHOLD_NUMERATOR
              // REGISTRY_ROTATION_THRESHOLD_DENOMINATOR)  # ceil division
        )

    def verify(self, previous_registry: 'AssetKeyRegistry' = None) -> bool:
        """Verify M-of-N signatures from the PREVIOUS registry's keys.

        At epoch=0 (genesis): self-signed. previous_registry is None
        and signatures are checked against self.authorized_keys.
        At epoch>0: signatures must come from previous_registry.authorized_keys.
        """
        if self.epoch == 0 and previous_registry is None:
            # Genesis: self-signed
            signing_keys = self.authorized_keys
        elif previous_registry is not None:
            signing_keys = previous_registry.authorized_keys
            if self.previous_root != previous_registry.registry_root:
                return False  # Chain break
        else:
            return False  # Non-genesis requires previous registry

        required_m = self.rotation_threshold(len(signing_keys))
        valid = 0
        payload = self._signing_payload()
        for pk, sig in self.signatures:
            if pk in signing_keys:
                if Ed25519_Verify(pk, payload, sig):
                    valid += 1
        return valid >= required_m

    def _signing_payload(self) -> bytes:
        return msgpack.packb({
            "asset_id": self.asset_id,
            "asset_genesis_cid": self.asset_genesis_cid,
            "registry_root": self.registry_root,
            "epoch": self.epoch,
            "previous_root": self.previous_root,
        })

    def contains(self, pk: bytes, proof: List[Tuple[bytes, str]]) -> bool:
        """Verify Merkle membership of pk in registry_root."""
        return verify_merkle_proof(merkle_leaf_hash(pk), proof, self.registry_root)


@dataclass
class IssuerKeyRotation:
    """Registry update record. Adds/removes authorized keys.
    Signed by M-of-N existing authorized keys (threshold per §42).
    Published as content-addressed blob and announced on registry topic.

    CONSTRAINT: removed_keys MUST NOT reduce authorized_keys below
    REGISTRY_MIN_SIGNERS. This prevents a rotation that empties
    the registry (bricking the asset).

    CONSTRAINT: A key removed in epoch E MUST NOT sign IssuerEligibleRoots
    with registry_epoch > E. Nodes reject eligible roots where
    issuer_pk ∉ registry at the eligible root's registry_epoch.
    Revocation latency for eligible root signing: one epoch
    (same as witness eligibility revocation, §19.9).
    """
    asset_id: bytes
    asset_genesis_cid: bytes
    old_registry_root: bytes
    new_registry_root: bytes
    added_keys: List[bytes]
    removed_keys: List[bytes]
    effective_epoch: int
    signatures: List[Tuple[bytes, bytes]]  # M-of-N from old registry

    def verify(self, old_registry: AssetKeyRegistry) -> bool:
        """Verify using protocol-defined threshold from old registry."""
        required_m = AssetKeyRegistry.rotation_threshold(
            len(old_registry.authorized_keys))
        if old_registry.registry_root != self.old_registry_root:
            return False  # Must reference correct previous registry
        if self.effective_epoch != old_registry.epoch + 1:
            return False  # Epochs must be sequential
        # Check resulting key count
        new_count = (len(old_registry.authorized_keys)
                     + len(self.added_keys) - len(self.removed_keys))
        if new_count < REGISTRY_MIN_SIGNERS:
            return False  # Cannot empty registry
        valid = 0
        payload = self._signing_payload()
        for pk, sig in self.signatures:
            if pk in old_registry.authorized_keys:
                if Ed25519_Verify(pk, payload, sig):
                    valid += 1
        return valid >= required_m

    def _signing_payload(self) -> bytes:
        return msgpack.packb({
            "asset_id": self.asset_id,
            "asset_genesis_cid": self.asset_genesis_cid,
            "old_registry_root": self.old_registry_root,
            "new_registry_root": self.new_registry_root,
            "added_keys": self.added_keys,
            "removed_keys": self.removed_keys,
            "effective_epoch": self.effective_epoch,
        })

    def apply(self, old_registry: AssetKeyRegistry) -> AssetKeyRegistry:
        """Produce the new registry from old + rotation.
        Caller MUST call verify() first."""
        new_keys = sorted(
            [k for k in old_registry.authorized_keys if k not in self.removed_keys]
            + self.added_keys
        )
        return AssetKeyRegistry(
            asset_id=self.asset_id,
            asset_genesis_cid=self.asset_genesis_cid,
            authorized_keys=new_keys,
            registry_root=merkle_root(new_keys),
            epoch=self.effective_epoch,
            previous_root=old_registry.registry_root,
            signatures=self.signatures,  # Carried forward for verification
        )

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'IssuerKeyRotation':
        return cls(**msgpack.unpackb(data))
```

### 49.9.2 Registry Announcement

```python
@dataclass
class ForcedFinalizationAnnouncement:
    """Broadcast by W_FF witnesses on INAT_ORACLE_TOPIC to notify
    recipient that forced finalization completed and Phase 4 is possible.
    Recipient's app-level daemon triggers Phase 4 on receipt."""
    session_id: bytes
    nullifier_commit: bytes
    forced_finalization_cid: bytes
    spend_cid: bytes
    recipient_identity_commit: bytes
    timestamp: int

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'ForcedFinalizationAnnouncement':
        return cls(**msgpack.unpackb(data))


@dataclass
class RegistryAnnouncement:
    """Gossip message for registry publication.
    Nodes fetch the full registry blob via registry_cid."""
    asset_id: bytes
    epoch: int
    registry_cid: bytes             # CID of serialized AssetKeyRegistry blob
    issuer_pk: bytes                # Signing key (for quick rejection before fetch)
    signature: bytes                # Sign(issuer_sk, H(asset_id || epoch || registry_cid))

    def verify_quick(self) -> bool:
        """Quick signature check before fetching blob.
        Does NOT verify issuer_pk is in any registry —
        full registry verification happens after blob fetch."""
        payload = H(self.asset_id + self.epoch.to_bytes(8, 'big')
                    + self.registry_cid)
        return Ed25519_Verify(self.issuer_pk, payload, self.signature)

    def serialize(self) -> bytes:
        return msgpack.packb(asdict(self))

    @classmethod
    def deserialize(cls, data: bytes) -> 'RegistryAnnouncement':
        return cls(**msgpack.unpackb(data))
```

**Node behavior on RegistryAnnouncement:**
1. Quick-verify signature. Reject if invalid.
2. Reject if epoch ≤ cached registry epoch (stale).
3. Fetch AssetKeyRegistry blob via registry_cid.
4. Verify full registry against cached previous registry
   (AssetKeyRegistry.verify()).
5. Cache new registry. Retain previous for in-flight fold
   verification. GC registries older than
   REGISTRY_MAX_AGE_EPOCHS + 1.

### 49.10 Donation Structures

@dataclass
class DonationReceiptStore:
    """
    Node-local store for donation receipts pushed to this node.
    Issuer-sharded: each inbox_key maps to receipts for one
    (pool_pk, issuer_family_id) pair.
    """
    receipts: Dict[bytes, List[DonationReceipt]]  # inbox_key → receipts

    def store(self, inbox_key: bytes, receipt: DonationReceipt):
        if inbox_key not in self.receipts:
            self.receipts[inbox_key] = []
        self.receipts[inbox_key].append(receipt)

    def query(self, inbox_key: bytes, since_epoch: int = 0) -> List[DonationReceipt]:
        return [
            r for r in self.receipts.get(inbox_key, [])
            if r.epoch >= since_epoch
        ]

    def gc_swept(self, inbox_key: bytes, swept_cids: Set[bytes]):
        """Remove receipts for already-swept SpendRecords."""
        if inbox_key in self.receipts:
            self.receipts[inbox_key] = [
                r for r in self.receipts[inbox_key]
                if r.spend_record_cid not in swept_cids
            ]

---

License: Apache2.0
