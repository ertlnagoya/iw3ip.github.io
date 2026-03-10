# HA x SSI Publisher Sample (Phase 1)

## Goal

This hands-on shows the minimum pipeline that receives Home Assistant data via MQTT,
applies Consent VC policy checks, forwards only allowed data, and stores audit logs.

Pipeline:

`Home Assistant -> MQTT -> (optional Node-RED) -> Data Publisher -> Platform API`

## Shortest path

For a first pass, these four steps are enough.

1. `docker compose -f infra/docker-compose.yml up --build -d`
2. Register `consent_temperature.json`
3. Send the `temperature` request through `/simulate/publish`
4. Inspect `/audit/logs` and confirm `allow`

Branches after that:

- If you only want HTTP simulation: compare one `allowed` and one `denied` case
- If you want to inspect MQTT ingestion as well: publish `person_detected` with `mosquitto_pub`
- If you want to move into Phase 2: continue with the `flood_risk_high` or `possible_littering` hands-on pages

## What this page helps you understand

- the minimum Phase 1 flow: where data is received, checked, and recorded
- how `topic`, `payload`, and `purpose` form one request
- how to read `allowed`, `denied`, and the audit log

## Common stumbling points

- the relation between `topic` and `dataset_id` is easy to miss at first
- a Consent VC can exist and still produce `denied` if `purpose` does not match
- HTTP simulation and MQTT ingestion are two different checkpoints

## Prerequisites

- Docker / Docker Compose
- `curl`

Matching sample files:

- `examples/consent_temperature.json`
- `examples/consent_power.json`
- `examples/consent_person_detected.json`
- `examples/consent_flood_risk_high.json`
- `examples/consent_possible_littering.json`
- `examples/payload_temperature.json`
- `examples/payload_power.json`
- `examples/payload_person_detected.json`
- `examples/payload_flood_risk_high.json`
- `examples/payload_possible_littering.json`

Exercise programs:

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase1_ha_ssi_publisher/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase1_ha_ssi_publisher/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase1_ha_ssi_publisher/README.md)

This exercise asks learners to construct the minimum request body for `/simulate/publish`.  
The problem program focuses on how `topic`, `payload`, and `purpose` are combined into one request.

## Process table of contents

<details class="iw3ip-toc-details" open>
  <summary>Preparation: start the publisher and register Consent VCs</summary>
  <p>First confirm that the publisher is running and then register the minimum Consent VCs. These steps are the prerequisites for the later allow/deny checks.</p>
  <ol>
    <li><a href="#1-start-services">Start services</a></li>
    <li><a href="#2-register-consent-vcs">Register Consent VCs</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 1: compare allowed and denied through HTTP simulation</summary>
  <p>Next, compare an allowed request and a denied request through the same API entry point. This is where the relation between `purpose` and `dataset_id` becomes clear.</p>
  <ol>
    <li><a href="#3-allowed-case">Allowed case</a></li>
    <li><a href="#4-denied-case">Denied case</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 2: inspect MQTT ingestion and audit logs</summary>
  <p>Finally, send data through MQTT instead of HTTP simulation and check what remains in the audit log.</p>
  <ol>
    <li><a href="#5-test-mqtt-ingestion-path">Test MQTT ingestion path</a></li>
    <li><a href="#6-check-audit-logs">Check audit logs</a></li>
    <li><a href="#7-stop-services">Stop services</a></li>
  </ol>
</details>

## How to read this page

This page is meant to explain the Phase 1 baseline carefully. If you only need a short confirmation path, `Shortest path` is enough. If you want to understand consent-based sharing properly, it is better to compare both `allowed` and `denied` before moving on to the MQTT path.

## Phase 1: Confirm the minimum baseline

## 1. Start services

```bash
docker compose -f infra/docker-compose.yml up --build -d
```

Health check:

```bash
curl http://localhost:8080/health
```

Expected:

```json
{"status":"ok","service":"publisher"}
```

## 2. Register Consent VCs

```bash
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_temperature.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_power.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_person_detected.json
```

Additional files used by the Phase 2 hands-on pages:

- `examples/consent_flood_risk_high.json`
- `examples/consent_possible_littering.json`

At this point the minimum policy setup is ready. The next step is to compare a request that is allowed and a request that is denied through the same API entry point.

## Phase 1: Compare allowed and denied

## 3. Allowed case

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/state/sensor/temperature",
    "payload":{
      "entity_id":"sensor.living_room_temperature",
      "state":"24.1",
      "attributes":{"unit_of_measurement":"C"},
      "ts":"2026-02-28T10:00:00Z",
      "source":"home_assistant"
    },
    "purpose":"research"
  }'
```

Expected:

```json
{"status":"allowed","dataset_id":"home/env/temperature"}
```

## 4. Denied case

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/state/sensor/power",
    "payload":{
      "entity_id":"sensor.main_power",
      "state":"520",
      "attributes":{"unit_of_measurement":"W"},
      "ts":"2026-02-28T10:00:10Z",
      "source":"home_assistant"
    },
    "purpose":"marketing"
  }'
```

Expected:

```json
{"status":"denied","dataset_id":"home/energy/power","reason":"no_matching_consent"}
```

The important point here is that a request can have the correct structure and still be rejected when `purpose` does not match. Next, inspect the same overall behavior through the MQTT path.

## Phase 1: Inspect MQTT ingestion and audit logs

## 5. Test MQTT ingestion path

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/person_detected \
  -m '{"event_type":"person_detected","data":{"camera_id":"front_door","confidence":0.93},"ts":"2026-02-28T10:00:20Z","source":"edge_inference"}'
```

## 6. Check audit logs

```bash
curl http://localhost:8080/audit/logs?limit=10
```

Checkpoints:

- actions include `allow`, `deny`, and/or `send_error`
- each record includes `dataset_id`, `purpose`, and `message_hash`

## 7. Stop services

```bash
docker compose -f infra/docker-compose.yml down
```

## Extension hints

- Phase 2: prioritize `homeassistant/event/...` for event-driven sharing
- Phase 3: add SSI Gateway (PEP) before publisher for VC presentation control

If you want to move directly into Phase 2, continue with:

- [Environment and disaster event sharing sample (Phase 2)](environment-disaster.md)
- [USB webcam event sharing sample (Phase 2)](webcam-event-sharing.md)
