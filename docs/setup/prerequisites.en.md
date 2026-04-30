# Prerequisites (install tools)

Before the hands-on, install a handful of tools on your PC and confirm they work.
We add a one-line **"why this tool?"** to each item. Jargon is unpacked where it appears.

> Already have everything? Skip to the [verification checklist](#verification-checklist) and head straight to [Quickstart](quickstart.md).

## The big picture — why each tool

The hands-on builds a small **"practice city for safe data sharing"** on your laptop.
The city is made of these buildings (= processes):

| Building (process) | Role | Tool needed |
|---|---|---|
| Local blockchain | Records sale/purchase transactions | **Node.js** (`npx hardhat node`) |
| Marketplace UI | List / buy in the browser | **Node.js** + browser + **MetaMask** |
| Storage server | Holds the actual data | **Rust** (`cargo run`) |
| Distributed file store | Replicates files | **Docker** (IPFS container) |
| Publisher / mediators | Glue between everything | **Docker** |

So you need: **Docker / Node.js / Rust / Git / browser + MetaMask**.

## Required

### 1. Docker

Runs the whole city in one command.

- **macOS / Windows**: install [Docker Desktop](https://www.docker.com/products/docker-desktop/) — has a GUI for managing containers.
- **Linux**: distro-provided `docker` package, or the official install script.
- Site: <https://www.docker.com/>

> **Memory**: bump Docker Desktop's Settings → Resources → Memory to **6 GB or more**. The default 2 GB is not enough.

### 2. Node.js (ships with npm)

The JavaScript runtime, used by the marketplace UI and the local blockchain.

- Site: <https://nodejs.org/>
- Pick the **LTS** (Long Term Support) version
- Need to switch versions? [nvm](https://github.com/nvm-sh/nvm) or [volta](https://volta.sh/) help

### 3. Rust (ships with cargo)

Runs the storage server (`simple-storage`).

- Install: <https://www.rust-lang.org/tools/install>
- One-liner: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

### 4. Git

To clone the lesson repo.

- Site: <https://git-scm.com/>
- macOS ships it with Xcode CLI tools; Linux distros usually have it preinstalled.

### 5. Browser + MetaMask extension

For buying data on the marketplace (used in Part 1).

- Browser: any of Chrome / Edge / Firefox / Brave
- [MetaMask](https://metamask.io/) — install as a browser extension
- Think of it as "an Ethereum-style wallet that connects to a local chain"

## Optional (used in feature-extensions part)

Add these if you plan to do Part 2.

### Smartphone + SSI wallet app

A container for the VCs (digital credentials) you receive.

- iPhone or Android works
- Recommended: **Sphereon Wallet** (App Store / Play Store)
- What the app looks like:

| Credential list | Offer dialog |
|---|---|
| <img src="../hands-on/images/ha-ssi-wallet/01-credential-list.png" alt="credential list" width="220"> | <img src="../hands-on/images/ha-ssi-wallet/02-credential-offer.png" alt="credential offer" width="220"> |

### VS Code + Dev Containers (recommended)

For people who want to read the publisher source code.

- VS Code: <https://code.visualstudio.com/>
- Extension: [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

## Background helpful (not required)

You can do the hands-on without these, but they speed up the learning:

- **Command-line basics** (`cd`, `ls`, `cat`)
- **HTTP/REST basics** (GET / POST and JSON)
- **Git basics** (clone, branch)
- **Browser DevTools** (F12 — Network / Console)

## Verification checklist

Open a terminal and run these. They should all print a version number:

```bash
docker --version
node --version
npm --version
cargo --version
git --version
```

Expected output:

```
Docker version 25.0.x, build ...
v20.x.x
10.x.x
cargo 1.7x.x (...)
git version 2.x.x
```

If you see `command not found`, go back to the section above and reinstall.

### Confirm Docker actually runs

```bash
docker run --rm hello-world
```

You should see "Hello from Docker!".

### Clone the lesson repo

```bash
git clone --recursive https://github.com/ertlnagoya/Blockchain_IoT_Marketplace.git
cd Blockchain_IoT_Marketplace
```

> **Don't forget `--recursive`**: the repo has submodules. Without it, some directories will be empty.
> If you already cloned without `--recursive`, run `git submodule update --init --recursive`.

The initial clone takes a few minutes (the repo is large).

## You can move on if…

- [ ] All five `--version` commands print a version
- [ ] `docker run hello-world` succeeds
- [ ] You have a `Blockchain_IoT_Marketplace` directory
- [ ] (Part 1) MetaMask shows up in your browser's extensions list
- [ ] (Part 2) Sphereon Wallet is installed on your phone

All checked → head to [Quickstart](quickstart.md).

## Per-OS notes

### Windows

- Recommended: **WSL2 + Docker Desktop**.
- Reference: <https://learn.microsoft.com/windows/wsl/>
- If path-separator issues bite, prefer doing everything inside WSL2.

### macOS

- Apple Silicon (M1–M4) works. The first container build can be slower than on Intel.

### Linux

- You may need permission to talk to the Docker daemon (`docker` group membership):
  `sudo usermod -aG docker $USER && newgrp docker`

## When things go wrong

- Port conflict → see "Ports" in [Troubleshooting](../operations/troubleshooting.md)
- Docker out of memory → bump Docker Desktop's memory limit to ≥ 6 GB
- Anything else → check [FAQ](../operations/faq.md)

## Words you'll learn here

| Term | Short meaning |
|---|---|
| **Terminal** | The text-based command app (macOS: Terminal, Windows: PowerShell or WSL) |
| **Submodule** | A repo embedded inside another repo |
| **Container** | A "boxed" application that Docker runs |
| **Extension** | A small add-on inside a browser. MetaMask is one |

For the full vocab list, see the [setup mini-glossary](index.md).
