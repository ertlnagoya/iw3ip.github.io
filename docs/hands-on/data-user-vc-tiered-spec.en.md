# Tiered access design spec

A design document defining how the Publisher narrows the camera view it
exposes — to **event / image / video** in three tiers — based on the
trust attributes carried by the data recipient's **DataUserVC**.

This is a design document, not a step-by-step. For the walk-through, see
[Tier the response by trust](data-user-vc-tiered.md).

## What you'll learn

- How the new **DataUserVC** relates to the existing five VCs
  (ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC)
- How the trustScore logic from Li et al.'s `DataUserVerifier.sol` is folded
  into the Phase 2 Publisher
- The unidirectional flow: purchase claim attributes (`data_user_attrs`) → trustScore → allowed_views
- Separation of concerns between data sources (HA / RaspberryPi / USB
  Webcam) and the Publisher

## Common pitfalls

- Evaluating "can this DataUserVC see the camera" **on-chain** breaks
  whenever the local Hardhat node restarts. We evaluate in **Publisher
  Python** instead.
- trustScore is **not** a simple sum: `full` requires
  `score >= 80 AND entityType ∈ {GovernmentOrganization, Police}`.
- `allowed_views` on a PurchaseViewerVC is **frozen when the VC is
  issued**; we never promote it after the fact. A ViewerVC carries no
  `allowed_views`, so its view is always `event` only.

## Goal

On top of the existing Phase 2 flow (VC presentation → verification →
token mint → `/platform/data`), we tier **what** the holder can fetch by
their DataUserVC attributes:

- entityType (GovernmentOrganization / Police / Enterprise / ResearchOrganization)
- purpose (CrimeSearch / TrafficManagement / Research)
- legalCompliance (true / false)
- dataHandlingPolicy (ISO27001 / other)
- misuseRecord (history of misuse)

We score these with the same weights as Li et al.'s `DataUserVerifier.sol`,
then collapse the score into `full` / `access` / `denied`.

## Prerequisites

The change is contained to **Publisher's verification, token, and
projection layers**. We reuse:

- The five existing VCs (ConsentVC / ViewerVC / ServiceVC /
  PurchaseViewerVC / SellerVC)
- OID4VCI / OID4VP / DCQL surfaces
- The bridge ↔ Hardhat Purchase event path
- All device-side code (Webcam / HA / Demo simulator) — unchanged

**`DataUserVerifier.sol` is treated as the canonical *logic source***;
we do not call it on-chain. See "Why no on-chain call" below.

## Architecture

```
[/marketplace/claim (data_user_attrs = the five DataUserVC attributes)]
        v
[trust_score.evaluate(entityType, purpose, legalCompliance,
                      dataHandlingPolicy, misuseRecord)]
        | -> trust_score: int, access_level: full/access/denied,
        |    allowed_views: subset of {event, image, video}
        v
[MarketplaceClaim.allowed_views]
        v
[PurchaseViewerVC issuance] -> VC.claims.allowed_views
        v
[/verifier/* with PurchaseViewerVC presentation]
        v
[ViewerToken.allowed_views]
        v
[/platform/data] selects image_cid / video_cid / video_duration_sec
```

`DataUserVC` alone mints **no token**. The same attributes are the *material*
for issuing a PurchaseViewerVC.

The original design had the wallet present the DataUserVC over OID4VP and derive the trust level from that check. There is no presentation definition for DataUserVC yet, so this path is not used today; the trust level comes from the `data_user_attrs` passed to `/marketplace/claim`.

## Relation to the five existing VCs

| VC | Role | Token | Carries allowed_views |
|---|---|---|---|
| ConsentVC | Provider consent | PolicyToken (5 min, single) | no |
| ViewerVC | Restricted viewer | ViewerToken (60 s, multi) | no (always event only) |
| ServiceVC | Service provider | ServiceToken (1 h, multi) | no |
| PurchaseViewerVC | Post-purchase view | ViewerToken (60 s, multi) | yes |
| SellerVC | Seller | SellerToken (24 h, multi) | no |
| **DataUserVC (new)** | **Recipient attributes** | **none (material only)** | **n/a** |

## Trust scoring

Implemented as a pure function in `publisher/app/ssi/trust_score.py`,
mirroring `DataUserVerifier.sol`. Tests pin the two together.

### Weights

Strings are matched **uppercase, exact** (spaces included). Unknown
labels fall back to **5**.

```
entityType (uppercase exact match):
  GOVERNMENTORGANIZATION | POLICE       -> 35
  ENTERPRISE                            -> 20
  RESEARCH ORGANIZATION                 -> 15  (note the space)
  unknown                               -> 5

purpose (uppercase exact match):
  CRIME SEARCH                          -> 25  (note the space)
  TRAFFIC MANAGEMENT                    -> 20
  RESEARCH                              -> 15
  unknown                               -> 5

legalCompliance == true                 -> +15
dataHandlingPolicy == "ISO27001"        -> +15
misuseRecord == false                   -> +10
misuseRecord == true                    -> -10
```

### Sample scores observed on iPhone real-device runs

| Profile | entityType | purpose | legal | policy | misuse | score |
|---|---|---|---|---|---|---|
| Tier 3 (full)   | `GovernmentOrganization` | `CrimeSearch` | true  | ISO27001 | false | **80** = 35+5+15+15+10 |
| Tier 2 (access) | `Enterprise`             | `Research`    | true  | ISO27001 | false | **75** = 20+15+15+15+10 |
| Tier 1 (denied) | `Enterprise`             | `Research`    | false | Other    | true  | **25** = 20+15+0+0-10  |

The `CrimeSearch` label does not match the Solidity-side
`"CRIME SEARCH"` (space-separated) string, so the current Python
implementation falls back to **5** for purpose (which is why Tier 3
hits 80, not 100). Hands-on output crosses the right tier boundaries
either way; if/when ZK / on-chain parity is wired in, the UI's input
labels should be normalized to the spaced form (follow-up TODO).

### Threshold → access level

```
score >= 80 AND entityType ∈ {GovernmentOrganization, Police}
                                          -> "full"
score >= 60                               -> "access"
otherwise                                 -> "denied"
```

### Access level → allowed_views

```
"full"   -> ["event", "image", "video"]
"access" -> ["event", "image"]
"denied" -> []   # no image or video
```

The original design rejected the claim itself on a `denied` verdict. The current implementation does not reject the claim; an empty `allowed_views` is treated as events only (`PurchaseViewerVC.event` is issued).

## API surface (deltas only)

### `GET /issuer/offer` (extended)

```
?type=DataUserVC
&entity_type=GovernmentOrganization
&purpose=CrimeSearch
&legal_compliance=true
&data_handling_policy=ISO27001
&misuse_record=false
```

For `DataUserVC`, `entity_type`, `legal_compliance`,
`data_handling_policy` and `misuse_record` are required; missing → 400.
`purpose` defaults to `read` when omitted.

### `GET /verifier/request` / `GET /verifier/request_object`

In the original design, `GET /verifier/request?vc_kind=DataUserVC` would create the presentation
request, and the DCQL `claims` in `/verifier/request_object` (fetched by the
wallet) would ask for the five attributes above (plus `subject_id`).

!!! note "Current implementation"
    The publisher's presentation check has code for DataUserVC, but the course repository does not yet provide a presentation definition for DataUserVC. So `GET /verifier/request?vc_kind=DataUserVC` returns 404. The hands-on passes the same five attributes directly in the `data_user_attrs` of `/marketplace/claim` to decide the trust level.

### `POST /verifier/*` (presentation endpoints)

When a `DataUserVC` is presented (as the note above says, this path is not reachable today because there is no presentation definition):

- the presentation match (`_local_pex_fallback`, used for DCQL or when the sidecar is absent) checks that all five claims are present, and the post-PEX checks then check `entityType`, `purpose` and `dataHandlingPolicy` again
- **no token is minted**
- response: `{trust_score, access_level, allowed_views, holder_did}`

### `POST /marketplace/claim` (extended)

```json
{
  "merchandise_address": "0x...",
  "buyer_eth_addr": "0x...",
  "tx_hash": "0x...",
  "dataset_id": "home/event/possible_littering",
  "data_user_attrs": {
    "entityType": "GovernmentOrganization",
    "purpose": "CrimeSearch",
    "legalCompliance": true,
    "dataHandlingPolicy": "ISO27001",
    "misuseRecord": false
  }
}
```

`merchandise_address`, `buyer_eth_addr`, `tx_hash`, and `dataset_id` are required. The response carries `claim_id`, `offer_url`, and `deeplink` (plus `merchandise_address`, `buyer_eth_addr`, `tx_hash` and `created`); the verdict (`allowed_views` etc.) is not in the response. It is reflected in the kind of PurchaseViewerVC that gets issued (`PurchaseViewerVC.full` / `.access` / `.event`).

If `data_user_attrs` is omitted, we claim with the minimum
`allowed_views=["event"]`.

### `GET /platform/data`

Filtered by `ViewerToken.allowed_views`:

| allowed_views contains | Output fields |
|---|---|
| `event` | the event JSON itself |
| `image` | adds `image_cid`, `image_url` |
| `video` | adds `video_cid`, `video_url`, `video_duration_sec` |

Keys not allowed are **omitted** (not nulled).

## Why no on-chain call

`DataUserVerifier.sol` is a research prototype with mismatches:

- VC format is W3C VC v1 JSON-LD (Phase 2 uses SD-JWT VC)
- DID method is `did:key` (Phase 2 uses `did:jwk`)
- gas + confirmation does not fit the OID4VP latency budget
- the local Hardhat node loses state on restart

So we re-implement only **the weights and thresholds** in Python and
keep the contract as a future production-migration surface.

`trust_score.py` is maintained as a **mirror** of `DataUserVerifier.sol`;
when either changes, the test suite enforces parity.

## ssi-ui

The `ssi-ui/` Next.js prototype is **deprecated**. Future work goes to
the Phase 2 wallet (`iw3ip-wallet`) + Publisher.

## Test plan

At the initial implementation, `tests/test_data_user_vc_tiered.py` covered 13 cases (case counts in parentheses). Later features have grown this file to 42 tests.

1. `evaluate()` pure-function tests (4)
   - GovernmentOrganization × Research × ISO27001 → full
   - Enterprise × Research × ISO27001 → access
   - low score → denied
   - even with score ≥ 80, non-gov/police never reaches full
2. `evaluate_from_claims()` wrapper (1)
3. Issuer metadata lists `DataUserVC` (1)
4. `/issuer/offer?type=DataUserVC` requires its attributes (400 without them; four are required, `purpose` is optional) (1)
5. `/marketplace/claim` trust promotion / default to event-only (2)
6. `PurchaseViewerVC` inherits `allowed_views` from the claim (1)
7. `/platform/data` projects Tier 1 / 2 / 3 (3)

At the initial implementation, the suite totalled 88 tests (75 existing + 13 new), and no existing test failed.

## Future work

- Bundle DataUserVC into a Verifiable Presentation
- Re-deploy `DataUserVerifier.sol` on a low-fee chain (e.g. Polygon
  zkEVM) and demote `trust_score.py` to a fallback
- Make `allowed_views` time-window aware (e.g. video only at night)

## Tier extension: semantic-level redaction (VLM)  {#tier-vlm}

The design so far (matching §1–§7 of the [hands-on page](data-user-vc-tiered.md) and §8–§9 of its [follow-up page](data-user-vc-tiered-media.md)) gates access by **dropping media keys** (Tier 2 hides video,
Tier 1 hides image and video). The next step is to **derive new content
from the same source via VLM inference + face blurring (frontal faces in the MVP) and project
those derivatives per tier**. Tier 1 is no longer "you get nothing
useful"; it now ships a **PII-redacted summary** so a low-trust
recipient can know *what happened* without learning *who did it*.

### Updated tier definitions

| Tier | access level | Example (score) | Derivatives shipped |
|---|---|---|---|
| **3** Full | `full` | gov + crime + ISO27001 (80) | raw image / video + every text derivative |
| **2** Access | `access` | enterprise + research + ISO27001 (75) | **face-blurred image** + **detailed text** (named entities present) |
| **1** Summary | `summary` (new) | enterprise + unknown purpose + legalCompliance only (50–59) | **summary text only** (PII-redacted, no image) |
| 0 Denied | `denied` | unqualified (<50) | events only (the current implementation does not reject the claim) |

A new `summary` value is added to `access_level`: only when the publisher's
`VLM_BACKEND` is non-empty (the VLM extension is on), scores 50–59 map to `summary`. With `VLM_BACKEND` empty, the `denied`
semantics are unchanged — `score < 60` is still judged `denied` and can view events only.

### Derivative schema

`/platform/data` row objects gain these keys:

| Key | Content | Visible at tier |
|---|---|---|
| `image_url_redacted` | URL of an image with faces blurred (frontal faces only in the MVP) | 2 + 3 |
| `image_cid_redacted` | IPFS CID of the redacted image (when option C — IPFS delivery, [hands-on §9](data-user-vc-tiered-media.md#9-option-c-local-kubo-ipfs-daemon-for-distributed-delivery) — is on) | 2 + 3 |
| `description_full` | VLM-generated detailed description (named entities present) | 2 + 3 |
| `description_summary` | VLM-generated summary (PII-redacted) | 1 + 2 + 3 |
| `description_model` | VLM model id + version (audit) | every tier |
| `description_generated_at` | Inference timestamp (ISO8601) | every tier |

The existing `image_url` / `image_cid` / `video_url` / `video_cid` keys
become **Tier 3 only**; the old "Tier 2 sees `image_url`" projection is
replaced by `image_url_redacted`.

### New allowed_views labels

```
allowed_views ⊆ {
  "event",                # existing: metadata
  "image",                # existing: raw image_url / image_cid (Tier 3 only)
  "video",                # existing: raw video_url / video_cid (Tier 3 only)
  "image_redacted",       # new: image_url_redacted / image_cid_redacted (Tier 2+)
  "description_full",     # new: description_full (Tier 2+)
  "description_summary",  # new: description_summary (Tier 1+)
}
```

### Updated tier → allowed_views mapping

```
"full"    -> ["event", "image", "video",
              "image_redacted",
              "description_full", "description_summary"]
"access"  -> ["event",
              "image_redacted",
              "description_full", "description_summary"]
"summary" -> ["event",
              "description_summary"]
"denied"  -> []
```

`full` is a strict superset of `access`. `access` is the middle band:
no raw media but a blurred image plus full text. `summary` is text
only — no image at all, redacted or otherwise.

### VLM pipeline contract

When the publisher's `VLM_BACKEND` (and `IMAGE_REDACTION_BACKEND`) are set, `/provider/publish`
gains two new steps (`--profile vlm` only starts the Ollama containers):

```
provider page
  ├── /media/upload stores the source image (existing)
  └── POST /provider/publish
        └── pipeline.process_message
              ├── existing: consent / policy evaluation (stops here on deny)
              ├── (new) VLMClient.describe(image_url)
              │     -> {description_full, description_summary, description_model, description_generated_at}
              ├── (new) ImageRedactor.blur_pii(image_url)
              │     -> {image_url_redacted, image_cid_redacted}
              └── existing: platform_client.send(envelope)
```

Degrade policy when VLM is missing or fails:

| Condition | Publisher behaviour |
|---|---|
| VLM extension off (`VLM_BACKEND` empty) | derivative keys are **not generated** (legacy behaviour — Tier 1/2 fall back to the old "key drop" model) |
| extension on, VLM call failed | description keys omitted; `image_url_redacted` is still attempted (face blur doesn't need the VLM) |
| extension on, blur failed | redacted-image keys omitted; descriptions still ship. Tier 2 receivers get text without a redacted image |
| extension on, both succeed | all keys generated |

Degraded responses carry **`processing_warnings: ["vlm_unavailable", ...]`**
so the receiver can tell what was skipped.

### VLM backends

- **MVP**: local LLaVA via Ollama as a compose service (`ollama run llava`)
- **Future**: swap for a larger multimodal LLM (Qwen2-VL, etc.) or an
  external API, switched by `VLM_BACKEND=ollama|openai|anthropic`
- **stub** (`VLM_BACKEND=stub`): test mode that returns deterministic
  dummy strings

### Face blur

- **MVP**: OpenCV Haar cascade face detection + Gaussian blur, dedup on
  the resulting bytes' sha256 so re-runs are free
- **Future**: license plates (OCR-based), screens (sharp-rectangle
  detection), ID badges, etc., as plug-ins behind a `RedactionPipeline`
  interface

In the MVP, the VLM and the face blur take only the image (`image_url`) as input
(frame extraction + per-frame blur is future work). When both an image and a video
are present, the derivatives are made from the image. With a video only, neither the
VLM nor the blur runs: Tier 3 sees the video as before, and Tier 2 gets the event only.

### Redaction-leak detection (research TODO)

VLMs structurally cannot guarantee PII removal from `description_summary`.
This spec **separates detection into a post-check stage** and records it
as a research TODO; the MVP does not ship it.

Sketch:

1. NER-extract proper nouns (people / organisations / places) from `description_full`
2. Diff against `description_summary` for any leftover matches
3. If any survive, cross-check against a known-PII / per-country name
   dictionary and raise `processing_warnings: ["redaction_leak_suspected"]`
   when the confidence is high
4. Flagged rows queue for human review via the audit path

### Audit-log impact

`description_model` and `description_generated_at` are attached to
each row of `/platform/data` (passed to receivers at every tier), not to
`/audit/logs`. The audit row (`AuditLogRecord` in `audit/models.py`) has
no such fields. To trace **which VLM output reached whom under which
tier at which time**, correlate `/audit/logs` rows with `/platform/data`
rows. If the VLM is upgraded, the same source image can produce
different descriptions; `description_generated_at` separates them.

### Test plan (deltas)

A new file `tests/test_data_user_vc_tiered_vlm.py` covers (case counts at the initial implementation in parentheses):

1. VLM extension off: the existing tier projection does not regress (5)
2. stub VLM backend produces the expected `description_*` shape (3)
3. Per-tier `/platform/data` projection matrix (4)
   - Tier 3: raw + derivative keys
   - Tier 2: image_redacted + description_full + description_summary; no raw image/video
   - Tier 1: description_summary only
   - Tier 0 (denied): events only (the original design rejected the claim with 403; the current implementation does not)
4. Degrade matrix when VLM call fails (2)
5. `processing_warnings` propagation (2)

At the initial implementation this added 16 cases, kept independent from the existing 88 (this file now has 29 tests). The most important check is that nothing changes when the VLM extension is off.

### Future work (recorded for the spec)

- Video → frame extraction → VLM aggregate for Tier 2/1 video derivatives
- VLM **reproducibility**: either restrict to backends that produce the
  same description for the same `(image, prompt)` tuple, or accept
  divergence with `generated_at` separation
- The redaction-leak post-check above
- A **VLM cache layer**: reuse descriptions across uploads keyed by
  source image sha256 (mirrors the existing image dedup)
- VLM **cost gating**: throttle calls per DataUserVC tier
