# USB Webcam Event Sharing Sample (Phase 2)

This hands-on is a **Phase 2: Event / Intelligence Sharing** extension of the existing [USB Webcam sample](webcam.md).  
In Phase 1, the USB webcam sample detected `person_detected` and `possible_littering`, generated event files, and connected them to productization.  
This page moves to the next step: **sending detected events into the data-sharing platform and controlling sharing based on Consent VC**.

Pipeline:

`USB webcam -> webcam-bridge -> event -> Publisher -> Consent VC check -> Platform API / Audit Log`

## What this hands-on demonstrates

- how to extend a Phase 1 "detect and visualize" flow into a Phase 2 "share detected events" flow
- why it is useful to share only events such as `person_detected` or `possible_littering`, instead of sharing camera images
- how the same event can become `allowed` or `denied` depending on `purpose`
- how to inspect sharing results and audit results separately through `/platform/ingest` and `/audit/logs`

## Prerequisites

- you understand the [USB Webcam sample](webcam.md)
- you can run the [HA x SSI Publisher sample](ha-ssi-publisher.md)
- Docker / Docker Compose is available
- `curl` is available

Matching sample files:

- `examples/consent_possible_littering.json`
- `examples/payload_possible_littering.json`

Exercise programs:

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_webcam_event_sharing/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_webcam_event_sharing/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_webcam_event_sharing/README.md)

This exercise asks learners to write the code that sends a `possible_littering` event to `/simulate/publish`.  
The problem program makes it clear that a Phase 1 detection event becomes a Phase 2 condition-controlled sharing event.

## Scenario

Suppose a community camera detects possible littering in a public park.  
The system follows this policy:

- do not share the camera video itself
- share only the `possible_littering` event
- allow sharing for community cleaning or research
- deny sharing for advertising

The shared event may look like this:

```json
{
  "event_type": "possible_littering",
  "data": {
    "camera_id": "webcam-401",
    "location": "park-north",
    "object_class": "bottle",
    "confidence": 0.87
  },
  "ts": "2026-03-08T11:00:00Z",
  "source": "edge_inference"
}
```

When sent to `homeassistant/event/possible_littering`, the Publisher normalizes it to `dataset_id = home/event/possible_littering`.

The same event content is also available in `examples/payload_possible_littering.json` in the source repository.

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

This hands-on allows `home/event/possible_littering` only for `community_cleaning` and `research`.

The same consent content is also available in `examples/consent_possible_littering.json` in the source repository.

```bash
curl -X POST http://localhost:8080/consents \
  -H 'Content-Type: application/json' \
  -d '{
    "vc_id": "consent-littering-001",
    "subject_did": "did:example:park-community",
    "dataset_id": "home/event/possible_littering",
    "allowed_purposes": ["community_cleaning", "research"],
    "retention_days": 14,
    "reshare_allowed": false,
    "valid_from": "2026-02-01T00:00:00Z",
    "valid_to": "2026-05-01T00:00:00Z",
    "signature": "PLACEHOLDER"
  }'
```

Check:

```bash
curl http://localhost:8080/consents
```

## 3. Reproduce an allowed case

Inject the littering event with `purpose = community_cleaning`.

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/possible_littering",
    "payload":{
      "event_type":"possible_littering",
      "data":{
        "camera_id":"webcam-401",
        "location":"park-north",
        "object_class":"bottle",
        "confidence":0.87
      },
      "ts":"2026-03-08T11:00:00Z",
      "source":"edge_inference"
    },
    "purpose":"community_cleaning"
  }'
```

Expected:

```json
{"status":"allowed","dataset_id":"home/event/possible_littering"}
```

## 4. Inspect what reached the Platform API

```bash
curl http://localhost:8080/platform/ingest
```

Checkpoints:

- `dataset_id` is `home/event/possible_littering`
- `purpose` is `community_cleaning`
- `payload.message_type` is `event`
- `payload.payload.event_type` is `possible_littering`

## 5. Reproduce a denied case

Send the same event again, but with `purpose = advertising`.

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/possible_littering",
    "payload":{
      "event_type":"possible_littering",
      "data":{
        "camera_id":"webcam-401",
        "location":"park-north",
        "object_class":"bottle",
        "confidence":0.87
      },
      "ts":"2026-03-08T11:01:00Z",
      "source":"edge_inference"
    },
    "purpose":"advertising"
  }'
```

Expected:

```json
{"status":"denied","dataset_id":"home/event/possible_littering","reason":"no_matching_consent"}
```

## 6. Check the audit logs

```bash
curl http://localhost:8080/audit/logs?limit=10
```

Checkpoints:

- both `allow` and `deny` are recorded
- `dataset_id` is `home/event/possible_littering`
- `purpose` is `community_cleaning` or `advertising`
- `message_hash` is recorded

## 7. Try the MQTT path as well

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/possible_littering \
  -m '{"event_type":"possible_littering","data":{"camera_id":"webcam-401","location":"park-north","object_class":"bottle","confidence":0.87},"ts":"2026-03-08T11:02:00Z","source":"edge_inference"}'
```

Then check the audit logs again:

```bash
curl http://localhost:8080/audit/logs?limit=10
```

## What changed from Phase 1

In the Phase 1 USB webcam sample, the main focus was:

1. detect a situation with the camera
2. create an event file
3. connect it to the productization flow

In this Phase 2 sample, the focus shifts to:

1. generate an event at the edge
2. decide whether that event may be shared
3. share only the permitted event
4. keep an auditable record of the sharing decision

The key change is that the theme moves from **detection** to **condition-based sharing**.

## Success criteria

- `home/event/possible_littering` is handled as an event-sharing dataset
- `community_cleaning` becomes `allowed`
- `advertising` becomes `denied`
- allowed events appear in `/platform/ingest`
- `allow` / `deny` records appear in `/audit/logs`

## Extension directions

1. combine `person_detected` and `possible_littering` to define richer sharing conditions
2. split sharing destinations across community cleaning, research, and municipality workflows
3. make the policy explicit that raw video is not shared and only events are shared
4. connect the results to the mobile viewer app for event lists and notifications
