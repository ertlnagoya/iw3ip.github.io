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

- camera cannot be opened:
  - check whether another application is using the camera
  - try another `--camera-index`
- expected event does not appear:
  - improve lighting and framing
  - first verify the pipeline with mock mode
- no productization:
  - verify `mediator-owner` status and output path
