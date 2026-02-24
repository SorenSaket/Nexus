---
sidebar_position: 2
title: Design Decisions
---

# Design Decisions Log

This page documents the key architectural decisions made during NEXUS protocol design, including alternatives considered and the rationale for each choice.

## Network Stack: Reticulum as Initial Transport

| | |
|---|---|
| **Chosen** | Use [Reticulum Network Stack](https://reticulum.network/) as the initial transport implementation; treat it as a [swappable layer](#transport-layer-swappable-implementation) |
| **Alternatives** | Clean-room implementation from day one, libp2p, custom protocol |
| **Rationale** | Reticulum already solves transport abstraction, cryptographic identity, mandatory encryption, sender anonymity (no source address), and announce-based routing — all proven on LoRa at 5 bps. NEXUS extends it with CompactPathCost annotations and economic primitives rather than reinventing a tested foundation. The transport layer is an implementation detail: NEXUS defines the interface it needs and can switch to a clean-room implementation in the future without affecting any layer above. |

## Routing: Kleinberg Small-World with Cost Weighting

| | |
|---|---|
| **Chosen** | Greedy forwarding on a Kleinberg small-world graph with cost-weighted scoring |
| **Alternatives** | Pure Reticulum announce model, Kademlia, BGP-style routing, Freenet-style location swapping |
| **Rationale** | The physical mesh naturally forms a small-world graph: short-range radio links serve as lattice edges, backbone/gateway links serve as Kleinberg long-range contacts. Greedy forwarding achieves O(log² N) expected hops — a formal scalability guarantee. Cost weighting trades path length for economic efficiency. Unlike Freenet, no location swapping is needed because destination hashes are self-assigned and Reticulum announcements build the navigable topology. |

## Payment: Stochastic Relay Rewards

| | |
|---|---|
| **Chosen** | Probabilistic micropayments via VRF-based lottery (channel update only on wins) |
| **Alternatives** | Per-packet accounting, per-minute batched accounting, subscription-based, random-nonce lottery |
| **Rationale** | Per-packet and batched payment require frequent channel state updates, consuming ~2-4% bandwidth on LoRa links. Stochastic rewards achieve the same expected income but trigger updates only on lottery wins — reducing economic overhead by ~10x. Adaptive difficulty ensures fairness across traffic levels. The law of large numbers guarantees convergence for active relays. **The lottery uses a VRF (ECVRF-ED25519-SHA512-TAI, RFC 9381)** rather than a random nonce to prevent relay nodes from grinding nonces to win every packet. The VRF produces exactly one verifiable output per (relay key, packet) pair, reusing the existing Ed25519 keypair. |

## Settlement: CRDT Ledger

| | |
|---|---|
| **Chosen** | CRDT ledger (GCounters + GSet) |
| **Alternatives** | Blockchain, federated sidechain |
| **Rationale** | Partition tolerance is non-negotiable. CRDTs converge without consensus. A blockchain requires global ordering, which is impossible when network partitions are expected operating conditions. **Tradeoff**: double-spend prevention is probabilistic, not perfect. Mitigated by channel deposits, credit limits, reputation staking, and blacklisting — making cheating economically irrational for micropayments. |

## Communities: Emergent Trust Neighborhoods

| | |
|---|---|
| **Chosen** | Trust graph with emergent neighborhoods (no explicit zones) |
| **Alternatives** | Explicit zones with admin keys and admission policies (v0.8 design) |
| **Rationale** | Explicit zones require someone to create and manage them — centralized thinking in decentralized clothing. They impose UX burden and artificially fragment communities. Trust neighborhoods emerge naturally from who you trust: free communication between trusted peers, paid between strangers. No admin, no governance, no admission policies. Communities form the same way they form in real life — through relationships, not administrative acts. The trust graph provides Sybil resistance economically (vouching peers absorb debts). |

## Compaction: Epoch Checkpoints with Bloom Filters

| | |
|---|---|
| **Chosen** | Epoch checkpoints with bloom filters |
| **Alternatives** | Per-settlement garbage collection, TTL-based expiry |
| **Rationale** | The settlement GSet grows without bound. Bloom filters at 0.01% FPR compress 1M settlement hashes from ~32 MB to ~2.4 MB. A settlement verification window during the grace period recovers any settlements lost to false positives. Epochs are triggered by settlement count (~10,000 batches), not wall-clock time, for partition tolerance. |

## Compute Contracts: NXS-Byte

| | |
|---|---|
| **Chosen** | NXS-Byte (minimal bytecode, ~50 KB interpreter) |
| **Alternatives** | Full WASM everywhere |
| **Rationale** | ESP32 microcontrollers can't run a WASM runtime. NXS-Byte provides basic contract execution on even the most constrained devices. WASM is offered as an optional capability on nodes with sufficient resources. |

## Encryption: Ed25519 + X25519

| | |
|---|---|
| **Chosen** | Ed25519 for identity/signing, X25519 for key exchange (Reticulum-compatible) |
| **Alternatives** | RSA, symmetric-only |
| **Rationale** | Ed25519 has 32-byte public keys (compact for radio), fast signing/verification, and is widely proven. X25519 provides efficient Diffie-Hellman key exchange. Compatible with Reticulum's crypto model. RSA keys are too large for constrained links. |

## Source Privacy: No Source Address

| | |
|---|---|
| **Chosen** | No source address in packet headers (inherited from Reticulum) |
| **Alternatives** | Onion routing |
| **Rationale** | Onion routing adds significant overhead for radio links (multiple encryption layers, circuit establishment). Omitting the source address is free and effective against casual observation. Full traffic analysis resistance via onion routing is deferred to future work. |

## Naming: Neighborhood-Scoped, No Global Namespace

| | |
|---|---|
| **Chosen** | Community-label-scoped names (e.g., `alice@portland-mesh`) |
| **Alternatives** | Global names via consensus |
| **Rationale** | Global consensus contradicts partition tolerance. Community labels are self-assigned and informational — no authority, no uniqueness enforcement. Multiple disjoint clusters can share a label; resolution is proximity-based. Local petnames provide a fallback. |

## Cost Annotations: Compact Path Cost (No Per-Relay Signatures)

| | |
|---|---|
| **Chosen** | 6-byte constant-size `CompactPathCost` (running totals updated by each relay, no per-relay signatures) |
| **Alternatives** | Per-relay signed CostAnnotation (~84 bytes per hop), aggregate signatures, signature-free with Merkle proof |
| **Rationale** | Per-relay signatures make announces grow linearly with path length — 84 bytes × N hops. On a 1 kbps LoRa link with 3% routing budget, this limits convergence to ~1 announce per 22+ seconds. CompactPathCost uses 6 bytes total regardless of path length: log-encoded cumulative cost, worst-case latency, bottleneck bandwidth, and hop count. Per-relay signatures are unnecessary because routing decisions are local (you trust your link-authenticated neighbor), trust is transitive at each hop, and the market enforces honesty (overpriced relays get routed around, underpriced relays lose money). The announce itself remains signed by the destination node. |

## Congestion Control: Three-Layer Design

| | |
|---|---|
| **Chosen** | Link-level CSMA/CA + per-neighbor token bucket + 4-level priority queuing + economic cost response |
| **Alternatives** | Pure CSMA/CA only, rigid TDMA, end-to-end TCP-style congestion control |
| **Rationale** | A single mechanism is insufficient across the bandwidth range (500 bps to 10 Gbps). CSMA/CA handles collision avoidance on half-duplex radio. Token buckets enforce fair sharing across neighbors. Priority queuing ensures real-time traffic (voice) isn't starved by bulk transfers. Economic cost response (quadratic cost increase under congestion) signals scarcity through the existing cost routing mechanism, causing natural traffic rerouting without new protocol extensions. End-to-end congestion control (TCP-style) is wrong for a mesh — the bottleneck is typically a single constrained link, and hop-by-hop control responds faster. Rigid TDMA wastes bandwidth when some slots are unused. |

## Transport Layer: Swappable Implementation

| | |
|---|---|
| **Chosen** | Define transport as an interface with requirements; use Reticulum as the current implementation |
| **Alternatives** | Hard dependency on Reticulum, clean-room from day one |
| **Rationale** | Reticulum provides a proven transport layer tested at 5 bps on LoRa, saving significant implementation effort. But coupling NEXUS to Reticulum's codebase or community roadmap creates fragility. Instead, NEXUS defines the transport interface it needs (transport-agnostic links, announce-based routing, mandatory encryption, sender anonymity) and uses Reticulum as the current implementation. NEXUS extensions are carried as opaque payload in the announce DATA field — a clean separation. Three participation levels (L0 transport-only, L1 NEXUS relay, L2 full NEXUS) ensure interoperability with the underlying transport. This allows a future clean-room implementation without affecting any layer above transport. |

## Storage: Pay-Per-Duration with Erasure Coding

| | |
|---|---|
| **Chosen** | Bilateral storage agreements, pay-per-duration, Reed-Solomon erasure coding, lightweight challenge-response proofs |
| **Alternatives** | Filecoin-style PoRep/PoSt with on-chain proofs; Arweave-style one-time-payment permanent storage; simple full replication |
| **Rationale** | Filecoin's Proof of Replication requires GPU-level computation (minutes to seal a sector) — impossible on ESP32 or Raspberry Pi. Arweave's permanent storage requires a blockchain endowment model and assumes perpetually declining storage costs. Both require global consensus. NEXUS uses simple Blake3 challenge-response proofs (verifiable in under 10ms on ESP32) and bilateral agreements settled via payment channels. Erasure coding (Reed-Solomon) provides the same durability as 3x replication at 1.5x storage overhead. The tradeoff: we can't prove a node stores data *uniquely* (no PoRep), but we can prove it stores data *at all* — and the data owner doesn't care how the node organizes its disk. |

## Mobile Handoff: Presence Beacons + Credit-Based Fast Start

| | |
|---|---|
| **Chosen** | Transport-agnostic presence beacons, credit-based fast start, roaming cache with channel preservation |
| **Alternatives** | Pre-negotiated handoff (cellular-style), pure re-discovery from scratch, always-connected overlay |
| **Rationale** | Cellular handoff requires a central controller — incompatible with decentralized mesh. Pure re-discovery works but is slow for latency-sensitive sessions. Presence beacons (20 bytes, broadcast every 10 seconds on any interface) let mobile nodes passively discover local relays before connecting. Credit-based fast start allows immediate relay if the mobile node has visible CRDT balance, while the payment channel opens in the background. The roaming cache preserves channels for previously visited areas, enabling zero-latency reconnection on return. No explicit teardown needed — old agreements expire via `valid_until`. |

## Light Client Trust: Content Verification + Multi-Source Queries

| | |
|---|---|
| **Chosen** | Three-tier verification: content-hash check (Tier 1), owner-signature check (Tier 2), multi-source queries (Tier 3) |
| **Alternatives** | Merkle proofs over DHT state, SPV-style state root verification, full DHT replication on clients |
| **Rationale** | Merkle proofs require a global state root, which contradicts partition tolerance (no consensus). SPV-style verification has the same problem. Full DHT replication is too bandwidth-heavy for phones. Instead, content addressing (Tier 1) gives zero-overhead verification for the most common case — `Blake3(data) == key` proves authenticity regardless of relay honesty. Signed objects (Tier 2) prevent forgery of mutable data. Multi-source queries (Tier 3, N=2-3 independent nodes) detect censorship and staleness. Trusted relays skip to single-source for all tiers. Overhead is minimal: at most one extra 192-byte query for critical lookups via untrusted relays. |

## Epoch Triggers: Adaptive Multi-Criteria

| | |
|---|---|
| **Chosen** | Three-trigger system: settlement count ≥ 10,000 (large mesh), GSet size ≥ 500 KB (memory pressure), or partition-proportional threshold with gossip-round floor (small partitions) |
| **Alternatives** | Fixed 10,000-settlement-only trigger, wall-clock timer, per-node independent compaction |
| **Rationale** | A 20-node village on LoRa with low traffic might take months to reach 10,000 settlements. ESP32 nodes (520 KB usable RAM) would exhaust memory first. The adaptive trigger fires at max(200, active_set × 10) settlements with a 1,000-gossip-round floor (~17 hours), keeping GSet under ~6.4 KB for small partitions. The 500 KB GSet size trigger is a safety net regardless of partition size. Proposer eligibility adapts too — only 3 direct links needed (not 10) in small partitions. Wall-clock timers were rejected because they require clock synchronization. Per-node compaction was rejected because it fragments global state. |

## Ledger Scaling: Merkle-Tree Snapshots with Sparse Views

| | |
|---|---|
| **Chosen** | Merkle-tree account snapshot; full tree on backbone nodes, sparse view + Merkle root on constrained devices, on-demand balance proofs (~640 bytes) |
| **Alternatives** | Flat snapshot everywhere, sharded epochs, neighborhood-only ledgers |
| **Rationale** | At 1M nodes the flat snapshot is ~32 MB — unworkable on ESP32 or phones. Sharded epochs fragment the ledger and complicate cross-shard verification. Neighborhood-only ledgers lose global balance consistency. A Merkle tree over the sorted account snapshot gives the best of both: backbone nodes store the full tree (32 MB, feasible on SSD), constrained devices store only their own balance + channel partners + trust neighbors (~1.6 KB for 50 accounts) plus the Merkle root. Any balance can be verified on demand with a ~640-byte proof (20 tree levels × 32 bytes). No global state transfer needed. |

## Transforms/Inference: Compute Capability, Not Protocol Primitive

| | |
|---|---|
| **Chosen** | STT/TTS/inference as compute capabilities in the marketplace |
| **Alternatives** | Dedicated transform layer (considered in v0.5 draft) |
| **Rationale** | Speech-to-text, translation, and other transforms are just compute. Making them protocol primitives over-engineers the foundation. The capability marketplace already handles discovery, negotiation, execution, verification, and payment for arbitrary compute functions. |
