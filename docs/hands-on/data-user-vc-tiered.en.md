# Tier the response by trust (DataUserVC / Stage T)

Switch the response between Tier 3 / 2 / 1 by the receiver's trust (DataUserVC). Delivering the actual image / video bytes (§8–§11) continues in [Deliver images and video with tiered access](data-user-vc-tiered-media.md), and semantic intermediate representation and trust-aware rendering (§12–§13) in [Tier by semantic level](data-user-vc-tiered-semantic.md) — long but thorough across the three pages.

> **What you'll do**: Watch the response narrow from video → image → summary as the trust level drops
>
> **Prerequisites**: [HA SSI Wallet](ha-ssi-wallet.en.md) and [Webcam event sharing](webcam-event-sharing.en.md)
>
> **What you need**: PC + smartphone (iw3ip-wallet)
>
> **Time**: ~60 min (the two follow-up pages take about 90 min and 60–90 min more)

This walk-through builds on the
[Mobile SSI Wallet sample](ha-ssi-wallet.md) and the
[USB Webcam Event Sharing sample](webcam-event-sharing.md) and exercises
**narrowing the camera view to three tiers based on the recipient's
trust attributes**.

Read [DataUserVC × Tiered Access Spec](data-user-vc-tiered-spec.md) first
if you want the design rationale.

Pipeline:

`Wallet -> DataUserVC presentation -> trustScore -> /marketplace/claim -> PurchaseViewerVC -> /platform/data shows/hides image|video`

## Quickest path

1. Bring up publisher / hardhat / bridge
2. Issue **DataUserVC** with three different profiles (gov-full, enterprise-access, low-deny)
3. Hit `/marketplace/claim` for each and compare `allowed_views`
4. Present PurchaseViewerVC and inspect `/platform/data` differences

## What you'll learn

- The OID4VCI / OID4VP flow for DataUserVC
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

- You have run the [Mobile SSI Wallet sample](ha-ssi-wallet.md) once
- You understand the [USB Webcam Event Sharing sample](webcam-event-sharing.md)
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
docker compose -f infra/docker-compose.yml up -d publisher bridge mosquitto
```

Sanity check:

```bash
curl -s localhost:8080/health | jq .
curl -s localhost:8080/.well-known/openid-credential-issuer \
  | jq '.credential_configurations_supported | keys'
# -> ["ConsentVC", "DataUserVC", "PurchaseViewerVC", "SellerVC", "ServiceVC", "ViewerVC"]
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

Open the `offer_url` from the §3 response in a PC browser (replace its host part with `<HOST_IP>`), scan the QR code with the wallet, and receive the PurchaseViewerVC of each claim. The wallet shows a separate card per tier. Then present it in the same way as Step 8 of [Stage 5](marketplace-vc-bridge.md).
Once the resulting ViewerToken is in hand, hit `/platform/data`:

| Profile | `event` | `image_cid` | `video_cid` |
|---|---|---|---|
| 3a Tier 3 (gov full) | yes | yes | yes |
| 3b Tier 2 (enterprise) | yes | yes | **no** |
| 3c Tier 1 (default) | yes | **no** | **no** |

Put the ViewerToken you obtained into the shell variable `VIEWER_TOKEN`,
then run:

```bash
curl -s -H "authorization: Bearer $VIEWER_TOKEN" \
     'localhost:8080/platform/data?dataset_id=home/event/possible_littering' | jq .
```

Confirm the `image_cid` / `video_cid` keys are **missing** (the keys
are dropped, not nulled).

## 5. Audit log

```bash
curl -s localhost:8080/audit/logs | jq '.[-5:]'
```

You should see a `vc_kind: "DataUserVC"` verify line, followed by the
`claim`, the token mint, and the `/platform/data` fetch — all stitched
together by `holder_did` / `claim_id`.

## 6. Cross-check with tests

```bash
cd ~/program/Blockchain_IoT_Marketplace
uv run pytest tests/test_data_user_vc_tiered.py -v
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
    - Always use the `deeplink` returned from `/marketplace/claim`
      directly. Driving the wallet from `/issuer/offer?claim_id=...` is
      now also OK after the fix, but the deeplink path is the simplest.

## Where to go next

- [Deliver images and video with tiered access](data-user-vc-tiered-media.md) — continues with §8–§11 (HTTP media gateway, IPFS, PWA viewer, PWA Provider)
- [Tier by semantic level](data-user-vc-tiered-semantic.md) — continues with §12–§13 (semantic-level tiering with a VLM, semantic intermediate representation)
- [Mobile SSI Wallet sample](ha-ssi-wallet.md) — bring up the Phase 2 wallet
- [DataUserVC × Tiered Access Spec](data-user-vc-tiered-spec.md) — design rationale
- [USB Webcam Event Sharing sample](webcam-event-sharing.md) — listing comes from here
