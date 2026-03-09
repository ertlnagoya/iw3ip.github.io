# Mobile Viewer

## Goal

Confirm that a smartphone can browse the event feed and proceed to purchase actions when needed.

## What this page helps you understand

- what is required to open the viewer from a smartphone
- what the URL `http://<LAN_IP>:5173/mobile` actually means
- which screens confirm viewing and purchase behavior

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
The problem program asks learners to complete the function that generates `http://<LAN_IP>:5173/mobile`, so they understand exactly what the phone should open.

## Serve frontend on LAN

```bash
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

## Open from smartphone

```txt
http://<YOUR_PC_LAN_IP>:5173/mobile
```

Use MetaMask mobile in-app browser for purchase actions.

Example:

```txt
http://192.168.1.20:5173/mobile
```

## Success example

- `/mobile` opens from the smartphone
- event feed and merchandise list are visible
- purchase flow can proceed through MetaMask Mobile

## Troubleshooting

- Symptom: the smartphone cannot connect
  - Check: both devices are on the same network
  - Check: PC firewall settings allow access
  - Check: the PC LAN IP did not change
- Symptom: the purchase action does not proceed
  - Check: MetaMask Mobile in-app browser is being used
  - Check: the local chain configuration exists inside MetaMask
