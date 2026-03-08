# HUSKYLENS2 Sample

## Goal

Use HUSKYLENS2 (or mock input) to generate events and confirm that they are productized in the pipeline.

## Official links

- HUSKYLENS2 product page: <https://www.dfrobot.com/product-2995.html>
- DFRobot official site: <https://www.dfrobot.com/>

## Prerequisites

- HUSKYLENS2 device, or a mock test environment
- `sensor-bridge` available
- `mediator-owner` already running and watching `raw_data/output`
- for serial mode, the serial device path is visible from the PC

## Exercise Programs

- [Problem program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/huskylens2_mock/problem_program.py)
- [Answer program](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/huskylens2_mock/answer_program.py)
- [Exercise guide](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/huskylens2_mock/README.md)

This exercise asks learners to build a mock event file under `mediator-owner/raw_data/output`.  
In the problem program, the main task is to complete `build_event()` and understand the minimum JSON structure for a HUSKYLENS2 detection event.

## Mock first

```bash
cd sensor-bridge
python3 huskylens_bridge.py --mode mock --output-dir ../mediator-owner/raw_data/output
```

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

- `pyserial` missing:
  - install with `pip install pyserial`
- serial port unknown:
  - check `/dev/ttyUSB0`, `/dev/tty.usbserial-*`, or OS device manager
- no productization after file generation:
  - verify `mediator-owner` is running
  - verify output path matches `../mediator-owner/raw_data/output`
