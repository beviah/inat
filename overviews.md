# Inat Protocol — Fractal Design Overviews

---

## L1 — Conceptual Overview (One Paragraph)

Inat is a bilateral value-transfer protocol achieving transaction finality without a shared ledger. Each wallet holds a **slot** — a value container whose liveness is anchored in a single persistent counter (`protocol_seq`) stored in the wallet's own P2P document. To transfer value, sender and recipient each sign a commitment; a randomly-selected quorum of **witnesses** attests the transfer is valid; the nullifier is reconstructed via threshold cryptography once both sides have committed, making reversal mathematically impossible; and the entire history of the slot collapses into a constant-size **ZK proof** that any party can verify in under a millisecond. The infrastructure layer (Iroh P2P) provides transport, documents, and content-addressed storage; Inat adds value logic, witness selection, and cryptographic finality on top. No blockchain, no consensus, no central operator.

---

## L2 — System Overview

### Five Pillars

| Pillar | Role | Key Mechanism |
|--------|------|---------------|
| **Wallet / Slot** | Value containers with deterministic identity | `slot_commit = H(sk ‖ seq)`; liveness = `protocol_seq == creation_seq` |
| **4-Phase Lifecycle** | Ordered state machine from initiation to finality | PENDING_SEND → COMMITTED → SPENT → FINALIZED |
| **Witness System** | Distributed validation without a registry | VRF self-selection within asset-scoped gossip shards; 21-of-29 quorum |
| **Threshold Crypto** | Inevitable finalization once both parties commit | Feldman VSS; nullifier reconstructed from ≥quorum shares |
| **Proof Folding** | O(1) verification of unbounded history | Recursive ZK fold; ~560-byte proof regardless of depth |

### Data Plane

```
Owner Device          Iroh Network              Witnesses (~29)
─────────────         ─────────────             ───────────────
sk, proof (local)  ↔ wallet doc (oracle)    ↔ attest via QUIC
                   ↔ blobs (proofs/records)
                   ↔ nullifiers (ephemeral)
```

- **Wallet doc** is the sole authoritative liveness record. Only the owner writes it; Iroh's monotonic sequencing prevents rollback.
- **Blobs** hold proofs, spend records, witness bundles. Content-addressed, immutable, GC-safe after folding.
- **Nullifiers** are ephemeral race-detection records, GC-safe once the oracle advances.

### Trust Boundaries

- Witnesses fetch mutable state **independently** — they never trust requester-supplied slot status.
- ZK proofs are **self-authenticating** — source doesn't matter, math verifies.
- Issuers gate **witness eligibility** (who can attest); they cannot gate **transacting** (who can send/receive).

---

## L3 — Architectural Overview

### 1. Identity & Slot Model

Every wallet is one keypair (`sk`/`pk`). All slots derive deterministically:

```
slot_commit      = H("inat_slot:"      ‖ sk ‖ seq)
nullifier        = H("inat_nullifier:" ‖ sk ‖ seq)
nullifier_commit = H(nullifier)
identity_commit  = H(pk)            ← wallet doc key / oracle lookup
```

Liveness rule: `protocol_seq == creation_seq → live; > creation_seq → dead`.
Wallet doc is the oracle. No separate oracle subsystem.

### 2. Transaction Lifecycle (4 Phases)

```
Phase 1 — INITIATE
  Sender: increment own protocol_seq (N→N+1), sign commitment,
          broadcast IntentEnvelope to asset-scoped gossip shard.
  W₁ (VRF-selected, ~29 invited, 21 fee-eligible):
    - Independently fetch sender's wallet doc
    - Verify ZK proof π₁, VRF eligibility, nullifier absent
    - Return attestation; receive encrypted Shamir share

Phase 2 — ACCEPT & COMMIT                      ← Point of no return
  Recipient: increment own protocol_seq (M→M+1), sign commitment.
  W₂₃ (fresh shard, same asset):
    - Independently fetch BOTH wallet docs
    - Verify bilateral binding (σ_s, σ_r natively; proof π₂₃ via ZK)
    - W₂₃ quorum achieved → COMMITTED

Phase 3 — SHARE RELEASE
  Sender relays W₂₃ bundle to each W₁ (via stored reply_paths).
  W₁ witnesses: verify bundle, decrypt share, publish to Iroh blob.
  Any party: reconstruct nullifier from ≥quorum shares, publish.

Phase 4 — CONFIRM & FOLD                       ← New shard (W₄)
  7 DA validators confirm data availability.
  Fold circuit: previous proof + current spend record → new
  collapsed proof (~560 bytes, O(1) verification).
```

Recovery path: Two-phase witnessed abort (R₁ then R₂, different shards) from PENDING_SEND or PENDING_RECEIVE — not available post-COMMITTED.

### 3. Witness Selection & Shard Architecture

```
Asset-scoped gossip shard (~1000 Inat nodes)
  │
  ├─ Shard depth adapts per asset: split at 2400 nodes, merge at 600
  ├─ VRF threshold: (TARGET_WITNESSES / SHARD_TARGET_SIZE) × 2²⁵⁶ [constant]
  ├─ Distance-weighted scoring: far nodes pass threshold more easily
  │    adjusted_T = VRF_THRESHOLD × min(xor_dist(node, anchor), MAX_BOOST)
  └─ Two-round engagement:
       Round 1: IntentEnvelope broadcast → VRF winners send eligibility proof
       Round 2: IntentPayload unicast → full attestation (only ~29 see details)
```

Each phase targets a **different shard**: `seed_{n+1} = H(seed_n ‖ H(σ_r) ‖ beacon ‖ rand)`. Attacker controlling one shard loses control at each hop.

### 4. Issuer Model & Key Rotation

Asset identity anchored at `asset_genesis_cid` (immutable). Authorized keys in a rolling **AssetKeyRegistry** (Merkle tree, M-of-N signed). Key rotation = update registry; no token migration. Fold circuit verifies `issuer_pk ∈ issuer_registry_root` via Merkle proof (~21K constraints). Each witness carries an `EligibilityProof` — the full signed `IssuerEligibleHeader` plus its Merkle membership proof — inline in `WitnessEligibility` responses. Headers are delivered directly from issuer to eligible wallets at epoch rollover (~hourly); they are NOT published as blobs and NOT gossiped. Attestation bundles are self-contained.

### 5. Proof Folding

```
Depth 1: verify(issuer_sig, genesis_record) → π_fold_1
Depth N: verify(π_fold_{N-1}) + verify(π₁, π₂₃) + verify(7 DA confirmations)
         + verify(VRF selection, shard membership, seed chain)
         + verify(beacon recency chain, issuer registry chain)
         → π_fold_N  (~560 bytes, <1ms verification)
```

σ_s and σ_r are **not verified inside ZK** (avoids ~400K non-native constraints). They are bound into the fold hash chain and verified natively (Ed25519) by witnesses and third parties.

### 6. State Replication

```
Wallet doc (oracle) ──── k=50 XOR-closest nodes subscribe
                    ──── Owner writes; Iroh monotonicity enforces no rollback
                    ──── Witnesses read (never write)

Three propagation channels:
  1. Gossip broadcast       → fast (~1–5s), best-effort
  2. Doc sync push (k=50)   → reliable, after each witness event
  3. Pulse anti-entropy      → heals gaps every 5 minutes
```

Witnesses fail closed on unreachable docs: if wallet doc cannot be fetched, attestation is rejected.

### 7. Fee & Economic Model

```
Transaction: input = amount + change + (quorum × fee_share)
             fee_share has no output slot → "fee gap"

Sweep (deferred mint, unified circuit):
  Witness proves: pk ∈ witness_root of N SpendRecords
  → mints new issuer-bound slot into claimant's own wallet
  → sweep accumulator (durable Merkle root in wallet doc) prevents double-claim
  → NullifierStore provides fast-path race detection (ephemeral, GC-safe)

Optional donation:
  Sender opts in → 22nd witness_root entry = pool_pk (ghost witness)
  Guardian sweeps via same circuit into guardian's own wallet,
  then transfers to final destination via normal protocol
  Protocol has zero mandatory allocation to any fund
```

### 8. Security Properties (Summary)

| Threat | Defense |
|--------|---------|
| Double-spend | Oracle seq + Iroh-blob nullifier + terminal state check |
| **Equivocation / split-brain wallet doc** | Witnesses independently fetch from k=50 subscribers (not from requester); Iroh per-author monotonicity prevents owner from presenting different seq values to different shards; shard hopping means W₁ and W₂₃ fetch at different times from independent node sets |
| Fake-abort double-spend | Witnessed recovery (21/29 quorum × 2 phases) + independent state fetch |
| Sender griefing | Threshold encryption (can't reveal nullifier alone) |
| Receiver griefing | Bilateral signatures required for commitment |
| Witness Sybil | Issuer-gated eligibility + VRF threshold + fold circuit shard membership proof |
| Shard grinding | Shard hopping (seed depends on σ_r + beacon) + ZK shard prefix constraint |
| Eclipse | 50 doc subscribers + Pulse anti-entropy + owner-only writes |
| Key compromise | Registry rotation; old keys can't sign new eligible roots post-rotation |
| Drand outage | Stalls new transactions; cannot corrupt in-flight state; witnesses reject stale beacons |
| Double-sweep | Durable sweep accumulator (Merkle root in wallet doc, k=50 replicated) + NullifierStore fast-path; mirrors transfer double-spend architecture |

---

## L4 — Detailed Design Reference

### 1. Identifiers & Derivations (all hashes are BLAKE3)

```
identity_commit      = H(pk)
slot_commit          = H("inat_slot:"      ‖ sk ‖ seq)
nullifier            = H("inat_nullifier:" ‖ sk ‖ seq)
nullifier_commit     = H(nullifier)
forward_commit       = H("inat_slot:"      ‖ sk ‖ seq+1)
diversity_anchor     = H("inat_diversity_anchor:" ‖ seed)
asset_id             = H("inat_asset:" ‖ asset_genesis_cid ‖ name ‖ params)
```

Note: `forward_commit` uses the same `"inat_slot:"` domain as `slot_commit` — it is simply the slot commitment at `seq+1`. There is no separate forward domain.

Liveness: `protocol_seq == creation_seq → live; > creation_seq → dead (spent or recovered)`

### 2. Wallet Document Schema (Iroh doc, owner-writable only)

| Key | Value | Notes |
|-----|-------|-------|
| `identity:protocol_seq` | int | The oracle. Iroh monotonic — can only increase |
| `identity:pk_commit` | H(pk) | Wallet lookup key |
| `identity:sweep_root` | bytes(32) | Durable sweep accumulator Merkle root |
| `identity:sweep_count` | int | Total swept pairs |
| `identity:sweep_epoch` | int | Monotonic sweep batch counter |
| `slot:{commit}:status` | enum | READY / PENDING_SEND / PENDING_RECEIVE / COMMITTED / SPENT / FINALIZED / RECOVERED / BURNED |
| `slot:{commit}:data` | blob | Slot payload |
| `tx:{session}:phase` | enum | Active transaction context |
| `archive:{id}:spend_cid` | CID | Reference into Iroh blobs |

Terminal states (witnesses reject spend/recovery for these): `{SPENT, FINALIZED, RECOVERED, BURNED}`

### 3. Phase Seed Chain

Beacon values come from **Drand** (threshold BLS randomness beacon, publicly verifiable). Each phase requires a fresh beacon round within `MAX_BEACON_AGE_ROUNDS = 20` (~10 min). A Drand liveness failure stalls transaction initiation but cannot corrupt state — witnesses reject stale beacons and the sender simply waits or retries.

```
seed₁   = H(slot_commit ‖ beacon₁   ‖ randomness₁   ‖ asset_id)
seed₂₃  = H(seed₁       ‖ H(σ_r)    ‖ beacon₂₃      ‖ randomness₂₃)
seed₄   = H(seed₂₃      ‖ spend_cid ‖ beacon₄       ‖ randomness₄)
```

Each phase gossips to `derive_shard_topic(seedₙ, depth, asset_id)` — a different shard each time. `seed₂₃` depends on σ_r (unpredictable until Phase 1 completes), providing grinding resistance.

### 4. Witness Selection (fixed 29/21)

```
VRF_THRESHOLD = (TARGET_WITNESSES / SHARD_TARGET_SIZE) × 2²⁵⁶   [constant]

xor_dist        = max(uint128(xor(node_id, anchor) >> 128), 1)
adjusted_T      = VRF_THRESHOLD × min(xor_dist, MAX_DIVERSITY_BOOST)
selected        = uint256(vrf(sk, seed)) < adjusted_T

diversity_score = uint128(vrf_output[:16]) × xor_dist
ranked          = sort(winners, by=score DESC)[:29]          # capped set
fee_eligible    = first 21 valid attestations received       # witness_root
```

Fold circuit enforces (constraint 6a–6e):
- `vrf_verify(witness_pk, phase_seed, output, proof)` passes
- `uint256(output) < VRF_THRESHOLD`
- `prefix_bits(H(witness_pk), depth) == prefix_bits(H(phase_seed ‖ asset_id), depth)`
- `declared_depth` consistent across all three phases
- witness PKs unique within set

### 5. Two-Round Witness Engagement

**Round 1 — IntentEnvelope** (broadcast to ~1000 shard nodes):
`{ seed, session_id, asset_id, nullifier_commit, beacon_round, shard_depth }`
→ ~29 VRF winners reply with `WitnessEligibility { vrf_proof, eligibility_proof, reply_path }` where `eligibility_proof` carries the full signed `IssuerEligibleHeader` plus Merkle proof inline (no CID, no blob fetch)

**Round 2 — IntentPayload** (unicast to each selected witness):
`{ identity_commit, asset_id, π₁_or_π₂₃, public_inputs, poly_commits }`
→ Each witness independently fetches wallet doc(s), verifies, returns attestation

Note: Encrypted shares are delivered **after** attestation collection via separate `ShareDelivery` messages — they are not part of the IntentPayload.

### 6. ZK Circuits

**Phase 4 — DA confirmation.** W₄ (7 VRF-selected validators, fresh shard) independently verify that all Phase 1–3 data is retrievable from Iroh blobs. If W₄ validators are unavailable or dishonestly attest availability for missing data, the fold stalls — but state does not corrupt, because the fold circuit itself re-verifies the spend record content. A stalled fold triggers the forced-finalization path (72 hr) where a fresh W_FF quorum publishes the nullifier_commit (not the preimage — shares are unavailable). DA validators have no ability to steal funds; their only power is to delay finalization.

| Circuit | Prover | Key Proves |
|---------|--------|-----------|
| **π₁** (Phase 1) | Sender | Owns slot; nullifier correctly derived; VSS poly_commits valid; balance sufficient; forward commit |
| **π₂₃** (Phase 2) | Recipient | Owns recipient sk; amount commits match; lineage valid; `seed₂₃ = H(seed₁ ‖ H(σ_r) ‖ beacon)`; bilateral binding |
| **π_fold** | Any party (typically participant) | Previous fold valid; both phase proofs valid; 7 DA confirmations; seed chain; VRF + shard membership all phases; issuer registry chain; beacon recency gap ≤ MAX_FOLD_BEACON_GAP_ROUNDS |
| **π_recovery** | Owner | Owns sk; balance preserved; new_seq = old_seq + 1; ≥21 witness attestations valid (R₁ and R₂) |
| **π_sweep** | Witness / Guardian | pk ∈ witness_root of N SpendRecords; sum(shares) = claimed_amount; sweep accumulator non-membership proofs valid; new accumulator root correct; output bound to claimant's own wallet; protocol_seq binding; epoch monotonicity |
| **π_capacity** | Recipient (discovery) | ∃ slot with correct issuer and balance ≥ min; hides which slot |

**σ_s / σ_r** are verified natively (Ed25519) by witnesses and third parties — not inside ZK — then bound into the fold hash chain via `sig_binding = H(σ_s ‖ σ_r ‖ sender_pk ‖ recipient_pk)`.

### 7. Collapsed Proof Public Surface (~560 bytes total)

```
Identity:    owner_pk_commit, nullifier_commit
Slot:        current_slot_commit, current_seq
Lineage:     genesis_cid, asset_genesis_cid, issuer_registry_root,
             issuer_family_id, depth
Witnesses:   witness_set_root, beacon_round, confirmation_root
Eligibility: issuer_eligible_root, eligible_root_epoch, eligible_header_asset_id,
             eligible_header_issuer_pk, eligible_header_eligible_count,
             eligible_header_issued_at, eligible_header_signature
             (header fields bound into fold hash chain; signature verified
              natively at fold-verify time, not inside the circuit)
Signatures:  sigma_s, sigma_r, sender_pk, recipient_pk  (native verify)
Proof:       fold_proof (~400 bytes)
```

### 8. Data Discovery — Push-Then-Query Pattern

| Store | Write | Responsible Peers | Read |
|-------|-------|-------------------|------|
| Wallet docs (oracle) | Owner writes; Iroh doc sync | k=50 XOR-closest | Any subscriber |
| Nullifiers | Publisher pushes commit | k=20 XOR-closest | Witness queries |
| Content blobs | Publisher pushes blob | k=20 XOR-closest | Reader queries |
| Key registries | Issuer publishes + gossip on `inat/registry/v1/{asset_prefix}` | Gossip + blob store | Cache current + previous epoch |

Responsible peer set is deterministic: `get_closest_peers(key, k)` — no content-routing DHT needed.

### 9. Shard Lifecycle

```
TARGET_SHARD_SIZE     = 1000 nodes
SHARD_SPLIT_THRESHOLD = 2400  (sustained > 10 min → depth++)
SHARD_MERGE_THRESHOLD = 600   (sustained < 10 min → depth--)
DEPTH_CHANGE_COOLDOWN = 10 min
```

SHARD_MERGE_THRESHOLD is derived: `0.8 × SHARD_MIN_DENSITY = 0.8 × 750 = 600`. At this density, expected VRF self-selections = 17.4 — merge triggers before quorum becomes unreachable.

Depth adapts per asset independently. Witness verification tolerates ±1 bit lag during splits. Fold circuit constraint 6e enforces consistent declared_depth across all three phases of one transaction.

### 10. Recovery Protocol

**Allowed from:** PENDING_SEND or PENDING_RECEIVE (not post-COMMITTED).

```
Phase R₁ (at RECOVERY_TIMEOUT = 1 hour):
  Owner signs RecoveryRequest
  seed_R1 = H(slot_commit ‖ beacon_round ‖ beacon_randomness)
  W_R1 (21/29 fresh VRF, distinct from original W₁/W₂₃) independently:
    - Verify owner signature
    - Verify slot status ∈ {PENDING_SEND, PENDING_RECEIVE}
    - Verify nullifier not published
    - Verify oracle seq consistent
    - Verify timeout elapsed
  ≥21/29 APPROVED → R1_attestation_root = Merkle(R₁ attestation sigs)

Phase R₂ (fresh VRF, different shard):
  seed_R2 = H(seed_R1 ‖ H(R1_attestation_root) ‖ beacon_round_2 ‖ randomness_2)
  W_R2 (21/29) verify:
    - R₁ attestation signatures valid (≥21)
    - Investigation results consistent
    - Nullifier still unpublished
  ≥21/29 APPROVED →
    - recovery_seq = stuck_seq + 1
    - Oracle incremented
    - New slot created with identical balance
    - Old slot → RECOVERED (terminal)
```

**Safety under ambiguous W₂₃ state:** Recovery does NOT require verifying whether the original W₂₃ quorum was achieved. Both parties recover independently to new sequence numbers (stuck_seq + 1). If W₂₃ quorum WAS achieved and shares release later (publishing the orphaned nullifier), the nullifier maps to a RECOVERED (terminal) slot — no value can be spent from it.

**Anti-grinding:** R₁'s seed is grindable (owner controls slot_commit), but R₂'s seed depends on R₁_attestation_root (unpredictable). Different shards for R₁ and R₂.

**Forced finalization** (COMMITTED state, 72 hr elapsed, normal share collection failed): W_FF quorum (21/29, fresh VRF, disjoint from all prior sets) verifies COMMITTED state + timeout + nullifier absent. Publishes **nullifier_commit only** (not the preimage — shares are unavailable). The W₂₃ quorum + π₂₃ together constitute sufficient spend evidence. Proceeds to Phase 4.

### 11. Fee Model (Fixed 29/21)

```
Standard:  total_fee = 21 × fee_share
           fee_commit = Pedersen(21 × FEE_SHARE, blinding)
           balance:   input = amount + change + (21 × fee_share)

Donation:  total_fee = 22 × fee_share  (21 witnesses + 1 ghost pool_pk)
           fee_commit = Pedersen(22 × FEE_SHARE, blinding)
           balance:   input = amount + change + (22 × fee_share)
```

**Sweep (deferred mint):** Witnesses and the donation guardian claim accumulated fees via the unified π_sweep circuit. The sweep has two layers of double-claim prevention, mirroring the transfer double-spend architecture:

| Layer | Transfer | Sweep |
|-------|----------|-------|
| Fast-path (ephemeral) | NullifierStore | NullifierStore (sweep_nullifier) |
| Durable (authoritative) | Oracle seq kills input slot | sweep_root accumulator in wallet doc |
| Replication | Wallet doc k=50 | Same wallet doc k=50 |
| ZK binding | Fold circuit binds nullifier to seq | Sweep circuit binds leaf to sweep_root + seq |

**Sweep accumulator:** Each wallet doc stores a `sweep_root` — a sorted Merkle tree of all `H(claimant_pk ‖ spend_record_cid)` pairs ever swept. The sweep circuit proves non-membership of new claims against the previous root, then outputs the updated root incorporating those claims. The root is written atomically with `protocol_seq` to the wallet doc.

**Output wallet binding:** Sweep outputs MUST go to the claimant's own wallet: `new_slot_commit = H("inat_slot:" ‖ claimant_sk ‖ protocol_seq)`. This ensures the accumulator and the output live in the same document, preventing re-sweep attacks after NullifierStore GC.

**Donation flow:** Sender opts in → `pool_pk` added as 22nd entry in `witness_root`. Circuit proves `H(pool_pk) == HARDCODED_POOL_ID`. Guardian sweeps into guardian's own wallet (pool_pk doc), then transfers to final destination via normal protocol. The extra transfer is the cost of accumulator durability.

### 12. Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `WITNESS_TOTAL` | 29 | VRF-selected witnesses per phase |
| `WITNESS_QUORUM` | 21 | Required attestations for quorum |
| `REDUNDANCY_TARGET` | 50 | Doc subscribers per wallet |
| `NEIGHBORHOOD_SIZE` | 20 | Pulse XOR neighbors |
| `MAX_SUBSCRIBED_DOCS` | 5,000,000 | Per-node cap |
| `RECOVERY_TIMEOUT` | 1 hour | PENDING abort window |
| `FORCED_FINALIZATION_TIMEOUT` | 72 hours | COMMITTED fallback |
| `PULSE_INTERVAL` | 5 min | Anti-entropy period |
| `MAX_BEACON_AGE_ROUNDS` | 20 | ~10 min; per-phase freshness |
| `MAX_FOLD_BEACON_GAP_ROUNDS` | 2400 | ~20 hours; consecutive fold gap |
| `ELIGIBLE_ROOT_MAX_AGE_EPOCHS` | 2 | ~2 hours grace for in-flight tx |
| `REGISTRY_MAX_AGE_EPOCHS` | 3 | Max staleness in fold proof |
| `ASSET_MIN_SUPPORTERS` | 100,000 | Per-asset activation threshold |
| `SHARD_TARGET_SIZE` | 1000 | Target nodes per shard |
| `SHARD_MIN_DENSITY` | 750 | Derived: ceil(21 × 1000 / 29) + margin |
| `SHARD_MERGE_THRESHOLD` | 600 | 0.8 × SHARD_MIN_DENSITY |
| `SHARD_SPLIT_THRESHOLD` | 2400 | 1.2 × SHARD_MAX_DENSITY |
| `MAX_SWEEP_BATCH_SIZE` | 5000 | Max claims per sweep proof |

### 13. Size & Timing Budget

| Item | Size | Time |
|------|------|------|
| Collapsed fold proof | ~560 bytes | <1ms verify |
| Phase proof element (PLONK) | ~2.5 KB | 5ms verify |
| SpendRecord | ~5 KB | — |
| Witness bundle (21) | ~3.5 KB | — |
| Nullifier | 32 bytes | — |
| ZK proof generation (sender) | — | 2s target / 10s max |
| ZK proof generation (recipient) | — | 1s target / 5s max |
| Single attestation round-trip | — | 500ms target / 2s max |
| Full attestation collection | — | 5s target / 30s max |
| End-to-end transaction | — | 30s target / 120s max |
