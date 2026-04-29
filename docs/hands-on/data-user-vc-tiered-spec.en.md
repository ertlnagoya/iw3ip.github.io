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

```
entityType:
  GovernmentOrganization | Police       -> 35
  Enterprise                            -> 20
  ResearchOrganization                  -> 15
  other                                 -> 0

purpose:
  CrimeSearch                           -> 25
  TrafficManagement                     -> 20
  Research                              -> 15
  other                                 -> 0

legalCompliance == true                 -> +15
dataHandlingPolicy == "ISO27001"        -> +15
misuseRecord == false                   -> +10
misuseRecord == true                    -> -10
```

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
