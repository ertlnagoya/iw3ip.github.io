# Quickstart (run the stack)

Using the tools you installed in [Prerequisites](prerequisites.md), start the full hands-on environment together.
Estimated time: **15–30 min** (the first run pulls Docker images and runs `npm install`).

> This starts **6 separate processes**, each in its own terminal.
> Follow the steps in order.

## What you are starting — overview

```
┌─────────────────────────────────────────────────────────────┐
│  Browser (Chrome etc.) — http://localhost:5173               │
│  └─ MetaMask handles the purchase                            │
└──────────────────────────┬──────────────────────────────────┘
                           │
            ┌──────────────▼──────────────┐
            │  iot-market-ui (1)           │ ← Svelte/Vite frontend
            │  Node.js                     │
            └──────┬───────────────┬───────┘
                   │               │
       ┌───────────▼─────┐   ┌────▼──────────────┐
       │ iot-market (2)  │   │ simple-storage (3)│
       │ Hardhat node    │   │ Rust              │
       │ + contracts     │   │ data store        │
       └─────────────────┘   └──────┬────────────┘
                                    │
                            ┌───────▼───────┐
                            │ ipfs (4)      │
                            │ + PostgreSQL  │
                            │ Docker        │
                            └───┬───────┬───┘
                                │       │
                  ┌─────────────▼──┐ ┌─▼─────────────────┐
                  │ mediator-owner │ │ mediator-buyer    │
                  │ (5) Rust       │ │ (6) Rust          │
                  └────────────────┘ └───────────────────┘
```

(1) frontend, (2) blockchain, (3) data store, (4) distributed file store, (5)(6) mediators.

## Startup order — top to bottom

> **Open six terminals.** Each process stays running in its own terminal.
> macOS Terminal: ⌘+T. VS Code: "Split Terminal".

### 0. Move into the lesson repo

All commands run from the repo root (`Blockchain_IoT_Marketplace/`).

```bash
cd Blockchain_IoT_Marketplace
```

### 1. Start the local blockchain

Terminal 1:

```bash
cd iot-market
npx hardhat node
```

Expected output (excerpt):

```
Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/

Accounts
========
WARNING: These accounts ... use only for testing.

Account #0: 0xf39F... (10000 ETH)
Private Key: 0xac09...
...
```

> **Don't close this terminal** — the chain has to keep running.
>
> The displayed **Account #0 + Private Key** will be imported into MetaMask shortly. Keep them visible.

### 2. Deploy the contracts

Terminal 2:

```bash
cd iot-market
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
```

On success, contract addresses are printed.

### 3. Start the frontend

Terminal 3:

```bash
cd iot-market-ui
npm install   # first time only
npm run dev
```

Expected output:

```
  VITE v4.x  ready in 500 ms
  ➜  Local:   http://localhost:5173/
```

Open <http://localhost:5173> in a browser. If the marketplace UI appears, the frontend is running.

### 4. Start the storage server

Terminal 4:

```bash
cd simple-storage
cargo run
```

The first run takes a few minutes to compile Rust crates.
When `Listening on ...` appears, it has started.

### 5. Start IPFS + PostgreSQL (Docker)

Terminal 5:

```bash
cd ipfs
docker compose up -d
```

`-d` runs detached. Check status:

```bash
docker compose ps
```

If the State of both `ipfs` and `postgres` is **Up**, they are running.

### 6. Start the mediators (owner + buyer)

Terminal 6 (owner):

```bash
cd mediator-owner
cargo run -- settings/owner_1.yaml
```

Terminal 7 (buyer):

```bash
cd mediator-buyer
cargo run --bin mediator-b
```

When both show `Listening on ...`, they have started.

## Verify operation

1. Open <http://localhost:5173>
2. Set up **MetaMask** (next section)
3. When the marketplace listings appear, you are ready to move on to hands-on Part 1

## MetaMask local-chain setup

Connect MetaMask to the local Hardhat chain.

1. Open the MetaMask extension
2. Network selector → **Add custom network**
3. Fill in:
   - Network name: `Hardhat Local`
   - RPC URL: `http://localhost:8545`
   - Chain ID: `31337`
   - Currency: `ETH`
4. Save and switch to the new network

Import the **Account #0 Private Key** from step 1:

1. MetaMask top-right icon → **Import account**
2. Paste the private key (`0x` + 64 hex)
3. If the imported account shows a balance of **10000 ETH**, the import succeeded

Reference (network add screen):

![MetaMask Network setup](../assets/how_to_network.png)

## You can move on if…

- [ ] All terminals 1–7 are alive (no crash)
- [ ] <http://localhost:5173> renders the marketplace UI
- [ ] MetaMask is on the `Hardhat Local` network
- [ ] The MetaMask account shows `10000 ETH`

All checked → start [hands-on Part 1](../hands-on/index.en.md).

## Simpler alternative: a minimal stack with docker compose only

If opening seven terminals is cumbersome, you can use Docker Compose to start just the publisher-related components.
For the shortest path focused on ingesting and observing data without physical devices, see the following page.

- [Home Assistant Demo Simulator sample](../hands-on/ha-demo-simulator.md)

This alone is enough to try the first section of Part 1.

## Common stumbles

| Symptom | Cause | Fix |
|---|---|---|
| `npx: command not found` | Node.js missing | revisit Node.js in [Prerequisites](prerequisites.md) |
| `cargo: command not found` | Rust missing | revisit Rust in [Prerequisites](prerequisites.md) |
| `port 5173 already in use` | Another app holds the port | kill it or pick a different port |
| `error connecting to docker` | Docker Desktop not running | start Docker Desktop |
| MetaMask "Nonce too high" | Chain restarted out of sync | Settings → Advanced → Reset Account |
| MetaMask rejects `localhost:8545` | Typo in RPC URL | make sure `http://` prefix is there |

Anything else: [Troubleshooting](../operations/troubleshooting.md), [FAQ](../operations/faq.md).

## Words you'll learn here

| Term | Short meaning |
|---|---|
| **Hardhat** | Tool that runs an Ethereum-compatible local chain |
| **Contract** | Code that runs on the blockchain |
| **Deploy** | Place a contract onto the chain |
| **Chain ID** | Numeric ID of a network (Hardhat's default is 31337) |
| **Vite** | Fast frontend dev server |

More: [Hardhat Basics](../foundations/hardhat-basics.md), [SSI/DID/VC Basics](../foundations/ssi-did-vc-basics.md).
