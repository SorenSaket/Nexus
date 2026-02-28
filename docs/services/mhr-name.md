---
sidebar_position: 5
title: "MHR-Name"
---

# Naming (MHR-Name)

MHR-Name provides human-readable names scoped to [hierarchical scopes](../economics/trust-neighborhoods#hierarchical-scopes). There is no global namespace — names resolve locally based on proximity and trust.

## Scope-Based Names

Names follow the format `name@scope`:

```
maryam@geo:tehran
alice@geo:portland
relay-7@geo:backbone-west
pikachu-fan@topic:gaming/pokemon
```

The scope portion uses the [HierarchicalScope](../economics/trust-neighborhoods#hierarchical-scopes) format. Geographic scopes are prefixed with `geo:`, interest scopes with `topic:`.

Resolution works by proximity:

1. Parse the scope from the name (e.g., `geo:portland`)
2. Search nearby nodes whose scopes match
3. Within that cluster, resolve the name via local naming records
4. Use the returned NodeID for routing

## Why No Global Namespace?

A global namespace requires global consensus on name ownership. Global consensus contradicts Mehr's partition tolerance requirement. If two partitioned networks both register the name "alice," there is no partition-safe way to resolve the conflict.

Scope-based names solve this:
- Names are locally consistent within their scope
- Different communities can have different "alice" users — they are `alice@geo:portland` and `alice@geo:tehran`
- No global coordination needed
- Scopes are self-assigned, not centrally managed

## Name Registration

Names are registered locally by announcing a name binding:

```
NameBinding {
    name: String,                       // "alice"
    scope: HierarchicalScope,           // Geo("portland") or Topic("gaming", "pokemon")
    node_id: NodeID,
    signature: Ed25519Signature,
}
```

Name bindings propagate via gossip within the matching scope. Conflicts (two nodes claiming the same name in the same scope) are resolved by precedence:

1. **Trust-weighted** (highest priority) — if one claimant is in your trust graph and the other is not, the trusted claimant wins regardless of timing. Between two trusted claimants, the one with the shorter trust distance wins.
2. **First-seen** (tiebreaker) — if both claimants have equal trust status (both trusted at the same distance, or both untrusted), the first binding your node received wins.
3. **Local petnames** (ultimate fallback) — if you need guaranteed resolution regardless of conflicts, assign a local petname (see below). Petnames override all network name resolution.

## Cross-Scope Resolution

To resolve a name in a different scope:

1. Query propagates toward nodes whose scopes match the target
2. Nearest matching cluster responds with the name binding
3. Result is cached locally with a TTL

Because scopes are self-assigned, resolution finds the **nearest** cluster with that scope. This is usually what you want — `alice@geo:portland` resolves to the Portland cluster nearest to you.

**Collision across regions**: Two different cities named "portland" (e.g., Portland, Oregon and Portland, Maine) would both match `geo:portland`. Proximity-based resolution handles this naturally — you'll reach whichever Portland is closer in the mesh. To disambiguate explicitly, use a longer scope path: `alice@geo:us/oregon/portland` vs. `alice@geo:us/maine/portland`.

## Backward Compatibility

The old `name@community-label` format maps to the new scope format:

```
alice@portland-mesh  →  alice@geo:portland
relay-7@backbone-west  →  relay-7@geo:backbone-west
```

Nodes running older software continue to use the flat `community_label` field. New nodes check both `scopes` and `community_label` for resolution.

## Local Petnames

As a fallback and privacy feature, each user can assign **petnames** — local nicknames for NodeIDs on their own device:

```
User's petname mapping (local only, never shared):
  "Mom"    → 0x3a7f...b2c1
  "Work"   → 0x8e2d...f4a9
  "Doctor" → 0x1b5c...d3e7
```

Petnames are:
- Completely private (stored only on the user's device)
- Not resolvable by anyone else
- Useful when community names aren't available or desired
- The most censorship-resistant naming possible — no one can take your petnames from you
