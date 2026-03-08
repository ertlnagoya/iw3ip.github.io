# Environment and Disaster Event Sharing Sample (Phase 2)

This hands-on is designed as a minimal **Phase 2: Event / Intelligence Sharing** exercise.  
Instead of sharing camera images or raw sensor streams, it shares disaster-related events generated at the edge.

This page uses the existing `HA x SSI Publisher` sample as the base and reproduces the following flow with simulated data:

`simulated sensor / edge inference -> disaster event -> Consent VC check -> allow / deny -> audit log`

## What this hands-on demonstrates

- how Phase 1 data sharing can be extended into Phase 2 event sharing
- why sharing events can be more practical than sharing raw data
- how `purpose` changes the result between `allowed` and `denied`
- how to inspect `/platform/ingest` and `/audit/logs` separately

## Prerequisites

- you can run the [HA x SSI Publisher sample](ha-ssi-publisher.md)
- Docker / Docker Compose is available
- `curl` is available

This hands-on assumes the source-code repository `Blockchain_IoT_Marketplace` on branch `codex/ha-ssi-publisher-sample`.

Matching sample files:

- `examples/consent_flood_risk_high.json`
- `examples/payload_flood_risk_high.json`

Exercise programs:

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_environment_disaster/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_environment_disaster/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_environment_disaster/README.md)

This exercise asks learners to complete the request that sends a `flood_risk_high` event to `/simulate/publish`.  
The problem program makes the `allowed` / `denied` difference visible at code level by changing only the `purpose`.

## Scenario

Assume an edge sensor near a river generates a `flood_risk_high` event based on rising water level and surrounding conditions.  
The event may be shared for municipal disaster response or research, but should not be shared for advertising.

Instead of sharing the raw sensor stream, we share an event like this:

```json
{
  "event_type": "flood_risk_high",
  "data": {
    "sensor_id": "river-west-01",
    "location": "west_river_area",
    "water_level_m": 1.82,
    "severity": "high"
  },
  "ts": "2026-03-08T10:15:05Z",
  "source": "edge_inference"
}
```

When this event is sent to `homeassistant/event/flood_risk_high`, the Publisher normalizes it to `dataset_id = home/event/flood_risk_high`.

The same event content is also available in `examples/payload_flood_risk_high.json` in the source repository.

## 1. Start the services

Run the following in the source-code repository:

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

## 2. Register a Consent VC for this hands-on

This hands-on allows `home/event/flood_risk_high` only for `disaster_response` and `research`.

The same consent content is also available in `examples/consent_flood_risk_high.json` in the source repository.

```bash
curl -X POST http://localhost:8080/consents \
  -H 'Content-Type: application/json' \
  -d '{
    "vc_id": "consent-flood-risk-001",
    "subject_did": "did:example:river-community",
    "dataset_id": "home/event/flood_risk_high",
    "allowed_purposes": ["disaster_response", "research"],
    "retention_days": 30,
    "reshare_allowed": false,
    "valid_from": "2026-02-01T00:00:00Z",
    "valid_to": "2026-05-01T00:00:00Z",
    "signature": "PLACEHOLDER"
  }'
```

Expected:

```json
{"status":"stored", ...}
```

Check:

```bash
curl http://localhost:8080/consents
```

## 3. Reproduce an allowed case

Inject a disaster event with `purpose = disaster_response`.

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/flood_risk_high",
    "payload":{
      "event_type":"flood_risk_high",
      "data":{
        "sensor_id":"river-west-01",
        "location":"west_river_area",
        "water_level_m":1.82,
        "severity":"high"
      },
      "ts":"2026-03-08T10:15:05Z",
      "source":"edge_inference"
    },
    "purpose":"disaster_response"
  }'
```

Expected:

```json
{"status":"allowed","dataset_id":"home/event/flood_risk_high"}
```

## 4. Inspect what reached the Platform API

If the event is allowed, it is forwarded to the dummy Platform API inside the Publisher.

```bash
curl http://localhost:8080/platform/ingest
```

Checkpoints:

- `dataset_id` is `home/event/flood_risk_high`
- `purpose` is `disaster_response`
- `payload.message_type` is `event`
- `payload.payload.event_type` is `flood_risk_high`

## 5. Reproduce a denied case

Now send the same event with `purpose = advertising`.

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/flood_risk_high",
    "payload":{
      "event_type":"flood_risk_high",
      "data":{
        "sensor_id":"river-west-01",
        "location":"west_river_area",
        "water_level_m":1.82,
        "severity":"high"
      },
      "ts":"2026-03-08T10:16:05Z",
      "source":"edge_inference"
    },
    "purpose":"advertising"
  }'
```

Expected:

```json
{"status":"denied","dataset_id":"home/event/flood_risk_high","reason":"no_matching_consent"}
```

## 6. Check the audit logs

```bash
curl http://localhost:8080/audit/logs?limit=10
```

Checkpoints:

- both `allow` and `deny` are recorded
- `dataset_id` is `home/event/flood_risk_high`
- `purpose` is `disaster_response` or `advertising`
- `message_hash` is recorded

## 7. Try the MQTT path as well

You can reproduce the same event-sharing flow through MQTT instead of HTTP simulation.

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/flood_risk_high \
  -m '{"event_type":"flood_risk_high","data":{"sensor_id":"river-west-01","location":"west_river_area","water_level_m":1.82,"severity":"high"},"ts":"2026-03-08T10:17:05Z","source":"edge_inference"}'
```

Then check the audit logs again:

```bash
curl http://localhost:8080/audit/logs?limit=10
```

## Success criteria

This hands-on is successful if you can confirm all of the following:

- `home/event/flood_risk_high` is handled as an event-sharing dataset
- `disaster_response` becomes `allowed`
- `advertising` becomes `denied`
- allowed events appear in `/platform/ingest`
- both allow and deny outcomes appear in `/audit/logs`

## Why this is Phase 2

Phase 1 focused mainly on state data such as temperature and power.  
This hands-on moves to the next step: **sharing events interpreted at the edge instead of sharing raw data directly**.

The focus therefore changes as follows:

- Phase 1: `state` and raw-like data
- Phase 2: `event` and inference-oriented outputs

## Natural extensions

1. add `high_co2` or `abnormal_vibration` using the same pattern
2. keep water-level values private and share only events
3. connect the events to map visualization or notification apps
4. split Consent VCs by sharing destination
