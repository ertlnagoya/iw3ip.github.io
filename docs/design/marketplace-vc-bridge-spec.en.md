# Marketplace VC Bridge — v1 / v2 Design Spec

!!! abstract
    Design spec for connecting the marketplace to the smartphone SSI
    wallet. The current system is **v1**; the system this document
    specifies is **v2**. **M1 (spec) draft.**

> The Japanese version ([marketplace-vc-bridge-spec.md](marketplace-vc-bridge-spec.md))
> is the canonical source. This page lists the structural points only —
> consult the JA spec for full details and Mermaid diagrams.

## 1. v1 / v2 relationship

| Term | What it refers to |
| --- | --- |
| **v1** | Current Marketplace + MetaMask + encrypted-IPFS delivery |
| **v2** | v1 plus a bridge service, PurchaseViewerVC, and publisher data API |

**v2 does not replace v1.** Both delivery lanes coexist after a single
purchase: v1 (encrypted URI → buyer decrypts) and v2 (PurchaseViewerVC
→ wallet → ViewerToken → `/platform/data`).

## 2. Common vs divergent

### 2.1 Reused as-is in v2
- `iot-market` Solidity contracts (Merchandise / IoTMarket / PubKey)
- MetaMask payment flow
- Hardhat local chain, IPFS, PostgreSQL `ipfs_records`
- iot-market-ui home / list / existing purchase flow

### 2.2 New in v2
- **bridge service** (Node, in `bridge/`)
- **PurchaseViewerVC** (4th VC kind on the publisher)
- `POST /marketplace/claim`
- `GET /platform/data?merchandise=<addr>`
- iot-market-ui `/purchased/[txHash]` for VC delivery

### 2.3 Kept in v2 (v1 lane stays)
- `Merchandise.emitUpload(encryptURI)` + Upload event
- `PubKey` registry
- Encrypted-URI → decrypt flow

### 2.4 Out of scope (v1 nor v2)
- KYC / identity-proofing VCs
- did:ethr or other on-chain DID protocols
- Multi-chain support
- Auction / negotiated pricing

## 3. Actor responsibility matrix

| Action | v1 | v2 |
| --- | --- | --- |
| Discovery | iot-market-ui | iot-market-ui (same) |
| Payment | MetaMask + `purchase()` | MetaMask + `purchase()` (same) |
| Buyer identity | Ethereum address | Ethereum address + did:jwk |
| Authorization check | `confirmedBuyers` mapping | `confirmedBuyers` + PurchaseViewerVC presentation |
| Data fetch | encrypted URI → decrypt | `GET /platform/data?merchandise=<addr>` (Bearer) |
| Audit | on-chain Upload event | publisher audit log (eth_addr, did, tx_hash, jti) |

## 4. PurchaseViewerVC schema

Required claims beyond a regular ViewerVC:
- `merchandise_address`
- `buyer_eth_addr`
- `tx_hash`
- `dataset_id`
- `purchased_at`
- `allowed_actions: ["read"]`

VCT: `https://iw3ip.example/credentials/PurchaseViewerVC/v1`
TTL: 24 h (subject to review)

## 5. eth_addr ↔ did:jwk binding

**MVP**: bridge passes `buyer_eth_addr` to publisher in the OID4VCI offer
context; publisher records `eth_addr ↔ did:jwk` in the audit log when
the wallet finishes credential receipt. Trusts the bridge — explicitly
flagged as "hands-on use only, production needs §7.2."

**Production (future)**: EIP-712 signed assertion that the buyer owns
the `tx_hash`. Not implemented in MVP; called out in spec only.

## 6. Milestones

| ID | Content | Duration |
| --- | --- | --- |
| **M1** | Design spec (this doc) | 1 wk |
| **M2** | bridge skeleton + `/marketplace/claim` | 1 wk |
| **M3** | PurchaseViewerVC + eth↔did binding | 3-4 d |
| **M4** | `/platform/data?merchandise=<addr>` + tests | 3-4 d |
| **M5** | iot-market-ui post-purchase delivery | 3-4 d |
| **M6** | Hands-on docs | 3-4 d |

## 7. Related

- [SSI Wallet hands-on (Stage 1)](../hands-on/ha-ssi-wallet.md)
- [SSI Viewer hands-on (Stage 3)](../hands-on/ha-ssi-viewer.md)
- [SSI Service hands-on (Stage 4 prep)](../hands-on/ha-ssi-service.md)
- Future: `hands-on/marketplace-vc-bridge.md` (M6)
