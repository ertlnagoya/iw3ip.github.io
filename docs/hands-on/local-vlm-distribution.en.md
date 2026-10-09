# Enrich camera data with a local VLM

Analyze frames from a laptop PC and a USB webcam with a **local VLM (vision-language model)** running on the PC, attach semantic text to them, and distribute that AI-enriched data over the IoT data-distribution platform. The key point is that the images never leave for an external cloud — the analysis is done entirely locally.

> **What you'll do**: analyze a webcam image with a local VLM and distribute the generated semantic data (descriptions) over the platform
>
> **Prerequisites**: [Environment setup](../setup/index.en.md) and [Quickstart](../setup/quickstart.en.md) completed
>
> **What you need**: laptop PC + USB webcam + Docker (including the VLM)
>
> **Time**: approx. 40–60 min (the first model pull takes a while)

!!! note "Where this page fits"
    This is a Part 3 (intelligence integration) hands-on. Camera capture basics
    are covered in [USB webcam sample](webcam.md), and per-tier projection with a
    VLM is in §12 of [DataUserVC × tiered access](data-user-vc-tiered.md). This
    page focuses on experiencing "**camera → local VLM → distribute semantic
    data**" as a single flow.

## Goal

Instead of distributing raw images, distribute the **semantic derivatives** a local VLM produces. You generate a detailed description (`description_full`) and a privacy-scrubbed summary (`description_summary`), and experience a design that preserves privacy and data sovereignty by running the AI locally.

## What this page covers

- how to embed a local VLM (Ollama) into the platform and generate semantic data from an image
- the idea of distributing "semantic derivatives" rather than raw images
- separating a PII-bearing detailed description from a redacted summary
- that the model used and any degraded steps (`processing_warnings`) are auditable

## Overview

```
┌───────────────┐   snapshot.jpg   ┌──────────────────────────┐
│ USB webcam     │ ───────────────▶ │ publisher (--profile vlm) │
│ + laptop       │                  │  └─ Ollama (llava/moondream)│
└───────────────┘                  └───────────┬──────────────┘
                                               │ description_full / summary
                                               ▼
                                     ┌────────────────────┐
                                     │ received via         │
                                     │ /platform/data       │
                                     └────────────────────┘
```

## Prerequisites

- you can start the v1 stack from [Quickstart](../setup/quickstart.en.md)
- a USB webcam is recognized by the PC (trying the [USB webcam sample](webcam.md) first makes this easier)
- Docker can use the `--profile vlm` profile (the first model pull downloads several GB)

## Shortest path

For a first pass, these four steps are enough.

1. start the services with `--profile vlm` and wait for the model pull
2. capture a single still image from the webcam
3. submit that image to the platform and let the local VLM analyze it
4. confirm the semantic data (`description_*`) is distributed via `/platform/data`

## 1. Start the services with the VLM

The VLM is opt-in. Adding `--profile vlm` also starts the Ollama service and `vlm-pull`, which pre-fetches the model.

```bash
cd ~/program/Blockchain_IoT_Marketplace
export VLM_BACKEND=ollama
# For a CPU-only laptop, the lightweight moondream is recommended.
# Use llava (the default) if you want higher quality and have time.
export VLM_MODEL=moondream
export IMAGE_REDACTION_BACKEND=opencv
docker compose -f infra/docker-compose.yml --profile vlm up -d \
  publisher bridge mosquitto vlm vlm-pull
```

Wait until `vlm-pull` finishes fetching the model (first time only, several GB).

```bash
docker compose -f infra/docker-compose.yml logs -f vlm-pull
# exit the logs once the pull is complete
```

Confirm the publisher is up.

```bash
curl -s http://localhost:8080/health | jq .
# -> {"status": "ok", ...}
```

!!! tip "Choosing a model"
    `llava` gives richer descriptions but takes several minutes per image on CPU
    only. `moondream` is small (about 1.7 GB) and responds in tens of seconds to a
    few minutes even on CPU. Get the flow working with `moondream` first, then
    switch to `VLM_MODEL=llava` to compare quality. On slow machines, raise
    `VLM_TIMEOUT_SEC` (default 180 s).

## 2. Capture one frame from the webcam

Grab a single still image from the USB webcam and save it as `snapshot.jpg`. The script uses OpenCV; install it with `pip install opencv-python` if needed.

```python
# capture_snapshot.py
import cv2

cap = cv2.VideoCapture(0)          # if you have multiple cameras, try 1, 2, ...
ok, frame = cap.read()
cap.release()
if not ok:
    raise SystemExit("Could not capture from the camera (check index / usage)")
cv2.imwrite("snapshot.jpg", frame)
print("saved snapshot.jpg")
```

```bash
python capture_snapshot.py
```

!!! note "No camera / cannot open it"
    You can substitute any JPEG as `snapshot.jpg`. If capture fails due to camera
    index or usage, see the `mock` mode and troubleshooting in the
    [USB webcam sample](webcam.md).

## 3. Get the "meaning" from the local model

POST the captured image to the publisher's `/semantic/analyze`. The local model analyzes it and returns only a structured **Semantic Intermediate Representation (SIR)** — **never the raw pixels**. No wallet or VC is required, so you can try it immediately.

```bash
curl -s -X POST http://localhost:8080/semantic/analyze \
  -F "file=@snapshot.jpg" \
  -F "source_device_id=laptop-webcam-01" | jq .
```

An example SIR (values depend on the image and analyzer):

```json
{
  "people": {"count": 1, "identities": []},
  "objects": [ ... ],
  "sensitive_regions": [ {"type": "face"} ],
  "scene_summary": "...",
  "privacy_risk_score": 0.4,
  "analyzer_version": "..."
}
```

!!! note "Choosing the analyzer"
    The default is a deterministic stub. For real OpenCV face detection, set
    `SEMANTIC_ANALYZER_BACKEND=vision` when starting the publisher. Combined with
    `--profile vlm`, the distribution step below also attaches VLM descriptions
    (`description_*`).

## 4. Distribute the semantic data over the platform

Submit the image to the publisher and publish an event that points at it, so the AI-derived data enters the distribution pipeline. The exercise program in the next section runs consent registration → `/media/upload` → `/simulate/publish` in one go.

If you started with `--profile vlm`, the publisher also calls the VLM here and attaches `description_full` / `description_summary` / `description_model` / `processing_warnings` to the row.

!!! info "Consumer-side retrieval is tier-gated"
    To read the attached `description_*` on the consumer side you need a
    ViewerToken obtained by presenting a DataUserVC (`/platform/data` returns 401
    for a plain GET). Projecting `description_full` / `description_summary` per
    trust tier is covered in §12 of [DataUserVC × tiered access](data-user-vc-tiered.md).
    The core of this page is "camera → local model semantic enrichment →
    distribute over the platform".

## 5. Verification points

- `/semantic/analyze` returns structured semantic data (SIR) only, never the raw image
- the meaning is produced with the local model only, without sending images to a cloud
- when `--profile vlm` is used, the distributed row carries `description_*` (descriptions and model ID)

## Something to think about

- When you distribute "semantic derivatives" instead of the raw image, what changes from the standpoint of privacy and data sovereignty?
- Organize the benefits and limits (speed, accuracy, operations) of running locally instead of using a cloud API.

## Work through the exercise

Exercise programs are provided so first-time students can progress step by step: **read the overall structure (Step A) → fill in the template (Step B) → add a feature from scratch (Step C)**.

Exercise programs:

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/local_vlm_distribution/problem_program.py) (with TODOs)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/local_vlm_distribution/answer_program.py)
- [Exercise guide (with a structure walkthrough)](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/local_vlm_distribution/README.md)

### Step A. Read the structure and the program

From the README "Big picture", grasp the three functions: `analyze_frame()` (POST to `/semantic/analyze` to get the SIR), `summarize_sir()` (one-line summary of the SIR), and `distribute_frame()` (register consent → upload → publish). The point is to understand the "keep the image in, let the meaning out" flow.

### Step B. Fill in the two TODOs (task)

Implement the two TODOs in `problem_program.py`.

- **TODO 1 `summarize_sir()`**: read people count, object/sensitive-region counts, and the scene summary from the SIR and return a one-line summary
- **TODO 2 `build_event()`**: build the event payload that carries the uploaded image (`image_url`) into the platform

```bash
python examples/hands_on/local_vlm_distribution/problem_program.py \
  --base-url http://localhost:8080 --image snapshot.jpg
```

Compare with `answer_program.py`. Add `--distribute` to run the distribution too.

### Step C. Add a feature from scratch (advanced)

Without a template, implement one of these yourself:

- **Richer summary**: list each `sensitive_regions[].type` and warn when `privacy_risk_score` is high
- **Frame sampling**: capture N frames on a timer and distribute only the one with the highest privacy risk
- **New derived field**: add a value computed from the SIR (e.g. `people_count`) to the event `data`
- **Custom analyzer**: implement a new `SemanticAnalyzer` in `publisher/app/semantic_analyzer.py` and switch to it with `SEMANTIC_ANALYZER_BACKEND`

## Going further (extension directions)

- **Natural-language search and processing**: run natural-language search and summarization over the accumulated semantic data with an LLM ([LLM Planner](llm-planner.md) / [Regional safety assistant](regional-safety-assistant.md)).
- **Remote device control**: based on the analysis result, separate `plan` and `execute` to operate a device (same Part 3).
- **Raspberry Pi + Pi camera**: split the roles so the Pi captures and the PC/host runs inference (this page keeps everything on the laptop first).

## Common issues

- the first model pull is large and slow → wait for completion in the `vlm-pull` logs; do it when you have bandwidth to spare
- inference is slow / times out on CPU → use the lightweight `VLM_MODEL=moondream`, or raise `VLM_TIMEOUT_SEC`
- `processing_warnings` shows `vlm_unavailable` → check that the `vlm` service is up and the model pull is complete (`docker compose ... ps` / `logs`)
- the camera cannot be opened → change the camera index or stop the process holding it (see [USB webcam sample](webcam.md))

## Related pages

- Prerequisites: [USB webcam sample](webcam.md) / [Quickstart](../setup/quickstart.en.md)
- Deep dive: [DataUserVC × tiered access (§8 media integration, §12 VLM tier)](data-user-vc-tiered.md) / [LLM Planner](llm-planner.md)
- Design: [DataUserVC × tiered access control spec (tier extension: semantic-level redaction)](data-user-vc-tiered-spec.md)
