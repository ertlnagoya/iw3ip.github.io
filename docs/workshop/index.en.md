# Workshop

This chapter is for instructors and TAs to decide **how to explain the full picture and how to route participants into the correct hands-on pages**.  
The intended flow is to explain the whole structure first, and then guide learners into Phase 1, 2, or 3 depending on time, equipment, and learning goals.

## Workshop and Hands-on

- **Workshop**: overall explanation, learning order, time allocation, branching, and wrap-up
- **Hands-on**: actual participant execution steps

The basic flow is:

1. explain the overall structure in the Workshop
2. move participants to the appropriate Hands-on page
3. collect outcomes
4. return to the Workshop for interpretation and next steps

## The overall picture that should be explained first

IW3IP becomes much easier to understand when the workshop is framed in three phases.

| Phase | What the instructor should emphasize | What participants will experience |
|---|---|---|
| Phase 1: Data Exchange | where data is generated and where it moves | generate data from cameras or sensors and inspect the result |
| Phase 2: Event Sharing | why the system moves from raw data to event sharing | inspect Consent VC, purpose, and audit logging |
| Phase 3: Intelligence Integration | how a human request becomes a plan and execution flow | inspect request -> plan -> execute -> UI |

## Phase Cards for Facilitation

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Phase 1 / Data Exchange</div>
    <h3>📡 Best for the first practical step</h3>
    <p>This phase helps participants see where data is created and where it moves.</p>
    <p class="iw3ip-phase-links"><a href="../hands-on/huskylens2.md">HUSKYLENS2</a> / <a href="../hands-on/webcam.md">USB Webcam</a> / <a href="../hands-on/ha-ssi-publisher.md">HA x SSI Publisher</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Phase 2 / Event Sharing</div>
    <h3>🧭 Best for explaining conditional sharing</h3>
    <p>This phase is where instructors explain consent, purpose, and audit as reasons for allow and deny decisions.</p>
    <p class="iw3ip-phase-links"><a href="../hands-on/webcam-event-sharing.md">Webcam Event Sharing</a> / <a href="../hands-on/environment-disaster.md">Environment Disaster</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Phase 3 / Intelligence Integration</div>
    <h3>🧠 Best for showing the full intelligent flow</h3>
    <p>This phase shows request, planning, execution, and the frontend demo as one connected flow.</p>
    <p class="iw3ip-phase-links"><a href="../hands-on/regional-safety-assistant.md">Regional Safety Assistant</a> / <a href="../hands-on/llm-planner.md">LLM Planner</a></p>
  </div>
</div>

## Recommended explanation order

### Introduction

Start by explaining:

- that the workshop grows from Phase 1 to Phase 2 to Phase 3
- that each phase has a different learning target
- that learners will branch depending on devices and available time

### What should be clear before the exercise starts

- what devices are available
- how far the session should go
  - Phase 1 only
  - up to Phase 2
  - up to Phase 3
- whether the frontend demo will be shown

## Recommended timeline (120 minutes)

1. 0-15 min: explain the full picture
   - difference between Phase 1 / 2 / 3
   - how far today's session will go
2. 15-30 min: environment check
3. 30-55 min: Phase 1
4. 55-85 min: Phase 2
5. 85-110 min: Phase 3 or frontend demo
6. 110-120 min: wrap-up

## Destination pages by phase

### Phase 1: Data Exchange

These are the best starting points for first-time learners.

- with HUSKYLENS2: [HUSKYLENS2 sample](../hands-on/huskylens2.md)
- with only a USB camera: [USB webcam sample](../hands-on/webcam.md)
- to show the Home Assistant and SSI flow: [HA x SSI Publisher sample](../hands-on/ha-ssi-publisher.md)
- to show the viewer side as well: [Mobile Viewer](../hands-on/mobile-viewer.md)

### Phase 2: Event Sharing

This phase explains conditional sharing and audit after Phase 1.

- camera-based event sharing: [USB webcam event sharing sample (Phase 2)](../hands-on/webcam-event-sharing.md)
- non-camera event sharing: [Environment and disaster event sharing sample (Phase 2)](../hands-on/environment-disaster.md)

### Phase 3: Intelligence Integration

This phase covers natural-language requests, plans, execution, and the frontend demo.

- core structure: [Regional safety assistant sample (Phase 3)](../hands-on/regional-safety-assistant.md)
- practical LLM path: [LLM Planner hands-on](../hands-on/llm-planner.md)
- design understanding: [LLM Planner replacement spec](../hands-on/llm-planner-spec.md)

## Shortest demo path

If time is limited or you want to show the whole picture first, `assistant-demo` is the clearest path.

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

This starts:

- `assistant-demo`
- `llm-mock`
- `assistant-ui`

Matching page:

- [LLM Planner hands-on](../hands-on/llm-planner.md)

## Related pages

- [Workshop / Prerequisites](prerequisites.md)
- [Workshop / Quickstart](quickstart.md)
- [Workshop / Facilitator Guide](facilitator-guide.md)
- [Hands-on / Overview](../hands-on/index.md)
- [Course Guide](../foundations/course-guide.md)
- [Learning Roadmap](../foundations/roadmap.md)
