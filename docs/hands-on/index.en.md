# Hands-on

This section is the practical part of IW3IP where readers run the system themselves.  
It is easiest to begin with the **Phase 1 -> Phase 2 -> Phase 3** structure and then move into the detailed pages.

## Overall structure

The hands-on content grows in three phases.

| Phase | What you learn | Typical checkpoints |
|---|---|---|
| Phase 1: Data Exchange | how raw data moves from sensors/cameras into the platform | data is generated, shared, and viewed |
| Phase 2: Event Sharing | how events or inference results are shared instead of full raw data | `allowed` / `denied`, consent, audit log |
| Phase 3: Intelligence Integration | how human requests are interpreted into collection, evaluation, and control | `plan`, `execute`, `planner_diagnostics`, UI |

## Phase Map

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Phase 1 / Data Exchange</div>
    <h3>📡 Collect and pass data</h3>
    <p>Start by understanding the basic path from cameras or sensors into the sharing platform.</p>
    <p>If you want to start without physical devices, <a href="ha-demo-simulator.md">Home Assistant Demo Simulator</a> is the shortest path.</p>
    <p class="iw3ip-phase-links"><a href="ha-demo-simulator.md">HA Demo Simulator</a> / <a href="huskylens2.md">HUSKYLENS2</a> / <a href="webcam.md">USB Webcam</a> / <a href="ha-ssi-publisher.md">HA x SSI Publisher</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Phase 2 / Event Sharing</div>
    <h3>🧭 Share events under conditions</h3>
    <p>Move from full raw-data sharing to event and inference-result sharing with consent, purpose, and audit.</p>
    <p class="iw3ip-phase-links"><a href="webcam-event-sharing.md">Webcam Event Sharing</a> / <a href="environment-disaster.md">Environment Disaster</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Phase 3 / Intelligence Integration</div>
    <h3>🧠 Interpret requests and trigger actions</h3>
    <p>This phase connects human requests to planning, execution, diagnostics, and the frontend demo.</p>
    <p class="iw3ip-phase-links"><a href="regional-safety-assistant.md">Regional Safety Assistant</a> / <a href="llm-planner.md">LLM Planner</a></p>
  </div>
</div>

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

1. If you want to start without physical devices, begin with the [Home Assistant Demo Simulator sample](ha-demo-simulator.md)
2. If you want a hardware-based path, use the [HUSKYLENS2 sample](huskylens2.md) or the [USB webcam sample](webcam.md)
3. [HA x SSI Publisher sample](ha-ssi-publisher.md)
4. [USB webcam event sharing sample (Phase 2)](webcam-event-sharing.md) or [Environment and disaster event sharing sample (Phase 2)](environment-disaster.md)
5. [Regional safety assistant sample (Phase 3)](regional-safety-assistant.md)
6. [LLM Planner hands-on](llm-planner.md)

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

### Phase 1 page groups

<div class="iw3ip-subphase-grid">
  <div class="iw3ip-subphase-card">
    <div class="iw3ip-subphase-kicker">Suggested first entry</div>
    <h4>Start without physical devices</h4>
    <p>Use these pages when you want to understand the full path first without wiring or camera setup. They let you inspect MQTT, consent, and audit behavior in a controlled way.</p>
    <p><a href="ha-demo-simulator.md">Home Assistant Demo Simulator sample</a>: connect Phase 1, Phase 2, and Phase 3 from Home Assistant demo without physical devices</p>
    <p><a href="ha-ssi-publisher.md">HA x SSI Publisher sample</a>: learn the basic structure of Home Assistant, MQTT, Consent VC, and audit logging</p>
  </div>
  <div class="iw3ip-subphase-card">
    <div class="iw3ip-subphase-kicker">Device input basics</div>
    <h4>Phase 1 device pages</h4>
    <p>These pages focus on physical or mock device input and on the path from event-file generation to productization. The recommended order is mock first, then real device input, then troubleshooting.</p>
    <p><a href="huskylens2.md">HUSKYLENS2 sample</a>: generate event files from HUSKYLENS2 and a PC</p>
    <p><a href="webcam.md">USB webcam sample</a>: learn the basic camera-based path with a common webcam</p>
  </div>
  <div class="iw3ip-subphase-card">
    <div class="iw3ip-subphase-kicker">Result inspection</div>
    <h4>View and confirm outputs</h4>
    <p>After data is generated and shared, this page helps learners inspect the result from another device and confirm what was actually exposed to the user side.</p>
    <p><a href="mobile-viewer.md">Mobile Viewer</a>: inspect shared results from a smartphone</p>
  </div>
</div>

### Matching hands-on pages

Phase 1 walks through **one full round-trip from sensor input to
downstream sharing**. The pages stack in dependency order — earlier
stages are thin, later ones add structure on top.

#### What each stage adds and what you can learn

##### Stage 1A: bring up one input source (`huskylens2` / `webcam` / `ha-demo-simulator`)

- **Capabilities introduced**
    - input device → JSON payload generation
    - inspection of event files / simulator output
    - (ha-demo-simulator only) Home Assistant container + Mosquitto
      running without physical devices
- **What you learn**
    - Where data is born from sensors or images
    - JSON payload schema (timestamp, payload, source)
    - The fallback entry point when you don't have hardware (simulator)

##### Stage 1B: hook into the pipeline (`ha-ssi-publisher`)

- **Added on top of Stage 1A**
    - MQTT subscriber → publisher → `/platform/ingest` flow
    - Permission filtering through `/consents` JSON registration
    - Audit log persistence (`/audit/logs`) with purpose / allow-deny
    - Consent VC validity window (`valid_from` / `valid_to`) checks
- **What you learn**
    - The minimal pipeline: input → decision → delivery → audit
    - The role of Consent VC and how `purpose` × `dataset_id` decides
    - The relationship between the publisher container and the
      external API (`PLATFORM_API_URL`)

##### Stage 1C: viewing the result (`mobile-viewer`)

- **Added on top of Stage 1B**
    - Procedure for opening the `iot-market-ui` from a phone browser
    - LAN IP / `0.0.0.0` exposure setup
- **What you learn**
    - The user-side perspective: where shared results become visible
    - Minimum cross-device network requirements (PC ↔ phone)
    - (Note) viewing here is unauthenticated; VC-gated read is
      introduced later in Phase 2 Stage 3 (`ha-ssi-viewer`)

### Phase 1 success criteria

- data or event files are generated
- shared results can be checked through APIs or UI
- learners can explain where data is generated and where it is shared
- learners can explain `/consents` registration and its audit log effect

## Phase 2: Event Sharing

### Goal of this phase

- move from full raw-data sharing to event sharing
- understand control by Consent VC and purpose
- explain the meaning of `allowed` / `denied` and the audit log

### Matching hands-on pages

Phase 2 ships four hands-on pages. Together they exercise the **same
authorization logic (dataset × purpose) over two transports
(JSON registration vs wallet presentation) and two directions
(write vs read)**.

| Hands-on | Consent storage | Transport | Gated operation | Token |
| --- | --- | --- | --- | --- |
| [USB webcam event sharing](webcam-event-sharing.md) | publisher server | `/consents` JSON | write (`POST /platform/ingest`) | none (registration-based) |
| [Environment and disaster event sharing](environment-disaster.md) | publisher server | `/consents` JSON | write | none |
| [Mobile SSI wallet sample (Stage 1)](ha-ssi-wallet.md) | mobile wallet | OID4VP presentation | write | PolicyToken (5 min, single-use) |
| [SSI Viewer sample (Stage 3)](ha-ssi-viewer.md) | mobile wallet | OID4VP presentation | read (`GET /platform/data`) | ViewerToken (60 s, multi-use) |

#### What each stage adds and what you can learn

Following these in order lets you **deepen the authorization model
incrementally**, with each stage adding a focused capability on top
of the previous one.

##### Stage 0: baseline (`webcam-event-sharing` / `environment-disaster`)

- **Capabilities introduced**
    - `/consents` JSON registration on the publisher
    - MQTT topic → `dataset_id` normalization → `(dataset_id, purpose)`
      → `allow` / `deny`
    - persistent audit log via `/audit/logs`
- **What you learn**
    - The difference between "raw-data sharing" and "event sharing"
    - The three axes: consent, purpose, dataset
    - The structure and reading of audit log fields
      (`raw_topic`, `purpose`, `reason`)

##### Stage 1: wallet-side write authz (`ha-ssi-wallet`)

- **Added on top of Stage 0**
    - ConsentVC issuance via OID4VCI (`/issuer/offer` etc.)
    - ConsentVC presentation via OID4VP
      (`/verifier/request` / `/verifier/response`)
    - Short-lived PolicyToken (5 min, single-use)
    - Bearer-authenticated `POST /platform/ingest` (PolicyToken)
    - Audit log gains `holder_did`, `vc_hash`, `presentation_verified`
- **What you learn**
    - What it means for the consenter's identity to be VC-backed
      (a verifiable trail of "who consented")
    - Overview of SD-JWT VC, `did:jwk`, and DCQL
    - Why "single-use tokens" fit write authorization
    - **Equivalence and differences** between JSON registration and
      the wallet path (the side-by-side appendix table)

##### Stage 2: cross-link from existing samples (docs only)

- **Added on top of Stage 1** (no backend change)
    - "Wallet-mode authorization" appendix appended to
      `webcam-event-sharing.md` / `environment-disaster.md`
    - One-event curl recipe routing through the wallet
- **What you learn**
    - What stays and what changes when an existing sample is
      wallet-ified
    - Why running continuous MQTT through the wallet needs a different
      mechanism (M2M ServiceVC) — the limit imposed by PolicyToken's
      single-use semantics

##### Stage 3: wallet-side read authz (`ha-ssi-viewer`)

- **Added on top of Stage 1**
    - ViewerVC (read-side credential with claims `dataset_id` and
      `allowed_actions=["read"]`)
    - `vc_kind=ViewerVC` dispatch in `/verifier/request`
    - ViewerToken (60 s TTL, multi-use within the TTL)
    - Bearer-authenticated `GET /platform/data?dataset_id=...`
      (ViewerToken)
    - Audit log entries with `raw_topic=platform/data` for reads
    - Issuer metadata exposes both ConsentVC and ViewerVC
- **What you learn**
    - The symmetric design of VC-gated write and read authorization
    - The semantic difference between single-use (write) and
      TTL-multi-use (read)
    - Role separation between ConsentVC and ViewerVC, and the
      independence of the PolicyToken / ViewerToken namespaces
      (cross-use is rejected)

#### Design symmetry

```
              write                          read
              ───────────────────────────   ───────────────────────────
JSON-based   │ POST /consents              │ (no equivalent yet)
Wallet-based │ ConsentVC → PolicyToken     │ ViewerVC → ViewerToken
              ↓ POST /platform/ingest      ↓ GET /platform/data
```

Continuous MQTT under wallet mode requires a **multi-use M2M ServiceVC**,
covered by a future hands-on.

### Phase 2 success criteria

- both `allowed` and `denied` can be reproduced
- learners can explain the difference between `purpose` and `dataset_id`
- learners can explain the role of `/audit/logs` and `/platform/ingest`
- learners can explain the **equivalence and differences** between
  the wallet path (Stage 1 / 3) and the JSON-registration path

## Phase 3: Intelligence Integration

### Goal of this phase

- convert a natural-language human request into a plan
- understand collection, evaluation, and control as one flow
- understand why the planner can be replaced by an LLM-based planner

### Matching hands-on pages

Phase 3 builds the **request → plan → execute** loop incrementally.
You first run a minimal rule-based loop, then swap the planner for
an LLM, then read the spec page.

#### What each stage adds and what you can learn

##### Stage 3A: minimal request → plan → execute loop (`regional-safety-assistant`)

- **Capabilities introduced**
    - The `assistant` service (`/plan`, `/execute` APIs)
    - **Rule-based** request → plan conversion
    - Planner reads events accumulated in Phase 2
      (`/audit/logs`, `/platform/ingest`)
    - Frontend demo (`assistant` web UI) visualizing plan results
- **What you learn**
    - The loop: request → plan → execute → verify
    - How the assistant consumes Phase 2 events as input
      (the dependency direction)
    - Where rule-based planners break down (branching, long context)

##### Stage 3B: replace the planner with an LLM (`llm-planner`)

- **Added on top of Stage 3A**
    - LLM-backed planner with the same I/O contract
    - Tool declarations the LLM can call (e.g. `/audit/logs` lookup)
    - `planner_diagnostics` exposing the LLM's reasoning trace
    - Coverage of natural-language variation and compound conditions
- **What you learn**
    - The benefit of "swap the planner under the same I/O contract"
    - How LLM tool calls (function calling) map to plan steps
    - How to read `planner_diagnostics` to follow the LLM's reasoning
    - **Where each fits**: when rule-based vs LLM is the right choice

##### Stage 3C: spec-level understanding (`llm-planner-spec`)

- **Added on top of Stage 3B** (docs only)
    - Documented I/O contract, error cases, and evaluation metrics
- **What you learn**
    - The specification surface you'd reproduce when building your
      own planner
    - The groundwork for hybrid or verified-LLM approaches beyond
      rule-based / LLM

##### (Future) Stage 3D: VC-gated plan execution (Stage 4, planned)

A future hands-on will tie the PolicyToken / ViewerToken introduced
in Phase 2 Stage 1/3 to plan execution: each tool call asserts the
required dataset scope via VC, so the planner can only invoke
operations the holder has been authorized for. Details will land in
a future page.

### Phase 3 success criteria

- a natural-language request can be converted into a `plan`
- `executed` can be confirmed when conditions are met
- the fields in `planner_diagnostics` can be explained
- results can be inspected from the frontend demo
- learners can explain the rule-based vs LLM trade-off in terms of
  I/O contract and fit

## Relation to Workshop

This chapter is the participant execution part of the overall flow described in [Workshop Overview](../workshop/index.md).  
Instructors and TAs are expected to explain the structure and branching in the Workshop chapter first, and then guide participants into the detailed pages here.
