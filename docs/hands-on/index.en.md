# Hands-on

This section contains participant execution recipes.

## Learning Goals (University Students)

- explain behavior based on execution evidence
- explain `allowed` / `denied` by policy conditions
- explain on-chain/off-chain boundaries in the data flow

## Position of This Chapter

This chapter is where learners verify concepts by actually running the system after reading the foundation pages.  
Basic execution and validation should be possible using this site alone, without requiring external references.

Each page now also links to a matching `problem program`, `answer program`, and `exercise guide`.  
For workshop use, the intended flow is to start with the problem version and move to the answer or guide only when needed.

## Order

1. Choose one input source:
   - [HUSKYLENS2 sample](huskylens2.md)
   - [USB webcam sample](webcam.md)
   - [USB webcam event sharing sample (Phase 2)](webcam-event-sharing.md)
   - [HA x SSI Publisher sample](ha-ssi-publisher.md)
   - [Environment and disaster event sharing sample (Phase 2)](environment-disaster.md)
   - [Regional safety assistant sample (Phase 3)](regional-safety-assistant.md)
2. Validate the success criteria for the selected sample
3. Share outcomes, then move to the next sample if needed

## Common success criteria

- Camera paths (HUSKYLENS2 / Webcam):
  - event file is generated in `mediator-owner/raw_data/output`
  - merchandise appears in frontend and can be purchased
- Camera Phase 2 path:
  - `home/event/possible_littering` can be reproduced as an event-sharing dataset
  - learners can confirm `allowed` / `denied` using `community_cleaning` and `advertising`
- HA x SSI Publisher path:
  - both `allowed` and `denied` are reproducible via `/simulate/publish`
  - `/audit/logs` records `allow` / `deny` / `send_error`
- Non-camera Phase 2 path:
  - `home/event/flood_risk_high` can be reproduced as an event-sharing dataset
  - learners can confirm `allowed` / `denied` using `disaster_response` and `advertising`
- Phase 3 path:
  - a natural-language request can be converted into a `plan`
  - `light_on` and `send_notification` become `executed` when the condition is triggered
  - `/assistant/executions` can be used to inspect the execution history
