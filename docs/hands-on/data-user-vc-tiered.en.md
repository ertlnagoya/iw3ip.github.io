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

## 11. PWA Provider (the data-provider side)

§10 covered the **receiver** side reading data through the PWA Viewer.
§11 covers the symmetric **provider** side: a PWA where a data provider
authenticates with SSI and uploads + publishes data through the same
publisher container — `/provider/start` + `/provider` + `/provider/publish`
replace the curl steps in `provider_with_media.py` with a browser-only
flow.

```
[PC browser] ── /provider/start                      [publisher]
   │  ↓ QR + long-poll                                  │
   │     scan QR with iPhone wallet → present SellerVC ►│ mint SellerToken
   │  ↑ /verifier/status returns seller_token + licensed_datasets
   │  PC page renders the licensed_datasets list
   │  ↓ pick one and click "Continue to upload"          │
   │  /provider?pt=<seller_token>&ds=<dataset_id>        │
   │     file pick / camera capture / browser recorder ►│ /media/upload
   │     ↑ URL + (CID)                                   │
   │     "Publish" button                                │
   │     POST /provider/publish (Bearer SellerToken) ───►│ use_seller_token
   │                                                     │ → process_message
   │  ↑ {"status":"allowed", ...}                         │
```

`/provider/start` is the SellerVC counterpart of `/buyer/start` — same
OID4VP plumbing, just with `vc_kind=SellerVC` and a SellerToken minted
on success.

### 11.1 Open the page

No special prep beyond having `docker compose ... up -d publisher`
running. From a PC browser:

```
http://192.168.68.53:8080/provider/start?ds=home/event/possible_littering
```

`ds=` is a **hint** only (SellerVC verification isn't dataset-scoped),
but it shows in the page header so the operator knows which dataset
they're working on.

### 11.2 Present a SellerVC (OID4VP)

PC shows a QR. Scan it from the iPhone wallet and present a **SellerVC**
(not a PurchaseViewerVC — they're different cards in the wallet).

On success the page swaps to a result panel:

- Green banner "SellerVC 提示が承認されました" / "Presentation accepted"
- The full **licensed_datasets** list straight from the SellerVC claims
- `seller_id` and the SellerToken expiry (default 24 h)
- A "Continue to upload →" button next to a radio list

Click through to land on `/provider?pt=<seller_token>&ds=<chosen ds>`.

!!! note "Deny UX"
    If the SellerVC is missing `seller_id` or `licensed_datasets`,
    `/verifier/status` surfaces `human_message_ja` / `human_message_en`
    via a red banner (reason codes: `missing_seller_id` /
    `missing_licensed_datasets`), and offers a one-click reload to retry
    with a different VC.

### 11.3 Three ways to provide data

The `/provider` page exposes **three peer-equivalent input modes**, all
funneling through the same `/media/upload` URL/CID delivery — the
receiver side at `/viewer` cannot tell which one was used.

| Mode | Behaviour | Best on |
|---|---|---|
| 📁 File picker | OS file dialog, pick an existing file | PC + phone |
| 📷 Camera capture | `<input capture="environment">` — iPhone Safari opens the rear camera directly; PC falls back to file picker | iPhone (capture-on-the-spot) |
| 🔴 Browser recorder | `MediaRecorder` against `getUserMedia({video,audio})`. Click 録画開始, see live preview, click 停止 to auto-upload as WebM/VP9 (Safari falls back to MP4) | PC webcam |

All three call the same `uploadBlob()` pipeline: preview →
`POST /media/upload` → result panel → enable the Publish button.
Storage stays bounded because uploads are deduplicated by SHA-256.

!!! tip "Why browser recording matters for demos"
    PC webcam → SSI auth → publish in **one browser tab** is a 30-second
    demo of "data with provenance attached." It complements the
    [USB Webcam Event Sharing sample](webcam-event-sharing.md): that
    one streams continuously, this one captures discrete clips with
    on-the-spot attribution.

When you stop a recording, the `video_duration_sec` form field is
**auto-filled** with the measured length.

### 11.4 Publish + the licensed_datasets gate

Once an upload completes, the Publish button enables. The form holds:

- `topic` — defaults to `homeassistant/event/<last segment of ds>`
- `purpose` — defaults to `community_cleaning`
- `camera_id`, `video_duration_sec`
- `extra payload keys` — optional JSON merge

Click **Publish event**. The browser POSTs to `/provider/publish` with
`Authorization: Bearer <seller_token>`. Server-side:

1. `schemas.normalize(topic, payload)` derives `dataset_id` from the topic.
2. `ssi_state.use_seller_token(token, dataset_id=<resolved>)` enforces
   that `dataset_id ∈ licensed_datasets[]`.
3. On success it calls `processor.process_message` (same path as
   `/simulate/publish`).
4. Response is augmented with `seller_token_jti` + `register_count`.

`SellerToken` is **multi-use** (same semantics as `/marketplace/register`),
so a single SellerVC presentation covers a whole upload session — you
can watch `register_count` grow in the response details panel.

Deny cases:

| Failure | HTTP | reason |
|---|---|---|
| missing `Authorization` header | 401 | `missing_authorization_header` |
| unknown SellerToken | 401 | `seller_token_unknown` |
| expired SellerToken | 401 | `seller_token_expired` |
| dataset (resolved from topic) not in `licensed_datasets[]` | 403 | `seller_token_dataset_not_licensed` |
| topic not recognised by `normalize()` | 400 | `unsupported_topic:...` |

All denials are recorded in `/audit/logs` with `action=deny`.

### 11.5 Confirm from the receiver side

The footer of `/provider` auto-generates a link to `/buyer/start` for
the same dataset. Open it in a separate tab, present a Tier 3
PurchaseViewerVC, and the image / video you just published renders in
`/viewer` (the same screen as §10.5 screenshots).

That makes the full **provide → auth → publish → receive → verify** loop
a **two-tab** experience in the browser.

### 11.6 Symmetry with `/buyer/start`

| Aspect | `/buyer/start` (§10) | `/provider/start` (§11) |
|---|---|---|
| VC presented | PurchaseViewerVC | **SellerVC** |
| Token minted on verify | ViewerToken (TTL 60 s) | **SellerToken** (TTL 24 h) |
| Used as Bearer for | `/platform/data` | **`/provider/publish`** |
| Single-use vs multi-use | multi-use (continuous read) | **multi-use** (one auth, many publishes) |
| Dataset scoping | VC is bound to a dataset_id | SellerVC carries `licensed_datasets[]` set |
| Deny message shape | §10.7 common format | same `human_message_ja/en` shape |

### 11.7 Troubleshooting

| Symptom | Fix |
|---|---|
| `/provider/start` says no SellerVC offer received | Issue one first: `POST /issuer/offer?vc_kind=SellerVC&seller_id=...&licensed_datasets=...` and scan the offer URI from the wallet |
| 📷 Camera capture opens the file picker on iPhone Safari | Some iOS versions don't honor the combination `accept="image/*,video/*" capture="environment"`. Switching `accept` to `video/*` only forces the camera reliably |
| 🔴 Browser recorder button does nothing | `getUserMedia` only works on HTTPS or `localhost`. Opening over a LAN IP (`http://192.168.x.x`) makes the browser deny camera/mic permission. Use `localhost:8080` or run behind HTTPS |
| `/provider/publish` returns 403 `seller_token_dataset_not_licensed` | The dataset_id derived from `topic` isn't in your SellerVC's `licensed_datasets[]`. Example: `topic=homeassistant/event/possible_littering` → dataset_id is `home/event/possible_littering` |
| Publish response has `status: send_error` | The publisher's `PLATFORM_API_URL` is unreachable. The SellerToken gate did pass and `register_count` did increment — auth was OK, downstream delivery failed |

### 11.8 Real-device validation status

As of 2026-04-30 the following paths are **not yet validated on a real
device** — screenshots and observed values will be appended once they
are exercised:

- iPhone Safari `capture="environment"` → on-the-spot capture → upload → publish
- PC Chrome MediaRecorder → auto-upload → publish
- Firefox VP8 fallback
- macOS Safari MP4 codec fallback

The implementation and OID4VP plumbing have been verified at the unit
test level (78/78 + 71/71 pass) across the three PRs
`feat/stage-t-provider-start-c1`, `feat/stage-t-provider-c2`, and
`feat/stage-t-provider-camera`.

## 12. Semantic-level redaction (VLM + face blur)

§1–§11 gate access by **dropping media keys** — Tier 2 hides video,
Tier 1 hides image and video. §12 derives **new content from the same
source via VLM inference + face/PII blurring** and projects those
derivatives per tier. Tier 1 stops being "you get nothing useful" and
ships a **PII-redacted summary** instead.

Design rationale: see [DataUserVC × Tiered Access Spec § "Tier extension"](data-user-vc-tiered-spec.md#tier-extension-semantic-level-redaction-vlm).

### 12.1 Updated tier definitions

| Tier | access level | Example (score) | Derivatives shipped |
|---|---|---|---|
| **3** Full | `full` | gov + crime + ISO27001 (80) | raw image / video + redacted image + full text + summary text |
| **2** Access | `access` | enterprise + research + ISO27001 (75) | **face/PII-blurred image** + **detailed text** (named entities) + summary |
| **1** Summary | `summary` (new) | enterprise + research only (60–) | **summary text only** (PII-redacted, no image) |
| 0 Denied | `denied` | unqualified (<60) | claim is rejected |

A new `summary` value joins the `access_level` enum; `denied` is
unchanged. The `/platform/data` row schema gains:

| Key | Content | Visible at tier |
|---|---|---|
| `image_url_redacted` | URL of an image with faces / people / license plates blurred | 2 + 3 |
| `image_cid_redacted` | IPFS CID of the redacted image (when option C is on) | 2 + 3 |
| `description_full` | VLM-generated detailed description (named entities present) | 2 + 3 |
| `description_summary` | VLM-generated summary (PII-redacted) | 1 + 2 + 3 |
| `description_model` | VLM model id + version (audit) | every tier |
| `description_generated_at` | Inference timestamp (ISO8601) | every tier |
| `processing_warnings` | List of degraded steps (`vlm_unavailable`, `redaction_unavailable`) | every tier |

### 12.2 Bring up with `--profile vlm`

VLM and face blur are **opt-in**. With the profile off, §1–§11 keep
working under the legacy 3-tier projection.

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml --profile vlm up -d \
  publisher hardhat bridge mosquitto vlm vlm-pull
```

Switch the publisher backends on:

```bash
export VLM_BACKEND=ollama
export IMAGE_REDACTION_BACKEND=opencv
docker compose -f infra/docker-compose.yml --profile vlm up -d publisher
```

Wait for `vlm-pull` to finish pulling `llava` (first time only, multi-GB):

```bash
docker compose -f infra/docker-compose.yml logs -f vlm-pull
# wait for "vlm model llava ready" then exit
```

Toggling profile off vs on against the same dataset shows that the
`description_*` keys appear only when on.

### 12.3 Four DataUserVC offers

The §2 set extended with a **summary-tier** profile.

#### 12.3.a Tier 3 (full) — same as §2a

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=GovernmentOrganization&purpose=CrimeSearch&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

#### 12.3.b Tier 2 (access) — same as §2b

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

#### 12.3.c Tier 1 (summary, new) — enterprise + unknown purpose + legal compliance only

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=unknown&legal_compliance=true&data_handling_policy=other&misuse_record=false' | jq .
# score = 20 + 5 + 15 + 0 + 10 = 50 -> summary (only when VLM profile is on)
```

#### 12.3.d Tier 0 (denied) — same as §2c

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true' | jq .
```

### 12.4 Four `/marketplace/claim` calls + `/platform/data` comparison

Same routine as §3: claim → PurchaseViewerVC → present → ViewerToken → fetch.
With VLM profile on:

| Profile | `event` | `image_url` | `video_url` | `image_url_redacted` | `description_full` | `description_summary` |
|---|---|---|---|---|---|---|
| 12.3.a Tier 3 (full) | yes | yes | yes | yes | yes | yes |
| 12.3.b Tier 2 (access) | yes | **no** | **no** | yes | yes | yes |
| 12.3.c Tier 1 (summary) | yes | **no** | **no** | **no** | **no** | yes |
| 12.3.d Tier 0 (denied) | the claim itself returns `access_level: "denied"` |

With profile **off** the legacy 3-tier projection runs (no derivative
keys appear). Claim 12.3.c then resolves to `denied` since `summary`
is profile-on only.

### 12.5 Verifying face blur

Open `image_url_redacted` from a Tier 2 response in a browser. You
should see **the same scene as `image_url` (Tier 3 only) but with
faces blurred**.

| Original (`image_url`, Tier 3 only) | Blurred (`image_url_redacted`, Tier 2+) |
|---|---|
| ![pre-redaction (TBD)](images/data-user-vc-tiered/vlm/12-pre-blur.png){ width=300 } | ![post-redaction (TBD)](images/data-user-vc-tiered/vlm/12-post-blur.png){ width=300 } |

Internally the publisher:

1. fetches the source from `/media/<sha>.<ext>`
2. runs OpenCV Haar-cascade face detection
3. applies a 51×51 Gaussian blur to each face region
4. re-encodes in the original format
5. POSTs the result back to its own `/media/upload` (which dedups +
   optionally pushes to IPFS)
6. surfaces the resulting URL/CID as `image_url_redacted` /
   `image_cid_redacted`

Subsequent uploads of the same source skip the blur entirely — the
sha256 dedup hits.

!!! note "MVP limitations"
    The Haar cascade catches **frontal faces only**. Side profiles,
    occluded faces, and low-resolution faces pass through. License
    plates / ID badges / sharp-rectangle screen detection are
    [future work in the spec](data-user-vc-tiered-spec.md#tier-extension-semantic-level-redaction-vlm).

### 12.6 Inspecting VLM output

Compare `description_full` (Tier 2/3) against `description_summary`
(Tier 1+) to confirm proper-noun stripping:

```bash
curl -s -H "authorization: Bearer $VIEWER_TOKEN_TIER3" \
     localhost:8080/platform/data?dataset_id=home/event/possible_littering \
  | jq '.rows[0] | {description_full, description_summary, description_model}'
```

Example output:

```json
{
  "description_full": "John Smith dropped a Coca-Cola bottle near the Hibiya station entrance at around 14:32.",
  "description_summary": "An adult dropped a piece of litter near a public location during the afternoon.",
  "description_model": "ollama/llava"
}
```

The `_full` line keeps the person's name, brand, and place. The
`_summary` line collapses to "an adult / public location / afternoon".

!!! warning "Possible redaction leaks"
    LLaVA-class VLMs are probabilistic; **`description_summary` may
    structurally still contain PII** (e.g. clothing details that
    re-identify, building names that look generic). The spec
    [records detection as a research TODO](data-user-vc-tiered-spec.md#tier-extension-semantic-level-redaction-vlm)
    (NER diff + PII dictionary). For production, queue Tier 1 outputs
    for human review.

### 12.7 Degrade behavior

If either VLM or blur fails, publishing keeps going and
`processing_warnings[]` tells the receiver what was skipped.

Stop the VLM only:

```bash
docker compose -f infra/docker-compose.yml stop vlm
# /provider/publish still succeeds; /platform/data carries
# processing_warnings: ["vlm_unavailable"]
# image_url_redacted is still generated (face blur is VLM-free)
# description_* keys are absent
```

Stop both:

```bash
docker compose -f infra/docker-compose.yml stop vlm
# also clear IMAGE_REDACTION_BACKEND on the publisher and restart
# /platform/data: processing_warnings: ["vlm_unavailable", "redaction_unavailable"]
# Tier 2 / Tier 1 receivers get only the raw keys (= profile-OFF parity)
```

`image_url` / `video_url` always remain visible at Tier 3 even when
derivatives are degraded — the receiver contract is "expected key
missing → check `processing_warnings`".

### 12.8 Relation to §11 (PWA Provider)

Images uploaded through `/provider` (§11) automatically flow through
the VLM pipeline when profile vlm is on. The **provider page itself
needs no change**; derivative generation is server-side. The
receiver-side `/viewer` currently prefers raw keys (`image_url` /
`video_url`); a follow-up PR is needed to teach `/viewer` to display
`image_url_redacted` when only the redacted variant is allowed
([scenario tracked below](#12-9)).

### 12.9 Troubleshooting

| Symptom | Fix |
|---|---|
| `vlm-pull` returns "pull model manifest: file does not exist" | The container can't reach the image registry. From inside: `curl https://registry.ollama.ai`. Or change `VLM_MODEL` to a different name (e.g. `llava:7b`) |
| First `/provider/publish` times out | LLaVA cold start (loading into VRAM) takes 30–60s. Confirm `vlm-pull` reported "model ready"; subsequent calls are fast |
| `description_full` and `description_summary` come back identical | LLaVA may have ignored the prompt difference. `docker compose logs vlm` should show two distinct `/api/generate` calls; if not, check the prompt strings in `vlm_client.py` |
| `image_url_redacted` looks unblurred | Haar cascade catches only frontal faces. Side / occluded / small faces pass through — swap in a DNN detector if needed |
| `description_*` keys appear with profile OFF | Bug — the legacy projection should never emit derivative keys. The regression test `test_pipeline_no_injectors_keeps_legacy_envelope` covers this; if you see it, file an issue |
| `processing_warnings: ["vlm_unavailable"]` keeps firing | Ollama not responding or `VLM_API_URL` wrong. From the publisher: `docker compose exec publisher curl http://vlm:11434/api/version` |

### 12.10 Real-device validation log

| Scenario | Environment | Status | Observations |
|---|---|---|---|
| **V1** Tier 3 has every key | macOS Chrome + Ollama (llava) | ⚠️ partial (2026-04-30) | Pipeline logs confirm `vlm_describe_done full_len=280 summary_len=172` + `opencv_blur_faces detected=1`. **Per-tier `/platform/data` projection deferred — needs the iPhone OID4VP loop** |
| **V2** Tier 2 has redacted + text, no raw image | macOS Chrome | ⏳ pending | OID4VP needed; deferred |
| **V3** Tier 1 (summary) is text-only | macOS Chrome | ⏳ pending | OID4VP needed; deferred |
| **V4** Face blur visual confirmation | StyleGAN2 synthetic face (no real PII) → publish → Tier 2 receiver | ✅ **verified (2026-04-30)** | OpenCV detected 1 face, applied Gaussian blur (51×51), re-uploaded as a separate file. Face is unrecognizable in the output.<br>📷 [original](images/data-user-vc-tiered/vlm/V4-original.jpg) → [redacted](images/data-user-vc-tiered/vlm/V4-redacted.jpg) |
| **V5** description_full vs summary quality | StyleGAN2 sample | ⚠️ partial | VLM 2-stage prompting completed (`full_len=280`, `summary_len=172`). **Actual text diff requires ViewerToken-based fetch; deferred** |
| **V6** VLM-down degrade | timeout 60s effectively triggered vlm_unavailable | ✅ verified (2026-04-30) | `vlm describe failed: ollama call failed: timed out` → `processing_warnings: ["vlm_unavailable"]` emitted; `image_url_redacted` still generated. Δ3 timeout fix bumps default to 180s; CPU environments need `VLM_TIMEOUT_SEC=600` |
| **V7** Redaction-leak survey | large sample | 🔬 research | Spec's post-check implementation prerequisite; not started in this validation pass |

#### Bugs / constraints surfaced during V1–V6

Three operational findings emerged during this pass:

| # | Symptom | Cause / Resolution |
|---|---|---|
| 1 | `/provider/publish` followed by `vlm describe failed: ollama call failed: timed out` | CPU LLaVA-7B inference takes **~186s per stage** (even on a 1×1 pixel image). describe() needs 2 stages → ~6 min total |
| 2 | Δ3's default `VLM_TIMEOUT_SEC=60` couldn't complete | [Blockchain_IoT_Marketplace#45](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/45) bumps to 180s; CPU setups need `VLM_TIMEOUT_SEC=600` |
| 3 | LLaVA-7B on CPU is impractical for hands-on workshops | Use a smaller model (`bakllava`, `moondream`) or GPU / external API. Add to docs |

#### moondream re-test (2026-04-30 addendum)

Pulled `moondream` (1.7 GB, ~1/3 of llava) and re-ran the same pipeline:

| Metric | llava | moondream |
|---|---|---|
| Model size | 4.7 GB | **1.7 GB** |
| describe() time on CPU | ~6 min | **~21 s – 5 min** (depends on prompt + cold/warm) |
| Δ3 two-stage long prompts | Long output (`full_len=280, summary_len=172`) | **Fragments only** (`full_len=3, summary_len=10`) |
| Simple prompt ("Describe this image.") | Works but verbose | **High quality**: "A man with a beard and glasses... blurred green landscape" |

**Finding**: a small VLM (moondream) is dramatically faster but **does not respond well to the current long 2-stage prompts**. Either tune prompts per-backend (a `prompts: {model -> str}` dict in `vlm_client.py`) or unify all backends on shorter prompts.

Short-prompt unification trades off redaction-strength expressiveness, so a per-backend prompt dict is the cleaner path. Tracked as a TODO for a follow-up PR.

#### V4 visual comparison

| Original (synthetic) | OpenCV Haar-cascade blurred |
|---|---|
| ![original](images/data-user-vc-tiered/vlm/V4-original.jpg){ width=300 } | ![redacted](images/data-user-vc-tiered/vlm/V4-redacted.jpg){ width=300 } |
| Face details (eyes, nose, mouth) clearly visible | Central rectangular face region completely Gaussian-blurred; hair, ears, beard, and background unchanged |

The image is a **StyleGAN2 synthetic face** — no real-world PII involved.

The implementation is verified at unit-test level (155/155 pass) in
[Blockchain_IoT_Marketplace#44](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/44),
with the timeout fix in
[#45](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/45).

## Where to go next

- [Mobile SSI Wallet sample](ha-ssi-wallet.md) — bring up the Phase 2 wallet
- [DataUserVC × Tiered Access Spec](data-user-vc-tiered-spec.md) — design rationale
- [USB Webcam Event Sharing sample](webcam-event-sharing.md) — listing comes from here
