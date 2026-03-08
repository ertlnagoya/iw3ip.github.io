# Hands-on

This section is the practical part of IW3IP where learners run the system themselves.  
The intended reading order is to first understand the **Phase 1 -> Phase 2 -> Phase 3** structure, and then move into the detailed pages.

## Overall structure

The hands-on content grows in three phases.

| Phase | What you learn | Typical checkpoints |
|---|---|---|
| Phase 1: Data Exchange | how raw data moves from sensors/cameras into the platform | data is generated, shared, and viewed |
| Phase 2: Event Sharing | how events or inference results are shared instead of full raw data | `allowed` / `denied`, consent, audit log |
| Phase 3: Intelligence Integration | how human requests are interpreted into collection, evaluation, and control | `plan`, `execute`, `planner_diagnostics`, UI |

## What learners should understand first

- **Phase 1** is about how data enters the system.
- **Phase 2** is about how events are shared under conditions.
- **Phase 3** is about how AI interprets a request and coordinates multiple steps.

In other words, the learning path is:

1. make data exchange work
2. make conditional event sharing work
3. add request understanding and control

## Recommended learning order

### For first-time learners

1. [HUSKYLENS2 sample](huskylens2.md) or [USB webcam sample](webcam.md)
2. [HA x SSI Publisher sample](ha-ssi-publisher.md)
3. [USB webcam event sharing sample (Phase 2)](webcam-event-sharing.md) or [Environment and disaster event sharing sample (Phase 2)](environment-disaster.md)
4. [Regional safety assistant sample (Phase 3)](regional-safety-assistant.md)
5. [LLM Planner hands-on](llm-planner.md)

### For learners who want the shortest Phase 3 demo first

You can start the shortest Phase 3 demo with `assistant-demo`.

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

Open:

- `http://localhost:4173`

Matching page:

- [LLM Planner hands-on](llm-planner.md)

## Phase 1: Data Exchange

### Goal of this phase

- obtain data from cameras or sensors
- understand the basic path before advanced sharing control
- confirm visible outputs such as generated files or UI screens

### Matching hands-on pages

- [HUSKYLENS2 sample](huskylens2.md)
  - generate event files from HUSKYLENS2 and a PC
- [USB webcam sample](webcam.md)
  - learn the basic camera-based path with a common webcam
- [HA x SSI Publisher sample](ha-ssi-publisher.md)
  - learn the basic structure of Home Assistant, MQTT, Consent VC, and audit logging
- [Mobile Viewer](mobile-viewer.md)
  - inspect shared results from a smartphone

### Phase 1 success criteria

- data or event files are generated
- shared results can be checked through APIs or UI
- learners can explain where data is generated and where it is shared

## Phase 2: Event Sharing

### Goal of this phase

- move from full raw-data sharing to event sharing
- understand control by Consent VC and purpose
- explain the meaning of `allowed` / `denied` and the audit log

### Matching hands-on pages

- [USB webcam event sharing sample (Phase 2)](webcam-event-sharing.md)
  - share `possible_littering` as an event
- [Environment and disaster event sharing sample (Phase 2)](environment-disaster.md)
  - share `flood_risk_high` as an event

### Phase 2 success criteria

- both `allowed` and `denied` can be reproduced
- learners can explain the difference between `purpose` and `dataset_id`
- learners can explain the role of `/audit/logs` and `/platform/ingest`

## Phase 3: Intelligence Integration

### Goal of this phase

- convert a natural-language human request into a plan
- understand collection, evaluation, and control as one flow
- understand why the planner can be replaced by an LLM-based planner

### Matching hands-on pages

- [Regional safety assistant sample (Phase 3)](regional-safety-assistant.md)
  - learn the basic path from request to `plan` and `execute`
- [LLM Planner hands-on](llm-planner.md)
  - practical path for replacing the rule-based planner with an LLM planner
- [LLM Planner replacement spec](llm-planner-spec.md)
  - specification-oriented page for understanding the design in more depth

### Phase 3 success criteria

- a natural-language request can be converted into a `plan`
- `executed` can be confirmed when conditions are met
- the fields in `planner_diagnostics` can be explained
- results can be inspected from the frontend demo

## Relation to Workshop

This chapter is the participant execution part of the overall flow described in [Workshop Overview](../workshop/index.md).  
Instructors and TAs are expected to explain the structure and branching in the Workshop chapter first, and then guide participants into the detailed pages here.
