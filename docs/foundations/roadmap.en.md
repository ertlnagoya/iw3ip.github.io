# Learning Roadmap

This roadmap helps learners move from "it runs" to "I can explain why it works".

## Priority Order

| Priority | Topic | Goal | Estimated Time |
|---|---|---|---|
| P0 | Understand site navigation | know what to read in what order | 30 min |
| P0 | Hands-on reproduction | reproduce `allow/deny` and read audit logs | 2-3 hours |
| P1 | Blockchain basics + Hardhat | explain the role of local chain | 3-4 hours |
| P1 | SSI/DID/VC basics | explain consent-based policy meaning | 3-4 hours |
| P2 | Advanced (events/PEP) | explain Phase 2/3 design direction | 2-3 hours |

## Short Learning Path

1. [Course Guide](course-guide.md)
2. [Blockchain Basics](blockchain-basics.md)
3. [Hardhat Basics](hardhat-basics.md)
4. [SSI/DID/VC Basics](ssi-did-vc-basics.md)
5. [Quickstart](../workshop/quickstart.md)
6. [HA x SSI Publisher Sample](../hands-on/ha-ssi-publisher.md)

These six pages provide a compact path through the basic concepts and the minimum hands-on flow.

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

## How to Use External Materials

- Prefer this site first for the basic path.
- Use external sources for standards, deeper technical detail, and follow-up study.
