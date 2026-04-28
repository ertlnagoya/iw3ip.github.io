# Marketplace × Wallet bridge (v2 / Stage 5)

!!! abstract "Hands-on for the v2 lane"
    End-to-end path from a marketplace purchase to a wallet-held
    PurchaseViewerVC and finally to gated data retrieval.
    See the [Marketplace VC Bridge design spec](../design/marketplace-vc-bridge-spec.md)
    for architecture details.

> The Japanese page ([marketplace-vc-bridge.md](marketplace-vc-bridge.md))
> is the canonical step-by-step with full expected outputs and
> troubleshooting. This page lists the structural points only.

## What you'll learn

- v1 (`emitUpload(encryptURI)`) and v2 (`PurchaseViewerVC`) **coexist**
  after one purchase
- `/marketplace/claim` is **idempotent on tx_hash** — the bridge listener
  and the iot-market-ui can both call it safely
- `eth_addr ↔ did:jwk` binding lands in the audit log on credential
  receipt (`raw_topic=marketplace/issued`,
  `reason=eth_did_bound:claim=...:eth=...:tx=...`)
- `/platform/data?merchandise=<addr>` reverse-resolves the dataset

## Step overview

| Step | What it verifies |
| --- | --- |
| 1 | Hardhat node up, 20 test accounts available |
| 2 | PubKey + IoTMarket + Merchandise×5 deployed |
| 3 | Buyer registers a key in the PubKey contract (purchase precondition) |
| 4 | publisher exposes PurchaseViewerVC; bridge subscribes to 5 merchandises |
| 5 | MetaMask connected to Hardhat, Account #2 imported |
| 6 | Purchase event captured by bridge → `/marketplace/claim` audit row |
| 7 | iPhone wallet receives PurchaseViewerVC; eth↔did binding audited |
| 8 | Presentation yields a ViewerToken |
| 9 | `GET /platform/data?merchandise=<addr>` reverse-lookup returns 200 |
| 10 | Four-row audit chain visible (claim → issued → oid4vp → data) |

See the JA page for the exact commands, expected outputs, and the
trouble-shooting notes from the e2e validation we ran:
- MetaMask `chainId` errors → reset account, or fall back to Hardhat console
- `0x295f0a57` revert → forgot Step 3 (PubKey registration)
- iPhone wallet crashes on deeplink → `BRIDGE_PUBLIC_PUBLISHER_URL` not set,
  or Metro bundler not running, or wallet metadata cache stale
- bridge silent → old filter-based listener; pull main and rebuild
- iPhone can't reach publisher → LAN IP changed, update env

## Audit log shape (one purchase produces ≥4 rows)

| raw_topic | reason |
| --- | --- |
| `marketplace/claim` | `claim_received:<id>:tx=...` |
| `marketplace/issued` | `eth_did_bound:claim=<id>:eth=...:tx=...` |
| `oid4vp/response` | `ok` |
| `platform/data` | `viewer_token_used:<jti>:<read_count>` |

`marketplace/claim` (subject = ETH address) and `marketplace/issued`
(subject = did:jwk) link via the same `claim_id`. That linkage is
the heart of Stage 5.

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
