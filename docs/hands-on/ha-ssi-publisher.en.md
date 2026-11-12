# Pull from Home Assistant (HA SSI Publisher)

Build a minimal setup that receives Home Assistant state over MQTT and decides whether to share it with a Consent VC.

> **What you'll do**: MQTT → publisher → Consent VC check → audit log
>
> **Prerequisites**: [Prerequisites (install tools)](../setup/prerequisites.md), ideally [Run the whole pipeline without hardware (HA Demo Simulator)](ha-demo-simulator.md). The [Quickstart (run the stack)](../setup/quickstart.md) stack is not used
>
> **Keep running**: nothing. This page starts the publisher. If IPFS from Quickstart is running, stop it first (see [Prerequisites](#prerequisites) below)
>
> **What you need**: PC (Docker)
>
> **Time required**: approx. 45 min

!!! note "Where this page belongs"
    - This is section 1.2 of Part 1 ([Hands-on](index.md#12-capture-data-from-a-device)). Pick just one of the three pages in 1.2.
    - "Phase 1–3" on this page are development stages of the platform, not the learning order ([Hands-on](index.md#part-phase-stage)).
    - Previous page: [Run the whole pipeline without hardware (HA Demo Simulator)](ha-demo-simulator.md)
    - Next page: section 1.3 of [Hands-on](index.md#13-trade-on-the-marketplace-v1-lane) (trade on the marketplace)

## Goal

This covers a minimal setup that receives Home Assistant data via MQTT,
decides whether to allow it with a Consent VC, then forwards only allowed data and records audit logs.

Pipeline:

`Home Assistant -> MQTT -> (optional Node-RED) -> Data Publisher -> Platform API`

## Shortest path

For a first pass, these four steps are enough.

1. `PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml up --build -d`
2. Register `consent_temperature.json`
3. Send the `temperature` request through `/simulate/publish`
4. Inspect `/audit/logs` and confirm `allow`

Branches after that:

- If you only want HTTP simulation: compare one `allowed` and one `denied` case
- If you want to inspect MQTT ingestion as well: publish `person_detected` with `mosquitto_pub`
- If you want to move into Phase 2: continue with the `flood_risk_high` or `possible_littering` hands-on pages

## What this page covers

- the minimal Phase 1 setup: where data is received, where it is decided, and where it is recorded
- how to assemble `topic`, `payload`, and `purpose` into one request
- how to read `allowed`, `denied`, and the audit log

## Common issues

- the relation between `topic` and `dataset_id` is hard to grasp at first
- a Consent VC can exist and still produce `denied` if `purpose` does not match
- HTTP simulation and MQTT ingestion are two different checkpoints

## Prerequisites

- Docker / Docker Compose
- `curl`
- if IPFS started in [Quickstart (run the stack)](../setup/quickstart.md) is still running, stop it first from the source repository with `cd ~/program/Blockchain_IoT_Marketplace/ipfs && docker compose down` (IPFS and the publisher both use port 8080)

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

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase1_ha_ssi_publisher/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase1_ha_ssi_publisher/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase1_ha_ssi_publisher/README.md)

This exercise asks learners to construct the minimum request body for `/simulate/publish`.  
The problem program focuses on how `topic`, `payload`, and `purpose` are combined into one request.

## Phase 1: Confirm the minimal setup

### 1. Start services

`PLATFORM_INGEST_READ_ENABLED=true` enables `GET /platform/ingest`, which you use in step 6 to check the delivered data (it is a demo-only API that returns everything without authentication, so it is off by default).

```bash
cd ~/program/Blockchain_IoT_Marketplace
PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml up --build -d
```

Health check:

```bash
curl http://localhost:8080/health
```

Expected:

```json
{"status":"ok","service":"publisher"}
```

### 2. Register Consent VCs

```bash
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_temperature.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_power.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_person_detected.json
```

Additional files used by the Phase 2 hands-on pages:

- `examples/consent_flood_risk_high.json`
- `examples/consent_possible_littering.json`

The setup needed for the checks is now complete. Next, compare a request that is allowed and a request that is denied.

## Phase 1: Compare allowed and denied

### 3. Allowed case

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

### 4. Denied case

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

A request with the correct structure is still rejected when `purpose` does not match. Next, check the same decision through the MQTT path.

## Phase 1: Inspect MQTT ingestion and audit logs

### 5. Test MQTT ingestion path

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/person_detected \
  -m '{"event_type":"person_detected","data":{"camera_id":"front_door","confidence":0.93},"ts":"2026-02-28T10:00:20Z","source":"edge_inference"}'
```

### 6. Check audit logs

```bash
curl 'http://localhost:8080/audit/logs?limit=10'
```

Checkpoints:

- actions include `allow`, `deny`, and/or `send_error`
- each record includes `dataset_id`, `purpose`, and `message_hash`

Also confirm that the allowed data reached the destination (Platform API).

```bash
curl http://localhost:8080/platform/ingest
```

Checkpoints:

- the data from step 3 (`"dataset_id":"home/env/temperature"`) is included (the denied `home/energy/power` from step 4 is not)
- if `{"detail":"platform_ingest_read_disabled"}` is returned, `PLATFORM_INGEST_READ_ENABLED=true` was missing in step 1. Restart with it (see "`GET /platform/ingest` returns 404" in [Troubleshooting](../operations/troubleshooting.md)). The restart clears the data held in the publisher's memory, so send step 3 again before checking

### 7. Stop services

```bash
docker compose -f infra/docker-compose.yml down
```

## Extension hints

- Phase 2: prioritize `homeassistant/event/...` for event-driven sharing
- Phase 3: add SSI Gateway (PEP) before publisher for VC presentation control

If you want to move directly into Phase 2, continue with:

- [Share disaster events](environment-disaster.md) (Phase 2)
- [Share detections, not raw frames](webcam-event-sharing.md) (Phase 2)

## Next steps

One page from 1.2 is enough. Next is section 1.3 of [Hands-on](index.md#13-trade-on-the-marketplace-v1-lane), where you list an item, buy it, and receive it.

- 1.3 does not use the publisher. First stop it with the command in [7. Stop services](#7-stop-services).
- 1.3 uses the [Quickstart (run the stack)](../setup/quickstart.md) stack (terminals 1–7). Start it after the publisher is stopped, because the Quickstart IPFS and the publisher both use port 8080.
- If you chose this page in 1.2, list an item in 1.3 by running the sample of [Detect with a USB webcam (OpenCV)](webcam.md) in mock mode.
