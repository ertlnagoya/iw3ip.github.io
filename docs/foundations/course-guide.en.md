# Course Guide

!!! abstract "Where this page belongs"
    This page is in the "Workshop" section for instructors and TAs. Participants should start from the [Learning Roadmap](roadmap.md).

This chapter explains how IW3IP material can be used in classes, workshops, and practical exercises.

Running the samples in order already teaches the material, but a class runs more smoothly when you decide in advance where to present the overall picture, where to do hands-on work, and where to reflect.

## Target Learners

- Undergraduate CS/CE students (2nd-4th year)
- Basic familiarity with networking, Web/API, databases, and programming

## Learning Outcomes

- Explain decentralized data-sharing architecture
- Understand consent-based policy decisions (Consent VC)
- Explain on-chain/off-chain separation rationale
- Report experimental results with reproducible evidence

## How to Use This Material

The basic learning and hands-on can be followed within the flow of this site.

- Main topics covered in this site:
  - fundamental terminology
  - system overview
  - quickstart
  - hands-on execution
  - result interpretation
- External material becomes useful when:
  - you want deeper technical background
  - you need literature review for reports or research
  - you want to confirm standards or primary sources

In class, it is more efficient to align the prerequisites using the explanations in this site before expanding into external references.

## Example Placement in a 15-Session Course

| Session | Theme | Related Chapters |
|---|---|---|
| 1-2 | Introduction and motivation | [Platform Overview](../platform-overview.md) |
| 3-4 | Blockchain fundamentals | [Blockchain Basics](blockchain-basics.md) / [Hardhat Basics (Blockchain Used in This Site)](hardhat-basics.md) |
| 5-6 | SSI/DID/VC fundamentals | [SSI / DID / VC Basics](ssi-did-vc-basics.md) |
| 7-9 | Implementation exercises | [Environment Setup](../setup/index.md) + [Hands-on](../hands-on/index.md) |
| 10-12 | Validation and comparison | [Troubleshooting](../operations/troubleshooting.md) + [FAQ](../operations/faq.md) |
| 13-15 | Final presentation | [Learning Roadmap](roadmap.md) + [References and Further Learning Materials](references.md) |

## Page sets by number of exercise sessions

Assuming 90-minute sessions, these are the pages to cover depending on how many sessions are available for exercises. Times are the sums of the estimates at the top of each page. Students are expected to finish the prerequisites (tool installation) on their own beforehand. The page order is mostly the same as the short learning path in the participants' [Learning Roadmap](roadmap.md) (Prerequisites → Run the whole pipeline without hardware → Quickstart → §1.2 of Part 1 → §1.3). The 1-session and 3-session sets use [Pull from Home Assistant (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md) for §1.2, so Quickstart comes after §1.2 (at the start of session 2 in the 3-session set).

| Sessions available | Pages to cover | Aim |
|---|---|---|
| 1 | [Run the whole pipeline without hardware (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) (approx. 20 min) → [Pull from Home Assistant (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md) (approx. 45 min) → review | Confirm consent-based `allow` / `deny` and the audit log |
| 3 (sessions 7–9 in the table above) | Session 1: the single-session set above. Session 2: stop the publisher from session 1, then [Quickstart (run the stack)](../setup/quickstart.md) (approx. 15–30 min) → [Detect with a USB webcam (OpenCV)](../hands-on/webcam.md) (approx. 30 min; mock mode is enough) → listing and purchase in §1.3 of [Hands-on](../hands-on/index.md#13-trade-on-the-marketplace-v1-lane) → if time allows, [Open the marketplace and buy on a phone (v1)](../hands-on/mobile-viewer.md) (optional; approx. 20 min). Session 3: [Share detections, not raw frames](../hands-on/webcam-event-sharing.md) (approx. 45 min) → [Share disaster events](../hands-on/environment-disaster.md) (approx. 30 min) | Go from ingestion through trading to event sharing, without the SSI wallet (the phone purchase uses MetaMask Mobile) |
| 5 | The 3-session set plus: Session 4: [Authenticate the data provider (Consent VC / Stage 1)](../hands-on/ha-ssi-wallet.md) (approx. 60 min). Session 5: [Authenticate the viewer (Viewer VC / Stage 3)](../hands-on/ha-ssi-viewer.md) (approx. 45 min) → [Authenticate machines (Service VC / Stage 4 prep)](../hands-on/ha-ssi-service.md) (approx. 45 min) | Issue and present VCs, and compare write, read, and machine-to-machine authorization |
| Advanced (intensive course, thesis work) | Choose from the marketplace integration ([Hand a credential to the buyer's wallet (Bridge / Stage 5)](../hands-on/marketplace-vc-bridge.md) to [Back the seller's identity (Seller VC / Stage 7)](../hands-on/marketplace-seller-vc.md), about 3 hours 15 min), [Tier the response by trust (DataUserVC / Stage T)](../hands-on/data-user-vc-tiered.md) and the two following pages (about 3.5–4 hours), image provenance (Stage P, P0–P3: [Prove where an image came from (C2PA / Stage P)](../hands-on/c2pa-provenance.md) to [Trace a VLM summary back to the image (C2PA / Stage P, P3)](../hands-on/c2pa-provenance-vlm.md), about 1.5–2 hours), and Part 3 (about 4–5 hours) | Linking payment and authorization, tiering by trust, proving image provenance, AI-based decisions |

Sessions that use the wallet (session 4 onward) need `iw3ip-wallet` on the students' phones. Because building it takes time, the instructor should distribute a prepared build in advance.

In a 15-session plan, sessions 10–12 ("Validation and comparison") can also be used for the Part 3 pages [Run the regional safety assistant](../hands-on/regional-safety-assistant.md) (approx. 60 min) and [Let an AI plan the steps (LLM Planner)](../hands-on/llm-planner.md) (approx. 60 min).

## Suggested Assessment

- Reproducibility (40%): can run the workflow end-to-end
- Conceptual explanation (40%): can explain why `allow/deny` happened
- Improvement proposal (20%): quality of Phase 2/3 extension ideas

## Minimum Report Requirements

- execution environment (OS, Docker versions)
- command history and logs
- comparison of `allow` and `deny` cases
- at least one improvement proposal
