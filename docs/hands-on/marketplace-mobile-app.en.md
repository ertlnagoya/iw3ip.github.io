# Marketplace mobile app (Stage A — minimal PWA)

!!! abstract "PWA experience for laypeople"
    Wraps the existing `iot-market-ui` as a Progressive Web App so the
    Phase 2 backend can be exercised entirely through smartphone taps —
    no command line needed. Pairs with the Stage 8 trust-model work and
    paves the way for Stage 9 (case B / fully native RN app).

> The Japanese page ([marketplace-mobile-app.md](marketplace-mobile-app.md))
> is the canonical step-by-step. This page summarizes the structure.

## What this hands-on covers

- iot-market-ui as an installable PWA (Add to Home Screen on iOS / Chrome install on Android)
- Polished mobile flow for the existing deeplink integration
  (publisher / iw3ip-wallet / MetaMask Mobile)
- Local purchase history surfaced via `/my-data`, populated from
  `localStorage` when `/purchased/[txHash]` succeeds

## Step overview

| Step | What it verifies |
| --- | --- |
| M0 | iot-market-ui Vite dev server serves the manifest + service worker |
| M1 | Add to Home Screen on iOS Safari produces a standalone app icon |
| M2 | `/welcome` 3-step onboarding completes |
| M3 | Existing purchase flow (Stage 5/6/7) works in MetaMask Mobile in-app browser |
| M4 | iw3ip-wallet receives the PurchaseViewerVC via the deeplink |
| M5 | `/my-data` lists the purchase from `localStorage` and fetches data with a pasted ViewerToken |

## Limits

- **ViewerToken paste**: still manual; the auto-flow requires a wallet
  with embedded publisher client (Stage 9 / case B)
- **Three apps**: the PWA, MetaMask Mobile, iw3ip-wallet. A single-app
  experience needs Stage 9
- **iOS PWA constraints**: no push notifications / Bluetooth in PWA mode
- **localStorage purchase history is per-device**: switching devices
  loses the visible history (the on-chain Purchase record is preserved)

## Related

- [VC Architecture Overview](../design/vc-architecture-overview.md)
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md)
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md)
