# Design

The Design section collects the design specs that define **how each IW3IP feature is built and why that approach was chosen**. Where the [Hands-on](../hands-on/index.md) section is for running the system step by step and checking the results, Design explains the mechanisms and the reasons behind the decisions. It contains no procedures or commands; it covers the kinds of VCs and tokens, the APIs, the data flows, the options considered, and the open issues.

Read it after the hands-on when you want to know why things behave as they do, when you extend or implement a feature, or when you build on it for research. You do not need it just to work through the hands-on.

## How the sections differ

| Section | Purpose | What it contains | Main readers | When to read |
|---|---|---|---|---|
| [Hands-on](../hands-on/index.md) | Understand IW3IP by running it | commands, expected output, checkpoints | participants | during the exercises, in order |
| Design (this section) | Explain the mechanisms and design decisions | VCs, tokens, APIs, data flows, chosen approaches and reasons, open issues | people extending or implementing features, participants doing advanced study, instructors | after the hands-on, or when changing a feature |
| [Support](../operations/index.md) | Get unstuck during the exercises | checks and fixes per symptom, frequently asked questions | participants and TAs | when a step does not work as described |
| [Course Guide](../foundations/course-guide.md) | Plan and run a course | plans by number of sessions, assessment, session timing | instructors and TAs | when preparing a course |

## Design specs

| Design spec | Scope | Matching hands-on |
|---|---|---|
| [VC architecture overview (Phase 2 / Stages 0–7, T, A, P)](vc-architecture-overview.md) | the 6 VC kinds and 4 token kinds, which VC authorizes which API, the list of Stages | all of Part 2 |
| [Marketplace VC Bridge — v1 / v2 Design Spec](marketplace-vc-bridge-spec.md) | connecting the marketplace and the SSI wallet (v1 / v2) | [Hand a credential to the buyer's wallet (Bridge / Stage 5)](../hands-on/marketplace-vc-bridge.md), [Read ServiceVC-written data with a PurchaseViewerVC (Stage 6)](../hands-on/marketplace-vc-end-to-end.md) |
| [SellerVC — marketplace listing governance VC (Stage 7 design spec)](seller-vc-spec.md) | verifying the seller's identity | [Back the seller's identity (Seller VC / Stage 7)](../hands-on/marketplace-seller-vc.md) |
| [Tiered access design spec](../hands-on/data-user-vc-tiered-spec.md) | tailoring the response to the receiver's trust level (DataUserVC, VLM-derived data) | the three Stage T pages, starting with [Tier the response by trust (DataUserVC / Stage T)](../hands-on/data-user-vc-tiered.md) |
| [C2PA provenance pipeline — provenance for camera footage and sensor data (Stage P design spec)](c2pa-provenance-spec.md) | recording and signing image provenance, validation at the publisher, recording on chain, provenance of VLM summaries (P0–P3 implemented) | [Prove where an image came from (C2PA / Stage P)](../hands-on/c2pa-provenance.md), [Connect provenance to the publisher and the chain (C2PA / Stage P, P1 and P2)](../hands-on/c2pa-provenance-publisher.md), [Trace a VLM summary back to the image (C2PA / Stage P, P3)](../hands-on/c2pa-provenance-vlm.md) |
| [LLM Planner replacement spec](../hands-on/llm-planner-spec.md) | design boundaries for replacing the regional safety assistant's planner with an LLM | [Let an AI plan the steps (LLM Planner)](../hands-on/llm-planner.md) |

## How to read the design specs

- Many specs were written before the implementation. Where the implementation differs, the spec says so in a note (a "→ In the implementation" line, an "in the original design / in the current implementation" contrast, or a "Current implementation" box). The source code of the course repository and the hands-on steps are the reference for actual behaviour.
- "Stage" in the text is the step number of the Phase 2 hands-on (listed in §5 of the [VC architecture overview (Phase 2 / Stages 0–7, T, A, P)](vc-architecture-overview.md)). The relation between Part, Phase and Stage is explained in [Hands-on](../hands-on/index.md#part-phase-stage).
