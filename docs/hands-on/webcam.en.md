# USB Webcam Sample (Littering)

## Goal

Generate `person_detected` / `possible_littering` events with only a USB webcam, even without HUSKYLENS2.

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

## Mock first

```bash
cd webcam-bridge
python3 webcam_litter_bridge.py --mode mock --output-dir ../mediator-owner/raw_data/output
```

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
