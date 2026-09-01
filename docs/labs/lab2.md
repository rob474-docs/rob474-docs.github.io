---
layout: default
title: Lab 2
nav_order: 2
parent: Labs
last_modified_at: 2026-09-01 12:00:00 -0400
---

# Lab 2: PX4 Flight Controller Software Setup and Basic Tuning

**Course:** Uncrewed Aerial Systems  
**Prerequisites:** Lab 1 (Dynamometer Characterization), basic familiarity with Linux terminal  
**Estimated Time:** 3 hours  
**Hardware Required:**
- Custom quadrotor with MicoAir743v2 AIO flight controller running PX4
- USB-C cable
- RC transmitter and receiver (bound and ready)
- Battery (for ESC calibration and motor testing)
- Personal laptop with USB port

---

## Overview

In this lab you will configure a PX4-based quadrotor from a fresh firmware state to a flight-ready system. You will set up the PX4 development toolchain, build firmware from source, flash it to the flight controller, step through all mandatory calibrations, and configure flight modes.

By the end of this lab your drone will be ready for its first autonomous hover test in Lab 3.

---

## Part 1: Install QGroundControl

QGroundControl (QGC) is the primary ground control station (GCS) for PX4. It handles sensor calibration, parameter management, and real-time telemetry display.

### Linux

1. Download the latest QGroundControl AppImage from the official releases page:
   ```
   https://docs.qgroundcontrol.com/master/en/qgc-user-guide/getting_started/download_and_install.html
   ```

2. Make the AppImage executable and run it:
   ```bash
   chmod +x QGroundControl-x86_64.AppImage
   ./QGroundControl-x86_64.AppImage
   ```

3. If QGC fails to open the serial port, add yourself to the `dialout` group:
   ```bash
   sudo usermod -aG dialout $USER
   ```

4. You must now log out and log back in for these changes to take effect.


### Windows

Download and run the `.exe` installer from the same link above. No additional drivers are needed for the MicoAir743v2 on Windows 10/11.

### macOS

Download the `.dmg` and drag to Applications. Grant serial port permissions if prompted by macOS privacy settings.

---

## Part 2: Set Up the PX4 Development Toolchain

The PX4 toolchain lets you build firmware from source and flash it directly to the board. We will compile and upload a custom firmware build in Part 4.

**Reference:** [https://docs.px4.io/main/en/dev_setup/dev_env_linux_ubuntu](https://docs.px4.io/main/en/dev_setup/dev_env_linux_ubuntu)

1. Create a working directory and clone the PX4 source:
   ```bash
   mkdir ~/uas
   cd ~/uas
   git clone https://github.com/PX4/PX4-Autopilot.git --recursive
   ```
   The `--recursive` flag is required — PX4 uses many git submodules. This download is large (~2 GB) and may take several minutes.

2. Run the Ubuntu setup script to install all required dependencies (compilers, udev rules, Python packages):
   ```bash
   bash ./PX4-Autopilot/Tools/setup/ubuntu.sh
   ```
   Enter your password when prompted. This script will install packages system-wide.

3. **Log out and log back in** after the script completes. This is required for new udev rules and group memberships to take effect.

---

## Part 3: Flash the PX4 Bootloader and Firmware (One-Time Setup)

> **Note:** This step only needs to be done once per board. The MicoAir743v2 ships with a manufacturer bootloader that is not compatible with PX4's upload tool. We flash a combined image that installs the PX4 bootloader and stock firmware in one step, so that future firmware builds can be uploaded directly from the build system.

**Helpful references:**
- [https://micoair.com/flightcontroller_micoair743v2_aio_45a/](https://micoair.com/flightcontroller_micoair743v2_aio_45a/)
- [https://micoair.com/docs/loading-firmware-micoair743/](https://micoair.com/docs/loading-firmware-micoair743/)
- [YouTube walkthrough](https://www.youtube.com/watch?v=53Lv2s-gBa8)

### 3.1 Download the Combined Bootloader + Firmware Image

Download `MicoAir743v2_PX4-1.16.2_Bootloader+Firmware.bin` from the MicoAir firmware repository:

```
https://github.com/micoair/MicoAir743v2/tree/main/Firmware/PX4/1.16
```

Save the `.bin` file to a known location on your laptop.

### 3.2 Enter DFU Mode

DFU (Device Firmware Upgrade) mode allows direct low-level access to the STM32 chip, bypassing any existing firmware.

> **Important:** Use a USB cable with data lines — charge-only cables will not work and the board will not appear on your computer.

1. Unplug the USB cable from the board.
2. **Hold the BOOT button** on the MicoAir743v2 (small button on the board, labeled BOOT).
3. While still holding BOOT, **plug in the USB-C cable** to your laptop.
4. Hold for 2 more seconds, then release the BOOT button.

Verify the board is in DFU mode:
```bash
lsusb | grep 0483
```
You should see `STMicroelectronics STM32 BOOTLOADER`. If nothing appears, try a different USB cable or repeat the BOOT button sequence.

### 3.3 Flash with dfu-util (Linux)

1. Install dfu-util:
   ```bash
   sudo apt install dfu-util
   ```

2. Flash the combined image:
   ```bash
   sudo dfu-util -a 0 --dfuse-address 0x08000000 -D MicoAir743v2_PX4-1.16.2_Bootloader+Firmware.bin
   ```

3. When complete you should see `File downloaded successfully`. Unplug and replug the USB cable — the board will reboot into PX4 firmware and appear as `/dev/ttyACM0`.

### 3.4 Flash with STM32CubeProgrammer (Windows / macOS)

1. Download and install STM32CubeProgrammer (free ST account required):
   ```
   https://www.st.com/en/development-tools/stm32cubeprog.html
   ```
2. Open STM32CubeProgrammer and select **USB** as the connection type.
3. Click **Connect**.
4. Click **Erase chip** (full chip erase).
5. Click **Open file** and select `MicoAir743v2_PX4-1.16.2_Bootloader+Firmware.bin`.
6. Click **Download**. You should see "File download complete".
7. Disconnect and reconnect the USB cable to reboot into PX4 firmware.

---

## Part 4: Build and Flash PX4 Firmware from Source

With the PX4 bootloader installed, you can now build firmware from source and flash it directly.

**Reference:** [https://docs.px4.io/main/en/dev_setup/building_px4](https://docs.px4.io/main/en/dev_setup/building_px4)

1. Navigate to the PX4-Autopilot directory:
   ```bash
   cd ~/uas/PX4-Autopilot
   ```

2. Run the build and upload command for the MicoAir743v2:
   ```bash
   make micoair_h743-v2_default upload
   ```
   This will compile the firmware (~5–10 minutes on first build, faster on subsequent builds).

3. When the build completes, the terminal will print:
   ```
   Waiting for bootloader...
   ```
   At this point, **plug in the USB-C cable** to the board. The PX4 bootloader will be detected and firmware upload will begin automatically.

4. Wait for the upload to finish. The terminal will confirm with `Erase`, `Program`, `Verify` steps followed by a success message. The board will reboot into the new firmware.

> **Subsequent builds:** After the first flash, you can rebuild and reflash at any time with the same `make micoair_h743-v2_default upload` command. You do not need to re-enter DFU mode — the PX4 bootloader handles updates automatically.

---

## Part 5: Connect to the Flight Controller

1. Plug the MicoAir743v2 into your laptop with a USB-C cable.

2. Verify the OS sees the device:
   ```bash
   lsusb | grep -i mico
   ls /dev/ttyACM*
   ```
   You should see `Bus 001 Device 005: ID 1b8c:0036 Altium Limited MicoAir743v2` and `/dev/ttyACM0`.

3. Open QGroundControl. It will auto-detect the flight controller and connect — you should see the vehicle icon appear in the top bar and telemetry values update within a few seconds.

4. Click the **Q** logo (top-left) → **Vehicle Setup** to enter the configuration panels.

> **Note:** If QGC connects but immediately disconnects, the FC may be in bootloader mode. Power-cycle the board by unplugging and replugging the USB cable.

---

## Part 6: Vehicle Frame Selection

PX4 must know the motor layout to mix control outputs correctly.

1. Go to **Vehicle Setup → Airframe**.
2. In the search box type `Generic Quadcopter` or scroll to **Quadrotor X**.
3. Select **Generic Quadcopter (Quadrotor X)** and click **Apply and Restart**.
4. The FC will reboot and reconnect.

> Your quadrotor uses a standard X configuration: motors 1 (front-right), 2 (rear-left), 3 (front-left), 4 (rear-right), with alternating spin directions.

### Disable Magnetometer Pre-arm Check

The MicoAir743v2 does not have an onboard magnetometer. Suppress the pre-arm warning by setting the following parameter in **Vehicle Setup → Parameters**:

| Parameter | Value | Reason |
|---|---|---|
| `SYS_HAS_MAG` | 0 (None) | No magnetometer onboard |

---

## Part 7: Sensor Calibration

PX4 requires calibration of the onboard sensors before it will arm. Work through each calibration in order.

### 7.1 Board Orientation

If the flight controller is not mounted in the default orientation (arrow pointing forward), set the rotation here. If it is mounted flat with the arrow facing forward, leave at default.

1. Go to **Vehicle Setup → Sensors**.
2. Under **Orientations**, click Set Orientations
3. Select the orientation that matches how the board is physically mounted on the frame, in this case **Roll 180**.

### 7.2 Accelerometer

1. Go to **Vehicle Setup → Sensors → Accelerometer**.
2. Click **Start** and follow the 6-orientation procedure (level, on each side, upside down).
3. Place the drone on a level surface for the first orientation. Use a small bubble level if available.

### 7.3 Gyroscope

1. Go to **Vehicle Setup → Sensors → Gyroscope**.
2. Set the drone on a stable, level surface and click **Start**.
3. Do not move the drone for ~10 seconds while data is collected.

### 7.4 Level Horizon

1. Go to **Vehicle Setup → Sensors → Level Horizon**.
2. Place the drone on a flat surface and click **OK**.
3. This sets the software-level reference so PX4 knows what "flat" looks like for this specific mounting.

---

## Part 8: Radio and MAVLink Setup

This platform uses ExpressLRS (ELRS) for both RC control and MAVLink telemetry. Before calibrating the radio, configure PX4 to send MAVLink data over the TELEM1 port (wired to the ELRS receiver).

**Reference:** [https://www.expresslrs.org/software/mavlink/#configuring-elrs-tx-rx-for-mavlink](https://www.expresslrs.org/software/mavlink/#configuring-elrs-tx-rx-for-mavlink)

### 8.1 Configure MAVLink over ELRS

In **Vehicle Setup → Parameters**, set the following:

| Parameter | Value | Reason |
|---|---|---|
| `SER_TEL1_BAUD` | 460800 8N1 | Match ELRS MAVLink baud rate |
| `MAV_0_CONFIG` | TELEM1 | Route MAVLink stream to TELEM1 |
| `MAV_0_RATE` | 9600 B/s | Limit MAVLink rate to ELRS bandwidth |

After setting these parameters, **save and reboot** the flight controller (Parameters → Tools → Save, then power-cycle).

### 8.2 Radio Calibration

1. Power on your RC transmitter.
2. Go to **Vehicle Setup → Radio**.
3. Verify **Mode 2** is selected (throttle on left stick).
4. Click **Calibrate** and follow the prompts — move each stick to its full range (all corners and extremes) as instructed.
5. Verify the green bars respond correctly to each input:
   - **Channel 1:** Roll (right stick left/right)
   - **Channel 2:** Pitch (right stick up/down)
   - **Channel 3:** Throttle (left stick up/down)
   - **Channel 4:** Yaw (left stick left/right)
6. Click **Next** and then **Finish**.

### 8.3 Identify Switch Channels

Channels 1–4 (roll, pitch, throttle, yaw) are fixed by the Mode 2 stick layout and were confirmed in the calibration above. Channels 5 and up are auxiliary switches (SA, SB, SC, SD, ...), and **which physical switch lands on which channel number depends on your transmitter's model/mixer setup** — it is not guaranteed to match any specific lab example. You need to identify your own mapping before configuring flight modes in Part 9.

1. Stay on **Vehicle Setup → Radio** (outside of the calibration wizard). Below the calibration button is a live channel monitor — one bar per channel — that updates in real time from a connected transmitter.
2. Flip one switch at a time on your TX and watch the bars. The bar that moves identifies the channel number for that switch (e.g., "flipping SA moves the Channel 5 bar" → SA is on Channel 5).
3. Repeat for every switch you intend to use (arm switch, flight mode switch, kill switch), and write down the channel number for each. For a 3-position switch, confirm it produces three distinct, stable bar positions (low/center/high).
4. If your channel numbers differ from the example in Part 9 below, use your actual numbers when assigning **Mode Channel**, **Arm switch**, and **Kill Switch** — the *function* each switch performs is up to you, PX4 does not care which channel number it lives on as long as it's assigned correctly.

> You can also read the raw numeric value for any channel via **Analyze Tools → MAVLink Inspector → `RC_CHANNELS`** (fields `chan1_raw`...`chan18_raw`) if you want exact values instead of reading bars.

---

## Part 9: Flight Modes

Configure your RC transmitter switches to select PX4 flight modes.

> The channel numbers below (5, 6, 8) match a common ExpressLRS default layout, but **use the channel numbers you identified for your own transmitter in Part 8.3**, not necessarily these exact numbers.

1. Go to **Vehicle Setup → Flight Modes**.
2. Set **Mode Channel** to **Channel 6** (switch **SB** on the transmitter — 3-position).
3. Set the three switch positions to:
   - **Position 1 (up):** Stabilized (manual stabilized, good for learning)
   - **Position 2 (center):** Altitude Control (holds altitude using barometer)
   - **Position 3 (down):** Position Control (holds position using optical flow — requires optical flow setup in Part 12)
4. Set **Arm switch** to **Channel 5** (switch **SA**).
5. Set **Kill Switch** to **Channel 8** (switch **SD**) — this immediately cuts all motor output in an emergency.

> For initial flights in this lab, fly in **Stabilized** mode. Once the optical flow module is configured (Part 12), Position Control becomes available for indoor hover.

---

## Part 10: Actuator Configuration

Assign motor outputs and verify spin direction before first flight.

> **Warning:** Remove all propellers before this step. Motor shafts will spin.

### 10.1 Assign Motors and Set Geometry

1. Go to **Vehicle Setup → Actuators**.
2. Under the motor geometry diagram, assign each motor output to the correct ESC/motor port on the MicoAir743v2.
3. For each motor, enter its **Position X** and **Position Y** values (in meters) in the geometry panel. These define the motor's location relative to the vehicle's center of gravity in PX4's body frame (X = forward, Y = right):

   | Motor | Position | Position X (m) | Position Y (m) |
   |---|---|---|---|
   | Motor 1 | Front-right | 0.046 | 0.046 |
   | Motor 2 | Rear-left | -0.046 | -0.046 |
   | Motor 3 | Front-left | 0.046 | -0.046 |
   | Motor 4 | Rear-right | -0.046 | 0.046 |

   Leave **Position Z** at 0 for all four motors (they sit in the same horizontal plane as the FC).

4. Verify the geometry matches the physical frame — motors should be placed at the correct arms in the diagram (front-left, front-right, rear-left, rear-right), and the on-screen quad outline should update to match the values above.

> These values are for the standard lab quadrotor frame (46 mm motor offset along each axis from the center of gravity, ~130 mm motor-to-motor diagonal). If your frame differs, measure the actual X/Y offset of each motor hub from the FC/CG and use those values instead.

### 10.2 Verify and Correct Motor Spin Direction

Refer to the PX4 X-frame motor spin convention:

| Motor | Position | Required Spin |
|---|---|---|
| Motor 1 | Front-right | Counter-clockwise (CCW) |
| Motor 2 | Rear-left | Counter-clockwise (CCW) |
| Motor 3 | Front-left | Clockwise (CW) |
| Motor 4 | Rear-right | Clockwise (CW) |

1. In **Vehicle Setup → Actuators**, click the slider to enable motor testing ("I understand this will spin motors").
2. Spin each motor individually at low throttle and verify rotation direction by watching the shaft or propeller adapter.
3. If a motor spins the wrong direction, **swap any two of its three motor phase wires** at the ESC connector to reverse direction.

> Motor direction reversal via software parameter is not confirmed to work reliably on this board. Use the wire-swap method.

---

## Part 11: Battery and Power Setup

1. Go to **Vehicle Setup → Power**.
2. Set **Number of Cells** to match your battery (typically 4S = 4 cells for a quadrotor of this class).
3. Set **Full Voltage (per cell):** 4.20 V  
   **Empty Voltage (per cell):** 3.50 V
4. If you have a current sensor, calibrate it by entering the measured shunt resistance. For the MicoAir743v2 onboard sensor, use the value specified in the MicoAir documentation.
5. Set a **Low Battery Warning** at ~20% and **Critical Battery Failsafe** at ~10% to trigger a land-in-place action (since this platform has no GPS, Return to Launch is not available).

---

## Part 12: Configure Optical Flow

The MTF-01 optical flow and range sensor module provides velocity and altitude measurements that allow Position Control mode indoors without GPS.

**Reference:** [https://micoair.com/optical_range_sensor_mtf-01/](https://micoair.com/optical_range_sensor_mtf-01/)

### 12.1 Configure the MTF-01 Sensor (One-Time Setup)

The MTF-01 must be configured for PX4 MAVLink output using MicoAssistant before connecting to the flight controller.

1. Connect the MTF-01 directly to your laptop via USB.
2. Download and open **MicoAssistant**: [https://micoair.com/assistant/](https://micoair.com/assistant/)
3. Connect to the sensor in MicoAssistant. You should see live optical flow and range data in the interface.
4. Set the following parameters in MicoAssistant:
   - Output mode: **PX4 MAVLink**
   - Baud rate: **115200**
5. Disconnect the MTF-01 from the laptop and connect it to the **TELEM2** port on the MicoAir743v2.

> The coordinate frame shown in MicoAssistant will appear different from what the sensor sends via MAVLink to PX4 — this is expected. PX4 applies its own frame transformation.

### 12.2 Configure PX4 Parameters for Optical Flow

In **Vehicle Setup → Parameters**, set the following. Parameters marked "→ Then reboot" require a full power cycle before the next parameter takes effect.

| Parameter | Value | Note |
|---|---|---|
| `MAV_1_CONFIG` | TELEM 2 | Route optical flow MAVLink to TELEM2 — **reboot after setting** |
| `MAV_1_MODE` | Normal | |
| `SER_TEL2_BAUD` | 115200 8N1 | Match MTF-01 baud rate |
| `EKF2_OF_CTRL` | Enabled | Enable optical flow in EKF2 |
| `EKF2_RNG_CTRL` | Enabled (conditional) | Enable rangefinder in EKF2 |
| `EKF2_HGT_REF` | Range sensor | Use rangefinder as height reference — **reboot after setting** |
| `SENS_FLOW_ROT` | No Rotation | MTF-01 mounted in default orientation |
| `SENS_FLOW_MAXHGT` | 8 m | Maximum valid range of MTF-01 |
| `SENS_FLOW_RATE` | 100 Hz | Sensor update rate |

After setting all parameters and rebooting, verify the sensor is working in two steps:

1. **Confirm raw data is streaming:** Open **Widgets → MAVLink Inspector** in QGC. Expand **`OPTICAL_FLOW_RAD`** and watch `quality` (0–255) — it should read **100+** while the sensor is held over a textured, well-lit surface, and drop toward 0 over a plain or dark surface. Slide the frame sideways by hand and confirm `integrated_x`/`integrated_y` respond. Expand **`DISTANCE_SENSOR`** and confirm `current_distance` tracks height as you raise/lower the frame, staying within the `SENS_FLOW_MAXHGT` ceiling set above.
2. **Confirm PX4 is actually fusing it:** Open **Analyze Tools → MAVLink Console** and run `listener estimator_status_flags`. This prints roughly 70 boolean flags covering every sensor type EKF2 knows how to fuse (magnetometer, GPS, airspeed, external vision, fixed-wing, ...). Most of them do not apply to this platform, so it's easy to get lost in the list. For this quadrotor's configuration (no magnetometer, no GPS, flow + rangefinder for height/position), only the flags below matter:

   | Flag | Expected value here | What it tells you |
   |---|---|---|
   | `cs_opt_flow` | **True** | Optical flow is being fused into the main filter for horizontal velocity |
   | `cs_rng_hgt` | **True** | Rangefinder is the primary height reference (matches `EKF2_HGT_REF` = Range sensor) |
   | `cs_rng_kin_consistent` | **True** | Rangefinder readings are physically consistent with the vehicle's motion |
   | `reject_optflow_x` / `reject_optflow_y` | **False** | EKF2 is *not* throwing out flow measurements as bad. If either flips to True, suspect poor lighting, a low-texture surface, or vibration |
   | `fs_bad_optflow_x` / `fs_bad_optflow_y` | **False** | No persistent flow fault has been declared |
   | `cs_rng_fault` | **False** | No rangefinder fault |

   Everything else in the list — `cs_mag_*`, `cs_gnss_*`, `cs_ev_*`, `cs_fixed_wing`, `cs_wind`, and similar — is expected to read **False** on this build. That is normal, not a sign of a problem: those flags exist for sensors and vehicle types this platform doesn't have (magnetometer, GPS, external vision, fixed-wing).

   > **Common point of confusion:** `cs_yaw_align` will also read **False**, and will stay that way. Yaw alignment normally comes from a magnetometer or a GPS heading, and this platform has neither (`SYS_HAS_MAG` = 0, `EKF2_GPS_CTRL` = 0). The vehicle still has a self-consistent heading from the gyroscope — it's just not pinned to true/magnetic north, and it will drift slowly over a long flight. This is expected behavior for a flow-only setup, not a fault to chase down.

   > **Another point of confusion:** `cs_opt_flow_terrain` and `cs_rng_terrain` will likely read **False** even when everything is healthy. These track a separate *terrain sub-estimator* that PX4 uses mainly when baro or GPS is the primary height source and it still needs a height-above-ground estimate on the side. Since this platform already uses the rangefinder directly as its primary height reference (`cs_rng_hgt` = True), that secondary terrain filter isn't needed and normally won't engage. Don't confuse these two with `cs_opt_flow` / `cs_rng_hgt` above — those are the ones that actually confirm the sensor chain is working.

   As a quick intuitive check, watch the **Local Position** / velocity readout in the Fly view instrument panel while sliding the frame sideways by hand — with props off and disarmed, the velocity estimate should move with your hand in real time if flow is being fused correctly.

---

## Part 13: Safety Checks and Arming

PX4 requires all pre-arm checks to pass before it will allow arming.

1. In QGC, the top status bar will show red/yellow warnings if any check is failing.
2. Common pre-arm failures and fixes:

| Failure | Fix |
|---|---|
| `Accel not calibrated` | Repeat Part 7.2 |
| `RC not calibrated` | Repeat Part 8.2 |
| `No RC signal` | Check transmitter is on and bound |
| `GPS not locked` | This platform has no GPS — disable GPS requirement (see step 3) |

3. Disable the GPS pre-arm check since this platform has no GPS. In **Vehicle Setup → Parameters**, set `EKF2_GPS_CTRL` = 0. On older PX4 firmware the equivalent parameter is `COM_ARM_WO_GPS` = 1.

4. **Arm test (props off, indoors):**  
   - Switch flight mode to Stabilized.  
   - Hold left stick down-right for 2 seconds to arm.  
   - Motors should begin spinning at idle. Confirm all 4 motors spin.  
   - Disarm: hold left stick down-left for 2 seconds.

---

## Part 14: MAVProxy and pymavlink (Development Toolchain)

In future labs you will write Python scripts to command the drone autonomously. Install the development tools now.

```bash
pip install pymavlink MAVProxy
```

### Verify Connection with MAVProxy

```bash
mavproxy.py --master=/dev/ttyACM0 --baudrate=57600 --console
```

You should see heartbeat messages and MAVLink traffic in the console. Type `status` to see vehicle state. Type `exit` to quit.

### Verify with pymavlink (Python)

```python
from pymavlink import mavutil
import time

connection = mavutil.mavlink_connection('/dev/ttyACM0', baud=57600)
connection.wait_heartbeat()
print(f"Connected: system {connection.target_system}, component {connection.target_component}")

connection.mav.request_data_stream_send(
    connection.target_system,
    connection.target_component,
    mavutil.mavlink.MAV_DATA_STREAM_EXTRA1,
    10,  # 10 Hz
    1    # start
)

for _ in range(10):
    msg = connection.recv_match(type='ATTITUDE', blocking=True, timeout=2)
    if msg:
        print(f"Roll: {msg.roll:.3f} rad  Pitch: {msg.pitch:.3f} rad  Yaw: {msg.yaw:.3f} rad")
    time.sleep(0.1)
```

Save this as `verify_connection.py` and run it with `python3 verify_connection.py`. You should see live attitude data from the flight controller.

---

## Lab Deliverables

Submit the following before the next lab session:

1. **Screenshot** of QGC with all pre-arm checks passing (green status bar).
2. **Screenshot** of the Sensors page showing all sensors calibrated (green checkmarks).
3. **Python script output:** Paste the terminal output of `verify_connection.py` showing 10 lines of attitude data.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| QGC won't connect | Wrong port permissions | `sudo usermod -aG dialout $USER` then re-login |
| QGC connects then drops | FC in bootloader / power issue | Check USB cable quality; try powered hub |
| Board not detected in DFU mode (`lsusb` shows nothing) | Charge-only USB cable | Use a data-capable USB cable — many USB-C cables carry power only |
| `make upload` hangs at "Waiting for bootloader" | Board not detected | Unplug/replug USB after build completes |
| `make upload` fails after bootloader step | Old manufacturer bootloader still present | Repeat Part 3 (DFU bootloader flash) |
| Motors don't all spin | ESC not armed or wrong DSHOT config | Verify `DSHOT_CONFIG` parameter; check wiring |
| Drone drifts in Stabilized | Level horizon not calibrated | Redo Level Horizon calibration (Part 7.4) |
| Position Control drifts or won't engage | Optical flow not configured or no rangefinder lock | Verify MTF-01 wiring and repeat Part 12 |
| GPS pre-arm check fails | No GPS on platform | Set `EKF2_GPS_CTRL` = 0 in Parameters |

