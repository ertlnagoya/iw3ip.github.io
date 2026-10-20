# Marketplace as a phone app (Stage A)

Using the Phase 2 backend as is, you use `iot-market-ui` as a home screen app (PWA: Progressive Web App) on your phone. You go from purchase through VC receipt to viewing by tapping, with no command line.

> **What you'll do**: Use iot-market-ui as a PWA on your phone and go through purchase and VC receipt
>
> **Prerequisites**: [Marketplace VC Bridge](marketplace-vc-bridge.md) (Stage 5)
>
> **What you need**: PC + smartphone (iw3ip-wallet) + MetaMask mobile
>
> **Time required**: approx. 30 min

!!! abstract "A PWA version that needs no command line"
    In this hands-on you use the backend built in Phase 2 as is, and use `iot-market-ui`
    as a home screen app on iPhone/Android.
    Without any command-line operation, you go from purchase through VC receipt to
    viewing the data by tapping on the phone screen. The PWA version on this page is called "option A", and a native
    app (React Native) with the wallet function built in is called "option B". Option B is planned
    as Stage 9.

!!! tip "Same data as Stage 0"
    In Stage A, you choose what to purchase from the list of Merchandise registered by the
    deploy script of Stage 6 case B and later. If you choose `home/event/possible_littering`
    (Merchandise #2), you can confirm that the same camera events as in Stage 0
    (see the [Hands-on overview](index.md#part-phase-stage) for the list of Stages)
    ([webcam-event-sharing](webcam-event-sharing.md))
    can be read through a purchase (PurchaseViewerVC).
    `home/event/flood_risk_high` (Merchandise #4) has no PurchaseViewerVC
    Presentation Definition registered, so you cannot get as far as viewing it.

## Goals

- Use iot-market-ui as a **PWA (Progressive Web App)**
- Confirm the existing deeplink integration (publisher / iw3ip-wallet / MetaMask Mobile)
  as a single flow that beginners can follow without getting lost
- Confirm that the purchase history is stored locally on the device and can be accessed
  later from `/my-data`

## What you'll learn

- **"ホーム画面に追加" (Add to Home Screen)** in iPhone Safari turns the IoT market into an app
- After a purchase, pressing the deeplink button on the `/purchased/[txHash]` screen
  launches iw3ip-wallet, which receives the VC and then returns to the browser
- `/my-data` lists past purchases
- The three steps of `/welcome` let even a first-time user get set up

## Common pitfalls

- PWA restrictions in iOS Safari: some APIs (push notifications, Bluetooth, and so on) are restricted.
  This is not a problem within the scope of this hands-on
- If Universal Links / App Links are not configured on the wallet side,
  the automatic return to the browser after receipt does not happen. **Even if it does not return, the flow
  is not interrupted as long as you go back to Safari manually**
- localStorage is local to the device. If you use the same wallet on another device, the history is
  not shared

## Pages to read first

- [VC Architecture Overview](../design/vc-architecture-overview.md) — the overall picture
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md) — the hands-on the purchase flow comes from
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md) — the seller-side flow

## Prerequisites

- You have completed the Stage 5 / 6 / 7 hands-ons. If you restarted Hardhat since then, first redo [Stage 6 Step E0](marketplace-vc-end-to-end.md#step-e0-start-the-prerequisite-environment-same-as-stage-5) (start the node, deploy, register the public key, recreate the bridge)
- You can start publisher + bridge + Hardhat + iot-market-ui
- iw3ip-wallet (iPhone) works
- `$HOST_IP` in commands and `<HOST_IP>` in URLs stand for the PC's LAN IP. In each terminal you use, first run `export HOST_IP=<your PC's LAN IP>` (see the [Hands-on overview](index.md#host-ip) for how to find it). Replace `<HOST_IP>` in URLs you type into a browser or phone with the same IP
- The steps assume a Mac and an iPhone, with the course repository cloned to `~/program/Blockchain_IoT_Marketplace` and the wallet to `~/program/iw3ip-wallet`

## Overview

```
[First launch]
   iPhone Safari → http://<HOST_IP>:5173/welcome
        ↓ complete the 3 steps → "マーケットへ進む" (Go to the market)
[Add to Home Screen]
   Safari menu → Add to Home Screen
   → the IW3IP app icon is added (standalone display)
[Launch]
   tap the icon → "/" of iot-market-ui opens in standalone mode
[Purchase]
   select a product → confirm in the MetaMask Mobile in-app browser → send the tx
   → automatically moves to /purchased/[txHash]
   → "ウォレットで開く" (Open in wallet) button → iw3ip-wallet launches → VC received
[History + viewing]
   hamburger menu → "購入履歴 / データを見る" (Purchase history / View data)
   → list of past purchases in /my-data
   → "閲覧チケットを取得" (Get viewing ticket) → present in the wallet
   → paste the ViewerToken → data is displayed
```

---

## Step M0. Start the PWA-enabled iot-market-ui

### What to check

- The PWA manifest and service worker are built into the Vite dev server
  (the service worker is registered only when the page is opened over https or on localhost. It is not registered at `http://<HOST_IP>:5173`, but you can still add the app to the home screen)
- When opened in iPhone Safari, it can be turned into an app with `Add to Home Screen`

### Steps

Terminal D (iot-market-ui). If the iot-market-ui from Stage 5 or 6 is still running, stop it with Ctrl+C first (while 5173 is taken, Vite starts on another port):

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only
# Rebuild the publisher and bridge from the pulled code (calls from iot-market-ui need the publisher's CORS support)
docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d

cd iot-market-ui
cat > .env.local <<EOF
VITE_RPC_URL=http://$HOST_IP:8545
VITE_PUBLISHER_URL=http://$HOST_IP:8080
EOF
npm run dev -- --host 0.0.0.0 --port 5173
```

### Expected output

```
VITE v5.x.x  ready in xxx ms
➜  Network: http://<HOST_IP>:5173/
```

When you open `http://<HOST_IP>:5173/welcome` in iPhone Safari, the 3-step
onboarding screen appears.

---

## Step M1. Add to Home Screen (PWA install)

### What to check

- **"ホーム画面に追加" (Add to Home Screen)** in the iOS Safari **share menu** creates
  an app icon
- When you launch the app, it runs in standalone mode without the Safari URL bar

### Steps (iPhone)

1. Open `http://<HOST_IP>:5173/` in Safari
2. Tap the **share button** (a square with an upward arrow) at the bottom of the screen
3. Select **"ホーム画面に追加" (Add to Home Screen)**
4. Check the name (for example, `IW3IP`) and tap **"追加" (Add)**
5. The IW3IP icon appears on the home screen → tap it to launch

### Expected result

- An icon (the default favicon) is shown on the home screen
- When launched from the icon, the app is shown in standalone mode without a URL bar or tab bar
- Page transitions in the app work as usual (`/`, `/welcome`, `/my-data`,
  `/seller`, `/merchandise/[address]`)

!!! note "On Android (Chrome)"
    `http://<HOST_IP>:5173` is not https, so the service worker is not registered
    and no install prompt appears automatically.
    Add it from the menu → "ホーム画面に追加" (Add to Home screen).

---

## Step M2. First-time onboarding `/welcome`

### What to check

- The 3-step display guides you through preparing MetaMask + iw3ip-wallet
- "もう入っています" (Already installed) takes you to the next step
- Finally, "マーケットへ進む" (Go to the market) moves to `/`

### Steps (iPhone, launched from the home screen app or Safari)

Open `http://<HOST_IP>:5173/welcome` → read the three steps shown and
tap through them.

### Expected display

```
STEP 1 / 3 - お財布アプリ (MetaMask)                  # Wallet app for payment
[MetaMask を入手する] [もう入っています →]             # [Get MetaMask] [Already installed →]

STEP 2 / 3 - 閲覧チケット用アプリ (IW3IP Wallet)       # App for viewing tickets
[もう入っています →]                                   # [Already installed →]

STEP 3 / 3 - 準備完了                                  # Ready
[マーケットへ進む] [購入履歴を見る]                    # [Go to the market] [View purchase history]
```

Hands-on participants should already have set up both apps in Stages 1–7, so
tapping "もう入っています" (Already installed) twice takes you to STEP 3. Then tap
"マーケットへ進む" (Go to the market) to move to `/`.

---

## Step M3. Purchase flow (rechecking the Stage 5/6/7 flow on mobile)

### What to check

- On the existing `/merchandise/[address]` page, using the MetaMask Mobile in-app
  browser completes the purchase and moves automatically to `/purchased/[txHash]`
- **"ウォレットで開く" (Open in wallet)** is shown on `/purchased/[txHash]`

### Steps (iPhone, MetaMask Mobile)

0. Set up MetaMask Mobile (first time only; the same settings as [Stage 5 Step 5](marketplace-vc-bridge.md#step-5-configure-metamask))
    - Add a network: Network Name `Hardhat localhost`, RPC URL `http://<HOST_IP>:8545`, Chain ID `31337`, Currency `ETH`
    - After adding it, switch to `Hardhat localhost`
    - Account menu → "Import private key" → enter the private key of Account #2 (noted in Stage 5 Step 1). Only an account whose PubKey is registered (Stage 6 Step E0; if you have restarted Hardhat since, redo E0) can purchase
1. Open MetaMask Mobile → in the built-in browser (the globe icon at the bottom right), open
   `http://<HOST_IP>:5173/merchandise/0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0`
   (Merchandise #0, created by the deploy script. Do not use #1, which may already have been purchased in Stage 5)
2. (Optional) Hamburger menu → tap "Connect your Wallet" to connect MetaMask.
   On a phone-width screen the "Connect your wallet!" button at the top right is not shown. The button in step 3 connects automatically, so you can skip this step
3. Press the "Purchase!" button and **Confirm** on the MetaMask side
4. After a few seconds, the page moves automatically to `/purchased/<txHash>`, and a QR code and the
   "ウォレットで開く" (Open in wallet) button are shown

This purchase history is saved in the `localStorage` of the MetaMask Mobile in-app browser you used.
Continue Steps M4 and M5 in this in-app browser.

!!! note "Alternative using a PC browser"
    If operating MetaMask Mobile on the iPhone is difficult, you may open the same URL with the
    MetaMask extension in Chrome on a PC and purchase there. After the purchase, the page likewise
    moves to `/purchased/[txHash]`. However, the `/my-data` history is stored only on the
    device used for the purchase.

---

## Step M4. Receive the VC in iw3ip-wallet

### What to check

- The deeplink on `/purchased/[txHash]` launches the wallet → confirmation screen
- When you approve, the PurchaseViewerVC is stored in the wallet

### Steps (iPhone)

On `/purchased/<txHash>`, tap **"ウォレットで開く" (Open in wallet)**.

iw3ip-wallet launches and shows the "IW3IP Purchase Viewer (Event-only)" confirmation screen (shown as "IW3IP 購入閲覧（Tier 1 / イベントのみ）" in a Japanese-language wallet) → approve.

### Expected result

- A new entry in the wallet's VC list
- The claims include `merchandise_address`, `tx_hash`, `buyer_eth_addr`, `dataset_id`
- After approving, switch apps to go back to MetaMask Mobile (its in-app browser)

---

## Step M5. History and data viewing in `/my-data`

### What to check

- The purchase from Step M3 is stored in `localStorage` and listed in `/my-data`
- Tapping "閲覧チケットを取得" (Get viewing ticket) opens the wallet presentation screen
- After presenting, pasting the ViewerToken into the form on `/my-data` shows the JSON

### Steps

1. In the MetaMask Mobile in-app browser, open `http://<HOST_IP>:5173/my-data`
   (or use the hamburger menu → **"購入履歴 / データを見る" (Purchase history / View data)**)
2. The purchase from Step M3 is shown (dataset_id, merchandise, tx, date and time)
3. Tap **"閲覧チケットを取得" (Get viewing ticket)** → on the page that opens, tap **"ウォレットで開く" (Open in wallet)** → choose the PurchaseViewerVC in the wallet and present it
4. On the PC, check the token with `docker logs iw3ip-publisher 2>&1 | grep viewer_token_issued | tail -1`
   (`2>&1` is needed because the publisher logs to stderr)
5. Go back to `/my-data` and tap **"このデータを見る" (View this data)** for that purchase to show the "ViewerToken (Bearer)" field
6. Paste that token into the "ViewerToken (Bearer)" field
7. **"取得実行" (Fetch)** shows the JSON

!!! note "The ViewerToken is valid for 60 seconds"
    Do steps 3 to 7 without a break. Send the token you checked on the PC to the iPhone with Universal Clipboard or AirDrop.
    If you see `viewer_token_expired`, start again from step 3.

### Expected JSON

```json
{
  "dataset_id": "home/env/temperature",
  "count": ...,
  "read_count": 1,
  "seller_did": "unknown",
  "allowed_views": ["event"],
  "rows": [...]
}
```

Merchandise #0 has no seller registration, so `seller_did` is `"unknown"`.
`seller_did` contains the seller's `did:jwk:...` only when you purchase a Merchandise newly
deployed and registered in Stage 7 Steps S3–S4, without recreating the publisher since then
(the seller binding lives in the publisher's memory and is lost on recreation; the Merchandise
used in Stage 7 has already been purchased and cannot be bought again).

!!! note "Current limitation (to be resolved in Stage 9)"
    Having to paste the ViewerToken manually is the current usability
    issue. Completing the flow by tapping alone requires a native app with a
    built-in OID4VP client (option B, Stage 9).

---

## Completion matrix

| Step | What to check | Example output |
| --- | --- | --- |
| M0 | The PWA shell works | Check `manifest.webmanifest` in Safari's DevTools |
| M1 | Home screen icon | `IW3IP` icon on the iPhone home screen |
| M2 | 3-step onboarding | "マーケットへ進む" (Go to the market) moves to `/` |
| M3 | Transition after purchase | QR + deeplink on `/purchased/[txHash]` |
| M4 | VC received in the wallet | Claims include merchandise/tx/eth |
| M5 | History shown and data fetched in `/my-data` | JSON containing `dataset_id` and `rows` |

---

## Troubleshooting

### A. Add to Home Screen does not appear as an option

**Cause**: You are using a browser other than Safari, or fetching the manifest failed.

**Fix**:

- Confirm that you opened the page in Safari (the iOS default)
- Open `http://<HOST_IP>:5173/manifest.webmanifest` directly and check that JSON is
  displayed
- Confirm that you opened the page with the LAN IP, not `http://localhost`

### B. The app launches from the icon but the browser bar is shown

**Cause**: `display: "standalone"` in the manifest is not applied, or
Safari's cache.

**Fix**: Long-press the home screen icon to delete it → add it again in Safari.

### C. Tapping "ウォレットで開く" (Open in wallet) does not launch the wallet

**Cause**: The `openid-credential-offer://` scheme is not registered by the wallet.

**Fix**:

- Check that iw3ip-wallet is already running
- Check troubleshooting item C of Stage 5 (wallet cache, Metro bundler)

### D. The history in `/my-data` is empty

**Cause**: One of the following.

- Writing to `localStorage` on `/purchased/[txHash]` failed at purchase time (private browsing mode, for example)
- The publisher is old and lacks CORS support, so `/marketplace/claim` from `/purchased/[txHash]` failed (the history is saved only when the claim succeeds)

**Fix**:

- Turn off private browsing in Safari
- After `git pull`, rebuild the publisher with `cd ~/program/Blockchain_IoT_Marketplace && docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d`
- Make another purchase and check `/my-data` again

### E. Other

See also [Stage 5 troubleshooting](marketplace-vc-bridge.md#troubleshooting)
and the troubleshooting section of [Stage 7](marketplace-seller-vc.md).

---

## Limitations (to be resolved in Stage 9)

- **Pasting the ViewerToken**: Automatic retrieval requires an implementation on the wallet side that
  calls the publisher API (option B = Stage 9)
- **Switching between apps**: You move back and forth between three apps (PWA / MetaMask / iw3ip-wallet).
  Completing everything in one app is option B
- **Offline support**: It is a PWA, but the service worker is pass-through. Chain
  reads are always online
- **iOS notifications / background processing**: Limited by the iOS PWA restrictions

## Related

- [VC Architecture Overview](../design/vc-architecture-overview.md)
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md)
- [Stage 6: 4-VC end-to-end](marketplace-vc-end-to-end.md)
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md)
