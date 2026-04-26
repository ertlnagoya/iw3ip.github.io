# Mobile SSI Wallet Sample (Phase 2)

!!! note "This hands-on is under construction"
    This page drafts a VC verification demo based on a fork of
    Sphereon mobile-wallet at `ertlnagoya/iw3ip-wallet`.
    The companion wallet app and backend endpoints are in preparation.

## Purpose

Experience the flow where a smartphone SSI wallet presents a Consent VC,
the IW3IP backend verifies it via OID4VP, and only verified requests
receive shared event data.

The existing [HA x SSI Publisher Sample](ha-ssi-publisher.md) registers
a Consent-VC-shaped policy JSON directly via `/consents`. This sample
moves to the actual SSI model: **the wallet presents a VC -> verifier
validates it -> allowed/denied**.

Pipeline:

`Mobile wallet (holds VC) -> QR/Deeplink -> OID4VP Verifier -> Platform API -> Event sharing`

## What you learn

- The basic flow of issuing a VC to the wallet (OID4VCI) and presenting it (OID4VP)
- How the verifier's Presentation Definition (PEX) maps to the wallet's response
- A lightweight stack using `did:jwk` / `did:key` and SD-JWT VC
- How `allowed` / `denied` depends on VC content and `purpose`

## Common pitfalls

- If the Presentation Definition `input_descriptors` disagree with the VC
  claims, the wallet reports "no matching credential"
- DID method mismatches between issuer / wallet / verifier cause signature failures
- SD-JWT VC, W3C VC-JWT, and mdoc are not interchangeable; pin one format
- QR redirect fails if the phone and PC are on different networks
  (same prerequisite as [Mobile Viewer](mobile-viewer.md))

## Official links

- Sphereon mobile-wallet (upstream): <https://github.com/Sphereon-Opensource/mobile-wallet>
- `@sphereon/pex` (Presentation Exchange): <https://github.com/Sphereon-Opensource/pex>
- OpenID for Verifiable Presentations: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- OpenID for Verifiable Credential Issuance: <https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html>
- SD-JWT VC: <https://www.ietf.org/archive/id/draft-ietf-oauth-sd-jwt-vc-latest.html>

## Prerequisites

- Docker / Docker Compose available
- PC and phone on the same LAN
- Installable `iw3ip-wallet` (fork) build on the phone
  (TestFlight / internal APK / Expo dev build)
- You have completed the [HA x SSI Publisher Sample](ha-ssi-publisher.md) once

## Related repositories

- `iw3ip-wallet` (planned fork): `https://github.com/ertlnagoya/iw3ip-wallet` (in preparation)
  - upstream: Sphereon-Opensource/mobile-wallet
  - Branch policy: `main` tracks upstream; `iw3ip/*` carries IW3IP-specific changes
- `iw3ip-verifier` (OID4VP endpoint added inside Publisher, in preparation)

## Shortest path

The intended flow is 5 steps:

1. Start the publisher with the issuer / verifier profile enabled
2. Open `iw3ip-wallet` on the phone; scan the issuer QR to receive a Consent VC
3. Scan the verifier QR shown on the PC screen
4. The wallet selects the matching VC and presents it
5. Confirm `allow` in the publisher's `/audit/logs`

Next branches:

- To see a denial: change `purpose` to one not covered by the VC and re-present
- To see revocation: revoke the VC; the same presentation should become `denied`
- To compare with Phase 1: observe that audit logs now capture presentation
  events rather than direct `/consents` registration

## Stage-by-stage table of contents

<details class="iw3ip-toc-details" open>
  <summary>Setup: start publisher / issuer / verifier</summary>
  <p>Start the publisher with OID4VCI / OID4VP endpoints enabled.</p>
  <ol>
    <li><a href="#1-start">Start</a></li>
    <li><a href="#2-inspect-presentation-definition">Inspect Presentation Definition</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 1: issue a VC to the wallet</summary>
  <p>Issue a Consent VC from the issuer QR and confirm it is stored in the wallet.</p>
  <ol>
    <li><a href="#3-open-the-wallet">Open the wallet</a></li>
    <li><a href="#4-issue-via-issuer-qr">Issue via issuer QR</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 2: present the VC and compare allowed vs denied</summary>
  <p>Scan the verifier QR and compare behavior when `purpose` matches vs not.</p>
  <ol>
    <li><a href="#5-present-via-verifier-qr">Present via verifier QR</a></li>
    <li><a href="#6-denial-case">Denial case</a></li>
    <li><a href="#7-audit-log-check">Audit log check</a></li>
  </ol>
</details>

## How to read this page

This is the Phase 2 stage that treats the Consent VC as a wallet-presented
artifact rather than a pre-registered policy. Run the
[HA x SSI Publisher Sample](ha-ssi-publisher.md) and
[Environment Disaster Sample](environment-disaster.md) first, then come back
to this page to see the delta.

## 1. Start

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet up --build -d
```

Check:

```bash
curl http://localhost:8080/health
curl http://localhost:8080/.well-known/openid-credential-issuer
```

## 2. Inspect Presentation Definition

```bash
curl http://localhost:8080/verifier/presentation-definitions/consent-temperature
```

Points to verify:

- `input_descriptors` include constraints on `dataset_id` and `purpose`
- Format is `vc+sd-jwt`

## 3. Open the wallet

Open `iw3ip-wallet` on the phone and complete first-run setup.
An initial key is generated as `did:jwk`.

## 4. Issue via issuer QR

Open in the PC browser:

```txt
http://<PC_LAN_IP>:8080/issuer/offer?type=ConsentVC&dataset_id=home/env/temperature&purpose=research
```

Scan the displayed QR with the wallet, approve, and store the VC.

## 5. Present via verifier QR

Open in the PC browser:

```txt
http://<PC_LAN_IP>:8080/verifier/request?dataset_id=home/env/temperature&purpose=research
```

Scan the QR with the wallet and select the matching VC to present.

Expected result:

```json
{
  "status": "allowed",
  "dataset_id": "home/env/temperature",
  "policy_token": "VHA9X1d...",
  "policy_token_jti": "9b2fc3...",
  "expires_in": 300
}
```

`policy_token` is the short-lived authorization token used in §8.
It is valid for 5 minutes and single-use: one call to `/platform/ingest`
consumes it.

## 6. Denial case

Change `purpose` to `marketing` and present again.

Expected result:

```json
{"status":"denied","dataset_id":"home/env/temperature","reason":"purpose_mismatch"}
```

Holding the VC is not enough; the verifier's constraints must also match.

## 7. Audit log check

```bash
curl http://localhost:8080/audit/logs?limit=10
```

Points to verify:

- `presentation_verified` events are recorded with `allow` / `deny`
- `holder_did`, `vc_hash`, `purpose` are preserved
- Compared with HA x SSI Publisher, the log additionally records
  "who presented which VC" rather than just the policy outcome

## 8. Pull shared data

Pass the `policy_token` from §5 as a `Authorization: Bearer` header to
`/platform/ingest` to actually share data backed by the verified VC.
The legacy `/consents` JSON registration path still works without a header.

PolicyToken properties:

- TTL: 5 minutes (returned as `expires_in`)
- Single-use: one successful `/platform/ingest` call invalidates it
- Scope: only accepts a body whose `dataset_id` matches the issuance time
- Format: opaque string (server-side in-memory)

Request:

```bash
TOKEN=<policy_token>  # from the verifier response in §5

curl -X POST http://<PC_LAN_IP>:8080/platform/ingest \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":"home/env/temperature","purpose":"research","value":21.4}'
```

Expected result:

```json
{"status":"received","count":1}
```

The audit log gains a PolicyToken consumption entry:

```bash
curl 'http://<PC_LAN_IP>:8080/audit/logs?limit=5'
```

```json
{
  "action": "allow",
  "raw_topic": "platform/ingest",
  "reason": "policy_token_consumed:9b2fc3...",
  "dataset_id": "home/env/temperature",
  "purpose": "research",
  "holder_did": "did:jwk:...",
  "presentation_verified": "allow"
}
```

Error cases:

| Situation | HTTP | `detail` |
| --- | --- | --- |
| Reusing the same token | 403 | `policy_token_already_consumed` |
| Expired | 401 | `policy_token_expired` |
| Body `dataset_id` differs from token | 403 | `policy_token_dataset_mismatch` |
| Unknown token | 401 | `policy_token_unknown` |

To grab the token, either inspect the publisher container log when the wallet
finishes the presentation (the `/verifier/response` body is logged), or read
the completion screen on the phone. In a hands-on session, scraping the
publisher log into curl is the quickest route.

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet logs -f publisher | grep policy_token
```

## Extension ideas

- **Revocation**: expose Status List 2021 at `/verifier/status` and verify that
  a revoked VC results in `denied`
- **SIOPv2 holder auth**: verify the holder's DID alongside the presentation
- **Phase 3 integration**: require VC presentation before
  [LLM Planner](llm-planner.md) `plan` execution, lifting the check into
  the intelligence layer

## Troubleshooting

- Symptom: wallet shows "no matching credential"
  - Check: Presentation Definition `vct` / claims vs the VC's contents
- Symptom: signature verification error
  - Check: DID methods align across issuer / verifier / wallet
  - Check: clock skew (`iat` / `exp`) is within tolerance
- Symptom: QR does not open
  - Check: phone and PC are on the same LAN (see [Mobile Viewer](mobile-viewer.md))
