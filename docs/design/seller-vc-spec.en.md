# SellerVC — marketplace listing governance VC (Stage 7 design spec)

!!! abstract "Where this document fits"
    This is the design spec for **Stage 7**, which adds identity verification
    to Merchandise registration on iot-market. It introduces a **fifth VC**
    (Verifiable Credential), following ConsentVC / ViewerVC / ServiceVC /
    PurchaseViewerVC, and uses a VC to show who may sell which dataset. It is a draft written before implementation.
    "Stage" in this document refers to the step numbers of the Phase 2 hands-on series (listed in the "Seven hands-on stages" section of the [VC architecture overview](vc-architecture-overview.md)).
    Stage 7 corresponds to the hands-on [marketplace-seller-vc](../hands-on/marketplace-seller-vc.md), Stage 6 to [marketplace-vc-end-to-end](../hands-on/marketplace-vc-end-to-end.md), and Stage 5 to [marketplace-vc-bridge](../hands-on/marketplace-vc-bridge.md).
    Stage 8 and later have no hands-on yet and refer to items for future consideration.

## 1. Motivation

### Current state (up to Stage 6)

Anyone can call `IoTMarket.registerMerchandise(merchandiseAddr)`.
The owner of a Merchandise is fixed to the `msg.sender` of the constructor,
but nobody verifies whether that owner is a legitimate seller.

Concrete risks:

- A third party can register a Merchandise with fake data on IoTMarket
- If several sellers list the same dataset at the same time, the buyer cannot
  tell which one is the correct source
- The audit log records only Merchandise.owner (an eth address)

### What Stage 7 does

Before the seller calls `registerMerchandise()`, the publisher verifies that the
seller holds a SellerVC. Only when verification succeeds does the marketplace side
record that "this Merchandise comes from a legitimate seller".

The implementation does not change Merchandise.sol; the seller's identity is
recorded in the audit log off-chain (by the publisher). An on-chain guard is left for Stage 8 and later.

## 2. Differences between v6 and v7

Here, the current configuration up to Stage 6 is called v6, and the configuration of this spec (Stage 7) is called v7.

| Aspect | v6 (current) | v7 (this spec) |
| --- | --- | --- |
| Permission to register a Merchandise | Anyone | Anyone (compatible). The seller's identity is verified separately by the publisher |
| Seller identity | Merchandise.owner (eth) only | + did:jwk + SellerVC claims (`seller_id`, `licensed_datasets`; expiry is `exp`) |
| Detecting fraudulent listings | Not possible | Traceable through the publisher's audit row `marketplace/seller_registered` |
| What the buyer can refer to | Merchandise.owner | + `seller_did` included in the `/platform/data` response |

Compatibility with v6 is fully preserved. A seller that does not present a SellerVC
keeps working through the v6 path as before (the publisher records seller_did = "unknown").

## 3. SellerVC schema

```json
{
  "vct": "https://iw3ip.example/credentials/SellerVC/v1",
  "iss": "did:jwk:<publisher-issuer>",
  "sub": "did:jwk:<seller-holder>",
  "iat": 1735000000,
  "exp": 1766536000,
  "seller_id": "ertl-nagoya-seller-001",
  "licensed_datasets": [
    "home/env/temperature",
    "home/env/humidity"
  ],
  "iw3ip_issuer": "iw3ip-publisher-issuer",
  "cnf": { "jwk": { ... } }
}
```

Comparison with ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC:

| VC | TTL (lifetime) of the issued token | Number of uses | Claim that expresses the permission |
| --- | --- | --- | --- |
| ConsentVC | 5 min Token | single (one-time) | `allowed_purposes` |
| ViewerVC | 60 s Token | multi (many times within the TTL) | `allowed_actions=[read]` |
| ServiceVC | 1 h Token | multi | `allowed_actions=[write_continuous]` |
| PurchaseViewerVC | 60 s Token | multi | `allowed_actions=[read]` + purchase context |
| **SellerVC** | **24 h Token** | **multi** | **`licensed_datasets: [...]`** |

## 4. SellerToken and new endpoints

### 4.1 SellerToken

```python
@dataclass
class SellerToken:
    jti: str
    token: str
    seller_did: str
    licensed_datasets: list[str]
    issued_at: float
    expires_at: float           # 24h
    register_count: int = 0     # +1 per listing
```

It is used in the same way as a ServiceToken (multi-use with a long TTL). The differences are
that it carries `licensed_datasets` and that it can be used only with the register API.

### 4.2 New endpoint

#### `POST /marketplace/register`

After the seller's Merchandise has been registered on IoTMarket (after the
`registerMerchandise()` call on Hardhat), this endpoint notifies the publisher of
the identity of that Merchandise's seller.

Request:

```json
{
  "merchandise_address": "0x...",
  "seller_eth_addr": "0x...",
  "tx_hash": "0x..."   // registerMerchandise tx
}
```

Headers:

```
Authorization: Bearer <SellerToken>
```

Server-side verification:

1. The SellerToken is valid
2. Read `dataset_id` with Merchandise.getAllAdditionalInfo()
3. `dataset_id` is included in the SellerToken's `licensed_datasets`
4. (Optional) Verify that Merchandise.getOwner() == `seller_eth_addr`. Done only when `MARKETPLACE_HARDHAT_RPC` is set on the publisher

On success:

- Write a `marketplace/seller_registered` audit row (seller_did, dataset_id, merchandise_address, tx_hash)
- Keep a `merchandise -> seller_did` index internally
- 200 + `{registered: true, seller_did, dataset_id}`

On failure (typical cases):

| Reason | HTTP |
| --- | --- |
| `seller_token_unknown` | 401 |
| `seller_token_expired` | 401 |
| `dataset_not_licensed` | 403 |
| `merchandise_lookup_failed` | 502 |

### 4.3 Extension to an existing endpoint

The response of `GET /platform/data?merchandise=<addr>` optionally includes
`seller_did`.

```json
{
  "dataset_id": "home/env/temperature",
  "count": 5,
  "read_count": 1,
  "seller_did": "did:jwk:...",
  "rows": [...]
}
```

When `seller_did` is `"unknown"`, the seller of that Merchandise has not been registered
through the v7 path (it was listed through the v6 path, or it is not registered).

## 5. Data flow (v7 sequence)

```
[Seller]
  │  /issuer/offer?type=SellerVC&seller_id=...
  │  → receive in the wallet
  │
  │  /verifier/request?vc_kind=SellerVC
  │  → present from the wallet
  │  → SellerToken (24h)
  │
  │  Hardhat console / iot-market-ui (seller mode):
  │  deploy the Merchandise
  │  IoTMarket.registerMerchandise(addr)  ← on-chain
  │
  │  POST /marketplace/register
  │  Authorization: Bearer <SellerToken>
  │  → audit: marketplace/seller_registered
  │  → publisher records merchandise -> seller_did
  │
[Buyer (same as Stage 5/6)]
  │  Merchandise.purchase()
  │  → bridge → /marketplace/claim
  │  → receive PurchaseViewerVC → present → ViewerToken
  │
  │  GET /platform/data?merchandise=<addr>
  │  → existing dataset resolution logic + new: seller_did is also included
```

## 6. Issuance governance

**MVP** (Minimum Viable Product, the minimal implementation run in the hands-on): the publisher itself issues the SellerVC (the same policy as Stages 1–6).
Concretely, it is a simple API that takes seller_id and licensed_datasets as query parameters, as in
`/issuer/offer?type=SellerVC&seller_id=...&licensed_datasets=...`.

For an educational hands-on, this is enough to show with a VC "which datasets the seller
is permitted to handle".

**Production assumption (future)**: a separate "marketplace authority" service issues
the SellerVC, and the publisher only verifies it. This is outside the scope of this spec.

## 7. Seller mode in iot-market-ui

New route `/seller`:

1. Show the SellerVC presentation deeplink (if the seller does not have one yet, point to /issuer/offer)
2. Hold the SellerToken
3. Form for deploying a Merchandise (price, dataset_id, fileType, dataSize)
4. Run deploy + registerMerchandise + /marketplace/register in one sequence

In the MVP, `/seller` stays a simple page (the hands-on describes an alternative
procedure that uses the Hardhat console).

## 8. Additions to the audit log

New `raw_topic`:

- `marketplace/seller_registered` — `register_count:<jti>:N` (one seller
  registers its N-th Merchandise)

Example:

```json
{
  "raw_topic": "marketplace/seller_registered",
  "subject_did": "did:jwk:...",
  "purpose": "register",
  "reason": "seller_register:claim=...:eth=0x...:tx=0x...:dataset=home/env/temperature",
  "holder_did": "did:jwk:..."
}
```

## 9. Test strategy

### 9.1 Publisher unit tests (`tests/test_marketplace_seller_vc.py`, 8–10 cases)

- SellerVC issuance (appears in the issuer metadata)
- Presentation → SellerToken
- `/marketplace/register` returns 200 and binds seller_did
- `licensed_datasets` mismatch → 403
- Expired → 401
- Unknown token → 401
- `seller_did` included in the `/platform/data` response
- `seller_did = "unknown"` shown for an unregistered Merchandise

### 9.2 e2e

The e2e (end-to-end) test uses the hands-on procedure as is (as in Stage 5/6).

## 10. Milestones

| ID | Content | Duration |
| --- | --- | --- |
| **C1** | Design spec (this document) | (this page) |
| **C2** | publisher: SellerVC + SellerToken + state | 3–4 days |
| **C3** | publisher: `/marketplace/register` + `seller_did` in `/platform/data` | 2 days |
| **C4** | tests + e2e validation | 2 days |
| **C5** | iot-market-ui `/seller` page | 2–3 days |
| **C6** | Hands-on `marketplace-seller-vc.md` (JA + EN) | 2–3 days |

The total is 2–3 weeks, roughly the same as Stage 5/6.

## 11. Open questions

1. **Wildcard in `licensed_datasets`**
    - Should `["*"]` be allowed? It is useful for large sellers but needs an operational decision
    - Recommendation: explicit lists only in the MVP
2. **Verifying Merchandise.getOwner() == seller_eth_addr**
    - Make it mandatory in Stage 7, or defer it to Stage 8 and later
    - Recommendation: mandatory (the minimum needed against impersonation)
    - Implementation: made optional. The check runs only when `MARKETPLACE_HARDHAT_RPC` is set; otherwise it is skipped and the audit log records `owner_verify=skipped`
3. **How much to automate in the `/seller` UI**
    - Do the deploy to Hardhat inside the UI as well, or use the Hardhat console instead in the hands-on
    - Recommendation: the latter (keeps the UI effort down)
4. **Handling existing Merchandise (already deployed in Stage 6)**
    - Allow `/marketplace/register` to be called after the fact, or require a new deploy
    - Recommendation: allow it after the fact (for compatibility)

## 12. Related

- [Marketplace VC Bridge design spec (v1/v2)](marketplace-vc-bridge-spec.md)
- [SSI Wallet (Stage 1)](../hands-on/ha-ssi-wallet.md)
- [SSI Viewer (Stage 3)](../hands-on/ha-ssi-viewer.md)
- [SSI Service (Stage 4 prep)](../hands-on/ha-ssi-service.md)
- [Marketplace × Wallet bridge (Stage 5)](../hands-on/marketplace-vc-bridge.md)
- [Marketplace VC end-to-end (Stage 6)](../hands-on/marketplace-vc-end-to-end.md)
- [Back the seller's identity with a Seller VC (Stage 7)](../hands-on/marketplace-seller-vc.md) (the hands-on created in C6)
- [VC architecture overview](vc-architecture-overview.md) (the list of Stage numbers is in the "Seven hands-on stages" section)
