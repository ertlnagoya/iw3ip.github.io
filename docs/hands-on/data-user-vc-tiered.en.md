# DataUserVC × Tiered Access (Phase 2 extension)

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

## 0b. Choosing how to carry the actual image / video bytes

This walkthrough has three options for the data body itself. **Option B
(publisher-hosted HTTP media gateway, recommended)** is the
real-device-validated default; the iPhone walkthrough below covers it.

| Option | Source of `image` / `video` | Receiver can fetch the blob? | Effort | Use case |
|---|---|---|---|---|
| A | placeholder CID strings only | no | none | tier-projection demo only |
| **B** | publisher serves `/media/<sha256>.<ext>` | yes — direct HTTP | shipped | demo / hands-on |
| **C** (recommended) | local kubo IPFS daemon, content-addressed | yes — publisher's `/ipfs/<cid>` proxy + any public gateway | shipped (`--profile ipfs`) | production-flavoured distributed demo |

§2–§7 below cover the core DataUserVC + tier-projection loop. **Option
B real-data integration is in §8**, **Option C IPFS integration is in
§9.** Option C is a strict superset of B — `/media/upload`'s response
just gains a `cid` field, so the provider script needs no changes.

## 1. Bring up services

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml up -d publisher hardhat bridge mosquitto
```

Sanity check:

```bash
curl -s localhost:8080/healthz | jq .
curl -s localhost:8080/.well-known/openid-credential-issuer \
  | jq '.credential_configurations_supported | keys'
# -> ["ConsentVC", "DataUserVC", "PurchaseViewerVC", "SellerVC", "ServiceVC", "ViewerVC"]
```

`DataUserVC` must appear in the list.

## 2. Mint three DataUserVC offers

### 2a. Tier 3 (full) — government + crime search + ISO27001

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=GovernmentOrganization&purpose=CrimeSearch&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

### 2b. Tier 2 (access) — enterprise + research + ISO27001

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

### 2c. Tier 1 (denied) — enterprise + research + no policy + misuse

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true' | jq .
```

Scan each `credential_offer_uri` from your iPhone wallet and store the
DataUserVC.

## 3. Three `/marketplace/claim` calls

Reuse a `merchandise_id` already listed via webcam-event-sharing (list
one beforehand).

### 3a. Tier 3 — opens up to video

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{
    "merchandise_id": "M-0001",
    "buyer_did": "did:jwk:GOV_USER_DID",
    "data_user_attrs": {
      "entityType": "GovernmentOrganization",
      "purpose": "CrimeSearch",
      "legalCompliance": true,
      "dataHandlingPolicy": "ISO27001",
      "misuseRecord": false
    }
  }' | jq '.allowed_views, .access_level, .trust_score'
# -> ["event","image","video"]
#    "full"
#    80
```

### 3b. Tier 2 — image only

```bash
# data_user_attrs.entityType: "Enterprise", purpose: "Research"
# allowed_views: ["event", "image"], access_level: "access", score: 75
```

### 3c. Tier 1 — defaults when `data_user_attrs` is omitted

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{"merchandise_id":"M-0001","buyer_did":"did:jwk:LOW_USER_DID"}' | jq .
# allowed_views: ["event"]
```

## 4. Issue PurchaseViewerVC, present, fetch `/platform/data`

Issue a PurchaseViewerVC per buyer with
`/issuer/offer?vc_kind=PurchaseViewerVC&claim_id=...`, store it in the
wallet, then present it.

Once the resulting ViewerToken is in hand, hit `/platform/data`:

| Profile | `event` | `image_cid` | `video_cid` |
|---|---|---|---|
| 3a Tier 3 (gov full) | yes | yes | yes |
| 3b Tier 2 (enterprise) | yes | yes | **no** |
| 3c Tier 1 (default) | yes | **no** | **no** |

```bash
curl -s -H "authorization: Bearer $VIEWER_TOKEN" \
     localhost:8080/platform/data?dataset_id=home/event/possible_littering | jq .
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

## 8. Real-data integration (Option B — HTTP media gateway)

Through §7 you verified the **keys** `image_cid` / `video_cid` /
`video_duration_sec` appear or disappear per tier. To carry the actual
image / video bytes end-to-end, the provider side uploads a JPEG / MP4
fixture to the publisher's `/media/upload`, takes back the URL, and
folds it into the event payload as `image_url` / `video_url`.

### 8.1 One-shot provider script

```bash
cd ~/program/Blockchain_IoT_Marketplace
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080
```

The script:

1. Generates a 1×1 JPEG / MP4 fixture under `fixtures/`.
2. Uploads both via `POST /media/upload` (sha256 dedupe).
3. Posts a `possible_littering` event carrying `image_url` /
   `video_url` / `video_duration_sec` to `/simulate/publish`.

### 8.2 Receiver (unchanged — same Tier 3 / 2 / 1 flow as §3–§7)

After re-running §6 + §7 with the new event in flight you should see:

- **Tier 3 (gov full)** → `event` + `image_url` + `video_url` + `video_duration_sec`
- **Tier 2 (enterprise)** → `event` + `image_url` only
- **Tier 1 (default)** → `event` only

Tap `image_url` in iPhone Safari → the JPEG opens directly. To try a
real photo / clip:

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080 \
  --image /path/to/snapshot.jpg \
  --video /path/to/clip.mp4 \
  --video-duration-sec 12
```

### 8.3 Limitations of Option B

- ✅ Coexists with `image_cid` / `video_cid` (Option A / coming-up Option C).
- ❌ Not content-addressed — anyone with the URL can GET the blob.
- ❌ Single replica, no pinning.

When content-addressing + distributed storage matter, move on to
**Option C (real IPFS / Web3.Storage)**. The `/media/upload` response
shape (`{url, sha256, content_type, byte_size, cid, ipfs_gateway_url}`)
stays the same so the provider script doesn't change — only the
backend swaps.

## 9. Option C: local kubo IPFS daemon for distributed delivery

Option B served the blob from a single publisher instance. Option C
hashes the same blob into a **content-addressed CID** and stores it on
an IPFS network. Receivers can dereference the CID through **any IPFS
gateway** — the publisher's built-in `/ipfs/<cid>` proxy, the public
`https://ipfs.io/ipfs/<cid>`, etc. — so the data survives a publisher
outage.

### 9.1 Bring kubo up alongside the publisher

Activate the `ipfs` compose profile:

```bash
cd ~/program/Blockchain_IoT_Marketplace
export IPFS_API_URL=http://ipfs:5001
export IPFS_GATEWAY_URL=http://ipfs:8080

docker compose -f infra/docker-compose.yml --profile ipfs up -d --force-recreate publisher ipfs
docker compose -f infra/docker-compose.yml ps ipfs
# iw3ip-ipfs container should be Up
```

Verify the publisher picked up the env vars:

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer >/dev/null
docker compose -f infra/docker-compose.yml exec publisher \
  python -c "from publisher.app.config import Settings; \
             s=Settings(); \
             print('IPFS_API_URL=', s.ipfs_api_url); \
             print('IPFS_GATEWAY_URL=', s.ipfs_gateway_url)"
```

### 9.2 Confirm CIDs come back from upload

`provider_with_media.py` is unchanged but now sees `cid` and
`ipfs_gateway_url` in the response:

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080 \
  --image /tmp/stage_t_demo.jpg \
  --video /tmp/stage_t_demo.jpg
```

Expected output (`cid` starts with `bafy...`):

```json
[upload] {
  "image": {
    "url": "http://192.168.68.53:8080/media/<sha256>.jpg",
    "sha256": "...",
    "content_type": "image/jpeg",
    "byte_size": 7645,
    "cid": "bafkreigb...",
    "ipfs_gateway_url": "http://192.168.68.53:8080/ipfs/bafkreigb..."
  },
  ...
}
```

The provider script automatically folds the CID into the event payload
as `image_cid` / `video_cid`, so the receiver §3–§7 flow keeps
working.

### 9.3 Receiver: CID or URL — pick one

`/platform/data` now hands back **both** `image_cid` and `image_url`
for Tier 2 / Tier 3:

```bash
curl -s -H "authorization: Bearer $TOK_GOV" \
  "http://192.168.68.53:8080/platform/data?dataset_id=home/event/possible_littering" \
  | jq '.rows[0] | {image_cid, image_url, ipfs_gateway: ("http://192.168.68.53:8080/ipfs/"+.image_cid)}'
```

Open any of these on the buyer's iPhone Safari:

| Method | Example URL |
|---|---|
| Publisher HTTP gateway | `http://192.168.68.53:8080/media/<sha256>.jpg` (Option B compatible) |
| Publisher IPFS proxy | `http://192.168.68.53:8080/ipfs/<cid>` |
| Public IPFS gateway | `https://ipfs.io/ipfs/<cid>` (needs internet) |

The last entry is the punchline: even with the publisher offline, the
CID is enough to fetch the asset from any cooperating peer.

### 9.4 Pros, caveats, and follow-ups

- ✅ **Content-addressed**: CID is a hash of the bytes — tamper-evident, multi-gateway.
- ✅ **Publisher outage tolerant**: any peer with a replica can serve.
- ✅ **Option-B compatible**: response gains `cid` + `ipfs_gateway_url`; existing fields are unchanged.
- ⚠️ If the kubo daemon is unreachable, `/media/upload` still returns 200 but with `cid: null` — graceful fallback to Option B.
- ⚠️ Public-gateway resolution depends on IPFS network propagation (can take minutes).
- 🔜 Pair with a pinning service (Web3.Storage / Pinata) for cross-network durability — follow-up TODO.

### 9.5 Troubleshooting

| Symptom | Fix |
|---|---|
| `cid: null` in upload response | `docker compose ... ps ipfs` — bring kubo back up if down |
| `/ipfs/<cid>` returns 502 | Publisher can't reach `http://ipfs:8080`. Check Docker network membership |
| `/ipfs/<cid>` returns 404 | `IPFS_GATEWAY_URL` is empty. Set `.env` or export and recreate |
| Public gateway can't fetch | Behind NAT, kubo not visible to peers. `ipfs swarm peers` to verify |

## 10. PWA viewer (one UX for phone *and* PC)

§3–§9 walked through the dev-style flow ("call /verifier/request,
collect the token, curl /platform/data"). The publisher now ships a
**built-in PWA viewer** (`/buyer/start` + `/viewer`) so a buyer just
opens **one URL** and the rest is automatic, on iPhone Safari **or**
PC Chrome / Edge / Firefox.

```
[iPhone Safari] ─ /buyer/start                            [publisher]
   │  ↓ auto-redirect to deeplink                              │
   │  iw3ip-wallet opens → present Tier 3 ────────────────────►│ mint ViewerToken
   │  ↑ redirect_uri = /viewer?vt=...                          │
   │  Safari resumes → /viewer auto-renders image/video        │

[PC Chrome] ─ /buyer/start                                [publisher]
   │  ↓ shows QR + long-polls                                  │
   │  Scan QR with iPhone → wallet → present ────────────────► │
   │  ↑ /verifier/status returns viewer_url                    │
   │  PC browser auto-navigates to /viewer → image renders     │
```

### 10.1 Bring it up

No special setup. Open `/buyer/start?ds=<dataset_id>` from **the same
URL** on either device — the page detects the user agent and switches
mode:

```
iPhone Safari:  http://192.168.68.53:8080/buyer/start?ds=home/event/possible_littering
PC Chrome:      http://192.168.68.53:8080/buyer/start?ds=home/event/possible_littering
```

### 10.2 Same-device (iPhone) flow

1. Open the URL in Safari.
2. The page calls `/verifier/request` to fetch a deeplink.
3. `window.location = deeplink` launches **iw3ip-wallet** automatically.
4. In the wallet, pick "Purchase Viewer (Tier 3 / Full)" and present.
5. The wallet receives `redirect_uri = /viewer?vt=...&ds=...` and...
6. ...Safari **resumes automatically and the image / video renders**.

No more URL-copy-paste.

### 10.3 Cross-device (PC + iPhone) flow

1. Open the URL on a PC browser.
2. The page renders a **large QR code** (the OID4VP deeplink).
3. Scan it with the iPhone camera or directly inside the wallet → wallet → present.
4. The PC page long-polls `/verifier/status?state=...` every 2 s.
5. Once the wallet completes, the response includes `viewer_url`.
6. The PC navigates to `/viewer` and **the same image / video renders**.

### 10.4 What `/viewer` shows

`/viewer?vt=<viewer_token>&ds=<dataset_id>` renders:

- A **tier badge** (`event` / `event+image` / `event+image+video`) up top.
- `image_url` as an inline `<img>`.
- `video_url` as a playable `<video controls>`.
- `image_cid` (if Option C is on) as a clickable link to the publisher's `/ipfs/<cid>` proxy.
- Raw row JSON tucked under a `<details>` toggle.

If the ViewerToken's 60-second TTL has expired you get a 401 plus a
human-readable nudge to "reload from /buyer/start."

### 10.5 What each tier looks like (real-device screenshots)

#### Wallet side: three distinct cards

After receiving the three deeplinks, iw3ip-wallet (Sphereon mobile-wallet fork)
shows the PurchaseViewerVCs as three independent cards — same VCT, different
`credential_configuration_id`, different display name.

<figure markdown>
![iw3ip-wallet credential list with 3 PurchaseViewerVC tiers](images/data-user-vc-tiered/wallet-tier-cards.png){ width=320 }
<figcaption>
Wallet credential list (vertical scroll). All three share VCT
<code>https://iw3ip.example/credentials/PurchaseViewerVC/v1</code>;
the <code>credential_configuration_id</code> split (<code>.full</code> /
<code>.access</code> / <code>.event</code>) is what gives them their distinct
labels.
</figcaption>
</figure>

#### Viewer side: response shape changes per presented tier

Below are real-device `/viewer` screenshots from iPhone Safari (`◀ SphereonWallet`
in the top-left = Safari was opened by the wallet via `redirect_uri`). The
badge color and which media keys disappear tell the tier story at a glance.

| Tier 3 (gov / Full) | Tier 2 (enterprise / Access) | Tier 1 (default / Event-only) |
|---|---|---|
| ![Tier 3 viewer](images/data-user-vc-tiered/viewer-tier-3-full.png) | ![Tier 2 viewer](images/data-user-vc-tiered/viewer-tier-2-access.png) | ![Tier 1 viewer](images/data-user-vc-tiered/viewer-tier-1-event.png) |
| 🟢 `tier: event+image+video` | 🟠 `tier: event+image` | ⚪️ `tier: event` |
| inline image + video player | image only, video player gone | timestamp + raw payload only |
| `image_cid` / `image_url` / `video_url` / `video_duration_sec` all present | `image_cid` / `image_url` only | every media key dropped |

The three screenshots come from the **same dataset** (`home/event/possible_littering`)
fetched with three different PurchaseViewerVC tiers. The server-side data is
identical — what changes is which keys ViewerToken's `allowed_views` lets
through. That's Stage T's whole point.

### 10.6 Why both devices matter

| | Manual curl flow (§§3–9) | PWA viewer |
|---|---|---|
| Easy on phone | ✗ — copy URLs | ✓ |
| Works on PC | ✗ — no wallet on PC | ✓ — QR + long-poll |
| App install on PC | iw3ip-wallet (phone) | none beyond a browser |
| Re-fetch after TTL | redo curl | reload the page |
| Public demo cost | long handout | hand over one URL |

### 10.7 Troubleshooting

| Symptom | Fix |
|---|---|
| iPhone deeplink doesn't open the wallet | Tap the visible "Open in wallet" link; check Safari's "Open in App" permission |
| PC doesn't show a QR | The browser couldn't reach the CDN (`cdn.jsdelivr.net`). Plug in. |
| PC poll never finishes | Confirm the wallet actually presented a valid VC: `docker compose logs publisher` should show a `200 /verifier/response` |
| `/viewer` returns 401 | ViewerToken TTL (60 s) lapsed. Reload `/buyer/start` |
| `/buyer/start` shows a "dataset mismatch" red banner | See §10.8 below |

### 10.8 Deny UX (when the verifier rejects a presentation)

When the verifier rejects a presented VC, `/verifier/status` returns a
`reason` code together with a human-readable `human_message_ja` /
`human_message_en` pair. The `/buyer/start` page picks that up during
its long-poll and replaces the QR with a red banner plus a "re-present
from /buyer/start" link (see `publisher/app/ssi/verifier_routes.py`).

Main reason codes and their banner copy:

| reason | EN banner | JA |
|---|---|---|
| `dataset_mismatch` | The presented VC is bound to a different dataset. | 提示された VC のデータセットが、要求されたデータセットと一致しません。 |
| `action_not_allowed` | The presented VC does not include the required action (read). | 提示された VC では、このデータの読み取り権限がありません。 |
| `purpose_mismatch` | The presented VC's allowed_purposes does not cover this purpose. | 提示された VC の許可目的に、今回の用途が含まれていません。 |
| `missing_entityType` etc. | DataUserVC is missing *XXX*. | DataUserVC に *XXX* が含まれていません。 |
| `verification_failed` | VC verification failed. | VC の署名検証に失敗しました。 |

**Expected flow (not yet validated on a real device — implementation in Stage T):**

1. PC opens `/buyer/start?ds=home/event/possible_littering` → QR appears.
2. From the wallet, intentionally pick a PurchaseViewerVC bound to a *different* dataset (e.g. `home/event/another_dataset`) and present it.
3. publisher receives `/verifier/response` and records `{"verified": false, "reason": "dataset_mismatch"}`.
4. The PC's `/verifier/status` long-poll returns `status: "denied"` with the `human_message_*` strings.
5. The page replaces the QR with `<div class="deny-banner">The presented VC is bound to a different dataset.</div>` plus a re-present link to `/buyer/start?ds=...`.
6. `docker compose logs publisher` shows the audit row `_write_audit(action="presentation", reason="dataset_mismatch", verified="false")`.

To reproduce on a real device, reuse the §10.5 setup but issue an extra
`POST /issuer/offer?vc_kind=PurchaseViewerVC&claim_id=<other-claim>` so the
wallet ends up holding a 4th card bound to a different dataset, then scan
the QR with that card selected.

## Where to go next

- [Mobile SSI Wallet sample](ha-ssi-wallet.md) — bring up the Phase 2 wallet
- [DataUserVC × Tiered Access Spec](data-user-vc-tiered-spec.md) — design rationale
- [USB Webcam Event Sharing sample](webcam-event-sharing.md) — listing comes from here
