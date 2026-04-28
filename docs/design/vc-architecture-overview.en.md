# VC architecture overview (Phase 2 / Stages 1–7)

!!! abstract
    One-page synthesis of Phase 2: 5 VC kinds × 4 token kinds × 7 hands-on
    stages. Individual specs and hands-on pages are linked at the bottom.
    Verified end-to-end on a real iPhone wallet across all 7 stages;
    publisher backs it with 75 passing tests.

> The Japanese page ([vc-architecture-overview.md](vc-architecture-overview.md))
> is the canonical version with the full audit-chain table. This page lists
> the structural points only.

## TL;DR

Authorize **write / read / listing identity** by presenting VCs from a wallet
(or a service-side holder). Each VC encodes "who is allowed to do what" in
its claims; presentation yields a short-lived token; APIs are gated on the
token type matching their semantics.

## Five VCs

| Kind | Role | Claims | Holder |
| --- | --- | --- | --- |
| **ConsentVC** | single-use write consent | `dataset_id`, `allowed_purposes[]` | human (data provider) |
| **ViewerVC** | read for any dataset | `dataset_id`, `allowed_actions=["read"]` | human (viewer) |
| **ServiceVC** | M2M continuous write | `dataset_id`, `allowed_actions=["write_continuous"]` | service (publisher etc.) |
| **PurchaseViewerVC** | purchase-bound read | `dataset_id`, `merchandise_address`, `tx_hash`, `buyer_eth_addr` | human (buyer) |
| **SellerVC** | listing identity + dataset license | `seller_id`, `licensed_datasets[]` | human / service (seller) |

VCT: `https://iw3ip.example/credentials/<Kind>/v1`. Format SD-JWT VC,
issued via OID4VCI, presented via OID4VP (DCQL).

## Four tokens

| Token | TTL | Use | Gated API | Source VC |
| --- | --- | --- | --- | --- |
| **PolicyToken** | 5 min | single | `POST /platform/ingest` | ConsentVC |
| **ViewerToken** | 60 s | multi within TTL | `GET /platform/data` | ViewerVC / **PurchaseViewerVC** |
| **ServiceToken** | 1 h | multi within TTL | `POST /platform/ingest` | ServiceVC |
| **SellerToken** | 24 h | multi within TTL | `POST /marketplace/register` | SellerVC |

Token namespaces are isolated: e.g. SellerToken on `/platform/ingest` → 401.
ViewerToken minted from ViewerVC vs PurchaseViewerVC is treated identically
once issued.

## Authorization matrix

```
                write                       read
                ─────────────────           ─────────────────
                single-use   continuous     short      purchase-bound
─────────────────────────────────────────────────────────────────────
JSON-based     │ /consents POST  │ —          │ —         │ —
wallet (human) │ ConsentVC       │ —          │ ViewerVC  │ PurchaseViewerVC
wallet (M2M)   │ —               │ ServiceVC  │ —         │ —
─────────────────────────────────────────────────────────────────────
listing        │ SellerVC gates POST /marketplace/register
governance     │   then surfaces seller_did in /platform/data?merchandise=
```

SellerVC does not gate data APIs directly; it leaves an audit trail and
makes `seller_did` visible to buyers.

## Seven hands-on stages

| Stage | Page | Adds |
| --- | --- | --- |
| 0 | [webcam-event-sharing](../hands-on/webcam-event-sharing.md), [environment-disaster](../hands-on/environment-disaster.md) | `/consents` JSON, MQTT → ingest |
| 1 | [ha-ssi-wallet](../hands-on/ha-ssi-wallet.md) | OID4VCI/OID4VP + ConsentVC + PolicyToken |
| 3 | [ha-ssi-viewer](../hands-on/ha-ssi-viewer.md) | ViewerVC + ViewerToken + `/platform/data` |
| 4 prep | [ha-ssi-service](../hands-on/ha-ssi-service.md) | ServiceVC + ServiceToken (M2M) |
| 5 | [marketplace-vc-bridge](../hands-on/marketplace-vc-bridge.md) | bridge + PurchaseViewerVC + `merchandise=<addr>` lookup |
| 6 | [marketplace-vc-end-to-end](../hands-on/marketplace-vc-end-to-end.md) | 4-VC capstone + dataset_id from `additionalInfo` |
| 7 | [marketplace-seller-vc](../hands-on/marketplace-seller-vc.md) | SellerVC + `/marketplace/register` + `seller_did` |

## v1 / v2 marketplace coexistence

| Aspect | v1 (current) | v2 (Stage 5+) |
| --- | --- | --- |
| Identity | Ethereum address | + did:jwk |
| Authorization | `confirmedBuyers` mapping | + PurchaseViewerVC + ViewerToken |
| Delivery | `emitUpload(encryptURI)` decrypt | + `/platform/data?merchandise=` |
| Audit | on-chain Upload event | + publisher audit log (eth_did_bound, seller_register, viewer_token_used) |
| Listing identity | Merchandise.owner (eth) | + SellerVC seller_did |

Both lanes run from the same Purchase event; v2 is additive, never replacing v1.

## Three keys, one person

A hands-on participant simultaneously juggles three keys:

| Key | Lives in | Used for |
| --- | --- | --- |
| Ethereum key | MetaMask / Hardhat console | purchase, contract calls |
| Wallet did:jwk | iw3ip-wallet | OID4VCI/OID4VP subject, VC `cnf` |
| Issuer did:jwk | publisher container | signing issued VCs |

The `eth_did_bound` audit row written when the wallet completes credential
receipt is the off-chain assertion that "this Ethereum payer and this did:jwk
holder are the same person." Today the bridge is trusted; production needs
EIP-712 (Stage 8).

## Out-of-scope (planned for Stage 8+)

- EIP-712 binding of Ethereum address ↔ did:jwk (anti-impersonation)
- On-chain registration guard (Solidity-side proof requirement)
- VC revocation (Status List 2021)
- Trust framework for SellerVC issuance
- Multi-publisher token sharing (currently in-memory)
- Phase 3 LLM Planner integration (per-step VC proofs)

## Related

- Specs
    - [Marketplace VC Bridge (v1/v2 spec)](marketplace-vc-bridge-spec.md)
    - [SellerVC spec](seller-vc-spec.md)
- Hands-on
    - [Phase 2 overview](../hands-on/index.md#phase-2-event-sharing)
