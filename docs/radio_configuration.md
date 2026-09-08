---
layout: default
title: Radio Configuration
nav_order: 4
last_modified_at: 2026-09-08 12:00:00 -0400
---

# ExpressLRS Radio Configuration
{: .no_toc }

**Transmitter:** RadioMaster Pocket, internal 2.4 GHz ExpressLRS TX  
**Receiver:** RadioMaster XR2 2.4 GHz  
**Firmware:** ExpressLRS 4.0.1 (transmitter and receiver), Backpack 1.5.5  
**Tooling:** [ExpressLRS Configurator](https://github.com/ExpressLRS/ExpressLRS-Configurator/releases)  
**Prerequisites:** A vehicle built through [Assembly Instructions]({% link docs/assembly_instructions.md %}), with the receiver wired to UART1 / TELEM1

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Overview

This platform uses **ExpressLRS (ELRS)** for the radio link. ELRS carries both RC control and the MAVLink telemetry stream over the same 2.4 GHz connection, which is why the vehicle needs no separate telemetry radio.

There are **three** separate pieces of firmware to flash, and all three must agree:

| Device | Firmware | What it does |
|---|---|---|
| Transmitter (internal TX module) | ExpressLRS 4.0.1 | The RC link itself — sticks, switches, telemetry downlink |
| Backpack (ESP radio inside the transmitter) | Backpack 1.5.5 | Relays telemetry to the ground station over Wi-Fi |
| Receiver (RadioMaster XR2) | ExpressLRS 4.0.1 | The vehicle end of the RC link |

The transmitter and receiver pair using a **bind phrase** — a shared secret string that replaces the traditional bind-button procedure. Configure it once at flash time, and any transmitter and receiver carrying the same phrase will link automatically.

{: .warning }
> **Always power the remote controller before the vehicle.** Powering the vehicle first can leave it briefly reading an unbound or stale channel state.

---

## Part 1: Bind Phrases

Each radio and vehicle pair in the class uses its own phrase:

| Radio & vehicle number | Bind phrase |
|---|---|
| 1 | `MRobotics_UAS_01` |
| 2 | `MRobotics_UAS_02` |
| 3 | `MRobotics_UAS_03` |
| 4 | `MRobotics_UAS_04` |
| 5 | `MRobotics_UAS_05` |
| 6 | `MRobotics_UAS_06` |

{: .important }
> The bind phrase must be **byte-for-byte identical** on the transmitter, the backpack, and the receiver. It is case-sensitive, and a single missing underscore is enough to prevent the link from ever coming up — with no error message beyond a receiver that never binds. Copy and paste it rather than retyping it.

---

## Part 2: Transmitter Firmware

**Reference:** [ExpressLRS quick start — RadioMaster internal TX](https://www.expresslrs.org/quick-start/transmitters/rm-internal/)

### 2.1 Build the Firmware

Open the **ExpressLRS Configurator** and configure the build:

<a href="{{ '/assets/images/radio/elrs-tx-firmware-target.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-tx-firmware-target.png' | relative_url }}" alt="ExpressLRS Configurator showing release 4.0.1 selected with RadioMaster Pocket Internal 2.4GHz TX as the target and Wi-Fi flashing method" />
</a>

**Figure 1.** Transmitter build settings: release **4.0.1**, device category **RadioMaster 2.4 GHz**, device **RadioMaster Pocket Internal 2.4GHz TX**, flashing method **Wi-Fi**. *Click any figure to enlarge.*
{: .fs-3 .text-grey-dk-000 }

<a href="{{ '/assets/images/radio/elrs-tx-bindphrase.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-tx-bindphrase.png' | relative_url }}" alt="ExpressLRS Configurator device options with 2.4 GHz ISM regulatory domain and a custom binding phrase entered" />
</a>

**Figure 2.** Device options: regulatory domain **2.4 GHz ISM (Standard)**, binding phrase enabled and set to your vehicle's phrase from Part 1. Then click **BUILD**.
{: .fs-3 .text-grey-dk-000 }

<a href="{{ '/assets/images/radio/elrs-build-success.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-build-success.png' | relative_url }}" alt="ExpressLRS Configurator reporting a successful build with the resulting firmware.bin revealed in a file browser" />
</a>

**Figure 3.** A successful build. The Configurator opens a file browser containing `firmware.bin` — this is the file you upload to the radio in the next step.
{: .fs-3 .text-grey-dk-000 }

### 2.2 Flash over Wi-Fi

1. Turn on the radio.
2. **SYS → ExpressLRS → WiFi Connectivity → Enable WiFi**
3. Connect your laptop to the radio's Wi-Fi network.
4. Open `10.0.0.1` in a browser.
5. Upload `firmware.bin` and apply the update.

{: .note }
> **On macOS**, navigate to `10.0.0.1` directly in a normal browser window. The captive-portal window that pops up when you join the network will not let you choose a file to upload.

---

## Part 3: Backpack Firmware

The *backpack* is a second ESP-based radio inside the transmitter, separate from the ELRS link itself. It carries telemetry to the ground station over Wi-Fi.

### 3.1 Build the Firmware

<a href="{{ '/assets/images/radio/elrs-backpack-firmware.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-backpack-firmware.png' | relative_url }}" alt="ExpressLRS Configurator Backpack tab with release 1.5.5 selected and RadioMaster Pocket 2.4GHz TX as target" />
</a>

**Figure 4.** Backpack build settings, on the Configurator's **Backpack** tab: release **1.5.5**, device **RadioMaster Pocket 2.4GHz TX**, flashing method **Wi-Fi**.
{: .fs-3 .text-grey-dk-000 }

<a href="{{ '/assets/images/radio/elrs-backpack-bindphrase.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-backpack-bindphrase.png' | relative_url }}" alt="Backpack device options showing the same custom binding phrase entered as for the transmitter" />
</a>

**Figure 5.** The backpack takes the **same bind phrase** as the transmitter. Build, then flash.
{: .fs-3 .text-grey-dk-000 }

### 3.2 Flash over Backpack Wi-Fi

1. Turn on the radio.
2. **SYS → ExpressLRS → WiFi Connectivity → Enable Backpack WiFi**
3. Connect to the **backpack** Wi-Fi network — this is a different network from the one used in Part 2.
4. Open `10.0.0.1` in a browser and upload the firmware.

---

## Part 4: Receiver Firmware

### 4.1 Build the Firmware

<a href="{{ '/assets/images/radio/elrs-rx-firmware-target.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-rx-firmware-target.png' | relative_url }}" alt="ExpressLRS Configurator with release 4.0.1 and RadioMaster XR2 2.4GHz RX selected as the target" />
</a>

**Figure 6.** Receiver build settings: release **4.0.1**, device **RadioMaster XR2 2.4GHz RX**, flashing method **Wi-Fi**.
{: .fs-3 .text-grey-dk-000 }

<a href="{{ '/assets/images/radio/elrs-rx-bindphrase.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/elrs-rx-bindphrase.png' | relative_url }}" alt="Receiver device options with the same binding phrase as the transmitter and backpack" />
</a>

**Figure 7.** The receiver takes the **same bind phrase** again. All three — transmitter, backpack, receiver — must match.
{: .fs-3 .text-grey-dk-000 }

{: .important }
> Transmitter and receiver must run the **same ExpressLRS major version** (4.0.1 on both here). A version mismatch is a common cause of a link that will not come up despite correct bind phrases.

### 4.2 Flash over Wi-Fi

1. **Turn the radio transmitter off**, then power the receiver.
2. Wait for the receiver LED to indicate Wi-Fi mode — a fast flash.
3. Connect your laptop to the receiver's Wi-Fi network.
4. Open `10.0.0.1` in a browser and upload the firmware.

{: .note }
> The receiver only falls back to Wi-Fi mode when it powers up and finds no transmitter to bind to. If the radio is on, the receiver will link to it instead and never start its access point. This is why step 1 says to turn the transmitter off.

---

## Part 5: Configure the Link

With all three devices flashed, configure the link. On the radio, press **SYS** and enter **ExpressLRS Tools**:

| Setting | Value |
|---|---|
| Packet Rate | 500 Hz |
| Link Mode | MAVLink |

Then open the **Backpack** submenu:

| Setting | Value |
|---|---|
| Telemetry | WiFi |

{: .note }
> **Link Mode: MAVLink** is what allows the same ELRS link to carry both RC control and the MAVLink telemetry stream that QGroundControl uses. It pairs with the flight controller parameters below, which are set in [Lab 2, Part 8]({% link docs/labs/lab2.md %}):
>
> | Parameter | Value |
> |---|---|
> | `SER_TEL1_BAUD` | 460800 8N1 |
> | `MAV_0_CONFIG` | TELEM1 |
> | `MAV_0_RATE` | 9600 B/s |

### Reduce the Beeper Volume

Do this while you are in the menus — these radios are loud, and a lab full of them is unbearable:

- **SYS → RADIO SETUP**
- Volume: about **30 %**
- Beep Volume: **lowest setting**

---

## Part 6: Set Up the Telemetry Screen

To display vehicle telemetry on the radio:

1. Press **MDL** and scroll to **TELEMETRY**.
2. Select **Discover new sensors** and let it populate while the vehicle is powered and linked.
3. Scroll to **DISPLAY** and configure Screen 1 as shown below.

<a href="{{ '/assets/images/radio/radio-telemetry-display-setup.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/radio-telemetry-display-setup.jpg' | relative_url }}" alt="Radio DISPLAY configuration screen with Screen 1 set to Nums showing RxBt, Curr, Bat percent, Time and Capa fields" />
</a>

**Figure 8.** Telemetry display setup. Screen 1 is set to **Nums**, with `RxBt` (receiver battery voltage), `Curr` (current), `Bat%`, `Time` and `Capa` (consumed capacity).
{: .fs-3 .text-grey-dk-000 }

4. Press **TELE** to bring up the telemetry screen.

<a href="{{ '/assets/images/radio/radio-telemetry-screen.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/radio/radio-telemetry-screen.jpg' | relative_url }}" alt="Radio telemetry screen showing live receiver battery voltage, current draw, elapsed time and consumed capacity" />
</a>

**Figure 9.** The live telemetry screen in flight, showing pack voltage, current draw, elapsed timer and consumed capacity.
{: .fs-3 .text-grey-dk-000 }

{: .sanity_check }
> Power the radio first, then the vehicle. Within a few seconds the receiver LED should go solid and the telemetry screen should show a live pack voltage — roughly 8.4 V for a freshly charged 2S pack, and about 0.8 A of current draw sitting on the ground. If the voltage field stays blank, the link is up but telemetry is not; re-run **Discover new sensors** with the vehicle powered.

---

## Next Steps

With the link established, continue to **[Lab 2]({% link docs/labs/lab2.md %})** to configure PX4: firmware, sensor calibration, radio calibration, flight modes, and actuator assignment.

---

## Troubleshooting Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Receiver never binds | Bind phrase mismatch | Re-flash both ends with an identical, copy-pasted phrase (Part 1) |
| Receiver never binds | ExpressLRS version mismatch | Confirm both TX and RX are on 4.0.1 |
| Receiver does not start its Wi-Fi access point | Transmitter is powered on, so the receiver bound instead | Power the transmitter off, then re-power the receiver (Part 4.2) |
| Cannot upload firmware from the Wi-Fi captive portal (macOS) | Captive portal window blocks file selection | Open `10.0.0.1` in a normal browser window |
| Link is up but QGroundControl shows no telemetry | Link Mode not set to MAVLink | Set **Link Mode: MAVLink** in ExpressLRS Tools (Part 5) |
| Link is up but QGroundControl shows no telemetry | Flight controller serial parameters unset | Set `SER_TEL1_BAUD`, `MAV_0_CONFIG`, `MAV_0_RATE` — see [Lab 2, Part 8]({% link docs/labs/lab2.md %}) |
| Telemetry screen fields stay blank | Sensors never discovered | Re-run **Discover new sensors** with the vehicle powered and linked (Part 6) |
| Radio beeps constantly and loudly | Default volume settings | **SYS → RADIO SETUP**, reduce Volume and Beep Volume (Part 5) |

---

## References

- [ExpressLRS quick start — RadioMaster internal TX](https://www.expresslrs.org/quick-start/transmitters/rm-internal/)
- [ExpressLRS MAVLink configuration](https://www.expresslrs.org/software/mavlink/#configuring-elrs-tx-rx-for-mavlink)
- [ExpressLRS Configurator releases](https://github.com/ExpressLRS/ExpressLRS-Configurator/releases)
