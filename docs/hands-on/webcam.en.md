# USB Webcam Sample (Littering)

## Goal

Generate `person_detected` / `possible_littering` events with only a USB webcam, even without HUSKYLENS2.

## What this page helps you understand

- how to try a Phase 1 event-generation flow with only a USB webcam
- how `person_detected` and `possible_littering` are used differently
- which checkpoints show that the pipeline reached productization

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
- `mediator-owner` already running and watching `raw_data/output`
- in webcam mode, the camera device is recognized by the PC

## Exercise Programs

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/webcam_littering_mock/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/webcam_littering_mock/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/webcam_littering_mock/README.md)

This exercise generates a `possible_littering` mock event file for the USB webcam path.  
The problem program focuses on the minimum event structure and on the meaning of fields such as `camera_id`, `confidence`, and `event_type`.

## Shortest path

For a first pass, these four steps are enough.

1. start `webcam_litter_bridge.py` in `mock` mode
2. confirm that `*_webcam_event_*.txt` files are generated
3. confirm that `mediator-owner` productizes them
4. confirm that the merchandise list is updated in the frontend

Branches after that:

- If you only want to confirm the downstream pipeline: `mock` mode is enough
- If you want to inspect real camera detection: continue to `webcam` mode
- If the camera cannot be opened: check camera index and device ownership first

## Process table of contents

<details class="iw3ip-toc-details" open>
  <summary>Check 1: confirm event generation in mock mode</summary>
  <p>Start with mock mode so you can confirm that the downstream pipeline works even without the real camera.</p>
  <ol>
    <li><a href="#mock-first">Mock first</a></li>
    <li><a href="#success-example">Success example</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 2: inspect detections in webcam mode</summary>
  <p>Next, switch to the real USB webcam and inspect how `person_detected` and `possible_littering` are produced.</p>
  <ol>
    <li><a href="#usb-webcam">USB webcam</a></li>
    <li><a href="#event-types">Event types</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Check 3: review caveats and troubleshoot common failures</summary>
  <p>Finally, review the limits of the heuristic approach and the most common issues related to cameras and output paths.</p>
  <ol>
    <li><a href="#troubleshooting">Troubleshooting</a></li>
  </ol>
</details>

## How to read this page

This is a Phase 1 camera-input page. As with the HUSKYLENS2 page, it is usually easier to confirm the downstream path with `mock` first and only then move to the real camera path.

## Phase 1: Confirm the pipeline in mock mode

## Mock first

```bash
cd webcam-bridge
python3 webcam_litter_bridge.py --mode mock --output-dir ../mediator-owner/raw_data/output
```

At this point, the downstream pipeline has been confirmed. The next step is to switch to the physical webcam path and inspect the actual detections.

## Phase 1: Inspect detections in webcam mode

## USB webcam

```bash
python3 webcam_litter_bridge.py --mode webcam --camera-index 0 --output-dir ../mediator-owner/raw_data/output
```

## Event types

- `person_detected`
- `possible_littering`

## Success example

- mock mode generates event files without camera hardware
- webcam mode outputs `person_detected` or `possible_littering`
- generated events are productized and become purchasable

## Troubleshooting

- Symptom: the camera cannot be opened
  - Check: another application is not using the camera
  - Check: another `--camera-index` was tried
- Symptom: the expected event does not appear
  - Check: lighting and framing are appropriate
  - Check: the pipeline works in mock mode first
- Symptom: no productization happens
  - Check: `mediator-owner` is running
  - Check: the output path is correct
