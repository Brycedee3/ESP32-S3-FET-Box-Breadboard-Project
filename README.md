# ESP32-S3 FET Control Box

A modernized version of the UIUC FET Control Box (DD00003, 2003) for ECEN power-electronics labs.
An ESP32-S3 replaces the original analog PWM and logic chips. It drives two isolated
MOSFETs in one of three switching modes, and the box can be controlled from the front
panel, a desktop app over USB, or a web app over Bluetooth.

| Component | File | Talks to the box over |
|---|---|---|
| Firmware | `MCU Code` (`mcuRev10_7.ino`) | — |
| Desktop app | `APP Code` (`guiRev10_2.py`) | USB serial |
| Web app | `Website Code` (`FET_GUI_Rev10_3.html`) | Bluetooth Low Energy |

---

## Features

- **Frequency:** 20 Hz – 500 kHz, set exactly in 12.5 ns steps (80 MHz timer clock)
- **Duty cycle:** 0 – 100 % (0 % = both FETs hard off)
- **FET 2 modes**, named as on the legacy front panel:

  | Mode | Panel switch | FET 2 behavior |
  |---|---|---|
  | **Copy (q)** | Up | Switches identically to FET 1 |
  | **Alternate** | Middle | FETs take turns; each runs at f/2 (for push-pull) |
  | **Complement (q′)** | Down | Inverse of FET 1 with 150 ns dead time |

  In Alternate mode, FET 2's pulse is centered exactly in FET 1's off-time at every
  frequency. A scope will read each FET at **half** the set frequency and **half** the
  set duty. That is expected, and matches the original box.
- **Safety watchdog:** if the controlling app goes silent for 3 s, the outputs shut off
  and the screen shows **LINK LOST**.
- **Temperature:** DS18B20 heatsink probe, shown on the box and in both apps.
- **Over-temperature protection:** above 60 °C (3 consecutive readings) the FET and q(t) outputs turn off, the LCD shows an OVER TEMP alert, and the apps become read-only. Everything resumes automatically below 59 °C.
- **Student login:** email + UIN checked against a Supabase roster, with each session logged.

---

## 1. Firmware (ESP32-S3)

### Install
1. **Arduino IDE 2.x**: <https://www.arduino.cc/en/software>
2. **ESP32 board package**: File → Preferences → *Additional boards manager URLs*:
   ```
   https://espressif.github.io/arduino-esp32/package_esp32_index.json
   ```
   Then Tools → Board → Boards Manager → install **esp32 by Espressif Systems, version 3.x**
   (tested with **3.3.5**). Version 2.x will not work: the firmware uses the IDF 5 MCPWM driver.
3. **Libraries** (Sketch → Include Library → Manage Libraries):
   - **Adafruit ILI9341** (click *Install All* to also get Adafruit GFX and Adafruit BusIO)
   - **OneWire**
   - **DallasTemperature**

   BLE support is built into the ESP32 package; nothing extra to install.

### Board settings (Tools menu)
> ⚠️ The IDE saves these **per sketch folder**. Check them every time you open a new sketch.

| Setting | Value |
|---|---|
| Board | **ESP32S3 Dev Module** (not "ESP32 Family Device") |
| USB CDC On Boot | **Enabled** (otherwise the apps can't talk to the box over USB) |
| USB Mode | **Hardware CDC and JTAG** |
| Flash Size / PSRAM | Match your module (e.g. N16R8 → 16 MB / OPI PSRAM) |

### Flashing
1. Put `mcuRev10_7.ino` in a folder named `mcuRev10_7` (the IDE requires the names to match).
   Keep only **one** `.ino` file in that folder.
2. Close the Serial Monitor and any app that is using the COM port.
3. If the upload can't connect: unplug USB, **hold BOOT while plugging it back in**, release.
4. Upload and wait for `Hash of data verified`.
5. **Press RESET.** The native USB often does not restart the sketch by itself.

A healthy boot prints:
```
BOOT: start
PWM:res=... period=... ticks ...
BOOT: pwm ok
BOOT: tft ok
TEMP:23.87
```

### Pin map (PCB)
| GPIO | Function | GPIO | Function |
|---|---|---|---|
| 1 | PWM-A → FET 1 gate driver | 10 | TFT CS |
| 2 | PWM-B → FET 2 gate driver | 11 | TFT MOSI |
| 5 / 6 / 7 | Freq encoder A / B / switch | 12 | TFT SCK |
| 15 / 16 / 21 | Duty encoder A / B / switch | 14 | TFT DC |
| 13 | DS18B20 data (4.7 kΩ pull-up to 3.3 V) | 8 | TFT RESET |
| 38 / 39 | FET 2 mode switch (up / down) | 19 / 20 | USB D− / D+ |

Do not use: IO0, IO3, IO45, IO46 (strapping pins), IO35–37 (PSRAM), IO19/20 (USB).

The display pins are set at the top of the sketch (`TFT_*_PIN`). No TFT_eSPI setup files are needed.

---

## 2. Desktop app (Python, USB)

### Install
Requires **Python 3.10+** (tested on 3.13).
```bash
python -m pip install -r requirements.txt
```
or individually:
```bash
python -m pip install customtkinter pyserial Pillow supabase
```

### Run
```bash
python guiRev10_2.py
```
1. Log in with your email and UIN.
2. Select the box's COM port (the same one the Arduino IDE shows under Tools → Port) and click **Connect**.
3. The live monitor fills in with the box's current duty, frequency, mode, and temperature.

Optional files in the same folder:
- `tamu_logo.png`: logo shown in the corner (the app runs fine without it)
- `fet_usage_log.csv`: created automatically as a local session log

If the USB link drops briefly, the app reconnects automatically (orange status light)
for up to 15 s. Status messages are printed in the terminal.

---

## 3. Web app (Bluetooth)

No install needed, but Web Bluetooth has requirements:
- **Browser:** Chrome or Edge on Windows, macOS, Android, or ChromeOS.
  Firefox and Safari/iOS do **not** support Web Bluetooth.
- **Secure origin:** the page must be served over **https://** or from **localhost**.
  The easiest option is GitHub Pages (Settings → Pages). For local testing:
  ```bash
  python -m http.server 8000
  ```
  then open <http://localhost:8000/FET_GUI_Rev10_3.html>.
- Bluetooth must be turned on in the operating system.

Click **Connect**, choose **FET_Box_BLE**, and log in.

> ⚠️ **Known limitation:** if the Bluetooth link drops, the current firmware does **not**
> shut the outputs off. The FETs keep switching at the last setting until the app
> reconnects (automatic, within ~15 s) or someone uses the front panel. Over USB, the
> watchdog shuts the outputs off after 3 s.

---

## 4. Supabase (login + usage log)

Both apps check the student's email and UIN against a `roster` table and write each login
to `user_sessions`. The project URL and publishable key are set at the top of each app.

> ⚠️ Enable **Row Level Security** on both tables before real deployment. Without it, the
> public key allows anyone to read the roster. The recommended setup is a
> `verify_student()` database function; see project notes.

---

## Serial / BLE protocol

Commands are plain text lines ending in `\n`, identical over USB and BLE.

| App → Box | Meaning |
|---|---|
| `DUTY:<0-100>` | Set duty cycle (%) |
| `FREQ:<20-500000>` | Set frequency (Hz) |
| `FET2:COPY` / `FET2:ALT` / `FET2:ALTQ` | Copy / Complement / Alternate |
| `SOURCE:INT` / `SOURCE:EXT` | q(t) master (box outputs q(t) on the BNC) / slave (box follows an external q(t)) |
| `DSRC:INT` / `DSRC:EXT` | Duty from the knob / from the D BNC voltage (internal q(t) only) |
| `CONN:ON` / `CONN:OFF` | App takes / releases control |
| `HEARTBEAT:1` | Keep-alive, sent every 500 ms |

| Box → App | Meaning |
|---|---|
| `DUTY:<n>` | Duty actually applied (may be clamped in Alternate mode) |
| `FREQ:<n>` | Frequency applied |
| `FET2:<mode>` | Current mode (sent on connect and when the panel switch moves) |
| `TEMP:<°C>` / `TEMP:ERR` | Heatsink temperature, once per second |
| `THERMAL:TRIP` / `THERMAL:OK` | Over-temperature shutdown active / cleared (apps lock / unlock controls) |
| `SOURCE:<INT/EXT>` / `DSRC:<INT/EXT>` | Current q(t) source and duty source |
| `INFO:ext=WAITING_FOR_SIGNAL / LOCKED / NO_SIGNAL` | External q(t) status |

The panel switch and the apps share control of the mode: whichever changed it most recently wins.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Compilation error: exit status 1` with no details | Turn on File → Preferences → *Show verbose output during compile* and find the first `error:` line |
| Log shows `FQBN: esp32:esp32:esp32_family` | Wrong board. Select **ESP32S3 Dev Module** |
| App connects, then `Write timeout` | **USB CDC On Boot** is Disabled. Enable it and re-upload |
| Upload: `Could not open COMx` | Close the Serial Monitor / app; hold BOOT while plugging in; reselect the port |
| Nothing runs after upload | Press RESET |
| `TEMP:ERR` / `SENSOR ERR` | Check the probe wiring (yellow = data → IO13) and the 4.7 kΩ pull-up |
| Screen stays white | Check the display pins in the sketch and continuity of each TFT line on the PCB |
| Random **LINK LOST** shutdowns | USB dropout; try a short cable plugged directly into the PC, and disable USB selective suspend |

---

## References
- R. S. Balog, P. T. Krein, *FET Control Box Redesign 2002*, Design Document DD00003, UIUC, 2004.
