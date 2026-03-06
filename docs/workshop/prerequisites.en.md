# Prerequisites

This page is the starting point for students setting up the environment for the first time.  
Install each tool from its official website, then verify the version commands locally.

## Required

- Docker
  - Official site: <https://www.docker.com/>
  - On macOS / Windows, [Docker Desktop](https://www.docker.com/products/docker-desktop/) is the usual choice.
- VSCode + Dev Containers
  - VS Code: <https://code.visualstudio.com/>
  - Dev Containers extension: <https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers>
- Node.js / npm
  - Official site: <https://nodejs.org/>
  - Used for `iot-market` and `iot-market-ui`.
- Rust / Cargo
  - Official site: <https://www.rust-lang.org/tools/install>
  - Used for `simple-storage`, `mediator-owner`, and `mediator-buyer`.
- Git
  - Official site: <https://git-scm.com/>
  - Used to clone the repository.

## Recommended

- Docker Desktop (for `host.docker.internal`)
- MetaMask
  - Official site: <https://metamask.io/>
  - Used to connect the frontend to the local blockchain and test purchase flow.

## Recommended Prior Knowledge (University Level)

- HTTP/REST basics (GET/POST and JSON)
- basic CLI usage (`cd`, `ls`, `cat`)
- basic Git workflow (clone, branch)
- database basics (tables and logs)

## Participant checks

First, make sure all of the following commands work:

```bash
docker --version
npm -v
cargo --version
git --version
```

Expected:

- each command prints a version
- no `command not found` errors

Next, clone the repository:

```bash
git clone --recursive https://github.com/ertlnagoya/Blockchain_IoT_Marketplace.git
cd Blockchain_IoT_Marketplace
```

The first clone may take time because submodules are included.

## OS Notes

- Windows:
  - WSL2 + Docker Desktop is recommended.
  - Reference: <https://learn.microsoft.com/windows/wsl/>
- macOS:
  - On Apple Silicon, the first container builds may take longer.
- Linux:
  - Docker daemon permissions may require extra setup.

## What to Understand at This Stage

- `iot-market` / `iot-market-ui`: local blockchain and frontend
- `simple-storage`: storage for raw data
- `ipfs`: supporting metadata storage
- `mediator-owner` / `mediator-buyer`: processes that mediate data sharing

For background reading, see [Hardhat Basics](../foundations/hardhat-basics.md) and [SSI/DID/VC Basics](../foundations/ssi-did-vc-basics.md).

## Instructor checks

1. Prepare demo MetaMask accounts
2. Verify deployment logs for `iot-market`
3. Verify IPFS/PostgreSQL setup status
4. Verify at least one input path (HUSKYLENS2 or Webcam)
5. Distribute a report template before class
