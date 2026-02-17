# Inat Protocol

**Ledgerless Bilateral Settlement with Cryptographic Finality**

> *"Finality isn't agreed upon. It's mathematically non-constructible to contradict."*

---

## Abstract

Inat is an open protocol for bilateral value transfer achieving transaction finality through zero-knowledge proofs, threshold cryptography, and distributed witness attestation — without a shared ledger, central operator, or consensus mechanism. Each wallet holds a **slot** — a value container whose liveness is anchored in a single persistent counter stored in the wallet's own P2P document. To transfer value, sender and recipient each sign a commitment; a randomly-selected quorum of witnesses attests the transfer is valid; the nullifier is reconstructed via threshold cryptography once both sides have committed, making reversal mathematically impossible; and the entire history of the slot collapses into a constant-size proof that any party can verify in under a millisecond. No blockchain, no consensus, no central operator.

---

## 1. Introduction

### 1.1 The Problem

Every existing trustless payment system relies on a shared ledger — a single data structure that all participants must agree on. Bitcoin, Ethereum, and their descendants achieve this through consensus mechanisms that trade throughput for agreement. The ledger is the bottleneck: every transaction competes for space in it, every node must process it, and every participant must trust that it won't be rewritten.

This creates three fundamental constraints. Global state requires global agreement, which limits throughput. Every transaction is visible to every participant, which limits privacy. And every node stores every transaction, which limits scalability.

### 1.2 The Insight

Inat eliminates the shared ledger entirely. Instead:

- **Wallet documents** serve as the primary persistent anchor — each wallet's P2P document contains `protocol_seq`, the authoritative record of slot liveness. Monotonic sequencing prevents rollback; distributed subscriptions provide replication.
- **Deterministic nullifiers** published during active transactions provide race condition detection between concurrent spends.
- **Zero-knowledge proofs** validate balance sufficiency, witnessed transaction history, and issuer-attested genesis, folded into constant-size verification.
- **Threshold cryptography** makes finalization inevitable once bilateral commitment is achieved.
- **Distributed witnesses** selected via verifiable random functions attest transaction validity.

A dormant wallet needs only its document's sequence entry to determine slot liveness. The ZK proof then validates the claimed state: correct balance, properly witnessed history, valid issuer chain.

### 1.3 Design Principles

| Principle | Meaning |
|-----------|---------|
| **The oracle persists, the proof validates** | The wallet document is the persistent anchor; ZK proofs validate balance, witness history, and issuer genesis. Neither alone suffices. |
| **Witnesses validate math, not data** | ZK proofs encode all transaction logic; witnesses verify cryptographic validity only. |
| **Finalization is inevitable, not voluntary** | Once threshold shares are distributed, no party can prevent completion. |
| **Shard observers learn less than witnesses** | Two-phase witness engagement: gossip broadcasts carry only lottery data; identity and proof data reach only selected witnesses. |
| **Issuer-gated, decentralized-executed** | Witness admission is permissioned; transacting requires only a keypair and funded slot. The issuer is the bouncer for the witness pool, not for the venue. |
| **Symmetric participation** | Every funded, issuer-approved wallet is eligible to witness for others, incentivized via fee shares. |

### 1.4 Key Properties

| Property | How It's Achieved |
|----------|-------------------|
| **No double-spend** | Oracle sequence advancement + nullifier for in-flight race detection |
| **No sender griefing** | Threshold encryption prevents early nullifier reveal |
| **No receiver griefing** | Bilateral signatures required for commitment |
| **No witness griefing** | VRF selection + shard hopping across phases |
| **Privacy** | ZK proofs hide amounts, identities, and history from witnesses |
| **O(1) verification** | ~560 byte proof, <1ms verification, regardless of history depth |
| **GC-resistant** | Oracle persists; proof carries validated history; old data safely collected |
| **Issuer-gated identity** | VRF eligibility requires issuer approval + funded wallet |

### 1.5 Domain

Inat targets P2P game economies, virtual goods, network-native point systems, and any domain where an issuer defines assets for an application ecosystem. The protocol is agnostic to external valuation.

---

## 2. Architecture

### 2.1 Layer Separation

Inat operates as an application layer on top of existing P2P infrastructure (Iroh), which provides transport (QUIC), peer discovery (DHT), gossip (sharded topics), mutable documents (CRDT, per-author monotonic), and content-addressed blob storage. Inat adds value transfer logic, witness selection, attestation protocols, ZK circuits, and threshold cryptography on top.

Inat nodes are a strict subset of the P2P network. Plain infrastructure nodes route Inat traffic without understanding it.

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: INAT APPLICATION                                      │
│  Inat-enabled nodes, witness/attestation logic, 4-phase         │
│  protocol, ZK circuits, threshold crypto                        │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2: P2P INFRASTRUCTURE (Iroh)                             │
│  All nodes: DHT, gossip, documents, content store, QUIC         │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 1: INTERNET                                              │
│  TCP/QUIC transport                                             │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Network Topology

Inat nodes form a sharded overlay within the larger P2P network. Each node joins gossip shards based on its NodeID prefix AND the assets it supports. Shards are asset-scoped — a node whitelisting 5 assets joins 5 independent shard namespaces.

Transactions "hop" between shards across phases. Each phase's shard is derived from the previous phase's attestation entropy, so an attacker controlling one shard loses control at each hop.

```
GOSSIP SHARDS (prefix-based, ~1000 nodes each, per asset)

  Shard 0x3A...    Shard 0x7F...    Shard 0xB2...
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ ~1000    │    │ ~1000    │    │ ~1000    │
  └──────────┘    └──────────┘    └──────────┘
       │                                │
       └──── Phase 1 seed maps here     │
             Phase 2 seed hops here ────┘
```

Shard depth adapts per-asset based on supporter density: splitting when a shard exceeds 2400 nodes and merging below 600. At 100M nodes, depth ≈ 17 with ~131,072 shards. At 10K nodes, depth ≈ 4 with ~16 shards. Each transaction always touches ~1000 nodes regardless of total network size.

---

## 3. Identity and Value Model

### 3.1 Deterministic Slots

Every wallet is one keypair (`sk`/`pk`). All slots derive deterministically:

```
slot_commit      = H("inat_slot:"      ‖ sk ‖ seq)
nullifier        = H("inat_nullifier:" ‖ sk ‖ seq)
nullifier_commit = H(nullifier)
identity_commit  = H(pk)
```

Liveness rule: `protocol_seq == creation_seq → live; > creation_seq → dead`.

This enables wallet recovery from a single secret key, sequence-based liveness checking via a single integer, and ZK-verifiable nullifier binding. Each `(sk, seq)` pair maps to exactly one slot with no address reuse ambiguity.

### 3.2 Separation of Identity and Value

Inat cleanly separates two orthogonal concerns:

**Identity layer (slot liveness):** One `sk/pk` = one identity = one slot at any given sequence. Liveness is an O(1) lookup from the wallet document. Rollback prevention is enforced by infrastructure-level monotonic sequencing.

**Value layer (what the slot holds):** Balance commitments, lineage proofs, issuer family, and the collapsed ZK proof — all independent of identity. O(1) verification regardless of history depth.

Because these are separate, verifiers can fail fast on cheap identity checks (nullifier lookup, sequence check) before expensive ZK verification.

### 3.3 Value Flow

Each party independently manages their own slot chain. A transaction consumes the sender's current slot and populates the recipient's next slot. Neither party participates in deriving the other's slots.

```
SENDER (Alice):
  Slot seq=N (bal=100) ──spend──▶ Slot seq=N+1 (bal=30, change)
       │
       │ nullifier for seq=N published (race detection)
       │ wallet doc protocol_seq: N → N+1
       │
       │ 70 units transferred
       ▼
RECIPIENT (Bob):
  Slot seq=M (bal=50)           Slot seq=M+1 (bal=120, receives)
                                wallet doc protocol_seq: M → M+1
```

Both sequences advance independently. Both next slots are predetermined from `(sk, seq+1)`. Slot derivation is unilateral. Attestation is bilateral.

### 3.4 Privacy

Both sender and recipient have identical privacy guarantees. Neither party is more exposed than the other.

| Property | Sender | Recipient | Witnesses (~29) | Shard (~971) | Public |
|----------|--------|-----------|-----------------|--------------|--------|
| Identity (pk) | Hidden | Hidden | Hidden | Hidden | Hidden |
| Amount | Known | Known | Hidden | Hidden | Hidden |
| Nullifier | Known | Hidden | Hidden | Commit only | Hash only |
| Asset type | Known | Known | Known | Known | Hidden |

Privacy is structurally enforced through a type-level invariant: only commitment-only projections can be serialized into documents, wire messages, or witness requests. A privacy leak requires violating the projection boundary — it cannot happen through normal serialization paths.

---

## 4. Transaction Lifecycle

### 4.1 The Four Phases

```
┌────────────┐  ┌───────────┐  ┌──────────┐  ┌──────────┐
│PENDING_SEND│─▶│ COMMITTED │─▶│  SPENT   │─▶│FINALIZED │
└────────────┘  └───────────┘  └──────────┘  └──────────┘
     │               │              │              │
     ▼               ▼              ▼              ▼
 Phase 1:       Phase 2:       Phase 3:       Phase 4:
 Sender signs,  Recipient      W₁ releases    DA confirm,
 W₁ attests +   accepts, W₂₃   shares,        fold into
 holds shares   quorum=locked  nullifier pub  O(1) proof
```

**Phase 1 — Initiate.** Sender increments their own `protocol_seq`, signs a commitment, and broadcasts an intent to the asset-scoped gossip shard. ~29 VRF-selected witnesses independently fetch the sender's wallet document from the P2P network, verify the ZK proof, check that the nullifier hasn't been published, and return attestations. The sender then distributes encrypted Shamir shares to each attesting witness.

**Phase 2 — Accept & Commit.** Recipient signs their commitment, increments their own `protocol_seq`, and creates a receiving slot. A fresh set of ~29 witnesses in a *different* shard independently fetch *both* wallet documents, verify bilateral binding, and attest. Once 21 valid attestations arrive — the **point of no return** — the transaction is COMMITTED.

**Phase 3 — Share Release.** The sender relays the Phase 2 attestation bundle to each Phase 1 witness. They verify the bundle, decrypt their Shamir share, and publish it. Once 21+ shares are public, anyone can reconstruct the nullifier and publish it.

**Phase 4 — Confirm & Fold.** 7 VRF-selected validators in a third shard confirm data availability. The fold circuit absorbs the previous proof plus the current spend record into a new collapsed proof (~560 bytes, <1ms verification). The slot transitions to FINALIZED.

### 4.2 Point of No Return

The critical design property is that **Phase 2 quorum is irreversible**:

- Recovery is blocked — witnesses reject recovery requests for COMMITTED slots.
- The sender cannot withhold — shares are already distributed to Phase 1 witnesses.
- Finalization is inevitable — Phase 1 witnesses release shares upon seeing Phase 2 quorum.

After commitment, the only question is *when* finalization occurs, not *whether*.

### 4.3 Why Bilateral Signatures Matter

Both parties must sign because: the sender's signature proves intent to spend a specific slot to a specific recipient; the recipient's signature proves acceptance and creates binding; neither can be forged; and neither can claim ignorance — bilateral commitment is cryptographically proven. Signatures are verified natively (Ed25519) by witnesses and bound into the fold hash chain.

### 4.4 Discovery Protocol

Before Phase 1, sender and recipient establish contact through a layered-reveal discovery protocol with escalating commitment cost:

1. **Payment token** (public, safe to share via QR/URL/NFC) — contains recipient's identity commit, issuer, amount hint, expiry. Reveals nothing useful.
2. **Capacity proof** (recipient → sender, ZK) — proves a slot exists with correct issuer and sufficient balance without revealing which slot.
3. **Locked funds proof** (sender → recipient) — proves the sender locked funds, costing an irreversible sequence increment. This is the anti-sniffing gate.
4. **Spot reveal** (recipient → sender, ECDH-encrypted) — reveals the receiving slot address only to the sender.

Each step increases an attacker's probing cost. Recovery of a locked-but-unused slot requires a full 21/29 witness quorum.

---

## 5. Witness System

### 5.1 Shard-Based Self-Selection

There is no witness registry, no stake, and no credit system. Any Inat node that hears a transaction intent on its gossip shard checks its own VRF — if it passes threshold, it's a witness.

The VRF threshold is a hardcoded protocol constant:

```
VRF_THRESHOLD = (TARGET_WITNESSES / SHARD_TARGET_SIZE) × 2²⁵⁶
```

Shard depth adaptation maintains shard population near TARGET_SHARD_SIZE (~1000), so the ratio stays stable as the network grows. Heartbeats are used only for shard depth consensus (split/merge decisions), not for VRF calibration.

### 5.2 Two-Round Engagement

Privacy is preserved through a two-phase witness engagement:

**Round 1 — Envelope** (broadcast to ~1000 shard nodes): Contains only VRF lottery data — seed, session ID, nullifier commit, beacon round, shard depth. No identity, asset, or proof data. ~29 VRF winners reply with eligibility proofs.

**Round 2 — Payload** (unicast to each selected witness): Contains identity commit, asset ID, ZK proof, public inputs, and all verification material. Each witness independently fetches wallet documents, runs full verification, and replies via direct QUIC.

Result: ~971 shard nodes learn only "a transaction for this asset exists." Only ~29 selected witnesses see transaction details. Witnesses don't see each other's attestations.

### 5.3 Distance-Weighted Selection

To prevent local Sybil packing — an attacker owning many nodes in one small XOR range — the VRF threshold incorporates XOR distance from a seed-derived anchor. Nodes farther from the anchor pass the lottery more easily, forcing spatial diversity.

```
anchor      = H("inat_diversity_anchor:" ‖ seed)
xor_dist    = max(uint128(xor(node_id, anchor) >> 128), 1)
adjusted_T  = VRF_THRESHOLD × min(xor_dist, MAX_DIVERSITY_BOOST)
selected    = uint256(vrf_output) < adjusted_T
```

When more than 29 pass threshold, a diversity score (VRF output × XOR distance) ranks candidates. The top 29 receive shares; the first 21 to deliver valid attestations become fee-eligible.

### 5.4 Shard Hopping

Each transaction phase targets a **different shard**, derived from the previous phase's attestation entropy:

```
seed₁  = H(slot_commit ‖ beacon₁ ‖ randomness₁ ‖ asset_id)
seed₂₃ = H(seed₁       ‖ H(σ_r)  ‖ beacon₂₃    ‖ randomness₂₃)
seed₄  = H(seed₂₃      ‖ spend_cid ‖ beacon₄    ‖ randomness₄)
```

`seed₂₃` depends on the recipient's signature `σ_r` (unpredictable until Phase 1 completes), providing grinding resistance. An attacker cannot predict which shard Phase 2 will land in until Phase 1 completes. By then, Sybils in shard₁ are irrelevant.

The fold circuit enforces shard membership — each witness's NodeID must have the correct prefix for the shard derived from that phase's seed. A Sybil with a valid VRF proof but wrong shard prefix cannot appear in a valid fold proof.

At 100M nodes (~131,072 shards), an attacker controlling 10% of the network faces P(attack) ≈ 10⁻²⁷ per transaction — must control 21/29 in three independent shards.

### 5.5 Issuer-Gated Eligibility

All witnesses must hold an issuer-signed eligibility certificate and minimum balance of the asset. The issuer publishes an `IssuerEligibleRoot` each epoch (~1 hour) — a Merkle root of all eligible wallet PKs. Witnesses carry Merkle proofs of inclusion. The fold circuit verifies these proofs, so fake witnesses without issuer approval cannot produce valid fold proofs.

The issuer controls *who can witness*. The protocol determines *which* eligible witnesses are selected. The issuer cannot control transaction outcomes, retroactively censor finalized proofs, or gate who transacts.

---

## 6. Threshold Cryptography

### 6.1 The Problem

The sender knows the nullifier — it's derived from their secret key. Without protection, the sender could reveal it early (burning funds maliciously), threaten to reveal it (extorting the recipient), or abort after the recipient commits (griefing).

### 6.2 The Solution

During Phase 1, the sender splits the nullifier into 29 Shamir shares (21 required to reconstruct) and encrypts each to a specific witness's public key. Witnesses release their shares only after observing Phase 2 quorum.

```
Phase 1: Sender generates shares, distributes encrypted to W₁
Phase 2: W₂₃ quorum achieved → point of no return
Phase 3: Sender relays W₂₃ bundle to W₁ → W₁ verify, release shares
         21+ shares published → nullifier reconstructed → published
```

| Property | Guarantee |
|----------|-----------|
| Sender can't reveal early | Needs 21 witness shares |
| Sender can't withhold | Shares already distributed |
| Witnesses can't forge | Feldman commitments verify |
| Partial reveal is useless | Need exactly 21 shares |
| Finalization is inevitable | Once W₂₃ quorum, W₁ releases |

---

## 7. Proof Folding

### 7.1 The Problem

Validating a slot's claimed state naively requires walking every spend record back to genesis — checking each transition was witnessed, each balance correct. Cost: O(depth) verification, unbounded storage.

### 7.2 What the Proof Validates

The collapsed proof guarantees three things about a slot:

1. **Balance sufficiency** — the sender actually has the claimed amount.
2. **Witness history** — every prior transition had a legitimate quorum of VRF-selected witnesses attest it.
3. **Issuer genesis** — the chain traces to a valid issuer-signed mint.

### 7.3 Recursive Folding

Each transaction folds the previous proof into a new proof. The new proof validates: old proof was valid + new transition was properly witnessed + balance math checks out.

```
Depth 1: verify(issuer_sig, genesis_record) → π_fold_1
Depth N: verify(π_fold_{N-1}) + verify(phase proofs)
         + verify(7 DA confirmations) + verify(VRF selection)
         + verify(shard membership) + verify(seed chain)
         + verify(beacon recency) + verify(issuer registry)
         → π_fold_N  (~560 bytes, <1ms verification)
```

The verification cost is always <1ms whether the slot has been transferred once or a million times. The proof is ~560 bytes regardless of depth.

### 7.4 Relationship to the Wallet Document

The wallet document says: "protocol_seq = N → slot at seq=N is live" (persistent, owner-written, monotonic). The proof says: "slot at seq=N legitimately holds X balance with valid witnessed history back to issuer genesis." Both are required — wallet document for liveness, proof for legitimacy.

### 7.5 Temporal Integrity

Consecutive fold proofs must advance at a real-time rate. A beacon recency chain constrains the gap between folds to ~20 hours, preventing backdating or temporal gap exploitation.

---

## 8. State Management

### 8.1 Persistence Hierarchy

```
MUST PERSIST (Network — Wallet Documents):
  • protocol_seq, slot status, tx context
  • Owner is sole writer; monotonicity prevents rollback
  • XOR-sharded subscriptions (k=50) provide replication
  • This is the ONLY thing that must persist network-wide

MUST PERSIST (Locally by Owner):
  • Secret key (sk) — derives all slots, nullifiers
  • Current collapsed proof — ~560 bytes, self-authenticating

REAL-TIME ONLY (During Active Transactions):
  • Nullifiers — race detection; GC-safe once oracle advances

MAY GARBAGE COLLECT:
  • Old spend records (folded into proof)
  • Old witness bundles (folded into proof)
  • Session state (after finalization)
```

Network blobs are redundancy, not authority. Proofs are self-authenticating — source doesn't matter, math verifies. If a blob disappears, the owner provides their local copy.

### 8.2 Data Discovery

Because the infrastructure DHT is peer-routing (not content-routing), Inat uses a unified **push-then-query** pattern for all data discovery. The publisher pushes data to XOR-responsible peers at write time; readers query those same deterministic peers at read time.

| Store | Responsible Peers | Rationale |
|-------|-------------------|-----------|
| Wallet docs (oracle) | k=50 XOR-closest | Authoritative liveness anchor; high redundancy |
| Nullifiers | k=20 XOR-closest | Ephemeral race detection; lower redundancy sufficient |
| Content blobs | k=20 XOR-closest | Redundancy, not authority; owner holds local copy |

### 8.3 Propagation

Three channels ensure state reaches all relevant parties:

1. **Gossip broadcast** — fast (~1–5s), best-effort notification.
2. **Doc sync push** (k=50) — reliable, after each witness event.
3. **Pulse anti-entropy** — heals gaps every 5 minutes via neighbor summary exchange.

Witnesses fail closed on unreachable documents: if a wallet document cannot be fetched, attestation is rejected.

---

## 9. Recovery

### 9.1 Design Principle

All state transitions are witnessed — including recovery. Recovery from a stuck PENDING_SEND state uses the same 21/29 threshold as spending. This prevents the fake-abort double-spend attack where an attacker completes a transaction, then "recovers" the spent slot.

### 9.2 Two-Phase Recovery

Recovery uses two phases with a cooling period to prevent race conditions across network partitions:

**Phase A — Recovery Intent** (at 1-hour timeout): Owner publishes intent. W_RA witnesses (21/29) independently fetch slot state, verify it's still PENDING_SEND, verify no Phase 2 quorum exists, and attest. Sequence is NOT incremented.

**Cooling Period** (10 minutes, 2× Pulse interval): Allows Pulse to propagate any Phase 2 quorum from the other side of a partition, or propagate the recovery intent to the recipient's side.

**Phase B — Recovery Confirm** (after cooling): W_RB witnesses (21/29, fresh VRF selection) re-check that the slot is still PENDING_SEND and no Phase 2 quorum appeared during cooling. If confirmed, sequence increments, new slot created with identical balance, old slot transitions to RECOVERED (terminal).

**Reciprocal defense:** Phase 2 witnesses also check for recovery intent during their independent fetch. If recovery intent exists with valid attestations, Phase 2 is rejected. This creates mutual exclusion — at least one side sees the other's signal before completing.

Recovery is blocked from COMMITTED state onward. A COMMITTED transaction proceeds to finalization via the normal path, with a forced-finalization fallback at 72 hours if share collection fails.

---

## 10. Issuer Model

### 10.1 Asset Identity

Asset identity is anchored at `asset_genesis_cid` — the immutable CID of the asset's creation record:

```
asset_id = H("inat_asset:" ‖ asset_genesis_cid ‖ name ‖ params)
```

This identity is permanent and independent of any signing key. Key rotation, multi-issuer, and governance changes are all transparent to the asset's identity.

### 10.2 Key Registry

Authorized issuer keys are maintained in a rolling `AssetKeyRegistry` — a Merkle tree of public keys, signed by M-of-N existing authorized keys. Key rotation means updating the registry; no token migration is needed.

The fold circuit verifies `issuer_pk ∈ issuer_registry_root` via Merkle proof, and that each registry chains back to genesis through its predecessor. Verifiers need the registry root, not a specific public key.

### 10.3 What the Issuer Controls (and Doesn't)

| Controls | Cannot Control |
|----------|---------------|
| Who is eligible to witness (append-only, irrevocable) | Which eligible witnesses are selected (VRF) |
| Minimum balance for witness eligibility | Transaction outcomes (ZK proofs) |
| How many witness-eligible wallets per identity | Retroactive censorship of finalized proofs |
| | Who transacts with whom |
| | Removal of previously admitted witnesses |

The issuer is the bouncer for the witness pool, not for the venue.

---

## 11. Economics

### 11.1 Fee Principle

All mandatory fees go 100% to attesting witnesses. There are no mandatory protocol fund allocations, no hardcoded system wallets, and no percentage-based creator or developer splits.

```
Standard:  input = amount + change + (21 × fee_share)
           21 fee-eligible witnesses each earn 1 share

Donation:  input = amount + change + (22 × fee_share)
           21 witnesses + 1 ghost entry for voluntary donation
           Witnesses earn identically in both variants
```

### 11.2 Deferred Mint

Fees are not "sent" to witnesses during the transaction. They are a proven gap in the balance equation — value deducted from the sender with no output slot. Witnesses collect later via a sweep circuit that proves their participation across multiple transactions and mints a new issuer-bound slot with the accumulated amount.

Value is never created or destroyed. The fee is deducted from circulation, then re-enters when swept. SpendRecords are the proof that the debt exists. The sweep circuit is the proof that the debt is legitimately collected.

### 11.3 Double-Sweep Prevention

Sweep uses two layers of double-claim prevention, mirroring the transfer double-spend architecture:

| Layer | Transfer | Sweep |
|-------|----------|-------|
| Fast-path (ephemeral) | NullifierStore | NullifierStore (sweep_nullifier) |
| Durable (authoritative) | Oracle seq kills input slot | sweep_root accumulator in wallet doc |

Sweep outputs are bound to the claimant's own wallet so the accumulator and the output live in the same document, preventing re-sweep attacks after ephemeral store garbage collection.

### 11.4 Witness Sovereignty

Witnesses have full discretion over which issuers they validate. Each witness only earns fees denominated in issuers they chose to attest. Economically rational witnesses reject fees in worthless tokens — no protocol-level enforcement required.

### 11.5 Optional Donation

The donation system is voluntary, user-initiated, and has zero impact on protocol operation or witness compensation. It adds a guardian's public key as a 22nd entry in the witness root. The guardian collects via the same sweep circuit as witnesses, choosing the target wallet at sweep time. No hardcoded destination exists in the protocol.

If the guardian ignores donations in a particular issuer, they burn permanently. The protocol is indifferent.

---

## 12. Security Analysis

### 12.1 Threat Model

| Threat | Defense |
|--------|---------|
| Double-spend | Oracle seq + nullifier + terminal state check |
| Equivocation (split-brain wallet doc) | Witnesses independently fetch from k=50 subscribers; monotonicity prevents presenting different seq values; shard hopping means W₁ and W₂₃ fetch from independent node sets |
| Fake-abort double-spend | Two-phase witnessed recovery (21/29 quorum × 2 phases) + independent state fetch |
| Sender griefing | Threshold encryption (can't reveal nullifier alone) |
| Receiver griefing | Bilateral signatures required for commitment |
| Witness Sybil | Issuer-gated eligibility + VRF threshold + fold circuit shard membership proof |
| Shard grinding | Shard hopping (seed depends on σ_r + beacon) + ZK shard prefix constraint |
| Eclipse | 50 doc subscribers + Pulse anti-entropy + owner-only writes |
| Key compromise | Registry rotation; old keys can't sign new eligible roots post-rotation |
| Beacon outage | Stalls new transactions; cannot corrupt in-flight state |
| Double-sweep | Durable sweep accumulator + NullifierStore fast-path |

### 12.2 Evidence Trust Separation

Witnesses distinguish two categories of evidence:

**Self-authenticating** (submitted by requester, verified locally): ZK proofs, signatures, Pedersen commitments, Feldman VSS commitments, VRF proofs, beacon values.

**Mutable state** (fetched independently from P2P network): Slot status, sequence number, nullifier existence, session binding, asset ID.

Witnesses never trust requester-supplied values for mutable state. If a wallet document is unreachable, the attestation is rejected — witnesses cannot attest what they cannot verify.

### 12.3 Collusion Resistance

Even if sender and recipient collude, witness selection remains unpredictable. A colluding requester cannot fabricate slot state because witnesses fetch independently and only the wallet owner can write to their document.

Colluding parties who grind the recipient's signature to steer shard selection face two barriers: the fold circuit enforces shard prefix membership (costing O(2^depth) key-grinds per Sybil), and cross-phase depth consistency requires Sybil coverage across three independent shards. At depth=17, placing 1000 Sybils in three shards costs ~400M keypair grinds — and the target shards shift with each beacon round.

### 12.4 Eclipse Attack Resistance

Eclipse attacks face layered defenses: 50 XOR-sharded doc subscribers, witness doc sync push, Pulse anti-entropy every 5 minutes, gossip broadcast, and owner-only writes with ZK verification on use. A Sybil controlling subscriber nodes can only withhold data (liveness attack), not forge it (safety attack). Any single honest node in the shard can heal the rest.

---

## 13. Network Bootstrap and Scaling

### 13.1 Activation

Transaction processing requires ≥100,000 active supporters for the specific asset, measured by per-asset shard heartbeat density. Activation is per-asset, not global — a major stablecoin at depth=17 coexists with a niche token at depth=7 on the same network.

Below threshold, nodes join the network, broadcast heartbeats, and participate in data replication. Above threshold, transactions activate with no registration step.

### 13.2 Scaling

As the network grows, shard depth adapts via split/merge. Witness count and threshold remain constant. Each transaction always touches ~1000 nodes regardless of total network size.

| Asset Supporters | Depth | Shards | Nodes/Shard |
|------------------|-------|--------|-------------|
| 100K | 7 | 128 | ~780 |
| 1M | 10 | 1,024 | ~975 |
| 100M | 17 | 131,072 | ~760 |
| 1B | 20 | 1,048,576 | ~950 |

Per-node doc subscriptions are bounded regardless of network size at ~5M docs (~1GB storage). As wallets grow, more nodes are needed to maintain redundancy, but no single node is overwhelmed.

---

## 14. Performance

| Operation | Target | Maximum |
|-----------|--------|---------|
| ZK proof generation (sender) | 2s | 10s |
| ZK proof generation (recipient) | 1s | 5s |
| Fold proof verification | <1ms | 2ms |
| Single attestation round-trip | 500ms | 2s |
| Full attestation collection | 5s | 30s |
| End-to-end transaction | 30s | 120s |

| Data | Size |
|------|------|
| Collapsed fold proof | ~560 bytes |
| Spend record | ~5 KB |
| Witness bundle (21) | ~3.5 KB |
| Nullifier | 32 bytes |

---

## 15. Comparison

Inat differs from existing systems at a structural level:

**vs. Blockchains (Bitcoin, Ethereum):** No global ledger, no consensus, no block production. Transactions are bilateral with distributed attestation. No throughput ceiling imposed by block size or consensus latency.

**vs. Payment Channels (Lightning):** No channel setup, no liquidity locking, no routing. Every transaction is independent. No capital efficiency problem.

**vs. Privacy Coins (Zcash, Monero):** Similar hiding of amounts and addresses via ZK proofs. Key difference: no global UTXO set to analyze, no publicly visible transaction graph. Privacy comes from structural information scarcity, not obfuscation of public data.

**vs. DAGs (IOTA, Nano):** No global DAG structure to maintain. Finality is per-transaction, not dependent on subsequent transactions confirming previous ones. No coordinator or tip selection.

---

## 16. Conclusion

Inat demonstrates that trustless value transfer does not require a shared ledger. By combining a persistent wallet-level oracle, recursive ZK proof folding, threshold cryptography, and VRF-selected distributed witnesses, the protocol achieves transaction finality that is mathematically non-constructible to contradict — in ~30 seconds, at ~560 bytes, with <1ms verification, regardless of transaction history depth.

The protocol's scalability is horizontal: each transaction touches ~1000 nodes regardless of network size. Its privacy is structural: information scarcity, not obfuscation. Its economics are aligned: witnesses earn fees for computational work, not rent extraction.

Inat is not a blockchain without a chain. It is a fundamentally different architecture for value transfer — one where finality is a property of mathematics, not a product of agreement.

---

*For implementation details, circuit specifications, and normative definitions, see the Inat Protocol Specification and Quick Reference.*
