# Tier by semantic level (VLM and semantic representation / Stage T)

This page continues [Tier the response by trust](data-user-vc-tiered.md) (§0–§7) and [Deliver images and video with tiered access](data-user-vc-tiered-media.md) (§8–§11). It goes all the way to semantic intermediate representation and trust-aware rendering. Section numbers continue from the first page (§12–§13).

> **What you'll do**: Derive new content with VLM inference and face/PII blurring and project it per tier, then run trust-aware rendering on the semantic intermediate representation (SIR)
>
> **Prerequisites**: §0–§7 of [Tier the response by trust](data-user-vc-tiered.md) and §8–§11 of [Deliver images and video with tiered access](data-user-vc-tiered-media.md)
>
> **What you need**: PC + smartphone (iw3ip-wallet), and a publisher started with `--profile vlm` (Ollama)

## 12. Semantic-level redaction (VLM + face blur)

§1–§11 (the [first page](data-user-vc-tiered.md) and the [second page](data-user-vc-tiered-media.md)) gate access by **dropping media keys** — Tier 2 hides video,
Tier 1 hides image and video. §12 derives **new content from the same
source via VLM inference + face/PII blurring** and projects those
derivatives per tier. Tier 1 stops being "you get nothing useful" and
ships a **PII-redacted summary** instead.

Design rationale: see [DataUserVC × Tiered Access Spec § "Tier extension"](data-user-vc-tiered-spec.md#tier-vlm).

### 12.1 Updated tier definitions

| Tier | access level | Example (score) | Derivatives shipped |
|---|---|---|---|
| **3** Full | `full` | gov + crime + ISO27001 (80) | raw image / video + redacted image + full text + summary text |
| **2** Access | `access` | enterprise + research + ISO27001 (75) | **face/PII-blurred image** + **detailed text** (named entities) + summary |
| **1** Summary | `summary` (new) | enterprise + unknown purpose + legalCompliance only (50–59) | **summary text only** (PII-redacted, no image) |
| 0 Denied | `denied` | unqualified (<50) | claim is rejected |

A new `summary` value joins the `access_level` enum: with the VLM
profile on, scores 50–59 map to `summary` (with the profile off,
scores below 60 are still `denied`). The `/platform/data` row schema gains:

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
  publisher bridge mosquitto vlm vlm-pull
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

The [§2](data-user-vc-tiered.md#2-mint-three-datauservc-offers) set extended with a **summary-tier** profile.

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

Same routine as [§3](data-user-vc-tiered.md#3-three-marketplaceclaim-calls): claim → PurchaseViewerVC → present → ViewerToken → fetch.
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
| ![pre-redaction](images/data-user-vc-tiered/vlm/V4-original.jpg){ width="300" } | ![post-redaction](images/data-user-vc-tiered/vlm/V4-redacted.jpg){ width="300" } |

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
    [future work in the spec](data-user-vc-tiered-spec.md#tier-vlm).

### 12.6 Inspecting VLM output

Compare `description_full` (Tier 2/3) against `description_summary`
(Tier 1+) to confirm proper-noun stripping:

```bash
curl -s -H "authorization: Bearer $VIEWER_TOKEN_TIER3" \
     'localhost:8080/platform/data?dataset_id=home/event/possible_littering' \
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
    [records detection as a research TODO](data-user-vc-tiered-spec.md#tier-vlm)
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

Images uploaded through `/provider` ([§11](data-user-vc-tiered-media.md#11-pwa-provider-the-data-provider-side)) automatically flow through
the VLM pipeline when profile vlm is on. The **provider page itself
needs no change**; derivative generation is server-side. The
receiver-side `/viewer` shows **`image_url_redacted` with a 🔒 badge**
when raw keys are dropped, renders `description_full` /
`description_summary` as parallel green / orange panels, and shows an
inline warning banner per row when `processing_warnings[]` is set
(shipped in
[Blockchain_IoT_Marketplace#48](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/48)).

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
| ![original](images/data-user-vc-tiered/vlm/V4-original.jpg){ width="300" } | ![redacted](images/data-user-vc-tiered/vlm/V4-redacted.jpg){ width="300" } |
| Face details (eyes, nose, mouth) clearly visible | Central rectangular face region completely Gaussian-blurred; hair, ears, beard, and background unchanged |

The image is a **StyleGAN2 synthetic face** — no real-world PII involved.

The implementation is verified at unit-test level (155/155 pass) in
[Blockchain_IoT_Marketplace#44](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/44),
with the timeout fix in
[#45](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/45).

## 13. Semantic intermediate representation + trust-aware rendering

§12 stops at "drop or blur image keys per access_level". §13 takes
the next step: replace the implicit *frame -> VLM -> output* path with
an explicit four-layer pipeline:

  **frame -> SIR -> trust policy -> trust-aware renderer -> output**

Privacy-sensitive regions (faces, text, screens, whiteboards,
documents, name tags, ID cards, license plates, plus
`unknown_sensitive`) are structured as bounding boxes in the SIR.
The trust policy maps a `ViewerTrustLevel` × SIR pair to allowed
output kinds, and the renderer is the only place that ever combines
the source bytes with the policy decision. Low-trust receivers
cannot reach raw bytes by construction (fail-closed by design).

### 13.1 Architecture

```
[iPhone Safari /provider]   uploads frames as in §1-§2
       |  POST /media/upload
       v
[publisher SemanticAnalyzer]   /semantic/analyze
       |  ├─ MockSemanticAnalyzer   (test / MVP)
       |  ├─ VisionSemanticAnalyzer (OpenCV Haar + MSER)
       |  └─ Apple Vision / Core ML / VLM   (future)
       v
[Semantic Intermediate Representation (SIR) JSON]
       |  normalized bbox + sensitive_regions[] + events[] +
       |  privacy_risk_score + analyzer_version
       v
[TrustPolicyEngine]            /semantic/render or /semantic/render_url
       |  ViewerTrustLevel ∈ {anonymous,low,medium,high,owner,admin}
       |  -> DisclosurePolicy (allowed_outputs + mask_regions + audit_required)
       v
[TrustAwareRenderer]
       |  textSummary / eventList / redactedImage / lowResolutionImage /
       |  maskedVideoFrame / originalFrame
       v
[Viewer Output]
```

### 13.2 Trust × output matrix

| Level | textSummary | eventList | redactedImage | lowResolutionImage | maskedVideoFrame | originalFrame |
|---|---|---|---|---|---|---|
| `anonymous` | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ |
| `low` | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| `medium` | ✓ | ✓ | **✓** | **✓** | ✗ | ✗ |
| `high` | ✓ | ✓ | ✓ | ✓ | **✓** | ✗ |
| `owner` | ✓ | ✓ | ✓ | ✓ | ✓ | **✓** (audit) |
| `admin` | ✓ | ✓ | ✓ | ✓ | ✓ | **✓** (audit) |

`unknown_sensitive` is **always masked at HIGH and below** unless
`SEMANTIC_ALLOW_UNKNOWN_AT_HIGH=true` is explicitly set.

### 13.3 Bring up

Add `SEMANTIC_ANALYZER_BACKEND` to the existing VLM-tier compose run:

```bash
SEMANTIC_ANALYZER_BACKEND=vision \
  VLM_BACKEND=ollama VLM_MODEL=moondream IMAGE_REDACTION_BACKEND=opencv \
  docker compose -f infra/docker-compose.yml --profile vlm up -d publisher
```

| Env var | Value | Effect |
|---|---|---|
| `SEMANTIC_ANALYZER_BACKEND` | empty / `stub` | MockSemanticAnalyzer (deterministic) |
| `SEMANTIC_ANALYZER_BACKEND` | `vision` | OpenCV Haar cascade + MSER |
| `SEMANTIC_ALLOW_UNKNOWN_AT_HIGH` | `true` | Let HIGH viewers see unknown_sensitive |

### 13.4 /provider §1.5 "Run analysis" panel

After upload completes, the §1.5 panel becomes available on the
`/provider` page ("§1.5" is a section number inside that page, not a
section of this document). Pressing
"分析を実行":

1. POSTs the source blob to `/semantic/analyze` to retrieve the SIR
2. Overlays detected `sensitive_regions` as red bounding boxes on the uploaded image
3. Calls `/semantic/render` four times (anonymous / low / medium / high) and shows a card per tier (`granted_kinds` + `text_summary` + masked image inline when permitted)

This is an operator-facing **dry run**: see exactly what each tier
will receive before pressing Publish. Catches Haar misses or
unknown_sensitive over-reach early.

### 13.5 /viewer 🔬 toggle

The receiver-side `/viewer` gains an opt-in **🔬 「意味的レンダリングを使う (実験)」**
checkbox. Off (default) preserves the legacy projection byte-for-byte.
On switches to the semantic pipeline:

- The page derives a trust level from the existing `allowed_views`:
  `video → high`, `image|image_redacted → medium`,
  `description_* → low`, `event` only / unknown → `anonymous`
  (fail-closed)
- Each row calls `POST /semantic/render_url` (one round trip to
  fetch + analyze + render) and renders the result
- Toggle state persists in `localStorage` so a refresh keeps the
  selection

### 13.6 Fail-closed sites

- `_coerce_trust_level()`: unknown / None → ANONYMOUS
- `TrustPolicyEngine.evaluate()`: `privacy_risk_score >= 0.9` strips image kinds
- `TrustPolicyEngine.evaluate_safe()`: any internal exception → empty `allowed_outputs`
- `_compute_mask_plan()`: face / text / screen / whiteboard / document / id_card / name_tag / `unknown_sensitive` always masked at HIGH and below
- `TrustAwareRenderer.render()`: `ORIGINAL_FRAME` requires `trust_level in (OWNER, ADMIN)`
- `_render_masked()`: cv2 / decode / encode failures → text fallback
- `SemanticIntermediateRepresentation.empty()`: `privacy_risk_score=1.0` flags the analyzer-failure path
- `/semantic/render_url`: URL fetch failure → empty SIR + text-only

### 13.7 Server-side audit log

`/semantic/render` and `/semantic/render_url` write an audit row when
either condition fires:

- The policy returned `audit_required=True` (OWNER / ADMIN tier)
- The renderer actually produced image bytes (any tier)

Fields recorded: `ts`, `action`, `subject_did` (sir.source_device_id),
`purpose=semantic_render`, `reason=trust=...;kinds=...;image=yes|no;
audit_required=...;rationale=...`, `message_hash` (sir.frame_id),
`presentation_verified` (`owner_or_admin` or `policy_only`).

We never log the source bytes, the base64-encoded rendered image,
or any face encoding / extracted PII from the SIR.

### 13.8 API reference

| Endpoint | Input | Output | Notes |
|---|---|---|---|
| `POST /semantic/analyze` | multipart `file` + `source_device_id` | SIR JSON | Never echoes input bytes; analyzer error → `empty()` SIR |
| `POST /semantic/render` | `{trust_level, sir, image_url?}` | `{trust_level, granted_kinds[], text_summary, event_list, image_b64?, ...}` | Unknown trust_level → ANONYMOUS |
| `POST /semantic/render_url` | `{trust_level, image_url}` | same | One-shot fetch + analyze + render |

### 13.9 Relation to §12 VLM tier

| Aspect | §12 VLM tier | §13 semantic pipeline |
|---|---|---|
| Trust derivation | DataUserVC `allowed_views` baked at claim time | derived at runtime from `allowed_views`; legacy default kept |
| Derivative shape | text + face-blurred image | structured SIR (bbox + sensitive_regions + events) |
| Mask granularity | face only (`image_url_redacted`) | face + text + screen + whiteboard + document + id_card + name_tag + plate + unknown_sensitive |
| Receiver toggle | fixed at claim mint | runtime via `/viewer` 🔬 toggle |
| Failure mode | `processing_warnings: ["vlm_unavailable"]` | `empty()` SIR + text-only fallback |
| Audit | per-component (image_redactor / vlm_client) | unified hook in `/semantic/render*` |

The two coexist. The 🔬 toggle leaves the legacy Stage T projection
intact; it merely offers an opt-in alternative path.

### 13.10 Real-device validation log

| Scenario | Environment | Status | Verification point |
|---|---|---|---|
| **S1** /provider §1.5 panel | iPhone Safari (real photo) | ✅ **verified (2026-04-30)** | Photo captured on iPhone → uploaded to /provider → §1.5 "分析を実行" → analyzer=`vision-opencv` detected `face` (conf 0.85) + `unknown_sensitive` (0.50). Red bbox overlay shown on image, `privacy_risk_score=0.90`, 4-tier preview cards (anonymous / low / medium / high) all rendered correctly.<br>📷 [S1 screenshot](images/data-user-vc-tiered/semantic/S1-provider-analyze.png) |
| **S2** /viewer 🔬 toggle | iPhone Safari + Tier 3 PVC | ✅ **verified (2026-04-30)** | Tier 3 PVC (`PurchaseViewerVC.full`, tier=`event+image+video+image_redacted+description_full+...`) issued and presented on iPhone Safari /viewer. Toggling 🔬 on a single row: **OFF** → legacy display with raw photo + `vlm_unavailable` warning. **ON** → trust-aware via `/semantic/render_url`: `derived trust level: high`, `[high] kinds: textSummary + eventList`, "1人の人物 + 2件のセンシティブ領域" / "person_detected" text only, raw photo suppressed.<br>📷 [toggle OFF](images/data-user-vc-tiered/semantic/S2-viewer-toggle-off.png) / [toggle ON](images/data-user-vc-tiered/semantic/S2-viewer-toggle-on.png) |
| **S3** OWNER audit log | publisher only (curl) | ✅ **verified (2026-04-30)** | OWNER trust on `/semantic/render_url` returned image_b64 + every kind, and `/audit/logs` got a `semantic/render_url` row (`trust=owner;kinds=textSummary+redactedImage+eventList+originalFrame+maskedVideoFrame+lowResolutionImage;image=yes;audit_required=True`) |
| **S4** Vision analyzer face detection | publisher (`SEMANTIC_ANALYZER_BACKEND=vision`) | ✅ **verified (2026-04-30)** | Synthetic face image → `/semantic/analyze` returns SIR with `face` (conf 0.85, bbox x=0.14, y=0.19, w=0.70, h=0.70) + `unknown_sensitive` (from MSER). privacy_risk_score=0.9 (face 0.4 + unknown 0.5) |
| **S5** cv2-missing fail-closed | publisher (`sys.modules['cv2']=None` simulated) | ✅ **verified (2026-04-30)** | With cv2 / numpy stubbed out, `VisionSemanticAnalyzer.analyze()` raises `SemanticAnalyzerError("vision backend requires cv2 + numpy")`. Route demotes to `SemanticIntermediateRepresentation.empty()` (objects=0, sensitive=0, events=0, **privacy_risk_score=1.0**). For trust=anonymous/low/medium/**high** the renderer drops every image kind (`high_risk_score(1.00)>=ceiling(0.9); image kinds dropped`) and returns `textSummary + eventList` only. OWNER alone retains image_bytes via the `audit_required` path (expected). |

Implementation:
[Blockchain_IoT_Marketplace#49](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/49)
(pipeline) +
[#50](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/50)
(/provider §1.5) +
[#51](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/51)
(/viewer 🔬) +
[#52](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/52)
(audit hook). Unit tests at 190/190 pass.

## Where to go next

- Back: [Deliver images and video with tiered access](data-user-vc-tiered-media.md) — §8–§11 (delivering images and video)
- Start: [Tier the response by trust](data-user-vc-tiered.md) — §0–§7 (issuing DataUserVC and tiering the response)
