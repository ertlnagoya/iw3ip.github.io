# Design

The Design section collects the design specs that define **how each IW3IP feature is built and why that approach was chosen**. Where the [Hands-on](../hands-on/index.md) section is for running the system step by step and checking the results, Design explains the mechanisms and the reasons behind the decisions. It contains no procedures or commands; it covers the kinds of VCs and tokens, the APIs, the data flows, the options considered, and the open issues.

Read it after the hands-on when you want to know why things behave as they do, when you extend or implement a feature, or when you build on it for research. You do not need it just to work through the hands-on.

## How the sections differ

| Section | Purpose | What it contains | Main readers | When to read |
|---|---|---|---|---|
| [Hands-on](../hands-on/index.md) | Understand IW3IP by running it | commands, expected output, checkpoints | participants | during the exercises, in order |
| Design (this section) | Explain the mechanisms and design decisions | VCs, tokens, APIs, data flows, chosen approaches and reasons, open issues | people extending or implementing features, participants doing advanced study, instructors | after the hands-on, or when changing a feature |
| [Support](../operations/index.md) | Get unstuck during the exercises | checks and fixes per symptom, frequently asked questions | participants and TAs | when a step does not work as described |
| [Workshop](../foundations/course-guide.md) | Plan and run a course | plans by number of sessions, assessment, session timing | instructors and TAs | when preparing a course |

## Design specs

| Design spec | Scope | Matching hands-on |
|---|---|---|
| [VC Architecture Overview](vc-architecture-overview.md) | the 6 VC kinds and 4 token kinds, which VC authorizes which API, the list of Stages | all of Part 2 |
| [Marketplace VC Bridge spec](marketplace-vc-bridge-spec.md) | connecting the marketplace and the SSI wallet (v1 / v2) | [Stage 5](../hands-on/marketplace-vc-bridge.md), [Stage 6](../hands-on/marketplace-vc-end-to-end.md) |
| [SellerVC spec](seller-vc-spec.md) | verifying the seller's identity | [Stage 7](../hands-on/marketplace-seller-vc.md) |
| [Tiered access design spec](../hands-on/data-user-vc-tiered-spec.md) | tailoring the response to the receiver's trust level (DataUserVC, VLM-derived data) | the three [Stage T](../hands-on/data-user-vc-tiered.md) pages |
| [C2PA Provenance Pipeline spec](c2pa-provenance-spec.md) | recording and signing the provenance of images | [Stage P](../hands-on/c2pa-provenance.md) |
| [LLM Planner replacement spec](../hands-on/llm-planner-spec.md) | design boundaries for replacing the regional safety assistant's planner with an LLM | [LLM Planner](../hands-on/llm-planner.md) |

## How to read the design specs

- Many specs were written before the implementation. Where the implementation differs, the spec carries a "→ In the implementation" note. The source code of the course repository and the hands-on steps are the reference for actual behaviour.
- "Stage" in the text is the step number of the Phase 2 hands-on (listed in §5 of the [VC Architecture Overview](vc-architecture-overview.md)). The relation between Part, Phase and Stage is explained in the [Hands-on overview](../hands-on/index.md#part-phase-stage).
