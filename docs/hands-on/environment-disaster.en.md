# Environment and Disaster Event Sharing Sample (Phase 2)

This hands-on is designed as a minimal **Phase 2: Event / Intelligence Sharing** exercise.  
Instead of sharing camera images or raw sensor streams, it shares disaster-related events generated at the edge.

This page uses the existing `HA x SSI Publisher` sample as the base and reproduces the following flow with simulated data:

`simulated sensor / edge inference -> disaster event -> Consent VC check -> allow / deny -> audit log`

## Shortest path

For a first pass, these four steps are enough.

1. start the publisher
2. register `consent_flood_risk_high.json`
3. send `flood_risk_high` with `purpose = disaster_response`
4. inspect `/platform/ingest` and `/audit/logs`

Branches after that:

- If you only want to confirm the allowed path: stop after the `allowed` case
- If you want to understand consent properly: also send `advertising` and confirm the denied case
- If you want to inspect the MQTT path too: finish with `mosquitto_pub`

## What this page helps you understand

- why Phase 2 moves from raw-data sharing to event sharing
- how `flood_risk_high` becomes `allowed` or `denied`
- how to read `/platform/ingest` and `/audit/logs` separately

## Common stumbling points

- `dataset_id` and `event_type` are easy to mix up
- the `purpose` in the request must match `allowed_purposes` in the Consent VC
- even when the flow is correct, `allowed` and `denied` are confirmed in different places

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

## Process table of contents

<details class="iw3ip-toc-details" open>
  <summary>Preparation: start services and register the Consent VC</summary>
  <p>First start the publisher and register the Consent VC used in this event-sharing scenario. These are the prerequisites for all later allow/deny checks.</p>
  <ol>
    <li><a href="#1-start-the-services">Start the services</a></li>
    <li><a href="#2-register-a-consent-vc-for-this-hands-on">Register a Consent VC for this hands-on</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 1: compare allowed and denied</summary>
  <p>Next, use the same `flood_risk_high` event to compare an allowed case and a denied case. This is where the role of `purpose` becomes explicit.</p>
  <ol>
    <li><a href="#3-reproduce-an-allowed-case">Reproduce an allowed case</a></li>
    <li><a href="#4-inspect-what-reached-the-platform-api">Inspect what reached the Platform API</a></li>
    <li><a href="#5-reproduce-a-denied-case">Reproduce a denied case</a></li>
    <li><a href="#6-check-the-audit-logs">Check the audit logs</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 2: reproduce the same event flow through MQTT</summary>
  <p>Finally, reproduce the same event-sharing flow through MQTT instead of HTTP simulation and confirm that the policy logic is the same.</p>
  <ol>
    <li><a href="#7-try-the-mqtt-path-as-well">Try the MQTT path as well</a></li>
  </ol>
</details>

## How to read this page

This page is one of the core Phase 2 examples, so it is intentionally detailed. If you only need a fast confirmation path, the `allowed` case is enough. However, the main educational point here is that the same event can be either shared or rejected depending on the purpose, so the page works best when you read through the denied case as well.

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

## Phase 2: Prepare the event-sharing baseline

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

At this point, the minimum setup for evaluating `flood_risk_high` is ready. The next step is to send the same event under two different purposes and compare the result.

## Phase 2: Compare allowed and denied

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

By this point, it should be clear that Phase 2 is not only about generating an event, but about being able to explain why the event is shared or rejected. The last step is to confirm the same behavior through the MQTT path.

## Phase 2: Confirm the same policy through MQTT

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

## Appendix: wallet-mode authorization

The main flow registers `consent_flood_risk_high.json` via POST to
`/consents` — i.e. consent is stored as plaintext JSON on the publisher.
The [Mobile SSI wallet hands-on](ha-ssi-wallet.md) takes a different
route: residents or observers hold a VC in their phone wallet and present
it via OID4VP. The publisher ships ConsentVC PDs for both
`home/event/flood_risk_high` (matches the main flow exactly) and
`home/env/flood_risk_high`, so disaster events can be gated through
the wallet using the same dataset as the main hands-on.

### What's the same, what's different

| Aspect | Main flow (`/consents` JSON) | Wallet mode (Stage 1: ConsentVC) |
| --- | --- | --- |
| Consent storage | publisher server | mobile wallet |
| Transport | JSON POST | OID4VP (QR / deeplink) |
| Validity window | `valid_to` TTL | short-lived PolicyToken per presentation (5 min, single-use) |
| Revocation | DELETE `/consents` | revoke VC / drop from wallet |
| Audit log | `purpose` / `dataset_id` | also `holder_did` / `vc_hash` |

The **decision logic is identical**: `(dataset_id, purpose)` decides
`allow` / `deny`. What changes is whether the consenter's identity is
backed by a verifiable credential and how the consent travels. In a
disaster-response setting, the resident's wallet asserts "I authorize
sharing of my district's data."

### How to try it (sketch)

Gating every MQTT event through the wallet doesn't fit PolicyToken's
single-use semantics. Instead, send **one event** by hand to feel the
flow:

1. Issue a ConsentVC for the same dataset the main flow uses:
   ```
   http://localhost:8080/issuer/offer?type=ConsentVC&dataset_id=home/event/flood_risk_high&purpose=disaster_response
   ```
   (allowed purposes: `disaster_response`, `safety`, `emergency_planning`)
2. Receive in the wallet, then present:
   ```
   http://localhost:8080/verifier/request?dataset_id=home/event/flood_risk_high&purpose=disaster_response
   ```
3. Take the PolicyToken from the response and POST a Stage-0-shaped event:
   ```bash
   curl -X POST http://localhost:8080/platform/ingest \
     -H "Authorization: Bearer $POLICY" \
     -H "Content-Type: application/json" \
     -d '{
       "dataset_id":"home/event/flood_risk_high",
       "purpose":"disaster_response",
       "event_type":"flood_risk_high",
       "data":{"sensor_id":"river-west-01","location":"west_river_area","water_level_m":1.82,"severity":"high"},
       "ts":"2026-04-28T10:17:05Z",
       "source":"edge_inference"
     }'
   ```
4. Confirm `/audit/logs` shows `reason=policy_token_consumed:<jti>`

### Limits of this appendix

- A ConsentVC PD for `home/event/flood_risk_high`
  ([consent-event-flood-risk-high.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/ssi_wallet/consent-event-flood-risk-high.json))
  is registered, so the wallet flow ingests the exact same event
  shape (sensor_id / water_level_m / severity) that the main flow uses.
- Continuous MQTT under wallet mode requires a **multi-use M2M token
  (ServiceVC)**, which is left to a future hands-on.

### Related

- Symmetric read-side gating: [SSI Viewer sample](ha-ssi-viewer.md)
