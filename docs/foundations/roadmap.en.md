# Learning Roadmap

This roadmap helps learners move from "it runs" to "I can explain why it works".

When you are unsure which page to read first, use this as the reference.  
The order is the same whether you go through it quickly or over several weeks. Reading orders on other pages of this site are short versions of this page.

## Priority Order

High comes first, then Medium and Advanced.

| Priority | Topic | Goal | Estimated Time |
|---|---|---|---|
| High | Understand site navigation | know what to read in what order | approx. 30 min |
| High | Hands-on reproduction | reproduce `allow/deny` and read audit logs | approx. 2–3 hours |
| Medium | Blockchain basics + Hardhat | explain the role of local chain | approx. 3–4 hours |
| Medium | SSI/DID/VC basics | explain consent-based policy meaning | approx. 3–4 hours |
| Advanced | Advanced (events/PEP) | explain Phase 2/3 design direction | approx. 2–3 hours |

## Short Learning Path

1. [Blockchain Basics](blockchain-basics.md)
2. [Hardhat Basics (Blockchain Used in This Site)](hardhat-basics.md)
3. [SSI / DID / VC Basics](ssi-did-vc-basics.md)
4. [Prerequisites (install tools)](../setup/prerequisites.md)
5. [Run the whole pipeline without hardware (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) (run the whole flow without hardware)
6. [Quickstart (run the stack)](../setup/quickstart.md)
7. One page from §1.2 of Part 1: [Detect a person on an AI camera (HUSKYLENS2)](../hands-on/huskylens2.md), [Detect with a USB webcam (OpenCV)](../hands-on/webcam.md), or [Pull from Home Assistant (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md)
8. List and buy an item in §1.3 of Part 1 (§1.3 of [Hands-on](../hands-on/index.md#13-trade-on-the-marketplace-v1-lane))

This completes the basic understanding and the minimum hands-on. After that, go on to Part 2 and Part 3 of [Hands-on](../hands-on/index.md) as far as you need.

## 3-Week Plan (Example)

| Week | Learning Focus | Tasks | Deliverable |
|---|---|---|---|
| Week 1 | Run and observe | run the sample in [Pull from Home Assistant (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md), intentionally trigger `allowed/denied/send_error` | execution notes |
| Week 2 | Fundamentals | study Blockchain/Hardhat and SSI/DID/VC pages, summarize terms | personal glossary |
| Week 3 | Explain and improve | explain architecture and submit one improvement PR | 1 PR |

## Minimum Checklist

- [ ] can boot stack with `docker compose up`
- [ ] can observe `denied -> allowed` before and after registering a consent (a JSON record) with `/consents`
- [ ] can explain why `dataset_id` and `purpose` are policy keys
- [ ] can explain Hardhat as a local development chain
- [ ] can explain DID/VC as identity and permission primitives

## Read Next

1. The first page of the short learning path: [Blockchain Basics](blockchain-basics.md)
2. To look up standards or research background: [References and Further Learning Materials](references.md)
3. After the hands-on, if you want the mechanisms in detail: [Design](../design/index.md)

## How to Use External Materials

- Read through the explanations in this site first.
- Use external sources to check primary sources for terms, detailed specifications, and follow-up study.
