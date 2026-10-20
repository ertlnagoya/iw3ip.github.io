# Tier the response by trust (DataUserVC / Stage T)

Switch the response between Tier 3 / 2 / 1 by the receiver's trust (DataUserVC). Delivering the actual image / video bytes (§8–§11) continues in [Deliver images and video with tiered access](data-user-vc-tiered-media.md), and semantic intermediate representation and trust-aware rendering (§12–§13) in [Tier by semantic level](data-user-vc-tiered-semantic.md) — long but thorough across the three pages.

> **What you'll do**: Watch the response narrow from video → image → summary as the trust level drops
>
> **Prerequisites**: [Authenticate the data provider (Consent VC / Stage 1)](ha-ssi-wallet.en.md) and [Share detections, not raw frames](webcam-event-sharing.en.md)
>
> **What you need**: PC + smartphone (iw3ip-wallet)
>
> **Time required**: approx. 60 min (the two follow-up pages take about 90 min and 60–90 min more)

This walk-through builds on
[Authenticate the data provider (Consent VC / Stage 1)](ha-ssi-wallet.md) and
[Share detections, not raw frames](webcam-event-sharing.md) and exercises
**narrowing the camera view to three tiers based on the recipient's
trust attributes**.

Read [Tiered access design spec](data-user-vc-tiered-spec.md) first
if you want the design rationale.

Pipeline:

`DataUserVC issuance -> /marketplace/claim (data_user_attrs) -> trustScore -> PurchaseViewerVC -> /platform/data shows/hides image|video`

The DataUserVC is issued to the wallet so you can inspect it. The course repository has no presentation definition for presenting a DataUserVC from the wallet yet, so this page passes the same attributes directly in the `data_user_attrs` of `/marketplace/claim` to decide the trust level (§3).

## Quickest path

1. Bring up the publisher
2. Issue **DataUserVC** with three different profiles (gov-full, enterprise-access, low-deny)
3. Hit `/marketplace/claim` for each and compare `allowed_views`
4. Publish one event with image / video, present PurchaseViewerVC, and inspect `/platform/data` differences

## What you'll learn

- The OID4VCI flow that issues a DataUserVC, and the OID4VP flow that presents a PurchaseViewerVC
- How combinations of `entityType / purpose / legalCompliance /
  dataHandlingPolicy / misuseRecord` drive `full / access / denied`
- When `image_cid` / `video_cid` appear or disappear from
  `/platform/data`

## Common pitfalls

- Omitting `data_user_attrs` defaults to **`event` only** — image/video
  will not appear
- Even at `score >= 80`, leaving `entityType` as `Enterprise` does **not**
  reach `full`
- Changing `data_user_attrs` after a ViewerToken has been minted **does
  not** retroactively widen the existing token

## Prerequisites

- You have run [Authenticate the data provider (Consent VC / Stage 1)](ha-ssi-wallet.md) once
- You understand [Share detections, not raw frames](webcam-event-sharing.md)
- Docker / Docker Compose
- `curl`, `jq`

The command examples on this page assume that the course repository is cloned to `~/program/Blockchain_IoT_Marketplace`.

- `$HOST_IP` in commands and `<HOST_IP>` in URLs stand for the PC's LAN IP. In each terminal you use, first run `export HOST_IP=<your PC's LAN IP>` (see the [Hands-on overview](index.md#host-ip) for how to find it). Replace `<HOST_IP>` in URLs you type into a browser or phone with the same IP

## 0b. Choosing how to carry the actual image / video bytes

This walkthrough has three options for the data body itself. **Option B
(publisher-hosted HTTP media gateway, recommended)** is the
real-device-validated default; the iPhone walkthrough below covers it.

| Option | Source of `image` / `video` | Receiver can fetch the blob? | Effort | Use case |
|---|---|---|---|---|
| A | placeholder CID strings only | no | none | tier-projection demo only |
| **B** (recommended) | publisher serves `/media/<sha256>.<ext>` | yes — direct HTTP | shipped | demo / hands-on |
| C | local kubo IPFS daemon, content-addressed | yes — publisher's `/ipfs/<cid>` proxy + any public gateway | shipped (`--profile ipfs`) | production-flavoured distributed demo |

§2–§7 below cover the core DataUserVC + tier-projection loop. **Option
B real-data integration is in [§8](data-user-vc-tiered-media.md#8-real-data-integration-option-b--http-media-gateway)**, **Option C IPFS integration is in
[§9](data-user-vc-tiered-media.md#9-option-c-local-kubo-ipfs-daemon-for-distributed-delivery).** Option C is a strict superset of B — `/media/upload`'s response
just gains a `cid` field, so the provider script needs no changes.

## 1. Bring up services

```bash
cd ~/program/Blockchain_IoT_Marketplace
git pull --ff-only
docker compose -f infra/docker-compose.yml up --build -d publisher mosquitto
```

This page calls `/marketplace/claim` directly with curl, so the bridge and Hardhat are not started.

Sanity check:

```bash
curl -s localhost:8080/health | jq .
curl -s localhost:8080/.well-known/openid-credential-issuer \
  | jq '.credential_configurations_supported | keys'
# -> ["ConsentVC", "DataUserVC", "PurchaseViewerVC",
#     "PurchaseViewerVC.access", "PurchaseViewerVC.event", "PurchaseViewerVC.full",
#     "SellerVC", "ServiceVC", "ViewerVC"]
```

`DataUserVC` must appear in the list.

## 2. Mint three DataUserVC offers

Open the DataUserVC issuance page in a PC browser, scan the QR code with the phone wallet, and store the DataUserVC. Do this for each of the three profiles.

### 2a. Tier 3 (full) — government + crime search + ISO27001

```
http://<HOST_IP>:8080/issuer/offer?type=DataUserVC&entity_type=GovernmentOrganization&purpose=CrimeSearch&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false
```

### 2b. Tier 2 (access) — enterprise + research + ISO27001

```
http://<HOST_IP>:8080/issuer/offer?type=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false
```

### 2c. Tier 1 (denied) — enterprise + research + no policy + misuse

```
http://<HOST_IP>:8080/issuer/offer?type=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true
```

This profile scores 25 and the verdict is `denied`. `denied` is the name of the trustScore verdict. In the current implementation a `denied` verdict does not reject the purchase report (`/marketplace/claim`). A PurchaseViewerVC that can view events only (`PurchaseViewerVC.event`) is issued, and images, video, and descriptions are left out of the response.

## 3. Three `/marketplace/claim` calls

`/marketplace/claim` is the API that tells the publisher a purchase has happened. Normally the bridge calls it when it detects a purchase event; here we call it directly with curl to see only the tier differences. When `data_user_attrs` carries the same attributes as a DataUserVC, the publisher computes the trustScore and decides what that purchase may view.

`merchandise_address`, `buyer_eth_addr`, `tx_hash`, and `dataset_id` are required. No real purchase is involved here, so use any string for `tx_hash`, different for each call (calling again with the same `tx_hash` returns the same claim).

### 3a. Tier 3 — opens up to video

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{
    "merchandise_address": "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
    "buyer_eth_addr": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "tx_hash": "0xdemo-tier3",
    "dataset_id": "home/event/possible_littering",
    "data_user_attrs": {
      "entityType": "GovernmentOrganization",
      "purpose": "CrimeSearch",
      "legalCompliance": true,
      "dataHandlingPolicy": "ISO27001",
      "misuseRecord": false
    }
  }' | tee /tmp/claim.json | jq '{claim_id, offer_url}'
```

The response contains `claim_id`, `offer_url` (the PurchaseViewerVC issuance page), and `deeplink`. The tier that was decided is carried in the deeplink as the kind of PurchaseViewerVC to be issued. Extract it with:

```bash
jq -r .deeplink /tmp/claim.json \
  | python3 -c "import sys,json,urllib.parse; print(json.loads(urllib.parse.unquote(sys.stdin.read().split('credential_offer=',1)[1]))['credential_configuration_ids'])"
# -> ['PurchaseViewerVC.full']
```

| Kind | What can be viewed (`allowed_views`) | trustScore |
|---|---|---|
| `PurchaseViewerVC.full` | `event` / `image` / `video` | 80 or more, and a government organization etc. |
| `PurchaseViewerVC.access` | `event` / `image` | 60 or more |
| `PurchaseViewerVC.event` | `event` only | otherwise (verdict `denied`), or no `data_user_attrs` |

### 3b. Tier 2 — image only

Run the 3a command with `entityType` set to `Enterprise`, `purpose` set to `Research`, and `tx_hash` set to `0xdemo-tier2`. The trustScore is 75 and the kind in the deeplink is `['PurchaseViewerVC.access']`.

### 3c. Tier 1 — defaults when `data_user_attrs` is omitted

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{
    "merchandise_address": "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
    "buyer_eth_addr": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "tx_hash": "0xdemo-tier1",
    "dataset_id": "home/event/possible_littering"
  }' | tee /tmp/claim.json | jq '{claim_id, offer_url}'
```

The kind in the deeplink is `['PurchaseViewerVC.event']`.

## 4. Issue PurchaseViewerVC, present, fetch `/platform/data`

### 4a. Publish one event with image / video

`/platform/data` returns the event rows the publisher has received. If nothing has been published yet, `rows` is empty for every tier and you see no difference. First register a consent (ConsentVC), then publish one event that carries `image_cid` / `video_cid`. As in option A, the CIDs here are placeholder strings.

```bash
cd ~/program/Blockchain_IoT_Marketplace
curl -s -X POST localhost:8080/consents \
  -H 'content-type: application/json' \
  -d @examples/consent_possible_littering.json | jq .

curl -s -X POST localhost:8080/simulate/publish \
  -H 'content-type: application/json' \
  -d '{
    "topic": "homeassistant/event/possible_littering",
    "purpose": "community_cleaning",
    "payload": {
      "event_type": "possible_littering",
      "data": {"camera_id": "webcam-401", "object_class": "bottle", "confidence": 0.87},
      "image_cid": "QmDemoImage",
      "video_cid": "QmDemoVideo",
      "video_duration_sec": 12,
      "ts": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'",
      "source": "edge_inference"
    }
  }' | jq .
```

The second response should show `"status": "allowed"` and `"dataset_id": "home/event/possible_littering"`. If it says `denied`, check the consent registration.

Event rows are kept only in the publisher's memory. Restarting the publisher clears them, so redo this step after a restart. Uploading real image / video files and publishing them (`provider_with_media.py`) is covered in [§8](data-user-vc-tiered-media.md#8-real-data-integration-option-b--http-media-gateway).

### 4b. Receive and present PurchaseViewerVC

Open the `offer_url` from the §3 response in a PC browser (replace its host part with `<HOST_IP>`), scan the QR code with the wallet, and receive the PurchaseViewerVC of each claim. The wallet shows a separate card per tier (with an old publisher all three cards have the same name, so `git pull` and start with `--build` as in §1). Then open the following URL in a PC browser, scan the QR code with the wallet, and present the card of the tier you want to check. This is the same operation as Step 8 of [Hand a credential to the buyer's wallet (Bridge / Stage 5)](marketplace-vc-bridge.md), but the dataset is `home/event/possible_littering`.

```
http://<HOST_IP>:8080/verifier/request?dataset_id=home/event/possible_littering&vc_kind=PurchaseViewerVC
```

Once the resulting ViewerToken is in hand, hit `/platform/data`:

| Profile | `event` | `image_cid` | `video_cid` |
|---|---|---|---|
| 3a Tier 3 (gov full) | yes | yes | yes |
| 3b Tier 2 (enterprise) | yes | yes | **no** |
| 3c Tier 1 (default) | yes | **no** | **no** |

The ViewerToken appears in the publisher log. Right after presenting, extract it with:

```bash
docker compose -f infra/docker-compose.yml logs publisher 2>&1 | grep viewer_token_issued | tail -n 1
# -> ... viewer_token_issued vc_kind=PurchaseViewerVC jti=... token=<ViewerToken> dataset=home/event/possible_littering ttl=60s views=event+image+video
```

Put the `token=` value into the shell variable `VIEWER_TOKEN` and run the
command below. A ViewerToken is valid for only 60 seconds, so use it right away. If it expires, present again.

```bash
export VIEWER_TOKEN=<value of token=>
curl -s -H "authorization: Bearer $VIEWER_TOKEN" \
     'localhost:8080/platform/data?dataset_id=home/event/possible_littering' | jq .
```

For Tier 2 / Tier 1, confirm the `image_cid` / `video_cid` keys are **missing** (the keys
are dropped, not nulled).

## 5. Audit log

```bash
curl -s 'localhost:8080/audit/logs?limit=100' \
  | jq '[.[] | select(.raw_topic | IN("marketplace/claim", "marketplace/issued", "oid4vp/response", "platform/data"))]
        | reverse | .[] | {ts, raw_topic, action, reason}'
```

`/audit/logs` returns the newest rows first. This command keeps only the four kinds of rows related to this page and prints them oldest first.

For each tier, confirm the rows appear from top to bottom as `marketplace/claim` (claim accepted), `marketplace/issued` (PurchaseViewerVC issued), `oid4vp/response` (presentation verified), and `platform/data` (data read).

## 6. Cross-check with tests

Python 3.11 or later is required. If `python3 --version` is older than 3.11, run the `python3 -m venv` line below with a specific version, such as `python3.11 -m venv`.

```bash
cd ~/program/Blockchain_IoT_Marketplace
python3 -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn pydantic pydantic-settings httpx paho-mqtt \
  cryptography python-multipart "qrcode[pil]" pytest
python -m pytest tests/test_data_user_vc_tiered.py -v
```

The four pure-function tests
(GovernmentOrganization / Enterprise / low-score / `score>=80` but
non-gov/police) lock the parity between `trust_score.py` and
`DataUserVerifier.sol`.

## 7. Values observed on real-device runs

Walking through end-to-end on iPhone (iw3ip-wallet) produces these
values per tier in both the ViewerToken mint log
(`viewer_token_issued ... views=...`) and the `/platform/data` response.

| Tier | DataUserVC profile | trust_score | views | image_cid | video_cid | video_duration_sec |
|---|---|---|---|---|---|---|
| **3** gov | gov + crime + ISO27001 | 80 | `event+image+video` | yes | yes | yes |
| **2** ent | enterprise + research + ISO27001 | 75 | `event+image` | yes | **no** | **no** |
| **1** low | `data_user_attrs` omitted | n/a (default) | `event` | **no** | **no** | **no** |

The wallet shows three distinct PurchaseViewerVC cards (one per tier)
because the issuer publishes
`PurchaseViewerVC.full / .access / .event` as separate
`credential_configuration_ids` with distinct `display.name`s. All three
share the same VCT.

!!! tip "Pitfalls observed during real-device validation"
    A few first-run symptoms have been folded back into the codebase
    via PRs `fix/stage-t-purchase-viewer-binding` and
    `feat/stage-t-tier-display-and-projection`. On a current `main`
    you should not hit them, but if you do:

    - "No Available Credential" → the issuer must include `subject_id`
      in PurchaseViewerVC plain claims (now done).
    - Three identical cards → tier-aware
      `credential_configuration_id` per Tier (now done).
    - `image_cid` invisible at `/platform/data` even when supplied via
      `/simulate/publish` → the pipeline now hoists the media CIDs to
      the envelope's top level.
    - Since the fix, opening `/issuer/offer?claim_id=...` (`offer_url`) as in §4b
      gives you a card of the same tier as the claim. The `deeplink` in the `/marketplace/claim`
      response uses, with the default setting (`SSI_ISSUER_BASE_URL` set to the publisher's internal name), the host the claim was sent to. A claim sent to `localhost`, as on this page,
      yields a `deeplink` the phone cannot reach, so use `offer_url`. When the bridge is started with `BRIDGE_PUBLIC_PUBLISHER_URL=http://<HOST_IP>:8080`, it passes that URL to the publisher, so the `deeplink` works too.

## Where to go next

- [Deliver images and video with tiered access](data-user-vc-tiered-media.md) — continues with §8–§11 (HTTP media gateway, IPFS, PWA viewer, PWA Provider)
- [Tier by semantic level](data-user-vc-tiered-semantic.md) — continues with §12–§13 (semantic-level tiering with a VLM, semantic intermediate representation)
- [Authenticate the data provider (Consent VC / Stage 1)](ha-ssi-wallet.md) — bring up the Phase 2 wallet
- [Tiered access design spec](data-user-vc-tiered-spec.md) — design rationale
- [Share detections, not raw frames](webcam-event-sharing.md) — listing comes from here
