# Mobile Viewer (v1)

!!! note "v1 viewing flow"
    This page covers the **v1 (current marketplace + MetaMask)** path
    for inspecting shared results. For the v2 wallet-VC flow, see
    [Marketplace × Wallet bridge](marketplace-vc-bridge.md).

## Goal

Confirm that a smartphone can browse the event feed and proceed to purchase actions when needed.

## What this page helps you understand

- what is required to open the viewer from a smartphone
- which routes `iot-market-ui` exposes (`/`, `/merchandise/[address]`,
  `/purchased/[txHash]`, `/seller`)
- which screens confirm viewing and purchase behavior

!!! info "`/mobile` is not implemented"
    The legacy exercise program (`examples/hands_on/mobile_viewer/`)
    assumes a `/mobile` route, but the current `iot-market-ui` does
    not have one. Use the actual entry points listed below.

## Common stumbling points

- the frontend must be exposed on `0.0.0.0`, not only on localhost
- devices can appear to share Wi-Fi while still being on different network segments
- purchase actions are easiest to test through the MetaMask Mobile in-app browser

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

For a first pass, these four steps are enough.

1. start `iot-market-ui` on `0.0.0.0:5173`
2. check the PC LAN IP
3. open `http://<LAN_IP>:5173/` from the smartphone (homepage)
4. confirm that the search UI is visible

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
  <p>First expose the frontend on the LAN and confirm the PC IP address that the smartphone should use.</p>
  <ol>
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

This page is a connection guide for checking Phase 1 results from a smartphone. If you only need a quick confirmation path, opening the homepage `/` is enough. For workshop use, it is better to continue through the wallet/browser notes so learners understand what changes between simple viewing and purchase actions.

## Prepare the connection baseline

## Serve frontend on LAN

```bash
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

## Check the PC LAN IP

```bash
ipconfig getifaddr en0
```

At this point, you have the URL that the smartphone should open. The next step is to check that the mobile page actually renders from another device on the same LAN.

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

Use MetaMask mobile in-app browser for purchase actions.

Example:

```txt
http://192.168.1.20:5173/
```

## Success example

- the iot-market-ui homepage opens from the smartphone
- event feed and merchandise list are visible
- purchase flow can proceed through MetaMask Mobile

At this point, the viewer path itself is working. The last step is to keep the wallet/browser requirement and common network failures clearly separated, because those are the two places where learners usually stop.

## Purchase path and troubleshooting

## Troubleshooting

- Symptom: the smartphone cannot connect
  - Check: both devices are on the same network
  - Check: PC firewall settings allow access
  - Check: the PC LAN IP did not change
- Symptom: the purchase action does not proceed
  - Check: MetaMask Mobile in-app browser is being used
  - Check: the local chain configuration exists inside MetaMask
