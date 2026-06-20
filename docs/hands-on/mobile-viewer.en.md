# Mobile Viewer (v1)

Browse v1-marketplace data (current + MetaMask + encrypted IPFS) from a phone browser. The v2 VC flow is a separate page.

> **What you'll do**: Open purchased data on a phone browser
>
> **Prerequisites**: v1 stack running ([Quickstart](../setup/quickstart.en.md))
>
> **What you need**: PC + smartphone on the same network
>
> **Time**: ~20 min

!!! note "v1 viewing flow"
    This page covers the **v1 (current marketplace + MetaMask)** path
    for inspecting shared results. For the v2 wallet-VC flow, see
    [Marketplace × Wallet bridge](marketplace-vc-bridge.md).

## Goal

Confirm from a smartphone that you can browse the event feed and purchase the data you need.

## What this page covers

- what is required to connect to the viewer UI from a smartphone
- the roles of the routes that iot-market-ui exposes (`/`, `/merchandise/[address]`,
  `/purchased/[txHash]`, `/seller`)
- which screens to check for viewing and for purchase

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

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/mobile_viewer/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/mobile_viewer/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/mobile_viewer/README.md)

This exercise builds the smartphone access URL from the PC's LAN IP.
The legacy problem program assumes `/mobile`; in the current code
read it as the homepage `/` instead.

## Shortest path

For a first pass, these five steps are enough.

1. prepare `iot-market-ui/.env.local` and set the endpoint for your setup (`127.0.0.1` for PC-only, the LAN IP for a phone)
2. start `iot-market-ui` on `0.0.0.0:5173`
3. check the PC LAN IP
4. open `http://<LAN_IP>:5173/` from the smartphone (homepage)
5. confirm that the search UI is visible

Branches after that:

- **Specific item**: open `http://<LAN_IP>:5173/merchandise/<address>` directly
  (use a Merchandise address from the Hardhat deploy output)
- **Purchase path**: open the same URL in MetaMask Mobile
- **VC-mediated path (v2)**: jump to [Stage 5](marketplace-vc-bridge.md);
  the iot-market-ui exposes `/purchased/[txHash]` and `/seller` for the
  wallet-aware flows
- If the connection fails early: jump to the troubleshooting section first

## Process table of contents

<details class="iw3ip-toc-details" open>
  <summary>Preparation: expose the frontend and confirm the LAN IP</summary>
  <p>First configure the frontend endpoint (.env.local), expose the frontend on the LAN, and confirm the PC IP address that the smartphone should use.</p>
  <ol>
    <li><a href="#env-local">Configure the frontend endpoint (.env.local)</a></li>
    <li><a href="#serve-frontend-on-lan">Serve frontend on LAN</a></li>
    <li><a href="#check-the-pc-lan-ip">Check the PC LAN IP</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 1: open the mobile viewer from the smartphone</summary>
  <p>Next, open the homepage `/` (or a specific `/merchandise/[address]`) from the smartphone and confirm that the UI is visible.</p>
  <ol>
    <li><a href="#open-from-smartphone">Open from smartphone</a></li>
    <li><a href="#success-example">Success example</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 2: purchase path and troubleshooting</summary>
  <p>Finally, if needed, test the wallet-based purchase path and review the most common reasons why the phone cannot connect.</p>
  <ol>
    <li><a href="#open-from-smartphone">Open from smartphone</a></li>
    <li><a href="#troubleshooting">Troubleshooting</a></li>
  </ol>
</details>

## How to read this page

This page is a procedure for checking Phase 1 results from a smartphone. For a quick pass, opening the URL and viewing the list is enough. As a hands-on, it is better to confirm both "it runs on the same LAN" and "the wallet-based purchase entry point".

## Prepare the connection baseline

## 0. Configure the frontend endpoint (.env.local) { #env-local }

`iot-market-ui` reads the blockchain (RPC) and publisher endpoints from `iot-market-ui/.env.local`. If that file is missing or points at the wrong host, the merchandise page returns **HTTP 500**. Copy the template first and match it to your setup.

```bash
cd iot-market-ui
cp .env.example .env.local
```

Edit `.env.local` and set the host for your scenario.

- **PC-only check**: keep `127.0.0.1`.

    ```
    VITE_RPC_URL=http://127.0.0.1:8545
    VITE_PUBLISHER_URL=http://127.0.0.1:8080
    ```

- **From a phone**: replace it with the PC's LAN IP (see step 2).

    ```
    VITE_RPC_URL=http://<YOUR_PC_LAN_IP>:8545
    VITE_PUBLISHER_URL=http://<YOUR_PC_LAN_IP>:8080
    ```

!!! warning "Expose the Hardhat node on the LAN for phone access"
    The default `npx hardhat node` only listens on `127.0.0.1`, so a phone or
    MetaMask Mobile cannot reach it. For phone access, restart the node with
    `npx hardhat node --hostname 0.0.0.0` and allow `8545` through the PC firewall.

Restart the frontend (`npm run dev`) after editing `.env.local`.

## Serve frontend on LAN

```bash
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

## Check the PC LAN IP

```bash
ipconfig getifaddr en0
```

You can now build the URL to open from the smartphone. Next, open the page on the smartphone to confirm it.

## Confirm the mobile viewer from the smartphone

## Open from smartphone

```txt
http://<YOUR_PC_LAN_IP>:5173/
```

For an individual item (Merchandise contract address from the
Hardhat deploy):

```txt
http://<YOUR_PC_LAN_IP>:5173/merchandise/<merchandise_address>
```

Use MetaMask mobile in-app browser for purchase actions. Add the local chain
network to MetaMask Mobile beforehand:

- RPC URL: `http://<YOUR_PC_LAN_IP>:8545` (e.g. `http://192.168.1.20:8545`)
- Chain ID: `31337` / Currency symbol: `ETH`

If MetaMask rejects the network, confirm the node was started with
`--hostname 0.0.0.0` and that both `.env.local` and the MetaMask RPC use the
**PC's LAN IP, not 127.0.0.1**.

Example:

```txt
http://192.168.1.20:5173/
```

## Success example

- the iot-market-ui homepage opens from the smartphone
- event feed and merchandise list are visible
- purchase flow can proceed through MetaMask Mobile

The connection to the viewer UI is confirmed. If needed, try the purchase action; if you cannot connect, check the troubleshooting items below.

## Purchase path and troubleshooting

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
