# SSI Service Sample (Phase 2 / Stage 4 prep)

!!! note "M2M extension of the wallet hands-on"
    Assumes you've completed Stage 1 (ConsentVC + PolicyToken) and
    Stage 3 (ViewerVC + ViewerToken). This page covers the case where
    a **non-human service** holds a VC and writes continuously.

## Goal

ConsentVC's PolicyToken is **5 min, single-use** — designed for a human
consenting to one share at a time. But a **continuous publisher
(MQTT-driven)** can't tolerate re-presentation per event.

ServiceVC encodes "this service may write continuously," and
presentation yields a ServiceToken valid for 1 hour with multiple uses.
A publisher process holds this token and attaches it as Bearer to
every `/platform/ingest` call.

```
[service]  -- ServiceVC presentation (once) -> [Verifier]
                                                  |
                                            ServiceToken (1h, multi)
[service]  <- ServiceToken ----------------------+
[service]  -- POST /platform/ingest x N --------> [Publisher]
                                                  authz via ServiceToken
```

## What you'll learn

- Why "human VC (ConsentVC)" and "service VC (ServiceVC)" are split
- The three token shapes: single-use write, multi-use read,
  multi-use continuous write
- ServiceToken's 1-hour TTL with `write_count` accumulation
- How `/platform/ingest` accepts both PolicyToken and ServiceToken
  (PolicyToken first, fall through to ServiceToken on "unknown")

## Common pitfalls

- ConsentVC / ViewerVC / ServiceVC are **distinct**; cross-use is denied
- ServiceToken differs from ViewerToken in TTL and target endpoint;
  it works only on `/platform/ingest` (the read side `/platform/data`
  takes ViewerToken only)
- ServiceVC is M2M-oriented. It can be held in a phone wallet for
  experimentation, but properly belongs to a holder embedded in the
  publisher

## Prerequisites

- Completed [SSI Wallet hands-on (Stage 1)](ha-ssi-wallet.md)
- Publisher running with `feat/ssi-service-vc` or later
- iw3ip-wallet available if you want to hold ServiceVC on a phone

## 1. Start and inspect metadata

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet up --build -d
PUB=$(docker ps -qf name=publisher)

# Confirm ServiceVC is registered (the current release exposes 5:
# ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC)
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer | python3 -m json.tool | grep -A2 ServiceVC
```

Expected: `"vct": "https://iw3ip.example/credentials/ServiceVC/v1"` appears.

## 2. Issue a ServiceVC (substitute = phone wallet)

Production would issue to a publisher-embedded holder. For the hands-on,
the phone wallet is easiest. Open in a PC browser:

```
http://192.168.68.53:8080/issuer/offer?type=ServiceVC&dataset_id=home/env/temperature&purpose=write_continuous
```

Send the deeplink to the wallet via AirDrop and store the credential.

Claims:
- `dataset_id`: `home/env/temperature`
- `allowed_actions`: `["write_continuous"]`

## 3. Present the ServiceVC

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=ServiceVC
```

`vc_kind=ServiceVC` is required; without it the ConsentVC PD is selected.

Right after presentation, the publisher log shows:

```
service_token_issued jti=... token=... dataset=home/env/temperature ttl=3600s
```

`token=...` is the ServiceToken.

## 4. Continuous ingest with ServiceToken

```bash
SERVICE=<service_token>

for i in 1 2 3 4 5; do
  curl -s -X POST http://192.168.68.53:8080/platform/ingest \
    -H "Authorization: Bearer $SERVICE" \
    -H "Content-Type: application/json" \
    -d "{\"dataset_id\":\"home/env/temperature\",\"value\":$((20 + i))}" \
    | python3 -c "import json,sys; print(json.load(sys.stdin))"
done
```

Expected: all 5 return `{"status":"received","count":N}`.

Unlike PolicyToken, the second call doesn't trigger `403 already_consumed`.

## 5. Inspect write_count in the audit log

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool | head -60
```

Each ingest produces:

```json
{
  "action": "allow",
  "raw_topic": "platform/ingest",
  "reason": "service_token_used:<jti>:1",
  "dataset_id": "home/env/temperature",
  "purpose": "write_continuous",
  "holder_did": "did:jwk:...",
  "presentation_verified": "allow"
}
```

The trailing number in `reason` is `write_count`. Five calls produce 1〜5.

## 6. Error cases

| Situation | HTTP | `detail` |
| --- | --- | --- |
| Expired (1 h elapsed) | 401 | `service_token_expired` |
| Dataset mismatch | 403 | `service_token_dataset_mismatch` |
| Unknown token | 401 | `policy_token_unknown` (Stage 1 compat) |

The unknown-token case returns `policy_token_*` rather than `service_*`
on purpose — to preserve the Stage 1 user-visible error name.

## Three VCs side by side

| | ConsentVC (Stage 1) | ViewerVC (Stage 3) | ServiceVC (Stage 4 prep) |
| --- | --- | --- | --- |
| Purpose | write authz | read authz | M2M continuous write |
| Holder | human user | human viewer | service (e.g. publisher) |
| Token | PolicyToken | ViewerToken | ServiceToken |
| TTL | 5 min | 60 s | 1 h |
| Use | single-use | multi-use within TTL | multi-use within TTL |
| API | `POST /platform/ingest` | `GET /platform/data` | `POST /platform/ingest` |
| VC claim | `allowed_purposes` | `allowed_actions=[read]` | `allowed_actions=[write_continuous]` |

## Limits of this hands-on (future work)

- **No publisher-embedded holder yet**: ideally the publisher acquires
  a ServiceVC at startup, presents it once, holds the resulting token
  internally, and attaches it to every ingest. The hands-on uses a
  phone wallet + curl as a stand-in
- **No ServiceToken auto-renewal**: a background loop that re-presents
  5 minutes before expiry is not implemented
- **Issuance governance**: today the publisher self-issues ServiceVCs;
  who *should* issue them is a future policy decision

Once these land, Stage 4 LLM Planner can prove its dataset access for
each plan step via a VC presentation.

## Related

- [SSI Wallet sample (Stage 1)](ha-ssi-wallet.md): human single-use write
- [SSI Viewer sample (Stage 3)](ha-ssi-viewer.md): human multi-use read
- [Phase 2 hands-on appendix](webcam-event-sharing.md#補論-ウォレットモードによる認可):
  one-shot wallet mode (no ServiceVC required)
