# Environment Setup

This section covers what you need before starting the hands-on.
It assumes a high school or undergraduate student is encountering this for the first time, so terms and steps are explained in plain language.

## What you are building

In the IW3IP hands-on you will run the following on your own laptop:

- collect data from IoT devices (cameras, sensors, phones)
- share that data with other people **conditionally**
- purchase the shared data or have an AI evaluate it

You build this on your own PC as a local development environment for sharing data safely and conditionally with others. No real cloud or money is involved — everything runs locally.

## Overall structure

The hands-on is split into **Basic** and **Feature Extensions**.

```
 ┌──────────────────────────────────────────────────────────┐
 │  Environment Setup (these pages)                         │
 │   - install tools                                        │
 │   - start the stack                                      │
 │   - sanity-check                                         │
 └──────────────────────────────────────────────────────────┘
          ↓ if this works, you're ready
 ┌──────────────────────────────────────────────────────────┐
 │  Hands-on Part 1: Basic                                  │
 │   capture, view, and trade IoT data                      │
 └──────────────────────────────────────────────────────────┘
          ↓ go deeper if you want
 ┌──────────────────────────────────────────────────────────┐
 │  Hands-on Part 2: Feature Extensions                     │
 │   consent, conditional sharing, wallet, tiered access    │
 └──────────────────────────────────────────────────────────┘
          ↓ advanced
 ┌──────────────────────────────────────────────────────────┐
 │  Hands-on Part 3: Intelligence Integration               │
 │   let an AI interpret human requests and act on them     │
 └──────────────────────────────────────────────────────────┘
```

## Pick your track

| Goal | Path |
|---|---|
| Just see something running | [Quickstart](quickstart.md) → first section of Part 1 |
| Walk through the basic flow | Setup → all of Part 1 |
| Reach safe / conditional sharing | Setup → Part 1 → Part 2 |
| Full intelligence-integrated demo | Setup → Part 1 → Part 2 → Part 3 |

It is fine to stop partway. Each Part builds on the knowledge of the previous one step by step.

## Setup steps

Follow these in order. Each page ends with a "you can move on if…" checklist.

1. [Prerequisites (install tools)](prerequisites.md)
2. [Quickstart (run the stack)](quickstart.md)
3. If something breaks → [Troubleshooting](../operations/troubleshooting.md)

## Mini-glossary

One-liner definitions of acronyms you'll meet in the hands-on. Each chapter expands the terms it uses.

| Term | Short meaning |
|---|---|
| **IoT** | "Things on the network" — cameras, sensors, etc. |
| **VC** (Verifiable Credential) | A digital certificate, e.g. "this person works for the police" |
| **DID** | A unique ID for the holder of a VC |
| **SSI** (Self-Sovereign Identity) | You hold and present your own credentials |
| **wallet** | App (usually on your phone) that stores and presents VCs |
| **publisher** | The server that emits data |
| **viewer** | The party that consumes data |
| **purpose** | Why the data will be used (research, crime_search, …) — used to decide allow/deny |
| **Consent VC** | A VC that says "I allow this kind of sharing" |
| **DataUserVC** | A VC that describes how trustworthy the data user is |
| **PurchaseViewerVC** | A VC issued after a purchase, granting view access |
| **bridge** | A go-between that connects the marketplace to the wallet |
| **audit log** | A record of who looked at what, when |

## What you need

Bare minimum:

- **A PC** (macOS / Windows / Linux, 8 GB RAM or more recommended)
- **Internet** (for the initial docker pull / npm install)

Extra for feature-extension part:

- **A smartphone** (Android / iPhone) — to install the SSI wallet
- **MetaMask** browser extension — to buy data from the marketplace

Optional hardware (everything works without it):

- **HUSKYLENS2** or **USB webcam** — for the camera-based Phase 1 lessons
- **Raspberry Pi** — to run Home Assistant on real hardware

If you have none of the above, the **Home Assistant Demo Simulator** lets you do the entire walkthrough virtually.
