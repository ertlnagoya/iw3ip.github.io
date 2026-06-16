# Hands-on

This is the chapter where you run IW3IP yourself to understand it.
**Basic** establishes the overall picture, **Feature Extensions** covers safe sharing and tiered access, and **Intelligence Integration** shows the AI integration.

> Finish the [Environment Setup](../setup/index.md) before moving on to this page.

## Basic vs Feature Extensions

The hands-on explains the marketplace stack in two layers: **Basic (v1)** and **Feature Extensions (v2)**.
They are **additive lanes, not a replacement** — Basic alone is enough to buy and sell data.

### Basic (v1) — minimal setup

- List data on the marketplace, **purchase via MetaMask**, **download from encrypted IPFS** and decrypt
- All you need is a PC and MetaMask
- Part 1 of the hands-on builds this v1 path

### Feature Extensions (v2) — safe, conditional, tiered access

- After purchase a **smartphone SSI wallet** receives a **VC**
- The receiver presents that VC to fetch data. The contents shown vary by `purpose` and trust level
- Part 2 of the hands-on covers this v2 path plus the consent and tiered access that sit on top of it

```
Purchase (shared by v1 and v2)
    ↓
    ├── v1 lane: encrypted-IPFS URI → decrypt → data       ← covered in Part 1
    └── v2 lane: bridge → PurchaseViewerVC → wallet         ← covered in Part 2
                  → ViewerToken → /platform/data
```

Design detail: [Marketplace VC Bridge — v1 / v2 design spec](../design/marketplace-vc-bridge-spec.md).

## Overall structure

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Part 1 / Basic (v1)</div>
    <h3>📡 Capture and share IoT data</h3>
    <p>Pull data off cameras and sensors, view it, and trade it on the marketplace.</p>
    <p class="iw3ip-phase-links">
      <a href="ha-demo-simulator.md">HA Demo Simulator</a> /
      <a href="huskylens2.md">HUSKYLENS2</a> /
      <a href="webcam.md">USB Webcam</a> /
      <a href="ha-ssi-publisher.md">HA Publisher</a> /
      <a href="mobile-viewer.md">Mobile Viewer</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Part 2 / Feature Extensions (v2)</div>
    <h3>🧭 Consent, conditional sharing, tiered access</h3>
    <p>Use VCs and a wallet to share according to purpose and trust level. You can stop partway.</p>
    <p class="iw3ip-phase-links">
      <a href="ha-ssi-wallet.md">HA SSI Wallet</a> /
      <a href="webcam-event-sharing.md">Webcam Event Sharing</a> /
      <a href="environment-disaster.md">Environment Disaster</a> /
      <a href="marketplace-vc-bridge.md">VC Bridge</a> /
      <a href="data-user-vc-tiered.md">DataUserVC Tiered</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Part 3 / Intelligence Integration</div>
    <h3>🧠 Let an AI act on requests</h3>
    <p>An AI interprets free-form requests and uses Part 1/2 data as material to plan and act.</p>
    <p class="iw3ip-phase-links">
      <a href="regional-safety-assistant.md">Regional Safety Assistant</a> /
      <a href="llm-planner.md">LLM Planner</a>
    </p>
  </div>
</div>

## Paths by goal

| Goal | Path |
|---|---|
| See it run first | [HA Demo Simulator](ha-demo-simulator.md) only (approx. 15 min) |
| Cover all of Basic | All of Part 1 (60–90 min) |
| Reach safe / conditional sharing | Part 1 → Part 2 §2.1–§2.3 (+60 min) |
| Reach v2 marketplace integration | Part 1 → Part 2 §2.4 (+30 min) |
| Reach tiered access / semantic | Through Part 2 §2.5 (+30 min) |
| Reach AI integration | Through Part 3 (+30 min) |

Each hands-on page opens with a "What you'll learn / Prerequisites / What you need / Time required" block. If you get stuck partway, see [Troubleshooting](../operations/troubleshooting.md).

## Tech stack at a glance

What runs behind each sub-group. Part 2 is mixed — some sub-groups need the blockchain, others don't.

| Sub-group | publisher (Docker) | Blockchain (Hardhat) | MetaMask | SSI wallet (phone) | IPFS (encrypted) | Image proc. / LLM |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| **Part 1 / Capture data** | ✓ | – | – | – | – | – |
| **Part 1 / Trade (v1)** | ✓ | ✓ | ✓ | – | ✓ | – |
| **Part 2 / Add authentication** | ✓ | – | – | ✓ | – | – |
| **Part 2 / Wire wallet to marketplace (v2)** | ✓ | ✓ | ✓ | ✓ | – | – |
| **Part 2 / Process data before sharing** | ✓ | ✓ <small>(for Tier 3 e2e)</small> | ✓ <small>(same)</small> | ✓ | – | ✓ <small>(OpenCV / VLM optional)</small> |
| **Part 3 / Intelligence integration** | ✓ | – | – | – | – | ✓ <small>(LLM)</small> |

### VCs used

| Sub-group | Consent | Viewer | Service | Purchase&shy;Viewer | Seller | DataUser |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Part 2 / Add authentication | ✓ | ✓ | ✓ | – | – | – |
| Part 2 / Wire wallet to marketplace (v2) | – | – | ✓ | ✓ | ✓ | – |
| Part 2 / Process data before sharing | – | – | – | ✓ | – | ✓ |

### Reading the tables

- **publisher**: the hands-on server. Every sub-group brings it up with `docker compose up`.
- **Blockchain**: a local Hardhat chain plus deployed contracts. Required when the marketplace purchase is the entry point.
- **MetaMask**: signs / sends transactions on the local chain.
- **SSI wallet**: Sphereon Wallet on a phone — receives and presents VCs.
- **IPFS (encrypted)**: the v1 post-purchase delivery lane. v2 replaces this role with VCs.
- **Image proc. / LLM**: OpenCV (Tier 2 blur), Ollama (LLaVA / moondream for VLM), and the LLM planner in Part 3.

---

## Part 1: Basic — the core data path

### Goals

- See **where data is generated and where it ends up**
- Round-trip a **v1 marketplace purchase** (encrypted-IPFS delivery)

### 1.1 Run it first — check the whole flow without hardware

The Home Assistant simulator lets you run the entire walkthrough without any IoT device. A good place to begin.

- [HA Demo Simulator](ha-demo-simulator.md)

### 1.2 Capture data from a device

Build a path that pulls data off a camera or sensor and sends it to the publisher. Choose one that matches your hardware:

- [HUSKYLENS2](huskylens2.md) — purpose-built AI camera
- [USB Webcam](webcam.md) — generic USB camera + OpenCV
- [HA × SSI Publisher](ha-ssi-publisher.md) — Home Assistant integration

### 1.3 View on a phone

View the collected data in a phone browser.

- [Mobile Viewer](mobile-viewer.md)

### 1.4 Trade on the marketplace (v1 lane)

List → buy → decrypt-from-IPFS round trip. This is the completion point of **Basic (v1)**.

- List with iot-market-ui
- Buy with MetaMask
- Decrypt the encrypted data stored on IPFS

Detailed bring-up: [Quickstart](../setup/quickstart.md).

### Done when…

- `/platform/ingest` shows the publisher receiving your device's data
- A list-buy-receive round trip on the marketplace works

---

## Part 2: Feature Extensions — consent, conditional sharing, tiered access

### Goals

- Understand why **conditional sharing** is needed instead of "show everything"
- Implement that conditional sharing with **VCs and a wallet**
- Vary what is shown by trust level with **tiered access (Tier 3 / 2 / 1)**

### 2.1 Add consent (Consent VC)

Issue and present a "I consent to sharing for this purpose" VC.

- [HA × SSI Wallet](ha-ssi-wallet.md)

### 2.2 Event sharing — share inferences, not raw frames

Send "person detected: yes" / "littering: yes" events instead of the raw video.

- [Webcam Event Sharing](webcam-event-sharing.md)
- [Environment Disaster](environment-disaster.md)

### 2.3 Split read vs write roles

Use VCs to separate "read-only viewer" from "write-only service".

- [HA × SSI Viewer](ha-ssi-viewer.md)
- [HA × SSI Service](ha-ssi-service.md)

### 2.4 Marketplace × wallet (v2 lane)

Replace IPFS-encrypted delivery with **PurchaseViewerVC + ViewerToken**.

- [Marketplace VC Bridge](marketplace-vc-bridge.md) — stand up the bridge
- [Marketplace VC end-to-end (Stage 6)](marketplace-vc-end-to-end.md) — full buy-to-view path
- [Marketplace Seller VC (Stage 7)](marketplace-seller-vc.md) — seller-side VC
- [Marketplace Mobile App (Stage A)](marketplace-mobile-app.md) — dedicated mobile app

### 2.5 Tiered access by trust (Stage T)

Switch the response between Tier 3 (video + everything), Tier 2 (image + derived), and Tier 1 (summary only) based on the user's **DataUserVC**. Goes all the way to **semantic intermediate representation + trust-aware rendering**.

- [DataUserVC Tiered Access hands-on](data-user-vc-tiered.md)
- [DataUserVC Tiered Access spec](data-user-vc-tiered-spec.md)

### Done when…

- A Consent / DataUserVC / PurchaseViewerVC lands in your phone wallet
- Changing `purpose` or trust level flips `/platform/data` between allow / deny / different content
- The audit log records who looked at what, when, and for what purpose

---

## Part 3: Intelligence Integration — let an AI act on requests

### Goals

- An AI breaks a free-form request ("show me recent anomalies") into a **plan**
- That plan **executes** against Part 1/2 data and returns a response
- A frontend demo ties it all together

### 3.1 Regional Safety Assistant

Uses the "when / where / what happened" event stream from Part 1/2 to answer free-form requests.

- [Regional Safety Assistant](regional-safety-assistant.md)

### 3.2 LLM Planner

The "request → plan → execute → UI" decomposition pattern.

- [LLM Planner hands-on](llm-planner.md)
- [LLM Planner replacement spec](llm-planner-spec.md)

### Done when…

- A free-form request decomposes into plan steps that execute
- Results show up in the frontend demo
- `planner_diagnostics` lets you trace the decision

---

## When a term is unfamiliar

See the [setup mini-glossary](../setup/index.en.md). It explains almost every term you'll encounter in one line.
