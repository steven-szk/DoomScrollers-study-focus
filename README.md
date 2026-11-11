# Locked – by The Doomscrollers

Hackathon project that helps you **stay locked in while studying** and keeps you off your phone.
A webcam checks whether you are holding your phone or looking away from the screen, the phone's own tilt
sensor (via **phyphox → MATLAB**) shows whether it has been picked up off the table, and an **Arduino**
gives you feedback with an RGB LED and a vibration motor.

```
Webcam + phone tilt -> Python (main.py) -> Arduino (LED + vibration)
```

## Quick commands

```bash
# 1. Phone: open phyphox -> "Inclination" experiment -> enable "Allow remote access"
#    put the URL it shows into baseUrl in PhoneSensorData.m

# 2. MATLAB (current folder = this repo), keeps writing PhoneSensorData.csv
PhoneSensorData            # press any key in its little window to stop

# 3. Arduino: upload ArduinoCodev2/ArduinoCodev2.ino, set the serial port in main.py

# 4. Python
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python main.py                     # Ctrl+C to stop
```

Read more at: [How it works](#how-it-works) · [Hardware & wiring](#hardware--wiring) ·
[Setup](#setup) · [Running](#running) · [Troubleshooting](#troubleshooting)

---

```
Webcam ──> captured_images/photo.jpg ─┬─> YOLOv8s          (phone overlapping a person?)
                                      └─> MediaPipe Face Mesh (head pose: yaw / pitch)
Phone (phyphox) ──HTTP──> PhoneSensorData.m ──> PhoneSensorData.csv (tilt L/R, U/D)
                                      │
                                      v
                       main.py  ──>  GREEN / YELLOW / RED  ──>  log.txt
                                      │
                         Serial 9600  │  "S0" / "S1" / "S2"
                                      v
                     Arduino ──> RGB LED 1 (state), RGB LED 2 (white), vibration motor
                             ──> ultrasonic "D:<cm>" back to the PC (read, not used in the decision yet)
```

---

## How it works

Every loop (`time.sleep(0.3)` between checks) `main.py` does:

1. **Capture** (`camera.py`) – one frame from webcam `0`, saved over `captured_images/photo.jpg`.
2. **Phone in hand** (`phone_tracker.py`) – YOLOv8s (`yolov8s.pt`, COCO classes `0` person, `67` cell phone).
   Distracted if any phone box **overlaps** a person box → `"Doomscrolling"`.
3. **Head pose** (`eye_tracker.py`) – MediaPipe Face Mesh, 6 landmarks → `cv2.solvePnP` → pitch / yaw / roll.

   | Condition | Result | Counts as wandering |
   |-----------|--------|---------------------|
   | yaw > 15° | Looking right | yes |
   | yaw < −15° | Looking left | yes |
   | pitch > 15° | Looking down | no (only with phone off table, see below) |
   | pitch < −15° | Looking up | yes |
   | no face | Away from screen | yes |
   | otherwise | Locked in | no |

4. **Phone off table** (`phone_table_finder.py`) – reads the **last row** of `PhoneSensorData.csv`;
   off the table if `|LeftRight_deg|` or `|Vertical_deg|` > **10°**.
   The ultrasonic check (`D:<cm>`, on table if < 4 cm or out of range) is also computed, but
   `is_phone_off_table()` currently **only returns the orientation result** – ultrasonic needs refinement.
5. **Decision**

   ```
   distracted = phone_detected
             or (eyes_wandering and phone_off_table)
             or (looking_down   and phone_off_table)
   ```

6. **State → Arduino + `log.txt`**

   | State | Command | When | LED 1 | Vibration |
   |-------|---------|------|-------|-----------|
   | GREEN – locked in | `S0` | not distracted (timer resets) | green | off |
   | YELLOW – distracted | `S1` | distracted, less than `DISTRACTION_THRESHOLD` | yellow | off |
   | RED – distracted too long | `S2` | distracted for ≥ `DISTRACTION_THRESHOLD` | red | **on** |

   `DISTRACTION_THRESHOLD` is in **seconds** at the top of `main()`. It is `3` for the demo;
   set it to `300` for the 5-minute version from the slides.

`log.txt` is cleared on start and gets one line (`Green` / `Yellow` / `Red`) per loop, intended for the MATLAB GUI.

---

## Repository layout

| Path | Purpose |
|------|---------|
| `main.py` | Entry point: capture → trackers → decision → Arduino command + log, loops until Ctrl+C |
| `camera.py` | Opens webcam `0` on import; `capture_image()`, `release_camera()`. Run alone to take one photo |
| `phone_tracker.py` | Loads YOLOv8s on import; `person_status()` → `[(bool, msg)]` |
| `eye_tracker.py` | MediaPipe head pose; `is_not_looking()` → `[(bool, msg)]` |
| `phone_table_finder.py` | Ultrasonic + CSV tilt checks; `is_phone_off_table()`. Run alone to test |
| `arduino_controller.py` | `ArduinoController` serial class (`send_command`, `read_serial`, `close`). Falls back to simulation mode if the port won't open. Run alone to cycle S0 → S1 → S2 |
| `log.py` | `reset_file()` / `write_file()` for `log.txt` |
| `PhoneSensorData.m` | MATLAB: polls phyphox every 0.1 s, appends new samples to `PhoneSensorData.csv` |
| `app.mlapp` | MATLAB App Designer study timer (Start / Pause-Resume / End-Reset, `mm:ss`), R2025b |
| `ArduinoCodev1/` | First firmware – LEDs + vibration only (ultrasonic function not called, TRIG pin clashes with LED 2) |
| `ArduinoCodev2/` | **Current firmware** – ultrasonic moved to D7/D8, sends distance every 500 ms |
| `yolov8s.pt` | YOLOv8s COCO weights |
| `requirements.txt` | Python dependencies |
| `Locked Hackathon.pptx` | Hackathon presentation |

---

## Hardware & wiring

Arduino (Uno-style pinout) running `ArduinoCodev2`. RGB LEDs are driven **active-LOW** (common anode).

| Part | Arduino pin |
|------|-------------|
| Vibration motor | D2 |
| LED 1 (state) R / G / B | D9 / D10 / D11 |
| LED 2 (white) R / G / B | D3 / D5 / D6 |
| Ultrasonic TRIG | D7 |
| Ultrasonic ECHO | D8 |

> `ArduinoCodev1` uses TRIG = D3 / ECHO = D4, which clashes with LED 2 red (D3). Use v2.

Phone: any phone with **phyphox**, on the same network as the laptop
(default `baseUrl = "http://172.20.10.1"` in `PhoneSensorData.m`).

---

## Serial protocol (9600 baud)

| Direction | Message | Meaning |
|-----------|---------|---------|
| PC → Arduino | `S0` | GREEN, vibration off |
| PC → Arduino | `S1` | YELLOW, vibration off |
| PC → Arduino | `S2` | RED, vibration on |
| Arduino → PC | `D:<cm>` | Ultrasonic distance (every 500 ms) |
| Arduino → PC | `D:Out of range` | No echo (timeout) |
| Arduino → PC | `Serial is ready. Send S0, S1, or S2.` | Sent once on boot |

The Arduino only updates the LEDs / motor when the state changes.

---

## Setup

### Python

Use Python ≤ 3.12 (`mediapipe==0.10.13` is pinned for the `mp.solutions` API).

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Set the Arduino port in `main.py`:

```python
arduino = ArduinoController(port='/dev/cu.usbserial-10')   # macOS
# arduino = ArduinoController(port='COM12')                # Windows - check Device Manager
```

### MATLAB

- Open the repo folder as the **current folder** – the CSV is written to `pwd`, and `main.py` reads it from its own working directory.
- In phyphox, the script reads the buffers `t`, `tiltFlatLR`, `tiltFlatUD`.
- Edit `baseUrl` and `dt` at the top of `PhoneSensorData.m`.

### Arduino

Open `ArduinoCodev2/ArduinoCodev2.ino` in Arduino IDE and upload. No extra libraries needed.

---

## Running

### Full system

1. Start phyphox on the phone and run `PhoneSensorData` in MATLAB.
2. Plug in the Arduino.
3. `python main.py` from the repo folder. Ctrl+C closes the serial port and releases the camera.
4. Optional: run `app` in MATLAB for the study timer.

### Component tests

```bash
python camera.py               # take one photo -> captured_images/photo.jpg
python phone_table_finder.py   # off-table check with example serial data + current CSV
python arduino_controller.py   # cycle S0 -> S1 -> S2 every 2 s (uses default port COM12)
```

---

## Troubleshooting

- **`Failed to connect to Arduino` / `(Simulation): S0`** – wrong port in `main.py`, or the Serial Monitor in Arduino IDE is holding the port.
- **`Error: Could not open camera`** – another app is using the webcam, or change `cv2.VideoCapture(0)` to another index in `camera.py`.
- **`File not found: PhoneSensorData.csv`** – MATLAB is not running from the repo folder (phone is then treated as on the table).
- **Phone always "on table"** – check phyphox remote access is on and `baseUrl` is right; the CSV should keep growing.
- **Goes RED almost immediately** – `DISTRACTION_THRESHOLD` is `3` s; raise it.
- **Ultrasonic has no effect** – expected, see step 4 of [How it works](#how-it-works).

---

## Future work

From the presentation:

- Wireless charging
- Magnets for MagSafe
- Better sensor
- Offline streaming
- GUI
