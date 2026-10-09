# Detect a person on an AI camera (HUSKYLENS2)

Pull events off the HUSKYLENS2 AI camera and list them on the marketplace.

> **What you'll do**: Detect a person on HUSKYLENS2, emit an event, and list it
>
> **Prerequisites**: [Setup](../setup/index.en.md) + [Quickstart](../setup/quickstart.en.md)
>
> **What you need**: PC + HUSKYLENS2 (no device → use [USB webcam sample](webcam.en.md) instead)
>
> **Time**: ~45 min

## Goal

Use HUSKYLENS2 (or mock input) to generate events and confirm that they are productized in the pipeline.

## What this page helps you understand

- how HUSKYLENS2 detections are turned into event files
- when to use `mock` and when to use `serial`
- which outputs confirm that productization really happened

## Common stumbling points

- serial port naming and permissions often block the first run
- the pipeline stops if the `sensor-bridge` output path does not match what `mediator-owner` watches
- when real hardware is unstable, it is better to verify the pipeline in `mock` mode first

## Official links

- HUSKYLENS2 product page: <https://www.dfrobot.com/product-2995.html>
- DFRobot official site: <https://www.dfrobot.com/>

## Prerequisites

- HUSKYLENS2 device, or a mock test environment
- `sensor-bridge` available
- `mediator-owner` already running and watching `raw_data/output`
- for serial mode, the serial device path is visible from the PC
- Python 3 (serial mode also needs `pyserial`)

## Exercise Programs

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/huskylens2_mock/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/huskylens2_mock/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/huskylens2_mock/README.md)

This exercise asks learners to build a mock event file under `mediator-owner/raw_data/output`.  
In the problem program, the main task is to complete `build_event()` and understand the minimum JSON structure for a HUSKYLENS2 detection event.

## Shortest path

For a first pass, these four steps are enough.

1. start `huskylens_bridge.py` in `mock` mode
2. confirm that event files appear under `raw_data/output`
3. confirm that `mediator-owner` picks them up
4. confirm that merchandise appears in the frontend

Branches after that:

- If you only want to confirm the pipeline: `mock` mode is enough
- If you want to confirm the real device path: continue with `serial` mode
- If serial mode fails early: jump to the troubleshooting section first

## Phase 1: Confirm the pipeline in mock mode

## Mock first

```bash
cd sensor-bridge
python3 huskylens_bridge.py --mode mock --output-dir ../mediator-owner/raw_data/output
```

At this point, the downstream pipeline itself has been confirmed. The next step is to try the same flow with the physical serial input.

## Phase 1: Try the same flow with serial input

## Real device (serial)

```bash
python3 huskylens_bridge.py --mode serial --serial-port /dev/ttyUSB0 --output-dir ../mediator-owner/raw_data/output
```

## Expected

- `301_huskylens_*.txt` appears under `raw_data/output`
- Owner mediator detects and productizes
- merchandise appears in the frontend

## Success example

- mock mode generates event files without hardware
- serial mode keeps generating files as detections arrive
- the generated event is reflected as purchasable merchandise

## Troubleshooting

- Symptom: `pyserial` is missing
  - Action: install with `pip install pyserial`
- Symptom: the serial port is unknown
  - Check: `/dev/ttyUSB0`, `/dev/tty.usbserial-*`, or the OS device manager
- Symptom: no productization after file generation
  - Check: `mediator-owner` is running
  - Check: the output path matches `../mediator-owner/raw_data/output`
