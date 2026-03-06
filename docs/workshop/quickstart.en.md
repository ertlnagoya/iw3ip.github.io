# Quickstart

Goal: boot the whole stack first.

## What to know before starting

- The goal here is not to understand every subsystem first, but to boot the full stack and observe the flow.
- If you get stuck, check [Troubleshooting](../operations/troubleshooting.md).
- If blockchain terms are unfamiliar, read [Hardhat Basics](../foundations/hardhat-basics.md) first.

## Startup order

1. `iot-market` (chain + deploy)
2. `iot-market-ui`
3. `simple-storage`
4. `ipfs`
5. `mediator-owner`
6. `mediator-buyer`

## Core commands

### 1. Start local blockchain

`iot-market` uses Hardhat to start a local Ethereum-like development chain.  
Reference: <https://hardhat.org/docs>

```bash
cd iot-market
npx hardhat node
```

In another terminal:

### 2. Deploy contracts

```bash
cd iot-market
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
```

### 3. Start frontend

```bash
cd iot-market-ui
npm run dev
```

### 4. Start storage server

```bash
cd simple-storage
cargo run
```

### 5. Start IPFS and PostgreSQL

```bash
cd ipfs
docker compose up -d
```

### 6. Start owner-side process

```bash
cd mediator-owner
cargo run -- settings/owner_1.yaml
```

### 7. Start buyer-side process

```bash
cd mediator-buyer
cargo run --bin mediator-b
```

## Success check

- Frontend opens at `http://localhost:5173`
- Merchandise list is available
- No critical process exits with error

## Common beginner issues

- `npx: command not found`
  - Node.js is missing or not in PATH. Check <https://nodejs.org/>.
- `cargo: command not found`
  - Rust is not installed. Check <https://www.rust-lang.org/tools/install>.
- Cannot purchase from MetaMask
  - verify local network configuration and test accounts
  - reference: <https://metamask.io/>

![MetaMask network setup](../assets/how_to_network.png)
