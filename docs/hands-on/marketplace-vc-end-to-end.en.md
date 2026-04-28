# 4-VC end-to-end (v2 / Stage 6)

!!! abstract "Stage 1–5 capstone"
    Walks Seller-side **ServiceVC** continuous writes feeding into
    Buyer-side **PurchaseViewerVC** reads in a single session.
    Backend pieces are reused from Stages 1–5 + Stage 6 case B
    (dataset_id from `Merchandise.additionalInfo`).

> The Japanese page ([marketplace-vc-end-to-end.md](marketplace-vc-end-to-end.md))
> is the canonical step-by-step. This page lists the structural
> points only.

## What you'll learn

- Four VC kinds (ConsentVC / ViewerVC / **ServiceVC** / **PurchaseViewerVC**)
  cooperating around one dataset
- Merchandise contracts now carry `dataset_id` in `additionalInfo`,
  so the bridge resolves the dataset from chain state instead of
  a hardcoded fallback
- A multi-row audit chain ties ETH key → did:jwk → ServiceVC holder
  → PurchaseViewerVC holder around one dataset

## Step overview

| Step | What it verifies |
| --- | --- |
| E0 | Stage 5 environment up + new deploy with `dataset_id` in `additionalInfo` |
| E1 | Seller flow: present ServiceVC → ingest 5 rows |
| E2 | Buyer flow: purchase a Merchandise; bridge resolves dataset on-chain |
| E3 | Buyer wallet receives PurchaseViewerVC; eth↔did binding audited |
| E4 | Buyer presents → ViewerToken → `/platform/data?merchandise=<addr>` returns the seller's 5 rows |
| E5 | Audit log shows the chain (write × 5, claim, issued, oid4vp, data) tied by one dataset |

## Three-dataset deploy (case B)

The latest `deployMerchandiseWithIoTMarket.ts` distributes 5 merchandises
across 3 datasets:

| Merchandise | dataset_id |
| --- | --- |
| #0, #1, #3 | `home/env/temperature` |
| #2 | `home/env/humidity` |
| #4 | `home/env/flood_risk_high` |

The bridge picks each merchandise's dataset from `additionalInfo`,
falling back to `BRIDGE_DATASET_DEFAULT` for legacy (no-key)
deployments.

## Limits / future

- Same iPhone wallet is used for both seller and buyer roles in the
  hands-on; real seller/buyer separation requires two devices or
  in-wallet account switching
- SellerVC (gating Merchandise registration by holder identity) is
  out of scope; planned as a future hands-on (case C)
- EIP-712 binding for eth_addr ↔ did:jwk is still MVP-style

## Related

- [Marketplace VC Bridge spec](../design/marketplace-vc-bridge-spec.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
- [SSI Viewer (Stage 3)](ha-ssi-viewer.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
- [Marketplace × Wallet bridge (Stage 5)](marketplace-vc-bridge.md)
