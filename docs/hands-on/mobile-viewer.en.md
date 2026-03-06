# Mobile Viewer

## Goal

Confirm that a smartphone can browse the event feed and proceed to purchase actions when needed.

## Official links

- MetaMask Mobile: <https://metamask.io/>
- Vite (used by `npm run dev`): <https://vite.dev/>

## Prerequisites

- `iot-market-ui` can be started on the PC
- PC and smartphone are connected to the same LAN / Wi-Fi
- MetaMask Mobile or an equivalent wallet browser is available on the phone

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

- smartphone cannot connect:
  - verify both devices are on the same network
  - check PC firewall settings
  - verify the PC LAN IP did not change
- purchase action does not proceed:
  - use the MetaMask Mobile in-app browser
  - verify local chain configuration inside MetaMask
