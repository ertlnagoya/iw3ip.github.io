# Detect with a USB webcam (OpenCV)

Without a HUSKYLENS2, a generic USB webcam plus OpenCV is enough to emit `person_detected` and `possible_littering` events.

> **What you'll do**: Detect a person and littering candidates from a USB webcam, emit events
>
> **Prerequisites**: [Setup](../setup/index.en.md) + [Quickstart](../setup/quickstart.en.md)
>
> **What you need**: PC + USB webcam
>
> **Time**: ~30 min

## Goal

Generate `person_detected` / `possible_littering` events with only a USB webcam, even without HUSKYLENS2.

## What this page helps you understand

- how to try a Phase 1 event-generation flow with only a USB webcam
- how `person_detected` and `possible_littering` are used differently
- where to look in the generated event files

## Common stumbling points

- camera index and camera ownership are common first-run failures
- lighting and framing strongly affect the heuristic output
- detections can be generated while the downstream pipeline still fails to pick them up

## Official links

- OpenCV: <https://opencv.org/>
- Webcam overview (reference): <https://en.wikipedia.org/wiki/Webcam>

## Prerequisites

- USB webcam, or a mock execution setup
- `webcam-bridge` available
- the output folder `mediator-owner/raw_data/output` exists (the steps create it if missing)
- in webcam mode, the camera device is recognized by the PC
- Python 3 (webcam mode also needs `pip install ultralytics opencv-python`)

## Exercise Programs

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/webcam_littering_mock/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/webcam_littering_mock/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/webcam_littering_mock/README.md)

This exercise generates a `possible_littering` mock event file for the USB webcam path.  
The problem program focuses on the minimum event structure and on the meaning of fields such as `camera_id`, `confidence`, and `event_type`.

## Shortest path

For a first pass, these four steps are enough.

1. start `webcam_litter_bridge.py` in `mock` mode
2. confirm that `*_webcam_event_*.txt` files are generated
3. look at the file content (event type and score)
4. confirm that the `mediator-owner` terminal prints `Product deployed`

Branches after that:

- If you only want to confirm the downstream pipeline: `mock` mode is enough
- If you want to inspect real camera detection: continue to `webcam` mode
- If the camera cannot be opened: check camera index and device ownership first

## Phase 1: Confirm the pipeline in mock mode

## Mock first

Run from the top of the course repository (`Blockchain_IoT_Marketplace/`). Mock mode generates simulated events in place of a camera (no extra packages needed).

```bash
cd webcam-bridge
mkdir -p ../mediator-owner/raw_data/output
python3 webcam_litter_bridge.py \
  --mode mock \
  --camera-id 401 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-seconds 5
```

One line is printed every 5 seconds, as below. Press `Ctrl+C` to stop.

```
[webcam-bridge] mock mode start
[webcam-bridge] wrote ../mediator-owner/raw_data/output/401_webcam_event_1791549330.txt
[webcam-bridge] wrote ../mediator-owner/raw_data/output/401_webcam_event_1791549335.txt
```

Look at the generated files.

```bash
cat ../mediator-owner/raw_data/output/401_webcam_event_*.txt | head -10
```

```
# Webcam Event Snapshot
camera_id: 401
event_type: possible_littering
event_time_utc: 2026-10-09T12:35:30.906857+00:00
event_score: 0.801
summary: Bottle remained on ground area for long duration
details:
{"event_type": "possible_littering", "event_time_utc": "...", "event_score": 0.801, "summary": "Bottle remained on ground area for long duration", "object": {...}}
```

Each file is one event. `event_type` is the kind of event, `event_score` is its confidence, and `details` holds the underlying detection.

### Confirm registration as merchandise

If `mediator-owner` is running as set up in [Quickstart](../setup/quickstart.md), its terminal prints a line like the following each time an event file is written.

```
Product deployed: 0xc6ba8c3233ecf65b761049ef63466945c362edd2 for file 401_webcam_event_1791551565.txt
```

`mediator-owner` watches `raw_data/output` and registers each new event file on the chain as merchandise. `0x...` is the item's address. Open the following URL in a browser to see its page.

```
http://localhost:5173/merchandise/<the address shown>
```

For purchasing the item and receiving the data, see §1.4 of the [Hands-on overview](index.md). If nothing is printed, check that the course repository is up to date (`git pull`) and that `mediator-owner` was restarted.

Once event files are generated in mock mode, the next step is to switch to the physical webcam and inspect the actual detections.

## Phase 1: Inspect detections in webcam mode

## USB webcam

Webcam mode uses an object-detection model (YOLO). Install the packages with `pip install ultralytics opencv-python`. On the first run the model file (`yolov8n.pt`, a few MB) is downloaded automatically.

```bash
python3 webcam_litter_bridge.py --mode webcam --camera-index 0 --camera-id 401 --output-dir ../mediator-owner/raw_data/output
```

## Event types

- `person_detected`
- `possible_littering`

## Success example

- mock mode generates event files without camera hardware
- webcam mode outputs `person_detected` or `possible_littering`

## Troubleshooting

- Symptom: the camera cannot be opened
  - Check: another application is not using the camera
  - Check: another `--camera-index` was tried
- Symptom: the expected event does not appear
  - Check: lighting and framing are appropriate
  - Check: the pipeline works in mock mode first
- Symptom: no event file is created
  - Check: the output folder (`../mediator-owner/raw_data/output`) exists. If not, create it with `mkdir -p ../mediator-owner/raw_data/output`
- Symptom: events are not registered as merchandise (`Product deployed` is not printed)
  - Check: `mediator-owner` is running
  - Check: the output path matches `mediator-owner/raw_data/output`
  - Check: the course repository is up to date (after `git pull`, restart `mediator-owner`)
