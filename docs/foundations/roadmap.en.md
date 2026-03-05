# Learning Roadmap (Beginner)

This roadmap helps learners move from "it runs" to "I can explain why it works".

## Priority Order

| Priority | Topic | Goal | Estimated Time |
|---|---|---|---|
| P0 | Hands-on reproduction | reproduce `allow/deny` and read audit logs | 2-3 hours |
| P1 | Blockchain basics + Hardhat | explain the role of local chain | 3-4 hours |
| P1 | SSI/DID/VC basics | explain consent-based policy meaning | 3-4 hours |
| P2 | Advanced (events/PEP) | explain Phase 2/3 design direction | 2-3 hours |

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
