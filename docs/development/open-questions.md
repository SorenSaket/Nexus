---
sidebar_position: 3
title: Open Questions
---

# Open Questions

All 20 original v1.0 design questions have been resolved — their resolutions are documented in [Design Decisions](design-decisions) and the relevant spec pages. This page tracks new questions identified during the v1.0 review.

## 1. Multi-Admin Group Messaging

**Context**: Group messaging currently requires a single admin node to handle key rotation and member management. If the admin goes offline, no new members can join and no key rotation occurs.

**Question**: Should groups support threshold-based admin authority (e.g., 2-of-3 admins can rotate keys) using threshold signatures? What is the overhead on constrained devices?

**Relevant**: [Group Messaging](../applications/messaging#group-messaging)

## 2. Reputation Gossip vs. First-Hand Only

**Context**: The [reputation system](../protocol/security#reputation) is currently first-hand only — each node scores peers based on direct experience. This means a new node has no information about any peer until it interacts with them directly.

**Question**: Should there be a lightweight reputation referral mechanism where trusted peers share their scores? This would help new nodes make better initial decisions, but opens the door to reputation manipulation via trusted-peer collusion. What is the right tradeoff?

**Relevant**: [Reputation](../protocol/security#reputation), [Trust & Neighborhoods](../economics/trust-neighborhoods)

## 3. Onion Routing for High-Threat Environments

**Context**: The current privacy model relies on sender anonymity (no source address) and link encryption. This is insufficient against an adversary monitoring multiple links simultaneously. Onion routing was deferred as "future work" in v1.0.

**Question**: What is the minimum viable onion routing design for a mesh with 1 kbps LoRa links? Each encryption layer adds ~16 bytes (nonce) + ~16 bytes (Poly1305 tag) = 32 bytes per hop. On a 465-byte max payload, 3 hops of onion routing reduces usable payload to ~369 bytes (~21% overhead). Is this acceptable? Should onion routing be opt-in per packet (via PathPolicy)?

**Relevant**: [Security — Traffic Analysis Resistance](../protocol/security#traffic-analysis-resistance)

## 4. NXS-Byte Full Opcode Specification

**Context**: The NXS-Byte instruction set design constraints are established (integer-only, ~50 opcodes, stack-based, cycle-counted) but the full opcode table is not specified. This is implementation work for Phase 3.

**Questions**:
- What is the minimum opcode set needed for the core use cases (payment validation, moderation, access control, escrow)?
- Should there be a formal specification language (like Ethereum's Yellow Paper) or is a reference interpreter sufficient?
- How should gas/cycle costs be calibrated across ESP32, ARM, and x86?

**Relevant**: [NXS-Compute](../services/nxs-compute#nxs-byte-minimal-bytecode)

## 5. Bootstrap Emission Schedule Parameters

**Context**: The emission schedule defines ~1% of ceiling per year during bootstrap, halving every 2 years, with a 0.1% tail. But the exact parameters (initial reward per epoch, halving curve shape, tail transition point) are not specified numerically.

**Questions**:
- What is the initial minting reward per epoch in μNXS?
- Is the halving discrete (step function every 2 years) or continuous (exponential decay)?
- At what circulating supply percentage does the tail emission floor activate?
- How do these parameters interact with [partition minting](../economics/crdt-ledger#partition-minting-and-supply-convergence)?

**Relevant**: [NXS Token — Supply Model](../economics/nxs-token#supply-model)

## 6. Protocol Bridge Design (SSB, Matrix, Briar)

**Context**: The [roadmap](roadmap) lists "third-party protocol bridges" in Phase 4 but provides no design. Bridges between NEXUS and existing protocols (Secure Scuttlebutt, Matrix, Briar) would accelerate adoption.

**Questions**:
- Should bridges run as NXS-Compute contracts or as standalone gateway services?
- How does identity mapping work between NEXUS Ed25519 keys and external protocol identities?
- Who pays for bridged traffic — the bridge operator, the sender, or the recipient?

**Relevant**: [Roadmap — Phase 4](roadmap#phase-4-ecosystem)

## 7. Formal Verification Targets

**Context**: The CRDT ledger, payment channels, and epoch consensus are the most critical protocol components. Bugs in these could cause loss of funds or state corruption.

**Question**: Which components should receive formal verification, and what properties should be verified? Candidates:
- CRDT merge convergence (GCounter, GSet)
- Payment channel state machine (no balance can go negative, dispute resolution terminates)
- Epoch checkpoint correctness (no settlement is permanently lost)
- Bloom filter false positive impact on settlement loss probability

**Relevant**: [CRDT Ledger](../economics/crdt-ledger), [Payment Channels](../economics/payment-channels)
