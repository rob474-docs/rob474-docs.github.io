---
layout: default
title: Assembly Instructions
nav_order: 3
last_modified_at: 2026-09-08 12:00:00 -0400
---

# UAS Platform Assembly Instructions
{: .no_toc }

**Platform:** Michigan Robotics UAS learning platform  
**Flight controller:** MicoAir H743V2-AIO (STM32H743 @ 480 MHz, integrated 45 A 4-in-1 ESC), running PX4  
**Battery:** 2S LiPo, 1500 mAh  
**Takeoff weight:** under 250 g

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Overview

The Michigan Robotics UAS learning platform is a low-cost airframe that nonetheless shares its low-level architecture with systems currently designed in industry. It carries an STM32H743 processor running at 480 MHz under PX4 — powerful enough to host a high-level payload for advanced autonomy tasks while keeping a compact footprint and a high safety margin. A takeoff weight below 250 g keeps field operations flexible and largely unrestricted.

This page covers the **physical build**: assembling the frame, wiring power and motors, mounting the avionics stack, and verifying motor direction. Building the vehicle is the first of three stages:

1. **Assembly** — this page.
2. **[Radio Configuration]({% link docs/radio_configuration.md %})** — flashing and binding the ExpressLRS transmitter and receiver.
3. **[Lab 2]({% link docs/labs/lab2.md %})** — PX4 firmware, calibration, and flight-mode configuration.

<a href="{{ '/assets/images/assembly/hero-assembled-closeup.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/hero-assembled-closeup.jpg' | relative_url }}" alt="Close-up of the fully assembled quadrotor showing flight controller, optical flow module and propeller guard" />
</a>

**Figure 1.** The completed vehicle. The flight controller stack sits at the centre of the frame with the optical flow / range module directly below it, and the one-piece propeller guard rings all four rotors. *Click any figure to enlarge.*
{: .fs-3 .text-grey-dk-000 }

<a href="{{ '/assets/images/assembly/hero-in-flight.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/hero-in-flight.jpg' | relative_url }}" alt="The assembled quadrotor hovering above grass with battery mounted in the tray" />
</a>

**Figure 2.** The same vehicle in flight, with a 2S 1500 mAh pack in the battery tray and secured by the battery strap.
{: .fs-3 .text-grey-dk-000 }

---

## Before You Start

{: .warning }
> Read these three rules before you touch the hardware. Each one corresponds to a failure that has already happened on this platform.
>
> 1. **Always remove the propellers when setting up or testing on the bench.** Every bench procedure in this document — spin direction checks, calibration, arming tests — is performed with the props off.
> 2. **Always power the remote controller before the vehicle.** Powering the vehicle first can leave it briefly reading an unbound or stale channel state.
> 3. **Use the correct screw length for the motors.** The motor screws are the short ones. A screw that is too long will reach the windings and destroy the motor.

### Screws Used in This Build

| Fastener | Length | Used for |
|---|---|---|
| M2 (supplied with motor) | 4 mm | Motors to frame arms — **the short screws** |
| M2 | 8 mm | Propeller guard (×4), landing legs (×4), battery case (×4), optical flow module (with spacers) |
| M2 | 12 mm | Flight controller / autopilot, through the rubber dampers |

The two longer lengths follow from the stack-ups they have to clear:

- Optical flow module: board (1.2 mm) + spacer (3.5 mm) = 4.7 mm → **M2×8**
- Flight controller: rubber damper insert length 7.4 mm → **M2×12**

{: .note }
> Sort the screws by length into separate trays before you start. The M2×4 and M2×8 are easy to confuse by eye, and mixing them up is how motors get destroyed.

---

## Bill of Materials

| Qty | Item | Fasteners |
|---|---|---|
| 1 | Frame | — |
| 4 | Motors (7200 KV) | 4× M2×4 mm each, supplied with the motor |
| 1 | Propeller guard | 4× M2×8 mm |
| 4 | Landing legs | 4× M2×8 mm |
| 1 | Battery case / tray | 4× M2×8 mm |
| 1 | Battery strap | — |
| 1 | MicoAir H743V2-AIO flight controller | 4× M2×12 mm |
| 1 | MTF-01 optical flow / range module | 4× M2×8 mm + spacers |
| 1 | RadioMaster XR2 ELRS receiver | — |
| 1 | Low-ESR capacitor (220 µF) | soldered |
| 1 | XT60 power lead | soldered |
| 1 | 2S 1500 mAh LiPo battery | — |
| 4 | Propellers | — |

---

## Part 1: Frame and Motors

<a href="{{ '/assets/images/assembly/frame-and-motors.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/frame-and-motors.jpg' | relative_url }}" alt="The bare frame plate laid out below four brushless motors with their three phase wires" />
</a>

**Figure 3.** Starting parts: the frame plate and the four motors, each with three phase wires.
{: .fs-3 .text-grey-dk-000 }

1. Fasten each motor to its frame arm using **the shorter screws supplied with the motor (M2×4 mm)**.

   {: .warning }
   > Using the longer screws here will drive them into the motor windings and destroy the motor. This is the single most common way to ruin a motor during assembly.

2. Hand-tighten the screws firmly, but not so hard that the frame deforms or cracks around the mounting holes.
3. Route each motor's three phase wires along its arm toward the centre of the frame.

<a href="{{ '/assets/images/assembly/motors-mounted-wire-routing.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/motors-mounted-wire-routing.jpg' | relative_url }}" alt="All four motors bolted to the frame arms with phase wires routed toward the centre" />
</a>

**Figure 4.** All four motors mounted, with the phase wires routed inboard along the arms and gathered at the centre where the flight controller will sit. Leave the wires long for now; trim to length only once you know where each one has to reach.
{: .fs-3 .text-grey-dk-000 }

---

## Part 2: Power Wiring

<a href="{{ '/assets/images/assembly/fc-capacitor-power-lead.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/fc-capacitor-power-lead.jpg' | relative_url }}" alt="Flight controller on the frame with the low-ESR capacitor and XT60 power lead laid out below it" />
</a>

**Figure 5.** The flight controller positioned on the frame, with the low-ESR capacitor and the XT60 power lead ready to be soldered.
{: .fs-3 .text-grey-dk-000 }

Solder the capacitor and the power lead to the flight controller's battery pads:

1. Identify the **BATTERY+** and **BATTERY−** pads on the flight controller (see the port diagram in [Part 5](#part-5-port-and-pin-assignments)).
2. Solder the **low-ESR capacitor** across the battery pads, observing polarity — the marked stripe is the negative leg.
3. Solder the **XT60 power lead** to the same pads: red to **BATTERY+**, black to **BATTERY−**.

{: .warning }
> Reversing the capacitor or the power lead polarity will destroy the flight controller as soon as a battery is connected. Check both before the first power-up.

<a href="{{ '/assets/images/assembly/capacitor-soldered.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/capacitor-soldered.jpg' | relative_url }}" alt="Close-up of the capacitor and red and black power leads soldered to the flight controller battery pads, held in helping-hands clamps" />
</a>

**Figure 6.** The capacitor and power leads soldered to the battery pads. Use helping hands to hold the board — the pads are large but the surrounding components are not, and a slipped iron here ends the build.
{: .fs-3 .text-grey-dk-000 }

{: .note }
> The capacitor suppresses voltage spikes generated by the ESCs switching the motor phases. It is not optional: without it, those spikes couple into the gyroscope and show up as noise in the flight logs, and in the worst case they damage the board.

---

## Part 3: Motor Wiring and Spin Direction Check

1. Solder each motor's three phase wires to one of the four motor pad groups (**M1**–**M4**) on the flight controller. Keep track of which physical arm you connect to which pad group — you will need that mapping when you configure actuator outputs.
2. Keep the wire runs tidy and clear of the propeller discs.

<a href="{{ '/assets/images/assembly/motor-wires-connected.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/motor-wires-connected.jpg' | relative_url }}" alt="Flight controller mounted on the frame with all four sets of motor phase wires soldered to the motor pads" />
</a>

**Figure 7.** All four motors wired to the flight controller. Note the numbered motor pads (1–4) along the edges of the board.
{: .fs-3 .text-grey-dk-000 }

### Verify Spin Direction

{: .warning }
> **No propellers.** Confirm all four propellers are off the vehicle before applying power.

Once all motor wires are connected, power the vehicle and test each motor's direction of rotation using the motor test sliders on the **Actuators** page in QGroundControl (the full procedure is in [Lab 2, Part 10]({% link docs/labs/lab2.md %})).

1. Connect the flight controller to your laptop by USB, and connect the flight battery.
2. Enable the motor test slider and raise **one motor at a time**, slowly.
3. Raise the throttle only as far as you need to see or feel the direction — no further.
4. Note the direction of each motor against the required X-configuration pattern.
5. To reverse a motor that spins the wrong way, **swap any two of its three phase wires**.

<a href="{{ '/assets/images/assembly/spin-direction-test.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/spin-direction-test.jpg' | relative_url }}" alt="Vehicle without propellers on a bench connected to a laptop running QGroundControl and to a LiPo battery for the motor spin test" />
</a>

**Figure 8.** Spin direction test setup: vehicle on the bench with **no propellers**, connected to the laptop over USB and to the flight battery, with the QGroundControl actuator test page open.
{: .fs-3 .text-grey-dk-000 }

{: .sanity_check }
> Doing this now — before the propeller guard, landing legs and battery tray go on — means a phase swap is a two-minute job. Discovering a reversed motor after the vehicle is fully assembled means taking most of it apart again.

---

## Part 4: Vibration Isolation and Mounting the Stack

1. Insert the **rubber dampers** into the flight controller mounting holes. Tweezers make this much easier than fingers.

<a href="{{ '/assets/images/assembly/rubber-dampers.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/rubber-dampers.jpg' | relative_url }}" alt="Rubber vibration damping grommets being inserted into the flight controller mounting holes with tweezers" />
</a>

**Figure 9.** Rubber dampers being fitted to the flight controller mounting holes. These isolate the IMU from motor and frame vibration; skipping them produces noisy gyro data and poor attitude estimation.
{: .fs-3 .text-grey-dk-000 }

2. Mount the hardware to the frame:
   - **Optical flow / range module:** M2×8 mm screws with spacers.
   - **Flight controller:** M2×12 mm screws through the rubber dampers.
3. Dress the wiring so nothing is under tension and nothing can reach a propeller disc.

<a href="{{ '/assets/images/assembly/stack-assembled.jpg' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/stack-assembled.jpg' | relative_url }}" alt="Completed avionics stack mounted on the frame with the optical flow module ahead of the flight controller and the XT60 lead exiting to the rear" />
</a>

**Figure 10.** The completed stack. The optical flow / range module is mounted forward of the flight controller with its sensors facing down, and the XT60 lead exits to the rear toward the battery tray.
{: .fs-3 .text-grey-dk-000 }

{: .note }
> Mount the flow module so its two apertures point **straight down** with an unobstructed view of the ground. Anything in the field of view — a wire, a landing leg, a strap end — degrades the flow quality reading and, with it, position hold.

4. Fit the propeller guard (4× M2×8 mm), landing legs (4× M2×8 mm), and battery tray (4× M2×8 mm).

---

## Part 5: Port and Pin Assignments

**Reference:** [MicoAir H743V2-AIO product page](https://micoair.com/flightcontroller_micoair743v2_aio_45a/)

<a href="{{ '/assets/images/assembly/micoair-h743v2-ports.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/micoair-h743v2-ports.png' | relative_url }}" alt="Port diagram of the MicoAir H743V2-AIO flight controller showing motor pads, battery pads, UART headers, BOOT button, USB-C and TF card slot on both board faces" />
</a>

**Figure 11.** MicoAir H743V2-AIO port assignments, both faces. Note the locations of the **BOOT button** and **USB-C** connector — you will need both when flashing the PX4 bootloader — along with the four motor pads (M1–M4) and the battery pads.
{: .fs-3 .text-grey-dk-000 }

Peripheral connections used on this build:

| Peripheral | Port | PX4 designation |
|---|---|---|
| ELRS receiver | UART1 | TELEM1 |
| MTF-01 optical flow / range module | UART4 | TELEM2 |

These assignments determine the `MAV_0_CONFIG` and `MAV_1_CONFIG` parameter values in [Appendix A](#appendix-a-platform-parameter-reference). If you wire a peripheral to a different port, those parameters must change to match.

---

## Next Steps

The vehicle is now built. Continue with:

1. **[Radio Configuration]({% link docs/radio_configuration.md %})** — flash and bind the ExpressLRS transmitter, backpack, and receiver, and set up the telemetry screen.
2. **[Lab 2]({% link docs/labs/lab2.md %})** — QGroundControl, the PX4 toolchain, firmware, sensor calibration, flight modes, actuator assignment, power configuration, and optical flow.

Use the values in [Appendix A](#appendix-a-platform-parameter-reference) for the settings specific to this platform.

---

## Appendix A: Platform Parameter Reference

These are the parameter values recorded from a known-flying reference build. Use them alongside the procedures in Lab 2.

| Parameter | Value | Reason |
|---|---|---|
| `SYS_HAS_MAG` | 0 (None) | No magnetometer onboard |
| `MAV_TYPE` | 2 (Generic Quadrotor) | Airframe type |
| `SENS_BOARD_ROT` | 8 (Roll 180°) | Flight controller mounting orientation |
| `BAT1_N_CELLS` | 2 | 2S battery |
| `BAT1_A_PER_V` | 12.14 | Per board specs — **see note below** |
| `BAT1_V_DIV` | 21.12 | Per board specs — **see note below** |
| `MAV_0_CONFIG` | 101 (TELEM 1) | MAVLink over the ELRS link |
| `SER_TEL1_BAUD` | 460800 | ELRS MAVLink baud rate |
| `MAV_0_RATE` | 9600 B/s | Rate limit to fit ELRS bandwidth |
| `PWM_MAIN_TOIM0` | −6 (BDShot600) | Motor output protocol — **see note below** |

Battery voltage and cell limits from the reference build:

| Setting | Value |
|---|---|
| Source | Power Module / Analog |
| Number of cells (in series) | 2 |
| Full voltage (per cell) | 4.20 V |
| Empty voltage (per cell) | 3.20 V |

### Motor Output Mapping

The reference build's actuator assignment, which follows from which motor was soldered to which pad:

| Output | Assigned function |
|---|---|
| MAIN 1 | Motor 3 |
| MAIN 2 | Motor 4 |
| MAIN 3 | Motor 1 |
| MAIN 4 | Motor 2 |

<a href="{{ '/assets/images/assembly/qgc-actuator-outputs.png' | relative_url }}" class="image-link">
  <img src="{{ '/assets/images/assembly/qgc-actuator-outputs.png' | relative_url }}" alt="QGroundControl actuator outputs panel showing MAIN 1 to 4 set to DShot600 and assigned to motors 3, 4, 1 and 2" />
</a>

**Figure 12.** Actuator output assignment in QGroundControl, with **MAIN 1–4** set to **DShot600**.
{: .fs-3 .text-grey-dk-000 }

{: .warning }
> This mapping is **not** identity, and it is specific to how the reference vehicle was wired. Do not copy it blindly. Determine your own vehicle's mapping using the motor test described in [Part 3](#verify-spin-direction), then assign outputs to match.

{: .important }
> **Three values in this appendix need confirmation before you rely on them.**
>
> - `BAT1_A_PER_V` and `BAT1_V_DIV` are listed here as 12.14 and 21.12, but the reference build's Power configuration screen shows **4.14** and **1.12** for the same two settings. The pairs differ by a leading digit, which suggests a transcription error somewhere. Verify the vehicle's reported pack voltage and current against a multimeter and a current meter before trusting either set.
> - `PWM_MAIN_TOIM0` is recorded as BDShot600 (bidirectional DShot, which returns RPM telemetry), while the actuator screen shows plain **DShot600**. Confirm which the flyable configuration actually uses, and check the parameter name against the firmware you are running.

---

## Appendix B: Reference Build Data

Measurements from the first test flight of the reference vehicle (M-Air net, June 2026), useful as a baseline when checking your own build:

| Quantity | Value |
|---|---|
| Flight time achieved | 8 min 03 s |
| Vehicle weight without battery | 106.5 g |
| Battery weight (2S 1500 mAh, XT60) | 85.6 g |
| **Total weight** | **192.1 g** |
| Frame alone (no battery tray or adapter) | 84.3 g |
| Battery tray (first prototype) | 17.5 g |
| XT30→XT60 adapter | 4.3 g |
| Capacity consumed | 835 mAh → approx. 14 min projected endurance |
| Current draw, sitting on the ground | approx. 0.8 A |
| Motors | 7200 KV |

Observations from that flight:

- Motors and electronics were only marginally warm after the flight, never hot.
- The vehicle was stable in light wind, though not to the standard of a commercial GPS-stabilised platform.
- Measured efficiency worked out to roughly 5 W/g; with a 2S 4 Ah 18650 pack that projects to about 40 minutes of endurance.

---

## References

- [MicoAir H743V2-AIO flight controller](https://micoair.com/flightcontroller_micoair743v2_aio_45a/)
- [MicoAir MTF-01 optical flow / range sensor](https://micoair.com/optical_range_sensor_mtf-01/)
- [PX4 flight log analysis](https://logs.px4.io/) — upload logs downloaded via **Analyze Tools → MAVLink Log**
- [Sample flight log](https://logs.px4.io/plot_app?log=3ec9425e-2e2b-48c8-8503-fd58eade03df) — brief indoor flight in position hold, maiden flight with optical flow
