# Detect a person on an AI camera (HUSKYLENS2)

Turn detections from DFRobot's [1] HUSKYLENS2 AI camera [2] into event files.

> **What you'll do**: Generate event files from HUSKYLENS2 (or mock) detections and confirm they are registered as merchandise
>
> **Prerequisites**: [Environment Setup](../setup/index.en.md) + [Quickstart](../setup/quickstart.en.md)
>
> **What you need**: PC + HUSKYLENS2 (no device → use [Detect with a USB webcam (OpenCV)](webcam.md) instead)
>
> **Time required**: approx. 45 min

## Goal

Aggregate HUSKYLENS2 (or mock) detections every few seconds and write them out as event files. `mediator-owner` registers each file on the marketplace as merchandise.


## What this page helps you understand

- how HUSKYLENS2 detections are turned into event files
- when to use `mock` and when to use `serial`
- where to look in the generated event files

## Common stumbling points

- serial port naming and permissions often block the first run
- event files cannot be written if the output folder does not exist
- when real hardware is unstable, it is better to verify the pipeline in `mock` mode first

## Prerequisites

- HUSKYLENS2 device, or a mock test environment
- `sensor-bridge` available
- the output folder `mediator-owner/raw_data/output` exists (the steps create it if missing)
- for serial mode, the serial device path is visible from the PC
- Python 3 (serial mode also needs `pyserial`. Install them in a virtual environment (venv) at the top of the course repository (`cd ~/program/Blockchain_IoT_Marketplace && python3 -m venv .venv && source .venv/bin/activate && pip install pyserial`). In a newly opened terminal, run `source ~/program/Blockchain_IoT_Marketplace/.venv/bin/activate` first. On macOS there may be no `pip` command, and Homebrew Python rejects `pip install` outside a venv)

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
3. look at the file content (what was detected, with counts)
4. confirm that the `mediator-owner` terminal prints `Product deployed`

Branches after that:

- If you only want to confirm the pipeline: `mock` mode is enough
- If you want to confirm the real device path: continue with `serial` mode
- If serial mode fails early: check [Troubleshooting](#4-troubleshooting) first

## Phase 1: Confirm the pipeline in mock mode

### 1. Mock first

Run in the `sensor-bridge` folder of the course repository (the first `cd` below moves there). Mock mode generates simulated detections in place of a device.

```bash
cd ~/program/Blockchain_IoT_Marketplace/sensor-bridge
mkdir -p ../mediator-owner/raw_data/output
python3 huskylens_bridge.py \
  --mode mock \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-interval-sec 8
```

| Argument | Meaning |
|---|---|
| `--mode` | Input type: `mock` (simulated data), `serial` (device over serial), `tcp` |
| `--camera-id` | Camera ID used as the file name prefix |
| `--output-dir` | Where event files are written |
| `--flush-interval-sec` | How many seconds of detections go into one file |

One line is printed every 8 seconds, as below. Press `Ctrl+C` to stop.

```
[bridge] starting HuskyLens2 bridge
[bridge] mode=mock camera_id=301 output_dir=../mediator-owner/raw_data/output
[bridge] flush_interval_sec=8 min_confidence=0.50
[bridge] batch detections=12 file=../mediator-owner/raw_data/output/301_huskylens_1791549165.txt
[bridge] batch detections=13 file=../mediator-owner/raw_data/output/301_huskylens_1791549173.txt
```

Look at the generated files in another terminal (you can leave the bridge running).

```bash
ls ~/program/Blockchain_IoT_Marketplace/mediator-owner/raw_data/output
cat ~/program/Blockchain_IoT_Marketplace/mediator-owner/raw_data/output/301_huskylens_*.txt | head -20
```

```
# HuskyLens2 Detection Snapshot
camera_id: 301
window_start_utc: 2026-10-09T12:32:37.144510+00:00
window_end_utc: 2026-10-09T12:32:45.178510+00:00
total_detections: 12
unique_labels: 3
top_label: bike
max_confidence: 0.956
labels:
  - bike: 5
  - person: 4
  - car: 3
samples:
  - {"timestamp": "2026-10-09T12:32:38.149827+00:00", "label": "bike", "confidence": 0.6867, "id": "17", "x": 50.54, "y": 461.24, "w": 28.13, "h": 180.54, "source": "mock"}
  ...
```

Each file is one event that summarizes 8 seconds of detections. `labels` gives the count per detected label, and `samples` lists the individual detections (time, label, confidence, position and size on screen).

### Confirm registration as merchandise

If `mediator-owner` is running as set up in [Quickstart](../setup/quickstart.md), its terminal prints a line like the following each time an event file is written.

```
Product deployed: 0x948b3c65b89df0b4894abe91e6d02fe579834f8f for file 301_huskylens_1791551565.txt
```

`mediator-owner` watches `raw_data/output` and registers each new event file on the chain as merchandise. `0x...` is the item's address. Open the following URL in a browser to see its page.

```
http://localhost:5173/merchandise/<the address shown>
```

For purchasing the item and receiving the data, see §1.3 of the [Hands-on overview](index.md). If nothing is printed, check that the course repository is up to date (`git pull`) and that `mediator-owner` was restarted.

Once event files are generated in mock mode, the next step is to try the physical serial input.

## Phase 1: Try the same flow with serial input

### 2. Real device (serial)

For `--serial-port`, give the name of the port the HUSKYLENS2 is connected to. On macOS, replace it with the name you find with `ls /dev/tty.usb*` (on Linux it is something like `/dev/ttyUSB0`). Run serial mode in the `sensor-bridge` folder with the venv from the prerequisites (with `pyserial`) active (the first two lines below do this).

```bash
cd ~/program/Blockchain_IoT_Marketplace/sensor-bridge
source ../.venv/bin/activate
python3 huskylens_bridge.py \
  --mode serial \
  --serial-port /dev/tty.usbserial-XXXX \
  --baudrate 115200 \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output
```

### 3. Checkpoints

- `301_huskylens_*.txt` appears under `raw_data/output`
- `labels` in the file lists what was detected (person, etc.) with counts
- the `mediator-owner` terminal prints `Product deployed: 0x... for file 301_huskylens_...txt`

## Success example

- mock mode generates event files without hardware
- serial mode keeps generating files as detections arrive
- you can open the merchandise page and go as far as purchasing

### 4. Troubleshooting

- Symptom: `pyserial` is missing
  - Action: activate the venv (`source ~/program/Blockchain_IoT_Marketplace/.venv/bin/activate`), run `pip install pyserial`, and rerun in the same terminal
- Symptom: the serial port is unknown
  - Check: on macOS, run `ls /dev/tty.usb*` to find `/dev/tty.usbserial-…` or `/dev/tty.usbmodem…`; on Linux, `/dev/ttyUSB0`; or the OS device manager
- Symptom: the device cannot be read
  - Check: the cable carries data, not only power
  - Check: macOS / Linux permissions and the port name are correct
- Symptom: no event file is created
  - Check: the output folder (`../mediator-owner/raw_data/output`) exists. If not, create it with `mkdir -p ../mediator-owner/raw_data/output`
- Symptom: events are not registered as merchandise (`Product deployed` is not printed)
  - Check: `mediator-owner` is running
  - Check: the output path matches `mediator-owner/raw_data/output`
  - Check: the course repository is up to date (after `git pull`, restart `mediator-owner`)

## References

[1] DFRobot, "DFRobot," <https://www.dfrobot.com/>

[2] DFRobot, "HUSKYLENS2," product page, <https://www.dfrobot.com/product-2995.html>
