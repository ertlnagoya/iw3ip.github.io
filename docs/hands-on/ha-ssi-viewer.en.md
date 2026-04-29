# SSI Viewer Sample (Phase 2 / Stage 3)

!!! note "Sequel to the [SSI Wallet hands-on](ha-ssi-wallet.md)"
    Assumes you've completed Stage 1 (PolicyToken-gated write).
    This page covers Stage 3: gating **read** with VC presentation.

## Goal

Where ConsentVC authorized **writes** (ingest), **ViewerVC** authorizes
**reads** (data retrieval). You'll present a ViewerVC with the wallet,
receive a short-lived ViewerToken, and call `/platform/data` to fetch
sensor rows.

Pipeline:

`Mobile wallet (ViewerVC) -> OID4VP Verifier -> ViewerToken -> /platform/data`

## What you'll learn

- Why "write VC" and "read VC" are separated
- Issuing ViewerVC via OID4VCI and presenting via OID4VP
- ViewerToken's 60-second TTL with multi-use semantics
- How `/platform/data` reads are recorded in the audit log

## Common pitfalls

- ConsentVC and ViewerVC are **distinct**. Presenting a ConsentVC won't unlock `/platform/data`
- `/platform/data` only accepts ViewerToken (PolicyToken is rejected)
- ViewerToken TTL is 60 s. After expiry, present again

## Spec links

- OpenID for Verifiable Presentations: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- DCQL: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-digital-credentials-query-l>
- SD-JWT VC: <https://www.ietf.org/archive/id/draft-ietf-oauth-sd-jwt-vc-latest.html>

## Prerequisites

- Have completed §1〜§8 of the [SSI Wallet hands-on](ha-ssi-wallet.md)
- Publisher running with `feat/ssi-viewer-vc` or later
- `iw3ip-wallet` available on phone

## Related repositories

- [Blockchain_IoT_Marketplace](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace) — publisher
- [iw3ip-wallet](https://github.com/ertlnagoya/iw3ip-wallet) — mobile SSI wallet

## Shortest path

1. Start publisher
2. Write some data via ConsentVC (Stage 1 recap)
3. Issue a ViewerVC and store it in the wallet
4. Present ViewerVC and obtain a ViewerToken
5. Read via `/platform/data` and inspect audit log

## 1. Start

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet up --build -d
curl http://192.168.68.53:8080/health
```

Confirm the issuer metadata advertises **both** ConsentVC and ViewerVC:

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer | python3 -m json.tool | grep -A2 ViewerVC
```

## 2. Write data via ConsentVC (recap)

Follow [SSI Wallet hands-on §5〜§8](ha-ssi-wallet.md#5-verifier-qrで提示)
to present ConsentVC, get a PolicyToken, and POST to `/platform/ingest`.
Those rows are what §5 will read back.

## 3. Issue a ViewerVC

Open in the PC browser:

```txt
http://192.168.68.53:8080/issuer/offer?type=ViewerVC&dataset_id=home/env/temperature
```

Same flow as ConsentVC, but with **`type=ViewerVC`**. Send the QR/deeplink
to the wallet via AirDrop and store the credential.

The wallet should display a separate card with claims:

- `dataset_id`: `home/env/temperature`
- `allowed_actions`: `["read"]`

## 4. Present the ViewerVC

```txt
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=ViewerVC
```

`vc_kind=ViewerVC` selects `viewer-temperature.json` instead of the
ConsentVC-side PD. Send the QR to the wallet, present the credential.

Expected response:

```json
{
  "status": "allowed",
  "dataset_id": "home/env/temperature",
  "vc_kind": "ViewerVC",
  "viewer_token": "Xa9Hk...",
  "viewer_token_jti": "1f2c...",
  "expires_in": 60
}
```

## 5. Read via /platform/data

```bash
TOKEN=<viewer_token>

curl -s -H "Authorization: Bearer $TOKEN" \
  'http://192.168.68.53:8080/platform/data?dataset_id=home/env/temperature' | python3 -m json.tool
```

Expected response:

```json
{
  "dataset_id": "home/env/temperature",
  "count": 3,
  "read_count": 1,
  "rows": [
    {"dataset_id": "home/env/temperature", "value": 21.4, "purpose": "research"},
    {"dataset_id": "home/env/temperature", "value": 22.0, "purpose": "research"}
  ]
}
```

`read_count` shows how many times this token has been used. Within the
60-second TTL the same token can be reused; `read_count` increments each call.

## 6. Error cases

| Situation | HTTP | `detail` |
| --- | --- | --- |
| No header | 401 | `missing_authorization_header` |
| Unknown token | 401 | `viewer_token_unknown` |
| Expired (60 s elapsed) | 401 | `viewer_token_expired` |
| Query `dataset_id` mismatch | 403 | `viewer_token_dataset_mismatch` |
| Tried using a write-side PolicyToken | 401 | `viewer_token_unknown` (separate token space) |

Verify expiry:

```bash
sleep 65
curl -i -H "Authorization: Bearer $TOKEN" \
  'http://192.168.68.53:8080/platform/data?dataset_id=home/env/temperature'
# → 401 viewer_token_expired
```

## 7. Audit log

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool
```

ViewerVC reads appear as:

```json
{
  "action": "allow",
  "raw_topic": "platform/data",
  "reason": "viewer_token_used:1f2c...:1",
  "dataset_id": "home/env/temperature",
  "purpose": "read",
  "holder_did": "did:jwk:...",
  "presentation_verified": "allow"
}
```

The trailing number in `reason` is `read_count`. Three reads with the same
token produces three rows.

## Symmetry with ConsentVC

| Property | ConsentVC (Stage 1) | ViewerVC (Stage 3) |
| --- | --- | --- |
| Purpose | Write authz | Read authz |
| Endpoint | `/verifier/request?vc_kind=ConsentVC` | `/verifier/request?vc_kind=ViewerVC` |
| Token | PolicyToken | ViewerToken |
| TTL | 5 min | 60 s |
| Use | single-use | multi-use within TTL |
| API | `POST /platform/ingest` | `GET /platform/data` |
| Audit `raw_topic` | `platform/ingest` | `platform/data` |
| VC claim | `allowed_purposes` | `allowed_actions` |

## Next steps

ViewerVC reads use a **freely-chosen dataset_id**. Other authorization
shapes live in:

- [Stage 4 prep: SSI Service](ha-ssi-service.md) — M2M continuous write (ServiceVC)
- [Stage 5: Marketplace bridge](marketplace-vc-bridge.md) — purchase-bound read (PurchaseViewerVC)
- [Stage 6: 4-VC end-to-end](marketplace-vc-end-to-end.md) — capstone
- [Stage 7: SellerVC](marketplace-seller-vc.md) — marketplace seller identity

## Extension ideas

- **VC expiry**: shorten the credential's `exp` and combine with revocation
- **Multi-dataset ViewerVC**: extend the schema with `dataset_ids: [...]`
- **Phase 3 integration**: have the [LLM Planner](llm-planner.md) acquire ViewerTokens automatically before each tool call

## Troubleshooting

- Symptom: `no_presentation_definition_for_dataset`
  - Check: `examples/ssi_wallet/viewer-<dataset>.json` exists
  - Check: PD includes `iw3ip_vc_kind: "ViewerVC"`
- Symptom: wallet shows "no matching credential"
  - Check: stored a ViewerVC, not a ConsentVC
  - Check: passed `vc_kind=ViewerVC` to `/verifier/request`
- Symptom: `/platform/data` returns 401 viewer_token_unknown
  - Check: not accidentally using a write-side PolicyToken
  - Check: TTL (60 s) has not expired
