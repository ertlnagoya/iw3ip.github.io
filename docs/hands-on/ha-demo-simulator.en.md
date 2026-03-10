# Home Assistant Demo Simulator Sample (Phase 1 / Phase 2 / Phase 3)

## Goal

This hands-on is a **minimum simulation environment for connecting Home Assistant and IW3IP without physical sensors or cameras**.

It uses Home Assistant `demo` entities to reproduce the following local flow:

`Home Assistant demo -> MQTT -> (optional Node-RED) -> Data Publisher -> Platform API / Audit Log`

This page uses one environment to show Phase 1 state sharing, Phase 2 event sharing, and a Phase 3 assistant execution flow.

## What this page helps you understand

- how to reproduce `temperature`, `power`, `person_detected`, `flood_risk_high`, and `possible_littering` without real devices
- how IW3IP publisher receives, normalizes, and records data coming from Home Assistant through MQTT
- the difference between Phase 1 state sharing and Phase 2 event sharing
- how Consent VC changes `allowed` and `denied`
- how to bridge `/platform/ingest` records into the Phase 3 `assistant`

## Common stumbling points

- starting Home Assistant alone is not enough; you must add the `MQTT` integration before the scripts can publish
- if Consent VCs are not registered, the scripts may run but the expected sharing behavior will not appear
- Home Assistant can publish an event correctly and the publisher can still reject it if `purpose` does not match
- because there are no physical devices, the flow is easier to lose sight of, so check both `/platform/ingest` and `/audit/logs`
- in Phase 3 it is easy to mix up sharing and execution, because publisher and assistant are separate services

## Prerequisites

- Docker / Docker Compose
- a browser that can open `http://localhost:8123`
- `curl`

This hands-on assumes the source-code repository `Blockchain_IoT_Marketplace` on branch `codex/ha-demo-simulator`.

Main matching files:

- [README (English)](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/README.md)
- [README (Japanese)](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/README_ja.md)
- [examples/ha_demo/README.md](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/README.md)
- [configuration.yaml](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/home-assistant-demo/config/configuration.yaml)
- [scripts.yaml](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/home-assistant-demo/config/scripts.yaml)
- [run_phase3_from_ingest.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/run_phase3_from_ingest.py)

## 1. Start services

Start the simulation environment.

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo up --build -d
```

If you also want Node-RED:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo --profile nodered up --build -d
```

Check:

```bash
curl http://localhost:8080/health
```

Expected:

```json
{"status":"ok","service":"publisher"}
```

## 2. Initial Home Assistant setup

Open:

- `http://localhost:8123`

On the first run, create a local user. Then add the `MQTT` integration.

Settings:

- Host: `mosquitto`
- Port: `1883`

The purpose of this step is to make `mqtt.publish` available from Home Assistant scripts.

## 3. Register Consent VCs

Register the Consent VCs used by the demo datasets.

```bash
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_temperature.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_power.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_person_detected.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_flood_risk_high.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_possible_littering.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_suspicious_activity.json
```

Expected:

- each response includes `"status":"stored"`

Matching files:

- [consent_temperature.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_temperature.json)
- [consent_power.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_power.json)
- [consent_person_detected.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_person_detected.json)
- [consent_flood_risk_high.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_flood_risk_high.json)
- [consent_possible_littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_possible_littering.json)
- [consent_suspicious_activity.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_suspicious_activity.json)

## 4. Send demo data from Home Assistant

In Home Assistant, open `Developer Tools -> Actions` and run these scripts:

- `script.iw3ip_publish_demo_temperature`
- `script.iw3ip_publish_demo_power`
- `script.iw3ip_publish_demo_person_detected`
- `script.iw3ip_publish_demo_flood_risk_high`
- `script.iw3ip_publish_demo_possible_littering`
- `script.iw3ip_publish_demo_suspicious_activity`
- `script.iw3ip_publish_demo_phase3_safety_scenario`

These scripts call `mqtt.publish` defined in `scripts.yaml`.

Phase mapping:

- Phase 1:
  - `temperature`
  - `power`
  - `person_detected`
- Phase 2:
  - `flood_risk_high`
  - `possible_littering`
  - `suspicious_activity`

## 5. Check publisher results

After running a Home Assistant script, inspect the publisher side.

```bash
curl http://localhost:8080/platform/ingest
curl http://localhost:8080/audit/logs?limit=10
```

Checkpoints:

- `platform/ingest` contains sent records
- `audit/logs` contains `allow`
- `dataset_id`, `purpose`, `message_hash`, and `raw_topic` are visible

The important point here is not only whether data was sent, but also under which conditions it was allowed.

## 6. Check a denied case

The Home Assistant scripts mainly exercise allowed paths, so use HTTP simulation to confirm denial.

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/flood_risk_high",
    "payload":{
      "event_type":"flood_risk_high",
      "data":{"zone":"river-west","severity":"high"},
      "ts":"2026-03-10T10:05:00Z",
      "source":"home_assistant_demo"
    },
    "purpose":"advertising"
  }'
```

Expected:

```json
{"status":"denied","dataset_id":"home/event/flood_risk_high","reason":"no_matching_consent"}
```

This makes it clear that generating an event and being allowed to share it are separate questions.

## 7. Test the MQTT path directly

You can also publish directly to the same topic without Home Assistant.

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/possible_littering \
  -m @examples/ha_demo/payload_possible_littering.json
```

Related files:

- [payload_temperature.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_temperature.json)
- [payload_power.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_power.json)
- [payload_person_detected.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_person_detected.json)
- [payload_flood_risk_high.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_flood_risk_high.json)
- [payload_possible_littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_possible_littering.json)

## 8. Try Phase 3

In Phase 3, the events accumulated by the publisher are converted into `assistant` `observed_events`, and then you inspect `plan -> execute`.

Start:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo-phase3 up --build -d
```

Then run this Home Assistant script:

- `script.iw3ip_publish_demo_phase3_safety_scenario`

That script publishes the following sequence for `park-north`:

- three `possible_littering` events
- one `suspicious_activity` event

Next, bridge the publisher events into the assistant:

```bash
python3 examples/ha_demo/run_phase3_from_ingest.py \
  --request-file examples/ha_demo/phase3_request_park_safety.json
```

Expected:

- `execution.evaluation.triggered` is `true`
- `actions_executed` includes `light_on` and `send_notification`

At this stage the key point is that event sharing can become the input to a later intelligence step.

## 9. If you want to use Node-RED

Node-RED is useful when you want easier manual injection or time-based pseudo events.

Import flow:

- [nodered_flows.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/nodered_flows.json)

Node-RED is optional in this setup.  
It is better to first understand the full path with Home Assistant demo alone, and only then add Node-RED if you want clearer event injection.

## 10. What is Phase 1, Phase 2, and Phase 3 here?

One important point of this sample is that one environment can show both Phase 1 and Phase 2.

- Phase 1:
  - state sharing such as `temperature` and `power`
  - basic receive, normalize, send, and record flow
- Phase 2:
  - event sharing such as `flood_risk_high` and `possible_littering`
  - conditional sharing based on `purpose` and Consent VC
- Phase 3:
  - pass publisher events to the assistant and execute the plan
  - inspect `triggered` and `actions_executed`

This page also works as a bridge between these two hands-on pages:

- [HA x SSI Publisher sample (Phase 1)](ha-ssi-publisher.md)
- [Environment and disaster event sharing sample (Phase 2)](environment-disaster.md)
- [Regional safety assistant sample (Phase 3)](regional-safety-assistant.md)

## 11. Stop services

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo down
```

If Node-RED is also running:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo --profile nodered down
```

If you also started the Phase 3 path:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo-phase3 down
```
