# Learning Roadmap

This roadmap helps learners move from "it runs" to "I can explain why it works".

When you are unsure which page to read first, use this as the reference.  
The order is the same whether you go through it quickly or over several weeks.

## Priority Order

P0 comes first, then P1 and P2.

| Priority | Topic | Goal | Estimated Time |
|---|---|---|---|
| P0 | Understand site navigation | know what to read in what order | approx. 30 min |
| P0 | Hands-on reproduction | reproduce `allow/deny` and read audit logs | approx. 2–3 hours |
| P1 | Blockchain basics + Hardhat | explain the role of local chain | approx. 3–4 hours |
| P1 | SSI/DID/VC basics | explain consent-based policy meaning | approx. 3–4 hours |
| P2 | Advanced (events/PEP) | explain Phase 2/3 design direction | approx. 2–3 hours |

## Short Learning Path

1. [Blockchain Basics](blockchain-basics.md)
2. [Hardhat Basics](hardhat-basics.md)
3. [SSI/DID/VC Basics](ssi-did-vc-basics.md)
4. [Prerequisites](../setup/prerequisites.md)
5. [HA Demo Simulator](../hands-on/ha-demo-simulator.md) (run the whole flow without hardware)
6. [Quickstart](../setup/quickstart.md)
7. [HA x SSI Publisher Sample](../hands-on/ha-ssi-publisher.md)

These seven pages complete the basic understanding and the minimum hands-on.

## 3-Week Plan (Example)

| Week | Learning Focus | Tasks | Deliverable |
|---|---|---|---|
| Week 1 | Run and observe | run HA x SSI sample, intentionally trigger `allowed/denied/send_error` | execution notes |
| Week 2 | Fundamentals | study Blockchain/Hardhat and SSI/DID/VC pages, summarize terms | personal glossary |
| Week 3 | Explain and improve | explain architecture and submit one improvement PR | 1 PR |

## Minimum Checklist

- [ ] can boot stack with `docker compose up`
- [ ] can observe `denied -> allowed` by registering Consent VC
- [ ] can explain why `dataset_id` and `purpose` are policy keys
- [ ] can explain Hardhat as a local development chain
- [ ] can explain DID/VC as identity and permission primitives

## Read Next

1. [Blockchain Basics](blockchain-basics.md)
2. [Hardhat Basics](hardhat-basics.md)
3. [SSI/DID/VC Basics](ssi-did-vc-basics.md)
4. [References](references.md)
5. After the hands-on, if you want the mechanisms in detail: [Design](../design/index.md)

## How to Use External Materials

- Read through the explanations in this site first.
- Use external sources to check primary sources for terms, detailed specifications, and follow-up study.
