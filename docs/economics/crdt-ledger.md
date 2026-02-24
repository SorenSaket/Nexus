---
sidebar_position: 3
title: CRDT Ledger
---

# CRDT Ledger

The global balance sheet in NEXUS is a CRDT-based distributed ledger. Not a blockchain. No consensus protocol. No mining. CRDTs (Conflict-free Replicated Data Types) provide automatic, deterministic convergence without coordination — exactly what a partition-tolerant network requires.

## Why Not a Blockchain?

Blockchains require global consensus: all nodes must agree on the order of transactions. This is fundamentally incompatible with NEXUS's partition tolerance requirement. When a village mesh is disconnected from the wider network for days or weeks, it must still process payments internally. CRDTs make this possible.

## Account State

```
AccountState {
    node_id: NodeID,
    total_earned: GCounter,     // grow-only, per-node entries, merge = pointwise max
    total_spent: GCounter,      // grow-only, same structure
    // Balance = earned - spent (derived, never stored)
    settlements: GSet<SettlementHash>,  // dedup set
}
```

### How GCounters Work

A GCounter (grow-only counter) is a CRDT that can only increase. Each node maintains its own entry, and merging takes the pointwise maximum:

- Node A says "Node X has earned 100" and Node B says "Node X has earned 150"
- Merge result: "Node X has earned 150" (the higher value wins)
- This works regardless of the order updates arrive

Balance is always derived: `balance = total_earned - total_spent`. It is never stored directly.

## Settlement Flow

```
1. Alice and Bob settle their payment channel (SettlementRecord signed by both)
2. SettlementRecord is gossiped to the network
3. Each receiving node:
   - Verifies both signatures
   - Checks hash not in known settlement set (dedup via GSet)
   - If valid and new: increment Alice's spent, Bob's earned
   - Add hash to GSet
   - Gossip forward
4. Convergence: O(log N) gossip rounds
```

### Gossip Bandwidth

With [stochastic relay rewards](payment-channels), settlements happen far less frequently than under per-packet payment — channel updates only trigger on lottery wins. This dramatically reduces the volume of settlement records the CRDT ledger must gossip.

- Baseline gossip: proportional to settlement frequency (~100-200 bytes per settlement)
- On constrained links (< 10 kbps): batching interval increases, reducing overhead further
- Fits within **Tier 2 (economic)** of the [gossip bandwidth budget](../protocol/network-protocol#bandwidth-budget)

## Double-Spend Prevention

Double-spend prevention is **probabilistic, not perfect**. Perfect prevention requires global consensus, which contradicts partition tolerance. NEXUS mitigates double-spending through multiple layers:

1. **Channel deposits**: Both parties must have visible balance to open a channel
2. **Credit limits**: Based on locally-known balance
3. **Reputation staking**: Long-lived nodes get higher credit limits
4. **Fraud detection**: Overdrafts are flagged network-wide; the offending node is blacklisted
5. **Economic disincentive**: For micropayments, blacklisting makes cheating unprofitable — the cost of losing your identity and accumulated reputation exceeds any single double-spend gain

## Partition Minting and Supply Convergence

When the network is partitioned, each partition independently runs the emission schedule and mints NXS proportional to local relay work. On merge, the GCounter merge (pointwise max per account) preserves individual balance correctness — no one loses earned NXS. However, **total minted supply across all partitions exceeds what a single-partition emission schedule would have produced.**

```
Example:
  Epoch 5 emission schedule: 1000 NXS total
  Partition A (60% of nodes): mints 1000 NXS to its relays
  Partition B (40% of nodes): mints 1000 NXS to its relays
  On merge: total minted in epoch 5 = 2000 NXS (not 1000)
```

This is an accepted tradeoff of partition tolerance — the alternative (coordinated minting) requires global consensus, which is incompatible with the design. The overminting is bounded:

1. **Proportional to partition count**: Two partitions produce at most 2x; three produce at most 3x. Prolonged fragmentation into many partitions is rare in practice.
2. **Detectable on merge**: When partitions heal, nodes can observe that multiple epoch proposals exist for the same epoch number. Post-merge epochs resume normal single-emission-schedule minting.
3. **Self-correcting over time**: The emission schedule decays geometrically. A one-time overmint during a partition is a fixed quantity that becomes negligible relative to total supply. The asymptotic ceiling is unchanged — it is just approached slightly faster.
4. **Offset by lost keys**: The estimated 1-2% annual key loss rate dwarfs partition minting overshoot in most scenarios.

The protocol does not attempt to "claw back" overminted supply. The cost of the mechanism (requiring consensus) exceeds the cost of the problem (minor temporary supply inflation during rare partitions).

## Epoch Compaction

The settlement GSet grows without bound. The Epoch Checkpoint Protocol solves this by periodically snapshotting the ledger state.

```
Epoch {
    epoch_number: u64,
    timestamp: u64,

    // Frozen account balances at this epoch
    account_snapshot: Map<NodeID, (total_earned, total_spent)>,

    // Bloom filter of ALL settlement hashes included
    included_settlements: BloomFilter,

    // Active set: defines the 67% threshold denominator
    active_set_hash: Blake3Hash,    // hash of sorted NodeIDs active in last 2 epochs
    active_set_size: u32,           // number of nodes in the active set

    // Acknowledgment tracking
    ack_count: u32,
    ack_threshold: u32,             // 67% of active_set_size
    status: enum { Proposed, Active, Finalized, Archived },
}
```

### Epoch Triggers

An epoch is triggered when **any** of these conditions is met:

| Trigger | Threshold | Purpose |
|---------|-----------|---------|
| **Settlement count** | ≥ 10,000 batches | Standard trigger for large meshes |
| **GSet memory** | ≥ 500 KB | Protects constrained devices (ESP32 has ~520 KB usable RAM) |
| **Small partition** | ≥ max(200, active_set_size × 10) settlements AND ≥ 1,000 gossip rounds since last epoch | Prevents stagnation in small partitions |

The small-partition trigger ensures a 20-node village doesn't wait months for an epoch. At 200 settlements (the minimum), the GSet is ~6.4 KB — well within ESP32 capacity. The 1,000 gossip round floor (roughly 17 hours at 60-second intervals) prevents epochs from firing too rapidly in tiny partitions with bursty activity.

### Epoch Proposer Selection

Eligibility requirements adapt to partition size:

1. The node has processed ≥ min(10,000, current epoch trigger threshold) settlement batches since the last epoch
2. The node has direct links to ≥ min(3, active_set_size / 2) active nodes
3. No other epoch proposal for this `epoch_number` has been seen

In a 20-node partition, a node needs only 3 direct links (not 10) and 200 processed settlements (not 10,000) to propose.

**Conflict resolution**: If multiple proposals for the same `epoch_number` arrive, nodes ACK the one with the **highest settlement count** (most complete state). Ties broken by lowest proposer `destination_hash`.

Epoch proposals are rate-limited to one per node per epoch period. Proposals that don't meet eligibility are silently ignored.

### Epoch Lifecycle

1. **Propose**: An eligible node proposes a new epoch with a snapshot of current state. The proposal includes an `active_set_hash` — a Blake3 hash of the sorted list of NodeIDs that have participated in settlement within the last 2 epochs, as observed by the proposer. This fixes the denominator for the 67% threshold.
2. **Acknowledge**: Nodes compare against their local state. If they've seen the same or more settlements, they ACK. If they have unseen settlements, they gossip those first. A node ACKs the proposal's `active_set_hash` — even if its own view differs slightly, it agrees to use the proposer's set as the threshold denominator for this epoch.
3. **Activate**: At 67% acknowledgment (of the active set defined in the proposal), the epoch becomes active. Nodes can discard individual settlement records and use only the bloom filter for dedup. If a significant fraction of nodes reject the active set (NAK), the proposer must re-propose with an updated set after further gossip convergence.
4. **Verification window**: During the grace period (4 epochs after activation), any node can submit a **settlement proof** — the full `SettlementRecord` — for any settlement it believes was missed. If the settlement is valid (signatures check) and NOT in the epoch's bloom filter, it is applied on top of the snapshot.
5. **Finalize**: After the grace period, previous epoch data is fully discarded. The bloom filter is the final word.

### Late Arrivals After Compaction

When a node reconnects after an epoch has been compacted, it checks its unprocessed settlements against the epoch's bloom filter:
- **Present in filter**: Already counted, discard
- **Absent from filter**: New settlement, apply on top of snapshot. If within the verification window, submit as a settlement proof.

### Bloom Filter Sizing

| Data | Size |
|------|------|
| 1M settlement hashes (raw) | ~32 MB |
| Bloom filter (0.01% false positive rate) | ~2.4 MB |
| Target epoch frequency | ~10,000 settlement batches |
| Per-node storage target | Under 5 MB |

The false positive rate is set to **0.01% (1 in 10,000)** rather than 1%, because false positives cause legitimate settlements to be silently treated as duplicates. At 0.01%, the expected loss is negligible (~1 settlement per 10,000), and the verification window provides a recovery mechanism for any that are caught.

**Critical retention rule**: Both parties to a settlement **must retain the full `SettlementRecord`** until the epoch's verification window closes (4 epochs after activation). If both parties discard the record after epoch activation (believing it was included) and a bloom filter false positive caused it to be missed, the settlement would be permanently lost. During the verification window, each party independently checks that its settlements are reflected in the snapshot; if any are missing, it submits a settlement proof. Only after the window closes may the full record be discarded.

### Snapshot Scaling

At 1M+ nodes, the flat `account_snapshot` is ~32 MB — too large for constrained devices. The solution is a **Merkle-tree snapshot** with sparse views.

**Full snapshot** (backbone/gateway nodes only): The account snapshot is stored as a sorted Merkle tree keyed by NodeID. Only nodes that participate in epoch consensus need the full tree. At 1M nodes and 32 bytes per entry, this is ~32 MB — feasible for nodes with SSDs.

**Sparse snapshot** (everyone else): Constrained devices store only:
- Their own balance
- Balances of direct channel partners
- Balances of trust graph neighbors (Ring 0-2)
- The Merkle root of the full snapshot

For a typical node with ~50 relevant accounts: 50 × 32 bytes = 1.6 KB.

**On-demand balance verification**: When a constrained node needs a balance it doesn't have locally (e.g., to extend credit to a new node), it requests a Merkle proof from any capable peer:

```
BalanceProof {
    node_id: NodeID,
    total_earned: u64,
    total_spent: u64,
    merkle_siblings: Vec<Blake3Hash>,  // path from leaf to root
    epoch_number: u64,
}
// Size: ~640 bytes for 1M nodes (20 tree levels × 32-byte hashes)
```

The constrained node verifies the proof against the Merkle root it already has. This proves the balance is in the snapshot without storing the full 32 MB.

### Constrained Node Epoch Summary

LoRa nodes and other constrained devices don't participate in epoch consensus. They receive a compact summary from their nearest capable peer:

```
EpochSummary {
    epoch_number: u64,
    merkle_root: Blake3Hash,               // root of full account snapshot
    my_balance: (u64, u64),                // (total_earned, total_spent)
    partner_balances: Vec<(NodeID, u64, u64)>, // channel partners + trust neighbors
    bloom_segment: BloomFilter,            // relevant portion of settlement bloom
}
```

Typical size: under 5 KB for a node with 20-30 channel partners.
