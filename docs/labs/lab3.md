---
layout: default
title: Lab 3
nav_order: 3
parent: Labs
last_modified_at: 2026-09-17 12:00:00 -0400
---

# Lab 3: Writing Your Own Flight Controller

**Course:** Uncrewed Aerial Systems  
**Prerequisites:** [Lab 2]({% link docs/labs/lab2.md %}) (PX4 Setup and Tuning): working PX4 build toolchain, calibrated airframe, ELRS WiFi telemetry, optical flow configured  
**Estimated Time:** 4 hours  
**Hardware Required:**
- Custom quadrotor with MicoAir743v2 AIO flight controller running PX4
- USB-C cable
- RC transmitter and receiver (bound, MAVLink Link Mode configured per [Radio Configuration]({% link docs/radio_configuration.md %}) and [Lab 2 Part 8]({% link docs/labs/lab2.md %}#part-8-radio-and-mavlink-setup))
- Battery
- **Propellers removed** for all bench work in Parts 5–8
- Personal laptop with the PX4 toolchain from [Lab 2 Part 2]({% link docs/labs/lab2.md %}#part-2-set-up-the-px4-development-toolchain)

---

## Overview

In Lab 2 you configured PX4's built-in flight controller and let its autotune pick your gains. In this lab you replace that controller with your own.

You will add a new module to the PX4 firmware, wire it into the mode switch on your transmitter, and implement the inner three loops of a cascaded multicopter controller from scratch: **rate, attitude, and altitude**. Your module reads the same estimator output PX4's own controller reads, and produces the same torque and thrust commands that drive the motors.

By the end of this lab you will have flown a quadrotor on control code you wrote yourself.

> **Scope:** the outer **velocity** and **position** loops are Lab 4. Their template files ship with this module so you can see where they fit in the architecture, but they stay stubbed for now. You will not enable them, and the module works fine without them. Lab 4 adds motion capture, which is what makes a position loop worth closing on this platform.

> **On the relationship to PX4's controller:** you are not modifying `mc_rate_control` or `mc_att_control`. You are writing a parallel module that takes over when you flip a switch, and stays silent otherwise. PX4's controller remains available as a fallback, which matters when your code does not work, and at some point during this lab it will not work.

---

## Part 1: Control Architecture

Before writing code, understand the shape of what you are building.

### 1.1 The Cascade

A multirotor controller is a stack of nested loops. Each loop consumes a setpoint from the loop above it and produces a setpoint for the loop below:

```
  pilot / mission
        |
        v
  POSITION  ---- where should I be?        -> outputs desired velocity   [LAB 4]
        |
        v
  VELOCITY  ---- how fast should I move?   -> outputs desired tilt angle [LAB 4]
        |
        v
  ATTITUDE  ---- how should I be oriented? -> outputs desired body rates  <- Part 7
        |
        v
  RATE      ---- how fast should I rotate? -> outputs torque commands     <- Part 6
        |
        v
  MIXER     ---- what should each motor do?-> outputs per-motor thrust    (PX4's control allocator)
        |
        v
  ESCs / motors

  ALTITUDE  ---- how high should I be?     -> outputs collective thrust   <- Part 8
                 (independent of the roll/pitch/yaw cascade above)
```

This lab builds the bottom of the stack. Lab 4 adds the top.

Two properties of this structure drive everything else in the lab:

**Inner loops run faster and matter more.** The rate loop runs at gyro rate (667 Hz on this hardware) because rotational dynamics are fast. An untuned rate loop diverges in a fraction of a second. The position loop can run at tens of Hz because position changes slowly. If your rate loop is wrong, no amount of correctness in the outer loops will save you.

**Each loop assumes the one below it works.** The attitude loop commands a body rate and trusts that the rate loop achieves it. When you debug, always debug from the inside out. A "position hold doesn't work" bug is usually a rate or attitude bug.

This is why you will implement and test the loops in that order, and why `UAS_LOOP_EN` lets you enable them one at a time.

### 1.2 What Your Module Sees and Produces

**Inputs**, from EKF2, the same estimator PX4's own controller uses:

| Quantity | Source | Units | Notes |
|---|---|---|---|
| roll, pitch, yaw | `vehicle_attitude` | rad | Converted from quaternion for you |
| roll/pitch/yaw rate | `vehicle_angular_velocity` | rad/s | Gyro-driven, lowest latency signal you have |
| vx, vy, vz | `vehicle_local_position` | m/s | NED frame, fused optical flow. You use `vz` for altitude damping in Part 8; the horizontal pair is Lab 4 |
| x, y, z | `vehicle_local_position` | m | NED frame, dead-reckoned. Lab 4 |
| altitude | `vehicle_local_position` | m | Rangefinder preferred, baro/EKF2 fallback, **positive up** |

**Output**: normalized torque and thrust.

| Quantity | Range | Meaning |
|---|---|---|
| roll/pitch/yaw torque | [-1, 1] | Normalized, not physical N·m |
| thrust | [0, 1] | Collective upward thrust |

PX4's control allocator converts these into the four per-motor commands using the geometry you entered in [Lab 2 Part 10]({% link docs/labs/lab2.md %}#part-10-actuator-configuration). That mixing step stays with PX4 in this lab; your controller's job ends at torque and thrust.

### 1.3 Frame Conventions

Get these wrong and your vehicle will fly into a wall in a confidently incorrect direction.

- **Body frame (FRD):** x = forward, y = right, z = **down**
- **Local frame (NED):** x = north, y = east, z = **down**

The consequence that trips up nearly everyone: **z is positive downward.** A vehicle 2 m above the ground has `z = -2.0`, and `vz` is *positive while descending*. The `altitude` field in `VehicleState` is flipped to positive-up for convenience; nothing else is.

---

## Part 2: Install the Module Template

1. **Make a group for your team.** On [the course GitLab server](https://gitlab.eecs.umich.edu), go to **Groups → New group** and give it any name you like. Then, under **Manage → Members**, invite your teammates as **Developer**, and invite the instructor and your GSI by their **uniqname email addresses** as **Reporter**. That is their GitLab username, so they will appear as you type. Without that we cannot read your code to grade it. One group per team, made once, used for the rest of the term.

2. **Fork the template into your group.** Open [`rob474-f26/uas_control`](https://gitlab.eecs.umich.edu/rob474-f26/uas_control), click **Fork**, and select your team's group as the namespace. You now have your own copy at `https://gitlab.eecs.umich.edu/<your-group>/uas_control`, which is the repository you will work in and hand in. The course copy stays read-only: you cannot push to it, and you should not try.

3. **Clone your fork** into your PX4 source tree:
   ```bash
   cd ~/uas/PX4-Autopilot/src/modules
   git clone https://gitlab.eecs.umich.edu/<your-group>/uas_control.git
   ```
   Substitute your group's path. Check you cloned the right one before you start writing code:
   ```bash
   cd uas_control && git remote -v
   ```
   `origin` must point at **your group**, not at `rob474-f26`.

   > The module is its own git repository, separate from PX4-Autopilot. Commit your work to it as you go. `git status` inside `src/modules/uas_control` shows only your files, not the rest of the PX4 tree. Push to your fork regularly. Your source code deliverable at the end of this lab is this repository.

   > **If the template is updated during the term**, pull the change into your fork rather than re-cloning: add the course copy as a second remote once, `git remote add upstream https://gitlab.eecs.umich.edu/rob474-f26/uas_control.git`, then `git pull upstream main` when we tell you there is something to take. Your commits stay where they are.

4. Enable the module for your board. Open the board config:
   ```bash
   ~/uas/PX4-Autopilot/boards/micoair/h743-v2/default.px4board
   ```
   and add this line (keep the file alphabetically sorted if it already is):
   ```
   CONFIG_MODULES_UAS_CONTROL=y
   ```

5. Start the module at boot. Open the startup script:
   ```bash
   ~/uas/PX4-Autopilot/ROMFS/px4fmu_common/init.d/rc.mc_apps
   ```
   and add near the other controller starts:
   ```
   uas_control start
   ```

   > If you prefer to start it manually each session instead, skip this step and run `uas_control start` from the MAVLink Console. Starting at boot is more convenient; starting manually makes it obvious when your module is and is not running.

6. Verify it builds:
   ```bash
   cd ~/uas/PX4-Autopilot
   make micoair_h743-v2_default
   ```
   The template compiles as-is, with every controller stubbed but syntactically complete. **If it does not build before you have written any code, fix that before continuing**; you do not want to be debugging build configuration and control math at the same time.

7. Flash it. Nothing in this lab works until the module is actually on the board:
   ```bash
   make micoair_h743-v2_default upload
   ```
   Plug in USB when it says `Waiting for bootloader`, exactly as in [Lab 2 Part 4]({% link docs/labs/lab2.md %}#part-4-build-and-flash-px4-firmware-from-source). Every time you change controller code, this is the command that gets it onto the vehicle. An incremental rebuild plus flash takes under a minute.

   > **Before your first flash of this lab, back up your parameters:** QGC → **Vehicle Setup → Parameters → Tools → Save to file**. On this board the parameters live on the SD card, not in the flight controller, and a bad card or an interrupted write can silently reset everything you configured in Lab 2. A saved file restores it in one click (**Tools → Load from file**).

> **A note on PX4 API drift:** this template targets PX4 v1.17.0, the release pinned in Lab 2. uORB message field names do change between PX4 releases. If you hit a compile error like `'struct vehicle_local_position_s' has no member named 'xy_valid'`, check the actual field names in `~/uas/PX4-Autopilot/msg/` for your version and adjust. This is normal work when building on someone else's flight stack.

---

## Part 3: Tour the Template

```
src/modules/uas_control/
├── UasControl.cpp/.hpp     <- module scaffolding. DO NOT EDIT.
├── VehicleState.hpp            <- state/setpoint structs. Read this first.
├── RateController.cpp/.hpp     <- YOUR WORK (Part 6)
├── AttitudeController.cpp/.hpp <- YOUR WORK (Part 7)
├── AltitudeController.cpp/.hpp <- YOUR WORK (Part 8)
├── VelocityController.cpp/.hpp <- LAB 4. Leave stubbed.
├── PositionController.cpp/.hpp <- LAB 4. Leave stubbed.
├── uas_control_params.c    <- tunable parameters, exposed to QGC
├── CMakeLists.txt
└── Kconfig
```

Read `VehicleState.hpp` first. It defines every variable you have access to and documents the units and frames. Then read `UasControl.cpp` to see how the pieces are called, even though you will not edit it.

**Parameters.** Every gain is a PX4 parameter with the `UAS_` prefix, visible in QGC under **Vehicle Setup → Parameters**. You can retune live over the ELRS Backpack WiFi link (set up in [Radio Configuration, Part 5]({% link docs/radio_configuration.md %}#part-5-configure-the-link)) without reflashing. Reflashing takes minutes; changing a parameter takes seconds. Use the parameters.

**`UAS_LOOP_EN`** is a bitmask controlling which loops run. Build up from the inside out:

| Value | Loops active | |
|---|---|---|
| 1 | Rate only | Part 6 |
| 3 | Rate + attitude | Part 7 |
| 7 | + altitude | Part 8, **the target for this lab** |
| 15 | + velocity | Lab 4 |
| 31 | Full cascade | Lab 4 |

> Do not set `UAS_LOOP_EN` above 7 in this lab. The velocity and position controllers are stubbed and return zero, so enabling them commands level flight and zero velocity regardless of your stick input. The vehicle will refuse to translate and you will spend an hour debugging a controller you have not written yet.

---

## Part 4: Assign Your Controller to the Mode Switch

Your module activates on one `nav_state`: **Offboard** (`UAS_MODE_SLOT` = 14).

1. In QGC, go to **Vehicle Setup → Flight Modes**.
2. Using the mode channel you identified in [Lab 2 Part 8.3]({% link docs/labs/lab2.md %}#83-identify-switch-channels), set one switch position to **Offboard**.
3. Keep the other two positions as **Stabilized** and **Altitude**. These are your escape hatches, and unlike a naive setup they genuinely work; see 4.1 for why.
4. Confirm your **Kill Switch** ([Lab 2 Part 9]({% link docs/labs/lab2.md %}#part-9-flight-modes)) still works. It cuts motor output below the flight-control layer, so it works even if your code is in a tight loop doing something catastrophic.

> **Know your two abort paths before you arm anything:** flip the mode switch (returns to PX4's controller), or hit the kill switch (cuts all motor output). Practice reaching both without looking.

### 4.1 Why Offboard, and How PX4 Gets Out of Your Way

Your module publishes to `vehicle_torque_setpoint` and `vehicle_thrust_setpoint`. So does PX4's own `mc_rate_control`. Two publishers writing the same topic means the control allocator reads whichever arrived most recently, an interleaved mixture of two controllers, which is worse than either one alone.

Naively you would fix this by stopping PX4's controllers. That works, but it also disables Stabilized and Altitude, so you would be flying with no way back except a reboot you cannot perform in the air.

Offboard mode solves it properly. The `offboard_control_mode` message is how a controller outside the flight stack declares **which level** of the control stack it intends to command. Your module publishes it continuously with `thrust_and_torque` set:

```cpp
ocm.thrust_and_torque = true;   // everything else false
```

That tells PX4 to bypass its own position, attitude, and rate loops. `mc_att_control` and `mc_rate_control` stand down on their own while Offboard is active, and resume the instant you switch out. Nothing to stop, nothing to restart, and Stabilized and Altitude stay live the whole time.

Two consequences worth understanding:

**The heartbeat is a watchdog.** PX4 requires `offboard_control_mode` at better than 2 Hz. If your module crashes or hangs, the signal stops and PX4 fires the offboard-loss failsafe automatically. Note precisely what that covers: it catches a **dead** module, not a module that is alive and computing wrong answers. Your own code being confidently wrong is still your problem, and that is what the kill switch is for.

**The heartbeat must precede mode entry.** Offboard cannot be entered unless the signal is already streaming, which is why the module publishes it on every iteration whether or not it currently has control. Publishing the heartbeat does not take control; it only advertises that a thrust/torque source is alive and ready.

> **Semantic note:** Offboard normally means "a companion computer is flying this." Here an onboard module is filling that role. That is unusual but entirely legitimate, because `offboard_control_mode` is a uORB topic, and Commander does not care whether it arrived from a MAVLink link, a ROS 2 node, or a module running on the same chip.

---

## Part 5: Bench Test Setup

> ### PROPELLERS OFF
> Every step in Parts 5 through 9 is done with propellers removed. You will be arming the vehicle and commanding motor outputs from code that does not work yet. Do not skip this.

### 5.1 Offboard Failsafe Configuration

Because your controller runs as an Offboard source, configure what PX4 does if that source dies. In **Vehicle Setup → Parameters**:

| Parameter | Suggested value | Reason |
|---|---|---|
| `COM_OF_LOSS_T` | 0.5 s | How long PX4 waits after the heartbeat stops before declaring offboard lost. Short, because a dead controller is an emergency |
| `COM_OBL_RC_ACT` | Altitude (or Land) | What to do on offboard loss. **Not** Return to Launch, because this platform has no GPS |
| `COM_DISARM_PRFLT` | -1 *(bench only)* | PX4 auto-disarms 10 s after arming if you have not taken off. On a bench that kills every test. **Restore to 10 before Part 9.** |

> Verify these parameter names against your PX4 version; offboard-loss parameters have been renamed across releases. Search `COM_OF` and `COM_OBL` in the QGC parameter list to find the current equivalents.

Then confirm the module is running:

```
uas_control status
```

If it is not running, start it:

```
uas_control start
```

> Two bench-test quirks worth knowing: **"Disarming denied: not landed"**. Tilting an armed vehicle by hand can convince the land detector it is airborne, and the disarm switch is refused until it decides you have landed (a second or two). The kill switch does not care. And **the heartbeat stops the moment the module crashes**, so if `uas_control status` ever says the module is not running after a mode switch, that is your code, not PX4.

> **You do not need to stop `mc_rate_control` or `mc_att_control`.** They stand down on their own whenever Offboard is active, which is the whole point of Part 4.1. If you find yourself typing `mc_rate_control stop`, something else is wrong; go back and check `UAS_MODE_SLOT` is 14 and that the heartbeat is publishing.

### 5.2 Watching Your Controller Work

Three tools, in increasing order of usefulness:

**Module status**, a formatted snapshot of state and output:
```
uas_control status
```

**Live topic values**, raw uORB, updating:
```
listener vehicle_torque_setpoint
listener actuator_motors
listener vehicle_angular_velocity
listener offboard_control_mode
```
That last one is your first stop whenever Offboard mode refuses to engage: confirm it is publishing and that `thrust_and_torque` reads true.

**Logged flight data**. For anything involving oscillation or transients, log and plot. Real-time console output will not show you a 30 Hz oscillation. Download the `.ulg` from **Analyze Tools → Log Download** and open it in [PlotJuggler](https://plotjuggler.io/) or [Flight Review](https://logs.px4.io/). While your controller is active the module publishes its setpoints on PX4's standard topics (`vehicle_rates_setpoint`, `vehicle_attitude_setpoint`, and `vehicle_local_position_setpoint.z`), so "commanded vs. measured" plots come straight out of either tool with no custom tooling: rate setpoint against `vehicle_angular_velocity`, attitude setpoint against `vehicle_attitude`. Those plots are the deliverables for Parts 6–8.

### 5.3 First Smoke Test

With props off and `UAS_LOOP_EN = 1`:

1. Confirm the heartbeat is alive: `listener offboard_control_mode` should show fresh messages with `thrust_and_torque: True`.
2. Arm the vehicle, then flip the mode switch to **Offboard**.
3. QGC's message bar (and the MAVLink Console) should show `uas_control ENGAGED`.
4. Run `uas_control status`. Confirm attitude, rate, altitude, and velocity fields show plausible live values that respond when you tilt the airframe by hand.
5. Motors should sit at idle and not respond to stick input, because the rate controller is still stubbed and returns zero torque.
6. **Test the handoff.** Flip back to Stabilized. You should see `uas_control released`, and PX4's controller should take over. Verify by tilting the airframe and watching `listener actuator_motors` respond. Flip to Offboard and back a few times. This is the abort path you will rely on for the rest of the lab; confirm it works before you need it.

If you see live state values, `active: YES` in Offboard, and clean handoff in both directions, the plumbing works and everything from here is control math.

---

## Part 6: Rate Controller

**File:** `RateController.cpp`

The innermost loop. Takes desired body rates, produces normalized torques.

### 6.1 Implement

Implement all three axes in `RateController::update()`. The header documents what state you need to declare. Points to think through rather than guess at:

- **Anti-windup.** Clamp the integral to `_integral_limit`. Without it, holding a stick against a limit accumulates integral state that takes seconds to unwind, and the vehicle keeps rotating after you center the stick.
- **Derivative kick.** Differentiating the *error* spikes whenever the setpoint jumps. Differentiating the *measurement* does not. Pick one deliberately.
- **Derivative noise.** The gyro updates at 667 Hz and a raw finite difference of it is mostly noise, which `_kd` amplifies straight into the motors. Low-pass the derivative with a first-order filter at roughly a 30 Hz cutoff, as covered in lecture, before you multiply by the gain.
- **The shipped gains are P-only.** `UAS_RAT_RP_I` and `UAS_RAT_RP_D` default to zero. With P alone the vehicle is stable but visibly buzzy; adding D and then I is your tuning exercise in 6.4, not something to do blind at the desk.
- **Yaw is different.** Yaw torque comes from motor drag, not thrust differential, so it has far less authority than roll and pitch. It needs its own gains, which is why `_kp_yaw` is separate.
- **`reset()` matters.** It runs on disarm and mode entry. Stale integral state produces a lurch on re-arm.

### 6.2 Bench Test

Props off. `UAS_LOOP_EN = 1`. Arm and flip to Offboard. The stock controllers stand down on their own (Part 4.1); do not stop them.

1. `listener vehicle_torque_setpoint` and move the sticks. Torque values should respond in the right direction and magnitude.
2. **Verify sign by hand.** Hold the airframe and rotate it. Your controller should command torque *opposing* the rotation you impose. If it commands torque in the same direction, your sign is inverted. That is positive feedback and it will flip the vehicle instantly on a real flight.
3. `listener actuator_motors` and confirm individual motors respond differentially to stick input rather than all moving together.

### 6.3 Deliverable

A plot from a logged bench run showing commanded rate vs. measured rate for one axis while you disturb the airframe by hand, with the tracking error visible.

### 6.4 Closing the Loop on the Test Stand

The props-off checks above prove the *sign* of your loop. They cannot tell you whether it *tracks*. For that the loop has to move the airframe, which means propellers, which means the vehicle must be constrained. The lab has a single-axis test stand for exactly this: the airframe pivots freely about one axis (roll or pitch, depending on how you mount it) and is held rigidly in the others. Use it with the instructor present.

1. Mount the vehicle, props on, and **unplug the USB cable**. A tether drags on the free axis and completely corrupts the result. With a cable attached, one axis of this airframe needed five times the torque of the other and could not follow the stick. Use the ELRS wireless link for telemetry; plug USB back in only when disarmed, to pull the log.
2. While on the stand set `EKF2_OF_CTRL = 0`. The optical-flow velocity check will otherwise refuse to arm while the airframe is moving. **Restore to 1 before Part 8**, because the altitude loop and any free flight depend on it.
3. `UAS_LOOP_EN = 1`. Arm in Stabilized, bring the throttle to roughly hover (40–50 %), flip to Offboard. **Keep the throttle there until you are done.** Torque authority is proportional to thrust: at zero throttle the controller can request whatever it likes and nothing happens, and the vehicle becomes a pendulum on the stand.
4. Step the stick and hold; release; repeat in both directions. Then pull the log and plot rate setpoint against gyro. You are looking at rise time, overshoot, and whether the measured rate settles *on* the setpoint (integral) or just near it.
5. Tune here, not in the air, one gain at a time, logging every run. The shipped P-only gains should give a light, visible oscillation. Add `UAS_RAT_RP_D` until it is gone (too much D and a faster, rougher buzz from gyro noise appears instead, which indicates the filter cutoff). Then a little `UAS_RAT_RP_I` for the last bit of steady-state error; watch for the slow wander that means too much. Parameters change live; nothing needs a reflash.

Roll and pitch share gains because the airframe is symmetric, and on the stand with the cable off they should look nearly identical. If one axis needs very different gains from the other, suspect the mounting before the airframe.

---

## Part 7: Attitude Controller

**File:** `AttitudeController.cpp`

Takes desired angles, produces desired body rates for your rate loop.

### 7.1 Implement

Usually P-only. An integrator here fights the one in the rate loop and produces slow oscillation that is genuinely hard to diagnose. Start proportional; add more only with justification.

- **Wrap yaw error to [-π, π].** Use the provided `wrapPi()`. Without it, commanding -179° from +179° spins the vehicle 358° the long way instead of 2° the short way.
- **Clamp the requested rate.** An unclamped angle error commands a rate the inner loop cannot reach, the rate integrator winds up chasing it, and recovery is violent.

### 7.2 Bench Test

`UAS_LOOP_EN = 3`. Props off. Arm.

1. Tilt the airframe by hand and release. Commanded torque should act to restore level.
2. Stick deflection should map to a tilt *angle* now, not a rate. Hold the stick at half deflection and the commanded angle should hold steady rather than continuously integrating.
3. Yaw the airframe through ±180° by hand and confirm the yaw rate command does not jump when it crosses the wrap boundary. This is the wrapPi check.

4. **On the stand** (Part 6.4 setup, throttle near hover): the vehicle should hold level with sticks centered, hold a steady ~10° at half stick, and return to level without oscillating when you push the arm ~20° and let go.

### 7.3 Deliverable

A plot of commanded vs. achieved attitude for roll during hand disturbance, plus a short explanation of the gain relationship you found between your attitude P gain and your rate P gain.

---

## Part 8: Altitude Controller

**File:** `AltitudeController.cpp`

Independent of the roll/pitch/yaw cascade. Takes a height target, produces collective thrust.

### 8.1 Measure Hover Thrust First

Before writing the loop, measure `UAS_HOVER_THR`. Hover in **Stabilized** for ten seconds or so, then read PX4's hover-thrust estimator out of the log: the `hover_thrust_estimate.hover_thrust` topic in Flight Review or PlotJuggler settles on a number within a few seconds of lift-off (0.49 on the instructor's airframe; yours will differ with battery and build). **Do not guess.** A controller whose output is zero at zero error commands zero thrust and the vehicle drops. The hover feedforward is what makes this loop work at all.

### 8.2 Implement

- **Feedforward is the trick:** `thrust = hover_thrust + PID(altitude_error)`.
- **Use `state.vz` for damping** rather than differentiating altitude. It is already filtered by EKF2 and far less noisy than a differentiated rangefinder signal.
- **Watch the sign on `vz`.** It is NED, so positive means *descending*, while `altitude` is positive up. Getting this backwards converts your damping term into positive feedback. Reason it through on paper.
- **Clamp thrust to something like [0.1, 0.9]**, not the full range. Commanding zero thrust in flight is a free fall with no control authority.
- **Know where `state.altitude` comes from.** It is height above the floor from the downward rangefinder, tilt-compensated and steady to a centimetre when the vehicle is sitting on its legs, fused with the EKF's vertical velocity in a small complementary filter: `vz` carries the estimate between samples and through any stretch where the rangefinder stops making sense, the rangefinder trims it back. It is *not* the EKF's `z`: on this airframe that has the barometer in it, sits a metre off the ground, and jumps on touchdown. Why the fusion matters: the rangefinder reads to 8 m indoors but only to about 1.5 m over sunlit grass, and past its reach the readings flatten while the vehicle keeps going. An altitude loop that trusted them absolutely climbed the instructor's vehicle to 5 m. The module rejects a sample that disagrees with the prediction by more than `UAS_RNG_GATE`, and `UAS_MAX_ALT` (1.5 m) caps the target your stick can walk up to. If `altitude_valid` is false the module has no trustworthy height and you must hold hover thrust rather than close the loop.

### 8.3 Landing

An altitude hold has a second job you will discover the first time you try to land in your own mode: **it does not know how to stop.** With the throttle stick at the bottom the module walks the altitude target down to 0 m and your loop dutifully holds the vehicle *on the floor* at roughly hover thrust. The land detector never sees a landing, and when you flip the arm switch PX4 answers `Disarming denied: not landed`. Your kill switch still works, but that is not a landing.

The module tells you the pilot's intent: `sp.land` is true while the throttle stick is held at the bottom, and `setIdleThrust()` gives you PX4's motor-idle thrust (`MPC_MANTHR_MIN`). What to do about it is yours to write, in `update()`, after the normal loop:

1. **Decide you are down.** `sp.land` *and* `state.altitude` below a small threshold of about 0.15 m; the rangefinder reads 0.05–0.10 m on the legs. Above that, a bottomed stick just means "descend" and the loop already handles it.
2. **Hold the decision.** Once landing, stay landing until `sp.land` goes false, *whatever the altitude reads*. Without this hysteresis a height reading flickering around the threshold toggles the thrust between idle and hover a couple of times a second and the vehicle hops across the floor. (The instructor's airframe did exactly this.)
3. **Ramp, don't cut.** Take the thrust from wherever it is to the idle value over about a second. Cutting it drops the vehicle the last 10–15 cm onto its legs.
4. **Clear your integrator** while landing, or the next take-off inherits a trim learned while pinned to the floor.

You will know it works when you can land in Offboard, see `Landing detected` in QGC, and disarm with the switch, without reaching for Stabilized or the kill switch.

> **Taking off is not your job.** The module handles it: on the ground in altitude mode the motors idle and your controller is not called; pushing the throttle above 60 % flies an automatic take-off, stepping thrust to 1.12 × `UAS_HOVER_THR` until the rangefinder shows half of `UAS_TKO_ALT`, then hands your loop the vehicle with `UAS_TKO_ALT` (0.33 m) as its target and the integrator reset. **Centre the throttle**: it is ignored until you do, then it works as in flight. When PX4's land detector sees your landing complete, the module is back on the ground and the next throttle-up takes off again. Watch for `take-off to 0.33 m` and `airborne, target 0.33 m` in QGC. Why the rangefinder and not the EKF's `vz` here: at lift-off the barometer sits in the prop wash and `vz` has read 2 m/s *down* with the vehicle 10 cm up. The module also learns the airframe's steady roll/pitch torque in hover (`UAS_TRIM_ROLL`, `UAS_TRIM_PITCH`, saved on landing) and feeds it forward, so a take-off does not have to wait for your rate integrator to re-learn the trim. Without it the instructor's airframe, whose CG sits a little aft, left the ground 12° nose-up every time.

### 8.4 Bench Test

Props off, `UAS_LOOP_EN = 7`. You cannot test altitude hold on a bench, so verify what you can:

1. Confirm `altitude_valid` is true and `altitude` tracks height as you raise and lower the airframe by hand (this exercises the [Lab 2 Part 12]({% link docs/labs/lab2.md %}#part-12-configure-optical-flow) flow/rangefinder chain).
2. Raise the airframe above its setpoint and confirm commanded thrust *decreases*; lower it and confirm thrust *increases*.
3. Confirm thrust sits near `UAS_HOVER_THR` at zero error.
4. **Landing logic, props off:** set the vehicle on the bench, arm in Offboard, throttle stick at the bottom. `uas_control status` should show the thrust ramping to idle within a second and staying there. Lift the vehicle a few centimetres by hand. It must *stay* at idle (hysteresis). Raise the stick and the loop should resume.

### 8.5 Deliverable

Log plot showing altitude setpoint, measured altitude, and commanded thrust during the hand-raise test, with the sign relationship clearly visible, and from your first altitude-mode flight the last ten seconds before disarm showing the thrust ramp to idle and the `Landing detected` moment.

---

## Part 9: Flight Test

> **Instructor sign-off required before propellers go back on.** Do not proceed on your own.

### 9.1 Pre-Flight Requirements

Every one of these must hold:

- [ ] All bench tests in Parts 6–8 pass
- [ ] `UAS_LOOP_EN` is 7 or below (velocity/position are Lab 4)
- [ ] Sign checks verified by hand on every axis
- [ ] `UAS_MAX_TILT` and `UAS_MAX_RATE` set conservatively
- [ ] `UAS_HOVER_THR` measured on *this* airframe with *this* battery
- [ ] `UAS_MAX_ALT` at 1.5 m (outdoors: never higher, as that is the rangefinder's reach over grass)
- [ ] Kill switch tested this session
- [ ] **Mode-switch handoff tested both directions this session** (Part 5.3 step 6)
- [ ] `COM_OF_LOSS_T` and the offboard-loss action configured (Part 5.1)
- [ ] **Bench-only settings restored:** `COM_DISARM_PRFLT` back to 10, `EKF2_OF_CTRL` back to 1
- [ ] Flying in a netted area or with the vehicle tethered
- [ ] Instructor present

### 9.2 Your Abort Paths

You have three, in increasing order of severity. Know all three before you take off.

| Abort | What it does | When to use it |
|---|---|---|
| **Flip to Stabilized** | Hands control straight back to PX4's controller, mid-flight, instantly | Anything unexpected. This is your default reaction |
| **Kill switch** | Cuts all motor output below the flight-control layer | Your code is doing something violent and you need it to stop *now* |
| **Offboard-loss failsafe** | Fires automatically if the module dies | Not something you trigger; it is the net underneath you |

The first one is the important one, and it is the reason this lab uses Offboard mode. Because PX4's controllers stand down rather than being stopped, they are always sitting there ready to take back over. **Flipping out of your mode is a real, tested recovery**, not a hope.

What it does *not* protect against: a controller that is alive and confidently wrong can put the vehicle into an attitude PX4 cannot recover from before you finish reacting. Tethered or netted, conservative limits, instructor present. The mode switch buys you a second chance, not immunity.

### 9.3 Progression

Fly in this order, one step per flight, landing between each. (If you have a spare 3-position switch, `UAS_LOOP_SW` lets it select these three configurations in flight (see Lab 4 Part 1.4), but for the first flights, one configuration per flight, set as a parameter, is the discipline.)

**For the first two steps, take off in Stabilized.** Get to a stable hover on PX4's controller first, then flip to Offboard to hand over to your code. An untested controller is hardest to survive in exactly the moment you have the least altitude to recover in. Flip back to Stabilized to land. In rate and attitude mode there is no automatic take-off: the throttle stick is thrust, as in Stabilized.

1. `UAS_LOOP_EN = 1`: rate only. Expect to work the sticks constantly; this is normal, rate mode has no self-leveling.
2. `UAS_LOOP_EN = 3`: add attitude. Release the sticks and the vehicle should self-level.
3. `UAS_LOOP_EN = 7`: add altitude. Once the hover in step 2 is solid, **arm on the ground in Offboard** and push the throttle above 60 %: the module's automatic take-off (8.3) lifts the vehicle and hands it to your loop with 0.33 m as the target. Centre the throttle. It should settle and hold height; push up or down and it climbs or descends at up to `UAS_MAX_VZ`, never above `UAS_MAX_ALT`. Stick fully down descends; near the floor your landing logic (8.3) takes over and the disarm switch works. Take-off and landing in your own mode are both expected at this stage.

That is the end point for this lab. The vehicle will still drift horizontally with the sticks centered, because nothing is closing a loop on horizontal velocity yet, so it holds attitude and height but not position. **This is correct behavior, not a bug.** Lab 4 fixes it.

**Flip to Stabilized immediately** at any oscillation, unexpected drift beyond that expected horizontal wander, or anything that surprises you. Land on PX4's controller, then diagnose from the log rather than from memory.

### 9.4 Tuning

Start conservative and increase. These oscillation signatures apply to any cascaded controller, including PX4's own:

| Symptom | Likely cause | Response |
|---|---|---|
| Fast buzzy oscillation (>1 Hz) | Rate loop P or D too high | Reduce `UAS_RAT_RP_P`, then `UAS_RAT_RP_D` |
| Slow wobble (≤1 Hz) | Attitude loop P too high | Reduce `UAS_ATT_RP_P` |
| Sluggish, drifts before correcting | Gains too low | Increase P on the relevant loop |
| Overshoots and settles slowly | Insufficient damping | Increase D on the rate loop |
| Drifts steadily one direction | Missing or insufficient integral | Increase I on the relevant loop |
| Lurches on arm | Integrator not reset | Fix `reset()` |

---

## Lab Deliverables

1. **Source code:** your team's `uas_control` fork on GitLab, pushed, with completed `RateController.cpp`, `AttitudeController.cpp`, and `AltitudeController.cpp`. Confirm the instructor and your GSI are still members of your group with at least Reporter access, or we cannot grade it.
2. **Bench test evidence:** the plots from Parts 6.3, 7.3, and 8.5.
3. **Flight log:** a `.ulg` from your best flight, with the loop configuration you reached noted.
4. **Written analysis (2–3 pages):**
   - Your gain relationship between the rate and attitude loops, and why cascaded loops have a natural bandwidth separation
   - Your derivative-term choice (error vs. measurement) and its consequence
   - Comparison of your gains against Lab 2's autotune results, noting where they differ and why
   - What you observed about horizontal drift with attitude and altitude held, and what a velocity loop would need to measure to correct it (sets up Lab 4)

---

## Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| Module won't build | `CONFIG_MODULES_UAS_CONTROL=y` missing | Check `boards/micoair/h743-v2/default.px4board` (Part 2, step 4) |
| Build error on a uORB field name | PX4 API drift between versions | Check actual field names in `~/uas/PX4-Autopilot/msg/` |
| `uas_control: command not found` | Module not built into firmware | Rebuild and reflash after the board config change |
| `git push` to the module repo is rejected | You cloned the course copy, not your team's fork | `git remote -v`; `origin` must be your group (Part 2, steps 1–3) |
| `active: no` while armed in your mode | Mode slot mismatch | Confirm `UAS_MODE_SLOT` = 14 and the switch position is assigned to Offboard (Part 4) |
| Motors don't respond, state looks fine | Rate loop not enabled | `UAS_LOOP_EN` must have bit 0 set |
| **Offboard mode won't engage / rejected** | Heartbeat not streaming before mode entry | `listener offboard_control_mode` must be fresh with `thrust_and_torque: True`. Confirm the module is running |
| Offboard engages then immediately drops out | Heartbeat too slow or module stalling | Check `COM_OF_LOSS_T`; look for a blocking call in your controller code |
| Erratic motor output, fights itself | Two controllers publishing | Confirm you are in Offboard, not Acro or Stabilized. In Offboard, `mc_rate_control` stands down by itself, so do not stop it manually |
| Vehicle drops on mode entry | Thrust setpoint starting at zero | Check the throttle mapping and `UAS_HOVER_THR`; enter Offboard from a stable hover, not from the ground |
| Vehicle rotates the wrong way | Sign inverted in a controller | Redo the hand sign check (Part 6.2 step 2) |
| Won't translate, refuses stick input | `UAS_LOOP_EN` above 7 with Lab 4 loops stubbed | Set `UAS_LOOP_EN` to 7 or below |
| Drifts horizontally with sticks centered | Expected, as there is no velocity loop in this lab | Not a bug (Part 9.3). Lab 4 addresses it |
| Lurches on arm | Integrator state not cleared | Implement `reset()` in every controller |
| `Disarming denied: not landed` after landing in Offboard | Altitude loop still holding hover thrust on the floor | Implement landing (Part 8.3); until then land in Stabilized. Kill switch always works |
| Hops on the floor when landing | Landing decision toggling on a flickering height | Add hysteresis: once landing, stay landing until the stick comes up (Part 8.3) |
| NaN warning in console | Division by zero, likely `dt` | Check the dt guards; look for uninitialized state |
