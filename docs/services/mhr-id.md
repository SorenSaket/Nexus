---
sidebar_position: 6
title: "MHR-ID"
---

# Identity & Claims (MHR-ID)

Mehr identity is **self-certifying** — your public key is your identity, and no authority can revoke it. But identity is more than a key. People want to know: *Where are you? What do you care about? Are you the same person who used to have a different key?* Identity claims and vouches answer these questions through mesh-native peer attestation.

## Claims

An **IdentityClaim** is a signed assertion by a node about itself:

```
IdentityClaim {
    claimant: NodeID,
    public_key: Ed25519PublicKey,   // enables self-verification without prior key exchange
    claim_type: enum {
        GeoPresence {
            scope: HierarchicalScope,       // "I am in Portland"
        },
        CommunityMember {
            scope: HierarchicalScope,       // "I'm in the Pokemon community"
        },
        KeyRotation {
            old_key: PublicKey,             // previous identity
            new_key: PublicKey,             // current identity
        },
        Capability {
            cap_type: CapabilityType,      // "I operate a relay" / "I have 100GB storage"
            evidence: Option<Blake3Hash>,   // hash of proof data
        },
        ExternalIdentity {
            platform: String,              // "github", "twitter", etc.
            handle: String,                // username on that platform
        },
    },
    evidence: Option<Evidence>,     // proof backing the claim (embedded in claim_data)
    created: Timestamp,
    expires: Option<Timestamp>,     // None = no expiry (must be renewed by vouches)
    signature: Ed25519Sig,          // signed by claimant
}
```

### Claim Types

| Type | Purpose | Verification Method | Requires Proof? |
|------|---------|-------------------|----------------|
| **GeoPresence** | "I am present in this place" | Physical: [RadioRangeProof](#radiorangeproof) + peer vouches. Virtual: application-specific attestation | Depends on place type |
| **CommunityMember** | "I participate in this interest community" | Self-declared, no verification needed | No |
| **KeyRotation** | "My old key migrated to this new key" | Must be signed by both old and new keys | Yes |
| **Capability** | "I provide this service" | [Proof-of-service](../marketplace/verification) challenge-response | Yes |
| **ExternalIdentity** | "I am this person on an external platform" | Out-of-band (e.g., post a signed challenge on the platform) | Optional |

**CommunityMember claims are never verified** — anyone can declare interest in Pokemon. The value comes from the social graph: if 50 nodes you trust all claim `Topic("gaming", "pokemon")`, that's a real community.

**ExternalIdentity** is optional and only relevant for nodes with internet access. It's the Mehr equivalent of [FUTO ID](https://docs.polycentric.io/futo-id/) claims, but it's not a core feature — most mesh nodes don't have internet.

### Wire Format

| Field | Size | Description |
|-------|------|-------------|
| `claimant` | 16 bytes | Destination hash |
| `public_key` | 32 bytes | Ed25519 verifying key (enables self-verification without prior key exchange) |
| `claim_type` | 1 byte | 0=GeoPresence, 1=CommunityMember, 2=KeyRotation, 3=Capability, 4=ExternalIdentity |
| `claim_data_len` | 2 bytes | Length of claim_data (u16 LE) |
| `claim_data` | variable | Type-specific payload (includes evidence if applicable) |
| `created` | 8 bytes | Unix timestamp |
| `expires` | 1–9 bytes | 1 byte flag (0=no expiry, 1=has expiry) + 8 bytes timestamp if flag=1 |
| `signature` | 64 bytes | Ed25519 signature |

Minimum claim size: 124 bytes (no data, no expiry). Fits comfortably in a single LoRa frame.

## Vouches

A **Vouch** is a trust-weighted endorsement of someone else's claim:

```
Vouch {
    voucher: NodeID,                // who is vouching
    claim_hash: Blake3Hash,         // Blake3 hash of the IdentityClaim being vouched for
    confidence: u8,                 // 0-255: how confident the voucher is
    sequence: u64,                  // monotonic counter for superseding/revoking
    signature: Ed25519Sig,          // signed by voucher
}
```

### Vouch Properties

- **Trust-weighted**: A vouch from a node you trust directly is worth more than one from a stranger. Vouch weight decays with trust distance, just like [transitive credit](../economics/trust-neighborhoods#trust-based-credit): 100% for direct trusted peers, 10% for friend-of-friend, 0 beyond 2 hops.
- **Expiring**: Vouches are valid for a configurable period (default: 30 epochs). After expiry, the vouch must be renewed or the claim loses its verified status. This prevents stale geographic claims from persisting after someone moves.
- **Revocable**: A voucher can publish a revocation (vouch with `confidence: 0` for the same `claim_hash`) at any time.
- **Cumulative**: Multiple vouches for the same claim increase confidence. A geographic claim vouched by 10 trusted peers is stronger than one vouched by 1.

### Vouch Wire Format

| Field | Size | Description |
|-------|------|-------------|
| `voucher` | 16 bytes | Destination hash |
| `claim_hash` | 32 bytes | Blake3 hash of the claim |
| `confidence` | 1 byte | 0-255 |
| `sequence` | 8 bytes | Monotonic counter (LE). Higher sequence supersedes older vouches for the same (voucher, claim_hash) pair. |
| `signature` | 64 bytes | Ed25519 signature |

Total: 121 bytes. Lightweight enough to gossip freely.

## Verification Methods

```
                    Verification Hierarchy

  Country ─── aggregation of regions ──────────────── Lowest precision
     │
  Region ──── aggregation of cities
     │
  City ────── aggregation of neighborhoods
     │
  Neighborhood ── RadioRangeProof (LoRa beacons) ──── Highest precision
     │
  ┌──┴────────────────────────────────────────┐
  │  [Alice]  ···radio···  [Bob]              │
  │     │                    │                │
  │   witness              witness            │
  │     │                    │                │
  │     └──── [Prover] ─────┘                 │
  │           broadcasts                      │
  │           signed beacon                   │
  └───────────────────────────────────────────┘
```

### RadioRangeProof

The mesh-native equivalent of physical presence verification. If you can hear a node's LoRa radio, you're within physical range.

```
RadioRangeProof {
    prover: NodeID,                 // node proving presence
    witnesses: Vec<Witness>,        // nodes that heard the prover
    timestamp: Timestamp,
}

Witness {
    node_id: NodeID,
    rssi: i8,                       // received signal strength (dBm)
    snr: i8,                        // signal-to-noise ratio (dB)
    signature: Ed25519Sig,          // witness signs the observation
}
```

**How it works:**

1. Node broadcasts a signed presence beacon on LoRa (this already happens every 10 seconds via [presence beacons](../marketplace/discovery#presence-beacons))
2. Nearby nodes that receive the beacon can sign a Witness attestation: "I heard this node at this signal strength at this time"
3. Multiple witnesses from known locations triangulate the prover's approximate position
4. Witnesses with verified GeoPresence claims for the same area provide stronger attestation

**Range and precision:**

| Transport | Typical Range | Position Precision |
|-----------|-------------|-------------------|
| LoRa (rural) | 5–15 km | City/neighborhood level |
| LoRa (urban) | 1–5 km | Neighborhood level |
| WiFi | 30–100 m | Building level |
| Bluetooth | 10–30 m | Room level |

RadioRangeProof verifies **neighborhood-level** physical geo claims. It cannot verify city, region, or country claims directly — those use bottom-up aggregation. It also cannot verify virtual geo scopes (game servers, organizations) — those use application-specific verification such as server-signed attestations, admin vouches, or invite-chain proofs, handled at the application layer.

### Bottom-Up Aggregation (Physical Geo Scopes)

Higher-level physical geo claims are verified by aggregating verified sub-scope claims:

```
Verification levels:

Neighborhood: RadioRangeProof
  "I'm in Hawthorne" ← proved by radio witnesses in Hawthorne

City: Aggregation of neighborhoods
  "Portland exists" ← N nodes have verified claims for Portland neighborhoods
  (hawthorne + pearl + alberta + ... = Portland)

Region: Aggregation of cities
  "Oregon exists" ← nodes have verified claims across Portland, Eugene, Bend

Country: Aggregation of regions
  And so on upward.
```

No single node proves "I'm in Oregon." The **network** proves Oregon exists collectively because many nodes have independently verified neighborhood-level presence across Oregon's geography. This is inherently Sybil-resistant — you can't fake physical presence across multiple locations simultaneously.

**Aggregation thresholds:**

| Level | Minimum Verified Sub-claims | Description |
|-------|---------------------------|-------------|
| City | 3+ verified neighborhoods | At least 3 distinct neighborhood clusters |
| Region | 2+ verified cities | At least 2 cities with verified neighborhoods |
| Country | 2+ verified regions | At least 2 regions with verified cities |

These thresholds are intentionally low — the system bootstraps from small meshes. As the network grows, the aggregation becomes denser and more trustworthy naturally.

### Peer Attestation

For claims that can't be machine-verified, trusted peers vouch based on personal knowledge:

```
Alice knows Bob is her neighbor:
    → Alice vouches for Bob's GeoPresence("...", "portland", "hawthorne")
    → Alice's vouch weight = her trust score relative to the verifier

Dave knows Eve runs a reliable relay:
    → Dave vouches for Eve's Capability(relay, ...)
    → Dave's vouch weight = his trust score relative to the verifier
```

Peer attestation is the **fallback** for everything. RadioRangeProof automates geographic verification, proof-of-service automates capability verification, but peer attestation always works — even for claims no machine can verify ("this person is a good curator").

### Transitive Confidence

Vouch weight decays with trust distance, following the same model as [transitive credit](../economics/trust-neighborhoods#trust-based-credit):

```
Vouch from direct trusted peer:      confidence × 1.0
Vouch from friend-of-friend:         confidence × 0.1
Vouch from 3+ hops away:             ignored (0 weight)
```

This means a node calculates the **effective verification level** of any claim by summing trust-weighted vouches from its own perspective. Different nodes may see different verification levels for the same claim, depending on their position in the trust graph. This is by design — there is no global authority on what's verified.

## Claim Lifecycle

```
1. CREATE: Node publishes IdentityClaim (signed, stored as immutable DataObject)
2. GOSSIP: Claim propagates via MHR-DHT within relevant scope
3. VOUCH: Peers who can verify the claim publish Vouches
4. VERIFY: Other nodes calculate trust-weighted verification level
5. RENEW: Vouches expire after 30 epochs; vouchers re-vouch if claim still valid
6. REVOKE: Claimant publishes a new claim superseding the old one,
           or vouchers publish confidence=0 revocations
```

### Storage and Propagation

- Claims are stored as **immutable DataObjects** in [MHR-Store](mhr-store)
- Geographic claims propagate within the claimed scope (a Portland claim gossips within Portland)
- Interest claims propagate within the interest scope
- Vouches propagate with the claims they reference
- Both are lightweight enough for LoRa (~121–124 bytes)

## Integration with Existing Systems

### Reputation

Claims feed into the existing [reputation system](../protocol/security#sybil-resistance):

```
PeerReputation additions:
    claim_verification_level: u8,   // 0-255: aggregate verification score
    vouch_count: u16,               // number of active vouches received
```

The `claim_verification_level` is computed locally by each node based on the trust-weighted vouches it sees. A node with high verification level + high service reputation is the most trustworthy participant in the network.

### Key Rotation

The `KeyRotation` claim type works alongside the existing [KeyCompromiseAdvisory](../protocol/security#key-compromise-advisory):

```
Key migration flow:
  1. Generate new key pair
  2. Publish KeyRotation claim signed by BOTH old and new keys
     (equivalent to KeyCompromiseAdvisory with SignedByBoth evidence)
  3. Trusted peers vouch for the rotation
  4. Services (storage, compute, pub/sub) migrate agreements to new key
  5. Old key's reputation transfers to new key (trust-weighted)
```

KeyRotation claims with only the old key's signature are treated with suspicion (could be attacker with compromised key). Both-key signatures are strong evidence of legitimate migration.

### Naming

Geographic claims enable scoped naming: `alice@geo:portland` resolves only if Alice has a verified GeoPresence claim for Portland scope — see [MHR-Name](mhr-name).

### Voting

Verified geographic claims are **prerequisites for geographic voting** — a node cannot vote on Portland issues without a verified Portland-area GeoPresence claim. See [Voting](../applications/voting) for the eligibility model.

## Comparison with Other Identity Systems

| | FUTO ID (Polycentric) | Mehr Identity Claims |
|---|---|---|
| **Primary purpose** | Link centralized platform accounts | Verify mesh-native properties (location, service, community) |
| **Verification** | Crawlers scrape platforms / OAuth challenges | RadioRangeProof / proof-of-service / peer attestation |
| **Trust model** | PGP-style Web of Trust (binary vouch) | Trust-weighted vouches with transitive decay |
| **Key recovery** | None | KeyRotation claim (signed by both keys) |
| **Internet required** | Yes (must reach platforms) | No (works on LoRa mesh with zero internet) |
| **Geographic proof** | Not supported | RadioRangeProof via LoRa beacon witnesses |
| **Sybil resistance** | Social (number of vouches) | Economic (trust = absorb debts) + social (vouches) |
| **Confidence** | Binary (vouched or not) | Graduated (0–255 confidence × trust distance decay) |
