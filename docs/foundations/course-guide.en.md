# Course Guide

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
| 3-4 | Blockchain fundamentals | [Blockchain Basics](blockchain-basics.md) / [Hardhat Basics](hardhat-basics.md) |
| 5-6 | SSI/DID/VC fundamentals | [SSI/DID/VC Basics](ssi-did-vc-basics.md) |
| 7-9 | Implementation exercises | [Environment Setup](../setup/index.md) + [Hands-on](../hands-on/index.md) |
| 10-12 | Validation and comparison | [Troubleshooting](../operations/troubleshooting.md) + [FAQ](../operations/faq.md) |
| 13-15 | Final presentation | [Learning Roadmap](roadmap.md) + [References](references.md) |

## Page sets by number of exercise sessions

Assuming 90-minute sessions, these are the pages to cover depending on how many sessions are available for exercises. Times are the sums of the estimates at the top of each page. Students are expected to finish the prerequisites (tool installation) on their own beforehand.

| Sessions available | Pages to cover | Aim |
|---|---|---|
| 1 | [HA Demo Simulator](../hands-on/ha-demo-simulator.md) (20 min) → [HA × SSI Publisher](../hands-on/ha-ssi-publisher.md) (45 min) → review | Confirm consent-based `allow` / `deny` and the audit log |
| 3 (sessions 7–9 in the table above) | Session 1: the single-session set above. Session 2: [Quickstart](../setup/quickstart.md) (30 min) → [USB Webcam](../hands-on/webcam.md) (30 min; mock mode is enough) → [Mobile Viewer](../hands-on/mobile-viewer.md) (20 min). Session 3: [USB Webcam Event Sharing](../hands-on/webcam-event-sharing.md) (45 min) → [Environment / Disaster Event Sharing](../hands-on/environment-disaster.md) (30 min) | Go from ingestion through trading to event sharing, without a phone wallet |
| 5 | The 3-session set plus: Session 4: [Consent VC](../hands-on/ha-ssi-wallet.md) (60 min). Session 5: [Viewer VC](../hands-on/ha-ssi-viewer.md) (45 min) → [Service VC](../hands-on/ha-ssi-service.md) (45 min) | Issue and present VCs, and compare write, read, and machine-to-machine authorization |
| Advanced (intensive course, thesis work) | Choose from the marketplace integration ([Stage 5](../hands-on/marketplace-vc-bridge.md) to [Stage 7](../hands-on/marketplace-seller-vc.md), about 4 hours), [tiered access](../hands-on/data-user-vc-tiered.md) (about 4 hours), and Part 3 (about 3 hours) | Linking payment and authorization, tiering by trust, AI-based decisions |

Sessions that use the wallet (session 4 onward) need `iw3ip-wallet` on the students' phones. Because building it takes time, the instructor should distribute a prepared build in advance.

In a 15-session plan, sessions 10–12 ("Validation and comparison") can also be used for the Part 3 pages [Regional Safety Assistant](../hands-on/regional-safety-assistant.md) (60 min) and [LLM Planner](../hands-on/llm-planner.md) (60 min).

## Suggested Assessment

- Reproducibility (40%): can run the workflow end-to-end
- Conceptual explanation (40%): can explain why `allow/deny` happened
- Improvement proposal (20%): quality of Phase 2/3 extension ideas

## Minimum Report Requirements

- execution environment (OS, Docker versions)
- command history and logs
- comparison of `allow` and `deny` cases
- at least one improvement proposal
