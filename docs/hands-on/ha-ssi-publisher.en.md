# HA x SSI Publisher Sample (Phase 1)

## Goal

This hands-on shows the minimum pipeline that receives Home Assistant data via MQTT,
applies Consent VC policy checks, forwards only allowed data, and stores audit logs.

Pipeline:

`Home Assistant -> MQTT -> (optional Node-RED) -> Data Publisher -> Platform API`

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
