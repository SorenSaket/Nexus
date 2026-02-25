---
sidebar_position: 1
title: Roadmap
---

# Implementation Roadmap

The Mehr implementation follows a **product-first** strategy: get a usable app into people's hands as fast as possible, then layer on capabilities driven by real usage. The protocol spec is comprehensive because it needs to be — but implementation is ruthlessly phased. Each phase delivers something people can use, not just something that passes tests.

```
                    Implementation Strategy

  Phase 1          Phase 2           Phase 3           Phase 4
  ─────────        ─────────         ─────────         ─────────
  PHONE APP        RANGE +           ECONOMICS +       FULL
  (MVP)            SOCIAL            TOKEN              ECOSYSTEM

  WiFi/BLE mesh    LoRa relay nodes  Payment channels   Advanced compute
  Messaging        Social feeds      VRF lottery        Licensing
  Trust graph      Content           CRDT ledger        Onion routing
  Free tier only   MHR-Store/DHT     Gateway operators  Protocol bridges

  Users: early     Users: growing    Users: economy     Users: mature
  adopters         communities       bootstraps         ecosystem
```

**Principle**: The free tier (trusted peer communication) is a complete product on its own. MHR tokens, economics, and advanced features come only after there are real users generating real traffic. Token follows utility, never leads it.

---

## Phase 1: Mesh Messenger (MVP)

**Focus**: A phone app that lets people message and call each other over a local mesh — no internet, no tokens, no hardware purchases. The free tier only.

**Why phone first**: 4+ billion smartphones already deployed. WiFi Direct gives 50-200m range, BLE gives 10-100m. Zero additional hardware cost. App stores handle distribution. This is the fastest path to real users.

### Milestone 1.1: Core Protocol Library (Rust)

- `NodeIdentity` (Ed25519 keypair generation, destination hash derivation, X25519 conversion)
- Link-layer encryption (X25519 ECDH + ChaCha20-Poly1305, counter-based nonces, key rotation)
- Packet framing (Reticulum-compatible: header, addresses, context, data)
- Interface abstraction (WiFi Direct, BLE — phone-native transports first)
- Announce generation and forwarding with Ed25519 signature verification
- Target `no_std` compatibility from day one (same code on phones and ESP32 later)

**Acceptance**: Two phones can establish an encrypted link over WiFi Direct, exchange announces, and forward packets. Unauthenticated nodes are rejected.

### Milestone 1.2: Routing + Gossip

- `CompactPathCost` (6-byte encoding/decoding, log-scale math, relay update logic)
- Routing table (`RoutingEntry` with cost, latency, bandwidth, hop count, reliability)
- Greedy forwarding with `PathPolicy` scoring (Cheapest, Fastest, Balanced)
- Gossip protocol (60-second rounds, bloom filter state summaries, delta exchange)
- Bandwidth budget enforcement (4-tier allocation, constrained-link adaptation)
- Announce propagation rules (event-driven + 30-min refresh, hop limit, expiry, link failure detection)

**Acceptance**: A 5-phone mesh converges routing tables within 3 gossip rounds. Packets are forwarded via cost-optimal paths. Removing a phone causes re-routing within 3 minutes. Gossip overhead stays within 10% budget.

### Milestone 1.3: Trust Graph + Free Relay

- `TrustConfig` implementation (trusted peers, cost overrides, scopes)
- Free relay logic (sender trusted AND destination trusted → no lottery, no channels)
- Adding/removing trusted peers (the only social action)

**Acceptance**: Trusted peers relay traffic for free with zero economic overhead. The full messaging stack works with zero tokens in circulation.

### Milestone 1.4: Messaging

- E2E encrypted messaging (store-and-forward, offline delivery)
- Group messaging with co-admin delegation
- Congestion control (CSMA/CA, priority queuing, backpressure)

**Acceptance**: Two phones exchange encrypted messages via multi-hop mesh. Messages to offline nodes are stored and delivered when the recipient reconnects. Group messaging works with 3+ participants.

### Milestone 1.5: Phone App

- Android app (Kotlin/Rust FFI) — Android first for broader device support and sideloading
- iOS app (Swift/Rust FFI) — follows Android
- Contact management (add trusted peers via QR code, NFC, or manual key entry)
- Messaging UI (conversations, groups, media sending adapted to link quality)
- Voice on WiFi links (Opus codec)
- Background mesh relay (phone relays traffic while in pocket)
- Multi-transport handoff (WiFi Direct ↔ BLE, seamless)

**Acceptance**: A non-technical user can install the app, add a friend via QR code, and exchange messages — all without internet. Voice calls work on WiFi links. The app relays traffic for the mesh in the background.

### Phase 1 Deliverable

**A mesh messenger app for phones.** Install, add friends, message and call — no internet, no tokens, no special hardware. This is the product people use and tell others about.

**Target audiences**: festivals/events, campuses, communities with unreliable internet, privacy-conscious groups, disaster preparedness, rural areas.

---

## Phase 2: Range + Social

**Focus**: Extended range via cheap LoRa hardware, social features as a growth driver, and the service primitives that support them.

### Milestone 2.1: LoRa Transport

- LoRa interface implementation (SX1276/SX1262 via `no_std` Rust)
- Off-the-shelf hardware support:
  - Heltec WiFi LoRa 32 (~$15)
  - LILYGO T-Beam (~$25, with GPS)
  - RAK WisBlock (~$30, modular)
  - RNode (Reticulum-native)
- LTE-M and NB-IoT interface support (carrier-managed LPWAN)
- Multi-interface bridging (phone WiFi ↔ LoRa relay ↔ WiFi gateway)
- Solar relay firmware (ESP32 L1: transport, routing, gossip — runs on $30 solar kit)

**Acceptance**: A LoRa relay extends the phone mesh to 5-15 km range. A phone sends a message that hops: phone → WiFi → LoRa relay → WiFi → destination phone. Solar relay runs unattended for 30+ days.

### Milestone 2.2: MHR-Store

- `DataObject` types (Immutable, Mutable, Ephemeral)
- Storage agreements (bilateral, pay-per-duration)
- Proof of storage (Blake3 Merkle challenge-response)
- Erasure coding (Reed-Solomon, default schemes by size)
- Repair protocol (detect failure → assess → reconstruct → re-store)
- Garbage collection (7-tier priority)
- Kickback fields (revenue sharing between storage node and content author)

**Acceptance**: A node stores a DataObject with replication factor 3 across the mesh. Proof-of-storage challenges pass. Erasure coding reconstructs from k of n chunks. Kickback flows correctly on retrieval.

### Milestone 2.3: MHR-DHT + MHR-Pub

- DHT routing (XOR distance + cost weighting, α=0.7)
- k=3 replication with cost-bounded storage set
- Lookup and publication protocols
- Subscription types (Key, Prefix, Node, Scope)
- Delivery modes (Push, Digest, PullHint)
- Scope-based subscriptions (geographic + interest feeds)
- Bandwidth-adaptive mode selection

**Acceptance**: A node publishes a DataObject and it's discoverable via DHT lookup from any node in the mesh. MHR-Pub delivers notifications to scope subscribers within 2 gossip rounds.

### Milestone 2.4: Social Layer

- `PostEnvelope` (free layer) + `SocialPost` (paid layer) — mutable DataObjects
- `UserProfile` (display name, bio, avatar, scopes, claims)
- Hierarchical scopes (Geo + Topic) with scope matching
- Five feed types: follow, geographic, interest, intersection, curated
- `CuratedFeed` with curator kickback
- Publishing flow (post_id generation, envelope propagation)
- Editing (mutable DataObject semantics, sequence versioning)
- Replies, boosts, references
- Media tiering (blurhash thumbnails on LoRa, full media on WiFi)
- MHR-Name (scope-based naming, conflict resolution, petnames)

**Acceptance**: A user publishes a post tagged with geographic and interest scopes. The post's envelope appears in subscribers' feeds. Readers pay to fetch full content. Kickback flows to author. Curated feeds work end-to-end.

### Milestone 2.5: Test Networks

- Deploy 3-5 physical test networks (urban, rural, campus, event)
- Each network: 10-50 nodes (phones + LoRa relays) across at least 2 transports
- Instrument for: routing convergence, gossip bandwidth, storage reliability, social UX
- Run for at least 4 weeks per network
- Document: failure modes, parameter tuning, real-world performance

**Acceptance**: Test networks operate continuously for 4 weeks. Users report messaging and social features work reliably. Published test report with metrics.

### Phase 2 Deliverable

**A mesh social network with extended range.** Phones for messaging + social, $30 LoRa relays for range. Users can follow people, browse geographic and interest feeds, curate content, and host profiles — all without internet. Content propagates based on demand economics.

---

## Phase 3: Economic Layer + Token

**Focus**: MHR token genesis, payment infrastructure, and the transition from free-only to a functioning mesh economy. Only now — with real users and real traffic — do economics matter.

### Milestone 3.1: Payment Channels

- VRF lottery implementation (ECVRF-ED25519-SHA512-TAI per RFC 9381)
- Adaptive difficulty (local per-link, formula: `win_prob = target_updates / observed_packets`)
- `ChannelState` (200 bytes, dual-signed, sequence-numbered)
- Channel lifecycle (open, update on win, settle, dispute with 2,880-round window, abandon after 4 epochs)
- `SettlementRecord` generation and dual-signature

**Acceptance**: Two nodes relay 1,000 packets. The relay wins the VRF lottery approximately `1000 × win_probability` times (within 2σ). Channel updates occur only on wins. Settlement produces a valid dual-signed record. Dispute resolution correctly rejects old states.

### Milestone 3.2: CRDT Ledger + Epoch Compaction

- `AccountState` (GCounter for earned/spent, GSet for settlements)
- GCounter merge (pointwise max per-node entries)
- GCounter rebase at epoch compaction (prevents overflow from money velocity)
- Settlement flow (validation: 2 sig checks + Blake3 hash + GSet dedup, gossip forward)
- Balance derivation (`earned - spent`, reject negative)
- Epoch trigger logic (3-trigger: settlement count, GSet size, small-partition adaptive)
- Epoch lifecycle (Propose → Acknowledge at 67% → Activate → Verify → Finalize)
- Merkle-tree snapshot (full on backbone, sparse on constrained)
- `BalanceProof` generation and verification

**Acceptance**: A 20-node network triggers epochs correctly. Balances converge across the mesh. GCounter rebase keeps counters bounded. ESP32 nodes operate with sparse snapshots under 5 KB.

### Milestone 3.3: Token Genesis + Proof-of-Service Mining

- Emission schedule implementation (10^12 μMHR/epoch, halving every 100,000 epochs, shift clamp at 63)
- Tail emission floor (0.1% of circulating supply annually)
- `RelayWinSummary` aggregation per epoch
- Mint distribution proportional to verified VRF lottery wins
- Channel-funded relay payments (coexist with minting)

**Acceptance**: The first epoch mints MHR and distributes it to relay nodes. Distribution is proportional to wins. Minting and channel payments coexist. Token supply follows the emission schedule.

### Milestone 3.4: Reputation + Credit

- `PeerReputation` scoring (relay, storage, compute scores 0-10000)
- Score update formulas (success: diminishing gains, failure: 10% penalty)
- Trust-weighted referrals (1-hop, capped at 50%)
- Transitive credit (direct: full, friend-of-friend: 10%, 3+ hops: none)
- `CreditState` tracking per grantee
- Credit rate limiting per trust distance and per epoch

**Acceptance**: Reliable nodes build reputation. Credit extends through trust graph. A friend-of-friend gets exactly 10% of the direct credit line. Default handling absorbs debt correctly.

### Milestone 3.5: Content Economics + Propagation

- Content propagation through scope hierarchy (neighborhood → city → region)
- Demand-driven promotion (retrieval count thresholds)
- Self-funding content detection (kickback exceeds storage cost)
- Interest relay nodes (bridge topic communities across geography)
- Local-first interest propagation (local validation before global spread)
- Content governance (individual filtering, community standards, trust revocation)

**Acceptance**: Popular content auto-promotes from neighborhood to city scope. Self-funding content persists without author payment. Interest content propagates through relay nodes after local validation.

### Milestone 3.6: Gateway Operators

- Gateway trust-based onboarding (add consumer to trusted_peers, extend credit)
- Fiat billing integration (subscription, prepaid, pay-as-you-go — gateway's choice)
- Cloud storage via gateway (consumer stores files, gateway handles MHR)
- Gateway-provided connectivity (ethernet ports, WiFi access points)

**Acceptance**: A consumer signs up with a gateway, pays fiat, and uses the network without seeing MHR. Traffic flows through gateway trust. Consumer can switch gateways without losing identity.

### Phase 3 Deliverable

**A functioning mesh economy.** MHR tokens enter circulation through proof-of-service. Relay operators earn by forwarding traffic. Content creators earn through kickback. Gateway operators bridge fiat consumers to the mesh economy. The economic layer is live, validated on real networks.

---

## Phase 4: Full Ecosystem

**Focus**: Advanced capabilities, application richness, and ecosystem growth.

### Milestone 4.1: Identity + Governance

- Identity claims and vouches (GeoPresence, CommunityMember, KeyRotation, Capability, ExternalIdentity)
- RadioRangeProof (geographic verification via LoRa beacons)
- Peer attestation and transitive confidence
- Vouch lifecycle (create, gossip, verify, renew, revoke)
- Voting prerequisites (geographic eligibility from verified claims)

### Milestone 4.2: MHR-Compute (MHR-Byte)

- 47-opcode interpreter implementation in Rust
- Cycle cost enforcement (ESP32-calibrated)
- Resource limit enforcement (max_memory, max_cycles, max_state_size)
- Compute delegation via capability marketplace
- Reference test vector suite for cross-platform conformance
- WASM execution environment for gateway/backbone nodes

### Milestone 4.3: Rich Applications

- Voice (Codec2 on LoRa, Opus on WiFi, bandwidth bridging, seamless handoff)
- Digital licensing (LicenseOffer, LicenseGrant, verification chain, off-network verifiability)
- Cloud storage (client-side encryption, erasure coding, sync between devices, file sharing)
- Forums (append-only logs, moderation contracts)
- Marketplace (listings, escrow contracts)

### Milestone 4.4: Interoperability + Privacy

- Third-party protocol bridges (SSB, Matrix, Briar) — [standalone gateway services](design-decisions#protocol-bridges-standalone-gateway-services) with identity attestation
- Onion routing implementation (`PathPolicy.ONION_ROUTE`, per-packet layered encryption)
- Private compute tiers (Split Inference, Secret Sharing, TEE)

### Milestone 4.5: Ecosystem Growth

- Developer SDK and documentation
- Community-driven capability development
- Hardware partnerships and reference design refinement (informed by real deployment data)
- Custom hardware (only if demand justifies — let usage data guide form factors)

### Phase 4 Deliverable

A full-featured decentralized platform with rich applications, privacy-enhanced routing, identity governance, and interoperability with existing protocols.

---

## Implementation Language

The primary implementation language is **Rust**, chosen for:

- Memory safety without garbage collection (critical for embedded targets)
- `no_std` support for ESP32 firmware
- Strong ecosystem for cryptography and networking
- Single codebase from microcontroller to server
- FFI to Kotlin (Android) and Swift (iOS) for phone apps

## Platform Targets

| Platform | Implementation | Phase |
|----------|---------------|-------|
| Android phone | Rust core + Kotlin UI via FFI | Phase 1 |
| iOS phone | Rust core + Swift UI via FFI | Phase 1 |
| Raspberry Pi / Linux | Rust native (bridge, gateway, storage) | Phase 1-2 |
| ESP32 + LoRa | Rust `no_std` (L1 relay) | Phase 2 |
| Desktop (Linux, macOS, Windows) | Rust native (full node) | Phase 2 |

All implementations speak the same wire protocol and interoperate on the same network.

## Test Network Strategy

Real physical test networks, not simulations:

- Simulation cannot capture the realities of radio propagation, WiFi interference, and real-world device failure modes
- Each test network should represent a different deployment scenario (campus, urban, rural, event)
- Test networks validate both the protocol and the user experience
- Phase 1 test networks are phone-only (WiFi/BLE mesh)
- Phase 2 test networks add LoRa relays
- Phase 3 test networks enable economics and measure token dynamics

## Why This Order

The old roadmap (protocol-first, hardware-first) would produce a working mesh with no users. This roadmap produces users first, then gives them progressively more capabilities:

| Phase | What Users Get | What the Network Gets |
|-------|---------------|---------------------|
| 1 | Encrypted mesh messaging on phones | Real users, real trust graphs |
| 2 | Social feeds, extended range, content | Real content, real traffic patterns |
| 3 | Token economy, earning/spending, gateways | Real economic data, real market pricing |
| 4 | Rich apps, privacy, interoperability | Mature ecosystem |

Each phase is viable on its own. Phase 1 is a useful product even if Phase 2 never ships. Phase 2 is a useful product even if economics never launches. No phase depends on token speculation or hardware manufacturing for its value.

## Implementability Assessment

Phase 1 is **fully implementable** with the current specification. All protocol-level gaps have been resolved:

| Component | Spec Status | Key References |
|-----------|------------|----------------|
| Identity + Encryption | Complete | [Security](../protocol/security) |
| Packet format + CompactPathCost | Complete (wire format specified) | [Network Protocol](../protocol/network-protocol#mehr-extension-compact-path-cost) |
| Routing + Announce propagation | Complete (scoring, announce rules, expiry, failure detection) | [Network Protocol](../protocol/network-protocol#routing) |
| Gossip protocol | Complete (bloom filter, bandwidth budget, 4-tier) | [Network Protocol](../protocol/network-protocol#gossip-protocol) |
| Congestion control | Complete (3-layer, priority levels, backpressure) | [Network Protocol](../protocol/network-protocol#congestion-control) |
| Trust neighborhoods | Complete (free relay, credit, scopes) | [Trust & Neighborhoods](../economics/trust-neighborhoods) |
| Messaging | Complete (E2E, store-and-forward, groups) | [Messaging](../applications/messaging) |
| Social layer | Complete (envelope/post, feeds, curation, editing) | [Social](../applications/social) |
| VRF lottery + Payment channels | Complete (RFC 9381, difficulty formula, channel lifecycle) | [Payment Channels](../economics/payment-channels) |
| CRDT ledger + Settlement | Complete (validation rules, GCounter merge, GSet dedup, rebase) | [CRDT Ledger](../economics/crdt-ledger) |
| Hardware targets | Complete (ESP32 + Pi reference designs) | [Reference Designs](../hardware/reference-designs) |
