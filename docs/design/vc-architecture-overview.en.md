# VC architecture overview (Phase 2 / Stages 0–7)

!!! abstract "Where this document fits"
    A bird's-eye summary of what Phase 2 built: **5 VC kinds × 4 token
    kinds × 7 hands-on stages**. See the individual pages for the details
    of each spec. End-to-end behavior is verified with 75 publisher tests,
    and all 7 stages were verified on a real iPhone.

## 1. Overview

**Writing data, reading data, and listing identity** are authorized by a human or a machine presenting a VC. Each VC records "who is allowed to do what" as claims, a presentation yields a short-lived token, and the APIs are gated by that token.

## 2. Five VC kinds

| Kind | Role | Main claims | Expected holder |
| --- | --- | --- | --- |
| **ConsentVC** | **Single-use** consent to write data | `dataset_id`, `allowed_purposes[]` | Human (data provider) |
| **ViewerVC** | Read data (any dataset) | `dataset_id`, `allowed_actions=["read"]` | Human (viewer) |
| **ServiceVC** | M2M continuous write | `dataset_id`, `allowed_actions=["write_continuous"]` | Service (publisher etc.) |
| **PurchaseViewerVC** | Purchase-bound read | `dataset_id`, `merchandise_address`, `tx_hash`, `buyer_eth_addr` | Human (buyer) |
| **SellerVC** | Listing identity + dataset license | `seller_id`, `licensed_datasets[]` | Human / service (seller) |

VCs are issued as SD-JWT VC (`vct: https://iw3ip.example/credentials/<Kind>/v1`) and held by the wallet. They are presented via OID4VP (DCQL).

## 3. Four token kinds

| Token | TTL | Uses | Gated API | Source VC |
| --- | --- | --- | --- | --- |
| **PolicyToken** | 5 min | Single | `POST /platform/ingest` | ConsentVC |
| **ViewerToken** | 60 s | Multiple within TTL | `GET /platform/data` | ViewerVC / **PurchaseViewerVC** |
| **ServiceToken** | 1 h | Multiple within TTL | `POST /platform/ingest` | ServiceVC |
| **SellerToken** | 24 h | Multiple within TTL | `POST /marketplace/register` | SellerVC |

ViewerToken is issued from both ViewerVC and PurchaseViewerVC, but **the namespace is the same** (once issued, the two are treated identically). Otherwise each token kind has its own namespace and cannot be reused elsewhere (for example, using a SellerToken on ingest returns 401).

## 4. Authorization matrix (which VC gates which API)

```
                    write                              read
                    ───────────────────────────────    ────────────────────────────
                    single-use        continuous       short-term   purchase-bound
────────────────────────────────────────────────────────────────────────────────────
JSON registration │ /consents POST  │ —              │ —          │ —
Wallet (human)    │ ConsentVC       │ —              │ ViewerVC   │ PurchaseViewerVC
Wallet (M2M)      │ —               │ ServiceVC      │ —          │ —
────────────────────────────────────────────────────────────────────────────────────
Listing governance│ —               │ —              │ —          │ —
 (cross-cutting)  │   ↓ SellerVC gates "/marketplace/register"
```

**SellerVC does not gate the data APIs directly.** Instead, it leaves evidence of the seller's identity at listing time in the audit log, and that identity is included as `seller_did` in the buyer's `/platform/data?merchandise=` response.

## 5. Seven hands-on stages

```
Dependency graph of the Phase 2 hands-on

  Stage 0 (baseline: /consents JSON registration)
   ├── webcam-event-sharing
   └── environment-disaster
        ↓
  Stage 1 ── ha-ssi-wallet         ConsentVC + PolicyToken (single-use write)
        ↓
  Stage 3 ── ha-ssi-viewer         ViewerVC + ViewerToken (multi-use read)
        ↓
  Stage 4 prep ── ha-ssi-service   ServiceVC + ServiceToken (M2M continuous)
        ↓
  Stage 5 ── marketplace-vc-bridge PurchaseViewerVC (purchase-bound read)
        ↓
  Stage 6 ── marketplace-vc-end-to-end
                                   Service (write) × PurchaseViewer (read) combined
        ↓
  Stage 7 ── marketplace-seller-vc SellerVC (listing identity)
```

Details of each stage:

| Stage | Hands-on | Main additions |
| --- | --- | --- |
| 0 | [webcam-event-sharing](../hands-on/webcam-event-sharing.md), [environment-disaster](../hands-on/environment-disaster.md) | `/consents` JSON registration, MQTT → ingest |
| 1 | [ha-ssi-wallet](../hands-on/ha-ssi-wallet.md) | OID4VCI/OID4VP + ConsentVC + PolicyToken |
| 3 | [ha-ssi-viewer](../hands-on/ha-ssi-viewer.md) | ViewerVC + ViewerToken + `GET /platform/data` |
| 4 prep | [ha-ssi-service](../hands-on/ha-ssi-service.md) | ServiceVC + ServiceToken (M2M) |
| 5 | [marketplace-vc-bridge](../hands-on/marketplace-vc-bridge.md) | bridge service + PurchaseViewerVC + `merchandise=<addr>` reverse lookup |
| 6 | [marketplace-vc-end-to-end](../hands-on/marketplace-vc-end-to-end.md) | 4 VC kinds working together on one dataset + dataset_id obtained dynamically from `additionalInfo` |
| 7 | [marketplace-seller-vc](../hands-on/marketplace-seller-vc.md) | SellerVC + `/marketplace/register` + on-chain `getOwner()` verification + `seller_did` included in the response |

## 6. v1 / v2 marketplaces running side by side

From Stage 5 onward, the existing marketplace (v1) and the new scheme (v2) are designed so that **both paths run from the same Purchase event**.

| Aspect | v1 (current) | v2 (Stage 5+) |
| --- | --- | --- |
| Identifier | Ethereum address | + did:jwk |
| Authorization | `confirmedBuyers` mapping | + PurchaseViewerVC + ViewerToken |
| Delivery | Decrypting `emitUpload(encryptURI)` | + `/platform/data?merchandise=<addr>` |
| Audit | on-chain Upload event | + publisher audit log (eth_did_bound, seller_register, viewer_token_used, etc.) |
| Listing identity | Merchandise.owner (eth) | + seller_did of the SellerVC |

Details: [Marketplace VC Bridge design spec](marketplace-vc-bridge-spec.md), [SellerVC design spec](seller-vc-spec.md)

## 7. Audit log chain

A single purchase chains up to 8 audit log rows (when Stages 4-prep, 5, and 7 are all involved):

| # | raw_topic | reason | Subject (subject_did) |
| --- | --- | --- | --- |
| 1 | `oid4vp/response` | `ok` (SellerVC presented) | seller did:jwk |
| 2 | `marketplace/seller_registered` | `seller_register:...:owner_verify=verified` | seller did:jwk |
| 3 | `oid4vp/response` | `ok` (ServiceVC presented) | service did:jwk |
| 4 | `platform/ingest` × N | `service_token_used:<jti>:<n>` | service did:jwk |
| 5 | `marketplace/claim` | `claim_received:<id>:tx=...` | `eth:<buyer_addr>` |
| 6 | `marketplace/issued` | `eth_did_bound:claim=...:eth=...:tx=...` | buyer did:jwk |
| 7 | `oid4vp/response` | `ok` (PurchaseViewerVC presented) | buyer did:jwk |
| 8 | `platform/data` | `viewer_token_used:<jti>:<n>` | buyer did:jwk |

Points to note:

- `marketplace/claim` (eth) and `marketplace/issued` (did:jwk) are **linked by `claim_id`**, tying the same person to two different keys
- The `seller_did` in `marketplace/seller_registered` matches the `seller_did` in the `/platform/data` response (traceability from listing to viewing)

## 8. Token namespace isolation

Each token is **valid only on its dedicated API**:

| Token | Valid API | If reused elsewhere |
| --- | --- | --- |
| PolicyToken | `POST /platform/ingest` | `/platform/data`: 401 / `/marketplace/register`: 401 |
| ViewerToken | `GET /platform/data` | `/platform/ingest`: 401 / `/marketplace/register`: 401 |
| ServiceToken | `POST /platform/ingest` | `/platform/data`: 401 / `/marketplace/register`: 401 |
| SellerToken | `POST /marketplace/register` | `/platform/ingest`: 401 / `/platform/data`: 401 |

`/platform/ingest` **accepts both** PolicyToken and ServiceToken, but internally it tries PolicyToken first and falls back to ServiceToken only on failure (`unknown`), for compatibility and uniform error messages.

## 9. Keys and identity

Hands-on participants handle the following **three kinds of keys** at the same time:

| Key | Location | Purpose |
| --- | --- | --- |
| **Ethereum key** (Account #N) | MetaMask / Hardhat console | Paying for Merchandise purchases, calling contracts |
| **Wallet did:jwk** | iw3ip-wallet | Subject of OID4VCI/OID4VP, `cnf` of the VC |
| **Publisher issuer key** (did:jwk) | publisher container | Signing issued VCs |

The `eth_did_bound` audit row of Stage 5 records the link between the **Ethereum key ↔ did:jwk**. This is an off-chain mechanism (the publisher's audit log) for asserting that **the "ETH payer" and the "VC presenter" are the same person** (currently an MVP that simply trusts the bridge; it is planned to be strengthened with EIP-712 signatures = Stage 8).

## 10. Known limitations (to be considered for Stage 8+)

The following are **intentionally out of scope** for Stages 0–7 of Phase 2:

- **Identity binding with EIP-712 signatures** (protection against eth ↔ did:jwk impersonation)
- **On-chain guard** (requiring a SellerToken proof on the Solidity side when registering Merchandise)
- **Revocation**: VC revocation lists using Status List 2021 or similar
- **Trust framework**: verifying the identity of the "marketplace authority" that can issue SellerVCs
- **Token sharing across multiple publishers** (currently in-memory, so distributed operation is not possible)
- **Connection to the Phase 3 LLM Planner** (proving the authorization of each plan step with a VC)

## 11. Related pages

- Design specs
    - [Marketplace VC Bridge (v1/v2 spec)](marketplace-vc-bridge-spec.md) — structure of Stages 5/6
    - [SellerVC design spec (Stage 7)](seller-vc-spec.md) — design decisions of Stage 7
- Hands-on
    - [Hands-on overview (Part 2)](../hands-on/index.md)
    - For each stage, see the table in §5
