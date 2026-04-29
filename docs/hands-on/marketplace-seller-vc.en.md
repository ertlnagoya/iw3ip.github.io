# Seller VC for marketplace identity (Stage 7)

!!! abstract "5th VC kind: SellerVC"
    Adds a layer that proves **who is selling** a dataset on the
    marketplace. Same pattern as Stages 1–6: present → token → API.
    Spec: [SellerVC design doc](../design/seller-vc-spec.md).

> The Japanese page ([marketplace-seller-vc.md](marketplace-seller-vc.md))
> is the canonical step-by-step. This page summarizes the structure.

## What you'll learn

- 5th VC kind (SellerVC) with `seller_id` + `licensed_datasets`
- SellerToken (24 h, multi-use) used only by `/marketplace/register`
- Optional on-chain verification of `Merchandise.getOwner() == seller_eth_addr`
  via `MARKETPLACE_HARDHAT_RPC` env
- New audit row `marketplace/seller_registered`
- Buyer's `/platform/data?merchandise=` response now carries `seller_did`

## Step overview

| Step | What it verifies |
| --- | --- |
| S0 | Publisher up with C2+C3 code; metadata exposes SellerVC |
| S1 | Seller wallet receives a SellerVC (seller_id + licensed_datasets) |
| S2 | Presentation yields a SellerToken (24 h) |
| S3 | Seller deploys a Merchandise + registers in IoTMarket (Hardhat console) |
| S4 | `POST /marketplace/register` binds merchandise → seller_did; audit row |
| S5 | Buyer's `/platform/data?merchandise=` returns `seller_did` |
| S6 | iot-market-ui `/seller` page does the same as S2-S4 from the browser |

## Owner verify modes (spec Q2)

`MARKETPLACE_HARDHAT_RPC`:
- unset → `owner_verify=skipped` (default in tests/dev)
- set + RPC OK → `owner_verify=verified` (success) or 403 `owner_mismatch`
- set + RPC fail → 502 `chain_rpc_failed` (fail-closed)

## Common pitfalls (from e2e validation)

| Symptom | Root cause | Fix |
| --- | --- | --- |
| Hardhat console rejects multi-line `try`/`catch` | REPL evaluates one line at a time | Collapse to a one-liner |
| `getCode(addr)` returns `"0x"` after Hardhat restart | Restart wipes on-chain state | Re-run `deployMerchandiseWithIoTMarket.ts`; use script-deployed Merchandises only |
| Purchase fails with "function returned an unexpected amount of data" | PubKey contract is gone or buyer has no PubKey entry | Re-register with `pubKey.registerKey("[...]")` for the affected account |
| Bridge silent on a new Purchase | Bridge cursor stuck on old block after Hardhat restart | `docker compose ... up -d --force-recreate bridge` |
| Wallet error "Retrieving an access token ... failed with status: 400" | Wallet replayed an already-consumed `pre_authorized_code` | Discard old deeplink, take a fresh one from the latest `bridge: claim ok` log |
| MetaMask "invalid chainId" | Stale nonce/chainId cache after Hardhat restart | Settings → Advanced → Clear activity tab data |

See also Stage 5 troubleshooting for issues that span both stages.

## Limits / future

- on-chain registration guard (Solidity-side) is Stage 8+
- eth_addr ↔ did:jwk binding is bridge-trusted MVP (no EIP-712)
- `/marketplace/register` trusts the dataset_id in the POST body; future
  hardening reads it back from `Merchandise.getAllAdditionalInfo()`

## Related

- [SellerVC design spec](../design/seller-vc-spec.md)
- [Marketplace × Wallet bridge (Stage 5)](marketplace-vc-bridge.md)
- [Marketplace VC end-to-end (Stage 6)](marketplace-vc-end-to-end.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
