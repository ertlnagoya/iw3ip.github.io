# FAQ

Questions about how to proceed and how the material is positioned. For isolating execution errors, see [Troubleshooting](troubleshooting.md).

## Q. Which sample should I start with?

A. With or without real hardware, after [Prerequisites (install tools)](../setup/prerequisites.md), go in this order: [Run the whole pipeline without hardware (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) (§1.1 of Part 1) → [Quickstart (run the stack)](../setup/quickstart.md) → §1.2 of Part 1. In §1.2, pick one of [Detect a person on an AI camera (HUSKYLENS2)](../hands-on/huskylens2.md), [Detect with a USB webcam (OpenCV)](../hands-on/webcam.md), and [Pull from Home Assistant (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md), depending on what you have. If you pick HA SSI Publisher, run Quickstart after §1.2 (before §1.3). `assistant-demo` ([Let an AI plan the steps (LLM Planner)](../hands-on/llm-planner.md)), which starts with one command, is Part 3 content and is not needed the first time.

## Q. Can I proceed without real devices or a smartphone?

A. Part 1 needs only a PC, except for the phone viewing step (§1.4, optional). The page [Run the whole pipeline without hardware (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) runs without hardware, and the HUSKYLENS2 and USB webcam pages have a mock mode. The Part 2 hands-on pages that use the wallet need a smartphone. §1.4 and Stage A in Part 2 also need MetaMask Mobile on the phone.

## Q. Do I need real money or cryptocurrency?

A. No. The blockchain is a local Hardhat chain on your PC, and the ETH used for purchases is test ETH.

## Q. What is the difference between Part, Phase, and Stage?

A. Part is the learning order on this site, Phase is the development stage of the platform, and Stage is a sequence label for the Part 2 hands-on pages built around VCs (it also covers the preparation in Stage 0 and Stage P, which signs image provenance. P0 of Stage P runs on its own without VCs; P1–P3 connect to the publisher and the chain and use the per-tier ViewerToken). See [Hands-on](../hands-on/index.md#part-phase-stage).
