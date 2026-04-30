# DataUserVC × Tiered Access Spec

A design document defining how the Publisher narrows the camera view it
exposes — to **event / image / video** in three tiers — based on the
trust attributes carried by the data recipient's **DataUserVC**.

This is a design document, not a step-by-step. For the walk-through, see
[DataUserVC × Tiered Access](data-user-vc-tiered.md).

## What you'll learn

- How the new **DataUserVC** relates to the existing five VCs
  (ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC)
- How the trustScore logic from Li-lab's `DataUserVerifier.sol` is folded
  into the Phase 2 Publisher
- The unidirectional flow: VC verification → trustScore → allowed_views
- Separation of concerns between data sources (HA / RaspberryPi / USB
  Webcam) and the Publisher

## Common pitfalls

- Evaluating "can this DataUserVC see the camera" **on-chain** breaks
  whenever the local Hardhat node restarts. We evaluate in **Publisher
  Python** instead.
- trustScore is **not** a simple sum: `full` requires
  `score >= 80 AND entityType ∈ {GovernmentOrganization, Police}`.
- `allowed_views` on ViewerVC / PurchaseViewerVC is **frozen at token
  mint time**; we never promote it after the fact.

## Goal

On top of the existing Phase 2 flow (VC presentation → verification →
token mint → `/platform/data`), we tier **what** the holder can fetch by
their DataUserVC attributes:

- entityType (GovernmentOrganization / Police / Enterprise / ResearchOrganization)
- purpose (CrimeSearch / TrafficManagement / Research)
- legalCompliance (true / false)
- dataHandlingPolicy (ISO27001 / other)
- misuseRecord (history of misuse)

We score these with the same weights as Li-lab's `DataUserVerifier.sol`,
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
[Wallet (holds DataUserVC)]
        | OID4VP (DCQL: DataUserVC)
        v
[Publisher /verifier/*]
        | post-PEX checks
        v
[trust_score.evaluate(entityType, purpose, legalCompliance,
                      dataHandlingPolicy, misuseRecord)]
        | -> trust_score: int, access_level: full/access/denied,
        |    allowed_views: subset of {event, image, video}
        v
[/marketplace/claim] -> MarketplaceClaim.allowed_views
        v
[PurchaseViewerVC issuance] -> VC.claims.allowed_views
        v
[/verifier/* with PurchaseViewerVC presentation]
        v
[ViewerToken.allowed_views]
        v
[/platform/data] selects image_cid / video_cid / video_duration_sec
```

`DataUserVC` alone mints **no token**. It is the *credential material*
for issuing ViewerVC / PurchaseViewerVC.

## Relation to the five existing VCs

| VC | Role | Token | Carries allowed_views |
|---|---|---|---|
| ConsentVC | Provider consent | PolicyToken (5 min, single) | no |
| ViewerVC | Restricted viewer | ViewerToken (60 s, multi) | yes |
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
"denied" -> []   # claim itself is rejected
```

## API surface (deltas only)

### `POST /issuer/offer` (extended)

```
?vc_kind=DataUserVC
&entity_type=GovernmentOrganization
&purpose=CrimeSearch
&legal_compliance=true
&data_handling_policy=ISO27001
&misuse_record=false
```

For `DataUserVC`, all five attributes are required; missing → 400.

### `POST /verifier/request_object`

Accepts `vc_kind=DataUserVC` and asks DCQL for the five claims.

### `POST /verifier/*` (presentation endpoints)

When a `DataUserVC` is presented:
- post-PEX checks the five claims are present
- **no token is minted**
- response: `{trust_score, access_level, allowed_views, holder_did}`

### `POST /marketplace/claim` (extended)

```json
{
  "merchandise_id": "...",
  "buyer_did": "...",
  "data_user_attrs": {
    "entityType": "GovernmentOrganization",
    "purpose": "CrimeSearch",
    "legalCompliance": true,
    "dataHandlingPolicy": "ISO27001",
    "misuseRecord": false
  }
}
```

If `data_user_attrs` is omitted, we claim with the minimum
`allowed_views=["event"]`.

### `GET /platform/data`

Filtered by `ViewerToken.allowed_views`:

| allowed_views contains | Output fields |
|---|---|
| `event` | the event JSON itself |
| `image` | adds `image_cid` |
| `video` | adds `video_cid`, `video_duration_sec` |

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

`tests/test_data_user_vc_tiered.py` covers 13 cases:

1. `evaluate()` pure-function tests (4)
   - GovernmentOrganization × Research × ISO27001 → full
   - Enterprise × Research × ISO27001 → access
   - low score → denied
   - even with score ≥ 80, non-gov/police never reaches full
2. `evaluate_from_claims()` wrapper (1)
3. Issuer metadata lists `DataUserVC` (1)
4. `/issuer/offer?vc_kind=DataUserVC` requires the five attributes (1)
5. `/marketplace/claim` trust promotion / default to event-only (2)
6. `PurchaseViewerVC` inherits `allowed_views` from the claim (1)
7. `/platform/data` projects Tier 1 / 2 / 3 (3)

Total: 88 = 75 existing + 13 new. No regressions.

## Future work

- Bundle DataUserVC into a Verifiable Presentation
- Re-deploy `DataUserVerifier.sol` on a low-fee chain (e.g. Polygon
  zkEVM) and demote `trust_score.py` to a fallback
- Make `allowed_views` time-window aware (e.g. video only at night)

## Tier extension: semantic-level redaction (VLM)

§1–§9 above gate access by **dropping media keys** (Tier 2 hides video,
Tier 1 hides image and video). The next step is to **derive new content
from the same source via VLM inference + face/PII blurring and project
those derivatives per tier**. Tier 1 is no longer "you get nothing
useful"; it now ships a **PII-redacted summary** so a low-trust
recipient can know *what happened* without learning *who did it*.

### Updated tier definitions

| Tier | access level | Example (score) | Derivatives shipped |
|---|---|---|---|
| **3** Full | `full` | gov + crime + ISO27001 (80) | raw image / video + every text derivative |
| **2** Access | `access` | enterprise + research + ISO27001 (75) | **face/PII-blurred image** + **detailed text** (named entities present) |
| **1** Summary | `summary` (new) | enterprise + research only (60–) | **summary text only** (PII-redacted, no image) |
| 0 Denied | `denied` | unqualified (<60) | claim is rejected |

A new `summary` value is added to `access_level`. The `denied`
semantics are unchanged — `score < 60` still rejects the claim.

### Derivative schema

`/platform/data` row objects gain these keys:

| Key | Content | Visible at tier |
|---|---|---|
| `image_url_redacted` | URL of an image with faces / people / license plates blurred | 2 + 3 |
| `image_cid_redacted` | IPFS CID of the redacted image (when option C is on) | 2 + 3 |
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

When the publisher is started with `--profile vlm`, `/provider/publish`
gains two new steps:

```
provider page
  ├── /media/upload stores the source image (existing)
  └── POST /provider/publish
        └── pipeline.process_message
              ├── (new) VLMClient.describe(image_url)
              │     -> {description_full, description_summary, model, generated_at}
              ├── (new) ImageRedactor.blur_pii(image_url)
              │     -> {image_url_redacted, image_cid_redacted}
              ├── existing: consent / policy evaluation
              └── existing: platform_client.send(envelope)
```

Degrade policy when VLM is missing or fails:

| Condition | Publisher behaviour |
|---|---|
| `--profile vlm` off | derivative keys are **not generated** (legacy behaviour — Tier 1/2 fall back to the old "key drop" model) |
| profile on, VLM call failed | description keys omitted; `image_url_redacted` is still attempted (face blur doesn't need the VLM) |
| profile on, blur failed | redacted-image keys omitted; descriptions still ship. Tier 2 receivers get text without a redacted image |
| profile on, both succeed | all keys generated |

Degraded responses carry **`processing_warnings: ["vlm_unavailable", ...]`**
so the receiver can tell what was skipped.

### VLM backends

- **MVP**: local LLaVA via Ollama as a compose service (`ollama run llava`)
- **Future**: swap for a larger multimodal LLM (Qwen2-VL, etc.) or an
  external API, switched by `VLM_BACKEND=ollama|openai|anthropic`
- **stub** (`VLM_BACKEND=stub`): test mode that returns deterministic
  dummy strings

### Face / PII blur

- **MVP**: OpenCV Haar cascade face detection + Gaussian blur, dedup on
  the resulting bytes' sha256 so re-runs are free
- **Future**: license plates (OCR-based), screens (sharp-rectangle
  detection), ID badges, etc., as plug-ins behind a `RedactionPipeline`
  interface

If the source content_type is video, the MVP **does not produce a
redacted image** (frame extraction + per-frame blur is future work).
Tier 3 sees the video as before; Tier 2 just gets the text derivatives.

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

`audit/logs` now records `description_model` and `description_generated_at`
so we can replay **which VLM output reached whom under which tier at
which time**. If the VLM is upgraded, the same source image can produce
different descriptions; `generated_at` separates them.

### Test plan (deltas)

A new file `tests/test_data_user_vc_tiered_vlm.py` covers:

1. `--profile vlm` off: existing tier projection regresses to nothing (5)
2. stub VLM backend produces the expected `description_*` shape (3)
3. Per-tier `/platform/data` projection matrix (4)
   - Tier 3: raw + derivative keys
   - Tier 2: image_redacted + description_full + description_summary; no raw image/video
   - Tier 1: description_summary only
   - Tier 0 (denied): claim itself returns 403
4. Degrade matrix when VLM call fails (2)
5. `processing_warnings` propagation (2)

Total +16 cases, kept independent from the existing 88 (the
profile-off regression test is the most important).

### Future work (recorded for the spec)

- Video → frame extraction → VLM aggregate for Tier 2/1 video derivatives
- VLM **reproducibility**: either restrict to backends that produce the
  same description for the same `(image, prompt)` tuple, or accept
  divergence with `generated_at` separation
- The redaction-leak post-check above
- A **VLM cache layer**: reuse descriptions across uploads keyed by
  source image sha256 (mirrors the existing image dedup)
- VLM **cost gating**: throttle calls per DataUserVC tier
