# Marketplace × Wallet bridge (v2 / Stage 5)

!!! abstract "Hands-on for the v2 lane"
    Walks the end-to-end path from a marketplace purchase to a
    wallet-held PurchaseViewerVC and finally to gated data retrieval.
    See the [Marketplace VC Bridge design spec](../design/marketplace-vc-bridge-spec.md)
    for architecture details.

> The Japanese page ([marketplace-vc-bridge.md](marketplace-vc-bridge.md))
> is the canonical step-by-step. This page summarizes the structural
> points only.

## What you'll learn

- v1 (`emitUpload(encryptURI)`) and v2 (`PurchaseViewerVC`) **coexist**
  after one purchase
- `/marketplace/claim` is **idempotent on tx_hash** — the bridge listener
  and the iot-market-ui can both call it safely
- `eth_addr ↔ did:jwk` binding lands in the audit log on credential
  receipt (`raw_topic=marketplace/issued`,
  `reason=eth_did_bound:claim=...:eth=...:tx=...`)
- `/platform/data?merchandise=<addr>` reverse-resolves the dataset

## Common pitfalls

- The bridge in compose talks to a **host-side Hardhat** via
  `host.docker.internal:8545` (Hardhat is not co-located in compose)
- iot-market-ui POSTs to `/marketplace/claim` directly; this races the
  bridge listener but idempotency makes it safe
- PurchaseViewerVC, ConsentVC, ViewerVC, and ServiceVC are **distinct**
  in the wallet — four different cards
- The deeplink only works after the wallet is paired

## Architecture sketch

```
buyer's MetaMask  --purchase()-->  Merchandise contract
                                       │  Purchase event
                                       ├──> bridge listener
                                       │      POST /marketplace/claim
                                       └──> iot-market-ui /purchased/[tx]
                                              POST /marketplace/claim
                                              (idempotent)
                                                ↓
                                         publisher
                                                ↓ deeplink
                                          iw3ip-wallet
                                            (OID4VCI)
                                          PurchaseViewerVC
                                                ↓ OID4VP
                                          ViewerToken
                                                ↓
                            GET /platform/data?merchandise=<addr>
```

## Audit log shape (one purchase produces ≥4 rows)

| raw_topic | reason |
| --- | --- |
| `marketplace/claim` | `claim_received:<id>:tx=...` |
| `marketplace/issued` | `eth_did_bound:claim=<id>:eth=...:tx=...` |
| `oid4vp/response` | `ok` |
| `platform/data` | `viewer_token_used:<jti>:<read_count>` |

## Limits

- eth_addr ↔ did:jwk binding is bridge-trusted in MVP (production
  needs EIP-712 — see spec §7.2)
- `dataset_id` discovery from Merchandise contract is hardcoded in
  the front-end; reading from `additionalInfo` is a future tweak
- ServiceVC (Stage 4 prep) and PurchaseViewerVC (Stage 5) live in
  separate lanes; combining the two is left to a future hands-on

## Related

- [Marketplace VC Bridge spec (v1/v2)](../design/marketplace-vc-bridge-spec.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
- [SSI Viewer (Stage 3)](ha-ssi-viewer.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
