# Mehr Network

**A Decentralized Capability Marketplace Over Transport-Agnostic Mesh**

Mehr is a decentralized network where every resource — bandwidth, compute, storage, connectivity — is a discoverable, negotiable, verifiable, payable capability. Nodes participate at whatever level their hardware allows. Nothing is required except a cryptographic keypair.

> *Mehr* (مهر) — Persian for bond, covenant, and sun. In Zoroastrian tradition, the deity of contracts and mutual obligation.

## Why Mehr?

- **A village with no ISP** can still communicate internally over LoRa radio
- **A country under internet shutdown** can maintain mesh connectivity between citizens
- **A community** can run its own local network and bridge to the wider internet through any available uplink
- **Every device** — from a $30 solar-powered relay to a GPU workstation — contributes what it can and pays for what it needs

## Core Principles

| Principle | Description |
|-----------|-------------|
| **Transport Agnostic** | Any medium that can move bytes is a valid link — 500 bps radio to 10 Gbps fiber |
| **Capability Agnostic** | Nodes advertise what they can do; hardware determines capability, the market determines role |
| **Partition Tolerant** | Network fragmentation is expected operation, not an error state |
| **Anonymous by Default** | Packets carry no source address; identity is a cryptographic keypair |
| **Free Local, Paid Routed** | Direct neighbors communicate for free; you pay only when packets traverse others' infrastructure |

## Protocol Stack

| Layer | Name | Purpose |
|-------|------|---------|
| 0 | Physical Transport | Wraps LoRa, WiFi, BLE, cellular, TCP/IP behind a uniform interface |
| 1 | Network Protocol | Identity, addressing, routing, and gossip |
| 2 | Security | Encryption, authentication, and privacy |
| 3 | Economic Protocol | MHR token, stochastic relay rewards, CRDT ledger, trust neighborhoods |
| 4 | Capability Marketplace | Discovery, agreements, and verification |
| 5 | Service Primitives | MHR-Store, MHR-DHT, MHR-Pub, MHR-Compute |
| 6 | Applications | Messaging, social, voice, naming, forums, hosting |

## Documentation

The full specification is available at **[mehr.network](https://mehr.network)**.

### Local Development

```bash
npm install
npm start        # dev server at localhost:3000
npm run build    # production build
```

### Generate PDF

```bash
npm run generate-pdf    # outputs static/mehr-protocol-spec-v1.0.pdf
```

Requires Chromium (via Puppeteer). On CI without Chrome (e.g., Vercel), the build skips PDF generation and serves the pre-committed PDF.

## Project Structure

```
docs/
├── protocol/          Transport, network protocol, security
├── economics/         MHR token, payment channels, CRDT ledger, trust
├── marketplace/       Discovery, agreements, verification
├── services/          MHR-Store, MHR-DHT, MHR-Pub, MHR-Compute
├── applications/      Messaging, social, voice, naming, forums, hosting
├── hardware/          Reference designs, device tiers
├── development/       Roadmap, design decisions, resolved questions
├── introduction.md    Protocol overview
├── faq.md             Frequently asked questions
└── specification.md   Full spec summary with download button
```

## Status

**v1.0 — Design Complete, Pre-Implementation.** All architectural and implementation-level questions have been resolved across five rounds of spec review. The protocol is ready for Phase 1 implementation.

## License

Copyright Mehr Network Contributors.
