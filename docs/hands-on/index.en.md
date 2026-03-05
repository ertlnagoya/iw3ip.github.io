# Hands-on

This section contains participant execution recipes.

## Learning Goals (University Students)

- explain behavior based on execution evidence
- explain `allowed` / `denied` by policy conditions
- explain on-chain/off-chain boundaries in the data flow

## Order

1. Choose one input source:
   - [HUSKYLENS2 sample](huskylens2.md)
   - [USB webcam sample](webcam.md)
   - [HA x SSI Publisher sample](ha-ssi-publisher.md)
2. Validate the success criteria for the selected sample
3. Share outcomes, then move to the next sample if needed

## Common success criteria

- Camera paths (HUSKYLENS2 / Webcam):
  - event file is generated in `mediator-owner/raw_data/output`
  - merchandise appears in frontend and can be purchased
- HA x SSI Publisher path:
  - both `allowed` and `denied` are reproducible via `/simulate/publish`
  - `/audit/logs` records `allow` / `deny` / `send_error`
