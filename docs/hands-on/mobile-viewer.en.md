# Open the marketplace and buy on a phone (v1)

Open the v1 marketplace (purchase with MetaMask; data is handed over through simple-storage on :3000) in a phone browser and go as far as buying an item. The v2 VC flow is a separate page.

> **What you'll do**: Open the marketplace screen in a phone browser and walk through a purchase
>
> **Prerequisites**: v1 stack running ([Quickstart](../setup/quickstart.en.md))
>
> **What you need**: PC + smartphone on the same network
>
> **Time required**: approx. 20 min

!!! note "Open and buy on v1"
    This page covers opening the marketplace and buying from a phone on
    **v1 (current marketplace + MetaMask)**. For the v2 wallet-VC phone flow, see
    [Marketplace as a phone app (Stage A)](marketplace-mobile-app.md).

## Goal

Confirm that you can open the marketplace from a smartphone and buy an item.

## What this page covers

- what is required to reach the marketplace UI from a smartphone
- the roles of the routes that iot-market-ui exposes (`/`, `/merchandise/[address]`,
  `/purchased/[txHash]`, `/seller`)
- which screens to check when opening and when buying

!!! info "`/mobile` is not implemented"
    The legacy exercise program (`examples/hands_on/mobile_viewer/`)
    assumes a `/mobile` route, but the current `iot-market-ui` does
    not have one. Use the actual entry points listed below.

## Common issues

- if the PC side is not exposed on `0.0.0.0`, the smartphone cannot reach it
- even on what looks like the same Wi-Fi, a split network prevents the connection
- purchase actions do not progress well unless done through the MetaMask Mobile browser

## Official links

- MetaMask Mobile: <https://metamask.io/>
- Vite (used by `npm run dev`): <https://vite.dev/>

## Prerequisites

- `iot-market-ui` can be started on the PC
- PC and smartphone are connected to the same LAN / Wi-Fi
- MetaMask Mobile or an equivalent wallet browser is available on the phone

## Exercise Programs

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/mobile_viewer/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/mobile_viewer/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/mobile_viewer/README.md)

This exercise builds the smartphone access URL from the PC's LAN IP.
The legacy problem program assumes `/mobile`; in the current code
read it as the homepage `/` instead.

## Shortest path

For a first pass, these five steps are enough.

1. prepare `iot-market-ui/.env.local` and set the endpoint for your setup (`127.0.0.1` for PC-only, the LAN IP for a phone)
2. start `iot-market-ui` on `0.0.0.0:5173`
3. check the PC LAN IP
4. open `http://<HOST_IP>:5173/` from the smartphone (homepage)
5. confirm that the search UI is visible

Branches after that:

- **Specific item**: open `http://<HOST_IP>:5173/merchandise/<address>` directly
  (use the `Product deployed: 0x…` address shown in the `mediator-owner` terminal (created by running the USB webcam sample in mock mode as in §1.3 of the [Hands-on overview](index.md); the 5 items deployed in step 2 of the Quickstart deliver no data when bought))
- **Purchase path**: open the same URL in MetaMask Mobile
- **VC-mediated path (v2)**: go to [Marketplace as a phone app (Stage A)](marketplace-mobile-app.md)
- If the connection fails early: check [Troubleshooting](#troubleshooting) first

## Prepare the connection baseline

### 0. Configure the frontend endpoint (.env.local) { #env-local }

`iot-market-ui` reads the blockchain (RPC) and publisher endpoints from `iot-market-ui/.env.local`. If it points at the wrong host, the merchandise page returns **HTTP 500** (if the file is missing, `127.0.0.1` is used for RPC and `localhost` for the publisher, so the phone cannot connect). Copy the template first and match it to your setup.

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market-ui
cp .env.example .env.local
```

Edit `.env.local` and set the host for your scenario.

- **PC-only check**: keep `127.0.0.1`.

    ```
    VITE_RPC_URL=http://127.0.0.1:8545
    VITE_PUBLISHER_URL=http://127.0.0.1:8080
    ```

- **From a phone**: replace it with the PC's LAN IP. Find it with `ipconfig getifaddr en0` on macOS (the same command as in step 2).

    ```
    VITE_RPC_URL=http://<HOST_IP>:8545
    VITE_PUBLISHER_URL=http://<HOST_IP>:8080
    ```

!!! warning "Expose the Hardhat node on the LAN for phone access"
    The default `npx hardhat node` only listens on `127.0.0.1`, so a phone or
    MetaMask Mobile cannot reach it. For phone access, restart the node with
    `npx hardhat node --hostname 0.0.0.0` and allow `8545` through the PC firewall.

    Restarting the node resets the chain and the deployed contracts are gone.
    After restarting, redo step 2 (deploy) of the [Quickstart](../setup/quickstart.en.md)
    and restart `mediator-owner` and `mediator-buyer` as well. Earlier item addresses no longer work.
    Run the USB webcam (or HUSKYLENS2) mock again and use the newly shown `Product deployed` address.

Restart the frontend (`npm run dev`) after editing `.env.local`.

### 1. Start the PC side on the LAN

First stop the `npm run dev` running in Terminal 3 of the [Quickstart](../setup/quickstart.en.md) with Ctrl+C (it uses the same port 5173; otherwise the new server starts on another port). Then run this in the terminal you moved into `iot-market-ui` in step 0.

```bash
npm run dev -- --host 0.0.0.0 --port 5173
```

### 2. Check the PC LAN IP

Run this in another terminal; the step 1 terminal is still running `npm run dev`.

```bash
ipconfig getifaddr en0
```

`ipconfig getifaddr en0` is a macOS command. On Windows use `ipconfig`; on Linux use `ip addr`. Replace `<HOST_IP>` in the following steps with the IP address you found here (the [Hands-on overview](index.md#host-ip) also lists how to check it on each OS).

## Open the marketplace from the smartphone

### 3. Open from the smartphone

```txt
http://<HOST_IP>:5173/
```

Example (homepage):

```txt
http://192.168.1.20:5173/
```

To open an individual item directly (use the `Product deployed: 0x…` address shown in the `mediator-owner` terminal (created by running the USB webcam sample in mock mode as in §1.3 of the [Hands-on overview](index.md); the 5 items deployed in step 2 of the Quickstart deliver no data when bought; an item already bought in §1.3 cannot be bought again)):

```txt
http://<HOST_IP>:5173/merchandise/<merchandise_address>
```

### 4. Notes on purchasing

- To purchase from the smartphone, use the MetaMask Mobile in-app browser
- Add the local chain network to MetaMask Mobile beforehand:
    - RPC URL: `http://<HOST_IP>:8545` (e.g. `http://192.168.1.20:8545`)
    - Chain ID: `31337` / Currency symbol: `ETH`
- Also import the **Account #2 Private Key** into MetaMask Mobile and buy with that account (the same key as in [Quickstart](../setup/quickstart.md#metamask-local-chain-setup)). Only Account #1 (`mediator-owner`) and Account #2 (`mediator-buyer`) have a registered public key. The buyer-side mediator runs as Account #2, so buying from any other account either reverts or never delivers the data
- If MetaMask rejects the network, confirm the node was started with `--hostname 0.0.0.0` and that both `.env.local` and the MetaMask RPC use the **PC's LAN IP, not 127.0.0.1**

### 5. What to check

- The iot-market-ui search UI or a merchandise page opens on the smartphone
- The purchase action runs on an individual Merchandise page
- After purchase, the page moves to `/purchased/[txHash]`. This v2 screen calls the publisher's `/marketplace/claim` to show a QR code for receiving a VC. On v1 the publisher is not running, so it shows "エラー: ..." (error), but the purchase itself has completed
- Confirm the purchase in the PC's `mediator-buyer` log: `You bought a product of ...` followed by `File downloaded successfully` and `Verification result: true` means the data arrived from simple-storage (:3000)

## Success example

- The iot-market-ui homepage opens from the smartphone
- You can proceed from a merchandise page to the purchase flow
- You can reach the purchase confirmation screen through MetaMask Mobile

- The `mediator-buyer` log shows the purchase and download lines

If you cannot connect or the purchase does not proceed, check the troubleshooting items below.

## Troubleshooting

- Symptom: the smartphone cannot connect
  - Check: both devices are on the same network
  - Check: PC firewall settings allow access
  - Check: the PC LAN IP did not change
- Symptom: the merchandise page returns HTTP 500
  - Check: `VITE_RPC_URL` in `iot-market-ui/.env.local` points at the running Hardhat node (`8545`)
  - Check: the endpoint is not someone else's IP or a wrong port such as `8546`
  - See the "Merchandise page returns 500" item in [Troubleshooting](../operations/troubleshooting.md)
- Symptom: the purchase action does not proceed / cannot add the network to MetaMask
  - Check: MetaMask Mobile in-app browser is being used
  - Check: the MetaMask RPC and the node's exposure match (`--hostname 0.0.0.0` + LAN IP)
  - See the "Cannot add the network to MetaMask" item in [Troubleshooting](../operations/troubleshooting.md)
- Symptom: the purchase transaction fails (revert) / `mediator-buyer` never prints `You bought a product of`
  - Check: the account selected in MetaMask Mobile is Account #2 (other accounts have no registered public key and cannot buy)
  - Check: whether you already bought that item with the same account (an item bought in §1.3 cannot be bought again; use a `Product deployed` address you have not bought yet)
  - Fix: import the same Account #2 Private Key as in [Quickstart](../setup/quickstart.md#metamask-local-chain-setup) and buy again with that account
  - See the "Purchase does not succeed" item in [Troubleshooting](../operations/troubleshooting.md)
