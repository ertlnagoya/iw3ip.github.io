# Enrich camera data with a local VLM and distribute it (Part 3)

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
  publisher hardhat bridge mosquitto vlm vlm-pull
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

Grab a single still image from the USB webcam and save it as `snapshot.jpg`.

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

## 3. Submit the image and let the local VLM analyze it

Submit the captured image to the publisher's media gateway. With the VLM profile enabled, the publisher calls Ollama to generate the descriptions.

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://localhost:8080 \
  --image snapshot.jpg
```

The publisher generates two kinds of description from the ingested image with the local VLM (tens of seconds to a few minutes per image on CPU).

- `description_full` … a detailed description including names, objects, and text
- `description_summary` … a summary with personal information removed (PII-redacted)

## 4. Confirm the semantic data is distributed

Fetch `/platform/data` and confirm the VLM-generated semantic data is present.

```bash
curl -s http://localhost:8080/platform/data | jq .
```

What to look at:

| Key | Content |
|---|---|
| `description_full` | detailed description (with names/proper nouns) |
| `description_summary` | summary (PII-redacted) |
| `description_model` | the VLM model ID used (for audit) |
| `description_generated_at` | inference time (ISO8601, for audit) |
| `processing_warnings` | steps where inference/blur failed (e.g. `vlm_unavailable`) |

If you can confirm that **semantic data is distributed over the platform without sharing the raw image**, the core of this hands-on is complete. Projecting `description_full` / `description_summary` per trust tier is covered in §12 of [DataUserVC × tiered access](data-user-vc-tiered.md).

## 5. Verification points

- `description_summary` conveys the meaning of the scene without exposing the raw image
- `description_model` and `description_generated_at` record the source and time of inference
- semantic data is generated and distributed with the local VLM only, without sending images to a cloud

## Something to think about

- When you distribute "semantic derivatives" instead of the raw image, what changes from the standpoint of privacy and data sovereignty?
- Organize the benefits and limits (speed, accuracy, operations) of running locally instead of using a cloud API.

## Extensions

- **Natural-language search and processing**: run natural-language search and summarization over the accumulated semantic data with an LLM ([LLM Planner](llm-planner.md) / [Regional safety assistant](regional-safety-assistant.md)).
- **Remote device control**: based on the analysis result, separate `plan` and `execute` to operate a device (same Part 3).
- **Raspberry Pi + Pi camera**: split the roles so the Pi captures and the PC/host runs the VLM (this page keeps everything on the laptop first).

## Common issues

- the first model pull is large and slow → wait for completion in the `vlm-pull` logs; do it when you have bandwidth to spare
- inference is slow / times out on CPU → use the lightweight `VLM_MODEL=moondream`, or raise `VLM_TIMEOUT_SEC`
- `processing_warnings` shows `vlm_unavailable` → check that the `vlm` service is up and the model pull is complete (`docker compose ... ps` / `logs`)
- the camera cannot be opened → change the camera index or stop the process holding it (see [USB webcam sample](webcam.md))

## Related pages

- Prerequisites: [USB webcam sample](webcam.md) / [Quickstart](../setup/quickstart.en.md)
- Deep dive: [DataUserVC × tiered access (§8 media integration, §12 VLM tier)](data-user-vc-tiered.md) / [LLM Planner](llm-planner.md)
- Design: [DataUserVC × tiered access control spec (tier extension: semantic-level redaction)](data-user-vc-tiered-spec.md)
